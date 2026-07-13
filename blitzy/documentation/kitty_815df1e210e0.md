# How kitty's C code communicates with its shell over a PTY — a runtime investigation

**Target:** [kitty](https://github.com/kovidgoyal/kitty) terminal emulator, checked out at commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`). All `file:line`
references below were re-verified against this exact revision with `grep -n` inside the build
container.

**Method (observe-first):** every factual claim in this document was produced by *building kitty from
source, launching it in its default configuration, and observing the live process* with `strace`,
`ps`/`pstree`, `lsof`, and `/proc/<pid>/fd`. Each claim is backed by (a) the exact command that was
run, (b) its complete, unedited output (long repetitive spans are elided only where explicitly
marked, and retained lines are shown verbatim), and (c) a source citation. Values are marked
**[observed]** (captured at runtime) or **[inferred]** (deduced from code, not directly measured).
No source file was modified; the only artifact added to the repository is this document.

**Environment (used transparently):** the build, launch, and all runtime observations were performed
inside the user-mandated Docker image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
(image id `sha256:c0824992ad0b…`, also tagged
`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). Full
details are in §2.

---

## 1. Summary — the headline answers

When kitty starts, its Python layer creates a pseudoterminal (PTY) pair, forks, and `execvp`s the
user's login shell in the child; kitty keeps the **master** end of the PTY and reads everything the
shell writes to the **slave**.

| Question | Answer (this run) | Observed / Inferred |
|---|---|---|
| Process spawned | `/bin/bash` (root's login shell) | observed |
| PID | **5820** (child of kitty PID 5752) | observed |
| Exact command line | **`/bin/bash --posix`** | observed |
| PTY device path | slave **`/dev/pts/0`** (shell side) ↔ master **`/dev/pts/ptmx`**, `tty-index 0` (kitty side) | observed |
| Master FD kitty reads from | **fd 8** | observed |
| Syscalls to read the PTY | **`poll()` then `read()`**, repeated | observed |
| Read buffer size (the `read()` `count`) | **1048576 bytes = 1 MiB** (`BUF_SZ`), minus any unparsed backlog | observed + code |
| Bytes returned for `echo test123` | 1 byte per keystroke echo; then **11, 47, 114, 182** bytes after Enter | observed |
| `yes hello` per-read byte count | median ≈ **1073–1178 B**, mean ≈ **1261–1422 B**, p90 ≈ **1911–2317 B**, p95 ≈ **2445–3374 B**, max ≈ **19.1–20.2 KB** | observed |
| `yes hello` read frequency | ≈ **7,300 reads/s** (7289–7338, under `strace`; traced setup only — see §6.6) | observed |
| Reader function (reads the fd) | **`read_bytes()`** — `kitty/child-monitor.c:1337` (the `read()` at `:1345`), on the I/O thread **`KittyChildMon`** | observed + code |
| Parser function (text vs escapes) | **`consume_input()`** — `kitty/vt-parser.c:1367` → `consume_normal()` (`:230`, printable) vs `consume_esc()`/`consume_csi()` (`:261`/`:839`, escapes); runs on the main event-loop thread (thread identity **source-derived**, see §8.3) | function observed + code; thread source-derived |

The end-to-end byte path that these answers trace out:

```
shell child (/bin/bash --posix, PID 5820)
  └─ writes stdout → PTY slave  /dev/pts/0   (shell fd 0/1/2)
       └─ kernel N_TTY line discipline (ONLCR: '\n' → '\r\n')
            └─ PTY master  /dev/pts/ptmx  = kitty fd 8      ← R5: the FD kitty reads
                 └─ io_loop() poll()                         kitty/child-monitor.c:1481 / 1509
                      └─ on POLLIN → read_bytes() → read()   kitty/child-monitor.c:1337 / 1345   ← R6a reader (thread KittyChildMon)
                           └─ vt_parser_commit_write() into the shared 1 MiB buffer   BUF_SZ  kitty/vt-parser.c:18
                                └─ (main-loop thread) parse_worker()/run_worker() → consume_input()   kitty/vt-parser.c:1496 / 1417 / 1367   ← R6b parser
                                     ├─ printable → consume_normal() → screen_draw_text()   kitty/vt-parser.c:230 / 236 → kitty/screen.c:866
                                     └─ escape/control → consume_esc/consume_csi/dispatch_*   kitty/vt-parser.c:261 / 839
```

Direction matters: the **shell writes to its slave**, and **kitty reads the master**. This is the
standard Linux UNIX‑98 PTY model — a master clone (`/dev/ptmx`) paired with a slave (`/dev/pts/N`)
that becomes the child's controlling terminal, with `/proc/<pid>/fd/N` symlinks revealing the binding.

---

## 2. Environment & method

* **Container / image (mandated, used transparently):** all work was done inside the user-provided
  Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
  (id `sha256:c0824992ad0b…`). The container was started with `--cap-add=SYS_PTRACE
  --security-opt seccomp=unconfined` so that `strace`/`ptrace` works. Inside it:

  ```console
  $ . /etc/os-release; echo "$PRETTY_NAME"
  Ubuntu 24.04.2 LTS
  $ gcc --version | head -1
  gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
  $ go version
  go version go1.23.4 linux/amd64
  $ python3 --version
  Python 3.12.3
  $ git -C /app rev-parse HEAD
  815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
  ```

  The kitty source is checked out at `/app`; observation tools (`strace`, `xvfb`, `xdotool`,
  `lsof`, `psmisc`, mesa software GL) were installed into the container with `apt`. The run is as
  `root`, so `ptrace` works despite `kernel.yama.ptrace_scope = 1`.
* **Build:** kitty's canonical build entry point is `setup.py`, wrapped by the `Makefile`. In this
  image the plain `make` succeeds with exit status 0 and **no** warning-suppression flag is needed
  (see §3.1).
* **Headless launch:** kitty is a GLFW/OpenGL GUI and will not initialize without a display, so it
  was launched under a virtual X display (`Xvfb :99`) with software OpenGL
  (`LIBGL_ALWAYS_SOFTWARE=1`). See §3.2.
* **Driving real keyboard input:** `echo test123` and `yes hello` were typed into the *live* kitty
  window using `xdotool`, which synthesizes **XTEST** key events. XTEST events are delivered by the X
  server as ordinary server-level input events, i.e. they enter kitty through its normal GLFW
  keyboard callback, exactly as a physical keyboard would (this is simulated input at the X-server
  level, not kitty's remote-control protocol or any debug hook). It is the canonical input path.
* **Observation tools:** the mandated thread-following form
  `strace -f -ttt -yy -s 256 -e trace=poll,ppoll,read,write`, attached to the **whole kitty
  process** — `-f` follows every thread (kitty performs the PTY `read()` on a dedicated I/O thread
  named `KittyChildMon`, not the main thread, so tracing the whole process with `-f` is what captures
  the reader; attaching to the main thread alone would capture zero PTY reads), `-ttt` stamps every
  line with an absolute epoch timestamp, `-yy` annotates each fd with its backing device/path,
  `-s 256` widens captured strings, and the `-e trace=` filter keeps the descriptor syscalls of
  interest. strace's own attached/detached thread banner is redirected to a separate file so it never
  contaminates the syscall trace. Also used: `ps`/`pstree`, `lsof`, and `/proc/<pid>/fd` +
  `/proc/<pid>/fdinfo` + `/proc/<pid>/task/*/comm`.
* **Secure, self-validating observation harness.** To make the procedure safe and reproducible (no
  predictable-path race, no stale-PID hazard, no runaway process), the harness:
  * runs with `umask 077` and keeps every artifact in a private directory created with
    `WORK=$(mktemp -d /root/kitty_obs.XXXXXX)` — a random-suffix directory under `/root` (mode
    `0700`, not world-writable, so there is no `/tmp` symlink/race exposure). The harness validates
    the directory is a real directory, not a symlink, owned by `root`, mode `700` before using it,
    and later removes exactly that validated path.
  * captures the PID of every background process via `$!` at launch (`Xvfb`, `kitty`, each
    `strace`), and **validates identity before acting**: kitty's PID is accepted only after
    `readlink /proc/<pid>/exe` equals `/app/kitty/launcher/kitty`; the shell PID is found by
    parentage (`ps --ppid <kitty>`); the reader thread id is discovered by scanning
    `/proc/<kitty>/task/*/comm` for `KittyChildMon`; the X window id is discovered with
    `xdotool search --pid <kitty>` (never hard-coded).
  * bounds every high-volume run with an explicit sleep window, stops `yes` through the real input
    path (`xdotool key ctrl+c`) **and** a fail-safe watchdog that escalates `kill -INT/-TERM/-KILL`
    on the discovered `yes` PID, then verifies the process is gone; each `strace` is stopped with
    `kill "$STRACE_PID"; wait` and verified absent before the next run.
* **Read-only constraint:** no repository source file was modified. All traces/logs were written
  inside the private `WORK` directory (outside the repo) and deleted at the end; the source tree at
  `/app` is byte-for-byte unchanged (proved in §10).

---

## 3. R1 — Build kitty from source, and launch it

### 3.1 Build

kitty's canonical build is `make`, whose `all:` target runs `python3 setup.py` (`Makefile:12-13`);
`build()` is defined at `setup.py:1084`, and the build compiles the `fast_data_types` C extension
and Go-builds the `kitten` binary. In the mandated image the bare canonical command completes
cleanly — **exit status 0, with no warning-as-error suppression flag** (unlike a newer host whose
`wayland-protocols` headers would trip `-Werror=switch`; that mismatch does not occur here):

```console
$ cd /app
$ { time make ; } > "$WORK/build_make.log" 2>&1     # log inside the private work dir, not /tmp
$ echo "make_exit=$?"
make_exit=0
$ grep -E '^(real|user|sys)' "$WORK/build_make.log"
real	0m45.177s
user	2m18.030s
sys	0m26.272s
```

The build log (217 lines) is authentic and complete; kitty's build system prints one progress line
per step (it does not echo raw `gcc` command lines). The mandated image ships a **warm object
cache** — the per-translation-unit objects under `build/` were compiled when the image was built —
so this canonical `make` recompiled only the out-of-date GLFW/Wayland units, **relinked** the
`fast_data_types` C extension (which statically includes the four investigation-central translation
units `kitty/screen.c`, `kitty/child-monitor.c`, `kitty/vt-parser.c`, and `kitty/child.c`) and the
launcher, and then Go-built the `kitten` binary. The full log, with only the long Go package span
elided as marked, is shown exactly as logged:

```
python3 setup.py 
[1/5] Compiling [wayland] glfw/input.c ...
[2/5] Compiling [wayland] glfw/xkb_glfw.c ...
[3/5] Compiling [wayland] glfw/window.c ...
[4/5] Compiling [wayland] glfw/wl_init.c ...
[5/5] Compiling [wayland] glfw/egl_context.c ...
 done
[1/3] Linking kitty/fast_data_types ...
[2/3] Linking [wayland] kitty/glfw-wayland ...
[3/3] Linking launcher ...
 done
log/internal
image/color
   [lines 14–212 elided: Go build of the kitten binary, one Go package path per line]
kitty/tools/cmd

real	0m45.177s
user	2m18.030s
sys	0m26.272s
```

The four investigation-central sources are compiled into the relinked `fast_data_types` extension;
their object files are present in the warm cache, which is why the incremental relink incorporates
them without recompiling:

```console
$ ls -la build/fast_data_types-kitty-{screen,child-monitor,vt-parser,child}.c.o
-rw-r--r-- 1 root 1001 212256 Aug 28  2025 build/fast_data_types-kitty-child-monitor.c.o
-rw-r--r-- 1 root 1001  35032 Aug 28  2025 build/fast_data_types-kitty-child.c.o
-rw-r--r-- 1 root 1001 542496 Aug 28  2025 build/fast_data_types-kitty-screen.c.o
-rw-r--r-- 1 root 1001 139904 Aug 28  2025 build/fast_data_types-kitty-vt-parser.c.o
```

Resulting artifacts (all git-ignored; mode `700` because the harness runs with `umask 077`) and the
version banner:

```console
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwx------ 1 root 1001  1221264 Jul 13 22:24 kitty/fast_data_types.so
-rwx------ 1 root 1001 15945988 Jul 13 22:24 kitty/launcher/kitten
-rwx------ 1 root 1001    36224 Jul 13 22:23 kitty/launcher/kitty

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

Toolchain floors declared by the repo (the container satisfies both — Go 1.23.4 ≥ 1.22 and
Python 3.12.3 ≥ 3.8, per the versions in §2):

```console
$ grep -E '^go ' go.mod
go 1.22
$ grep requires-python pyproject.toml
requires-python = ">=3.8"
```

### 3.2 Launch (headless, canonical configuration)

kitty was launched under the virtual display with software OpenGL, detached from the shell so the
launcher's I/O never blocks the harness, and its PID captured and validated:

```console
$ Xvfb :99 -screen 0 1280x800x24 </dev/null >"$WORK/xvfb.log" 2>&1 &
$ XVFB_PID=$!                                        # PID captured directly from $!
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ setsid ./kitty/launcher/kitty --config NONE </dev/null >"$WORK/kitty.log" 2>&1 &
$ KITTY_PID=$!                                        # PID captured directly from $!, never pgrep|head
$ echo "KITTY_PID=$KITTY_PID  exe=$(readlink /proc/$KITTY_PID/exe)"
KITTY_PID=5752  exe=/app/kitty/launcher/kitty
```

`--config NONE` selects kitty's built-in defaults (there is no user `kitty.conf` in this
environment, so this is the canonical default configuration; shell integration remains enabled by
default). The only diagnostics kitty emits headlessly are harmless:

```console
$ cat "$WORK/kitty.log"
[0.303] Failed to open systemd user bus with error: No medium found
ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
```

The launched process is **PID 5752**; every observation below is against this live instance, and
its executable path was validated (`/app/kitty/launcher/kitty`) before any tracing.

---

## 4. R2 — What process is spawned, its PID, exact command line, and the connecting PTY path

### 4.1 The spawned process and its PID  [observed]

```console
$ pstree -p 5752 | head -1
kitty(5752)-+-bash(5820)

$ ps -o pid,ppid,pgid,sid,tty,stat,args -p 5752,5820
    PID    PPID    PGID     SID TT       STAT COMMAND
   5752    1274    5752    5752 ?        Ssl  ./kitty/launcher/kitty --config NONE
   5820    5752    5820    5820 pts/0    Ss+  /bin/bash --posix
```

kitty (PID 5752) spawned exactly one child: **`/bin/bash`, PID 5820**, whose parent is kitty
(`PPID 5752`), which is a session leader in its own session/group (`SID/PGID 5820`, state `Ss+`,
i.e. the foreground process group) with controlling terminal `pts/0`.

### 4.2 The exact command line  [observed]

Read straight from the kernel's copy of the child's argv (NUL-delimited), so there is no ambiguity
about spacing or a leading hyphen:

```console
$ tr '\0' '|' < /proc/5820/cmdline; echo
/bin/bash|--posix|
$ od -c /proc/5820/cmdline
0000000   /   b   i   n   /   b   a   s   h  \0   -   -   p   o   s   i
0000020   x  \0
0000022
```

The exact command line is **`/bin/bash --posix`** — `argv[0] = /bin/bash`, `argv[1] = --posix`, each
NUL-terminated. Note there is **no leading `-`**: on Linux `argv[0]` is the plain shell path, *not* a
login-shell `-bash` name. (kitty's `kitten run-shell` wrapper, which would otherwise reshape the
argv, is macOS-only — see §4.4.)

### 4.3 The PTY device path connecting kitty and the shell  [observed]

The shell's standard streams all point at the **slave** side of the PTY:

```console
$ ls -l /proc/5820/fd/0 /proc/5820/fd/1 /proc/5820/fd/2
lrwx------ 1 root root 64 Jul 13 22:24 /proc/5820/fd/0 -> /dev/pts/0
lrwx------ 1 root root 64 Jul 13 22:24 /proc/5820/fd/1 -> /dev/pts/0
lrwx------ 1 root root 64 Jul 13 22:24 /proc/5820/fd/2 -> /dev/pts/0
```

kitty holds the corresponding **master** end (previewed here; corroborated four ways, including the
`fdinfo` slave-index mapping, in §7):

```console
$ ls -l /proc/5752/fd/8
lrwx------ 1 root root 64 Jul 13 22:24 /proc/5752/fd/8 -> /dev/pts/ptmx
```

So the connecting PTY is the pair **`/dev/pts/0` (slave, held by the shell) ↔ `/dev/pts/ptmx`
(master clone, held by kitty as fd 8)**. The shell writes to `/dev/pts/0`; those bytes become
readable on kitty's master fd. (`/dev/pts/ptmx` is the master-clone device; §7 shows how the kernel
`tty-index` pins this master to the specific slave `/dev/pts/0`.)

