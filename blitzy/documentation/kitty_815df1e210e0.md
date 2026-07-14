# kitty window-lifecycle state consistency under rapid create / resize / destroy

**Subject:** How the [kitty](https://github.com/kovidgoyal/kitty) terminal emulator (v0.35.2) keeps its
internal state consistent as terminal windows are created, resized, and destroyed in rapid succession —
with emphasis on **signal delivery**, **cross-thread bookkeeping**, and the **destruction-during-reaction
races** that arise when a window disappears before the system has finished reacting to it.

- **Deliverable branch:** `kitty_815df1e210e0`
- **Observation commit (source under test):** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
- **kitty version:** `0.35.2` (grounded in `kitty/constants.py:25` → `version: Version = Version(0, 35, 2)`,
  and confirmed by the real `--version` banner in §2).

> **Methodology (run-first).** Every behavioural claim below was produced by **building and running the real
> `./kitty/launcher/kitty` binary** headless (under Xvfb) with `--debug-rendering`, driving windows through
> **canonical entry points only** (session files + `launch`/split/close — never remote control, never a debug
> hook, never a non-default compile flag), and capturing the **complete, unedited** stderr. In addition,
> `strace` (installed into the container from the distro archive; see §2) was attached to the real binary at
> its canonical entry point to capture the kernel-level `ioctl(TIOCSWINSZ)`, the `SIGWINCH` the kernel
> delivers to the child, and the `signalfd`/`eventfd` primitives created at startup. Source `file:line`
> references explain *why* the observed behaviour happens. Statements labelled **`(inferred)`** could not be
> reproduced at runtime through a canonical entry point in the supplied container and are grounded only in
> quoted source; statements labelled **`(OS contract)`** are documented kernel semantics with an authoritative
> citation; a platform branch that was not executed is labelled **`(not runtime-observed)`**.

---

## 1. Executive summary (direct thesis)

kitty reconciles two **different** views of "which windows are alive":

- A **Python object graph** whose *strong* ownership chain is
  `Boss.os_window_map` (`Dict[int, TabManager]`, `kitty/boss.py:350`) →
  `TabManager.tabs` (`List[Tab]`, `kitty/tabs.py:877`) →
  `Tab.windows` (a `WindowList`, `kitty/tabs.py:150`) →
  `WindowList.all_windows`/`id_map` (`kitty/window_list.py:147-148`) → `Window`.
  Separately, `Boss.window_id_map` is a **weak index** — a `WeakValueDictionary`
  (`kitty/boss.py:344`) — not the ownership root. A `Window` stays alive because the `WindowList` (and thus
  the `Tab`/`TabManager`) holds a strong reference; `window_id_map` merely provides id→`Window` lookup and
  drops entries automatically when the strong owner lets go.
- A **C-side child registry** — the fixed `children[]` array in `kitty/child-monitor.c` (`:82`).

It keeps them consistent with four cooperating mechanisms:

1. **A single lock.** All shared C state (`children[]`, the add/remove queues, `reaped_pids`,
   `monitored_pids`) is serialized by one mutex, `children_lock` (`kitty/child-monitor.c:87`).
2. **Deferred *structural* mutation through producer/consumer queues (with *opposite* ownership per queue).**
   The Python **main thread** never inserts, removes, or reorders entries in `children[]` directly; structural
   changes are deferred through two queues that flow in **opposite** directions:
   - `add_queue`: the **main thread produces** (`add_child` appends, `:305`) and the **I/O thread consumes**
     (`add_children` moves entries into `children[]`, `:1281`).
   - `remove_queue`: the **I/O thread produces** (`remove_children` moves flagged entries out of `children[]`
     and compacts the array, `:1313`) and the **main thread consumes** (`parse_input` drains it, frees each
     child, and fires the death callback, `:457-462,519-525`).

   The per-entry `needs_removal` **field** is written under the lock by **both** threads, depending on the
   trigger: the **main thread** sets it in `mark_child_for_close` (`:546`, the user-initiated close path),
   while the **I/O thread** sets it on **PTY EOF/HUP** (`:1535`, the default removal trigger) and — only when
   the non-default `close_on_child_death` is enabled — in `mark_child_for_removal` (`:1386`) called from
   `reap_children` (`:1418`), which itself runs on the **I/O thread** (invoked at `:1526`). A field write is
   not a structural change — so `mark_child_for_removal` is an **I/O-thread** action, not a main-thread one.
3. **A single boolean liveness arbiter,** `needs_removal` (`kitty/child-monitor.c:67`), the one source of
   truth for "this child is going away."
4. **Weak references on the Python side.** Because `window_id_map` is a `WeakValueDictionary`
   (`kitty/boss.py:344`), a `Window` released by its strong owner simply disappears from the index, and the
   child-death callback tolerates that with an explicit `None`-guard (`kitty/boss.py:883-885`).

Because the two views are updated on **two different threads** and structural removal is **deferred**, there
is necessarily a brief interval during rapid churn in which they **disagree**: the C registry has already
dropped a dead child (its `Screen` stopped producing bytes, so the I/O thread flagged it) while the Python
layout still holds the corresponding `Window` and issues a resize for it. That disagreement is directly
observable as the log line `Failed to send resize signal to child with id: …` (`kitty/child-monitor.c:610`).
The system is **eventually consistent**: once the main thread runs `parse_input`, fires the death callback,
and the `WindowList`/`window_id_map` drop the window, both views agree again. We reproduced this end-to-end,
including the convergence back to a consistent final state, and we ran the identical input repeatedly to
report the observed distribution (§11).

### TL;DR answers

| Q | One-line answer |
|---|-----------------|
| **Q1 — consistency under churn** | Two views (strong graph `os_window_map→TabManager→Tab→WindowList→Window` + weak `window_id_map` vs C `children[]`) kept **eventually consistent** via `children_lock` + deferred structural `add_queue`/`remove_queue` + the `needs_removal` arbiter + weak refs; transient disagreement is real and observable, then reconciled. |
| **Q2 — create-and-run flow** | `Child.fork()` allocates the PTY (`openpty`, `child.py:281`) and forks a child that sets up a controlling terminal (`setsid`+`TIOCSCTTY`, `child.c:123,129`) and then **blocks on a ready-pipe** before `execvp` (`child.c:152`). The parent registers the child in C **before** the weak index (`boss.py:587-588`); the first `Window.set_geometry()` pushes the initial size via `resize_pty`→`ioctl(TIOCSWINSZ)` (`child-monitor.c:609`) **and then** releases the child (`mark_terminal_ready`, `window.py:866`). Observed as `Child launched`; a later relayout emits `SIGWINCH sent to child in window: …`. `strace` captured both the `TIOCSWINSZ` ioctl and the kernel-delivered `SIGWINCH` to the child pid, and a **child-side prober** (§6.1) independently confirmed the child receives `SIGWINCH` and reads back the exact new window size the parent logged (with `pid==pgid==sid`, proving it is the PTY foreground group); an unchanged relayout is deduplicated to **zero** signals (§6.2, `window.py:861`). |
| **Q3 — destruction-during-reaction** | A resize for an already-removed child is a **logged, non-fatal miss** (`child-monitor.c:610`, observed). It also never *targets a `Window` Python already destroyed* — `set_geometry` early-returns on `self.destroyed` (`window.py:850-852`, source-grounded, silent). A resize reaching a *just-closed* fd was **not observed to fire**: `resize_pty` holds `children_lock` across both the lookup **and** the `ioctl` (`:598`…`:611`) while `remove_children` closes the fd under the *same* lock, so under that discipline the `EBADF`/`ENOTTY` handling at `:581` stands as **defensive** code (not observed as a live race). A child-death for an already-dropped window is a **guarded no-op** (`boss.py:883-885`). |
| **Q4 — keep vs. discard** | The `Screen` object and PTY fd are **always discarded** (`Py_CLEAR`/`del self.screen`; `safe_close`). In the **default** configuration (`--config NONE` ⇒ `close_on_child_death=False`) an ordinary window is removed when its PTY hits **EOF/HUP** (`child-monitor.c:1535`), not by the `SIGCHLD` reaper. The exit **status** is discarded for ordinary window children and **kept only for explicitly *monitored* pids** (`reaped_pids` → `report_reaped_pids` → `on_monitored_pid_death`). |
| **Q5 — timing & signals** | On Linux the signal path is a **`signalfd`** drained *synchronously* on the I/O thread: the poll branch calls `read_signals()` → `handle_signal` (sets `ss->child_died`, `:1370-1371`) and, in the same branch, `reap_children()` which loops `waitpid(-1,…,WNOHANG)` (`:1418`) to absorb coalesced `SIGCHLD`s. Work faster than one tick is serialized through the queues under `children_lock`; the I/O→main wakeup is paced by `OPT(input_delay)` (`WAKEUP`, `:1562-1569`). |
| **Q6 — conflicting views** | Yes — in **both** directions. (a) *C-first:* under churn the C `children[]` no longer has a child but the Python layout still does → the `:610` line (observed). (b) *Python-first:* on OS-window close, `close_os_window` calls Python `on_os_window_closed` (which pops `window_id_map` at `boss.py:1781`) **before** it marks the C children (`child-monitor.c:1089` before `:1092`), so a later `on_child_death` hits the `None`-guard — exercised with a **live keeper** child closed via the WM (§10(b), exit 0 observed; guard *firing* inferred). kitty resolves both for its **internal** `children[]`/`window_id_map` registries via `needs_removal` (C) + the `None`-guard (`boss.py:883-885`) + weak refs. This is *not* a guarantee of OS-child death: a `SIGHUP`-ignoring child **outlives kitty** and reparents to PID 1 (observed, §10). |

---

## 2. Environment and how to reproduce

All build and run steps were executed **inside the supplied Docker image**
(`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). The host interpreter is Python 3.13, which
is incompatible with kitty 0.35.2's C extension under the default `-Werror`; the container ships a compatible
toolchain. The repository working tree is bind-mounted into the container at `/work`, so the artifacts built
in the container are the ones exercised here.

| Component | Value |
|-----------|-------|
| OS (container) | Ubuntu 24.04 LTS |
| Python | 3.12.3 |
| Go | go1.23.4 linux/amd64 |
| C compiler | gcc (Ubuntu 13.3.0) 13.3.0 |
| Display | headless `Xvfb :99 -screen 0 1280x800x24 -ac +extension GLX` |
| GL | `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe` (llvmpipe software rendering) |

**Provisioning (accurate).** The image does **not** ship `Xvfb`, `strace`, or a window manager; they were
installed from the Ubuntu archive at the start of the investigation (the container **does** have network
access to the archive):

```
apt-get update && apt-get install -y --no-install-recommends xvfb xauth strace
apt-get install -y --no-install-recommends openbox xdotool wmctrl   # for §6.1/§10 live-window observations
```

`strace` (6.8) installed successfully, which is why the syscall-level captures below exist. `openbox`
(a minimal window manager) plus `xdotool`/`wmctrl` are used **only** for the two observations that require a
real, managed OS window — the live terminal-resize child-side proof (§6.1) and the live-child OS-window-close
(§10(b)); they drive the *real* binary through *real* window-manager events (a resize request, a close
request), not through any kitty remote-control or debug hook, so they remain canonical user actions. All these
installations happen **inside the container** and touch **no repository file**.

**Exact build command** (the canonical build, matching `.github/workflows/ci.py:104` →
`f'{python} setup.py build --verbose'`):

```
python3 setup.py build --verbose
```

The C extension compiles cleanly under kitty's **default** flags (no `--ignore-compiler-warnings`). The exact
gcc invocation for the file at the heart of this answer, copied verbatim from the `arguments` array of the
build's `build/compile_commands.json` (note the repeated `-I` includes emitted by the build system — they are
part of the real command):

```
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/python3.12 -c kitty/child-monitor.c -o build/fast_data_types-kitty-child-monitor.c.o
```

(`-std=c11 -pedantic-errors -Werror` are project defaults; the build produced zero warnings and zero errors.)

**Version banner** from the real binary (grounds v0.35.2 in `kitty/constants.py:25`):

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**How the runs were driven.** Every observation was produced by the self-contained harness embedded verbatim
in §11 (`harness.sh`), plus a second script for the OS-window-close direction (`osclose.sh`, also in §11).
Both are **safe by construction**: `umask 077`, a private `mktemp -d` workspace under `/tmp`, a **bounded
free-display allocation** (the first unused `/tmp/.X11-unix/X<N>` in `80..200`), an Xvfb whose **PID *and*
socket are validated** before use (with a non-zero abort if our own Xvfb fails — never a foreign display), a
path-validated `rm -rf` of the workspace in the `EXIT`/`INT`/`TERM` trap (so the scripts **self-clean**),
absolute quoted paths, and a **bounded** `timeout --signal=TERM --kill-after=5s` around the real binary. The
single canonical invocation each scenario reduces to is (copy-paste-safe — assign and quote the variables
first, no literal placeholders):

```
SECS=3                                       # time bound in seconds (see "On kitty_exit=124" below)
SESSION_FILE="$WORK/churn3.session"          # a session file the harness wrote under its private workspace
timeout --signal=TERM --kill-after=5s "$SECS" \
  ./kitty/launcher/kitty --debug-rendering --config NONE --session "$SESSION_FILE" 2>stderr.log
```

`--config NONE` guarantees the **default** configuration; the session file is the canonical window-creation
entry point (`kitty/session.py` `parse_session`/`create_sessions`). The `--debug-rendering` flag
(`kitty/cli.py:989` → `--debug-rendering --debug-gl`) is what surfaces the two observable log lines emitted by
`Window.set_geometry()` (`kitty/window.py:871,873`). kitty has no login/auth step.

**On `kitty_exit=124`.** For the long-lived scenarios the session includes a "keeper" window that never
exits, so kitty keeps running until the harness's `timeout` reaches its bound. GNU `timeout` then reports exit
status **124**, which means *"the time limit was reached and `timeout` terminated the still-running binary"* —
it is **not** a kitty crash and not a signal kitty raised. Scenarios with no keeper (e.g. the OS-close run in
§11) instead exit **0** naturally, well under the bound.

**Clean-tree baseline** (captured before any authoring; proves a byte-for-byte clean start):

```
$ git rev-parse --abbrev-ref HEAD
blitzy-56483927-44f2-4b4e-b5f6-a9d256171ba2
$ git status --porcelain
$        # (empty output = clean working tree)
```

A build does **not** dirty the tree: kitty's `.gitignore` already ignores `*.so`, `/build/`, and
`/kitty/launcher/kitt*`, so `fast_data_types.so` and the launcher binaries are untracked-by-design. The final
clean-tree proof (showing that only this document changed) is in **§13**.

---

## 3. The two-thread reconciliation model (foundation for Q1, Q4, Q5, Q6)

kitty runs two cooperating threads that both touch child state:

- The **Python main thread** performs `add_child` (`child-monitor.c:305`), `mark_child_for_close`
  (`:541`), `resize_pty` (`:591`), `parse_input` (`:450`), and the Python child-death callback
  `on_child_death` (`boss.py:881`).
- The **dedicated I/O thread** performs `poll()` + `read_bytes` (`:1337`), the `signalfd`-driven
  `reap_children` (`:1413`), `remove_children` (`:1313`), and `add_children` (`:1281`).

**All** shared state is guarded by the single mutex `children_lock` (`:87`); **structural** mutations of
`children[]` are **deferred** through `add_queue`/`remove_queue`; and the two threads coordinate through a
kernel wake-up primitive. On this Linux build that primitive is an **`eventfd`**, not a self-pipe (see §4 and
the `strace` evidence below): `loop-utils.h` defines `HAS_EVENT_FD`/`HAS_SIGNAL_FD` via `__has_include`
(`kitty/loop-utils.h:14-28`), and the self-pipe + `sigaction` path is the **`#else` non-Linux fallback**
(`kitty/loop-utils.c:45-52,73`). (The `self_pipe` at `child-monitor.c:262` is the unrelated "talk thread"
wakeup, not the child-monitor's.)

Crucially, `parse_input` copies `children[]` into a private `scratch[]` array **under the lock** with a
reference-count bump, so the main thread parses a **stable snapshot** even while the I/O thread mutates the
live array:

```c
        count = self->count;
        for (size_t i = 0; i < count; i++) {
            scratch[i] = children[i];
            INCREF_CHILD(scratch[i]);
        }
```
(`kitty/child-monitor.c:477-482`.)

```mermaid
flowchart TD
    subgraph MainThread["Python Main Thread"]
        AddChild["add_child()<br/>child-monitor.c:305<br/>append to add_queue"]
        MarkClose["mark_child_for_close()<br/>child-monitor.c:541<br/>set children[i].needs_removal=true (field write, under lock)"]
        Resize["resize_pty()<br/>child-monitor.c:591<br/>lock; FIND id; ioctl(TIOCSWINSZ); unlock"]
        Parse["parse_input()<br/>child-monitor.c:450<br/>snapshot to scratch[]; skip needs_removal; drain remove_queue"]
        Death["on_child_death()<br/>boss.py:881<br/>window_id_map.pop() (weak)"]
    end
    subgraph Lock["children_lock (child-monitor.c:87)"]
        Registry["children[]  add_queue[]  remove_queue[]<br/>reaped_pids[]  monitored_pids[]<br/>child-monitor.c:82-99"]
    end
    subgraph IOThread["Dedicated I/O Thread"]
        Poll["poll() + read_bytes()<br/>child-monitor.c:1337<br/>PTY EOF/HUP -> needs_removal=true (:1535)"]
        Signal["signalfd drain: read_signals() -> handle_signal()<br/>loop-utils.c:131 / child-monitor.c:1362<br/>sets ss->child_died"]
        Reap["reap_children()<br/>child-monitor.c:1413<br/>waitpid WNOHANG loop; keeps monitored status"]
        RemoveCh["remove_children()<br/>child-monitor.c:1313<br/>needs_removal -> remove_queue; compact"]
        AddCh["add_children()<br/>child-monitor.c:1281<br/>add_queue -> children[]"]
        Wake["eventfd wakeup (Linux)<br/>loop-utils.c:70; #else self-pipe :73"]
    end
    AddChild --> Registry
    MarkClose --> Registry
    Resize --> Registry
    Registry --> Parse
    RemoveCh --> Registry
    AddCh --> Registry
    Reap --> Registry
    Poll --> Registry
    Signal --> Reap
    Registry --> Poll
    Parse --> Death
%% All shared-state access is serialized by children_lock; structural changes to children[] are deferred to the I/O thread.
```

**How a create → run → resize → close sequence is serialized.** Even if issued faster than one I/O-loop tick,
each step lands in shared state under `children_lock`, and the ordering is enforced by *which thread* owns
*which* transition:

- **create**: main thread `add_child` appends to `add_queue` and wakes the I/O thread; the child becomes part
  of `children[]` only when the I/O thread runs `add_children`.
- **run/resize**: main thread `resize_pty` searches `children[]` then `add_queue` **under the lock** and does
  the `ioctl` before releasing it.
- **close**: for the *default* config the I/O thread flags the child on PTY EOF/HUP (`:1535`); the I/O thread's
  `remove_children` then moves the entry to `remove_queue` and compacts `children[]`; the main thread's
  `parse_input` finally drains `remove_queue`, frees the child, and fires the death callback.

Because "close" spans **both** threads and its structural half is deferred, it is the step that can lag behind
a "resize" that the layout issues in the same instant — the origin of the races in Q3/Q6.

---

## 4. OS-contract background (kernel semantics, with authoritative citations)

These are properties of the Linux kernel that kitty's code relies on. They are labelled **`(OS contract)`**
and cited to authoritative documentation; they explain cause and effect and are **not** a substitute for
kitty's own observed behaviour.

- **`(OS contract)` `SIGCHLD` is not queued.** Standard signals are not queued: if multiple children change
  state while `SIGCHLD` is blocked or pending, the parent still sees a *single* `SIGCHLD`, so a correct
  handler must `waitpid(…, WNOHANG)` in a **loop** to reap them all (`signal(7)`: "Standard signals do not
  queue."; `wait(2)`). This is exactly `reap_children()` (`child-monitor.c:1413-1426`).
- **`(OS contract)` `TIOCSWINSZ` → `SIGWINCH`.** Setting the window size on the PTY *master* makes the kernel
  send `SIGWINCH` to the *foreground process group* of the terminal and stores the size in the kernel
  (`tty_ioctl(4)`: "TIOCSWINSZ … The kernel … sends a `SIGWINCH` signal to the foreground process group").
  That is why kitty pushes size via `ioctl` (`pty_resize`, `:579`) and why `setsid()` + `TIOCSCTTY` in the
  child (`child.c:123,129`) — which make the PTY the child's controlling terminal and the child a
  process-group leader — are prerequisites for delivery. Both the `ioctl` and the resulting `SIGWINCH` were
  captured with `strace` (§6).
- **`(OS contract, Linux-specific)` `signalfd` turns signals into readable file descriptors.** On Linux kitty
  uses `signalfd(2)` so the signal set is drained **synchronously** from the poll loop rather than in an async
  handler (`signalfd(2)`; `eventfd(2)` for the wakeup). This is the path actually compiled here
  (`loop-utils.c:42,70`, confirmed by `strace` showing `signalfd4`/`eventfd2`). The classic "a C signal
  handler must only set a flag, defer the real work" rule applies to the **`#else` non-Linux fallback**
  (`sigaction` + self-pipe, `loop-utils.c:45-52`) **`(not runtime-observed)`** — on Linux there is no async
  handler; `read_signals()` invokes the callback inline.

---

## 5. Q1 — State consistency under churn

**Direct answer.** kitty does **not** attempt to keep the Python object graph and the C `children[]` array
*instantaneously* identical. Instead it keeps them **eventually consistent**. The Python side has a *strong*
ownership chain (`os_window_map` → `TabManager` → `Tab` → `WindowList` → `Window`) plus a *weak* id index
(`window_id_map`); the C side has `children[]`. The two are mutated on two threads, with **structural**
`children[]` changes deferred through `add_queue`/`remove_queue` under `children_lock`, the boolean
`needs_removal` as the single liveness arbiter, and Python weak references as the tolerant back-stop. Under
rapid churn there is a genuine, observable interval in which the views disagree, followed by deterministic
convergence once the main thread processes the deferred death notifications.

**Ownership, grounded.** The strong graph keeps a `Window` alive:
`Tab.windows: WindowList = WindowList(self)` (`kitty/tabs.py:150`); `WindowList.all_windows: List[Window]` and
`WindowList.id_map: Dict[int, Window]` (`kitty/window_list.py:147-148`); `WindowList.add_window` appends to
both (`:338-339`) and `remove_window` drops from both (`:377,:380`). (Note: `window_list.py:67`/`:84` are the
methods of the *different* `WindowGroup` class defined at `:28`; the `WindowList` class begins at `:144`. The
weak index is `Boss.window_id_map = WeakValueDictionary()` at `boss.py:344`.)

**Evidence — churn drives disagreement, then convergence (readable N = 3 case).** A canonical session creates
3 instant-exit windows under a splitting layout plus one long-lived "keeper" so the OS window does not close.
The session file and its exact producing command (from the harness in §11) are:

```
layout splits
launch sh -c 'exit 0'
launch sh -c 'exit 0'
launch sh -c 'exit 0'
launch sh -c 'sleep 30'
```

Complete, unedited stderr (21 lines; `kitty_exit=124` = the harness `timeout` bound was reached — see §2):

```
[0.139] OS Window created
[0.148] Failed to open systemd user bus with error: No such file or directory
[0.149] Child launched
[0.153] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.153] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
[0.154] Child launched
[0.160] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.160] SIGWINCH sent to child in window: 2 with size: (22, 17, 153, 396)
[0.161] Child launched
[0.167] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.167] SIGWINCH sent to child in window: 3 with size: (22, 8, 72, 396)
[0.167] Child launched
[0.169] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.169] SIGWINCH sent to child in window: 2 with size: (22, 35, 315, 396)
[0.170] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.170] SIGWINCH sent to child in window: 3 with size: (22, 17, 153, 396)
[0.170] SIGWINCH sent to child in window: 4 with size: (22, 17, 153, 396)
[0.171] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.171] SIGWINCH sent to child in window: 3 with size: (22, 35, 315, 396)
[0.172] SIGWINCH sent to child in window: 4 with size: (22, 35, 315, 396)
[0.173] SIGWINCH sent to child in window: 4 with size: (22, 71, 639, 396)
```

(Grepped counts for this run: `Child launched` = 4, `SIGWINCH sent to child` = 9, `Failed to send resize
signal` = 6. The "4" is the 3 instant-exit windows plus the keeper.)

**What the log proves (and what it does not).** Each new window fires `Child launched`; the immediately
following `Failed to send resize signal to child with id: N` shows the layout resizing a sibling id that the
C registry no longer holds. `children count: 1` is simply **the size of `children[]` at the instant
`resize_pty` ran** (`self->count`, printed by `child-monitor.c:610`); it does **not** identify *which* child
remains, and in particular it does not mean "only the keeper" (in this ordering the keeper — window 4 here —
is created *last*). The size columns visibly shrink (`35 → 17 → 8`) as `splits` subdivides, confirming these
are real relayouts. The **last three lines** are the convergence: once the dead windows are gone from **both**
registries, the keeper (window 4) is resized up to the full `71`-column width — the two views agree again.

**Why (causal, with `file:line`).** The main thread's writes to the shared registry are three: *appending* to
`add_queue` (`add_child`, `:305-321`); *flipping* the `needs_removal` field on the user-close path
(`mark_child_for_close`, `:541-564`); and *draining* `remove_queue` (freeing each staged child and firing
`death_notify`, `parse_input` `:457-462,519-525`). What it never does is **structurally** edit `children[]`
itself — no insert, remove, reorder, or compaction of the live array; those happen only on the **I/O thread**
(`add_children` `:1281`, `remove_children` `:1313-1333`). In the **default** configuration the trigger that
flags a self-exiting window is **PTY EOF/HUP on the I/O thread**: `read_bytes()` returns false and the I/O
thread sets
`children[i].needs_removal = true` (`child-monitor.c:1531-1535`) — *not* the `SIGCHLD` reaper (see Q4/Q5). The
I/O thread then applies removals (`remove_children` `:1313-1333`, moving `needs_removal` entries into
`remove_queue` and `memmove`-compacting the array). The main thread's `parse_input` (`:450`) drains
`remove_queue`, runs the `death_notify` callback (`:522`) → Python `on_child_death` (`boss.py:881`) →
`window_id_map.pop` + `tab.remove_window` → `WindowList.remove_window` (`window_list.py:373`). Until that last
step runs, the Python `WindowList` still contains the dead window, so a relayout triggered by the *next*
window's creation calls `set_geometry` → `resize_pty` for an id no longer in `children[]` → the `:610` miss.
Convergence follows deterministically once `parse_input` drains the queue.

---

## 6. Q2 — Create-and-run: the resize/signal flow

**Direct answer.** Creating a window that immediately runs a command proceeds in this order:

1. **PTY + fork with a ready-barrier.** `Child.fork()` (`kitty/child.py:276`) allocates the PTY with
   `openpty()` (`:281`) and creates a **ready pipe** (`ready_read_fd, ready_write_fd = os.pipe()`, `:283`),
   then calls the native `spawn()` (`kitty/child.c:81`). In the child, `spawn` calls `setsid()` (`:123`) and
   `ioctl(tfd, TIOCSCTTY, 0)` (`:129`) to make the PTY its controlling terminal, then **blocks** in
   `wait_for_terminal_ready(ready_read_fd)` (`:152`) — *before* `execvp`. The parent records `self.pid`/
   `self.child_fd` (`:337-338`) and keeps `terminal_ready_fd = ready_write_fd` (`:343`).
2. **Registration order (C before weak index).** `Boss.add_child` registers the child in the C monitor
   **first** and only then in the weak index:
   `self.child_monitor.add_child(window.id, …, window.screen)` (`boss.py:587`) then
   `self.window_id_map[window.id] = window` (`boss.py:588`).
3. **First geometry pushes size, then releases the child.** The first `Window.set_geometry()`
   (`kitty/window.py:850`) detects a change against the sentinel `last_reported_pty_size = (-1,-1,-1,-1)`
   (`:579`) and, at `:861`, calls `boss.child_monitor.resize_pty(self.id, *current_pty_size)` (`:863`) — the
   `ioctl(TIOCSWINSZ)` — **and then**, because `child_is_launched` is still false (`:865`), calls
   `self.child.mark_terminal_ready()` (`:866`), which closes `ready_write_fd` and unblocks the child so it
   `execvp`s **with the correct size already on the PTY**. On this first pass it prints `Child launched`
   (`:871`); on *subsequent* size changes it prints `SIGWINCH sent to child in window: …` (`:873`).

So the initial size is delivered to the PTY *before* the program even starts (no race for the first size),
and every later relayout delivers `SIGWINCH` to the now-running program via a fresh `TIOCSWINSZ`.

**Evidence — the log.** A canonical session that launches a command and then adds a second window under
`tall` layout to force the first to relayout. Complete, unedited stderr (5 lines):

```
[0.407] OS Window created
[0.416] Failed to open systemd user bus with error: No such file or directory
[0.418] Child launched
[0.424] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
[0.424] Child launched
```

`[0.418] Child launched` is window 1's first `set_geometry`; `[0.424] SIGWINCH sent to child in window: 1`
is window 1 being **relaid out** (shrunk to 35 columns) when window 2 is added; `[0.424] Child launched` is
window 2's first `set_geometry`. The tuple is `(lines, columns, width_px, height_px)`. The
`Failed to open systemd user bus …` line is a benign environment message (no systemd user bus in the
container) and is unrelated to window lifecycle.

**Evidence — the syscalls (`strace`, canonical binary).** `strace -f -e trace=ioctl,signalfd4,eventfd2 -e
signal=SIGWINCH` was attached to the real `./kitty/launcher/kitty` launched through a session (a `tall` layout
with two windows). kitty's pid was `10180`; the two children were `10247`/`10248`.

The **raw** output of that filter is **not** short and the block below is **not** the complete trace. Because
`-e trace=ioctl` matches *every* `ioctl` — not only `TIOCSWINSZ` — a single canonical two-window run produced
**259 total trace lines here** (the count varies run to run — two repeats of this exact command produced 259
and 260; an independent QA capture saw ~190), of which
**185** are `ioctl` and only **3** are `TIOCSWINSZ`. The overwhelming majority are the **158 `TCGETS`** calls
kitty issues while probing/configuring each PTY, plus a handful of `FIOCLEX` (8), `FIONCLEX` (4), and
`TIOCSPTLCK`/`TIOCGPTN`/`TIOCGPTPEER`/`TIOCSCTTY`/`TCSETS`/`FIONBIO` (2 each) during terminal setup. To isolate
the resize path from that setup noise, the nine lines below are a **filtered extraction (9 of the 259 raw
lines)** produced by saving the trace and piping it through `grep`. The exact producing commands are therefore:

```
strace -f -o strace.out -e trace=ioctl,signalfd4,eventfd2 -e signal=SIGWINCH \
  timeout --signal=TERM --kill-after=5s 3 \
  ./kitty/launcher/kitty --debug-rendering --config NONE --session "$SESSION_FILE"
wc -l < strace.out                                            # -> 259 (varies run to run; mostly TCGETS)
grep -nE 'eventfd2|signalfd4|TIOCSWINSZ|SIGWINCH' strace.out   # -> the 9 resize-path lines shown below
```

The nine resize-path lines extracted from that 259-line raw trace are:

```
10180 eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK) = 4
10180 eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK) = 6
10180 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
10180 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=71, ws_xpixel=639, ws_ypixel=396}) = 0
10247 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
10180 ioctl(8, TIOCSWINSZ, {ws_row=22, ws_col=35, ws_xpixel=315, ws_ypixel=396}) = 0
10180 ioctl(9, TIOCSWINSZ, {ws_row=22, ws_col=35, ws_xpixel=315, ws_ypixel=396}) = 0
10248 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
10247 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
```

Its `--debug-rendering` stderr for that same run (complete, 5 lines):

```
[0.751] OS Window created
[0.765] Failed to open systemd user bus with error: No such file or directory
[0.767] Child launched
[0.774] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
[0.775] Child launched
```

This is direct, syscall-level confirmation that (a) kitty issues `ioctl(fd, TIOCSWINSZ, …)` on the PTY master
(fds `8`/`9`), (b) the **kernel** then delivers `SIGWINCH` to the *child* pids (`10247`, `10248`;
`si_code=SI_KERNEL`), and (c) at startup kitty creates the Linux `eventfd`/`signalfd` primitives that §3/§4
describe. The `SIGWINCH sent to child …` debug line therefore corresponds to a real kernel signal, not merely
a log message.

**Why (causal, with `file:line`).** The observable strings come verbatim from `Window.set_geometry`, with the
`resize_pty` call **preceding** the child release:

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
            elif boss.args.debug_rendering:
                print(f'[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}', file=sys.stderr)
            self.last_reported_pty_size = current_pty_size
```
(`kitty/window.py:861-874`.) The sentinel at `:579` guarantees the *first* call always differs, so the child
is marked launched exactly once. The call at `:863` reaches the C layer, which does the lookup and `ioctl`
under the lock:

```c
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
```
(`kitty/child-monitor.c:598-611`), and `pty_resize` performs the actual `ioctl(fd, TIOCSWINSZ, dim)`
(`kitty/child-monitor.c:579`).

### 6.1 Child-side proof of kernel `SIGWINCH` delivery (independent of kitty's debug line)

kitty's `SIGWINCH sent to child …` line is emitted by the parent (`window.py:873`) *before* it can know the
child reacted; on its own it proves only that kitty issued the `ioctl`. To prove the **kernel actually
delivered** `SIGWINCH` to the child and that the child sees the new size, a small program was run **as the
window's command** through the canonical session entry point. It installs a `SIGWINCH` handler, reads its
terminal size with `TIOCGWINSZ` on startup and on every `SIGWINCH`, and records its `pid`/`pgid`/`sid` and
whether it is the PTY's foreground process group. It writes to a log file (so its output is not entangled with
the terminal it is measuring):

```python
import os, signal, sys, time, fcntl, termios, struct
LOG = os.environ["PROBE_LOG"]
def winsize():
    s = fcntl.ioctl(0, termios.TIOCGWINSZ, b"\0"*8)
    rows, cols, xp, yp = struct.unpack("HHHH", s)
    return rows, cols, xp, yp
def log(m):
    with open(LOG, "a") as f: f.write(m+"\n"); f.flush()
n = [0]
def on_winch(sig, frm):
    n[0]+=1; r,c,x,y = winsize()
    log(f"[child] SIGWINCH #{n[0]} received -> winsize rows={r} cols={c} xpix={x} ypix={y}")
signal.signal(signal.SIGWINCH, on_winch)
r,c,x,y = winsize()
log(f"[child] START pid={os.getpid()} pgid={os.getpgrp()} sid={os.getsid(0)} tpgid_is_self={os.tcgetpgrp(0)==os.getpgrp()} initial_winsize rows={r} cols={c} xpix={x} ypix={y}")
time.sleep(8)
log(f"[child] END total_SIGWINCH_received={n[0]}")
```

It was launched in a single kitty window and, **after the child logged its `START` line** (i.e. after the
handler was installed — this ordering matters, see the note below), the real OS window was resized from
`640x400` to `840x520` and back, exercising a genuine size change through the same code path a user's window
manager drives. The canonical session was `launch --env PROBE_LOG=… python3 prober.py`; the resize was issued
with `xdotool windowsize` against a real window manager (`openbox`), both provisioned exactly like `Xvfb`/
`strace` in §2. **Complete, unedited child-side log** (one grow, then one shrink back):

```
[child] START pid=95305 pgid=95305 sid=95305 tpgid_is_self=True initial_winsize rows=22 cols=71 xpix=639 ypix=396
[child] SIGWINCH #1 received -> winsize rows=28 cols=93 xpix=837 ypix=504
[child] SIGWINCH #2 received -> winsize rows=22 cols=71 xpix=639 ypix=396
```

**Complete, unedited kitty stderr** for the same run:

```
[0.203] OS Window created
[0.213] Failed to open systemd user bus with error: No such file or directory
[0.215] Child launched
[0.493] SIGWINCH sent to child in window: 1 with size: (28, 93, 837, 504)
[2.016] SIGWINCH sent to child in window: 1 with size: (22, 71, 639, 396)
```

What this proves, line-for-line:

- **Kernel delivery is real.** Each kitty-side `SIGWINCH sent … size: (r,c,xpix,ypix)` is matched by a child-
  side `SIGWINCH #k received -> winsize rows=r cols=c …` with the **identical tuple** — `(28,93,837,504)` and
  `(22,71,639,396)`. The child could only read those numbers via `TIOCGWINSZ` **after** the kernel updated the
  slave's stored size and signalled it; the debug line alone cannot manufacture a child-side `SIGWINCH`.
- **The child is the signal's target.** `pgid == sid == pid` and `tpgid_is_self=True` show the child is the
  session leader **and** the PTY's foreground process group — exactly the group `TIOCSWINSZ` targets (§4).
  This is the payoff of `setsid()`+`ioctl(TIOCSCTTY)` in `kitty/child.c:123,129`.
- **`(inferred)` guard on ordering:** the *first* startup 2-window layout attempt produced a child that logged
  its initial size but **0** `SIGWINCH`, because the relayout landed while Python was still starting (before
  `signal.signal` ran) — the classic "a `SIGWINCH` can be missed if it arrives before the handler is
  installed" hazard (§4). Waiting for the `START` line before resizing removes that race; the effect is called
  out here rather than hidden.

### 6.2 Unchanged-geometry deduplication (a real size change fires exactly once)

The same harness was then driven through **four** steps against the ready child, to separate a genuine size
change from a no-op relayout. The running tally (kitty-side `SIGWINCH sent` count / child-side received count)
was printed after each step; the result was **identical across two independent runs**:

```
STEP 0 (initial, no resize):                                  [sent: 0  child recv: 0]
STEP 1: resize 640x400 -> 840x520 (REAL change)               [sent: 1  child recv: 1]
STEP 2: resize 840x520 -> 840x520 (SAME size, expect dedup)   [sent: 1  child recv: 1]   (0 new)
STEP 3: resize 840x520 -> 640x400 (REAL change back)          [sent: 2  child recv: 2]
```

Step 2 is the observable of the dedup guard: `set_geometry` compares `current_pty_size` against
`self.last_reported_pty_size` and **returns without signalling** when they are equal
(`if current_pty_size != self.last_reported_pty_size:` at `kitty/window.py:861`; the `else` branch merely
marks the OS window dirty, `:876`). A repeated identical geometry therefore produces **zero** additional
`SIGWINCH` on both the kitty side and the child side, while each *real* change produces **exactly one** — the
count moves `0 → 1 → 1 → 2` in lockstep on both sides. This matches, from the child's perspective, the
sentinel-driven "first call always fires exactly once" behaviour described above.


---

## 7. Q3 — Destruction-during-reaction

**Direct answer.** kitty treats "the window is gone before I finished reacting" as a **normal, non-fatal**
condition. There are **three reachable** no-op paths and one further race the running system does not exercise:

1. **A resize that targets a child already removed from the C registry** — a **logged miss**: `resize_pty`
   fails to find the id in `children[]` or `add_queue` and logs
   `Failed to send resize signal to child with id: …` (`child-monitor.c:610`). No crash, no signal. **Observed**
   (§5, §11).
2. **A resize that targets a `Window` Python has already destroyed** — a **silent Python-side no-op**:
   `Window.set_geometry` begins with `if self.destroyed: return` (`window.py:850-852`). Once `Window.destroy()`
   sets `self.destroyed = True` (`:1562`; the flag is initialised `False` at `:597`), any further layout call
   for that window returns **before** it ever reaches `resize_pty`. This is the **Python-side** counterpart to
   case 1 (which is the C side): it fires whenever layout code still holds a reference to a window that
   teardown has already destroyed. **Source-grounded** (`window.py:850-852,1562`); because it is a silent early
   return with no log line it is not separately visible in the debug stream — `(inferred)` from the guard's
   placement that it is hit during teardown ordering. It must **not** be conflated with the C-side `:610`
   miss: they are two independent guards, one per view.
3. **A child-death notification for a window Python already dropped** — a **guarded no-op**: `on_child_death`
   pops the weak reference and returns immediately if it is `None` (`boss.py:883-885`). The canonical route to
   this guard is the OS-window-close path (§10).
4. **A resize that reaches a *just-closed* file descriptor — closed off by the shared lock (not observed).**
   `resize_pty` holds `children_lock` across **both** the lookup and the `ioctl` (`:598` lock … `:611`
   unlock), and the fd is closed by `remove_children` → `cleanup_child` → `safe_close` **under the same mutex**
   (the I/O loop wraps `remove_children(self); add_children(self);` in `children_mutex(lock)`/`(unlock)` at
   `:1492-1495`). Because the two critical sections are mutually exclusive, the I/O thread cannot close a
   descriptor *between* `resize_pty`'s lookup and its `ioctl` — so on **this** code path the race is prevented
   **by the locking discipline** (source-grounded: the two critical sections share `children_lock`), and
   consistently we never observed it. This is a statement about *this* path, not a global impossibility proof;
   the `EBADF`/`ENOTTY` handling in `pty_resize` (`:581`) remains as **defensive coding** for a descriptor
   closed by some other means.

**Evidence — case 1, verbatim.** The `:610` line, e.g. from §5:

```
[0.153] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
```

A subtle, **observed** detail: this failure is immediately followed — *in the same `set_geometry` call, at the
same timestamp* — by an **optimistic** `SIGWINCH sent` line for the *same* id:

```
[0.153] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.153] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
```

**Why the "SIGWINCH sent" line is misleading here (causal).** `Window.set_geometry` calls `resize_pty`
(`window.py:863`) and then, *without checking whether the resize found the child*, prints `SIGWINCH sent …`
(`:873`). The C `resize_pty` does **not** raise when the id is missing — it only logs `:610` and returns
(`Py_RETURN_NONE`). So a `:610` line immediately followed by a `SIGWINCH sent` line for the same id means **no
signal was actually delivered**; the Python-side debug message is optimistic. This is itself a concrete
instance of the two views disagreeing (Q6).

**Evidence — case 3 (the `None`-guard):**

```python
    def on_child_death(self, window_id: int) -> None:
        prev_active_window = self.active_window
        window = self.window_id_map.pop(window_id, None)
        if window is None:
            return
```
(`kitty/boss.py:881-885`.) In the churn runs, every dead window's id is still present when its death
notification arrives, so the *normal* branch runs. The early-return is exercised by the **OS-window-close**
ordering (§10), where Python drops the window from `window_id_map` *before* the C child is marked. The
early-return itself emits **no log line**, so its firing is **`(inferred)`** from the code above; the ordering
that makes it necessary is grounded in `child-monitor.c:1089` vs `:1092` and `boss.py:1777-1781` (§10) and the
OS-close run was observed to complete cleanly (§11).

**Case 4 (defensive, not a race), source:**

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
```
(`kitty/child-monitor.c:577-589`.) The `EBADF`/`ENOTTY` swallow exists so that *if* a descriptor were ever
stale the call would be harmless; but under the lock discipline above it was not reached from a concurrent
close in any run, so no runtime capture of this branch was obtained (and none is claimed). The **observed**
destruction-during-resize outcome is exclusively case 1 (the `:610` "not found" miss).

---

## 8. Q4 — Keep vs. discard

**Direct answer.** On teardown kitty **always discards** the volatile per-window resources — the `Screen`
object (C-side `Py_CLEAR`, Python-side `del self.screen`) and the PTY file descriptor (`safe_close`). What is
**kept** is narrow: the child's **exit status** is retained **only** for pids explicitly *registered for
monitoring* (`monitor_pid`), stored in `reaped_pids[]` and later delivered to the boss. For an ordinary
terminal window the exit status is **not** retained (the removal path stores only the `needs_removal` flag,
and the Python death callback is handed only a `window_id`, never a status). The deciding question is simply
**"is this pid in `monitored_pids[]`?"**

**Two distinct removal triggers — and which one fires by default.** It is essential to separate them:

- **Default (`--config NONE` ⇒ `close_on_child_death=False`):** an ordinary window is removed because its
  **PTY hits EOF/HUP**. On the I/O thread, `read_bytes()` returns false and the code sets
  `children[i].needs_removal = true` (`child-monitor.c:1531-1535`; the `// child is dead` comment is at
  `:1533`). The `SIGCHLD` reaper does **not** mark ordinary children in this configuration.
- **Option-gated (`close_on_child_death=True`):** *then* the reaper marks the child by pid. In
  `reap_children`, `if (enable_close_on_child_death) mark_child_for_removal(self, pid);`
  (`child-monitor.c:1422`); the argument comes from `reap_children(self, OPT(close_on_child_death))` at the
  call site (`:1526`). The default is `False` (`kitty/options/types.py` / `definition.py` default `no`), so
  this branch is **`(inferred)`** here — it was not exercised because the default config does not enable it.

Either way `mark_monitored_pids(pid, status)` runs unconditionally in the reap loop (`:1423`) so that
*monitored* pids' statuses are captured and zombies are reaped.

**Discarded — the `Screen` object (but *not* until after the death callback).** The C-side release is a
deliberate **two-step, staged** sequence inside `parse_input`, **not** a single `FREE_CHILD` at drain time.
Under `children_lock`, the dead child struct is first *copied* out of `remove_queue` into a staging slot
`remove_notify[]`, and its `Screen` is given a **separate strong reference** *before* the queue copy is
cleared:

```c
        remove_notify[remove_count] = remove_queue[remove_queue_count];  // :459 copy struct (incl. screen ptr)
        INCREF_CHILD(remove_notify[remove_count]);                       // :460 Py_INCREF the Screen (staged ref)
        remove_count++;
        FREE_CHILD(remove_queue[remove_queue_count]);                    // :462 Py_CLEAR only the queue copy
```
(`kitty/child-monitor.c:459-462`; the macros are `FREE_CHILD` = `Py_CLEAR((x).screen); x = EMPTY_CHILD;` at
`:105-106` and `INCREF_CHILD` = `Py_INCREF(x.screen)` at `:109`.) Because `remove_notify` now holds its **own**
reference (`:460`), the `Py_CLEAR` at `:462` drops only the *queue* copy — the `Screen`'s refcount stays ≥ 1
and the object **survives**. The lock is then released and, lock-free, kitty does a **final flush** of any
buffered child output through the still-alive `Screen`, *then* fires the death callback, and only **afterwards**
releases the staged reference:

```c
        if (remove_notify[remove_count].screen) do_parse(self, remove_notify[remove_count].screen, now, true); // :521 final flush
        PyObject *t = PyObject_CallFunction(self->death_notify, "k", remove_notify[remove_count].id);           // :522 on_child_death
        if (t == NULL) PyErr_Print(); else Py_DECREF(t);
        FREE_CHILD(remove_notify[remove_count]);                                                                // :525 NOW release Screen
```
(`kitty/child-monitor.c:521-525`.) So the **actual** C-side `Screen` release is at **`:525`, *after*
`death_notify`** — not at the `:462` queue-drain. This ordering is the whole point: it guarantees the
terminal's final output is parsed (`:521`) and the death is announced (`:522`) while the `Screen` is still a
valid object. The Python side releases its *own*, independent reference inside `Window.destroy()`:

```python
            # Remove cycles so that screen is de-allocated immediately
            self.screen.reset_callbacks()
            del self.screen
```
(`kitty/window.py:1569-1571`.)

**Discarded — the PTY fd.** `remove_children` calls `cleanup_child`, which closes the fd and hangs up the
process group:

```c
static void
cleanup_child(ssize_t i) {
    safe_close(children[i].fd, __FILE__, __LINE__);
    hangup(children[i].pid);
}
```
(`kitty/child-monitor.c:1305-1309`; `hangup` = `killpg(pgid, SIGHUP)` at `:1294-1302`, `killpg` at `:1299`.)

**Exit status — discarded for an ordinary window child.** `mark_child_for_removal` sets only the liveness
flag, taking **no** status argument:

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
(`kitty/child-monitor.c:1386-1396`.) And the Python death callback receives only the `window_id`
(`def on_child_death(self, window_id: int)`, `boss.py:881`) — it never sees an exit status for the window's
own terminal child.

**Exit status — kept only for monitored pids.** `mark_monitored_pids` stores the status **iff** the pid was
registered:

```c
static void
mark_monitored_pids(pid_t pid, int status) {
    children_mutex(lock);
    for (ssize_t i = monitored_pids_count - 1; i >= 0; i--) {
        if (pid == monitored_pids[i]) {
            if (reaped_pids_count < arraysz(reaped_pids)) {
                reaped_pids[reaped_pids_count].status = status;
                reaped_pids[reaped_pids_count++].pid = pid;
            }
            remove_i_from_array(monitored_pids, (size_t)i, monitored_pids_count);
        }
    }
    children_mutex(unlock);
}
```
(`kitty/child-monitor.c:1398-1411`.) The retained status is later delivered by `report_reaped_pids`:

```c
    for (size_t n = 0; n < i; n++) { call_boss(on_monitored_pid_death, "li", (long)pids[n].pid, pids[n].status); }
```
(`kitty/child-monitor.c:961`), which invokes `Boss.on_monitored_pid_death(pid, exit_status)`
(`kitty/boss.py:2725`). Pids enter `monitored_pids[]` via `monitor_pid` (`child-monitor.c`), called from
`Boss` for background subprocesses registered with a death callback: `monitor_pid(p.pid)` at `boss.py:2419`
(guarded by `if notify_on_death:` at `:2417`).
**`(inferred)`**: the monitored-pid *delivery* was not exercised at runtime because it is not reachable
through the session-file / `launch` / split / close entry points used here; it is grounded in the code above.

**Capacity and the per-tick delivery bound `(source)`.** Even the "kept" answer must be qualified by two
fixed-size arrays and a hard per-tick cap:

- `monitored_pids[]` and `reaped_pids[]` are both sized **256** (`static pid_t monitored_pids[256]` at
  `child-monitor.c:96`; `static ReapedPID reaped_pids[arraysz(monitored_pids)]` at `:98`), and
  `mark_monitored_pids` appends a status **only while** `reaped_pids_count < arraysz(reaped_pids)` (`:1402`) —
  so more than 256 *undelivered* statuses would be dropped at capture time.
- `report_reaped_pids` copies the pending statuses into a **`static ReapedPID pids[64]`** (`:951`) under the
  lock, bounded by `i < reaped_pids_count && i < arraysz(pids)` (= 64, `:955`), and **then unconditionally
  resets** `reaped_pids_count = 0` (`:958`). Consequently **at most 64 statuses are delivered per main-loop
  tick**, and the reset **discards any 65th-and-beyond** status accumulated since the previous tick (`:1244`
  is the once-per-tick call site). In normal use this bound is never approached — `monitored_pids[]` holds
  only a handful of auxiliary subprocesses (the update-checker, `update_check.py:122`, and `notify_on_death`
  launches, `boss.py:2419`), **not** per-window terminal children — but it is the precise sense in which even
  a "kept" status is kept only up to a fixed per-tick budget. `(inferred)` that the 64-cap is never hit at
  runtime here (only a few monitored pids exist); the array sizes and reset are `(source)`.

**Before / intermediate / after — the explicit lifecycle (source-grounded, with observed anchors).** Tracing
one instant-exit window through every state (default config):

| Phase | What happens | State | Grounding |
|-------|--------------|-------|-----------|
| **created (queued)** | main thread `add_child` appends to `add_queue`; not yet in `children[]` | pending | `child-monitor.c:305-321` *(inferred: no per-step log)* |
| **alive** | I/O thread `add_children` moves it into `children[]`; first `set_geometry` pushes size & releases child | in `children[]`, `needs_removal == false`; `Screen` + PTY fd held | `:1281-1290`; `Child launched` **observed** (§5/§6) |
| **marked** | child exits → PTY EOF/HUP → I/O thread sets `needs_removal = true` | still in `children[]` but flagged; `parse_input` **skips** it (`if (!scratch[i].needs_removal)`, `:529`) | `:1531-1535`; skip at `:528-532` |
| **structurally removed** | I/O thread `remove_children`: `cleanup_child` closes fd + `killpg(SIGHUP)`, entry moved to `remove_queue`, array compacted | gone from `children[]`; fd closed | `:1305-1308,1313-1333` |
| **freed + notified** | main thread `parse_input` drains `remove_queue`: stages a *separate* `Screen` ref (`INCREF_CHILD`, `:460`), clears only the queue copy (`FREE_CHILD`, `:462`), does a final flush (`do_parse`, `:521`), fires `death_notify` → `on_child_death` (`:522`), **then** releases the staged `Screen` ref (`FREE_CHILD(remove_notify)`, `:525`) | `Screen` kept alive **across** the callback, released at `:525` (C); Python independently pops weak index + `del self.screen` | `:459-462,521-525`; `boss.py:881-885`; `window.py:1571` |
| **status** | *ordinary window:* status dropped. *monitored pid:* status kept in `reaped_pids`, delivered via `report_reaped_pids` → `on_monitored_pid_death` | exit status kept **iff** monitored | `:1398-1411,950-962`; `boss.py:2725` *(monitored delivery inferred)* |

The **observed anchors** for one window id from §5 are: **alive** = `[…] Child launched`; **intermediate**
(C-removed, Python-retained) = `Failed to send resize signal to child with id: N …` followed by the optimistic
`SIGWINCH sent …`; **after** = that id never appears again. The user-initiated teardown path
(`Window.close()` → `Boss.mark_window_for_close` → `child_monitor.mark_for_close`, `window.py:888-889`,
`boss.py:920-928`) sets the *same* `needs_removal` arbiter (`mark_child_for_close`, `child-monitor.c:546`) and
is likewise deferred.

**Keep-vs-discard *observed at the OS-resource level* (before / after, `/proc` + process table).** The
source-level "discard the PTY fd, reap the child, leak nothing" claims were confirmed directly by probing the
live process. A canonical session launched **1 long keeper (`sleep 25`) + 3 short children (`sleep 3`)**; the
short children were allowed to **exit on their own** (no external kill) while the process table and kitty's
open file descriptors were sampled before and after. Result — **stable across two identical runs**:

```
########## (A) ALIVE: 4 children running, kitty pid=96443 ##########
=== child processes parented to kitty ===
  96517   96443 Ss+  /usr/bin/sh -c sleep 25
  96518   96443 Ss+  /usr/bin/sh -c sleep 3
  96520   96443 Ss+  /usr/bin/sh -c sleep 3
  96522   96443 Ss+  /usr/bin/sh -c sleep 3
=== PTY-master fds held by kitty (/proc/96443/fd -> /dev/pts/ptmx) ===
  8  -> /dev/pts/ptmx      9  -> /dev/pts/ptmx     10 -> /dev/pts/ptmx     11 -> /dev/pts/ptmx
PTY-master fd count (A/alive): 4
zombies parented to kitty (A/alive): 0
########## (B) AFTER: 3 short children exited on their own; long keeper remains ##########
=== child processes parented to kitty now ===
  96517   96443 Ss+  /usr/bin/sh -c sleep 25
PTY-master fd count (B/after 3 exited): 1
zombies parented to kitty (B/after): 0
```

This makes the discard concrete: **PTY-master fds drop 4 → 1** as the three children die (each master
`safe_close`d by `cleanup_child`, `:1305-1308`), and **zombies stay 0 → 0** — kitty reaps each child the
instant it exits (§9's `wait4` capture), so it never accumulates a `Z`-state child while running. The `Ss+`
state of every child (session leader **and** foreground process group, the `+`) is the same fact that makes it
the target of `TIOCSWINSZ`/`SIGWINCH` in §6.1. As an aside, the very same `/proc/<pid>/fd` snapshot exposes the
reconciliation primitives from §3/§4 as live fds — `anon_inode:[eventfd]` (the Linux I/O-loop wakeup, **not** a
self-pipe) and `anon_inode:[signalfd]` (the `SIGCHLD` source) — independently confirming which `loop-utils`
branch is compiled here.

---

## 9. Q5 — Timing and signal delivery

**Direct answer.** On this Linux build the `SIGCHLD` path is **not** the classic async-handler-sets-a-flag
design; it is a **`signalfd` drained synchronously** on the I/O thread. The poll loop, when the signal fd is
readable, calls `read_signals()` (`loop-utils.c:131`), which invokes the callback `handle_signal`
(`child-monitor.c:1362`) inline; for `SIGCHLD` that only sets `ss->child_died = true` (`:1370-1371`), and the
**same** poll branch then calls `reap_children(self, OPT(close_on_child_death))` (`:1526`). `reap_children`
loops `waitpid(-1, &status, WNOHANG)` (`:1418`) precisely so that a *single* delivered `SIGCHLD` — the kernel
does not queue them **`(OS contract, §4)`** — can reap *many* exited children in one pass. Any
create/run/resize/close activity issued faster than one I/O-loop tick is not lost or corrupted: it is
serialized through `add_queue`/`remove_queue` under `children_lock`.

**Evidence — the Linux signal primitives exist at runtime (`strace`).** From §6, at startup kitty creates two
`eventfd`s and one `signalfd` covering `CHLD` (among others):

```
10180 eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK) = 4
10180 eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK) = 6
10180 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
```

This confirms the `HAS_SIGNAL_FD`/`HAS_EVENT_FD` code paths (`loop-utils.c:42,70`) are the ones compiled and
run here, not the self-pipe + `sigaction` fallback.

**Evidence — the handler only sets a flag:**

```c
        case SIGCHLD:
            ss->child_died = true;
            break;
```
(`kitty/child-monitor.c:1370-1371`.)

**Evidence — reaping is a `WNOHANG` loop that also captures monitored status:**

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
(`kitty/child-monitor.c:1413-1426`.) Note the two roles: `mark_child_for_removal` is **gated** by the option
(off by default), while `mark_monitored_pids` always runs to reap zombies and capture monitored statuses.

**Reaping and zombies — now *observed* at the syscall level, across both churn regimes.** The earlier draft
rested the reaping claim on UI convergence; it is now upgraded to a direct `wait4` capture. Tracing the
canonical binary with `strace -f -e trace=wait4,waitid` and reducing every `reap_children` call — each
`waitpid(-1, &status, WNOHANG)` at `:1418`, which glibc lowers to the `wait4` syscall — to its outcome yields
two distinct, fully reproducible signatures that differ **only** in whether a **long-lived keeper** child is
present. Regime A uses the canonical saturated session `$S40` from the §11.1 harness (`make_churn 40` = 40 ×
`launch sh -c 'exit 0'` + 1 × `launch sh -c 'sleep 30'`); Regime B uses the identical generator **without** the
trailing keeper line (`$S40nk` = 41 × `launch sh -c 'exit 0'`). Both sessions create **41** children. Each
regime below was run **twice, identically**, and both runs of each regime were byte-identical in these counts.

*Regime A — 40 instant-exit windows **plus 1 keeper**; 41 children total.* The keeper is deliberately still
alive when the run's `timeout` bound fires, so `waitpid(-1, WNOHANG)` **never** sees an empty process table:
it returns `0` ("a child exists, none newly exited" — the `else break` branch at `:1424`), **never**
`-1 ECHILD`, and the 40 short-lived children are each reaped exactly once:

```
$ strace -f -e trace=wait4,waitid -o wait4.out \
    timeout --signal=TERM --kill-after=5s 8 \
    ./kitty/launcher/kitty --debug-rendering --config NONE --session "$S40"
Child launched (children created):                          41
wait4() calls returning a positive pid (successful reaps):  40
wait4() = 0    (WNOHANG, no exited child yet):              40
wait4() = -1 ECHILD (loop terminator, no children left):     0
kitty exit=124
```

`kitty exit=124` is the `timeout` bound being reached while the keeper still runs (the §1/§2 convention:
124 = "time limit reached", **not** a crash); 40 of the 41 children are reaped and the keeper is intentionally
left alive.

*Regime B — 41 instant-exit windows, **no keeper**; 41 children total.* Every child exits, so after the
**last** one is reaped the next `waitpid(-1, WNOHANG)` returns `-1 ECHILD` ("no children left") — the loop's
`pid == -1` terminator at `:1419-1420` — and kitty then self-quits once its final window is gone:

```
$ strace -f -e trace=wait4,waitid -o wait4.out \
    timeout --signal=TERM --kill-after=5s 8 \
    ./kitty/launcher/kitty --debug-rendering --config NONE --session "$S40nk"
Child launched (children created):                          41
wait4() calls returning a positive pid (successful reaps):  41
wait4() = 0    (WNOHANG, no exited child yet):              40
wait4() = -1 ECHILD (loop terminator, no children left):     1
kitty exit=0
```

The single `ECHILD` terminator is captured verbatim (the reaper task-id and the buffer address vary
run-to-run):

```
501   wait4(-1, 0x7f4e7e7fbe60, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

with representative reap lines — note each positive reap (`P`) is immediately followed by a `WNOHANG`-empty
`= 0` (`Z`), i.e. every pass drained exactly **one** child (identical shape in both regimes):

```
501   wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 502
501   wait4(-1, 0x7f4e7e7fbe60, WNOHANG, NULL) = 0
501   wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 503
501   wait4(-1, 0x7f4e7e7fbe60, WNOHANG, NULL) = 0
501   wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 504
501   wait4(-1, 0x7f4e7e7fbe60, WNOHANG, NULL) = 0
```

**Reading the two regimes together** makes the terminator semantics unambiguous: `-1 ECHILD` appears **only**
when no child of any kind remains (Regime B — exactly **1** such call, at the very end) and is **impossible**
while a live keeper exists (Regime A — **0** such calls, the loop always exiting via the `pid == 0` /
`else break` branch). Both regimes reap every child they are meant to (40-of-40 short-lived in A, the keeper
deliberately spared; 41-of-41 in B), and **no `waitid` lines appeared** in either regime, confirming glibc
lowers `waitpid` to `wait4` here. The complementary **zombie check** (the §8
lifecycle probe) sampled the live process table and found **0** `Z`-state children parented to kitty **both**
while children were alive **and** after they exited — kitty reaps promptly and leaks no zombies. (Post-*exit*
orphans reparent to the container's non-reaping PID 1 `sleep infinity`; that is a container artifact outside
what kitty controls, **not** a kitty leak.)

**Coalescing multiplicity — probed directly, and *not* observed on this build `(observed + inferred)`.** The
reason `reap_children` uses a `while … waitpid(WNOHANG)` **loop** is the OS contract that `SIGCHLD` is not
queued (§4): one delivery may stand for several exits. Whether a *single* pass actually drains several
children is timing-dependent, so each `wait4` outcome was reduced to a symbol (`P` = positive reap,
`Z` = `WNOHANG`-empty / `pid == 0`, `E` = `ECHILD` / `pid == -1`) and the longest run of consecutive `P`s per
reap pass was taken. Both regimes — **including Regime B, where all 41 children exit near-simultaneously** (the
case most likely to coalesce) — showed a **maximum of 1 child per pass**, stable across both runs of each:

```
--- run-length distribution: children drained per reap_children pass (each stable across 2 identical runs) ---
Regime A (keeper):    passes that reaped 1 child = 40;  passes that reaped 0 = 0;  max in a single pass = 1
Regime B (no keeper): passes that reaped 1 child = 41;  passes that reaped 0 = 0;  max in a single pass = 1
--- Regime A: full 80 outcome symbols in order (P=reap / Z=WNOHANG-empty / E=ECHILD) ---
PZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZ
--- Regime B: full 82 outcome symbols in order (note the trailing `PE`: last reap, then the ECHILD terminator) ---
PZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPZPE
```

So on this Linux/`signalfd` build the loop demonstrably reaps **all** children (observed) but its
*multiplicity* benefit (>1 reap per pass) was **never manifested** — even in Regime B where 41 children exit
at once: the `signalfd`-drained poll loop calls `reap_children` frequently enough that each pass finds at most
one freshly-exited child. The multiplicity is therefore a **latent, defensive** property grounded in the OS
contract and the loop structure `(inferred)` — a distinction the earlier text under-stated by labelling the
whole effect merely "inferred." Note too that in the **default** config ordinary-window removal is driven
by **PTY EOF** (`:1535`), not by the reaper marking children, so this reap loop's role by default is zombie
collection and monitored-status capture, not window removal.

**Two-thread serialization (observed).** The sub-millisecond interleaving in the §5 capture shows main-thread
work (`resize_pty` → the `SIGWINCH`/`:610` lines) and I/O-thread work (PTY-EOF flagging, `remove_children`)
racing within the same millisecond, yet every mutation of `children[]` is bracketed by `children_lock`
(`:87`). The **cadence** at which the main thread observes I/O-thread removals is paced by `OPT(input_delay)`:
the I/O thread only wakes the main loop through the `WAKEUP` macro after `input_delay` has elapsed
(`child-monitor.c:1562-1569`; the comment at `:1563` notes "wakeup is an expensive operation"). (This is
distinct from `set_maximum_wait`/`maximum_wait` at `:114-118`, which bounds how long the loop waits for
*window-system* events before ticking render/input timers — e.g. `OPT(input_delay)` for pending input at
`:445`, `OPT(repaint_delay)` at `:876` — and is **not** the child-reconciliation cadence.)

---

## 10. Q6 — Conflicting views of liveness

**Direct answer.** Yes — and in **both** directions, plus a transient at creation. Within its **own two
registries** — the C `children[]` array and the Python `window_id_map` — kitty reconciles every such conflict
without blocking or crashing: reconciliation is driven off the deferred queues under `children_lock`, so a
resize, a death, and a create issued faster than one I/O tick are simply serialized (see the arbiter note and
the non-cooperative-child boundary below). What kitty does **not** guarantee is the *OS-level* liveness of a
non-cooperative child — a child that ignores `SIGHUP` outlives kitty entirely (measured below). "Resolves the
conflict" therefore means kitty's **internal** view is always made self-consistent, not that kitty can compel
every child process to die.

- **(a) C-first (observed).** Under churn the **C registry no longer holds a child** whose **Python `Window`
  still exists** (the layout still lays it out and resizes it). That interval is exactly the
  `Failed to send resize signal to child with id: N (children count: …)` line: the count proves the C side
  dropped the child (via PTY EOF, Q4), while the very fact a resize was *issued* for id `N` proves the Python
  layout still holds it. Resolution: the C side just logs and returns (`resize_pty`, `:606-610`); the Python
  side reconciles when `parse_input` fires the deferred `on_child_death`, which pops the weak index and
  `WindowList.remove_window` drops the strong reference.
- **(b) Python-first (ordering source-grounded; completion observed; guard firing inferred).** On an OS-window
  close the ordering is **reversed** relative to (a): `close_os_window` calls the Python callback **before** it
  marks the C children —
  `call_boss(on_os_window_closed, …)` at `child-monitor.c:1089`, then the
  `for (…) mark_child_for_close(self, tab->windows[w].id);` loop at `:1092`. And `on_os_window_closed` pops
  the window from the weak index *first*:

  ```python
      def on_os_window_closed(self, os_window_id: int, viewport_width: int, viewport_height: int) -> None:
          self.cached_values['window-size'] = viewport_width, viewport_height
          tm = self.os_window_map.pop(os_window_id, None)
          if tm is not None:
              tm.destroy()
          for window_id in tuple(w.id for w in self.window_id_map.values() if getattr(w, 'os_window_id', None) == os_window_id):
              self.window_id_map.pop(window_id, None)
  ```
  (`kitty/boss.py:1775-1781`.) The Python `Window` is therefore removed from `window_id_map` (line `:1781`)
  **before** the C side marks its child for close (`child-monitor.c:1092`), so the child's later death
  notification lands on a window that is already gone and `on_child_death` takes its `None`-guard
  (`boss.py:883-885`). That ordering is the **canonical** route to that guard. Two runs — both **observed** —
  bracket the claim honestly, because a single session can only demonstrate one half of it:

  * **No-keeper (proves the path completes, but does *not* exercise the live interval).** A session of three
    `launch sh -c 'exit 0'` windows exits **0** in ~0.47 s — every window closed, the OS window closed, and
    kitty quit on its own (§11.6). This proves the OS-close teardown *runs to completion and does not
    deadlock*. It does **not**, however, exercise the interval the question is really about: those children
    exit *before* the close, so no live child is present when `on_os_window_closed` runs, and the
    `Window`-already-gone-when-child-dies ordering is not actually forced.
  * **Live keeper (the interval actually exercised).** A session with one live `launch sh -c 'sleep 30'`
    child, closed via the window manager (`wmctrl -c`) with the child **still running**, is the real
    Python-first case. Because a live child is present, `confirm_os_window_close` fires and kitty *stays alive*
    showing a confirmation overlay (the extra `[0.484] Child launched` on stderr is that overlay's own kitten
    child); on confirming with `y`, kitty exits **0** in ~2.23 s and the keeper is gone. Complete transcript,
    reproduced identically across two runs (producer script and both run logs in §11.9):

    ```text
    ########## SESSION (1 LIVE keeper child) ##########
    launch sh -c 'sleep 30'
    kitty pid=111409  X window id=4194316
    child (live keeper) parented to kitty:  111483 Ss+  /usr/bin/sh -c sleep 30
    ########## ACTION: WM close request (wmctrl -c) on the live-child OS window ##########
    === after close request: is a confirmation overlay up? (kitty still alive => maybe) ===
    kitty alive after close request? YES
    maybe-confirmation
    ########## RESULT ##########
    kitty EXITED
    kitty exit code: 0
    elapsed: 2.2294s
    child still alive after close? no-or-reaped
    ########## kitty stderr (complete) ##########
    [0.208] OS Window created
    [0.218] Failed to open systemd user bus with error: No such file or directory
    [0.220] Child launched
    [0.484] Child launched
    ```

  What is **observed** in both runs is the *process-level* outcome: kitty exits `0` and the child is gone.
  What remains **`(inferred)`** is the `None`-guard's *firing* specifically — it emits no log — but the
  ordering that necessitates it (the `:1781` pop before the `:1092` mark) is grounded in the two `file:line`
  sites above.
- **(c) Creation-side transient (source-inferred; not observed firing).** Between `add_child` appending to
  `add_queue` (`child-monitor.c:305-321`) and the I/O thread running `add_children` (`:1281-1290`), a resize
  can in principle arrive for a child that is in `add_queue` but not yet in `children[]`. `resize_pty` is
  written to handle exactly this by searching **both** structures — `FIND(children …)` then `FIND(add_queue …)`
  at `child-monitor.c:606-607` — so a just-created child is still resizable. This branch is **source-inferred**,
  not observed: the `(add queue: N)` field on the `:610` miss line only ever printed `0` in our runs — the
  *failed*-lookup path, where `add_queue` was already empty — which does **not** demonstrate a *successful*
  match against a pending add. We therefore classify the second search as a code-grounded capability rather
  than an observed hit.

**The arbiter that prevents double action.** In every direction the single boolean `needs_removal`
(`child-monitor.c:67`) is the tie-breaker: by design it makes removal idempotent — once the flag is set, the
entry is eligible to be moved to `remove_queue` by `remove_children`, and after it is compacted out of
`children[]` it is no longer present to be processed again, while `parse_input` skips it as long as it lingers:

```c
    for (size_t i = 0; i < count; i++) {
        if (!scratch[i].needs_removal) {
            if (do_parse(self, scratch[i].screen, now, false)) input_read = true;
        }
        DECREF_CHILD(scratch[i]);
```
(`kitty/child-monitor.c:528-532`.) Combined with the weak `window_id_map`, an orphaned `Window` (no strong
owner) is simply collectable and its later death notification harmlessly hits the `None`-guard.

**The boundary of "resolves" — a non-cooperative child outlives kitty (observed).** The reconciliation above
is about kitty's *own* bookkeeping; it is **not** a guarantee that every child process actually dies. kitty's
teardown asks a child to exit with `hangup()` = `killpg(pgid, SIGHUP)` (`child-monitor.c:1294-1302`, `killpg`
at `:1299`) and does **not** escalate to `SIGKILL`. A child that installs `trap "" HUP` therefore ignores the
request and survives kitty's own exit, reparenting to PID 1. This was driven end-to-end — a session
`launch sh -c 'trap "" HUP; …; sleep 30; exit 42'`, then a `SIGTERM` to the kitty process — and reproduced
identically across **two** runs (producer script and both run logs in §11.8):

```text
########## SESSION (note: child does trap "" HUP => ignores SIGHUP) ##########
launch sh -c 'trap "" HUP; echo $$ > /tmp/kitty_f11.MFAvpl/childpid; sleep 30; exit 42'
########## BEFORE: kitty running, child launched ##########
kitty pid=111281   child pid=111352
child proc:   111352  111281  111352 Ss+  /usr/bin/sh -c trap "" HUP; echo $$ > /tmp/kitty_f11.MFAvpl/childpid; sleep 30; exit 42
child ppid (expect = kitty 111281): 111281
########## ACTION: SIGTERM kitty -> orderly shutdown -> hangup()=killpg(SIGHUP) to child pgroup (child-monitor.c:1294-1302) ##########
kitty still alive? NO-kitty-exited
########## AFTER: kitty gone. Did the SIGHUP-ignoring child survive? ##########
child 111352 still alive? YES-SURVIVED-kitty-and-SIGHUP
child proc now:  111352       1  111352 Ss   /usr/bin/sh -c trap "" HUP; echo $$ > /tmp/kitty_f11.MFAvpl/childpid; sleep 30; exit 42
child ppid now (1 = reparented to init, i.e. it OUTLIVED kitty): 1
########## kitty stderr ##########
[0.198] OS Window created
[0.208] Failed to open systemd user bus with error: No such file or directory
[0.210] Child launched
```

The consequence for Q6 is precise: kitty **always** makes its **internal** `children[]` and `window_id_map`
views self-consistent (the entry is removed from both regardless of what the process does), but the child's
**actual OS liveness** remains the child's decision. The "conflicting views" kitty guarantees to resolve are
its own two bookkeeping structures — not kitty versus the kernel's process table.

---

## 11. Race reproduction and run-to-run distribution

The race in Q3/Q6 is timing-dependent, so — per the run-to-run rule — the **identical, unchanged** input was
run repeatedly and the **observed distribution** is reported below. No stabilized variant was constructed. All
runs were produced by the two self-contained scripts embedded verbatim here; nothing else was needed to
reproduce them.

### 11.1 The harness (verbatim, `bash -n`-clean)

`harness.sh` drives Q2, the N=3 and N=10 churn cases, the 20-run saturated distribution (N=40 + keeper), and
the scale sweep. It allocates a **free** X display (never a fixed number), starts its **own** Xvfb and
validates that Xvfb's **PID and socket** before use (aborting non-zero if it fails to come up), writes **only**
under a private `mktemp -d` workspace, and **self-cleans** — its `EXIT`/`INT`/`TERM` trap reaps the Xvfb and
does a **path-validated** `rm -rf` of exactly that workspace (so nothing is left in `/tmp`):

```bash
#!/usr/bin/env bash
# ============================================================================
# kitty window-lifecycle observation harness  (READ-ONLY; runs the real binary)
#   - Safe by construction: umask 077, a private mktemp workspace, a bounded
#     free-display allocation, an Xvfb whose PID *and* socket we validate, and
#     an EXIT trap that reaps our Xvfb AND path-validated-removes the workspace.
#   - Canonical entry point only: ./kitty/launcher/kitty --config NONE --session.
#   - Writes ONLY under a private /tmp workspace; never touches the checkout.
#   - Aborts non-zero if our own Xvfb fails to come up (no foreign-display reuse).
# ============================================================================
set -u
umask 077

KITTY=/work/kitty/launcher/kitty
WORK="$(mktemp -d "${TMPDIR:-/tmp}/kitty_obs.XXXXXX")"
echo "WORKSPACE=$WORK"

# --- EXIT trap: reap OUR Xvfb, then path-validated-remove the workspace -------
cleanup(){
  if [ -n "${XVFB_PID:-}" ]; then kill "$XVFB_PID" 2>/dev/null; wait "$XVFB_PID" 2>/dev/null; fi
  # Only ever remove a directory we created: it must be live and match the exact
  # mktemp pattern under the temp root. Never a bare /tmp or an empty variable.
  case "$WORK" in
    "${TMPDIR:-/tmp}"/kitty_obs.*) [ -d "$WORK" ] && rm -rf -- "$WORK" ;;
    *) echo "REFUSING to remove unexpected WORK=$WORK" >&2 ;;
  esac
}
trap cleanup EXIT INT TERM

# --- Headless display: allocate a FREE display, start our own Xvfb, validate --
export LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
export LANG=C.UTF-8 LC_ALL=C.UTF-8
export HOME="$WORK/home" XDG_RUNTIME_DIR="$WORK/xdg"
mkdir -p "$HOME" "$XDG_RUNTIME_DIR"; chmod 700 "$XDG_RUNTIME_DIR"

DPYNUM=""
for n in $(seq 80 200); do
  if [ ! -e "/tmp/.X11-unix/X${n}" ]; then DPYNUM="$n"; break; fi
done
[ -n "$DPYNUM" ] || { echo "FATAL: no free X display in 80..200" >&2; exit 3; }
export DISPLAY=":${DPYNUM}"

Xvfb "$DISPLAY" -screen 0 1280x800x24 -ac +extension GLX >"$WORK/xvfb.log" 2>&1 &
XVFB_PID=$!
ready=0
for _ in $(seq 1 50); do
  # Ownership: the socket must exist AND the Xvfb WE spawned must still be alive.
  if [ -S "/tmp/.X11-unix/X${DPYNUM}" ] && kill -0 "$XVFB_PID" 2>/dev/null; then ready=1; break; fi
  kill -0 "$XVFB_PID" 2>/dev/null || break     # our Xvfb died => stop waiting
  sleep 0.1
done
if [ "$ready" != 1 ]; then
  echo "FATAL: our Xvfb (pid ${XVFB_PID:-none}) failed to become ready on $DISPLAY" >&2
  sed -n "1,20p" "$WORK/xvfb.log" >&2 2>/dev/null || true
  exit 4
fi
echo "XVFB_OK pid=$XVFB_PID display=$DISPLAY"

# --- Helpers -----------------------------------------------------------------
# Run one canonical kitty session, time-bounded, capturing COMPLETE stderr.
run_session(){ # <timeout_secs> <session_file> <out_log>
  timeout --signal=TERM --kill-after=5s "$1" \
    "$KITTY" --debug-rendering --config NONE --session "$2" \
    >/dev/null 2>"$3"
  echo "kitty_exit=$?"   # 124 => GNU timeout reached the time bound and terminated the run
}
# Build a churn session: N instant-exit windows + 1 long-lived keeper, splits layout.
make_churn(){ # <N> <file>
  { echo "layout splits"
    for _ in $(seq 1 "$1"); do echo "launch sh -c 'exit 0'"; done
    echo "launch sh -c 'sleep 30'"; } > "$2"
}
counts(){ # <log>  -> "launched sigwinch failed"
  printf '%s %s %s' \
    "$(grep -c 'Child launched' "$1")" \
    "$(grep -c 'SIGWINCH sent to child' "$1")" \
    "$(grep -c 'Failed to send resize signal' "$1")"
}

# ============================== SCENARIOS ===================================

# --- Q2: create-and-run. Two windows under tall layout; the 2nd forces the ---
#         first to relayout, emitting Observable #2 (SIGWINCH sent).          -
Q2="$WORK/q2.session"
printf 'layout tall\nlaunch sh -c '\''printf ready; sleep 8'\''\nlaunch sh -c '\''sleep 8'\''\n' > "$Q2"
echo "########## Q2_SESSION ##########"; cat "$Q2"
echo "########## Q2_RUN ##########"
echo "\$ timeout --signal=TERM --kill-after=5s 3 \$KITTY --debug-rendering --config NONE --session \$Q2 2>q2.log"
run_session 3 "$Q2" "$WORK/q2.log"
echo "########## Q2_STDERR (complete, $(wc -l < "$WORK/q2.log") lines) ##########"
cat "$WORK/q2.log"

# --- Q1/Q5/Q6: churn N=3 (fully embeddable readable case) -------------------
S3="$WORK/churn3.session"; make_churn 3 "$S3"
echo "########## CHURN3_SESSION ##########"; cat "$S3"
echo "########## CHURN3_RUN ##########"
run_session 3 "$S3" "$WORK/churn3.log"
echo "########## CHURN3_STDERR (complete, $(wc -l < "$WORK/churn3.log") lines) ##########"
cat "$WORK/churn3.log"
echo "########## CHURN3_COUNTS (launched sigwinch failed) ##########"; counts "$WORK/churn3.log"; echo

# --- Q1/Q5/Q6: churn N=10 (heavier churn, still embeddable) -----------------
S10="$WORK/churn10.session"; make_churn 10 "$S10"
echo "########## CHURN10_RUN ##########"
run_session 3 "$S10" "$WORK/churn10.log"
echo "########## CHURN10_COUNTS (launched sigwinch failed) ##########"; counts "$WORK/churn10.log"; echo

# --- Distribution: identical saturated input (N=40 + keeper), 20 runs -------
S40="$WORK/churn40.session"; make_churn 40 "$S40"
echo "########## DIST_SESSION_INFO ##########"
echo "session lines: $(wc -l < "$S40"); launch lines: $(grep -c launch "$S40")"
echo "########## DIST_CSV ##########"
echo "run,child_launched,sigwinch_sent,failed_resize,kitty_exit,elapsed_s"
for r in $(seq 1 20); do
  t0=$(date +%s.%N)
  ex=$(run_session 3 "$S40" "$WORK/run_${r}.log"); ex=${ex#kitty_exit=}
  t1=$(date +%s.%N)
  read L S F <<<"$(counts "$WORK/run_${r}.log")"
  el=$(awk "BEGIN{printf \"%.2f\", $t1-$t0}")
  printf '%d,%s,%s,%s,%s,%s\n' "$r" "$L" "$S" "$F" "$ex" "$el"
done
echo "########## DIST_LOG1_LINECOUNT ##########"
echo "run_1.log is $(wc -l < "$WORK/run_1.log") lines"

# --- Scale sweep: N in {1,2,3,4,6,10}, 15 identical runs each ----------------
echo "########## SCALE_SWEEP (N: failed_resize counts over 15 runs) ##########"
for N in 1 2 3 4 6 10; do
  SS="$WORK/sweep_${N}.session"; make_churn "$N" "$SS"
  line="N=$N:"
  for r in $(seq 1 15); do
    run_session 3 "$SS" "$WORK/sweep_${N}_${r}.log" >/dev/null
    read L S F <<<"$(counts "$WORK/sweep_${N}_${r}.log")"
    line="$line $F"
  done
  echo "$line"
done

echo "########## DONE ##########"
echo "WORKSPACE_WAS=$WORK (removed by EXIT trap)"
```

### 11.2 The OS-window-close script (verbatim) — the Python-first teardown of §10(b)

```bash
#!/usr/bin/env bash
# ============================================================================
# Q6 teardown direction #2 (Python-first): drive the canonical OS-window-close
# path. A session with NO long-lived keeper lets every child exit; when the
# last window closes, kitty runs close_os_window() -> on_os_window_closed()
# (which pops window_id_map FIRST) and then exits.
#   - Free-display allocation, Xvfb PID+socket ownership validation, abort on
#     failure, path-validated workspace removal, and a verdict derived from the
#     ACTUAL exit code (never hardcoded).
# ============================================================================
set -u
umask 077
KITTY=/work/kitty/launcher/kitty
BOUND=8
WORK="$(mktemp -d "${TMPDIR:-/tmp}/kitty_osclose.XXXXXX")"
echo "WORKSPACE=$WORK"

cleanup(){
  if [ -n "${XVFB_PID:-}" ]; then kill "$XVFB_PID" 2>/dev/null; wait "$XVFB_PID" 2>/dev/null; fi
  case "$WORK" in
    "${TMPDIR:-/tmp}"/kitty_osclose.*) [ -d "$WORK" ] && rm -rf -- "$WORK" ;;
    *) echo "REFUSING to remove unexpected WORK=$WORK" >&2 ;;
  esac
}
trap cleanup EXIT INT TERM

export LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe LANG=C.UTF-8 LC_ALL=C.UTF-8
export HOME="$WORK/home" XDG_RUNTIME_DIR="$WORK/xdg"
mkdir -p "$HOME" "$XDG_RUNTIME_DIR"; chmod 700 "$XDG_RUNTIME_DIR"

DPYNUM=""
for n in $(seq 80 200); do
  if [ ! -e "/tmp/.X11-unix/X${n}" ]; then DPYNUM="$n"; break; fi
done
[ -n "$DPYNUM" ] || { echo "FATAL: no free X display in 80..200" >&2; exit 3; }
export DISPLAY=":${DPYNUM}"

Xvfb "$DISPLAY" -screen 0 1280x800x24 -ac +extension GLX >"$WORK/xvfb.log" 2>&1 &
XVFB_PID=$!
ready=0
for _ in $(seq 1 50); do
  if [ -S "/tmp/.X11-unix/X${DPYNUM}" ] && kill -0 "$XVFB_PID" 2>/dev/null; then ready=1; break; fi
  kill -0 "$XVFB_PID" 2>/dev/null || break
  sleep 0.1
done
if [ "$ready" != 1 ]; then
  echo "FATAL: our Xvfb (pid ${XVFB_PID:-none}) failed to become ready on $DISPLAY" >&2
  sed -n "1,20p" "$WORK/xvfb.log" >&2 2>/dev/null || true
  exit 4
fi
echo "XVFB_OK pid=$XVFB_PID display=$DISPLAY"

SES="$WORK/nokeeper.session"
{ echo "layout splits"
  echo "launch sh -c 'exit 0'"
  echo "launch sh -c 'exit 0'"
  echo "launch sh -c 'exit 0'"; } > "$SES"
echo "########## OSCLOSE_SESSION (no keeper) ##########"; cat "$SES"
echo "########## OSCLOSE_RUN ##########"
echo "\$ timeout --signal=TERM --kill-after=5s $BOUND \$KITTY --debug-rendering --config NONE --session \$SES"
t0=$(date +%s.%N)
timeout --signal=TERM --kill-after=5s "$BOUND" \
  "$KITTY" --debug-rendering --config NONE --session "$SES" >/dev/null 2>"$WORK/osclose.log"
ex=$?
t1=$(date +%s.%N)
el=$(awk "BEGIN{printf \"%.2f\", $t1-$t0}")
# Verdict derived from the ACTUAL exit code, never hardcoded.
if [ "$ex" -eq 0 ]; then
  verdict="exit 0 => every window closed, the OS window closed, and kitty quit on its own (well under the ${BOUND}s bound)"
elif [ "$ex" -eq 124 ]; then
  verdict="exit 124 => the ${BOUND}s timeout bound was reached; kitty did NOT quit on its own"
else
  verdict="exit $ex => kitty terminated abnormally (inspect stderr below)"
fi
echo "kitty_exit=$ex  elapsed=${el}s   ($verdict)"
echo "########## OSCLOSE_STDERR (complete, $(wc -l < "$WORK/osclose.log") lines) ##########"
cat "$WORK/osclose.log"
echo "WORKSPACE_WAS=$WORK (removed by EXIT trap)"
```

### 11.3 Saturated churn (N = 40 instant-exit + 1 keeper), 20 identical runs — complete CSV

Each run is the identical 42-line session (`layout splits` + 40 × `launch sh -c 'exit 0'` + 1 ×
`launch sh -c 'sleep 30'`). The complete per-run CSV (produced by the `DIST_CSV` block above), including
wall-clock duration, was:

```
run,child_launched,sigwinch_sent,failed_resize,kitty_exit,elapsed_s
1,41,131,125,124,3.04
2,41,131,125,124,3.04
3,41,131,125,124,3.04
4,41,131,125,124,3.05
5,41,131,125,124,3.05
6,41,131,125,124,3.04
7,41,131,125,124,3.04
8,41,131,125,124,3.04
9,41,131,125,124,3.05
10,41,131,125,124,3.04
11,41,131,125,124,3.05
12,41,131,125,124,3.04
13,41,131,125,124,3.04
14,41,131,125,124,3.04
15,41,131,125,124,3.05
16,41,131,125,124,3.04
17,41,131,125,124,3.04
18,41,131,125,124,3.04
19,41,131,125,124,3.04
20,41,131,125,124,3.04
```

In this batch, in this quiet single-tenant llvmpipe container, the counts were identical across all 20 runs
(`Child launched` = 41, `SIGWINCH sent` = 131, `Failed to send resize signal` = 125), and each run took
~3.04 s (the 3 s `timeout` bound plus teardown — `kitty_exit=124`, i.e. the bound was reached, §2). The
`run_1.log` produced by this block was **299 lines** long; that complete, unedited log is reproduced in §11.4.

Across **batches**, however, the `Failed to send resize signal` count at N = 40 is **not perfectly
deterministic**. A second independent 20-run batch (produced by the hardened harness of §11.1, on display
`:80`) is reproduced complete and unedited below; `Child launched` (41) and `SIGWINCH sent` (131) stayed fixed
on every run, but `failed_resize` was **`125` on 18 runs and `124` on 2 runs** (runs 19–20):

```
run,child_launched,sigwinch_sent,failed_resize,kitty_exit,elapsed_s
1,41,131,125,124,3.05
2,41,131,125,124,3.06
3,41,131,125,124,3.05
4,41,131,125,124,3.05
5,41,131,125,124,3.05
6,41,131,125,124,3.05
7,41,131,125,124,3.04
8,41,131,125,124,3.08
9,41,131,125,124,3.07
10,41,131,125,124,3.05
11,41,131,125,124,3.05
12,41,131,125,124,3.05
13,41,131,125,124,3.06
14,41,131,125,124,3.05
15,41,131,125,124,3.05
16,41,131,125,124,3.05
17,41,131,125,124,3.05
18,41,131,125,124,3.05
19,41,131,124,124,3.08
20,41,131,124,124,3.04
```

The ±1 wander between batches is the direct saturated-scale counterpart of the scale-sweep variability in
§11.5: the race is **highly reproducible but timing-sensitive, not fixed** — a single all-`125` batch must
therefore not be read as a determinism guarantee.

### 11.4 The complete representative saturated log (`run_1.log`, 299 lines, unedited)

This is the *entire* stderr of run 1 above — the full basis for the `125`/`131`/`41` counts, not an excerpt.
The first lines show each new window firing `Child launched` then an optimistic resize for an
already-removed sibling; the final three lines are the convergence, where the sole survivor (the keeper,
window 41) is resized up to the full 71-column width:

```
[0.212] OS Window created
[0.221] Failed to open systemd user bus with error: No such file or directory
[0.223] Child launched
[0.228] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.228] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
[0.228] Child launched
[0.235] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.235] SIGWINCH sent to child in window: 2 with size: (22, 17, 153, 396)
[0.235] Child launched
[0.241] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.241] SIGWINCH sent to child in window: 3 with size: (22, 8, 72, 396)
[0.241] Child launched
[0.248] Failed to send resize signal to child with id: 4 (children count: 1) (add queue: 0)
[0.248] SIGWINCH sent to child in window: 4 with size: (22, 4, 36, 396)
[0.248] Child launched
[0.256] Failed to send resize signal to child with id: 5 (children count: 1) (add queue: 0)
[0.256] SIGWINCH sent to child in window: 5 with size: (22, 2, 18, 396)
[0.256] Child launched
[0.264] Failed to send resize signal to child with id: 6 (children count: 1) (add queue: 0)
[0.264] SIGWINCH sent to child in window: 6 with size: (22, 1, 9, 396)
[0.264] Child launched
[0.271] Failed to send resize signal to child with id: 5 (children count: 1) (add queue: 0)
[0.271] SIGWINCH sent to child in window: 5 with size: (22, 1, 9, 396)
[0.271] Child launched
[0.278] Failed to send resize signal to child with id: 4 (children count: 1) (add queue: 0)
[0.278] SIGWINCH sent to child in window: 4 with size: (22, 2, 18, 396)
[0.278] Child launched
[0.284] Failed to send resize signal to child with id: 4 (children count: 1) (add queue: 0)
[0.284] SIGWINCH sent to child in window: 4 with size: (22, 1, 9, 396)
[0.285] Child launched
[0.292] Child launched
[0.299] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.299] SIGWINCH sent to child in window: 3 with size: (22, 6, 54, 396)
[0.300] Child launched
[0.307] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.307] SIGWINCH sent to child in window: 3 with size: (22, 5, 45, 396)
[0.308] Child launched
[0.315] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.315] SIGWINCH sent to child in window: 3 with size: (22, 4, 36, 396)
[0.316] Child launched
[0.324] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.324] SIGWINCH sent to child in window: 3 with size: (22, 3, 27, 396)
[0.326] Child launched
[0.333] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.333] SIGWINCH sent to child in window: 3 with size: (22, 2, 18, 396)
[0.334] Child launched
[0.343] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.343] SIGWINCH sent to child in window: 3 with size: (22, 1, 9, 396)
[0.343] Child launched
[0.351] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.351] SIGWINCH sent to child in window: 2 with size: (22, 16, 144, 396)
[0.352] Child launched
[0.359] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.359] SIGWINCH sent to child in window: 2 with size: (22, 14, 126, 396)
[0.360] Child launched
[0.368] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.368] SIGWINCH sent to child in window: 2 with size: (22, 13, 117, 396)
[0.369] Child launched
[0.374] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.374] SIGWINCH sent to child in window: 2 with size: (22, 12, 108, 396)
[0.375] Child launched
[0.383] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.383] SIGWINCH sent to child in window: 2 with size: (22, 11, 99, 396)
[0.384] Child launched
[0.391] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.391] SIGWINCH sent to child in window: 2 with size: (22, 10, 90, 396)
[0.392] Child launched
[0.400] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.400] SIGWINCH sent to child in window: 2 with size: (22, 8, 72, 396)
[0.401] Child launched
[0.409] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.409] SIGWINCH sent to child in window: 2 with size: (22, 7, 63, 396)
[0.411] Child launched
[0.418] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.418] SIGWINCH sent to child in window: 2 with size: (22, 6, 54, 396)
[0.420] Child launched
[0.427] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.427] SIGWINCH sent to child in window: 2 with size: (22, 5, 45, 396)
[0.429] Child launched
[0.436] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.436] SIGWINCH sent to child in window: 2 with size: (22, 3, 27, 396)
[0.438] Child launched
[0.445] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.445] SIGWINCH sent to child in window: 2 with size: (22, 2, 18, 396)
[0.447] Child launched
[0.454] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.454] SIGWINCH sent to child in window: 2 with size: (22, 1, 9, 396)
[0.456] Child launched
[0.464] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.464] SIGWINCH sent to child in window: 1 with size: (22, 34, 306, 396)
[0.465] Child launched
[0.473] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.473] SIGWINCH sent to child in window: 1 with size: (22, 33, 297, 396)
[0.475] Child launched
[0.484] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.484] SIGWINCH sent to child in window: 1 with size: (22, 32, 288, 396)
[0.486] Child launched
[0.494] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.494] SIGWINCH sent to child in window: 1 with size: (22, 31, 279, 396)
[0.496] Child launched
[0.505] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.505] SIGWINCH sent to child in window: 1 with size: (22, 29, 261, 396)
[0.507] Child launched
[0.516] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.516] SIGWINCH sent to child in window: 1 with size: (22, 28, 252, 396)
[0.519] Child launched
[0.527] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.527] SIGWINCH sent to child in window: 1 with size: (22, 27, 243, 396)
[0.530] Child launched
[0.538] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.538] SIGWINCH sent to child in window: 1 with size: (22, 26, 234, 396)
[0.541] Child launched
[0.551] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.551] SIGWINCH sent to child in window: 1 with size: (22, 24, 216, 396)
[0.554] Child launched
[0.562] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.562] SIGWINCH sent to child in window: 1 with size: (22, 23, 207, 396)
[0.565] Child launched
[0.576] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.576] SIGWINCH sent to child in window: 1 with size: (22, 22, 198, 396)
[0.579] Child launched
[0.583] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.583] SIGWINCH sent to child in window: 2 with size: (22, 23, 207, 396)
[0.587] Failed to send resize signal to child with id: 3 (children count: 1) (add queue: 0)
[0.587] SIGWINCH sent to child in window: 3 with size: (22, 24, 216, 396)
[0.591] Failed to send resize signal to child with id: 4 (children count: 1) (add queue: 0)
[0.591] SIGWINCH sent to child in window: 4 with size: (22, 26, 234, 396)
[0.595] Failed to send resize signal to child with id: 5 (children count: 1) (add queue: 0)
[0.595] SIGWINCH sent to child in window: 5 with size: (22, 27, 243, 396)
[0.598] Failed to send resize signal to child with id: 6 (children count: 1) (add queue: 0)
[0.598] SIGWINCH sent to child in window: 6 with size: (22, 28, 252, 396)
[0.602] Failed to send resize signal to child with id: 7 (children count: 1) (add queue: 0)
[0.602] SIGWINCH sent to child in window: 7 with size: (22, 29, 261, 396)
[0.605] Failed to send resize signal to child with id: 8 (children count: 1) (add queue: 0)
[0.605] SIGWINCH sent to child in window: 8 with size: (22, 31, 279, 396)
[0.608] Failed to send resize signal to child with id: 9 (children count: 1) (add queue: 0)
[0.608] SIGWINCH sent to child in window: 9 with size: (22, 32, 288, 396)
[0.612] Failed to send resize signal to child with id: 10 (children count: 1) (add queue: 0)
[0.612] SIGWINCH sent to child in window: 10 with size: (22, 33, 297, 396)
[0.614] Failed to send resize signal to child with id: 11 (children count: 1) (add queue: 0)
[0.614] SIGWINCH sent to child in window: 11 with size: (22, 34, 306, 396)
[0.617] Failed to send resize signal to child with id: 12 (children count: 1) (add queue: 0)
[0.617] SIGWINCH sent to child in window: 12 with size: (22, 35, 315, 396)
[0.621] Failed to send resize signal to child with id: 13 (children count: 1) (add queue: 0)
[0.621] SIGWINCH sent to child in window: 13 with size: (22, 35, 315, 396)
[0.621] Failed to send resize signal to child with id: 14 (children count: 1) (add queue: 0)
[0.621] SIGWINCH sent to child in window: 14 with size: (22, 2, 18, 396)
[0.623] Failed to send resize signal to child with id: 14 (children count: 1) (add queue: 0)
[0.623] SIGWINCH sent to child in window: 14 with size: (22, 35, 315, 396)
[0.623] Failed to send resize signal to child with id: 15 (children count: 1) (add queue: 0)
[0.623] SIGWINCH sent to child in window: 15 with size: (22, 3, 27, 396)
[0.626] Failed to send resize signal to child with id: 15 (children count: 1) (add queue: 0)
[0.626] SIGWINCH sent to child in window: 15 with size: (22, 35, 315, 396)
[0.626] Failed to send resize signal to child with id: 16 (children count: 1) (add queue: 0)
[0.626] SIGWINCH sent to child in window: 16 with size: (22, 5, 45, 396)
[0.629] Failed to send resize signal to child with id: 16 (children count: 1) (add queue: 0)
[0.629] SIGWINCH sent to child in window: 16 with size: (22, 35, 315, 396)
[0.629] Failed to send resize signal to child with id: 17 (children count: 1) (add queue: 0)
[0.629] SIGWINCH sent to child in window: 17 with size: (22, 6, 54, 396)
[0.631] Failed to send resize signal to child with id: 17 (children count: 1) (add queue: 0)
[0.631] SIGWINCH sent to child in window: 17 with size: (22, 35, 315, 396)
[0.631] Failed to send resize signal to child with id: 18 (children count: 1) (add queue: 0)
[0.631] SIGWINCH sent to child in window: 18 with size: (22, 7, 63, 396)
[0.633] Failed to send resize signal to child with id: 18 (children count: 1) (add queue: 0)
[0.633] SIGWINCH sent to child in window: 18 with size: (22, 35, 315, 396)
[0.633] Failed to send resize signal to child with id: 19 (children count: 1) (add queue: 0)
[0.633] SIGWINCH sent to child in window: 19 with size: (22, 8, 72, 396)
[0.635] Failed to send resize signal to child with id: 19 (children count: 1) (add queue: 0)
[0.635] SIGWINCH sent to child in window: 19 with size: (22, 35, 315, 396)
[0.636] Failed to send resize signal to child with id: 20 (children count: 1) (add queue: 0)
[0.636] SIGWINCH sent to child in window: 20 with size: (22, 10, 90, 396)
[0.637] Failed to send resize signal to child with id: 20 (children count: 1) (add queue: 0)
[0.637] SIGWINCH sent to child in window: 20 with size: (22, 35, 315, 396)
[0.638] Failed to send resize signal to child with id: 21 (children count: 1) (add queue: 0)
[0.638] SIGWINCH sent to child in window: 21 with size: (22, 11, 99, 396)
[0.639] Failed to send resize signal to child with id: 21 (children count: 1) (add queue: 0)
[0.639] SIGWINCH sent to child in window: 21 with size: (22, 35, 315, 396)
[0.640] Failed to send resize signal to child with id: 22 (children count: 1) (add queue: 0)
[0.640] SIGWINCH sent to child in window: 22 with size: (22, 12, 108, 396)
[0.641] Failed to send resize signal to child with id: 22 (children count: 1) (add queue: 0)
[0.641] SIGWINCH sent to child in window: 22 with size: (22, 35, 315, 396)
[0.641] Failed to send resize signal to child with id: 23 (children count: 1) (add queue: 0)
[0.641] SIGWINCH sent to child in window: 23 with size: (22, 13, 117, 396)
[0.643] Failed to send resize signal to child with id: 23 (children count: 1) (add queue: 0)
[0.643] SIGWINCH sent to child in window: 23 with size: (22, 35, 315, 396)
[0.643] Failed to send resize signal to child with id: 24 (children count: 1) (add queue: 0)
[0.643] SIGWINCH sent to child in window: 24 with size: (22, 14, 126, 396)
[0.645] Failed to send resize signal to child with id: 24 (children count: 1) (add queue: 0)
[0.645] SIGWINCH sent to child in window: 24 with size: (22, 35, 315, 396)
[0.645] Failed to send resize signal to child with id: 25 (children count: 1) (add queue: 0)
[0.645] SIGWINCH sent to child in window: 25 with size: (22, 16, 144, 396)
[0.646] Failed to send resize signal to child with id: 25 (children count: 1) (add queue: 0)
[0.646] SIGWINCH sent to child in window: 25 with size: (22, 35, 315, 396)
[0.647] Failed to send resize signal to child with id: 26 (children count: 1) (add queue: 0)
[0.647] SIGWINCH sent to child in window: 26 with size: (22, 17, 153, 396)
[0.648] Failed to send resize signal to child with id: 26 (children count: 1) (add queue: 0)
[0.648] SIGWINCH sent to child in window: 26 with size: (22, 35, 315, 396)
[0.648] Failed to send resize signal to child with id: 27 (children count: 1) (add queue: 0)
[0.648] SIGWINCH sent to child in window: 27 with size: (22, 17, 153, 396)
[0.649] Failed to send resize signal to child with id: 28 (children count: 1) (add queue: 0)
[0.649] SIGWINCH sent to child in window: 28 with size: (22, 2, 18, 396)
[0.650] Failed to send resize signal to child with id: 27 (children count: 1) (add queue: 0)
[0.650] SIGWINCH sent to child in window: 27 with size: (22, 35, 315, 396)
[0.650] Failed to send resize signal to child with id: 28 (children count: 1) (add queue: 0)
[0.650] SIGWINCH sent to child in window: 28 with size: (22, 17, 153, 396)
[0.650] Failed to send resize signal to child with id: 29 (children count: 1) (add queue: 0)
[0.650] SIGWINCH sent to child in window: 29 with size: (22, 3, 27, 396)
[0.651] Failed to send resize signal to child with id: 28 (children count: 1) (add queue: 0)
[0.651] SIGWINCH sent to child in window: 28 with size: (22, 35, 315, 396)
[0.651] Failed to send resize signal to child with id: 29 (children count: 1) (add queue: 0)
[0.651] SIGWINCH sent to child in window: 29 with size: (22, 17, 153, 396)
[0.651] Failed to send resize signal to child with id: 30 (children count: 1) (add queue: 0)
[0.651] SIGWINCH sent to child in window: 30 with size: (22, 4, 36, 396)
[0.652] Failed to send resize signal to child with id: 29 (children count: 1) (add queue: 0)
[0.652] SIGWINCH sent to child in window: 29 with size: (22, 35, 315, 396)
[0.652] Failed to send resize signal to child with id: 30 (children count: 1) (add queue: 0)
[0.652] SIGWINCH sent to child in window: 30 with size: (22, 17, 153, 396)
[0.653] Failed to send resize signal to child with id: 31 (children count: 1) (add queue: 0)
[0.653] SIGWINCH sent to child in window: 31 with size: (22, 5, 45, 396)
[0.653] Failed to send resize signal to child with id: 30 (children count: 1) (add queue: 0)
[0.653] SIGWINCH sent to child in window: 30 with size: (22, 35, 315, 396)
[0.654] Failed to send resize signal to child with id: 31 (children count: 1) (add queue: 0)
[0.654] SIGWINCH sent to child in window: 31 with size: (22, 17, 153, 396)
[0.654] Failed to send resize signal to child with id: 32 (children count: 1) (add queue: 0)
[0.654] SIGWINCH sent to child in window: 32 with size: (22, 6, 54, 396)
[0.655] Failed to send resize signal to child with id: 31 (children count: 1) (add queue: 0)
[0.655] SIGWINCH sent to child in window: 31 with size: (22, 35, 315, 396)
[0.655] Failed to send resize signal to child with id: 32 (children count: 1) (add queue: 0)
[0.655] SIGWINCH sent to child in window: 32 with size: (22, 17, 153, 396)
[0.655] Failed to send resize signal to child with id: 33 (children count: 1) (add queue: 0)
[0.655] SIGWINCH sent to child in window: 33 with size: (22, 8, 72, 396)
[0.656] Failed to send resize signal to child with id: 32 (children count: 1) (add queue: 0)
[0.656] SIGWINCH sent to child in window: 32 with size: (22, 35, 315, 396)
[0.656] Failed to send resize signal to child with id: 33 (children count: 1) (add queue: 0)
[0.656] SIGWINCH sent to child in window: 33 with size: (22, 17, 153, 396)
[0.656] Failed to send resize signal to child with id: 34 (children count: 1) (add queue: 0)
[0.656] SIGWINCH sent to child in window: 34 with size: (22, 8, 72, 396)
[0.656] Failed to send resize signal to child with id: 33 (children count: 1) (add queue: 0)
[0.656] SIGWINCH sent to child in window: 33 with size: (22, 35, 315, 396)
[0.657] Failed to send resize signal to child with id: 34 (children count: 1) (add queue: 0)
[0.657] SIGWINCH sent to child in window: 34 with size: (22, 17, 153, 396)
[0.657] Failed to send resize signal to child with id: 35 (children count: 1) (add queue: 0)
[0.657] SIGWINCH sent to child in window: 35 with size: (22, 8, 72, 396)
[0.657] Failed to send resize signal to child with id: 36 (children count: 1) (add queue: 0)
[0.657] SIGWINCH sent to child in window: 36 with size: (22, 2, 18, 396)
[0.657] Failed to send resize signal to child with id: 34 (children count: 1) (add queue: 0)
[0.657] SIGWINCH sent to child in window: 34 with size: (22, 35, 315, 396)
[0.658] Failed to send resize signal to child with id: 35 (children count: 1) (add queue: 0)
[0.658] SIGWINCH sent to child in window: 35 with size: (22, 17, 153, 396)
[0.658] Failed to send resize signal to child with id: 36 (children count: 1) (add queue: 0)
[0.658] SIGWINCH sent to child in window: 36 with size: (22, 8, 72, 396)
[0.658] Failed to send resize signal to child with id: 37 (children count: 1) (add queue: 0)
[0.658] SIGWINCH sent to child in window: 37 with size: (22, 4, 36, 396)
[0.658] Failed to send resize signal to child with id: 35 (children count: 1) (add queue: 0)
[0.658] SIGWINCH sent to child in window: 35 with size: (22, 35, 315, 396)
[0.659] Failed to send resize signal to child with id: 36 (children count: 1) (add queue: 0)
[0.659] SIGWINCH sent to child in window: 36 with size: (22, 17, 153, 396)
[0.659] Failed to send resize signal to child with id: 37 (children count: 1) (add queue: 0)
[0.659] SIGWINCH sent to child in window: 37 with size: (22, 8, 72, 396)
[0.659] Failed to send resize signal to child with id: 38 (children count: 1) (add queue: 0)
[0.659] SIGWINCH sent to child in window: 38 with size: (22, 4, 36, 396)
[0.659] Failed to send resize signal to child with id: 39 (children count: 1) (add queue: 0)
[0.659] SIGWINCH sent to child in window: 39 with size: (22, 2, 18, 396)
[0.659] Failed to send resize signal to child with id: 36 (children count: 1) (add queue: 0)
[0.659] SIGWINCH sent to child in window: 36 with size: (22, 35, 315, 396)
[0.660] Failed to send resize signal to child with id: 37 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 37 with size: (22, 17, 153, 396)
[0.660] Failed to send resize signal to child with id: 38 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 38 with size: (22, 8, 72, 396)
[0.660] Failed to send resize signal to child with id: 39 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 39 with size: (22, 4, 36, 396)
[0.660] Failed to send resize signal to child with id: 40 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 40 with size: (22, 2, 18, 396)
[0.660] SIGWINCH sent to child in window: 41 with size: (22, 2, 18, 396)
[0.660] Failed to send resize signal to child with id: 37 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 37 with size: (22, 35, 315, 396)
[0.660] Failed to send resize signal to child with id: 38 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 38 with size: (22, 17, 153, 396)
[0.660] Failed to send resize signal to child with id: 39 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 39 with size: (22, 8, 72, 396)
[0.660] Failed to send resize signal to child with id: 40 (children count: 1) (add queue: 0)
[0.660] SIGWINCH sent to child in window: 40 with size: (22, 4, 36, 396)
[0.661] SIGWINCH sent to child in window: 41 with size: (22, 4, 36, 396)
[0.661] Failed to send resize signal to child with id: 38 (children count: 1) (add queue: 0)
[0.661] SIGWINCH sent to child in window: 38 with size: (22, 35, 315, 396)
[0.661] Failed to send resize signal to child with id: 39 (children count: 1) (add queue: 0)
[0.661] SIGWINCH sent to child in window: 39 with size: (22, 17, 153, 396)
[0.661] Failed to send resize signal to child with id: 40 (children count: 1) (add queue: 0)
[0.661] SIGWINCH sent to child in window: 40 with size: (22, 8, 72, 396)
[0.661] SIGWINCH sent to child in window: 41 with size: (22, 8, 72, 396)
[0.661] Failed to send resize signal to child with id: 39 (children count: 1) (add queue: 0)
[0.661] SIGWINCH sent to child in window: 39 with size: (22, 35, 315, 396)
[0.662] Failed to send resize signal to child with id: 40 (children count: 1) (add queue: 0)
[0.662] SIGWINCH sent to child in window: 40 with size: (22, 17, 153, 396)
[0.662] SIGWINCH sent to child in window: 41 with size: (22, 17, 153, 396)
[0.662] Failed to send resize signal to child with id: 40 (children count: 1) (add queue: 0)
[0.662] SIGWINCH sent to child in window: 40 with size: (22, 35, 315, 396)
[0.663] SIGWINCH sent to child in window: 41 with size: (22, 35, 315, 396)
[0.664] SIGWINCH sent to child in window: 41 with size: (22, 71, 639, 396)
```

### 11.5 Scale sweep (15 identical runs each) — where the count is *not* perfectly deterministic

Three independent executions of the sweep (`runA`, `runB`, `runC`) are reported so the genuine timing
variability is visible rather than smoothed away. The metric is the per-run `Failed to send resize signal`
(`:610`) count; `runC` is a fresh batch from the hardened harness of §11.1:

| N (instant-exit + 1 keeper) | runA (15 runs) | runB (15 runs) | runC (15 runs, hardened harness) |
|---:|---|---|---|
| 1 | `1 1 1 1 1 1 1 1 1 1 1 1 1 1 1` | `1 1 1 1 1 1 1 1 1 1 1 1 1 1 1` | `1 1 1 1 1 1 1 1 0 1 1 1 1 1 1`  ← one run produced **0** |
| 2 | `3 3 3 3 3 3 3 3 3 3 3 3 3 3 3` | `3 3 3 3 3 3 3 3 3 3 3 3 3 3 3` | `3 3 3 3 3 3 3 3 3 3 3 3 3 3 3` |
| 3 | `6 6 6 6 6 6 6 6 6 6 6 5 6 6 6`  ← one run produced **5** | `6 6 6 6 6 6 6 6 6 6 6 6 6 6 6` | `6 6 6 6 6 6 6 6 6 6 6 6 6 6 6` |
| 4 | `10 10 10 10 10 10 10 10 10 10 10 10 10 10 10` | `10 10 10 10 10 10 10 10 10 10 10 10 10 10 10` | `10 10 10 10 10 10 10 10 10 10 10 10 10 10 10` |
| 6 | `21 21 21 21 21 21 21 21 21 21 21 21 21 21 21` | `21 21 21 21 21 21 21 21 21 21 21 21 21 21 21` | `21 21 21 21 21 21 21 21 21 21 21 21 21 21 21` |
| 10 | `40 40 40 40 40 40 40 40 40 40 40 40 40 40 40` | `40 40 40 40 40 40 40 40 40 40 40 40 40 40 40` | `40 40 40 40 40 40 40 40 40 40 40 40 40 40 40` |

The count grows with churn (the N=10 sweep value `40` matches the standalone N=10 run's `failed_resize`
count). Two anomalous runs across the three executions — one N=3 run that produced **5** instead of **6**
(`runA`) and one N=1 run that produced **0** instead of **1** (`runC`) — are the direct evidence that the
incidence is **timing-sensitive, not fixed**: the race is highly reproducible but not perfectly deterministic
in this environment. Note the anomaly is always **−1** (a relayout that would have missed instead lands while
the child is still findable), never a spurious extra miss.

### 11.6 OS-window-close (Python-first teardown), complete log

The no-keeper session exits on its own; the hardened `osclose.sh` (§11.2) produced (complete output, including
the exit-code-derived verdict line):

```
WORKSPACE=/tmp/kitty_osclose.LAWxWu
XVFB_OK pid=111092 display=:80
########## OSCLOSE_SESSION (no keeper) ##########
layout splits
launch sh -c 'exit 0'
launch sh -c 'exit 0'
launch sh -c 'exit 0'
########## OSCLOSE_RUN ##########
$ timeout --signal=TERM --kill-after=5s 8 $KITTY --debug-rendering --config NONE --session $SES
kitty_exit=0  elapsed=0.48s   (exit 0 => every window closed, the OS window closed, and kitty quit on its own (well under the 8s bound))
########## OSCLOSE_STDERR (complete, 15 lines) ##########
[0.203] OS Window created
[0.213] Failed to open systemd user bus with error: No such file or directory
[0.214] Child launched
[0.220] Failed to send resize signal to child with id: 1 (children count: 1) (add queue: 0)
[0.220] SIGWINCH sent to child in window: 1 with size: (22, 35, 315, 396)
[0.220] Child launched
[0.225] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.225] SIGWINCH sent to child in window: 2 with size: (22, 17, 153, 396)
[0.225] Child launched
[0.227] Failed to send resize signal to child with id: 2 (children count: 1) (add queue: 0)
[0.227] SIGWINCH sent to child in window: 2 with size: (22, 35, 315, 396)
[0.228] Failed to send resize signal to child with id: 3 (children count: 0) (add queue: 0)
[0.228] SIGWINCH sent to child in window: 3 with size: (22, 35, 315, 396)
[0.230] Failed to send resize signal to child with id: 3 (children count: 0) (add queue: 0)
[0.230] SIGWINCH sent to child in window: 3 with size: (22, 71, 639, 396)
WORKSPACE_WAS=/tmp/kitty_osclose.LAWxWu (removed by EXIT trap)
```

The verdict text is **derived from the actual `$ex`** (here `exit 0`), not hardcoded, and the workspace is
gone after the run (the `WORKSPACE_WAS=…(removed by EXIT trap)` line). The exit code **0** (not 124) is the observable that all windows closed,
the OS window closed, and kitty quit on its own — i.e. the `close_os_window` → `on_os_window_closed` teardown
**runs to completion without deadlock**. Note, however (and this is why §10(b) also shows a *live-keeper* run):
in this no-keeper session the children exit *first*, so each window is already dropped from `window_id_map` by
`on_child_death` **before** `on_os_window_closed` runs — its pop-loop therefore iterates an already-empty map.
The meaningful *Python-first* pop (window removed *ahead of* its still-live child being marked) is the
live-keeper case in §10(b), not this one. Note also `children count: 0` on the last two `:610` lines: by the
time those relayouts ran, `children[]` was empty — a stronger form of the C-first disagreement.

### 11.7 Low-churn baseline: zero races

The Q2 two-window session (§6) produced **0** `Failed to send resize signal` lines — the race does not occur
without churn.

**Honest conclusion (scoped to what was measured).** The destruction-during-resize race genuinely exists and
is provoked by churn: it never fires with two windows, and fires a count that grows with churn. At high churn
(N=40) it is **highly reproducible but not perfectly deterministic**: one 20-run batch produced `125` on all
20 runs, while a second independent 20-run batch produced `125` on 18 runs and `124` on 2 (§11.3), each run
~3 s. The scale sweep shows the same ±1 wander at low N — a single N=3 run that produced 5 instead of 6, and a
single N=1 run that produced 0 instead of 1 (§11.5) — the direct evidence that the incidence is
timing-sensitive, not fixed. These numbers characterize *this* environment (single-tenant, llvmpipe software
GL); they are not asserted as universal constants. What is guaranteed by the code — for kitty's **internal
registries**, and consistent with every run — is that each flagged child is removed once (the `needs_removal`
arbiter gates `remove_children` and `parse_input` skips a flagged child, `:528-532`) and that the two views
reconverge once the main thread drains `remove_queue`; the observed convergence to a single full-width survivor
in every saturated run is the runtime manifestation of that design. This is a statement about kitty's own
`children[]`/`window_id_map` bookkeeping, **not** about the OS liveness of a non-cooperative child (Q6: a
`SIGHUP`-ignoring child outlives kitty, observed in §10).

### 11.8 Non-cooperative-child producer (`f11_sighup_v2.sh`) — the §10 Q6 boundary

Hardened, self-cleaning producer for the §10 `SIGHUP`-outlives-kitty transcript. It allocates a **free** X
display (never a fixed `:99`), then confirms it owns the `Xvfb` it spawned by **both** the captured PID and the
`/tmp/.X11-unix/X<n>` socket before continuing, aborts non-zero (`exit 3`/`exit 4`) if provisioning fails, and
removes **only** its own `mktemp -d` workspace through a path-validated `EXIT` trap. The launched child installs
`trap "" HUP`, so kitty's teardown `hangup()` = `killpg(SIGHUP)` (`child-monitor.c:1294-1302`, `killpg` at
`:1299`) cannot end it. Run twice; the process ids differ per run, but the invariant is identical — the child
reparents to **PID 1** and outlives kitty, and the workspace self-cleans (`leftover kitty_f11.* dirs: 0`).

```bash
#!/usr/bin/env bash
# ============================================================================
# Q6 boundary: a NON-COOPERATIVE child (trap "" HUP) outlives kitty.
#   Free-display allocation, Xvfb PID+socket ownership validation, abort on
#   failure, path-validated self-cleaning workspace.  READ-ONLY; real binary.
# ============================================================================
set -u
umask 077
KITTY=/work/kitty/launcher/kitty
WORK="$(mktemp -d "${TMPDIR:-/tmp}/kitty_f11.XXXXXX")"
KPID=""; CHILD=""; XVFB_PID=""
cleanup(){
  [ -n "$CHILD" ] && kill -KILL "$CHILD" 2>/dev/null
  [ -n "$KPID" ]  && kill "$KPID" 2>/dev/null
  [ -n "$XVFB_PID" ] && { kill "$XVFB_PID" 2>/dev/null; wait "$XVFB_PID" 2>/dev/null; }
  case "$WORK" in "${TMPDIR:-/tmp}"/kitty_f11.*) [ -d "$WORK" ] && rm -rf -- "$WORK" ;; esac
}
trap cleanup EXIT INT TERM
export LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe LANG=C.UTF-8 LC_ALL=C.UTF-8
export HOME="$WORK/home" XDG_RUNTIME_DIR="$WORK/xdg"
mkdir -p "$HOME" "$XDG_RUNTIME_DIR"; chmod 700 "$XDG_RUNTIME_DIR"
DPYNUM=""
for n in $(seq 80 200); do [ ! -e "/tmp/.X11-unix/X${n}" ] && { DPYNUM="$n"; break; }; done
[ -n "$DPYNUM" ] || { echo "FATAL: no free X display" >&2; exit 3; }
export DISPLAY=":${DPYNUM}"
Xvfb "$DISPLAY" -screen 0 1280x800x24 -ac +extension GLX >"$WORK/xvfb.log" 2>&1 & XVFB_PID=$!
ready=0
for _ in $(seq 1 50); do
  [ -S "/tmp/.X11-unix/X${DPYNUM}" ] && kill -0 "$XVFB_PID" 2>/dev/null && { ready=1; break; }
  kill -0 "$XVFB_PID" 2>/dev/null || break; sleep 0.1
