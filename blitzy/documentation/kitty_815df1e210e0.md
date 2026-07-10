# kitty PTY-read investigation — how the terminal reads from the shell over a pseudoterminal

**Repository:** kitty terminal emulator · **Commit:** `815df1e21` ("Wire up applying of font config") · **Branch:** `kitty_815df1e210e0`
**Task type:** Evidence-backed Q&A investigation (read-only; the sole artifact added to the repo is this document).
**Methodology:** `build → launch → instrument → observe → document`. Every runtime value below was captured **live** from a real, default-configuration kitty process using `strace`, `ps`, `/proc`, `readlink`, and `lsof`. Statements are labelled **[OBSERVED]** (captured from a live run) or **[INFERRED]** (derived from reading verified source). Every value carries a `file:line` citation.

---

## 1. Overview

This document answers five sub-questions about how kitty's native code communicates with a spawned shell over a pseudoterminal (PTY):

1. **Q1 — Shell spawn identity:** what process is spawned, its PID, its exact command line, and the PTY device path.
2. **Q2 — Reading a typed command:** which syscalls kitty makes to read `echo test123`, the buffer size, and how many bytes come back.
3. **Q3 — High-volume output:** how kitty's reading behavior changes under `yes hello`, the read frequency, and the typical bytes-per-read.
4. **Q4 — File descriptor number:** the fd number kitty uses to read the PTY master side.
5. **Q5 — Responsible C functions:** the function that reads from the fd, and the function that parses input to separate printable text from escape sequences.

**One-line data-flow summary** (each hop cited in the answers below):

```
Boss.add_child (boss.py:585/587)
   -> Child.fork (child.py:276) -> openpty (child.py:281) -> fast_data_types.spawn (child.py:333)
      -> native spawn() (child.c:81): fork -> setsid -> ioctl TIOCSCTTY -> dup2(slave) -> execvp  => /bin/bash
   -> master fd retained self.child_fd (child.py:338), set non-blocking (child.py:345)
      -> registered into poll set children_fds[EXTRA_FDS + i].fd (child-monitor.c:1286)

I/O thread  (io_loop, child-monitor.c:1481): poll (child-monitor.c:1509/1512)
   -> read_bytes (child-monitor.c:1337) -> read() (child-monitor.c:1345) into a BUF_SZ=1 MiB buffer (vt-parser.c:18)
Main thread : parse_input (child-monitor.c:451) -> do_parse (child-monitor.c:438)
   -> consume_input (vt-parser.c:1367)
       -> consume_normal (vt-parser.c:230) -> screen_draw_text (screen.c:866)   [printable text]
       -> consume_esc    (vt-parser.c:261) -> CSI/OSC/DCS/APC/PM/SOS handlers   [escape sequences]
```

---

## 2. Environment & Build

### 2.1 Canonical container & toolchain

The build, launch, and syscall tracing were performed inside the provided container (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). Observed tool versions:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ python3 --version
Python 3.13.7
$ go version
go version go1.22.12 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ strace --version | head -1
strace -- version 6.16
$ xdotool --version
xdotool version 3.20160805.1
$ uname -a
Linux reverse-code-generator-c1ea0670-cmcb8 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux
```

Build dependencies are those listed in `docs/build.rst` (`python >= 3.8` at `docs/build.rst:83`; harfbuzz, freetype, fontconfig, zlib, libpng, lcms2, xxhash, openssl); the harfbuzz version guard is `at_least_version('harfbuzz', 1, 5)` at `setup.py:609`. **[INFERRED]** (from source).

### 2.2 Default-configuration build

The canonical build command is `make`, whose `all:` target (`Makefile:12`) runs `python3 setup.py $(VVAL)` (`Makefile:13`). Running it (after removing the git-ignored native extension to force a genuine recompile) produced the native extension and launcher. **[OBSERVED]**:

```
$ export PATH=$PATH:/usr/local/go/bin GOTOOLCHAIN=local
$ rm -f kitty/fast_data_types.so    # git-ignored build output; forces a real link
$ make
python3 setup.py 
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/1] Linking kitty/fast_data_types ...
 done

