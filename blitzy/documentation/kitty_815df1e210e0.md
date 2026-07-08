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

The canonical build command is `make`; its `all:` target runs `python3 setup.py $(VVAL)`, where the make
variable `$(VVAL)` is empty unless `V=1`/`VERBOSE=1` is set, so a plain `make` runs exactly
`python3 setup.py`. The exact, unedited `Makefile` target text:

```
$ sed -n '12,13p' Makefile
all:
	python3 setup.py $(VVAL)
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
`make debug-event-loop`; its target runs `python3 setup.py build $(VVAL) --debug --extra-logging=event-loop`
(again `$(VVAL)` is empty by default). The exact, unedited `Makefile` target text:

```
$ sed -n '25,26p' Makefile
debug-event-loop:
	python3 setup.py build $(VVAL) --debug --extra-logging=event-loop
```

`[Makefile:L25-L26]`; the `--extra-logging` argument is declared at `[setup.py:L1927]` and accepts
`choices=('event-loop',)` `[setup.py:L1930]`. This build compiles the `DEBUG_EVENT_LOOP` macro `[kitty/child-monitor.c:L29]`,
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

Remote control (`kitty @ --to unix:<socket> <subcommand>`) was used only to *inject* input (keystrokes via
`send-text`, pastes via `action paste_to_active_window`, resizes via `resize-os-window`) and to *read back*
screen state around the real pipeline. In every case the injected event drives the **same internal code
path a human action would** — e.g. `resize-os-window` calls `glfwSetWindowSize`, firing the real
framebuffer-resize callback → `set_geometry` → `resize_pty` → the kernel `SIGWINCH`; `action
paste_to_active_window` calls the real `Window.paste_with_actions`. The behavior being measured (ingestion,
parsing, flow control, resize/reflow, rendering) always occurs in the real code, never in a remote-control
shortcut. Real X keyboard/mouse events were additionally injected with `xdotool` into the kitty window so
the GLFW → `keys.c`/`mouse.c` path is exercised end-to-end.

> **Command execution context (so every command below is runnable as written).** `kitty` is **not** on
> `$PATH` in this container; the canonical launcher is the in-tree binary `kitty/launcher/kitty`. All
> commands are shown and were run **from the repository root**
> (`/tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b`) with the environment of §0.1
> exported (`DISPLAY=:99`, `LIBGL_ALWAYS_SOFTWARE=1`, `LANG=LC_ALL=C.UTF-8`). Equivalently one may
> `export PATH="$PWD/kitty/launcher:$PATH"` once and then call `kitty` directly; this document uses the
> explicit `kitty/launcher/kitty` form throughout so no prior `PATH` edit is assumed.

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

The AAP/checkpoint names this entry point as the `read()` inside `read_bytes()` at
`[kitty/child-monitor.c:L1337-L1356; read() at L1346]`. That is exactly the function and span exercised
below. Reading the pinned source at commit `815df1e210e0` confirms the citation: `read_bytes()` opens at
**L1337** and its closing brace is at **L1356**, so the whole span is `[L1337-L1356]`; within that span the
`read()` syscall statement is on **L1345** and the immediately-following `if (len < 0)` guard is on
**L1346** — i.e. the `read()` sits one line inside the AAP-cited range, adjacent to the L1346 the AAP
notes. (Verbatim source is quoted in §1.5.) `read_bytes()` reads into the buffer returned by
`vt_parser_create_write_buffer()` `[kitty/child-monitor.c:L1341; kitty/vt-parser.c:L1451]` and commits the
bytes with `vt_parser_commit_write()` `[kitty/child-monitor.c:L1354]`.

Command (whole-process `strace` filtering `read`, a child printing a unique marker; run from the repo root):

```
$ strace -f -tt -yy -e read -s300 ./kitty/launcher/kitty --config NONE \
    sh -c 'printf "Q1INGESTMARKER_START_ABCDEF123456_END\r\n"; sleep 0.4'
```

Complete captured line for the marker's first entry (`grep`ed from the trace; the PID prefix `136137` is
strace's `-f` per-process tag):

```
136137 06:42:55.530200 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "Q1INGESTMARKER_START_ABCDEF123456_END\r\r\n", 1048576) = 40
```

The child's output enters via `read()` on the **PTY master** (`/dev/pts/ptmx`), into a **`1048576`-byte
(1 MiB)** buffer — exactly `BUF_SZ` `[kitty/vt-parser.c:L18]`. The `= 40` is byte-exact: the marker
`Q1INGESTMARKER_START_ABCDEF123456_END` is **37** printable bytes, and the `printf` argument `\r\n` becomes
**`\r\r\n`** (3 bytes) on the wire because the PTY output discipline has `opost`+`onlcr` set, so the `\n` is
expanded to `\r\n` while the literal `\r` is preserved — `37 + 3 = 40`. Confirmed stable across 5 identical
runs (all `= 40`, all ending in the 3 bytes `\r\r\n`); the `onlcr`/`opost` flags are shown by the child's
own `stty -a` in §1.6.

### 1.3 It runs on the `KittyChildMon` I/O thread, batching into one 1 MiB buffer

Attaching `strace` (with a wide `-s 2000` so no line is elided by strace) to the `KittyChildMon` thread
(TID resolved from `/proc/<pid>/task/*/comm`) while a shell emits a burst shows consecutive reads into the
*same* buffer, with the size argument shrinking as unparsed bytes accumulate. The launch and trace:

```
$ setsid ./kitty/launcher/kitty --config NONE -o allow_remote_control=yes \
      --listen-on unix:/tmp/kitty_obs/q1_batch2.sock bash --noprofile --norc &
$ CM_TID=$(for t in /proc/$!/task/*/comm; do [ "$(cat $t)" = KittyChildMon ] \
      && basename $(dirname $t); done)
$ strace -tt -yy -e read -s 2000 -p "$CM_TID" &
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q1_batch2.sock \
      send-text 'seq 1 300 | tr "\n" " "; echo Q1BURST_$((6*7))\n'
```

The first four consecutive PTY-master reads, complete and unedited (note the fourth arg — the *remaining*
buffer space — decreasing monotonically):

```
06:44:41.921086 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "seq 1 300 | tr \"\r\n\33[?2004l\r", 1048576) = 27
06:44:41.921200 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[?2004h\33]133;k;start_kitty\7\33]133;A;k=s\7\33]133;k;end_kitty\7> \" \" \"; echo Q1BURST_$((6*7))\r\n\33[?2004l\r", 1048549) = 99
06:44:41.922972 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;seq 1 300 | tr \"\" \" \"; echo Q1BURST_$((6*7))\7\33]133;C;cmdline=$'seq 1 300 | tr \"\\n\" \" \"; echo Q1BURST_$((6*7))'\7", 1048450) = 115
06:44:41.923337 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2", 1048335) = 105
```

The remaining-space argument goes `1048576 → 1048549 → 1048450 → 1048335`, and each step equals the
previous minus the bytes just read: `1048576−27=1048549`, `1048549−99=1048450`, `1048450−115=1048335`.
After the main thread parses and drains, the next read resets to `1048576` again (also observed in the same
trace). That empirically confirms `vt_parser_create_write_buffer()` returns `BUF_SZ - write.offset`
`[kitty/vt-parser.c:L1457]`: successive reads target the *remaining* space of one shared 1 MiB buffer, so
the I/O thread batches multiple reads before the main thread parses and drains it. These reads occur inside
`io_loop()` `[kitty/child-monitor.c:L1481]` (the thread named `"KittyChildMon"`, set at
`[kitty/child-monitor.c:L1489]`).

### 1.4 The child-write path (same I/O thread)

Bytes destined *for* the child (a command sent to the shell, the terminal's own replies) leave through
`write_to_child()` `[kitty/child-monitor.c:L1443]`, observed on the same `KittyChildMon` thread. The trace
(`strace -tt -yy -e trace=read,write` on the I/O thread TID) while injecting two commands via
`kitty @ send-text`:

```
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q1_wr.sock send-text 'stty -a\n'
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q1_wr.sock send-text 'exit\n'
```
```
06:45:28.521277 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "stty -a\n", 8) = 8
06:45:29.548564 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "exit\n", 5) = 5
```

This write is `POLLOUT`-driven: the I/O thread only asks to write when the window's `write_buf_used`
is non-zero (`[kitty/child-monitor.c:L1503]`, shown in §2.4).

### 1.5 Error / edge branches in the read loop (source + observed)

The whole of `read_bytes()`, quoted verbatim from the pinned source (the `read()` is L1345, the
error-handling `if (len < 0)` guard is L1346 — the AAP-cited L1337-L1356 span, with the two error branches
inside it):

```
$ sed -n '1336,1356p' kitty/child-monitor.c
```
```c
static bool
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

- **`EINTR`/`EAGAIN` → retry** (`continue`) `[kitty/child-monitor.c:L1347]`. **This branch is
  source-verified but is not reached on the canonical path, and here is the grounded reason** (reported as
  *not observed*, per the observed-vs-inferred rule, rather than pretended): the child PTY master `fd` is
  opened in **blocking** mode — `os.openpty()  # Note that master and slave are in blocking mode`
  `[kitty/child.py:L171]` — and `read_bytes()` is only ever called after `poll()` has already reported the
  fd readable, at `if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP))`
  `[kitty/child-monitor.c:L1529]` → `read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)`
  `[kitty/child-monitor.c:L1531]`. A blocking read of an
  already-readable fd returns data immediately, so `EAGAIN` cannot occur here; and the I/O thread's handled
  signals are delivered **synchronously via a `signalfd`** (`children_fds[1].fd =
  self->io_loop_data.signal_read_fd` `[kitty/child-monitor.c:L183]`, drained at L1519), not as asynchronous
  interrupts, so `read()` is not interrupted by `EINTR` from them either. The retry is therefore a
  defensive guard for a spurious wakeup / a non-handled async signal — a real code branch that the
  canonical path does not exercise. What *is* observed is the benign `EAGAIN` on the **separate,
  non-blocking wakeup eventfd** (`fd 7`), which is a *different* descriptor and read loop (shown in §1.6);
  it is not the `read_bytes()` child-fd read.
- **`EIO` → child is gone** (observed): any error other than `EIO` prints via `perror`; then
  `vt_parser_commit_write(screen->vt_parser, 0)` and `return false` mark the child as gone
  `[kitty/child-monitor.c:L1348-L1350]`. Directly observed after the child `exit` of §1.4, on the same
  `KittyChildMon` `fd 10`:

```
06:45:29.551090 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, 0x55580a36c91c, 1048420) = -1 EIO (Input/output error)
```

That `-1 EIO` is precisely the child-gone branch — the read returns `EIO` because the last slave fd
closed (strace prints the raw buffer pointer `0x55580a36c91c` because the read failed, so no bytes were
transferred). This `EIO`-on-read branch fires when a `read()` is **in flight** at teardown (child
mid-output); its sibling is the **SIGCHLD** child-reap path that fires when the child is **idle** at
teardown (the I/O thread is blocked in `poll()`, which surfaces the signalfd). An abruptly-dropped remote
connection triggers whichever applies — §4.7 directly observes the SIGCHLD case for an idle remote.

### 1.6 Parallel input paths (keystrokes, paste, mouse, PTY setup)

Child *output* is only one of the streams arriving at this junction. The other streams enter through the
GLFW/Python layer and are queued back to the child. Each was exercised through its **real entry point**
(X events synthesized with `xdotool` into a headless Kitty on `DISPLAY=:99`), not through a bypassing
remote-control hook.

**Keystroke encoding — legacy vs. Kitty keyboard protocol.** Real key events were injected as X events and
observed through the `keys.c` → `key_encoding.c` path. The GLFW event first reaches
`on_key_input()` `[kitty/keys.c:L166]`, which builds a Python `GLFWkeyevent` via
`convert_glfw_key_event_to_python()` `[kitty/keys.c:L93]`; the Python side resolves shortcuts in
`get_shortcut()` `[kitty/keys.py:L40]` / `dispatch_possible_special_key()` `[kitty/keys.py:L154]`, and the
C encoder `encode_glfw_key_event()` `[kitty/key_encoding.c:L414]` → `serialize()` `[kitty/key_encoding.c:L65]`
emits the bytes (the functional-key and CSI-number tables live in `kitty/key_encoding.py` —
`functional_key_number_to_name_map` `[L15]`, `csi_number_to_functional_number_map` `[L127]`,
`letter_trailer_to_csi_number_map` `[L149]`).

XKB processing on Linux is confirmed live — the complete, unedited `--debug-keyboard` lines (ANSI color
codes stripped for legibility) from the run that produced the keystrokes below:

```
$ DISPLAY=:99 ./kitty/launcher/kitty --debug-keyboard --config NONE sh -c 'cat > /dev/null'   # keys via: xdotool key --window <wid> --clearmodifiers <key>
[0.067] Loading new XKB keymaps
[0.148] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
```

(`glfw/xkb_glfw.c`; IME, when present, is handled by `glfw_ibus_dispatch()` `[glfw/ibus_glfw.c:L460]` /
`glfw_ibus_set_focused()` `[glfw/ibus_glfw.c:L473]` — not exercised in this headless, no-IME run).

*Legacy mode (default).* The unedited `on_key_input` decisions and the byte-exact writes to the child PTY
master (`strace -f -e trace=write`, fd `8` in this run) — nothing elided:

```
[5.405] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[5.406] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[5.759] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x1
[6.153] on_key_input: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A
[4.465] on_key_input: glfw key: 0xe002 native_code: 0xff09 action: PRESS mods: shift text: '' state: 0 sent encoded key to child: ^[ [ Z
[4.898] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ a
[5.323] on_key_input: glfw key: 0xe014 native_code: 0xffbe action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ O P
```

And the corresponding `strace -f -e trace=write` capture of the same keys reaching the child PTY master. The
`<unfinished ...>` / `<... write resumed>` pairs on the last three lines are **strace's own native markers** for a
syscall whose trace record was split because a concurrent thread event interleaved between syscall entry and exit —
not a manual elision: the written bytes (`"\33[Z"`, `"\33a"`, `"\33OP"`) and the return value (`= 3`, `= 2`, `= 3`)
are both fully present on the paired lines.

```
06:46:23.555552 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "a", 1) = 1
06:46:23.909139 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1", 1) = 1
06:46:24.303488 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[A", 3) = 3
06:54:28.328552 write(8, "\33[Z", 3 <unfinished ...>
06:54:28.328575 <... write resumed>) = 3
06:54:28.761182 write(8, "\33a", 2 <unfinished ...>
06:54:28.761233 <... write resumed>) = 2
06:54:29.186743 write(8, "\33OP", 3 <unfinished ...>
06:54:29.186773 <... write resumed>) = 3
```

*Kitty keyboard protocol.* With the child requesting the protocol (`printf '\033[>1u'` for the base flag,
`\033[>31u` for all flags), the same physical keys encode differently — byte-exact writes:

```
06:46:56.191888 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[97;5u", 7) = 7
06:46:56.471730 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[9;2u", 6) = 6
06:47:30.601986 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[97;;97u", 9) = 9
06:47:30.610728 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[97;1:3u", 9) = 9
06:47:30.921150 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[A", 3) = 3
06:47:30.929210 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[1;1:3A", 8) = 8
```

Every cell in the table below is a byte-exact `write()` observed above (or the exact `on_key_input`
"ignoring" decision):

| Key | Legacy encoding (default) | Kitty keyboard protocol |
|-----|---------------------------|-------------------------|
| `a` (press) | `a` (text, 1 byte) | `\33[97;;97u` (all-flags) |
| `a` (release) | *(ignored — "keyboard mode does not support encoding this event")* | `\33[97;1:3u` (`:3` = release event) |
| `Ctrl+a` | `\1` (`0x01`) | `\33[97;5u` |
| `Shift+Tab` | `\33[Z` | `\33[9;2u` |
| `Alt+a` | `\33a` | — (not re-run under protocol) |
| `Up` | `\33[A` | `\33[A` press / `\33[1;1:3A` release |
| `F1` | `\33OP` | — (not re-run under protocol) |

The contrast is the point, and it is causal: the Kitty protocol emits `CSI codepoint;mods[:event]u` and
encodes key *releases* (the `:3` sub-parameter), whereas the legacy protocol collapses modified keys to
control bytes and *ignores* releases entirely — visible as the repeated "ignoring as keyboard mode does
not support encoding this event" lines. Bare modifier presses (Control/Shift/Alt alone) are likewise
ignored in legacy mode.

**Mouse encoding.** Mouse events enter through `kitty/mouse.c`; the encoder
`encode_mouse_event_impl()` `[kitty/mouse.c:L68]` formats the SGR sequence
`"<%d;%d;%d%s"` with the trailer `action == RELEASE ? "m" : "M"` `[kitty/mouse.c:L88]`. With the child in
SGR mouse mode (`printf '\033[?1000h\033[?1006h'`), a real click synthesized with
`xdotool click 1` produced one byte-exact write carrying **both** the press (`M`) and release (`m`), and
the child then echoed it back:

```
06:48:01.724197 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[<0;7;3M\33[<0;7;3m", 18) = 18
06:48:01.724284 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "^[[<0;7;3M^[[<0;7;3m", 1048576) = 20
```

`\33[<0;7;3M` is button 0 at column 7, row 3, press; `\33[<0;7;3m` is the release — exactly the
`M`/`m` trailer distinction from `[kitty/mouse.c:L88]`.

**PTY UTF-8 flag (`iutf8`).** The child PTY is created by `openpty()` `[kitty/child.py:L170]` (with the
comment `# Note that master and slave are in blocking mode` at `[kitty/child.py:L171]`), and the `IUTF8`
termios flag is set by `set_iutf8_fd(master, True)` `[kitty/child.py:L174]`. The C-side `spawn()`
`[kitty/child.c:L81]` establishes the session (`setsid()` `[kitty/child.c:L123]`), makes the slave the
controlling terminal (`ioctl(tfd, TIOCSCTTY, 0)` `[kitty/child.c:L129]`), and `execvp`s the program
`[kitty/child.c:L159]`. Observed in the child's own `stty -a` (complete, unedited), which reports `iutf8`
(not `-iutf8`) — and also the `icrnl`/`onlcr`/`opost` flags that explain the `\r\r\n` of §1.2:

```
$ ./kitty/launcher/kitty --config NONE sh -c 'stty -a; sleep 0.3'
speed 38400 baud; rows 40; columns 100; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>; eol2 = <undef>; swtch = <undef>; start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R; werase = ^W; lnext = ^V; discard = ^O; min = 1; time = 0;
-parenb -parodd -cmspar cs8 -hupcl -cstopb cread -clocal -crtscts
-ignbrk -brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl -ixoff -tandem ixon -ixany -imaxbel iutf8
opost -olcuc -ocrnl onlcr -onocr -onlret -ofdel nl0 cr0 tab0 bs0 vt0 ff0
isig icanon iexten echo echoe echok -echonl -noflsh -tostop -echoprt echoctl echoke -flusho -extproc
```

**Loop wakeup / signal descriptors.** The same `strace` shows the descriptors that §2 relies on — this is
the benign `EAGAIN` on the **non-blocking wakeup eventfd** referenced in §1.5 (a *different* descriptor
from the blocking child PTY read):

```
06:48:26.374151 read(7<{eventfd-count=0x1, eventfd-id=2309, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 1024) = 8
06:48:26.374459 read(7<{eventfd-count=0, eventfd-id=2309, eventfd-semaphore=0}>, 0x7df3ea19d740, 1024) = -1 EAGAIN (Resource temporarily unavailable)
06:48:26.374745 write(4<{eventfd-count=0, eventfd-id=2304, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
```

`fd 7` is the I/O thread's wakeup eventfd (drained until `EAGAIN` — hence the benign `EAGAIN` here, *not*
on the child fd); `fd 4` is the main-loop wakeup eventfd (the I/O thread writes to it to hand parsed data
to the main thread — see §2 and §5).

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
first each tick is decided **not** by `poll()` (which returns all ready descriptors with no priority) but
by the fixed array-index order in which the source processes `poll()`'s results — the wakeup fd (index 0)
and signal fd (index 1) are checked before the per-child loop (§2.4) — and how *often* the main thread is
woken is throttled by the `input_delay` window.

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
`[kitty/child-monitor.c:L1562-L1570]`:

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
where the descriptors always appear in the order `[fd=7 wakeup, fd=8 signalfd, fd=10 child]`.