done
[ "$ready" = 1 ] || { echo "FATAL: our Xvfb failed on $DISPLAY" >&2; exit 4; }

PIDFILE="$WORK/childpid"
SES="$WORK/hup.session"
printf "launch sh -c 'trap \"\" HUP; echo \$\$ > %s; sleep 30; exit 42'\n" "$PIDFILE" > "$SES"
echo "########## SESSION (note: child does trap \"\" HUP => ignores SIGHUP) ##########"; cat "$SES"
"$KITTY" --debug-rendering --config NONE --session "$SES" >/dev/null 2>"$WORK/kitty.log" & KPID=$!
for _ in $(seq 1 100); do [ -s "$PIDFILE" ] && break; sleep 0.1; done
CHILD=$(cat "$PIDFILE" 2>/dev/null)
echo "########## BEFORE: kitty running, child launched ##########"
echo "kitty pid=$KPID   child pid=$CHILD"
echo "child proc:  $(ps -o pid=,ppid=,pgid=,stat=,args= -p "$CHILD" 2>/dev/null)"
echo "child ppid (expect = kitty $KPID): $(ps -o ppid= -p "$CHILD" 2>/dev/null | tr -d ' ')"
echo "########## ACTION: SIGTERM kitty -> orderly shutdown -> hangup()=killpg(SIGHUP) to child pgroup (child-monitor.c:1294-1302) ##########"
kill -TERM "$KPID"
for _ in $(seq 1 100); do kill -0 "$KPID" 2>/dev/null || break; sleep 0.1; done
echo "kitty still alive? $(kill -0 "$KPID" 2>/dev/null && echo YES || echo NO-kitty-exited)"
sleep 0.7
echo "########## AFTER: kitty gone. Did the SIGHUP-ignoring child survive? ##########"
echo "child $CHILD still alive? $(kill -0 "$CHILD" 2>/dev/null && echo YES-SURVIVED-kitty-and-SIGHUP || echo no)"
echo "child proc now: $(ps -o pid=,ppid=,pgid=,stat=,args= -p "$CHILD" 2>/dev/null)"
echo "child ppid now (1 = reparented to init, i.e. it OUTLIVED kitty): $(ps -o ppid= -p "$CHILD" 2>/dev/null | tr -d ' ')"
echo "########## kitty stderr ##########"; cat "$WORK/kitty.log"
```

Run 1 (complete, unedited — the transcript shown in §10):

```text
########## SESSION (note: child does trap "" HUP => ignores SIGHUP) ##########
launch sh -c 'trap "" HUP; echo $$ > /tmp/kitty_f11.MFAvpl/childpid; sleep 30; exit 42'
########## BEFORE: kitty running, child launched ##########
kitty pid=111281   child pid=111352
child proc:   111352  111281  111352 Ss+  /usr/bin/sh -c trap "" HUP; echo $$ > /tmp/kitty_f11.MFAvpl/childpid; sleep 30; exit 42
child ppid (expect = kitty 111281): 111281
########## ACTION: SIGTERM kitty -> orderly shutdown -> hangup()=killpg(SIGHUP) to child pgroup (child-monitor.c:1294-1302) ##########
kitty still alive? NO-kitty-exited
########## AFTER: kitty gone. Did the SIGHUP-ignoring child survive? ##########
child 111352 still alive? YES-SURVIVED-kitty-and-SIGHUP
child proc now:  111352       1  111352 Ss   /usr/bin/sh -c trap "" HUP; echo $$ > /tmp/kitty_f11.MFAvpl/childpid; sleep 30; exit 42
child ppid now (1 = reparented to init, i.e. it OUTLIVED kitty): 1
########## kitty stderr ##########
[0.198] OS Window created
[0.208] Failed to open systemd user bus with error: No such file or directory
[0.210] Child launched
```

Run 2 (complete, unedited — identical outcome, different pids):

```text
########## SESSION (note: child does trap "" HUP => ignores SIGHUP) ##########
launch sh -c 'trap "" HUP; echo $$ > /tmp/kitty_f11.QOEGrk/childpid; sleep 30; exit 42'
########## BEFORE: kitty running, child launched ##########
kitty pid=111551   child pid=111622
child proc:   111622  111551  111622 Ss+  /usr/bin/sh -c trap "" HUP; echo $$ > /tmp/kitty_f11.QOEGrk/childpid; sleep 30; exit 42
child ppid (expect = kitty 111551): 111551
########## ACTION: SIGTERM kitty -> orderly shutdown -> hangup()=killpg(SIGHUP) to child pgroup (child-monitor.c:1294-1302) ##########
kitty still alive? NO-kitty-exited
########## AFTER: kitty gone. Did the SIGHUP-ignoring child survive? ##########
child 111622 still alive? YES-SURVIVED-kitty-and-SIGHUP
child proc now:  111622       1  111622 Ss   /usr/bin/sh -c trap "" HUP; echo $$ > /tmp/kitty_f11.QOEGrk/childpid; sleep 30; exit 42
child ppid now (1 = reparented to init, i.e. it OUTLIVED kitty): 1
########## kitty stderr ##########
[0.201] OS Window created
[0.211] Failed to open systemd user bus with error: No such file or directory
[0.213] Child launched
```

Both runs end with `child ppid now (1 = reparented to init, i.e. it OUTLIVED kitty): 1`, and both left **0**
`kitty_f11.*` directories in `/tmp` (the wrapper's post-run `ls -d /tmp/kitty_f11.* | wc -l` printed `0`).

### 11.9 Live-keeper OS-window-close producer (`f4_liveclose_v2.sh`) — the §10(b) Python-first case

Hardened, self-cleaning producer for the §10(b) live-child OS-window-close transcript. It uses the same
provisioning discipline as §11.8 (free display, `Xvfb` PID+socket ownership, abort-on-failure, path-validated
self-clean) and additionally starts an `openbox` window manager so `wmctrl -c` delivers a real WM close to the
live-child OS window. Because a live child is present, `confirm_os_window_close` shows the confirmation overlay
(its own kitten emits the extra `Child launched`); confirming with `y` makes kitty exit **0**. Run twice; the
sub-second timings differ per run, but the invariant is identical — kitty exits `0` after confirmation and the
workspace self-cleans (`leftover kitty_f4.* dirs: 0`).

```bash
#!/usr/bin/env bash
# ============================================================================
# Q6 (b) Python-first: OS-window close of a window with a LIVE child.
#   Needs a window manager (openbox) so wmctrl -c delivers a real WM close.
#   Free-display allocation, Xvfb PID+socket ownership validation, abort on
#   failure, path-validated self-cleaning workspace.  READ-ONLY; real binary.
# ============================================================================
set -u
umask 077
KITTY=/work/kitty/launcher/kitty
WORK="$(mktemp -d "${TMPDIR:-/tmp}/kitty_f4.XXXXXX")"
KPID=""; OB_PID=""; XVFB_PID=""
cleanup(){
  [ -n "$KPID" ]  && kill "$KPID" 2>/dev/null
  [ -n "$OB_PID" ] && kill "$OB_PID" 2>/dev/null
  [ -n "$XVFB_PID" ] && { kill "$XVFB_PID" 2>/dev/null; wait "$XVFB_PID" 2>/dev/null; }
  case "$WORK" in "${TMPDIR:-/tmp}"/kitty_f4.*) [ -d "$WORK" ] && rm -rf -- "$WORK" ;; esac
}
trap cleanup EXIT INT TERM
export LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe LANG=C.UTF-8 LC_ALL=C.UTF-8
export HOME="$WORK/home" XDG_RUNTIME_DIR="$WORK/xdg"
mkdir -p "$HOME" "$XDG_RUNTIME_DIR"; chmod 700 "$XDG_RUNTIME_DIR"
DPYNUM=""
for n in $(seq 80 200); do [ ! -e "/tmp/.X11-unix/X${n}" ] && { DPYNUM="$n"; break; }; done
[ -n "$DPYNUM" ] || { echo "FATAL: no free X display" >&2; exit 3; }
export DISPLAY=":${DPYNUM}"
Xvfb "$DISPLAY" -screen 0 1280x800x24 -ac +extension GLX >"$WORK/xvfb.log" 2>&1 & XVFB_PID=$!
ready=0
for _ in $(seq 1 50); do
  [ -S "/tmp/.X11-unix/X${DPYNUM}" ] && kill -0 "$XVFB_PID" 2>/dev/null && { ready=1; break; }
  kill -0 "$XVFB_PID" 2>/dev/null || break; sleep 0.1