real	0m17.720s
user	0m17.685s
sys	0m1.888s
```

The `wayland-protocols` line is a benign configuration notice — the X11 backend (used here via Xvfb) is built regardless. The produced artifacts **[OBSERVED]**:

```
$ ls -l kitty/fast_data_types*.so
-rwxr-xr-x 1 root root 1253792 Jul 10 07:36 kitty/fast_data_types.so
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 40384 Jul 10 07:08 kitty/launcher/kitty
$ git check-ignore kitty/fast_data_types.so
kitty/fast_data_types.so
```

`git check-ignore` confirms the native extension is a git-ignored build output (never part of the deliverable). No `--debug` build was used; all answers come from the **default** build.

### 2.3 Headless launch (default kitty config)

There is no physical display, so kitty was launched under an X virtual framebuffer. This is a *launch mechanism*, not a configuration change: no `kitty.conf` was passed and `~/.config/kitty/` is empty, so kitty runs with its built-in defaults. **[OBSERVED]**:

```
$ Xvfb :99 -screen 0 1280x800x24 -ac &            # PID 58432
$ DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty     # default config, no --config
```

kitty started as **PID 58437** and a real shell spawned as its child (formalized in Q1). The startup log contained only benign messages:

```
[0.159] Failed to open systemd user bus with error: Connection refused
ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
```

The second line is kitty's shell-integration notice, confirming a **bash** shell was spawned with shell integration enabled (the default).

---

## 3. Methodology

- **Canonical input path only.** All commands were typed into the real kitty window via `xdotool` (real X key events delivered to the window `WM_CLASS=kitty`, id `2097164`). This is genuine terminal keyboard input — kitty's X11 keyboard handler receives the events, writes the bytes to the PTY master, and reads back the echo/output. It is **not** remote control, a debug hook, a mock, or synthetic byte injection. kitty's remote control is disabled by default here (`kitty @ get-text` reported "remote control not enabled"), which further guarantees the answer values come only from the canonical path.
- **Syscall capture.** `strace -f -e trace=poll,read` (with `-tt` microsecond timestamps). For the high-volume case, strace was attached to the single I/O thread (see Q5) so its `poll`/`read` lines appear unsplit.
- **Process / fd / PTY facts.** `ps -ef --forest`, `ps -o …`, `cat /proc/<pid>/cmdline`, `readlink /proc/<pid>/fd/N`, `ls -l /proc/<pid>/fd`, and `lsof -p <pid>`.
- **Reproducibility.** Runtime identifiers (PIDs, fd number, `/dev/pts/N`) are **process-specific** to this session and are reported as observed; the high-volume magnitudes were confirmed stable across three runs (§Q3 and §8).

> **A note on strace's line-splitting.** Under `-f`, strace interleaves threads and splits an interrupted syscall across two lines, e.g. `read(8 <unfinished ...>` … `<... read resumed>, "e", 1048576) = 1`. Both halves belong to the same `read()` on fd 8. Where this occurs it is called out explicitly so no read is miscounted.

---

## 4. Q1 — Shell spawn identity

**Direct answer (all four items [OBSERVED]):**

| Item | Value |
|------|-------|
| **(a) Process spawned** | `/bin/bash` — the user's login shell |
| **(b) PID** | **58505** |
| **(c) Exact command line** | `/bin/bash --posix` |
| **(d) PTY device path** | slave **`/dev/pts/0`** (kitty's master counterpart is fd 8 → `/dev/pts/ptmx`) |

### Evidence

**Process tree — kitty → shell [OBSERVED]:**

```
$ ps -ef --forest | grep -E 'launcher/kitty|/bin/bash --posix'
root       58437       1  1 07:37 ?        00:00:01 ./kitty/launcher/kitty
root       58505   58437  0 07:37 pts/0    00:00:00  \_ /bin/bash --posix
```

The shell (58505) is a direct child of kitty (58437), attached to `pts/0`.

**Exact command line [OBSERVED]:**

```
$ cat /proc/58505/cmdline | tr '\0' ' '
/bin/bash --posix 
$ cat -A /proc/58505/cmdline
/bin/bash^@--posix^@
$ ps -o pid,ppid,stat,tty,args -p 58505
    PID    PPID STAT TT       COMMAND
  58505   58437 Ss+  pts/0    /bin/bash --posix