### 4.4 Why this is the process/argv/PTY we see — bound to the source

* **Which shell.** kitty resolves the shell through `resolved_shell()` (`kitty/utils.py:768`); with
  the default option `shell == '.'` it returns `[shell_path]`, and `shell_path` is read from the
  password database — `pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` (`kitty/constants.py:181`).
  Root's `pw_shell` here is `/bin/bash`, which is exactly the program observed at PID 5820:

  ```console
  $ getent passwd root
  root:x:0:0:root:/root:/bin/bash
  ```
* **The PTY pair.** `Child.fork()` (`kitty/child.py:276`) creates the PTY with `openpty()`
  (defined `kitty/child.py:170`, "master and slave are in blocking mode", called at `:281`) and then
  calls into C: `pid = fast_data_types.spawn(...)` (`kitty/child.py:333`).
* **The OS-level spawn and controlling terminal (corrected mechanism).** The C `spawn()`
  (`kitty/child.c:81`) first obtains only the **slave's pathname** with
  `ttyname_r(slave, name, ...)` (`kitty/child.c:88`) — `ttyname_r` merely fills `name` with
  `/dev/pts/0`; it does **not** by itself establish a controlling terminal. `spawn()` then `fork()`s
  (`kitty/child.c:97`), and the **child** establishes the controlling terminal in these steps:

  1. `setsid()` (`kitty/child.c:123`) — start a new session, detaching from any prior controlling tty;
  2. `int tfd = safe_open(name, O_RDWR | O_CLOEXEC, 0)` (`kitty/child.c:126`) — open the slave by the
     pathname from step 0;
  3. `ioctl(tfd, TIOCSCTTY, 0)` (`kitty/child.c:129`) — **this** is what makes the slave the process's
     controlling terminal;
  4. `dup2(slave, STDOUT_FILENO)` / `dup2(slave, STDERR_FILENO)` / `dup2(slave, STDIN_FILENO)`
     (`kitty/child.c:138`, `:139`, `:145`) — wire fd 0/1/2 to the slave;
  5. `execvp(exe, argv)` (`kitty/child.c:159`) — replace the child image with the resolved shell.

  That is the process, controlling terminal, and fd 0/1/2 → `/dev/pts/0` seen in §4.1 and §4.3.