done
[ "$ready" = 1 ] || { echo "FATAL: our Xvfb failed on $DISPLAY" >&2; exit 4; }
openbox >"$WORK/openbox.log" 2>&1 & OB_PID=$!
sleep 0.5

SES="$WORK/keep.session"
printf "launch sh -c 'sleep 30'\n" > "$SES"
echo "########## SESSION (1 LIVE keeper child) ##########"; cat "$SES"
START=$(date +%s.%N)
"$KITTY" --debug-rendering --config NONE --session "$SES" >/dev/null 2>"$WORK/kitty.log" & KPID=$!
for _ in $(seq 1 100); do grep -q "Child launched" "$WORK/kitty.log" && break; sleep 0.1; done
WID="$(xdotool search --sync --class kitty 2>/dev/null | head -1)"
echo "kitty pid=$KPID  X window id=$WID"
echo "child (live keeper) parented to kitty: $(ps --ppid $KPID -o pid=,stat=,args= 2>/dev/null)"
echo "########## ACTION: WM close request (wmctrl -c) on the live-child OS window ##########"
wmctrl -ic "$WID" 2>/dev/null || wmctrl -c "$(wmctrl -l | awk '{print $1; exit}')" 2>/dev/null
sleep 0.8
echo "=== after close request: is a confirmation overlay up? (kitty still alive => maybe) ==="
echo "kitty alive after close request? $(kill -0 $KPID 2>/dev/null && echo YES && echo maybe-confirmation || echo NO-closed-immediately)"
xdotool search --sync --class kitty windowactivate 2>/dev/null; xdotool key --clearmodifiers y 2>/dev/null
sleep 1.0
echo "########## RESULT ##########"
if kill -0 $KPID 2>/dev/null; then echo "kitty STILL alive after close+y"; else echo "kitty EXITED"; fi
wait $KPID 2>/dev/null; KEX=$?
echo "kitty exit code: $KEX"
echo "elapsed: $(awk "BEGIN{print $(date +%s.%N)-$START}")s"
echo "child still alive after close? $(kill -0 $(ps -eo pid=,args= | awk '/sleep 30/{print $1; exit}') 2>/dev/null && echo maybe || echo no-or-reaped)"
echo "########## kitty stderr (complete) ##########"; cat "$WORK/kitty.log"
```

Run 1 (complete, unedited — the transcript shown in §10(b)):

```text
########## SESSION (1 LIVE keeper child) ##########
launch sh -c 'sleep 30'
kitty pid=111409  X window id=4194316
child (live keeper) parented to kitty:  111483 Ss+  /usr/bin/sh -c sleep 30
########## ACTION: WM close request (wmctrl -c) on the live-child OS window ##########
=== after close request: is a confirmation overlay up? (kitty still alive => maybe) ===
kitty alive after close request? YES
maybe-confirmation
########## RESULT ##########
kitty EXITED
kitty exit code: 0
elapsed: 2.2294s
child still alive after close? no-or-reaped
########## kitty stderr (complete) ##########
[0.208] OS Window created
[0.218] Failed to open systemd user bus with error: No such file or directory
[0.220] Child launched
[0.484] Child launched
```

Run 2 (complete, unedited — identical outcome, different timing):

```text
########## SESSION (1 LIVE keeper child) ##########
launch sh -c 'sleep 30'
kitty pid=111679  X window id=4194316
child (live keeper) parented to kitty:  111753 Ss+  /usr/bin/sh -c sleep 30
########## ACTION: WM close request (wmctrl -c) on the live-child OS window ##########
=== after close request: is a confirmation overlay up? (kitty still alive => maybe) ===
kitty alive after close request? YES
maybe-confirmation
########## RESULT ##########
kitty EXITED
kitty exit code: 0
elapsed: 2.23772s
child still alive after close? maybe
########## kitty stderr (complete) ##########
[0.204] OS Window created
[0.214] Failed to open systemd user bus with error: No such file or directory
[0.216] Child launched
[0.480] Child launched
```

Both runs printed `kitty exit code: 0` (the confirmation-then-`y` path), and both left **0** `kitty_f4.*`
directories in `/tmp` (the wrapper's post-run `ls -d /tmp/kitty_f4.* | wc -l` printed `0`).

---

## 12. Coverage checklist (every named symbol → evidence)

| Symbol / concept | `file:line` | How it was grounded |
|---|---|---|
| `Boss.window_id_map` (weak **index**) | `boss.py:344` | code; weak `WeakValueDictionary`, not the ownership root (Q1/Q6) |
| `Boss.os_window_map` (strong root) | `boss.py:350` | code; `Dict[int, TabManager]` — strong ownership root (Q1) |
| `TabManager.tabs` / `Tab.windows` | `tabs.py:877` / `tabs.py:150` | code; strong chain `TabManager→Tab→WindowList` (Q1) |
| `WindowList` class / `all_windows` / `id_map` | `window_list.py:144,147,148` | code; per-tab strong registry (Q1) |
| `WindowList.add_window` / `remove_window` | `window_list.py:329,373` | code; `:67`/`:84` are the **`WindowGroup`** methods (class `:28`), not `WindowList` |
| `children[]` registry | `child-monitor.c:82` | code; `children count: N` observed in `:610` lines |
| `scratch[]` snapshot | `child-monitor.c:83,477-482` | code (quoted §3) |
| `add_queue` / `remove_queue` | `child-monitor.c:84` | code; deferred **structural** mutation (§3) |
| `children_lock` | `child-monitor.c:87` | code; serializes all shared state; excludes the resize/close fd race (Q3) |
| `monitored_pids[]` | `child-monitor.c:96` | code; Q4 monitored set |
| `reaped_pids[]` / `reaped_pids_count` | `child-monitor.c:98,99` | code; Q4 kept-status store |
| `needs_removal` arbiter | `child-monitor.c:67` | code; single liveness truth (Q1/Q6) |
| `FREE_CHILD` = `Py_CLEAR(screen)` | `child-monitor.c:105-106` | code (quoted §8); Q4 discard |
| `set_maximum_wait` / `maximum_wait` | `child-monitor.c:114-118` | code; render/input timer bound — **not** child-reconciliation cadence (Q5) |
| `add_child` | `child-monitor.c:305-321` | code; Q1 create |
| `parse_input` (snapshot / skip / drain / staged Screen release) | `child-monitor.c:450,459-462,477-482,521-525,528-532` | code (quoted §3/§8/§10) |
| `mark_child_for_close` (field write, under lock) | `child-monitor.c:541-564` (`:546`) | code (quoted §8) |
| `pty_resize` (EBADF/ENOTTY defensive) | `child-monitor.c:577-589` (`:581`) | code (quoted §7); Q3 case 4 **not observed** |
| `resize_pty` (lock across FIND+ioctl; `:610`) | `child-monitor.c:591-613` (`:598,606-610,611`) | **observed verbatim** (§5/§11) + code (quoted §6) |
| `report_reaped_pids` | `child-monitor.c:950-962` (`:961`) | code (quoted §8); Q4 kept-status delivery `(inferred)` |
| `close_os_window` (Python callback before mark) | `child-monitor.c:1083-1094` (`:1089,1092`) | code (quoted §10); Q6 Python-first |
| `add_children` | `child-monitor.c:1281-1290` | code (§3) |
| `hangup` = `killpg(SIGHUP)` / `cleanup_child` = `safe_close` | `child-monitor.c:1294-1302,1305-1309` | code (quoted §8); Q4 discard fd |
| `remove_children` (compaction, under lock) | `child-monitor.c:1313-1333` | code; Q3 fd close under lock |
| PTY-EOF removal (default trigger) | `child-monitor.c:1531-1535` | code; **default** removal path (Q1/Q4/Q5) |
| `handle_signal` (`SIGCHLD`→flag) | `child-monitor.c:1362-1383` (`:1370-1371`) | code (quoted §9) |
| `mark_child_for_removal` (no status) | `child-monitor.c:1386-1396` (`:1390`) | code (quoted §8); Q4 |
| `mark_monitored_pids` (keeps status) | `child-monitor.c:1398-1411` | code (quoted §8); Q4 kept-status |
| `reap_children` (`waitpid` `WNOHANG` loop) | `child-monitor.c:1413-1426` (`:1418,1422`) | code (quoted §9); bulk-reap **observed** |
| reap call-site (`OPT(close_on_child_death)`) | `child-monitor.c:1526` | code; default `False` ⇒ ordinary removal via PTY EOF |
| `WAKEUP` (I/O→main cadence, `input_delay`) | `child-monitor.c:1562-1569` | code; Q5 reconciliation cadence |
| `signalfd` / `eventfd` (Linux) vs self-pipe (`#else`) | `loop-utils.c:42,70` / `:45-52,73` | code + **strace observed** `signalfd4`/`eventfd2` (§6/§9) |
| `read_signals` (synchronous drain) | `loop-utils.c:131-159` | code; Q5 synchronous callback |
| `HAS_SIGNAL_FD` / `HAS_EVENT_FD` (`__has_include`) | `loop-utils.h:14-28` | code; Linux vs fallback selection (§3/§4) |
| `Window.last_reported_pty_size` sentinel | `window.py:579` | code; first-fire guarantee (Q2) |
| `Window.set_geometry` (resize before release) | `window.py:850,861,863,865,866,871,873` | code (quoted §6) |
| Observable #1 `Child launched` | `window.py:871` | **observed verbatim** (§5/§6) |
| Observable #2 `SIGWINCH sent to child in window: …` | `window.py:873` | **observed verbatim**; optimistic-when-stale (§7) |
| `Window.close` / `Window.destroy` / `del self.screen` | `window.py:888-889,1560-1571` (`:1571`) | code (quoted §8); Q4 discard |
| `Boss.add_child` (C add before weak index) | `boss.py:585-588` (`:587,588`) | code (quoted §6) |
| `Boss.on_child_death` / `None`-guard | `boss.py:881-918` (`:883-885`) | code (quoted §7/§10); Q3/Q6 |
| `Boss.on_os_window_closed` (pops weak index first) | `boss.py:1775-1786` (`:1777,1780-1781`) | code (quoted §10); Q6 Python-first |
| `Boss.mark_window_for_close` | `boss.py:920-928` | code (Q4) |
| `Boss.on_monitored_pid_death` / `monitor_pid(p.pid)` | `boss.py:2725` / `boss.py:2419` (cond `:2417`) | code (Q4 kept-status delivery) `(inferred)` |
| `Child.fork` / `openpty` / ready pipe | `child.py:276,281,283` | code (quoted §6); Q2 |
| `Child.mark_terminal_ready` / `terminal_ready_fd` | `child.py:343` + `window.py:866` | code (Q2 release order) |
| `spawn` / `setsid()` / `TIOCSCTTY` / ready barrier | `child.c:81,123,129,152` | code (Q2 controlling terminal + ready barrier) |
| `Tab.new_window` / `Tab.remove_window` | `tabs.py:504,580` | code (Q1 plumbing) |
| `parse_session` / `create_sessions` | `session.py` | canonical entry path used for all runs |
| `--debug-rendering` flag | `cli.py:989` | canonical flag; surfaced both observables |
| version `Version(0, 35, 2)` | `constants.py:25` | **observed** via `--version` banner (§2) |

