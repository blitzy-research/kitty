# kitty PTY-read investigation — how the terminal reads from the shell over a pseudoterminal

**Repository:** kitty terminal emulator · **Commit:** `815df1e21` ("Wire up applying of font config") · **Branch:** `kitty_815df1e210e0`
**Task type:** Evidence-backed Q&A investigation (read-only; the sole artifact added to the repo is this document).
**Methodology:** `build → launch → instrument → observe → document`. Every runtime value below was captured **live** from a real, default-configuration kitty process using `strace`, `ps`, `/proc`, `readlink`, and `lsof`. Statements are labelled **[OBSERVED]** (captured from a live run) or **[INFERRED]** (derived from reading verified source). Every value carries a `file:line` citation.

**Redaction & reproducibility policy.** Runtime identifiers (PIDs, TIDs, the fd number, `/dev/pts/N`, the X window id) are **process-specific** to this session and are reported exactly as observed. Two classes of text are redacted to avoid leaking environment metadata, always with an explicit label that **preserves the observed byte length** so the reported `read()` counts remain verifiable: (1) the container **hostname** (shown as `[hostname redacted]`), and (2) the disposable-build **cwd path** that the shell places in its OSC-2 title (shown as `<cwd:NN B redacted>`). Nothing else is edited; strace output is otherwise quoted verbatim.

---

## 1. Overview

This document answers five sub-questions about how kitty's native code communicates with a spawned shell over a pseudoterminal (PTY):

1. **Q1 — Shell spawn identity:** what process is spawned, its PID, its exact command line, and the PTY device path.
2. **Q2 — Reading a typed command:** which syscalls kitty makes to read `echo test123`, the buffer size, and how many bytes come back.
3. **Q3 — High-volume output:** how kitty's reading behavior changes under `yes hello`, the read frequency, and the typical bytes-per-read.
4. **Q4 — File descriptor number:** the fd number kitty uses to read the PTY master side.
5. **Q5 — Responsible C functions:** the function that reads from the fd, and the function that parses input to separate printable text from escape sequences.

**One-line data-flow summary** (each hop cited in the answers below; identifiers are from this session's live run):

```
Boss.add_child (kitty/boss.py:585/587)
   -> Child.fork (kitty/child.py:276) -> openpty (kitty/child.py:281) -> fast_data_types.spawn (kitty/child.py:333)
      -> native spawn() (kitty/child.c:81): fork -> setsid -> ioctl TIOCSCTTY -> dup2(slave) -> execvp  => /bin/bash --posix (PID 100426)
   -> master fd retained self.child_fd = 8 (kitty/child.py:338), set non-blocking (kitty/child.py:345)
      -> registered into poll set children_fds[EXTRA_FDS + i].fd (kitty/child-monitor.c:1286)

I/O thread  (io_loop, kitty/child-monitor.c:1481; TID 100425 "KittyChildMon"): poll (kitty/child-monitor.c:1509/1512)
   -> read_bytes (kitty/child-monitor.c:1337) -> read() (kitty/child-monitor.c:1345) into a BUF_SZ=1 MiB buffer (kitty/vt-parser.c:18)
Main thread : parse_input (kitty/child-monitor.c:451) -> do_parse (kitty/child-monitor.c:438)
   -> consume_input (kitty/vt-parser.c:1367)
       -> consume_normal (kitty/vt-parser.c:230) -> screen_draw_text (kitty/screen.c:866)   [printable text]
       -> consume_esc    (kitty/vt-parser.c:261) -> CSI/OSC/DCS/APC/PM/SOS handlers   [escape sequences]
```

---

## 2. Environment & Build

### 2.1 Canonical container & toolchain

The build, launch, and syscall tracing were performed inside the user-provided container. Its image is identified two ways in the task setup (the same image, two registries): `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` and `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`. The image **tag** cannot be read from inside a running container, so it is reported here from the setup metadata (not observed); what **is** observable from inside is the cgroup/container id. **[OBSERVED]** container id and OS:

```
$ cat /proc/1/cgroup | head -1
0::/../cri-containerd-991458ab8b1dd5b46f5681cebe76fd44a435ea75eb91bfa4c2274a930f24c8d8.scope
$ grep PRETTY_NAME /etc/os-release
PRETTY_NAME="Ubuntu 25.10"
$ uname -s -r -m        # hostname deliberately omitted (see redaction policy)
Linux 6.6.122+ x86_64
```

**[OBSERVED]** tool versions (the launcher below is the disposable build from §2.2):

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
$ lsof -v 2>&1 | grep revision
    revision: 4.99.4
```

Build dependencies are those listed in `docs/build.rst` (`python >= 3.8` at `docs/build.rst:83`; harfbuzz, freetype, fontconfig, zlib, libpng, lcms2, xxhash, openssl); the harfbuzz version guard is `at_least_version('harfbuzz', 1, 5)` at `setup.py:609`. **[INFERRED]** (from source).

### 2.2 Default-configuration build (disposable copy, no repository mutation)

To build without touching the repository at all (not even its git-ignored artifacts), the checkout was copied into a private scratch tree and built there. The copy deliberately **excludes** any pre-built native extension/launcher, so `make` performs a genuine from-scratch link. The `time` wrapper is shown explicitly so the reported figures are unambiguous. The canonical build command is `make`, whose `all:` target (`Makefile:12`) runs `python3 setup.py $(VVAL)` (`Makefile:13`); `VVAL` is empty in the default (non-verbose) build. **[OBSERVED]**:

```
$ SCRATCH="$(mktemp -d /tmp/kitty_qa.XXXXXXXX)"          # private scratch, mode 0700
$ mkdir -p "$SCRATCH/kitty_pristine"
$ tar --exclude=./.git --exclude='*.so' \
      --exclude=./kitty/launcher/kitty --exclude=./kitty/launcher/kitten \
      -cf - . | ( cd "$SCRATCH/kitty_pristine" && tar -xf - )   # disposable copy; no prebuilt artifacts
$ cd "$SCRATCH/kitty_pristine"
$ export PATH="$PATH:/usr/local/go/bin" GOTOOLCHAIN=local
$ { time make ; }        # 'make' -> 'python3 setup.py $(VVAL)' (Makefile:13); a from-scratch build (Go cache cleared)
python3 setup.py
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/2] Linking kitty/fast_data_types ...
[2/2] Linking launcher ...
 done