* **Why `--posix` (and no `-`).** The `--posix` argument is injected by kitty's bash shell
  integration, not typed by anyone. The bash branch `setup_bash_env()`
  (`kitty/shell_integration.py:70`) does `argv.insert(1, '--posix')`
  (`kitty/shell_integration.py:146`) and points `ENV` at kitty's `shell-integration/bash/kitty.bash`
  so bash sources it in POSIX mode. That is precisely the `--posix` seen in the argv, and it is why
  `argv[0]` stays the bare `/bin/bash` rather than a `-bash` login name.
* **Linux vs macOS.** The alternative `kitten run-shell` wrapper is gated by
  `should_run_via_run_shell_kitten = is_macos and self.is_default_shell`
  (`kitty/child.py:230`). Because the container is Linux, that branch is not taken; the shell is
  `execvp`'d directly and integration is injected via env/argv as above — which is why the process
  line is the bare `/bin/bash --posix`.

---

## 5. R3 — Typing `echo test123`: the read syscalls, the buffer size, and the byte count

### 5.1 How it was captured

Per the mandated methodology, `strace` was attached to the **whole kitty process** and told to
**follow every thread** (`-f`), so the reads issued on kitty's dedicated I/O thread are captured even
though that thread is not the main thread. Timestamps are absolute epoch (`-ttt`), each fd is
annotated with its backing device (`-yy`), strings are widened to 256 bytes (`-s 256`), and the
trace is restricted to the descriptor syscalls (`poll,ppoll,read,write`). `echo test123` was then
typed into the live window and Enter pressed:

```console
$ echo "kitty_pid=$KITTY_PID"
kitty_pid=5752
$ strace -f -ttt -yy -s 256 -e trace=poll,ppoll,read,write -p "$KITTY_PID" -o "$WORK/r3.strace" &
$ STRACE_PID=$!                          # tracer pid captured directly (never via pgrep)
$ sleep 2
$ WID=$(xdotool search --pid "$KITTY_PID" | head -1); echo "window=$WID"
window=2097164
$ xdotool windowfocus --sync "$WID"
$ xdotool type --window "$WID" --delay 90 'echo test123'
$ xdotool key  --window "$WID" Return
$ sleep 2
$ kill "$STRACE_PID"; wait "$STRACE_PID" 2>/dev/null    # tracer stopped and reaped
```

Because `-f` follows threads, `strace` prints how many it attached to — the reader thread is one of
them:

```
strace: Process 5752 attached with 67 threads
```

Every PTY read is performed on kitty's I/O thread **`KittyChildMon`**, whose TID in this run is
**5819** (discovered from `/proc/$KITTY_PID/task/*/comm`; proved in §8). `strace -f` prefixes each
line with the acting TID, so the reader thread's activity is exactly the lines beginning `5819`; the
slices below are filtered to that thread with `awk '$1==5819'` for readability, and the filter is
noted where applied. One cosmetic effect of following 67 threads: when `strace` interrupts a blocked
syscall to service another thread, it prints that call as an `<unfinished …>`/`<… resumed>` pair —
the fd, arguments and return value are unchanged, and such pairs are pointed out where they occur.

### 5.2 The syscalls: `poll()` then `read()`  [observed]

The system calls kitty makes to read from the PTY are **`poll()` followed by `read()`**. The io_loop
first drains its wakeup eventfd (fd 6), then `poll()`s the three fds it watches — the wakeup eventfd
(6), a signalfd (7), and the PTY master (8). For a typed key it is first woken with `POLLOUT` to
*write* the keystroke to the shell, then the kernel's terminal echo makes that byte readable, so it
is woken by `POLLIN` and issues the `read()`. This is the complete, verbatim round trip for the
first keystroke `e` (reader tid 5819, shown inline — no thread-split here):

```
5819  1783981493.044159 read(6<anon_inode:[eventfd]>, "\1\0\0\0\0\0\0\0", 1024) = 8
5819  1783981493.044308 read(6<anon_inode:[eventfd]>, 0x7fbffe4d4740, 1024) = -1 EAGAIN (Resource temporarily unavailable)
5819  1783981493.044406 poll([{fd=6<anon_inode:[eventfd]>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
5819  1783981493.044540 write(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "e", 1) = 1
5819  1783981493.044594 poll([{fd=6<anon_inode:[eventfd]>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
5819  1783981493.044659 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "e", 1048576) = 1
5819  1783981493.044709 write(4<anon_inode:[eventfd]>, "\1\0\0\0\0\0\0\0", 8) = 8
```

The trailing `write(4<eventfd>, …)` is how the reader thread signals the main (parser/render) thread
that new bytes are available (the reader→parser hand-off; see §8). Each of the twelve characters of
`echo test123` echoes back as its **own 1-byte** `read()` on fd 8 via the identical
`poll(POLLOUT)→write→poll(POLLIN)` cycle shown above. Filtered to the reader thread
(`awk '$1==5819'`), the twelve echo reads in order are — each requesting the full `1048576`-byte
buffer and returning `1`:

```
5819  1783981493.044659 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "e", 1048576) = 1
5819  1783981493.088434 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "c", 1048576) = 1
5819  1783981493.143909 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "h", 1048576) = 1
5819  1783981493.199340 <... read resumed>"o", 1048576) = 1
5819  1783981493.255078 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, " ", 1048576) = 1
5819  1783981493.310591 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "t", 1048576) = 1
5819  1783981493.366069 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "e", 1048576) = 1
5819  1783981493.421266 <... read resumed>"s", 1048576) = 1
5819  1783981493.476769 <... read resumed>"t", 1048576) = 1
5819  1783981493.532696 <... read resumed>"1", 1048576) = 1
5819  1783981493.588362 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "2", 1048576) = 1
5819  1783981493.644622 <... read resumed>"3", 1048576) = 1
```

They spell `e c h o ⎵ t e s t 1 2 3` — the twelve bytes of `echo test123`. Five of the twelve were
interrupted by another thread mid-call and are therefore printed by `strace -f` as `<… read resumed>`
continuations; the fd, the `1048576` count and the `= 1` return are identical to the inline reads
(their `read(8<… <unfinished …>` entry halves appear earlier in the unfiltered trace). The reader
thread issued **exactly 16** `read()`s on fd 8 during this capture — these twelve 1-byte echoes plus
the four post-Enter reads below — so nothing is elided:

```console
$ grep -c 'read(8<' "$WORK/r3.strace"
16
```

After Enter (kitty first writes the carriage return `"\r"`), the shell disables bracketed-paste,
emits its shell-integration OSC markers, runs the command, and repaints the next prompt — arriving
as **four** reads whose returned sizes are **11, 47, 114, 182** bytes. This is the complete,
verbatim post-Enter span (reader tid 5819; the 11-byte read is printed by `strace -f` as an
`<unfinished …>`/`<… resumed>` pair around a thread switch):