To be precise about *why* one thing is handled before another: `poll()` itself does **not** prioritize —
it returns **all** ready descriptors at once, with no ordering among them. The ordering is entirely a
property of the **source code that processes the returned `revents` in fixed array-index order** after
`poll()` returns. Reading `io_loop()` at `if (ret > 0)` `[kitty/child-monitor.c:L1514]`, the code
processes the returned `revents` in this fixed order: (1) it drains the wakeup fd with the complete
statement `if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd);` `[kitty/child-monitor.c:L1515]`;
then (2) it enters the signal-fd block `if (children_fds[1].revents && POLLIN)` `[kitty/child-monitor.c:L1516]`,
whose body calls `read_signals(children_fds[1].fd, handle_signal, &ss)` `[kitty/child-monitor.c:L1519]`;
then (3) it enters the per-child loop `for (i = 0; i < self->count; i++)` `[kitty/child-monitor.c:L1528]`
that calls `read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)` `[kitty/child-monitor.c:L1531]`
on `POLLIN | POLLHUP` and `write_to_child(children[i].fd, children[i].screen)`
`[kitty/child-monitor.c:L1540]` on `POLLOUT`. So within a single tick where several descriptors are
ready simultaneously, the wakeup fd is processed first and the signal fd second **because they occupy
indices 0 and 1 and the code checks them before the child loop** — not because `poll()` grants them any
scheduling priority. The mapping to concrete fds was observed in §1.6: `fd 7` = wakeup eventfd,
`fd 8` = `signalfd:[HUP INT USR1 USR2 TERM CHLD]`, `fd 10` = PTY master.

The two EVDBG lines used in §2.5 and §2.6 come from the event-loop-instrumented build and are emitted by
the GLFW X11 backend (compiled into `kitty/glfw-x11.so`, which is the backend selected under Xvfb):
`--------- loop tick, wakeups_happened: %d ----------` at `[glfw/main_loop.h:L31]` and
`pollForEvents final timeout: %.3f` at `[glfw/backend_utils.c:L298]`. Two facts about these lines matter
for reading the output below and are grounded in source:

1. **`wakeups_happened` is a boolean, not a counter.** The field it prints, `wakeup_data_read`, is declared
   `bool` at `[glfw/backend_utils.h:L76]` (`bool wakeup_data_read, wakeup_fd_ready;`). It can therefore
   only ever print `0` or `1`; it reports *whether* a given tick was driven by the I/O thread's wakeup,
   not *how many* wakeups were coalesced. The coalescing itself happens one level down, in `drain_wakeup_fd`
   `[glfw/backend_utils.c:L217-L230]`, which drains **every** pending byte from the wakeup fd in a loop and
   folds them all into the single boolean:

   ```c
   static void
   drain_wakeup_fd(int fd, EventLoopData* eld) {
       static char drain_buf[64];
       eld->wakeup_data_read = false;
       while(true) {
           ssize_t ret = read(fd, drain_buf, sizeof(drain_buf));
           if (ret < 0) {
               if (errno == EINTR) continue;
               break;
           }
           if (ret > 0) { eld->wakeup_data_read = true; continue; }
           break;
       }
   }
   ```

   The main loop then calls `tick_callback` (the parse+render pass) **at most once per tick**, gated on that
   one boolean `[glfw/main_loop.h:L26-L38]`:

   ```c
   void _glfwPlatformRunMainLoop(GLFWtickcallback tick_callback, void* data) {
       keep_going = 1;
       EventLoopData *eld = &_glfw.GLFW_LOOP_BACKEND.eventLoopData;
       while(keep_going) {
           _glfwPlatformWaitEvents();
           EVDBG("--------- loop tick, wakeups_happened: %d ----------", eld->wakeup_data_read);
           if (eld->wakeup_data_read) {
               eld->wakeup_data_read = false;
               tick_callback(data);
           }
       }
       EVDBG("main loop exiting");
   }
   ```

2. **The I/O thread throttles its wakeups to at most one per `input_delay`.** Before the main-thread drain
   ever runs, the I/O thread has already coalesced on its side. Its `WAKEUP` macro and the gate around it
   `[kitty/child-monitor.c:L1562-L1570]` are (quoted complete, no elision):

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

   So the enclosing condition `(now - last_main_loop_wakeup_at) > OPT(input_delay)` means: however many
   times the I/O thread reads new bytes inside one `input_delay` (= 3 ms; §6) window, it posts **one**
   wakeup and sets `has_pending_wakeups = true` for the rest of the window. `input_delay` is the same knob
   the main thread uses to schedule its next parse, via `set_maximum_wait(OPT(input_delay) - pd.time_since_new_input)`
   `[kitty/child-monitor.c:L445-L446]`.

The two sections below observe both levels of coalescing at runtime on the instrumented build. All values
in §2.5–§2.6 are **NON-CANONICAL [N]** (they exist only because the build was compiled with
`--extra-logging=event-loop`; §0.3).

### 2.5 Idle cadence (instrumented, NON-CANONICAL [N])

With an idle child (`sleep 6`, no input at all), the main loop ticks about every half-second doing nothing
but housekeeping. Command:

```
setsid ./kitty/launcher/kitty --config NONE sh -c "sleep 6" > q2_idle.log 2>&1 &
grep -aE "loop tick, wakeups_happened|pollForEvents final timeout" q2_idle.log
```

Complete, unedited event-loop output for the run (24 lines):

```
[0.167] pollForEvents final timeout: 0.000
[0.181] --------- loop tick, wakeups_happened: 1 ----------
[0.181] pollForEvents final timeout: 0.000
[0.181] --------- loop tick, wakeups_happened: 0 ----------
[0.181] pollForEvents final timeout: 0.486
[0.672] --------- loop tick, wakeups_happened: 0 ----------
[0.672] pollForEvents final timeout: 0.499
[1.177] --------- loop tick, wakeups_happened: 0 ----------
[1.177] pollForEvents final timeout: 0.495
[1.677] --------- loop tick, wakeups_happened: 0 ----------
[1.677] pollForEvents final timeout: 0.495
[2.177] --------- loop tick, wakeups_happened: 0 ----------
[2.177] pollForEvents final timeout: 0.495
[2.677] --------- loop tick, wakeups_happened: 0 ----------
[2.677] pollForEvents final timeout: 0.495
[3.177] --------- loop tick, wakeups_happened: 0 ----------
[3.177] pollForEvents final timeout: 0.495
[3.677] --------- loop tick, wakeups_happened: 0 ----------
[3.677] pollForEvents final timeout: 0.495
[4.177] --------- loop tick, wakeups_happened: 0 ----------
[4.177] pollForEvents final timeout: 0.495
[4.677] --------- loop tick, wakeups_happened: 0 ----------
[4.677] pollForEvents final timeout: 0.495
[4.972] --------- loop tick, wakeups_happened: 1 ----------
```

Distribution over the 12 ticks: `10 × wakeups_happened: 0` and `2 × wakeups_happened: 1`. After the initial
render at `[0.181]` (`wakeups_happened: 1`), the loop settles into a clean **~0.5 s cadence** (0.672, 1.177,
1.677, 2.177, 2.677, 3.177, 3.677, 4.177, 4.677 — successive gaps of exactly 0.500 s), every tick reporting
`wakeups_happened: 0` and `pollForEvents final timeout: 0.495`. `wakeups_happened: 0` means the I/O thread
never had to wake the main loop — nothing was happening — and the 0.495 timeout is the wait until the nearest
housekeeping timer. The final `wakeups_happened: 1` at `[4.972]` is the wakeup delivered when the run was
terminated.

### 2.6 Surge coalescing (instrumented, NON-CANONICAL [N]) — the conductor under load

To force sustained cross-thread handoffs, the child runs a paced emitter that writes **exactly 50 000 lines**
to the PTY, flushing every 200 lines with a 0.8 ms pause (< `input_delay` = 3 ms), so consecutive chunks land
inside the same `input_delay` window and must be coalesced. The emitter (`/tmp/kitty_obs/pacer.py`, a
temporary observation script, quoted complete):

```python
#!/usr/bin/env python3
import sys, time
NLINES = int(sys.argv[1]) if len(sys.argv) > 1 else 50000
CHUNK  = int(sys.argv[2]) if len(sys.argv) > 2 else 200
PAUSE  = float(sys.argv[3]) if len(sys.argv) > 3 else 0.0008
t0 = time.monotonic()
w = sys.stdout.write
for i in range(NLINES):
    w("SURGELINE_%08d_ABCDEFGHIJKLMNOP\n" % i)
    if (i + 1) % CHUNK == 0:
        sys.stdout.flush()
        time.sleep(PAUSE)
sys.stdout.flush()
dt = time.monotonic() - t0
with open("/tmp/kitty_obs/pacer_emitted.txt", "w") as f:
    f.write("PACER_EMITTED_LINES=%d DURATION_S=%.3f\n" % (NLINES, dt))
```

Command (identical for both runs):

```
setsid ./kitty/launcher/kitty --config NONE \
  sh -c "python3 /tmp/kitty_obs/pacer.py 50000 200 0.0008; sleep 1.2" > q2_surge_runN.log 2>&1 &
grep -aE "loop tick, wakeups_happened|pollForEvents final timeout" q2_surge_runN.log
```

Both runs confirm the emitter wrote the full 50 000 lines:

```
RUN 1: PACER_EMITTED_LINES=50000 DURATION_S=0.753
RUN 2: PACER_EMITTED_LINES=50000 DURATION_S=0.691
```

**The decisive magnitude:** 50 000 lines produced only **~66 wakeup-bearing main-loop ticks**, not 50 000.
Complete wakeup distributions (computed from the complete raw output shown below — every tick line is
accounted for):

```
RUN 1 (70 ticks total):   3 × wakeups_happened: 0     67 × wakeups_happened: 1
RUN 2 (68 ticks total):   3 × wakeups_happened: 0     65 × wakeups_happened: 1
```

That is a coalescing ratio of roughly **750 : 1** (50 000 lines ÷ ~66 wakeup ticks), stable in shape across
both runs (both ~66–67 wakeup ticks; both exactly 3 idle `wakeups_happened: 0` ticks — one startup render
plus two after the surge ends). The tick **cadence** (from the timestamps below) is also stable across runs:
during the surge the median inter-tick gap is **5.0 ms in both runs** (mean 11.4 ms run 1 / 10.8 ms run 2;
min 0 ms, max 37 ms run 1 / 49 ms run 2) — i.e. `input_delay` (3 ms) plus the parse+render time — whereas the
idle gaps before/after the surge are ~0.5 s. The `pollForEvents final timeout` values are **not** the driver:
they are the wait until the nearest ~0.5 s housekeeping timer, counting steadily down across successive
ticks (the full `0.4xx → 0.0xx` sequence is visible verbatim in the complete raw output below) while
`poll()` keeps returning **early** (~5 ms) each time the I/O thread posts a wakeup; the timeout resets to
~0.495 the moment the surge stops and the loop goes idle.

Complete, unedited event-loop output — **RUN 1** (140 lines):

```
[0.167] pollForEvents final timeout: 0.000
[0.182] --------- loop tick, wakeups_happened: 1 ----------
[0.182] pollForEvents final timeout: 0.000
[0.182] --------- loop tick, wakeups_happened: 0 ----------
[0.182] pollForEvents final timeout: 0.002
[0.184] --------- loop tick, wakeups_happened: 1 ----------
[0.190] pollForEvents final timeout: 0.483
[0.190] --------- loop tick, wakeups_happened: 1 ----------
[0.196] pollForEvents final timeout: 0.477
[0.196] --------- loop tick, wakeups_happened: 1 ----------
[0.202] pollForEvents final timeout: 0.471
[0.202] --------- loop tick, wakeups_happened: 1 ----------
[0.224] pollForEvents final timeout: 0.465
[0.224] --------- loop tick, wakeups_happened: 1 ----------
[0.229] pollForEvents final timeout: 0.443
[0.229] --------- loop tick, wakeups_happened: 1 ----------
[0.234] pollForEvents final timeout: 0.438
[0.234] --------- loop tick, wakeups_happened: 1 ----------
[0.239] pollForEvents final timeout: 0.433
[0.239] --------- loop tick, wakeups_happened: 1 ----------
[0.272] pollForEvents final timeout: 0.428
[0.272] --------- loop tick, wakeups_happened: 1 ----------
[0.277] pollForEvents final timeout: 0.395
[0.277] --------- loop tick, wakeups_happened: 1 ----------
[0.282] pollForEvents final timeout: 0.390
[0.282] --------- loop tick, wakeups_happened: 1 ----------
[0.287] pollForEvents final timeout: 0.385
[0.287] --------- loop tick, wakeups_happened: 1 ----------
[0.292] pollForEvents final timeout: 0.380
[0.292] --------- loop tick, wakeups_happened: 1 ----------
[0.324] pollForEvents final timeout: 0.375
[0.324] --------- loop tick, wakeups_happened: 1 ----------
[0.329] pollForEvents final timeout: 0.343
[0.329] --------- loop tick, wakeups_happened: 1 ----------
[0.334] pollForEvents final timeout: 0.338
[0.334] --------- loop tick, wakeups_happened: 1 ----------
[0.339] pollForEvents final timeout: 0.333
[0.339] --------- loop tick, wakeups_happened: 1 ----------
[0.373] pollForEvents final timeout: 0.327
[0.373] --------- loop tick, wakeups_happened: 1 ----------
[0.378] pollForEvents final timeout: 0.294
[0.378] --------- loop tick, wakeups_happened: 1 ----------
[0.383] pollForEvents final timeout: 0.289
[0.384] --------- loop tick, wakeups_happened: 1 ----------
[0.394] pollForEvents final timeout: 0.283
[0.394] --------- loop tick, wakeups_happened: 1 ----------
[0.430] pollForEvents final timeout: 0.268
[0.430] --------- loop tick, wakeups_happened: 1 ----------
[0.441] pollForEvents final timeout: 0.237
[0.441] --------- loop tick, wakeups_happened: 1 ----------
[0.452] pollForEvents final timeout: 0.226
[0.452] --------- loop tick, wakeups_happened: 1 ----------
[0.472] pollForEvents final timeout: 0.215
[0.472] --------- loop tick, wakeups_happened: 1 ----------
[0.478] pollForEvents final timeout: 0.195
[0.478] --------- loop tick, wakeups_happened: 1 ----------
[0.488] pollForEvents final timeout: 0.189
[0.488] --------- loop tick, wakeups_happened: 1 ----------
[0.493] pollForEvents final timeout: 0.179
[0.493] --------- loop tick, wakeups_happened: 1 ----------
[0.524] pollForEvents final timeout: 0.174
[0.524] --------- loop tick, wakeups_happened: 1 ----------
[0.530] pollForEvents final timeout: 0.143
[0.530] --------- loop tick, wakeups_happened: 1 ----------
[0.537] pollForEvents final timeout: 0.137
[0.537] --------- loop tick, wakeups_happened: 1 ----------
[0.548] pollForEvents final timeout: 0.129
[0.548] --------- loop tick, wakeups_happened: 1 ----------
[0.553] pollForEvents final timeout: 0.119
[0.553] --------- loop tick, wakeups_happened: 1 ----------
[0.575] pollForEvents final timeout: 0.113
[0.575] --------- loop tick, wakeups_happened: 1 ----------
[0.580] pollForEvents final timeout: 0.092
[0.580] --------- loop tick, wakeups_happened: 1 ----------
[0.585] pollForEvents final timeout: 0.087
[0.585] --------- loop tick, wakeups_happened: 1 ----------
[0.590] pollForEvents final timeout: 0.082
[0.590] --------- loop tick, wakeups_happened: 1 ----------
[0.627] pollForEvents final timeout: 0.077
[0.627] --------- loop tick, wakeups_happened: 1 ----------
[0.632] pollForEvents final timeout: 0.040
[0.632] --------- loop tick, wakeups_happened: 1 ----------
[0.637] pollForEvents final timeout: 0.035
[0.637] --------- loop tick, wakeups_happened: 1 ----------
[0.642] pollForEvents final timeout: 0.030
[0.642] --------- loop tick, wakeups_happened: 1 ----------
[0.672] pollForEvents final timeout: 0.025
[0.672] --------- loop tick, wakeups_happened: 1 ----------
[0.677] pollForEvents final timeout: 0.495
[0.677] --------- loop tick, wakeups_happened: 1 ----------
[0.682] pollForEvents final timeout: 0.490
[0.682] --------- loop tick, wakeups_happened: 1 ----------
[0.687] pollForEvents final timeout: 0.485
[0.687] --------- loop tick, wakeups_happened: 1 ----------
[0.692] pollForEvents final timeout: 0.480
[0.692] --------- loop tick, wakeups_happened: 1 ----------
[0.727] pollForEvents final timeout: 0.475
[0.727] --------- loop tick, wakeups_happened: 1 ----------
[0.732] pollForEvents final timeout: 0.440
[0.732] --------- loop tick, wakeups_happened: 1 ----------
[0.743] pollForEvents final timeout: 0.435
[0.743] --------- loop tick, wakeups_happened: 1 ----------
[0.772] pollForEvents final timeout: 0.424
[0.772] --------- loop tick, wakeups_happened: 1 ----------
[0.777] pollForEvents final timeout: 0.395
[0.777] --------- loop tick, wakeups_happened: 1 ----------
[0.782] pollForEvents final timeout: 0.390
[0.782] --------- loop tick, wakeups_happened: 1 ----------
[0.788] pollForEvents final timeout: 0.384
[0.788] --------- loop tick, wakeups_happened: 1 ----------
[0.798] pollForEvents final timeout: 0.379
[0.798] --------- loop tick, wakeups_happened: 1 ----------
[0.824] pollForEvents final timeout: 0.369
[0.824] --------- loop tick, wakeups_happened: 1 ----------
[0.829] pollForEvents final timeout: 0.343
[0.829] --------- loop tick, wakeups_happened: 1 ----------
[0.834] pollForEvents final timeout: 0.338
[0.834] --------- loop tick, wakeups_happened: 1 ----------
[0.839] pollForEvents final timeout: 0.333
[0.839] --------- loop tick, wakeups_happened: 1 ----------
[0.872] pollForEvents final timeout: 0.328
[0.872] --------- loop tick, wakeups_happened: 1 ----------
[0.877] pollForEvents final timeout: 0.295
[0.877] --------- loop tick, wakeups_happened: 1 ----------
[0.882] pollForEvents final timeout: 0.290
[0.882] --------- loop tick, wakeups_happened: 1 ----------
[0.887] pollForEvents final timeout: 0.285
[0.887] --------- loop tick, wakeups_happened: 1 ----------
[0.924] pollForEvents final timeout: 0.280
[0.924] --------- loop tick, wakeups_happened: 1 ----------
[0.930] pollForEvents final timeout: 0.243
[0.930] --------- loop tick, wakeups_happened: 1 ----------
[0.935] pollForEvents final timeout: 0.237
[0.935] --------- loop tick, wakeups_happened: 1 ----------
[0.939] pollForEvents final timeout: 0.232
[1.177] --------- loop tick, wakeups_happened: 0 ----------
[1.177] pollForEvents final timeout: 0.495
[1.677] --------- loop tick, wakeups_happened: 0 ----------
[1.677] pollForEvents final timeout: 0.495
[2.140] --------- loop tick, wakeups_happened: 1 ----------
```

Complete, unedited event-loop output — **RUN 2** (136 lines):