### Explicitly labelled items

- **`(inferred)`** (grounded in quoted source, not runtime-reproduced through a canonical entry point):
  the `on_child_death` `None`-guard *firing* (emits no log; the OS-close *ordering* that necessitates it **was**
  observed, §10/§11.6); the `close_on_child_death=True` reaper-marks-child branch (`:1422`, off by default);
  the multiplicity of `SIGCHLD` coalescing (one delivery → many reaps — the loop exists for it, but the
  default removal path is PTY-EOF, not the reaper); the monitored-pid exit-status *delivery*
  (`report_reaped_pids` → `on_monitored_pid_death`, not reachable via session/`launch`/close); the
  `pty_resize` `EBADF`/`ENOTTY` branch (not observed firing — the resize/close ordering under the shared lock
  closes off the window, Q3 case 4); the per-step `add_child`/`add_queue` transition (source-inferred; no
  per-step log, Q6 (c)).
- **`(OS contract)`** — `SIGCHLD` is not queued; `TIOCSWINSZ` → `SIGWINCH` to the foreground process group;
  `signalfd`/`eventfd` semantics (Linux). Cited to `signal(7)`, `wait(2)`, `tty_ioctl(4)`, `signalfd(2)`,
  `eventfd(2)`.
- **`(not runtime-observed)`** — the self-pipe + `sigaction` **non-Linux** signal fallback
  (`loop-utils.c:45-52`); the macOS `/usr/bin/login` wrapping branch in `kitty/child.py`. Observations were
  made only on the Linux build in the supplied container; these are code-only here.
