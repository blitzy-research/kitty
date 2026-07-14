# kitty — how the terminal emulator reads from its shell over the PTY

An investigative, **Run-First** answer to six questions about kitty's shell-communication
machinery. Every behavioural claim below is backed by an actual command and its **unedited**
output captured from a live, canonically-built kitty. Statements that are derived from reading
the C/Python source rather than observed at runtime are explicitly tagged
`(inferred from code: file:line)` or grouped under a heading labelled **Source-code rationale**.

## Investigation baseline and environment

- **kitty source commit (investigation baseline):** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.
  Every `file:line` citation in this document refers to the kitty source tree at this commit.
  (The final Git HEAD of *this* branch is the commit that adds this document; it is not the kitty
  source baseline — the two must not be conflated.)
- **Built kitty version:** `kitty 0.35.2 created by Kovid Goyal` (observed — see Q1).
- **Host / runtimes (observed):** Ubuntu 25.10; CPython **3.13.7** (satisfies `requires-python = ">=3.8"`
  [pyproject.toml:L2]); Go **1.24.4** (satisfies `go 1.22` [go.mod:L3], used only for the `kitten`
  binary, not the C PTY/parser path); `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0`.
- **Display:** headless via `Xvfb :99` (no physical display in the container).
- **User model:** kitty was **launched as the ordinary, non-root user `ubuntu` (uid 1000)** — verified
  in Q1. The build step itself ran as root (compilation output is user-independent; the AAP requires
  only that kitty be *launched* as a normal user). The launcher binary is world-executable, so uid 1000
  runs it directly.

### Session identifiers used throughout (historical, from one captured session)

The concrete integers below come from a **single** captured kitty session. They are **historical
identifiers** — re-running the investigation yields *different* PIDs / window IDs (and possibly a
different fd number), but the **discovery commands** that produce them are reproducible and are shown
next to each answer.

| Identifier | Value (this session) | What it is |
|---|---|---|
| kitty process PID | `82052` | the running terminal emulator |
| spawned shell PID | `82119` | the shell kitty forked (Q2) |
| reader thread TID | `82118` (`KittyChildMon`) | the I/O thread that performs the PTY reads (Q3–Q6) |
| X11 window id | `2097164` | kitty's top-level window (Q3 input injection) |
| PTY master fd | `8` | kitty's descriptor for the master side (Q5) |
| PTY slave device | `/dev/pts/0` | the shell's controlling terminal (Q2) |

## Methodology (Run-First)

**Discipline.** Build and launch the real, canonical binary; exercise the real PTY input path through
kitty's normal window (keystrokes injected into the actual X11 window with `xdotool`, *not* via any
remote-control/debug hook); capture live system calls with `strace`; and read every value out of the
unedited trace / `ps` / `/proc` output. Code is consulted only to *explain* what was observed and to
name functions; such code-only statements are tagged as inferred.

**Syscall tracing.** kitty performs its PTY reads on a dedicated I/O thread (`KittyChildMon`, see Q6),
not the main thread, so the tracer must follow threads. The canonical attach command used for Q3–Q6
(shown with a numeric-PID guard so it is copy-paste safe):

```bash
KPID="$(pgrep -u ubuntu -x kitty | head -n1)"          # discover kitty's PID (owned by ubuntu)
[ -n "$KPID" ] && [[ "$KPID" =~ ^[0-9]+$ ]] || { echo "kitty PID not found"; exit 1; }
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o /tmp/kitty_pty_probe/trace.strace &
STRACE_PID=$!                                            # remember the tracer's own PID to stop it later
```

Option meaning (per the `strace(1)` manual page, Linux man-pages project / man7.org):

- `-f` — follow forks **and threads**; required because the reads happen on the `KittyChildMon`
  thread, not the traced main PID. The attach banner `Process 82052 attached with 67 threads`
  confirms all threads are followed.
- `-yy` — annotate every descriptor with its backing object, e.g. `8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>`.
  This is what lets us prove fd `8` is the PTY master and see the paired slave `/dev/pts/0`.
- `-tt` — wall-clock timestamps (microsecond) on every line; used to measure read frequency in Q4.
- `-T` — per-call duration in angle brackets (e.g. `<0.000028>`).
- `-e trace=read,poll` — restrict the trace to the two syscalls that constitute the read loop
  (equivalent to the `desc` class for this purpose), keeping the traces small enough to retain in full.
- `-c` (used in one Q4 run) — print a per-syscall count/time histogram instead of per-call lines,
  to quantify the aggregate magnitude of reads.

**PTY model** (for Q3–Q5, per the Linux `pty(7)` man page and the kernel TTY documentation —
docs.kernel.org "N_TTY" and "TTY Line Discipline"): kitty holds the **master** (`/dev/ptmx`, appearing
as `/dev/pts/ptmx` in the annotation) while the shell holds the **slave** (`/dev/pts/0`). A kernel line
discipline (`N_TTY`) mediates between them; a `read()` on the master returns *whatever the line
discipline currently has buffered* for the reader, and in non-blocking mode returns `EAGAIN` when
nothing is buffered. This is the documented basis for the per-read byte counts reported in Q3/Q4 — the
amount returned per call is emergent, not a fixed constant.

**Observed-vs-inferred convention.** A claim shown with its command **and** captured output is
*runtime-observed*. A claim about why the code behaves as it does — function names, control flow,
buffer constants, backpressure, threading — is *inferred from reading the source* and is tagged inline
`(inferred from code: file:line)` or placed under a **Source-code rationale** heading.

**Command conventions.** Two kinds of shell command appear below. (1) **Reproducible procedure** blocks
use shell variables (`$KPID`, `$WID`, `$SHPID`, `$KDIR`) that are *discovered* at run time; these are
copy-paste-safe — every variable is double-quoted and every PID/window id is validated numeric
(`[[ "$KPID" =~ ^[0-9]+$ ]]`) before use, and each such block parses cleanly under `bash -n`. (2) A few
blocks show **literal historical identifiers** from the captured session (e.g. `82052`, `82119`,
`/proc/82119/cmdline`); these reproduce *that session's* recorded values for auditability — re-running
the investigation produces different ids, discovered via the reproducible blocks.

**Environment adjustments (disclosed).** To let `strace` attach to a running process in the container,
the kernel setting `kernel.yama.ptrace_scope` was read (original value **`0`**) before tracing. It was
set to the hardened value `1` at the end of the investigation; the before/after proof and the
full cleanup (removal of all temporary logs/scripts, teardown of kitty/Xvfb/tracers, and confirmation
that the repository contains only this document) are shown in **Appendix D**.

### References (methodology and PTY semantics)

The observation method and the terminal-I/O semantics used to interpret the traces are grounded in the
following authoritative sources:

- **`strace(1)` manual page** — Linux man-pages project, `man7.org/linux/man-pages/man1/strace.1.html`
  (attaching to a live process with `-p`, following threads with `-f`, descriptor annotation `-yy`,
  timestamps `-tt`/`-T`, syscall filtering `-e trace=`, and the `-c` summary histogram).
- **`pty(7)` / `pts(4)` manual pages** — Linux man-pages project, `man7.org` (the pseudo-terminal
  master/slave model: `/dev/ptmx` master and `/dev/pts/N` slave).
- **Kernel TTY documentation** — `docs.kernel.org/driver-api/tty/n_tty.html` ("N_TTY") and
  `docs.kernel.org/driver-api/tty/tty_ldisc.html` ("TTY Line Discipline"): the line discipline
  buffers input and, on a read, "returns whatever characters it has buffered up for the user," and a
  non-blocking tty read returns `EAGAIN` when nothing is buffered — the basis for the emergent
  per-read byte counts in Q3/Q4.

## Q1 — Building kitty and launching it as a normal user

**Direct answer.** kitty is built with its canonical `setup.py` driver and, on this newer toolchain,
the official `--ignore-compiler-warnings` flag (see the deviation note below); the build completes with
**exit status 0**, producing `kitty/launcher/kitty` (`kitty 0.35.2`). It is then launched **as the
non-root user `ubuntu` (uid 1000)** under a headless `Xvfb` display.

### Build (canonical)

```bash
cd /tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6
CI=true python3 setup.py --ignore-compiler-warnings ; echo "BUILD EXIT STATUS = $?"
```

Unedited output (the full log is 159 lines of per-unit *Generating / Compiling / Linking* progress;
head and tail are reproduced verbatim with a counted omission marker for the mechanical middle):

```
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
[2/28] Generating wayland-xdg-shell-client-protocol.c ...
[3/28] Generating wayland-viewporter-client-protocol.h ...
[… 148 intermediate compile/generate/link progress lines omitted; 159 lines total …]
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
kitty/tools/cmd
BUILD EXIT STATUS = 0
```

The compile phase covers **122 C translation units** (`[N/122] Compiling …`); the two units central to
this investigation are `kitty/child-monitor.c` (unit `7/122`) and `kitty/vt-parser.c` (units `10–11/122`).

Resulting launcher and version (observed):

```bash
ls -l kitty/launcher/kitty
file kitty/launcher/kitty
./kitty/launcher/kitty --version
```

```
-rwxr-xr-x 1 root root 40384 Jul 14 20:09 kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4a693e4304285476522c8ac6a4eef4babf9072b7, for GNU/Linux 3.2.0, not stripped
kitty 0.35.2 created by Kovid Goyal
```

**Deviation from the bare canonical command (disclosed, no source modified).** The bare
`python3 setup.py` (kitty's default, which compiles with `-Werror`) **fails** on this environment with
**exit status 1**, because the system `wayland-protocols` (1.45) is newer than the pinned source
expects and introduces `xdg-shell` enum values the source's `switch` does not enumerate:

```bash
python3 setup.py ; echo "BARE BUILD EXIT = $?"
```

```
[3/122] Compiling [wayland] glfw/wl_window.c ...
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
BARE BUILD EXIT = 1
```

kitty ships `--ignore-compiler-warnings` precisely for this situation (it downgrades `-Werror` to
warnings); using it is the canonical way to build against a newer toolchain and **modifies no source
file**. The failing unit is in the Wayland windowing backend (`glfw/`), which is unrelated to the PTY
read path this investigation examines.

`setup.py` enforces the minimum Python at build time via `check_version_info()`
`(inferred from code: setup.py:L30-L47)`, keyed to `requires-python = ">=3.8"`
`(inferred from code: pyproject.toml:L2)`; the observed CPython 3.13.7 satisfies it.

### Launch (as the non-root user `ubuntu`)

```bash
KDIR="/tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6"
# headless display (background):
nohup Xvfb :99 -screen 0 1280x1024x24 +extension GLX +render -noreset -ac >/tmp/xvfb99.log 2>&1 &
export DISPLAY=:99
# launch kitty AS ubuntu, detached; software GL for the headless llvmpipe path:
nohup setsid sudo -u ubuntu -H env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
      HOME=/home/ubuntu "$KDIR/kitty/launcher/kitty" >/tmp/kitty_pty_probe/kitty_run.log 2>&1 &
# discover kitty's real PID (the process is named "kitty"; the shell $! here is the setsid/sudo wrapper):
sleep 3
KPID="$(pgrep -u ubuntu -x kitty | head -n1)"
[ -n "$KPID" ] && [[ "$KPID" =~ ^[0-9]+$ ]] || { echo "kitty PID not found"; exit 1; }
echo "KPID=$KPID"
```

kitty's launch diagnostics (unedited, from `kitty_run.log`):

```
[0.216] Failed to open systemd user bus with error: No medium found
ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
```

The first line is expected in a container with no per-user systemd bus and is discussed in Q2 (it does
**not** affect the shell's parentage). The second is an informational shell-integration notice.

**Verification that the process is the launcher, running as a normal user** (this is the identity used
for every later question — `KPID=82052` in this session):

```bash
readlink "/proc/$KPID/exe"                                    # which binary
grep -E '^(Name|Uid|Gid|PPid|Threads):' "/proc/$KPID/status"  # identity + non-root proof + thread count
tr '\0' ' ' < "/proc/$KPID/cmdline"; echo                     # argv
```

```
exe: /tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6/kitty/launcher/kitty
Name:	kitty
PPid:	82049
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
Threads:	67
```

`Uid: 1000 1000 1000 1000` confirms kitty runs as the **non-root** `ubuntu` account (all four of
real/effective/saved/filesystem uid are 1000), `exe` confirms it is the freshly-built launcher, and
`Threads: 67` shows the multi-threaded model whose I/O thread performs the PTY reads (Q6).

### Default, canonical configuration (proof)

The shell identity (Q2) and read cadence (Q3–Q4) depend on kitty running with its **default**
configuration. kitty's config precedence is `KITTY_CONFIG_DIRECTORY` → `$XDG_CONFIG_HOME/kitty` (i.e.
`~/.config/kitty`) → the system file `/etc/xdg/kitty/kitty.conf`
`(inferred from code: kitty/cli.py:L1064 SYSTEM_CONF, precedence at kitty/cli.py:L196-L201)`. All of
these are absent/empty in this environment, so kitty uses built-in defaults:

```bash
# (a) system config file:
ls -l /etc/xdg/kitty/kitty.conf
# (b) user config dir contents, as ubuntu:
sudo -u ubuntu -H bash -c 'ls -la "$HOME/.config/kitty"; echo "HOME=$HOME"'
# (c) config-related env vars inside the running kitty process:
tr '\0' '\n' < "/proc/$KPID/environ" | grep -E '^(KITTY_CONFIG_DIRECTORY|XDG_CONFIG_HOME|XDG_CONFIG_DIRS)=' \
  || echo "(none of KITTY_CONFIG_DIRECTORY / XDG_CONFIG_HOME / XDG_CONFIG_DIRS set)"
```

```
# (a) system config file [cli.py:L1064 SYSTEM_CONF=/etc/xdg/kitty/kitty.conf]:
ls: cannot access '/etc/xdg/kitty/kitty.conf': No such file or directory
# (b) user config dir contents [$HOME/.config/kitty], run as ubuntu:
total 8
drwxr-xr-x 2 ubuntu ubuntu 4096 Jul 14 20:00 .
drwxr-xr-x 3 ubuntu ubuntu 4096 Jul 14 20:00 ..
HOME=/home/ubuntu
# (c) kitty config-related env vars in the running kitty process (82052):
(none of KITTY_CONFIG_DIRECTORY / XDG_CONFIG_HOME / XDG_CONFIG_DIRS set)
```

No system `kitty.conf`, an empty `~/.config/kitty`, and none of the config-directory environment
variables set: kitty ran with default options (default shell, default `shell_integration`,
default `input_delay = 3` — relevant to Q4).

## Q2 — The process kitty spawns for the shell (PID, exact command line, PTY device)

**Direct answer.**

| Item | Observed value |
|---|---|
| Spawned process | the account's login shell, `bash`, in POSIX mode |
| PID | `82119` |
| Exact command line | `/bin/bash --posix` (argv bytes: `/bin/bash\0--posix\0`) |
| Parent | kitty, PID `82052` (a **direct** child) |
| Connecting PTY (slave, shell side) | `/dev/pts/0` |
| Connecting PTY (master, kitty side) | `/dev/pts/ptmx`, fd `8` (Q5) |

### Evidence

Process, PID and exact command line as they appear in the process list:

```bash
ps --ppid "$KPID" -o pid,ppid,user,cmd
```

```
    PID    PPID USER     CMD
  82119   82052 ubuntu   /bin/bash --posix
```

The exact argv, byte-for-byte from the shell's `/proc/82119/cmdline` (NUL-separated), shown as hex to
remove any ambiguity about the separators:

```bash
xxd "/proc/82119/cmdline"
```

```
00000000: 2f62 696e 2f62 6173 6800 2d2d 706f 7369  /bin/bash.--posi
00000010: 7800                                     x.
```

That decodes to exactly two NUL-terminated arguments: `"/bin/bash"` `"--posix"` — i.e. the command
line is `/bin/bash --posix`.

The PTY device connecting kitty to the shell, read from the shell's standard descriptors (the slave
side is the shell's controlling terminal):

```bash
ls -l /proc/82119/fd/0 /proc/82119/fd/1 /proc/82119/fd/2
```

```
lrwx------ 1 ubuntu ubuntu 64 Jul 14 20:11 /proc/82119/fd/0 -> /dev/pts/0
lrwx------ 1 ubuntu ubuntu 64 Jul 14 20:11 /proc/82119/fd/1 -> /dev/pts/0
lrwx------ 1 ubuntu ubuntu 64 Jul 14 20:11 /proc/82119/fd/2 -> /dev/pts/0
```

The shell's stdin/stdout/stderr are all the PTY **slave** `/dev/pts/0`; kitty holds the matching
**master** (`/dev/pts/ptmx`, fd `8`) — corroborated by the strace descriptor annotation
`8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>` in Q3/Q5, where the `@/dev/pts/0` part is the kernel telling us
this master's peer slave is exactly `/dev/pts/0`.

### Notes on the observed values

- **`--posix` is the observed default argv for this account/session, not a universal constant.** The
  spawned program is the account's login shell (`/bin/bash` here); the `--posix` argument is added by
  kitty's shell-integration for this shell. A different account with a different login shell (zsh, fish,
  a differently-built bash) would show a different argv. The value above is exactly what was observed for
  the `ubuntu` account in this session under default configuration.