... (Go package compilation for the bundled kitten tools) ...
real	0m58.454s
user	2m2.582s
sys	0m17.310s
```

The `wayland-protocols` lines are a benign configuration notice — the X11 backend (used here via Xvfb) is built regardless. The 58 s figure is a cold build (Go build cache cleared to force full compilation); a warm rebuild of just the native extension + launcher took `real 0m18.071s`. No `--debug` flag was used; all answers come from the **default** build. The produced artifacts **[OBSERVED]**:

```
$ ls -l kitty/fast_data_types*.so
-rwxr-xr-x 1 root root 1253792 ... kitty/fast_data_types.so
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root ... kitty/launcher/kitty
```

Because this is a throwaway copy under `/tmp`, the repository working tree is provably unchanged (see the footer for the closing `git status --porcelain`).

### 2.3 Default configuration & headless launch

**Default configuration is proven, not assumed.** kitty resolves its config dir in `_get_config_dir()` (`kitty/constants.py:87`–`131`): it honours `KITTY_CONFIG_DIRECTORY` (`:88`–`89`), then `XDG_CONFIG_HOME` (`:92`–`93`), then `XDG_CONFIG_DIRS` (`:97`), else falls back to `~/.config` (`:116`). All of those are unset and no user `kitty.conf` exists, so kitty runs on **built-in defaults**. **[OBSERVED]**:

```
$ for v in KITTY_CONFIG_DIRECTORY XDG_CONFIG_HOME XDG_CONFIG_DIRS KITTY_CONFIG; do
      printf '%s=[%s]\n' "$v" "${!v-<unset>}"; done
KITTY_CONFIG_DIRECTORY=[<unset>]
XDG_CONFIG_HOME=[<unset>]
XDG_CONFIG_DIRS=[<unset>]
KITTY_CONFIG=[<unset>]
$ ls -A ~/.config/kitty/ 2>/dev/null | wc -l      # 0 => directory empty
0
$ test -e /etc/xdg/kitty/kitty.conf && echo EXISTS || echo ABSENT
ABSENT
```

There is no physical display, so kitty was launched under an X virtual framebuffer. This is a *launch mechanism*, not a configuration change (no `--config` was passed). The Xvfb is **access-controlled** (a private MIT-MAGIC-COOKIE-1 auth file; `-nolisten tcp`; **no** `-ac`), its PID is captured for a scoped teardown, and kitty is started with the default config. **[OBSERVED]**:

```
$ export XAUTHORITY="$SCRATCH/Xauthority"
$ xauth -f "$XAUTHORITY" add :99 . "$(mcookie)"        # private cookie, mode 0600
$ setsid Xvfb :99 -screen 0 1280x800x24 -nolisten tcp -auth "$XAUTHORITY" \
      > "$SCRATCH/xvfb.log" 2>&1 &
$ XVFB_PID=$!            # => 100092  (killed in teardown; see footer)
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ setsid ./kitty/launcher/kitty > "$SCRATCH/kitty.log" 2>&1 &      # default config, no --config
```

kitty started as **PID 100358** and a real shell spawned as its child (formalized in Q1). The entire startup log was one benign message:

```
$ cat "$SCRATCH/kitty.log"
[0.157] Failed to open systemd user bus with error: Connection refused
```

---

## 3. Methodology

- **Canonical input path only.** Every command was typed into the real kitty window via `xdotool` — genuine X key events delivered to the window (`id 2097164`). kitty's X11 keyboard handler receives each event, writes the byte to the PTY master, and reads back the echo/output. This is **not** remote control, a debug hook, a mock, or synthetic byte injection. kitty's remote control is off by default (`allow_remote_control 'no'` at `kitty/options/definition.py:2969`), and invoking it here fails, so the answer values can only come from the canonical keyboard path. **[OBSERVED]**:

```
$ ./kitty/launcher/kitty @ ls ; echo exit=$?
Error: open /dev/tty: no such device or address
exit=1
```

- **Syscall capture.** `strace -tt -s <N> -e trace=poll,read[,write]` attached to a specific thread with `-p <TID>` and logged to a private file with `-o "$SCRATCH/…"`. For Q2, `-s 512` was used so the short output reads (≤163 B) are captured **in full** (no `"...` truncation). For Q3's flood, `-s 64` was used (the payload is the repeating `hello\r\n`), so flood reads show strace's `"...` truncation marker — this is disclosed, not hidden. Reading was traced on the **single I/O thread** so its `poll`/`read` lines appear unsplit (see the note below and Q5).
- **ptrace permissions (disclosed).** `kernel.yama.ptrace_scope = 1` in this container and was **not** changed. `strace` attaches only because the tracer and the target share the same owner (both uid 0); no global weakening (e.g. setting `ptrace_scope=0`) was performed:

```
$ cat /proc/sys/kernel/yama/ptrace_scope
1
```

- **Process / fd / PTY facts.** `ps -o …`, `cat -A /proc/<pid>/cmdline`, `readlink /proc/<pid>/fd/N`, `ls -l /proc/<pid>/fd/N`, and `lsof -p <pid> -a -d N`.
- **Scratch & cleanup.** All scripts and logs live under a single `mktemp -d` scratch dir (mode 0700) outside the repository and are removed afterward; the disposable build copy is removed too (see footer).
- **Reproducibility & scale.** The high-volume magnitudes (Q3) were confirmed across **three** unchanged 5 s runs plus a 10 s scale-up run; per-metric spread and the stability criterion are given in §9.

> **A note on strace's line-splitting.** Under `-f` (follow all threads) strace interleaves threads and splits an interrupted syscall across two lines, e.g. `read(8 <unfinished ...>` … `<... read resumed>, "e", 1048576) = 1`. Both halves are the same `read()` on fd 8. To avoid miscounting, the primary traces below attach to only the I/O thread (no `-f`), which yields unsplit lines; where the `-f` form is shown, the reconciliation is made explicit (Q2, "Reconciling the read count").

---

## 4. Q1 — Shell spawn identity

**Direct answer (all four items [OBSERVED]):**

| Item | Value |
|------|-------|
| **(a) Process spawned** | `/bin/bash` — the account-configured shell, started in POSIX mode |
| **(b) PID** | **100426** |
| **(c) Exact command line** | `/bin/bash --posix` |
| **(d) PTY device path** | slave **`/dev/pts/0`** (kitty's master counterpart is fd 8 → `/dev/pts/ptmx`) |

### Evidence

**Process tree — kitty → shell [OBSERVED]** (producing command shown):

```
$ ps -o pid,ppid,stat,tty,args -p 100358        # kitty
    PID    PPID STAT TT       COMMAND
 100358       1 Ssl  ?        ./kitty/launcher/kitty
$ ps -o pid,ppid,stat,tty,args --ppid 100358     # its children
    PID    PPID STAT TT       COMMAND
 100426  100358 Ss+  pts/0    /bin/bash --posix