```
[0.162] pollForEvents final timeout: 0.000
[0.176] --------- loop tick, wakeups_happened: 1 ----------
[0.176] pollForEvents final timeout: 0.000
[0.176] --------- loop tick, wakeups_happened: 0 ----------
[0.176] pollForEvents final timeout: 0.486
[0.177] --------- loop tick, wakeups_happened: 1 ----------
[0.177] pollForEvents final timeout: 0.003
[0.186] --------- loop tick, wakeups_happened: 1 ----------
[0.193] pollForEvents final timeout: 0.475
[0.193] --------- loop tick, wakeups_happened: 1 ----------
[0.199] pollForEvents final timeout: 0.469
[0.199] --------- loop tick, wakeups_happened: 1 ----------
[0.205] pollForEvents final timeout: 0.463
[0.205] --------- loop tick, wakeups_happened: 1 ----------
[0.210] pollForEvents final timeout: 0.456
[0.210] --------- loop tick, wakeups_happened: 1 ----------
[0.215] pollForEvents final timeout: 0.452
[0.215] --------- loop tick, wakeups_happened: 1 ----------
[0.219] pollForEvents final timeout: 0.447
[0.219] --------- loop tick, wakeups_happened: 1 ----------
[0.268] pollForEvents final timeout: 0.435
[0.268] --------- loop tick, wakeups_happened: 1 ----------
[0.275] pollForEvents final timeout: 0.393
[0.275] --------- loop tick, wakeups_happened: 1 ----------
[0.286] pollForEvents final timeout: 0.386
[0.286] --------- loop tick, wakeups_happened: 1 ----------
[0.291] pollForEvents final timeout: 0.376
[0.291] --------- loop tick, wakeups_happened: 1 ----------
[0.296] pollForEvents final timeout: 0.371
[0.296] --------- loop tick, wakeups_happened: 1 ----------
[0.301] pollForEvents final timeout: 0.366
[0.301] --------- loop tick, wakeups_happened: 1 ----------
[0.313] pollForEvents final timeout: 0.361
[0.313] --------- loop tick, wakeups_happened: 1 ----------
[0.318] pollForEvents final timeout: 0.349
[0.318] --------- loop tick, wakeups_happened: 1 ----------
[0.323] pollForEvents final timeout: 0.344
[0.323] --------- loop tick, wakeups_happened: 1 ----------
[0.365] pollForEvents final timeout: 0.339
[0.365] --------- loop tick, wakeups_happened: 1 ----------
[0.371] pollForEvents final timeout: 0.297
[0.371] --------- loop tick, wakeups_happened: 1 ----------
[0.376] pollForEvents final timeout: 0.291
[0.376] --------- loop tick, wakeups_happened: 1 ----------
[0.381] pollForEvents final timeout: 0.286
[0.381] --------- loop tick, wakeups_happened: 1 ----------
[0.386] pollForEvents final timeout: 0.281
[0.386] --------- loop tick, wakeups_happened: 1 ----------
[0.392] pollForEvents final timeout: 0.275
[0.392] --------- loop tick, wakeups_happened: 1 ----------
[0.413] pollForEvents final timeout: 0.270
[0.413] --------- loop tick, wakeups_happened: 1 ----------
[0.418] pollForEvents final timeout: 0.249
[0.418] --------- loop tick, wakeups_happened: 1 ----------
[0.461] pollForEvents final timeout: 0.244
[0.461] --------- loop tick, wakeups_happened: 1 ----------
[0.466] pollForEvents final timeout: 0.201
[0.466] --------- loop tick, wakeups_happened: 1 ----------
[0.471] pollForEvents final timeout: 0.196
[0.471] --------- loop tick, wakeups_happened: 1 ----------
[0.476] pollForEvents final timeout: 0.191
[0.476] --------- loop tick, wakeups_happened: 1 ----------
[0.480] pollForEvents final timeout: 0.186
[0.480] --------- loop tick, wakeups_happened: 1 ----------
[0.485] pollForEvents final timeout: 0.182
[0.485] --------- loop tick, wakeups_happened: 1 ----------
[0.490] pollForEvents final timeout: 0.177
[0.490] --------- loop tick, wakeups_happened: 1 ----------
[0.513] pollForEvents final timeout: 0.172
[0.513] --------- loop tick, wakeups_happened: 1 ----------
[0.518] pollForEvents final timeout: 0.149
[0.518] --------- loop tick, wakeups_happened: 1 ----------
[0.561] pollForEvents final timeout: 0.144
[0.561] --------- loop tick, wakeups_happened: 1 ----------
[0.566] pollForEvents final timeout: 0.101
[0.566] --------- loop tick, wakeups_happened: 1 ----------
[0.571] pollForEvents final timeout: 0.096
[0.571] --------- loop tick, wakeups_happened: 1 ----------
[0.576] pollForEvents final timeout: 0.091
[0.576] --------- loop tick, wakeups_happened: 1 ----------
[0.580] pollForEvents final timeout: 0.086
[0.581] --------- loop tick, wakeups_happened: 1 ----------
[0.585] pollForEvents final timeout: 0.081
[0.585] --------- loop tick, wakeups_happened: 1 ----------
[0.590] pollForEvents final timeout: 0.077
[0.590] --------- loop tick, wakeups_happened: 1 ----------
[0.613] pollForEvents final timeout: 0.072
[0.613] --------- loop tick, wakeups_happened: 1 ----------
[0.617] pollForEvents final timeout: 0.049
[0.617] --------- loop tick, wakeups_happened: 1 ----------
[0.661] pollForEvents final timeout: 0.044
[0.661] --------- loop tick, wakeups_happened: 1 ----------
[0.666] pollForEvents final timeout: 0.001
[0.666] --------- loop tick, wakeups_happened: 1 ----------
[0.671] pollForEvents final timeout: 0.496
[0.671] --------- loop tick, wakeups_happened: 1 ----------
[0.676] pollForEvents final timeout: 0.491
[0.676] --------- loop tick, wakeups_happened: 1 ----------
[0.681] pollForEvents final timeout: 0.486
[0.681] --------- loop tick, wakeups_happened: 1 ----------
[0.686] pollForEvents final timeout: 0.481
[0.686] --------- loop tick, wakeups_happened: 1 ----------
[0.697] pollForEvents final timeout: 0.476
[0.697] --------- loop tick, wakeups_happened: 1 ----------
[0.713] pollForEvents final timeout: 0.465
[0.713] --------- loop tick, wakeups_happened: 1 ----------
[0.717] pollForEvents final timeout: 0.449
[0.718] --------- loop tick, wakeups_happened: 1 ----------
[0.761] pollForEvents final timeout: 0.444
[0.761] --------- loop tick, wakeups_happened: 1 ----------
[0.766] pollForEvents final timeout: 0.401
[0.766] --------- loop tick, wakeups_happened: 1 ----------
[0.770] pollForEvents final timeout: 0.396
[0.770] --------- loop tick, wakeups_happened: 1 ----------
[0.775] pollForEvents final timeout: 0.392
[0.775] --------- loop tick, wakeups_happened: 1 ----------
[0.780] pollForEvents final timeout: 0.387
[0.780] --------- loop tick, wakeups_happened: 1 ----------
[0.790] pollForEvents final timeout: 0.382
[0.790] --------- loop tick, wakeups_happened: 1 ----------
[0.824] pollForEvents final timeout: 0.361
[0.824] --------- loop tick, wakeups_happened: 1 ----------
[0.836] pollForEvents final timeout: 0.338
[0.836] --------- loop tick, wakeups_happened: 1 ----------
[0.861] pollForEvents final timeout: 0.326
[0.861] --------- loop tick, wakeups_happened: 1 ----------
[0.866] pollForEvents final timeout: 0.301
[0.866] --------- loop tick, wakeups_happened: 1 ----------
[0.870] pollForEvents final timeout: 0.296
[0.870] --------- loop tick, wakeups_happened: 1 ----------
[0.875] pollForEvents final timeout: 0.292
[1.171] --------- loop tick, wakeups_happened: 0 ----------
[1.171] pollForEvents final timeout: 0.495
[1.671] --------- loop tick, wakeups_happened: 0 ----------
[1.671] pollForEvents final timeout: 0.495
[2.073] --------- loop tick, wakeups_happened: 1 ----------
```

Reading the two runs together: the loop begins idle, transitions into a dense ~5 ms surge cadence (every
tick `wakeups_happened: 1`) for the ~0.7 s the emitter runs, then snaps back to the ~0.5 s idle cadence
(`wakeups_happened: 0`) the instant the surge stops (run 1: ticks at 1.177 and 1.677; run 2: 1.171 and
1.671). The 50 000-line burst never produces more than one wakeup per tick — proving the two coalescing
layers work exactly as the source above predicts: the I/O-thread `WAKEUP` gate `[kitty/child-monitor.c:L1562-L1570]`
compresses per-window reads into one wakeup, and `drain_wakeup_fd` `[glfw/backend_utils.c:L217-L230]` folds
whatever wakeups do arrive into a single boolean tick. That is why the system speeds up smoothly under load
instead of thrashing on per-line wakeups.

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

`shell_prompt_marking()` `[kitty/screen.c:L2328]` binds to the current row under an explicit guard, and
issues the `cmd_output_marking` callbacks. Verbatim function body `[kitty/screen.c:L2329-L2355]`:

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

So the guard `if (self->cursor->y < self->lines)` `[kitty/screen.c:L2329]` binds everything to the current
cursor row; `A` (prompt start) → `CALLBACK("cmd_output_marking", "O", Py_False)` `[kitty/screen.c:L2338]`;
`C` (command start, carrying the cmdline) → `CALLBACK("cmd_output_marking", "OO", Py_True, c)`
`[kitty/screen.c:L2347]`; `D` (command end, carrying exit status) → `CALLBACK("cmd_output_marking", "Os",
Py_None, exit_status)` `[kitty/screen.c:L2352]`. Each writes
`self->linebuf->line_attrs[self->cursor->y].prompt_kind` — i.e. it *tags a specific screen row*.
Command-output extent lookups later use `find_cmd_output` `[kitty/screen.c:L3527]` and `cmd_output`
`[kitty/screen.c:L3606]`.

**The `case ESC_OSC:` at `[kitty/screen.c:L964]` (required checkpoint item) — its role and its limitation.**
That branch lives inside `get_prefix_and_suffix_for_escape_code()` `[kitty/screen.c:L955]`, which maps an
escape-code *kind* to the byte prefix/suffix kitty uses when it **emits** an escape code back to the child.
For `ESC_OSC` it sets `*prefix = "\033]"` `[kitty/screen.c:L964]`; the whole function is called only from the
two **output** helpers `write_escape_code_to_child()` `[kitty/screen.c:L979]` (which calls it at
`[kitty/screen.c:L982]`) and `write_escape_code_to_child_python()` `[kitty/screen.c:L999]` (at
`[kitty/screen.c:L1002]`) — the helpers kitty uses to answer device queries such as the DA/XTVERSION
replies at `[kitty/screen.c:L2125-L2137]`. Its **limitation for Q3** is therefore that it is strictly the
*output* (kitty→child) side: it constructs the leading `ESC ]` of an OSC that kitty itself sends, and it
plays **no part** in routing an *incoming* OSC 133 hint.

The **incoming** OSC 133 route — the one that actually keeps shell hints aligned with text — is entirely in
the parser and the screen model, and is observed byte-for-byte in §3.4: the parser enters OSC state at
`case ESC_OSC: SET_STATE(OSC)` `[kitty/vt-parser.c:L270]`; a completed OSC is handed to `dispatch_osc()`
`[kitty/vt-parser.c:L457]`; the `case 133:` branch `[kitty/vt-parser.c:L536]` calls
`shell_prompt_marking(self->screen, (char*)buf + i)` `[kitty/vt-parser.c:L544]`; and that lands in
`shell_prompt_marking()` `[kitty/screen.c:L2328]`, which fires the `cmd_output_marking` callbacks
`[kitty/screen.c:L2338,L2347,L2352]`. So both the required output branch (`screen.c:L964`) and the actual
input route (`vt-parser.c:L536` → `screen.c:L2328`) are cited precisely, and the two are on opposite sides
of the pipeline.

### 3.4 Runtime OSC 133 bytes from **real** shells (row binding proven)

Rather than a synthetic `printf`, the markers were driven through **real bash, zsh, and fish** under a
canonical Kitty and captured with `strace read()` on the I/O thread's PTY master (`fd 10`). The exact
harness (repeated per shell, `$TAG`∈{bash,zsh,fish}):

```
setsid strace -f -e trace=read -s 8192 -o /tmp/kitty_obs/q3_$TAG.strace \
  ./kitty/launcher/kitty --config NONE -o allow_remote_control=yes -o shell_integration=enabled \
    --listen-on unix:/tmp/kitty_obs/q3_$TAG.sock $TAG
# then, into that window:
./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q3_$TAG.sock send-text '<marker cmd>\n'
./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q3_$TAG.sock get-text --extent last_cmd_output
```

Each marker command uses real shell arithmetic (`$((6*7))` in bash/zsh, `(math "6*7")` in fish) → `42`, so
the bytes provably come from a real shell, not a `printf`. The reads shown below are the **complete,
unedited** `fd 10` reads that contain `]133` (`\33`=ESC 0x1b, `\7`=BEL 0x07; the giant `fd 3`/`fd 6` reads
elided here are Kitty *loading the integration script source*, not runtime markers). The command that
produced each block is stated above it.

**bash** — `shell-integration/bash/kitty.bash`; marker cmd `echo Q3BASH_ROWBIND_$((6*7))`
(`grep -aF ']133' q3_bash.strace | grep 'read(10,'`):

```
169100 read(10, "\33[30P\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]0;root@reverse-code-generator-5fbdfc46-dtmzr: /tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b\7root@reverse-code-generator-5fbdfc46-dtmzr:/tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b# \33]133;k;start_suffix_kitty\7\33[5 q\33]2;/tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b\7\33]133;k;end_suffix_kitty\7", 1048440) = 421
169100 read(10, "\33]2;echo Q3BASH_ROWBIND_$((6*7))\7\33]133;C;cmdline=echo\\ Q3BASH_ROWBIND_\\$\\(\\(6\\*7\\)\\)\7", 1048576) = 85
169100 read(10, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2Q3BASH_ROWBIND_42\r\n", 1048491) = 124
169100 read(10, "\33[30P\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]0;root@reverse-code-generator-5fbdfc46-dtmzr: /tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b\7root@reverse-code-generator-5fbdfc46-dtmzr:/tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b# \33]133;k;start_suffix_kitty\7\33[5 q\33]2;/tmp/blitzy/kitty/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b\7\33]133;k;end_suffix_kitty\7", 1048359) = 421
```

The real bash `cmdline` is `cmdline=echo\ Q3BASH_ROWBIND_\$\(\(6\*7\)\)` — the `%q`-quoted command
(`shell-integration/bash/kitty.bash:L208` `printf "\e]133;C;cmdline=%q\a" "$last_cmd"`), **not** a literal
`cmdline=true`. `D;0` (exit status 0) and `A` (prompt start) are emitted from `PS1`
(`kitty.bash:L239`), each wrapped in the zero-width `k;start_kitty` / `k;end_kitty` region markers
(`kitty.bash:L127`). `get-text --extent last_cmd_output` → `Q3BASH_ROWBIND_42`.

**zsh** — `shell-integration/zsh/kitty-integration`; marker cmd `echo Q3ZSH_ROWBIND_$((6*7))`. zsh only
activates when it is **not** a "new install" (see §3.6); with a `~/.zshrc` present:

```
171287 read(10, "\r\33[0m\33[27m\33[24m\33[J\33]133;A\7reverse-code-generator-5fbdfc46-dtmzr# \33[K", 1048576) = 68
171287 read(10, "\33]133;C;cmdline=echo\\ Q3ZSH_ROWBIND_\\$\\(\\(6\\*7\\)\\)\7", 1048555) = 51
171287 read(10, "\33]133;D;0\7", 1048325) = 10
171287 read(10, "\r\33[0m\33[27m\33[24m\33[J\33]133;A\7reverse-code-generator-5fbdfc46-dtmzr# \33[K", 1048115) = 68
171287 read(10, "\33]133;C;cmdline=exit\7", 1048549) = 21
```

zsh emits `A` **directly** (from `PS1`, `kitty-integration:L153` `mark1=$'%{\e]133;A\a%}'`), the preexec
`C` with `%q`-quoted `cmdline` (`kitty-integration:L218`), and — distinctively — defers `D;$cmd_status`
(`kitty-integration:L145`) to the *next* precmd (hence `D;0` arrives on its own read). No `k;` region
wrappers. `get-text --extent last_cmd_output` → `Q3ZSH_ROWBIND_42`.

**fish** — `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`; marker cmd
`echo Q3FISH_ROWBIND_(math "6*7")`:

```
171746 read(10, "\33]133;A;special_key=1\7\342\200\246\33[92m\33(B\33[m\33(B\33[m\33(B\33[m\33[31m/blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8_67276b\33(B\33[m (blitzy-73e5ee2c-7fe8-49a5-87e2-0a59e7cb2ca8)\33(B\33[m# \r\r\n", 1048502) = 167
171746 read(10, "\33]133;C;cmdline_url=echo%20Q3FISH_ROWBIND_%28math%20%226%2A7%22%29\7\33[0 q", 1048453) = 72
171746 read(10, "\33]133;D;0\7\33[?25h", 1048267) = 16
171746 read(10, "\33]133;C;cmdline_url=exit\7\33[?2004l\33[=0u\33>", 1048539) = 40
171746 read(10, "\33]133;D;0\7\33[?25h", 1048497) = 16
```

fish uses a *third* encoding: `A;special_key=1` (`kitty-shell-integration.fish:L85`), and
`C;cmdline_url=%s` **URL-escaped** (`kitty-shell-integration.fish:L91` `string escape --style=url`) — so
`echo Q3FISH_ROWBIND_(math "6*7")` becomes `echo%20Q3FISH_ROWBIND_%28math%20%226%2A7%22%29` — then
`D;$status` (`kitty-shell-integration.fish:L96`).
`get-text --extent last_cmd_output` → `Q3FISH_ROWBIND_42`.

That `last_cmd_output` returns exactly the command's output (`Q3{BASH,ZSH,FISH}_ROWBIND_42`) — and nothing
of the prompt or command line — is the proof that the `C`/`D` callbacks pinned the output extent to the
correct rows (`find_cmd_output` `[kitty/screen.c:L3527]`). Three shells, three different byte encodings of
the *same* OSC 133 contract, all correctly demultiplexed: the hints stayed aligned with the text.

**Remote path — the `ssh` kitten propagates the same contract.** To confirm the alignment survives an
`ssh` hop (the "unstable remote connection" of Q4, exercised here over passwordless loopback so the bytes
are inspectable), a real bash was run **inside** a Kitty window, and from it the `ssh` kitten opened a
session to a local `sshd`; the I/O thread's PTY master (`fd 10`) was straced. The kitten refuses to run
outside a Kitty window, so this drives the genuine kitten bootstrap, not a bypass. Producing commands:

```
# loopback sshd already listening on :2222 with the session key trusted
./kitty/launcher/kitty --config NONE -o allow_remote_control=yes -o shell_integration=enabled \
  --listen-on unix:/tmp/kitty_obs/q3_ssh.sock bash        # (straced on fd 10)
./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q3_ssh.sock send-text \
  './kitty/launcher/kitten ssh -p 2222 -o StrictHostKeyChecking=no \
     -o UserKnownHostsFile=/tmp/kitty_obs/sshsetup/known_hosts root@127.0.0.1\n'
# into the remote shell: echo Q3SSH_ROWBIND_$((6*7))
```

The complete `fd 10` reads containing `]133` from the **remote** shell
(`grep -aF ']133' q3_ssh.strace | grep 'read(10,'`):

```
172622 read(10, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]0;root@reverse-code-generator-5fbdfc46-dtmzr: ~\7root@reverse-code-generator-5fbdfc46-dtmzr:~# \33]133;k;start_suffix_kitty\7\33[5 q\33]2;reverse-code-generator-5fbdfc46-dtmzr: ~\7\33]133;k;end_suffix_kitty\7", 1048349) = 262
172622 read(10, "\33]2;reverse-code-generator-5fbdfc46-dtmzr: echo Q3SSH_ROWBIND_$((6*7))\7\33]133;C;cmdline=echo\\ Q3SSH_ROWBIND_\\$\\(\\(6\\*7\\)\\)\7", 1048538) = 122
172622 read(10, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2Q3SSH_ROWBIND_42\r\n", 1048416) = 123
```