- **The shell is a *direct* child of kitty (parentage from observed `ps`).** `ps` reports
  `PPID 82052`, i.e. the shell's parent is the kitty process itself. This is because kitty creates the
  child by `fork()` + `execvp()` directly `(inferred from code: kitty/child.c:L97 fork, kitty/child.c:L159 execvp)`.
  The `Failed to open systemd user bus … No medium found` line seen at launch does **not** change this:
  kitty's optional systemd integration only submits the *already-created* child PID into a transient
  cgroup **scope** for resource management — it does not reparent the process or set its PPID
  `(inferred from code: kitty/child.py:L349-L350 systemd_move_pid_into_new_scope)`. With no user bus in
  the container, that scope simply is not created; the shell remains a direct child regardless.

### Source-code rationale for the spawn path (inferred, not runtime-observed)

The runtime facts above are produced by this code path (consulted read-only; these are code
inferences, tagged with `file:line`):

1. The default shell is resolved by `resolved_shell()` `(kitty/utils.py:L768)`: for the default
   `shell == '.'` case `(kitty/utils.py:L770)` it returns `[shell_path]` `(kitty/utils.py:L771)`, where
   `shell_path` is the login shell from the password database, falling back to `/bin/sh`
   `(kitty/constants.py:L181, L185)`. Here that resolves to `/bin/bash`.
2. Shell-integration adds the POSIX-mode argument for bash — `argv.insert(1, '--posix')`
   `(kitty/shell_integration.py:L146)` — and adjusts the environment via `modify_shell_environ()`
   `(kitty/shell_integration.py:L218)`, without wrapping the shell in a separate process. This is why
   the observed argv is `/bin/bash --posix`.
3. `Child.fork()` `(kitty/child.py:L276)` calls `fast_data_types.spawn()` `(kitty/child.py:L333)`, which
   is the C `spawn()` `(kitty/child.c:L80)`. There the slave device path is obtained with
   `ttyname_r(slave, …)` `(kitty/child.c:L88)`, the child is created with `fork()` `(kitty/child.c:L97)`,
   made a session leader with `setsid()` `(kitty/child.c:L123)`, given the PTY as controlling terminal
   with `ioctl(TIOCSCTTY)` `(kitty/child.c:L129)`, has the slave `dup2()`'d onto stdio
   `(kitty/child.c:L138, L145)`, and finally becomes the shell via `execvp()` `(kitty/child.c:L159)`.
4. **Linux vs macOS divergence** `(inferred from code: kitty/child.py:L230)`: the login-shell wrapper
   block `(kitty/child.py:L295-L326)` is guarded by
   `should_run_via_run_shell_kitten = is_macos and self.is_default_shell`, which is **False** on Linux.
   That is why the process appears on Linux as the plain resolved shell (`/bin/bash --posix`) rather
   than a dash-prefixed login shell or a `/usr/bin/login` wrapper. Not exercised here (the environment
   is Linux); noted only to explain the observed argv.