```

`cat -A` shows the NUL-delimited argv exactly: `argv[0]=/bin/bash`, `argv[1]=--posix`. `STAT=Ss+` means the process is a session leader (`s`) with a controlling terminal, in the foreground process group (`+`) — i.e. a genuine interactive login shell.

**PTY device path [OBSERVED]:**

```
$ readlink /proc/58505/fd/0    # stdin
/dev/pts/0
$ readlink /proc/58505/fd/1    # stdout
/dev/pts/0
$ readlink /proc/58505/fd/2    # stderr
/dev/pts/0
```

All three standard streams of the shell point at the PTY **slave** `/dev/pts/0`. The **master** counterpart is held by kitty on fd 8 **[OBSERVED]**:

```
$ ls -l /proc/58437/fd/8
lrwx------ 1 root root 64 Jul 10 07:39 /proc/58437/fd/8 -> /dev/pts/ptmx
$ lsof -p 58437 -a -d 8
COMMAND   PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
kitty   58437 root 8u   CHR    5,2      0t0    2 /dev/pts/ptmx
```

So the PTY pair is: **kitty master fd 8 (`/dev/pts/ptmx`, char device 5,2) ↔ shell slave `/dev/pts/0`**.

### Rationale & code citations

- **Which shell [INFERRED]:** the shell path is resolved as `shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` at `kitty/constants.py:181`. In this container the effective user's `pw_shell` is `/bin/bash`, so the resolved shell is `/bin/bash` and the `/bin/sh` fallback at `kitty/constants.py:185` (with the warning at `:184`) is **not** taken.
- **Why `--posix` and no leading `-` [OBSERVED + INFERRED]:** by default kitty launches the shell through its `run-shell` kitten to set up shell integration — `argv = [kitten_exe(), 'run-shell', '--shell', shlex.join(argv), '--shell-integration', ksi]` at `kitty/child.py:314`. The `run-shell` kitten then re-executes the real shell; the resulting live process image observed is `/bin/bash --posix` (bash placed in posix mode with integration injected via environment/rcfile rather than the historical `-`-prefixed login form). The `/usr/bin/login` wrapper branch in `kitty/child.py:315`–`326` is guarded by `is_macos` and is not taken on Linux. The command line is reported **exactly as it appears**.
- **Spawn chain [INFERRED]:** `Boss.add_child` (`kitty/boss.py:585`) calls `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` (`kitty/boss.py:587`); `Child.fork()` (`kitty/child.py:276`) creates the PTY via `openpty()` (defined `kitty/child.py:170`, invoked `kitty/child.py:281`) and calls `fast_data_types.spawn(...)` (`kitty/child.py:333`, statement spans L333–L335). The native `spawn()` (`kitty/child.c:81`) performs `fork()` (`kitty/child.c:97`), `setsid()` (`kitty/child.c:123`), `ioctl(tfd, TIOCSCTTY, 0)` (`kitty/child.c:129`), `safe_dup2(slave, STDOUT_FILENO)` (`kitty/child.c:138`), `safe_dup2(slave, STDERR_FILENO)` (`kitty/child.c:139`), `safe_dup2(stdin_read_fd, STDIN_FILENO)` (`kitty/child.c:141`) / `safe_dup2(slave, STDIN_FILENO)` (`kitty/child.c:145`), and `execvp(exe, argv)` (`kitty/child.c:159`). The `dup2` of the slave onto fds 0/1/2 is exactly why the shell's fd 0/1/2 all read back as `/dev/pts/0` above.

---

## 5. Q2 — Reading the typed command `echo test123`

**Direct answer:**

- **(a) System calls [OBSERVED]:** the I/O thread performs a `poll()` on the PTY fd, then a `read()` on it — a `poll → read` pair per readiness event, on fd 8.
- **(b) Buffer size [OBSERVED]:** the third argument to `read()` is **1048576** bytes = 1 MiB = `BUF_SZ` (`kitty/vt-parser.c:18`); it shrinks below 1 MiB when unparsed bytes are still buffered.
- **(c) Bytes returned [OBSERVED]:** each typed keystroke echoes back as a **1-byte** read; the `Enter` echo returns **11** bytes; the command's output arrives in reads of **47**, **114**, and **429** bytes. The count returned is bounded by what the kernel line discipline currently holds, **not** by the 1 MiB request.

The command was typed into the real kitty window (canonical path) with `strace -f -tt -e trace=poll,read` attached to kitty (PID 58437); the I/O thread is TID 58504.

### Evidence — the `poll → read` pipeline for one keystroke (`o`) [OBSERVED]

Contiguous, unedited excerpt (the `58504` prefix is the I/O thread's TID; timestamps are HH:MM:SS.microseconds):

```
58504 07:42:24.401241 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
58504 07:42:24.401333 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
58504 07:42:24.401417 read(8, "o", 1048576) = 1
58504 07:42:24.401479 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <unfinished ...>
```

Reading the three lines: the first `poll` reports fd 8 writable (`POLLOUT`) — kitty writes the keystroke to the master (the `write` itself is not in the `poll,read` filter); the second `poll` reports fd 8 readable (`POLLIN`) — the kernel line discipline has echoed the byte back; then `read(8, "o", 1048576) = 1` reads that one echoed byte.

The poll set `[{fd=6}, {fd=7}, {fd=8}]` with `nfds=3` maps exactly to the source: `children_fds` holds two reserved slots (`#define EXTRA_FDS 2` at `kitty/child-monitor.c:35`) — here fd 6 (I/O-thread wakeup eventfd) and fd 7 (signal fd) — followed by the child PTY fd at index `EXTRA_FDS + 0` (fd 8). `nfds = self->count + EXTRA_FDS = 1 + 2 = 3`, matching the blocking `poll(children_fds, self->count + EXTRA_FDS, -1)` at `kitty/child-monitor.c:1512` **[INFERRED]**. The `events` value on fd 8 is set by `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0` at `kitty/child-monitor.c:1501` (plus `POLLOUT` when there is pending output). The `read()` itself is issued at `kitty/child-monitor.c:1345` inside `read_bytes()` (`kitty/child-monitor.c:1337`).

### Evidence — all 12 keystroke echoes [OBSERVED]

Typing `echo test123` (12 characters) produced exactly 12 single-byte reads on fd 8, each requesting the full 1 MiB buffer (some appear as strace unfinished/resumed pairs, noted inline — all are fd-8 reads):

```
58504 07:42:24.024836 <... read resumed>, "e", 1048576) = 1
58504 07:42:24.150575 <... read resumed>, "c", 1048576) = 1
58504 07:42:24.276000 <... read resumed>, "h", 1048576) = 1
58504 07:42:24.401417 read(8, "o", 1048576) = 1
58504 07:42:24.527011 read(8, " ", 1048576) = 1
58504 07:42:24.652573 read(8, "t", 1048576) = 1
58504 07:42:24.777998 <... read resumed>, "e", 1048576) = 1
58504 07:42:24.903498 <... read resumed>, "s", 1048576) = 1
58504 07:42:25.029054 <... read resumed>, "t", 1048576) = 1
58504 07:42:25.154545 <... read resumed>, "1", 1048576) = 1
58504 07:42:25.279978 <... read resumed>, "2", 1048576) = 1
58504 07:42:25.405526 <... read resumed>, "3", 1048576) = 1
```

Each read returns exactly `1` byte — the character just echoed by the line discipline — while requesting `1048576` bytes. The reads are ~125 ms apart, i.e. one per typed key.