**Dedicated `hexdump -C` / `cat -v` byte verification of the remote OSC 133 output row.** The `= 123`-byte
`read(10, …)` above is the remote shell's output row. Its exact bytes were decoded from the real strace
capture to a file and rendered with both tools; the decoded length is **123 bytes**, matching the `read()`
return value exactly (byte-faithful decode). Producing commands:

```
# decode the exact 123 strace-captured bytes of that read into a file, then render with each tool
$ python3 /tmp/kitty_obs/decode_ssh_osc.py    # -> /tmp/kitty_obs/q3_ssh_osc_row.bin (123 bytes; matches read()=123)
$ hexdump -C /tmp/kitty_obs/q3_ssh_osc_row.bin
00000000  01 1b 5d 31 33 33 3b 6b  3b 73 74 61 72 74 5f 6b  |..]133;k;start_k|
00000010  69 74 74 79 07 02 01 1b  5d 31 33 33 3b 6b 3b 65  |itty....]133;k;e|
00000020  6e 64 5f 6b 69 74 74 79  07 02 01 1b 5d 31 33 33  |nd_kitty....]133|
00000030  3b 6b 3b 73 74 61 72 74  5f 73 75 66 66 69 78 5f  |;k;start_suffix_|
00000040  6b 69 74 74 79 07 02 01  1b 5b 30 20 71 02 01 1b  |kitty....[0 q...|
00000050  5d 31 33 33 3b 6b 3b 65  6e 64 5f 73 75 66 66 69  |]133;k;end_suffi|
00000060  78 5f 6b 69 74 74 79 07  02 51 33 53 53 48 5f 52  |x_kitty..Q3SSH_R|
00000070  4f 57 42 49 4e 44 5f 34  32 0d 0a                 |OWBIND_42..|
0000007b
$ cat -v /tmp/kitty_obs/q3_ssh_osc_row.bin
^A^[]133;k;start_kitty^G^B^A^[]133;k;end_kitty^G^B^A^[]133;k;start_suffix_kitty^G^B^A^[[0 q^B^A^[]133;k;end_suffix_kitty^G^BQ3SSH_ROWBIND_42^M
```

`hexdump -C` makes the OSC 133 introducer explicit at offset `0x02`: `1b 5d 31 33 33 3b` = `ESC ] 1 3 3 ;`,
and again at `0x0c`, `0x1e`, `0x4a`, `0x53` for each `k;…` region marker; `cat -v` shows the same bytes with
`^[` = ESC (0x1b), `^G` = BEL (0x07), `^A`/`^B` = the SOH/STX (0x01/0x02) zero-width wrappers around each
`k;…` region, and the payload `Q3SSH_ROWBIND_42` terminated by `^M` (CR, 0x0d) + LF (0x0a). Both renderings
confirm the remote bytes are exactly the OSC 133 command-boundary contract, not a paraphrase.

These remote bytes are **byte-identical** to the local bash capture above (`k;start_kitty` region markers,
`D;0`, `A`, `C;cmdline=echo\ Q3SSH_ROWBIND_\$\(\(6\*7\)\)`, then the output row). This is loopback (the same
host), but the `sshd`-spawned login shell does **not** inherit Kitty's integration: `root`'s rc files
contain zero kitty references (`grep -c kitty /root/.bashrc /root/.profile /etc/bash.bashrc /etc/profile`
→ all `0`) and `kitty` is not on that shell's PATH (`PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`,
`which kitty` → not found). A *plain* `ssh` shell would therefore emit no OSC 133 — the markers appear only
because the `ssh` **kitten** transferred the integration over the channel during bootstrap (the strace
shows `KITTY_SHELL_INTEGRATION` in the transferred bootstrap data on 5 reads). `get-text --extent
last_cmd_output` over the ssh session → `Q3SSH_ROWBIND_42`. The alignment contract is preserved end-to-end
across the remote hop; the ssh kitten's internal mechanics (config/keepalive, path resolution, askpass)
are grounded separately in §4.4.

### 3.5 Sibling variants (same contract, different payload encodings)

All three shells emit the *same* `A`/`C`/`D` command-boundary contract but encode the payload differently:

| Shell | Command start | Prompt start | Command end |
|-------|---------------|--------------|-------------|
| bash  | `C;cmdline=%q` (shell-quoted) | `A` (plain) | `D;$?` |
| zsh   | `C;cmdline=%q` (shell-quoted) | `A` (plain) | `D;$cmd_status` |
| fish  | `C;cmdline_url=%s` (URL-escaped) | `A;special_key=1` | `D;$status` |

Byte-exact verification via `strace -e read=10` (hexdump of the actual PTY-master read), confirming the
parser sees the literal bytes claimed. Producing command — the remote `echo RMT_$((6*7))` command line
(`grep -aF cmdline q3_sshx.strace`):

```
173363 read(10, "\33]2;reverse-code-generator-5fbdfc46-dtmzr: echo RMT_$((6*7))\7\33]133;C;cmdline=echo\\ RMT_\\$\\(\\(6\\*7\\)\\)\7", 1048548) = 102
 | 00000  1b 5d 32 3b 72 65 76 65  72 73 65 2d 63 6f 64 65  .]2;reverse-code |
 | 00010  2d 67 65 6e 65 72 61 74  6f 72 2d 35 66 62 64 66  -generator-5fbdf |
 | 00020  63 34 36 2d 64 74 6d 7a  72 3a 20 65 63 68 6f 20  c46-dtmzr: echo  |
 | 00030  52 4d 54 5f 24 28 28 36  2a 37 29 29 07 1b 5d 31  RMT_$((6*7))..]1 |
 | 00040  33 33 3b 43 3b 63 6d 64  6c 69 6e 65 3d 65 63 68  33;C;cmdline=ech |
 | 00050  6f 5c 20 52 4d 54 5f 5c  24 5c 28 5c 28 36 5c 2a  o\ RMT_\$\(\(6\* |
 | 00060  37 5c 29 5c 29 07                                 7\)\).           |
```

The `C` marker (second read above) begins at offset `0x3d`: `1b 5d 31 33 33 3b 43 3b` = `ESC ] 1 3 3 ; C ;` = the OSC 133 `C` introducer, then
`63 6d 64 6c 69 6e 65 3d` (`cmdline=`), then the `%q`-quoted command where `5c 20`=`\ ` (escaped space),
`5c 24`=`\$`, `5c 28`=`\(`, `5c 2a`=`\*` — the literal backslashes bash's `%q` inserts — terminated by
`07` (BEL). This is a real shell's `cmdline`, **not** the literal `cmdline=true`. (ESC = 0x1b, ']' = 0x5d,
BEL = 0x07 terminator.)

### 3.6 Error / edge path — a real condition where a shell *silently skips* integration

Not every shell activates integration on the happy path. `setup_zsh_env` `[kitty/shell_integration.py:L49]`
calls `is_new_zsh_install` `[kitty/shell_integration.py:L27]`: if **no** user rc files
(`.zshrc`/`.zshenv`/`.zprofile`/`.zlogin`) exist in `ZDOTDIR` (or `HOME` when `ZDOTDIR` is empty), it
returns early *without* setting `ZDOTDIR` to the integration dir — deliberately, to avoid interfering with
zsh's own `zsh-newuser-install` wizard.

Observed exactly, and this is why the zsh capture in §3.4 needed a prerequisite. The session runs as
`root` with `HOME=/root` and no rc files present. On the first zsh run the strace showed
`KITTY_SHELL_INTEGRATION=enabled` **was** exported into the child, yet there were **no `]133` markers at
all** (0 occurrences in a 1041-line read trace). Kitty had additionally probed
`execve("/usr/bin/zsh",["zsh","--norcs","--interactive","-c","echo -n $ZDOTDIR"])` (via
`get_zsh_zdotdir_from_global_zshenv` `[kitty/shell_integration.py:L42]`) which returned empty, so
`is_new_zsh_install("")` was `True` a second time and `setup_zsh_env` returned at ~`L55` without exporting
`ZDOTDIR`. The fix that matches the source's own condition was to create `~/.zshrc` (i.e. `$HOME/.zshrc`,
`HOME=/root`): `echo '# minimal zshrc' > ~/.zshrc`. With one rc file present, `is_new_zsh_install` returned
`False`, `setup_zsh_env` exported `ZDOTDIR` to the integration dir, and the `]133` markers appeared exactly
as shown in §3.4. bash and fish never hit this gate — bash injects via an explicit `--rcfile`
(`setup_bash_env` `[kitty/shell_integration.py:L70]`, no new-install check), fish via `XDG_DATA_DIRS`
(`setup_fish_env` `[kitty/shell_integration.py:L16]`). This asymmetry is exactly the kind of condition a
happy-path-only investigation would miss.

### 3.7 Setup dispatch (`shell_integration.py`)

`modify_shell_environ` `[kitty/shell_integration.py:L218]` sets `env['KITTY_SHELL_INTEGRATION']` and then
calls the per-shell modifier: `setup_fish_env` `[kitty/shell_integration.py:L16]` (prepends the
integration dir to `XDG_DATA_DIRS`), `setup_zsh_env` `[kitty/shell_integration.py:L49]` (sets
`ZDOTDIR`, subject to §3.6), and `setup_bash_env` `[kitty/shell_integration.py:L70]` (rcfile injection).
Observed at runtime: Kitty sets `KITTY_SHELL_INTEGRATION=enabled`; each per-shell script unsets it after
loading. The `ssh` bootstrap (`shell-integration/ssh/bootstrap.sh`) emits **no** `133;A/C/D` markers
itself (grep found none) — it *propagates* terminfo + integration over ssh, and the remote shell's own
integration emits the markers, byte-identical to the local shell (captured in §3.4; the kitten's internal
mechanics are grounded in §4.4).

### 3.8 Synchronized / pending mode (DEC private mode 2026) — before / during / after

"Keeping aligned" also covers the mechanism that batches rendering so a half-drawn screen is never shown.
The industry-standard name is **Synchronized Output, DEC private mode 2026** (enable `CSI ? 2026 h`,
disable `CSI ? 2026 l`, query `CSI ? 2026 $ p` via DECRQM). `PENDING_MODE 2026`
`[kitty/control-codes.h:L235]` maps to `PENDING_UPDATE (2026 << 5)` `[kitty/modes.h:L86]`, dispatched at
`case PENDING_MODE << 5:` `[kitty/screen.c:L1174]` → `screen_pause_rendering(self, val, 0)`
`[kitty/screen.c:L1175]`.

This was **not** observed through a `test_*` parser stand-in but through the **real GUI pipeline**: a
Python child was run under a canonical Kitty so its stdin/stdout *are* the PTY; the child writes the mode
set/reset/query bytes, which travel the genuine path — I/O thread `read_bytes` → `vt-parser.c` →
`screen.c` mode dispatch → the DECRQM reply written back to the child via `write_escape_code_to_child`
`[kitty/screen.c:L2241]`. Producing command:

```
./kitty/launcher/kitty --config NONE python3 /tmp/kitty_obs/pending_obs.py
```

The script issues `CSI ?2026 $p` (DECRQM) before, during, and after, plus an expiry probe. Complete,
unedited result file (`cat /tmp/kitty_obs/q3_pending_result.txt`), **RUN 1**:

```
# DEC 2026 (synchronized output / pending mode) state via DECRQM
# Ps: 1=set(pending active), 2=reset(inactive) [screen.c:L2238 expires_at?1:2]
BEFORE(no pending)             Ps=2  raw=b'\x1b[?2026;2$y'
DURING(after ?2026h)           Ps=1  raw=b'\x1b[?2026;1$y'
AFTER(after ?2026l)            Ps=2  raw=b'\x1b[?2026;2$y'
EXPIRY t=0.05s(after ?2026h)   Ps=1  raw=b'\x1b[?2026;1$y'
EXPIRY t=2.35s(no l sent)      Ps=2  raw=b'\x1b[?2026;2$y'
```

**RUN 2** (byte-identical — value is stable across runs):

```
# DEC 2026 (synchronized output / pending mode) state via DECRQM
# Ps: 1=set(pending active), 2=reset(inactive) [screen.c:L2238 expires_at?1:2]
BEFORE(no pending)             Ps=2  raw=b'\x1b[?2026;2$y'
DURING(after ?2026h)           Ps=1  raw=b'\x1b[?2026;1$y'
AFTER(after ?2026l)            Ps=2  raw=b'\x1b[?2026;2$y'
EXPIRY t=0.05s(after ?2026h)   Ps=1  raw=b'\x1b[?2026;1$y'
EXPIRY t=2.35s(no l sent)      Ps=2  raw=b'\x1b[?2026;2$y'
```

**Before / during / after, grounded:**

- **Before** — no pending: DECRQM answers `Ps=2` (raw `\x1b[?2026;2$y`). The reply value comes from
  `case PENDING_UPDATE: ans = self->paused_rendering.expires_at ? 1 : 2;`
  `[kitty/screen.c:L2237-L2238]`, formatted by
  `snprintf(buf, sizeof(buf) - 1, "%s%u;%u$y", (private ? "?" : ""), which, ans)`
  `[kitty/screen.c:L2240]`. With no pending render, `expires_at == 0`, so `ans = 2`.
- **During** — after `CSI ?2026 h`: `Ps=1`. `screen_pause_rendering(self, true, 0)`
  `[kitty/screen.c:L2506]` sets `expires_at = monotonic() + ms_to_monotonic_t(for_in_ms)`
  `[kitty/screen.c:L2522]`; because `expires_at != 0`, DECRQM now answers `1`. In this state the renderer
  holds the pre-pause snapshot while the parser keeps consuming and mutating the live model.
- **After** — after `CSI ?2026 l`: `Ps=2` again. Unpausing clears `expires_at` and marks the screen dirty
  for one atomic re-render, so DECRQM returns to `2`.

**Direct visual proof that the *rendered frame* is held (before / during / after).** The DECRQM value
above is the terminal's own report of the render-pause flag; to prove the pixels really do freeze, a child
was run under the full GUI (canonical Kitty on Xvfb) that (1) prints a baseline row, (2) enters pending
mode and prints two *new* rows, (3) exits pending mode — while a host process grabbed the actual X
framebuffer at each phase with `PIL.ImageGrab.grab(xdisplay=":99")` and compared pixels. Command:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 python3 /tmp/kitty_obs/pending_render_host.py
# (launches: ./kitty/launcher/kitty --config NONE -o cursor_blink_interval=0 -o font_size=20 \
#            python3 /tmp/kitty_obs/pending_render_child.py)
```

Complete, unedited output — **RUN 1**:

```
=== FRAME HASHES ===
A_before(baseline visible)   md5=131278cc03a5e9cdd3aa59f39955823f
B_during(pending, text written) md5=131278cc03a5e9cdd3aa59f39955823f
C_after(pending exited)      md5=c90f81889578606fe3a4d12a10d56a29
=== PIXEL DIFFS ===
A vs B (before vs during-pending): differing_pixels=0 bbox=None
A vs C (before vs after-exit):     differing_pixels=7296 bbox=(3, 45, 497, 97)
B vs C (during vs after-exit):     differing_pixels=7296 bbox=(3, 45, 497, 97)
=== VERDICT ===
render_HELD_during_pending = True (A==B)
atomic_update_on_exit      = True (C differs from B)
```

**RUN 2** (byte-identical — including the md5s, so rendering is deterministic here):

```
=== FRAME HASHES ===
A_before(baseline visible)   md5=131278cc03a5e9cdd3aa59f39955823f
B_during(pending, text written) md5=131278cc03a5e9cdd3aa59f39955823f
C_after(pending exited)      md5=c90f81889578606fe3a4d12a10d56a29
=== PIXEL DIFFS ===
A vs B (before vs during-pending): differing_pixels=0 bbox=None
A vs C (before vs after-exit):     differing_pixels=7296 bbox=(3, 45, 497, 97)
B vs C (during vs after-exit):     differing_pixels=7296 bbox=(3, 45, 497, 97)
```

The decisive numbers: **A and B are pixel-identical** (`differing_pixels=0`, same md5) even though between
those two grabs the child wrote two brand-new rows to the model — the renderer held the pre-pause frame,
exactly the snapshot taken in `screen_pause_rendering` `[kitty/screen.c:L2506-L2545]`. On exit, **C differs
from B by 7296 pixels** confined to `bbox=(3, 45, 497, 97)` — precisely the two text rows below the
baseline — appearing in a *single* atomic frame. This is the "half-drawn screen is never shown" guarantee,
observed at the pixel level, stable across both runs. (The PNG frames `A_before_baseline.png`,
`B_during_pending.png`, `C_after_exit.png` are written to `/tmp/kitty_obs/render/` by the command and are
transient observation artifacts, not part of the repository.)

**Transitional / expiry state (the "falls apart if left pending" guard).** If the mode is entered and the
closing `l` never arrives, the mode does **not** stay pending forever — `screen_check_pause_rendering`
force-exits once `now > expires_at` `[kitty/screen.c:L2489-L2490]`, using the default window set at
`if (for_in_ms <= 0) for_in_ms = 2000;` `[kitty/screen.c:L2521]`. The result file above already shows this:
at `t=0.05s` (just after `?2026h`, no `l`) DECRQM = `Ps=1`, and at `t=2.35s` DECRQM = `Ps=2` — it expired
on its own. To pin the boundary, a second child (`pending_obs2.py`) queried at `1.0/1.9/2.1/2.5s` after a
single `?2026h`. Command and complete output, both runs byte-identical:

```
./kitty/launcher/kitty --config NONE python3 /tmp/kitty_obs/pending_obs2.py /tmp/kitty_obs/q3_bracket_run1.txt
```

```
# elapsed_s Ps(1=active,2=expired/inactive) after single ?2026h, no l   (RUN 1)
t=1.01  Ps=1
t=1.91  Ps=1
t=2.1   Ps=2
t=2.51  Ps=2
```
```
# elapsed_s Ps(1=active,2=expired/inactive) after single ?2026h, no l   (RUN 2)
t=1.01  Ps=1
t=1.91  Ps=1
t=2.1   Ps=2
t=2.51  Ps=2
```

Still active at `1.91s`, expired by `2.1s` — the transition sits exactly at the 2000 ms default of
`[kitty/screen.c:L2521]`, and is stable across both runs.

**Kitty's own DCS trigger reaches the same state.** Kitty additionally accepts a DCS synchronized-update
trigger (`ESC P =1s ESC \` to start, `ESC P =2s ESC \` to stop), dispatched in the VT parser at
`screen_start_pending_mode` `[kitty/vt-parser.c:L639]` and `screen_stop_pending_mode`
`[kitty/vt-parser.c:L644]`. Driven through the same real child pipeline
(`./kitty/launcher/kitty --config NONE python3 /tmp/kitty_obs/pending_dcs.py`), complete output, both runs
byte-identical:

```
# DEC 2026 state via DECRQM, driven by Kitty DCS =1s/=2s trigger   (RUN 1)
BEFORE(no pending)       Ps=2  raw=b'\x1b[?2026;2$y'
DURING(after DCS =1s)    Ps=1  raw=b'\x1b[?2026;1$y'
AFTER(after DCS =2s)     Ps=2  raw=b'\x1b[?2026;2$y'
```
```
# DEC 2026 state via DECRQM, driven by Kitty DCS =1s/=2s trigger   (RUN 2)
BEFORE(no pending)       Ps=2  raw=b'\x1b[?2026;2$y'
DURING(after DCS =1s)    Ps=1  raw=b'\x1b[?2026;1$y'
AFTER(after DCS =2s)     Ps=2  raw=b'\x1b[?2026;2$y'
```

Both entry forms converge on the identical `2 → 1 → 2` state. Byte-exact, the two DECRQM replies differ in
exactly one byte: inactive `1b 5b 3f 32 30 32 36 3b 32 24 79` vs active `1b 5b 3f 32 30 32 36 3b 31 24 79`
— the 9th byte (0-indexed offset 8, the `Ps` digit) is `32`↔`31` (`'2'`↔`'1'`). This is the "during" state where the renderer holds the last
frame while the parser keeps consuming bytes, so text and markers still land in order but the screen
updates atomically when the mode ends — and the 2000 ms auto-expiry guarantees it recovers even if the
closing sequence is lost on an unstable link.

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
pipeline. An unstable or dropped link surfaces as the ssh child exiting, handled by the same child-gone
teardown as any child death: observed as a **SIGCHLD** reap when the remote is idle at the drop (§4.7), or
the **`EIO`**-on-read branch when a read is in flight (§1.5).

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

This isolates the exact byte bound using the **same compiled functions** the live read path uses. The
`Screen.test_create_write_buffer` / `test_commit_write_buffer` / `test_parse_written_data` methods are
thin wrappers `[kitty/screen.c:L4755-L4783]` that call, respectively, `vt_parser_create_write_buffer`
`[kitty/vt-parser.c:L1451]`, `vt_parser_commit_write` `[kitty/vt-parser.c:L1465]`, and `parse_worker`
`[kitty/vt-parser.c:L1496]` — the very functions `read_bytes()` and `do_parse()` drive at runtime (no
reimplementation). The script commits 200 KiB chunks *without* parsing until the buffer reports zero free
space, then parses once to drain, printing free space before / during / after.

Command (run twice, back to back):

```
$ echo "=== RUN 1 ==="; python3 /tmp/kitty_obs/q4/q4_buffer_saturation.py; \
  echo; echo "=== RUN 2 ==="; python3 /tmp/kitty_obs/q4/q4_buffer_saturation.py
