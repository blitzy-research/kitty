# kitty — how the terminal emulator reads from its shell over the PTY

This document answers six questions about how the [kitty](https://github.com/kovidgoyal/kitty)
terminal emulator communicates with the shell it spawns, over the pseudo-terminal (PTY).
It is a **Run-First** investigation: kitty was built from source and launched, its real
PTY read pipeline was exercised with the exact inputs `echo test123` and `yes hello`, and
every behavioural claim is backed by the actual command that produced it and that command's
unedited output. Statements that are grounded in reading the C/Python source rather than in a
runtime capture are explicitly labelled **(inferred from code)**.

All `file:line` references are anchored to the source baseline commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

## Investigation baseline and environment

* **Project:** kitty terminal emulator (`kovidgoyal/kitty`).
* **Source baseline:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. The repository is read-only for
  this task: the *only* file added is this document, under `blitzy/documentation/`. No source
  file is modified (verified in Appendix D).
* **Built version:** `kitty 0.35.2`.
* **Host:** Ubuntu 25.10 container; CPython 3.13.7 (satisfies `requires-python >=3.8` in
  `pyproject.toml:L2`); Go 1.24.4 (satisfies `go 1.22` in `go.mod:L3`); gcc 15.2.0.
* **Display:** headless `Xvfb :99` with Mesa software GL (`llvmpipe`), because the container has
  no physical display. kitty itself, its build, and its PTY handling are unaffected by using a
  software GL backend.

### Session identifiers used throughout (run-specific)

The numeric identifiers below come from **one** captured investigation session. They are
**run-specific**: rebuilding and relaunching yields different PIDs/TID/fd-window values, which is
expected. What is *stable* across runs — and what the answers actually depend on — is the
**shape** of the evidence (which fd is the PTY master, which single thread reads it, the syscall
pair used, the requested buffer size, and the read cadence). Every table and trace excerpt in this
document is drawn from this one session unless stated otherwise.

| Identifier | Value (this session) | Meaning |
|---|---|---|
| kitty PID (`KPID`) | `199951` | the running kitty process (owned by user `ubuntu`, uid 1000) |
| shell PID (`SHPID`) | `200018` | the shell kitty spawned — `/bin/bash --posix` |
| reader thread (`TID`) | `200017` | kitty's I/O thread `KittyChildMon`; the *sole* reader of the PTY master |
| X11 window id (`WID`) | `2097164` | kitty's top-level window (`_NET_WM_PID=199951`) |
| PTY master fd | `8` | kitty's descriptor for `/dev/pts/ptmx` (the master side) |
| PTY slave | `/dev/pts/0` | the shell's controlling terminal (its stdin/stdout/stderr) |
| build id | `4a693e4304285476522c8ac6a4eef4babf9072b7` | launcher `BuildID[sha1]`, launcher size 40384 bytes |

## Methodology (Run-First)

1. **Build canonically and launch the real binary.** kitty is built with its own build driver
   `python3 setup.py` and launched via the produced launcher `kitty/launcher/kitty`. No debug hook,
   remote-control interface, mock, or fallback is used — the genuine PTY entry path is exercised.
2. **Spawn the default shell.** kitty is run with its default configuration (no `kitty.conf`, no
   config env vars — proven in Q1), so the shell it spawns and the read cadence reflect the
   canonical build.
3. **Exercise the exact inputs.** The two inputs are typed into the real kitty window through the
   X server (`xdotool type`/`key` against kitty's window id): the low-volume `echo test123` (Q3)
   and the high-volume `yes hello` (Q4).
4. **Trace the real syscalls.** While the inputs run, an strace is attached to the kitty PID:

   ```text
   strace -f -yy -tt -T -e trace=read,poll -p <KPID> -o <file>
   ```

   * `-f` follows threads — **essential**, because kitty reads the PTY on a dedicated I/O thread,
     not the main thread.
   * `-yy` annotates every descriptor with its backing object, so the PTY master shows up as
     `8</dev/pts/ptmx...>` and can be told apart from kitty's eventfd/signalfd descriptors.
   * `-tt` gives microsecond wall-clock timestamps and `-T` gives per-call durations; together they
     let read *frequency* (Q4) be measured directly from the trace.
   * `-e trace=read,poll` restricts the trace to the two calls that make up the read loop.

5. **Attribution, not eyeballing.** strace with `-f` interleaves all 67 threads and splits calls
   that block into `<unfinished ...>` / `<... read resumed>` halves. A small standard-library Python
   analyzer (full source in **Appendix E**) reconstructs each `read()` by pairing the two halves
   *per thread*, auto-detects the master fd as the descriptor whose annotation contains
   `/dev/pts/ptmx`, and reports totals, the requested-size multiset, the returned-byte distribution
   (with a log2 histogram), the active window, the read frequency, and which thread(s) did the
   reading. Because the analyzer source and the numbers it produces are both in this document, every
   Q3/Q4 statistic here is **recomputable from the deliverable alone**.

6. **Observation, not modification (ptrace note).** The trace was captured by a **root**
   (`CAP_SYS_PTRACE`) strace attaching to the `ubuntu`-owned kitty. `CAP_SYS_PTRACE` bypasses the
   kernel `yama` `ptrace_scope` restriction, so **no kernel setting was changed** to enable tracing;
   the ambient `ptrace_scope` value was left exactly as found. Tracing only *observes* the process.

### Excerpt conventions (so "verbatim" is unambiguous)

* Blocks labelled **verbatim** contain bytes copied directly from the captured artifact with
  nothing added or removed inside the fence. Where only part of a large trace is shown, the prose
  says so and states the line range; the fence still contains only genuine trace bytes.
* Inside strace `read(...)` lines, the `"..."` after a quoted string is **strace's own** default
  32-byte string-truncation marker (three ASCII dots), *not* an editorial ellipsis. No Unicode
  ellipsis (U+2026) appears in any trace excerpt (checked; see Appendix D).
* Control bytes print the way strace writes them: ESC is `\33`, BEL is `\7`, CR/LF are `\r`/`\n`.

### References (methodology and PTY semantics)

* strace(1) manual (`-f`, `-yy`, `-tt`, `-T`, `-e trace=`, `-c`) — man7.org.
* Linux pseudo-terminal architecture: the emulator holds the master (`/dev/ptmx`); the shell holds
  the slave (`/dev/pts/N`); the kernel `N_TTY` line discipline mediates and buffers between them.
  The line-discipline buffer is a few kilobytes, which is why each master `read()` returns far
  fewer bytes than kitty requests during a flood (Q4).

## Q1 — Building kitty and launching it as a normal user

**Answer.** kitty was built from source with its canonical build driver
`python3 setup.py` and launched as the non-root user `ubuntu` via the produced launcher
`kitty/launcher/kitty`. The built version is **kitty 0.35.2**.

### Build (canonical)

Two builds were run to be precise about the exact canonical command on this toolchain.

**(a) The bare `python3 setup.py`** is the upstream default, but on this newer toolchain
(wayland-protocols 1.45 adds `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that the pinned
`glfw/wl_window.c` switch does not yet handle) it stops at kitty's default `-Werror`. The command
and the tail of its log — the failing translation unit and the non-zero exit — are shown verbatim
(the build is parallel, so compile lines from other units interleave before the failure is
reported; note that gcc's diagnostics quote identifiers with Unicode marks, reproduced here exactly as emitted):

```text
$ ( python3 setup.py ; echo "BARE BUILD EXIT = $?" )
```
```text
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
 done
Compiling [wayland] glfw/wl_window.c ...
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
BARE BUILD EXIT = 1
```

**(b) The canonical build** adds kitty's own officially-supported `--ignore-compiler-warnings`
flag (which downgrades `-Werror` without editing any source) and sets `CI=true` to match kitty's
CI convention. It completes successfully (exit 0). The command, the first three and last six lines
of its 159-line log are shown; the middle is elided in this excerpt only (the full log is a build
artifact, not part of the repository):

```text
$ ( CI=true python3 setup.py --ignore-compiler-warnings ; echo "BUILD EXIT STATUS = $?" )
```
```text
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
[2/28] Generating wayland-xdg-shell-client-protocol.c ...
[3/28] Generating wayland-viewporter-client-protocol.h ...
   ...(151 intermediate compile/link lines elided in this excerpt)...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
BUILD EXIT STATUS = 0
```

The produced launcher — its permissions/size, ELF identity (`BuildID[sha1]`), and reported
version — as a command transcript (each `$` line is the command, the line(s) below it are that
command's output):

```text
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 40384 Jul 15 00:06 kitty/launcher/kitty
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4a693e4304285476522c8ac6a4eef4babf9072b7, for GNU/Linux 3.2.0, not stripped
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The build id `4a693e4304285476522c8ac6a4eef4babf9072b7` is reproducible: an independent rebuild of
the same baseline produced a byte-identical launcher.

### Launch (as the non-root user `ubuntu`)

kitty is launched as `ubuntu` (uid 1000) under the headless display. `setsid` detaches it from the
tracer's session; the software-GL env vars select the `llvmpipe` path for the headless server:

```text
$ nohup setsid sudo -u ubuntu -H env DISPLAY=:99 \
      LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe HOME=/home/ubuntu \
      ./kitty/launcher/kitty > kitty_run.log 2>&1 &
$ pgrep -u ubuntu -x kitty        # -> the real kitty PID (KPID)
199951
```

The process is owned by `ubuntu`, confirming it runs as a normal (non-root) user. Its identity is
read from `/proc/<KPID>` in three separate, unmodified captures.

**(i)** The executable behind the process — `readlink` prints the bare target path (no prefix of
its own):

```text
$ readlink /proc/199951/exe
/tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6/kitty/launcher/kitty
```

**(ii)** Selected `/proc/199951/status` fields — the process name, its parent, its uid set (all
`1000` = `ubuntu`), and its thread count:

```text
$ grep -E '^(Name|PPid|Uid|Gid|Threads):' /proc/199951/status
Name:	kitty
PPid:	199948
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
Threads:	67
```

**(iii)** The process command line (`/proc/199951/cmdline`, NUL-separated, rendered with the
trailing NUL shown as a space) — the bare launcher path, no arguments:

```text
$ tr '\0' ' ' < /proc/199951/cmdline ; echo
/tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6/kitty/launcher/kitty 
```

### Default, canonical configuration (proof)

The shell identity (Q2) and read cadence (Q3/Q4) depend on kitty running with its default
configuration. Three checks confirm no configuration overrides are in effect.

**(a)** No system config file exists at the path kitty reads
(`cli.py:L1064` defines `SYSTEM_CONF = '/etc/xdg/kitty/kitty.conf'`):

```text
$ ls -l /etc/xdg/kitty/kitty.conf
ls: cannot access '/etc/xdg/kitty/kitty.conf': No such file or directory
```

**(b)** The per-user config directory (`$HOME/.config/kitty` for `ubuntu`) is empty — it contains
no `kitty.conf` (only the `.`/`..` entries):

```text
$ sudo -u ubuntu -H ls -la /home/ubuntu/.config/kitty ; echo "HOME=$HOME(for ubuntu)"
total 8
drwxr-xr-x 2 ubuntu ubuntu 4096 Jul 14 20:00 .
drwxr-xr-x 3 ubuntu ubuntu 4096 Jul 14 20:00 ..
HOME=/home/ubuntu
```

**(c)** None of kitty's config-selecting environment variables are set in the running process
(`constants.py:_get_config_dir()` at `L87-L131` consults `KITTY_CONFIG_DIRECTORY` at `L88` and
`XDG_CONFIG_HOME` at `L92`):

```text
$ for v in KITTY_CONFIG_DIRECTORY XDG_CONFIG_HOME XDG_CONFIG_DIRS; do \
      tr '\0' '\n' < /proc/199951/environ | grep "^$v=" || true ; done
(none of KITTY_CONFIG_DIRECTORY / XDG_CONFIG_HOME / XDG_CONFIG_DIRS set)
```

Together these establish that kitty is running the **default, canonical** configuration.

## Q2 — The process kitty spawns for the shell (PID, exact command line, PTY device)

**Answer.** kitty spawned the process **`/bin/bash --posix`**, PID **200018**, as a direct child of
kitty (PPID 199951). The PTY device connecting kitty to that shell is the pair
`/dev/pts/ptmx` (master, held by kitty) ↔ **`/dev/pts/0`** (slave, the shell's controlling
terminal). The exact command line — as it appears in the process list and byte-for-byte in
`/proc/<pid>/cmdline` — is `/bin/bash --posix`.

### Evidence

**Process and parent** (`ps` filtered to children of kitty, PID 199951):

```text
$ ps --ppid 199951 -o pid,ppid,user,cmd
    PID    PPID USER     CMD
 200018  199951 ubuntu   /bin/bash --posix
```

**Exact command line, byte-for-byte** — `/proc/200018/cmdline` is NUL-separated; a hex dump removes
any doubt about the exact argv (`/bin/bash\0--posix\0`):

```text
$ xxd /proc/200018/cmdline
00000000: 2f62 696e 2f62 6173 6800 2d2d 706f 7369  /bin/bash.--posi
00000010: 7800                                     x.
```

The two NUL-separated fields are `argv[0] = /bin/bash` and `argv[1] = --posix`; there are no other
arguments.

**PTY device** — the shell's stdin/stdout/stderr are all the slave side `/dev/pts/0`:

```text
$ ls -l /proc/200018/fd/0 /proc/200018/fd/1 /proc/200018/fd/2
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:07 /proc/200018/fd/0 -> /dev/pts/0
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:07 /proc/200018/fd/1 -> /dev/pts/0
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:07 /proc/200018/fd/2 -> /dev/pts/0
```

**Default shell source** — `ubuntu`'s login shell is `/bin/bash`, so the resolved default shell is
`/bin/bash`:

```text
$ getent passwd ubuntu
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
```

### Notes on the observed values

* **Why `--posix`?** The `--posix` argument is *not* a login-shell dash prefix; it is inserted by
  kitty's **bash shell-integration**. `shell_integration.py:L146` does `argv.insert(1, '--posix')`,
  and `L134` sets `env['ENV']` to kitty's `kitty.bash` integration script. POSIX-mode bash sources
  `$ENV` on startup, so kitty injects its integration **through the environment**, not by wrapping
  the shell in an extra process. That is why the shell appears as a single `/bin/bash --posix`
  process and not, e.g., a login wrapper. This is the default (`shell_integration` enabled) behavior.
* **Linux vs macOS.** On Linux `should_run_via_run_shell_kitten = is_macos and self.is_default_shell`
  (`child.py:L230`) is `False`, so the macOS login-shell wrapper block (`child.py:L295` onward) is
  skipped; the process is the plain resolved shell with the integration argv adjustment above.

### Source-code rationale for the spawn path (inferred from code)

The observed values follow this path (Python → C):

* `resolved_shell()` (`utils.py:L768`) returns `[shell_path]` for the default case `q == '.'`
  (`L770-L771`); `shell_path` (`constants.py:L181`, fallback `/bin/sh` at `L185`) is `ubuntu`'s
  login shell `/bin/bash`.
* `Child.fork()` (`child.py:L276`) creates the PTY with `os.openpty()` (`child.py:L171`) and calls
  the C `spawn()` (`child.py:L333`).
* C `spawn()` is defined at `child.c:L80` (return type `static PyObject*`) / `L81` (signature). It
  derives the slave device path with `ttyname_r(slave, ...)` (`child.c:L88`) — this is the
  `/dev/pts/0` path the shell ends up on — then in the child `fork()` (`L97`) it calls `setsid()`
  (`L123`), sets the controlling terminal with `ioctl(..., TIOCSCTTY, ...)` (`L129`), wires the
  slave to stdio with `dup2()` (`L138`, `L145`), and finally `execvp()`s the shell (`L159`).

## Q3 — Reading a small input (`echo test123`): syscalls, requested size, bytes returned

**Answer.** kitty reads the PTY with a `poll()` + `read()` pair on its I/O thread: `poll()` waits
for the master fd to become readable (`POLLIN`), then `read()` pulls the bytes. Each `read()`
requests up to the free space in kitty's 1 MiB parser buffer — here **1048576 bytes** (the full
`BUF_SZ`) at first, shrinking slightly to `1048565`, `1048518`, `1048404` on the final reads as a
few unparsed bytes accumulate. For the whole `echo test123` interaction there were **16 reads on
the master fd returning 617 bytes in total**: reads 1–12 are the single-byte echoes of the 12 typed
characters, and reads 13–16 (11 + 47 + 114 + 433 bytes) are the command output and prompt redraw
after Return.

### Input injection into the real kitty window (observed)

The exact user input is typed into kitty's window (id 2097164, `_NET_WM_PID=199951`) through the X
server:

```text
$ xdotool type --window 2097164 'echo test123'
$ xdotool key  --window 2097164 Return
```

### Trace capture (observed)

The tracer is attached to kitty just before injecting, following all threads and annotating fds:

```text
$ strace -f -yy -tt -T -e trace=read,poll -p 199951 -o echo.strace
strace: Process 199951 attached with 67 threads
```

The banner confirms the tracer attached to all 67 threads (the reader is one of them).

### The read loop, verbatim (observed)

A genuine contiguous slice (echo.strace lines 20–28) around the first keystroke `e`. strace with
`-f` interleaves threads and splits a blocking call into `<unfinished ...>` / `<... poll resumed>`
halves; here thread `200017` (the PTY reader) polls fd 8, the poll resumes readable, and the
`read(8..., "e", 1048576) = 1` returns the single echoed byte. Lines from the main thread `199951`
(polling its X11 socket fd 3) are shown exactly as they interleave — nothing removed:

```text
200017 00:08:07.675470 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN|POLLOUT}], 3, -1 <unfinished ...>
199951 00:08:07.675551 <... poll resumed>) = 1 ([{fd=3, revents=POLLOUT}]) <0.000085>
200017 00:08:07.675576 <... poll resumed>) = 1 ([{fd=8, revents=POLLOUT}]) <0.000030>
199951 00:08:07.675666 poll([{fd=3<UNIX-STREAM:[792387857->792414338]>, events=POLLIN}], 1, -1 <unfinished ...>
200017 00:08:07.675725 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1 <unfinished ...>
199951 00:08:07.675832 <... poll resumed>) = 1 ([{fd=3, revents=POLLIN}]) <0.000111>
200017 00:08:07.675845 <... poll resumed>) = 1 ([{fd=8, revents=POLLIN}]) <0.000019>
200017 00:08:07.675862 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "e", 1048576) = 1 <0.000018>
200017 00:08:07.675938 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1 <unfinished ...>
```

After Return, the shell emits the command output and the shell-integration prompt redraw. The last
four master reads (verbatim) return 11, 47, 114 and 433 bytes. Note the requested size stepping
down `1048576 → 1048565 → 1048518 → 1048404`: each request is `BUF_SZ` minus the bytes already
sitting unparsed in the buffer, so the decrements equal the previous returns
(`1048576 − 11 = 1048565`, `1048565 − 47 = 1048518`, `1048518 − 114 = 1048404`). The `\33`, `\7`
bytes are ESC and BEL of the OSC 133 shell-integration sequences; the trailing `"..."` is strace's
own 32-byte string truncation:

```text
200017 00:08:08.244384 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\n\33[?2004l\r", 1048576) = 11 <0.000010>
200017 00:08:08.246009 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;echo test123\7\33]133;C;cmdline"..., 1048565) = 47 <0.000016>
200017 00:08:08.246289 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1\33]133;k;start_kitty\7\2\1\33]133;k;e"..., 1048518) = 114 <0.000015>
200017 00:08:08.247206 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[?2004h\33[59P\33]133;k;start_kitty"..., 1048404) = 433 <0.000035>
```

### All 16 master reads for `echo test123` (observed)

Reconstructed by the analyzer (Appendix E) by pairing the `<unfinished ...>`/`<... read resumed>`
halves per thread and keeping only reads on the `/dev/pts/ptmx` master fd. `t_rel` is milliseconds
since the first master read:

| # | t_rel (ms) | requested (bytes) | returned (bytes) | TID |
|---|-----------:|------------------:|-----------------:|-----|
| 1 | 0.000 | 1048576 | 1 | 200017 |
| 2 | 1.105 | 1048575 | 1 | 200017 |
| 3 | 2.167 | 1048574 | 1 | 200017 |
| 4 | 9.806 | 1048576 | 1 | 200017 |
| 5 | 10.907 | 1048575 | 1 | 200017 |
| 6 | 20.513 | 1048576 | 1 | 200017 |
| 7 | 23.701 | 1048575 | 1 | 200017 |
| 8 | 32.142 | 1048576 | 1 | 200017 |
| 9 | 43.174 | 1048576 | 1 | 200017 |
| 10 | 44.343 | 1048575 | 1 | 200017 |
| 11 | 55.725 | 1048576 | 1 | 200017 |
| 12 | 57.289 | 1048575 | 1 | 200017 |
| 13 | 568.522 | 1048576 | 11 | 200017 |
| 14 | 570.147 | 1048565 | 47 | 200017 |
| 15 | 570.427 | 1048518 | 114 | 200017 |
| 16 | 571.344 | 1048404 | 433 | 200017 |

SUM returned bytes = 617 across 16 reads on fd 8

The analyzer's summary for this trace (recomputable by running `analyze.py echo.strace` — see
Appendix E) — note the requested-size multiset (mostly the full `1048576`), that the returned bytes
sum to 617, that **fd 8 had zero EAGAIN and zero short/zero-length reads**, and that a **single
thread, TID 200017**, issued all 16 reads:

```text
MASTER fd=8 (/dev/pts/ptmx<char 5:2 @/dev/pts/0)
  reads total(incl err/0)=16  data-returning=16
  fd8 EAGAIN=0  fd8 zero-length=0
  window=0.571s  freq=28 reads/s
  bytes/read: min=1 median=1 mean=38.6 max=433
  total bytes=617 (0.00 MiB)  throughput=0.00 MiB/s
  window boundaries: first=487.675862s last=488.247206s
  requested sizes (size:count): 1048404:1, 1048518:1, 1048565:1, 1048574:1, 1048575:5, 1048576:7
  returned-size histogram [2^b .. 2^(b+1)) : count]:
    [      1 ..       2) : 12
    [      8 ..      16) : 1
    [     32 ..      64) : 1
    [     64 ..     128) : 1
    [    256 ..     512) : 1
  reader TID(s): {'200017': 16}
  read errors by (fd, errno):
    fd=4 EAGAIN: 13
    fd=6 EAGAIN: 13
    fd=8 (PTY master) errors: 0
```

### Source-code rationale (inferred from code)

* The reader is `read_bytes()` (`child-monitor.c:L1337`), which calls `read()` (`L1345`) into the
  buffer returned by `vt_parser_create_write_buffer()`; that buffer's size is `*sz = BUF_SZ - offset`
  (`vt-parser.c:L1457`), and `BUF_SZ` is `1024u * 1024u` = 1 MiB (`vt-parser.c:L18`). This is why the
  requested size is ≈ 1 MiB and shrinks by exactly the unparsed backlog.
* `poll()` gates each read because the master fd is non-blocking (`child.py:L345`); the I/O loop
  arms `POLLIN` on the master only while the parser has room (`child-monitor.c:L1501` via
  `vt_parser_has_space_for_input()`).
* The single-byte reads 1–12 are the terminal echoing each typed character back on the master as it
  is typed; the four reads after Return are the command's output plus the OSC 133 prompt-marking
  redraw emitted by the shell-integration script.

## Q4 — Reading a high-volume stream (`yes hello`): how the read behaviour changes

**Answer.** Under `yes hello` the same `poll()` + `read()` loop runs continuously and at high
frequency: roughly **7,000 reads per second** on the master fd (mean **6,973/s** across three runs).
Each individual `read()` still *requests* ≈ 1 MiB, but the kernel PTY line-discipline buffer caps
what it *returns* to a few kilobytes — the **mean return is ≈ 1,375 bytes** and the median ≈ 1,167
bytes, with the largest single read in any run being **20,433 bytes (≈ 19.95 KiB, binary)**. So the
change from Q3 is *cadence and volume*, not mechanism: many more reads, each far below the requested
buffer size, driven back-to-back while data is available. The reader is still the single I/O thread
(TID 200017) and the master fd still never returns EAGAIN.

### Scale, duration, and the stability criterion (stated up front)

`yes hello` was streamed for an **~8-second** traced window in **three independent runs** (Rule 1
requires ≥ 2). Between runs the stream was stopped with Ctrl-C and the prompt confirmed recovered.
The per-read byte counts are capped by the kernel, so they are stable regardless of run length; the
read *frequency* is the quantity that could drift, so its stability is the criterion. Across the
three runs the read frequency stayed within **7.01%** and the mean bytes/read within **4.69%** (full
table below) — i.e. stable.

### Run 1 — raw evidence and analysis (`yes hello`)

A genuine contiguous slice (yes1.strace lines 60000–60028), all from the reader thread 200017, shows
the sustained cadence: each `poll()` on `[fd6 eventfd, fd7 signalfd, fd8 ptmx]` returns `fd8`
readable, and the following `read(8..., "hello\r\nhello\r\n...", <req>) = <ret>` pulls one PTY-buffer's
worth. Two things are visible directly in the bytes: (1) returns are ~800–1,700 here, far below the
~1 MiB request; (2) the **requested size steps down by exactly the previous return**
(`1030341 − 1169 = 1029172`, `1029172 − 1405 = 1027767`, and so on) — the parser is briefly behind, so
`BUF_SZ − offset` shrinks — until a `poll(..., -1)` shows the parser caught up. The poll timeout
argument cycles `0 / -1 / 2 / 1` (the `input_delay` coalescing window; see the cadence subsection):

```text
200017 00:13:57.562488 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}]) <0.000010>
200017 00:13:57.562551 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1030341) = 1169 <0.000012>
200017 00:13:57.562595 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}]) <0.000011>
200017 00:13:57.562661 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1029172) = 1405 <0.000013>
200017 00:13:57.562725 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000011>
200017 00:13:57.562786 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1027767) = 1673 <0.000029>
200017 00:13:57.562849 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
200017 00:13:57.562917 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1026094) = 1589 <0.000013>
200017 00:13:57.562960 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000014>
200017 00:13:57.563031 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1024505) = 821 <0.000014>
200017 00:13:57.563086 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000011>
200017 00:13:57.563150 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1023684) = 1202 <0.000011>
200017 00:13:57.563196 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000011>
200017 00:13:57.563260 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1022482) = 1225 <0.000008>
200017 00:13:57.563297 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
200017 00:13:57.563358 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1021257) = 1094 <0.000007>
200017 00:13:57.563397 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000013>
200017 00:13:57.563459 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1020163) = 838 <0.000016>
200017 00:13:57.563511 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000018>
200017 00:13:57.563581 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1019325) = 1393 <0.000018>
200017 00:13:57.563633 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
200017 00:13:57.563702 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1017932) = 1577 <0.000013>
200017 00:13:57.563754 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000030>
200017 00:13:57.563858 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1016355) = 1575 <0.000012>
200017 00:13:57.563904 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
200017 00:13:57.563969 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1014780) = 1587 <0.000014>
200017 00:13:57.564012 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000015>
200017 00:13:57.564083 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1013193) = 1598 <0.000014>
200017 00:13:57.564131 poll([{fd=6<{eventfd-count=0, eventfd-id=575, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000010>
```

The analyzer's summary for Run 1 (recomputable via `analyze.py yes1.strace`; the log2 histogram of
returned sizes and the active-window boundaries are included so the frequency and distribution can
be re-derived from this document):

