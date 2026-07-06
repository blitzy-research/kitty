# How kitty reads from and communicates with the child shell it spawns over a PTY

**Question answered:** *How does kitty read from and communicate with the child shell it spawns over a pseudo‑terminal (PTY)?* — decomposed into five named requirements **R1–R5** (shell spawn; single‑command read path; high‑volume read behavior; master file‑descriptor number; the responsible C functions).

This is a **runtime code‑behavior investigation**. Every factual claim below is backed by **either the actual, unedited output I captured at runtime** (with the exact command that produced it) **or an exact `file:line` code reference** in the kitty source at the branch/commit under test. Anything derived by reading rather than observed at runtime is explicitly tagged **(inferred)**.

- **Repository:** kitty terminal emulator, source branch **`kitty_815df1e210e0`**, `HEAD` **`815df1e21`** (`git rev-parse --short HEAD` → `815df1e21`).
- **Built version:** `kitty 0.35.2 created by Kovid Goyal`.
- **Method summary:** built kitty from source with the canonical entry point, launched it headless under Xvfb in its default configuration, then drove the **real keyboard→PTY path** (X11 keystroke injection with `xdotool`, i.e. `XTEST` synthetic key events delivered to the focused kitty window) — **not** kitty's remote‑control interface — while tracing the I/O thread with `strace` and inspecting the process/descriptor state with `ps`, `pstree`, `lsof`, and `/proc/<pid>/fd`.
- **Dynamic values disclaimer:** the shell PID (**59571**), the PTY master fd number (**8**), and the slave device (**`/dev/pts/0`**) are assigned dynamically per run and are reported **as observed in this run**, not as fixed constants (see R1/R4 for why).

---

## Summary / TL;DR

Kitty creates a PTY master/slave pair in Python (`os.openpty()`, `kitty/child.py:171`), forks in C (`kitty/child.c:97`), makes the slave the child's controlling terminal and dups it onto the child's stdin/stdout/stderr (`kitty/child.c:129`,`:138‑146`), then `execvp`s the user's login shell (`kitty/child.c:159`). The parent keeps the **master** fd (`self.child_fd = master`, `kitty/child.py:338`) and sets it **non‑blocking** (`os.set_blocking(child_fd, False)`, `kitty/child.py:345`). A dedicated I/O thread — `io_loop()` (`kitty/child-monitor.c:1481`, created at `:291`) — `poll()`s the master and, on `POLLIN`, calls **`read_bytes()`** (`kitty/child-monitor.c:1337`) which does `read(fd, buf, available_buffer_space)` (`:1345`) into the VT parser's single **1 MiB** buffer (`BUF_SZ`, `kitty/vt-parser.c:18`). The parser state machine **`consume_input()`** (`kitty/vt-parser.c:1367`) → **`consume_normal()`** (`:230`) then separates printable text (sent to `screen_draw_text()`, `:236`) from escape sequences (which flip the machine into the ESC state, `:238`).

| # | Question | Answer (observed this run) | Basis |
|---|----------|-----------------------------|-------|
| **R1** | Spawned process, PID, exact command line, PTY device path | process **`bash`** (`/usr/bin/bash`), PID **59571**, command line **`/bin/bash --posix`**, slave device **`/dev/pts/0`** | observed (`ps`,`/proc`) + code |
| **R2** | Read syscalls, buffer size, bytes returned (`echo test123`) | gating **`poll()`** then **`read(8, buf, size)`**; requested size **1 048 576 B = 1 MiB (`BUF_SZ`)** when the buffer is empty, smaller (`BUF_SZ − offset`) when partly full; the `echo` output `test123\r\n` came back in a single **114‑byte** read | observed (`strace`) + code |
| **R3** | Read frequency & typical bytes/read (`yes hello`) | reads become **continuous / back‑to‑back**; **native** ≈ **100 000–127 000 reads/s**, ≈ **73–90 bytes/read**, ≈ **8.7–9.3 MB/s** (stable over 4 runs); **under `strace`** ≈ **8 800–9 700 reads/s**, **median ≈ 525–553 B/read** (max ≈ 20 KB) over 2 runs; per‑read size capped at 1 MiB (`BUF_SZ`) | observed (`/proc` io + `strace`) + code |
| **R4** | PTY master fd number | fd **8** → `/dev/pts/ptmx` (dynamic per run) | observed (`/proc/<pid>/fd`,`lsof`) + code |
| **R5** | Reader function / parser function | reader **`read_bytes()`** (`kitty/child-monitor.c:1337`, `read()` at `:1345`); parser **`consume_input()`/`consume_normal()`** (`kitty/vt-parser.c:1367`/`:230`) | observed behavior + code |

---

## 1. Build & Launch (canonical, default configuration)

### 1.1 Build — exact command

Kitty's canonical build entry point is `python3 setup.py`, which the `Makefile` invokes for the `all` target:

```
# Makefile:12-13
all:
	python3 setup.py $(VVAL)
```

I built from the repository root with exactly that command:

```
$ python3 setup.py
...
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
$ echo $?
0
```