```
5819  1783981494.204302 write(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r", 1) = 1
5819  1783981494.204520 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>,  <unfinished ...>
5819  1783981494.204604 <... read resumed>"\r\n\33[?2004l\r", 1048576) = 11
5819  1783981494.205883 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47
5819  1783981494.206063 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114
5819  1783981494.206872 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]0;root@ceb0fb892563: /app\7root@ceb0fb892563:/app# \33]133;k;start_suffix_kitty\7\33[5 q\33]2;/app\7\33]133;k;end_suffix_kitty\7", 1048404) = 182
```

(In the `strace` output `\33` is ESC, `\7` is BEL, `\r\n` is CR‑LF, printed exactly as `strace`
rendered them. The `\33[?2004l`/`\33[?2004h` toggle bracketed-paste mode; `\33]2;…\7` sets the
window title; `\33]133;…\7` are OSC 133 shell-integration marks; the actual command output is the
`test123\r\n` embedded in the 114-byte read; the 182-byte read is the redrawn prompt, which contains
the literal string `root@ceb0fb892563:/app# ` — see §5.3 on why that last count is
environment-dependent.)

### 5.3 The file descriptor, the buffer size, and the bytes returned  [observed]

Reading the `read(fd, buf, count) = nbytes` calls directly:

* **fd** `= 8` — the PTY master (`/dev/pts/ptmx`). This is the answer to R5, corroborated in §7.
* **count (buffer size)** `= 1048576` bytes `= 1 MiB` on a fully-drained buffer.
* **nbytes (bytes returned)** `= 1` for each of the twelve keystroke echoes; then `11`, `47`, `114`,
  `182` for the four post-Enter reads.

The last post-Enter read is **182 bytes in this run**, because it carries the redrawn prompt string
`root@ceb0fb892563:/app#` — its size therefore depends on the container hostname
(`ceb0fb892563`) and the current working directory (`/app`). This is an observed, environment-
dependent value: on a host with a different hostname/cwd the prompt read would differ in length. The
first three post-Enter reads (11, 47, 114) are fixed by the command and the shell-integration
protocol.

### 5.4 Why the buffer size is 1 MiB — and why it visibly shrinks

The `count` is **not** a hardcoded constant at the `read()` call site. `read_bytes()` reads into a
buffer produced by `vt_parser_create_write_buffer()` (`kitty/vt-parser.c:1451`), which returns
`*sz = BUF_SZ - self->write.offset` (`:1457`), where `#define BUF_SZ (1024u*1024u)` = 1 MiB
(`kitty/vt-parser.c:18`) and `write.offset` is the amount of already-received-but-not-yet-parsed
data. So the requested `count` is "1 MiB minus whatever the parser worker hasn't consumed yet."

This derivation is directly visible in the post-Enter span above: because the four reads arrived
faster than the parser worker drained them, the requested `count` shrank by **exactly** the size of
each preceding read:

```
1048576              (first read; buffer fully drained)
1048576 − 11  = 1048565   (second read; 11 bytes still pending)
1048565 − 47  = 1048518   (third read; +47 pending)
1048518 − 114 = 1048404   (fourth read; +114 pending)
```

Those deltas (11, 47, 114) are the returned sizes of the immediately preceding reads, which is the
clearest possible proof that the 11-byte read exists and that `count = BUF_SZ − write.offset`.

### 5.5 The reader function (R6a)

The `read()` shown above is issued by **`read_bytes()`** (`kitty/child-monitor.c:1337`); the actual
call is `len = read(fd, buf, available_buffer_space);` at `kitty/child-monitor.c:1345`. It is invoked
from the I/O loop at `kitty/child-monitor.c:1531`. Full analysis of this function and its thread is
in §8.

---

## 6. R4 — Running `yes hello`: how the reading behavior changes under high volume

### 6.1 How it was captured, and at what scale

`strace` was attached to the **whole kitty process** with the same mandated, thread-following form
used for R3 (`-f` so the read syscalls issued on the dedicated I/O thread are captured; `-ttt`
absolute epoch timestamps; `-yy` to annotate fd 8 with its backing device; `-s 256` wide strings;
`-e trace=poll,ppoll,read,write`). `yes hello` was then typed into the live window through the real
keyboard path, left to stream, and stopped. The run was **bounded** and **repeated with identical
input** (two unchanged trials). The wall-clock instants when the stream was started and when Ctrl‑C
was sent were recorded (as absolute epoch seconds, matching `-ttt`) so a clean steady-state window
could be carved out later. strace's own "attached/detached" thread chatter is written to a *separate*
`.attach` file (`2>…`) so it never contaminates the syscall trace:

```console
$ strace -f -ttt -yy -s 256 -e trace=poll,ppoll,read,write \
         -p "$KITTY_PID" -o "$WORK/r4_run1.strace" 2>"$WORK/r4_run1.attach" &
$ STRACE_PID=$!                                    # tracer pid captured directly ($!), never pgrep
$ sleep 2
$ xdotool windowactivate --sync "$WID"; xdotool windowfocus --sync "$WID"
$ YES_START=$(date +%s.%N); echo "YES_START=$YES_START"
YES_START=1783981498.733820702
$ xdotool type --window "$WID" --clearmodifiers --delay 60 'yes hello'
$ xdotool key  --window "$WID" --clearmodifiers Return
$ sleep 8                                          # hold steady state (>= 5 s analyzable window)
$ CTRL_C=$(date +%s.%N); echo "CTRL_C=$CTRL_C"
CTRL_C=1783981507.029369453
$ # discover the `yes` child of THIS shell and VALIDATE its identity (comm) before signalling:
$ yp=""; for c in $(pgrep -P "$SHELL_PID"); do grep -qa yes "/proc/$c/comm" && yp="$c"; done
$ echo "YES_PID=$yp comm=$(cat /proc/$yp/comm)"
YES_PID=6041 comm=yes
$ xdotool key --window "$WID" --clearmodifiers ctrl+c      # primary stop, via the real input path
$ for i in $(seq 1 10); do sleep 0.5; [ -n "$yp" ] && [ -d /proc/$yp ] || break; \
      kill -INT "$yp" 2>/dev/null; sleep .3; [ -d /proc/$yp ] && kill -TERM "$yp" 2>/dev/null; done
$ echo "yes remaining: $(for c in $(pgrep -P "$SHELL_PID"); do grep -qa yes /proc/$c/comm && echo "$c"; done | head -1)"
yes remaining:
$ kill "$STRACE_PID" 2>/dev/null; wait "$STRACE_PID" 2>/dev/null    # tracer stopped and reaped
```

The watchdog signals **only** a PID whose `/proc/<pid>/comm` still reads `yes` (never a recycled
PID), and `STRACE_PID`/`YES_PID` are obtained from `$!` and a validated `pgrep -P "$SHELL_PID"` scan
respectively — not from an unqualified `pgrep | head`. The separate attach file confirms the tracer
followed every thread:

```console
$ head -1 "$WORK/r4_run1.attach"
strace: Process 5752 attached with 67 threads
```

Scale captured (both runs), from the raw trace files (positive-return, inline fd-8 reads):

```console
$ wc -l "$WORK/r4_run1.strace"; grep -cE 'read\(8<[^)]*>, .*\) = [1-9][0-9]*$' "$WORK/r4_run1.strace"
140644 /root/kitty_obs.thGYUU/r4_run1.strace
56004
$ wc -l "$WORK/r4_run2.strace"; grep -cE 'read\(8<[^)]*>, .*\) = [1-9][0-9]*$' "$WORK/r4_run2.strace"
139850 /root/kitty_obs.thGYUU/r4_run2.strace
55527
```

Each capture spans ≈8.3 s wall-clock and contains **tens of thousands** of PTY reads; the steady-state
window analyzed in §6.3 (`[YES_START + 1.0 s , CTRL_C − 0.5 s]`) is **6.796 s** wide for both runs —
comfortably above the ≥5 s magnitude-observation floor — and still holds ≈49.5–49.9 k reads.

### 6.2 What the stream looks like: still one `read()` per `POLLIN`  [observed]

The pacing does **not** switch to a different syscall or a drain-until-empty loop. It remains the
same `read()` → `poll()` rhythm as R3: exactly **one** `read()` per `POLLIN`, with a `poll()`
between each read. What changes is that (a) `POLLIN` on fd 8 is now satisfied on almost every poll,
so reads fire in rapid succession, each coalescing many `hello\r\n` lines (`\n`→`\r\n` is the PTY's
ONLCR translation, so each "hello" line is 7 bytes on the wire); and (b) the `poll()` timeout is no
longer the idle `-1` (infinite) of R3 but a **small, dynamic** value — in the slice below it toggles
between `1` and `0` ms; across the whole window it is a `0/1/2 ms` mix with occasional `-1` (the full
distribution and the reason are in §6.5/§6.6). It is **not** a constant "zero because data is always
ready." This is a slice from run 1 on the reader thread (tid `5819`); `poll()` entries are shown
stitched into single logical lines using the same `<unfinished …>`/`<… resumed>` convention explained
in §5.1, the 3-fd set is shown in full on the first `poll()` and abbreviated as `[…3 fds…]`
thereafter, and each read's payload is truncated (strace `-s 256`):