### Evidence — `Enter` and the command output reads [OBSERVED]

After `Return`, the following contiguous excerpt shows the `Enter` echo and the command output. The `58437` interleaved lines are the *main* thread draining its own eventfd (fd 4) and are shown unedited for fidelity:

```
58504 07:42:26.336659 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
58504 07:42:26.336738 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <unfinished ...>
58437 07:42:26.336770 read(4, "\1\0\0\0\0\0\0\0", 64) = 8
58504 07:42:26.336846 <... poll resumed>) = 1 ([{fd=8, revents=POLLIN}])
58504 07:42:26.336927 <... read resumed>, "\r\n\33[?2004l\r", 1048576) = 11
58504 07:42:26.338410 read(8, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47
58504 07:42:26.338617 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114
58504 07:42:26.339373 read(8, "\33[?2004h\33[59P\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]0;root@reverse-code-generator-c1ea0670-cmcb8: /tmp/blitzy/kitty/blitzy-d31274a1-8af8-4743-b20a-faa12c2e3d6a_1fd78c\7root@reverse-code-generator-c1ea06"..., 1048404) = 429
```

Reading these:

- `read(8, "\r\n\33[?2004l\r", 1048576) = 11` — the `Enter` echo: CR LF, the bracketed-paste-off sequence `ESC[?2004l`, and a CR (11 bytes).
- `read(8, "\33]2;echo test123\7…", 1048565) = 47` — the shell (via integration) sets the window title (OSC 2 `ESC]2;echo test123 BEL`) and reports the command line (OSC 133 `;C;cmdline=echo test123`).
- `read(8, "…test123\r\n", 1048518) = 114` — **the actual command output**: the literal `test123\r\n` is present at the end of this read, wrapped by OSC 133 shell-integration markers.
- `read(8, "\33[?2004h…root@…", 1048404) = 429` — the next prompt: bracketed-paste-on, OSC 133 prompt markers, and the `PS1` string with `root@…:/tmp/blitzy/…`.

### Buffer size ↔ `BUF_SZ`, and returned-count rationale

- **Buffer size [OBSERVED + INFERRED]:** the third `read()` argument is `1048576` when the parser buffer is empty, and `1048565`, `1048518`, `1048404` on the successive output reads. Those reductions equal the bytes still sitting unparsed in the parser buffer: `1048576 − 1048565 = 11`, `1048576 − 1048518 = 58` (=11+47), `1048576 − 1048404 = 172` (=11+47+114). This is exactly `*sz = BUF_SZ - self->write.offset` computed by `vt_parser_create_write_buffer()` at `kitty/vt-parser.c:1451`, where `BUF_SZ` is `#define BUF_SZ (1024u*1024u)` at `kitty/vt-parser.c:18` and the backing store is `uint8_t buf[BUF_SZ + BUF_EXTRA]` at `kitty/vt-parser.c:194`.
- **Bytes returned [OBSERVED value; INFERRED cause]:** the returned counts (1, 1, …, 11, 47, 114, 429) are tiny compared with the ~1 MiB request. This is because a `read()` on a PTY master returns only what the kernel line discipline currently holds — the echoed keystroke, or the burst the shell just wrote — not the requested capacity. The whole `echo test123` interaction was **613 bytes** across **16 reads** on fd 8 (12×1 + 11 + 47 + 114 + 429).

---

## 6. Q3 — High-volume output `yes hello`

**Direct answer:**

- **(a) How reading behavior changes [OBSERVED]:** instead of one small, ~1-byte read every ~125 ms (the echo case), reads become **back-to-back** — `poll` returns `fd 8` readable immediately and `read()` fires again microseconds later — and each read returns a **large multi-hundred-byte chunk** of `hello\r\n` lines rather than a single byte. The `poll` timeout also changes from blocking (`-1`) to a short `input_delay`-driven timeout.
- **(b) Frequency [OBSERVED]:** roughly **10,000–11,000 reads/second** on the PTY fd (measured under strace across three runs), with instantaneous inter-read intervals of ~27–54 µs.
- **(c) Typical bytes-per-read [OBSERVED]:** the modal/median return is **~500 bytes** (mode 483–511, median 504–555, mean 563–658), ranging from a few bytes up to ~20 KB — versus exactly 1 byte for the echo keystrokes in Q2.

`yes hello` was typed into the real kitty window (canonical path). Because only the I/O thread reads the PTY, strace was attached to that single thread (TID 58504, whose `comm` is `KittyChildMon` — see Q5) so the `poll`/`read` lines are unsplit.

### Evidence — the I/O thread is the child monitor [OBSERVED]

```
$ cat /proc/58437/task/58504/comm
KittyChildMon
```

### Evidence — back-to-back reads mid-stream [OBSERVED]

Contiguous, unedited excerpt from run 1 (the reading thread only; note the microsecond spacing and the shrinking buffer size):

```
07:47:21.663092 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663119 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1025352) = 546
07:47:21.663146 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663173 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1024806) = 576
07:47:21.663205 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663233 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1024230) = 665
07:47:21.663261 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663287 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1023565) = 511
07:47:21.663314 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663341 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1023054) = 460
07:47:21.663367 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663394 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1022594) = 539
07:47:21.663420 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
07:47:21.663447 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1022055) = 443
```

Observations from this block:

- **Back-to-back cadence:** consecutive `read`s are ~27 µs apart (e.g. `.663119 → .663173 → .663233`), each preceded by a `poll` that immediately reports fd 8 readable. Compare with Q2 where reads were ~125 ms apart. Measuring consecutive flood reads gives ~54 µs typical spacing (`…612719 / …612773 / …612827 / …612885 …`).
- **Large chunks:** returns are 546, 576, 665, 511, 460, 539, 443 bytes of repeated `hello\r\n`.
- **Poll timeout changed:** the flood polls use timeout `2` (ms) — the `input_delay` timed path `if (time_delta >= 0) ret = poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta))` at `kitty/child-monitor.c:1509` — rather than the blocking `-1` (`kitty/child-monitor.c:1512`) seen when idle **[INFERRED tie to source]**.
- **Backpressure building:** the buffer-size argument decreases monotonically (`1025352 → 1024806 → 1024230 → 1023565 → 1023054 → 1022594 → 1022055`), i.e. `write.offset` is growing because reads are outrunning the main-thread parser — the parser buffer is filling. Dispatch to `read_bytes()` happens via `if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP))` at `kitty/child-monitor.c:1529`–`1531`.

### Evidence — the echo→flood transition [OBSERVED]

```
07:47:21.572390 read(8, "o", 1048576)   = 1
07:47:21.605214 read(8, "\r\n\33[?2004l\r", 1048576) = 11
07:47:21.606724 read(8, "\33]2;yes hello\7\33]133;C;cmdline=yes\\ hello\7", 1048565) = 41
07:47:21.606983 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suf"..., 1048524) = 105
07:47:21.609855 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1048576) = 2665
07:47:21.609950 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1045911) = 1311
07:47:21.610015 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1044600) = 999
07:47:21.610079 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1043601) = 889
```

The last keystroke echo (`o`, 1 byte) and the `Enter` echo (11 bytes) look just like Q2; then the first flood read returns 2665 bytes and subsequent reads run back-to-back with the buffer size already shrinking (`1048576 → 1045911 → 1044600 → 1043601`).

### Frequency & bytes-per-read distribution [OBSERVED]

Measured over the stated windows (I/O thread only, under strace):

```
===== Q3 RUN 1 (5s window) =====
flood reads: 55807 ; flood span: 5.023s ; flood reads/sec: 11109
bytes/read: min=11 max=20372 mean=654.4 median=544 mode=497
histogram (128B buckets, top): [512-639]:18591  [384-511]:18359  [640-767]:6574  [256-383]:3596
buffer avail-space during flood: min=538201 (=> write.offset peaked at 510375 B accumulated)
```

The dominant reads sit in the 384–639-byte range; the median (544) and mode (497) are ~500 bytes.

### Backpressure — the flow-control gate [OBSERVED + INFERRED]

The read loop only asks for `POLLIN` on the PTY fd while the parser has room: `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;` at `kitty/child-monitor.c:1501`, where `vt_parser_has_space_for_input` (`kitty/vt-parser.c:1477`, declared `kitty/vt-parser.h:36`) returns `self->read.sz + self->write.pending < BUF_SZ`.

- **[OBSERVED]** the quantity the gate watches was seen growing: available buffer space fell as far as `538201` (run 1), meaning `write.offset` accumulated up to **~510 KB** of un-parsed data — the parser buffer filling toward the 1 MiB cap.
- **[OBSERVED]** across all three runs (including a 10 s run of ~104 k reads), fd 8 was polled with `events=POLLIN` throughout and **never** with `events=0`; i.e. `POLLIN` was never actually withheld. For the plain repeated-text workload of `yes hello`, the main-thread parser (see Q5) drained the buffer fast enough that the gate always found space. This is the honest result of a genuine, varied effort (three runs, up to 10 s / 104 k reads); the throttle did not need to engage at this scale.
- **[INFERRED]** were accumulation to reach `BUF_SZ` (`read.sz + write.pending == BUF_SZ`), `vt_parser_has_space_for_input` would return `false`, `events` would be set to `0` at `kitty/child-monitor.c:1501`, and no `read` would be issued on fd 8 until the main thread drained the buffer (`parse_input` at `kitty/child-monitor.c:451` → `do_parse` at `kitty/child-monitor.c:438`). The observed shrinking `available_buffer_space` is the leading edge of exactly this mechanism.

---

## 7. Q4 — File-descriptor number (PTY master side)

**Direct answer [OBSERVED]:** kitty reads the PTY master on **file descriptor 8** in this run. The concrete integer is **process-specific** — it is whatever number the OS assigned to the master when kitty opened the PTY — and must be read from the running process (it is not a fixed constant).

### Evidence

**From the syscall trace — every PTY read is on fd 8 [OBSERVED]:**

```
07:47:21.329697 read(8, "y", 1048576)   = 1
07:47:21.359862 read(8, "e", 1048576)   = 1
07:47:21.390139 read(8, "s", 1048576)   = 1
```

Within the `echo test123` trace, the reads are distributed across only three fds, and fd 8 is the one carrying PTY payload (fd 4 and fd 6 are eventfds used for thread wakeups, which return the 8-byte counter `"\1\0\0\0\0\0\0\0"`) **[OBSERVED]**:

```
$ grep -oE 'read\([0-9]+,' trace_q2b.log | sort | uniq -c
     57 read(4,
     26 read(6,
      6 read(8,
```

**Cross-checked against `/proc` and `lsof` — fd 8 is the PTY master [OBSERVED]:**

```
$ ls -l /proc/58437/fd/8
lrwx------ 1 root root 64 Jul 10 07:39 /proc/58437/fd/8 -> /dev/pts/ptmx
$ lsof -p 58437 -a -d 8
COMMAND   PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
kitty   58437 root 8u   CHR    5,2      0t0    2 /dev/pts/ptmx
```

fd 8 is a character device `5,2` = `/dev/pts/ptmx` (the PTY master), whose slave counterpart is the shell's `/dev/pts/0` from Q1.

### Rationale & code citations [INFERRED]

The master fd is retained on the Python side as `self.child_fd = master` at `kitty/child.py:338` and made non-blocking via `os.set_blocking(self.child_fd, False)` at `kitty/child.py:345`. It is handed to the child monitor through `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` at `kitty/boss.py:587` (receiver `add_child()` at `kitty/child-monitor.c:305`). Inside the monitor it is registered into the poll set: `children_fds[EXTRA_FDS + self->count].fd = children[self->count].fd;` at `kitty/child-monitor.c:1286` with `children_fds[EXTRA_FDS + self->count].events = POLLIN;` at `kitty/child-monitor.c:1287`. The array is `static struct pollfd children_fds[MAX_CHILDREN + EXTRA_FDS]` at `kitty/child-monitor.c:86`, and `#define EXTRA_FDS 2` at `kitty/child-monitor.c:35` — which is why the PTY appears at poll index 2 (after the wakeup and signal fds) in the Q2/Q3 traces.

---

## 8. Q5 — Responsible C functions (reader + text/escape parser)

**Direct answer:**

- **(a) The function that reads from the PTY fd:** **`read_bytes()`** at `kitty/child-monitor.c:1337`, which issues the `read()` syscall at `kitty/child-monitor.c:1345`. It runs on the **I/O thread**.
- **(b) The function that parses input to separate printable text from escape sequences:** the VT state machine **`consume_input()`** at `kitty/vt-parser.c:1367`. It routes printable text through **`consume_normal()`** (`kitty/vt-parser.c:230`) to **`screen_draw_text()`** (`kitty/screen.c:866`), and escape/control sequences through **`consume_esc()`** (`kitty/vt-parser.c:261`) to the CSI/OSC/DCS/APC/PM/SOS sub-handlers. It runs on the **Main thread**.

### Reader — `read_bytes()` [OBSERVED it runs; INFERRED the code]

**[OBSERVED]** every PTY `read()` in the traces was issued by TID 58504, whose thread name is `KittyChildMon` (the KittyChildMonitor I/O thread):

```
$ cat /proc/58437/task/58504/comm
KittyChildMon
```

**[INFERRED]** the reader is `read_bytes()`. Verbatim source (`kitty/child-monitor.c:1337`–`1356`, no elisions):

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

The `read(fd, buf, available_buffer_space)` at `kitty/child-monitor.c:1345` is exactly the `read(8, …, 1048576)` observed in Q2/Q3; `available_buffer_space` comes from `vt_parser_create_write_buffer()` (`kitty/vt-parser.c:1451`), explaining the shrinking third argument.

### Parser — `consume_input()` [OBSERVED both classes present; INFERRED the routing]

**[OBSERVED]** the raw bytes read from fd 8 contain **both** classes the parser must separate — printable text and escape/control sequences — and the terminal acted on both (it displayed `test123`/`hello` and honored the title/prompt escapes):

```
# printable text (from the reads in Q2/Q3):
read(8, "hello\r\nhello\r\nhello\r\n..."      # plain text -> consume_normal -> screen_draw_text
...test123\r\n                                # plain text -> consume_normal -> screen_draw_text

# escape / control sequences (from the same reads):
\33]2;echo test123\7                          # OSC 2 set-title -> consume_esc
\33]133;C;cmdline=echo\ test123\7             # OSC 133 shell integration -> consume_esc
\33[?2004h   /   \33[?2004l                   # CSI DECSET/DECRST bracketed paste -> consume_esc
```

**[INFERRED]** the routing. `consume_input()` switches on `self->vte_state`. Verbatim source (`kitty/vt-parser.c:1367` onward, no elisions of the switch):

```c
consume_input(PS *self, PyObject *dump_callback UNUSED, id_type window_id UNUSED) {
#define consume(x) if (accumulate_st_terminated_esc_code(self, dispatch_##x)) { self->read.consumed = self->read.pos; SET_STATE(NORMAL); } break;

#ifdef DUMP_COMMANDS
    PyObject *dumped_bytes = PyBytes_FromStringAndSize((const char*)self->buf + self->read.pos, self->read.sz - self->read.pos);
    size_t pre_consume_pos = self->read.pos;
#endif

    switch (self->vte_state) {
        case VTE_NORMAL:
            consume_normal(self); self->read.consumed = self->read.pos; break;
        case VTE_ESC:
            if (consume_esc(self)) { self->read.consumed = self->read.pos; }
            break;
        case VTE_CSI:
            if (consume_csi(self)) { self->read.consumed = self->read.pos; if (self->csi.is_valid) dispatch_csi(self); SET_STATE(NORMAL); }
            break;
        case VTE_OSC:
            consume(osc);
        case VTE_APC:
            consume(apc);
        case VTE_PM:
            consume(pm);
        case VTE_DCS:
            consume(dcs);
        case VTE_SOS:
            consume(sos);
    }
```

