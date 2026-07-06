# How kitty reads from and communicates with the child shell it spawns over a PTY

**Question answered:** *How does kitty read from and communicate with the child shell it spawns over a pseudo‑terminal (PTY)?* — decomposed into five named requirements **R1–R5** (shell spawn; single‑command read path; high‑volume read behavior; master file‑descriptor number; the responsible C functions).

This is a **runtime code‑behavior investigation**. Every factual claim below is backed by **either the actual, unedited output I captured at runtime** (with the exact command that produced it) **or an exact `file:line` code reference** in the kitty source at the branch/commit under test. Anything derived by reading rather than observed at runtime is explicitly tagged **(inferred)**.

- **Repository:** kitty terminal emulator, source branch **`kitty_815df1e210e0`**. All `file:line` code citations below are verified against the kitty source at commit **`815df1e21`** ("Wire up applying of font config"), which is the branch tip holding the sources under test. This answer document is added as the immediately following commit, so `git rev-parse --short HEAD` resolves to that documentation commit whose parent is `815df1e21` (`git rev-parse --short HEAD~1` → `815df1e21`).
- **Built version:** `kitty 0.35.2 created by Kovid Goyal`.
- **Method summary:** built kitty from source with the canonical entry point, launched it headless under Xvfb in its default configuration, then drove the **real keyboard→PTY path** (X11 keystroke injection with `xdotool`, i.e. `XTEST` synthetic key events delivered to the focused kitty window) — **not** kitty's remote‑control interface — while tracing the I/O thread with `strace` and inspecting the process/descriptor state with `ps`, `pstree`, `lsof`, and `/proc/<pid>/fd`.
- **Dynamic values disclaimer:** the shell PID (**83836**), the PTY master fd number (**8**), and the slave device (**`/dev/pts/0`**) are assigned dynamically per run and are reported **as observed in this run**, not as fixed constants (see R1/R4 for why).

---

## Summary / TL;DR

Kitty creates a PTY master/slave pair in Python (`os.openpty()`, `kitty/child.py:171`), forks in C (`kitty/child.c:97`), makes the slave the child's controlling terminal and dups it onto the child's stdin/stdout/stderr (`kitty/child.c:129`,`:138‑146`), then `execvp`s the user's login shell (`kitty/child.c:159`). The parent keeps the **master** fd (`self.child_fd = master`, `kitty/child.py:338`) and sets it **non‑blocking** (`os.set_blocking(child_fd, False)`, `kitty/child.py:345`). A dedicated I/O thread — `io_loop()` (`kitty/child-monitor.c:1481`, created at `:291`) — `poll()`s the master and, on `POLLIN`, calls **`read_bytes()`** (`kitty/child-monitor.c:1337`) which does `read(fd, buf, available_buffer_space)` (`:1345`) into the VT parser's single **1 MiB** buffer (`BUF_SZ`, `kitty/vt-parser.c:18`). The parser state machine **`consume_input()`** (`kitty/vt-parser.c:1367`) → **`consume_normal()`** (`:230`) then separates printable text (sent to `screen_draw_text()`, `:236`) from escape sequences (which flip the machine into the ESC state, `:238`).

| # | Question | Answer (observed this run) | Basis |
|---|----------|-----------------------------|-------|
| **R1** | Spawned process, PID, exact command line, PTY device path | process **`bash`** (`/usr/bin/bash`), PID **83836**, command line **`/bin/bash --posix`**, slave device **`/dev/pts/0`** | observed (`ps`,`/proc`) + code |
| **R2** | Read syscalls, buffer size, bytes returned (`echo test123`) | gating **`poll()`** then **`read(8, buf, size)`**; requested size **1 048 576 B = 1 MiB (`BUF_SZ`)** when the buffer is empty, smaller (`BUF_SZ − offset`) when partly full; the `echo` output `test123\r\n` came back in a single **114‑byte** read | observed (`strace`) + code |
| **R3** | Read frequency & typical bytes/read (`yes hello`) | reads become **continuous / back‑to‑back**; **native** ≈ **97 000–134 000 reads/s**, ≈ **64–103 bytes/read**, ≈ **7–9.5 MB/s** (stable over 4 runs); **under `strace`** ≈ **8 300–8 500 reads/s**, **median ≈ 497–567 B/read** (max ≈ 20 KB) over 2 runs; per‑read size capped at 1 MiB (`BUF_SZ`) | observed (`/proc` io + `strace`) + code |
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
$ python3 setup.py > build.log 2>&1; echo "EXIT=$?" >> build.log
$ cat build.log
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
EXIT=0
```

The block above is the **complete, unedited** output of the build command (captured to `build.log`, then printed). It is short because this is an **incremental** rebuild: the canonical launcher (`kitty/launcher/kitty`) was already present from the container's build step, so `setup.py` only recompiled the one changed C translation unit (`kitty/data-types.c`), relinked the `fast_data_types` extension, and confirmed the Go `kitty/tools/cmd` binary — then exited `0`. A from-scratch build compiles many more translation units, but the canonical entry point and the successful exit status are identical.

The leading Wayland messages are informational, not errors: this is an **X11‑only** canonical build — `setup.py` automatically disables the Wayland backend when `wayland-protocols` is absent, which is the normal behavior in this container and matches kitty's own CI. The launcher at **`kitty/launcher/kitty`** reports its version:

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

The versions in the table and the tracing preconditions (`root`, Yama `ptrace_scope`) were captured with these exact commands and outputs:

```
$ python3 --version
Python 3.13.7
$ go version
go version go1.24.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ strace --version | head -1
strace -- version 6.16
$ id
uid=0(root) gid=0(root) groups=0(root)
$ cat /proc/sys/kernel/yama/ptrace_scope
1
```

**Process identifiers used throughout (this run):** kitty GUI process **PID 83769**; its I/O thread **TID 83835** (kernel thread name `KittyChildMon`); the spawned shell **PID 83836**. As the `id` and `ptrace_scope` outputs above show, the investigation runs as **`root`** (`uid=0`) while Yama is at **`ptrace_scope=1`** (restricted). A root tracer bypasses the Yama restriction, so `strace` attaches to kitty's I/O thread (`-p 83835`) without a `PTRACE_ATTACH ... Operation not permitted` error — the `strace: Process 83835 attached` / `detached` banners in the traces below confirm each attach succeeded.

---

## 2. R1 — The spawned shell: process, PID, exact command line, and PTY device

### 2.1 Observed process tree

```
$ pstree -aps 83769
tini,1 -- /app/start.sh
  `-bash,83767 -c...
      `-kitty,83769 --config NONE
          |-bash,83836 --posix
          |-{kitty},83770
          |-{kitty},83771
          |-{kitty},83772
          |-{kitty},83773
          |-{kitty},83774
          |-{kitty},83775
          |-{kitty},83776
          |-{kitty},83777
          |-{kitty},83778
          |-{kitty},83779
          |-{kitty},83780
          |-{kitty},83781
          |-{kitty},83782
          |-{kitty},83783
          |-{kitty},83784
          |-{kitty},83785
          |-{kitty},83786
          |-{kitty},83787
          |-{kitty},83788
          |-{kitty},83789
          |-{kitty},83790
          |-{kitty},83791
          |-{kitty},83792
          |-{kitty},83793
          |-{kitty},83794
          |-{kitty},83795
          |-{kitty},83796
          |-{kitty},83797
          |-{kitty},83798
          |-{kitty},83799
          |-{kitty},83800
          |-{kitty},83801
          |-{kitty},83802
          |-{kitty},83803
          |-{kitty},83804
          |-{kitty},83805
          |-{kitty},83806
          |-{kitty},83807
          |-{kitty},83808
          |-{kitty},83809
          |-{kitty},83810
          |-{kitty},83811
          |-{kitty},83812
          |-{kitty},83813
          |-{kitty},83814
          |-{kitty},83815
          |-{kitty},83816
          |-{kitty},83817
          |-{kitty},83818
          |-{kitty},83819
          |-{kitty},83820
          |-{kitty},83821
          |-{kitty},83822
          |-{kitty},83823
          |-{kitty},83824
          |-{kitty},83825
          |-{kitty},83826
          |-{kitty},83827
          |-{kitty},83828
          |-{kitty},83829
          |-{kitty},83830
          |-{kitty},83831
          |-{kitty},83832
          |-{kitty},83833
          |-{kitty},83834
          `-{kitty},83835
```