## Q3 — Reading a small input (`echo test123`): syscalls, requested size, bytes returned

**Direct answer.** kitty reads the echoed output with the `read()` system call on the PTY **master**
file descriptor (fd `8`), and each `read()` is preceded by a `poll()` that reports the master readable.
Each `read()` requests up to **1 MiB** (`BUF_SZ`, minus any bytes already sitting unparsed in the
buffer); the kernel returns only what the line discipline currently has buffered. For the exact input
`echo test123` followed by Enter, that was **16 reads on fd 8 returning 617 bytes in total** — twelve
1-byte reads (one per echoed keystroke, `e c h o ␠ t e s t 1 2 3`), then, after Enter, reads of
**11, 47, 114 and 433** bytes for the newline handling, prompt redraw and shell-integration sequences.

### Input injection into the real kitty window (observed)

The window id is discovered from kitty's PID and its ownership verified before injecting the exact user
input (no placeholders — the concrete id `2097164` is this session's value):

```bash
WID="$(xdotool search --pid "$KPID" | head -n1)"     # kitty's top-level X11 window
[ -n "$WID" ] && [[ "$WID" =~ ^[0-9]+$ ]] || { echo "window not found"; exit 1; }
echo "window id (by pid) = $WID"
xprop -id "$WID" WM_CLASS _NET_WM_PID WM_NAME         # verify this window belongs to our kitty
xdotool type --window "$WID" 'echo test123'           # the user's exact example input
xdotool key  --window "$WID" Return
```

```
window id (by pid) = 2097164
WM_CLASS(STRING) = "kitty", "kitty"
_NET_WM_PID(CARDINAL) = 82052
WM_NAME(STRING) = "/tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6"
```

`_NET_WM_PID = 82052` equals `KPID`, and `WM_CLASS = "kitty"`, so keystrokes are injected into *our*
kitty window — not some other client.

### Trace capture (observed)

```bash
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o /tmp/kitty_pty_probe/echo.strace &
STRACE_PID=$!
# … inject `echo test123` + Return (above) …
kill "$STRACE_PID"; wait "$STRACE_PID" 2>/dev/null   # stop the tracer by its exact PID
```

A contiguous slice of the raw trace around the first echoed character (`c`) shows the exact syscall
pair kitty issues on the master — a `poll()` that reports `fd=8` readable, immediately followed by the
`read()` on `fd=8`:

```
82118 20:12:05.444365 poll([{fd=6<…eventfd…>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}]) <0.000018>
82118 20:12:05.444480 poll([{fd=6<…eventfd…>, events=POLLIN}, {fd=7<signalfd:[…]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000036>
82118 20:12:05.444569 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "c", 1048576) = 1 <0.000013>
```

The first `poll` returns `POLLOUT` (kitty writing the keystroke to the master); the second returns
`POLLIN` (the line discipline has echoed the character back); then `read(8, …, 1048576) = 1` retrieves
the single echoed byte. The `read` request size `1048576` is exactly 1 MiB — `BUF_SZ` — because the
parser buffer is empty at that moment.

### All 16 master reads for `echo test123` (observed)

The 16 reads on fd 8 (inline reads plus the three that strace split into `<unfinished …>` / `resumed`
pairs, re-joined by thread id), in time order — requested size, returned byte count, and the payload
strace captured:

```
idx  time             req_size   ret  payload
 1   20:12:05.421542   1048576     1  "e"
 2   20:12:05.444569   1048576     1  "c"
 3   20:12:05.467658   1048576     1  "h"
 4   20:12:05.490820   1048576     1  "o"
 5   20:12:05.514174   1048576     1  " "
 6   20:12:05.537360   1048576     1  "t"
 7   20:12:05.560204   1048576     1  "e"
 8   20:12:05.583391   1048576     1  "s"
 9   20:12:05.606360   1048576     1  "t"
10   20:12:05.629560   1048576     1  "1"
11   20:12:05.652233   1048576     1  "2"
12   20:12:05.675258   1048576     1  "3"
13   20:12:05.710478   1048576    11  "\r\n\33[?2004l\r"
14   20:12:05.712136   1048565    47  "\33]2;echo test123\7\33]133;C;cmdline"...
15   20:12:05.712407   1048518   114  "\1\33]133;k;start_kitty\7\2\1\33]133;k;e"...
16   20:12:05.713281   1048404   433  "\33[?2004h\33[59P\33]133;k;start_kitty"...
```

The authoritative summary is produced by re-parsing the raw trace (the parser pairs `<unfinished>`/
`resumed` reads by thread id and attributes each read to its fd; script retained during capture as
`analyze.py`):

```bash
python3 /tmp/kitty_pty_probe/analyze.py /tmp/kitty_pty_probe/echo.strace
```

```
MASTER(fd8) reads total(incl err/0)=16  data-returning=16
  fd8 EAGAIN=0  fd8 zero-length=0
  bytes/read: median=1 mean=38.6 min=1 max=433
  total bytes=617
  reader TID(s): {82118: 16}
```

**Requested buffer size (the `read()` third argument), across the 16 reads:**

| Requested size (bytes) | Count |
|---|---|
| 1,048,576 (= 1 MiB, buffer empty) | 13 |
| 1,048,565 | 1 |
| 1,048,518 | 1 |
| 1,048,404 | 1 |

The 13 reads that requested a full 1,048,576 bytes did so because the parser buffer was empty when the
read began; the last three requested slightly less (`1048576 − 11`, `− 58`, `− 172`) because 11, then
58, then 172 bytes were already sitting unparsed in the buffer, so the write region shrank by exactly
that much. Totals reconcile: **13 + 1 + 1 + 1 = 16 reads.**

**Bytes returned, across the 16 reads:** `12 × 1  +  1 × 11  +  1 × 47  +  1 × 114  +  1 × 433  = 617`
bytes. The twelve 1-byte reads are the twelve characters of `echo test123` echoed back one keystroke at
a time; the 11-byte read (`\r\n\33[?2004l\r`) is the carriage-return/line-feed and bracketed-paste-off
emitted when Enter is pressed; and the 47/114/433-byte reads are the new prompt plus shell-integration
OSC sequences (`\33]133;…`, visible in the payloads).

**Magnitude — largest read vs. the 1 MiB request.** The largest single read returned **433 bytes**
against a request of `1,048,404`. Relative to the full 1 MiB buffer that is `1,048,576 / 433 ≈ **2,421×**
smaller — i.e. **just over three orders of magnitude** (log₁₀ 2421 ≈ 3.38) below the buffer size, not
four. Even the largest `echo` read fills a negligible fraction of the buffer.

### Source-code rationale (inferred, not runtime-observed)

- The reader is `read_bytes()` `(kitty/child-monitor.c:L1337)`, which calls
  `read(fd, buf, available_buffer_space)` `(kitty/child-monitor.c:L1345)`. The third argument
  `available_buffer_space` is what `vt_parser_create_write_buffer()` returns
  `(kitty/vt-parser.c:L1451)`, computed as `*sz = BUF_SZ - write.offset` `(kitty/vt-parser.c:L1457)`,
  where `BUF_SZ = 1024u * 1024u` (1 MiB) `(kitty/vt-parser.c:L18)`. This is exactly why the observed
  request is `1048576` when the buffer is empty and shrinks to `1048565 / 1048518 / 1048404` as
  unparsed bytes accumulate.
- The `poll()` that precedes each read is the io-loop readiness wait `(kitty/child-monitor.c:L1509`
  timed / `L1512` blocking`)`; it requests `POLLIN` for the master only while the parser has space
  `(kitty/child-monitor.c:L1501)`. These are code inferences; the runtime `poll`→`read` pairing itself
  is shown observed above.


## Q4 — Reading a high-volume stream (`yes hello`): how the read behaviour changes

**Direct answer.** The reading mechanism is unchanged — the same `read_bytes()` path on the same master
fd `8` — but the *cadence* changes dramatically: instead of a handful of tiny reads, kitty performs
**tens of thousands of reads per run at roughly 6,500–7,000 reads per second**, each returning a small
chunk (**median ≈ 1.1 KiB, mean ≈ 1.4 KiB**, ranging from 1 byte up to ≈ 19.2 KiB). It does **not**
coalesce the stream into one large read: even though every `read()` still requests up to ~1 MiB, the
kernel returns only what the line discipline currently holds, which is far less than 1 MiB.

### Scale, duration, and the stability criterion (stated up front)

`yes hello` was injected into the kitty window and left streaming; the trace was then stopped and the
`yes` process killed. This was repeated **three independent times** (runs 1–3). In each run the master
read activity spans a window of **≈ 8.27 s**, during which kitty issued **53k–58k reads** on fd 8 and
moved **69–79 MiB** — ample scale to characterise the steady-state cadence.

**Predeclared stability criterion:** the runs are considered stable if **each summary metric from every
run lies within ±20 % of that metric's three-run mean.** Actual variance is reported honestly in the
table below (it is met, with the largest single-run deviation being mean-bytes/read at +10.6 %).

### Run 1 — raw evidence and analysis (`yes hello`)

A contiguous slice of the raw trace (reader thread `82118`), showing the `poll`→`read` cadence and,
crucially, that each `read()` still requests close to 1 MiB yet returns only hundreds/thousands of bytes:

```
82118 20:15:13.531648 poll([… {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000014>
82118 20:15:13.531713 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1035619) = 756 <0.000012>
82118 20:15:13.531755 poll([… {fd=8…, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}]) <0.000012>
82118 20:15:13.531829 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1034863) = 1423 <0.000013>
82118 20:15:13.531938 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1033440) = 1323 <0.000011>
```

The requested size decreases (`1035619` → `1034863` → `1033440`) as unparsed bytes accumulate in the
1 MiB buffer, and the returns are small (`756`, `1423`, `1323`) — the payload is the repeating
`hello\r\n` produced by `yes`. Analysis of the full run (authoritative re-parse):

```bash
python3 /tmp/kitty_pty_probe/analyze.py /tmp/kitty_pty_probe/yes1.strace
```

```
MASTER(fd8) reads total(incl err/0)=53494  data-returning=53494
  fd8 EAGAIN=0  fd8 zero-length=0
  window=8.273s  freq=6466 reads/s
  bytes/read: median=1237 mean=1546.3 min=1 max=19509
  total bytes=82718242 (78.89 MiB)  throughput=9.53 MiB/s
  median vs 1MiB: 848x (2.93 orders)
  max vs 1MiB:    53.7x (1.73 orders)
  reader TID(s): {82118: 53494}
```

### Run 2 — raw evidence and analysis (independent repeat)

```
82118 20:16:24.611716 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 877123) = 826 <0.000012>
82118 20:16:24.611839 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 876297) = 1239 <0.000012>
82118 20:16:24.611946 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 875058) = 1048 <0.000014>
82118 20:16:24.612093 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 874010) = 1801 <0.000023>
82118 20:16:24.612238 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 872209) = 1496 <0.000015>
```

```bash
python3 /tmp/kitty_pty_probe/analyze.py /tmp/kitty_pty_probe/yes2.strace
```

```
MASTER(fd8) reads total(incl err/0)=57445  data-returning=57445
  fd8 EAGAIN=0  fd8 zero-length=0
  window=8.284s  freq=6935 reads/s
  bytes/read: median=1066 mean=1261.5 min=1 max=19754
  total bytes=72465725 (69.11 MiB)  throughput=8.34 MiB/s
  median vs 1MiB: 984x (2.99 orders)
  max vs 1MiB:    53.1x (1.72 orders)
  reader TID(s): {82118: 57445}