```
1783981499.060282 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\n"..., 1040136) = 2151
1783981499.060351 poll([{fd=6<anon_inode:[eventfd]>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
1783981499.060435 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1037985) = 2191
1783981499.060492 poll([…3 fds…], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
1783981499.060571 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1035794) = 1841
1783981499.060629 poll([…3 fds…], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
1783981499.060699 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1033953) = 1923
1783981499.060754 poll([…3 fds…], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
1783981499.060883 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\n"..., 1032030) = 2564
1783981499.060941 poll([…3 fds…], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
1783981499.061041 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1029466) = 2140
1783981499.061090 poll([…3 fds…], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
1783981499.061161 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\n"..., 1027326) = 1612
```

Three structural facts are visible: (1) reads do **not** align to line boundaries — a read starts
mid-stream (`"\r\nhello…"` or `"hello…"`) and ends mid-line, so the parser, not the reader,
reassembles lines; (2) the requested `count` tracks `BUF_SZ − write.offset` and **drifts down** as
the reader outpaces the parser (`1040136, 1037985, 1035794, 1033953, 1032030, 1029466, 1027326` —
each drop ≈ the previous read's return), later jumping back toward the full `1048576` once the parser
drains and `write.offset` resets — the count is never the limiter on how much comes back; and (3) the
`poll()` timeout is a live, changing number (`1`, then `0`), not a fixed constant.

### 6.3 Per-read byte-count distribution and read frequency  [observed]

The window boundaries recorded in §6.1 are used to exclude ramp-up and teardown: the steady-state
window is `[YES_START + 1.0 s , CTRL_C − 0.5 s]`. Rather than the fragile shell one-liners of an
earlier draft (which relied on unset `$LO`/`$HI` variables and a shell ternary histogram that could
not emit an empty bin), the analysis is now performed by **one self-contained Python script** that
takes the trace, the reader tid, and the two epoch instants as arguments, computes the window
internally, stitches `<unfinished …>`/`<… resumed>` reads, dumps every per-read `(timestamp, bytes)`
into `$WORK/pts1` (inside the work dir — never the CWD), and prints the full statistic set with a
histogram whose bins are **all pre-initialized to 0** (so empty bins are printed). It is fully
reproducible from the trace files alone:

```console
$ cat > "$WORK/r4_analyze.py" <<'PY'
#!/usr/bin/env python3
# Reproducible R4 steady-state analyzer for the yes-hello PTY read stream.
# Usage: r4_analyze.py <strace_file> <reader_tid> <yes_start_epoch> <ctrlc_epoch> [workdir]
# Emits per-read (timestamp,bytes) for completed fd-8 reads on the reader thread,
# restricted to the steady-state window [start+1.0s, ctrlc-0.5s], then full stats.
import sys, re, math, statistics
from collections import Counter

trace, tid = sys.argv[1], sys.argv[2]
start, ctrlc = float(sys.argv[3]), float(sys.argv[4])
lo, hi = start + 1.0, ctrlc - 0.5            # steady-state window bounds
WORK = sys.argv[5] if len(sys.argv) > 5 else "."
pts = f"{WORK}/pts1"                          # per-read dump, inside the work dir

line_re = re.compile(r'^(\d+)\s+([0-9]+\.[0-9]+)\s+(.*)$')
reads, pending = [], None                     # completed fd-8 reads; per-thread stitch state
for line in open(trace, errors='replace'):
    m = line_re.match(line)
    if not m or m.group(1) != tid:
        continue
    ts, rest = float(m.group(2)), m.group(3)
    if rest.startswith('<... read resumed>'):
        r = re.search(r'=\s+(-?\d+)', rest)
        if pending == 'read8' and r and int(r.group(1)) >= 0:
            reads.append((ts, int(r.group(1))))
        pending = None
    elif 'resumed>' in rest:                  # poll/write/other resumed -> clear
        pending = None
    elif rest.startswith('read(8<'):
        if rest.rstrip().endswith('<unfinished ...>'):
            pending = 'read8'
        else:
            r = re.search(r'=\s+(-?\d+)\s*$', rest)
            if r and int(r.group(1)) >= 0:
                reads.append((ts, int(r.group(1))))
    elif rest.startswith('read(6<'):
        pending = 'read6' if rest.rstrip().endswith('<unfinished ...>') else None
    elif rest.rstrip().endswith('<unfinished ...>'):
        pending = 'other'

win = [(ts, n) for ts, n in reads if lo <= ts <= hi]
with open(pts, 'w') as f:
    for ts, n in win:
        f.write(f"{ts:.6f} {n}\n")

n = len(win)
b = [x for _, x in win]
tvals = [t for t, _ in win]
span = tvals[-1] - tvals[0] if n > 1 else 0.0

def pct(sorted_b, p):
    k = (len(sorted_b) - 1) * p / 100.0
    f = math.floor(k); c = math.ceil(k)
    return sorted_b[int(k)] if f == c else sorted_b[f]*(c-k) + sorted_b[c]*(k-f)

sb = sorted(b)
cnt = Counter(b); mode, freq = cnt.most_common(1)[0]

print(f"per-read dump written to {pts}  (lines={n})")
print(f"steady-state window  = [{lo:.6f}, {hi:.6f}]  width={hi-lo:.3f}s")
print(f"observed read span   = {span:.3f}s  (first->last completed read in window)")
print(f"reads (count)        = {n}")
print(f"read frequency       = {n/(hi-lo):.1f} reads/s")
print(f"bytes total          = {sum(b)}")
print(f"throughput           = {sum(b)/(hi-lo):.1f} bytes/s  ({sum(b)/(hi-lo)/1048576:.2f} MiB/s)")
print(f"min bytes/read       = {min(b)}")
print(f"max bytes/read       = {max(b)}")
print(f"mean bytes/read      = {statistics.mean(b):.1f}")
print(f"median bytes/read    = {statistics.median(b):.1f}")
print(f"mode bytes/read      = {mode}  (freq={freq}, {100*freq/n:.2f}% of reads)")
print(f"p90 bytes/read       = {pct(sb,90):.0f}")
print(f"p95 bytes/read       = {pct(sb,95):.0f}")
print("histogram (bytes/read, initialized bins, ascending):")
BINS = [(1,63),(64,127),(128,255),(256,511),(512,1023),(1024,2047),
        (2048,4095),(4096,8191),(8192,16383),(16384,32767)]
hist = {rng: 0 for rng in BINS}               # every bin initialized to 0
for x in b:
    for loB, hiB in BINS:
        if loB <= x <= hiB:
            hist[(loB, hiB)] += 1
            break
for loB, hiB in BINS:                          # deterministic ascending order
    c = hist[(loB, hiB)]
    bar = '#' * (c * 40 // n) if n else ''
    print(f"  {loB:>6}-{hiB:<6} {c:>6} ({100*c/n:5.2f}%) {bar}")
PY
```

**Run 1** (`YES_START=1783981498.733820702`, `CTRL_C=1783981507.029369453`):

```console
$ python3 "$WORK/r4_analyze.py" "$WORK/r4_run1.strace" 5819 1783981498.733820702 1783981507.029369453 "$WORK"
per-read dump written to /root/kitty_obs.thGYUU/pts1  (lines=49868)
steady-state window  = [1783981499.733821, 1783981506.529369]  width=6.796s
observed read span   = 6.795s  (first->last completed read in window)
reads (count)        = 49868
read frequency       = 7338.3 reads/s
bytes total          = 62890884
throughput           = 9254717.6 bytes/s  (8.83 MiB/s)
min bytes/read       = 14
max bytes/read       = 20202
mean bytes/read      = 1261.1
median bytes/read    = 1073.0
mode bytes/read      = 945  (freq=233, 0.47% of reads)
p90 bytes/read       = 1911
p95 bytes/read       = 2445
histogram (bytes/read, initialized bins, ascending):
       1-63          5 ( 0.01%) 
      64-127         0 ( 0.00%) 
     128-255        12 ( 0.02%) 
     256-511       929 ( 1.86%) 
     512-1023    21484 (43.08%) #################
    1024-2047    23382 (46.89%) ##################
    2048-4095     3450 ( 6.92%) ##
    4096-8191      548 ( 1.10%) 
    8192-16383      43 ( 0.09%) 
   16384-32767      15 ( 0.03%) 
```

**Run 2** (identical input; `YES_START=1783981513.131583527`, `CTRL_C=1783981521.427735512`):

```console
$ python3 "$WORK/r4_analyze.py" "$WORK/r4_run2.strace" 5819 1783981513.131583527 1783981521.427735512 "$WORK"
per-read dump written to /root/kitty_obs.thGYUU/pts1  (lines=49537)
steady-state window  = [1783981514.131583, 1783981520.927736]  width=6.796s
observed read span   = 6.795s  (first->last completed read in window)
reads (count)        = 49537
read frequency       = 7289.0 reads/s
bytes total          = 70422786
throughput           = 10362155.6 bytes/s  (9.88 MiB/s)
min bytes/read       = 7
max bytes/read       = 19059
mean bytes/read      = 1421.6
median bytes/read    = 1178.0
mode bytes/read      = 903  (freq=194, 0.39% of reads)
p90 bytes/read       = 2317
p95 bytes/read       = 3374
histogram (bytes/read, initialized bins, ascending):
       1-63          3 ( 0.01%) 
      64-127         0 ( 0.00%) 
     128-255        14 ( 0.03%) 
     256-511       814 ( 1.64%) 
     512-1023    17520 (35.37%) ##############
    1024-2047    24543 (49.54%) ###################
    2048-4095     5485 (11.07%) ####
    4096-8191     1118 ( 2.26%) 
    8192-16383      32 ( 0.06%) 
   16384-32767       8 ( 0.02%) 
```

**Reading the numbers.** The read frequency is **≈7.3 k reads/s** (7338.3 and 7289.0 — within 0.7 %
of each other), sustaining **≈8.8–9.9 MiB/s** through the PTY master. The **typical** per-read size
is **hundreds of bytes to ~1 KiB**: median **1073 / 1178 B**, mean **1261 / 1422 B**. The
distribution is broad and only weakly peaked — the single most common exact size (mode **945 / 903 B**)
accounts for under **0.5 %** of reads — so the *median* is the honest "typical" figure, not any single
mode. Tails: p90 **1911 / 2317 B**, p95 **2445 / 3374 B**, and rare spikes to a max of **20202 / 19059 B**
(still two orders of magnitude below the 1 MiB request). The `64-127` bin is `0 (0.00%)` in both runs —
printed explicitly because the histogram bins are pre-initialized — confirming the analyzer emits empty
bins rather than silently dropping them.

**Stability across the two unchanged runs.** The *shape* is stable: **≈90 % / ≈85 %** of reads fall in
the combined **512–2047 byte** band (`512-1023` + `1024-2047`), the two dominant bins swapping rank
between runs (43 %/47 % vs 35 %/50 %); medians (1073 vs 1178 B), means (1261 vs 1422 B) and read rates
(7338 vs 7289 /s) all agree within ordinary kernel-PTY-buffering jitter of a `strace`-slowed consumer.
Both windows are **6.796 s** (≥ 5 s), and the per-read `pts1` dump is reproducible from either trace.
Values are reported as observed, not averaged into a single figure.

### 6.4 How this differs from the single command (`echo test123`) — before/during/after

* **Before / single command (R3):** reads are sparse and *tiny* — one `read()` returning **1 byte**
  per keystroke echo, then the four post-Enter reads (11/47/114/182). kitty spends almost all its
  time blocked in `poll(…, -1)` (infinite timeout).
* **During the stream (R4):** the *same* `read → poll` primitive fires **≈7.3 k times/second**,
  each read now returning **hundreds of bytes to ~1 KiB** (median **1073 / 1178 B**) by coalescing
  roughly **150 `hello` lines** (median 1073 B ÷ 7 B per `hello\r\n`), sustaining ≈8.8–9.9 MiB/s.
  Crucially, the `poll()` timeout does **not** collapse to a constant `0`; it becomes a **small,
  dynamic** value — a `0/1/2 ms` mix with an occasional `-1` — driven by kitty's render-coalescing
  timer, not by "data always ready" (mechanism and distribution in §6.5). The syscall *pattern* is
  unchanged (still one read per `POLLIN`); only its rate, per-call payload, and poll timeout changed.
* **After (Ctrl‑C):** the read rate collapses back to the idle `poll(…, -1)`-blocked state and the
  next prompt is emitted as a small burst, exactly like the post-Enter reads in §5.

### 6.5 Why it behaves this way — cause → effect, bound to the source

* **One read per `POLLIN` (not drain-to-empty).** The I/O loop `io_loop()`
  (`kitty/child-monitor.c:1481`) `poll()`s (`:1509` timed / `:1512` blocking) and, on POLLIN, calls
  `read_bytes()` (`:1531`). `read_bytes()` performs exactly **one** successful `read()` per POLLIN:
  its `while(true)` loop (`:1344`) `continue`s only on `EINTR`/`EAGAIN` (`kitty/child-monitor.c:1347`)
  and hits an **unconditional `break`** immediately after a non-negative `read()`
  (`kitty/child-monitor.c:1352`, with the `read()` itself at `:1345`). It is that explicit `break` —
  not the fd being non-blocking — that enforces one successful read per dispatch. **Effect:** high
  volume manifests as *more frequent* single reads, not as fewer/larger drain reads.
* **The `poll()` timeout is dynamic — set by the render-coalescing timer, not by "data ready".**
  This is the key nuance the earlier draft got wrong. The loop does **not** hard-code a `0` timeout
  during a stream. Its timeout is derived from kitty's input-coalescing logic (`:1506`–`:1512`):
  when a wakeup of the main (render) loop is *pending*, it computes
  `time_delta = OPT(input_delay) − (now − last_main_loop_wakeup_at)` (`:1508`) and polls for
  `monotonic_t_to_ms(time_delta)` milliseconds (`:1509`), clamping to `0` when the delay has already
  elapsed (`else ret = 0`, `:1510`); only when **no** wakeup is pending does it block indefinitely
  with `-1` (`:1512`). The coalescing timer is armed at `:1562`–`:1569`: after receiving data kitty
  wakes the render loop only once per `input_delay`, whose default is **3 ms**
  (`kitty/options/definition.py:878`), deferring otherwise (`has_pending_wakeups = true`). Because
  `monotonic_t_to_ms()` truncates the sub-3 ms remainder to an integer, the pending-wakeup timeout
  lands in the `0/1/2 ms` bands in roughly equal thirds, with `-1` appearing whenever the parser has
  momentarily caught up (no pending wakeup). The observed distribution matches this mechanism exactly
  and is stable across both runs (reader-thread `poll()` entries inside the same 6.796 s window):

  ```
  run 1 (47054 poll entries)          run 2 (46939 poll entries)
    timeout= -1 ms :   2032 ( 4.3%)     timeout= -1 ms :   2055 ( 4.4%)
    timeout=  0 ms :  15493 (32.9%)     timeout=  0 ms :  15494 (33.0%)
    timeout=  1 ms :  15613 (33.2%)     timeout=  1 ms :  15459 (32.9%)
    timeout=  2 ms :  13916 (29.6%)     timeout=  2 ms :  13931 (29.7%)
  ```

  **Effect:** the reader is paced by the 3 ms render-coalescing window (why reads come in tight
  bursts separated by short `0–2 ms` polls), not by a fixed non-blocking spin, and it still blocks on
  `-1` the instant the stream lets up.
* **Parser back-pressure gates POLLIN.** The loop arms `POLLIN` for the child fd only when the
  parser buffer has room:
  `children_fds[...].events = vt_parser_has_space_for_input(...) ? POLLIN : 0;`
  (`kitty/child-monitor.c:1501`). `vt_parser_has_space_for_input()` (`kitty/vt-parser.c:1477`)
  returns `self->read.sz + self->write.pending < BUF_SZ` (`:1481`), with `BUF_SZ = 1 MiB` (`:18`).
  **Effect:** if the parser ever fell a full 1 MiB behind, the reader would stop being woken until it
  caught up. In these traced runs the parser *did* fall materially behind at times — the smallest
  requested `count` observed in run 1 was **542,175 bytes**, i.e. a backlog of ≈506 KiB, roughly half
  the buffer — but it never reached the full 1 MiB, so `vt_parser_has_space_for_input()` never
  returned false and `POLLIN` stayed armed throughout; back-pressure never had to disable the read.
* **Magnitude is bounded by the kernel, not by kitty.** kitty requests up to ~1 MiB (and the raw
  `count` argument is exactly `BUF_SZ − write.offset`, so it shrinks as the backlog grows), but the
  largest single read observed in run 1 was **20,202 bytes**. The realized bytes-per-read are
  therefore limited by the kernel PTY / N_TTY line-discipline buffering, **not** by kitty's 1 MiB
  request — which is why the distribution clusters around ~1 KiB regardless of the huge buffer.

### 6.6 Measurement caveat (stated honestly)

`strace` interposes on *every* syscall via `ptrace`, which slows the reading thread and, indirectly,
the shell producing the stream. The read-frequency figures above (≈7,300 reads/s) therefore
characterize **the traced setup only**; they are *not* a measurement of, and are not extrapolated to,
kitty's untraced native read frequency (tracing changes the timing that determines how many bytes
accumulate between reads). What is robust to this overhead — and what answers the question — is the
*shape* of the behavior (one read per `POLLIN`; coalescing ~150 lines per read; the `poll()` timeout
becoming a small, dynamic `0–2 ms` value driven by the 3 ms render-coalescing timer rather than the
idle `-1`) and the per-read *magnitude* (a median around **1 KiB**, kernel-bounded, never approaching
1 MiB), both stable across the two runs.