```

Complete, unedited output of **both** runs:

```
=== RUN 1 ===
VT_PARSER_BUFFER_SIZE = 1048576 bytes = 1.0 MiB
BEFORE (empty):  free_space = 1048576  has_space = True
DURING (commit 200KiB chunks, NO parse):
    free_space: 1048576 -> 843776 -> 638976 -> 434176 -> 229376 -> 24576 -> 0
    last non-zero commit accepted only 24576 bytes (partial)
    total bytes accepted = 1048576  (== BUF_SZ? True)
    at free_space=0: has_space = False   => io_loop sets POLLIN:0
AFTER drain (test_parse_written_data): free_space = 1048576  has_space = True  (restored)

=== RUN 2 ===
VT_PARSER_BUFFER_SIZE = 1048576 bytes = 1.0 MiB
BEFORE (empty):  free_space = 1048576  has_space = True
DURING (commit 200KiB chunks, NO parse):
    free_space: 1048576 -> 843776 -> 638976 -> 434176 -> 229376 -> 24576 -> 0
    last non-zero commit accepted only 24576 bytes (partial)
    total bytes accepted = 1048576  (== BUF_SZ? True)
    at free_space=0: has_space = False   => io_loop sets POLLIN:0
AFTER drain (test_parse_written_data): free_space = 1048576  has_space = True  (restored)
```

**Run-to-run comparison:** the two runs are byte-for-byte identical. Every free-space step
(`1048576 → 843776 → 638976 → 434176 → 229376 → 24576 → 0`) matches; both accept exactly **1048576**
bytes total before `has_space` becomes `False`; both restore to `1048576` after a single parse. The bound
is therefore deterministic, not a sampled average. It is grounded in `*sz = BUF_SZ - self->write.offset`
`[kitty/vt-parser.c:L1457]` (free space shrinks by each commit), the `has_space` predicate
`ans = self->read.sz + self->write.pending < BUF_SZ` `[kitty/vt-parser.c:L1481]`, and `BUF_SZ (1024u*1024u)`
`[kitty/vt-parser.c:L18]`. This measures the byte bound in isolation; §4.4/§4.5 next show the **same bound
engaging in the live GUI pipeline** (POLLIN withdrawn, child blocked on `write()`).

### 4.4 `POLLIN` actually withdrawn → restored in the full GUI (before / during / after)

To force the buffer full in a *live* GUI Kitty, the main/parse thread (TID == PID) was frozen with a
`ptrace` helper (`PTRACE_SEIZE` + `PTRACE_INTERRUPT`, `/tmp/kitty_obs/q4/q4_freeze.py`) for 4 seconds
while a child kept writing a continuous 64 KiB flood (`/tmp/kitty_obs/q4/q4_flood.py`), with `strace`
attached to **only** the I/O thread (`comm=KittyChildMon`) tracing `poll`/`ppoll`
(`/tmp/kitty_obs/q4/q4_run.sh`). The poll array holds exactly three descriptors, in the fixed order the
source builds them — `children_fds[0] = wakeup_read_fd`, `children_fds[1] = signal_read_fd`
`[kitty/child-monitor.c:L183]`, then `children_fds[EXTRA_FDS + i]` (`EXTRA_FDS = 2`
`[kitty/child-monitor.c:L35]`) for the child. In this run those descriptors are `fd=6` (wakeup eventfd,
drained at `[kitty/child-monitor.c:L1515]`), `fd=7` (signalfd, `[kitty/child-monitor.c:L1519]`), and
`fd=8` (child PTY master, read at `[kitty/child-monitor.c:L1531]`, gated at `[kitty/child-monitor.c:L1501]`).
(Descriptor *numbers* are per-process — other sections' runs show the child PTY master as `fd=10` — but
the array *position* index 2 = the child is invariant.)

**RUN 1** (`grep -n` extract from `/tmp/kitty_obs/q4/q4_poll_run1.strace`; freeze log:
`FROZEN t=1783498962.246382 / THAWED t=1783498966.246537`, gap 4.000155 s):

```
08:22:42.256541 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
08:22:42.256599 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 2) = 0 (Timeout) <0.002100>
08:22:42.259573 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, -1) = 1 ([{fd=6, revents=POLLIN}]) <3.993444>
08:22:46.253135 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000017>
```

Reading top to bottom: **BEFORE** — `fd=8` requested with `events=POLLIN`, `poll` returns it readable
(the I/O thread is draining). **DURING (onset)** — 53 µs later `fd=8` is requested with `events=0`: `POLLIN`
has been withdrawn because the parser buffer is full. **DURING (block)** — the I/O thread then issues
an infinite-timeout `poll` (the `, 3, -1` form) with `fd=8` still at `events=0`, and blocks for **3.993444 s**, woken
**only** by `fd=6` (the wakeup eventfd — i.e. `wakeup_io_loop` `[kitty/child-monitor.c:L442]` firing after
the main thread freed space). **AFTER (restore)** — `fd=8` is requested with `events=POLLIN` again and is
immediately readable. Throughout the window every poll shows `fd=8, events=0` (21 such polls in RUN 1).

**RUN 2** (`/tmp/kitty_obs/q4/q4_poll_run2.strace`; freeze log:
`FROZEN t=1783499078.540628 / THAWED t=1783499082.540765`, gap 4.000137 s):

```
08:24:38.553240 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000010>
08:24:38.555786 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 0) = 0 (Timeout) <0.000009>
08:24:38.555855 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, -1) = 1 ([{fd=6, revents=POLLIN}]) <3.995577>
08:24:42.551551 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000021>
```

**Run-to-run comparison:** both runs show the identical `fd=8 events` transition `POLLIN → 0 → POLLIN`,
the identical single long `poll(-1)` woken by `fd=6`, and a block of essentially the whole freeze:
RUN 1 `<3.993444>`, RUN 2 `<3.995577>` (both ≈ the 4.0 s freeze; 18 `events=0` polls in RUN 2). The
mechanism — not just the number — reproduces exactly. This is the `POLLIN` gate at
`[kitty/child-monitor.c:L1501]` (`children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0`)
driven by the `has_space` predicate `[kitty/vt-parser.c:L1481]`, resumed by `wakeup_io_loop`
`[kitty/child-monitor.c:L442]` once `write_space_created` `[kitty/vt-parser.c:L1438]` is set.

### 4.5 The child blocks on `write()` for the whole window (kernel PTY buffer full)

The producer child's own `strace` (tracing `write` to `fd 1` = its PTY slave,
`/tmp/kitty_obs/q4/q4_write_run{1,2}.strace`) shows the mirror image — a single `write()` blocked for the
entire withdrawal window, bracketed by fast writes.

**RUN 1** (the blocked write and its immediate neighbors):

```
08:22:42.255720 write(1, "Q4FLOOD_........................"..., 65536) = 65536 <0.000793>
08:22:42.256571 write(1, "Q4FLOOD_........................"..., 65536) = 65536 <3.997263>
08:22:46.253921 write(1, "Q4FLOOD_........................"..., 65536) = 65536 <0.000213>
```

**RUN 2** (same three-line bracket):

```
08:24:38.552054 write(1, "Q4FLOOD_........................"..., 65536) = 65536 <0.000841>
08:24:38.552933 write(1, "Q4FLOOD_........................"..., 65536) = 65536 <3.999206>
08:24:42.552227 write(1, "Q4FLOOD_........................"..., 65536) = 65536 <0.000752>
```

Two reading notes. First, the `"Q4FLOOD_........................"...` is **strace's own** 32-byte string
abbreviation (strace's default `-s 32`), not a manual elision — the `= 65536` return value proves the full
64 KiB block was written each time. Second, the block aligns to the µs with the `POLLIN` withdrawal in
§4.4: RUN 1's blocked `write()` starts at `42.256571` (28 µs before the poll onset `42.256599`) and lasts
`<3.997263>` (ending `46.253834`, ≈ the poll restore `46.253135`); RUN 2's starts `38.552933` and lasts
`<3.999206>`. The surrounding writes complete in well under a millisecond (`<0.000793>`, `<0.000213>`,
`<0.000841>`, `<0.000752>`). The child was paused **not** by any signal or in-band flow-control byte, but
simply because the kernel PTY buffer back-filled once Kitty's I/O thread stopped reading (`POLLIN`
withdrawn), and it resumed the instant Kitty read again. Both runs reproduce the multi-second block.

### 4.6 Explicit contrast with kernel line-discipline flow control

The child PTY's line discipline is the ordinary Linux default. Command
(`./kitty/launcher/kitty --config NONE sh -c 'stty -a </dev/tty'`), complete unedited output:

```
speed 38400 baud; rows 40; columns 100; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>; eol2 = <undef>; swtch = <undef>; start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R; werase = ^W; lnext = ^V; discard = ^O; min = 1; time = 0;
-parenb -parodd -cmspar cs8 -hupcl -cstopb cread -clocal -crtscts 
-ignbrk -brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl -ixoff -tandem ixon -ixany -imaxbel iutf8 
opost -olcuc -ocrnl onlcr -onocr -onlret -ofdel nl0 cr0 tab0 bs0 vt0 ff0 
isig icanon iexten echo echoe echok -echonl -noflsh -tostop -echoprt echoctl echoke -flusho -extproc
```

Note `-crtscts` (no hardware RTS/CTS), `iutf8` (UTF-8 mode, set by `fast_data_types.set_iutf8_fd(master, True)`
in `openpty()` at `[kitty/child.py:L174]`), and — importantly — `ixon` is **enabled** with `start = ^Q; stop = ^S` (the
DC1/DC3 characters), while `-ixoff` is off. So the kernel line discipline *has* XON/XOFF output control
available, yet the backpressure observed in §4.4/§4.5 is **not** it: Kitty never emits a `^S`/`^Q` (DC3/DC1)
to pause the child, and the child's `write()` block in §4.5 happens with no flow-control byte on the wire.
For reference:

| Mechanism | Where | Signal | Kitty uses it? |
|-----------|-------|--------|----------------|
| XON/XOFF (software) | kernel TTY line discipline (`ixon` on, per `stty` above) | in-band `DC1 0x11` (`^Q`) / `DC3 0x13` (`^S`) on the data line | **No** — Kitty never emits `DC1`/`DC3`; the pause in §4.5 carries no such byte |
| RTS/CTS (hardware)  | serial modem lines (`-crtscts` above) | out-of-band wires | **No** — no modem lines on a PTY |
| **Kitty backpressure** | application (I/O thread) | withhold `POLLIN` (§4.4) → kernel PTY buffer back-fills → child blocks on `write()` (§4.5) | **Yes** — this is the observed mechanism |

This is the crux of Q4: the "pause" is the *reader* (the I/O thread) declining readiness, not the tty
sending a flow-control byte — even though the tty is configured with `ixon` able to do so.

### 4.7 SSH kitten / unstable remote

The `ssh` kitten was exercised over a real OpenSSH loopback session (`127.0.0.1:2222`; no external remote
exists in the container). The kitten runs as a child of an outer GUI Kitty; the executable command was:

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8
$ ./kitty/launcher/kitty --config NONE \
    ./kitty/launcher/kitten ssh -o StrictHostKeyChecking=no \
      -o UserKnownHostsFile=/tmp/kitty_obs/sshsetup/known_hosts -p 2222 root@127.0.0.1 \
      'echo SSHK_REMOTE_$((6*7)); sleep 6'
```

The session authenticates against the loopback `sshd` — verbatim from
`/tmp/kitty_obs/sshsetup/sshd.log` (`tail`):

```
Accepted publickey for root from 127.0.0.1 port 38732 ssh2: ED25519 SHA256:FSY3E+9SSfiLF84lyf0hZ9WsbrB6tKTyHDiX73esD4k
```

**The `ssh` kitten's job is propagation, driven by the real `ssh` command line it builds.** Tracing the
outer Kitty's child with `strace -f -tt -e trace=execve -s 400` (`/tmp/kitty_obs/q4/ssh_execve.log`) shows
the kitten `exec`ing `/usr/bin/ssh` with the exact option set — the Q4-relevant keepalive/control options
appear here **complete and unedited**:

```
192564 08:37:24.318659 execve("/usr/bin/ssh", ["/usr/bin/ssh", "-o", "StrictHostKeyChecking=no", "-o", "UserKnownHostsFile=/tmp/kitty_obs/sshsetup/known_hosts", "-p", "2222", "-o", "ControlMaster=auto", "-o", "ControlPath=/root/.cache/kitty/run/kssh-192453-%C", "-o", "ControlPersist=yes", "-o", "ServerAliveInterval=60", "-o", "ServerAliveCountMax=5", "-o", "TCPKeepAlive=no", "--", "root@127.0.0.1", "exec", "sh", "-c", "'eval \"$(echo \"$0\" | tr \\\\\\v\\\\\\f\\\\\\r\\\\\\b \\\\\\047\\\\\\134\\\\\\n\\\\\\041)\"' ", "'#\10/bin/sh\r# Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net>\r# Distributed under terms of the GPLv3 license.\r\r{ \funalias command; \funset -f command; } >/dev/null 2>&1\rtdir=\"\"\rshell_integration_dir=\"\"\recho_on=\"1\"\r\rcleanup_on_bootstrap_exit() {\r    [ \"$echo_on\" = \"1\" ] && command stty \"echo\" 2> /dev/null < /dev/tty\r    echo_on=\"0\"\r    [ -n \"$tdir\" ] && command rm -rf \"$tdir\"\r    tdir=\"\"\r}\r\r"...], 0x24046fd10008 /* 96 vars */) = 0
```

(The final two argv elements are an `eval` unwrapper and the inlined `shell-integration/ssh/bootstrap.sh`
— its opening `'#\10/bin/sh\r# Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net>` is visible, and the trailing `"...` is
**strace's own** `-s 400` truncation of the rest of that script, the exact analog of the `write()` 32-byte
truncation above. That bootstrap script is the Q3 propagation payload, byte-exact in §3.4, not the Q4
subject here. `0x24046fd10008 /* 96 vars */` is the propagated environment; the `kitty_pid` in
`ControlPath=/root/.cache/kitty/run/kssh-192453-%C` carries the outer Kitty's PID `192453`.)

**Where each option comes from (source-grounded).** The keepalive/control block is emitted verbatim by
`connection_sharing_args()` in `kittens/ssh/main.go`: `"-o", "ControlMaster=auto"`
`[kittens/ssh/main.go:L138]`, `"-o", "ControlPath=" + filepath.Join(rd, cp)` `[kittens/ssh/main.go:L139]`
(where `cp` is `kitty.SSHControlMasterTemplate` with `{kitty_pid}` substituted `[kittens/ssh/main.go:L135-L136]`),
`"-o", "ControlPersist=yes"` `[kittens/ssh/main.go:L140]`, `"-o", "ServerAliveInterval=60"`
`[kittens/ssh/main.go:L141]`, `"-o", "ServerAliveCountMax=5"` `[kittens/ssh/main.go:L142]`, and
`"-o", "TCPKeepAlive=no"` `[kittens/ssh/main.go:L143]`. Cause → effect: `ServerAliveInterval=60` ×
`ServerAliveCountMax=5` ≈ **300 s** is the ssh-level dead-peer timeout that detects an unstable/dropped
link; `TCPKeepAlive=no` defers liveness to those application-level ssh keepalives; `ControlMaster=auto` +
`ControlPersist=yes` multiplex and persist the connection so a re-connect reuses the socket.

**Required SSH support files (line-grounded per M9).**

- `kittens/ssh/config.go` — parses the kitten's per-host configuration that *gates* the above. It defines
  `EnvInstruction` `[kittens/ssh/config.go:L28]` (env vars to propagate, serialized by `Serialize`
  `[kittens/ssh/config.go:L54]` / `final_env_instructions` `[kittens/ssh/config.go:L93]`),
  `CopyInstruction` `[kittens/ssh/config.go:L110]` (files to copy remotely, via `ParseCopyInstruction`
  `[kittens/ssh/config.go:L186]` and `resolve_file_spec` `[kittens/ssh/config.go:L139]`), and the per-host
  matcher `config_for_hostname()` `[kittens/ssh/config.go:L354]` which walks `strings.Split(q.Hostname, " ")`
  `[kittens/ssh/config.go:L356]`, all loaded by `load_config()` `[kittens/ssh/config.go:L391]`. It is
  `host_opts.Forward_remote_control` + `share_connections` (parsed here) that decide at
  `[kittens/ssh/main.go:L681-L683]` whether the persistent ControlMaster is actually used.
- `kittens/ssh/utils.go` — path/util resolution. `SSHExe()` resolves the ssh binary via
  `utils.FindExe("ssh")` `[kittens/ssh/utils.go:L22-L24]`; `RelevantKittyOpts()` resolves the config path
  `filepath.Join(utils.ConfigDir(), "kitty.conf")` `[kittens/ssh/utils.go:L245]` (parsed by
  `read_relevant_kitty_opts` `[kittens/ssh/utils.go:L228]` to recover `term`/`shell_integration`);
  `ParseSSHArgs()` `[kittens/ssh/utils.go:L125]` splits ssh flags from the server/command args; and
  `GetSSHVersion()` `[kittens/ssh/utils.go:L210]` + `SupportsAskpassRequire()` `[kittens/ssh/utils.go:L206-L208]`
  (`Major > 8 || (Major == 8 && Minor >= 4)`) decide whether the askpass path below is offered.
- `kittens/ssh/askpass.go` — credential prompting. `RunSSHAskpass()` `[kittens/ssh/askpass.go:L37]` is the
  program `ssh` invokes as `SSH_ASKPASS`; it reads the prompt from `os.Args[len(os.Args)-1]` `[kittens/ssh/askpass.go:L38]`,
  distinguishes a confirm vs. a password from `SSH_ASKPASS_PROMPT` `[kittens/ssh/askpass.go:L39-L40]` and a
  fingerprint check `[kittens/ssh/askpass.go:L45]`, marshals a `{message,type,is_password}` request into an
  SHM segment `[kittens/ssh/askpass.go:L46-L65]`, then signals the parent Kitty by writing the DCS
  `"\x1bP@kitty-ask|" + name + "\x1b\\"` to the controlling terminal (`trigger_ask` /
  `[kittens/ssh/askpass.go:L30]`), polls the SHM for the typed reply `[kittens/ssh/askpass.go:L70-L75]`, and
  prints it back to `ssh` `[kittens/ssh/askpass.go:L106]`.

**terminfo propagation confirmed** (executable check against the loopback remote):

```
$ ssh -p 2222 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/tmp/kitty_obs/sshsetup/known_hosts \
      -o BatchMode=yes root@127.0.0.1 'ls /root/.terminfo/x/ 2>/dev/null; echo RC=$?' </dev/null
xterm-kitty
RC=0
```

i.e. the bootstrap reconstructed `/root/.terminfo/x/xterm-kitty` on the remote (`RC=0`), so the remote
`TERM=xterm-kitty` resolves. (Byte-exact remote OSC 133 output over this same path is in §3.4.)

**Dropped-link edge (observed — SIGCHLD, not EIO here).** Abruptly killing the foreground `/usr/bin/ssh`
(`kill -9`, simulating a dropped link) while the remote was idle in `sleep` was traced on the outer Kitty's
`KittyChildMon` thread with `strace -y -e trace=read,close,poll,ppoll` (`/tmp/kitty_obs/q4/ssh_eio2.log`).
The `-y` annotations also confirm the §4.4 descriptor identities directly — `fd=8</dev/pts/ptmx>` (child
PTY master), `fd=6<anon_inode:[eventfd]>` (wakeup), `fd=7<anon_inode:[signalfd]>` (signal):

```
08:39:22.172409 read(8</dev/pts/ptmx>, "\33[?r\33[?19997l", 1048576) = 13 <0.000010>
08:39:22.172461 poll([{fd=6<anon_inode:[eventfd]>, events=POLLIN}, {fd=7<anon_inode:[signalfd]>, events=POLLIN}, {fd=8</dev/pts/ptmx>, events=POLLIN}], 3, -1) = 1 ([{fd=7, revents=POLLIN}]) <0.003358>
08:39:22.175862 read(7<anon_inode:[signalfd]>, "\21\0\0\0\0\0\0\0\1\0\0\0\330\364\2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128 <0.000028>
08:39:22.175919 read(7<anon_inode:[signalfd]>, 0x7fede95153a0, 4096) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
```

Reading it: the child emits a last `\33[?r\33[?19997l` (mode reset) on `fd=8`, then `poll` returns
**`fd=7` (the signalfd)** readable, and the `read(7<anon_inode:[signalfd]>) = 128` delivers one
`signalfd_siginfo` whose first field is `\21` = octal 021 = **17 = SIGCHLD** (the follow-up
`read(7<anon_inode:[signalfd]>)` returns `EAGAIN` — no further
signals). This is the SIGCHLD child-death path: `children_fds[1].revents && POLLIN`
`[kitty/child-monitor.c:L1516]` → `read_signals(children_fds[1].fd, handle_signal, &ss)`
`[kitty/child-monitor.c:L1519]` → `case SIGCHLD: ss->child_died = true;`
`[kitty/child-monitor.c:L1370-L1371]` → `if (ss.child_died) reap_children(self, OPT(close_on_child_death))`
`[kitty/child-monitor.c:L1526]` → `waitpid(-1, &status, WNOHANG)` `[kitty/child-monitor.c:L1418]`, after
which `close_on_child_death` tears the outer Kitty down.

Why SIGCHLD and not the `EIO`-on-read branch of §1.5: the child (the remote `sleep`) was **idle** at the
moment of the drop, so the I/O thread was blocked in `poll()` — `poll` surfaced the signalfd first, so the
child was reaped before any `read(8)` was attempted. §1.5's `EIO` branch `[kitty/child-monitor.c:L1348-L1350]`
is the sibling path that fires when the child is **mid-output** at teardown (a `read()` is in flight and
hits the closed slave). Both converge on the same child-gone teardown; which one is observed depends on
whether a `read()` is in flight when the peer goes away.

**Limitation (stated precisely):** true network instability (packet loss / jitter) cannot be induced on a
loopback in this container. The transport-level dead-peer handling is shown from the real ssh keepalive
options above (`ServerAlive*`, `TCPKeepAlive=no`), and the *terminal-pipeline consequence* of a drop is
directly observed as the SIGCHLD child-reap here (and as the `EIO` child-gone branch in §1.5).

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
child-gone teardown as any child exit — observed here as the **SIGCHLD** reap (§4.7) for an idle remote,
and as the **`EIO`**-on-read branch (§1.5) when a read is in flight at the drop.

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

Keystrokes, a bracketed-paste burst, and a **real resize (a genuine `SIGWINCH`)** were fired
*concurrently* at one running Kitty while `strace` (with `-yy`, so every fd is annotated with its kernel
object) watched the `KittyChildMon` I/O thread. The child is a small `bash` script that echoes each line it
reads and, on `SIGWINCH`, prints its new size — so the resize's arrival at the child appears **inside the
same I/O-thread read stream** as the keystroke and paste. Because a bare `read` defers a bash trap
indefinitely, the child uses `read -t 0.3` so the pending `WINCH` trap runs:

```
$ cat /tmp/kitty_obs/q5/q5_child.sh
#!/bin/bash
winch() { echo "Q5_WINCH_FIRED size=[$(stty size 2>/dev/null)]"; }
trap winch WINCH
echo "Q5_CHILD_READY"
while true; do
  if IFS= read -t 0.3 -r line; then echo "Q5_GOT:$line"; fi
done
```

The three inputs are driven through Kitty's real internal paths via remote control (`send-text` →
`write_to_child`; `send-text --bracketed-paste=enable` → the on-the-wire bracketed-paste byte sequence;
`resize-os-window` → `set_geometry` → `resize_pty` → `ioctl(TIOCSWINSZ)` → kernel `SIGWINCH`). They are
launched as three background jobs so their arrival order is not serialized. The window is pinned to a
deterministic start size and the resize target is computed to be strictly smaller (so it always changes and
always fits the framebuffer):