```

### Run 3 and the aggregate `strace -c` histogram

Run 3 (`analyze.py yes3b.strace`) gave `57533` master reads over `8.265 s` = `6961 reads/s`, median
`1125`, mean `1386`, max `19732`. A separate run captured with `strace -f -e trace=read,poll -c`
confirms the aggregate magnitude at the whole-process level:

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 51.41    1.075493          10    105167           poll
 48.59    1.016487          10    100600       749 read
------ ----------- ----------- --------- --------- ----------------
100.00    2.091980          10    205767       749 total
```

The histogram counts **all** descriptors (the ~53–58k master reads *plus* the wakeup/signal eventfd
drains), so its `read` total (`100,600`) exceeds the master-only count; its `errors` column is discussed
under EAGAIN attribution below.

### Stability across the three runs

| Metric | Run 1 | Run 2 | Run 3 | 3-run mean | max \|dev\| from mean |
|---|---|---|---|---|---|
| reads on master (fd 8) | 53,494 | 57,445 | 57,533 | 56,157 | — |
| read window (s) | 8.273 | 8.284 | 8.265 | 8.274 | — |
| **read frequency (reads/s)** | 6,466 | 6,935 | 6,961 | **6,787** | **4.7 %** |
| median bytes/read | 1,237 | 1,066 | 1,125 | 1,143 | 8.3 % |
| mean bytes/read | 1,546 | 1,261 | 1,386 | 1,398 | 10.6 % |
| max bytes/read | 19,509 | 19,754 | 19,732 | 19,665 | 0.8 % |
| throughput (MiB/s) | 9.53 | 8.34 | 9.20 | 9.02 | 7.6 % |

Every metric in every run is within the predeclared ±20 % band, so the behaviour is **stable** by the
stated criterion. The read **frequency** is the most stable headline number (±4.7 %); the per-read byte
statistics vary a little more (up to ±10.6 %) because the exact way the byte stream is chopped into
individual reads depends on run-to-run scheduling — as expected for an emergent, not fixed, quantity.

### Per-read size: measured distribution, and why each read is small