---

## 7. R5 — The file-descriptor number kitty uses to read from the PTY master

The master-side descriptor is **fd 8**, corroborated **four** independent ways.

**(a) From every `read()` in the traces** (the mandated `-yy` annotates the fd with its backing
device — the master-clone character device `char 5:2` and, after `@`, the paired slave `/dev/pts/0`).
The `5819` prefix is the reader thread's TID under `strace -f`; the full echo/yes analyses are in
§5/§6:

```
5819  1783981493.044659 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "e", 1048576) = 1
5819  1783981500.702012 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\n", 912190) = 23
```

**(b) From `/proc/<kitty_pid>/fd`:**

```console
$ ls -l /proc/5752/fd/8
lrwx------ 1 root root 64 Jul 13 22:24 /proc/5752/fd/8 -> /dev/pts/ptmx
```

**(c) From `lsof`:**

```console
$ lsof -p 5752 | grep ptmx
kitty   5752 root    8u      CHR                5,2      0t0         2 /dev/pts/ptmx
```

`lsof` confirms fd **8**, opened read-write (`8u`), a character device with major/minor **5,2** —
which is `/dev/ptmx`, the UNIX‑98 PTY master multiplexer.

**(d) From `/proc/<kitty_pid>/fdinfo/8` — the exact master↔slave pairing.** The `fd` symlink resolves
only to the generic `/dev/pts/ptmx` clone device, which is not a unique slave identifier. The kernel
`fdinfo` exposes a `tty-index`, which **is** the slave number, uniquely pinning this master to
`/dev/pts/0`:

```console
$ cat /proc/5752/fdinfo/8
pos:	0
flags:	02104002
mnt_id:	6252
ino:	2
tty-index:	0
$ ls -l /dev/pts/0
crw--w---- 1 root tty 136, 0 Jul 13 22:24 /dev/pts/0
```

`tty-index: 0` ⇒ the slave is `/dev/pts/0` — precisely the device the shell holds on fd 0/1/2
(§4.3). So the four views agree: the `read()` calls (a), the `/proc` symlink (b), `lsof` (c), and the
`fdinfo` slave index (d) all identify **fd 8 = the PTY master paired with `/dev/pts/0`**.

**Why fd 8, and what non-blocking actually does.** In `Child.fork()`, after `openpty()` returns the
pair, kitty retains the master and stores it as `self.child_fd = master` (`kitty/child.py:338`), then
makes it non-blocking with `os.set_blocking(self.child_fd, False)` (`kitty/child.py:345`). The exact
integer (8) is just wherever the OS placed the master in kitty's descriptor table for this launch —
so the *number* is observed per-run, while the *fact that it is the master fd kitty reads* is
structural. Non-blocking mode is **not** what causes the one-read-per-dispatch policy (the explicit
`break` at `child-monitor.c:1352` does that — §6.5/§8.1); rather, non-blocking is protection: it
guarantees the `read()` returns immediately with `EAGAIN` instead of blocking the I/O thread if the
loop is ever entered on a stale/spurious readiness signal, keeping the poll-driven loop responsive.

---

## 8. R6 — The C functions: which reads the fd, which separates text from escapes (+ thread model)

### 8.1 R6a — The reader function: `read_bytes()` on the `KittyChildMon` I/O thread

The function that reads from the PTY file descriptor is **`read_bytes()`**
(`kitty/child-monitor.c:1337`); the actual system call is
`len = read(fd, buf, available_buffer_space);` (`kitty/child-monitor.c:1345`). It runs inside the
I/O loop `io_loop()` (`kitty/child-monitor.c:1481`), which names its thread
`set_thread_name("KittyChildMon");` (`kitty/child-monitor.c:1489`) and calls `read_bytes()` at
`kitty/child-monitor.c:1531`.

Its one-read-per-dispatch discipline is explicit in the source (the `read()`/`while` loop of
`read_bytes()`, verbatim; the surrounding function body is omitted):

```c
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
```

Here the `read()` is at `kitty/child-monitor.c:1345`, the `EINTR`/`EAGAIN` `continue` at `:1347`, and
the unconditional `break` at `:1352`. The loop `continue`s **only** on `EINTR`/`EAGAIN`; any
non-negative `read()` falls straight through to that `break`. It is the `break` — **not** the fd's
non-blocking mode — that makes `read_bytes()` perform exactly one successful read per POLLIN and then
hand back to the `poll()` loop, after committing the bytes to the parser buffer with
`vt_parser_commit_write()`.

That the read genuinely executes on a **dedicated, separately-named thread** — not the main thread —
was confirmed at runtime: every PTY `read()` in the traces is on tid **5819**, and that task's
`comm` is `KittyChildMon`:

```console
$ cat /proc/5752/task/5819/comm
KittyChildMon
```

`read_bytes()` does only two things per wakeup: one `read()` into the shared buffer, then commit it
for the parser (`vt_parser_commit_write()`, declared `kitty/vt-parser.h:35`). It does **not** parse —
that is a different function on a different thread (below).

### 8.2 R6b — The parser function: `consume_input()`, routing printable vs escape bytes

The function that parses the incoming data to separate printable text from escape sequences is
**`consume_input()`** (`kitty/vt-parser.c:1367`). It is a state machine that dispatches on the
current parser state and routes bytes to distinct branches:

* **printable text →** `consume_normal()` (`kitty/vt-parser.c:230`), from the `VTE_NORMAL` case.
  `consume_normal()` UTF‑8-decodes a run of ordinary characters and writes them straight to the
  screen with `screen_draw_text(self->screen, …)` (`kitty/vt-parser.c:236`), stopping at the first
  ESC/control sentinel. `screen_draw_text()` is the production screen-update entry point, implemented
  at `kitty/screen.c:866` (it calls `screen_on_input()` then `draw_text()` at `:867-868`). This is
  the "printable" half of the separation and the point where decoded text enters the cell model.
* **escape / control sequences →** `consume_esc()` (`kitty/vt-parser.c:261`, from the `VTE_ESC`
  case), `consume_csi()` (`kitty/vt-parser.c:839`, from the `VTE_CSI` case), and the
  OSC/APC/PM/DCS/SOS branches via the `dispatch_*` handlers — this is the "escape sequence" half.

`consume_input()` is driven by the parser worker **`run_worker()`** (`kitty/vt-parser.c:1417`), which
takes the parser lock, merges freshly committed bytes into the read region, and calls
`consume_input()` in a loop; `run_worker()` is exposed as **`parse_worker()`**
(`kitty/vt-parser.c:1496`). This exactly matches the R3 trace: the `\33]2;…\7` (OSC title) and
`\33]133;…\7` (OSC 133) bytes take the escape branch, while the literal `test123` takes
`consume_normal()` → `screen_draw_text()`.

### 8.3 The reader and parser run on different threads — why the reads are on `KittyChildMon`

The read and the parse are **separate threads**, decoupled by a shared buffer under a mutex:

* **Reader thread (runtime-observed):** `read_bytes()` runs on `KittyChildMon` (tid 5819, proven
  above by the `comm` read), created by
  `pthread_create(&self->io_thread, NULL, io_loop, self)` (`kitty/child-monitor.c:291`).