```text
MASTER fd=8 (/dev/pts/ptmx<char 5:2 @/dev/pts/0)
  reads total(incl err/0)=56817  data-returning=56817
  fd8 EAGAIN=0  fd8 zero-length=0
  window=8.106s  freq=7010 reads/s
  bytes/read: min=1 median=1129 mean=1338.7 max=18786
  total bytes=76058405 (72.53 MiB)  throughput=8.95 MiB/s
  window boundaries: first=833.775356s last=841.881037s
  returned-size histogram [2^b .. 2^(b+1)) : count]:
    [      1 ..       2) : 9
    [      2 ..       4) : 1
    [      8 ..      16) : 1
    [     32 ..      64) : 1
    [     64 ..     128) : 1
    [    128 ..     256) : 20
    [    256 ..     512) : 945
    [    512 ..    1024) : 20612
    [   1024 ..    2048) : 29409
    [   2048 ..    4096) : 4966
    [   4096 ..    8192) : 807
    [   8192 ..   16384) : 37
    [  16384 ..   32768) : 8
  reader TID(s): {'200017': 56817}
  read errors by (fd, errno):
    fd=4 EAGAIN: 570
    fd=6 EAGAIN: 13
    fd=8 (PTY master) errors: 0
```

### Run 2 — analysis (independent repeat)