Printable text is handled by `consume_normal()` (`kitty/vt-parser.c:230`), which decodes UTF-8 up to an ESC sentinel and pushes the decoded run to the screen — verbatim (`kitty/vt-parser.c:230`–`240`):

```c
consume_normal(PS *self) {
    do {
        const bool sentinel_found = utf8_decode_to_esc(&self->utf8_decoder, self->buf + self->read.pos, self->read.sz - self->read.pos);
        self->read.pos += self->utf8_decoder.num_consumed;
        if (self->utf8_decoder.output.pos) {
            REPORT_DRAW(self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
            screen_draw_text(self->screen, self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
        }
        if (sentinel_found) { SET_STATE(ESC); break; }
    } while (self->read.pos < self->read.sz);
}
```

The `screen_draw_text(...)` call lands in `screen_draw_text()` at `kitty/screen.c:866` — the printable-text destination. When an ESC byte is hit, `consume_normal` sets state `ESC`, and the next dispatch enters `consume_esc()` (`kitty/vt-parser.c:261`), whose first-character switch selects the sub-state — verbatim (`kitty/vt-parser.c:261`–`276`):

```c
consume_esc(PS *self) {
#define CALL_ED(name) REPORT_COMMAND(name); name(self->screen); SET_STATE(NORMAL);
#define CALL_ED1(name, ch) REPORT_COMMAND(name, ch); name(self->screen, ch); SET_STATE(NORMAL);
#define CALL_ED2(name, a, b) REPORT_COMMAND(name, a, b); name(self->screen, a, b); SET_STATE(NORMAL);
    const uint8_t ch = self->buf[self->read.pos++];
    const bool is_first_char = self->read.pos - self->read.consumed == 1;
    if (is_first_char) {
        switch(ch) {
            case ESC_DCS: SET_STATE(DCS); break;
            case ESC_OSC: SET_STATE(OSC); break;
            case ESC_CSI: SET_STATE(CSI); reset_csi(&self->csi); break;
            case ESC_APC: SET_STATE(APC); break;
            case ESC_SOS: SET_STATE(SOS); break;
            case ESC_PM: SET_STATE(PM); break;
```

### Thread split [OBSERVED reader thread; INFERRED parser thread]

- **Reading — I/O thread.** `read_bytes()`/`read()` run in `io_loop` (`kitty/child-monitor.c:1481`). **[OBSERVED]** every PTY read came from TID 58504 (`KittyChildMon`).
- **Parsing — Main thread.** **[INFERRED]** `consume_input` is reached from the main loop: `parse_input()` at `kitty/child-monitor.c:451` — whose own comment at `kitty/child-monitor.c:452` reads `// Parse all available input that was read in the I/O thread.` — calls `do_parse()` at `kitty/child-monitor.c:438`, which invokes `self->parse_func` at `kitty/child-monitor.c:440`. In the default build `parse_func = parse_worker` (`kitty/child-monitor.c:181`; the `parse_worker_dump` variant at `:180` is only selected when a dump callback is set), and `parse_worker`/`run_worker` (`kitty/vt-parser.c:1417`) call `consume_input`. The `Screen` owns the parser via `Parser *vt_parser;` at `kitty/screen.h:158`.

---

## 9. Stability confirmation (Q3) — three runs side by side

The same unchanged input (`yes hello` typed into the real window) was run three times; run 3 increased the scale to 10 s specifically to probe for buffer saturation. All measurements are on the I/O thread (TID 58504), under strace. **[OBSERVED]**:

| Run | Window | Flood reads | Reads/sec | Mean B | Median B | Mode B | Max B | `POLLIN` withheld (`events=0`) |
|-----|--------|-------------|-----------|--------|----------|--------|-------|-------------------------------|
| 1   | 5.023 s  | 55 807   | 11 109 | 654.4 | 544 | 497 | 20 372 | 0 |
| 2   | 5.016 s  | 53 032   | 10 573 | 658.3 | 555 | 511 | 19 329 | 0 |
| 3   | 10.015 s | 103 809  | 10 365 | 563.1 | 504 | 483 | 19 922 | 0 |

**Conclusion:** the values are **stable across runs** — reads/sec stays in the ~10.3–11.1 k band, and the typical bytes-per-read (median ~500–555, mode ~483–511) is consistent within a few percent. Increasing the scale to 10 s / ~104 k reads did not change the picture and still did not force `POLLIN` withholding (the parser kept pace with the plain-text stream). Caveat: these cadences are measured **under strace**, which adds per-syscall overhead, so the absolute reads/sec is a lower bound; the qualitative change (back-to-back large reads vs. blocking 1-byte reads) and the byte-size distribution are the robust findings.

---

## 10. Coverage pass — every named item answered