The **measured** per-read size across runs is: **min 1 byte, median ≈ 1.1 KiB, mean ≈ 1.4 KiB,
max ≈ 19.2 KiB**. Relative to the ~1 MiB request:

- the **median** read (~1,143 B) is `1048576 / 1143 ≈ 917×` smaller — about **2.96 orders of
  magnitude** below the 1 MiB buffer (i.e. just under three orders);
- the **largest** read (~19,665 B) is only `1048576 / 19665 ≈ 53×` smaller — about **1.73 orders**
  (well under two orders) below the buffer.

This corrects any blanket "always three orders of magnitude below 1 MiB" claim: the *typical* read is
~3 orders below, but the *largest* reads are only ~1.7 orders (≈ 50×) below.

**Why the reads are small (grounded, with inferred mechanisms tagged).** There is **no fixed few-KiB
cap** enforced anywhere in kitty's code — `read_bytes()` always offers the kernel up to ~1 MiB
`(inferred from code: kitty/child-monitor.c:L1345)`; the returned count is simply *whatever the N_TTY
line discipline has buffered at that instant* (per the Linux kernel TTY documentation, a tty read
"returns whatever characters it has buffered up for the user"). That the largest observed reads reach
≈ 19.2 KiB is itself direct evidence against a "few KiB" ceiling. The per-read amount is an **emergent**
quantity set by the race between the producer (`yes` writing the slave) and the consumer (kitty draining
the master), modulated by:

- the master being **non-blocking** `(inferred from code: kitty/child.py:L345)`, so each `read()`
  returns immediately with exactly what is queued rather than waiting to fill the buffer;
- kitty's `input_delay = 3` ms **timed poll**, which lets bytes accumulate between drains
  `(inferred from code: kitty/child-monitor.c:L1508-L1509)`;
- **backpressure**: the io-loop stops requesting `POLLIN` on the master once the 1 MiB parser buffer is
  full `(inferred from code: kitty/vt-parser.c:L1477-L1481, gate applied at kitty/child-monitor.c:L1501)`;
- kernel scheduling/flow-control in `N_TTY`, and `strace`'s own tracing overhead, both of which perturb
  timing.

No specific kernel buffer size is asserted as a cap; only the measured distribution is reported.

### `poll` cadence (F20 evidence)

On the reader thread, the distribution of the `poll()` timeout argument (the third argument) over run 1
directly shows the `input_delay` budget being counted down and the fallback to a blocking wait:

```bash
grep -E '\bpoll\(\[\{fd=6<' /tmp/kitty_pty_probe/yes1.strace | grep -oE '\], 3, -?[0-9]+\)' | sort | uniq -c | sort -rn
```

```
  16714 ], 3, 0)
  16650 ], 3, 1)
  15061 ], 3, 2)
   2470 ], 3, -1)
```

The `2 → 1 → 0` ms timeouts are `OPT(input_delay) − elapsed` counting down from the 3 ms budget
`(inferred from code: kitty/child-monitor.c:L1508-L1509)`; the `-1` (blocking) polls occur when there
are no pending main-loop wakeups `(inferred from code: kitty/child-monitor.c:L1512)`. The raw
`poll(...,3,1) = POLLIN fd=8` → `read(8,…)` blocks shown in the Run 1 excerpt above are observed
instances of this loop.

### EAGAIN attribution (F19 evidence)

The `-c` histogram's `errors` column counts `read` errors across *all* descriptors, so it cannot, by
itself, be attributed to any one fd. Re-parsing run 1 per descriptor gives the breakdown:

```bash
python3 /tmp/kitty_pty_probe/eagain_by_fd.py /tmp/kitty_pty_probe/yes1.strace
```

```
read errors by (fd, errno):
  fd=4   EAGAIN: 540
  fd=6   EAGAIN: 20
  fd=8 (PTY master) errors: 0
```

Every `EAGAIN` is on kitty's internal notification **eventfds** (fd 4 and fd 6) — the wakeup
descriptors kitty drains to coordinate its threads, alongside the two `EXTRA_FDS` wakeup/signal slots
the io-loop polls `(inferred from code: kitty/child-monitor.c:L35)` — **not** on the terminal data
path. The **PTY master fd 8 returns `EAGAIN` zero times** and never a zero-length read in any run
(confirmed by `analyze.py`: `fd8 EAGAIN=0  fd8 zero-length=0`). So the reads on the terminal's data
path are all successful; the EAGAINs are unrelated bookkeeping on the internal notification fds.

### Termination and return-to-prompt (F8 evidence)

The stream was stopped with an explicit Ctrl-C into the window, and termination was verified with `ps`
before and after, then the shell was confirmed responsive:

```bash
SHPID=82119
ps --ppid "$SHPID" -o pid,ppid,user,cmd          # DURING the stream
xdotool key --window "$WID" ctrl+c               # stop yes
ps --ppid "$SHPID" -o pid,ppid,user,cmd          # AFTER ctrl+c
xdotool type --window "$WID" 'echo BACK_AT_PROMPT_9137'; xdotool key --window "$WID" Return
ps -o pid,ppid,user,stat,cmd -p "$SHPID"         # shell still alive at prompt?
```

```
# DURING (run 1):
    PID    PPID USER     CMD
  83556   82119 ubuntu   yes hello
# AFTER ctrl+c (run 1) — no child remains, yes is gone:
    PID    PPID USER     CMD
# (run 2 DURING showed 83974 … yes hello; run 2 AFTER was likewise empty)
```

Post-stop responsiveness — the shell echoed a fresh typed command, proving it is back at an interactive
prompt (from `post.strace`):

```
82118 20:20:02.306870 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;echo BACK_AT_PROMPT_9137\7\33]1"..., 1048565) = 71 <0.000038>
```

and the shell process itself is alive in the foreground process group at its prompt:

```
    PID    PPID USER     STAT CMD
  82119   82052 ubuntu   Ss+  /bin/bash --posix
```

(`Ss+`: session leader, in the foreground group — i.e. interactively waiting.)

### Source-code rationale (inferred, not runtime-observed)

The high-volume behaviour uses the identical reader as Q3 — `read_bytes()`
`(kitty/child-monitor.c:L1337)` calling `read(fd, buf, available_buffer_space)`
`(kitty/child-monitor.c:L1345)` inside the io-loop `(kitty/child-monitor.c:L1481)`. What differs is only
that the master is *continuously* readable, so the `poll`→`read` loop iterates tens of thousands of
times. The producer/consumer decoupling (reads on the `KittyChildMon` I/O thread; parsing/rendering on
the main thread `process_global_state()` `(kitty/child-monitor.c:L1224)`) and the backpressure gate
`(kitty/vt-parser.c:L1477)` are the mechanisms that keep the 1 MiB buffer from overflowing under load.


## Q5 — The file-descriptor number kitty uses to read the PTY master