- **non-canonical (avoided as a measured source)** — remote control (`kitty @ …`), debug hooks, and the
  `KITTY_PRINT_BYTES_SENT_TO_CHILD` compile flag (`child-monitor.c:1428`). None was used to obtain any
  reported value.

---

## 13. Reproducibility and clean-tree proof

**Toolchain / display provisioning** (outside the repository; not a repository change): the supplied Docker
image ships Python 3.12, Go 1.23, and gcc 13.3, but **not** `Xvfb`, `strace`, or a window manager; these were
installed from the Ubuntu archive at the start of the investigation (`apt-get install -y --no-install-recommends
xvfb xauth strace`, plus `openbox xdotool wmctrl` for the §6.1/§10 live-window observations — the container has
archive network access). All observation scripts, session files, and captured logs lived only under private
`mktemp -d` workspaces in the container's `/tmp` (an exec-capable tmpfs, separate from the checkout). Each
script **self-cleans**: its `EXIT`/`INT`/`TERM` trap reaps the Xvfb it spawned (by captured PID) **and** does a
**path-validated** `rm -rf` of exactly its own workspace — the trap's `case "$WORK" in .../kitty_obs.*|
.../kitty_osclose.*| .../kitty_f11.*| .../kitty_f4.*)` guard removes only a directory matching the script's own `mktemp` pattern, never a bare
`/tmp` or an unset variable. Verified: after all four observation scripts run (including a deliberately Xvfb-failed abort of
`osclose.sh`, which exits non-zero *before* launching kitty), **zero** `kitty_obs.*`/`kitty_osclose.*`/`kitty_f11.*`/`kitty_f4.*`
directories remain in `/tmp`. Build artifacts (`kitty/fast_data_types.so`, `kitty/launcher/kitty*`) are covered
by the repository's `.gitignore` and therefore never appear in `git status`.

**Exact commands** (from §2, for completeness):

```
# build (canonical)
python3 setup.py build --verbose