The build succeeded (exit code `0`). It is an **X11‑only** canonical build: `setup.py` automatically disables the Wayland backend when `wayland-protocols` is absent, which is the normal behavior in this container and matches kitty's own CI. The build produces the launcher at **`kitty/launcher/kitty`**, which reports its version:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

`setup.py` enforces the minimum Python version at startup via `check_version_info()` (`setup.py:30`); the repo manifests declare `requires-python = ">=3.8"` (`pyproject.toml:2`) and `go 1.22` (`go.mod:3`).

### 1.2 Launch — exact command (headless, default config)

Kitty is a GPU‑accelerated OpenGL GUI, so it needs a display. I launched it headlessly under a virtual X server, in its **default, unconfigured** state (`--config NONE`):

```
# 1) headless X server
$ Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp &

# 2) kitty in default configuration (software GL for the headless GPU)
$ DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8 \
      ./kitty/launcher/kitty --config NONE &
```

`--config NONE` documents that no user configuration is applied — this is exactly the default behavior a normal user gets out of the box. All keystrokes in the sections below were delivered to the **real terminal** by injecting X11 key events into the focused kitty window with `xdotool` (`XTEST`); kitty's remote‑control interface (`kitten @ …`) was **never** used.

### 1.3 Tool / version record

| Tool | Version | Role |
|------|---------|------|
| kitty (built) | 0.35.2 | subject under test |
| Python | 3.13.7 | canonical build interpreter / kitty embeds CPython |
| Go | go1.24.4 | builds the `kitten` binary (satisfies `go.mod` ≥ 1.22) |
| gcc | 15.2.0 | C11 toolchain |
| strace | 6.16 | syscall tracing of the I/O thread |
| Xvfb | 21.1.18 | headless X server |
| xdotool | 3.20160805.1 | real keyboard→PTY keystroke injection (XTEST) |
| lsof / pstree / ps | 4.99 / 23.7 / procps | process & descriptor inspection |

**Process identifiers used throughout (this run):** kitty GUI process **PID 59503**; its I/O thread **TID 59570** (kernel thread name `KittyChildMon`); the spawned shell **PID 59571**. `strace` attaches cleanly because the investigation runs as `root` (so Yama `ptrace_scope=1` does not block attaching to the child threads).

---

## 2. R1 — The spawned shell: process, PID, exact command line, and PTY device

### 2.1 Observed process tree

```
$ pstree -aps 59503
tini,1 -- /app/start.sh
  `-kitty,59503 --config NONE
      |-bash,59571 --posix
      |-{kitty},59505
      ... (rendering / worker threads elided) ...
      `-{kitty},59570
```

kitty (PID 59503) has exactly **one** child process: a shell, **PID 59571**. (The `{kitty},NNNNN` entries are threads of the kitty process, not child processes.)

### 2.2 Spawned process name, PID, and exact command line

```
$ ps -o pid,ppid,args -p 59571
    PID    PPID COMMAND
  59571   59503 /bin/bash --posix

$ cat /proc/59571/comm
bash
$ readlink /proc/59571/exe
/usr/bin/bash
$ tr '\0' ' ' < /proc/59571/cmdline; echo
/bin/bash --posix
```

- **Spawned process (name):** `bash` — the executable resolves to `/usr/bin/bash`.
- **PID:** **59571** (this run; dynamic).
- **Exact command line (verbatim from the process list):** **`/bin/bash --posix`**.

**Why bash, and why `--posix`?** The default shell is the current user's login shell, resolved by kitty at `kitty/constants.py:181`:

```python
# kitty/constants.py:181
shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'
```

For the default (unconfigured) case, `resolved_shell()` returns exactly `[shell_path]` (`kitty/utils.py:768‑771`: `q = getattr(opts, 'shell', '.')`; `if q == '.': ans = [shell_path]`). On this host the login shell of the running user is `/bin/bash` (confirmed independently: `getent passwd root` → `…:/bin/bash`, and `pwd.getpwuid(os.geteuid()).pw_shell` → `/bin/bash`), so kitty spawns bash.

The trailing **`--posix`** is added by kitty's **default‑enabled bash shell integration**, not by me. When shell integration is not disabled, `kitty/child.py:265‑267` calls `modify_shell_environ(...)`, and for bash `kitty/shell_integration.py:146` does `argv.insert(1, '--posix')` and points the shell at kitty's integration script via `env['ENV']` (`kitty/shell_integration.py:134`). This is corroborated by the child's environment:

```
$ tr '\0' '\n' < /proc/59571/environ | grep -E 'KITTY_SHELL_INTEGRATION|^ENV=|TERM=|KITTY_PID'
ENV=/tmp/blitzy/kitty/blitzy-1c0b6d1e-8884-44c6-810e-14d23dfb0918_4f8a70/shell-integration/bash/kitty.bash
KITTY_PID=59503
KITTY_SHELL_INTEGRATION=enabled
TERM=xterm-kitty
```

`TERM=xterm-kitty` matches kitty's default `term` option (`kitty/options/definition.py:3242`). So `/bin/bash --posix` **is** the canonical default command line — reported exactly as observed.

