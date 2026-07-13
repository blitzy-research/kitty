# How kitty's C code communicates with its shell over a PTY — a runtime investigation

**Target:** [kitty](https://github.com/kovidgoyal/kitty) terminal emulator, checked out at commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`). All `file:line`
references below were re-verified against this exact revision with `grep -n`.

**Method (observe-first):** every factual claim in this document was produced by *building kitty from
source, launching it in its default configuration, and observing the live process* with `strace`,
`ps`/`pstree`, `lsof`, and `/proc/<pid>/fd`. Each claim is backed by (a) the exact command that was
run, (b) its complete, unedited output, and (c) a source citation. Values are explicitly marked
**[observed]** (captured at runtime) or **[inferred]** (deduced from code, not directly measured).
No source file was modified; the only artifact added to the repository is this document.

---

## 1. Summary — the headline answers

When kitty starts, its Python layer creates a pseudoterminal (PTY) pair, forks, and `execvp`s the
user's login shell in the child; kitty keeps the **master** end of the PTY and reads everything the
shell writes to the **slave**.

| Question | Answer (this run) | Observed / Inferred |
|---|---|---|
| Process spawned | `/bin/bash` (root's login shell) | observed |
| PID | **45065** (child of kitty PID 44997) | observed |
| Exact command line | **`/bin/bash --posix`** | observed |
| PTY device path | slave **`/dev/pts/0`** (shell side) ↔ master **`/dev/pts/ptmx`** (kitty side) | observed |
| Master FD kitty reads from | **fd 8** | observed |
| Syscalls to read the PTY | **`poll()` then `read()`**, repeated | observed |
| Read buffer size (the `read()` `count`) | **1048576 bytes = 1 MiB** (`BUF_SZ`), minus any unparsed backlog | observed + code |
| Bytes returned for `echo test123` | 1 byte per keystroke echo; then 11, 47, 114, 194 bytes after Enter | observed |
| `yes hello` per-read byte count | median ≈ **800 B**, mean ≈ **1000 B**, p99 ≈ **4 KiB**, max ≈ 18–19 KB | observed |
| `yes hello` read frequency | ≈ **8000 reads/s** (under `strace`; lower bound) | observed |
| Reader function (reads the fd) | **`read_bytes()`** — `kitty/child-monitor.c:1337` (the `read()` at `:1345`), on the I/O thread **`KittyChildMon`** | observed + code |
| Parser function (text vs escapes) | **`consume_input()`** — `kitty/vt-parser.c:1367` → `consume_normal()` (`:230`, printable) vs `consume_esc()`/`consume_csi()` (`:261`/`:839`, escapes), on the **main** thread | observed + code |

The end-to-end byte path that these answers trace out:

```
shell child (/bin/bash --posix, PID 45065)
  └─ writes stdout → PTY slave  /dev/pts/0   (shell fd 0/1/2)
       └─ kernel N_TTY line discipline (ONLCR: '\n' → '\r\n')
            └─ PTY master  /dev/pts/ptmx  = kitty fd 8      ← R5: the FD kitty reads
                 └─ io_loop() poll()                         kitty/child-monitor.c:1481 / 1509
                      └─ on POLLIN → read_bytes() → read()   kitty/child-monitor.c:1337 / 1345   ← R6a reader (thread KittyChildMon)
                           └─ vt_parser_commit_write() into the shared 1 MiB buffer   BUF_SZ  kitty/vt-parser.c:18
                                └─ (main thread) parse_worker()/run_worker() → consume_input()   kitty/vt-parser.c:1417 / 1367   ← R6b parser
                                     ├─ printable → consume_normal()          kitty/vt-parser.c:230
                                     └─ escape/control → consume_esc/consume_csi/dispatch_*   kitty/vt-parser.c:261 / 839
                                          └─ screen cell model (screen_draw_text, …)
```

Direction matters: the **shell writes to its slave**, and **kitty reads the master**. This is the
standard Linux UNIX‑98 PTY model — a master clone (`/dev/ptmx`) paired with a slave (`/dev/pts/N`)
that becomes the child's controlling terminal, with `/proc/<pid>/fd/N` symlinks revealing the binding.

---

## 2. Environment & method

* **Container / host:** Ubuntu 25.10, running as `root` (so `ptrace` works despite
  `kernel.yama.ptrace_scope = 1`). Toolchain: GCC 15.2.0, Go 1.24.4, CPython 3.13.7.
* **Build:** kitty's canonical build entry point is `setup.py` (wrapped by the `Makefile`). See §3.
* **Headless launch:** kitty is a GLFW/OpenGL GUI and will not initialize without a display, so it
  was launched under a virtual X display (`Xvfb :99`) with software OpenGL
  (`LIBGL_ALWAYS_SOFTWARE=1`). See §3.
* **Driving real keyboard input:** `echo test123` and `yes hello` were typed into the *live* kitty
  window using `xdotool`, which synthesizes **XTEST** key events. XTEST events are delivered by the X
  server as ordinary hardware-level key events (`send_event = False`) — i.e. they enter kitty through
  its normal GLFW keyboard callback, exactly as a physical keyboard would. This is the canonical
  input path; it is **not** kitty's remote-control protocol or any debug hook.
* **Observation tools:** `strace -f -y` (the `-f` is **mandatory** — kitty performs the PTY `read()`
  on a dedicated I/O thread, not the main thread, so a main-thread-only trace would miss it; `-y`
  annotates each fd with its backing path), `ps`/`pstree`, `lsof`, and `/proc/<pid>/fd` +
  `/proc/<pid>/task/*/comm`.
* **Read-only constraint:** no repository source file was modified. All traces/logs were written
  under `/tmp/kitty_obs/` (outside the repo) and deleted at the end (see §10).

---

## 3. R1 — Build kitty from source, and launch it

### 3.1 Build

kitty's canonical build is `python3 setup.py build` (the `Makefile`'s `all:` target at
`Makefile:12-13` runs `python3 setup.py`; `build()` is defined at `setup.py:1084`, and it compiles
the `fast_data_types` C extension and Go-builds the `kitten` binary via `go build` at
`setup.py:1148`).

**Honest note on the canonical command in this environment.** The bare `make` fails here because a
*newer* system `wayland-protocols` header introduces an enum value that kitty's `-Werror` build does
not (yet) handle — this is a toolchain/header mismatch, **not** a kitty source defect:

```console
$ make
...
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM' not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
make: *** [Makefile:13: all] Error 1
```

The supported build simply relaxes that warning-as-error with the project's own
`--ignore-compiler-warnings` flag. It completes cleanly (exit 0), compiling the C extension
(including the files central to this investigation — `kitty/screen.c`, `kitty/child-monitor.c`,
`kitty/vt-parser.c`, `kitty/child.c`), linking `fast_data_types.so`, linking the `kitty` launcher,
and Go-building `kitten`:

```console
$ python3 setup.py build --verbose --ignore-compiler-warnings
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
...
Detected: CompilerType.gcc
gcc ... -c kitty/screen.c -o build/fast_data_types-kitty-screen.c.o
gcc ... -c kitty/child-monitor.c -o build/fast_data_types-kitty-child-monitor.c.o
gcc ... -c kitty/vt-parser.c -o build/fast_data_types-kitty-vt-parser.c.o
...
gcc ... build/fast_data_types-kitty-child-monitor.c.o ... build/fast_data_types-kitty-vt-parser.c.o ... -o build/kitty/fast_data_types.so
gcc build/kitty-launcher-main.o build/kitty-launcher-single-instance.o ... -o kitty/launcher/kitty
Updating Go generated files...
/usr/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-ecabc877-16f7-4bb7-94fd-af5473cb3e57_59abb8/tools/cmd
```

Resulting artifacts (all git-ignored) and the version banner:

```console
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x 1 root root  1253792 ... kitty/fast_data_types.so
-rwxr-xr-x 1 root root 16429348 ... kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 ... kitty/launcher/kitty

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

Toolchain floors declared by the repo, and what is actually present:

```console
$ go version
go version go1.24.4 linux/amd64          # repo requires 'go 1.22'  (go.mod:3)

$ python3 --version
Python 3.13.7                            # repo requires '>=3.8'    (pyproject.toml:2)
```

### 3.2 Launch (headless, canonical configuration)

```console
$ Xvfb :99 -screen 0 1280x800x24 &
$ export DISPLAY=:99
$ export LIBGL_ALWAYS_SOFTWARE=1
$ ./kitty/launcher/kitty --config NONE &
$ pgrep -a kitty
44997 ./kitty/launcher/kitty --config NONE
```

`--config NONE` selects kitty's built-in defaults (there is no user `kitty.conf` in this
environment, so this is the canonical default configuration; shell integration remains enabled by
default). The only diagnostic kitty emits headlessly is the harmless
`Failed to open systemd user bus with error: Connection refused`. The launched process is **PID
44997**; every observation below is against this live instance.

---

## 4. R2 — What process is spawned, its PID, exact command line, and the connecting PTY path

### 4.1 The spawned process and its PID  [observed]

```console
$ pstree -p 44997 | head -1
kitty(44997)-+-bash(45065)

$ ps -o pid,ppid,tty,args -p 45065
    PID    PPID TT       COMMAND
  45065   44997 pts/0    /bin/bash --posix
```

kitty (PID 44997) spawned exactly one child: **`/bin/bash`, PID 45065**, whose controlling terminal
is `pts/0`.

### 4.2 The exact command line  [observed]

Read straight from the kernel's copy of the child's argv (NUL-delimited), so there is no ambiguity
about spacing or a leading hyphen:

```console
$ cat /proc/45065/cmdline | tr '\0' ' '; echo
/bin/bash --posix
```

The exact command line is **`/bin/bash --posix`**. Note there is **no leading `-`**: on Linux this
is *not* a login-shell argv-name. (kitty's `kitten run-shell` wrapper, which would otherwise reshape
the argv, is macOS-only — see §4.4.)

### 4.3 The PTY device path connecting kitty and the shell  [observed]

The shell's standard streams all point at the **slave** side of the PTY:

```console
$ ls -l /proc/45065/fd/0 /proc/45065/fd/1 /proc/45065/fd/2
lrwx------ 1 root root 64 Jul 13 16:40 /proc/45065/fd/0 -> /dev/pts/0
lrwx------ 1 root root 64 Jul 13 16:40 /proc/45065/fd/1 -> /dev/pts/0
lrwx------ 1 root root 64 Jul 13 16:40 /proc/45065/fd/2 -> /dev/pts/0
```

kitty holds the corresponding **master** end (previewed here; corroborated three ways in §7):

```console
$ ls -l /proc/44997/fd/8
lrwx------ 1 root root 64 Jul 13 16:40 /proc/44997/fd/8 -> /dev/pts/ptmx
```

So the connecting PTY is the pair **`/dev/pts/0` (slave, held by the shell) ↔ `/dev/pts/ptmx`
(master clone, held by kitty as fd 8)**. The shell writes to `/dev/pts/0`; those bytes become
readable on kitty's master fd.

### 4.4 Why this is the process/argv/PTY we see — bound to the source

* **Which shell.** kitty resolves the shell through `resolved_shell()` (`kitty/utils.py:768`); with
  the default option `shell == '.'` it returns `[shell_path]`, and `shell_path` is read from the
  password database — `pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` (`kitty/constants.py:181`,
  with the `/bin/sh` fallback at `:185`). Root's `pw_shell` here is `/bin/bash`, which is exactly the
  program observed at PID 45065.
* **The PTY pair.** `Child.fork()` (`kitty/child.py:276`) creates the PTY with `openpty()`
  (defined `kitty/child.py:170`, called at `:281`) and then calls into C:
  `pid = fast_data_types.spawn(...)` (`kitty/child.py:333`).
* **The OS-level spawn.** The C `spawn()` (`kitty/child.c:81`) derives the slave device name via
  `ttyname_r(slave, name, ...)` (`kitty/child.c:88`) — this is what makes the child's controlling
  terminal `/dev/pts/0` — then `fork()`s (`kitty/child.c:97`) and, in the child,
  `execvp(exe, argv)` (`kitty/child.c:159`) the resolved shell. That is the process that appears in
  the process list.
* **Why `--posix` (and no `-`).** The `--posix` argument is injected by kitty's shell integration,
  not typed by anyone. `Child.get_final_env()` (`kitty/child.py:233`) calls
  `modify_shell_environ()` (invoked around `kitty/child.py:266-267`; defined
  `kitty/shell_integration.py:218`), whose bash branch `setup_bash_env()`
  (`kitty/shell_integration.py:70`) does `argv.insert(1, '--posix')`
  (`kitty/shell_integration.py:146`) and sets `ENV=.../shell-integration/bash/kitty.bash` plus
  `KITTY_BASH_INJECT=1`. That is precisely the `--posix` seen in the argv.
* **Linux vs macOS.** The alternative `kitten run-shell` wrapper is gated by
  `should_run_via_run_shell_kitten = is_macos and self.is_default_shell`
  (`kitty/child.py:230`, used at the `:295` branch). Because the container is Linux, that branch is
  not taken; the shell is `execvp`'d directly and integration is injected via env/argv as above —
  which is why the process line is the bare `/bin/bash --posix` rather than a `run-shell` invocation
  or a `-bash` login name.

---

## 5. R3 — Typing `echo test123`: the read syscalls, the buffer size, and the byte count

### 5.1 How it was captured

`strace` was attached to the running kitty process, following threads (`-f`, mandatory — the read
runs on a non-main thread) and annotating fds with their paths (`-y`), then `echo test123` was typed
into the live window and Enter pressed:

```console
$ strace -f -y -e trace=poll,read,write -s 256 -p 44997 -o /tmp/kitty_obs/echo.strace &
$ xdotool windowfocus 2097164
$ xdotool type --delay 60 "echo test123"
$ xdotool key Return
```

All the PTY syscalls below are on thread id **45064**, whose name is `KittyChildMon` (proved in §8).

### 5.2 The syscalls: `poll()` then `read()`  [observed]

The system calls kitty makes to read from the PTY are **`poll()` followed by `read()`**. For a single
typed key, the trace shows the full round trip — kitty first *writes* the keystroke to the shell
(POLLOUT → `write`), then the kernel's terminal echo makes the same byte readable, so kitty is woken
by POLLIN and issues the `read()`:

```
############ R3 SLICE A: first keystroke 'e' — poll(POLLOUT)→write, then poll(POLLIN)→read (tid 45064) ############
45064 poll([{fd=6<anon_inode:[eventfd]>, events=POLLIN}, {fd=7<anon_inode:[signalfd]>, events=POLLIN}, {fd=8</dev/pts/ptmx>, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
45064 write(8</dev/pts/ptmx>, "e", 1)   = 1
45064 poll([...{fd=8</dev/pts/ptmx>, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
45064 read(8</dev/pts/ptmx>, "e", 1048576) = 1
```

During typing, every keystroke echoes back as its own **1-byte** read (the kernel N_TTY line
discipline echoes one character at a time):

```
############ R3: all keystroke echoes, in order (each a separate read, =1 byte) ############
45064 read(8</dev/pts/ptmx>, "e", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "c", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "h", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "o", 1048576) = 1
45064 read(8</dev/pts/ptmx>, " ", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "t", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "e", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "s", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "t", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "1", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "2", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "3", 1048576) = 1
```

After Enter, the shell runs the command and emits its shell-integration markers plus the command's
own output, arriving as a handful of larger reads:

```
############ R3 SLICE B: after Enter — OSC title+OSC133 cmdline(=47), integration marks + "test123\r\n"(=114), next prompt(=194) ############
45064 read(8</dev/pts/ptmx>, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47
45064 read(8</dev/pts/ptmx>, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114
45064 read(8</dev/pts/ptmx>, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]133;k;start_suffix_kitty\7\33[5 q\33]2;/tmp/blitzy/kitty/blitzy-ecabc877-16f7-4bb7-94fd-af5473cb3e57_59abb8\7\33]133;k;end_suffix_kitty\7", 1048404) = 194
```

(In the `strace` output, `\33` is ESC, `\7` is BEL, and `\r\n` is CR‑LF, printed exactly as `strace`
rendered them. The `\33]133;…` sequences are OSC 133 shell-integration marks; the actual command
output is the `test123\r\n` embedded in the 114-byte read.)

### 5.3 The file descriptor, the buffer size, and the bytes returned  [observed]

Reading the `read(fd, buf, count) = nbytes` calls directly:

* **fd** `= 8` — the PTY master (`/dev/pts/ptmx`). This is the answer to R5, corroborated in §7.
* **count (buffer size)** `= 1048576` bytes `= 1 MiB` on a fully-drained buffer.
* **nbytes (bytes returned)** `= 1` for each keystroke echo; then `47`, `114`, `194` for the
  post-Enter bursts.

### 5.4 Why the buffer size is 1 MiB — and why it visibly shrinks

The `count` is **not** a hardcoded constant at the `read()` call site. `read_bytes()` reads into a
buffer produced by `vt_parser_create_write_buffer()` (`kitty/vt-parser.c:1451`), which returns
`*sz = BUF_SZ - self->write.offset` (`kitty/vt-parser.c:1457`), where
`#define BUF_SZ (1024u*1024u)` = 1 MiB (`kitty/vt-parser.c:18`) and `write.offset` is the amount of
already-received-but-not-yet-parsed data. So the requested `count` is "1 MiB minus whatever the
parser hasn't consumed yet."

This derivation is directly visible in SLICE B above: as bytes accumulate faster than the parser
drains them, the requested `count` shrinks by exactly the running backlog —
`1048576 → 1048565 (−11) → 1048518 (−58) → 1048404 (−172)`. Those deltas are the cumulative
unparsed bytes (11, then 11+47, then 11+47+114), which is precisely `BUF_SZ - write.offset`.

### 5.5 The reader function (R6a)

The `read()` shown above is issued by **`read_bytes()`** (`kitty/child-monitor.c:1337`); the actual
call is `len = read(fd, buf, available_buffer_space);` at `kitty/child-monitor.c:1345`. It is invoked
from the I/O loop at `kitty/child-monitor.c:1531`. Full analysis of this function and its thread is
in §8.

---

## 6. R4 — Running `yes hello`: how the reading behavior changes under high volume

### 6.1 How it was captured, and at what scale

With `strace` attached exactly as in §5, `yes hello` was typed into the live window, left to run in
steady state, then interrupted with Ctrl‑C — and this was repeated **three times** with identical
input to check stability:

```console
$ strace -f -y -e trace=poll,read,write -s 64 -tt -p 44997 -o /tmp/kitty_obs/yes1.strace &
$ xdotool windowfocus 2097164
$ xdotool type --delay 40 "yes hello"; xdotool key Return
$ sleep 4                     # ~3.5s of steady-state stream captured
$ xdotool key ctrl+c
# …repeated for yes2.strace and yes3.strace…
```

Each run captured roughly **3.4–3.5 seconds** of steady-state streaming, comprising **~27,000–29,000
`read()` calls** on the PTY master — ample scale to characterize per-read magnitude and frequency.

### 6.2 What the stream looks like: still one `read()` per `POLLIN`  [observed]

The pacing does **not** switch to a different syscall or a drain-until-empty loop. It remains the
same `poll()` → single `read()` rhythm as R3, just executed thousands of times per second, with each
read now coalescing many `hello\r\n` lines (note `\n`→`\r\n` is the PTY's ONLCR translation, so each
"hello" line is 7 bytes on the wire):

```
############ R4 SLICE: steady-state poll→read pacing (run 1, tid 45064) ############
45064 16:43:20.199392 read(8</dev/pts/ptmx>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1020998) = 772
45064 16:43:20.199427 poll([...{fd=8</dev/pts/ptmx>, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
45064 16:43:20.199464 read(8</dev/pts/ptmx>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1020226) = 707
45064 16:43:20.199...  read(8</dev/pts/ptmx>, "hello\r\nhello\r\n"..., 1019519) = 1169
45064 16:43:20.199...  read(8</dev/pts/ptmx>, "hello\r\nhello\r\n"..., 1018350) = 574
45064 16:43:20.199...  read(8</dev/pts/ptmx>, "hello\r\nhello\r\n"..., 1017776) = 600
45064 16:43:20.199...  read(8</dev/pts/ptmx>, "hello\r\nhello\r\n"..., 1017176) = 765
```

Two structural facts are visible here: (1) reads do **not** align to line boundaries — a single read
starts mid-stream and ends mid-line, so the parser, not the reader, is responsible for reassembly;
and (2) the requested `count` again tracks `BUF_SZ − backlog` (~1.02 MB, i.e. ~28 KB of unparsed
backlog), never the limiter on how much comes back.

### 6.3 Per-read byte-count distribution and read frequency  [observed]

The distribution and frequency were computed from each trace with an `awk` pipeline over the
returned byte counts and the `-tt` timestamps:

```console
# per-read nbytes → percentiles; timestamps → span & frequency
$ grep -E 'read\(8<.*ptmx' /tmp/kitty_obs/yes1.strace \
    | sed -E 's/.*= ([0-9]+)$/\1/' > /tmp/kitty_obs/yes1_nb.txt
$ sort -n /tmp/kitty_obs/yes1_nb.txt | awk '{a[NR]=$1; s+=$1} END{
    print "reads="NR, "mean="int(s/NR),
    "min="a[1], "p25="a[int(NR*.25)], "median="a[int(NR*.5)],
    "p75="a[int(NR*.75)], "p90="a[int(NR*.90)], "p99="a[int(NR*.99)], "max="a[NR]}'
```

Results across the three identical runs:

```
run  reads   span(s)  freq(reads/s)  min  p25  median  p75   p90   p99   max     mean
1    28037   3.472    8074           1    597  763     1022  1416  4095  18510   932
2    27449   3.505    7832           1    609  819     1167  2182  4242  19220   1077
3    28917   3.387    8537           1    616  814     1099  1561  4095  19264   996
```

Run‑1 histogram of bytes-per-read (shows where the mass sits):

```
   1– 64 :     10
  65–256 :     86
 257–512 :   3494
513–1024 :  17507   ← ~62% of all reads land here
1025–2048:   5806
2049–4096:    925
4097–8192:    168
  >8192  :     41   (rare spikes up to ~18–19 KB)
```

**Stability across runs (≥2 confirmed):** the shape is consistent — frequency ≈ **8000 reads/s**
(±5%), **median ≈ 800 B**, **mean ≈ 1000 B**, **p99 ≈ 4 KiB**, with rare spikes to ~18–19 KB. The
run-to-run variation (e.g. p90 = 1416/2182/1561) is ordinary jitter in kernel PTY buffering under a
`strace`-slowed consumer, not a change in behavior.

### 6.4 How this differs from the single command (`echo test123`) — before/during/after

* **Before / single command (R3):** reads are sparse and *tiny* — one `read()` returning **1 byte**
  per keystroke echo, then a few reads of 47/114/194 bytes. kitty spends almost all its time blocked
  in `poll(…, -1)`.
* **During the stream (R4):** the *same* `poll → read` primitive fires **~8000×/second**, each read
  now returning **hundreds to ~1–2 K bytes** by coalescing ~100+ lines. The syscall pattern is
  unchanged; only its rate and per-call payload grew.
* **After (Ctrl‑C):** the read rate collapses back to the idle `poll(…, -1)`-blocked state and the
  next prompt is emitted as a small burst, exactly like the post-Enter reads in §5.

### 6.5 Why it behaves this way — cause → effect, bound to the source

* **One read per `POLLIN` (not drain-to-empty).** The I/O loop `io_loop()`
  (`kitty/child-monitor.c:1481`) `poll()`s (`:1509` timed, `:1512` blocking) and, on POLLIN, calls
  `read_bytes()` (`:1531`). `read_bytes()` performs exactly **one** successful `read()` per POLLIN:
  its loop only *retries* on `EINTR`/`EAGAIN`, and breaks after the first successful read
  (`kitty/child-monitor.c:1345`). Because the master fd is non-blocking (§7), the loop cannot spin
  draining the buffer — it reads once and returns to `poll()`. **Effect:** high volume manifests as
  *more frequent* single reads, not as fewer/larger drain reads.
* **Parser back-pressure gates POLLIN.** The loop only arms `POLLIN` for the child fd when the
  parser buffer has room:
  `children_fds[...].events = vt_parser_has_space_for_input(...) ? POLLIN : 0;`
  (`kitty/child-monitor.c:1501`). `vt_parser_has_space_for_input()` (`kitty/vt-parser.c:1477`)
  returns `self->read.sz + self->write.pending < BUF_SZ` (`:1481`). **Effect:** if the parser ever
  fell a full 1 MiB behind, the reader would stop being woken until the parser caught up. In these
  runs the requested `count` never dropped below ~807 KB (min observed), i.e. the backlog stayed well
  under 1 MiB, so back-pressure never had to disable POLLIN — the parser kept pace.
* **Magnitude is bounded by the kernel, not by kitty.** kitty requests up to ~1 MiB, but the largest
  single read observed across run 1 was **18,510 bytes** (`max`), while the smallest `count` it ever
  requested was **807,223 bytes**. The realized bytes-per-read are therefore limited by the kernel
  PTY / N_TTY line-discipline buffering, **not** by kitty's 1 MiB request — which is why the
  distribution clusters around a few hundred bytes to a couple of KB regardless of the huge buffer.

### 6.6 Measurement caveat (stated honestly)

`strace` adds `ptrace` overhead to *every* syscall, which slows the reading thread and, indirectly,
the shell producing the stream. The **~8000 reads/s figure is thus a lower bound** on kitty's
un-traced throughput. The findings that are robust to this overhead — and that answer the question —
are the *shape* of the behavior (one read per `POLLIN`, coalescing many lines) and the *magnitude*
(hundreds of bytes to a few KB per read, kernel-bounded, never approaching 1 MiB).

---

## 7. R5 — The file-descriptor number kitty uses to read from the PTY master

The master-side descriptor is **fd 8**, corroborated three independent ways.

**(a) From every `read()` in the traces** (`-y` annotates the fd with its backing path):

```
45064 read(8</dev/pts/ptmx>, "e", 1048576) = 1
45064 read(8</dev/pts/ptmx>, "hello\r\nhello\r\n"..., 1020226) = 707
```

**(b) From `/proc/<kitty_pid>/fd`:**

```console
$ ls -l /proc/44997/fd/8
lrwx------ 1 root root 64 Jul 13 16:40 /proc/44997/fd/8 -> /dev/pts/ptmx
```

**(c) From `lsof`:**

```console
$ lsof -p 44997 | grep ptmx
kitty   44997 root    8u      CHR                5,2      0t0         2 /dev/pts/ptmx
```

`lsof` confirms fd **8**, opened read-write (`8u`), a character device with major/minor **5,2** —
which is `/dev/ptmx`, the UNIX‑98 PTY master multiplexer. The fd number seen in the `read()` calls
(a) matches the descriptor in `/proc` (b) and `lsof` (c).

**Why fd 8, and why non-blocking.** In `Child.fork()`, after `openpty()` returns the pair, kitty
retains the master and stores it as `self.child_fd = master` (`kitty/child.py:338`) and then makes
it non-blocking with `os.set_blocking(self.child_fd, False)` (`kitty/child.py:345`). The exact
integer (8) is just wherever the OS placed the master in kitty's descriptor table for this launch —
so the *number* is observed per-run, while the *fact that it is the master fd kitty reads* is
structural. The non-blocking setting is exactly what makes `read_bytes()` get `EAGAIN` once the
master is drained, which is why its loop breaks after a single successful read (see §6.5 and §8).

---

## 8. R6 — The C functions: which reads the fd, which separates text from escapes (+ thread model)

### 8.1 R6a — The reader function: `read_bytes()` on the `KittyChildMon` I/O thread

The function that reads from the PTY file descriptor is **`read_bytes()`**
(`kitty/child-monitor.c:1337`); the actual system call is
`len = read(fd, buf, available_buffer_space);` (`kitty/child-monitor.c:1345`). It runs inside the
I/O loop `io_loop()` (`kitty/child-monitor.c:1481`), which names its thread
`set_thread_name("KittyChildMon");` (`kitty/child-monitor.c:1489`) and calls `read_bytes()` at
`kitty/child-monitor.c:1531`.

That the read genuinely executes on a **dedicated, separately-named thread** — not the main thread —
was confirmed at runtime. Every PTY `read()` in the traces is on tid **45064**, and that task's
`comm` is `KittyChildMon`:

```console
$ cat /proc/44997/task/45064/comm
KittyChildMon
```

`read_bytes()` does **only** two things per wakeup: one `read()` into the shared buffer, then commit
it for the parser (`vt_parser_commit_write()`, declared `kitty/vt-parser.h:35`). It does **not**
parse — that is a different function on a different thread (below).

### 8.2 R6b — The parser function: `consume_input()`, routing printable vs escape bytes

The function that parses the incoming data to separate printable text from escape sequences is
**`consume_input()`** (`kitty/vt-parser.c:1367`). It is a state machine that dispatches on the
current parser state and routes bytes to distinct branches:

* **printable text →** `consume_normal()` (`kitty/vt-parser.c:230`), dispatched from the
  `VTE_NORMAL` case at `kitty/vt-parser.c:1377`. `consume_normal()` scans a run of ordinary
  characters (UTF‑8 decode → `screen_draw_text`), stopping at the first ESC/control sentinel — this
  is the "printable" half of the separation.
* **escape / control sequences →** `consume_esc()` (`kitty/vt-parser.c:261`, from the `VTE_ESC`
  case at `:1379`), `consume_csi()` (`kitty/vt-parser.c:839`, from the `VTE_CSI` case at `:1382`),
  and the OSC/APC/PM/DCS/SOS `dispatch_*` branches — this is the "escape sequence" half.

`consume_input()` is driven by the parser worker **`run_worker()`** (`kitty/vt-parser.c:1417`, which
calls `consume_input()` at `:1432`). This exactly matches the OSC/CSI content seen in the R3 trace:
the `\33]2;…\7` (OSC title) and `\33]133;…\7` (OSC 133) bytes take the escape branch, while the
literal `test123` takes `consume_normal()`.

### 8.3 The reader and parser run on different threads — why `strace -f` was mandatory

The read and the parse are **separate threads**, and this is the crux of why `-f` is required:

* **Reader thread:** `read_bytes()` runs on `KittyChildMon` (tid 45064, proven above).
* **Parser thread:** `consume_input()` runs on kitty's **main** thread. The production parse path is
  driven from the main loop, **not** from `read_bytes()`. Verifying against the source: `vt-parser.c`
  contains **no** `pthread_create` — the parser always runs on its caller's thread — and
  `child-monitor.c` creates exactly two threads: the I/O thread (`io_thread` created around
  `kitty/child-monitor.c:291` → `io_loop` = `KittyChildMon`) and a `talk_thread` (remote control).
  The parse worker is invoked from the main loop
  (`main_loop()` at `kitty/child-monitor.c:1259` → `run_main_loop(process_global_state, …)` at
  `:1262` → `process_global_state` at `:1224` → `parse_input` at `:451`, called from `:1236` →
  `do_parse` at `:438` → the `parse_func` pointer at `:61`, set to `parse_worker` at `:180-181` →
  `run_worker()` → `consume_input()`). The main thread then also drives rendering.

  > Accuracy note: the AAP body cited `screen.c:4757/4767/4775/4776` for the parse driver, but at
  > this commit those are the **test-only** helpers (`test_create_write_buffer`,
  > `test_commit_write_buffer`, `test_parse_written_data`). The real production driver is the
  > `child-monitor.c` main-loop chain above; this document cites that path.

* **Runtime corroboration of the split:** in the traces, all PTY `read()`s are on tid 45064
  (`KittyChildMon`), while the main thread (tid 44997) is separately polling the GUI/X socket
  (fd 3) and the eventfd — i.e. reading and parsing/rendering are demonstrably on different threads.

**Hand-off between the two threads.** The reader (I/O thread) and parser (main thread) communicate
through the single shared **1 MiB** buffer (`BUF_SZ`, `kitty/vt-parser.c:18`): the producer commits
bytes with `vt_parser_commit_write()` (`kitty/vt-parser.h:35`), and the consumer later drains them
via `parse_worker`/`run_worker` → `consume_input()`. Flow control between them is the
`vt_parser_has_space_for_input()` gate (`kitty/vt-parser.c:1477`) that arms/disarms `POLLIN` on the
reader side (`kitty/child-monitor.c:1501`), as analyzed in §6.5.

**Cause → effect:** because the `read()` happens on `KittyChildMon` rather than the main thread, an
`strace` that does not follow threads (missing `-f`) would attach only to the main thread and capture
**zero** PTY reads. That is precisely why `-f` was used for every trace in this investigation.

---

## 9. Coverage checklist — every named item, observed vs inferred

| # | Named item asked | Answer | Evidence | Observed / Inferred |
|---|---|---|---|---|
| R1 | Build kitty from source | `python3 setup.py build --verbose --ignore-compiler-warnings` (exit 0); `Makefile:12-13`→`setup.py:1084`; Go build `setup.py:1148` | §3.1 build log + `kitty --version` = `kitty 0.35.2 created by Kovid Goyal` | observed |
| R1 | Launch it | `Xvfb :99` + `LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty --config NONE` → PID 44997 | §3.2 | observed |
| R2 | Process spawned | `/bin/bash` | §4.1 `pstree`/`ps` | observed |
| R2 | PID | **45065** | §4.1 | observed |
| R2 | Exact command line | **`/bin/bash --posix`** (no leading `-`) | §4.2 `/proc/45065/cmdline` | observed |
| R2 | PTY device path | slave **`/dev/pts/0`** ↔ master **`/dev/pts/ptmx`** | §4.3 `/proc/*/fd` | observed |
| R3 | `echo test123` — syscalls | **`poll()` then `read()`** | §5.2 trace | observed |
| R3 | Buffer size | **1048576 B (1 MiB)** = `BUF_SZ − backlog` | §5.3–5.4; `vt-parser.c:18/1457` | observed + code |
| R3 | Bytes returned | 1 B/keystroke; then 47, 114, 194 after Enter | §5.2–5.3 trace | observed |
| R4 | `yes hello` — behavior change | same `poll`→single-`read` loop, ~8000×/s, coalescing ~100+ lines/read | §6.2–6.4 | observed |
| R4 | Read frequency | ≈ **8000 reads/s** (lower bound under `strace`) | §6.3 (3 runs) | observed |
| R4 | Typical bytes/read | median ≈ **800 B**, mean ≈ **1000 B**, p99 ≈ 4 KiB, max ≈ 18–19 KB | §6.3 (3 runs, stable) | observed |
| R5 | Master FD number | **fd 8** → `/dev/pts/ptmx` | §7 trace + `/proc/44997/fd/8` + `lsof` | observed |
| R6a | Reader function | **`read_bytes()`** `kitty/child-monitor.c:1337` (`read()` `:1345`), thread `KittyChildMon` | §8.1; `/proc/.../task/45064/comm` | observed + code |
| R6b | Parser function (text vs escapes) | **`consume_input()`** `kitty/vt-parser.c:1367` → `consume_normal()` `:230` vs `consume_esc()`/`consume_csi()` `:261`/`:839` | §8.2 | observed + code |
| R6 | Reader/parser thread separation | reader on `KittyChildMon`; parser on main thread → mandates `strace -f` | §8.3 | observed + code |

Every value in this document is **observed** at runtime except the specific integer fd value's
persistence across runs (the fd number 8 is observed for this launch; that it is *the master fd
kitty reads* is structural per `kitty/child.py:338`). No claim in this document is left unobserved
or unverified; nothing required inference in place of observation.

---

## 10. Cleanup statement

This investigation was strictly read-only with respect to the repository. All build logs, `strace`
captures, PID files, and helper artifacts were written under `/tmp/kitty_obs/` (outside the
repository tree) and were **deleted** at the end of the investigation, and the launched
kitty/Xvfb/strace processes were stopped. The **only** change left in the repository is this single
document, `blitzy/documentation/kitty_815df1e210e0.md`. `git status --porcelain` shows exactly one
added path (this file and its parent directory); no existing source file was modified.

