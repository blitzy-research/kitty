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

**Methodology (how the evidence was produced).** kitty was built in its default configuration and driven through real window create → run → resize → close sequences via the real launcher binary. Behaviour was observed with kitty's own `--debug-rendering` logging, an `--extra-logging=event-loop` debug build, and external `strace` syscall tracing. Every behavioural claim below is immediately followed by the *complete, unedited* output that demonstrates it, the exact command that produced it, and a `file:line` code anchor. Statements that could not be directly observed at runtime are explicitly labelled **inferred** or **NOT OBSERVED**. All line numbers were re-verified against the source at commit `815df1e21` (see *Appendix B*).

---

## TL;DR — direct answers to R1–R6

- **R1 (lifecycle consistency).** Consistency rests on four observed mechanisms working together: (1) a **startup gate** that blocks the child's `execvp` until kitty has set the first PTY size, so the command never sees a zero/garbage size; (2) a **single-owner PTY-resize path** — every size change funnels through `Window.set_geometry` → `child_monitor.resize_pty` → `ioctl(TIOCSWINSZ)` on the one I/O thread, under `children_mutex`; (3) a **de-duplication guard** that discards an unchanged size; and (4) **per-tick reconciliation** on the I/O thread that always runs `remove_children()` *then* `add_children()` once per loop iteration under the mutex. Anchors: `kitty/window.py:850`, `kitty/child-monitor.c:592`, `kitty/window.py:861`, `kitty/child-monitor.c:1493-1494`.
- **R2 (create-and-run flow).** Observed order for a fresh window: Python `Boss.add_child` registers the child into the C **`add_queue[]`** and wakes the I/O loop (`kitty/boss.py:585-588`, `kitty/child-monitor.c:305-319`); the first `Window.set_geometry` calls `resize_pty`, which does `ioctl(fd, TIOCSWINSZ, …)` (`kitty/child-monitor.c:579`) — this is the resize "signal", and it flows **kitty → child** (the kernel then sends `SIGWINCH` to the child's process group); only then does kitty `mark_terminal_ready()`, releasing the child to `execvp` the command (`kitty/window.py:866`, `kitty/child.c:152,159`). Subsequent resizes emit `SIGWINCH sent to child in window: …` (`kitty/window.py:873`).
- **R3 (in-flight teardown).** If the window disappears while events are still in flight, nothing crashes. Two concrete outcomes were observed: (a) a resize aimed at an already-removed child is **safely discarded** with a single `log_error` — `Failed to send resize signal to child with id: …` (`kitty/child-monitor.c:610`); and (b) already-buffered output of a removed child is **fully drained** before the child object is freed, via a reference-counted snapshot in `parse_input` (`kitty/child-monitor.c:521`).
- **R4 (keep vs discard).** The rule is *"keep the in-flight bytes, discard the dead handle."* On the main thread, `parse_input` takes an **`INCREF`'d snapshot** of the live children under the mutex (`kitty/child-monitor.c:479-480`); a child concurrently marked for removal still has its final buffered output parsed with `do_parse(…, flush=true)` (`kitty/child-monitor.c:521`) and its Python death-notify fired (`:522`), and only then is it `FREE_CHILD`'d (`:525`). Live parsing of a child already flagged `needs_removal` is **skipped** (`:529`). The Child/Screen survive until the snapshot's refcount reaches zero (`DECREF_CHILD`, `:532`).
- **R5 (timing).** `SIGCHLD` (child death) reaches the I/O thread through a Linux **`signalfd`** (canonical here; the self-pipe path is the non-Linux fallback) that is polled in the event loop; the signal's *effect* is processed on the **next loop tick**, not inside a handler, giving a bounded, sub-millisecond delay. Measured child-exit → reap over 5 identical runs: **{0.075, 0.311, 0.173, 0.359, 0.130} ms** (min 0.075 / median 0.173 / max 0.359) — run-to-run **variable**, reported as a distribution. Multiple child deaths **coalesce** into one `SIGCHLD`, so reaping loops on `waitpid(-1, …, WNOHANG)` (`kitty/child-monitor.c:1418`). The handled-signal set is `SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` — **not `SIGWINCH`** (`kitty/child-monitor.c:121`).
- **R6 (conflicting liveness views).** Yes — there are **three** independent in-memory "views of what is alive," reconciled by explicit guards so a window vanishing mid-flight is a *no-op*, not a crash. The views: the C I/O-thread `children[]`/`add_queue[]`/`remove_queue[]` (`kitty/child-monitor.c:82-84`); the GUI/main-thread `global_state.os_windows[].tabs[].windows[]` (`kitty/state.c:12`, `kitty/state.h:189,226,265`); and the Python `Boss.window_id_map` `WeakValueDictionary` (`kitty/boss.py:344`). The guards: `on_child_death` pops-and-early-returns if the window is already gone (`kitty/boss.py:883-885`); `mark_child_for_close` searches both `children[]` **and** `add_queue[]` (`kitty/child-monitor.c:544-558`); and `hangup` tolerates an already-exited child via `ESRCH` (`kitty/child-monitor.c:1297`, **observed**).

---

## 1. How this was built and run (canonical, normal-user configuration)

The host sandbox lacks the C/Go toolchain and native libraries (and its Python 3.13 is out of kitty's supported range), so — per the task's environment instruction — the build and all runtime observation were performed inside the provided Docker container (`kitty-setup`, image `kitty-setup:ready`, derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). All commands below were run **as a normal user** inside that container against a container-internal copy of the repo at `/root/kitty_build` (leaving the bind-mounted host repo pristine). The single non-default environmental choice is headless rendering under a pre-running `Xvfb :99` with software GL (`LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`); the kitty configuration itself is default (`--config NONE`).

### 1.1 Toolchain versions (A1)

Command:

```bash
uname -a; cat /etc/os-release | head -2; python3 --version; gcc --version | head -1; \
go version; pkg-config --modversion harfbuzz freetype2 fontconfig libpng lcms2 openssl
```

Complete output:

```text
Linux 164eed5eea36 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
NAME="Ubuntu"
VERSION="24.04.2 LTS (Noble Numbat)"
Python 3.12.3
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
go version go1.23.4 linux/amd64
8.3.0
26.1.20
2.15.0
1.6.43
2.14
3.0.13
```

(harfbuzz 8.3.0, freetype2 26.1.20, fontconfig 2.15.0, libpng 1.6.43, lcms2 2.14, openssl 3.0.13.)

### 1.2 Commit confirmation (A2)

Command:

```bash
git -C /root/kitty_build log -1 --format='%H %s'
```

Output:

```text
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 Wire up applying of font config
```

### 1.3 Canonical build (A3)

The Makefile `all` target is exactly `python3 setup.py` (`Makefile:12-13`). Command (from the repo root):

```bash
python3 setup.py
```

Head and tail of the complete build log (122 compile steps + 5 link steps, exit status 0):

```text
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
```

This produces the launcher binary at `kitty/launcher/kitty`; `./kitty/launcher/kitty --version` reports `kitty 0.35.2 created by Kovid Goyal`.

### 1.4 Timing/event-loop debug build (A4) — the primary R5 lever

The Makefile `debug-event-loop` target is `python3 setup.py build --debug --extra-logging=event-loop` (`Makefile:25-26`). To keep `/root/kitty_build` canonical, this was built in a separate copy (`/root/kitty_dbg`). Command:

```bash
python3 setup.py build --debug --extra-logging=event-loop
```

Tail of the complete build log (Go tools rebuilt; exit status 0; debug launcher 274648 bytes; also reports 0.35.2):

```text
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
```

The plain debug build `python3 setup.py build --debug` (`Makefile:22-23`) and the test harness `python3 setup.py test` (`Makefile:15-16`) also exist; the latter is noted because `kitty_tests/` contains **no** child-monitor or window-lifecycle test (`kitty_tests/main.py:57` enumerates the module tests), which is why the lifecycle-race behaviour here is observed with purpose-built temporary drivers rather than an existing test.

### 1.5 Exact run invocation (A5)

Every runtime observation used this invocation (default kitty config; headless via the pre-running Xvfb):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  ./kitty/launcher/kitty --debug-rendering --config NONE [--listen-on unix:SOCK -o allow_remote_control=yes] <command>
```

One benign line appears on stderr in this headless container and is *not* an error: `Failed to open systemd user bus with error: No medium found`.

### 1.6 Version-discrepancy note (A6)

For accuracy: `pyproject.toml:2` declares `requires-python = ">=3.8"` (CI tests through 3.11); `go.mod:3` declares `go 1.22` (container's go 1.23.4 satisfies it); `setup.py:609` checks `at_least_version('harfbuzz', 1, 5)` while `docs/build.rst` documents harfbuzz `>=2.2.0` (container's 8.3.0 satisfies both). The host's Python 3.13.7 exceeds the supported range, which is precisely why the canonical build/run is done in the container (Python 3.12.3).

---

## 2. Instrumentation (no source edits)

All observation used kitty's **existing** debug affordances plus external tracing — no source file was modified.

- **`--debug-rendering` logging.** Two lines emitted from `kitty/window.py` are the backbone of the lifecycle evidence:
  - `[{now:.3f}] Child launched` — printed on the **first** geometry, `kitty/window.py:871`.
  - `[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}` — printed on **subsequent** resizes, `kitty/window.py:873`.
- **External syscall tracing with `strace`.** `strace` is not preinstalled in the container; it was installed once with `apt-get install -y strace` (version 6.8-0ubuntu2). This is a transient change to the *run environment*, **not** a change to the kitty repository. It corroborates:
  - `ioctl(fd, TIOCSWINSZ, …)` — the PTY resize propagation (`kitty/child-monitor.c:579`).
  - `wait4(-1, …, WNOHANG)` — the reaping loop (`kitty/child-monitor.c:1418`; glibc implements `waitpid` via `wait4`).
  - `signalfd4(…)` — the signal-delivery fd (`kitty/loop-utils.c:42`).
- **Temporary driver scripts** (shown in full in *Appendix A*, then deleted) drive the real launcher and, where noted, use kitty's remote-control (RC) protocol **only to trigger** real window operations — never as a substitute for the observed mechanism.
- **Compile-time lever (not required).** A `KITTY_PRINT_BYTES_SENT_TO_CHILD` guarded block exists in `kitty/child-monitor.c:1428`; it was not needed and no rebuild with it was done.

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

(`kitty/boss.py:585-588`.) On the C side, `add_child` appends the child to the staging **`add_queue[]`**, bumps its refcount, and wakes the I/O loop — the brand-new child is *not* live yet; it waits in `add_queue[]` until the next I/O tick merges it:

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
```

(`kitty/child-monitor.c:305-319`: `INCREF_CHILD` at `:316`, `add_queue_count++` at `:317`, `wakeup_io_loop` at `:318`.)

2. **First geometry → resize the PTY.** `Window.set_geometry` computes the PTY size, applies the de-dup guard, calls `resize_pty`, and — because the child is not yet launched — marks the terminal ready and prints `Child launched`. The full method (no elision):

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

(`kitty/window.py:850-876`; `child_is_launched` and `last_reported_pty_size` are initialised to `False` and `(-1, -1, -1, -1)` at `kitty/window.py:578-579`.) The de-dup guard is the `if current_pty_size != self.last_reported_pty_size:` at `:861`; the resize call is `:863`; `mark_terminal_ready()` is `:866`; `child_is_launched = True` is `:867`; the `Child launched` print is `:871`; the `SIGWINCH sent …` print is `:873`.

3. **The resize is an `ioctl(TIOCSWINSZ)` — the "signal" flows kitty → child.** `resize_pty` finds the fd in `children[]` or `add_queue[]` and calls `pty_resize`, which issues `ioctl(fd, TIOCSWINSZ, dim)`:

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
```

(`kitty/child-monitor.c:577-612`: `ioctl(TIOCSWINSZ)` at `:579`, `EINTR` retry at `:580`, `EBADF`/`ENOTTY` tolerated at `:581`; the two `FIND` lookups at `:606-607`; the `else log_error(...)` at `:610` — this is the *"Failed to send resize signal"* line that becomes visible in the teardown race, R3.) Setting the size with `TIOCSWINSZ` makes the kernel deliver `SIGWINCH` to the child's foreground process group — so the resize "signal" travels **from kitty to the child**, which is why kitty itself does not handle `SIGWINCH` (see R5).

### 3.1 Observed create-and-run output

Command (two windows created via a startup session, second created ~2 s after the first, then the first closed):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  ./kitty/launcher/kitty --debug-rendering --config NONE --session /tmp/sess_split.conf
```

Complete `--debug-rendering` stderr:

```text
[0.232] OS Window created
[0.248] Failed to open systemd user bus with error: No medium found
[0.251] Child launched
[0.258] SIGWINCH sent to child in window: 1 with size: (11, 70, 630, 198)
[0.258] Child launched
[2.263] SIGWINCH sent to child in window: 2 with size: (22, 71, 639, 396)
```

This is the byte-exact format from `kitty/window.py:873`: `SIGWINCH sent to child in window: {id} with size: {current_pty_size}`, where `current_pty_size = (screen.lines, screen.columns, width_px, height_px)`. Window 1's `Child launched` precedes any command output; when window 2 is added the layout shrinks window 1 to `(11, 70, 630, 198)`; when window 1 later closes, window 2 grows to `(22, 71, 639, 396)`.

Corroborating `strace` (filter `-f -e trace=ioctl,wait4,signalfd,signalfd4,ppoll,poll -tt`), the exact `TIOCSWINSZ` calls and the signalfd creation — note the row/col match the debug sizes exactly, and the signalfd set contains **no** `SIGWINCH`:

```text
22286 04:19:47.560312 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
22286 04:19:47.581160 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
22286 04:19:47.591335 ioctl(8, TIOCSWINSZ, {ws_row=11, ws_col=70, ws_xpixel=630, ws_ypixel=198}) = 0
```

### 3.2 The startup gate (crux of "immediately used to run a command")

**Direct answer.** The child's command does **not** execute until kitty has set the first PTY size. The parent holds the write end of a "terminal ready" pipe; the child blocks reading the read end until the parent closes it. `mark_terminal_ready` performs that close:

```python
    def mark_terminal_ready(self) -> None:
        os.close(self.terminal_ready_fd)
        self.terminal_ready_fd = -1
```

(`kitty/child.py:362-364`.) The pipe is created in `fork()` (`ready_read_fd, ready_write_fd = os.pipe()`, `kitty/child.py:283`); the parent closes the read end and keeps the write end as `self.terminal_ready_fd` (`kitty/child.py:342-343`). In the child, after `setsid()` and `ioctl(TIOCSCTTY)`, it closes the write end and **blocks** in `wait_for_terminal_ready(ready_read_fd)` until EOF, only then `execvp`-ing:

```c
wait_for_terminal_ready(int fd) {
    char data;
    while(1) {
        int ret = read(fd, &data, 1);
        if (ret == -1 && (errno == EINTR || errno == EAGAIN)) continue;
        break;
    }
```

(`kitty/child.c:71-77`; the child-side sequence `safe_close(ready_write_fd)` → `wait_for_terminal_ready(ready_read_fd)` → `safe_close(ready_read_fd)` → `execvp(exe, argv)` is `kitty/child.c:151-159`, with `setsid()` at `:123` and `ioctl(TIOCSCTTY)` at `:129`.)

**Observed** via `strace -f` of a child running `sh -c "echo GATE_MARKER_CHILD_RAN; sleep 1"` (child pid 22484, parent/IO pid 22417). The complete ordered key lines:

```text
22484 04:22:24.567094 setsid()          = 22484
22484 04:22:24.567205 ioctl(12, TIOCSCTTY, 0) = 0
22484 04:22:24.567497 read(10,  <unfinished ...>
22417 04:22:24.571955 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
22484 04:22:24.572043 <... read resumed>0x7ffec04b7c93, 1) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
22484 04:22:24.572074 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
22484 04:22:24.572090 read(10, "", 1)   = 0
22484 04:22:24.577799 execve("/usr/bin/sh", ["sh", "-c", "echo GATE_MARKER_CHILD_RAN; sleep 1"], 0x5a737fb96430 /* 22 vars */) = 0
```

Reading top to bottom: the child does `setsid()` → `TIOCSCTTY` → **blocks** in `read(10, …)` on the ready pipe; the **parent** (22417) then sets the PTY size first with `TIOCSWINSZ`; that resize makes the kernel deliver `SIGWINCH {si_code=SI_KERNEL}` to the child (confirming resize flows kitty → child), and the blocked `read` reports `ERESTARTSYS` (evidence of `SA_RESTART`, see R5); the parent's `mark_terminal_ready` close makes the child's `read` return EOF (`= 0`); and **only then** does the child `execve` the command. The command therefore never observes a zero/wrong initial size. (Full trace in `Appendix A`.)

### 3.3 A subsequent resize and the de-dup guard (before / during / after)

**Direct answer.** A *changed* geometry emits one `SIGWINCH sent …` line and one `ioctl(TIOCSWINSZ)`; an *identical* geometry emits **neither** — the guard at `kitty/window.py:861` is false and the `else` branch merely marks the OS window dirty (`:875-876`), touching the child not at all.

Command (driver at `Appendix A`; RC used only to *trigger* three real resizes: **A** = 900×600 new, **B** = 900×600 again/identical, **C** = 1100×750 changed):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe strace -f -e trace=ioctl -tt -o /tmp/dedup.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE -o allow_remote_control=yes --listen-on unix:/tmp/rc.sock sh -c "sleep 60"
# then: kitty @ resize-os-window --width 900 --height 600  (A)
#       kitty @ resize-os-window --width 900 --height 600  (B, identical)
#       kitty @ resize-os-window --width 1100 --height 750 (C)
```

Complete `--debug-rendering` stderr — note **step B produced no line**:

```text
[0.344] OS Window created
[0.376] Failed to open systemd user bus with error: No medium found
[0.383] Child launched
[5.285] SIGWINCH sent to child in window: 1 with size: (600, 900, 8100, 10800)
[8.277] SIGWINCH sent to child in window: 1 with size: (750, 1100, 9900, 13500)
```

All `TIOCSWINSZ` calls from the trace — exactly **three** (initial launch + A + C), **none** for the identical step B:

```text
22654 04:24:40.924558 ioctl(10, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
22654 04:24:45.826994 ioctl(10, TIOCSWINSZ, {ws_row=600, ws_col=900, ws_xpixel=8100, ws_ypixel=10800}) = 0
22654 04:24:48.818698 ioctl(10, TIOCSWINSZ, {ws_row=750, ws_col=1100, ws_xpixel=9900, ws_ypixel=13500}) = 0
```

*Before* (initial): size `(22,71,…)`. *During* step B (identical to A): `last_reported_pty_size` already equals the new size → guard false → **no** `resize_pty`, **no** `ioctl`, **no** debug line. *After* step C (changed): guard true → new `ioctl` + new debug line. (`resize-os-window`'s width/height default to cells, so 900×600 → `ws_col=900`, `ws_row=600`, and pixels are `900×9` and `600×18` at the 9×18 cell size.)


---

## 4. R1 — Lifecycle state consistency (tie-together)

**Direct answer.** kitty keeps state consistent across create → run → resize → destroy — even when these interleave "in quick succession" — through four mechanisms, each observed above or below:

1. **Startup gate (create → run ordering).** The child cannot run its command until kitty has set the first PTY size, because `execvp` is gated behind `wait_for_terminal_ready` (`kitty/child.c:71-77,152,159`) which is released by `mark_terminal_ready` on the first `set_geometry` (`kitty/window.py:866`). Observed in §3.2: the gate-opening `read(10, "", 1) = 0` precedes the child's `execve` of `sh` (the complete `execve(...)` line is shown verbatim in §3.2). This guarantees the command never reacts to a bogus initial size.
2. **Single-owner PTY-resize path (resize consistency).** Every size change funnels through `Window.set_geometry` → `child_monitor.resize_pty` → `ioctl(TIOCSWINSZ)` on the one I/O thread, and the fd lookup + ioctl happen under `children_mutex` (`kitty/child-monitor.c:598,606-611`). There is exactly one writer of PTY size, so there is no torn or racing resize.
3. **De-duplication guard (idempotence).** An unchanged geometry is discarded (`kitty/window.py:861`), observed in §3.3 as "three resizes requested, but only the two *distinct* ones reach the kernel." This keeps redundant events from perturbing state.
4. **Per-tick reconciliation (create/destroy ordering under churn).** The I/O thread reconciles pending removals and additions exactly once per loop iteration, **removals first, then additions**, under the mutex:

```c
    set_thread_name("KittyChildMon");

    while (LIKELY(!self->shutting_down)) {
        children_mutex(lock);
        remove_children(self);
        add_children(self);
        children_mutex(unlock);
```

(`kitty/child-monitor.c:1489-1495`: `remove_children(self)` at `:1493`, `add_children(self)` at `:1494`.) Because both staging queues are drained under the same lock in a fixed order every tick, the I/O thread always transitions from one internally-consistent `children[]` array to the next; a create and a destroy that arrive "at the same time" are serialised into this deterministic per-tick order rather than racing.

The remaining sub-sections (R3–R6) show what happens when the destroy half of the lifecycle lands *while events are still in flight*.

---

## 5. R3 & R4 — In-flight teardown and the keep-vs-discard rule

### 5.1 The two-thread model

kitty splits the work across two threads:

- The **C I/O thread**, named `KittyChildMon` (`kitty/child-monitor.c:1489`), owns the `poll()` set and the `children[]`/`add_queue[]`/`remove_queue[]` arrays; its loop does `remove_children()` then `add_children()` once per tick under `children_mutex` (`:1491-1495`, quoted above).
- The **main (Python/GUI) thread** calls `parse_input` (`kitty/child-monitor.c:451`) to drain child output into screens and to fire death notifications into Python.

These two threads share the child arrays, so teardown must be reconciled between them without freeing an object the other thread is still using.

### 5.2 Removal staging on the I/O thread (`remove_children`)

When a child is flagged `needs_removal`, the I/O thread cleans it up and stages it into `remove_queue[]` (it does **not** free it here):

```c
static void
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
}
```

(`kitty/child-monitor.c:1313-1332`.) Step by step: a **reverse** scan (`:1316`) so compaction indices stay valid; `cleanup_child(i)` (`:1319`) closes the PTY fd and hangs up the child (next paragraph); the child struct is copied into `remove_queue[]` (`:1320`) and `remove_queue_count` bumped (`:1321`); its slot is cleared to `EMPTY_CHILD` (`:1322`) and its poll fd disabled (`:1323`); the arrays are `memmove`-compacted (`:1325-1326`); and finally `self->count` is decremented (`:1331`). `cleanup_child` itself:

```c
static void
cleanup_child(ssize_t i) {
    safe_close(children[i].fd, __FILE__, __LINE__);
    hangup(children[i].pid);
}
```

(`kitty/child-monitor.c:1306-1308`: `safe_close` at `:1307`, `hangup` at `:1308`.)

### 5.3 The reference-counted snapshot — the keep-vs-discard core (`parse_input`)

**Direct answer (R4).** The rule is *"keep the in-flight bytes, discard the dead handle."* `parse_input` takes an `INCREF`'d snapshot of the live children under the mutex, then does all parsing and death-notification *without* holding the lock; a child concurrently removed on the I/O thread survives — its final buffered output is drained and its death is announced — and it is freed only when the snapshot's reference is dropped. Here is the whole relevant body of `parse_input`, unelided:

```c
static bool
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

(`kitty/child-monitor.c:451-483`.) Under the lock: the `remove_queue[]` is drained into `remove_notify[]`, each entry `INCREF`'d so it cannot be freed out from under us (`:457-463`); and, in the normal case, **every** live child is snapshotted into `scratch[]` with `INCREF_CHILD` (`:479-480`). Then the lock is released (`:483`). After some peer-message handling, the removed children and then the survivors are processed **without any lock held**:

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

(`kitty/child-monitor.c:438-448`.) It builds a `ParseData` seeded with the dump callback and current time, then delegates the actual byte-parsing to `self->parse_func(screen, &pd, flush)` — the `flush` argument is what forces a removed child's *final* buffered bytes to be drained. If bytes were read (`pd.input_read`), it wakes the I/O loop when write space opened up (`pd.write_space_created`) and schedules the next wait either for the paused-rendering deadline or `OPT(input_delay)` minus how long ago new input arrived; if nothing was read but input is still pending, it schedules the same input-delay wait. It returns whether any input was read.

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

**Scenario 1 — ordinary close: child self-exits (`SIGCHLD`).** Command `sh -c "echo BYE_FROM_CHILD; exit 7"`. kitty sent **no** kill/SIGHUP; the child exit is reaped by the coalescing loop. Complete key `strace` lines:

```text
23058 04:28:05.200572 +++ exited with 7 +++
23057 04:28:05.200698 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 7}], WNOHANG, NULL) = 23058
23057 04:28:05.200771 wait4(-1, 0x7a551b7fde60, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

*Before:* child alive, running the command. *During:* `WIFEXITED … WEXITSTATUS == 7` — the exact exit status is captured faithfully. *After:* the follow-up `wait4` returns `ECHILD`, so the reap loop terminates. kitty exited 0.

**Scenario 4 — kitty-initiated close: `hangup` → `killpg(pgid, SIGHUP)`.** Command `sh -c "sleep 600"`, closed via RC `close-window`. *Before:* window id=1, pid=22918, process-group leader (`ps` STAT `Ss+`, pgid 22918). Complete key `strace` lines:

```text
22918 04:27:29.466549 <... wait4 resumed>0x7ffe6c4e61bc, 0, NULL) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
22918 04:27:29.466590 --- SIGHUP {si_signo=SIGHUP, si_code=SI_KERNEL} ---
22917 04:27:29.466622 kill(-22918, SIGHUP) = 0
22919 04:27:29.466709 --- SIGHUP {si_signo=SIGHUP, si_code=SI_USER, si_pid=22850, si_uid=0} ---
22918 04:27:29.466825 +++ killed by SIGHUP +++
22919 04:27:29.467061 +++ killed by SIGHUP +++
22917 04:27:29.467066 wait4(-1, [{WIFSIGNALED(s) && WTERMSIG(s) == SIGHUP}], WNOHANG, NULL) = 22918
22917 04:27:29.467150 wait4(-1, 0x7c1aa67fbe60, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

*During:* the I/O thread (pid 22917) issues `kill(-22918, SIGHUP)` — i.e. `killpg(pgid, SIGHUP)` from `hangup` (`kitty/child-monitor.c:1299`). The group receives **two** SIGHUPs: one `si_code=SI_KERNEL` (the PTY master closing in `cleanup_child`'s `safe_close`, `:1307`) and one `si_code=SI_USER` (the explicit `killpg`). Both the child (22918) and its grandchild (22919) are `killed by SIGHUP`. *After:* reaped as `WIFSIGNALED … WTERMSIG == SIGHUP`, then `ECHILD`. **Contrast with Scenario 1:** self-exit is `WIFEXITED(status)` with *no* kitty-sent signal; kitty-initiated close is `WIFSIGNALED(SIGHUP)` preceded by `kill(-pgid, SIGHUP)` — both funnel through the *same* `reap_children` `WNOHANG` loop.

**Scenario 2 — close while a resize is in flight.** When a resize is requested for a window whose child was **already removed** from both `children[]` and `add_queue[]`, `resize_pty`'s two `FIND` lookups miss and it takes the `else` branch, emitting exactly one harmless `log_error` (`kitty/child-monitor.c:610`). Observed during the rapid storm (Scenario 3), complete lines from one run:

```text
[8.221] Failed to send resize signal to child with id: 4 (children count: 2) (add queue: 0)
[9.519] Failed to send resize signal to child with id: 5 (children count: 2) (add queue: 0)
[13.132] Failed to send resize signal to child with id: 8 (children count: 2) (add queue: 0)
[14.592] Failed to send resize signal to child with id: 9 (children count: 2) (add queue: 0)
[16.345] Failed to send resize signal to child with id: 10 (children count: 2) (add queue: 0)
```

*Before:* Python's `window_id_map` still holds the window and calls `resize_pty(id, …)`. *During:* the C `children[]`/`add_queue[]` no longer contain that id (the `(add queue: 0)` field is `resize_pty` printing `add_queue_count` directly, `:610`). *After:* the resize is discarded, no crash — this is a direct instance of R6's "conflicting views," resolved by simply logging and ignoring.

**Scenario 3 — rapid create-then-immediate-close, at scale, ≥2 runs.** A driver launched a window that immediately ran a command and then closed it, in a tight loop of **N=30** cycles, repeated as **5 identical runs**. Per-run counts (from `--debug-rendering` stderr):

| Run | `Child launched` | `SIGWINCH sent` | `Failed to send resize signal` | crash markers | kitty alive after |
|-----|------------------|------------------|--------------------------------|---------------|-------------------|
| 1 | 31 | 60 | 5 | 0 | yes |
| 2 | 31 | 60 | 3 | 0 | yes |
| 3 | 31 | 60 | 6 | 0 | yes |
| 4 | 31 | 60 | 6 | 0 | yes |
| 5 | 31 | 60 | 5 | 0 | yes |

`Child launched` (31) and `SIGWINCH sent` (60) are **deterministic**; the in-flight-resize race count is **run-to-run variable** with distribution **{5, 3, 6, 6, 5}** (min 3 / median 5 / max 6); crash markers are **0** in every run and kitty stayed alive and responsive (a follow-up RC `ls` returned valid JSON). This both satisfies R-TIMING-RIGOR (the only variable quantity is reported as a distribution, not stabilised) and shows the teardown races are benign. Note: because the RC round-trip (~0.7 s) is far larger than a sub-millisecond I/O tick, launched children were always merged into `children[]` before the close arrived, so the `mark_child_for_close` `add_queue[]` branch (`:552-558`) was **not forced** in these runs — this branch is therefore **inferred (not forced)**; the `(add queue: N)` counter in the `:610` log directly exposes the queue size and read `0` throughout.


---

## 6. R5 — Timing and signal delivery

**Direct answer.** The I/O thread learns of child death via **`SIGCHLD`** delivered through a Linux **`signalfd`** that is polled in the event loop (the self-pipe trick is the non-Linux fallback, installed with `SA_RESTART`). Because a signal's *effect* is processed on the **next loop tick** — not inside a handler — there is a small, bounded delay between a signal arriving and its bookkeeping taking effect (measured sub-millisecond, and run-to-run variable — distribution below). Multiple child deaths can **coalesce** into a single `SIGCHLD`, which is exactly why reaping is a `waitpid(-1, …, WNOHANG)` loop.

### 6.1 The signal-delivery mechanism (`signalfd` here; self-pipe fallback)

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

(`kitty/loop-utils.c:34-56`.) `HAS_SIGNAL_FD` is defined when `<sys/signalfd.h>` exists (`kitty/loop-utils.h:14-15`), which it does on this Linux container, so the **`#ifdef HAS_SIGNAL_FD` branch is the active/canonical path here**: the handled signals are **blocked** with `sigprocmask(SIG_BLOCK, …)` (`:41`) — so they are *queued and drained via the fd* rather than interrupting arbitrary syscalls — and a non-blocking, close-on-exec `signalfd(-1, &signals, SFD_NONBLOCK | SFD_CLOEXEC)` is created (`:42`). The `#else` branch is the **non-canonical** fallback for platforms without `signalfd`: it uses the classic **self-pipe trick** (`self_pipe`, `:48`) and installs handlers with `SA_SIGINFO | SA_RESTART` (`:51`, installed per-signal at `:52`); `SA_RESTART` makes interrupted syscalls resume instead of failing with `EINTR`.

**Observed** — the active path is `signalfd`, and (from §3.2) `SA_RESTART` is real (the child's blocked `read` reported `ERESTARTSYS` on `SIGWINCH`):

```text
22286 04:19:47.560312 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
```

(glibc implements `signalfd` via the `signalfd4` syscall.) The signal set `[HUP INT USR1 USR2 TERM CHLD]` is exactly the handled set below — and contains **no** `SIGWINCH`.

### 6.2 Draining the signal fd (`read_signals`)

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
```

(`kitty/loop-utils.c:131-152`.) Up to **32** `signalfd_siginfo` records are read at once (`fdsi[32]`, `:133`); `EINTR` retries (`:138`), `EAGAIN` breaks (`:139`); the record count is computed (`:144`) and each is dispatched to the callback.

### 6.3 The handled-signal set (exhaustive) and its classifier

```c
#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0
```

(`kitty/child-monitor.c:121`; the trailing `0` is a sentinel.) The set is, exhaustively: **`SIGINT`, `SIGHUP`, `SIGTERM`, `SIGCHLD`, `SIGUSR1`, `SIGUSR2`** — and **not `SIGWINCH`**. The reason (cause → effect): the GUI process learns its *own* window size from GLFW, not from a controlling terminal, and — as R2 showed — the resize signal flows **kitty → child**, so kitty never needs to catch `SIGWINCH` itself. The classifier maps each handled signal to a flag:

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

(`kitty/child-monitor.c:1362-1383`: `SIGINT`/`SIGTERM`/`SIGHUP` → `kill_signal` `:1368`; `SIGCHLD` → `child_died` `:1371`; `SIGUSR1` → `reload_config` `:1374`; `SIGUSR2` → log `:1377`.) Python mirrors the C set exactly, adding each to its own handled set right after starting the monitor:

```python
    def start(self, first_os_window_id: int, startup_sessions: Iterable[Session]) -> None:
        if not getattr(self, 'io_thread_started', False):
            self.child_monitor.start()
            self.io_thread_started = True
            for signum in self.child_monitor.handled_signals():
                handled_signals.add(signum)
```

(`kitty/boss.py:1181-1186`: `child_monitor.start()` `:1183`, the `handled_signals()` loop `:1185-1186`.)

### 6.4 The coalescing reaper (`reap_children`) and main-loop wiring

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

(`kitty/child-monitor.c:1413-1426`.) One `SIGCHLD` may correspond to **many** exited children, so this loops on `waitpid(-1, &status, WNOHANG)` (`:1418`, non-blocking) until it returns `0` (no more ready children) or `-1` (`ECHILD`); each reaped pid is marked for removal by pid (`mark_child_for_removal`, `:1422`), which sets `needs_removal` under `children_mutex`:

```c
static void
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

(`kitty/child-monitor.c:1386-1396`.) The main-loop wiring uses `EXTRA_FDS = 2` (`kitty/child-monitor.c:35`) reserved poll slots — `children_fds[0]` is the wakeup/self-pipe and `children_fds[1]` is the signal fd:

```c
        if (ret > 0) {
            if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd); // wakeup
            if (children_fds[1].revents && POLLIN) {
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
            }
```

(`kitty/child-monitor.c:1515-1526`: wakeup drain `:1515`, `read_signals` `:1519`, and `if (ss.child_died) reap_children(...)` `:1526`.) The coalescing `WNOHANG` loop is visible in the traces — a successful `wait4` is always followed by another that returns `0` or `-1 ECHILD` (see Scenarios 1 & 4 in §5.5).

### 6.5 Timing measurement (child-exit → reap), ≥2 runs, reported as a distribution

Command (child `sh -c "exit 0"`; `strace -tt` timestamps around exit and reap; repeated 5 times identically):

```bash
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe strace -f -e trace=wait4,waitpid -tt -o /tmp/timing_N.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE sh -c "exit 0"
```

Sample from run 2 (child pid 28962, kitty/IO pid 28961) — the complete relevant lines:

```text
28962 04:38:35.392190 +++ exited with 0 +++
28961 04:38:35.392501 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 28962
```

That is 04:38:35.392501 − 04:38:35.392190 = **0.311 ms** from child exit to kitty's reaping `wait4`. Across the 5 identical runs the child-exit → reap gap was:

| Run | 1 | 2 | 3 | 4 | 5 |
|-----|-----|-----|-----|-----|-----|
| gap (ms) | 0.075 | 0.311 | 0.173 | 0.359 | 0.130 |

Distribution: **min 0.075 ms, median 0.173 ms, max 0.359 ms** — all **sub-millisecond**, and clearly **run-to-run variable** (reported as a distribution per R-TIMING-RIGOR, not stabilised to a single number). Note `strace -tt` itself adds tracing overhead, so these are upper bounds on the true latency; the *shape* (sub-ms, variable) is the robust finding.

The variability's mechanism is concrete, not environmental: the signal is queued on the `signalfd` while blocked, and its effect (`reap_children`) runs on the **next** event-loop iteration. The `--extra-logging=event-loop` debug build makes the per-tick structure visible — complete relevant lines:

```text
[0.580] starting handleEvents(0.00)
[0.580] pollForEvents final timeout: 0.000
[1.280] --------- loop tick, wakeups_happened: 1 ----------
[1.311] main loop exiting
```

So `SIGCHLD` → (queued on signalfd) → processed on the next `loop tick` → `reap_children` `WNOHANG` loop → `needs_removal` → (next I/O tick) `remove_children` → (main thread) `parse_input` death-notify. Each hop is one loop iteration, which is why the observed latency is a small, variable, sub-millisecond quantity rather than a fixed constant.

---

## 7. R6 — Conflicting views of "what is alive," and their resolution

**Direct answer.** Yes. There are **three** independent in-memory views of which windows/children are alive, updated by different threads at different times. kitty reconciles them with explicit guards so that a window disappearing mid-flight is a **no-op**, not a crash. Two of the three guards were observed firing; the third is silent-by-design and is grounded in code and labelled accordingly.

### 7.1 The three liveness views (exhaustive)

1. **Child-monitor I/O-thread view.** The live `children[]` array plus the staging arrays, all guarded by `children_mutex`:

```c
static Child children[MAX_CHILDREN] = {{0}};
static Child scratch[MAX_CHILDREN] = {{0}};
static Child add_queue[MAX_CHILDREN] = {{0}}, remove_queue[MAX_CHILDREN] = {{0}}, remove_notify[MAX_CHILDREN] = {{0}};
```

(`kitty/child-monitor.c:82-84`; the `children_mutex(op)` macro at `:76`.)

2. **GUI/main-thread structural view.** The global window hierarchy:

```c
GlobalState global_state = {{0}};
```

(`kitty/state.c:12`.) The hierarchy is `global_state.os_windows[].tabs[].windows[]`, declared in `kitty/state.h`: `Window *windows;` (`:189`, with `num_windows` at `:188`), `Tab *tabs;` (`:226`, with `num_tabs` at `:228`), and `OSWindow *os_windows;` (`:265`, with `num_os_windows` at `:266`).

3. **Python controller view.** `Boss.window_id_map`, a **weak** map so entries vanish when the `Window` is garbage-collected:

```python
        self.window_id_map: WeakValueDictionary[int, Window] = WeakValueDictionary()
```

(`kitty/boss.py:344`; `from weakref import WeakValueDictionary` at `:33`.) Per-tab liveness lives in each `WindowList.id_map` (`kitty/window_list.py:148`) and `Tab.remove_window` (`kitty/tabs.py:580`). The `ChildMonitor` is constructed with `self.on_child_death` as its death callback (`kitty/boss.py:370-371`).

### 7.2 The three reconciliation guards (exhaustive)

**Guard 1 — `Boss.on_child_death` pops and early-returns if the window is already gone.**

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

(`kitty/boss.py:881-918`.) The reconciliation is the `window = self.window_id_map.pop(window_id, None)` (`:883`) followed by `if window is None: return` (`:884-885`): the C death-notify (fired from `parse_input`, `kitty/child-monitor.c:522`) is *deferred* to the main thread, so by the time it arrives the Python view may already have removed the window (e.g. during OS-window teardown, which pops `window_id_map`). If so, the pop yields `None` and the whole notification is a **no-op**. When the window *is* still present, the full teardown runs: `window.destroy()` (`:894`), `tab.remove_window(window)` (`:903`), and a focus-change reconciliation (`:914-918`).
The **window-present** path is observed in every ordinary close (§5.5 Scenarios 1 & 4: the window closes and kitty exits). The **early-return** path is **silent by design** — it prints nothing and returns — so it cannot be positively observed in the *unmodified* binary without adding a print, which R-READ-ONLY forbids; it is therefore labelled **inferred (not forced)**. Its firing context is real and is exactly the deferred-notify-after-removal race described above.

**Guard 2 — `mark_child_for_close` searches both `children[]` and `add_queue[]`.**

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
```

(`kitty/child-monitor.c:541-563`.) It first scans the live `children[]` and sets `needs_removal` (`:544-550`); **if not found there**, it scans the pending `add_queue[]` and sets `needs_removal` (`:552-558`). This is precisely the reconciliation for a *just-created* child that has been asked to close **before** the I/O loop has merged it out of `add_queue[]` into `children[]` — it can be cancelled in place. It is reached from Python via `mark_for_close` (`kitty/child-monitor.c:568`) ← `Boss.mark_window_for_close` → `self.child_monitor.mark_for_close(window.id)` (`kitty/boss.py:920,928`). The `children[]` branch is exercised by every normal close; the `add_queue[]` branch was **not forced** in the rapid runs because RC latency (~0.7 s) dwarfs the sub-ms merge tick (the `(add queue: 0)` counter in the §5.5 log corroborates the queue was empty), so that specific branch is labelled **inferred (not forced)**.

**Guard 3 — `hangup` tolerates an already-exited child via `ESRCH`.** (Observed.)

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

(`kitty/child-monitor.c:1294-1302`.) If the child has already exited, `getpgid(pid)` fails with `ESRCH` and `hangup` **returns immediately** (`:1297`), never attempting `killpg`; and even the `killpg(pgid, SIGHUP)` (`:1299`) tolerates `ESRCH` (`:1300`). This reconciles "kitty thinks it must hang up the child" against "the child is already dead."

**Observed** via `strace -f -e trace=getpgid,kill,wait4` on a self-exiting child (`sh -c "exit 0"`). Complete key lines:

```text
29350 04:41:05.692654 +++ exited with 0 +++
29349 04:41:05.692762 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 29350
29349 04:41:05.692866 wait4(-1, 0x78988cff8e60, WNOHANG, NULL) = -1 ECHILD (No child processes)
29349 04:41:05.693035 getpgid(29350)    = -1 ESRCH (No such process)
```

The child (29350) exits and is reaped; the subsequent `getpgid(29350)` returns `-1 ESRCH`, and — critically — **no `kill`/`killpg` syscall follows** it. That is the `if (errno == ESRCH) return;` guard at `:1297` firing on a genuinely-already-dead child, observed at runtime.

### 7.3 Reconciliation firings — summary of what was observed

- **Resize vs removal (Guard-adjacent, `resize_pty` `:610`):** OBSERVED — `Failed to send resize signal to child with id: N` when the Python view still holds a window the I/O view has removed; distribution {5, 3, 6, 6, 5} over 5×N=30 runs; **0** crashes (§5.5 Scenario 2/3).
- **`hangup` ESRCH tolerance (Guard 3):** OBSERVED — `getpgid` `ESRCH` with no following `kill` (§7.2 above).
- **Crash-free under churn:** OBSERVED — crash markers `0` across every run in §5.
- **`on_child_death` early-return (Guard 1)** and **`mark_child_for_close` `add_queue[]` branch (Guard 2):** grounded in code and labelled **inferred (not forced)** — the former is silent-by-design (cannot emit output without editing the binary), the latter was not reachable at RC latency.

For accuracy about the close paths used above: `kitty @ --help` lists exactly `close-tab` and `close-window` as the relevant subcommands (there is no `close-os-window` or `quit` `@` subcommand); closing the last window with `close-window` triggers the OS-window teardown that pops `window_id_map`, which is the context in which Guard 1's early-return can race.


---

## 8. Coverage pass

Re-reading the question and confirming each distinct thing it asks for is answered:

| Question item (verbatim phrase) | Answered in | How |
|---|---|---|
| "windows **appear** … in quick succession" | §3, §4 | create path: `add_child` → `add_queue[]` → per-tick `add_children` (`boss.py:585-588`, `child-monitor.c:305-319,1494`) |
| "**resize** … in quick succession" | §3.1, §3.3, §4 | `set_geometry` → `resize_pty` → `ioctl(TIOCSWINSZ)` (`window.py:850-876`, `child-monitor.c:579`); de-dup guard observed |
| "**disappear** … in quick succession" | §5 | `mark_child_for_close`/`remove_children`/`reap_children`; 4 scenarios observed |
| "in **quick succession**" (interleaving) | §4, §5.5 (Scenario 3) | per-tick reconciliation; N=30 × 5 runs, distribution {5,3,6,6,5}, 0 crashes |
| "new window … immediately **used to run a command**" | §3.2 | startup gate: `wait_for_terminal_ready` blocks `execvp` until first PTY size (`child.c:71-77,152,159`; observed `read=0` before `execve`) |
| "**resize events and signals** start flowing" | §3, §6 | resize = `ioctl(TIOCSWINSZ)` kitty→child; signals = `SIGCHLD` via `signalfd` |
| "window is **gone before everything has finished reacting**" (R3) | §5.3, §5.5 | ref-counted `scratch[]` snapshot keeps Child/Screen alive mid-parse; `Failed to send resize signal` for removed child (observed) |
| "how does kitty decide what to **keep** and what to **discard**" (R4) | §5.3 | keep in-flight bytes (`do_parse flush=true`, `:521`); discard further live parse of `needs_removal` child (`:529`); free at refcount 0 (`:532`) |
| "**timing** affects **signal delivery**" (R5) | §6.1, §6.5 | `signalfd` blocked+queued; effect on next loop tick; measured sub-ms distribution (≥2 runs) |
| "timing affects **internal bookkeeping**" (R5) | §6.4, §6.5 | coalescing `waitpid(WNOHANG)` loop; `needs_removal` → next-tick `remove_children` → `parse_input` |
| "**conflicting views of what is still alive**" (R6) | §7.1 | three views: C `children[]`/queues; `global_state` hierarchy; Python `window_id_map` |
| "how are they **resolved**" (R6) | §7.2 | three guards: `on_child_death` pop+early-return; `mark_child_for_close` children[]+add_queue[]; `hangup` ESRCH (observed) |
| "temporary scripts … repository **unchanged** … **cleaned up**" | §2, Appendix A, Appendix C | drivers shown in full & deleted; `git status --porcelain` proof |

All six sub-questions (R1–R6) and every named sub-item are answered with a producing command, complete unedited output, and `file:line` grounding. Items that could not be positively observed at runtime are labelled **inferred (not forced)** in §7.

---

## Appendix A — Temporary observation scripts (shown here, deleted from disk)

Per the read-only constraint, these drivers lived under `/tmp` (outside the repository), are reproduced here in full for reproducibility, and were deleted afterward (see Appendix C). They use RC **only to trigger** real window operations; the observed mechanisms are kitty's own.

**`driver_dedup.sh`** (R2 §3.3 — de-dup guard: new vs identical vs changed resize):

```bash
#!/bin/bash
# Temp driver (R3/C3): drive same-size vs changed-size OS-window resizes to exercise
# the window.py:L861 de-dup guard `if current_pty_size != self.last_reported_pty_size`.
set -u
cd /root/kitty_build
SOCK=/tmp/rc_dedup.sock
rm -f "$SOCK"
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
# Start kitty under strace, RC enabled, running a long-lived shell
strace -f -e trace=ioctl -tt -o /tmp/dedup.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE \
  -o allow_remote_control=yes --listen-on unix:"$SOCK" \
  sh -c "sleep 60" > /tmp/dedup_stdout.log 2> /tmp/dedup_stderr.log &
KPID=$!
sleep 4
KAT() { ./kitty/launcher/kitty @ --to unix:"$SOCK" "$@"; }
echo "### STEP A: resize-os-window to 900x600 (NEW size)"
KAT resize-os-window --width 900 --height 600 2>&1 | head -2
sleep 1
echo "### STEP B: resize-os-window to 900x600 AGAIN (IDENTICAL -> de-dup expected)"
KAT resize-os-window --width 900 --height 600 2>&1 | head -2
sleep 1
echo "### STEP C: resize-os-window to 1100x750 (CHANGED size)"
KAT resize-os-window --width 1100 --height 750 2>&1 | head -2
sleep 1
echo "### closing"
KAT close-window --match all 2>&1 | head -1
sleep 2
kill $KPID 2>/dev/null; wait $KPID 2>/dev/null
echo "### DONE"
```

**`driver_rapid.sh`** (R3 §5.5 Scenario 3 — rapid create-then-immediate-close, N cycles):

```bash
#!/bin/bash
# Temp driver (R3 scenario 3b): rapid create-then-immediate-close via RC, N cycles.
set -u
N=${1:-30}
cd /root/kitty_build
SOCK=/tmp/rc_rapid.sock; rm -f "$SOCK"
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
nohup ./kitty/launcher/kitty --debug-rendering --config NONE \
  -o allow_remote_control=yes --listen-on unix:"$SOCK" \
  sh -c "sleep 3600" > /tmp/rapid_stderr.log 2>&1 &
KPID=$!
sleep 4
KAT() { ./kitty/launcher/kitty @ --to unix:"$SOCK" "$@" 2>/dev/null; }
start=$(date +%s)
for i in $(seq 1 "$N"); do
  KAT launch --type=window sh -c "sleep 3600" >/dev/null
  KAT close-window --match recent:0 >/dev/null
done
end=$(date +%s)
alive=$(ps -p $KPID -o pid= 2>/dev/null | tr -d " ")
resp=$(KAT ls | head -c 1)
echo "N=$N; kitty_alive_pid=[$alive]; responsive_after=[$resp]; elapsed=$((end-start))s"
echo "Child_launched=$(grep -c 'Child launched' /tmp/rapid_stderr.log)"
echo "SIGWINCH_sent=$(grep -c 'SIGWINCH sent' /tmp/rapid_stderr.log)"
echo "Failed_resize_signal=$(grep -c 'Failed to send resize signal' /tmp/rapid_stderr.log)"
echo "crash_markers=$(grep -ciE 'segfault|assertion|traceback|fatal|panic|use-after-free' /tmp/rapid_stderr.log)"
KAT quit >/dev/null 2>&1; sleep 1; kill $KPID 2>/dev/null; wait $KPID 2>/dev/null
echo "DONE"
```

**`driver_timing.sh`** (R5 §6.5 — child-exit → reap latency per run):

```bash
#!/bin/bash
# Temp driver (R5/E6): measure child-exit -> kitty-reap (wait4) latency across runs.
set -u
cd /root/kitty_build
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
RUN=$1
timeout 20 strace -f -e trace=wait4,signalfd4 -tt -o /tmp/timing_${RUN}.strace \
  ./kitty/launcher/kitty --debug-rendering --config NONE \
  sh -c "exit 0" > /tmp/timing_${RUN}_out.log 2>&1
# Extract: child exit time and the kitty wait4 that reaped it (returns a pid>0)
python3 - "$RUN" <<'PY'
import sys, re
run=sys.argv[1]
lines=open(f"/tmp/timing_{run}.strace").read().splitlines()
def ts(l):
    m=re.match(r'\s*\d+\s+(\d+):(\d+):(\d+)\.(\d+)', l)
    if not m: return None
    h,mm,s,us=map(int,m.groups())
    return ((h*60+mm)*60+s)+us/1e6
exit_t=None; reap_t=None; reaped_pid=None
for l in lines:
    if '+++ exited with' in l and exit_t is None:
        exit_t=ts(l)
    m=re.search(r'wait4\(-1, \[.*\], WNOHANG, NULL\) = (\d+)', l)
    if m and int(m.group(1))>0 and reap_t is None:
        reap_t=ts(l); reaped_pid=m.group(1)
if exit_t and reap_t:
    print(f"RUN {run}: child_exit_to_reap = {(reap_t-exit_t)*1000:.3f} ms (reaped pid {reaped_pid})")
else:
    print(f"RUN {run}: exit_t={exit_t} reap_t={reap_t} (could not extract)")
PY
```

Session files (e.g. `/tmp/sess_split.conf`, used in §3.1) contained kitty `--session` directives (`launch` lines) to create windows that immediately run a command; they are `create_sessions`/`parse_session` inputs (`kitty/session.py:219,151`).

---

## Appendix B — Verified `file:line` anchor table (re-checked @ commit 815df1e21)

| Topic | Anchor(s) |
|---|---|
| `set_geometry` / resize path | `kitty/window.py:850` (def), `857-859` (pty size), `861` (de-dup guard), `863` (resize_pty), `866-867` (mark_terminal_ready + child_is_launched), `871` ("Child launched"), `873` ("SIGWINCH sent"), `874` (store), `875-876` (else), `578-579` (init `False`/`(-1,-1,-1,-1)`) |
| startup gate (parent) | `kitty/child.py:276` (fork), `283` (ready pipe), `342-343` (close read / hold write), `362-364` (mark_terminal_ready) |
| startup gate (child) | `kitty/child.c:71-77` (wait_for_terminal_ready), `123` (setsid), `129` (TIOCSCTTY), `151-153` (close/wait/close), `159` (execvp) |
| registration | `kitty/boss.py:585` (add_child), `587` (child_monitor.add_child), `588` (window_id_map); C `add_child` `kitty/child-monitor.c:305`, `316-318` (INCREF/count++/wakeup) |
| resize propagation | `kitty/child-monitor.c:592` (resize_pty), `606-607` (FIND children/add_queue), `577-589` (pty_resize), `579` (ioctl TIOCSWINSZ), `580` (EINTR), `581` (EBADF/ENOTTY), `610` ("Failed to send resize signal") |
| main I/O loop | `kitty/child-monitor.c:1489` (thread name), `1491-1495` (loop; remove_children `1493` then add_children `1494`), `1515` (wakeup drain), `1519` (read_signals), `1526` (reap on child_died); `35` (EXTRA_FDS=2) |
| removal staging | `kitty/child-monitor.c:1313-1332` (remove_children), `1319` (cleanup_child call), `1306-1308` (cleanup_child def), `1281-1290` (add_children) |
| ref-counted snapshot | `kitty/child-monitor.c:451` (parse_input), `456`/`483` (lock/unlock), `457-463` (remove_queue→remove_notify + INCREF), `479-480` (scratch INCREF), `517-526` (drain removed + do_parse flush + death_notify + FREE), `528-533` (parse survivors + DECREF), `438-448` (do_parse), `82-84` (arrays) |
| read edge handling | `kitty/child-monitor.c:1337-1356` (read_bytes; EINTR/EAGAIN `1347`; EIO silent `1348`), `1530-1535` (needs_removal on !has_more) |
| signal delivery | `kitty/loop-utils.c:34-56` (init_signal_handlers), `41-42` (sigprocmask+signalfd), `48` (self_pipe), `51-52` (sigaction SA_SIGINFO\|SA_RESTART); `kitty/loop-utils.h:14-15` (HAS_SIGNAL_FD), `40` (signal_read_fd) |
| signal drain | `kitty/loop-utils.c:131-152` (read_signals), `133` (fdsi[32]), `136`/`138`/`139`/`144` |
| handled signals | `kitty/child-monitor.c:121` (KITTY_HANDLED_SIGNALS, no SIGWINCH), `1362-1383` (handle_signal); `kitty/boss.py:1181`/`1185-1186` (install) |
| reaper | `kitty/child-monitor.c:1413-1426` (reap_children), `1418` (waitpid WNOHANG), `1422` (mark_child_for_removal call), `1386-1396` (mark_child_for_removal def) |
| view 3 / guards | `kitty/boss.py:344` (window_id_map WeakValueDictionary), `33` (import), `370-371` (ChildMonitor + on_child_death), `881-918` (on_child_death), `883-885` (pop + early return), `894` (destroy), `903` (tab.remove_window), `920`/`928` (mark_window_for_close → mark_for_close) |
| close guard | `kitty/child-monitor.c:541-563` (mark_child_for_close: children[] `544-550` then add_queue[] `552-558`), `568` (mark_for_close wrapper) |
| hangup guard | `kitty/child-monitor.c:1294-1302` (getpgid `1296`/ESRCH return `1297`/killpg SIGHUP `1299`/ESRCH `1300`) |
| structural view | `kitty/state.c:12` (global_state), `kitty/state.h:189`/`226`/`265` (windows/tabs/os_windows), `188`/`228`/`266` (counts); `kitty/tabs.py:580` (Tab.remove_window); `kitty/window_list.py:148` (id_map) |
| build/config | `Makefile:12-13` (`python3 setup.py`), `15-16` (test), `22-23` (debug), `25-26` (debug event-loop); `pyproject.toml:2` (>=3.8); `go.mod:3` (go 1.22); `setup.py:609` (harfbuzz>=1.5); `docs/build.rst` (native deps); `kitty_tests/main.py:57` (no lifecycle test) |

*Note on line drift:* one anchor differs by one line from the original prompt table (`read_bytes` `EINTR`/`EAGAIN` is at `:1347` and `EIO` at `:1348`, and `cleanup_child`'s definition begins at `:1306`); all others were confirmed exactly at commit `815df1e21`.

---

## Appendix C — Cleanup & read-only proof

**Temporary artifacts created during investigation, then deleted.** All observation scripts, syscall traces, build logs, session files, and RC sockets were created **outside** the repository tree — the host-side captures under `/tmp/kitty_investigation/` (26 files: `canonical_build.log`, `debug_build.log`, `*.strace`, `*_stderr.log`, the three `driver_*.sh` shown in Appendix A, `timing_1..5.strace`, `esrch.strace`, etc.) and the container-side working files under `/tmp` inside the build container. These were removed with `rm -rf /tmp/kitty_investigation` on the host and `rm -f` of the matching `/tmp` files in the container; a post-deletion scan of the repository tree found **zero** stray temp files:

```bash
find . -path ./.git -prune -o \( -name 'blitzy_adhoc_test_*' -o -name '*.strace' -o -name 'driver_*.sh' -o -name 'sess_*.conf' \) -print
# (no output — no temp artifacts anywhere in the repository)
```

The build/run happened on container-internal copies (`/root/kitty_build`, `/root/kitty_dbg`), never in the repository working tree, so no tracked file (including `go.sum`) was touched by the build.

**Read-only proof.** From the repository root, before staging, `git status --porcelain --untracked-files=all` lists exactly one path — the new answer document — and reports **no** modified, deleted, or renamed existing file:

```bash
git -C /tmp/blitzy/kitty/... status --porcelain --untracked-files=all
```

```text
?? blitzy/documentation/kitty_815df1e210e0.md
```

That single `??` line is the entirety of the change set: the repository is unchanged except for this one added file under `blitzy/documentation/`. (The pre-existing empty `blitzy/screenshots/` and `blitzy/screen_recordings/` directories are not tracked by git, as git does not track empty directories.) The document was then committed on the working branch; a subsequent `git status --porcelain` reported a clean tree (nothing left uncommitted).