**Platform note (inferred from code, not exercised):** On Linux the login shell is spawned **directly**, as observed. The `run-shell` kitten wrapper (and `/usr/bin/login`) is **macOS‑only**: `kitty/child.py:230` sets `should_run_via_run_shell_kitten = is_macos and self.is_default_shell`. That path was not taken here and is noted only as a documented platform variant.

### 2.3 The PTY device that connects kitty to the shell

The shell's standard streams are all connected to the **slave** side of the PTY:

```
$ readlink /proc/59571/fd/0    ->  /dev/pts/0
$ readlink /proc/59571/fd/1    ->  /dev/pts/0
$ readlink /proc/59571/fd/2    ->  /dev/pts/0
```

- **PTY device path (slave, as seen by the shell):** **`/dev/pts/0`** (this run; dynamic — the `N` in `/dev/pts/N` depends on what the kernel allocates).

This wiring is set up in C: after `fork()` (`kitty/child.c:97`) the child obtains the slave name with `ttyname_r(slave, …)` (`kitty/child.c:88`), makes it the controlling terminal with `ioctl(tfd, TIOCSCTTY, 0)` (`:129`), dups the slave onto stdout/stderr/stdin (`safe_dup2(slave, STDOUT_FILENO/STDERR_FILENO/STDIN_FILENO)`, `:138‑146`), and finally `execvp(exe, argv)` (`:159`). The master/slave pair itself is created earlier in Python by `os.openpty()` (`kitty/child.py:171`).

---

## 3. R4 — The PTY master file‑descriptor number kitty reads from

kitty holds the **master** end of the PTY. In this run it is **fd 8**:

```
$ ls -l /proc/59503/fd
lr-x------ 1 root root 64 Jul  6 22:00 0 -> /dev/null
l-wx------ 1 root root 64 Jul  6 22:00 1 -> /tmp/kitty_probe/kitty.log
l-wx------ 1 root root 64 Jul  6 22:00 2 -> /tmp/kitty_probe/kitty.log
lrwx------ 1 root root 64 Jul  6 22:00 3 -> socket:[521056143]
lrwx------ 1 root root 64 Jul  6 22:00 4 -> anon_inode:[eventfd]
lrwx------ 1 root root 64 Jul  6 22:00 5 -> /memfd:allocation fd (deleted)
lrwx------ 1 root root 64 Jul  6 22:00 6 -> anon_inode:[eventfd]
lrwx------ 1 root root 64 Jul  6 22:00 7 -> anon_inode:[signalfd]
lrwx------ 1 root root 64 Jul  6 22:00 8 -> /dev/pts/ptmx

$ lsof -p 59503 | grep -i -E 'ptmx|/dev/pts'
kitty   59503 root    8u   CHR    5,2   0t0   2   /dev/pts/ptmx
```

- **Master fd number kitty reads the PTY from:** **8** (this run; dynamic).
- It is the master side: it points at `/dev/pts/ptmx` (character device major/minor **5,2**), i.e. the multiplexer that hands out the master. The matching slave is the shell's `/dev/pts/0` (R1).

**The fd number is assigned dynamically** — it is whatever the kernel returns from `os.openpty()` (`kitty/child.py:171`), stored in the parent as `self.child_fd = master` (`kitty/child.py:338`) and tracked in C as `children[i].fd`. It therefore differs between runs and **must not** be treated as a fixed constant; here it happened to be 8.

The other descriptors seen in the `poll()` set below are, from the same listing: **fd 6 = an eventfd** (the I/O‑thread wakeup) and **fd 7 = a signalfd**. Only **fd 8** is the child PTY master.

---

## 4. R2 — `echo test123`: the read syscalls, the buffer size, and the bytes returned

I attached `strace` to the I/O thread (TID 59570) and typed `echo test123` + Return into the real terminal via `xdotool`:

```
$ strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p 59570 -o /tmp/kitty_probe/echo.strace &
$ DISPLAY=:99 xdotool windowfocus 2097164
$ DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'echo test123'
$ DISPLAY=:99 xdotool key  --clearmodifiers Return
```

### 4.1 The syscalls: a gating `poll()` then a `read()`

Every read of the PTY master is gated by a `poll()`. The read syscall is the plain **`read(2)`** (not `readv`), always on **fd 8**. As I typed, each keystroke was echoed by bash and read back one byte at a time (the injection delay of 45 ms/key separates them). These are the **actual, unedited** strace lines:

```
22:02:16.442273 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}]) <0.000011>
22:02:16.442352 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000100>
22:02:16.442471 read(8, "e", 1048576)   = 1 <0.000011>
22:02:16.460438 read(8, "c", 1048576)   = 1 <0.000011>
22:02:16.483168 read(8, "h", 1048576)   = 1 <0.000010>
22:02:16.506059 read(8, "o", 1048576)   = 1 <0.000009>
22:02:16.528882 read(8, " ", 1048576)   = 1 <0.000010>
22:02:16.551707 read(8, "t", 1048576)   = 1 <0.000010>
22:02:16.574740 read(8, "e", 1048576)   = 1 <0.000011>
22:02:16.597571 read(8, "s", 1048576)   = 1 <0.000010>
22:02:16.620410 read(8, "t", 1048576)   = 1 <0.000010>
22:02:16.643275 read(8, "1", 1048576)   = 1 <0.000010>
22:02:16.666295 read(8, "2", 1048576)   = 1 <0.000010>
22:02:16.689079 read(8, "3", 1048576)   = 1 <0.000010>
```