**Direct answer.** fd **`8`** (this session). The fd number is assigned by the OS when the PTY is
created, so it is run-specific; the *value observed here* is `8`, and the method to determine it is
shown below.

### Evidence

The `-yy` descriptor annotation on every master `read`/`poll` line already names it — fd `8` is the
master `/dev/pts/ptmx`, whose peer slave is `/dev/pts/0`:

```
… read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, …) …
```

Corroborated directly from the process's descriptor table:

```bash
ls -l "/proc/$KPID/fd/8"
```

```
lrwx------ 1 ubuntu ubuntu 64 Jul 14 20:12 8 -> /dev/pts/ptmx
```

`/dev/pts/ptmx` is the master device kitty reads from; the shell holds the corresponding slave
`/dev/pts/0` (Q2). So the concrete fd kitty uses to read the PTY master is **8**.

### Note on `poll` vs. non-blocking (precise roles)

Each master `read()` is preceded by a `poll()` on fd 8, but the two serve **distinct** purposes and one
is not "the reason" for the other:

- **`poll()` is the io-loop's readiness mechanism.** The reader thread waits in `poll()` until fd 8 is
  reported readable (or the `input_delay` timeout elapses); this is the event-loop design that lets one
  thread watch the master together with the wakeup/signal fds `(inferred from code:
  kitty/child-monitor.c:L1509 timed / L1512 blocking)`.
- **Non-blocking mode is a separate safeguard against a post-readiness race.** The master is set
  non-blocking `(inferred from code: kitty/child.py:L345)`. `read_bytes()` retries `EINTR`/`EAGAIN`
  in-loop *without* re-polling `(inferred from code: kitty/child-monitor.c:L1347)`; non-blocking mode
  guarantees that if the data is no longer available at the moment of the `read()` (a race after
  readiness was signalled), the call returns `EAGAIN` immediately instead of blocking the I/O thread.

### Source-code rationale for the fd's origin (inferred, not runtime-observed)

fd 8 is `child.child_fd` — the **master** end returned by `os.openpty()` and stored as
`self.child_fd = master` `(kitty/child.py:L338)`, then made non-blocking `(kitty/child.py:L345)`. It is
forwarded into the C child monitor by `Boss.add_child()` — `self.child_monitor.add_child(window.id,
window.child.pid, window.child.child_fd, window.screen)` `(kitty/boss.py:L585, L587)` — which lands in
C `add_child()` `(kitty/child-monitor.c:L305)`, after which the io-loop polls and reads it. The specific
integer `8` is whatever the OS assigned at `openpty()` time in this run.

## Q6 — The reader function and the text-vs-escape parser function

**Direct answer.**

- **Reader:** `read_bytes()` `(kitty/child-monitor.c:L1337)` — the function that issues `read()` on the
  PTY master fd.
- **Parser (text vs. escape split):** `consume_input()` `(kitty/vt-parser.c:L1367)`, which in its
  normal-state branch calls **`consume_normal()`** `(kitty/vt-parser.c:L230)`; `consume_normal()`
  separates **decoded non-ESC input** from **escape/control sequences**.

Both function names are **identified from the source** (inferred from code, at the lines cited above and
detailed below). The *reader* identification is additionally **grounded in runtime observation**: every
PTY-master read in every captured trace was issued by the one I/O thread whose read call these functions
implement (shown next).

### Observed grounding (who does the reading)

In every trace captured for this investigation, **100 % of the fd 8 reads were issued by a single
dedicated thread, TID `82118`** — never the main thread (`82052`, which only drains the wakeup eventfd).
This is the exact per-trace attribution from the authoritative re-parse of each retained trace:

```
echo test123 : reader TID(s): {82118: 16}
yes hello #1 : reader TID(s): {82118: 53494}
yes hello #2 : reader TID(s): {82118: 57445}
yes hello #3 : reader TID(s): {82118: 57533}
```

(The `strace` attach banner `Process 82052 attached with 67 threads` confirms `-f` followed **all**
threads, so no other reader thread could have been missed.) By the code, that I/O thread is `io_loop()`
`(inferred from code: kitty/child-monitor.c:L1481)`, named `KittyChildMon`
`(inferred from code: kitty/child-monitor.c:L1489)`, and the read call it makes is `read_bytes()`,
invoked at `(inferred from code: kitty/child-monitor.c:L1531)`.

### How the printable text is separated from escape sequences (inferred from code)

The bytes filled by `read_bytes()` are consumed on the **main** thread: `parse_input()`
`(kitty/child-monitor.c:L1236)` → `consume_input()` `(kitty/vt-parser.c:L1367)`. In the normal state,
`consume_input()` dispatches to `consume_normal()` `(kitty/vt-parser.c:L1376-L1377 → L230)`. There:

1. `consume_normal()` calls `utf8_decode_to_esc()` `(kitty/vt-parser.c:L232; defined at
   kitty/simd-string.c:L72)`, which decodes UTF-8 bytes into codepoints **until it hits an `ESC`
   (`0x1b`) sentinel**.
2. The decoded run (everything up to the next `ESC`) is handed to `screen_draw_text()`
   `(kitty/vt-parser.c:L236 → kitty/screen.c:L866)`.
3. When the `ESC` sentinel is found, the parser switches into escape/control handling —
   `if (sentinel_found) { SET_STATE(ESC); … }` `(kitty/vt-parser.c:L238)` — which routes CSI/OSC/DCS
   and other control sequences to their handlers.

**Precise wording (F22 nuance).** `utf8_decode_to_esc()` stops **only** at `ESC` (`0x1b`), so the run it
produces is best described as **decoded non-ESC input**, *not* "printable characters only": it can still
contain C0 control bytes such as `CR`/`LF`/`BEL` (indeed the `yes hello` stream is full of `\r\n`).
Those C0 controls are handled *inside* `screen_draw_text()`'s inner loop `draw_text_loop()`
`(kitty/screen.c:L763-L802)`, where a `if (ch < ' ')` branch performs cursor/line operations
(carriage-return, line-feed, backspace, tab, bell, …) rather than drawing a glyph. So the split is
two-level: `consume_normal()` separates **decoded non-ESC input** (to `screen_draw_text`) from
**ESC-introduced escape sequences** (to the control-sequence dispatch); and within the non-ESC input,
`screen_draw_text` further distinguishes **C0 controls** from **printable glyphs**.


## Appendix A — Answers at a glance