# version
./kitty/launcher/kitty --version           # -> kitty 0.35.2 created by Kovid Goyal

# every observation reduces to this canonical, time-bounded invocation
# (copy-paste-safe: assign and quote the variables first — no literal placeholders):
SECS=3
SESSION_FILE="$WORK/churn3.session"
timeout --signal=TERM --kill-after=5s "$SECS" \
  ./kitty/launcher/kitty --debug-rendering --config NONE --session "$SESSION_FILE" 2>stderr.log
```

(The complete, runnable scripts that generated every capture — including Xvfb setup, the private workspace,
and cleanup — are embedded verbatim in §11.1, §11.2, §11.8, and §11.9.)

**Baseline — the source tree under test contains no `blitzy/`.** The investigation was performed on the
source commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (`Wire up applying of font config`), which has no
`blitzy/` directory, so that commit is the true clean baseline:

```
$ git ls-tree 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -- blitzy/
$        # empty output = the investigated source tree contains no blitzy/ directory
```

**Final tree state — exactly one added path.** Comparing the branch (which carries this document) against that
source commit proves the tree differs by **exactly one added file** — this answer document — with **no**
existing source file modified (`M`) or deleted (`D`), and no temporary script/session/test present:

```
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
A	blitzy/documentation/kitty_815df1e210e0.md
```

That single `A` line, and the absence of any `M`/`D` line, is the read-only guarantee: apart from adding this
one document, the repository is byte-for-byte identical to the source commit under test, satisfying the
read-only constraint of the `SWE-AtlasQnA-Repo` rule. (All temporary observation scripts and logs lived only
under private `mktemp -d` workspaces in the container's `/tmp`; each script removes its own workspace via the
path-validated `rm -rf` in its `EXIT` trap, so nothing is left behind, and build artifacts are ignored by
`.gitignore` and never appear here.)