Note the `{fd=8, revents=POLLOUT}` line: that is kitty **writing** the typed keystrokes to the master (the outbound half of the conversation), gated by the same `poll()`. The inbound reads are the bash **echo** of those keystrokes.

When I pressed Return, bash executed the command and the output came back. This is the **complete, unedited** `poll()`→`read()` sequence for the command result:

```
22:02:17.017906 read(8, "\r\n\33[?2004l\r", 1048576) = 11 <0.000010>
22:02:17.017962 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.001361>
22:02:17.019356 read(8, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47 <0.000015>
22:02:17.019393 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000178>
22:02:17.019592 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114 <0.000011>
22:02:17.019621 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000429>
22:02:17.020084 read(8, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]133;k;start_suffix_kitty\7\33[5 q\33]2;/tmp/blitzy/kitty/blitzy-1c0b6d1e-8884-44c6-810e-14d23dfb0918_4f8a70\7\33]133;k;end_suffix_kitty\7", 1048404) = 194 <0.000017>
```

### 4.2 What the evidence says

- **Read syscalls:** the gating **`poll()`** (on the fd set `{6, 7, 8}`, requesting `POLLIN` for the child fd 8) followed by **`read(8, buf, size)`**. No `readv`/`ppoll` was used for the PTY read. This is exactly `read_bytes()` → `read(fd, buf, available_buffer_space)` (`kitty/child-monitor.c:1337`, `:1345`), driven by the poll loop that calls it on `POLLIN` (`:1531`).
- **Buffer size requested (the 3rd `read` argument):**
  - **1 048 576 bytes = 1 MiB** whenever the parser buffer was empty (all the per‑keystroke reads: `read(8, "e", 1048576)`), which equals `BUF_SZ` (`kitty/vt-parser.c:18`, `#define BUF_SZ (1024u*1024u)`).
  - **Smaller than 1 MiB when the buffer already held un‑parsed bytes:** `1048565`, `1048518`, `1048404`. These equal `BUF_SZ − self->write.offset` — exactly the formula at `kitty/vt-parser.c:1457` (`*sz = BUF_SZ - self->write.offset`). For example `1048576 − 1048565 = 11`, matching the 11 bytes read just before but not yet drained by the parser worker.
- **Bytes returned (the `read` return value):** small counts for a single command — **1** byte per echoed keystroke, then **11**, **47**, **114**, **194** for the newline/mode‑reset, the shell‑integration title+command markers, the actual command **output**, and the redrawn prompt. The command's own output `test123\r\n` is plainly visible at the end of the **`= 114`** read (the leading `\33]133;…` bytes are bash shell‑integration OSC 133 markers).

So for a single small command the pattern is: **one `poll()` per readiness edge, then one `read(8, buf, ≤1 MiB)`**, each returning only the handful of bytes currently available. The 1 MiB is the **capacity offered** to `read()`, not the amount returned.

---

## 5. R3 — `yes hello`: how the read behavior changes under a continuous, high‑volume stream

`yes hello` writes an unbounded stream of `hello\n` lines. I typed it into the real terminal (again via `xdotool`, never remote control), let it run for a **fixed 5‑second window**, then stopped it with `Ctrl+C`. Because `strace` itself perturbs syscall timing, I measured the magnitude **two independent ways** and repeated each to confirm stability:

1. **Under `strace`** — gives the exact syscall sequence and per‑read byte counts.
2. **Natively via `/proc/<pid>/task/<tid>/io`** (the kernel's `syscr` read‑syscall counter and `rchar` bytes‑read counter, sampled before/after with **no** tracer attached) — gives the true, unperturbed magnitude.

### 5.1 The syscall pattern (under `strace`) — reads become continuous and back‑to‑back

```
$ strace -tt -T -s 24 -e trace=read,readv,poll,ppoll -p 59570 -o /tmp/kitty_probe/yes1.strace &
$ DISPLAY=:99 xdotool windowfocus 2097164
$ DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'yes hello'
$ DISPLAY=:99 xdotool key  --clearmodifiers Return
   # ... 5 s ...
$ DISPLAY=:99 xdotool key  --clearmodifiers ctrl+c
```

Representative **consecutive** lines mid‑stream (unedited):

```
22:03:53.269896 read(8, "\r\nhello\r\nhello\r\nhello\r\nh"..., 1014726) = 590 <0.000012>
22:03:53.269933 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}]) <0.000020>
22:03:53.269979 read(8, "hello\r\nhello\r\nhello\r\nhel"..., 1014136) = 756 <0.000013>
22:03:53.270016 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}]) <0.000011>
22:03:53.270054 read(8, "hello\r\nhello\r\nhello\r\nhel"..., 1013380) = 670 <0.000012>
22:03:53.270113 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
22:03:53.270151 read(8, "\r\nhello\r\nhello\r\nhello\r\nh"..., 1012710) = 672 <0.000012>
22:03:53.270187 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
22:03:53.270225 read(8, "\r\nhello\r\nhello\r\nhello\r\nh"..., 1012038) = 625 <0.000012>
22:03:53.270301 read(8, "hello\r\nhello\r\nhello\r\nhel"..., 1011413) = 775 <0.000012>
```

Two things change dramatically versus the idle/typing case:

1. **Reads are continuous and back‑to‑back.** The gating `poll()` almost always returns **immediately** — its timeout argument is now `0` or a small value (`2`), and it returns `1 ([{fd=8, revents=POLLIN}])` without waiting, because data is essentially always available. (Contrast the idle case in §7, where `poll(..., -1)` blocks indefinitely.)
2. **Each read coalesces many lines.** Instead of 1 byte per read, each `read()` now returns hundreds of bytes of `hello\r\nhello\r\n…` — dozens of lines per read.
3. **The requested size shrinks below 1 MiB.** The 3rd `read` argument decreases across consecutive reads (`1014726 → 1014136 → 1013380 → 1012710 → 1012038 → 1011413 …`). That is `BUF_SZ − self->write.offset` (`kitty/vt-parser.c:1457`) with the offset growing because the parser worker (a separate thread) has not yet drained everything — i.e. the buffer is filling.

### 5.2 Frequency and bytes/read — measured, two ways, repeated for stability

**Under `strace` (5‑second window, computed from the `read(8,…)` lines):**

| Run | reads | window | reads/sec | bytes/read (min / median / mean / max) |
|-----|-------|--------|-----------|-----------------------------------------|
| 1 | 50 678 | 5.24 s | **9 670** | 1 / **525** / 639.5 / 20 305 |
| 2 | 46 283 | 5.27 s | **8 775** | 1 / **553** / 672.1 / 19 789 |

Distribution of bytes/read (run 1): `<256 B`: 536; `256–1 KB`: 46 805; `1–4 KB`: 3 218; `4–16 KB`: 77; `16–64 KB`: 42. Requested‑size range observed: run 1 `min 901 000 … max 1 048 576`; run 2 `min 631 712 … max 1 048 576` (i.e. the buffer occupancy grew to ~150 KB and ~417 KB respectively before the parser caught up — never reaching the 1 MiB cap in these runs).

**Native, no tracer (sampling `/proc/59503/task/59570/io`):**

```
# before/after over the timed window:  syscr rchar
before = 114494 37287339
after  = 657972 86006158
# => reads = 543478,  bytes = 48718819,  over 5.01 s
```

| Run | window | reads/sec | bytes/read | throughput |
|-----|--------|-----------|------------|------------|
| A | 5.01 s | **108 505** | **90 B** | 9.3 MB/s |
| B | 5.02 s | 101 759 | 90 B | 8.7 MB/s |
| C | 5.01 s | 126 867 | 73 B | 8.8 MB/s |
| D | 10.01 s | 117 888 | 77 B | 8.7 MB/s |

**Stability:** both measurements are stable across repeats. Natively kitty performs **≈ 100 000–127 000 reads/second at ≈ 73–90 bytes/read for a steady ≈ 8.7–9.3 MB/s**; under `strace` the rate is **≈ 8 800–9 700 reads/second at a median ≈ 525–553 bytes/read** (with occasional reads up to ~20 KB). I re‑ran until the numbers repeated within the same range; increasing the window from 5 s to 10 s (run D) did not shift them, confirming the observed magnitude is real and not a startup transient.

### 5.3 Reconciling the two numbers, and the role of `input_delay` (3 ms)

The native and traced figures differ because **`strace` adds per‑syscall `ptrace` overhead that slows the reader**: a slower reader lets *more* bytes accumulate between reads (median jumps from ~90 B to ~525 B) and issues *fewer* reads per second (~9 k vs ~110 k). Both are legitimate observations of the same behavior at different reader speeds; the **native** numbers are the true magnitude.

The key qualitative result answers R3 directly: under a saturating stream, **kitty's reads change from occasional/1‑byte (idle/typing) to continuous, back‑to‑back reads that each coalesce many lines**, bounded above by the **1 MiB** buffer (`BUF_SZ`, `kitty/vt-parser.c:18`). The coalescing knob is `input_delay` (**default 3 ms**, `kitty/options/definition.py:878`): the poll timeout is computed as `time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at)` and used as `poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta))` (`kitty/child-monitor.c:1508‑1509`) — you can see the resulting `3` and `2` timeout values in the strace above. But under a truly saturating stream the timeout rarely governs, for two reasons visible in the code and the trace: (a) `poll()` returns immediately anyway because data is already ready (timeout `0` lines), and (b) the option is, by its own documentation, **"ignored when the input buffer is almost full"** (`kitty/options/definition.py:878`). That is why the reads become large and continuous rather than being paced at 3 ms intervals.

---

## 6. R5 — The responsible C functions

### 6.1 The reader — `read_bytes()` (`kitty/child-monitor.c:1337`)

The function that reads from the PTY master file descriptor is **`read_bytes(int fd, Screen *screen)`**. Its `read()` at line 1345 is the exact syscall observed on fd 8 throughout §4 and §5:

```c
// kitty/child-monitor.c:1337
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;

    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space); // :1341
    if (!available_buffer_space) return true;                                                 // :1342

    while(true) {
        len = read(fd, buf, available_buffer_space);           // :1345  <-- the PTY-master read
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;   // :1347  retry (non-blocking master)
            if (errno != EIO) perror("Call to read() from child fd failed"); // :1348
            vt_parser_commit_write(screen->vt_parser, 0);      // :1349
            return false;                                      // :1350  child dead
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);            // :1354  hand bytes to the parser
    return len != 0;                                           // :1355  len==0 => child dead
}
```

- The **buffer** and its **available size** come from `vt_parser_create_write_buffer()` (`kitty/vt-parser.c:1451`), whose returned size is `BUF_SZ − self->write.offset` (`:1457`) — this is precisely the varying 3rd argument to `read()` I observed (1 048 576 down to 631 712).
- After reading, the bytes are committed to the parser with `vt_parser_commit_write()` (`:1354`, defined at `kitty/vt-parser.c:1465`).
- **Threading context:** `read_bytes()` runs on the dedicated I/O thread `io_loop()` (`kitty/child-monitor.c:1481`), created by `pthread_create(&self->io_thread, NULL, io_loop, self)` (`:291`) — the kernel thread I traced as `KittyChildMon`/TID 59570. `io_loop()` builds the `poll()` set (requesting `POLLIN` for a child only while `vt_parser_has_space_for_input()` is true, `:1501`), calls `poll()` (`:1509`/`:1512`), and on `POLLIN`/`POLLHUP` invokes `has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)` (`:1531`). This is the reader half of the answer.

### 6.2 The parser that separates printable text from escape sequences — `consume_input()` → `consume_normal()`

The function that parses the incoming bytes and separates **printable text** from **escape sequences** is the VT state machine **`consume_input()`** (`kitty/vt-parser.c:1367`), which dispatches on the current state:

```c
// kitty/vt-parser.c:1367
consume_input(PS *self, PyObject *dump_callback UNUSED, id_type window_id UNUSED) {
    ...
    switch (self->vte_state) {
        case VTE_NORMAL:
            consume_normal(self); self->read.consumed = self->read.pos; break;   // :1377
        case VTE_ESC:
            if (consume_esc(self)) { self->read.consumed = self->read.pos; }      // :1379
            break;
        case VTE_CSI:
            if (consume_csi(self)) { ... }
            break;
        ...
    }
}
```

The actual text/escape split happens inside **`consume_normal()`** (`kitty/vt-parser.c:230`):

```c
// kitty/vt-parser.c:230
consume_normal(PS *self) {
    do {
        const bool sentinel_found = utf8_decode_to_esc(&self->utf8_decoder,
                self->buf + self->read.pos, self->read.sz - self->read.pos);      // :232  decode until an ESC sentinel
        self->read.pos += self->utf8_decoder.num_consumed;
        if (self->utf8_decoder.output.pos) {
            REPORT_DRAW(...);
            screen_draw_text(self->screen, self->utf8_decoder.output.storage,
                             self->utf8_decoder.output.pos);                      // :236  printable text -> screen
        }
        if (sentinel_found) { SET_STATE(ESC); break; }                           // :238  escape byte -> switch to ESC state
    } while (self->read.pos < self->read.sz);
}
```

- **Printable text** is decoded by `utf8_decode_to_esc()` (`:232`), which consumes bytes **up to the next ESC sentinel**, and is written to the screen via **`screen_draw_text()`** (`:236`) — the printable‑text sink. In the `echo test123` capture, the `test123` characters flow through exactly this path.
- **Escape sequences** are detected when `utf8_decode_to_esc()` reports `sentinel_found`; `consume_normal()` then does `SET_STATE(ESC)` (`:238`), handing control to `consume_esc()` / `consume_csi()` on subsequent iterations of `consume_input()` (`:1379` and the `VTE_CSI` case). The `\33]133;…\7` (OSC 133 shell‑integration) and `\33[?2004h` (bracketed‑paste) sequences visible in the `echo` reads take this branch.

**Threading / API boundary:** the parser is driven by `parse_worker()` → `run_worker()` (`kitty/vt-parser.c:1496`/`:1417`), invoked from `kitty/screen.c:4775‑4776`. The reader (§6.1) and the parser communicate only through the thread‑safe API declared in `kitty/vt-parser.h:34‑38` (`vt_parser_create_write_buffer`, `vt_parser_commit_write`, `vt_parser_has_space_for_input`, `parse_worker`); the `Screen` owns the parser (`kitty/screen.h:158`, `Parser *vt_parser;`). This decoupling is exactly why the observed `read()` byte counts (child‑monitor.c) and the 1 MiB buffer capacity (vt‑parser.c) line up: the reader fills a region of the parser's single locked buffer, then the parser thread drains it.

---

## 7. Edge and transitional states

I exercised the read loop through its transitions, not just the happy path.

**Idle — `poll()` blocks with no reads.** With no output pending, the I/O thread parks in a blocking `poll(..., -1)`. Attaching `strace` to the quiescent thread yields a single blocking `poll()` that never returns (I detached while it was still blocked):

```
$ strace -tt -T -e trace=read,readv,poll,ppoll -p 59570 -o /tmp/kitty_probe/idle_block.strace
22:09:22.773216 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:09:22.773267 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <detached ...>
```

The final line is the idle state: `poll(..., -1)` — a **blocking, infinite‑timeout** wait with **zero reads**. Independently, the kernel confirms the thread is parked in poll: `cat /proc/59503/task/59570/wchan` → `do_sys_poll`. This is the blocking branch at `kitty/child-monitor.c:1512` (`poll(children_fds, self->count + EXTRA_FDS, -1)`), taken when there are no pending wakeups.

**Typing — one small read per keystroke/line.** Shown in §4: `read(8, "e", 1048576) = 1`, etc.; the command output arrives in a single 114‑byte read.

**High volume — sustained larger, back‑to‑back reads with the requested size shrinking below 1 MiB.** Shown in §5: `poll()` returns immediately (timeout `0`/`2`), reads coalesce many lines, and the 3rd `read` argument decreases (`1014726 → … → 1011413`) as the buffer fills (`BUF_SZ − offset`, `kitty/vt-parser.c:1457`).

**Return to idle — the blocking‑`poll()` pattern is restored.** After `Ctrl+C` stops `yes`, tracing the I/O thread again shows **no reads** and the thread back in `do_sys_poll`:

```
$ strace -tt -T -e trace=read,readv,poll,ppoll -p 59570 -o /tmp/kitty_probe/idle2.strace   # (empty: 0 reads)
$ cat /proc/59503/task/59570/wchan
do_sys_poll
```

**Non‑blocking master / retry branch.** The master is non‑blocking (`os.set_blocking(child_fd, False)`, `kitty/child.py:345`), so `read_bytes()` wraps `read()` in a retry loop for `EINTR`/`EAGAIN` (`kitty/child-monitor.c:1347`). Because a read is only issued after `poll()` reports `POLLIN`, I observed **zero `EAGAIN` on fd 8** across both `yes` runs — data was always available when the read ran. The non‑blocking‑drain pattern *was* observed on the wakeup **eventfd (fd 6)**, which is read until it returns `EAGAIN`:

```
22:02:16.442050 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8 <0.000018>
22:02:16.442212 read(6, 0x7bdfdfd9d740, 1024) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
```

**Child‑death branches (inferred — not exercised).** `read_bytes()` treats `len == 0` and `EIO` as the child having exited (`kitty/child-monitor.c:1349‑1350`, `:1355`); I did not kill the shell, so these branches were not triggered here. Labeled **(inferred)** from the code.

**Buffer‑full gating (inferred — not triggered).** When the parser buffer fills completely, `read_bytes()` returns early without reading (`if (!available_buffer_space) return true;`, `kitty/child-monitor.c:1342`) and `io_loop()` stops requesting `POLLIN` for that child (`events = … ? POLLIN : 0`, `:1501`). In my runs the parser kept pace, so I never observed a `poll()` entry with `fd=8, events=0` (grep count: 0 in both `yes` traces). This gating is therefore reported **(inferred)** from the code, not observed at runtime.

---

## 8. Reasoning & inferred‑vs‑observed

**End‑to‑end story, tied to evidence.** kitty allocates the PTY in Python (`os.openpty()`, `child.py:171`), forks/execs the shell in C so the shell's stdio is the slave `/dev/pts/0` (`child.c:88`,`:138‑159`), and keeps the non‑blocking master (fd 8 this run; `child.py:338`,`:345`). Its I/O thread `io_loop()` `poll()`s that master and calls `read_bytes()` → `read(8, buf, BUF_SZ−offset)` on readiness (`child-monitor.c:1481`,`:1531`,`:1337`,`:1345`), depositing bytes into the parser's 1 MiB buffer (`vt-parser.c:18`,`:1451`). `consume_input()`/`consume_normal()` then split printable text (→ `screen_draw_text`) from escape sequences (→ `SET_STATE(ESC)`) (`vt-parser.c:1367`,`:230`,`:236`,`:238`). Every arrow here is backed by a captured `strace`/`ps`/`lsof`/`/proc` line above and a verified `file:line`.

**Values observed at runtime (this run; dynamic):** shell PID **59571**; command line **`/bin/bash --posix`**; slave **`/dev/pts/0`**; master fd **8**; the `poll()`+`read()` syscalls, the requested buffer sizes (`1048576`, `1048565`, `1048518`, `1048404`, `1014726` …), and the byte counts (`1`, `11`, `47`, `114`, `194`, and the `yes` distribution); read frequency and bytes/read for `yes hello` (native ≈ 110 k reads/s @ ≈ 80 B; strace ≈ 9 k reads/s @ median ≈ 540 B). These are specific to this run — the PID, the fd number, and the `/dev/pts/N` index will differ next time.

**Facts grounded in code (stable across runs):** the reader is `read_bytes()` and the syscall is `read(fd, buf, available_buffer_space)` (`child-monitor.c:1337`/`:1345`); the buffer cap is `BUF_SZ` = 1 MiB (`vt-parser.c:18`); the requested size is `BUF_SZ − write.offset` (`:1457`); the poll timeout derives from `input_delay` = 3 ms (`options/definition.py:878`, `child-monitor.c:1508‑1512`); the parser is `consume_input()`/`consume_normal()` (`vt-parser.c:1367`/`:230`); the default shell resolution is `constants.py:181` + `utils.py:768`.

**Explicitly inferred (not directly observed here):**
- The **child‑death** handling (`len==0`/`EIO`) in `read_bytes()` (`child-monitor.c:1349‑1350`,`:1355`) — code‑supported, not exercised.
- The **buffer‑full `POLLIN`‑off gating** (`child-monitor.c:1342`,`:1501`) — code‑supported, never triggered because the parser kept pace.
- The **macOS `run-shell`/`login` wrapper** variant (`child.py:230`) — Linux took the direct‑spawn path; the macOS path is noted from code only.
- The strace‑vs‑native frequency gap is attributed to `strace`'s per‑syscall `ptrace` overhead; the direction (tracer slows the reader ⇒ fewer, larger reads) is a reasoned interpretation of the two measured datasets, while the datasets themselves are observed.

---

## Appendix A — Reproducibility (exact commands)

```bash
# Build (canonical entry point) — Makefile:12-13
python3 setup.py
./kitty/launcher/kitty --version                     # -> kitty 0.35.2

# Launch headless in default config
Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp &
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8 \
    ./kitty/launcher/kitty --config NONE &
KPID=$(pgrep -n -f 'launcher/kitty')                 # kitty GUI pid  (59503 this run)
IOTID=$(ps -T -p "$KPID" -o tid,comm | awk '/KittyChildMon/{print $1}')  # io thread (59570)
WID=$(DISPLAY=:99 xdotool search --class kitty | head -1)

# R1/R4 — process & descriptors
SPID=$(ps --ppid "$KPID" -o pid=)                    # shell pid (59571 this run)
pstree -aps "$KPID"; ps -o pid,ppid,args -p "$SPID"
ls -l /proc/$KPID/fd; lsof -p "$KPID" | grep -iE 'ptmx|/dev/pts'
for fd in 0 1 2; do readlink /proc/$SPID/fd/$fd; done

# R2 — echo test123 under strace (real keyboard->PTY path)
strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p "$IOTID" -o echo.strace &
DISPLAY=:99 xdotool windowfocus "$WID"
DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'echo test123'
DISPLAY=:99 xdotool key  --clearmodifiers Return

# R3 — yes hello: strace window + native /proc io counters, repeated
strace -tt -T -s 24 -e trace=read,readv,poll,ppoll -p "$IOTID" -o yes1.strace &
DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'yes hello'; DISPLAY=:99 xdotool key Return
# ... fixed 5 s window ...; then: DISPLAY=:99 xdotool key ctrl+c
# native magnitude (no tracer): sample syscr & rchar before/after over the window
awk -F': ' '/^syscr/{print $2} /^rchar/{print $2}' /proc/$KPID/task/$IOTID/io
```

## Appendix B — Verified code reference index (HEAD `815df1e21`)

| Area | Reference |
|------|-----------|
| Default shell | `kitty/constants.py:181`; `kitty/utils.py:768‑771` |
| PTY create / keep master / non‑blocking | `kitty/child.py:171`, `:338`, `:345`; direct‑vs‑macOS `:230` |
| bash shell integration (`--posix`) | `kitty/child.py:265‑267`; `kitty/shell_integration.py:134`, `:146` |
| Fork / controlling TTY / dup / exec | `kitty/child.c:88`, `:97`, `:129`, `:138‑146`, `:159` |
| I/O thread & poll loop | `kitty/child-monitor.c:291`, `:1481`, `:1501`, `:1508‑1512`, `:1531` |
| Reader `read_bytes()` + `read()` | `kitty/child-monitor.c:1337`, `:1341`, `:1342`, `:1345`, `:1347‑1350`, `:1354‑1355` |
| 1 MiB buffer / write API / requested size | `kitty/vt-parser.c:18`, `:1451`, `:1457`, `:1465`, `:1477` |
| Parser (text vs. escape) | `kitty/vt-parser.c:1367`, `:1377`, `:1379`, `:230`, `:232`, `:236`, `:238`; workers `:1417`, `:1496` |
| Reader↔parser API / Screen owns parser | `kitty/vt-parser.h:34‑38`; `kitty/screen.c:4775‑4776`; `kitty/screen.h:158` |
| `input_delay` (3 ms) / default `term` | `kitty/options/definition.py:878`, `:3242` |
| Build / manifests | `Makefile:12‑13`; `setup.py:30`; `pyproject.toml:2`; `go.mod:3` |

*All values labeled “this run” (PID 59571, master fd 8, `/dev/pts/0`) are dynamic and were captured live; they will differ on other runs. All `file:line` references were verified against the source at `HEAD 815df1e21`.*