```text
MASTER fd=8 (/dev/pts/ptmx<char 5:2 @/dev/pts/0)
  reads total(incl err/0)=58276  data-returning=58276
  fd8 EAGAIN=0  fd8 zero-length=0
  window=8.095s  freq=7199 reads/s
  bytes/read: min=1 median=1146 mean=1381.6 max=20433
  total bytes=80513424 (76.78 MiB)  throughput=9.49 MiB/s
  window boundaries: first=1064.587529s last=1072.682519s
  returned-size histogram [2^b .. 2^(b+1)) : count]:
    [      1 ..       2) : 9
    [      2 ..       4) : 1
    [      8 ..      16) : 1
    [     32 ..      64) : 2
    [     64 ..     128) : 2
    [    128 ..     256) : 13
    [    256 ..     512) : 282
    [    512 ..    1024) : 20117
    [   1024 ..    2048) : 31750
    [   2048 ..    4096) : 4946
    [   4096 ..    8192) : 1103
    [   8192 ..   16384) : 33
    [  16384 ..   32768) : 17
  reader TID(s): {'200017': 58276}
  read errors by (fd, errno):
    fd=4 EAGAIN: 558
    fd=6 EAGAIN: 17
    fd=8 (PTY master) errors: 0
```

### Run 3 — analysis, and the aggregate `strace -c` histogram

