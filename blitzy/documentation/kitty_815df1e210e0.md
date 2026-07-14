# How kitty keeps its internal state consistent across the rapid window lifecycle

A runtime-evidenced investigation of the kitty terminal emulator (branch `kitty_815df1e210e0`, commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, kitty `0.35.2`).

## The question

> I want to understand how kitty keeps its internal state consistent when terminal windows appear, resize, and disappear in quick succession. When a new window is created and immediately used to run a command, resize events and signals start flowing through the system. What happens if the window is gone before everything has finished reacting to those changes? How does kitty decide what state to keep and what to discard? I am especially interested in how timing affects signal delivery and internal bookkeeping, and whether there are moments where the system has to resolve conflicting views of what is still alive. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.

---

## Executive summary (direct answers)

- **Q1 — Create-then-immediately-use.** A new window's child is **staged into an add-queue and registered with the C child-monitor *before* the tab layout runs**, precisely so the first layout-driven resize can find it. That first resize is what brings the PTY online: it transmits the real terminal size to the kernel (`ioctl(TIOCSWINSZ)`), flips the window out of its sentinel state, and marks the terminal ready so the child is released to `exec`. **Observed:** the very first `ioctl(TIOCSWINSZ)` on each new fd *succeeds* (`= 0`), proving the child was already registered before layout resized it.

- **Q2 — Reactions to an already-gone window.** kitty **tolerates** stale window ids and file descriptors. The resize path scans **both** the live `children[]` array **and** the not-yet-drained `add_queue[]`; when an id is in neither it logs a specific diagnostic and simply continues; a closed fd's `ioctl` error (`EBADF`/`ENOTTY`) is silently absorbed; and the Python teardown is **idempotent** (`pop(id, None)` then early-return). **Observed:** `Failed to send resize signal to child with id: 3 (children count: 4) (add queue: 0)` logged 108 times in one run with **zero** exceptions and a **clean exit**.