```

The shell (100426) is a direct child of kitty (100358), attached to `pts/0`.

**Exact command line [OBSERVED]:**

```
$ cat -A /proc/100426/cmdline
/bin/bash^@--posix^@
```

`cat -A` shows the NUL-delimited argv exactly: `argv[0]=/bin/bash`, `argv[1]=--posix` (`^@` is the NUL separator; there is no trailing text). `STAT=Ss+` means the process is a **session leader** (`s`) with a controlling terminal, in the **foreground** process group (`+`). It is **not** a login shell: `argv[0]` is `/bin/bash`, not the `-`-prefixed `-bash` form, and there is no `--login`. The precise characterization is therefore *the account-configured interactive shell running in POSIX mode, as session leader of the foreground process group with `/dev/pts/0` as its controlling terminal*.

**PTY device path [OBSERVED]:**

```
$ for fd in 0 1 2; do readlink /proc/100426/fd/$fd; done   # shell stdin/stdout/stderr
/dev/pts/0
/dev/pts/0
/dev/pts/0
```

All three standard streams of the shell point at the PTY **slave** `/dev/pts/0`. The **master** counterpart is held by kitty on fd 8 **[OBSERVED]**:

```
$ ls -l /proc/100358/fd/8
lrwx------ 1 root root 64 ... /proc/100358/fd/8 -> /dev/pts/ptmx
$ lsof -p 100358 -a -d 8
COMMAND    PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
kitty   100358 root 8u   CHR    5,2      0t0    2 /dev/pts/ptmx
```

So the PTY pair is: **kitty master fd 8 (`/dev/pts/ptmx`, char device 5,2) ↔ shell slave `/dev/pts/0`**.

### Rationale & code citations

- **Which shell [INFERRED]:** the shell path is resolved as `shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` at `kitty/constants.py:181`. In this container the effective user's `pw_shell` is `/bin/bash`, so the resolved shell is `/bin/bash` and the `/bin/sh` fallback at `kitty/constants.py:185` is **not** taken.
- **Why `--posix` and no leading `-` [OBSERVED + INFERRED]:** by default kitty launches the shell through its `run-shell` kitten to set up shell integration — `argv = [kitten_exe(), 'run-shell', '--shell', shlex.join(argv), '--shell-integration', ksi]` at `kitty/child.py:314`. The `run-shell` kitten re-executes the real shell in POSIX mode with integration injected via environment/rcfile rather than the historical `-`-prefixed login form; the resulting live process image observed is exactly `/bin/bash --posix`. The command line is reported **exactly as it appears**.
- **Spawn chain [INFERRED]:** `Boss.add_child` (`kitty/boss.py:585`) calls `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` (`kitty/boss.py:587`); `Child.fork()` (`kitty/child.py:276`) creates the PTY via `openpty()` (defined `kitty/child.py:170`, invoked `kitty/child.py:281`) and calls `fast_data_types.spawn(...)` (`kitty/child.py:333`, statement spans L333–L335). The native `spawn()` (`kitty/child.c:81`) performs `fork()` (`kitty/child.c:97`), `setsid()` (`kitty/child.c:123`), `ioctl(tfd, TIOCSCTTY, 0)` (`kitty/child.c:129`), `safe_dup2(slave, STDOUT_FILENO)` (`kitty/child.c:138`), `safe_dup2(slave, STDERR_FILENO)` (`kitty/child.c:139`), the STDIN dup2 (`kitty/child.c:141`/`145`), and `execvp(exe, argv)` (`kitty/child.c:159`). The `dup2` of the slave onto fds 0/1/2 is exactly why the shell's fd 0/1/2 all read back as `/dev/pts/0` above.

---

## 5. Q2 — Reading the typed command `echo test123`

**Direct answer:**

- **(a) System calls [OBSERVED]:** per keystroke the I/O thread does `poll()` (fd 8 reports `POLLOUT`) → `write(8, …)` (kitty writes the key to the PTY master) → `poll()` (fd 8 reports `POLLIN`, the line discipline echoed it) → `read(8, …)`. The `read()` is the syscall that pulls the byte(s) off the PTY master; it is always paired with a preceding `poll()`.
- **(b) Buffer size [OBSERVED]:** the third argument to `read()` is **1048576** bytes = 1 MiB = `BUF_SZ` (`kitty/vt-parser.c:18`); it shrinks below 1 MiB when unparsed bytes are still buffered.
- **(c) Bytes returned [OBSERVED]:** each typed keystroke echoes back as a **1-byte** read; the `Enter` read returns **11** bytes; the command's title/marker/output/prompt arrive in reads of **47**, **114**, and **163** bytes. The whole interaction is **16 reads / 347 bytes** on fd 8. The count returned is bounded by what the kernel line discipline currently holds, **not** by the 1 MiB request.

The command was typed into the real kitty window (canonical path). strace was attached to the single I/O thread (TID **100425**), so lines are unsplit:

```
$ strace -tt -s 512 -e trace=poll,read,write -p 100425 -o "$SCRATCH/trace_q2_clean.log"
$ xdotool type   --window 2097164 --delay 120 "echo test123"
$ xdotool key    --window 2097164 Return
```

### Evidence — the `poll → write → poll → read` pipeline for one keystroke (`e`) [OBSERVED]

Contiguous, unedited excerpt (timestamps are HH:MM:SS.microseconds):

```
08:40:12.178407 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
08:40:12.178451 write(8, "e", 1)        = 1
08:40:12.178484 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
08:40:12.178588 read(8, "e", 1048576)   = 1
08:40:12.178618 write(4, "\1\0\0\0\0\0\0\0", 8) = 8
```

Reading these five lines: the first `poll` reports fd 8 **writable** (`POLLOUT`); `write(8, "e", 1)` puts the keystroke onto the master — this `write` is **directly observed** (the `write` syscall is included in the trace filter, so there is no need to infer it from `POLLOUT`). The second `poll` reports fd 8 **readable** (`POLLIN`) because the kernel line discipline echoed the byte back; `read(8, "e", 1048576) = 1` reads that one echoed byte; finally `write(4, …, 8)` bumps the main-thread wakeup eventfd (fd 4) so the parser runs.

The poll set `[{fd=6}, {fd=7}, {fd=8}]` with `nfds=3` maps exactly to the source: `children_fds` holds two reserved slots (`#define EXTRA_FDS 2` at `kitty/child-monitor.c:35`) — here fd 6 (I/O-thread wakeup eventfd) and fd 7 (signal fd) — followed by the child PTY fd at index `EXTRA_FDS + 0` (fd 8). `nfds = self->count + EXTRA_FDS = 1 + 2 = 3`, matching the blocking `poll(children_fds, self->count + EXTRA_FDS, -1)` at `kitty/child-monitor.c:1512` **[INFERRED]**. The `events` value on fd 8 is set by `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0` at `kitty/child-monitor.c:1501` (plus `POLLOUT` while kitty has a keystroke queued to write). The `read()` itself is issued at `kitty/child-monitor.c:1345` inside `read_bytes()` (`kitty/child-monitor.c:1337`).

### Evidence — all 12 keystroke echoes [OBSERVED]

Typing `echo test123` (12 characters) produced exactly 12 single-byte reads on fd 8, each requesting the full 1 MiB buffer:

```
08:40:12.178588 read(8, "e", 1048576)   = 1
08:40:12.238864 read(8, "c", 1048576)   = 1
08:40:12.299275 read(8, "h", 1048576)   = 1
08:40:12.359613 read(8, "o", 1048576)   = 1
08:40:12.420019 read(8, " ", 1048576)   = 1
08:40:12.480414 read(8, "t", 1048576)   = 1
08:40:12.540824 read(8, "e", 1048576)   = 1
08:40:12.601197 read(8, "s", 1048576)   = 1
08:40:12.661523 read(8, "t", 1048576)   = 1
08:40:12.721924 read(8, "1", 1048576)   = 1
08:40:12.782312 read(8, "2", 1048576)   = 1
08:40:12.842681 read(8, "3", 1048576)   = 1
```

