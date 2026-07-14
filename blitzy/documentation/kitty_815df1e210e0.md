# How kitty Talks to Its Shell Over a PTY — A Run-First Investigation

**Repository:** [kovidgoyal/kitty](https://github.com/kovidgoyal/kitty)
**Source branch:** `kitty_815df1e210e0` · **HEAD commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Platform:** Linux (Ubuntu 25.10 container, image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`)
**Runtimes observed:** CPython **3.13.7**, Go **1.24.4**, gcc **15.2.0**; built **kitty 0.35.2**

> **Note on the runtime version.** The task brief predicted CPython 3.12.3; the container this investigation actually ran in provides **CPython 3.13.7**. All values below are the *observed* ones. 3.13.7 comfortably satisfies kitty's `requires-python = ">=3.8"` [pyproject.toml:L2].

## Methodology (Run-First)

Every behavioral answer below was produced by **building and running kitty from source and observing the live process**, not by reading code alone. The runtime pipeline was captured with the Linux syscall tracer `strace` attached to the running kitty process:

```bash
strace -f -yy -tt -T -e trace=read,poll -p <kitty_pid> -o <logfile>
```

* `-f` — **follow threads.** kitty performs its PTY reads on a dedicated I/O thread (`io_loop`, thread name `KittyChildMon` [kitty/child-monitor.c:L1481, L1489]), *not* the main thread, so tracing without `-f` would capture zero PTY reads.
* `-yy` — annotate every file descriptor with its backing path (this is what reveals which fd is the `/dev/pts/ptmx` master — see Q5).
* `-tt` / `-T` — per-call wall-clock timestamps and durations (used to measure read frequency in Q4).
* `-e trace=read,poll` — restrict the trace to the two syscalls of interest.
* `-c` (used in a separate run) — a per-syscall count/time histogram, used to quantify magnitude in Q4.

Genuine keyboard input (`echo test123`, `yes hello`) was injected into the real GUI window with `xdotool` (synthetic X `KeyPress` events → kitty's GLFW key handler → kitty writes to the PTY master → the shell). **No remote-control hook (`kitty @`), debug interface, mock, or fallback was used.** The kitty source tree was **not modified**; the only new file in the repository is this document.

Conventions used throughout:
* Each claim is followed by the **exact command run** and its **unedited output**.
* Statements that could not be observed at runtime and are instead derived from reading the source are explicitly tagged **`(inferred from code: file:line)`**.
* Environment adjustments that are *not* repository changes (installing `strace`/`xdotool`, setting `ptrace_scope=0`, starting `Xvfb`) are noted where relevant.

---

## Q1 — Build kitty from source and launch it

**Direct answer.** kitty was built with its canonical `setup.py` driver and launched from the produced launcher binary:

* **Build command:** `python3 setup.py --ignore-compiler-warnings` (the canonical `all:` target is `python3 setup.py $(VVAL)` [Makefile:L12-L13]).
* **Launcher produced:** `kitty/launcher/kitty`.
* **Launch command (headless):** `./kitty/launcher/kitty` under a virtual display (`Xvfb :99`), default configuration.
* **Running kitty PID:** `47973`.
* **Result:** `kitty 0.35.2`, built against CPython 3.13.7 / gcc 15.2.0.

### Runtime versions

```bash
python3 --version; go version; gcc --version | head -1
```

```
Python 3.13.7
go version go1.24.4 linux/amd64
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

The minimum Python version is enforced at build time by `check_version_info()` [setup.py:L30], called at [setup.py:L47], which reads `requires-python = ">=3.8"` [pyproject.toml:L2]. *(Curiosity, inferred from code: the failure message string in `check_version_info()` reads "calibre requires Python …" — an upstream copy-paste artifact; the enforced minimum is still Python ≥ 3.8.)* The Go toolchain (`go 1.22` [go.mod:L3]) builds the separate **`kitten`** Go binary; **it is not part of the C PTY/parser read path** exercised in Q3–Q6.

### Build (canonical)

```bash
CI=true python3 setup.py --ignore-compiler-warnings
```

First and last lines of the build (122 C translation units, then 5 link steps):

```
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
...
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
```

Note that the two C files central to this investigation are compiled here: `kitty/child-monitor.c` (unit 7/122) and `kitty/vt-parser.c` (units 10–11/122).

**Deviation from the bare canonical command, reported explicitly.** The truly bare invocation `python3 setup.py` (no flag) **fails** on this newer toolchain, because kitty builds its C with `-Werror` and the system `wayland-protocols` (1.45) has added `xdg-shell` enum values the pinned GLFW source does not handle:

```bash
CI=true python3 setup.py          # no --ignore-compiler-warnings
```

```
[3/122] Compiling [wayland] glfw/wl_window.c ...
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
```

`--ignore-compiler-warnings` is kitty's **own official** setup.py flag for exactly this situation; using it changes **no source file** and only relaxes `-Werror`. The failure is in the Wayland windowing backend (`glfw/wl_window.c`), which is unrelated to the PTY/parser read path under study.

### Launch (headless, default configuration)

Because kitty is a GLFW/OpenGL GUI application and the container has no physical display, it was launched under a virtual X display with Mesa software rendering (`llvmpipe`):

```bash
Xvfb :99 -screen 0 1280x1024x24 +extension GLX +render -noreset &
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
./kitty/launcher/kitty          # default config: no user kitty.conf present
KPID=$(pgrep -f 'kitty/launcher/kitty' | head -1); echo "kitty PID=$KPID"
ps -o pid,ppid,nlwp,comm,cmd -p "$KPID"
```

```
kitty PID=47973
    PID    PPID NLWP COMMAND         CMD
  47973       1   67 kitty           ./kitty/launcher/kitty
```

No user `kitty.conf` exists (`~/.config/kitty/kitty.conf` absent, `KITTY_CONFIG_DIRECTORY` empty), so this launch uses kitty's **built-in defaults**, including default `shell_integration`. The only message on kitty's stderr was benign:

```
[0.161] Failed to open systemd user bus with error: Connection refused
```

This is the container having no systemd *user* session; it corresponds to the best-effort `systemd_move_pid_into_new_scope()` call [kitty/child.py:L349] failing gracefully. A useful side effect (confirmed in Q2) is that the shell therefore remains a **direct child** of kitty.

---

## Q2 — What process does kitty spawn for the shell? PID, exact command line, and PTY device path

**Direct answer.**

| Item | Observed value |
|------|----------------|
| Spawned process | **`/bin/bash --posix`** |
| PID | **`48041`** (direct child of kitty PID 47973) |
| Exact command line | **`/bin/bash --posix`** (argv = `["/bin/bash", "--posix"]`) |
| PTY device path (shell side) | **`/dev/pts/0`** (the slave; the shell's stdin/stdout/stderr) |

### Evidence

```bash
ps --ppid 47973 -o pid,ppid,cmd          # kitty's children
ps -o pid,ppid,cmd -p 48041              # the shell
cat /proc/48041/cmdline | tr '\0' ' '    # human-readable argv
xxd /proc/48041/cmdline                  # show the NUL delimiters
ls -l /proc/48041/fd/0 /proc/48041/fd/1 /proc/48041/fd/2   # the PTY it is attached to
```

```
    PID    PPID CMD
  48041   47973 /bin/bash --posix

/bin/bash --posix

00000000: 2f62 696e 2f62 6173 6800 2d2d 706f 7369  /bin/bash.--posi
00000010: 7800                                     x.

lrwx------ 1 root root 64 Jul 14 19:18 /proc/48041/fd/0 -> /dev/pts/0
lrwx------ 1 root root 64 Jul 14 19:18 /proc/48041/fd/1 -> /dev/pts/0
lrwx------ 1 root root 64 Jul 14 19:18 /proc/48041/fd/2 -> /dev/pts/0
```

The `xxd` dump proves the exact argv structure: the bytes are `/bin/bash\0--posix\0`, i.e. `argv[0] = "/bin/bash"` and `argv[1] = "--posix"` (the NUL bytes `00` are the argument delimiters). All three of the shell's standard descriptors point at **`/dev/pts/0`** — the slave end of the pseudo-terminal. (kitty holds the *master* end; that is Q5.)

### Why this is the process, PID, argv, and PTY — cause → effect against the code

**Which shell.** With kitty's default `shell` option (`.`), `resolved_shell()` returns `[shell_path]` — the single default-shell case:

```
kitty/utils.py:L768   def resolved_shell(opts=...):
kitty/utils.py:L770       if q == '.':
kitty/utils.py:L771           ans = [shell_path]
```

`shell_path` is the invoking user's login shell (or `/bin/sh` as a fallback):

```
kitty/constants.py:L181   shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'
kitty/constants.py:L185   shell_path = '/bin/sh'      # fallback on KeyError
```

The investigation ran as `root`, whose passwd entry is `root:x:0:0:root:/root:/bin/bash`, so `shell_path = /bin/bash` — matching the observed `argv[0]`.

**Why `--posix`, and why it is *not* a deviation.** `argv[1] = "--posix"` is injected by kitty's **default** bash shell-integration, not typed by a user. `modify_shell_environ()` [kitty/shell_integration.py:L218] dispatches to the per-shell modifier `setup_bash_env()`, which inserts the flag:

```
kitty/shell_integration.py:L146   argv.insert(1, '--posix')
```

kitty then points bash's `ENV`/`KITTY_BASH_POSIX_ENV` at its integration script so the shell sources kitty's bash integration while running in POSIX mode. Because default `shell_integration` is *enabled*, `/bin/bash --posix` is the **canonical default** argv on Linux — no configuration was changed to produce it.

**How the PTY and the process are created (spawn path).** The Python `Child` object opens the pty and forks/exec's the shell:

```
kitty/child.py:L171   master, slave = os.openpty()
kitty/child.py:L276   def fork(self):
kitty/child.py:L333   pid = fast_data_types.spawn(...)
```

`fast_data_types.spawn()` is the C function `spawn()` [kitty/child.c:L80-L81], which obtains the **slave device path** (the `/dev/pts/0` seen above) via `ttyname_r()`, forks, creates a new session, sets the controlling terminal, wires the slave to the child's stdio, and finally `execvp()`s the shell:

```
kitty/child.c:L88    if (ttyname_r(slave, name, sizeof(name) - 1) != 0) { ... }   # slave PTY path
kitty/child.c:L97    pid_t pid = fork();
kitty/child.c:L123   if (setsid() == -1) ...
kitty/child.c:L129   if (ioctl(tfd, TIOCSCTTY, 0) == -1) ...
kitty/child.c:L138   if (safe_dup2(slave, STDOUT_FILENO) == -1) ...    # slave -> child stdio
kitty/child.c:L159   execvp(exe, argv);
```

**Linux vs macOS divergence (cause → effect).** On Linux the shell appears as the **plain resolved path** `/bin/bash`, *not* a login-style `-bash` under `/usr/bin/login`. That is because the login-shell wrapper is gated on macOS only:

```
kitty/child.py:L230   self.should_run_via_run_shell_kitten = is_macos and self.is_default_shell
kitty/child.py:L295   if self.should_run_via_run_shell_kitten:   # macOS-only login wrapper block
```

Since `is_macos` is False here, `should_run_via_run_shell_kitten` is False, so the block at [kitty/child.py:L295] is skipped and kitty exec's `/bin/bash` directly (plus the shell-integration `--posix`). This is exactly what the process list shows.

---

## Q3 — Reading a small input: typing `echo test123`

**Direct answer.** To read the shell's output, kitty's I/O thread issues a **`poll()`** on the master fd and, when it returns `POLLIN`, a **`read()`** on that same fd:

* **System calls:** `poll(...)` → `read(8, buf, 1048576)` on fd `8` (`/dev/pts/ptmx`), both on the `KittyChildMon` I/O thread (TID 48040).
* **Requested buffer size:** **1048576 bytes** (exactly 1 MiB) whenever the parser buffer is empty; it shrinks to `1048576 − pending` when unconsumed bytes remain (values `1048565`, `1048518`, `1048404` seen below).
* **Bytes returned:** **small.** Each per-keystroke echo returned **1** byte; after Enter the command's output + shell-integration sequences arrived in reads of **11, 47, 114, and 429** bytes. 16 `read()`s on the master were captured for the whole `echo test123` episode.

### Evidence

```bash
# tracer already attached: strace -f -yy -tt -T -e trace=read,poll -p 47973 -o echo.strace
xdotool type --window <win> --delay 45 'echo test123'
xdotool key  --window <win> Return
```

The `poll()`+`read()` pairs immediately after Enter (verbatim from `echo.strace`):

```
48040 19:20:07.056462 poll([{fd=6<{eventfd-count=0, eventfd-id=890, eventfd-semaphore=0}>, events=POLLIN}, {fd=7<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}]) <0.000014>
48040 19:20:07.056538 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\n\33[?2004l\r", 1048576) = 11 <0.000015>
48040 19:20:07.057959 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33]2;echo test123\7\33]133;C;cmdline"..., 1048565) = 47 <0.000019>
48040 19:20:07.058155 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1\33]133;k;start_kitty\7\2\1\33]133;k;e"..., 1048518) = 114 <0.000011>
48040 19:20:07.058919 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\33[?2004h\33[59P\33]133;k;start_kitty"..., 1048404) = 429 <0.000017>
```

The character-by-character echo (I typed with a 45 ms inter-key delay) shows one-byte reads, e.g.:

```
48040 19:20:06.681263 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "1", 1048576) = 1 <0.000012>
48040 19:20:06.704151 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "2", 1048576) = 1 <0.000014>
48040 19:20:06.727140 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "3", 1048576) = 1 <0.000014>
```

Requested-size distribution over the 16 master reads of this episode:

```
     12   1048576          # empty parser buffer: full BUF_SZ
      1   1048565          # = 1048576 - 11  (11 bytes pending)
      1   1048518          # = 1048576 - 58  (11+47 pending)
      1   1048404          # = 1048576 - 172 (11+47+114 pending)
```

### Rationale — cause → effect

* The `poll()` monitors exactly three descriptors — a wakeup `eventfd` (fd 6), a `signalfd` (fd 7), and the PTY master (fd 8) — i.e. `self->count (=1 child) + EXTRA_FDS`; this is the timed/blocking `poll()` in the I/O loop [kitty/child-monitor.c:L1509 (timed), L1512 (blocking, `-1`)]. When the shell writes, `poll` returns `revents=POLLIN` on fd 8.
* The following `read()` is the one syscall at [kitty/child-monitor.c:L1345] (`len = read(fd, buf, available_buffer_space);`), inside the reader `read_bytes()` [kitty/child-monitor.c:L1337].
* The **requested size** is `available_buffer_space`, computed by `vt_parser_create_write_buffer()` as `*sz = BUF_SZ - self->write.offset` [kitty/vt-parser.c:L1457], where `BUF_SZ` is `1024u*1024u = 1048576` [kitty/vt-parser.c:L18]. On an empty buffer the offset is 0, so the request is the full **1048576** — exactly the third argument observed. As unconsumed bytes accumulate ahead of the consumer thread, the offset grows and the request shrinks by that many bytes (the 1048565 / 1048518 / 1048404 values are `1048576` minus the running pending total), directly demonstrating the sizing formula.
* The **bytes returned** are small because `echo test123` produces very little output: the literal command echo, the `test123` line, and the surrounding shell-integration OSC sequences (e.g. `\33]2;echo test123\7` sets the window title; the `\33]133;…` markers are OSC-133 semantic-prompt codes). The largest single read of the episode was 429 bytes — four orders of magnitude below the 1 MiB requested.

---

## Q4 — Reading a high-volume stream: running `yes hello`

**Direct answer.** Under a sustained stream, kitty's behavior does **not** change to one big read — it performs **many repeated small reads**, each far below the 1 MiB it requests:

* **Read cadence:** on the order of **~6,000 reads per second** on the master fd (Run 1: **6,576/s**, Run 2: **5,820/s**).
* **Typical bytes per read:** **~1.1–1.3 KiB** (Run 1 median **1,118** bytes, Run 2 median **1,302** bytes; means 1,339 / 1,562), with a tail up to ~20–22 KiB — i.e. **far below the 1,048,576 bytes requested** each time.
* Every read during the burst returned data (0 `EAGAIN`, 0 zero-length).

**Scale / duration:** each run streamed `yes hello` for a **~5.25 s** window (measured from the first to the last master read in the trace), stopped with a genuine `Ctrl-C`. Two independent runs with identical tracing were used to confirm stability, plus a third `-c` histogram run for aggregate magnitude.

### Evidence — Run 1 (timed detail)

```bash
strace -f -yy -tt -T -e trace=read,poll -p 47973 -o yes1.strace &
xdotool type --window <win> --delay 45 'yes hello'; xdotool key --window <win> Return
sleep 5; xdotool key --window <win> ctrl+c        # stop `yes`
```

Representative consecutive master reads (verbatim from `yes1.strace`) — note the `hello\r\n` payload, the ~1 KiB return values, and the request size shrinking as the buffer fills:

```
48040 19:22:03.652225 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1016831) = 971 <0.000011>
48040 19:22:03.652336 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1015860) = 947 <0.000022>
48040 19:22:03.652501 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1014913) = 1510 <0.000015>
48040 19:22:03.652626 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1013403) = 1003 <0.000015>
48040 19:22:03.652751 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1012400) = 873 <0.000009>
48040 19:22:03.652880 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1011527) = 772 <0.000010>
48040 19:22:03.652991 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1010755) = 943 <0.000015>
48040 19:22:03.653118 read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1009812) = 968 <0.000015>
… (thousands of similar ~1 KiB reads over ~5.25 s omitted) …
```

Aggregate for Run 1 (computed from the trace):

```
data-returning read(8) calls = 34549   EAGAIN = 0   zero-length = 0
streaming window            = 5.254 s   (first→last master read timestamp)
read frequency              = 6576 reads/s
bytes/read: median = 1118   mean = 1339   min = 1   max = 20888
```

### Evidence — Run 2 (independent, identical tracing → stability check)

```
data-returning read(8) calls = 30588   EAGAIN = 0   zero-length = 0
streaming window            = 5.256 s
read frequency              = 5820 reads/s
bytes/read: median = 1302   mean = 1562   min = 1   max = 22181
```

| Metric | Run 1 | Run 2 |
|--------|------:|------:|
| master `read()` calls | 34,549 | 30,588 |
| streaming window (s) | 5.254 | 5.256 |
| **read frequency (reads/s)** | **6,576** | **5,820** |
| **bytes/read (median)** | **1,118** | **1,302** |
| bytes/read (mean) | 1,339 | 1,562 |
| bytes/read (max) | 20,888 | 22,181 |
| bytes/read (min) | 1 | 1 |

The two runs agree: **~6,000 reads/s** and a **~1.1–1.3 KiB typical read**, always three orders of magnitude below the 1 MiB requested. The values are stable.

### Evidence — Run 3 (`-c` histogram, aggregate magnitude)

```bash
strace -f -y -c -e trace=read,poll -p 47973 -o yes3.count &
# type `yes hello`, stream ~5 s, Ctrl-C, then SIGINT strace to emit its summary
```

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.85    0.704997          11     59559           poll
 49.15    0.681300          11     56913       431 read
------ ----------- ----------- --------- --------- ----------------
100.00    1.386297          11    116472       431 total
```

Over ~5 s kitty issued **56,913 `read()`** and **59,559 `poll()`** calls (these totals span *all* descriptors — the master, the wakeup `eventfd`, the X socket — so they exceed the master-only counts above; the 431 `read` "errors" are `EAGAIN` retries on the non-PTY fds). The magnitude confirms the qualitative finding: a high-volume stream is drained by **tens of thousands of small reads**, not a handful of large ones.

### Rationale — cause → effect (why many small reads, not one big one)

1. **The kernel PTY line-discipline buffer caps each read.** kitty *asks* for up to `BUF_SZ − offset` ≈ 1,048,576 bytes [kitty/vt-parser.c:L1457, L18], but a `read()` on a PTY master can only return what the kernel's terminal (N_TTY line-discipline) buffer currently holds — on the order of a few kilobytes. A fast producer like `yes` refills that small kernel buffer continuously, so kitty drains it in ~1 KiB chunks. **The per-read size is bounded by the kernel buffer, not by kitty's 1 MiB buffer** — this is the crux, and it is why the returned counts (~1 KiB) are so much smaller than the requested count (1 MiB).
2. **Backpressure.** The I/O thread only asks for `POLLIN` on a child fd when the parser still has room:
   ```
   kitty/child-monitor.c:L1501   children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
   kitty/vt-parser.c:L1477        vt_parser_has_space_for_input(const Parser *p) { ...
   kitty/vt-parser.c:L1481            ans = self->read.sz + self->write.pending < BUF_SZ;
   ```
   If the parser/render side falls behind and the 1 MiB buffer fills, `POLLIN` is withheld and reads pause until space frees. *(Note: the backpressure gate is at line **1501** in this checkout; line 1500 is a commented-out `printf`.)*
3. **`input_delay` coalescing.** The loop uses a *timed* `poll()` whose timeout is the remaining `input_delay` budget, which lets several kernel writes coalesce before the consumer is woken:
   ```
   kitty/child-monitor.c:L1508   monotonic_t time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at);
   kitty/child-monitor.c:L1509   if (time_delta >= 0) ret = poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta));
   ```
   This is visible in the traces as `poll(..., 3, 1)` and `poll(..., 3, 0)` (1 ms / 0 ms timeouts) interspersed with the blocking `poll(..., 3, -1)` at [kitty/child-monitor.c:L1512].
4. **Producer/consumer decoupling.** Reading (the `io_loop` thread) and parsing/rendering (the main thread `process_global_state()` [kitty/child-monitor.c:L1224] → `parse_input()` [kitty/child-monitor.c:L1236]) run on **different threads**, decoupled by the 1 MiB double buffer. The reader can keep draining the kernel buffer while the parser works through what has already been read.

*(Caveat, stated honestly: `strace` instruments every syscall and slows the traced process, so the absolute reads/second above are measured **under tracing** and are a conservative figure; the two runs nonetheless agree closely. The bytes-per-read figure is governed by the kernel buffer and is the robust, load-characterising quantity.)*

---

## Q5 — The concrete file-descriptor number of the PTY master

**Direct answer.** kitty reads the PTY master from **file descriptor `8`**, which backs `/dev/pts/ptmx`. (The number is environment-specific — it is whatever `os.openpty()` returned in this process — but its *origin* is fixed by the code path below. The slave the shell holds is `/dev/pts/0`, from Q2.)

### Evidence

From `/proc`:

```bash
ls -l /proc/47973/fd/ | grep -iE 'ptmx|pts'
```

```
lrwx------ 1 root root 64 Jul 14 19:19 8 -> /dev/pts/ptmx
```

Corroborated by the `-yy` fd annotation on every master `read()`/`poll()` in the traces, which resolves fd 8 to the master and even shows the connected slave:

```
read(8</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, ...)
                     └─ char device 5:2 = /dev/ptmx, peered with /dev/pts/0
```

### Rationale — cause → effect

The fd is the `os.openpty()` **master**, stored on the `Child` object and marked non-blocking:

```
kitty/child.py:L171   master, slave = os.openpty()
kitty/child.py:L338   self.child_fd = master
kitty/child.py:L345   os.set_blocking(self.child_fd, False)      # non-blocking master
```

Non-blocking mode [kitty/child.py:L345] is precisely why the I/O loop **`poll()`s before each `read()`** (Q3/Q4) instead of blocking in `read()`. The fd is then handed to the C child monitor:

```
kitty/boss.py:L585        def add_child(self, window):
kitty/boss.py:L587            self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)
kitty/child-monitor.c:L305    add_child(ChildMonitor *self, PyObject *args) {   # PyArg_ParseTuple(args, "kiiO", ...)
```

`add_child()` records the fd into the I/O thread's `children_fds` array, where it becomes the `fd=8` seen in every traced `poll()`/`read()`.

---

## Q6 — The reader function and the parser function

**Direct answer.**

* **Reader (reads from the PTY fd):** **`read_bytes()`** — `kitty/child-monitor.c:L1337`. It performs the actual `read()` on the master fd at `kitty/child-monitor.c:L1345`.
* **Parser (separates printable text from escape sequences):** **`consume_input()`** — `kitty/vt-parser.c:L1367` (the parse-loop dispatcher). Its normal-mode branch **`consume_normal()`** — `kitty/vt-parser.c:L230` — is what actually splits text from escapes.

### The reader — observation-corroborated

`read_bytes()` [kitty/child-monitor.c:L1337] contains the read syscall and its error handling:

```
kitty/child-monitor.c:L1341   uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
kitty/child-monitor.c:L1345   len = read(fd, buf, available_buffer_space);        # ← the PTY read
kitty/child-monitor.c:L1347   if (errno == EINTR || errno == EAGAIN) continue;    # retry
kitty/child-monitor.c:L1348   if (errno != EIO) perror("Call to read() from child fd failed");
kitty/child-monitor.c:L1354   vt_parser_commit_write(screen->vt_parser, len);
kitty/child-monitor.c:L1355   return len != 0;                                    # zero-length read ⇒ child exit
```

This is not merely inferred: **100 % of the master reads captured in Q3/Q4 were issued by thread TID 48040**, whose name is `KittyChildMon` — the `io_loop` I/O thread [kitty/child-monitor.c:L1481, L1489] that calls `read_bytes()` [invoked at kitty/child-monitor.c:L1531]:

```bash
cat /proc/47973/task/48040/comm
grep -E 'read\(8<' yes1.strace | grep -oE '^[0-9]+' | sort | uniq -c
```

```
KittyChildMon
  34549 48040
```

The `strace` attach banner likewise confirms the multi-threaded target that made `-f` mandatory:

```
strace: Process 47973 attached with 67 threads
```

### The parser — text-vs-escape split *(function identification inferred from code: a syscall tracer sees `read`/`poll`, not internal C calls)*

`consume_input()` [kitty/vt-parser.c:L1367] is a state-machine dispatcher (a `switch` on `self->vte_state`). In the `VTE_NORMAL` state it calls `consume_normal()`:

```
kitty/vt-parser.c:L1376   case VTE_NORMAL:
kitty/vt-parser.c:L1377       consume_normal(self); self->read.consumed = self->read.pos; break;
```

`consume_normal()` [kitty/vt-parser.c:L230] is where printable text is separated from escape/control sequences. It scans bytes only up to the next ESC sentinel, emits the decoded printable run to the screen, and hands control back to the escape machinery when an ESC is hit:

```
kitty/vt-parser.c:L232   const bool sentinel_found = utf8_decode_to_esc(&self->utf8_decoder, self->buf + self->read.pos, self->read.sz - self->read.pos);
kitty/vt-parser.c:L236   screen_draw_text(self->screen, self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);   # printable text sink
kitty/vt-parser.c:L238   if (sentinel_found) { SET_STATE(ESC); break; }                                                # hand off escape/control
```

So `utf8_decode_to_esc()` [kitty/vt-parser.c:L232] consumes a maximal run of printable UTF-8 bytes up to (but not including) the next `ESC`; that run is drawn via `screen_draw_text()` [kitty/screen.c:L866]; and when the `ESC` sentinel is found, `SET_STATE(ESC)` [kitty/vt-parser.c:L238] transitions the parser toward the escape/CSI/OSC/DCS dispatch handlers. That is exactly the "split printable text from escape sequences" behavior the question asks about.

The reader and parser run on **different threads**, coupled through the 1 MiB double buffer: the reader (`read_bytes` on `io_loop`) fills the write side via `vt_parser_create_write_buffer()`/`vt_parser_commit_write()` [kitty/vt-parser.c:L1451, L1465; declared in kitty/vt-parser.h:L34-L35], and the consumer main thread drives `consume_input()` via `process_global_state()` → `parse_input()` [kitty/child-monitor.c:L1224, L1236].

---

## Appendix — commands, environment, and integrity

### A. Environment adjustments (not repository changes)

* Installed observation tooling: `strace`, `xdotool`, `xxd` (analogous to `ps`/`/proc`; none is a project dependency).
* Relaxed ptrace so `strace` could attach in the container: `echo 0 > /proc/sys/kernel/yama/ptrace_scope` (already `0` here).
* Started a virtual display for the GUI: `Xvfb :99` with `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`.

### B. Key commands (in order)

```bash
# Q1 build & launch
CI=true python3 setup.py --ignore-compiler-warnings
Xvfb :99 -screen 0 1280x1024x24 +extension GLX +render -noreset &
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
./kitty/launcher/kitty
KPID=$(pgrep -f 'kitty/launcher/kitty' | head -1)

# Q2 shell identity
ps --ppid "$KPID" -o pid,ppid,cmd
SHPID=$(pgrep -P "$KPID" | head -1)
cat /proc/$SHPID/cmdline | tr '\0' ' '; xxd /proc/$SHPID/cmdline
ls -l /proc/$SHPID/fd/0 /proc/$SHPID/fd/1 /proc/$SHPID/fd/2

# Q3/Q4/Q5 tracing (reads happen on the KittyChildMon thread → -f is required)
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o echo.strace   # then type: echo test123
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o yes1.strace   # then: yes hello (~5 s), Ctrl-C
strace -f -yy -tt -T -e trace=read,poll -p "$KPID" -o yes2.strace   # repeat for stability
strace -f -y  -c            -e trace=read,poll -p "$KPID" -o yes3.count   # magnitude histogram
ls -l /proc/$KPID/fd/ | grep -iE 'ptmx|pts'                          # Q5: fd 8 -> /dev/pts/ptmx

# Q6 corroboration
cat /proc/$KPID/task/48040/comm                                      # -> KittyChildMon
```

### C. Answer summary

| # | Question | Answer |
|---|----------|--------|
| Q1 | Build & launch | `python3 setup.py --ignore-compiler-warnings` → `kitty/launcher/kitty`; launched headless (`Xvfb :99`), kitty PID **47973**, kitty **0.35.2** on CPython **3.13.7** |
| Q2 | Spawned shell | **`/bin/bash --posix`**, PID **48041**, PTY **`/dev/pts/0`** (slave) |
| Q3 | `echo test123` reads | `poll()`→`read(8, buf, **1048576**)`; returns small counts (1 B per keystroke echo; **11/47/114/429** B on execution) |
| Q4 | `yes hello` reads | **many small reads**: **~6,000/s** (6,576 & 5,820 across two runs), typical **~1.1–1.3 KiB/read** (median 1,118 & 1,302), ≪ 1 MiB requested |
| Q5 | Master fd | **fd 8** → `/dev/pts/ptmx` |
| Q6 | Reader / parser | reader **`read_bytes()`** [child-monitor.c:L1337]; parser **`consume_input()`** [vt-parser.c:L1367] → **`consume_normal()`** [vt-parser.c:L230] |

### D. Integrity statement

No file in the kitty source tree was created, modified, or deleted during this investigation. All temporary artifacts (traces, logs, helper output) were written **outside** the repository (under `/tmp`) and deleted afterward. The only file this task adds to the repository is this document, `blitzy/documentation/kitty_815df1e210e0.md`.