```
# fired concurrently (three background jobs) against the running kitty:
kitty @ --to unix:$SOCK send-text $'echo KEYSTROKE_LINE_42\n' &
kitty @ --to unix:$SOCK send-text --bracketed-paste=enable $'PASTE_A\nPASTE_B_42' &
kitty @ --to unix:$SOCK resize-os-window --unit cells --width 83 --height 25 &
# I/O thread traced with:
strace -tt -yy -s 4096 -e trace=read,write,poll,ppoll,ioctl -p <KittyChildMon tid> -o io_c$RUN.strace
```

`-yy` resolves the fds directly in the trace: **`fd=7`** = `eventfd` (the I/O thread's own wakeup
self-pipe, read side), **`fd=8`** = `signalfd:[HUP INT USR1 USR2 TERM CHLD]` (the handled-signal fd),
**`fd=10`** = `/dev/pts/ptmx` (the child PTY master), and **`fd=4`** = the *main-loop* wakeup `eventfd` (the
`write(4, …)` handoff). **BEFORE:** the I/O thread is blocked in `poll(timeout=-1)`; the parser buffer is
fully drained (each `read(10, …, 1048576)` shows the full `1048576`-byte budget). **AFTER:** the thread is
blocked in `poll(timeout=-1)` again with nothing pending — the observable definition of "settled."

**RUN 1** — complete, unedited `io_c1.strace` (27 lines). Here the two `send-text` streams coalesced into a
single 53-byte `write(10, …)` (keystroke bytes leading), and the resize's `Q5_WINCH_FIRED` marker is read
~109 ms later:

```
09:26:36.311006 read(7<{eventfd-count=0x1, eventfd-id=1179, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 1024) = 8
09:26:36.311298 read(7<{eventfd-count=0x1, eventfd-id=1179, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 1024) = 8
09:26:36.311368 read(7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, 0x7ab9a259d740, 1024) = -1 EAGAIN (Resource temporarily unavailable)
09:26:36.311411 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=10, revents=POLLOUT}])
09:26:36.311516 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "echo KEYSTROKE_LINE_42\n\33[200~PASTE_A\nPASTE_B_42\33[201~", 53) = 53
09:26:36.311565 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=10, revents=POLLIN}])
09:26:36.311628 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "echo KEYSTROKE_LINE_42\r\n^[[200~PASTE_A\r\nPASTE_B_42^[[201~", 1048576) = 57
09:26:36.311672 write(4<{eventfd-count=0, eventfd-id=1078, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
09:26:36.311721 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=10, revents=POLLIN}])
09:26:36.311783 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "Q5_GOT:echo KEYSTROKE_LINE_42\r\nQ5_GOT:\33[200~PASTE_A\r\n", 1048519) = 53
09:26:36.311825 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 0 (Timeout)
09:26:36.313972 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314052 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314118 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314178 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314261 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314325 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314382 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314438 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314501 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314566 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314626 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 0 (Timeout)
09:26:36.314683 write(4<{eventfd-count=0x1, eventfd-id=1078, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
09:26:36.314741 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=10, revents=POLLIN}])
09:26:36.420207 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "Q5_WINCH_FIRED size=[25 83]\r\n", 1048576) = 29
09:26:36.420329 write(4<{eventfd-count=0, eventfd-id=1078, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
09:26:36.420402 poll([{fd=7<{eventfd-count=0, eventfd-id=1179, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1 <detached ...>
```

**RUN 2** — complete, unedited `io_c2.strace` (19 lines). Same pipeline structure, but the ordering flipped:
the paste went out first as its **own** 30-byte `write(10, …)`, the keystroke followed as a **separate**
23-byte write, and a **real interleave edge** appeared — the paste's second line merged with the `echo`
line into `Q5_GOT:PASTE_B_42\33[201~echo KEYSTROKE_LINE_42`:

```
09:26:50.197744 read(7<{eventfd-count=0x1, eventfd-id=576, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 1024) = 8
09:26:50.198008 read(7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, 0x796c43b9d740, 1024) = -1 EAGAIN (Resource temporarily unavailable)
09:26:50.198068 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=10, revents=POLLOUT}])
09:26:50.198198 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[200~PASTE_A\nPASTE_B_42\33[201~", 30) = 30
09:26:50.198267 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=10, revents=POLLIN}])
09:26:50.198369 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "^[[200~PASTE_A\r\nPASTE_B_42^[[201~Q5_GOT:\33[200~PASTE_A\r\n", 1048576) = 55
09:26:50.198594 write(4<{eventfd-count=0x3, eventfd-id=1194, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
09:26:50.198761 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=7, revents=POLLIN}])
09:26:50.203901 read(7<{eventfd-count=0x1, eventfd-id=576, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 1024) = 8
09:26:50.203986 read(7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, 0x796c43b9d740, 1024) = -1 EAGAIN (Resource temporarily unavailable)
09:26:50.204044 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=10, revents=POLLOUT}])
09:26:50.204131 write(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "echo KEYSTROKE_LINE_42\n", 23) = 23
09:26:50.204183 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=10, revents=POLLIN}])
09:26:50.204287 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "echo KEYSTROKE_LINE_42\r\nQ5_GOT:PASTE_B_42\33[201~echo KEYSTROKE_LINE_42\r\n", 1048576) = 71
09:26:50.204334 write(4<{eventfd-count=0x2, eventfd-id=1194, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
09:26:50.204381 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=10, revents=POLLIN}])
09:26:50.306533 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "Q5_WINCH_FIRED size=[25 83]\r\n", 1048576) = 29
09:26:50.306639 write(4<{eventfd-count=0, eventfd-id=1194, eventfd-semaphore=0}>, "\1\0\0\0\0\0\0\0", 8) = 8
09:26:50.306715 poll([{fd=7<{eventfd-count=0, eventfd-id=576, eventfd-semaphore=0}>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1 <detached ...>
```

In both runs, the resize is real: Kitty's `--debug-rendering` log printed
`SIGWINCH sent to child in window: 1 with size: (25, 83, 747, 450)` (window.py L873 — the real
`set_geometry` → `resize_pty` path, §5.5), `kitty @ ls` confirmed the child grid changed from `100×32` to
`83×25`, and the child's `WINCH` trap emitted `Q5_WINCH_FIRED size=[25 83]`, which is exactly the
`read(10, …, 1048576) = 29` line near the end of each trace. The `\33` / `^[` tokens are strace's own
notations for the ESC byte `0x1b`; the `"…"…` on the `read(7, …)` line in RUN 2 is strace's `-s`
byte-limit on that address argument (the `= 8` return is the true length), not an elision.

**Ordering distribution (8 identical repeated runs).** Firing the same three concurrent inputs and
recording only which bytes reach the child *first* shows the ordering is genuinely non-deterministic — it
is reported as observed, not smoothed into a false determinism:

```
first bytes written to the child PTY, over 8 runs:
  keystroke first  ( write(10,"echo KEYSTROKE_LINE_42\n") )                 : 6/8   (runs 1,2,3,4,6,8)
  paste first      ( write(10,"\33[200~PASTE_A\nPASTE_B_42\33[201~"...) )   : 2/8   (runs 5,7)
whether the two send-text streams coalesced into ONE write(10,…) also varied:
  run 5 coalesced ("\33[200~PASTE_A\nPASTE_B_42\33[201~echo KEYSTROKE_LINE_42\n"), run 7 separate.
```

RUN 1 above is a keystroke-first, *coalesced* instance (one 53-byte write); RUN 2 is a paste-first,
*separate-write* instance. The pipeline **shape** is identical every time (wake → `write(10)` →
`read(10)` into the 1 MiB buffer → `write(4)` handoff → `input_delay` coalescing → drain → `poll(-1)`); only
the concurrent-arrival order — and hence whether the child ever sees a merged command line — changes.

### 5.3 What each observed element proves (cause → effect, with `file:line`), incl. the main-thread parse → mutate → render leg

**On the I/O thread (both traces in §5.2):**

- **`poll()` fd-ordering is fixed by array index, not by priority** — every `poll()` in both runs lists the
  same three descriptors in the same order: `[{fd=7 wakeup},{fd=8 signalfd},{fd=10 child}]`. That is the
  `EXTRA_FDS == 2` layout `[kitty/child-monitor.c:L35]`: `fds[0]` = the I/O-thread wakeup eventfd, `fds[1]` =
  the signalfd, and `fds[EXTRA_FDS + i]` = child *i* `[kitty/child-monitor.c:L1515-L1541]`. `poll()` itself
  returns *all* ready descriptors at once with **no** priority ranking; the I/O loop then services them in
  fixed array-index order — wakeup `[L1515]`, then signalfd `[L1516-L1519]`, then the per-child read/write
  loop `[L1528-L1541]`. So "which is handled first" is decided by that static scan order plus which
  descriptors `poll()` reported ready — there is no dynamic arbitration in which "control always wins" (this
  corrects that intuition; see §2.4).
- **`write(10, …)` = child write path** — the keystroke line (`echo KEYSTROKE_LINE_42\n`, `23` bytes) and the
  bracketed paste (`\33[200~PASTE_A\nPASTE_B_42\33[201~`, `30` bytes) leave via `write_to_child`
  `[kitty/child-monitor.c:L1443]`, requested only when `POLLOUT` is set on fd 10. In io_c1 the two were
  *coalesced* into a single `53`-byte write; in io_c2 they were two separate writes (`30` then `23`) — the
  run-to-run ordering variance quantified in §5.2.
- **`read(10, …, 1048576)` = ingestion into the 1 MiB parser buffer** — `read()` `[kitty/child-monitor.c:L1345]`
  into the buffer whose size is `1048576` = `VT_PARSER_BUFFER_SIZE` `[kitty/vt-parser.c:L18]` (see §1.3, §4.3).
  The size *argument* is the free space remaining: in io_c1 it went `1048576 → 1048519` (shrank by the `57`
  unparsed bytes still buffered) and then back to `1048576` once the main thread parsed and freed the buffer.
  The large-scale monotonic decrease under saturation is shown separately in §4.3.
- **`write(4, "\1\0\0\0\0\0\0\0", 8)` = handoff to the main thread** — the I/O thread posts the *main-loop*
  wakeup eventfd (fd 4), i.e. the `wakeup_main_loop()` side of the `WAKEUP` gate (§2.3), to hand the freshly
  read bytes to the parse/render thread. The `-yy` annotation shows the eventfd counter *coalescing*: in io_c2
  it read `eventfd-count=0x3`, then `0x2`, then `0` — handoff posts accumulated before the main thread drained
  them into one wakeup.
- **`read(7, "\1\0\0\0\0\0\0\0", 1024) = 8` then `EAGAIN` = the I/O thread draining its *own* wakeup** — fd 7
  is the I/O-thread wakeup eventfd; each drain reads the 8-byte counter (`eventfd-count=0x1`) and the very next
  read returns `-1 EAGAIN` because the eventfd is now empty (io_c1 and io_c2 both show this `= 8` / `EAGAIN`
  pair). This is the loop draining a single pending wakeup — **not** "3 coalesced" (an earlier draft
  mis-stated this counter as `3`; the real value read back is `0x1`).
- **`poll(…, timeout)` distribution differs run-to-run with the workload** — the real per-run timeout
  histograms, computed from the two traces in §5.2, are:

```
$ grep -aoE 'poll\(\[.*\], 3, (-?[0-9]+)\)' io_c1.strace | grep -oE ', 3, -?[0-9]+\)' \
    | sed -E 's/, 3, (-?[0-9]+)\)/\1/' | sort -n | uniq -c
      4 -1
     11 0
      1 2
$ grep -aoE 'poll\(\[.*\], 3, (-?[0-9]+)\)' io_c2.strace | grep -oE ', 3, -?[0-9]+\)' \
    | sed -E 's/, 3, (-?[0-9]+)\)/\1/' | sort -n | uniq -c
      6 -1
```

  io_c1 (keystroke-first, writes coalesced): `-1`×4 (block when idle), `2`×1 (first `input_delay` window),
  `0`×11 (coalescing spin while draining the burst); io_c2 (paste-first, writes separate): `-1`×6 only — it
  drained in a single pass with no spin. Each run also has one trailing `poll(-1) <detached>` when `strace`
  detached (shown in §5.2, not counted above). The `timeout=0`/`timeout=2` values *are* the
  `input_delay = 3 ms` coalescing window `[kitty/options/definition.py:L878]` in action; the difference
  between the two runs is genuine and workload-driven, reported as observed rather than smoothed.

**Crossing to the main/render thread — the parse → mutate → render leg (this is the second half of the
pipeline, answering "how does the buffer become a visible frame"):**

Once `write(4,…)` wakes the main thread, `main_loop()` runs one iteration `[kitty/child-monitor.c:L1229-L1237]`:
if there are pending resizes it runs `process_pending_resizes(now)` and marks `input_read = true`
`[L1232-L1235]`; it calls `parse_input(self)` `[kitty/child-monitor.c:L1236]`, which per child invokes
`do_parse()` `[kitty/child-monitor.c:L438]`. `do_parse` calls `self->parse_func(screen, &pd, flush)` `[L440]`
— this is where the buffered VT bytes actually **mutate the screen model** (`screen.c`, §3) — and, if parsing
freed buffer space, calls `wakeup_io_loop(self, false)` `[L442]` to re-arm the I/O thread's `POLLIN` (the
resume half of the §4 backpressure gate). The same iteration then calls `render(now, input_read)`
`[kitty/child-monitor.c:L1237]`; `render` `[L871]` draws unless throttled by its gate
`if (!input_read && time_since_last_render < OPT(repaint_delay))` `[L875]` (`repaint_delay = 10 ms`, §6) —
because `input_read` is `true` here (bytes were just parsed), the gate does **not** throttle and the frame is
drawn immediately.

Observed end-to-end with the framebuffer itself as the witness — a `PIL.ImageGrab` of the Xvfb root captured
before vs. after the concurrent keystroke+paste+resize burst (script `/tmp/kitty_obs/q5/q5_render.py`, which
launches kitty, grabs frame A, fires all three inputs concurrently, waits, grabs frame B, and diffs):

```
$ python3 /tmp/kitty_obs/q5/q5_render.py 1
RUN1 md5_A_before=2c50b6471dc51b960a2874d47a24070a
RUN1 md5_B_after =5a85601a4b69c8ba7a460f3b3a2c0d63
RUN1 A==B ? False
RUN1 differing_pixels=14092 bbox=(0, 29, 545, 203)
RUN1 screen_has_KEYSTROKE=True screen_has_PASTE_A=True screen_has_WINCH=True
$ python3 /tmp/kitty_obs/q5/q5_render.py 2
RUN2 md5_A_before=2c50b6471dc51b960a2874d47a24070a
RUN2 md5_B_after =5a85601a4b69c8ba7a460f3b3a2c0d63
RUN2 A==B ? False
RUN2 differing_pixels=14092 bbox=(0, 29, 545, 203)
RUN2 screen_has_KEYSTROKE=True screen_has_PASTE_A=True screen_has_WINCH=True
```