Each read returns exactly `1` byte — the character just echoed by the line discipline — while requesting `1048576` bytes. The reads are ~60 ms apart, i.e. one per typed key (the `--delay 120` between key press/release yields ~60 ms per character).

### Evidence — `Enter` and the command output reads [OBSERVED]

After `Return`, the following contiguous excerpt shows the `Enter` read and the command output/prompt (`-s 512` keeps every read complete):

```
08:40:13.308979 read(8, "\r\n\33[?2004l\r", 1048576) = 11
08:40:13.310317 read(8, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47
08:40:13.310535 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114
08:40:13.311001 read(8, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]133;k;start_suffix_kitty\7\33[5 q\33]2;<cwd:37 B redacted>\7\33]133;k;end_suffix_kitty\7", 1048404) = 163
```

Reading these (the fourth read's OSC-2 title held the 37-byte disposable-build cwd, redacted here; its returned count is unchanged at **163**):

- `read(8, "\r\n\33[?2004l\r", 1048576) = 11` — the `Enter` **read of 11 bytes**, of which only **2 bytes are the terminal echo** of the pressed key (`\r\n`, i.e. CR LF); the remaining **9 bytes are shell/line-discipline control**, not echo: `\33[?2004l` (8 bytes, the `CSI ?2004l` "bracketed-paste-off" the shell emits before running the command) plus a final `\r` (1 byte). So: 11 bytes = 2 echo + 9 control.
- `read(8, "\33]2;echo test123\7…", 1048565) = 47` — the shell (via integration) sets the window title (OSC 2 `ESC]2;echo test123 BEL`) and reports the command line (OSC 133 `;C;cmdline=echo test123`).
- `read(8, "…test123\r\n", 1048518) = 114` — **the actual command output**: the literal `test123\r\n` is present at the end of this read, wrapped by OSC 133 shell-integration markers.
- `read(8, "\33[?2004h…", 1048404) = 163` — the next prompt: bracketed-paste-on (`CSI ?2004h`), OSC 133 prompt markers, the exit-status report `133;D;0` (previous command succeeded), cursor-shape `CSI 5 q`, and the OSC-2 title carrying the cwd. (Under `bash --posix` the `PS1` itself is minimal, which is why this prompt read is 163 B rather than the hundreds of bytes a decorated `PS1` would add.)

### Reconciling the read count — 16 reads on fd 8 [OBSERVED]

The clean single-thread trace above contains exactly **16** `read(8, …)` lines (12 keystroke echoes + the 11/47/114/163-byte reads). A trace taken with `-f` (all 67 threads) tells the *same* story but splits interrupted reads, so it must be reconciled before counting:

```
# same interaction, traced with -f (follow all threads):
$ grep -c 'read(8,'            trace_q2_f.log     # naive literal match
4
$ grep -c 'read(8 <unfinished' trace_q2_f.log     # reads split by the scheduler
12
# example of one split read (both halves are TID 100425, fd 8, one read):
100425 08:38:34.559138 read(8 <unfinished ...>
100425 08:38:34.559154 <... read resumed>, "e", 1048576) = 1
```

A naive `grep -c 'read(8,'` on the `-f` log returns **4** because 12 of the reads were interrupted and appear as `read(8 <unfinished ...>` / `<... read resumed>` pairs. Reconciled: 4 literal + 12 split = **16**, matching the clean single-thread trace exactly. The clean trace (no `-f`) is used as the primary evidence throughout.

### Buffer size ↔ `BUF_SZ`, and returned-count rationale

- **Buffer size [OBSERVED + INFERRED]:** the third `read()` argument is `1048576` when the parser buffer is empty, and `1048565`, `1048518`, `1048404` on the successive output reads. Those reductions equal the bytes still sitting unparsed in the parser buffer: `1048576 − 1048565 = 11`, `1048576 − 1048518 = 58` (=11+47), `1048576 − 1048404 = 172` (=11+47+114). This is exactly `*sz = BUF_SZ - self->write.offset` computed by `vt_parser_create_write_buffer()` at `kitty/vt-parser.c:1451`, where `BUF_SZ` is `#define BUF_SZ (1024u*1024u)` at `kitty/vt-parser.c:18` and the backing store is `uint8_t buf[BUF_SZ + BUF_EXTRA]` at `kitty/vt-parser.c:194`.
- **Bytes returned [OBSERVED value; INFERRED cause]:** the returned counts (1, 1, …, 11, 47, 114, 163) are tiny compared with the ~1 MiB request. A `read()` on a PTY master returns only what the kernel line discipline currently holds — the echoed keystroke, or the burst the shell just wrote — not the requested capacity. The whole `echo test123` interaction was **347 bytes** across **16 reads** on fd 8 (12×1 + 11 + 47 + 114 + 163).

---

## 6. Q3 — High-volume output `yes hello`

**Direct answer:**

- **(a) How reading behavior changes [OBSERVED]:** instead of one ~1-byte read every ~60 ms (the echo case), reads become **back-to-back** — each `read()` is immediately preceded by a `poll()` that reports fd 8 readable, and the next `read()` fires ~55 µs later — and each read returns a **large multi-hundred-byte chunk** of `hello\r\n` lines rather than a single byte. The `poll` timeout also changes from blocking (`-1`, when idle) to a short timed value (`1`–`2` ms) during the stream.
- **(b) Frequency [OBSERVED]:** roughly **10,200–10,900 reads/second** on the PTY fd (measured under strace across four runs). Two distinct intervals must not be conflated: the **read-to-read cadence** (consecutive reads) has median **55–57 µs**, while the **poll-to-read latency** (the `poll` immediately before a read → that read) has median **28 µs**. The ~28 µs is *not* the inter-read interval.
- **(c) Typical bytes-per-read [OBSERVED]:** the robust "typical" is **median 504–546 B** and **mode 469–511 B**; the mean is higher and less stable (557–668 B) because the distribution is heavy-tailed (individual reads range from single digits up to ~14–20 KB). Contrast Q2, where every keystroke read was exactly 1 byte.

`yes hello` was typed into the real kitty window (canonical path); strace was attached to the single I/O thread (TID 100425, `comm=KittyChildMon` — see Q5) so lines are unsplit; the stream was interrupted with a real Ctrl-C keystroke. The exact per-run commands (documented script):

```
$ strace -tt -s 64 -e trace=poll,read -p 100425 -o "$SCRATCH/trace_q3_run1.log" &
$ xdotool type --window 2097164 --delay 60 "yes hello"
$ xdotool key  --window 2097164 Return
$ sleep 5                                   # flood window (stated scale)
$ xdotool key  --window 2097164 ctrl+c      # real Ctrl-C keystroke -> SIGINT to the foreground pgrp
```

### Evidence — the I/O thread is the child monitor [OBSERVED]

```
$ cat /proc/100358/task/100425/comm
KittyChildMon
```

### Evidence — before → during → after

**Before/transition (idle → flood).** The last keystroke echoes, the `Enter` read (11 B), the title/marker reads, then the poll timeout switches from blocking `-1` to timed as the flood starts **[OBSERVED]**:

```
08:42:54.754767 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
08:42:54.754834 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
08:42:54.754894 read(8, "\r\n\33[?2004l\r", 1048576) = 11
08:42:54.756252 read(8, "\33]2;yes hello\7\33]133;C;cmdline=yes\\ hello\7", 1048565) = 41
08:42:54.756460 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_"..., 1048524) = 105
08:42:54.756490 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 0 (Timeout)
```

**During (back-to-back reads mid-stream).** Contiguous, unedited excerpt from run 1 (the `"...` marker is strace's `-s 64` truncation of the repeating `hello\r\n` payload — disclosed; note the microsecond spacing and the shrinking buffer size) **[OBSERVED]**:

```
08:42:57.057112 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1019251) = 383
08:42:57.057139 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
08:42:57.057167 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1018868) = 380
08:42:57.057198 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
08:42:57.057228 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1018488) = 602
08:42:57.057256 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
08:42:57.057284 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1017886) = 418
08:42:57.057311 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
08:42:57.057338 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1017468) = 490
```

Observations from this block:

- **Read-to-read cadence vs poll-to-read latency (distinct):** consecutive reads are ~55 µs apart (`.057112 → .057167 → .057228 → .057284` ≈ 55/61/56 µs), whereas the `poll` immediately before a read precedes it by only ~28 µs (`.057139 → .057167`). The ~28 µs figure is the poll-to-read latency, **not** the inter-read interval; the read-to-read cadence (~55 µs) is the real frequency signal. (Compare Q2 where reads were ~60 ms apart.)
- **Large chunks:** returns are 383, 380, 602, 418, 490 bytes of repeated `hello\r\n`.
- **Poll timeout changed:** the flood polls use timeout `1`–`2` ms — the `input_delay` timed path `if (time_delta >= 0) ret = poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta))` at `kitty/child-monitor.c:1509` — rather than the blocking `-1` (`kitty/child-monitor.c:1512`) seen when idle **[INFERRED tie to source]**.
- **Buffer filling:** the buffer-size argument decreases (`1019251 → 1018868 → 1018488 → 1017886 → 1017468`), i.e. `write.offset` grows because reads briefly outrun the main-thread parser. Dispatch to `read_bytes()` happens via `if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP))` at `kitty/child-monitor.c:1529`–`1531`.

**After (Ctrl-C → prompt recovery).** A real Ctrl-C keystroke sends SIGINT to the foreground process group; `yes` dies and the prompt returns. The final read carries the shell-integration exit-status report `133;D;130` (**130 = 128 + SIGINT(2)** — direct proof the interrupt reached `yes`), then the poll returns to blocking `-1` (idle) **[OBSERVED]**:

```
08:42:59.786354 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
08:42:59.787010 read(8, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;130\7\33]133;A\7\33]133;k;end_kitt"..., 1048576) = 165
08:42:59.787045 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 0 (Timeout)
08:42:59.789363 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <detached ...>
```

The buffer-size argument is back to the full `1048576` (the parser has drained everything), and the final blocking `poll(…, -1)` shows kitty idle again — the prompt has recovered. (The 165-B prompt read again carries the cwd in its OSC-2 title; the two extra bytes vs the Q2 163-B prompt are exactly the `130` exit code vs `0`.)

### Frequency & bytes-per-read distribution — documented stats [OBSERVED]

A documented Python script parsed each raw trace. It classifies a fd-8 read as a "flood read" if its payload contains `hello`; it reports the exact timestamp endpoints, the rate as an explicit division, the byte-count distribution (incl. mode), and — separately — the read-to-read cadence and the poll-to-read latency. Full output for run 1:

```
$ python3 q3_stats.py trace_q3_run1.log
flood_reads(n)          = 53995
first_ts                = 31374.756252
last_ts                 = 31379.785598
span_seconds            = 5.029346
reads_per_sec           = 53995/5.029346 = 10735.9883
bytes_total             = 31916126
bytes/read  min         = 41
bytes/read  max         = 20209
bytes/read  mean        = 591.094
bytes/read  median      = 504.0
bytes/read  mode        = 469  (freq=756, ties=1)
read->read  min/med/mean/max us = 49.0/55.0/58.0/2524.0
poll->read  min/med/mean/max us = 22.0/28.0/29.4/4935.0
backpressure fd8_events=0 polls = 0
poll timeouts (=0)      = 157
byte histogram (100B buckets): [0-99]=64 [100-199]=146 [200-299]=1012 [300-399]=6953 [400-499]=17875 [500-599]=14443 [600-699]=5687 [700-799]=2467 [800-inf]=5348
```

The rate is shown as the exact division `53995 / 5.029346 = 10735.9883 reads/s` (endpoints and precision explicit, so there is no rounding ambiguity). The dominant reads sit in the 400–599-byte range; median (504) and mode (469) are ~500 bytes. Per-run values and cross-run spread are in §9.

### Backpressure — the flow-control gate [OBSERVED engaged; run-to-run variable]

The read loop only asks for `POLLIN` on the PTY fd while the parser has room: `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;` at `kitty/child-monitor.c:1501`, where `vt_parser_has_space_for_input` (`kitty/vt-parser.c:1477`, body `ans = self->read.sz + self->write.pending < BUF_SZ;` at `kitty/vt-parser.c:1481`, declared `kitty/vt-parser.h:36`) reports whether the 1 MiB buffer still has space. Because the per-read request size is `BUF_SZ − write.offset` (`self->write.offset = self->read.sz + self->write.pending; *sz = BUF_SZ - self->write.offset;` in `vt_parser_create_write_buffer`, `kitty/vt-parser.c:1451`), the occupancy the gate watches equals `BUF_SZ − requested_size`, and the gate returns `false` — setting `events = 0` for fd 8 — **iff occupancy reaches `BUF_SZ` (100 %)**.

- **[OBSERVED]** peak occupancy is strongly **run-to-run variable and does reach the 1 MiB ceiling**. An expanded canonical sweep of **18 `yes hello` floods** (fourteen 8 s idle-host runs, one 20 s idle-host scale-up of ~205 k reads, and three 8 s runs under induced host-CPU contention — all typed into the real window, interrupted with a real Ctrl-C, traced on the I/O thread) measured per-run peak occupancy = `BUF_SZ − min(request size)` spanning **13.86 % up to 99.94 %** — i.e. from nearly empty to essentially the full 1 MiB cap. Representative peaks: an idle-host 8 s run reached **1,047,919 B (99.94 %)** (smallest read request `657 B`), another **957,168 B (91.28 %)**, another **812,562 B (77.49 %)**, while the majority stayed at ~14–23 %. The buffer therefore does fill to capacity on some runs, not merely "partway" toward the cap.
- **[OBSERVED]** `POLLIN` **withholding (full backpressure engagement) was directly observed** — in a minority of runs. Counting polls that watch fd 8 with `events=0`, engagement occurred in **2 of the 18 runs**: an idle-host 8 s run with **37** such polls (99.94 % peak occupancy) and a contended-host 8 s run with **1** (99.80 %); the other 16 runs — including the 20 s / ~205 k-read scale-up — showed **0**. Engagement is therefore **run-to-run variable and scheduling-sensitive**: whenever the main-thread parser keeps pace, occupancy stays low and `POLLIN` is never withheld; when the parser transiently falls behind (more likely under host-CPU contention, as on a shared multi-core host), occupancy hits `BUF_SZ` and the gate withholds `POLLIN`. Because engagement is intrinsically intermittent, its per-run *count* is **not** stable across runs — the stable, reproducible facts are the per-read/rate metrics in §9, while the reproducible *qualitative* fact confirmed here is that engagement **can and does** occur. The raw, unedited transition from an idle-host run (buffer filling → request size shrinking → `POLLIN` withheld) is:

  ```
  11:09:13.347120 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 5091) = 1867
  11:09:13.347149 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
  11:09:13.347178 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 3224) = 1876
  11:09:13.347217 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
  11:09:13.347247 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1348) = 1348
  11:09:13.347275 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 1) = 0 (Timeout)
  ```

  The request size collapses (`5091 → 3224 → 1348 B`) as the parser buffer fills, and the very next `poll` drops fd 8 to `events=0` — `POLLIN` withheld. A contended-host run shows the complementary **resume** path, where the withheld poll returns because the main thread posts its 8-byte wakeup token on fd 6:

  ```
  11:13:35.069865 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 2111) = 2111
  11:13:35.069900 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 1) = 1 ([{fd=6, revents=POLLIN}])
  11:13:35.070511 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8
  ```

- **[OBSERVED mechanism; INFERRED only for the exact `BUF_SZ` equality]** the two exit paths above are exactly the designed backpressure cycle: once occupancy reaches `BUF_SZ`, `vt_parser_has_space_for_input` returns `false`, `events` is set to `0` at `kitty/child-monitor.c:1501`, and no `read` is issued on fd 8 until the main thread drains the buffer via `parse_input` (`kitty/child-monitor.c:451`) → `do_parse` (`kitty/child-monitor.c:438`) and signals the I/O thread (the 8-byte `read(6, "\1\0…", 1024)` wakeup above). That engagement occurs (a poll with `events=0`) is **[OBSERVED]**; that it triggers *precisely at* `BUF_SZ` rather than some lower watermark is **[INFERRED]** from the gate predicate `read.sz + write.pending < BUF_SZ` (`kitty/vt-parser.c:1481`), independently corroborated by the observed request-size shrinkage to a few hundred bytes immediately before each `events=0`.

---

## 7. Q4 — File-descriptor number (PTY master side)

**Direct answer [OBSERVED]:** kitty reads the PTY master on **file descriptor 8** in this run. The concrete integer is **process-specific** — it is whatever number the OS assigned to the master when kitty opened the PTY — and must be read from the running process (it is not a fixed constant).

### Evidence

**From the syscall trace — every PTY payload read is on fd 8 [OBSERVED].** In the clean `echo test123` I/O-thread trace, reads land on exactly two fds; only fd 8 carries PTY payload, while fd 6 is the I/O thread's own wakeup eventfd (its reads return the 8-byte counter `"\1\0\0\0\0\0\0\0"` or `EAGAIN`):

```
$ grep -oE 'read\([0-9]+,' trace_q2_clean.log | sort | uniq -c
     26 read(6,
     16 read(8,
$ grep 'read(6,' trace_q2_clean.log | head -1
08:40:12.178208 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8
```

In each Q3 flood run the same holds — every PTY read is on fd 8 (run 1: 54,009 `read(8,` lines; the only non-fd-8 reads, 22 of them, are the fd-6 eventfd wakeups):

```
$ grep -c 'read(8,' trace_q3_run1.log
54009
```

**Cross-checked against `/proc` and `lsof` — fd 8 is the PTY master [OBSERVED]:**

```
$ ls -l /proc/100358/fd/8
lrwx------ 1 root root 64 ... /proc/100358/fd/8 -> /dev/pts/ptmx
$ lsof -p 100358 -a -d 8
COMMAND    PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
kitty   100358 root 8u   CHR    5,2      0t0    2 /dev/pts/ptmx
```

fd 8 is a character device `5,2` = `/dev/pts/ptmx` (the PTY master), whose slave counterpart is the shell's `/dev/pts/0` from Q1. It stayed fd 8 across all captures (Q1, Q2, and the four Q3 floods).

### Rationale & code citations [INFERRED]

The master fd is retained on the Python side as `self.child_fd = master` at `kitty/child.py:338` and made non-blocking via `os.set_blocking(self.child_fd, False)` at `kitty/child.py:345`. It is handed to the child monitor through `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` at `kitty/boss.py:587` (receiver `add_child()` at `kitty/child-monitor.c:305`). Inside the monitor it is registered into the poll set: `children_fds[EXTRA_FDS + self->count].fd = children[self->count].fd;` at `kitty/child-monitor.c:1286` with `children_fds[EXTRA_FDS + self->count].events = POLLIN;` at `kitty/child-monitor.c:1287`. The array is `static struct pollfd children_fds[MAX_CHILDREN + EXTRA_FDS]` at `kitty/child-monitor.c:86`, and `#define EXTRA_FDS 2` at `kitty/child-monitor.c:35` — which is why the PTY appears at poll index 2 (after the wakeup and signal fds) in the Q2/Q3 traces.

---

## 8. Q5 — Responsible C functions (reader + text/escape parser)

**Direct answer:**

- **(a) The function that reads from the PTY fd:** **`read_bytes()`** at `kitty/child-monitor.c:1337`, which issues the `read()` syscall at `kitty/child-monitor.c:1345`. It runs on the **I/O thread**.
- **(b) The function that parses input to separate printable text from escape sequences:** the VT state machine **`consume_input()`** at `kitty/vt-parser.c:1367`. It routes printable text through **`consume_normal()`** (`kitty/vt-parser.c:230`) to **`screen_draw_text()`** (`kitty/screen.c:866`), and escape/control sequences through **`consume_esc()`** (`kitty/vt-parser.c:261`) to the CSI/OSC/DCS/APC/PM/SOS sub-handlers. It runs on the **Main thread**.

### Reader — `read_bytes()` [OBSERVED it runs; INFERRED the code]

**[OBSERVED]** every PTY `read()` in the traces was issued by TID 100425, whose thread name is `KittyChildMon` (the KittyChildMonitor I/O thread):

```
$ cat /proc/100358/task/100425/comm
KittyChildMon
```

**[INFERRED]** the reader is `read_bytes()`. Verbatim source, `kitty/child-monitor.c:1337`–`1356`:

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

**[OBSERVED]** the raw bytes read from fd 8 contain **both** classes the parser must separate — printable text and escape/control sequences — and the terminal acted on both (it displayed `test123`/`hello` and honored the title/prompt escapes). The following lines are *quoted from the Q2/Q3 reads above* (they are excerpts of real reads, annotated to show the routing — the annotations after `#` are explanatory, not part of the raw bytes):

```
# printable text (from the reads in Q2/Q3):
hello\r\nhello\r\n...        # plain text -> consume_normal -> screen_draw_text
...test123\r\n               # plain text -> consume_normal -> screen_draw_text

# escape / control sequences (from the same reads):
\33]2;echo test123\7         # OSC 2 set-title            -> consume_esc
\33]133;C;cmdline=echo...\7  # OSC 133 shell integration  -> consume_esc
\33[?2004h  /  \33[?2004l    # CSI DECSET/DECRST (bracketed paste) -> consume_esc
```

**[INFERRED]** the routing. `consume_input()` switches on `self->vte_state`. Verbatim excerpt, `kitty/vt-parser.c:1367`–`1385` (an excerpt: the `switch` continues past line 1385 with the remaining `consume(...)` cases — `VTE_APC`, `VTE_PM`, `VTE_DCS`, `VTE_SOS` — through the closing brace):

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
```

Printable text is handled by `consume_normal()` (`kitty/vt-parser.c:230`), which decodes UTF-8 up to an ESC sentinel and pushes the decoded run to the screen — verbatim, `kitty/vt-parser.c:230`–`240`:

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

The `screen_draw_text(...)` call lands in `screen_draw_text()` at `kitty/screen.c:866` — the printable-text destination. When an ESC byte is hit, `consume_normal` sets state `ESC`, and the next dispatch enters `consume_esc()` (`kitty/vt-parser.c:261`), whose first-character switch selects the sub-state. Verbatim excerpt, `kitty/vt-parser.c:261`–`274` (an excerpt: the same `switch` continues past line 274 with the `IS_ESCAPED_CHAR` label at line 275, `ESC_RIS`/`ESC_IND`/… single-char escapes, and a `default`, ending at `kitty/vt-parser.c:288`):

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

This first-character `switch` is the exact point where an escape sequence's introducer selects the OSC/CSI/DCS/APC/PM/SOS sub-state — i.e. where escape sequences are separated from the printable text handled by `consume_normal`.

### Thread split [OBSERVED reader thread; INFERRED parser thread]

- **Reading — I/O thread.** `read_bytes()`/`read()` run in `io_loop` (`kitty/child-monitor.c:1481`). **[OBSERVED]** every PTY read came from TID 100425 (`KittyChildMon`).
- **Parsing — Main thread.** **[INFERRED]** `consume_input` is reached from the main loop: `parse_input()` at `kitty/child-monitor.c:451` — whose own comment at `kitty/child-monitor.c:452` reads `// Parse all available input that was read in the I/O thread.` — calls `do_parse()` at `kitty/child-monitor.c:438`, which invokes `self->parse_func`. In the default build `parse_func = parse_worker` (`kitty/child-monitor.c:181`; the `parse_worker_dump` variant at `:180` is only selected when a dump callback is set), and `parse_worker`/`run_worker` (`kitty/vt-parser.c:1417`) call `consume_input`. The `Screen` owns the parser via `Parser *vt_parser;` at `kitty/screen.h:158`.

---

## 9. Stability confirmation (Q3) — four runs, per-metric spread

The same unchanged input (`yes hello` typed into the real window, interrupted with a real Ctrl-C) was run **three times at a 5 s scale** (the stability triple) plus a **10 s scale-up** run (run 4) to probe for buffer saturation. All measurements are on the I/O thread (TID 100425), under strace, via the documented `q3_stats.py`. Every run recovered the prompt (exit-status `133;D;130`). **[OBSERVED]**:

| Run | Wall window | Flood reads | Reads/sec | Mean B | Median B | Mode B | Max B | read→read med (µs) | poll→read med (µs) | `POLLIN` withheld (this run) |
|-----|-------------|-------------|-----------|--------|----------|--------|-------|--------------------|--------------------|-------------------|
| 1   | 5.029346 s  | 53 995      | 10 735.99 | 591.09 | 504 | 469 | 20 209 | 55 | 28 | 0 |
| 2   | 5.039648 s  | 51 410      | 10 201.11 | 668.35 | 546 | 511 | 20 186 | 57 | 28 | 0 |
| 3   | 5.030843 s  | 51 555      | 10 247.79 | 557.35 | 523 | 504 | 14 389 | 57 | 28 | 0 |
| 4 (10 s scale-up) | 10.049398 s | 109 463 | 10 892.49 | 585.80 | 530 | 511 | 20 237 | 55 | 28 | 0 |

**Per-metric spread across the three 5 s runs** (range ÷ mean):

| Metric | min | max | spread |
|--------|-----|-----|--------|
| poll→read latency (median µs) | 28 | 28 | **0.00 %** |
| read→read cadence (median µs) | 55 | 57 | **3.55 %** |
| reads/sec | 10 201.11 | 10 735.99 | **5.15 %** |
| bytes/read median | 504 | 546 | **8.01 %** |
| bytes/read mode | 469 | 511 | **8.49 %** |
| bytes/read mean | 557.35 | 668.35 | **18.33 %** |

**Explicit stability criterion & conclusion.** Treating a metric as *stable* when its range ÷ mean ≤ 10 % across the three unchanged 5 s runs: the timing metrics (poll→read 0.00 %, read→read 3.55 %), the read rate (5.15 %), and the byte central tendency (median 8.01 %, mode 8.49 %) **all pass**, and the 10 s scale-up (run 4) falls inside the same bands. The one metric that does **not** pass is the **mean** bytes/read (18.33 %) — because the per-read size distribution is heavy-tailed (individual reads span single digits to ~14–20 KB), the arithmetic mean is a poor stability estimator; the **median (~504–546 B)** and **mode (~469–511 B)** are the robust "typical bytes-per-read" figures. Increasing the scale to 10 s / ~109 k reads did not change the per-read/rate picture. In these particular four runs the main-thread parser kept pace and the `POLLIN`-withheld count was `0` — but that outcome is **run-to-run variable, not a ceiling**: an expanded canonical sweep (18 floods) drove peak occupancy to **~99.9 %** and **did withhold `POLLIN`** (poll `events=0` on fd 8) in some runs, so the `0` column above reflects these four scheduling-lucky runs rather than a scale limit (see §6, *Backpressure — the flow-control gate*, for the raw `events=0` evidence and the full distribution). Caveat: these cadences are measured **under strace**, which adds per-syscall overhead, so absolute reads/sec is a lower bound; the qualitative change (back-to-back large reads vs. blocking 1-byte reads) and the byte-size distribution are the robust findings.

---

## 10. Coverage pass — every named item answered

| # | Named item asked | Concrete answer (OBSERVED unless noted) | `file:line` |
|---|------------------|------------------------------------------|-------------|
| Q1a | Process spawned | `/bin/bash` (account shell, POSIX mode — not a login shell) | `kitty/constants.py:181` |
| Q1b | PID | `100426` | `ps` / `/proc/100426` |
| Q1c | Exact command line | `/bin/bash --posix` (`/bin/bash^@--posix^@`) | argv `kitty/child.py:314`; exec `kitty/child.c:159` |
| Q1d | PTY device path | slave `/dev/pts/0`; master `/dev/pts/ptmx` (fd 8) | dup2 `kitty/child.c:138`–`145` |
| Q2a | System calls | `poll()` then `read()` on fd 8 (write `poll→write` for the keystroke also observed) | `kitty/child-monitor.c:1512`/`1509`; read `:1345` |
| Q2b | Buffer size | `1048576` (= `BUF_SZ` 1 MiB); shrinks by `write.offset` | `kitty/vt-parser.c:18`; `:1451` |
| Q2c | Bytes returned | 1 per keystroke; Enter 11 (2 echo + 9 control); output 47/114/163 → **347 total, 16 reads** | reader `kitty/child-monitor.c:1337` |
| Q3a | How behavior changes | blocking 1-byte reads → back-to-back multi-hundred-byte reads; poll timeout −1 → 1–2 ms | `kitty/child-monitor.c:1509`/`1512`; gate `:1501` |
| Q3b | Frequency | ~10 201–10 892 reads/sec (4 runs); read→read median 55–57 µs (poll→read latency 28 µs, distinct) | dispatch `kitty/child-monitor.c:1529`–`1531` |
| Q3c | Typical bytes/read | median 504–546, mode 469–511 (robust); mean 557–668; up to ~20 KB | buffer `kitty/vt-parser.c:18` |
| Q3 stability | ≥2-run stability | stable across 3×5 s + 1×10 s; per-metric spread + criterion | §9 |
| Q3 backpressure | flow-control gate | occupancy run-to-run variable, peaking ~14 % → **99.94 %**; `POLLIN` **withheld (`events=0`) — OBSERVED** in a minority of runs (engagement at `BUF_SZ`) | `kitty/child-monitor.c:1501`; `kitty/vt-parser.c:1481` |
| Q4 | fd integer | **8** (process-specific; `/dev/pts/ptmx`) | reg. `kitty/child-monitor.c:1286`; retain `kitty/child.py:338` |
| Q5a | Reader function | `read_bytes()` — I/O thread (TID 100425 `KittyChildMon`) | `kitty/child-monitor.c:1337` (read `:1345`) |
| Q5b | Parser function | `consume_input()` — Main thread | `kitty/vt-parser.c:1367` |
| Q5b | Text path | `consume_normal()` → `screen_draw_text()` | `kitty/vt-parser.c:230` → `kitty/screen.c:866` |
| Q5b | Escape path | `consume_esc()` → CSI/OSC/DCS/APC/PM/SOS | `kitty/vt-parser.c:261` |

---

## 11. Observed-vs-Inferred summary

| Fact | Classification | Basis |
|------|----------------|-------|
| Shell is `/bin/bash`, PID 100426, cmdline `/bin/bash --posix`, session-leader/foreground (not login) | **[OBSERVED]** | `ps`, `cat -A /proc/100426/cmdline` |
| PTY slave `/dev/pts/0`; master fd 8 `/dev/pts/ptmx` (5,2) | **[OBSERVED]** | `readlink /proc/.../fd`, `ls -l`, `lsof` |
| `poll → write → poll → read` per keystroke on fd 8; read buffer arg `1048576` | **[OBSERVED]** | `strace -tt -s 512` (write included) |
| Returned bytes: 1/keystroke, 11 Enter (2 echo + 9 control), 47/114/163 output, 347 total / 16 reads | **[OBSERVED]** | `strace` (clean single-thread; `-f` reconciled) |
| `yes hello`: ~10.2–10.9 k reads/s; read→read 55–57 µs vs poll→read 28 µs; median ~500 B; stable ×3 + 10 s | **[OBSERVED]** | `strace` + `q3_stats.py`, 4 runs |
| Ctrl-C → exit status `133;D;130` (SIGINT) → prompt recovers | **[OBSERVED]** | `strace` after-state |
| Parser-buffer occupancy is run-to-run variable (peak ~14 % → 99.94 %); `POLLIN` **withheld (`events=0`) observed** in some runs, not in others | **[OBSERVED]** | `strace` read 3rd arg + `events=` field |
| fd number = 8 | **[OBSERVED]** | `strace`, `/proc`, `lsof` |
| Reader executes on the I/O thread (`KittyChildMon`, TID 100425) | **[OBSERVED]** | `/proc/.../task/100425/comm` + `strace` |
| Both text and escape bytes are present in the reads | **[OBSERVED]** | `strace` payloads |
| Shell resolution `pw_shell or '/bin/sh'` | **[INFERRED]** | `kitty/constants.py:181` |
| Spawn chain (fork/setsid/TIOCSCTTY/dup2/execvp) | **[INFERRED]** | `kitty/child.c:81`–`159` |
| `read()` lives in `read_bytes()`; buffer sizing formula | **[INFERRED]** | `kitty/child-monitor.c:1337`/`1345`; `kitty/vt-parser.c:1451` |
| Backpressure engagement — `POLLIN` withheld (poll `events=0` on fd 8) under buffer saturation — **directly observed** (run-to-run variable); that it triggers *precisely at* `BUF_SZ` is inferred from the gate predicate | **[OBSERVED engagement; INFERRED exact threshold]** | `strace` `events=0` polls; `kitty/child-monitor.c:1501`; `kitty/vt-parser.c:1481` |
| Text/escape routing `consume_input`→`consume_normal`/`consume_esc` | **[INFERRED]** | `kitty/vt-parser.c:1367`/`230`/`261` |
| Parsing dispatched on the Main thread | **[INFERRED]** | `kitty/child-monitor.c:451`/`452`/`438` |
| fd registration into the poll set | **[INFERRED]** | `kitty/child-monitor.c:1286`/`1287`/`35`/`86` |

---

*All runtime values above were captured from a default-configuration kitty (v0.35.2) built with `make` (`python3 setup.py`, `Makefile:13`) from a disposable copy under `/tmp`, and launched headless under an access-controlled Xvfb, with input delivered exclusively through the canonical terminal keyboard path. Temporary observation scripts, strace logs, and the disposable build copy were created only under a private `mktemp -d` scratch directory and removed after the investigation; the Xvfb (PID 100092) and kitty (PID 100358) processes were terminated in teardown. The repository working tree is unchanged apart from this document, as confirmed by `git status --porcelain` returning empty for all tracked source files (only this file is added).*