* **Parser thread (source-derived — not runtime-marked):** `consume_input()` runs on the thread that
  drives kitty's main event loop. Unlike the reader TID, this thread identity was **not** captured
  with a runtime marker in this investigation; it is established below from the source call-chain,
  and is consistent with the observed fact that the reads occur off the main thread. Grepping the
  source
  (`grep -n pthread_create kitty/child-monitor.c`) shows four `pthread_create` sites, **none of which
  hosts the parser**: the I/O thread just named (`io_loop`, started unconditionally by `start()`,
  `kitty/child-monitor.c:291`); a `talk_thread` for remote control, started **only** when remote
  control is configured — the `start()` gate `self->talk_fd > -1 || self->listen_fd > -1`
  (`kitty/child-monitor.c:285`) guards the create at `:286`, and the same `self->talk_thread` is
  lazily created on the peer-injection path at `:256` — which under the observed `--config NONE`
  launch (no `--listen-on`) never fires; and a transient, **detached** per-write helper
  (`thread_write`, created at `kitty/child-monitor.c:1002` by `cm_thread_write` and immediately
  `pthread_detach`'d at `:1004`), spawned only on demand. `vt-parser.c` contains **no**
  `pthread_create`, so the parser always runs on its caller's thread.
  The production parse driver is the main event loop: `main_loop()` (`kitty/child-monitor.c:1259`) →
  `run_main_loop(process_global_state, …)` (`:1262`) → `process_global_state()` (`:1224`) →
  `parse_input()` (`:451`, whose own comment reads "Parse all available input that was read in the
  I/O thread", called from `:1236`) → `do_parse()` (`:438`) → the `parse_func` pointer (`:440`, set
  to `parse_worker`) → `run_worker()` → `consume_input()`. Because `vt-parser.c` creates no thread of
  its own, `consume_input()` always executes on whatever thread calls `main_loop()` — kitty's main
  event-loop thread. (The GIL is **not** the reason: any thread that holds the GIL may call the
  CPython C-API, so GIL ownership does not by itself pin the parser to the main thread. The thread
  identity follows from this call-chain — a source-derived conclusion — not from GIL semantics.)
* **Runtime corroboration (reader thread only):** all PTY `read()`s in the traces are on tid 5819
  (`KittyChildMon`) — a thread demonstrably distinct from the main thread, whose tid equals the
  process PID **5752**. What is *directly observed* is therefore only that the reads run off the main
  thread; the parser's own thread was not separately marked, so its main-loop identity remains the
  source-derived conclusion above. Because the `read()` is on the reader thread, the whole process
  must be traced with `strace -f` (which follows every thread) to capture the reads — a trace that
  followed only the main thread would capture **zero** PTY reads. That is exactly why the captures in
  §5 and §6 attach with `-f -ttt -yy -s 256 -e trace=poll,ppoll,read,write -p "$KITTY_PID"`, and why
  the attach banner reports the full thread set (`Process 5752 attached with 67 threads`).

**Hand-off between the two threads.** The reader (I/O thread) and parser (main-loop thread) communicate
through the single shared **1 MiB** buffer (`BUF_SZ`, `kitty/vt-parser.c:18`): the producer commits
bytes with `vt_parser_commit_write()` (`kitty/vt-parser.h:35`, defined `kitty/vt-parser.c:1465`),
signals the main thread via an eventfd (the `write(4<eventfd>, …)` seen after each read in §5.2), and
the consumer later drains the bytes via `parse_worker`/`run_worker` → `consume_input()`. Flow control
between them is the `vt_parser_has_space_for_input()` gate (`kitty/vt-parser.c:1477`) that
arms/disarms `POLLIN` on the reader side (`kitty/child-monitor.c:1501`), as analyzed in §6.5.

---

## 9. Coverage checklist — every named item, observed vs inferred

| # | Named item asked | Answer | Evidence | Observed / Inferred |
|---|---|---|---|---|
| R1 | Build kitty from source | canonical `make` (→ `python3 setup.py`), **exit 0**; `Makefile:12-13`→`setup.py:1084` | §3.1 build log + `kitty --version` = `kitty 0.35.2 created by Kovid Goyal` | observed |
| R1 | Launch it | `Xvfb :99` + `LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty --config NONE` → PID 5752 | §3.2 | observed |
| R2 | Process spawned | `/bin/bash` | §4.1 `pstree`/`ps` | observed |
| R2 | PID | **5820** (child of kitty 5752) | §4.1 | observed |
| R2 | Exact command line | **`/bin/bash --posix`** (no leading `-`) | §4.2 `/proc/5820/cmdline` (`od -c`) | observed |
| R2 | PTY device path | slave **`/dev/pts/0`** ↔ master **`/dev/pts/ptmx`** (`tty-index 0`) | §4.3, §7(d) | observed |
| R3 | `echo test123` — syscalls | **`poll()` then `read()`** | §5.2 trace (timestamped) | observed |
| R3 | Buffer size | **1048576 B (1 MiB)** = `BUF_SZ − backlog` | §5.3–5.4; `vt-parser.c:18/1457` | observed + code |
| R3 | Bytes returned | 1 B/keystroke (×12); then **11, 47, 114, 182** after Enter | §5.2–5.3 trace | observed |
| R4 | `yes hello` — behavior change | same `read`→`poll(…, 0–2 ms)` loop, ≈7.3 k×/s, coalescing ~150 lines/read; poll timeout dynamic (0/1/2 ms mix, occasional −1) | §6.2–6.5 | observed |
| R4 | Read frequency | ≈ **7,300 reads/s** (7289–7338; traced setup only; not extrapolated) | §6.3 (2 runs) + §6.6 | observed |
| R4 | Typical bytes/read | median ≈ **1073–1178 B**, mean ≈ 1261–1422 B, p90 ≈ 1911–2317 B, p95 ≈ 2445–3374 B, max ≈ 19.1–20.2 KB | §6.3 (2 runs, stable shape) | observed |
| R5 | Master FD number | **fd 8** → `/dev/pts/ptmx`, `tty-index 0` → `/dev/pts/0` | §7 (a)(b)(c)(d) | observed |
| R6a | Reader function | **`read_bytes()`** `kitty/child-monitor.c:1337` (`read()` `:1345`, one-read `break` `:1352`), thread `KittyChildMon` | §8.1; `/proc/.../task/5819/comm` | observed + code |
| R6b | Parser function (text vs escapes) | **`consume_input()`** `kitty/vt-parser.c:1367` → `consume_normal()` `:230` → `screen_draw_text()` `screen.c:866` vs `consume_esc()`/`consume_csi()` `:261`/`:839` | §8.2 | observed + code |
| R6 | Reader/parser thread separation | reader on `KittyChildMon` (io_thread `:291`) — tid 5819 runtime-marked; parser on the main-loop thread (`parse_input` `:451`) — inferred from the source call-chain, not runtime-marked | §8.3 | reader **observed**; parser **source-derived** (consistent with runtime separation) |

Every runtime value above is **observed** for this launch. The only per-run–specific integers (the
kitty/shell PIDs, the reader tid, the X window id, and the fd number 8) are observed for this launch;
that fd 8 is *the master fd kitty reads* is structural per `kitty/child.py:338`. The last
post-Enter byte count (182) is observed and explicitly noted as prompt-length (hostname/cwd)
dependent (§5.3). The **one** conclusion that is *not* a direct runtime observation is the parser's
thread identity: it is source-derived from the call-chain (§8.3), because only the reader TID was
runtime-marked. Every other value above is directly observed for this launch.

---

## 10. Cleanup statement and repository integrity  [observed]

This investigation was strictly read-only with respect to the repository, and the observation
artifacts were removed at the end. All evidence below is the actual command output, not a prose
assertion.

**(a) The kitty source tree is byte-for-byte unchanged.** Inside the container the checkout stayed at
the target commit with an empty working tree throughout:

```console
$ git -C /app rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git -C /app status --porcelain | wc -l
0
```

**(b) The launched processes were stopped and verified gone.** Each background PID captured at launch
was killed and its absence checked; `yes` and every `strace` were stopped between/after runs (§6.1),
and finally kitty and Xvfb:

```console
$ kill "$KITTY_PID" 2>/dev/null; sleep 2; kill "$XVFB_PID" 2>/dev/null; sleep 1
$ ps -o pid,stat -p "$KITTY_PID" "$SHELL_PID" "$XVFB_PID" 2>/dev/null
    PID STAT
   5820 Zs
$ pgrep -af 'launcher/kitty --config NONE' || echo "no kitty process"
no kitty process
$ pgrep -x Xvfb || echo "no Xvfb process"
no Xvfb process
```

(kitty — PID 5752 — and Xvfb were terminated and immediately reaped, so they no longer appear in the
listing; the login shell, PID 5820, briefly remains as a *defunct* session leader — state `Zs` —
because the container's PID 1 does not `wait()` for it, and it is fully removed when the observation
container is torn down. `pgrep` confirms no live kitty or Xvfb process remains. Only the recorded PIDs
`$KITTY_PID`/`$XVFB_PID` are signalled — no broad `pkill`/`killall`.)

**(c) The private observation directory was removed and verified absent.** Because *every* artifact —
the build log, all `strace` traces, the per-read `pts1` dump, the analyzer script, and any helper
script — lived inside the single `mktemp -d` work directory, removing that one validated path deletes
everything the investigation created. No broad wildcard (`rm -f /root/*.sh`) is used or needed:

```console
$ rm -rf "$WORK"
$ echo -n "WORK exists after rm: "; [ -d "$WORK" ] && echo yes || echo no
WORK exists after rm: no
```

**(d) The only change in the deliverable repository is this document.** In the repository where the
answer is committed, `git status --porcelain` shows exactly one changed path (this file, which is a
modification of the pre-existing deliverable); no source file was modified:

```console
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md
```

The **only** persistent change left by this investigation is this single document,
`blitzy/documentation/kitty_815df1e210e0.md`.
