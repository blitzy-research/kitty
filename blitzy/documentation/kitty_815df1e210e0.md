# How kitty's C code communicates with the shell process over a PTY

This document answers six questions about how the [kitty](https://sw.kovidgoyal.net/kitty/) terminal emulator's C code talks to the shell it spawns over a pseudoterminal (PTY). Every runtime value below was **observed by actually building, launching, and tracing kitty** — not inferred from reading alone — and every structural claim carries a `file:line` citation into the source tree. Where a value depends on the observation method (e.g. `strace` overhead), that is stated explicitly.

## Investigation environment & methodology

- **Repository:** branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config"), checked out at `/tmp/blitzy/kitty/kitty_815df1e210e0_77c5cb`. The source tree was treated as **read-only**; the only artifact produced is this document.
- **Build:** `python3 setup.py build` (compiles the C extension modules and the native launcher `kitty/launcher/main.c`).
- **Headless launch:** kitty is a GPU/GUI terminal, so it was run under a virtual framebuffer (`Xvfb :99`) with software OpenGL (`LIBGL_ALWAYS_SOFTWARE=1`, Mesa llvmpipe). Remote control (`--listen-on unix:/tmp/kitty.sock -o allow_remote_control=yes`) was used to drive keystrokes reliably.
- **Observation tools:** `strace -f -e trace=read,poll -yy -ttt` (follow threads, filter to `read`/`poll`, resolve fd→device paths, absolute timestamps); `ps`; and `/proc/<pid>/{cmdline,fd,task}`.
- All PIDs/fds below come from a single kitty session: **kitty PID `36659`**, its I/O thread **TID `36729` (`KittyChildMon`)**, and the spawned **shell PID `36730`**.

> **Caveat that matters for Q3/Q4:** `strace` uses `ptrace`, which adds overhead to every intercepted syscall. This *slows* kitty's read loop, which changes the observed magnitude of reads (it lowers read frequency and inflates bytes-per-read, because more data accumulates in the PTY between reads). For Q4 the traced numbers are therefore corroborated with an independent, un-traced PTY probe.

---

## Q1 — Build kitty from source and launch it

**Build.** kitty was built from source via its Python build entry point:

```
$ python3 setup.py build
```

The resulting native launcher reports:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**Launch (headless).** Because kitty renders with the GPU, it was launched under Xvfb with software GL:

```
$ DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty \
      --listen-on unix:/tmp/kitty.sock -o allow_remote_control=yes
```

Software rendering was confirmed with:

```
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)
```

kitty started successfully; the only startup log line was a harmless bus warning:

```
[0.246] Failed to open systemd user bus with error: Connection refused
```

**Running kitty PID (observed):** `36659`.

```
$ pgrep -af launcher/kitty
36659 ./kitty/launcher/kitty --listen-on unix:/tmp/kitty.sock -o allow_remote_control=yes
```

*Rationale:* `setup.py` is the documented build entry point; it compiles the C extensions and the native launcher `kitty/launcher/main.c`, producing the `kitty/launcher/kitty` binary that was then launched.

---

## Q2 — What shell is spawned, its PID, exact command line, and the PTY device path

**(a) Process spawned:** `bash`. **(b) PID:** `36730`. **(c) Exact command line:** `/bin/bash --posix`.

Observed in the process list as a child of kitty (PID `36659`):

```
$ ps --ppid 36659 -o pid,ppid,comm,args
  PID  PPID COMMAND         COMMAND
36730 36659 bash            /bin/bash --posix
```

The exact command line, read straight from the kernel via `/proc/36730/cmdline` (arguments are NUL-separated), is `/bin/bash --posix`:

```
$ tr '\0' ' ' < /proc/36730/cmdline ; echo
/bin/bash --posix
$ cat -v /proc/36730/cmdline ; echo          # ^@ marks the real NUL separators
/bin/bash^@--posix^@
```

Cross-checked with `ps`:

```
$ ps -p 36730 -o pid,tty,args
  PID TT       COMMAND
36730 pts/0    /bin/bash --posix
```

**(d) PTY device path connecting kitty and the shell:** the shell's stdin/stdout/stderr are the PTY **slave** `/dev/pts/0`:

```
$ for fd in 0 1 2; do printf 'fd %s -> ' $fd; readlink /proc/36730/fd/$fd; done
fd 0 -> /dev/pts/0
fd 1 -> /dev/pts/0
fd 2 -> /dev/pts/0
```

*Rationale (grounding the observation in source):*
- The shell command is resolved in Python by `resolved_shell()` — with the default `shell .` option it returns `[shell_path]` (`ans = [shell_path]`) [kitty/utils.py:768].
- `shell_path` is the user's login shell: `shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` [kitty/constants.py:181] (falling back to `/bin/sh` [kitty/constants.py:185]). For the `root` user here the login shell is `/bin/bash`, which is why the executable is `/bin/bash`.
- The `--posix` argument is **not** typed by the user — it is injected by kitty's shell integration. `Child.final_env`/spawn calls `modify_shell_environ(opts, env, self.argv)` [kitty/child.py:267], whose bash path `setup_bash_env()` [kitty/shell_integration.py:70] both sets `env['ENV']` to kitty's bash integration script [kitty/shell_integration.py:134] and inserts the flag with `argv.insert(1, '--posix')` [kitty/shell_integration.py:146]. That is exactly why the process list shows `/bin/bash --posix`.
- The fork/exec of that command happens in the native `spawn()` [kitty/child.c:81]: it obtains the slave device path via `ttyname_r(slave, name, sizeof(name) - 1)` [kitty/child.c:88], `fork()`s [kitty/child.c:97], makes the slave the controlling terminal with `ioctl(tfd, TIOCSCTTY, 0)` [kitty/child.c:129], and finally `execvp(exe, argv)` [kitty/child.c:159]. The slave path returned there is the `/dev/pts/0` observed above.

---

## Q3 — Reading a small input: `echo test123`

With `strace` attached to kitty, `echo test123` was typed into the running terminal. The reads happen on the I/O thread (**TID 36729**), which first **waits in `poll()` for `POLLIN`** on the PTY master and then calls **`read()`**.

**(a) System call(s) used:** `poll()` (to wait until the PTY master is readable) followed by `read()`. A resolved poll returning `revents=POLLIN` on fd 10, immediately followed by the `read()`, looks like this:

```
36729 1782937399.460351 poll([{fd=7<anon_inode:[eventfd]>, events=POLLIN}, {fd=8<signalfd:[HUP INT USR1 USR2 TERM CHLD]>, events=POLLIN}, {fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 1 ([{fd=10, revents=POLLIN}])
36729 1782937399.460494 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\1\33]133;k;start_kitty\7\2...\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048506) = 114
```

Once the shell's output is fully drained, the next `poll()` **times out** (`= 0`) and the I/O thread goes idle (it then blocks in `poll(..., -1)` [kitty/child-monitor.c:1512] until more data arrives):

```
36729 1782937399.461032 poll([...{fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 1) = 0 (Timeout)
```

**(b) Buffer size used:** the first `read()` requests **`1048576` bytes = 1 MiB**, which is exactly kitty's parser buffer `BUF_SZ`:

```
36729 1782937399.459006 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "echo test123\r\n\33[?2004l\r", 1048576) = 23
```

**(c) Bytes returned:** the shell's output for this one line arrived as **four reads returning `23`, `47`, `114`, and `169` bytes** (total `353`):

```
36729 ... read(10<...>, "echo test123\r\n\33[?2004l\r", 1048576) = 23
36729 ... read(10<...>, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048553) = 47
36729 ... read(10<...>, "\1\33]133;k;start_kitty\7\2...\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048506) = 114
36729 ... read(10<...>, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7...", 1048392) = 169
```

The literal command output `test123\r\n` is the last 9 bytes of the **114-byte** read; everything else is the terminal echo of the typed line plus bash **shell-integration** OSC sequences (bracketed-paste toggles `\33[?2004l`/`\33[?2004h`, OSC 2 title, and OSC 133 prompt/command markers), which are present because `/bin/bash --posix` sources kitty's integration (see Q2).

*Rationale (grounding in source):*
- The reader is `read_bytes()`, which asks the parser for a write buffer and then does `len = read(fd, buf, available_buffer_space)` [kitty/child-monitor.c:1345].
- `available_buffer_space` is computed by `vt_parser_create_write_buffer()` as `*sz = BUF_SZ - self->write.offset` [kitty/vt-parser.c:1457], where `#define BUF_SZ (1024u*1024u)` = `1048576` [kitty/vt-parser.c:18]. On the first read the buffer is empty (`write.offset == 0`), so the request is the full `1048576`.
- The subsequent request sizes decrease exactly by what was already buffered: `1048553 = 1048576 - 23`, `1048506 = 1048576 - (23+47)`, `1048392 = 1048576 - (23+47+114)`. This is `write.offset = read.sz + write.pending` [kitty/vt-parser.c:1456] in action.
- The returned byte count is simply whatever the kernel had available on the PTY at that instant; for this tiny input the reads are small and one-shot.

---

## Q4 — Reading a high-volume stream: `yes hello`

`yes hello` was then run to produce a continuous stream, traced for a representative ~4-second window before being stopped with Ctrl-C.

**(a) How the reading behavior changes.** For the tiny `echo` (Q3), the I/O thread did four reads and then its next `poll()` **timed out** (`= 0 (Timeout)`) and it went idle. Under `yes hello` the thread instead spins in a **tight, continuous `read -> poll -> read` loop** in which every `poll()` returns immediately with `revents=POLLIN` (data is always available — it never times out):

```
36729 1782937584.361158 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 986694) = 1827
36729 1782937584.361227 poll([...{fd=10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, events=POLLIN}], 3, 2) = 1 ([{fd=10, revents=POLLIN}])
```

(The poll timeout is a small `input_delay`-bounded value — `2` ms here — because the main thread has a pending parse wakeup: `monotonic_t time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at)` [kitty/child-monitor.c:1508], versus the fully-idle `poll(..., -1)` [kitty/child-monitor.c:1512] seen after `echo`.)

A second visible change: the **request size now shrinks below 1 MiB** as unparsed data accumulates between the main thread's parse cycles. Three consecutive reads request `1044303`, then `1043314`, then `1042175` bytes — and each decrease equals the previous read's return (`1044303 - 989 = 1043314`; `1043314 - 1139 = 1042175`):

```
36729 1782937584.338839 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1044303) = 989
36729 1782937584.338946 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "hello\r\nhello\r\nhello\r\nhello\r\nhell"..., 1043314) = 1139
36729 1782937584.339046 read(10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhe"..., 1042175) = 1101
```

This is `*sz = BUF_SZ - self->write.offset` [kitty/vt-parser.c:1457] observed live: the buffer is filling with data awaiting the parser.

Reads are **gated by backpressure**: the I/O thread only asks for `POLLIN` on a child when the parser reports free space — `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0` [kitty/child-monitor.c:1501], where the predicate is `self->read.sz + self->write.pending < BUF_SZ` [kitty/vt-parser.c:1481]. In this run the gate **never closed**: across the whole trace fd 10 was requested with `events=POLLIN` `31,185` times and with `events=0` **`0`** times. In other words, the 1 MiB buffer never filled — the (SIMD-accelerated) main-thread parser kept draining `hello\r\n` faster than it arrived, so kitty never had to throttle reads for this simple text. (Throttling to `events=0` would only occur if the parser fell ~1 MiB behind.) Parsing itself is batched on a timer: it runs once `input_delay` (default **3 ms**, `opt('input_delay', '3', ...)` [kitty/options/definition.py:878]) has elapsed, or when the buffer comes within 16 KiB of full [kitty/vt-parser.c:1425]; screen repaints are separately throttled by `repaint_delay` (default **10 ms** [kitty/options/definition.py:866]).

**(b) Frequency of reads.** During the streaming window kitty issued **`31,180` reads on fd 10 over a `4.034 s` span -> ~ `7,729` reads/second (~ `129.4 us` between reads)** — *as measured under strace*:

```
fd10 reads counted: 31180
first ts: 1782937584.336019  last ts: 1782937588.370408
streaming span: 4.034 s
read frequency: 7,729 reads/sec  (under strace; strace adds overhead)
avg interval  : 129.4 microseconds between reads
```

Because strace slows the read loop, an independent un-traced PTY probe (a small helper that mimics kitty's path: `openpty()`, fork/exec `yes hello`, then `read(master, 1 MiB)`) reads **far more often** — about **`98,560` reads/second**:

```
no-strace PTY probe (buffer request size = 1048576 = BUF_SZ):
  duration ~2.0s, reads=197119, total_bytes=22915695
  read freq   = 98,560 reads/sec
```

**(c) Typical byte count per read.** Under strace the reads are ~1 KB each — **median `1059` bytes, mean `1217` bytes** (min `2`, max `20827`; `0` reads reached even 64 KiB):

```
num reads on fd10: 29504
total bytes read : 35908408
min / median / mean / max: 2 1059.0 1217.1 20827
reads >= 65536 : 0
```

Un-traced (probe), where the reader keeps up, the reads are much smaller — **median `84` bytes, mean `116` bytes** — and every common size is a whole multiple of 7 (because `hello\r\n` is 7 bytes on the wire after the PTY's `\n`->`\r\n` translation):

```
min/median/mean/max bytes = 2/84/116.3/19754
top read sizes (bytes:count): [(42, 5964), (35, 5890), (49, 5797), (63, 5684), (70, 5064), (56, 5057)]
```

*Interpretation:* the two methods bracket the real behavior. In **both** cases each read returns only a few bytes to a few kilobytes — orders of magnitude below the 1 MiB request — and reads are issued at very high frequency in a continuous loop. The absolute numbers differ because strace's per-syscall overhead slows the reader (fewer, larger reads); without it kitty reads more often in smaller chunks. What does **not** change is the mechanism: a tight poll/read loop, a request size of `BUF_SZ - write.offset`, and reads bounded by the parser's free space.

---

## Q5 — The file-descriptor number kitty uses to read the PTY master

**Observed fd number: `10`.** Listing kitty's open descriptors shows exactly one PTY master (`/dev/pts/ptmx`):

```
$ ls -l /proc/36659/fd | grep -iE 'pts|ptmx'
lrwx------ 1 root root 64 ... 10 -> /dev/pts/ptmx
```

`strace`'s `-yy` annotation independently confirms fd `10` is the master whose peer is the slave `/dev/pts/0` — note `10</dev/pts/ptmx<char 5:2 @/dev/pts/0>>` in every read line above.

*Rationale (grounding in source):* the master fd is stored on the Python `Child` object as `self.child_fd = master` [kitty/child.py:338], where `master, slave = os.openpty()` created the pair in blocking mode [kitty/child.py:170-171] and it was then set non-blocking with `os.set_blocking(self.child_fd, False)` [kitty/child.py:345] (which is why the C reader tolerates `EAGAIN`/`EINTR` [kitty/child-monitor.c:1346-1347]). That fd is handed to the native monitor by `add_child`: `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` [kitty/boss.py:587]. The concrete integer (`10`) is assigned by the OS at `openpty()` time and must be observed at runtime — it is not a constant in the source.

---

## Q6 — The C functions that read and that parse the PTY data

**(a) The function that reads from the PTY file descriptor:** `read_bytes()` in `kitty/child-monitor.c`, which runs on the I/O thread's `io_loop()`.

```c
// kitty/child-monitor.c:1337
read_bytes(int fd, Screen *screen) {
    ...
    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    ...
        len = read(fd, buf, available_buffer_space);   // kitty/child-monitor.c:1345
```

It is called from the poll loop `io_loop()` [kitty/child-monitor.c:1480-1481] via `read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)` [kitty/child-monitor.c:1531].

**(b) The function that parses incoming data to separate printable text from escape sequences:** `consume_input()` in `kitty/vt-parser.c` [kitty/vt-parser.c:1367]. It is a state machine driven by `self->vte_state` (type `VTEState` [kitty/vt-parser.c:160], whose values are `VTE_NORMAL, VTE_ESC, VTE_CSI, VTE_OSC, VTE_DCS, VTE_APC, VTE_PM, VTE_SOS`). In the normal state it routes **printable text** to the screen, and on hitting an escape sentinel it switches into the escape-processing states:

```c
// kitty/vt-parser.c:1367
consume_input(PS *self, ...) {
    ...
    switch (self->vte_state) {
        case VTE_NORMAL:
            consume_normal(self); ...      // printable text path
        case VTE_ESC: ...                  // escape-sequence path
```

`consume_normal()` [kitty/vt-parser.c:230] decodes UTF-8 text and draws it via `screen_draw_text(...)` [kitty/vt-parser.c:236], but stops the moment an ESC is found and transitions the state machine with `SET_STATE(ESC)` [kitty/vt-parser.c:238] — that is the exact point where printable text is separated from escape sequences:

```c
// kitty/vt-parser.c:229-240
consume_normal(PS *self) {
    do {
        const bool sentinel_found = utf8_decode_to_esc(&self->utf8_decoder, self->buf + self->read.pos, self->read.sz - self->read.pos);
        self->read.pos += self->utf8_decoder.num_consumed;
        if (self->utf8_decoder.output.pos) {
            ...
            screen_draw_text(self->screen, self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
        }
        if (sentinel_found) { SET_STATE(ESC); break; }
    } while (self->read.pos < self->read.sz);
}
```

The screen owns the parser instance the reader fills and this function drains: `Parser *vt_parser;` [kitty/screen.h:158].

*Threading note:* `read_bytes()` (reading) runs on the **I/O thread** (`KittyChildMon`, TID `36729` here), while `consume_input()` (parsing) runs on the **main thread** — which is why the 1 MiB `BUF_SZ` buffer and the `POLLIN` backpressure gate exist to decouple the two.

---

## Coverage summary

| Question | Named item | Answer (observed / cited) |
|---|---|---|
| Q1 | build | `python3 setup.py build` → `kitty 0.35.2 created by Kovid Goyal` |
| Q1 | launch + PID | headless under Xvfb + llvmpipe; kitty **PID 36659** |
| Q2 | process | `bash` |
| Q2 | PID | `36730` |
| Q2 | exact command line | `/bin/bash --posix` (`/proc/36730/cmdline` = `/bin/bash^@--posix^@`) |
| Q2 | PTY device path | `/dev/pts/0` (slave, from `/proc/36730/fd/{0,1,2}`) |
| Q3 | system call(s) | `poll()` for `POLLIN`, then `read()` |
| Q3 | buffer size | `1048576` bytes = 1 MiB = `BUF_SZ` [vt-parser.c:18] |
| Q3 | bytes returned | `23`, `47`, `114`, `169` (total `353`); `test123\r\n` inside the 114-byte read |
| Q4 | behavior change | blocking single reads → continuous `poll/read` loop; request size shrinks by `write.offset`; `POLLIN` gate stayed open (buffer never full) |
| Q4 | frequency | ≈ `7,729` reads/s under strace (129 µs apart); ≈ `98,560` reads/s un-traced |
| Q4 | typical bytes/read | median `1059` B / mean `1217` B under strace; median `84` B un-traced; always ≪ 1 MiB request |
| Q5 | master fd number | **`10`** (`/proc/36659/fd/10 -> /dev/pts/ptmx`) |
| Q6 | reading function | `read_bytes()` [child-monitor.c:1337] → `read()` [:1345] on `io_loop()` |
| Q6 | parsing function | `consume_input()` [vt-parser.c:1367]; text via `consume_normal()`/`screen_draw_text` [:230,:236]; escapes via `VTEState` machine [:160] |

## Notes on fidelity

- Every runtime number here was observed from a live kitty session (PID 36659 / shell 36730 / fd 10 / `/dev/pts/0`); nothing was assumed.
- The one honest negative result: the `POLLIN`-backpressure throttle (`events=0`) was **not** observed for `yes hello`, because the parser kept up and the 1 MiB buffer never filled. The mechanism is real (cited above) but did not engage for this simple, high-rate-but-cheap-to-parse text.
- `strace` overhead measurably changes read magnitude, so Q4's traced frequency/size are reported alongside an un-traced PTY probe rather than as absolute truth.
- The user's example commands `echo test123` and `yes hello` were used verbatim. All temporary scripts and trace logs were kept outside the repository and deleted afterward, leaving the kitty source tree byte-for-byte unchanged.