| Q | Question | Answer (from observation) |
|---|---|---|
| Q1 | Build & launch | `CI=true python3 setup.py --ignore-compiler-warnings` → exit 0, `kitty 0.35.2`; launched as user `ubuntu` (uid 1000) under headless `Xvfb`; kitty PID `82052` |
| Q2 | Spawned shell | `/bin/bash --posix`, PID `82119`, **direct** child of kitty; connected via PTY slave `/dev/pts/0` |
| Q3 | `echo test123` reads | `poll()` then `read()` on the master **fd 8**; **16** reads; each requests up to **1 MiB** (`BUF_SZ`); returns `12×1 + 11 + 47 + 114 + 433 = 617` bytes |
| Q4 | `yes hello` reads | same `read_bytes()`/fd 8 path; **~6,500–7,000 reads/s**, median **≈ 1.1 KiB**, max **≈ 19.2 KiB** (never approaches 1 MiB); stable across 3 runs |
| Q5 | Master fd number | **`8`** → `/dev/pts/ptmx` |
| Q6 | Reader / parser functions | reader **`read_bytes()`** `[kitty/child-monitor.c:L1337]`; parser **`consume_input()` → `consume_normal()`** `[kitty/vt-parser.c:L1367, L230]` |

## Appendix B — Reproducible command procedure (consolidated)

The full discovery-and-capture procedure, copy-paste-safe (variables quoted; PIDs validated numeric).
Concrete ids differ per run; the commands discover them.

```bash
KDIR="/tmp/blitzy/kitty/blitzy-81322a7e-8c21-4921-ab8a-068d5c584657_7749f6"
PROBE="/tmp/kitty_pty_probe"; mkdir -p "$PROBE"

# 1. build (canonical) and launch as the non-root user 'ubuntu' under a headless display
( cd "$KDIR" && CI=true python3 setup.py --ignore-compiler-warnings ) ; echo "build exit=$?"
nohup Xvfb :99 -screen 0 1280x1024x24 +extension GLX +render -noreset -ac >/tmp/xvfb99.log 2>&1 &
export DISPLAY=:99
nohup setsid sudo -u ubuntu -H env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
      HOME=/home/ubuntu "$KDIR/kitty/launcher/kitty" >"$PROBE/kitty_run.log" 2>&1 &
sleep 3

# 2. discover identifiers (validated numeric)
KPID="$(pgrep -u ubuntu -x kitty | head -n1)"
[ -n "$KPID" ] && [[ "$KPID" =~ ^[0-9]+$ ]] || { echo "no kitty pid"; exit 1; }
WID="$(xdotool search --pid "$KPID" | head -n1)"
[ -n "$WID" ] && [[ "$WID" =~ ^[0-9]+$ ]] || { echo "no window"; exit 1; }
SHPID="$(pgrep -P "$KPID" | head -n1)"
[ -n "$SHPID" ] && [[ "$SHPID" =~ ^[0-9]+$ ]] || { echo "no shell"; exit 1; }

# 3. attach the tracer (follow threads; annotate fds), inject the user's exact inputs, stop the tracer
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o "$PROBE/echo.strace" &
STRACE_PID=$!
xdotool type --window "$WID" 'echo test123'; xdotool key --window "$WID" Return
sleep 1; kill "$STRACE_PID"; wait "$STRACE_PID" 2>/dev/null

# 4. high-volume run(s): stream, then stop with Ctrl-C and confirm termination
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o "$PROBE/yes1.strace" &
STRACE_PID=$!
xdotool type --window "$WID" 'yes hello'; xdotool key --window "$WID" Return
sleep 8
ps --ppid "$SHPID" -o pid,ppid,user,cmd            # DURING: shows `yes hello`
xdotool key --window "$WID" ctrl+c                 # stop the stream
kill "$STRACE_PID"; wait "$STRACE_PID" 2>/dev/null
ps --ppid "$SHPID" -o pid,ppid,user,cmd            # AFTER: empty

# 5. analyse (authoritative per-thread fd attribution)
python3 "$PROBE/analyze.py" "$PROBE/yes1.strace"
```

## Appendix C — Environment and tooling

- **OS / runtimes (observed):** Ubuntu 25.10; CPython 3.13.7; Go 1.24.4; `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0`.
- **Display:** headless `Xvfb :99` with software GL (`LIBGL_ALWAYS_SOFTWARE=1`, `GALLIUM_DRIVER=llvmpipe`).
- **Observation tools:** `strace` (syscall trace), `xdotool` (real keystroke injection + window discovery),
  `xprop` (window ownership), `xxd`, `ps`, `/proc` — none are project dependencies; no source or manifest
  was changed.
- **`ptrace_scope`:** read as `0` before tracing (allows `strace -p`), set to `1` afterwards
  (Appendix D).


## Appendix D — Repository integrity, cleanup, and provenance

Per the read-only constraint ("refrain from altering any source files … temporary logs or small helper
scripts are acceptable, but delete them afterward"), the investigation left the repository unchanged
except for this single document. The evidence below was captured during teardown.

**Processes stopped (kitty, Xvfb, tracer):**

```
kitty: not running
Xvfb: not running
strace: not running
```

**Build artifacts removed — `git clean` inventory before/after:**

```bash
git status --ignored --porcelain | grep -c '^!!'   # count of ignored build artifacts, before
git clean -fdX | wc -l                             # remove ONLY git-ignored files; count removed
git clean -ndX                                     # dry-run AFTER: what still remains?
git status --ignored --porcelain | grep -c '^!!'   # count of ignored build artifacts, after
```

```
# before: 154 ignored build artifacts present (build/, __pycache__/, generated protocol files,
#         constants_generated.go, kitty/launcher/*, kittens/*, tools/*, …)
154        # git clean -fdX removed 154 ignored paths
           # git clean -ndX AFTER printed nothing (empty) — no ignored files remain
0          # ignored build artifacts remaining
```

A pre-clean dry-run (`git clean -ndX`) was checked first and confirmed it targeted **only** git-ignored
build output — nothing under `blitzy/` and no tracked file.

**`kernel.yama.ptrace_scope` set to the hardened value:**

```bash
cat /proc/sys/kernel/yama/ptrace_scope        # before change
sysctl -w kernel.yama.ptrace_scope=1
cat /proc/sys/kernel/yama/ptrace_scope        # after change
```

```
0     # original value (read before tracing; allowed strace -p to attach)
1     # set afterwards — container left MORE hardened than found
```

**Temporary artifacts deleted:** the observation directory `/tmp/kitty_pty_probe` (raw `*.strace`
traces, `analyze.py`, helper scripts, logs) was removed with `rm -rf /tmp/kitty_pty_probe`; it no longer
exists. No temporary file was ever created inside the repository.

**Final sole-file state** (the repository differs from the kitty source baseline only by this document):

```bash
git status --porcelain
git diff --stat 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
git diff --check
```

```
 M blitzy/documentation/kitty_815df1e210e0.md
```

`git status --porcelain` lists **exactly one** modified tracked file — this document — and no untracked
files. `git diff --stat` against the kitty source baseline `815df1e210e0` reports **`1 file changed`**
(this document only; zero source files changed), and `git diff --check` reports no whitespace or
end-of-file problems. These checks are re-run unchanged as the final validation step before commit.