```text
MASTER fd=8 (/dev/pts/ptmx<char 5:2 @/dev/pts/0)
  reads total(incl err/0)=54325  data-returning=54325
  fd8 EAGAIN=0  fd8 zero-length=0
  window=8.096s  freq=6710 reads/s
  bytes/read: min=1 median=1225 mean=1403.2 max=20258
  total bytes=76230395 (72.70 MiB)  throughput=8.98 MiB/s
  window boundaries: first=1264.178648s last=1272.274828s
  returned-size histogram [2^b .. 2^(b+1)) : count]:
    [      1 ..       2) : 9
    [      2 ..       4) : 1
    [      8 ..      16) : 1
    [     32 ..      64) : 1
    [     64 ..     128) : 2
    [    128 ..     256) : 3
    [    256 ..     512) : 443
    [    512 ..    1024) : 16444
    [   1024 ..    2048) : 30943
    [   2048 ..    4096) : 5888
    [   4096 ..    8192) : 551
    [   8192 ..   16384) : 36
    [  16384 ..   32768) : 3
  reader TID(s): {'200017': 54325}
  read errors by (fd, errno):
    fd=4 EAGAIN: 572
    fd=6 EAGAIN: 11
    fd=8 (PTY master) errors: 0
```

A separate `strace -c -f` run (counts only, all fds/threads) over a comparable ~8 s window quantifies
the syscall magnitude: `poll` and `read` dominate almost equally, with the `read` errors being the
EAGAIN on the non-PTY fds (see EAGAIN attribution). This counts every thread's reads (including the
wakeup eventfd), so its `read` total exceeds the master-only count:

```text
$ strace -c -f -e trace=read,poll -p 199951 -o yes_c.txt   # ~8 s of `yes hello`
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.83    1.139673          11     98731           poll
 49.17    1.102409          11     94470       683 read
------ ----------- ----------- --------- --------- ----------------
100.00    2.242082          11    193201       683 total
```

### Stability across the three runs

```text
Q4 THREE-RUN STABILITY (yes hello, ~8s window each, master fd 8)
==================================================================
       total reads: [56817, 58276, 54325]  mean=56472.7  range[54325..58276]  spread=7.00%
      freq reads/s: [7010, 7199, 6710]  mean=6973.0  range[6710..7199]  spread=7.01%
   mean bytes/read: [1338.7, 1381.6, 1403.2]  mean=1374.5  range[1338.7..1403.2]  spread=4.69%
 median bytes/read: [1129, 1146, 1225]  mean=1166.7  range[1129..1225]  spread=8.23%
    max bytes/read: [18786, 20433, 20258]  mean=19825.7  range[18786..20433]  spread=8.31%
         MiB total: [72.53, 76.78, 72.7]  mean=74.0  range[72.53..76.78]  spread=5.74%
  MiB/s throughput: [8.95, 9.49, 8.98]  mean=9.1  range[8.95..9.49]  spread=5.91%
        reader TID: all three = 200017 (identical)
        fd8 EAGAIN: all three = 0 (identical)

STABILITY VERDICT: read frequency stable within 7.01% and mean bytes/read within 4.69% across 3 independent runs (>= 2 required).
```

**Reading the maxima correctly.** The per-run *largest single read* values are 18,786 / 20,433 /
20,258 bytes. The **single largest read observed across all three runs is 20,433 bytes
(= 19.954 KiB binary)**, in Run 2. The **mean of the three per-run maxima is 19,825.7 bytes
(= 19.361 KiB binary)** — this is an *average of maxima*, not itself "the largest read", and is
reported separately to avoid conflating the two. All KiB figures here are binary (÷1024).

### Per-read size: measured distribution, and why each read is small

The log2 histograms in the three run summaries above tell a consistent story: the overwhelming
majority of reads land in `[512 .. 2048)` bytes (Run 1: 20,612 reads in `[512..1024)` and 29,409 in
`[1024..2048)`), with a thin tail up to ~20 KiB and essentially nothing near the ~1 MiB request. In
order-of-magnitude terms, each `read()` **requests** `BUF_SZ` = 1,048,576 bytes but the mean
**return** of ≈ 1,375 bytes is about **763× smaller**, and even the largest observed return
(20,433 bytes) is about **51× smaller** than the request. The cause is not kitty: it is the Linux
`N_TTY` line-discipline buffer on the PTY, which is only a few kilobytes, so the master `read()`
drains at most one buffer's worth per call no matter how large the request. This is the direct,
observed reason `yes hello` produces *many small reads* rather than one big read.

### `poll` cadence and `input_delay` coalescing (observed)

The distribution of the `poll()` timeout argument on the reader thread, across the three runs,
explains the cadence. A timeout of `-1` is a *blocking* poll (used when the parser has caught up and
there is nothing pending); timeouts of `0/1/2` ms are *timed* polls — kitty's `input_delay`
countdown (default 3 ms) that briefly coalesces incoming bursts before handing them to the parser:

```text
### poll timeouts run1
poll() calls with a parseable timeout arg: 54129
timeout(ms):count  (-1 = block forever)
    -1 : 2676
     0 : 17714
     1 : 17581
     2 : 16158

### poll timeouts run2
poll() calls with a parseable timeout arg: 55783
timeout(ms):count  (-1 = block forever)
    -1 : 2711
     0 : 18259
     1 : 18246
     2 : 16567

### poll timeouts run3
poll() calls with a parseable timeout arg: 51600
timeout(ms):count  (-1 = block forever)
    -1 : 2666
     0 : 16968
     1 : 16702
     2 : 15264
```

The `~2,700` blocking (`-1`) polls per run are the moments the stream momentarily drained; the
~16,000–18,000 polls at each of `0/1/2` ms are the coalescing countdown running during the flood.
This corresponds to the timed poll at `child-monitor.c:L1509` versus the blocking poll at `L1512`.

### EAGAIN attribution (observed)

Every run summary reports **`fd8 EAGAIN=0`** and **`fd=8 (PTY master) errors: 0`**: the master fd
never returned EAGAIN, because `poll()` only lets the code `read()` when `fd8` is already readable.
The EAGAIN counts the analyzer *does* report (e.g. Run 1: `fd=4 EAGAIN: 570`, `fd=6 EAGAIN: 13`)
are on kitty's **wakeup eventfd** descriptors (fd 4 = main-loop wakeup, fd 6 = I/O-loop wakeup), not
on the PTY — a distinction the per-(fd,errno) attribution in the analyzer makes explicit. This is a
correction worth stating plainly: EAGAIN in these traces is an eventfd artifact, never a PTY-master
event.

### Termination and return-to-prompt (observed)

The last master reads of Run 1 (verbatim) show the stream ending: the `hello\r\n` payloads taper,
a `read = 2` returns just `"\r\n"`, and the final read returns 435 bytes beginning with
`\33[?2004h` (bracketed-paste enable) and the OSC 133 `\33]133;k;start_kitty...` prompt marker — i.e.
the prompt being redrawn after Ctrl-C:

```text
200017 00:14:01.878748 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 860474) = 1071 <0.000016>
200017 00:14:01.878926 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 859403) = 1521 <0.000015>
200017 00:14:01.879079 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 857882) = 1398 <0.000013>
200017 00:14:01.879233 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 856484) = 1194 <0.000013>
200017 00:14:01.879859 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\n", 855290) = 2 <0.000016>
200017 00:14:01.881037 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[?2004h\33[59P\33]133;k;start_kitty"..., 855288) = 435 <0.000020>
```

After the runs, the shell is alive at a clean prompt and no `yes` process remains:

```text
$ ps -o pid,ppid,user,stat,args -p 200018
 PID PPID USER STAT COMMAND
 200018 199951 ubuntu Ss+ /bin/bash --posix

$ ps --ppid 200018 -o pid,stat,args   # children of the shell
(no children)

$ pgrep -u ubuntu -x yes && echo RUNNING || echo 'no yes process'
no yes process
```

A live prompt-responsiveness probe — typing `echo PROMPT_OK_777` after the flood — produces a single
small master read (59 bytes) that echoes the typed command via OSC 2 / OSC 133 sequences, and it is
still read by the same thread 200017, with the cadence back to Q3-like single small reads:

```text
200017 00:25:10.721205 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;echo PROMPT_OK_777\7\33]133;C;c"..., 1048565) = 59 <0.000018>
```

### Source-code rationale (inferred from code)

* The read loop is `io_loop()` (`child-monitor.c:L1481`): it `poll()`s (`L1509` timed / `L1512`
  blocking) and calls `read_bytes()` (`L1337`, `read()` at `L1345`) whenever the master is readable
  and the parser has room (`vt_parser_has_space_for_input()`, `L1477`, gate
  `read.sz + write.pending < BUF_SZ` at `L1481`).
* The requested size `BUF_SZ − offset` (`vt-parser.c:L1457`) shrinks during a burst because the
  producer (I/O thread) outruns the consumer (main-thread parser at `process_global_state()`,
  `L1224`, calling `parse_input()` at `L1236`); when the parser commits, `offset` resets and the
  next request returns to the full ~1 MiB.
* The small per-read returns are a kernel property of the PTY line discipline, independent of kitty.

## Q5 — The file-descriptor number kitty uses to read the PTY master