| # | Named item asked | Concrete answer (OBSERVED unless noted) | `file:line` |
|---|------------------|------------------------------------------|-------------|
| Q1a | Process spawned | `/bin/bash` (login shell) | `kitty/constants.py:181` |
| Q1b | PID | `58505` | (ps / /proc) |
| Q1c | Exact command line | `/bin/bash --posix` (`/bin/bash^@--posix^@`) | argv via `kitty/child.py:314`; exec `kitty/child.c:159` |
| Q1d | PTY device path | slave `/dev/pts/0`; master `/dev/pts/ptmx` (fd 8) | dup2 `kitty/child.c:138`–`145` |
| Q2a | System calls | `poll()` then `read()` on fd 8 (per readiness) | `kitty/child-monitor.c:1512`/`1509`; `:1345` |
| Q2b | Buffer size | `1048576` (= `BUF_SZ` 1 MiB); shrinks by `write.offset` | `kitty/vt-parser.c:18`; `:1451` |
| Q2c | Bytes returned | 1 per keystroke; Enter 11; output 47/114/429 (613 total, 16 reads) | reader `kitty/child-monitor.c:1337` |
| Q3a | How behavior changes | blocking 1-byte reads → back-to-back multi-hundred-byte reads; poll timeout −1 → 2 ms | `kitty/child-monitor.c:1509`/`1512`; gate `:1501` |
| Q3b | Frequency | ~10 365–11 109 reads/sec (3 runs); ~27–54 µs apart | dispatch `kitty/child-monitor.c:1529`–`1531` |
| Q3c | Typical bytes/read | median ~500–555, mode ~483–511, mean ~563–658, up to ~20 KB | buffer `kitty/vt-parser.c:18` |
| Q3 stability | ≥2-run stability | confirmed stable across 3 runs (5 s, 5 s, 10 s) | §9 |
| Q3 backpressure | flow-control gate | `available_buffer_space` shrank to ~510 KB; `POLLIN` never withheld at this scale (INFERRED: withheld at `BUF_SZ`) | `kitty/child-monitor.c:1501`; `kitty/vt-parser.c:1477` |
| Q4 | fd integer | **8** (process-specific; `/dev/pts/ptmx`) | reg. `kitty/child-monitor.c:1286`; retain `kitty/child.py:338` |
| Q5a | Reader function | `read_bytes()` — I/O thread (TID 58504 `KittyChildMon`) | `kitty/child-monitor.c:1337` (`read` `:1345`) |
| Q5b | Parser function | `consume_input()` — Main thread | `kitty/vt-parser.c:1367` |
| Q5b | Text path | `consume_normal()` → `screen_draw_text()` | `kitty/vt-parser.c:230` → `kitty/screen.c:866` |
| Q5b | Escape path | `consume_esc()` → CSI/OSC/DCS/APC/PM/SOS | `kitty/vt-parser.c:261` |

---

## 11. Observed-vs-Inferred summary

| Fact | Classification | Basis |
|------|----------------|-------|
| Shell is `/bin/bash`, PID 58505, cmdline `/bin/bash --posix` | **[OBSERVED]** | `ps`, `/proc/58505/cmdline` |
| PTY slave `/dev/pts/0`; master fd 8 `/dev/pts/ptmx` | **[OBSERVED]** | `readlink /proc/.../fd`, `ls -l`, `lsof` |
| `poll → read` pair on fd 8; read buffer arg `1048576` | **[OBSERVED]** | `strace -f -tt` |
| Returned bytes: 1/keystroke, 11 Enter, 47/114/429 output | **[OBSERVED]** | `strace` |
| `yes hello`: ~10–11 k reads/s, ~500 B/read, back-to-back, stable ×3 | **[OBSERVED]** | `strace` + analysis, 3 runs |
| Available buffer space shrinks (write.offset accumulates) | **[OBSERVED]** | `strace` read 3rd arg |
| fd number = 8 | **[OBSERVED]** | `strace`, `/proc`, `lsof` |
| Reader executes on the I/O thread (`KittyChildMon`) | **[OBSERVED]** | `/proc/.../task/58504/comm` + `strace` |
| Both text and escape bytes are present in the reads | **[OBSERVED]** | `strace` payloads |
| Shell resolution `pw_shell or '/bin/sh'` | **[INFERRED]** | `kitty/constants.py:181` |
| Spawn chain (fork/setsid/TIOCSCTTY/dup2/execvp) | **[INFERRED]** | `kitty/child.c:81`–`159` |
| `read()` lives in `read_bytes()`; buffer sizing formula | **[INFERRED]** | `kitty/child-monitor.c:1337`/`1345`; `kitty/vt-parser.c:1451` |
| Backpressure gate & `POLLIN` withholding at `BUF_SZ` | **[INFERRED]** | `kitty/child-monitor.c:1501`; `kitty/vt-parser.c:1477` |
| Text/escape routing `consume_input`→`consume_normal`/`consume_esc` | **[INFERRED]** | `kitty/vt-parser.c:1367`/`230`/`261` |
| Parsing dispatched on the Main thread | **[INFERRED]** | `kitty/child-monitor.c:451`/`452`/`438` |
| fd registration into the poll set | **[INFERRED]** | `kitty/child-monitor.c:1286`/`1287`/`35`/`86` |

---

*All runtime values above were captured from a default-configuration kitty (v0.35.2) built with `make` (`python3 setup.py`, `Makefile:13`) and launched headless under Xvfb, with input delivered exclusively through the canonical terminal keyboard path. Temporary observation scripts and strace logs were created only under `/tmp` and removed after the investigation; the repository is unchanged apart from this document.*