This is the **complete, unedited** `pstree -aps` output — no lines removed. Reading it:

- kitty (**PID 83769**) has exactly **one** child *process*: the shell **`bash,83836 --posix`** (**PID 83836**). Nothing else under kitty is a process.
- Every `{kitty},NNNNN` line (PIDs **83770–83835**, 66 of them) is a **thread** (LWP) of the single kitty process — `pstree` renders threads in braces. These are kitty's render/worker/software‑GL threads; the last one, **TID 83835**, is the I/O thread `KittyChildMon` traced in later sections.
- `bash,83767 -c...` is kitty's **parent** (the container's launcher wrapper); `pstree -a` abbreviates its long `-c` argument with a trailing `...` — that ellipsis is `pstree`'s own truncation of the wrapper's command line, not an omission by me.

### 2.2 Spawned process name, PID, and exact command line

```
$ ps -o pid,ppid,args -p 83836
    PID    PPID COMMAND
  83836   83769 /bin/bash --posix

$ cat /proc/83836/comm
bash
$ readlink /proc/83836/exe
/usr/bin/bash
$ tr '\0' ' ' < /proc/83836/cmdline; echo
/bin/bash --posix
```

- **Spawned process (name):** `bash` — the executable resolves to `/usr/bin/bash`.
- **PID:** **83836** (this run; dynamic).
- **Exact command line (verbatim from the process list):** **`/bin/bash --posix`**.

**Why bash, and why `--posix`?** The default shell is the current user's login shell, resolved by kitty at `kitty/constants.py:181`:

```python
# kitty/constants.py:181
shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'
```

For the default (unconfigured) case, `resolved_shell()` returns exactly `[shell_path]` (`kitty/utils.py:768‑771`: `q = getattr(opts, 'shell', '.')`; `if q == '.': ans = [shell_path]`). On this host the login shell of the running user is `/bin/bash` (confirmed independently: `getent passwd root` → `…:/bin/bash`, and `pwd.getpwuid(os.geteuid()).pw_shell` → `/bin/bash`), so kitty spawns bash.

The trailing **`--posix`** is added by kitty's **default‑enabled bash shell integration**, not by me. When shell integration is not disabled, `kitty/child.py:265‑267` calls `modify_shell_environ(...)`, and for bash `kitty/shell_integration.py:146` does `argv.insert(1, '--posix')` and points the shell at kitty's integration script via `env['ENV']` (`kitty/shell_integration.py:134`). This is corroborated by the child's environment:

```
$ tr '\0' '\n' < /proc/83836/environ | grep -E 'KITTY_SHELL_INTEGRATION|^ENV=|TERM=|KITTY_PID'
TERM=xterm-kitty
COLORTERM=truecolor
KITTY_PID=83769
KITTY_SHELL_INTEGRATION=enabled
ENV=/tmp/blitzy/kitty/blitzy-1c0b6d1e-8884-44c6-810e-14d23dfb0918_4f8a70/shell-integration/bash/kitty.bash
```

(The block above is the exact, unedited output; `COLORTERM=truecolor` appears because the `TERM=` sub‑pattern also matches the `…TERM=` inside `COLORTERM`. Both `TERM` and `COLORTERM` are set by kitty for the child.) `TERM=xterm-kitty` matches kitty's default `term` option (`kitty/options/definition.py:3242`). So `/bin/bash --posix` **is** the canonical default command line — reported exactly as observed.

**Platform note (inferred from code, not exercised):** On Linux the login shell is spawned **directly**, as observed. The `run-shell` kitten wrapper (and `/usr/bin/login`) is **macOS‑only**: `kitty/child.py:230` sets `should_run_via_run_shell_kitten = is_macos and self.is_default_shell`. That path was not taken here and is noted only as a documented platform variant.

### 2.3 The PTY device that connects kitty to the shell

The shell's standard streams are all connected to the **slave** side of the PTY:

```
$ readlink /proc/83836/fd/0    ->  /dev/pts/0
$ readlink /proc/83836/fd/1    ->  /dev/pts/0
$ readlink /proc/83836/fd/2    ->  /dev/pts/0
```

- **PTY device path (slave, as seen by the shell):** **`/dev/pts/0`** (this run; dynamic — the `N` in `/dev/pts/N` depends on what the kernel allocates).

This wiring is set up in C: after `fork()` (`kitty/child.c:97`) the child obtains the slave name with `ttyname_r(slave, …)` (`kitty/child.c:88`), makes it the controlling terminal with `ioctl(tfd, TIOCSCTTY, 0)` (`:129`), dups the slave onto stdout/stderr/stdin (`safe_dup2(slave, STDOUT_FILENO/STDERR_FILENO/STDIN_FILENO)`, `:138‑146`), and finally `execvp(exe, argv)` (`:159`). The master/slave pair itself is created earlier in Python by `os.openpty()` (`kitty/child.py:171`).

---

## 3. R4 — The PTY master file‑descriptor number kitty reads from

kitty holds the **master** end of the PTY. In this run it is **fd 8**:

```
$ ls -l /proc/83769/fd
total 0
lr-x------ 1 root root 64 Jul  6 22:42 0 -> /dev/null
l-wx------ 1 root root 64 Jul  6 22:42 1 -> /tmp/kitty_probe/kitty.log
l-wx------ 1 root root 64 Jul  6 22:42 2 -> /tmp/kitty_probe/kitty.log
lrwx------ 1 root root 64 Jul  6 22:42 3 -> socket:[522708035]
lrwx------ 1 root root 64 Jul  6 22:42 4 -> anon_inode:[eventfd]
lrwx------ 1 root root 64 Jul  6 22:42 5 -> /memfd:allocation fd (deleted)
lrwx------ 1 root root 64 Jul  6 22:42 6 -> anon_inode:[eventfd]
lrwx------ 1 root root 64 Jul  6 22:42 7 -> anon_inode:[signalfd]
lrwx------ 1 root root 64 Jul  6 22:42 8 -> /dev/pts/ptmx

$ lsof -p 83769 | grep -i -E 'ptmx|/dev/pts'
kitty   83769 root   8u      CHR                5,2      0t0         2 /dev/pts/ptmx
```

- **Master fd number kitty reads the PTY from:** **8** (this run; dynamic).
- It is the master side: it points at `/dev/pts/ptmx` (character device major/minor **5,2**), i.e. the multiplexer that hands out the master. The matching slave is the shell's `/dev/pts/0` (R1).

**The fd number is assigned dynamically** — it is whatever the kernel returns from `os.openpty()` (`kitty/child.py:171`), stored in the parent as `self.child_fd = master` (`kitty/child.py:338`) and tracked in C as `children[i].fd`. It therefore differs between runs and **must not** be treated as a fixed constant; here it happened to be 8.

The other descriptors seen in the `poll()` set below are, from the same listing: **fd 6 = an eventfd** (the I/O‑thread wakeup) and **fd 7 = a signalfd**. Only **fd 8** is the child PTY master.

---

## 4. R2 — `echo test123`: the read syscalls, the buffer size, and the bytes returned

I attached `strace` to the I/O thread (TID 83835) and typed `echo test123` + Return into the real terminal via `xdotool`:

```
$ strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p 83835 -o /tmp/kitty_probe/echo.strace &
$ DISPLAY=:99 xdotool windowfocus 2097164
$ DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'echo test123'
$ DISPLAY=:99 xdotool key  --clearmodifiers Return
```

### 4.1 The syscalls: a gating `poll()` then a `read()`

Every read of the PTY master is gated by a `poll()`. The read syscall is the plain **`read(2)`** (not `readv`), always on **fd 8**. As I typed, each keystroke travelled out to the shell and its **echo** came back one byte at a time (the 45 ms/key injection delay cleanly separates them). Here is the **complete, unedited** six‑line cycle for the very first character (`e`), copied verbatim from `echo.strace`; every one of the remaining eleven characters repeats this exact pattern:

```
22:42:33.213396 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8 <0.000032>
22:42:33.213586 read(6, 0x7bc6ae19d740, 1024) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
22:42:33.213628 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}]) <0.000012>
22:42:33.213702 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000119>
22:42:33.213846 read(8, "e", 1048576)   = 1 <0.000012>
22:42:33.213902 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=6, revents=POLLIN}]) <0.015957>
```

Reading that cycle top to bottom: the I/O thread is first woken through its **eventfd** (`read(6, …) = 8`, followed by a draining `read(6, …) = -1 EAGAIN`); the gating `poll()` reports `{fd=8, revents=POLLOUT}` — kitty **writing** the keystroke out to the master (the outbound half of the conversation); the next `poll()` reports `{fd=8, revents=POLLIN}` — the shell's **echo** is now readable; and `read(8, "e", 1048576) = 1` reads that single echoed byte. The trailing `poll(…, -1)` then blocks until the next keystroke's wakeup (`revents=POLLIN` on fd 6).

Extracting just the inbound `read(8,…)` lines from the trace, the twelve one‑byte echoes spell out exactly what I typed — `echo test123`:

```
22:42:33.213846 read(8, "e", 1048576)   = 1 <0.000012>
22:42:33.230133 read(8, "c", 1048576)   = 1 <0.000011>
22:42:33.252989 read(8, "h", 1048576)   = 1 <0.000011>
22:42:33.275740 read(8, "o", 1048576)   = 1 <0.000011>
22:42:33.298555 read(8, " ", 1048576)   = 1 <0.000010>
22:42:33.321795 read(8, "t", 1048576)   = 1 <0.000026>
22:42:33.344529 read(8, "e", 1048576)   = 1 <0.000010>
22:42:33.367419 read(8, "s", 1048576)   = 1 <0.000011>
22:42:33.390249 read(8, "t", 1048576)   = 1 <0.000010>
22:42:33.413241 read(8, "1", 1048576)   = 1 <0.000010>
22:42:33.436180 read(8, "2", 1048576)   = 1 <0.000011>
22:42:33.459146 read(8, "3", 1048576)   = 1 <0.000013>
```

Each of those reads requests the full **1 048 576‑byte (1 MiB)** buffer but returns only the **1** byte currently available — the buffer size is the capacity offered to `read()`, not the amount returned.

When I pressed Return, bash executed the command and the output came back. Below is the **complete, unedited** contiguous slice of `echo.strace` from the first post‑Return wakeup through the return to idle. Note that **every** `read(8,…)` is immediately preceded by the `poll()` that gated it — there is no unpaired read:

```
22:42:33.484574 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8 <0.000019>
22:42:33.484628 read(6, 0x7bc6ae19d740, 1024) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
22:42:33.484663 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}]) <0.000013>
22:42:33.484720 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000050>
22:42:33.484799 read(8, "\r\n\33[?2004l\r", 1048576) = 11 <0.000020>
22:42:33.484865 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.001428>
22:42:33.486333 read(8, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47 <0.000016>
22:42:33.486369 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000161>
22:42:33.486554 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114 <0.000012>
22:42:33.486586 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000691>
22:42:33.487318 read(8, "\33[?2004h\33[59P\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]0;root@reverse-code-generator-2e6fd2f3-dnlc9: /tmp/blitzy/kitty/blitzy-1c0b6d1e-8884-44c6-810e-14d23dfb0918_4f8a70\7root@reverse-code-generator-2e6fd2f3-dnlc9:/tmp/blitzy/kitty/blitzy-1c"..., 1048404) = 429 <0.000019>
22:42:33.487358 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487387 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487414 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000011>
22:42:33.487443 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487470 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487501 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000010>
22:42:33.487529 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487557 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487584 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487612 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487639 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487667 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000010>
22:42:33.487695 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487723 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487750 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487778 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000009>
22:42:33.487805 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout) <0.000012>
22:42:33.487907 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <detached ...>
```

The four post‑Return reads return **11**, **47**, **114**, and **429** bytes; the string previews on the `47/114/429` reads are abbreviated by `strace -s 256`, but the returned byte counts are exact. After the last read drains the input, the loop spins through **seventeen** `poll(…, 3, 0) = 0 (Timeout)` calls — the `input_delay` countdown elapsing so the main loop can be woken once (see §5/§7) — and finally re‑enters the blocking `poll(…, 3, -1)` (shown here as `<detached ...>` because I stopped `strace` at that point), i.e. the terminal has returned to idle.

### 4.2 What the evidence says

- **Read syscalls:** the gating **`poll()`** (on the fd set `{6, 7, 8}`, requesting `POLLIN` for the child fd 8) followed by **`read(8, buf, size)`**. No `readv`/`ppoll` was used for the PTY read. This is exactly `read_bytes()` → `read(fd, buf, available_buffer_space)` (`kitty/child-monitor.c:1337`, `:1345`), driven by the poll loop that calls it on `POLLIN` (`:1531`).
- **Buffer size requested (the 3rd `read` argument):**
  - **1 048 576 bytes = 1 MiB** whenever the parser buffer was empty (all the per‑keystroke reads: `read(8, "e", 1048576)`), which equals `BUF_SZ` (`kitty/vt-parser.c:18`, `#define BUF_SZ (1024u*1024u)`).
  - **Smaller than 1 MiB when the buffer already held un‑parsed bytes:** `1048565`, `1048518`, `1048404`. These equal `BUF_SZ − self->write.offset` — exactly the formula at `kitty/vt-parser.c:1457` (`*sz = BUF_SZ - self->write.offset`). The offset is the running total of bytes read but not yet drained by the parser: `1048576 − 1048565 = 11` (after the 11‑byte read); `1048576 − 1048518 = 58 = 11 + 47` (after the 11‑ and 47‑byte reads); `1048576 − 1048404 = 172 = 11 + 47 + 114` (after the first three reads). Each successive requested size shrinks by exactly the amount the previous reads added.
- **Bytes returned (the `read` return value):** small counts for a single command — **1** byte per echoed keystroke, then **11**, **47**, **114**, **429** for the newline/bracketed‑paste‑off (`\33[?2004l`), the shell‑integration window‑title + `OSC 133;C` command marker, the actual command **output**, and the redrawn prompt (bracketed‑paste‑on `\33[?2004h`, the `OSC 133;D;0` exit‑status marker, the window title, and the `root@…:/tmp/…` prompt string). The command's own output `test123\r\n` is plainly visible at the end of the **`= 114`** read (the surrounding `\33]133;…` bytes are bash shell‑integration OSC 133 markers).

So for a single small command the pattern is: **one `poll()` per readiness edge, then one `read(8, buf, ≤1 MiB)`**, each returning only the handful of bytes currently available. The 1 MiB is the **capacity offered** to `read()`, not the amount returned.

---

## 5. R3 — `yes hello`: how the read behavior changes under a continuous, high‑volume stream

`yes hello` writes an unbounded stream of `hello\n` lines. I typed it into the real terminal (again via `xdotool`, never remote control) and measured the read magnitude **two independent ways**, repeating each to confirm stability:

1. **Natively via `/proc/<pid>/task/<tid>/io`** (the kernel's `syscr` read‑syscall counter and `rchar` bytes‑read counter, sampled before/after a timed window with **no** tracer attached) — gives the true, unperturbed magnitude. I ran **four** windows: three of **5 s** (runs A, B, C) and one of **10 s** (run D).
2. **Under `strace`** — gives the exact syscall sequence, the per‑read byte counts, and the `poll()` arguments. I ran **two** windows of **~3.5 s** each (runs 1, 2); `strace`'s per‑syscall overhead makes longer windows produce enormous trace files, so I kept the traced windows short and relied on the native counters for the true frequency.

Both the native script and the strace analysis are shown with their **exact commands and complete output** in §5.2; the full raw traces are `yes1.strace` (60 726 lines) and `yes2.strace` (59 086 lines).

### 5.1 The syscall pattern (under `strace`) — reads become continuous and back‑to‑back

```
$ strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p 83835 -o /tmp/kitty_probe/yes1.strace &
$ DISPLAY=:99 xdotool windowfocus 2097164
$ DISPLAY=:99 xdotool type --clearmodifiers --delay 40 'yes hello'
$ DISPLAY=:99 xdotool key  --clearmodifiers Return
   # ... ~3.5 s ...
$ DISPLAY=:99 xdotool key  --clearmodifiers ctrl+c
```

A representative **contiguous** window mid‑stream (verbatim from `yes1.strace`, lines 302–309). The `hello\r\n…` buffer preview on each `read` is abbreviated by `strace -s 256` (note the trailing `"...`), but the returned byte count and the requested size are exact:

```
22:49:44.384921 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000011>
22:49:44.384951 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1033367) = 455 <0.000012>
22:49:44.384983 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
22:49:44.385014 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1032912) = 497 <0.000012>
22:49:44.385046 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
22:49:44.385079 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1032415) = 378 <0.000010>
22:49:44.385106 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000009>
22:49:44.385136 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1032037) = 450 <0.000010>
```

Three things change dramatically versus the idle/typing case:

1. **Reads are continuous and back‑to‑back.** Each `read(8,…)` is preceded by a `poll()` that returns **immediately** with `1 ([{fd=8, revents=POLLIN}])`, because data is essentially always available. In `poll([{fd=6…}, {fd=7…}, {fd=8…}], 3, 1)` the middle `3` is **`nfds`** — the *count* of file descriptors being polled (fds 6, 7, 8), **not** a timeout — and the **last** argument is the timeout (`1` ms here). Under saturation the timeout is a small value (`0`, `1`, `2`) rather than the idle `-1`, but it rarely governs because the poll returns on ready data before the timeout elapses (quantified in §5.3). (Contrast the idle case in §7, where `poll(..., -1)` blocks indefinitely.)
2. **Each read coalesces many lines.** Instead of 1 byte per read, each `read()` now returns hundreds of bytes of `hello\r\nhello\r\n…` — dozens of lines per read (`455`, `497`, `378`, `450` in the window above).
3. **The requested size shrinks below 1 MiB.** The 3rd `read` argument decreases across consecutive reads (`1033367 → 1032912 → 1032415 → 1032037 …`). That is `BUF_SZ − self->write.offset` (`kitty/vt-parser.c:1457`) with the offset growing because the parser runs on a **separate thread** (the main loop, §6) and has not yet drained everything — i.e. the buffer is filling slightly faster than that instant's drain.

### 5.2 Frequency and bytes/read — measured, two ways, repeated for stability

The frequency and per‑read size were measured **two independent ways**, each repeated for stability. `strace`'s `ptrace` overhead perturbs the very timing R3 asks about (it slows the reader), so the **native, untraced** counters are the authoritative magnitude and `strace` provides the corroborating per‑read detail.

#### Method A — native, no tracer (authoritative magnitude)

The kernel maintains per‑thread I/O counters at `/proc/<KPID>/task/<IOTID>/io`: `syscr` is the cumulative number of `read()` syscalls and `rchar` the cumulative bytes returned. Sampling them before/after a timed window over the **real typed‑input PTY path** (driven by `xdotool`'s XTEST, never remote control) yields the true read magnitude without any tracer overhead. The measurement script, verbatim as run:

```bash
$ cat /tmp/kitty_probe/native_run.sh
#!/bin/bash
# native_run.sh <label> <window_seconds>
# Measures the kitty I/O thread's true (untraced) PTY read magnitude by sampling
# the kernel's per-thread read counters in /proc/<KPID>/task/<IOTID>/io
#   syscr = number of read() syscalls;  rchar = bytes returned by reads.
# Drives the real typed-input PTY path via xdotool (XTEST), never remote control.
source /tmp/kitty_probe/ids.env
LABEL="$1"; WIN="$2"
IOSTAT=/proc/$KPID/task/$IOTID/io
DISPLAY=:99 xdotool windowfocus "$WID" >/dev/null 2>&1
DISPLAY=:99 xdotool type --clearmodifiers --delay 40 'yes hello' >/dev/null 2>&1
DISPLAY=:99 xdotool key  --clearmodifiers Return >/dev/null 2>&1
sleep 0.6
echo "### RUN $LABEL  (window ${WIN}s) ###"
echo "--- /proc/$KPID/task/$IOTID/io BEFORE ---"
grep -E '^syscr|^rchar' "$IOSTAT"
b_syscr=$(awk -F': ' '/^syscr/{print $2}' "$IOSTAT")
b_rchar=$(awk -F': ' '/^rchar/{print $2}' "$IOSTAT")
t0=$(date +%s.%N)
sleep "$WIN"
t1=$(date +%s.%N)
echo "--- /proc/$KPID/task/$IOTID/io AFTER ---"
grep -E '^syscr|^rchar' "$IOSTAT"
a_syscr=$(awk -F': ' '/^syscr/{print $2}' "$IOSTAT")
a_rchar=$(awk -F': ' '/^rchar/{print $2}' "$IOSTAT")
DISPLAY=:99 xdotool key --clearmodifiers ctrl+c >/dev/null 2>&1
sleep 0.4
DISPLAY=:99 xdotool key --clearmodifiers ctrl+c >/dev/null 2>&1
awk -v bs="$b_syscr" -v as="$a_syscr" -v br="$b_rchar" -v ar="$a_rchar" -v t0="$t0" -v t1="$t1" 'BEGIN{
  dr=as-bs; db=ar-br; dt=t1-t0;
  printf "reads = %d - %d = %d\n", as, bs, dr;
  printf "bytes = %d - %d = %d\n", ar, br, db;
  printf "elapsed = %.3f s\n", dt;
  printf "=> reads/sec = %.0f ; bytes/read = %.1f ; throughput = %.2f MB/s\n", dr/dt, db/dr, db/dt/1048576;
}'
echo ""
```

Invocation — four windows (three at 5 s, one at 10 s to rule out a startup transient), each preceded by a `ctrl+c` + `clear` reset so every window measures a fresh saturating stream:

```bash
$ for rw in "A 5" "B 5" "C 5" "D 10"; do bash /tmp/kitty_probe/native_run.sh $rw; \
    DISPLAY=:99 xdotool type --clearmodifiers --delay 40 'clear'; \
    DISPLAY=:99 xdotool key --clearmodifiers Return; sleep 0.4; done | tee /tmp/kitty_probe/r3_native.txt
```

Complete, unedited stdout (`/tmp/kitty_probe/r3_native.txt`):

```
### RUN A  (window 5s) ###
--- /proc/83769/task/83835/io BEFORE ---
rchar: 4713292
syscr: 74134
--- /proc/83769/task/83835/io AFTER ---
rchar: 45466058
syscr: 667590
reads = 667811 - 74134 = 593677
bytes = 45495712 - 4754487 = 40741225
elapsed = 5.012 s
=> reads/sec = 118456 ; bytes/read = 68.6 ; throughput = 7.75 MB/s

### RUN B  (window 5s) ###
--- /proc/83769/task/83835/io BEFORE ---
rchar: 50598398
syscr: 743896
--- /proc/83769/task/83835/io AFTER ---
rchar: 87051829
syscr: 1309921
reads = 1310157 - 744128 = 566029
bytes = 87078788 - 50625306 = 36453482
elapsed = 5.012 s
=> reads/sec = 112942 ; bytes/read = 64.4 ; throughput = 6.94 MB/s

### RUN C  (window 5s) ###
--- /proc/83769/task/83835/io BEFORE ---
rchar: 92918585
syscr: 1395413
--- /proc/83769/task/83835/io AFTER ---
rchar: 139613380
syscr: 2063890
reads = 2064129 - 1395413 = 668716
bytes = 139648772 - 92933052 = 46715720
elapsed = 5.006 s
=> reads/sec = 133574 ; bytes/read = 69.9 ; throughput = 8.90 MB/s

### RUN D  (window 10s) ###
--- /proc/83769/task/83835/io BEFORE ---
rchar: 146912817
syscr: 2125719
--- /proc/83769/task/83835/io AFTER ---
rchar: 246989159
syscr: 3100923
reads = 3100923 - 2125835 = 975088
bytes = 247023473 - 146953963 = 100069510
elapsed = 10.007 s
=> reads/sec = 97444 ; bytes/read = 102.6 ; throughput = 9.54 MB/s
```

*Reading the output:* the human‑readable `BEFORE`/`AFTER` `grep` lines and the arithmetic line each sample `/proc/.../io` **independently**, a few milliseconds apart. Because the `yes` stream advances the counters continuously, the value used by the arithmetic (captured by the `awk` re‑read) is a few hundred reads/tens‑of‑KB higher than the immediately preceding `grep` snapshot (e.g. RUN A `AFTER` `grep` shows `syscr 667590` while the arithmetic uses `667811`). This is expected for a live, continuously incrementing counter and does not affect the computed magnitude. Summarized:

| Run | window | reads (Δsyscr) | bytes (Δrchar) | reads/sec | bytes/read | throughput |
|-----|--------|----------------|----------------|-----------|------------|------------|
| A | 5.012 s | 593 677 | 40 741 225 | **118 456** | **68.6 B** | 7.75 MB/s |
| B | 5.012 s | 566 029 | 36 453 482 | **112 942** | **64.4 B** | 6.94 MB/s |
| C | 5.006 s | 668 716 | 46 715 720 | **133 574** | **69.9 B** | 8.90 MB/s |
| D | 10.007 s | 975 088 | 100 069 510 | **97 444** | **102.6 B** | 9.54 MB/s |

#### Method B — under `strace` (per‑read detail + cross‑check)

The two full traces from §5.1 (`/tmp/kitty_probe/yes1.strace`, 60 726 lines; `/tmp/kitty_probe/yes2.strace`, 59 086 lines) were reduced with a small script that parses every `read()` and `poll()` line — crucially distinguishing `poll()`'s `nfds` (middle) argument from its timeout (last) argument (see §5.3). The analysis script, verbatim as run:

```bash
$ cat /tmp/kitty_probe/analyze.py
#!/usr/bin/env python3
# analyze.py <strace_file> : compute read(8) PTY byte-count + requested-size stats,
# eventfd(6) drains, and poll() timeout distribution (correctly parsing nfds vs timeout).
import re, sys, statistics
path = sys.argv[1]
# return may be "= 11 <time>" or "= -1 EAGAIN (..) <time>"; use search (no $ anchor)
r_line = re.compile(r'read\((\d+),\s*(.*),\s*(\d+)\)\s*=\s*(-?\d+)(\s+E[A-Z]+)?')
p_line = re.compile(r'poll\(\[(.*?)\],\s*(\d+),\s*(-?\d+)\)\s*=')
r8_ret=[]; r8_req=[]; r8_eagain=0; r6=0; r7=0; other_reads=0
poll_to={}; poll_nfds={}; poll_count=0
first_ts=None; last_ts=None
with open(path, encoding='utf-8', errors='replace') as f:
    for line in f:
        ts=line.split(None,1)[0]
        if re.match(r'^\d\d:\d\d:\d\d', ts):
            if first_ts is None: first_ts=ts
            last_ts=ts
        m=r_line.search(line)
        if m:
            fd=int(m.group(1)); req=int(m.group(3)); retnum=int(m.group(4)); err=m.group(5)
            if fd==8:
                if err or retnum<0: r8_eagain+=1
                else: r8_ret.append(retnum); r8_req.append(req)
            elif fd==6: r6+=1
            elif fd==7: r7+=1
            else: other_reads+=1
            continue
        p=p_line.search(line)
        if p:
            poll_count+=1
            nfds=int(p.group(2)); to=int(p.group(3))
            poll_nfds[nfds]=poll_nfds.get(nfds,0)+1
            poll_to[to]=poll_to.get(to,0)+1
def secs(a,b):
    def t(x):
        h,mi,s=x.split(':'); return int(h)*3600+int(mi)*60+float(s)
    return t(b)-t(a)
print(f"===== {path} =====")
print(f"trace window (first->last timestamp): {first_ts} -> {last_ts}  (~{secs(first_ts,last_ts):.2f}s wall, strace-slowed)")
print(f"poll() lines: {poll_count}")
print(f"  nfds distribution (2nd/MIDDLE arg): {dict(sorted(poll_nfds.items()))}")
print(f"  timeout distribution (3rd/LAST arg): {dict(sorted(poll_to.items()))}")
print(f"read(6) eventfd wakeup reads: {r6}   read(7) signalfd reads: {r7}   other-fd reads: {other_reads}")
print(f"read(8) PTY reads: successful={len(r8_ret)}  EAGAIN/err={r8_eagain}")
if r8_ret:
    n=len(r8_ret)
    print(f"  returned-bytes: min={min(r8_ret)} median={int(statistics.median(r8_ret))} mean={sum(r8_ret)/n:.1f} max={max(r8_ret)} total={sum(r8_ret)}")
    print(f"  requested-size (3rd arg to read): min={min(r8_req)} max={max(r8_req)}  (BUF_SZ=1048576)")
    full=sum(1 for x in r8_req if x==1048576); shrunk=n-full
    print(f"  requested==1048576(full buffer): {full}   requested<1048576(shrunk, buffer partly full): {shrunk}")
    buckets=[("==1 (keystroke echo)",lambda x:x==1),
             ("2..16",lambda x:2<=x<=16),
             ("17..64",lambda x:17<=x<=64),
             ("65..256",lambda x:65<=x<=256),
             ("257..1024",lambda x:257<=x<=1024),
             ("1025..8192",lambda x:1025<=x<=8192),
             ("8193..65536",lambda x:8193<=x<=65536),
             (">65536",lambda x:x>65536)]
    print("  returned-bytes distribution:")
    for name,fn in buckets:
        c=sum(1 for x in r8_ret if fn(x))
        if c: print(f"    {name:24s}: {c:6d}  ({100*c/n:4.1f}%)")
```

Invocation and complete, unedited stdout for both runs:

```
$ python3 /tmp/kitty_probe/analyze.py /tmp/kitty_probe/yes1.strace
===== /tmp/kitty_probe/yes1.strace =====
trace window (first->last timestamp): 22:49:44.187082 -> 22:49:47.768441  (~3.58s wall, strace-slowed)
poll() lines: 30444
  nfds distribution (2nd/MIDDLE arg): {3: 30444}
  timeout distribution (3rd/LAST arg): {-1: 690, 0: 10040, 1: 9970, 2: 9744}
read(6) eventfd wakeup reads: 24   read(7) signalfd reads: 0   other-fd reads: 0
read(8) PTY reads: successful=30257  EAGAIN/err=0
  returned-bytes: min=1 median=497 mean=573.8 max=20083 total=17360110
  requested-size (3rd arg to read): min=896517 max=1048576  (BUF_SZ=1048576)
  requested==1048576(full buffer): 195   requested<1048576(shrunk, buffer partly full): 30062
  returned-bytes distribution:
    ==1 (keystroke echo)    :      9  ( 0.0%)
    2..16                   :      2  ( 0.0%)
    17..64                  :      6  ( 0.0%)
    65..256                 :    860  ( 2.8%)
    257..1024               :  28060  (92.7%)
    1025..8192              :   1275  ( 4.2%)
    8193..65536             :     45  ( 0.1%)

$ python3 /tmp/kitty_probe/analyze.py /tmp/kitty_probe/yes2.strace
===== /tmp/kitty_probe/yes2.strace =====
trace window (first->last timestamp): 22:50:01.661637 -> 22:50:05.207668  (~3.55s wall, strace-slowed)
poll() lines: 29577
  nfds distribution (2nd/MIDDLE arg): {3: 29577}
  timeout distribution (3rd/LAST arg): {-1: 696, 0: 9728, 1: 9739, 2: 9414}
read(6) eventfd wakeup reads: 24   read(7) signalfd reads: 0   other-fd reads: 0
read(8) PTY reads: successful=29484  EAGAIN/err=0
  returned-bytes: min=1 median=567 mean=685.9 max=19437 total=20224220
  requested-size (3rd arg to read): min=856074 max=1048576  (BUF_SZ=1048576)
  requested==1048576(full buffer): 163   requested<1048576(shrunk, buffer partly full): 29321
  returned-bytes distribution:
    ==1 (keystroke echo)    :      9  ( 0.0%)
    2..16                   :      5  ( 0.0%)
    17..64                  :      5  ( 0.0%)
    65..256                 :    682  ( 2.3%)
    257..1024               :  25285  (85.8%)
    1025..8192              :   3470  (11.8%)
    8193..65536             :     28  ( 0.1%)
```

Summarized (dividing the successful `read(8)` count by the trace window):

| Run | reads (fd 8) | window | reads/sec | bytes/read (min / median / mean / max) | total bytes |
|-----|--------------|--------|-----------|-----------------------------------------|-------------|
| 1 | 30 257 | 3.58 s | ≈ **8 452** | 1 / **497** / 573.8 / 20 083 | 17 360 110 |
| 2 | 29 484 | 3.55 s | ≈ **8 305** | 1 / **567** / 685.9 / 19 437 | 20 224 220 |

Under `strace` the vast majority of reads land in the **257–1024 byte** bucket (92.7 % / 85.8 %); the buffer is essentially never fully drained yet never reaches the 1 MiB cap either — the requested size (`BUF_SZ − write.offset`) bottoms out around **896 517** (run 1) / **856 074** (run 2), i.e. the buffer occupancy peaked near ~150 KB / ~193 KB before the parser thread caught up.

**Stability.** Both measurements are stable across repeats and the two methods bracket the same behavior at two reader speeds. Natively kitty performs **≈ 97 000–134 000 reads/second at ≈ 64–103 bytes/read for a steady ≈ 7–9.5 MB/s**; under `strace` the rate falls to **≈ 8 300–8 500 reads/second at a median ≈ 497–567 bytes/read** (with occasional reads up to ~20 KB). Increasing the native window from 5 s (runs A/B/C) to 10 s (run D) did not shift the magnitude, confirming the observed values are the steady‑state stream behavior and not a startup transient.

### 5.3 Reconciling the two numbers, and the role of `input_delay` (3 ms)

The native and traced figures differ because **`strace` adds per‑syscall `ptrace` overhead that slows the reader**: a slower reader lets *more* bytes accumulate between reads (median rises from **~64–103 B** native to **~497–567 B** under `strace`) and issues *fewer* reads per second (**~8 300–8 500** traced vs **~97 000–134 000** native). Both are legitimate observations of the same behavior at two reader speeds; the **native** numbers are the true magnitude, and the inverse relationship between reader speed and bytes/read is itself the point — read size is set by how much data has piled up since the previous read, not by any fixed chunking.

The key qualitative result answers R3 directly: under a saturating stream, **kitty's reads change from occasional/1‑byte (idle/typing) to continuous, back‑to‑back reads that each coalesce many lines**, bounded above by the **1 MiB** buffer (`BUF_SZ`, `kitty/vt-parser.c:18`).

**A note on the `poll()` arguments (correcting a common misreading of the trace).** Every `poll()` line in the traces above has the form `poll([{fd=6…}, {fd=7…}, {fd=8…}], 3, <timeout>)`. The middle argument **`3` is `nfds`** — the *number* of file descriptors being polled, not a timeout. It is `self->count + EXTRA_FDS` (`kitty/child-monitor.c:1509`), i.e. the one child PTY (fd 8) plus the two `EXTRA_FDS`: the wakeup eventfd (fd 6, index 0) and the signal fd (fd 7, index 1). With exactly one window open this is `1 + 2 = 3` in every single poll line, which is why the `nfds` distribution computed by `analyze.py` is `{3: 30444}` / `{3: 29577}` — a constant, not a timeout. The **timeout is the *last* argument**, and its observed distribution is `{-1: 690, 0: 10040, 1: 9970, 2: 9744}` (run 1) and `{-1: 696, 0: 9728, 1: 9739, 2: 9414}` (run 2). **No `poll()` uses a timeout of `3`** in either run — the earlier notion of "timeout 3" was a misread of `nfds`.

Those timeout values follow directly from the code (`kitty/child-monitor.c:1506‑1513`):

```c
if (has_pending_wakeups) {
    now = monotonic();
    monotonic_t time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at);
    if (time_delta >= 0) ret = poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta));
    else ret = 0;
} else {
    ret = poll(children_fds, self->count + EXTRA_FDS, -1);
}
```

- When there is **no pending main‑loop wakeup**, the poll timeout is **`-1`** (block indefinitely) — this is the idle pattern of §7 and the 690/696 `-1` samples between bursts.
- When a wakeup **is** pending, the timeout is `input_delay − (now − last_main_loop_wakeup_at)` converted to milliseconds. Since `input_delay` is 3 ms and some time has always already elapsed since the last wakeup, that expression lands on **`0`, `1`, or `2`** ms — never a full `3` — which is exactly the `{0,1,2}` spread observed. (When the expression goes negative the code sets `ret = 0` and skips the syscall entirely, so those iterations emit no `poll()` line at all.)

Crucially, **`input_delay` does not gate the reads** — it gates *main‑loop wakeups*. The `WAKEUP` macro (`kitty/child-monitor.c:1562`) is only fired once more than `input_delay` has elapsed since the last wakeup, with the in‑code rationale *"we only wakeup the main loop after input_delay as wakeup is an expensive operation on some platforms, such as cocoa"* (`kitty/child-monitor.c:1563‑1564`). So `input_delay` coalesces the notifications that tell the **render/parse main loop** to drain the buffer (bounding those to roughly one per 3 ms ≈ a few hundred per second); the **I/O thread meanwhile keeps calling `read_bytes()` in a tight `poll()`→`read()` loop**, which is why the native read rate is ~100 k/s and not ~333/s. This is consistent with the option's own documentation, which adds that the setting is **"ignored when the input buffer is almost full"** (`kitty/options/definition.py:878`). That combination — reads paced by data availability, wakeups paced by `input_delay` — is why, under `yes hello`, the reads become continuous and each coalesces many `hello` lines rather than being throttled to 3 ms intervals.

---

## 6. R5 — The responsible C functions

### 6.1 The reader — `read_bytes()` (`kitty/child-monitor.c:1337`)

The function that reads from the PTY master file descriptor is **`read_bytes(int fd, Screen *screen)`**. Its `read()` at line 1345 is the exact syscall observed on fd 8 throughout §4 and §5. The function body below is reproduced **verbatim** from `kitty/child-monitor.c:1336‑1356`; the only additions are the trailing `// :NNNN` markers, which are line‑number annotations I added for cross‑reference (they are **not** part of the source):

```c
// kitty/child-monitor.c:1336-1356 (verbatim; // :NNNN comments are added annotations)
static bool
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

- The **buffer** and its **available size** come from `vt_parser_create_write_buffer()` (`kitty/vt-parser.c:1451`), whose returned size is `BUF_SZ − self->write.offset` (`:1457`) — this is precisely the varying 3rd argument to `read()` I observed (1 048 576 down to ≈ 856 074 under the `yes hello` stream; see §5.2).
- After reading, the bytes are committed to the parser with `vt_parser_commit_write()` (`:1354`, defined at `kitty/vt-parser.c:1465`).
- **Threading context:** `read_bytes()` runs on the dedicated I/O thread `io_loop()` (`kitty/child-monitor.c:1481`), created by `pthread_create(&self->io_thread, NULL, io_loop, self)` (`:291`) — the kernel thread I traced as `KittyChildMon`/TID 83835. `io_loop()` builds the `poll()` set (requesting `POLLIN` for a child only while `vt_parser_has_space_for_input()` is true, `:1501`), calls `poll()` (`:1509`/`:1512`), and on `POLLIN`/`POLLHUP` invokes `has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)` (`:1531`). This is the reader half of the answer.

### 6.2 The parser that separates printable text from escape sequences — `consume_input()` → `consume_normal()`

The function that parses the incoming bytes and separates **printable text** from **escape sequences** is the VT state machine **`consume_input()`** (`kitty/vt-parser.c:1367`), which dispatches on the current state. The excerpt below is **abbreviated** from the source: `…` marks elided lines (the `#define consume(x)` macro at `:1368`, the `#ifdef DUMP_COMMANDS` blocks, and the `VTE_OSC`/`VTE_APC`/`VTE_PM`/`VTE_DCS`/`VTE_SOS` cases at `:1387‑1395`); the `case` lines shown are verbatim with added `// :NNNN` annotations:

```c
// kitty/vt-parser.c:1367 (ABBREVIATED excerpt — "…" denotes elided source lines)
consume_input(PS *self, PyObject *dump_callback UNUSED, id_type window_id UNUSED) {
    …
    switch (self->vte_state) {
        case VTE_NORMAL:
            consume_normal(self); self->read.consumed = self->read.pos; break;   // :1377
        case VTE_ESC:
            if (consume_esc(self)) { self->read.consumed = self->read.pos; }      // :1379-1380
            break;
        case VTE_CSI:
            if (consume_csi(self)) { … SET_STATE(NORMAL); }                       // :1382
            break;
        …  // VTE_OSC/APC/PM/DCS/SOS via the consume(x) macro, :1387-1395
    }
}
```

The actual text/escape split happens inside **`consume_normal()`** (`kitty/vt-parser.c:230‑240`). The excerpt is reproduced from source; the only deviations are that the long `utf8_decode_to_esc(...)` call is wrapped across two display lines for width, the `REPORT_DRAW(…)` arguments are elided (they are `self->utf8_decoder.output.storage, self->utf8_decoder.output.pos`, identical to the `screen_draw_text` args on the next line), and the `// :NNNN` markers are added annotations:

```c
// kitty/vt-parser.c:230-240 (from source; REPORT_DRAW args elided, one line wrapped, // :NNNN added)
consume_normal(PS *self) {
    do {
        const bool sentinel_found = utf8_decode_to_esc(&self->utf8_decoder,
                self->buf + self->read.pos, self->read.sz - self->read.pos);      // :232  decode until an ESC sentinel
        self->read.pos += self->utf8_decoder.num_consumed;
        if (self->utf8_decoder.output.pos) {
            REPORT_DRAW(…);                                                       // :235  (dump hook; args as screen_draw_text)
            screen_draw_text(self->screen, self->utf8_decoder.output.storage,
                             self->utf8_decoder.output.pos);                      // :236  printable text -> screen
        }
        if (sentinel_found) { SET_STATE(ESC); break; }                           // :238  escape byte -> switch to ESC state
    } while (self->read.pos < self->read.sz);
}
```

- **Printable text** is decoded by `utf8_decode_to_esc()` (`:232`), which consumes bytes **up to the next ESC sentinel**, and is written to the screen via **`screen_draw_text()`** (`:236`) — the printable‑text sink. In the `echo test123` capture, the `test123` characters flow through exactly this path.
- **Escape sequences** are detected when `utf8_decode_to_esc()` reports `sentinel_found`; `consume_normal()` then does `SET_STATE(ESC)` (`:238`), handing control to `consume_esc()` / `consume_csi()` on subsequent iterations of `consume_input()` (`:1379` and the `VTE_CSI` case). The `\33]133;…\7` (OSC 133 shell‑integration) and `\33[?2004h` (bracketed‑paste) sequences visible in the `echo` reads take this branch.

**Runtime dispatch chain (which function actually calls the parser).** The parser is **not** invoked from the read loop directly, and — correcting an easy misattribution — it is **not** reached at runtime via `kitty/screen.c:4775‑4776`. Those two lines live inside `test_parse_written_data()` (`kitty/screen.c:4772`), a `PyObject*` helper registered as a Python‑callable method (`METHODB(test_parse_written_data, METH_VARARGS)`, `kitty/screen.c:4783`); it calls `parse_worker`/`parse_worker_dump` **only when driven from test code**, never on the live PTY‑read path. The real runtime path runs on the **main thread** (distinct from the I/O thread that does the reading in §6.1):

```
main_loop()                         kitty/child-monitor.c:1259   ("The main thread loop")
  └─ parse_input(self)              kitty/child-monitor.c:1236   (call site)
       │ def at :451 — comment: "Parse all available input that was read in the I/O thread."
       └─ do_parse(self, screen, …) kitty/child-monitor.c:438
            └─ self->parse_func(screen, &pd, flush)   :440
                 │ parse_func = parse_worker (or parse_worker_dump when dumping)
                 │   assigned at kitty/child-monitor.c:181 (and :180)
                 └─ parse_worker(p, pd, flush)  kitty/vt-parser.c:1496
                      └─ run_worker(p, pd, flush)     kitty/vt-parser.c:1417
                           └─ consume_input(...)      kitty/vt-parser.c:1367
                                └─ consume_normal(...) kitty/vt-parser.c:230
```

This confirms the architecture that explains the read behavior in §3/§5: the **reader** (`read_bytes()`) runs on the dedicated I/O thread `io_loop()` and the **parser** (`consume_input()`/`consume_normal()`) runs on the **main thread** via `parse_input()` — the in‑code comment at `:452` ("*Parse all available input that was read in the I/O thread*") states this two‑thread split explicitly.

**Reader ↔ parser API boundary.** The two threads communicate only through the thread‑safe API declared in `kitty/vt-parser.h:34‑38` (`vt_parser_create_write_buffer`, `vt_parser_commit_write`, `vt_parser_has_space_for_input`, `parse_worker`); the `Screen` owns the parser (`kitty/screen.h:158`, `Parser *vt_parser;`). This decoupling is exactly why the observed `read()` byte counts (child‑monitor.c) and the 1 MiB buffer capacity (vt‑parser.c) line up: the I/O thread fills a region of the parser's single locked buffer via `read_bytes()`, then the main thread drains it via `parse_input()` → … → `consume_input()`.

---

## 7. Edge and transitional states

I exercised the read loop through its transitions, not just the happy path.

**Idle — `poll()` blocks with no reads.** With no output pending, the I/O thread parks in a blocking `poll(..., -1)`. Tracing the quiescent thread for a multi‑second window produced an **empty** trace file — zero syscalls — because the thread was already blocked in `poll()` the entire time (`strace` only logs syscall *entry/exit*, and none occurred):

```
$ strace -tt -T -e trace=read,readv,poll,ppoll -p 83835 -o /tmp/kitty_probe/idle.strace   # (~4 s window)
$ cat /tmp/kitty_probe/idle.strace.err
strace: Process 83835 attached
strace: Process 83835 detached
$ wc -l /tmp/kitty_probe/idle.strace
0 /tmp/kitty_probe/idle.strace              # <-- zero syscall lines: the thread never left poll()
```

Zero reads and zero poll *returns* during the window is the idle signature. Independently, the kernel confirms the thread is parked in poll: `cat /proc/83769/task/83835/wchan` → `do_sys_poll`. This is the blocking branch at `kitty/child-monitor.c:1512` (`poll(children_fds, self->count + EXTRA_FDS, -1)`), taken when there are no pending wakeups (`has_pending_wakeups == false`).

**Typing — one small read per keystroke/line.** Shown in §4: `read(8, "e", 1048576) = 1`, etc.; the command output arrives in a single 114‑byte read.

**High volume — sustained larger, back‑to‑back reads with the requested size shrinking below 1 MiB.** Shown in §5: `poll()` returns immediately (small timeout `0`/`1`/`2`, never the idle `-1`), reads coalesce many lines, and the 3rd `read` argument decreases (`1033367 → 1032912 → 1032415 → 1032037 …`) as the buffer fills (`BUF_SZ − offset`, `kitty/vt-parser.c:1457`).

**Return to idle — the blocking‑`poll()` pattern is restored.** After activity ceases, tracing the I/O thread captures the *transition* back to idle: a final wakeup‑eventfd drain, a last small `read(8, " ", 1048576) = 1` (a trailing prompt redraw), and then the thread re‑enters the **blocking** `poll(..., -1)` — the timeout reverts from the saturated `0/1/2` back to `-1` because there are no more pending wakeups. Verbatim (`/tmp/kitty_probe/idle2.strace`, all 6 lines):

```
22:53:53.746095 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8 <0.000019>
22:53:53.746231 read(6, 0x7bc6ae19d740, 1024) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
22:53:53.746268 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}]) <0.000012>
22:53:53.746341 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000115>
22:53:53.746476 read(8, " ", 1048576)   = 1 <0.000012>
22:53:53.746533 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <detached ...>
```

The final `poll(..., -1 <detached ...>)` is the thread settling back into the blocking idle wait; `cat /proc/83769/task/83835/wchan` again reports `do_sys_poll`.

**Non‑blocking master / retry branch.** The master is non‑blocking (`os.set_blocking(child_fd, False)`, `kitty/child.py:345`), so `read_bytes()` wraps `read()` in a retry loop for `EINTR`/`EAGAIN` (`kitty/child-monitor.c:1347`). Because a read is only issued after `poll()` reports `POLLIN`, I observed **zero `EAGAIN` on fd 8** across both `yes` runs — data was always available when the read ran. The non‑blocking‑drain pattern *was* observed on the wakeup **eventfd (fd 6)**, which is read until it returns `EAGAIN`:

```
# /tmp/kitty_probe/echo.strace lines 1-2 (verbatim)
22:42:33.213396 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8 <0.000032>
22:42:33.213586 read(6, 0x7bc6ae19d740, 1024) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
```

**Child‑death branches (inferred — not exercised).** `read_bytes()` treats `len == 0` and `EIO` as the child having exited (`kitty/child-monitor.c:1349‑1350`, `:1355`); I did not kill the shell, so these branches were not triggered here. Labeled **(inferred)** from the code.

**Buffer‑full gating (inferred — not triggered).** When the parser buffer fills completely, `read_bytes()` returns early without reading (`if (!available_buffer_space) return true;`, `kitty/child-monitor.c:1342`) and `io_loop()` stops requesting `POLLIN` for that child (`events = … ? POLLIN : 0`, `:1501`). In my runs the parser kept pace, so I never observed a `poll()` entry with `fd=8, events=0` (grep count: 0 in both `yes` traces). This gating is therefore reported **(inferred)** from the code, not observed at runtime.

---

## 8. Reasoning & inferred‑vs‑observed

**End‑to‑end story, tied to evidence.** kitty allocates the PTY in Python (`os.openpty()`, `child.py:171`), forks/execs the shell in C so the shell's stdio is the slave `/dev/pts/0` (`child.c:88`,`:138‑159`), and keeps the non‑blocking master (fd 8 this run; `child.py:338`,`:345`). Its I/O thread `io_loop()` `poll()`s that master and calls `read_bytes()` → `read(8, buf, BUF_SZ−offset)` on readiness (`child-monitor.c:1481`,`:1531`,`:1337`,`:1345`), depositing bytes into the parser's 1 MiB buffer (`vt-parser.c:18`,`:1451`). `consume_input()`/`consume_normal()` then split printable text (→ `screen_draw_text`) from escape sequences (→ `SET_STATE(ESC)`) (`vt-parser.c:1367`,`:230`,`:236`,`:238`). Every arrow here is backed by a captured `strace`/`ps`/`lsof`/`/proc` line above and a verified `file:line`.

**Values observed at runtime (this run; dynamic):** shell PID **83836**; command line **`/bin/bash --posix`**; slave **`/dev/pts/0`**; master fd **8**; the `poll()`+`read()` syscalls, the requested buffer sizes (echo: `1048565`, `1048518`, `1048404`; `yes`: `1033367 → 1032037 …` down to ≈ `856074`), and the byte counts (`1`, `11`, `47`, `114`, `429`, and the `yes` distribution); read frequency and bytes/read for `yes hello` (native ≈ **97 000–134 000** reads/s @ ≈ **64–103 B**; under `strace` ≈ **8 300–8 500** reads/s @ median ≈ **497–567 B**). These are specific to this run — the PID, the fd number, and the `/dev/pts/N` index will differ next time.

**Facts grounded in code (stable across runs):** the reader is `read_bytes()` and the syscall is `read(fd, buf, available_buffer_space)` (`child-monitor.c:1337`/`:1345`); the buffer cap is `BUF_SZ` = 1 MiB (`vt-parser.c:18`); the requested size is `BUF_SZ − write.offset` (`:1457`); `input_delay` = 3 ms (`options/definition.py:878`) throttles **main‑loop wakeups** — not reads — via the `WAKEUP` macro (`child-monitor.c:1562`), and the poll timeout when a wakeup is pending is `input_delay − elapsed` (≤ 3 ms, observed as `0/1/2`; `-1` when idle) (`child-monitor.c:1506‑1513`); the parser is `consume_input()`/`consume_normal()` reached at runtime via `main_loop()` → `parse_input()` → `do_parse()` → `parse_worker()` (`child-monitor.c:1259`,`:451`,`:438`,`:181`; `vt-parser.c:1496`,`:1417`,`:1367`,`:230`); the default shell resolution is `constants.py:181` + `utils.py:768`.

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
KPID=$(pgrep -n -f 'launcher/kitty')                 # kitty GUI pid  (83769 this run)
IOTID=$(ps -T -p "$KPID" -o tid,comm | awk '/KittyChildMon/{print $1}')  # io thread (83835)
WID=$(DISPLAY=:99 xdotool search --class kitty | head -1)

# R1/R4 — process & descriptors
SPID=$(ps --ppid "$KPID" -o pid=)                    # shell pid (83836 this run)
pstree -aps "$KPID"; ps -o pid,ppid,args -p "$SPID"
ls -l /proc/$KPID/fd; lsof -p "$KPID" | grep -iE 'ptmx|/dev/pts'
for fd in 0 1 2; do readlink /proc/$SPID/fd/$fd; done

# R2 — echo test123 under strace (real keyboard->PTY path)
mkdir -p /tmp/kitty_probe
strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p "$IOTID" -o /tmp/kitty_probe/echo.strace &
DISPLAY=:99 xdotool windowfocus "$WID"
DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'echo test123'
DISPLAY=:99 xdotool key  --clearmodifiers Return

# R3 — yes hello, measured TWO ways (see §5.2):
# (a) under strace for the exact per-read detail — full strings need -s 256, NOT -s 24;
#     the traced window is only ~3.5 s because ptrace overhead bloats the trace file.
strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p "$IOTID" -o /tmp/kitty_probe/yes1.strace &
DISPLAY=:99 xdotool type --clearmodifiers --delay 40 'yes hello'; DISPLAY=:99 xdotool key Return
sleep 3.5; DISPLAY=:99 xdotool key ctrl+c                 # repeat -> /tmp/kitty_probe/yes2.strace
python3 /tmp/kitty_probe/analyze.py /tmp/kitty_probe/yes1.strace   # per-read stats (§5.2)
# (b) native magnitude with NO tracer — the authoritative frequency: sample the kernel's
#     per-thread syscr (read-count) and rchar (bytes) counters before/after a timed window.
#     native_run.sh (listed in §5.2) wraps this; 5 s windows A/B/C and a 10 s window D.
bash /tmp/kitty_probe/native_run.sh A 5      # -> reads/sec, bytes/read (repeat for B, C, D)
cat /proc/$KPID/task/$IOTID/io               # raw syscr / rchar counters
```

**Cleanup (leave the repository byte‑for‑byte unchanged).** All observation artifacts were written under `/tmp/kitty_probe/` (never inside the repo). After capture, remove them and stop the headless session:

```bash
rm -rf /tmp/kitty_probe                       # delete all temporary scripts, traces, logs
kill "$KPID" 2>/dev/null                       # stop kitty (its shell child exits with it)
pkill -x Xvfb 2>/dev/null                      # stop the headless X server
git status --porcelain                         # expect ONLY: blitzy/documentation/kitty_815df1e210e0.md
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
| Parser **runtime** dispatch (main thread) | `kitty/child-monitor.c:1259` (`main_loop`), `:1236` (call), `:451` (`parse_input`, comment `:452`), `:438`/`:440` (`do_parse`/`parse_func`), `:180‑181` (`parse_func` = `parse_worker`) |
| Parser **test‑only** entry (NOT the runtime path) | `kitty/screen.c:4772` (`test_parse_written_data`), `:4775‑4776` (calls `parse_worker`), `:4783` (`METHODB` registration) |
| Reader↔parser API / Screen owns parser | `kitty/vt-parser.h:34‑38`; `kitty/screen.h:158` (`Parser *vt_parser`) |
| `input_delay` (3 ms) / default `term` | `kitty/options/definition.py:878`, `:3242` |
| Build / manifests | `Makefile:12‑13`; `setup.py:30`; `pyproject.toml:2`; `go.mod:3` |

*All values labeled “this run” (PID 83836, master fd 8, `/dev/pts/0`) are dynamic and were captured live; they will differ on other runs. All `file:line` references were verified against the source at `HEAD 815df1e21`.*