**Answer.** kitty read the PTY master on **file descriptor 8**, which `/proc` and strace both show
backed by `/dev/pts/ptmx` (the master side; the shell's slave side is `/dev/pts/0`).

### Evidence

The concrete fd, and kitty's full descriptor table, from `/proc/199951/fd`, plus the strace `-yy`
annotation that labels fd 8 as the ptmx master (`char 5:2` is the `/dev/ptmx` device, `@/dev/pts/0`
names the slave it is paired with):

```text
=== Q5 master fd ===
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:25 /proc/199951/fd/8 -> /dev/pts/ptmx

total 0
lr-x------ 1 ubuntu ubuntu 64 Jul 15 00:25 0 -> /dev/null
l-wx------ 1 ubuntu ubuntu 64 Jul 15 00:25 1 -> /tmp/kitty_qa_rerun/kitty_run.log
l-wx------ 1 ubuntu ubuntu 64 Jul 15 00:25 2 -> /tmp/kitty_qa_rerun/kitty_run.log
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:08 3 -> socket:[792387857]
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:08 4 -> anon_inode:[eventfd]
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:25 5 -> /memfd:allocation fd (deleted)
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:25 6 -> anon_inode:[eventfd]
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:25 7 -> anon_inode:[signalfd]
lrwx------ 1 ubuntu ubuntu 64 Jul 15 00:25 8 -> /dev/pts/ptmx

strace -yy annotation:
200017 00:13:53.775356 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>> <unfinished ...>
```

The descriptor layout is worth reading: fd 0 → `/dev/null`; fd 1/2 → the launch log; fd 3 → the X11
socket; **fd 4 → an eventfd** (the main-loop wakeup); fd 5 → a memfd; **fd 6 → an eventfd** (the
I/O-loop wakeup); **fd 7 → a signalfd**; and **fd 8 → `/dev/pts/ptmx`** (the PTY master). The I/O
thread's `poll()` set seen throughout the traces is exactly `[fd6, fd7, fd8]`: the two
`EXTRA_FDS` (`child-monitor.c:L35`) — the wakeup eventfd (`loop-utils.c:L70`) and the signalfd
(`loop-utils.c:L42`), tracked as `children_fds[0]`/`children_fds[1]` at `child-monitor.c:L183` —
plus the one child master fd. (fd 4 is the *main* loop's separate wakeup eventfd, polled on the main
thread.)

### Note on `poll` vs. non-blocking (precise roles)

The master fd is made non-blocking (`child.py:L345`, `os.set_blocking(child_fd, False)`); `poll()`
is what makes the loop *wait* for readability, and because a read is only issued after `poll()`
reports `POLLIN` on fd 8, the non-blocking `read()` effectively never has to return EAGAIN on the
master (matching the observed `fd8 EAGAIN=0`).

### Source-code rationale for the fd's origin (inferred from code)

fd 8 is the master end created by `os.openpty()` in `Child.__init__`/`fork()`
(`child.py:L171`) and stored as `self.child_fd` (`child.py:L338`). It is handed to the C child
monitor by `Boss.add_child()` (`boss.py:L585`, call at `L587`) which calls the C `add_child()`
(`child-monitor.c:L305`); from then on the I/O thread polls and reads that descriptor. The specific
integer (8) is assigned by the kernel at `openpty()` time and is therefore run-specific, but its
*role* — kitty's single PTY-master descriptor — is fixed.

## Q6 — The reader function and the text-vs-escape parser function

**Answer.** The function that reads from the PTY file descriptor is **`read_bytes()`**
(`child-monitor.c:L1337`), which issues the `read()` at `L1345`. The function that parses the
incoming data — separating printable text from escape/control sequences — is **`consume_input()`**
(`vt-parser.c:L1367`); its normal-text branch **`consume_normal()`** (`vt-parser.c:L230`) is what
actually walks the bytes and splits printable runs from escapes.

### Observed grounding (who does the reading)

Every trace attributes 100% of the master-fd reads to a single thread, TID 200017, and that thread's
name in `/proc` is `KittyChildMon` — the name kitty gives its I/O thread
(`set_thread_name("KittyChildMon")`, `child-monitor.c:L1489`, inside `io_loop()`). Its read counts
per trace, and the total thread count (matching the strace attach banner), confirm it is the one and
only PTY reader among 67 threads:

```text
=== Q6 reader thread ===
comm: KittyChildMon
thread count: 67
reader TID master-fd read counts per trace:
  echo.strace : 16 fd8-read lines
  yes1.strace : 56817 fd8-read lines
```

Because `io_loop()` is the body of the `KittyChildMon` thread and `io_loop()` calls `read_bytes()`
(`child-monitor.c:L1531`) which calls `read()` (`L1345`), the observed reader thread ties directly
to `read_bytes()` as the reading function.

### How the printable text is separated from escape sequences (inferred from code)

The read bytes are handed to the parser on the main thread: `process_global_state()`
(`child-monitor.c:L1224`) calls `parse_input()` (`L1236`), which calls the VT parser's
`consume_input()` (`vt-parser.c:L1367`). `consume_input()` dispatches on parser state (`L1377`); in
the normal (non-escape) state it calls `consume_normal()` (`vt-parser.c:L230`). `consume_normal()`:

* calls `utf8_decode_to_esc()` (`vt-parser.c:L232`; defined in `simd-string.c:L72`) which decodes a
  **run of printable UTF-8 text up to — but not including — the next ESC byte**, and
* hands that decoded run to `screen_draw_text()` (`vt-parser.c:L236`; `screen.c:L866`) to be drawn,
* then, when it stops at an ESC (`\33`), switches the parser into escape-handling with
  `SET_STATE(ESC)` (`vt-parser.c:L238`) so the following bytes are dispatched as a control/escape
  sequence rather than drawn.

So the split is: `utf8_decode_to_esc()` finds the boundary between printable text and the next
escape, `screen_draw_text()` consumes the printable side, and `SET_STATE(ESC)` routes the escape
side to the control-sequence handlers. This is exactly the `hello\r\n...` (printable) vs
`\33[?2004h`, `\33]133;...` (escape/OSC) division visible in the Q3/Q4 read payloads above.

## Appendix A — Answers at a glance

| Q | Direct answer |
|---|---|
| **Q1 build/launch** | Built from source with `CI=true python3 setup.py --ignore-compiler-warnings` (bare `python3 setup.py` fails only on this newer toolchain's `-Werror`); launched `kitty/launcher/kitty` as non-root user `ubuntu`. Version **kitty 0.35.2**. |
| **Q2 shell** | Process **`/bin/bash --posix`**, PID **200018**, child of kitty (199951). PTY = `/dev/pts/ptmx` (master) ↔ **`/dev/pts/0`** (slave). `--posix` is injected by kitty's bash shell-integration (`shell_integration.py:L146`). |
| **Q3 echo test123** | `poll()` + `read()` on the master fd. Each `read()` requests up to `BUF_SZ − offset` ≈ **1,048,576 bytes**. **16 reads, 617 bytes total** (12 single-byte echoes + 11/47/114/433 after Return). fd 8 EAGAIN = 0. |
| **Q4 yes hello** | Same loop, continuous and fast: mean **6,973 reads/s** (3-run spread 7.01%). Each read still requests ≈1 MiB but returns a kernel-capped few KB: **mean ≈ 1,375 B, median ≈ 1,167 B**; **largest single read 20,433 B (≈ 19.95 KiB binary)**; mean-of-per-run-maxima 19,826 B (≈ 19.36 KiB). Stable across 3 runs. |
| **Q5 fd number** | **fd 8**, backed by `/dev/pts/ptmx`. |
| **Q6 functions** | Reader **`read_bytes()`** (`child-monitor.c:L1337`, `read()` at `L1345`); parser **`consume_input()`** (`vt-parser.c:L1367`) with normal-text branch **`consume_normal()`** (`L230`) splitting printable text (`utf8_decode_to_esc` → `screen_draw_text`) from escapes (`SET_STATE(ESC)`). |

## Appendix B — Reproducible command procedure (consolidated)

```text
# 1. build (canonical) and launch as the non-root user 'ubuntu' under a headless display
Xvfb :99 -screen 0 1280x1024x24 +extension GLX +render -noreset -ac &
git clean -fdX                                   # remove ignored build artifacts
CI=true python3 setup.py --ignore-compiler-warnings
nohup setsid sudo -u ubuntu -H env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
      GALLIUM_DRIVER=llvmpipe HOME=/home/ubuntu ./kitty/launcher/kitty >kitty_run.log 2>&1 &

# 2. discover identifiers
KPID=$(pgrep -u ubuntu -x kitty)
SHPID=$(ps --ppid "$KPID" -o pid= | tr -d ' ')
WID=$(DISPLAY=:99 xdotool search --pid "$KPID" | head -1)

# 3. attach tracer (follow threads; annotate fds), inject the exact inputs, stop the tracer
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o echo.strace &
SP=$!
DISPLAY=:99 xdotool type --window "$WID" 'echo test123'
DISPLAY=:99 xdotool key  --window "$WID" Return
kill "$SP"; wait "$SP" 2>/dev/null

# 4. high-volume run(s): stream, then stop with Ctrl-C and confirm termination
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o yes1.strace &
SP=$!
DISPLAY=:99 xdotool type --window "$WID" 'yes hello'; DISPLAY=:99 xdotool key --window "$WID" Return
sleep 8
DISPLAY=:99 xdotool key --window "$WID" ctrl+c
kill "$SP"; wait "$SP" 2>/dev/null

# 5. analyse (authoritative per-thread fd attribution) - see Appendix E for the scripts
python3 analyze.py echo.strace
python3 read_table.py echo.strace
python3 analyze.py yes1.strace
python3 poll_timeouts.py yes1.strace
```

## Appendix C — Environment and tooling

* **Toolchain (observed):**

```text
Python 3.13.7
go version go1.24.4 linux/amd64
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

* **Tracer:** `strace` with `-f` (follow threads — the reader is a non-main thread), `-yy` (annotate
  fds with backing paths), `-tt` (µs wall-clock timestamps), `-T` (per-call durations),
  `-e trace=read,poll` (restrict to the read loop), and `-c` (per-syscall counts, for the aggregate).
* **Input injection:** `xdotool type`/`key` against kitty's X11 window id.
* **Introspection:** `ps`, `/proc/<pid>/{status,cmdline,environ,fd}`, `getent`, `xprop`.

## Appendix D — Repository integrity, cleanup, and provenance

**No source file was modified.** The only change versus the pristine kitty source baseline
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` is the addition of this one document under
`blitzy/documentation/`. Verbatim git evidence:

```text
branch: blitzy-81322a7e-8c21-4921-ab8a-068d5c584657
HEAD:   dcddab9db8b74c896adc595f2dd0d6f5862521b5
baseline: 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
--- git status --porcelain (empty = clean) ---
--- git diff --name-status vs baseline ---
A	blitzy/documentation/kitty_815df1e210e0.md
--- git diff --check vs baseline (empty = clean) ---
--- non-doc files changed vs baseline ---
0
```

`git status --porcelain` is empty (clean working tree); `git diff --name-status` versus the baseline
lists only `A  blitzy/documentation/kitty_815df1e210e0.md`; `git diff --check` is empty (no
whitespace/conflict errors); and the count of non-document files changed is `0`.

**ptrace / kernel settings — nothing was modified.** Tracing was performed by a root
(`CAP_SYS_PTRACE`) strace, which bypasses the kernel `yama` `ptrace_scope` gate, so **no `sysctl`
value was changed** to enable it; the ambient `ptrace_scope` was left as found. Tracing only
observes the target.

**Cleanup.** The build artifacts (gitignored) and all temporary observation files — the strace
captures, the extracted trace slices, and the analyzer scripts under the working directory — are
transient and are removed after the investigation; they are not part of the repository. The
analyzer *source* is preserved in Appendix E so the results remain reproducible from this document.

## Appendix E — Analyzer source (so every statistic is recomputable)

The Q3/Q4 statistics in this document are produced by three small standard-library Python scripts.
Their full source is embedded here so the numbers above can be recomputed from this document alone:
save each script, capture a trace with the command in its docstring, and run it. All three parse the
strace format described in the Methodology (pairing `<unfinished ...>`/`<... read resumed>` halves
per thread, auto-detecting the master fd as the descriptor annotated `/dev/pts/ptmx`).

### `analyze.py` — authoritative per-fd read attribution, distribution, histogram, EAGAIN-by-fd

```python
#!/usr/bin/env python3
"""
analyze.py - authoritative per-fd read attribution for a kitty PTY strace.

Input: an strace file produced with
    strace -f -yy -tt -T -e trace=read,poll -p <kitty_pid> -o FILE
(-f follows every thread; -yy annotates each fd with its backing object;
 -tt gives microsecond wall-clock timestamps; -T gives per-call durations.)

What it does:
  * Reconstructs read() calls that strace split into "<unfinished ...>" /
    "<... read resumed>" halves (the fd is on the unfinished half; the
    requested size + return value are on the resumed half), pairing them
    per-thread (TID).
  * Auto-detects the PTY *master* fd as the descriptor whose -yy annotation
    contains "/dev/pts/ptmx".
  * Reports, for the master fd: total reads, data-returning reads, EAGAIN and
    zero-length counts, the active time window (first..last read timestamp),
    read frequency, the requested-size multiset, the returned-bytes
    distribution (min/median/mean/max + a log2 histogram), total bytes and
    throughput, and which thread(s) issued the reads.
  * Also prints a per-(fd,errno) read-error table so EAGAIN can be attributed
    to the exact descriptor it occurred on.

Usage:  python3 analyze.py FILE.strace [--fd N]
Only the Python standard library is used.
"""
import re
import sys
import statistics

# TID  HH:MM:SS.uuuuuu  <rest of line>
LINE = re.compile(r'^(\d+)\s+(\d+):(\d+):(\d+)\.(\d+)\s+(.*)$')
# fd number immediately after "read(" on an inline or unfinished read
READ_FD = re.compile(r'^read\((\d+)<')
# "<ann>" for a given fd anywhere on the line (to learn each fd's backing path)
FD_ANN = re.compile(r'\b(\d+)<([^>]*(?:<[^>]*>)?[^>]*)>')
# tail of a completed/resumed read: ", <count>) = <ret>[ ERRNO] [<dur>]
READ_TAIL = re.compile(r',\s*(\d+)\)\s*=\s*(-?\d+)(?:\s+(E[A-Z]+))?')


def to_sec(h, m, s, us):
    return int(h) * 3600 + int(m) * 60 + int(s) + int(us) / 1_000_000.0


def main():
    path = sys.argv[1]
    force_fd = None
    if '--fd' in sys.argv:
        force_fd = int(sys.argv[sys.argv.index('--fd') + 1])

    pending = {}                 # TID -> (fd, start_ts) for an unfinished read
    fd_paths = {}                # fd -> backing annotation (first seen)
    reads = []                   # (fd, count, ret, errno, ts, tid)

    with open(path, 'r', errors='replace') as fh:
        for raw in fh:
            m = LINE.match(raw)
            if not m:
                continue
            tid = m.group(1)
            ts = to_sec(m.group(2), m.group(3), m.group(4), m.group(5))
            rest = m.group(6)

            for fdnum, ann in FD_ANN.findall(rest):
                fd_paths.setdefault(int(fdnum), ann)

            if rest.startswith('read('):
                fm = READ_FD.match(rest)
                fd = int(fm.group(1)) if fm else None
                if rest.rstrip().endswith('<unfinished ...>'):
                    pending[tid] = (fd, ts)          # wait for the resumed half
                    continue
                tm = READ_TAIL.search(rest)
                if fd is not None and tm:
                    reads.append((fd, int(tm.group(1)), int(tm.group(2)),
                                  tm.group(3), ts, tid))
            elif rest.startswith('<... read resumed>'):
                fd, start_ts = pending.pop(tid, (None, ts))
                tm = READ_TAIL.search(rest)
                if fd is not None and tm:
                    reads.append((fd, int(tm.group(1)), int(tm.group(2)),
                                  tm.group(3), start_ts, tid))

    # pick the master fd = the one backed by /dev/pts/ptmx
    master = force_fd
    if master is None:
        for fd, ann in sorted(fd_paths.items()):
            if 'ptmx' in ann:
                master = fd
                break
    if master is None:
        print('no /dev/pts/ptmx fd found; fds seen:', sorted(fd_paths))
        return

    mr = [r for r in reads if r[0] == master]
    data = [r for r in mr if r[2] > 0]
    eagain = sum(1 for r in mr if r[3] == 'EAGAIN')
    zero = sum(1 for r in mr if r[2] == 0)
    sizes = [r[2] for r in data]
    tids = {}
    for r in mr:
        tids[r[5]] = tids.get(r[5], 0) + 1

    print(f'MASTER fd={master} ({fd_paths.get(master,"?")})')
    print(f'  reads total(incl err/0)={len(mr)}  data-returning={len(data)}')
    print(f'  fd{master} EAGAIN={eagain}  fd{master} zero-length={zero}')
    if data:
        tmin = min(r[4] for r in data)
        tmax = max(r[4] for r in data)
        window = tmax - tmin
        total = sum(sizes)
        print(f'  window={window:.3f}s  freq={len(data)/window:.0f} reads/s'
              if window > 0 else '  window=0')
        print(f'  bytes/read: min={min(sizes)} median={int(statistics.median(sizes))} '
              f'mean={statistics.mean(sizes):.1f} max={max(sizes)}')
        print(f'  total bytes={total} ({total/1048576:.2f} MiB)'
              + (f'  throughput={total/1048576/window:.2f} MiB/s' if window > 0 else ''))
        print(f'  window boundaries: first={tmin:.6f}s last={tmax:.6f}s')
        print(f'  requested sizes (size:count): '
              + ', '.join(f'{s}:{c}' for s, c in sorted(
                  {r[1]: sum(1 for x in mr if x[1] == r[1]) for r in mr}.items())))
        # log2 histogram of returned bytes
        buckets = {}
        for n in sizes:
            b = n.bit_length() - 1 if n > 0 else 0     # floor(log2 n)
            buckets[b] = buckets.get(b, 0) + 1
        print('  returned-size histogram [2^b .. 2^(b+1)) : count]:')
        for b in sorted(buckets):
            lo, hi = 1 << b, 1 << (b + 1)
            print(f'    [{lo:>7} .. {hi:>7}) : {buckets[b]}')
    print(f'  reader TID(s): {tids}')

    # per-(fd,errno) read-error attribution (all fds)
    errs = {}
    for fd, cnt, ret, errno, ts, tid in reads:
        if errno:
            errs[(fd, errno)] = errs.get((fd, errno), 0) + 1
    if errs:
        print('  read errors by (fd, errno):')
        for (fd, errno), c in sorted(errs.items()):
            tag = ' (PTY master)' if fd == master else ''
            print(f'    fd={fd}{tag} {errno}: {c}')
    print(f'    fd={master} (PTY master) errors: {sum(1 for r in mr if r[3])}')


if __name__ == '__main__':
    main()
```

### `read_table.py` — ordered per-read dump of the master fd (used for the Q3 16-read table)

```python
#!/usr/bin/env python3
"""
read_table.py - ordered per-read dump of the PTY master fd from a kitty strace.

Pairs "<unfinished ...>"/"<... read resumed>" halves per TID (the fd is on the
unfinished half; the requested size + return are on the resumed half), keeps
only reads on the /dev/pts/ptmx master fd, and prints them in issue order as a
Markdown table:  # | t_rel(ms) | requested(bytes) | returned(bytes) | TID
t_rel is milliseconds since the first master read in the trace.

Usage:  python3 read_table.py FILE.strace [--fd N]
Standard library only.
"""
import re
import sys

LINE = re.compile(r'^(\d+)\s+(\d+):(\d+):(\d+)\.(\d+)\s+(.*)$')
READ_FD = re.compile(r'^read\((\d+)<')
FD_ANN = re.compile(r'\b(\d+)<([^>]*(?:<[^>]*>)?[^>]*)>')
READ_TAIL = re.compile(r',\s*(\d+)\)\s*=\s*(-?\d+)(?:\s+(E[A-Z]+))?')


def to_sec(h, m, s, us):
    return int(h) * 3600 + int(m) * 60 + int(s) + int(us) / 1_000_000.0


def main():
    path = sys.argv[1]
    force_fd = int(sys.argv[sys.argv.index('--fd') + 1]) if '--fd' in sys.argv else None
    pending, fd_paths, reads = {}, {}, []
    with open(path, 'r', errors='replace') as fh:
        for raw in fh:
            m = LINE.match(raw)
            if not m:
                continue
            tid, ts, rest = m.group(1), to_sec(*m.group(2, 3, 4, 5)), m.group(6)
            for fdnum, ann in FD_ANN.findall(rest):
                fd_paths.setdefault(int(fdnum), ann)
            if rest.startswith('read('):
                fm = READ_FD.match(rest)
                fd = int(fm.group(1)) if fm else None
                if rest.rstrip().endswith('<unfinished ...>'):
                    pending[tid] = (fd, ts)
                    continue
                tm = READ_TAIL.search(rest)
                if fd is not None and tm:
                    reads.append((ts, fd, int(tm.group(1)), int(tm.group(2)),
                                  tm.group(3), tid))
            elif rest.startswith('<... read resumed>'):
                fd, start_ts = pending.pop(tid, (None, ts))
                tm = READ_TAIL.search(rest)
                if fd is not None and tm:
                    reads.append((start_ts, fd, int(tm.group(1)), int(tm.group(2)),
                                  tm.group(3), tid))
    master = force_fd
    if master is None:
        for fd, ann in sorted(fd_paths.items()):
            if 'ptmx' in ann:
                master = fd
                break
    mr = sorted([r for r in reads if r[1] == master], key=lambda r: r[0])
    if not mr:
        print('no master reads found')
        return
    t0 = mr[0][0]
    print('| # | t_rel (ms) | requested (bytes) | returned (bytes) | TID |')
    print('|---|-----------:|------------------:|-----------------:|-----|')
    for i, (ts, fd, req, ret, errno, tid) in enumerate(mr, 1):
        rv = f'{ret}' if errno is None else f'{ret} {errno}'
        print(f'| {i} | {(ts - t0) * 1000:.3f} | {req} | {rv} | {tid} |')
    total = sum(r[3] for r in mr if r[4] is None and r[3] > 0)
    print(f'\nSUM returned bytes = {total} across {len(mr)} reads on fd {master}')


if __name__ == '__main__':
    main()
```

### `poll_timeouts.py` — distribution of the `poll()` timeout argument (the `input_delay` cadence)

```python
#!/usr/bin/env python3
"""
poll_timeouts.py - distribution of the poll(2) timeout argument on kitty's
I/O thread from a strace. kitty arms a *timed* poll while the input_delay
countdown is active and a *blocking* (timeout = -1) poll otherwise
(child-monitor.c: timed poll L1509, blocking poll L1512). This shows how
often each timeout value is used during a sustained stream.

Usage: python3 poll_timeouts.py FILE.strace
Standard library only.
"""
import re
import sys

# poll([ ...fd array... ], NFDS, TIMEOUT) = RET
POLL = re.compile(r'poll\(\[.*\],\s*(\d+),\s*(-?\d+)\)\s*=')

counts = {}
with open(sys.argv[1], 'r', errors='replace') as fh:
    for line in fh:
        m = POLL.search(line)
        if m:
            timeout = int(m.group(2))
            counts[timeout] = counts.get(timeout, 0) + 1

total = sum(counts.values())
print(f'poll() calls with a parseable timeout arg: {total}')
print('timeout(ms):count  (-1 = block forever)')
for t in sorted(counts):
    print(f'  {t:>4} : {counts[t]}')
```

Running `analyze.py echo.strace` reproduces the Q3 summary (16 reads, 617 bytes, TID 200017);
`read_table.py echo.strace` reproduces the 16-read table (sum 617); `analyze.py yes1.strace`
reproduces the Run 1 summary (56,817 reads, 72.53 MiB, TID 200017); and `poll_timeouts.py` on any
`yes` trace reproduces the timeout distribution — all as printed in the sections above.