- **Q3 — Keep-versus-discard.** A per-child boolean **`needs_removal`** is the switch. A child is discarded when its PTY reaches EOF, its fd becomes invalid (`POLLNVAL`), or — only when `close_on_child_death` is enabled — when it is reaped. Discard runs through `remove_children` → `cleanup_child` → `hangup` (which sends `SIGHUP` to the child's process group). The default of `close_on_child_death` is **`no`**, so a bare child exit does **not**, by itself, force the window closed. **Observed:** `killpg(pgrp=7101, SIGHUP) = 0` during teardown of live children; canonical defaults dumped live as `close_on_child_death = False`.

- **Q4 — Timing.** Asynchronous POSIX signals are **decoupled from state mutation**: on Linux kitty blocks the handled signals and reads them from a **`signalfd`** that the IO thread drains at a safe point (the classic self-pipe is the macOS/BSD fallback, *not* used here). Multiple child deaths **coalesce** — several `SIGCHLD` collapse into one or few `signalfd` reads, and one `waitpid(-1, WNOHANG)` loop reaps them all. Rapid resizes are **de-duplicated** — a size equal to the last one reported is never re-sent, and the sentinel size is never transmitted. **Observed:** `read(signalfd=9, …) = 128 [1 signalfd_siginfo: SIGCHLD]` followed by two `waitpid` reaps; `pipe2` count = 0; 39 resize ioctls with **0** duplicate-size and **0** sentinel transmissions.

- **Q5 — Conflicting liveness views.** Yes — there are real moments where the **main/UI thread** (Python `Boss`) and the **IO thread** (`KittyChildMon`) disagree about what is alive. They are reconciled by a single `children_lock` mutex, a **remove-before-add** staged reconciliation performed every IO-loop iteration under that lock, and **idempotent** Python pops that tolerate a window being seen dead twice. **Observed:** the main thread issued resizes for ids `3/4/2` that the IO thread had already removed, with `children count` fluctuating across `{0,1,2,3,4,5,6}` within a single run — yet exit was clean with zero Python errors.

Each answer is defended in full below with the exact source location that performs the work and the unedited runtime output that demonstrates it.

---

## Environment & Methodology

**Run-first discipline.** Every behavioural claim in this document was produced by building/running the real software and capturing output *before* writing. Source locations (`file:line`) are given for grounding, but the evidence is the observed output placed next to each claim. Statements that could not be directly observed are explicitly labelled **inferred**, with the attempts that were made.

**Canonical container.** All observation was performed inside the user-specified canonical image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (derived tag `kitty-qna:latest` adds only the headless/observation tooling — Xvfb, Mesa software GL — and bakes `LIBGL_ALWAYS_SOFTWARE=1`). The repository checkout inside the container is at the same commit as the deliverable branch.

**Toolchain banners (verbatim):**

```
$ uname -a
Linux ae6e9abee2f2 6.6.122+ #1 SMP Thu Apr 2 09:59:00 UTC 2026 x86_64 GNU/Linux
$ python3 --version
Python 3.12.3
$ go version
go version go1.23.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ glxinfo | grep -iE "renderer|OpenGL version"
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.24.04.2
```

**Build.** kitty is built canonically. The `Makefile` default target is `all: python3 setup.py $(VVAL)` (`Makefile:12`), i.e. **`python3 setup.py build`**; `setup.py:609` enforces `at_least_version('harfbuzz', 1, 5)`. The container ships kitty pre-built with exactly that command (release build); the debug/event-loop variant is `python3 setup.py build --debug --extra-logging=event-loop` (`Makefile:25`). The canonical pre-built release binary was used so that reported option values are canonical.

**Headless launch.** kitty is a GPU/OpenGL application, so it is run under a virtual framebuffer with software GL:

```
Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
```

**Default configuration.** kitty is always launched with `--config NONE` (no `kitty.conf`), so `close_on_child_death` and `resize_debounce_time` are observed at their defaults. Individual contrasts use canonical `-o KEY=VALUE` overrides (e.g. `-o close_on_child_death=yes`).

**Canonical entry points only.** The window lifecycle is driven through a startup **session file** (`kitty --session <file>`; parsed by `parse_session` at `kitty/session.py:151`, materialised by `create_sessions` at `kitty/session.py:219`, honouring the `launch` and `resize_window` directives at `kitty/session.py:184,203`). The non-canonical remote-control bypass **`kitty @ signal-child` was explicitly excluded** and never used to stand in for the real PTY/kernel signal path.

**Observation tools.**
- `kitty --debug-rendering` — a *runtime* kitty flag (not a `setup.py` flag) that emits the `Child launched` and `SIGWINCH sent to child …` lines from `kitty/window.py`.
- An **`LD_PRELOAD` syscall shim** (`/tmp/libshim.so`) that intercepts `ioctl(TIOCSWINSZ)`, `waitpid`/`wait4`, `signalfd`/`signalfd4`, `read` (of the signalfd), `rt_sigprocmask`, and `killpg`, logging `<monotonic_ts> tid=<tid> comm=<threadname> <call…>`.

> **Why a shim and not `strace`?** `strace`/`ptrace` is blocked in this environment: the host kernel has `kernel.yama.ptrace_scope=3` (immutable), so `strace` fails with `PTRACE_TRACEME: Operation not permitted` even with `--cap-add=SYS_PTRACE --security-opt seccomp=unconfined`. The setup log endorses the `LD_PRELOAD` shim as the ptrace-free substitute. Thread attribution through the shim is reliable: `comm=kitty` is the main/UI thread, `comm=KittyChildMon` is the IO thread.

**Platform boundary.** All observations are on **Linux/X11 headless**. For `resize_debounce_time` this is the **on-end** code path (the first number, `0.1`). The macOS OS-notification branch (the second number, `0.5`, redraw-after-pause) is described from source and official docs and is **labelled "not directly observed on this platform."**

**Two threads, one shared state (the model this document explains).** kitty runs a Python **main/UI thread** (the `Boss`, driving GLFW, layout, and Python-side teardown) and a dedicated **IO thread** (`KittyChildMon`, running the child-monitor `io_loop`) that performs PTY read/write, signal reaping, and add/remove reconciliation. All shared child state is guarded by one `pthread_mutex_t children_lock` (`kitty/child-monitor.c:87`) accessed through the `children_mutex(op)` macro (`kitty/child-monitor.c:77`).

---

## Q1 — What happens on the create → immediately-use → first-resize path?

**Direct answer.** When a window is created, its child (fork + PTY) is **staged into a C add-queue and registered with the child-monitor *before* the tab's layout runs**, because the layout's first resize must be able to find the child. That **first resize is the moment the PTY/screen comes online**: `Window.set_geometry` computes the real terminal grid, sees it differs from the initial sentinel, calls `resize_pty` (which issues `ioctl(TIOCSWINSZ)`), marks the terminal ready (releasing the child to `exec`), flips `child_is_launched` true, and records the size as `last_reported_pty_size`. The proof that the ordering is correct is that the **first `ioctl(TIOCSWINSZ)` on each new fd succeeds** rather than logging "Failed to send resize signal".

### Mechanism (cause → effect), grounded in source

1. **Create and the ordering constraint.** `Tab.new_window` (`kitty/tabs.py:504`) creates the window, then — with the explicit comment *"Must add child before laying out so that resize_pty succeeds"* (`kitty/tabs.py:534`) — calls `get_boss().add_child(window)` (`kitty/tabs.py:535`) **before** it lays out the tab. This is the whole reason the child is registered eagerly.

2. **Registration stages the child (it does not enter the live array yet).** `Boss.add_child` (`kitty/boss.py:585`) asserts the child has a `pid` and `child_fd` (`:586`), calls `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` (`:587`), then records `self.window_id_map[window.id] = window` (`:588`). On the C side, `add_child` (`kitty/child-monitor.c:305`) takes `children_mutex(lock)` (`:307`), checks `MAX_CHILDREN` (`:308`), copies the child into `add_queue[add_queue_count]`, increments `add_queue_count` (`:317`), and calls `wakeup_io_loop` (`:319`). **The new child goes to the staging `add_queue`, not directly into `children[]`** — the IO loop drains it later (see Q5).

3. **First resize brings the PTY online.** `Window.set_geometry` (`kitty/window.py:850`) computes `current_pty_size` and guards on `if current_pty_size != self.last_reported_pty_size:` (`kitty/window.py:861`). `last_reported_pty_size` starts at the sentinel `(-1, -1, -1, -1)` (`kitty/window.py:579`), so the guard is true on the first layout. Inside it calls `boss.child_monitor.resize_pty(self.id, *current_pty_size)` (`kitty/window.py:863`), stamps `self.last_resized_at = monotonic()` (`kitty/window.py:864`; field initialised at `:562`), and on the first pass runs `self.child.mark_terminal_ready(); self.child_is_launched = True` (`kitty/window.py:866`; `child_is_launched` initialised `False` at `:578`). Under `--debug-rendering` it prints `Child launched` on that first pass (`kitty/window.py:871`) and `SIGWINCH sent to child …` on *subsequent* size-changing passes (`kitty/window.py:873`), finally assigning `self.last_reported_pty_size = current_pty_size` (`kitty/window.py:874`).

4. **kitty issues the ioctl; the kernel delivers SIGWINCH.** `resize_pty` ultimately reaches `pty_resize` → `ioctl(fd, TIOCSWINSZ, dim)` (`kitty/child-monitor.c:579`). The debug label "SIGWINCH sent to child" describes the *effect*: setting the window size via `TIOCSWINSZ` causes the **kernel** to deliver `SIGWINCH` to the foreground process group — kitty does not itself raise `SIGWINCH` for the child.

### Observed output

Command (canonical session that opens three split windows, each immediately launching a command; the act of adding each window resizes its live siblings, producing the post-launch `SIGWINCH` lines):

```
LD_PRELOAD=/tmp/libshim.so SHIM_LOG=/tmp/shim_q1.log \
  ./kitty/launcher/kitty --config NONE --debug-rendering \
  -o close_on_child_death=no --session /tmp/obs.session
```

`--debug-rendering` output (`dbg_q1.log`, verbatim):

```
[0.153] OS Window created
[0.163] Failed to open systemd user bus with error: No medium found
[0.164] Child launched
[0.169] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
[0.169] Child launched
[0.176] SIGWINCH sent to child in window: 2 with size: (22, 17, 153, 396)
[0.176] Child launched
```

Shim output for the same run (`shim_q1.log`, verbatim; **all on `tid=1440 comm=kitty` — the main/UI thread**):

```
1883219.851518463 tid=1440 comm=kitty signalfd(fd=-1, mask=[INT HUP TERM CHLD USR1 USR2 ], flags=SFD_NONBLOCK|SFD_CLOEXEC|) = 9
1883219.859805906 tid=1440 comm=kitty ioctl(fd=10, TIOCSWINSZ, {rows=22 cols=71 xpix=639 ypix=396}) = 0
1883219.863983391 tid=1440 comm=kitty ioctl(fd=10, TIOCSWINSZ, {rows=22 cols=35 xpix=315 ypix=396}) = 0
1883219.864230706 tid=1440 comm=kitty ioctl(fd=11, TIOCSWINSZ, {rows=22 cols=35 xpix=315 ypix=396}) = 0
1883219.871032250 tid=1440 comm=kitty ioctl(fd=11, TIOCSWINSZ, {rows=22 cols=17 xpix=153 ypix=396}) = 0
1883219.871670613 tid=1440 comm=kitty ioctl(fd=12, TIOCSWINSZ, {rows=22 cols=17 xpix=153 ypix=396}) = 0
```

### What this evidence proves

- **Add-before-layout ordering is real.** The **first** `ioctl(TIOCSWINSZ)` on each new fd (`fd=10` at `cols=71`, `fd=11` at `cols=35`, `fd=12` at `cols=17`) returns `= 0`. If layout had resized before the child was registered, that first resize would instead have logged "Failed to send resize signal to child …" (the Q2 diagnostic). Success on the first resize is the observable consequence of `kitty/tabs.py:534-535`.
- **The sentinel is never transmitted.** No `ioctl` carries `rows=-1`/`cols=-1`; the first `ioctl` per fd already carries a real grid. This is the `last_reported_pty_size` sentinel (`kitty/window.py:579`) being *compared against* but never *sent* — the guard at `kitty/window.py:861` only fires the ioctl once a real size is known.
- **`Child launched` prints exactly once per window** (three windows → three `Child launched` lines), matching the `if not self.child_is_launched:` one-shot at `kitty/window.py:866,871`. The `SIGWINCH sent to child …` lines are the *second* resize of an already-launched sibling (window 1 shrinks `cols 71 → 35`, window 2 `35 → 17`) — exactly the sizes echoed by both the debug line and the shim ioctl (e.g. window 1 → `(22, 35, 315, 396)` ⇔ `ioctl(fd=10, … cols=35 …)`).
- **The resize happens on the main/UI thread** (`comm=kitty`), i.e. layout drives the PTY size synchronously as part of `set_geometry`.

**Before / after state.** Before the first resize, `last_reported_pty_size == (-1,-1,-1,-1)` and `child_is_launched == False` (source sentinels at `kitty/window.py:578-579`). After it, `child_is_launched == True` and `last_reported_pty_size` holds the real grid — evidenced by the fact that a *second* resize of the same fd is what produces the `SIGWINCH sent to child` line rather than another `Child launched`.

---

## Q2 — What happens if the window is gone before everything has finished reacting?

**Direct answer.** Nothing breaks — stale window ids and file descriptors are **tolerated by design**. When a resize (or close) targets a window that has already departed, kitty (a) looks for the id in **both** the live `children[]` array **and** the pending `add_queue[]`, (b) if the id is in neither, logs a specific, non-fatal diagnostic and continues, (c) if the fd is present but already closed, absorbs the `ioctl` error `EBADF`/`ENOTTY` silently, and (d) resolves the Python-side double bookkeeping with an **idempotent** `pop(id, None)` that early-returns when the window is already gone. The process does **not** crash and does **not** surface an exception.

### Mechanism (cause → effect), grounded in source

1. **The resize dual-scan.** `resize_pty` (`kitty/child-monitor.c:592`) takes `children_mutex(lock)` (`:598`), runs `FIND(children, self->count)` to locate the id in the live array (`:606`), and if not found (`fd == -1`) runs `FIND(add_queue, add_queue_count)` (`:607`) to check the *not-yet-drained* staging queue. Only if **both** miss does it log:
   `log_error("Failed to send resize signal to child with id: %lu (children count: %u) (add queue: %zu)", …)` (`kitty/child-monitor.c:610`). This is a log-and-continue, not an error return — the stale resize is dropped harmlessly.

2. **Closed-fd tolerance.** `pty_resize` (`kitty/child-monitor.c:577`) issues `ioctl(fd, TIOCSWINSZ, dim)` (`:579`), retries on `EINTR` (`:580`), and — crucially — its error branch is `if (errno != EBADF && errno != ENOTTY)` (`:581`): a `Bad file descriptor` or `Not a typewriter` on a fd that closed underneath the resize is **swallowed**; only *other* errno values are logged as a real failure.

3. **Close dual-scan.** The same "check both collections" pattern protects closing: `mark_child_for_close` (`kitty/child-monitor.c:541`) scans the live `children[]` and sets `needs_removal = true` (`:546`), *and* scans `add_queue[]` setting `needs_removal = true` (`:554`) — so a window closed while it is *still only staged* is handled correctly.

4. **Idempotent Python teardown.** `Boss.on_child_death` (`kitty/boss.py:881`) does `window = self.window_id_map.pop(window_id, None)` (`:883`) then `if window is None: return` (`:884-885`). Because it uses `pop(…, None)` rather than `del`, a second death/close notification for the same id is a harmless no-op instead of a `KeyError`.

### Observed output

Command (rapid create → run → resize → destroy burst: a splits session of 20 windows, each running a command then exiting after a short staggered lifetime, so the layout keeps trying to resize windows that have already died):

```
/tmp/rapid_cycle.sh 1 20 0.05 yes      # RUNS=1 NWIN=20 LIFE=0.05 close_on_child_death=yes
```

`--debug-rendering` output (`dbg_run1.log`, verbatim first lines; the id-not-found diagnostic from `kitty/child-monitor.c:610`):

```
[1.194] Failed to send resize signal to child with id: 4 (children count: 4) (add queue: 0)
[1.203] Failed to send resize signal to child with id: 4 (children count: 4) (add queue: 0)
[1.218] Failed to send resize signal to child with id: 3 (children count: 4) (add queue: 0)
[1.226] Failed to send resize signal to child with id: 3 (children count: 4) (add queue: 0)
[1.234] Failed to send resize signal to child with id: 3 (children count: 4) (add queue: 0)
[1.244] Failed to send resize signal to child with id: 3 (children count: 4) (add queue: 0)
```

Total occurrences and the tolerance proof for the same run:

```
$ grep -c "Failed to send resize signal" dbg_run1.log
108
$ grep -cEi "Traceback|OSError|Exception|Fatal|assert" dbg_run1.log shim_run1.log
dbg_run1.log:0
shim_run1.log:0
# kitty process exit status for the run: 0
```

### What this evidence proves

- **The stale-id path is exercised and tolerated.** The diagnostic string matches `kitty/child-monitor.c:610` character-for-character (including the `(children count: N) (add queue: M)` suffix). It fired **108 times** in a single run while windows 3 and 4 (early diers) were still being targeted by the splits relayout — and the run still exited `0` with **zero** tracebacks/exceptions.
- **`add queue: 0` in every line** shows the staging queue had already been drained into `children[]` by the time the resize ran, so the pending-branch dual-scan (`:607`) also missed — the id was genuinely gone from both views. This is the Q5 divergence surfacing as an observable Q2 diagnostic.
- **The run continues normally afterward.** In the same log the surviving window later reclaims the full width, e.g. `SIGWINCH sent to child in window: 20 with size: (22, 71, 639, 396)`, and kitty quits cleanly.

### On the `EBADF`/`ENOTTY` tolerance specifically (labelled: defensive guard, not reachable via the canonical resize)

The tolerance at `kitty/child-monitor.c:581` is a real, cited defensive branch, but it was **never triggered** through the canonical path across every attempt: `tolerated_EBADF = 0` and `tolerated_ENOTTY = 0` for `close=yes NWIN=20 LIFE=0.05` (44 stale-id diagnostics), `close=no NWIN=24 LIFE=0.04` (69–75), and `close=yes NWIN=40 LIFE=0.03` (108–110). A direct micro-experiment isolated the errno behaviour:

```
# probe (own PTY, outside kitty):
ioctl on master AFTER sole child exit: ret=0 errno=0 (ok)
ioctl on CLOSED master fd:             ret=-1 errno=9 (Bad file descriptor)   # EBADF
```

**Grounded conclusion (inferred for the in-kitty trigger, with attempts stated):** `EBADF`/`ENOTTY` arises only on an *already-closed* fd, but in kitty the fd close in `cleanup_child` (`kitty/child-monitor.c:1307`) and the removal from `children[]` happen **atomically under `children_lock`**, and `resize_pty` holds that *same* lock during its `FIND` + `ioctl` (`:598`). Consequently the resize path either sees a live entry with an open fd (ioctl `= 0`) or does not find the entry at all (the "Failed to send resize signal" diagnostic) — it never observes a live entry pointing at a closed fd. The race therefore manifests as the **id-not-found diagnostic**, not as a tolerated `ioctl` error. Attempts to force the `EBADF` branch: varied `close_on_child_death` yes/no, `NWIN` 10–40, `LIFE` 0.03–0.12, staggered lifetimes, and the splits layout — none produced a tolerated `EBADF`/`ENOTTY`. The tolerance is thus a correctly-placed defensive guard for fd states the lock discipline makes unreachable on this path.

---

## Q3 — How does kitty decide what state to keep and what to discard?

**Direct answer.** The decision hinges on a single per-child boolean, **`needs_removal`**. A child is marked for discard when one of three things happens: its PTY read reaches EOF/error, its fd is reported invalid (`POLLNVAL`), or — **only if `close_on_child_death` is enabled** — it is reaped by `waitpid`. Marked children are torn down by `remove_children` → `cleanup_child` → `hangup`, which closes the fd and sends `SIGHUP` to the child's process group. Everything not marked is **kept**. Because the default of `close_on_child_death` is **`no`**, a plain child exit does not by itself set `needs_removal`; the discard on the default path is driven by the PTY reaching EOF, not by the reap.

### Mechanism (cause → effect), grounded in source

1. **The switch.** Each child carries `bool needs_removal` (`kitty/child-monitor.c:67`). Nothing is discarded unless this is set.

2. **What sets it.**
   - **PTY EOF/error:** in the IO loop, a child whose PTY read returns no-more-data / hangup sets `needs_removal = true` (`kitty/child-monitor.c:1535`, within the `io_loop` read handling).
   - **`POLLNVAL`:** a child whose fd is unexpectedly closed sets `needs_removal = true` and logs `The child %lu had its fd unexpectedly closed` (`kitty/child-monitor.c:1545,1547`).
   - **Reap, gated:** `reap_children` (`kitty/child-monitor.c:1413`) loops `waitpid(-1, &status, WNOHANG)` (`:1418`) and calls `mark_child_for_removal(self, pid)` **only** inside `if (enable_close_on_child_death)` (`:1422`); it *always* calls `mark_monitored_pids(pid, status)` (`:1423`). `mark_child_for_removal` (`:1386`) sets `needs_removal = true` (`:1390`). The gate value comes from the `io_loop` call `reap_children(self, OPT(close_on_child_death))` (`:1526`).

3. **How discard executes.** `remove_children` (`kitty/child-monitor.c:1313`) iterates the array **backward** (`:1316`), and for each `needs_removal` child (`:1317`) calls `cleanup_child(i)` (`:1319`), moves the child into `remove_queue[]` (`:1320-1321`), zeroes the slot with `EMPTY_CHILD` (`:1322`), invalidates `children_fds[…].fd = -1` (`:1323`), `memmove`-compacts the array (`:1325-1328`), and decrements `self->count` (`:1331`). `cleanup_child` (`:1306`) is `safe_close(fd)` (`:1307`) + `hangup(pid)` (`:1308`). `hangup` (`:1294`) computes the process group with `getpgid` (returning early on `ESRCH`, `:1297`) and sends `killpg(pgid, SIGHUP)` (`:1299`), tolerating `ESRCH` (`:1300`).

4. **Python-side teardown.** Once the C side notifies the death (see Q5 for the cross-thread handoff), `Boss.on_child_death` (`kitty/boss.py:881`) runs `window.destroy()` (`kitty/boss.py:894`) and `tab.remove_window(window)` (`kitty/tabs.py:580`, called around `kitty/boss.py:903`), removing the window from the Python liveness view.

### Observed output

**Canonical defaults (dumped live inside the container, default config):**

```
$ python3 -c "from kitty.options.types import defaults as d; \
    print('close_on_child_death =', d.close_on_child_death); \
    print('resize_debounce_time =', d.resize_debounce_time)"
close_on_child_death = False
resize_debounce_time = (0.1, 0.5)
```

This matches the source defaults `close_on_child_death: bool = False` (`kitty/options/types.py:500`) / `opt('close_on_child_death', 'no', …)` (`kitty/options/definition.py:2920`).

**The reap loop on the IO thread** (`shim_c3.log`, verbatim; note the `waitpid(-1, WNOHANG) = 0` terminator that ends each drain):

```
1883865.061148193 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 3141  (WIFEXITED=1 status=0 WIFSIGNALED=0)
1883865.061157040 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 3034  (WIFEXITED=1 status=0 WIFSIGNALED=0)
1883865.061163495 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 0
```

**Hangup / `SIGHUP` to the process group during teardown of still-live children** (`shim_dedup.log`, verbatim; `comm=KittyChildMon` = IO thread, matching `kitty/child-monitor.c:1299`):

```
1883930.749041670 tid=7099 comm=KittyChildMon killpg(pgrp=7101, SIGHUP) = 0
1883930.749060063 tid=7099 comm=KittyChildMon killpg(pgrp=7100, SIGHUP) = 0
```

**The `close_on_child_death=yes` contrast** (canonical `-o` override): a smoke session `launch sh -c "echo SMOKE_OK; sleep 0.5"` run with `-o close_on_child_death=yes` caused the window to close as soon as the child exited (kitty quit, exit `0`) — i.e. the reap force-marked the window via `mark_child_for_removal` (`kitty/child-monitor.c:1422`). Under the default `close_on_child_death=no`, that force-mark is skipped.

### What this evidence proves

- **`needs_removal` + `waitpid`-gated reaping is the keep/discard engine.** The `waitpid(-1, WNOHANG)` loop with its `= 0` terminator is exactly `reap_children` (`kitty/child-monitor.c:1418`), running on the IO thread; the `killpg(…, SIGHUP)` calls are `hangup` (`kitty/child-monitor.c:1299`) during `cleanup_child`.
- **`ESRCH` tolerance shows up as an absence.** When children had *already* exited before teardown, **no** `killpg` line appears — `getpgid` returned `ESRCH` and `hangup` early-returned (`kitty/child-monitor.c:1297`). When children were still live at teardown, the two `killpg(pgrp=…, SIGHUP) = 0` lines above appear, one per process group. Both outcomes match the source's `ESRCH`-tolerant hangup.
- **Default vs `yes` differ observably.** With `yes`, the window closes on child exit; with the default `no`, the reap does not force removal.

### Honest limitation (labelled: default hold-open case not reproducible headlessly)

The documented default behaviour — "the terminal remains open when the child exits *as long as there are still other processes outputting to the terminal*" (see Option defaults below) — could **not** be reproduced in this headless session-file harness. Two attempts were made under `close_on_child_death=no`: (A) a foreground shell that exits leaving a backgrounded `sleep 300 &`; (B) a `setsid` survivor. In both, kitty exited `0` (the window was discarded). A direct PTY probe explains why:

```
# after the foreground session-leader shell exits (a live bg process still exists):
read(master) = -1 errno=5 (Input/output error)      # EIO, not EAGAIN
```

**Grounded conclusion (source/doc-derived, attempts stated):** on Linux, once the session-leader (controlling process) of the PTY exits, a `read` of the master returns `EIO`. `read_bytes` (`kitty/child-monitor.c:1337`) treats that as end-of-input and the IO loop sets `needs_removal` (`:1535`) — so the sole-child window closes regardless of `close_on_child_death`. The doc-described "hold open" requires a process that keeps the PTY *session* alive by continuing to hold/write the slave (a genuine interactive foreground successor), which the non-interactive session-file path cannot sustain headlessly. A passive backgrounded `sleep` is not "outputting to the terminal", so it does not keep the master readable. The default-hold semantics are therefore reported from the source and the official documentation, with the reproduction gap and its root cause stated plainly rather than faked.

---

## Q4 — How does timing affect signal delivery and internal bookkeeping?

**Direct answer.** Timing is deliberately **decoupled from state mutation**. Asynchronous POSIX signals never mutate kitty state in the handler; instead the handled signals are blocked and made readable as data on a polled descriptor that the IO thread drains at a safe point in its loop. On Linux that descriptor is a **`signalfd`** (the classic self-pipe trick is the macOS/BSD fallback and is *not* used here — **CORRECTION #1**). Because standard signals do not queue, a rapid burst of `SIGCHLD` **coalesces** into one or a few `signalfd` reads, and a single `waitpid(-1, WNOHANG)` loop then reaps *all* ready children per wake. Rapid resizes are **de-duplicated** — a size identical to the last one reported is never re-sent, and the sentinel is never transmitted — and, on macOS, additionally **debounced** by `resize_debounce_time`.

### Mechanism (cause → effect), grounded in source

1. **CORRECTION #1 — Linux uses `signalfd`, not a self-pipe.** `kitty/loop-utils.h:14-17` defines `HAS_SIGNAL_FD` when `<sys/signalfd.h>` is available; the self-pipe `signal_fds[2]` is declared only `#ifndef HAS_SIGNAL_FD` (`kitty/loop-utils.h:35-37`) and `self_pipe()` only `#ifdef __APPLE__` (`kitty/loop-utils.h:52-53`). In `init_signal_handlers` (`kitty/loop-utils.c`), the `HAS_SIGNAL_FD` branch does `sigprocmask(SIG_BLOCK, &signals, NULL)` (`:41`) + `signalfd(-1, &signals, SFD_NONBLOCK | SFD_CLOEXEC)` (`:42`); the `#else` branch does `self_pipe(...)` + `sigaction(..., SA_SIGINFO | SA_RESTART)` (`:48-52`). The signal-writing `handle_signal` (`kitty/loop-utils.c:15`) compiles only in the self-pipe (`#ifndef HAS_SIGNAL_FD`) path — i.e. macOS/BSD. `read_signals` (`kitty/loop-utils.c:131`) reads `struct signalfd_siginfo fdsi[32]` (`:133`). The handled set `KITTY_HANDLED_SIGNALS` = `SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` (`kitty/child-monitor.c:121`), masked via `mask_variadic_signals` (`kitty/child-monitor.c:124`, invoked `:151`).

2. **Decoupling = signal timing separated from state mutation.** The `signalfd` is created and the mask installed on the **main thread** at startup; the signal bytes are **read on the IO thread** inside the loop and only *then* is any state touched. So the moment a signal *arrives* (arbitrary, async) is divorced from the moment its effect is *applied* (a controlled point in the IO loop).

3. **SIGCHLD coalescing.** The IO-loop signal callback is `handle_signal` at `kitty/child-monitor.c:1362` (a *different* function from the macOS self-pipe writer at `kitty/loop-utils.c:15` — same name, distinct roles). It collapses any `SIGCHLD` into a single flag `ss->child_died = true` in the `SignalSet { bool kill_signal, child_died, reload_config; }` (`kitty/child-monitor.c:1359,1370-1371`). The loop then reaps **once** per iteration: `if (ss.child_died) reap_children(self, OPT(close_on_child_death))` (`kitty/child-monitor.c:1526`), and `reap_children`'s `waitpid(-1, WNOHANG)` loop drains *all* ready children. Standard signals are not queued, so N near-simultaneous deaths need not produce N reads — the reap loop compensates.

4. **CORRECTION #2 — resize debounce field mapping (`on_end = 0.1`, `on_pause = 0.5`).** The struct order is `struct { monotonic_t on_end, on_pause; } resize_debounce_time;` (`kitty/state.h:82`); the option-to-C mapping is `on_end = tuple[0]` (`kitty/options/to-c.h:349`) and `on_pause = tuple[1]` (`:350`), with the default tuple `(0.1, 0.5)` (`kitty/options/types.py:568`, `kitty/options/definition.py:1182-1183`). `process_pending_resizes` (`kitty/child-monitor.c:1042`) uses `OPT(resize_debounce_time).on_pause` in the macOS `from_os_notification` branch (`:1055`) and `OPT(resize_debounce_time).on_end` in the general/Linux `else` branch (`:1062`). **Therefore the first number, `on_end = 0.1`, is the value in effect on the Linux/X11 observation platform** (the on-end path); `on_pause = 0.5` is the macOS redraw-after-pause value. The AAP's phrasing "on-pause 0.1 / on-end 0.5" is **swapped** relative to code and docs; the observed Linux value is `on_end = 0.1`.

5. **Resize de-duplication.** A resize is issued only when the new size differs from `last_reported_pty_size` (guard at `kitty/window.py:861`; field/sentinel at `:579`), stamped by `last_resized_at` (`:562,864`). Identical consecutive sizes are suppressed, and the `(-1,-1,-1,-1)` sentinel is never transmitted.

### Observed output

**CORRECTION #1 — `signalfd` created on the main thread, read on the IO thread; `pipe2` never used** (`shim_c3.log`, verbatim):

```
1883863.645942828 tid=2853 comm=kitty         sigprocmask(SIG_BLOCK, [INT HUP TERM CHLD USR1 USR2 ]) = 0
1883863.746221313 tid=2853 comm=kitty         signalfd(fd=-1, mask=[INT HUP TERM CHLD USR1 USR2 ], flags=SFD_NONBLOCK|SFD_CLOEXEC|) = 9
1883865.060621373 tid=2919 comm=KittyChildMon read(signalfd=9, count=4096) = 128  [1 signalfd_siginfo: SIGCHLD ]
```

```
$ grep -c "pipe2" shim_c3.log
0
```

**SIGCHLD coalescing — one signal delivery draining multiple children in a single reap loop** (`shim_c3.log`, verbatim, all `comm=KittyChildMon`). Two forms were captured. First, a single 128-byte read (one `signalfd_siginfo`) followed by **two** real reaps:

```
1883865.061108296 tid=2919 comm=KittyChildMon read(signalfd=9, count=4096) = 128  [1 signalfd_siginfo: SIGCHLD ]
1883865.061148193 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 3141  (WIFEXITED=1 status=0 WIFSIGNALED=0)
1883865.061157040 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 3034  (WIFEXITED=1 status=0 WIFSIGNALED=0)
1883865.061163495 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 0
```

Second, a 256-byte read (two `signalfd_siginfo` in one syscall) immediately drained to `EAGAIN`, then two reaps:

```
1883865.060906870 tid=2919 comm=KittyChildMon read(signalfd=9, count=4096) = 256  [2 signalfd_siginfo: SIGCHLD SIGCHLD ]
1883865.060910696 tid=2919 comm=KittyChildMon read(signalfd=9, count=4096) = -1 Resource temporarily unavailable (errno=11)
1883865.060927103 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 2946  (WIFEXITED=1 status=0 WIFSIGNALED=0)
1883865.060935776 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 3117  (WIFEXITED=1 status=0 WIFSIGNALED=0)
1883865.060938768 tid=2919 comm=KittyChildMon waitpid(pid=-1, opts=WNOHANG|) = 0
```

Across the whole gated run (20 windows released to die together), this produced fewer signal reads than deaths — e.g. **17 `SIGCHLD` reads for 20 reaps** in one run (3 deaths coalesced):

```
$ grep -c "signalfd_siginfo: SIGCHLD" gate_shim_2.log   # signal deliveries
17
$ grep -cE "waitpid\(pid=-1.* = [1-9][0-9]* " gate_shim_2.log   # real reaps
20
```

**Resize de-duplication** (analysis of `shim_c3.log`, 20 progressively-splitting windows):

```
$ grep -c "TIOCSWINSZ" shim_c3.log                              # total resize ioctls
39
$ grep -cE "TIOCSWINSZ, \{rows=-1|cols=-1" shim_c3.log          # sentinel transmissions
0
# consecutive duplicate-size ioctls on the same fd: 0
# example single-fd size sequence (fd=12), every step distinct:
cols=17 cols=8 cols=6 cols=5 cols=4 cols=3 cols=2 cols=1
```

### What this evidence proves

- **CORRECTION #1 holds at runtime.** The mask `[INT HUP TERM CHLD USR1 USR2]` and the `signalfd(… SFD_NONBLOCK|SFD_CLOEXEC) = 9` are created on `comm=kitty` (main), and `read(signalfd=9 …)` happens on `comm=KittyChildMon` (IO); `pipe2` occurs **0** times — Linux is on the `signalfd` path (`kitty/loop-utils.h:14-17`), the self-pipe is macOS/BSD-only. This *is* the decoupling: signal creation on one thread, consumption/mutation on another at a safe point.
- **Coalescing is real and safe.** One `signalfd` read (whether it carried 1 or 2 `signalfd_siginfo`) is followed by a `waitpid(-1, WNOHANG)` loop that reaps *every* ready child and ends on `= 0`. Fewer reads than reaps (17 vs 20) is the observable signature of `SignalSet.child_died` collapsing multiple `SIGCHLD` (`kitty/child-monitor.c:1370-1371`) with a single reap call per loop (`:1526`).
- **De-duplication is complete.** 39 real resize ioctls, **0** sentinel transmissions, **0** consecutive same-size ioctls; every ioctl maps to a genuine grid change (the `fd=12` sequence shrinks monotonically with no repeats) — exactly the `last_reported_pty_size` guard (`kitty/window.py:861`).

### Live-resize debounce timing (labelled: interval value confirmed; live-resize behaviour inferred)

`process_pending_resizes` (`kitty/child-monitor.c:1042`) only runs during an interactive live-resize driven by a window-manager configure-event storm. Headless, no WM sends configure events, and the input injectors (`xdotool`/`wmctrl`/`xte`) are absent; the session `resize_window` directive stores only the *last* resize spec per window (`kitty/session.py:126-133`), so a same-size storm cannot be replayed through the canonical path. The **interval value** `on_end = 0.1` is confirmed from both the source struct/mapping and the live default tuple `(0.1, 0.5)`; the **live-resize debounce *behaviour*** (waiting `on_end` after resizing pauses before asking the program to redraw) is therefore **inferred** from source + official docs, with the reproduction attempts stated above.

---

## Q5 — Are there moments where the system must resolve conflicting views of what is still alive?

**Direct answer.** Yes. The **main/UI thread** (Python `Boss`, holding `window_id_map` and `WindowList.all_windows[]`) and the **IO thread** (`KittyChildMon`, holding the C `children[]` array) maintain two independent liveness views that **transiently diverge** during a rapid burst: the main thread can still be laying out and resizing a window that the IO thread has *already* removed. These divergences are reconciled by three cooperating mechanisms: (1) a single `children_lock` mutex serialising all mutations of the shared child state; (2) a **remove-before-add** staged reconciliation performed every IO-loop iteration under that lock; and (3) **idempotent** Python pops (`pop(id, None)`) that tolerate a dead window being observed twice. The divergence is directly observable; the safety (clean exit, no crash, every window reconciled) is invariant.

### Mechanism (cause → effect), grounded in source

1. **The two views.**
   - *Python (main/UI):* `Boss.window_id_map` (set at `kitty/boss.py:588`) and `WindowList.all_windows[]` (`kitty/window_list.py:147`; class at `:144`; `add_window` at `:329`, `remove_window` at `:373`, `group_for_window` at `:264`). Layout iterates this view to compute geometry and call resizes.
   - *C (IO):* the `children[MAX_CHILDREN]` array (`kitty/child-monitor.c:82`) plus the staging `add_queue[]` / `remove_queue[]` / `remove_notify[]` (`:84`), `monitored_pids[256]` (`:96`), and `reaped_pids[]` (`:98`).

2. **The single lock.** All of the above shared state is mutated under `pthread_mutex_t children_lock` (`kitty/child-monitor.c:87`) via the `children_mutex(op)` macro (`kitty/child-monitor.c:76-77`, expanding to `pthread_mutex_##op(&children_lock)`). (Separate `talk_lock`/`buf_lock` guard remote-control and the screen buffer.)

3. **Remove-before-add reconciliation.** The `io_loop` (`kitty/child-monitor.c:1481`, thread named via `set_thread_name("KittyChildMon")` at `:1489`) begins each iteration with, under the lock: `children_mutex(lock); remove_children(self); add_children(self); children_mutex(unlock);` (`kitty/child-monitor.c:1492-1495`). Removal (`remove_children`, `:1313`, moving `needs_removal` children into `remove_queue[]`) runs **strictly before** addition (`add_children`, `:1281`, draining `add_queue[]` into `children[]`) — both inside the same lock hold. This deterministic order means that within one iteration a departing child is fully retired before any new child is admitted.

4. **Cross-thread death handoff, performed without the lock.** Departures are reported to Python from the **main thread**, not the IO thread: `process_global_state` (`kitty/child-monitor.c:1224`, the registered main-loop callback at `:1218`) calls `parse_input` (`:1236`), which drains `remove_queue[]` into `remove_notify[]` **under** `children_mutex(lock)` (`kitty/child-monitor.c:454-462`), releases the lock, and *then* calls `death_notify` (= `Boss.on_child_death`) per id **with no lock held** (`:522`) — the comment at `:516-517` explains this is required because the locks are non-recursive and the Python callback may re-enter the module. The `death_notify` slot is wired by `ChildMonitor(self.on_child_death, …)` (`kitty/boss.py:370-371`) → C `self->death_notify` (`kitty/child-monitor.c:163,177`).

5. **Idempotent reconciliation of a double view.** The *same* window id can be retired by **two** independent Python paths — per-child-death `Boss.on_child_death` `pop(window_id, None)` (`kitty/boss.py:883-885`) and per-OS-window-close `Boss.on_os_window_closed` `pop(window_id, None)` (`kitty/boss.py:1780-1781`). Both use `pop(…, None)`, so whichever fires second finds the id already gone and does nothing. (`Boss.mark_window_for_close` at `:920` does *not* pop; it only sets `needs_removal` via `child_monitor.mark_for_close` at `:928`.)

6. **Secondary consumer of the same reap loop.** The identical reaping loop also feeds **background-process** liveness: `Boss` imports `monitor_pid` (`kitty/boss.py:98`) and calls `monitor_pid(p.pid)` (`kitty/boss.py:2419`); C `monitor_pid` (`kitty/child-monitor.c:933`) records into `monitored_pids[256]`; `reap_children`'s `mark_monitored_pids` (`:1398`, called `:1423`) records reaped status into `reaped_pids[]`; C notifies via `on_monitored_pid_death`; and `Boss.on_monitored_pid_death` pops with the same idempotent pattern `background_process_death_notify_map.pop(pid, None)` (`kitty/boss.py:2725-2726`). One `waitpid(-1, WNOHANG)` loop thus serves both window children and monitored background pids, each with its own idempotent pop.

### Observed output

Command (rapid teardown of 24 split windows, many dying near-simultaneously, then the OS window closing at quit):

```
LD_PRELOAD=/tmp/libshim.so SHIM_LOG=/tmp/shim_q5.log \
  timeout 6 ./kitty/launcher/kitty --config NONE --debug-rendering \
  -o close_on_child_death=yes --session /tmp/q5.session
```

**The divergence, observed** (`dbg_q5.log`): the main thread issued resizes for ids the IO thread had already removed, and the reported live child count fluctuated across **seven distinct values within the single run**, while the add-queue was always drained:

```
$ grep -oE "children count: [0-9]+" dbg_q5.log | sort -u
children count: 0
children count: 1
children count: 2
children count: 3
children count: 4
children count: 5
children count: 6
$ grep -oE "add queue: [0-9]+" dbg_q5.log | sort -u
add queue: 0
```

**The reconciliation held — zero Python errors across the burst** (`dbg_q5.log`):

```
$ grep -cEi "Traceback|KeyError|OSError|Exception" dbg_q5.log
0
# kitty exit status: 0
```

### What this evidence proves

- **The two views genuinely diverge.** The "Failed to send resize signal to child with id: N (children count: M) (add queue: 0)" diagnostic (from `kitty/child-monitor.c:610`) is the *main thread* resizing ids that the *IO thread* has already pulled from `children[]`. The fact that `children count` takes **{0,1,2,3,4,5,6}** within one run is the live array being shrunk concurrently by `remove_children` on the IO thread while the main thread reads a momentarily-stale Python view — the exact "conflicting views of what is still alive" the question asks about.
- **`add queue: 0` corroborates remove-before-add.** Because `add_children` has already drained `add_queue[]` into `children[]` under the lock, the resize dual-scan's pending branch finds nothing — the staged queue is empty precisely because addition follows removal within the same locked section (`kitty/child-monitor.c:1492-1495`).
- **Idempotent pops reconcile the double view without crashing.** A plain `del window_id_map[id]` would raise `KeyError` on the second of the two teardown paths (child death *and* OS-window close both firing for the same id at quit). Zero `KeyError`/`Traceback` across a 24-window rapid teardown is the observable signature that the `pop(id, None)` guards (`kitty/boss.py:883-885`, `:1780-1781`) are exercised and absorb the double view.

### Labelled inferences for Q5

- **Remove-before-add ordering** is a *deterministic source fact* (`kitty/child-monitor.c:1492-1495`), not a runtime race, and `add_children`/`remove_children` contain **no logging** (event-loop `EVDBG` covers only "Processing global state" at `kitty/child-monitor.c:1225`), so a `--extra-logging=event-loop` rebuild adds no *direct* trace of the ordering. It is therefore reported as **source-confirmed ordering with observable corroboration** (`add queue: 0` in the divergence diagnostic).
- **The specific "second `on_child_death` finds `window is None` and returns early" branch** (`kitty/boss.py:884-885`) could not be printed directly: kitty's embedded interpreter does not honour an injected `PYTHONPATH`/`sitecustomize` (tested — the marker never appeared), and the repository must remain unmodified. That exact branch is therefore **inferred from source**, corroborated by the observed zero-crash rapid teardown.

---

## Option defaults (web-validated)

The two configurable behaviours central to Q3 and Q4 were observed at their defaults (`close_on_child_death = False`, `resize_debounce_time = (0.1, 0.5)` — dumped live above) and their *semantics* were validated against the official kitty documentation as **external references** (not the primary observation).

**`close_on_child_death` (default `no`).** The official documentation (`sw.kovidgoyal.net/kitty/conf/`, mirrored in the Debian `kitty.conf(5)` manpage) states that with the default the terminal stays open when the child exits **as long as other processes are still outputting to the terminal** (e.g. disowned or backgrounded processes), whereas `yes` closes the window as soon as the child exits — and warns that with `yes`, background processes still using the terminal can fail silently because their stdin/stdout/stderr stop working. This confirms the code path: `needs_removal` is set on true PTY EOF regardless, and `reap_children` only *force-marks* the window for removal when `close_on_child_death` is enabled (`kitty/child-monitor.c:1422`). It also frames the honest Q3 reproduction gap precisely: the "hold open" case requires processes that keep the PTY readable, which a passive backgrounded `sleep` does not.

**`resize_debounce_time` (default `(0.1, 0.5)`).** The official documentation states that on platforms such as macOS, where the OS sends explicit start/end live-resize events, the **second** number is used for redraw-after-pause and the first is ignored (redraw is immediate at end of resize); on **other systems only the first number is used** so kitty is "ready" quickly after resizing ends without continuously redrawing. This confirms **CORRECTION #2** exactly: the first number (`on_end = 0.1`, `kitty/state.h:82` / `kitty/options/to-c.h:349`) is the value used on the Linux/X11 observation platform (the `else` branch at `kitty/child-monitor.c:1062`), and the second number (`on_pause = 0.5`, `kitty/options/to-c.h:350`) is the macOS redraw-after-pause value (`kitty/child-monitor.c:1055`).

**SIGUSR1 corroboration.** The documentation's manual config-reload mechanism, "send kitty the SIGUSR1 signal with `kill -SIGUSR1 $KITTY_PID`", corroborates that `SIGUSR1` is one of the handled signals routed through the same `signalfd` — matching the observed mask `[INT HUP TERM CHLD USR1 USR2]` and `KITTY_HANDLED_SIGNALS` (`kitty/child-monitor.c:121`).

**Non-canonical bypass, excluded.** The `kitty @ signal-child` remote-control command can send signals to a child, but it bypasses the real PTY/kernel path; per the run-first rule it is **non-canonical and was not used** for any observation in this document.

---

## Cross-thread reconciliation summary

The whole machinery reduces to a two-thread model over one lock:

- **Main/UI thread (Python `Boss`)** owns `window_id_map` (`kitty/boss.py:588`) and `WindowList.all_windows[]` (`kitty/window_list.py:147`). It creates windows (`Tab.new_window`, `kitty/tabs.py:504`), stages children before layout (`Boss.add_child`, `kitty/boss.py:585-588`), drives resizes through `set_geometry`/`resize_pty` (`kitty/window.py:850-874`), and performs Python teardown in `Boss.on_child_death` (`kitty/boss.py:881`). It also runs `parse_input` (`kitty/child-monitor.c:1236`), which delivers death notifications **outside** the lock.
- **IO thread (`KittyChildMon`)** owns the C `children[]` array (`kitty/child-monitor.c:82`) and the staging queues. Its `io_loop` (`:1481`) reconciles **remove-before-add** under `children_lock` each iteration (`:1492-1495`), polls PTY fds (setting `needs_removal` on EOF `:1535` / `POLLNVAL` `:1545`), drains the `signalfd`, and reaps with `waitpid(-1, WNOHANG)` (`:1418`).
- **Shared, lock-guarded state:** `children[]`, `add_queue[]`, `remove_queue[]`, `remove_notify[]`, `monitored_pids[256]`, `reaped_pids[]` — all mutated only under `children_lock` (`kitty/child-monitor.c:77,87`).

New children flow **main → `add_queue` → (IO drains) → `children[]`**; departing children flow **`children[]` → `remove_queue` → (main drains, no lock) → `on_child_death` pop**. The removal-precedes-addition order, the dual-scan of both live and staged collections on every resize/close, and the idempotent Python pops are what keep the two liveness views consistent even while they momentarily disagree.

---

## Distribution of outcomes (the timing question, reproduced repeatedly)

Because the question is explicitly about "quick succession" and "conflicting views", the same unchanged input was replayed and the **distribution** of outcomes reported — not a single tidy number.

**Distribution 1 — stale-id resize race.** Fixed input `/tmp/dist.session` (24 split windows, staggered lifetimes), `timeout 6 ./kitty/launcher/kitty --config NONE --debug-rendering -o close_on_child_death=yes --session /tmp/dist.session`, run **8×**:

| run | exit | "Failed to send resize" count | distinct `children count` values | signalfd reads | reaps | python errors |
|----:|-----:|------------------------------:|---------------------------------:|---------------:|------:|--------------:|
| 1 | 0 | 59 | 7 | 24 | 24 | 0 |
| 2 | 0 | 58 | 6 | 24 | 24 | 0 |
| 3 | 0 | 61 | 7 | 24 | 24 | 0 |
| 4 | 0 | 59 | 7 | 24 | 24 | 0 |
| 5 | 0 | 60 | 7 | 24 | 24 | 0 |
| 6 | 0 | 62 | 7 | 24 | 24 | 0 |
| 7 | 0 | 59 | 6 | 24 | 24 | 0 |
| 8 | 0 | 59 | 7 | 24 | 24 | 0 |

*Variable (timing-dependent):* the stale-id diagnostic count ranges **58–62**, and the number of distinct live-child-count values is **6–7**. *Invariant:* exit `0`, **0** Python errors, and all **24** children reaped, every run. Staggered deaths ⇒ signalfd reads == reaps == 24 (no coalescing for this input).

**Distribution 2 — SIGCHLD coalescing.** Gated input `/tmp/gate.session` (20 windows busy-waiting on a release file, freed together so deaths synchronise), `-o close_on_child_death=yes`, run **6×**:

| run | exit | signalfd SIGCHLD reads | reaps | coalesced (reaps − reads) | python errors |
|----:|-----:|-----------------------:|------:|--------------------------:|--------------:|
| 1 | 0 | 20 | 20 | 0 | 0 |
| 2 | 0 | 17 | 20 | 3 | 0 |
| 3 | 0 | 16 | 20 | 4 | 0 |
| 4 | 0 | 17 | 20 | 3 | 0 |
| 5 | 0 | 19 | 20 | 1 | 0 |
| 6 | 0 | 19 | 20 | 1 | 0 |

*Variable (timing-dependent):* signalfd reads range **16–20**, i.e. **0–4** deaths coalesce per run. *Invariant:* **20** reaps (all children reaped), exit `0`, **0** Python errors, every run.

**Interpretation.** The run-to-run inconsistency the question is about is real and reproduced: *how many* stale-id resizes fire (58–62) and *how much* SIGCHLD coalescing occurs (0–4) depend on scheduling. What is deterministic is the **safety**: clean exit, zero crashes, every child reaped, every window reconciled — the consistency guarantees hold regardless of timing. Coalescing itself is input-timing-dependent: staggered deaths give a 1:1 signal:reap ratio, synchronised deaths give reads < reaps.

---

## Inferred vs observed (coverage ledger)

| Claim | Status | Basis |
|-------|--------|-------|
| Add-before-layout ordering (first resize ioctl succeeds) | **Observed** | `shim_q1.log` first ioctl per fd `= 0`; `kitty/tabs.py:534-535` |
| `Child launched` / `SIGWINCH sent to child` one-shot vs repeat | **Observed** | `dbg_q1.log`; `kitty/window.py:866,871,873` |
| Sentinel `(-1,-1,-1,-1)` never transmitted | **Observed** | 0 sentinel ioctls in `shim_c3.log`; `kitty/window.py:579,861` |
| Stale-id resize tolerated (diagnostic, no crash) | **Observed** | `dbg_run1.log` ×108, 0 exceptions; `kitty/child-monitor.c:610` |
| `EBADF`/`ENOTTY` tolerance triggered *in kitty* | **Inferred** (defensive; unreachable via canonical resize under `children_lock`) | probe micro-experiment; `kitty/child-monitor.c:581`; attempts listed |
| Idempotent Python pop resolves double view (no `KeyError`) | **Observed** (corroboration) | 24-window teardown, 0 `KeyError`; `kitty/boss.py:883-885,1780-1781` |
| `needs_removal` → `remove_children` → `hangup` `SIGHUP` | **Observed** | `shim_dedup.log` `killpg(…,SIGHUP)=0`; `kitty/child-monitor.c:1299,1313` |
| `waitpid(-1, WNOHANG)` reap loop, gated by `close_on_child_death` | **Observed** | `shim_c3.log` reap loop; `kitty/child-monitor.c:1418,1422,1526` |
| Default hold-open "while other processes use the terminal" | **Inferred** (headless PTY returns `EIO` after session-leader exit) | probe `read(master)=EIO`; docs; attempts listed |
| CORRECTION #1 — Linux uses `signalfd`, not self-pipe | **Observed** | `signalfd(...)=9` + `read(signalfd=9)`; `pipe2`=0; `kitty/loop-utils.h:14-17` |
| SIGCHLD coalescing (reads < reaps) | **Observed** | `shim_c3.log` / `gate_shim_2.log` 17 reads / 20 reaps; `kitty/child-monitor.c:1370-1371,1526` |
| Resize de-duplication (0 dup, 0 sentinel) | **Observed** | `shim_c3.log` 39 ioctls, 0 dup; `kitty/window.py:861` |
| CORRECTION #2 — `on_end=0.1`, `on_pause=0.5` | **Observed** (values) + doc-validated | live tuple `(0.1,0.5)`; `kitty/state.h:82`, `kitty/options/to-c.h:349-350` |
| Live-resize debounce *behaviour* on pause/end | **Inferred** (no WM/injector headless) | source + official docs; attempts listed |
| Remove-before-add ordering each IO iteration | **Source-confirmed** + corroborated (`add queue: 0`) | `kitty/child-monitor.c:1492-1495`; no logging in add/remove |
| Divergent liveness views during burst | **Observed** | `dbg_q5.log` `children count` ∈ {0..6}, `add queue: 0` |

---

## References (read-only citation targets — none modified)

- `kitty/child-monitor.c` — staging, `needs_removal` (`:67`), `children_mutex` (`:77`) / `children_lock` (`:87`), shared arrays (`:82-98`), `add_child` (`:305`), `mark_child_for_close` (`:541`), `pty_resize` (`:577`), `resize_pty` (`:592`, diagnostic `:610`), `monitor_pid` (`:933`), `process_pending_resizes` (`:1042`), `parse_input` handoff (`:454-462,516-522`), `KITTY_HANDLED_SIGNALS` (`:121`), `add_children` (`:1281`), `remove_children` (`:1313`), `hangup` (`:1294`) / `cleanup_child` (`:1306`), `SignalSet`/`handle_signal` (`:1359-1371`), `mark_monitored_pids` (`:1398`), `reap_children` (`:1413`, gate `:1422`), `io_loop` (`:1481`, remove-before-add `:1492-1495`, reap `:1526`, `POLLNVAL` `:1545`).
- `kitty/loop-utils.c`, `kitty/loop-utils.h` — `signalfd` vs self-pipe selection (`.h:14-17,35-37,52-53`), `init_signal_handlers` (`.c:41-52`), `read_signals` (`.c:131-133`).
- `kitty/boss.py` — `add_child` (`:585-588`), `on_child_death` (`:881-885`), `on_os_window_closed` (`:1780-1781`), `mark_window_for_close` (`:920-928`), `monitor_pid` use (`:98,2419`), `on_monitored_pid_death` (`:2725-2726`), `ChildMonitor(...)` wiring (`:370-371`).
- `kitty/tabs.py` — `new_window` (`:504`), add-before-layout (`:534-535`), `remove_window` (`:580`).
- `kitty/window_list.py` — `WindowList` (`:144`), `all_windows[]` (`:147`), `group_for_window` (`:264`), `add_window` (`:329`), `remove_window` (`:373`).
- `kitty/window.py` — `last_resized_at` (`:562`), `child_is_launched` (`:578`), `last_reported_pty_size` sentinel (`:579`), `set_geometry` (`:850`), resize guard/calls (`:861-874`).
- `kitty/child.py` — `Child` fork + PTY spawn, `mark_terminal_ready`.
- `kitty/options/definition.py`, `kitty/options/types.py`, `kitty/options/utils.py`, `kitty/options/to-c.h` — `close_on_child_death` (`types.py:500`, `definition.py:2920`), `resize_debounce_time` (`types.py:568`, `definition.py:1182-1183`, `utils.py:673`, `to-c.h:349-350`).
- `kitty/state.h` — `resize_debounce_time { on_end, on_pause }` (`:82`).
- `kitty/session.py` — `parse_session` (`:151`), `create_sessions` (`:219`), `launch`/`resize_window` directives (`:184,203`).
- `kitty/main.py` — canonical launch entry point.
- `Makefile` (`:12,22,25`), `setup.py` (`:609`), `dev.sh` — canonical build commands.

**External documentation references** (validation only): official kitty configuration docs `sw.kovidgoyal.net/kitty/conf/`; `kitty.conf(5)` manpage (`manpages.debian.org/trixie/kitty/kitty.conf.5.en.html`).

---

*Repository integrity: this investigation was read-only. No existing repository file was modified; the sole persistent artifact is this document. All temporary observation scripts and logs were kept under `/tmp` and removed afterward.*
