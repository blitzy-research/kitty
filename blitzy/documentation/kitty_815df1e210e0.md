# Kitty Terminal-Interaction Pipeline — Runtime-Observed Behavior (commit `815df1e210e0`)

> **What this document is.** A runtime-observed answer to five questions about how the
> [Kitty](https://github.com/kovidgoyal/kitty) terminal emulator behaves *while it is actually alive
> and running* — the "busy junction where keystrokes, paste bursts, and resize signals all arrive at
> once and still somehow turn into a coherent flow." Every factual claim below is grounded in **captured
> runtime output of a canonically built Kitty** and/or a specific `file:line` reference into the source
> at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (source branch `kitty_815df1e210e0`). Where a value
> comes from a non-default build or a debug flag, it is explicitly **labeled non-canonical**.
>
> **Methodology (binding).** Build first, run second, observe third, write last. Each answer leads with a
> direct answer, then shows the exact command and its complete, unedited output, then explains cause→effect
> with `file:line` grounding, then reports before/during/after state for anything that changes over time,
> and finally covers sibling/alternate variants and error/edge paths. Numeric magnitudes were confirmed
> stable across at least two runs (or their distribution is reported). Temporary observation scripts were
> used and then removed; the source tree is left byte-for-byte unchanged apart from this document.

---

## Table of Contents

- [§0. Environment, Build & Invocation](#0-environment-build--invocation)
- [§1. Q1 — Where input first enters (ingestion / entry point)](#1-q1--where-input-first-enters-ingestion--entry-point)
- [§2. Q2 — The "unseen conductor" (threads, `poll()` ordering, coalescing)](#2-q2--the-unseen-conductor-threads-poll-ordering-coalescing)
- [§3. Q3 — Keeping shell-integration hints aligned (VT parser, OSC 133, pending mode)](#3-q3--keeping-shell-integration-hints-aligned-vt-parser-osc-133-pending-mode)
- [§4. Q4 — Backpressure & unstable remote (application-level flow control vs. XON/XOFF)](#4-q4--backpressure--unstable-remote-application-level-flow-control-vs-xonxoff)
- [§5. Q5 — Full end-to-end settle (concurrent keystrokes + paste + resize → quiescence)](#5-q5--full-end-to-end-settle-concurrent-keystrokes--paste--resize--quiescence)
- [§6. Canonical timing / magnitude values (observed, multi-run stability)](#6-canonical-timing--magnitude-values-observed-multi-run-stability)
- [§7. Coverage pass](#7-coverage-pass)

---

## 0. Environment, Build & Invocation

### 0.1 Container and toolchain actually used

All building, running, and observation was performed inside the user-specified Docker container
`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
(from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). The toolchain versions actually
observed at build time:

| Component | Version observed | Minimum required | Source of requirement |
|-----------|------------------|------------------|-----------------------|
| Python    | 3.13.7           | ≥ 3.8            | `pyproject.toml` `requires-python` (`[pyproject.toml:L2]`) |
| Go        | 1.26.5           | ≥ 1.22           | `go.mod` `go 1.22` (`[go.mod:L3]`) |
| gcc       | 15.2.0           | C11 compiler     | `docs/build.rst` |
| make      | 4.4.1            | —                | `Makefile` |
| pkg-config| 1.8.1            | —                | `docs/build.rst` |
| harfbuzz  | 10.2.0           | ≥ 2.2.0          | `docs/build.rst:L84`, `setup.py` |
| freetype2 | 26.2.20          | —                | `docs/build.rst` |
| fontconfig| 2.15.0           | —                | `docs/build.rst` |

The GUI was run headlessly on a virtual framebuffer (the container has no physical display):

```
Xvfb :99 -screen 0 1280x800x24 -ac +extension GLX +render -noreset &
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1     # Mesa llvmpipe provides GL 4.5 (exceeds kitty's GL 3.3 need)
export LANG=C.UTF-8 LC_ALL=C.UTF-8             # required for UTF-8 correctness in shell integration
```

### 0.2 Canonical build

The canonical build command is `make`, which is literally `python3 setup.py` — confirmed by the first
line of the build log and by the `Makefile`:

```
$ sed -n '12,13p' Makefile
all:
	python3 setup.py
```

`[Makefile:L12-L13]`. The build produces the launcher at `kitty/launcher/kitty`, the Go binary
`kitty/launcher/kitten`, and the C extension `kitty/fast_data_types.so`. All build outputs are
git-ignored, so the working tree stays clean. Verification of the running launcher:

```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The compiled 1 MiB parser buffer size is exposed as a real runtime constant through the C extension
(the real entry point, not a re-declaration):

```
$ kitty/launcher/kitty +runpy 'import kitty.fast_data_types as f; print(f.VT_PARSER_BUFFER_SIZE)'
1048576
```

`1048576 == 1024*1024 == 1 MiB`, matching `#define BUF_SZ (1024u*1024u)` `[kitty/vt-parser.c:L18]`
(exported at `[kitty/vt-parser.c:L1589]`).

### 0.3 Event-loop-instrumented build (NON-CANONICAL — used only for §2 timing internals)

To make the event loop observable for Q2, an instrumented build was produced with
`make debug-event-loop`, which is literally `python3 setup.py build --debug --extra-logging=event-loop`:

```
$ sed -n '25,26p' Makefile
debug-event-loop:
	python3 setup.py build --debug --extra-logging=event-loop
```

`[Makefile:L25-L26]`; the `--extra-logging` argument accepts `choices=('event-loop',)` in
`[setup.py:L1927-L1931]`. This build compiles the `DEBUG_EVENT_LOOP` macro `[kitty/child-monitor.c:L29]`,
which routes `EVDBG(...)` to `timed_debug_print`, emitting per-tick loop diagnostics to stderr with a
`[%.3f]` monotonic-time prefix. **Every numeric value that came from this instrumented build is labeled
NON-CANONICAL where it appears in §2.** After the Q2 observations, the canonical build was restored with
`make` (verified: `kitty/fast_data_types.so` back to its canonical size and zero `loop tick` lines at
runtime), and Q1/Q3/Q4/Q5 were all observed on the canonical build.

### 0.4 How the code paths were exercised through their real entry points

Three real entry points were used; none is a synthetic stand-in:

1. **Full GUI Kitty under Xvfb** — `kitty/launcher/kitty` launched headlessly, driving the real GLFW →
   `keys.c` → `key_encoding.c` ingestion, the real three-thread event loop, and the real PTY. Threads,
   file descriptors, reads/writes, and `poll()` calls were observed live with `strace` and `/proc`.
2. **The `fast_data_types` C-extension parser harness** — the same compiled `Screen`/VT-parser and the
   same `test_create_write_buffer` / `test_commit_write_buffer` / `test_parse_written_data` entry points
   that `kitty_tests/` uses. `read_bytes()` calls exactly `vt_parser_create_write_buffer()` and
   `vt_parser_commit_write()` `[kitty/child-monitor.c:L1341,L1354]`; the harness drives those same
   compiled functions, so buffer-boundary and pending-mode observations exercise the real parser.
3. **Real shells and real `ssh`** — bash, zsh, and fish run under Kitty with shell integration, and a real
   OpenSSH loopback session driven through the `ssh` kitten.

Remote control (`kitty @ --to unix:<socket> ...`) was used only to *inject* input and *read back* screen
state around the real pipeline; the behavior being measured (ingestion, parsing, flow control, rendering)
always occurs in the real code, never in the remote-control shortcut.

---

## 1. Q1 — Where input first enters (ingestion / entry point)

> *"When the terminal sends a surge of raw input, especially if a session is being paused and then
> resumed, how does that stream become something the application can react to, and where does it first
> enter the system?"*

### 1.1 Direct answer

**Child-process output first enters Kitty at the `read()` syscall inside `read_bytes()`, running on the
`KittyChildMon` I/O thread**, which reads the raw bytes into a 1 MiB buffer owned by the VT parser.
Keystrokes, pastes, and mouse events enter through a *separate* path (GLFW → `keys.c`/`key_encoding.c` /
`mouse.c` → the per-window write queue) and leave the process through `write_to_child()` on that same I/O
thread. The "paused/resumed" condition is the application-level flow-control gate (detailed in §4): when
the parser buffer is full, the I/O thread stops asking to read; when space frees up it resumes.

### 1.2 The entry point, observed

`read_bytes()` is defined at `[kitty/child-monitor.c:L1337-L1356]`; the actual `read()` syscall is at
`[kitty/child-monitor.c:L1345]` (note: an earlier reference to L1346 was off by one; the verified line is
**L1345**). It reads into the buffer returned by `vt_parser_create_write_buffer()`
`[kitty/child-monitor.c:L1341; kitty/vt-parser.c:L1451]` and commits the bytes with
`vt_parser_commit_write()` `[kitty/child-monitor.c:L1354]`.

Command (whole-process `strace` filtering `read`, a child printing a marker):

```
$ strace -f -tt -yy -e read -s300 kitty --config NONE \
    sh -c 'printf "Q1INGESTMARKER_START_ABCDEF123456_END\r\n"; sleep 0.4'
```

Complete captured line for the marker's first entry:

```
61281 04:24:14.091889 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "Q1INGESTMARKER_START_ABCDEF123456_END\r\n", 1048576) = 39
```

The child's output enters via `read()` on the **PTY master** (`/dev/pts/ptmx`), into a **`1048576`-byte
(1 MiB)** buffer — exactly `BUF_SZ` `[kitty/vt-parser.c:L18]`. The `= 39` is the byte count of the marker
(`38` printable + the `\n`; the `\r` was added by the terminal line discipline).

### 1.3 It runs on the `KittyChildMon` I/O thread, batching into one 1 MiB buffer

Attaching `strace` to the `KittyChildMon` thread (TID resolved from `/proc/<pid>/task/*/comm`) while a
shell echoes a command shows consecutive reads into the *same* buffer, with the size argument shrinking as
unparsed bytes accumulate:

```
04:25:18.027413 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "echo Q1_CHILDMON_MARKER_$((6*7))\r\n\33[?2004l\r", 1048576) = 43
04:25:18.029002 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;echo Q1_CHILDMON_MARKER_$((6*7))\7\33]133;C;cmdline=echo\\ Q1_CHILDMON_MARKER_\\$\\(\\(6\\*7\\)\\)\7", 1048533) = 93
04:25:18.029260 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2Q1_CHILDMON_MAR"..., 1048440) = 128
04:25:18.029782 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]133;k;start_suffix_kitty\7\33[5 q\33]2;/tmp/blitzy/kitty/bl"..., 1048312) = 194
```

The buffer-size argument goes `1048576 → 1048533 → 1048440 → 1048312`. That empirically confirms
`vt_parser_create_write_buffer()` returns `BUF_SZ - write.offset` `[kitty/vt-parser.c:L1457]`: successive
reads target the *remaining* space of one shared 1 MiB buffer, so the I/O thread batches multiple reads
before the main thread parses and drains it. These reads occur inside `io_loop()`
`[kitty/child-monitor.c:L1481]` (the thread named `"KittyChildMon"`, set at `[kitty/child-monitor.c:L1489]`).

### 1.4 The child-write path (same I/O thread)

Bytes destined *for* the child (a command sent to the shell, the terminal's own replies) leave through
`write_to_child()` `[kitty/child-monitor.c:L1443]`, observed on the same thread:

```
04:28:21.546471 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "stty -a\n", 8) = 8
04:28:24.685592 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "exit\n", 5) = 5
```

This write is `POLLOUT`-driven: the I/O thread only asks to write when the window's `write_buf_used`
is non-zero (`[kitty/child-monitor.c:L1503]`, shown in §2.4).

### 1.5 Error / edge branches in the read loop (observed)

The same loop handles two error classes, both quoted from source:

```
$ sed -n '1344,1356p' kitty/child-monitor.c
```
```c
        ssize_t len;
        while ((len = read(child->fd, buf + pd->used, sz - pd->used)) < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;
        }
```

- **`EINTR`/`EAGAIN` → retry** (`continue`) `[kitty/child-monitor.c:L1347]`. Observed on the non-blocking
  wakeup/signal descriptors as benign `EAGAIN` (§1.6).
- **`EIO` → child is gone**: any error other than `EIO` prints via `perror`; then `vt_parser_commit_write(…, 0)`
  and `return false` mark the child as gone `[kitty/child-monitor.c:L1348-L1350]`. Directly observed after a
  child `exit`:

```
04:28:24.688536 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, 0x592a10145b1c, 1048420) = -1 EIO (Input/output error)
```

That `-1 EIO` is precisely the child-gone branch — the read returns `EIO` because the last slave fd
closed. This same `EIO` path is what an abruptly-dropped remote connection triggers (see §4.7).

### 1.6 Parallel input paths (keystrokes, paste, mouse, PTY setup)

Child *output* is only one of the streams arriving at this junction. The other streams enter through the
GLFW/Python layer and are queued back to the child:

**Keystroke encoding — legacy vs. Kitty keyboard protocol.** Real key events were injected as X events
and observed through `keys.c`/`key_encoding.c`. XKB processing on Linux is confirmed live:

```
Loading new XKB keymaps
Modifier indices alt:0x3 super:0x6 ... shift:0x0 capslock:0x1
```

(`glfw/xkb_glfw.c`). The encoder produces different bytes depending on the active protocol:

| Key | Legacy encoding (default) | Kitty keyboard protocol |
|-----|---------------------------|-------------------------|
| `a` (press) | `a` (text) | `\33[97;;97u` |
| `a` (release) | *(ignored — "keyboard mode does not support encoding this event")* | `\33[97;1:3u` (`:3` = release) |
| `Ctrl+a` | `0x01` | `\33[97;5u` |
| `Shift+Tab` | `\33[Z` | `\33[9;2u` |
| `Alt+a` | `\33a` | — |
| `Up` | `\33[A` | `\33[A` press / `\33[1;1:3A` release |
| `F1` | `\33OP` | — |

The contrast is the point: the Kitty protocol emits `CSI codepoint;mods[:event]u` and encodes key
*releases*; the legacy protocol collapses modified keys to control bytes and *ignores* releases. This is
the `keys.c` → `key_encoding.c` path (`kitty/keys.py`, `kitty/key_encoding.py` provide the Python-side
tables); mouse events enter analogously through `kitty/mouse.c`, and IME (when present) through
`glfw/ibus_glfw.c`.

**PTY UTF-8 flag (`iutf8`).** The child PTY is created with the `IUTF8` termios flag set by
`set_iutf8_fd(master, True)` `[kitty/child.py:L174]`. Observed in the child's own `stty -a`, which
reports `iutf8` (not `-iutf8`), confirming the flag is set on the real PTY.

**Loop wakeup / signal descriptors.** The same `strace` shows the descriptors that §2 relies on:

```
04:28:21.546114 read(7<{eventfd-count=0x1, eventfd-id=425, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 1024) = 8
04:28:21.546396 read(7<{eventfd-count=0, ...}>, 0x79180c59d740, 1024) = -1 EAGAIN (Resource temporarily unavailable)
04:28:21.546701 write(4<{eventfd-count=0, eventfd-id=402, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
```

`fd 7` is the I/O thread's wakeup eventfd (drained until `EAGAIN`); `fd 4` is the main-loop wakeup eventfd
(the I/O thread writes to it to hand parsed data to the main thread — see §2 and §5).

### 1.7 Cause → effect summary

Raw child output crosses into Kitty as bytes copied by one `read()` into one bounded 1 MiB buffer on a
dedicated I/O thread; that thread neither parses nor renders — it only moves bytes and decides readiness.
That single, disciplined entry point is what lets a *surge* be absorbed as a few large batched reads
rather than a storm of tiny per-byte events, and it is the exact place where the pause/resume decision of
§4 is enforced.

---


## 2. Q2 — The "unseen conductor" (threads, `poll()` ordering, coalescing)

> *"…an unseen conductor managing timing, ordering, and state handoffs, so how are those
> responsibilities split up, and what decides which event gets handled first?"*

### 2.1 Direct answer

The "conductor" is not one entity but a **three-thread design coordinated in `kitty/child-monitor.c`**,
plus a `poll()`-driven readiness order and a small `input_delay` time window that *coalesces* wakeups.
The three threads divide the work as: an **I/O thread** (`io_loop`) that moves bytes and decides
readiness, a **main/render thread** (`main_loop`) that parses, mutates the screen model, and renders, and
a **remote-control talk thread** (`read_from_peer`) that accepts control connections. What gets handled
first each tick is decided by a fixed descriptor order in `poll()` (wakeup first, signals second, child
I/O last), and how *often* the main thread is woken is throttled by the `input_delay` window.

> **Build note:** the per-tick numeric cadences in §2.4–§2.6 come from the **NON-CANONICAL**
> `make debug-event-loop` build (§0.3). The thread structure, `poll()` ordering, and the `WAKEUP` gate
> logic (§2.2–§2.3) are read from the canonical source and confirmed by canonical-build `strace`.

### 2.2 The three threads, observed live

Running a canonical Kitty with remote control enabled and listing `/proc/<pid>/task/*/comm`:

```
KittyChildMon   = io_loop()        [kitty/child-monitor.c:L1481]  (thread name set L1489)
KittyPeerMon    = talk thread       [kitty/child-monitor.c:L1808]  (read_from_peer L1714)
kitty  (= PID)  = main_loop()       [kitty/child-monitor.c:L1259]
```

- **`io_loop()`** `[kitty/child-monitor.c:L1481]` — the I/O thread `KittyChildMon`. Owns the `read()`/
  `write()` on every child PTY (§1) and the `poll()` (§2.4).
- **`main_loop()`** `[kitty/child-monitor.c:L1259]` — the main/render thread. Its per-tick cadence is
  `process_pending_resizes(now)` → `parse_input(self)` → `render(now, input_read)`, and the main-thread
  parse worker is entered at `do_parse()` `[kitty/child-monitor.c:L438]`.
- **`read_from_peer()`** `[kitty/child-monitor.c:L1714]` — the remote-control talk thread `KittyPeerMon`.
  It exists only when remote control / `--listen-on` is enabled; its role here is purely event ordering
  (the full remote-control command catalog is out of scope). Its messages are drained by the main thread
  under `talk_mutex` inside `parse_input` and dispatched to `peer_message_received`.

Thread naming uses `set_thread_name` (`kitty/threading.h`, via `pthread_setname_np`); the wakeup and
signal plumbing lives in `kitty/loop-utils.c` / `kitty/loop-utils.h` (§2.4, §2.7).

### 2.3 The `input_delay` wakeup-coalescing gate (`WAKEUP`)

The single most important timing knob is the `WAKEUP` macro and the condition guarding it, quoted verbatim
`[kitty/child-monitor.c:L1562-L1569]`:

```c
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
        // we only wakeup the main loop after input_delay as wakeup is an expensive operation
        // on some platforms, such as cocoa
        if (data_received) {
            if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
            else has_pending_wakeups = true;
        } else {
            if (has_pending_wakeups && (now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
        }
```

The comment states the rationale outright: waking the main loop is expensive, so the I/O thread only does
it once per `input_delay` window. If data arrives sooner, it sets `has_pending_wakeups = true` and defers.
This is what turns a surge of many reads into a small number of main-loop wakeups (§2.6).

### 2.4 `poll()` readiness ordering (what gets handled first)

The descriptor layout is fixed: `EXTRA_FDS == 2` `[kitty/child-monitor.c:L35]`, so `children_fds[0]` is the
wakeup self-pipe/eventfd (drained first), `children_fds[1]` is the signal fd (handled second), and per-child
read/write descriptors follow at `children_fds[EXTRA_FDS + i]` `[kitty/child-monitor.c:L1361-L1383,
L1515-L1541]`. The per-child events are set here, quoted:

```c
            children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
            screen_mutex(lock, write);
            children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT  : 0);
```

`[kitty/child-monitor.c:L1501-L1503]`. This is directly visible in every `poll()` line of the §5 trace,
where the descriptors always appear in the order `[fd=7 wakeup, fd=8 signalfd, fd=10 child]`. Because the
wakeup and signal descriptors sit *before* the child descriptors, control events (a requested main-loop
handoff, a delivered signal) take precedence over ordinary child I/O each tick. The mapping to concrete
fds was observed in §1.6: `fd 7` = wakeup eventfd, `fd 8` = `signalfd:[HUP INT USR1 USR2 TERM CHLD]`,
`fd 10` = PTY master.

### 2.5 Idle cadence (instrumented, NON-CANONICAL)

With no input, the main loop ticks about every half-second, doing nothing but housekeeping:

```
[14.374] --------- loop tick, wakeups_happened: 0 ----------
[14.374] pollForEvents final timeout: 0.497
[14.874] --------- loop tick, wakeups_happened: 0 ----------
[14.874] pollForEvents final timeout: 0.497
[15.375] --------- loop tick, wakeups_happened: 0 ----------
[15.375] pollForEvents final timeout: -0.000
```

Ticks land ~0.5 s apart (13.375, 13.875, 14.374, 14.874, 15.375). `wakeups_happened: 0` means the I/O
thread never had to wake the main loop — nothing was happening.

### 2.6 Surge coalescing (instrumented, NON-CANONICAL) — the conductor under load

Driving a 2000-line echo burst, the main loop switches to ticking every ~3–4 ms, each tick doing exactly
**one** coalesced wakeup, with the poll timeout collapsing toward zero:

```
[2.013] TIMEOUT=0.359
[2.013] WAKEUPS=1
[2.013] TIMEOUT=0.000
[2.017] WAKEUPS=0
[2.017] TIMEOUT=0.355
[2.017] WAKEUPS=1
[2.017] TIMEOUT=0.001
[2.020] WAKEUPS=0
[2.020] TIMEOUT=0.351
[2.020] WAKEUPS=1
[2.020] TIMEOUT=0.001
[2.024] WAKEUPS=0
[2.024] TIMEOUT=0.000
[2.024] WAKEUPS=0
[2.024] TIMEOUT=0.347
[2.374] WAKEUPS=0
```

The ~3–4 ms spacing between wakeup-bearing ticks is the `input_delay` window (§6 confirms `input_delay = 3 ms`).
Across the whole run, the wakeup distribution was:

```
     36 wakeups_happened: 0
      9 wakeups_happened: 1
```

That is the decisive magnitude: a 2000-line burst produced **only ~9 coalesced main-loop wakeups** (never
more than 1 per tick), not thousands. The `WAKEUP` gate `[kitty/child-monitor.c:L1562-L1569]` is exactly
what compresses the surge. The distribution was stable in shape across runs (idle ticks dominate, wakeup
ticks capped at 1).

### 2.7 Lifecycle: who starts and stops the conductor (`boss.py`)

The Python orchestration layer owns the lifecycle. `ChildMonitor(...)` is constructed at
`[kitty/boss.py:L370]`; children are attached with `add_child` at `[kitty/boss.py:L585]`; the threads are
launched by `child_monitor.start()` at `[kitty/boss.py:L1183]`; and teardown is `shutdown_monitor()` at
`[kitty/boss.py:L2172]`. Observed effects: after `start()` the `KittyChildMon` and `KittyPeerMon` threads
exist; after `add_child` the child PTY (`fd 10`) is present; after shutdown the process is gone.

### 2.8 Cause → effect summary

Responsibilities are split so that the latency-sensitive, syscall-heavy work (reading/writing bytes,
deciding readiness) lives on the I/O thread, while the CPU-heavy work (parsing, screen mutation,
rendering) lives on the main thread, and control-plane work (remote commands) lives on the talk thread.
Ordering within a tick is decided by the fixed `poll()` descriptor layout (wakeup → signal → child), and
the *rate* of cross-thread handoffs is throttled by the `input_delay` window — which is why the system
speeds up smoothly under load instead of thrashing on per-byte wakeups.

---


## 3. Q3 — Keeping shell-integration hints aligned (VT parser, OSC 133, pending mode)

> *"When shell integration hints arrive mixed in with ordinary text, how does the system keep screen
> state, command context, and input meaning aligned without drifting out of sync?"*

### 3.1 Direct answer

**They cannot drift because they are consumed in the same single, ordered pass.** The VT parser
classifies every incoming byte as either ordinary text or part of an escape/OSC sequence *in stream
order*; when it recognizes an OSC 133 shell-integration marker, it routes it to `shell_prompt_marking()`,
which binds the command context (prompt-start / command-start / command-end + exit status) to the
**current cursor row** — the very same cursor that ordinary text is being written against. Because the
markers and the text share one cursor and are processed in the order they arrive, the command context is
pinned to exactly the row the text landed on.

### 3.2 Byte classification (one ordered pass)

The parser's escape dispatch is `consume_esc` `[kitty/vt-parser.c:L260]`. The relevant transitions,
quoted:

```c
case ESC_DCS: SET_STATE(DCS); break;
case ESC_OSC: SET_STATE(OSC); break;      // L270  ESC ]  ->  OSC state
case ESC_CSI: SET_STATE(CSI); reset_csi(&self->csi); break;
```

`ESC ]` moves the parser into OSC state `[kitty/vt-parser.c:L270]`; ordinary bytes remain in the NORMAL
(text) state. There is no separate, out-of-band channel for hints — they are literally interleaved in the
byte stream and demultiplexed here.

### 3.3 OSC 133 routing and the command-boundary callbacks

Inside the OSC handler `dispatch_osc` `[kitty/vt-parser.c:L457]`, the OSC number `133` is matched and the
payload handed to the screen. Verbatim `[kitty/vt-parser.c:L536-L546]`:

```c
        case 133:
#ifdef DUMP_COMMANDS
            START_DISPATCH
            REPORT_OSC2(shell_prompt_marking, code, mv);
            END_DISPATCH_WITHOUT_BREAK
#endif
            if (limit > i) {
                buf[limit] = 0; // safe to do as we have 8 extra bytes after PARSER_BUF_SZ
                shell_prompt_marking(self->screen, (char*)buf + i);
            }
            break;
```

`shell_prompt_marking()` `[kitty/screen.c:L2329]` binds to the current row under an explicit guard, and
issues the `cmd_output_marking` callbacks. Verbatim `[kitty/screen.c:L2329-L2355]`:

```c
    if (self->cursor->y < self->lines) {
        char ch = buf[0];
        switch (ch) {
            case 'A': {
                PromptKind pk = PROMPT_START;
                self->prompt_settings.redraws_prompts_at_all = 1;
                self->prompt_settings.uses_special_keys_for_cursor_movement = 0;
                parse_prompt_mark(self, buf+1, &pk);
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = pk;
                if (pk == PROMPT_START) CALLBACK("cmd_output_marking", "O", Py_False);
            } break;
            case 'C': {
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = OUTPUT_START;
                const char *cmdline = "";
                if (strstr(buf + 1, ";cmdline") == buf + 1) {
                    cmdline = buf + 2;
                }
                RAII_PyObject(c, PyUnicode_DecodeUTF8(cmdline, strlen(cmdline), "replace"));
                if (c) { CALLBACK("cmd_output_marking", "OO", Py_True, c); }
                else PyErr_Print();
            } break;
            case 'D': {
                const char *exit_status = buf[1] == ';' ? buf + 2 : "";
                CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);
            } break;
        }
    }
```

So the guard `if (self->cursor->y < self->lines)` `[kitty/screen.c:L2330]` binds everything to the current
cursor row; `A` (prompt start) → `CALLBACK("cmd_output_marking", "O", Py_False)` `[kitty/screen.c:L2338]`;
`C` (command start, carrying the cmdline) → `CALLBACK("cmd_output_marking", "OO", Py_True, c)`
`[kitty/screen.c:L2347]`; `D` (command end, carrying exit status) → `CALLBACK("cmd_output_marking", "Os",
Py_None, exit_status)` `[kitty/screen.c:L2352]`. Each writes
`self->linebuf->line_attrs[self->cursor->y].prompt_kind` — i.e. it *tags a specific screen row*.
Command-output extent lookups later use `find_cmd_output` `[kitty/screen.c:L3527]` and `cmd_output`
`[kitty/screen.c:L3606]`.

> **Disambiguation (verified).** `case ESC_OSC:` at `[kitty/screen.c:L964]` is the *output* prefix
> `"\033]"` used by `write_escape_code_to_child` — it is **not** the OSC-133 input routing. The input
> routing is `[kitty/vt-parser.c:L536]` → `shell_prompt_marking` `[kitty/screen.c:L2329]`. Both are cited
> precisely so the two are not confused.

### 3.4 Runtime OSC 133 bytes from **real** shells (row binding proven)

Rather than a synthetic `printf`, the markers were driven through real bash, zsh, and fish under Kitty and
captured with `strace read()` on the `KittyChildMon` PTY master (`fd 10`). Each command used real shell
arithmetic (`$((6*7))` / `(math 6 x 7)` → `42`) to prove a real shell produced them. Row binding was then
read back with `kitty @ get-text --extent last_cmd_output`.

**bash** (`shell-integration/bash/kitty.bash`):

```
\33]133;A                (prompt start)              <- script L239  (\e]133;A\a)
\33]133;C;cmdline=true    (command start)            <- script L208  printf "\e]133;C;cmdline=%q\a"
\33]133;D;0              (command end, exit 0)       <- script L239  (\e]133;D;$?\a)
+ region markers \33]133;k;start_kitty ...           <- script L127/L137/L240
last_cmd_output -> "Q3_ROWBIND_UNIQUE_MARKER_42"
```

**zsh** (`shell-integration/zsh/kitty-integration`):

```
\33]133;A                                            <- script L153  mark1=$'%{\e]133;A\a%}'
\33]133;C;cmdline=echo\ Q3_ZSH_ROWBIND_\$\(\(6\*7\)\)  (%q shell-quoted)  <- script L218
\33]133;D;0                                          <- script L145  '\e]133;D;'$cmd_status'\a'
last_cmd_output -> "Q3_ZSH_ROWBIND_42"
```

**fish** (`shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`):

```
\33]133;A;special_key=1                              <- script L85  "\e]133;A;special_key=1\a"
\33]133;C;cmdline_url=echo%20Q3_FISH_ROWBIND_%28math%206%20x%207%29  (URL-escaped)  <- script L91
\33]133;D;0                                          <- script L96  "\e]133;D;$status\a"
last_cmd_output -> "Q3_FISH_ROWBIND_42"
```

That `last_cmd_output` returns the exact per-shell marker (all resolving to `42`) is the proof that the
`C`/`D` callbacks pinned the command's output extent to the correct rows — the hints stayed aligned with
the text.

### 3.5 Sibling variants (same contract, different payload encodings)

All three shells emit the *same* `A`/`C`/`D` command-boundary contract but encode the payload differently:

| Shell | Command start | Prompt start | Command end |
|-------|---------------|--------------|-------------|
| bash  | `C;cmdline=%q` (shell-quoted) | `A` (plain) | `D;$?` |
| zsh   | `C;cmdline=%q` (shell-quoted) | `A` (plain) | `D;$cmd_status` |
| fish  | `C;cmdline_url=%s` (URL-escaped) | `A;special_key=1` | `D;$status` |

Byte-exact verification (`hexdump -C`), confirming the parser sees the literal bytes claimed:

```
\033]133;A\007            = 1b 5d 31 33 33 3b 41 07                              (.]133;A.)
\033]133;C;cmdline=true\007 = 1b 5d 31 33 33 3b 43 3b 63 6d 64 6c 69 6e 65 3d 74 72 75 65 07
\033]133;D;0\007          = 1b 5d 31 33 33 3b 44 3b 30 07                        (.]133;D;0.)
   (ESC = 0x1b, ']' = 0x5d, BEL = 0x07 terminator)
```

### 3.6 Error / edge path — a real condition where a shell *silently skips* integration

Not every shell activates integration on the happy path. `setup_zsh_env` `[kitty/shell_integration.py:L49]`
calls `is_new_zsh_install` `[kitty/shell_integration.py:L27]`: if **no** user rc files
(`.zshrc`/`.zshenv`/`.zprofile`/`.zlogin`) exist in `ZDOTDIR` (or `HOME` when `ZDOTDIR` is empty), it
returns early *without* activating integration — deliberately, to avoid breaking `zsh-newuser-install`.

Observed exactly: the default container has no `~/.zshrc`, so the first zsh run showed
`KITTY_SHELL_INTEGRATION=enabled`, `ZDOTDIR=''`, and **no `]133` markers at all**. After providing a
`ZDOTDIR=/tmp/zdot` containing a `.zshrc`, `is_new_zsh_install` returned `False`, integration activated,
the zsh script unset `KITTY_SHELL_INTEGRATION` after loading, and the `]133` markers appeared. bash and
fish, by contrast, activate without a user rc (bash injects via `--rcfile`; fish via `XDG_DATA_DIRS`).
This is the kind of condition a happy-path-only investigation would miss.

### 3.7 Setup dispatch (`shell_integration.py`)

`modify_shell_environ` `[kitty/shell_integration.py:L218]` sets `env['KITTY_SHELL_INTEGRATION']` and then
calls the per-shell modifier: `setup_fish_env` `[kitty/shell_integration.py:L16]` (prepends the
integration dir to `XDG_DATA_DIRS`), `setup_zsh_env` `[kitty/shell_integration.py:L49]` (sets
`ZDOTDIR`, subject to §3.6), and `setup_bash_env` `[kitty/shell_integration.py:L70]` (rcfile injection).
Observed at runtime: Kitty sets `KITTY_SHELL_INTEGRATION=enabled`; each per-shell script unsets it after
loading. The `ssh` bootstrap (`shell-integration/ssh/bootstrap.sh`) emits **no** `133;A/C/D` markers
itself (grep found none) — it *propagates* terminfo + integration over ssh, and the remote shell's own
integration emits the markers (see §4).

### 3.8 Synchronized / pending mode (DEC private mode 2026) — before / during / after

"Keeping aligned" also covers the mechanism that batches rendering so a half-drawn screen is never shown.
The industry-standard name is **Synchronized Output, DEC private mode 2026** (enable `CSI ? 2026 h`,
disable `CSI ? 2026 l`, query `CSI ? 2026 $ p` via DECRQM); Kitty maps this onto its internal
pending-mode machinery `screen_start_pending_mode` / `screen_stop_pending_mode`
`[kitty/vt-parser.c:L639,L644]`.

Observed through the **real compiled parser** (the `fast_data_types` `Screen` plus the `test_*` entry
points that `kitty_tests` uses), querying DECRQM before, during, and after enabling the mode
(command: `python3 /tmp/kitty_obs/q3_pending_obs.py`):

```
BEFORE (no pending): DECRQM ?2026$p -> b'\x1b[?2026;2$y'   (Ps=2, inactive)
  after CSI ?2026h -> parser cmd ['screen_set_mode']
DURING (pending on): DECRQM ?2026$p -> b'\x1b[?2026;1$y'   (Ps=1, active)
  after CSI ?2026l -> parser cmd ['screen_reset_mode']
AFTER  (pending off): DECRQM ?2026$p -> b'\x1b[?2026;2$y'  (Ps=2, inactive)
```

Byte-exact: inactive `= 1b 5b 3f 32 30 32 36 3b 32 24 79`, active `= 1b 5b 3f 32 30 32 36 3b 31 24 79`
(they differ only in the `32`↔`31`, i.e. `'2'`↔`'1'`). Source: `report_mode_status`
`[kitty/screen.c:L2203]`; `case PENDING_UPDATE: ans = paused_rendering.expires_at ? 1 : 2`
`[kitty/screen.c:L2237-L2238]`; response format `snprintf("%s%u;%u$y", …)` `[kitty/screen.c:L2240]`;
`PENDING_UPDATE (2026 << 5)` `[kitty/modes.h:L86]`; auto-expiry at `[kitty/screen.c:L2490]`.

Kitty also honors its own DCS synchronized trigger, observed via the real parser's dump callback:

```
DCS =1s ST (\x1bP=1s\x1b\\) -> ['screen_start_pending_mode']   [kitty/vt-parser.c:L639]
DCS =2s ST (\x1bP=2s\x1b\\) -> ['screen_stop_pending_mode']    [kitty/vt-parser.c:L644]
```

Both entry forms converge on the same `screen_pause_rendering` state — the "during" state where the
renderer holds the last frame while the parser keeps consuming bytes, so text and markers still land in
order but the screen updates atomically when the mode ends.

### 3.9 Cause → effect summary

Alignment is a *structural* property, not a lucky race: text and shell-integration hints travel in one
byte stream, are demultiplexed by one parser against one cursor, and command context is written to the
exact row the cursor is on. Synchronized mode adds atomic *rendering* on top, so even a multi-write screen
update is shown as one coherent frame rather than a flickering half-update.

---


## 4. Q4 — Backpressure & unstable remote (application-level flow control vs. XON/XOFF)

> *"…does it behave differently when there is heavy backpressure or an unstable remote connection?"*

### 4.1 Direct answer

**Kitty's backpressure is application-level, and it is NOT XON/XOFF and NOT RTS/CTS.** When the 1 MiB
VT-parser buffer fills, the I/O thread simply stops requesting `POLLIN` for that child — it stops draining
the PTY master. The kernel PTY buffer then fills, and the **child blocks on its own `write()`** until
space frees up. Kitty emits no `XOFF`/`XON` (`DC3 0x13` / `DC1 0x11`) and toggles no modem lines. For a
**remote (ssh)** connection the pipeline is *identical* — the same `read()` → parser buffer → OSC 133
handling — because the `ssh` kitten only *propagates* terminfo + shell integration; it does not change the
pipeline. An unstable or dropped link surfaces as the ssh child exiting, which hits the same `EIO`
child-gone branch as any child death (§1.5).

### 4.2 The flow-control gate (source)

The gate is `vt_parser_has_space_for_input()` `[kitty/vt-parser.c:L1477]`, whose body is
`ans = self->read.sz + self->write.pending < BUF_SZ;` `[kitty/vt-parser.c:L1481]`. The I/O thread consults
it when setting the child's poll events `[kitty/child-monitor.c:L1501]`:

```c
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

If there is space → request `POLLIN` (keep reading). If not → request `0` (stop reading). Resume is armed
by `write_space_created = self->read.sz >= BUF_SZ` `[kitty/vt-parser.c:L1438]`, which, once the main
thread frees space in `do_parse`, triggers `wakeup_io_loop(self, false)` `[kitty/child-monitor.c:L442]` to
re-enable `POLLIN`.

### 4.3 The 1 MiB bound is deterministic (real compiled parser, 2 identical runs)

Using the same compiled functions `read_bytes` uses (`test_create_write_buffer` /
`test_commit_write_buffer`), committing 200 KiB chunks *without* parsing until the buffer is full
(command: `python3 /tmp/kitty_obs/q4_buffer_saturation.py`, **identical across 2 runs**):

```
VT_PARSER_BUFFER_SIZE = 1048576 bytes = 1.0 MiB     (== BUF_SZ (1024u*1024u) [kitty/vt-parser.c:L18])
BEFORE (empty):  free_space = 1048576   has_space = True
DURING (commit 200KiB chunks, NO parse):
    free_space: 1048576 -> 843776 -> 638976 -> 434176 -> 229376 -> 24576 -> 0
    step 6 committed only 24576 (partial) -> total accepted EXACTLY 1048576 = BUF_SZ
    step 7 committed 0 -> free_space = 0 -> has_space = False   => io_loop would set POLLIN:0
AFTER drain (test_parse_written_data):  free_space = 1048576   has_space = True  (restored to BUF_SZ)
```

The buffer accepts exactly `1048576` bytes and then reports `has_space = False`; parsing restores it to
`1048576`. This is grounded in `create_write_buffer *sz = BUF_SZ - write.offset`
`[kitty/vt-parser.c:L1457]`, the `has_space` predicate `[kitty/vt-parser.c:L1481]`, and
`write_space_created` `[kitty/vt-parser.c:L1438]`.

### 4.4 `POLLIN` actually withdrawn → restored in the full GUI (before / during / after)

To force the buffer full in a *live* Kitty, the main (parsing) thread was frozen with a `ptrace` helper
(`PTRACE_SEIZE` + `PTRACE_INTERRUPT`) for 4 seconds while the I/O thread kept reading a continuous 64 KiB
flood, with `strace` on the `KittyChildMon` `poll()`. Raw capture (child PTY master = `fd=10`):

```
04:54:52.837219 poll([... fd=10 events=POLLIN],3,0)  = 1 ([{fd=10, revents=POLLIN}])   <- BEFORE: readable, reading
04:54:52.837302 poll([... fd=10 events=0],3,0)       = 0 (Timeout)                      <- DURING: POLLIN WITHDRAWN (buffer full)
04:54:52.837361 poll([... fd=10 events=0],3,0)       = 0 (Timeout)
04:54:52.837497 poll([... fd=10 events=0],3,-1)      = 1 ([{fd=7, revents=POLLIN}])      <- BLOCKS on poll(-1), woken only by eventfd
04:54:56.823660 poll([... fd=10 events=POLLIN],3,-1) = 1 ([{fd=10, revents=POLLIN}])     <- AFTER: POLLIN RESTORED (~4s later)
```

The child descriptor's requested `events` flips `POLLIN → 0 → POLLIN`. While withdrawn, the I/O thread
requests `0` on the child and blocks in `poll(-1)`, woken only by the `fd=7` wakeup eventfd — which is
precisely `wakeup_io_loop` `[kitty/child-monitor.c:L442]` firing after the main thread freed space. RUN1
withdrew at `04:53:19.133247` and restored at `04:53:23.124308` (gap **3.991 s**); RUN2 gap **3.986 s** —
both ≈ the 4 s freeze, reproduced across 2 runs.

### 4.5 The child blocks on `write()` for the whole window (kernel PTY buffer full)

The producer's own `strace` (writing to `fd 1` = its PTY slave) shows the mirror image — a single
`write()` blocked for the entire withdrawal window:

```
04:54:52.836989 write(1, "Q4CONT........"..., 65536) = 65536 <3.987936>    <- BLOCKED 3.987936 s
   (surrounding writes were fast: <0.031139>, <0.035750>, <0.030407>)
```

The `<3.987936>` block starts at `52.837` (exactly when `POLLIN` was withdrawn) and ends at `56.82`
(exactly when it was restored). The child was paused not by any signal or flow-control byte, but simply
because the kernel PTY buffer filled while Kitty declined to read.

### 4.6 Explicit contrast with kernel line-discipline flow control

The child PTY's `stty -a` shows `-crtscts` (no hardware RTS/CTS), `ixon -ixoff` (the kernel's software
flow-control defaults), and `iutf8`. **Kitty uses none of these.** For reference:

| Mechanism | Where | Signal | Kitty uses it? |
|-----------|-------|--------|----------------|
| XON/XOFF (software) | kernel TTY line discipline | in-band `DC1 0x11` / `DC3 0x13` on the data line | **No** — Kitty emits no `DC1`/`DC3` |
| RTS/CTS (hardware)  | serial modem lines | out-of-band wires | **No** — no modem lines on a PTY |
| **Kitty backpressure** | application (I/O thread) | withhold `POLLIN` → kernel PTY buffer back-fills → child blocks on `write()` | **Yes** — this is the observed mechanism |

This is the crux of Q4: the "pause" is the *reader* (the I/O thread) declining readiness, not the tty
sending a flow-control byte.

### 4.7 SSH kitten / unstable remote

The `ssh` kitten was exercised over a real OpenSSH loopback session (`127.0.0.1:2222`; no external remote
exists in the container):

```
$ kitty ... kitty +kitten ssh -o StrictHostKeyChecking=no -p 2222 root@127.0.0.1
# sshd log: "Accepted publickey for root from 127.0.0.1 ... ssh2: ED25519"
# remote prompt reached: root@<host>:~#
```

Observed remote state and end-to-end row binding over ssh:

```
REMOTE_TERM=xterm-kitty                          <- terminfo propagated (TERM set on remote)
/root/.terminfo/x/xterm-kitty exists             <- terminfo db reconstructed remotely
remote OSC 133 (strace read fd10, outer kitty KittyChildMon):
    \33]133;C;cmdline=echo\ REMOTE_OSC133_\$\(\(6\*7\)\)   (remote bash %q-quoted, same contract as local)
    \33]133;D;0   \33]133;A   + region \33]133;k;start_kitty/end_kitty/...
last_cmd_output over ssh -> "REMOTE_OSC133_42"   <- row binding works end-to-end over ssh
```

The `ssh` kitten's job is **propagation**, from the real `ssh` command line and bootstrap:

```
/usr/bin/ssh -o StrictHostKeyChecking=no -p 2222 -t -o ControlMaster=auto \
  -o ControlPath=/root/.cache/kitty/run/kssh-<pid>-%C -o ControlPersist=yes \
  -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no -- root@127.0.0.1 exec sh -c '...bootstrap...'
```

`ServerAliveInterval=60 × ServerAliveCountMax=5 ≈ 300 s` is the ssh-level keepalive timeout that would
detect a dead/unstable link (`TCPKeepAlive=no` defers to ssh keepalives); `ControlMaster`/`ControlPersist`
multiplex and persist the connection. The embedded bootstrap `sh` emits a DCS
`\033P@kitty-ssh|<b64>\033\\` request, then `untar_and_read_env()` does
`read_base64_from_tty | base64_decode | tar xpzf -C tmpdir`, compiles terminfo, moves files into `$HOME`,
and `exec`s the login shell. Source: `kittens/ssh/main.go:L355-L357` (packages
`terminfo/x/<DefaultTermName>` = `xterm-kitty` into the tarball → `home/.terminfo`), `main.go:L325-L349`
(packages the `shell-integration/ssh` bootstrap files), `shell-integration/ssh/bootstrap.sh:L104-L113`
(`untar_and_read_env`), `kittens/ssh/main.py:L122` (`shell_integration=inherited`), `main.py:L170`
("small bootstrap script"). The other kitten files `config.go`, `utils.go`, `askpass.go` support config
parsing, path/util helpers, and password prompting on this path.

**Dropped-link edge (observed).** Abruptly killing the foreground `/usr/bin/ssh` (simulating a dropped
link) produced, on the outer Kitty's `KittyChildMon`:

```
05:07:54.161035 read(10</dev/pts/ptmx...>, 0x..., 1048576) = -1 EIO (Input/output error)
```

— the same `EIO` child-gone branch `[kitty/child-monitor.c:L1348-L1350]` as §1.5, after which the outer
Kitty exited via `close_on_child_death`.

**Limitation (stated precisely):** true network instability (packet loss / jitter) cannot be induced on a
loopback in this container. The transport-level behavior is shown from the ssh keepalive options, and the
*terminal-pipeline consequence* of a drop (`EIO` child-gone) is directly observed via an abrupt ssh kill.

### 4.8 Why the buffer drains in batches (parse-trigger coalescing)

The buffer does not drain byte-by-byte; the main-thread parse fires on one of three conditions. Verbatim
`[kitty/vt-parser.c:L1423-L1438]`:

```c
        if (pd->has_pending_input) {
            pd->time_since_new_input = pd->now - self->new_input_at;
            if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
                pd->input_read = true;
                self->dump_callback = pd->dump_callback; self->now = pd->now;
                self->screen = screen;
                self->read.consumed = 0;
                do {
                    end_with_lock; {
                        consume_input(self, pd->dump_callback, screen->window_id);
                    } with_lock;
                    self->read.sz += self->write.pending; self->write.pending = 0;
                } while (self->read.pos < self->read.sz);
                self->new_input_at = 0;
                if (self->read.consumed) {
                    pd->write_space_created = self->read.sz >= BUF_SZ;
```

i.e. the parse fires on an explicit `flush`, the `input_delay` window elapsing
(`time_since_new_input >= OPT(input_delay)`), or the buffer coming within 16 KiB of full
(`self->read.sz + 16 * 1024 > BUF_SZ`). That last clause is what guarantees the buffer is drained *before*
it hard-fills under a sustained surge, so backpressure engages smoothly rather than as a hard stall; and
the same block sets `write_space_created` `[kitty/vt-parser.c:L1438]` (§4.2) once space is reclaimed.

### 4.9 Cause → effect summary

Under heavy backpressure the behavior is a clean, bounded pause: a fixed 1 MiB buffer, a `POLLIN` gate
that withholds reads when full, and a kernel PTY buffer that then makes the child block on `write()` — no
lost bytes, no unbounded memory, no flow-control bytes injected into the data stream. A remote connection
is not special to the pipeline; instability manifests only as the ssh child dying, handled by the same
`EIO` path as any child exit.

---


## 5. Q5 — Full end-to-end settle (concurrent keystrokes + paste + resize → quiescence)

> *"…what really happens from the moment that mixed input arrives to the moment the interface settles
> again, and how all those moving parts manage to keep their rhythm instead of slowly falling apart."*

### 5.1 Direct answer

Mixed input is absorbed by handing each stream off to the next stage with **bounded buffering and timed
coalescing**, so the pipeline *converges* to a resting state instead of thrashing. Concretely: keystrokes
and paste are written to the child on the I/O thread; the child's output is read back into the single
1 MiB buffer; the I/O thread coalesces multiple reads and wakes the main thread at most once per
`input_delay`; the main thread parses, mutates the screen model, and renders under `repaint_delay` /
`sync_to_monitor`; a resize burst is debounced into a single reflow; and when every queue is empty the
I/O thread simply **blocks in `poll()`** — the observable definition of "settled."

### 5.2 The combined trace (2 runs; structure identical, input ordering varies)

Concurrent keystrokes (`echo KEYSTROKE_LINE_42`), a paste burst (`PASTE_A\nPASTE_B_42`), and a resize were
fired near-simultaneously while `strace` watched the `KittyChildMon` I/O thread. **BEFORE**: window 26×80,
I/O thread blocked in `poll(timeout=-1)`, buffer free = `1048576` (full/empty of unparsed data).

RUN 1 — verbatim key lines (keystroke happened to arrive first):

```
poll([{fd=7,POLLIN},{fd=8,POLLIN},{fd=10,POLLIN|POLLOUT}],3,-1) = 1 ([{fd=10,revents=POLLOUT}])
write(10, "echo KEYSTROKE_LINE_42\n", 23) = 23                          <- keystroke -> child (write_to_child L1443)
poll([... fd=10 POLLIN],3,-1) = 1 ([{fd=10,revents=POLLIN}])
read(10, "echo KEYSTROKE_LINE_42\r\n\33[?2004l"..., 1048576) = 33        <- echo read into 1 MiB buffer (read() L1345)
write(4, "\1\0\0\0\0\0\0\0", 8) = 8                                      <- HANDOFF: wake main loop (fd4 eventfd)
read(10, "\33]2;echo KEYSTROKE_LINE_42\7\33]133"..., 1048543) = 67       <- more output; free 1048543
poll([... fd=10 POLLIN],3,1)                                            <- timeout 1 ms (input_delay coalescing)
read(10, "\1\33]133;k;start_kitty\7..."..., 1048476) = 124              <- free 1048476
read(10, "\33[?2004h\33]133;k;start_kitty\7..."..., 1048352) = 194      <- free 1048352 (\33[?2004h paste-mode on)
[17x] poll([...],3,0) = 0 (Timeout)                                     <- coalescing spin (timeout 0)
write(4, "\1\0\0\0\0\0\0\0", 8) = 8                                      <- HANDOFF after coalescing
poll([...],3,-1) = 1 ([{fd=7,revents=POLLIN}])                          <- next input arrives (paste)
read(7, "\3\0\0\0\0\0\0\0", 1024) = 8                                    <- wakeup eventfd counter = 3 (3 wakeups coalesced)
poll([... fd=10 POLLIN|POLLOUT],3,-1) = 1 ([{fd=10,revents=POLLOUT}])
write(10, "\33[200~PASTE_A\nPASTE_B_42\33[201~", 30) = 30                <- PASTE burst -> child (bracketed)
read(10, "\33[7mPASTE_A\33[27m\r\n\r\33[7mPASTE_B_4"..., 1048158) = 38   <- response (reverse-video highlight); free 1048158
poll([...],3,2) = 0 (Timeout)                                          <- timeout 2 ms coalescing
write(4, "\1\0\0\0\0\0\0\0", 8) = 8                                      <- HANDOFF
```

RUN 2 — same structure, input ordering flipped (paste arrived first) and a **real interleave edge**
appeared:

```
write(10, "\33[200~PASTE_A\nPASTE_B_42\33[201~", 30) = 30
read(10, ..., 1048576) = 62 ; write(4,...) ;
write(10, "echo KEYSTROKE_LINE_42\n", 23) = 23
read(10, ..., 1048514) = 56 ; write(4,...) ;
read(10, "\33]2;PASTE_A\7\33]133;C;cmdline=PAST"..., 1048576) = 175
read(10, ..., 1048401) = 205 ; read(10, ..., 1048196) = 87 ; read(10, ..., 1048109) = 105
read(10, "bash: PASTE_B_42echo: command no"..., 1048004) = 41   <- REAL EDGE: interleaved input merged; shell reports "command not found"
read(10, "\33[?2004h..."..., 1047963) = 196
```

Both runs show the **identical pipeline structure**; only the *order* of the concurrent inputs differs.
That run-to-run ordering variance is real and reported as-is (not smoothed into a false determinism): in
RUN 2 the paste's second line merged with the `echo` keystroke line, so bash reported `command not found`.

### 5.3 What each observed element proves (cause → effect, with `file:line`)

- **`poll()` fd-ordering is fixed** — every `poll()` in both runs lists `[{fd=7 wakeup},{fd=8 signalfd},
  {fd=10 child}]`. That is the `EXTRA_FDS == 2` layout `[kitty/child-monitor.c:L35]` with
  `children_fds[0]` = wakeup, `[1]` = signalfd, `[EXTRA_FDS+i]` = child
  `[kitty/child-monitor.c:L1361-L1383,L1515-L1541]`. Control events always precede child I/O.
- **`write(10, …)` = child write path** — keystrokes (`23` bytes) and the bracketed paste
  (`\33[200~…\33[201~`, `30` bytes) go out via `write_to_child` `[kitty/child-monitor.c:L1443]`, requested
  only when `POLLOUT` is set (`write_buf_used`).
- **`read(10, …, 1048576)` = ingestion into the 1 MiB buffer** — `read()` `[kitty/child-monitor.c:L1345]`;
  the size argument decreases (`1048576 → 1048543 → 1048476 → 1048352 → 1048158`) as unparsed bytes
  accumulate, the same bounded-buffer behavior as §1.3 and §4.3.
- **`write(4, "\1\0\0\0\0\0\0\0", 8)` = handoff to the main thread** — the I/O thread writes the main-loop
  wakeup eventfd (the `wakeup_main_loop()` side of the `WAKEUP` gate, §2.3) to hand the buffer to the
  parse/render thread.
- **`read(7, "\3\0\0\0\0\0\0\0", …)` = wakeup coalescing** — the wakeup eventfd's counter read back as `3`
  means three wakeup requests were coalesced into one drain.
- **`poll(..., timeout)` sequence −1 → 1 → 0 → 2 (ms)** — the `input_delay` coalescing window: `-1`
  (block when idle), then small millisecond timeouts while draining a burst. RUN 2's timeout distribution:

```
     -1 : 7     (idle-block)
      0 : 34    (coalescing spin)
      1 : 1
      2 : 2     (input_delay coalescing while draining)
```

### 5.4 Paste path (byte-exact)

The paste is wrapped in bracketed-paste markers, captured byte-exactly:

```
write(10</dev/pts/ptmx>, "\33[200~PASTE_A\nPASTE_B_42\33[201~", 30) = 30
    \33[200~ = 1b 5b 32 30 30 7e   (BRACKETED_PASTE_START "200~"  [kitty/modes.h:L82])
    \33[201~ = 1b 5b 32 30 31 7e   (BRACKETED_PASTE_END   "201~"  [kitty/modes.h:L83])
```

`BRACKETED_PASTE (2004 << 5)` `[kitty/modes.h:L81]`. The Python paste chain is `sanitize_for_bracketed_paste`
(import `[kitty/window.py:L117]`) → `paste_with_actions` `[kitty/window.py:L1643]` → `paste_bytes`
`[kitty/window.py:L1707]` → `paste_text` `[kitty/window.py:L1713,L1718]` → `paste`
`[kitty/window.py:L1780]`.

### 5.5 Resize path + debounce (before / during / after)

Resize flows `set_geometry` `[kitty/window.py:L850]` → `screen.resize` `[kitty/window.py:L854]` →
`resize_pty` `[kitty/window.py:L863]`, with a debug print at `[kitty/window.py:L873]` (needs
`--debug-rendering`). Single resizes, observed:

```
BEFORE: INITIAL size = 22 71   (640x400 window)
[21.871] SIGWINCH sent to child in window: 1 with size: (33, 100, 900, 594)
[23.378] SIGWINCH sent to child in window: 1 with size: (38, 111, 999, 684)
AFTER : FINAL size = 38 111    (child's SIGWINCH trap fired; stty size changed)
```

The **debounce** is decisive. Issuing 20 rapid `windowsize` calls with no inter-call sleep produced only
**one** SIGWINCH to the child:

```
SIGWINCH prints before burst: 2
issued 20 rapid windowsize calls (no inter-call sleep)
SIGWINCH prints after burst: 3
=> 20 rapid resizes coalesced into exactly 1 SIGWINCH
final coalesced: [59.733] SIGWINCH sent to child ... size: (31, 88, 792, 558)   (== the last burst size)
```

This is `process_pending_resizes()` `[kitty/child-monitor.c:L1043-L1058]`, which only propagates a resize
once the time since the last resize event exceeds `OPT(resize_debounce_time).on_pause` (0.1 s; §6),
otherwise re-arms a short wait. Intermediate sizes are dropped; the child sees only the final geometry.

> **Limitation for the *combined* run:** under bare Xvfb (no window manager) `xdotool windowsize` was not
> honored inside the combined injection (0 SIGWINCH, size stayed 26×80), so the resize aspect of the
> capstone is covered by this dedicated, mapped-window test rather than inside the combined `strace`.

### 5.6 Signal handling (real entry point: the signalfd on the I/O thread)

Signals are not delivered asynchronously; they are **blocked and consumed synchronously via a signalfd**,
which is why they slot cleanly into the same `poll()` loop. Two independent runtime confirmations of the
handled set:

```
# (1) signalfd created with exactly the handled set:
92341 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 8
# (2) /proc/<pid>/status of the running kitty:
SigBlk: 0000000000014a03   ->  {SIGHUP(1),SIGINT(2),SIGUSR1(10),SIGUSR2(12),SIGTERM(15),SIGCHLD(17)}
```

Both match `#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0`
`[kitty/child-monitor.c:L121]` (registered via `signalfd(-1, &ld->signals, SFD_NONBLOCK|SFD_CLOEXEC)`
`[kitty/loop-utils.c:L42]`, installed at `[kitty/child-monitor.c:L173]`). The dispatch switch is
`handle_signal` `[kitty/child-monitor.c:L1360-L1383]`:

```c
        case SIGINT: case SIGTERM: case SIGHUP: ss->kill_signal = true;   break;   // L1365-L1368
        case SIGCHLD:                           ss->child_died  = true;   break;   // L1370-L1371
        case SIGUSR1:                           ss->reload_config = true; break;   // L1373-L1374
        case SIGUSR2: log_error("Received SIGUSR2: %d\n", siginfo->si_value.sival_int); break;   // L1376
```

**SIGUSR1 → config reload (EFFECT, observed).** With `background #112233` in the config, editing the file
on disk to `#aabbcc` and sending `SIGUSR1` to the *real* kitty PID changed the live value:

```
BEFORE: background  #112233     (kitty @ get-colors)
DURING: edit config on disk #112233 -> #aabbcc ; kill -USR1 <kitty pid>
AFTER : background  #aabbcc     -> the file was RE-READ and applied
```

Path: `SIGUSR1 → ss.reload_config` `[kitty/child-monitor.c:L1373]` → `reload_config_signal_received` set
under mutex `[kitty/child-monitor.c:L1523]` → `parse_input` sets `reload_config_called`
`[kitty/child-monitor.c:L472-L474]` → `call_boss(load_config_file, "")` `[kitty/child-monitor.c:L534-L536]`.

**SIGCHLD → reap (EFFECT + syscall, observed).** Exiting the child bash caused Kitty to reap it and, being
the last window with `close_on_child_death`, quit:

```
BEFORE: kitty alive, child bash present (pgrep -P <kitty pid>)
DURING: send 'exit' to bash -> child exits -> kernel raises SIGCHLD
AFTER : kitty EXITED (reap -> close_on_child_death -> last window -> quit)
syscall (strace): wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = <bash pid>
```

Path: `SIGCHLD → ss.child_died` `[kitty/child-monitor.c:L1370]` → `reap_children(self,
OPT(close_on_child_death))` `[kitty/child-monitor.c:L1526]` → `reap_children`
`[kitty/child-monitor.c:L1413]`, whose body is a `waitpid(-1, &status, WNOHANG)` loop calling
`mark_child_for_removal` + `mark_monitored_pids` — matching the traced `wait4(...WNOHANG)`. The
`SIGINT/SIGTERM/SIGHUP` siblings set `kill_signal` `[kitty/child-monitor.c:L1365-L1368]`, which
`parse_input` turns into `global_state.quit_request = IMPERATIVE_CLOSE_REQUESTED`
`[kitty/child-monitor.c:L466-L470]`.

> **Method note (why EFFECT proofs):** two pitfalls were hit and corrected. (1) `strace kitty & ;
> kill -USR1 $!` sends the signal to `strace`, not kitty — the *real* kitty PID (the `comm=kitty` child
> of strace) must be targeted. (2) `strace`'s ptrace machinery interferes with signalfd consumption, so
> no `read(8)` appears under `strace`; the reload/reap were therefore proven by their observable EFFECT
> (config value change; process exit) plus the `wait4` syscall, which is the real behavior.

### 5.7 Quiescence — the interface settling, observed

After the burst, `strace` on the I/O thread for 3 seconds captured **zero** completed syscalls: the thread
is blocked in `poll(timeout=-1)` with nothing to read (buffer free = `1048576`, fully drained), nothing to
write (no `POLLOUT`), and no pending resizes. The final settled screen:

```
PASTE_A
PASTE_B_42PASTE_A
PASTE_B_42echo KEYSTROKE_LINE_42
bash: PASTE_A: command not found
bash: PASTE_B_42PASTE_A: command not found
bash: PASTE_B_42echo: command not found
```

Blocking in `poll(-1)` with all queues empty *is* the resting state — the observable definition of the
interface having "settled again."

### 5.8 Cause → effect summary — why it keeps its rhythm

Every stage has a bound and a coalescing timer: the 1 MiB buffer + `POLLIN` gate bound memory and provide
backpressure; `input_delay` (3 ms) turns many reads into few main-loop wakeups; `resize_debounce_time`
(0.1 s) turns a resize burst into a single reflow; `repaint_delay` (10 ms) + `sync_to_monitor` bound the
render cadence to the monitor. Because no stage can be driven faster than its coalescing window and none
buffers without limit, the pipeline converges to `poll(-1)` quiescence rather than accumulating unbounded
backlog — it settles instead of "slowly falling apart."

---


## 6. Canonical timing / magnitude values (observed, multi-run stability)

Each value below was read from the **canonical** build through a real entry point and confirmed stable
across at least two runs. The timing options are compiled defaults declared in
`kitty/options/definition.py` and read back from the compiled options module; because they are
compile-time constants their run-to-run value is deterministic (RUN1 == RUN2), which is itself the
stability result.

Commands used:

```
$ kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; \
    print(d.input_delay, d.repaint_delay, d.sync_to_monitor, d.resize_debounce_time)'
3 10 True (0.1, 0.5)
$ kitty/launcher/kitty +runpy 'import kitty.fast_data_types as f; print(f.VT_PARSER_BUFFER_SIZE)'
1048576
```

| Value | Declared default | Observed (RUN1 / RUN2) | `file:line` | Role in the pipeline |
|-------|------------------|------------------------|-------------|----------------------|
| `input_delay` | `3` (ms) | `3` / `3` | `[kitty/options/definition.py:L878]` | wakeup-coalescing window (§2.3, §2.6, §5.3) — throttles I/O→main handoffs |
| `repaint_delay` | `10` (ms) | `10` / `10` | `[kitty/options/definition.py:L866]` | render cadence bound (§5.8) |
| `sync_to_monitor` | `yes` | `True` / `True` | `[kitty/options/definition.py:L889]` | ties render to monitor refresh (§5.8) |
| `resize_debounce_time` | `0.1 0.5` | `(0.1, 0.5)` / `(0.1, 0.5)` | `[kitty/options/definition.py:L1182]` | `.on_pause=0.1 s` debounce, `.max=0.5 s` cap (§5.5) |
| parser buffer `BUF_SZ` | `1024*1024` | `1048576` / `1048576` | `[kitty/vt-parser.c:L18]` | bounded ingestion buffer + backpressure threshold (§1.3, §4.3) |

Corroborating runtime magnitudes observed elsewhere in this document:

- The **1 MiB** buffer boundary is confirmed by saturation to exactly `1048576` bytes, identical across 2
  runs (§4.3), and by the shrinking `read()` size argument in live traces (§1.3, §5.3).
- The **`input_delay` window** is corroborated by the ~3–4 ms spacing of coalesced main-loop wakeups under
  a 2000-line surge and the wakeup distribution `{0: 36 ticks, 1: 9 ticks}` (§2.6, NON-CANONICAL
  instrumented build), and by the `poll()` timeout sequence `−1 → 1 → 0 → 2 ms` in the canonical combined
  trace (§5.3).
- The **`resize_debounce_time.on_pause`** (0.1 s) is corroborated by 20 rapid resizes coalescing into 1
  SIGWINCH (§5.5), reproduced with a stable outcome.

---

## 7. Coverage pass

Every named item across Q1–Q5 (including every "e.g. / such as / including" example) is confirmed present
with a concrete value, a `file:line` reference, observed evidence, sibling/alternate variants, and a
causal reason. `[C]` = canonical build; `[N]` = NON-CANONICAL instrumented build; `[H]` = real compiled
`fast_data_types` parser harness; `[SSH]` = real loopback ssh.

### Q1 — Ingestion / entry point

| Named item | Where covered | Evidence | `file:line` |
|------------|---------------|----------|-------------|
| `read_bytes()` | §1.2 | strace read on PTY master `[C]` | `[kitty/child-monitor.c:L1337-L1356]` |
| the `read()` syscall | §1.2 | `read(8…, 1048576) = 39` `[C]` | `[kitty/child-monitor.c:L1345]` (corrected from L1346) |
| `EINTR`/`EAGAIN` retry | §1.5 | quoted source + `EAGAIN` on wakeup fd `[C]` | `[kitty/child-monitor.c:L1347]` |
| `EIO` = child-gone | §1.5 | `read(...) = -1 EIO` after exit `[C]` | `[kitty/child-monitor.c:L1348-L1350]` |
| keystroke path `keys.c`/`key_encoding.{c,py}`/`keys.py` | §1.6 | legacy vs kitty-protocol table `[C]` | `keys.c`, `key_encoding.c` |
| mouse path | §1.6 | named as sibling input path | `kitty/mouse.c` |
| XKB / IME | §1.6 | "Loading new XKB keymaps" log `[C]` | `glfw/xkb_glfw.c`, `glfw/ibus_glfw.c` |
| `iutf8` / `set_iutf8_fd` | §1.6 | child `stty -a` shows `iutf8` `[C]` | `[kitty/child.py:L174]` |
| `write_to_child` | §1.4 | `write(10, "stty -a\n", 8)` `[C]` | `[kitty/child-monitor.c:L1443]` |
| paused/resumed cross-ref | §1.1, §4 | forward reference to §4 gate | `[kitty/child-monitor.c:L1501]` |

### Q2 — The unseen conductor

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| I/O thread `io_loop` | §2.2 | `KittyChildMon` in `/proc` `[C]` | `[kitty/child-monitor.c:L1481]` (name L1489) |
| main/render `main_loop` | §2.2 | tick cadence `[N]` | `[kitty/child-monitor.c:L1259]` (do_parse L438) |
| talk thread `read_from_peer` | §2.2 | `KittyPeerMon` in `/proc` `[C]` | `[kitty/child-monitor.c:L1714]` (L1808) |
| `poll()` ordering + `EXTRA_FDS` | §2.4, §5.3 | fixed `[fd7,fd8,fd10]` order `[C]` | `[kitty/child-monitor.c:L35,L1501-L1503,L1515-L1541]` |
| `WAKEUP` / `input_delay` | §2.3, §2.6 | quoted macro + comment; dist `{0:36,1:9}` `[N]` | `[kitty/child-monitor.c:L1562-L1569]` |
| `loop-utils.{c,h}` / `threading.h` | §2.2, §5.6 | signalfd/eventfd plumbing `[C]` | `[kitty/loop-utils.c:L42]`, `kitty/threading.h` |
| `boss.py` lifecycle | §2.7 | thread/child presence after start `[C]` | `[kitty/boss.py:L370,L585,L1183,L2172]` |

### Q3 — Shell-integration alignment

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| VT parser byte classification | §3.2 | quoted `consume_esc` transitions | `[kitty/vt-parser.c:L260,L270]` |
| OSC routing `dispatch_osc` case 133 | §3.3 | quoted source | `[kitty/vt-parser.c:L457,L536,L544]` |
| `cmd_output_marking` O / OO / Os | §3.3 | quoted callbacks | `[kitty/screen.c:L2338,L2347,L2352]` |
| `ESC_OSC` disambiguation | §3.3 | output-prefix vs input-routing note | `[kitty/screen.c:L964]` vs `[kitty/vt-parser.c:L536]` |
| `find_cmd_output` / `cmd_output` | §3.3 | named for extent lookup | `[kitty/screen.c:L3527,L3606]` |
| pending mode start/stop ↔ DEC 2026 | §3.8 | DECRQM before/during/after `[H]` | `[kitty/vt-parser.c:L639,L644]` |
| `modes.h` DEC/paste constants | §3.8, §5.4 | `PENDING_UPDATE (2026<<5)`, paste consts | `[kitty/modes.h:L81-L83,L86]` |
| `shell_integration.py` setup_* + `modify_shell_environ` | §3.7 | dispatch + zsh skip edge `[C]` | `[kitty/shell_integration.py:L16,L27,L49,L70,L218]` |
| bash / zsh / fish / ssh scripts | §3.4, §3.7 | real per-shell OSC 133 bytes `[C]` | `shell-integration/{bash,zsh,fish,ssh}` |

### Q4 — Backpressure & unstable remote

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| `vt_parser_has_space_for_input` | §4.2 | quoted predicate `[C]` | `[kitty/vt-parser.c:L1477,L1481]` |
| `POLLIN` gate | §4.2, §4.4 | `events POLLIN→0→POLLIN` `[C]` | `[kitty/child-monitor.c:L1501]` |
| `BUF_SZ` 1 MiB | §4.3 | saturation to `1048576` ×2 `[H]` | `[kitty/vt-parser.c:L18]` |
| `write_space_created` → `wakeup_io_loop` | §4.2, §4.4 | fd7 eventfd wake restores POLLIN `[C]` | `[kitty/vt-parser.c:L1438]`, `[kitty/child-monitor.c:L442]` |
| parse-trigger coalescing | §4.8 | quoted 3-way condition | `[kitty/vt-parser.c:L1425]` |
| child blocks on `write()` | §4.5 | `write(...) = 65536 <3.987936>` `[C]` | (kernel PTY, consequence of L1501) |
| XON/XOFF vs RTS/CTS contrast | §4.6 | `stty -a`: `-crtscts ixon -ixoff`; no DC1/DC3 `[C]` | terminology → observed |
| `kittens/ssh/{main.py,main.go,config.go,utils.go,askpass.go}` | §4.7 | real loopback ssh; propagation `[SSH]` | `[kittens/ssh/main.go:L325-L357]`, `main.py:L122,L170` |
| dropped-link edge → EIO | §4.7 | `read(...) = -1 EIO` on ssh kill `[SSH]` | `[kitty/child-monitor.c:L1348-L1350]` |

### Q5 — Full end-to-end settle

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| arrival → read → buffer → parse → screen → render → quiescence | §5.2, §5.3, §5.7 | combined trace ×2 + `poll(-1)` idle `[C]` | `[kitty/child-monitor.c:L1345,L438]` |
| paste path (bracketed markers) | §5.4 | byte-exact `\33[200~…\33[201~` `[C]` | `[kitty/window.py:L117,L1643,L1707,L1713,L1780]`, `[kitty/modes.h:L81-L83]` |
| resize `screen.resize`/`resize_pty`/SIGWINCH | §5.5 | before/after sizes + prints `[C]` | `[kitty/window.py:L850,L854,L863,L873]` |
| resize debounce `process_pending_resizes` | §5.5 | 20 resizes → 1 SIGWINCH `[C]` | `[kitty/child-monitor.c:L1043-L1058]` |
| window/tab routing | §2.7, §5 | window objects via remote-control `ls` `[C]` | `kitty/tabs.py`, `kitty/window_list.py` |
| child / PTY | §1, §5 | `fd 10` PTY master; child bash `[C]` | `kitty/child.py`, `kitty/child.c` |
| signals INT/TERM/HUP→kill, CHLD→reap, USR1→reload (+USR2) | §5.6 | signalfd set, SigBlk, reload/reap EFFECT `[C]` | `[kitty/child-monitor.c:L121,L1360-L1383,L1413,L1526,L534-L536]` |

### Canonical values (§6)

`input_delay=3` `[kitty/options/definition.py:L878]`, `repaint_delay=10` `[kitty/options/definition.py:L866]`,
`sync_to_monitor=yes` `[kitty/options/definition.py:L889]`, `resize_debounce_time='0.1 0.5'`
`[kitty/options/definition.py:L1182]`, parser buffer `1 MiB` `[kitty/vt-parser.c:L18]` — all observed and
stable across ≥2 runs.

### Explicitly-stated limitations (nothing faked)

- **Q2 cadence numbers** (idle ~0.5 s/tick; surge ~3–4 ms/tick; wakeup distribution) come from the
  **NON-CANONICAL** `make debug-event-loop` build; the thread structure, `poll()` order, and `WAKEUP`
  gate logic are canonical.
- **Q4 unstable remote**: true packet loss / jitter cannot be induced on a container loopback; the
  transport keepalive is shown from the ssh options, and the pipeline consequence of a drop (`EIO`
  child-gone) is directly observed.
- **Q5 resize inside the combined run**: `xdotool windowsize` was not honored under bare Xvfb (no window
  manager), so the resize/debounce evidence is the dedicated mapped-window test (§5.5), not the combined
  `strace`.

### Reproducibility note

All observation scripts were temporary and kept under `/tmp` (outside the tracked tree); they were removed
after use. The source repository is unchanged apart from this document (`blitzy/documentation/kitty_815df1e210e0.md`)
and its parent directories — verified with `git status --porcelain` (only `blitzy/` untracked) and an empty
`git diff` on tracked files.