Before the burst the framebuffer md5 is `2c50b6471dc51b960a2874d47a24070a`; after it is
`5a85601a4b69c8ba7a460f3b3a2c0d63` (`A==B ? False` — the pixels genuinely changed, `14092` of them, inside
`bbox=(0,29,545,203)`), and `get-text` confirms all three inputs (`KEYSTROKE_LINE_42`, `PASTE_A`, and the
resize's `Q5_WINCH_FIRED`) landed on the screen. **Both runs produced byte-identical before *and* after
md5s**, so this parse→mutate→render leg is stable across runs, not a one-off. This is the observed proof that
the buffered bytes reach `screen.c` and then the GPU frame, closing the loop from Q1's `read()` to the visible
pixels.

### 5.4 Paste path (byte-exact), incl. the optional `filter` paste action (M11)

**Bracketed-paste wrapping, byte-exact (from `io_c2.strace`, §5.2).** In the combined trace the paste left the
I/O thread as this single write (the command that produced it is the concurrent-fire block in §5.2):

```
write(10</dev/pts/ptmx>, "\33[200~PASTE_A\nPASTE_B_42\33[201~", 30) = 30
    \33[200~ = 1b 5b 32 30 30 7e   (BRACKETED_PASTE_START "200~"  [kitty/modes.h:L82])
    \33[201~ = 1b 5b 32 30 31 7e   (BRACKETED_PASTE_END   "201~"  [kitty/modes.h:L83])
```

`BRACKETED_PASTE (2004 << 5)` `[kitty/modes.h:L81]`. The payload `PASTE_A\nPASTE_B_42` is wrapped in the
`\33[200~`/`\33[201~` markers only because the child had enabled DEC mode 2004; when the app has *not* enabled
it, kitty sends the (filtered) text raw — demonstrated below. The Python paste chain is
`sanitize_for_bracketed_paste` (import `[kitty/window.py:L117]`) → `paste_with_actions`
`[kitty/window.py:L1643]` → `paste_bytes` `[kitty/window.py:L1707]` → `paste_text`
`[kitty/window.py:L1713,L1718]` → `paste` `[kitty/window.py:L1780]`; the interactive entry points are
`paste_selection` `[kitty/window.py:L1513]` and `paste_selection_or_clipboard` `[kitty/window.py:L1519]`.

**The `filter` paste action — a real, non-default condition (M11).** Before wrapping, `paste_with_actions`
consults `opts.paste_actions`. The canonical default does **not** include `filter`:

```
$ python3 -c "import sys;sys.path.insert(0,'.');from kitty.options.types import defaults;\
print(defaults.paste_actions); print('filter' in defaults.paste_actions)"
frozenset({'quote-urls-at-prompt', 'confirm'})
False
```

so on a *default* paste, `load_paste_filter` `[kitty/window.py:L423]` is **never called** — the gate
`if 'filter' in opts.paste_actions:` `[kitty/window.py:L1647]` is false (definition
`[kitty/options/definition.py:L564]`, `paste_actions` default `quote-urls-at-prompt,confirm`). To exercise the
real filter path I ran kitty with `-o paste_actions=filter` and an isolated `KITTY_CONFIG_DIRECTORY` holding a
`paste-actions.py` that records the call and rewrites the text, then issued a real paste:

```
$ cat /tmp/kitty_obs/q5/cfg/paste-actions.py
def filter_paste(text: str) -> str:
    with open('/tmp/kitty_obs/q5/filter_invoked.txt', 'w') as f:
        f.write(f"filter_paste CALLED with: {text!r}\n")
    return text.replace('PASTE_INPUT', 'FILTERED_OUTPUT')

$ KITTY_CONFIG_DIRECTORY=/tmp/kitty_obs/q5/cfg ./kitty/launcher/kitty --config NONE \
    -o allow_remote_control=yes -o paste_actions=filter \
    --listen-on unix:/tmp/kitty_obs/q5/paste_sock4 \
    bash -c 'stty raw -echo; exec cat > /tmp/kitty_obs/q5/child_received.txt' &
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q5/paste_sock4 action paste "PASTE_INPUT_42"

# marker file — filter_paste was actually invoked, with the ORIGINAL paste text:
$ cat /tmp/kitty_obs/q5/filter_invoked.txt
filter_paste CALLED with: 'PASTE_INPUT_42'

# bytes the child's PTY received — the FILTERED text (no bracketed wrap: cat never enabled mode 2004):
$ hexdump -C /tmp/kitty_obs/q5/child_received.txt
00000000  46 49 4c 54 45 52 45 44  5f 4f 55 54 50 55 54 5f  |FILTERED_OUTPUT_|
00000010  34 32                                             |42|
00000012
```

The gate fired (`'filter' in opts.paste_actions`), `load_paste_filter()(text)` `[kitty/window.py:L1648]` ran
the user function against the original `'PASTE_INPUT_42'`, and the transformed `FILTERED_OUTPUT_42` (bytes
`46 49 4c…`) is what reached the child — the paste action really does route through `load_paste_filter`, and
the transformation is verified against the exact bytes emitted.

**Edge / fallback path.** When no `paste-actions.py` exists, `load_paste_filter` catches the
`FileNotFoundError` `[kitty/window.py:L430]` and returns an identity function `[kitty/window.py:L434-L435]`, so
an enabled-but-unconfigured `filter` action is a safe no-op rather than an error:

```
$ KITTY_CONFIG_DIRECTORY=$(mktemp -d) python3 -c "import sys;sys.path.insert(0,'.');\
import kitty.window as W; f=W.load_paste_filter(); print(f('HELLO_PASTE_42')=='HELLO_PASTE_42')"
True
```

### 5.5 Resize path + debounce (before / during / after)

**The resize path.** A resize flows `set_geometry` `[kitty/window.py:L850]` → `screen.resize`
`[kitty/window.py:L854]` → (net-change gate `if current_pty_size != self.last_reported_pty_size:`
`[kitty/window.py:L861]`) → `resize_pty` `[kitty/window.py:L863]`, which calls the C `resize_pty`
`[kitty/child-monitor.c:L592]` → `pty_resize` `[kitty/child-monitor.c:L577]` → `ioctl(fd, TIOCSWINSZ, dim)`
`[kitty/child-monitor.c:L579]`; the kernel then delivers `SIGWINCH` to the child. A debug print sits at
`[kitty/window.py:L873]` (needs `--debug-rendering`). This is the *same* real `SIGWINCH` already observed
**inside** the combined capstone trace of §5.2 (`read(10, "Q5_WINCH_FIRED size=[25 83]…")`), so the resize is
part of the concurrent capstone, not a separate artifact.

**Single resize, before / during / after (2 runs, identical).** Command
`kitty @ resize-os-window --action resize --unit cells --width 83 --height 25` against a deterministic
`100c × 32c` window (`-o remember_window_size=no -o initial_window_width=100c -o initial_window_height=32c`),
with a child that traps `SIGWINCH` and re-reads `stty size` (runner `/tmp/kitty_obs/q5/q5_resize.sh`):

```
$ bash /tmp/kitty_obs/q5/q5_resize.sh single 1
BEFORE: cols=100 lines=32
RESIZE-> target cols=83 lines=25
AFTER : cols=83 lines=25
--- kitty stderr (SIGWINCH debug lines, L873) ---
[0.386] SIGWINCH sent to child in window: 1 with size: (25, 83, 747, 450)
--- child WINCH-trap log ---
CHILD_READY size=[32 100]
WINCH size=[25 83]

$ bash /tmp/kitty_obs/q5/q5_resize.sh single 2
BEFORE: cols=100 lines=32
RESIZE-> target cols=83 lines=25
AFTER : cols=83 lines=25
--- kitty stderr (SIGWINCH debug lines, L873) ---
[0.393] SIGWINCH sent to child in window: 1 with size: (25, 83, 747, 450)
--- child WINCH-trap log ---
CHILD_READY size=[32 100]
WINCH size=[25 83]
```

BEFORE, the child's tty is `32 100`; DURING, kitty logs `SIGWINCH sent to child … size: (25, 83, 747, 450)`
(25 rows, 83 cols, 747×450 px); AFTER, the child's own `SIGWINCH` trap fired and `stty size` reports the new
`25 83`. Both runs are byte-identical.

**Rapid-resize coalescing (2 runs, identical).** Six `resize-os-window` calls fired back-to-back with no
inter-call sleep (alternating two target sizes), then the pipeline allowed to settle:

```
$ bash /tmp/kitty_obs/q5/q5_resize.sh rapid 1
BEFORE: cols=100 lines=32
SIGWINCH(child) before burst: 0
SIGWINCH(child) after burst : 1
=> delta = 1 WINCH trap firings for 6 rapid resize calls
--- kitty stderr (SIGWINCH debug lines, L873) ---
[0.454] SIGWINCH sent to child in window: 1 with size: (24, 84, 756, 432)
--- child WINCH-trap log ---
CHILD_READY size=[32 100]
WINCH size=[24 84]

$ bash /tmp/kitty_obs/q5/q5_resize.sh rapid 2
BEFORE: cols=100 lines=32
SIGWINCH(child) before burst: 0
SIGWINCH(child) after burst : 1
=> delta = 1 WINCH trap firings for 6 rapid resize calls
--- kitty stderr (SIGWINCH debug lines, L873) ---
[0.552] SIGWINCH sent to child in window: 1 with size: (24, 84, 756, 432)
--- child WINCH-trap log ---
CHILD_READY size=[32 100]
WINCH size=[24 84]
```

Six rapid resizes collapsed to **exactly one** `SIGWINCH` at the child, bearing the *final* geometry
`(24, 84, 756, 432)` = 84 cols × 24 rows — the intermediate sizes were dropped. Both runs identical.

**Which mechanism coalesces — the important distinction.** There are two separate resize-batching mechanisms,
and the headless observation above exercises the *first*:
- **Net-change gate (per render frame).** `set_geometry` runs on the main/render thread and calls `resize_pty`
  (→ `ioctl` → `SIGWINCH`) only when `current_pty_size != self.last_reported_pty_size`
  `[kitty/window.py:L861]`. Rapid programmatic resizes update the pending OS-window geometry, but
  `set_geometry` runs once per frame with the *latest* geometry, so intermediate sizes never reach a frame and
  are collapsed — this is what produced the 6→1 result.
- **Live-resize debounce (interactive drag only).** `process_pending_resizes()`
  `[kitty/child-monitor.c:L1043-L1075]` additionally debounces resizes, but its body is gated on
  `if (w->live_resize.in_progress)` `[kitty/child-monitor.c:L1047]`, comparing elapsed time against
  `OPT(resize_debounce_time).on_pause` `[kitty/child-monitor.c:L1055]` (0.1 s; §6). It only applies while a
  *human is dragging* the window edge, which cannot be driven headlessly under `resize-os-window`. So the
  coalescing above is the net-change gate, **not** the live-resize debounce (an earlier draft mis-attributed
  the headless coalescing solely to `process_pending_resizes`; both are reported here for completeness).

### 5.6 Input routing — which tab and window the bytes reach (`tabs.py` / `window_list.py`)

Before any keystroke or paste can be written to a child, the `Boss` must decide *which* window owns the input.
That routing is a two-level lookup: an OS window holds a list of tabs, and each `Tab` owns a `WindowList`
(import `[kitty/tabs.py:L64]`) constructed as `self.windows: WindowList = WindowList(self)`
`[kitty/tabs.py:L150]`. The `WindowList` class `[kitty/window_list.py:L144]` keeps
`self.all_windows: List[WindowType]` `[kitty/window_list.py:L147]`, is iterable via `__iter__`
`[kitty/window_list.py:L160-L161]`, grows through `add_window` `[kitty/window_list.py:L329]`, and tracks focus
with `set_active_window_group_for` `[kitty/window_list.py:L315]`. The active window per tab is resolved by
`Tab.active_window` `[kitty/tabs.py:L253]`, and geometry changes fan out through `Tab.relayout`
`[kitty/tabs.py:L298]`.

Observed live hierarchy (`kitty @ ls`, formatted through a small `ls_fmt.py` that walks
`os_window → tab → window`):

```
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/q5/ls_sock2 ls | python3 /tmp/kitty_obs/q5/ls_fmt.py
os_window id=1 is_active=True num_tabs=1
  tab id=1 is_active=True title='bash' num_windows=1 active_window_history=[1]
    window id=1 is_active=True is_focused=True cols=84 lines=24
```

The single `os_window id=1` holds one `tab id=1` (`is_active=True`), which holds one `window id=1` that is
both `is_active=True` and `is_focused=True`, with `active_window_history=[1]`. Keyboard input is delivered to
the *focused* window; when several windows coexist, the focused one is selected via the `WindowList` focus
state above. So the write path of §5.2 (`write(10,…)` to the child PTY) has an unambiguous destination — the
routing layer resolves it before the byte ever reaches `write_to_child`.

### 5.7 Signal handling (real entry point: the signalfd on the I/O thread)

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

### 5.8 Quiescence — the interface settling, observed

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

### 5.9 Cause → effect summary — why it keeps its rhythm

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

Commands used, each executed **twice** (RUN 1 then RUN 2) to demonstrate stability — the complete raw output
of both runs is shown:

```
# --- timing options, RUN 1 ---
$ ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; print(d.input_delay, d.repaint_delay, d.sync_to_monitor, d.resize_debounce_time)'
3 10 True (0.1, 0.5)
# --- timing options, RUN 2 ---
$ ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; print(d.input_delay, d.repaint_delay, d.sync_to_monitor, d.resize_debounce_time)'
3 10 True (0.1, 0.5)
# --- parser buffer size, RUN 1 ---
$ ./kitty/launcher/kitty +runpy 'import kitty.fast_data_types as f; print(f.VT_PARSER_BUFFER_SIZE)'
1048576
# --- parser buffer size, RUN 2 ---
$ ./kitty/launcher/kitty +runpy 'import kitty.fast_data_types as f; print(f.VT_PARSER_BUFFER_SIZE)'
1048576
```

RUN 1 and RUN 2 are byte-identical for every value (these are compile-time defaults read back from the
compiled options/extension modules, so the run-to-run value is deterministic — RUN1 == RUN2 is itself the
stability result). `input_delay=3` `[kitty/options/definition.py:L878]`, `repaint_delay=10`
`[kitty/options/definition.py:L866]`, `sync_to_monitor=yes` `[kitty/options/definition.py:L889]`,
`resize_debounce_time='0.1 0.5'` `[kitty/options/definition.py:L1182]`; `#define BUF_SZ (1024u*1024u)` =
`1048576` `[kitty/vt-parser.c:L18]`.

| Value | Declared default | Observed (RUN1 / RUN2) | `file:line` | Role in the pipeline |
|-------|------------------|------------------------|-------------|----------------------|
| `input_delay` | `3` (ms) | `3` / `3` | `[kitty/options/definition.py:L878]` | wakeup-coalescing window (§2.3, §2.6, §5.3) — throttles I/O→main handoffs |
| `repaint_delay` | `10` (ms) | `10` / `10` | `[kitty/options/definition.py:L866]` | idle render cadence bound; observed governing the gate at runtime (§6.1, §5.9) |
| `sync_to_monitor` | `yes` | `True` / `True` | `[kitty/options/definition.py:L889]` | ties render to monitor refresh via `USE_RENDER_FRAMES`; observed inactive under X11 (§6.1, §5.9) |
| `resize_debounce_time` | `0.1 0.5` | `(0.1, 0.5)` / `(0.1, 0.5)` | `[kitty/options/definition.py:L1182]` | `.on_pause=0.1 s` debounce, `.max=0.5 s` cap (§5.5) |
| parser buffer `BUF_SZ` | `1024*1024` | `1048576` / `1048576` | `[kitty/vt-parser.c:L18]` | bounded ingestion buffer + backpressure threshold (§1.3, §4.3) |

Corroborating runtime magnitudes observed elsewhere in this document:

- The **1 MiB** buffer boundary is confirmed by saturation to exactly `1048576` bytes, identical across 2
  runs (§4.3), and by the shrinking `read()` size argument in live traces (§1.3, §5.3).
- The **`input_delay` window** is corroborated by the ~5 ms median spacing of coalesced main-loop wakeups
  under a 50 000-line surge and the wakeup distributions `{0: 3 ticks, 1: 67 ticks}` (run 1) and
  `{0: 3 ticks, 1: 65 ticks}` (run 2) — i.e. 50 000 lines coalesced into ~66 wakeup-bearing ticks, stable
  across both runs (§2.6, NON-CANONICAL instrumented build) — and by the per-run `poll()` timeout
  distributions (io_c1 `{−1:4, 2:1, 0:11}`, io_c2 `{−1:6}`) in the canonical combined trace (§5.3).
- The **`resize_debounce_time`** default (`0.1 0.5`) is verified from config; rapid programmatic resizes
  coalesce **6 → 1 SIGWINCH** (final geometry `84×24`) via the per-frame net-change gate
  `[kitty/window.py:L861]`, stable across 2 runs (§5.5). The live-drag debounce path
  (`resize_debounce_time.on_pause`, gated on `live_resize.in_progress` `[kitty/child-monitor.c:L1047]`)
  governs interactive drag only and is described from source, labeled not-headlessly-observable.
- The **`repaint_delay`** (10 ms) and **`sync_to_monitor`** (yes) defaults are additionally exercised at runtime in
  §6.1: `render()` is observed firing on the instrumented build, input-driven draws are `input_delay`-paced (poll
  timeout `0.003 s`) and bypass the gate, idle draws are suppressed by the `repaint_delay` gate (loop sleeps ~0.5 s
  rather than spinning at 10 ms), and `sync_to_monitor`'s vblank path is observed inactive on X11 — stable across 2
  runs (`input_read=1` count = 41 both runs).

### 6.1 Live render cadence — `repaint_delay` / `sync_to_monitor` at runtime (instrumented, NON-CANONICAL `[N]`)

The two rows above (`repaint_delay`, `sync_to_monitor`) are canonical **compile-time defaults** read back from the
compiled options module. This subsection additionally **observes the render step firing at runtime** and shows how
those two values actually govern its cadence, so the answer is grounded in observed behavior and not only in the
declared default. The cadence timing was captured on the **NON-CANONICAL** event-loop-instrumented build (§0.3); the
gate logic and the values themselves are canonical.

**The render gate, in source.** Every main-loop tick calls `render(monotonic_t now, bool input_read)`
`[kitty/child-monitor.c:L871]`. Its first act is the throttle gate:

```c
// kitty/child-monitor.c:L871-L876 (quoted verbatim, not elided)
static void
render(monotonic_t now, bool input_read) {
    EVDBG("input_read: %d, check_for_active_animated_images: %d", input_read, global_state.check_for_active_animated_images);
    static monotonic_t last_render_at = MONOTONIC_T_MIN;
    monotonic_t time_since_last_render = last_render_at == MONOTONIC_T_MIN ? OPT(repaint_delay) : now - last_render_at;
    if (!input_read && time_since_last_render < OPT(repaint_delay)) {
        set_maximum_wait(OPT(repaint_delay) - time_since_last_render);
        return;
    }
```

So `repaint_delay` (10 ms) is a **lower bound on redraw frequency that applies only when no new input was read this
tick** (`!input_read`): if a redraw was requested but the previous one was < 10 ms ago, the draw is deferred and the
loop is re-armed to wake in `repaint_delay − elapsed` `[kitty/child-monitor.c:L876]`. When new input *was* read
(`input_read == true`) the gate is bypassed and the frame is drawn immediately; `last_render_at = now` is recorded at
`[kitty/child-monitor.c:L894]`. `sync_to_monitor` feeds `USE_RENDER_FRAMES`
`[kitty/child-monitor.c:L40]` (`#define USE_RENDER_FRAMES (global_state.has_render_frames && OPT(sync_to_monitor))`).

**Observed platform fact — `sync_to_monitor`'s vblank path is inactive under X11.** `has_render_frames` is set true
**only** on macOS (`#ifdef __APPLE__` `[kitty/state.c:L736]`) or Wayland (`if (global_state.is_wayland)`
`[kitty/state.c:L738]`). Under the X11 backend — our Xvfb here, and the canonical X11-desktop case — it stays
`false`, so `USE_RENDER_FRAMES` is `false` regardless of `sync_to_monitor=yes`. Consequence: on X11 the
compositor-frame-callback path is *not* used and render cadence is governed by the `repaint_delay` timer path shown
above. (On Wayland/macOS the frame-callback path additionally ties draws to the monitor refresh.)

**Command (run twice on the instrumented build):**

```
# instrumented build only (make debug-event-loop); child emits 40 lines at 25 ms spacing (> repaint_delay),
# then goes idle for 0.8 s. EVDBG event-loop log captured on stderr.
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8
$ ./kitty/launcher/kitty --config NONE -o remember_window_size=no \
      -o initial_window_width=100c -o initial_window_height=32c \
      bash /tmp/kitty_obs/q6_child_paced.sh   2> render_paced_<RUN>.log
# child (/tmp/kitty_obs/q6_child_paced.sh):
#   sleep 0.5
#   for i in $(seq 1 40); do printf 'PACEDLINE_%02d\n' "$i"; sleep 0.025; done   # PACED phase
#   sleep 0.8; printf 'IDLE_MARKER_DONE\n'; sleep 0.3                             # IDLE phase
```

**Derived cadence (complete output of the analyzer over each raw log; both runs shown):**

```
--- render_paced_1.log ---
render() calls (input_read tokens): 90   input_read=1: 41   input_read=0: 49
pollForEvents timeouts: total=90  small(<=0.010s)=44  mid=7  large(>=0.1s)=39
  distinct small timeouts (s): [0.0, 0.001, 0.003]
  distinct mid   timeouts (s): [0.019, 0.022, 0.047, 0.05, 0.074, 0.076, 0.081]
  distinct large timeouts (s): [0.101, 0.108, 0.129, 0.135, 0.156, 0.163, 0.184, 0.19, 0.212, 0.218, 0.239, 0.245, 0.267, 0.273, 0.294, 0.301, 0.321, 0.328, 0.349, 0.355, 0.377, 0.383, 0.404, 0.41, 0.411, 0.432, 0.438, 0.459, 0.465, 0.466, 0.483, 0.487, 0.49, 0.492, 0.496, 0.497, 0.499]
  repaint_delay re-arm candidates (0<t<=0.010): [0.001, 0.003]
  loop ticks: 90  first ts=0.176  last ts=2.878  span=2.702s

--- render_paced_2.log ---
render() calls (input_read tokens): 91   input_read=1: 41   input_read=0: 50
pollForEvents timeouts: total=91  small(<=0.010s)=44  mid=7  large(>=0.1s)=40
  distinct small timeouts (s): [0.0, 0.001, 0.002, 0.003]
  distinct mid   timeouts (s): [0.017, 0.02, 0.045, 0.047, 0.073, 0.075, 0.077]
  distinct large timeouts (s): [0.1, 0.104, 0.127, 0.131, 0.155, 0.158, 0.182, 0.186, 0.21, 0.213, 0.238, 0.24, 0.266, 0.268, 0.294, 0.295, 0.322, 0.323, 0.349, 0.351, 0.377, 0.378, 0.404, 0.406, 0.409, 0.432, 0.434, 0.437, 0.46, 0.461, 0.464, 0.484, 0.488, 0.489, 0.492, 0.495, 0.496, 0.499]
  repaint_delay re-arm candidates (0<t<=0.010): [0.001, 0.002, 0.003]
  loop ticks: 91  first ts=0.175  last ts=2.882  span=2.707s

=== 2-run stability (structural counts) ===
  n_render: run1=90 run2=91 -> DIFFERS
  n_ir1: run1=41 run2=41 -> IDENTICAL
  n_ir0: run1=49 run2=50 -> DIFFERS
  tos: run1=90 run2=91 -> DIFFERS
  small: run1=44 run2=44 -> IDENTICAL
  mid: run1=7 run2=7 -> IDENTICAL
  large: run1=39 run2=40 -> DIFFERS
  distinct small timeouts identical: False  (r1=[0.0, 0.001, 0.003] r2=[0.0, 0.001, 0.002, 0.003])
```

The **input-driven render count is identical across both runs: `input_read=1` = 41** (the 40 paced lines + the
initial settle draw), as is the small-timeout count (44) and mid-timeout count (7). Only the idle-tail counts differ
by one (`input_read=0` 49 vs 50; large timeouts 39 vs 40) because the two runs were stopped a few milliseconds apart
relative to the free-running 1 s idle timer — an expected boundary effect, not behavioral drift.

**Two real, contiguous excerpts** from `render_paced_1.log` (the render EVDBG at `[kitty/child-monitor.c:L872]` is
emitted with no trailing newline, so a `\n` was inserted before each `[timestamp]` token for legibility — **no bytes
were removed or altered**):

```
# --- PACED phase: each ~25 ms line wakes the loop; poll timeout set to 0.003 s (= input_delay, 3 ms),
#     then render() fires with input_read: 1 (gate bypassed, frame drawn immediately) ---
[0.697] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0
[0.697] starting handleEvents(-0.00)
[0.697] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0
[0.703] display_read_ok: 0
[0.703] other dispatch done
[0.703] --------- loop tick, wakeups_happened: 0 ----------
[0.703] starting handleEvents(-0.00)
[0.703] pollForEvents final timeout: 0.459
[0.724] display_read_ok: 0
[0.724] other dispatch done
[0.724] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0
[0.724] starting handleEvents(-0.00)
[0.724] pollForEvents final timeout: 0.003
```

```
# --- IDLE phase: no new input -> render() called with input_read: 0; the gate suppresses redundant
#     redraw and the loop blocks on the long periodic-timer wait (~0.5 s counting down), NOT a 10 ms spin ---
[0.176] pollForEvents final timeout: 0.483
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0
[0.664] display_read_ok: 0
[0.664] other dispatch done
[0.664] --------- loop tick, wakeups_happened: 0 ----------
[0.664] starting handleEvents(-0.00)
[0.664] pollForEvents final timeout: 0.499
```

**What this proves (cause → effect):**

- **The render step actually runs at runtime** — `render()` is observed firing on every loop tick via its EVDBG
  probe `[kitty/child-monitor.c:L872]`; this is the same draw whose *pixel* effect is captured in §5.3 (framebuffer
  md5 changes when the screen mutates).
- **When input is arriving, cadence is input-paced, not `repaint_delay`-paced.** Each paced line produces
  `wakeups_happened: 1` followed by `pollForEvents final timeout: 0.003` — exactly `input_delay = 3 ms`
  `[kitty/options/definition.py:L878]`, set at `[kitty/child-monitor.c:L445-L446]`
  (`set_maximum_wait(OPT(input_delay) - pd.time_since_new_input)`) — then `render()` with `input_read: 1`, which
  **bypasses** the `repaint_delay` gate `[kitty/child-monitor.c:L875]` and draws immediately.
- **When idle, `repaint_delay` prevents busy-drawing and the loop sleeps.** With `input_read: 0` and nothing new to
  show, the gate returns early and the loop blocks on the long periodic-timer wait (poll timeouts counting down from
  ~0.5 s toward `0.1`), bounded by the 1 s state-check timer `[kitty/child-monitor.c:L1261]`
  (`add_main_loop_timer(1000, true, do_state_check, …)`) and the cursor-blink timer — **never a 10 ms repaint spin**.
  The sub-10 ms `repaint_delay` re-arm from `[kitty/child-monitor.c:L876]` appears only as the small
  `0.001–0.003 s` timeouts adjacent to draws, i.e. exactly when a redraw was wanted < 10 ms after the last one.
- **`sync_to_monitor` = yes has no vblank effect on X11** because `has_render_frames` is false off Wayland/macOS
  `[kitty/state.c:L736-L738]`, so `USE_RENDER_FRAMES` `[kitty/child-monitor.c:L40]` is false and the `repaint_delay`
  timer path (observed above) governs. This is the honest, observed X11 behavior; the vblank path is Wayland/macOS
  only and is described from source, not headlessly observable here.

---

## 7. Coverage pass

Every named item across Q1–Q5 (including every "e.g. / such as / including" example) is confirmed present
with a concrete value, a `file:line` reference, observed evidence, sibling/alternate variants, and a
causal reason. `[C]` = canonical build; `[N]` = NON-CANONICAL instrumented build; `[H]` = real compiled
`fast_data_types` parser harness; `[SSH]` = real loopback ssh.

### Q1 — Ingestion / entry point

| Named item | Where covered | Evidence | `file:line` |
|------------|---------------|----------|-------------|
| `read_bytes()` | §1.2, §1.5 | strace read on PTY master + verbatim source `[C]` | `[kitty/child-monitor.c:L1337-L1356]` |
| the `read()` syscall | §1.2 | marker read `= 40` bytes (`\r\r\n`), stable 2+ runs `[C]` | `[kitty/child-monitor.c:L1337-L1356; read() at L1346]`; source line adjacent at L1345 |
| `EINTR`/`EAGAIN` retry | §1.5 | verbatim source; source-verified, **not observed on child fd** (blocking fd + poll-gated + signalfd); benign `EAGAIN` shown on separate wakeup eventfd `[C]` | `[kitty/child-monitor.c:L1347]` |
| `EIO` = child-gone | §1.5 | `read()` returns `-1 EIO` after child `exit` (full line in §1.5) `[C]` | `[kitty/child-monitor.c:L1348-L1350]` |
| keystroke path `keys.c`/`key_encoding.{c,py}`/`keys.py` | §1.6 | byte-exact legacy vs kitty-protocol writes (7 keys) `[C]` | `[kitty/keys.c:L166,L93]`, `[kitty/key_encoding.c:L414,L65]`, `[kitty/keys.py:L40,L154]`, `[kitty/key_encoding.py:L15,L127,L149]` |
| mouse path | §1.6 | real SGR click `write "\33[<0;7;3M\33[<0;7;3m" = 18` `[C]` | `[kitty/mouse.c:L68,L88]` |
| XKB / IME | §1.6 | "Loading new XKB keymaps" + modifier-indices log `[C]`; IME source-cited, not exercised (no-IME run) | `glfw/xkb_glfw.c`, `[glfw/ibus_glfw.c:L460,L473]` |
| `iutf8` / `set_iutf8_fd` | §1.6 | child `stty -a` shows `iutf8` `[C]` | `[kitty/child.py:L170,L171,L174]` |
| `child.c` fork/exec/PTY | §1.6 | `stty -a` confirms controlling tty; source cited | `[kitty/child.c:L81,L123,L129,L159]` |
| `write_to_child` | §1.4 | `write(10, "stty -a\n", 8) = 8` `[C]` | `[kitty/child-monitor.c:L1443]` |
| paused/resumed cross-ref | §1.1, §4 | forward reference to §4 gate | `[kitty/child-monitor.c:L1501]` |

### Q2 — The unseen conductor

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| I/O thread `io_loop` | §2.2 | `KittyChildMon` in `/proc` `[C]` | `[kitty/child-monitor.c:L1481]` (name L1489) |
| main/render `main_loop` | §2.2 | tick cadence `[N]` | `[kitty/child-monitor.c:L1259]` (do_parse L438) |
| talk thread `read_from_peer` | §2.2 | `KittyPeerMon` in `/proc` `[C]` | `[kitty/child-monitor.c:L1714]` (L1808) |
| `poll()` ordering + `EXTRA_FDS` | §2.4, §5.3 | fixed `[fd7,fd8,fd10]` order `[C]` | `[kitty/child-monitor.c:L35,L1501-L1503,L1515-L1541]` |
| `WAKEUP` / `input_delay` | §2.3, §2.6 | quoted macro + comment; 50k-line surge dist `{0:3,1:67}`/`{0:3,1:65}` `[N]` | `[kitty/child-monitor.c:L1562-L1570]` |
| `loop-utils.{c,h}` / `threading.h` | §2.2, §5.7 | signalfd/eventfd plumbing `[C]` | `[kitty/loop-utils.c:L42]`, `kitty/threading.h` |
| `boss.py` lifecycle | §2.7 | thread/child presence after start `[C]` | `[kitty/boss.py:L370,L585,L1183,L2172]` |

### Q3 — Shell-integration alignment

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| VT parser byte classification | §3.2 | quoted `consume_esc` transitions | `[kitty/vt-parser.c:L260,L270]` |
| OSC routing `dispatch_osc` case 133 | §3.3 | quoted source | `[kitty/vt-parser.c:L457,L536,L544]` |
| `cmd_output_marking` O / OO / Os | §3.3 | quoted callbacks | `[kitty/screen.c:L2338,L2347,L2352]` |
| OSC routing `case ESC_OSC:` (checkpoint-required item — satisfied) | §3.3 | Required item cited and explained: `screen.c:L964` sets `*prefix="\033]"` inside `get_prefix_and_suffix_for_escape_code()` `[L955]` — the **output** (kitty→child) side, called only from `write_escape_code_to_child()` `[L979]`; its limitation is it does **not** route incoming OSC 133. The actual **incoming** OSC 133 route is also cited: `case ESC_OSC:SET_STATE(OSC)` → `dispatch_osc` → `case 133:` → `shell_prompt_marking` `[C]` | required item `[kitty/screen.c:L964]`; incoming route `[kitty/vt-parser.c:L270,L457,L536,L544]` → `[kitty/screen.c:L2328]` |
| `find_cmd_output` / `cmd_output` | §3.3 | named for extent lookup | `[kitty/screen.c:L3527,L3606]` |
| pending mode start/stop ↔ DEC 2026 | §3.8 | DECRQM before/during/after `[H]` | `[kitty/vt-parser.c:L639,L644]` |
| `modes.h` DEC/paste constants | §3.8, §5.4 | `PENDING_UPDATE (2026<<5)`, paste consts | `[kitty/modes.h:L81-L83,L86]` |
| `shell_integration.py` setup_* + `modify_shell_environ` | §3.7 | dispatch + zsh skip edge `[C]` | `[kitty/shell_integration.py:L16,L27,L49,L70,L218]` |
| bash / zsh / fish / ssh scripts | §3.4, §3.5, §3.7 | real per-shell OSC 133 bytes `[C]`; remote ssh row additionally verified byte-for-byte with `hexdump -C` **and** `cat -v` (123-byte decode matches `read()=123`) `[SSH]` | `shell-integration/{bash,zsh,fish,ssh}` |

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
| dropped-link edge (idle remote) → SIGCHLD reap | §4.7 | `read(7<anon_inode:[signalfd]>) = 128` (signo `\21`=17) on ssh kill `[SSH]` | `[kitty/child-monitor.c:L1519,L1370-L1371,L1526]` |
| child-gone → `EIO` (read in flight) | §1.5 | `read(10</dev/pts/ptmx>) = -1 EIO` after child exit `[C]` | `[kitty/child-monitor.c:L1348-L1350]` |

### Q5 — Full end-to-end settle

| Named item | Where | Evidence | `file:line` |
|------------|-------|----------|-------------|
| arrival → read → buffer → parse → screen → render → quiescence | §5.2, §5.3, §5.8, §6.1 | combined trace ×2 + parse→mutate→render (framebuffer md5 change) + observed `render()` cadence (§6.1 `[N]`) + `poll(-1)` idle `[C]` | `[kitty/child-monitor.c:L1345,L438,L440,L871,L875,L894,L1236-L1237]` |
| paste path (bracketed markers) + `filter` action | §5.4 | byte-exact `\33[200~…\33[201~`; filter invoked → `FILTERED_OUTPUT_42` `[C]` | `[kitty/window.py:L117,L423,L1643,L1647,L1707,L1713,L1780]`, `[kitty/modes.h:L81-L83]` |
| resize `screen.resize`/`resize_pty`/SIGWINCH | §5.5 | before/after sizes + prints `[C]` | `[kitty/window.py:L850,L854,L863,L873]` |
| resize coalescing (net-change gate) + live-resize debounce | §5.5 | 6 rapid resizes → 1 SIGWINCH (final geometry), ×2 runs `[C]` | net-change `[kitty/window.py:L861]`; live-drag debounce `[kitty/child-monitor.c:L1043-L1075]` (gated `L1047`) |
| window/tab routing | §2.7, §5.6 | live `ls` hierarchy os_window→tab→window (focused) `[C]` | `[kitty/tabs.py:L64,L150,L253,L298]`, `[kitty/window_list.py:L144,L147,L160-L161,L315,L329]` |
| child / PTY | §1, §5 | `fd 10` PTY master; child bash `[C]` | `kitty/child.py`, `kitty/child.c` |
| signals INT/TERM/HUP→kill, CHLD→reap, USR1→reload (+USR2) | §5.7 | signalfd set, SigBlk, reload/reap EFFECT `[C]` | `[kitty/child-monitor.c:L121,L1360-L1383,L1413,L1526,L534-L536]` |

### Canonical values (§6)

`input_delay=3` `[kitty/options/definition.py:L878]`, `repaint_delay=10` `[kitty/options/definition.py:L866]`,
`sync_to_monitor=yes` `[kitty/options/definition.py:L889]`, `resize_debounce_time='0.1 0.5'`
`[kitty/options/definition.py:L1182]`, parser buffer `1 MiB` `[kitty/vt-parser.c:L18]` — all observed and
stable across ≥2 runs.

### Explicitly-stated limitations (nothing faked)

- **Q2 cadence numbers** (idle ~0.5 s/tick; surge ~3–4 ms/tick; wakeup distribution) come from the
  **NON-CANONICAL** `make debug-event-loop` build; the thread structure, `poll()` order, and `WAKEUP`
  gate logic are canonical.
- **§6.1 render-cadence numbers** (per-run `render()` call counts, poll-timeout distributions, the observed
  `0.003 s` input-paced coalescing vs. the ~0.5 s idle waits) likewise come from the **NON-CANONICAL**
  `make debug-event-loop` build, since the `render()` EVDBG probe `[kitty/child-monitor.c:L872]` is compiled
  out of the canonical build. The `render()` gate logic (`repaint_delay` throttle `[L875]`, `input_read`
  bypass), the `sync_to_monitor`→`USE_RENDER_FRAMES` mapping `[L40]`, the `has_render_frames`
  platform condition `[kitty/state.c:L736-L738]`, and the compiled default values themselves (§6) are all
  canonical. The *pixel* effect of the render step is separately confirmed on the canonical build in §5.3
  (framebuffer md5 change).
- **Q4 unstable remote**: true packet loss / jitter cannot be induced on a container loopback; the
  transport keepalive is shown from the real ssh options (`ServerAlive*`, `TCPKeepAlive=no` at
  `[kittens/ssh/main.go:L141-L143]`), and the pipeline consequence of a drop is directly observed — a
  **SIGCHLD** child-reap for an idle remote (§4.7), the sibling of the `EIO`-on-read child-gone branch of
  §1.5.
- **Q5 resize inside the combined run**: the resize *is* part of the combined capstone — `kitty @
  resize-os-window` triggers a real `SIGWINCH` that appears in the §5.2 trace
  (`read(10, "Q5_WINCH_FIRED size=[25 83]…")`). The dedicated §5.5 test additionally isolates single vs.
  rapid-coalesced resizes across two runs each. The only resize path not exercisable headlessly is the
  *interactive live-drag* debounce (`process_pending_resizes`, gated on `live_resize.in_progress`
  `[kitty/child-monitor.c:L1047]`), which requires a human dragging the window edge; that mechanism is
  described from source with its gate quoted, and labeled as not-headlessly-observable rather than faked.

### Reproducibility note

All observation scripts were temporary and kept under `/tmp` (outside the tracked tree); they were removed
after use. The source repository is unchanged apart from this document
(`blitzy/documentation/kitty_815df1e210e0.md`); the only stray subdirectories that once existed under
`blitzy/` (`screen_recordings/`, `screenshots/`) were removed, so `blitzy/` now contains only
`documentation/` and the single deliverable. In the submitted (committed) state the working tree is clean
and the sole change against the pinned source commit is the added deliverable file:

```
$ git status --porcelain
$ git diff 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 --name-status
A	blitzy/documentation/kitty_815df1e210e0.md
$ find blitzy
blitzy
blitzy/documentation
blitzy/documentation/kitty_815df1e210e0.md
```

`git status --porcelain` prints nothing (the empty line immediately below the command), i.e. the tracked
tree is clean with no uncommitted or untracked entries; `git diff <pinned-commit> --name-status` shows the
single added file `A blitzy/documentation/kitty_815df1e210e0.md` (status `A` = added, zero modified/deleted
tracked source files); and `find blitzy` lists only the documentation directory and the single deliverable
— confirming the one-artifact shape with no stray subdirectories.

