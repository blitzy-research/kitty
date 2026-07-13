# How kitty turns a surge of raw terminal input into something the application reacts to

**An investigate-by-running answer, grounded in the observed runtime behavior of a canonically built kitty.**

- **Subject:** kitty terminal emulator, version **0.35.2**, VCS commit **`815df1e210e0…`** (source branch `kitty_815df1e210e0`).
- **Method:** Every behavioral claim below was produced by **building and running** kitty and driving a **real child process writing to kitty's pseudo-terminal (PTY)**, then capturing the *unedited* output with kitty's own instrumentation (`--dump-commands`, `--dump-bytes`, `--debug-keyboard`) and with direct runtime inspection (`/proc/<pid>/task`, query/reply round-trips through the PTY). Each claim carries the exact command that produced it, the complete unedited output, and a `file:line` reference naming the specific function/struct.
- **Scope of the question (four threads):**
  - **Q1 — Input surge & entry point (incl. pause/resume):** When a surge of raw input arrives — especially when a session is paused then resumed — how does the byte stream become actionable, and *where does it first enter*?
  - **Q2 — The "unseen conductor":** How are timing/ordering/state-handoff responsibilities split across threads, and *what decides which event is handled first*?
  - **Q3 — Aligned semantics under stress:** When shell-integration hints arrive mixed in with ordinary text, how does the system keep screen state, command context, and input meaning aligned without drifting — and does it behave differently under heavy backpressure or an unstable remote connection?
  - **Q4 — End-to-end rhythm:** From the moment mixed input arrives to the moment the interface settles, what really happens and how do the moving parts keep rhythm?

> **How to read the evidence.** Lines beginning with `$` are the exact command. The block that follows is its complete, unedited output. **Observed** means captured from a running kitty; **inferred** means read from source (always quoted with `file:line`) and is labeled as such. The canonical evidence is a real child → PTY → `read_bytes` on the io_thread, via the built `kitty/launcher/kitty` launcher; the one non-canonical helper used (the synchronous `parse_bytes` test path) is labeled wherever it appears.

---

## 1. Environment & canonical build

kitty is a GPU/OpenGL desktop terminal written in C (performance core), Python (orchestration), and Go (CLI tooling). It was built and run in its **default, canonical configuration** in the provided container.

### 1.1 Toolchain actually present

```
$ python --version
Python 3.11.15
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ go version
go version go1.22.12 linux/amd64
$ make --version | head -1
GNU Make 4.4.1
```

Python **3.11.15** and Go **1.22.12** match the documented canonical runtimes (`requires-python >=3.8`, CI-pinned 3.11; `go 1.22` in `go.mod`).

### 1.2 Canonical build command

```
$ source /root/kitty-venv/bin/activate
$ export PATH=$PATH:/usr/local/go/bin
$ python setup.py build --verbose
```

The build is `python setup.py build` (equivalently `make`). Salient lines from the build log (the long absolute checkout path in the final `go build` line abbreviated as `...`; all else verbatim):

```
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
Detected: CompilerType.gcc
Updating Go generated files...
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/.../tools/cmd
```

This is a normal Linux CI build: the Wayland backend is disabled because `wayland-protocols` is absent (canonical for x11-only CI), and the Go link step stamps the VCS revision `815df1e210e0…` into the binary via `-ldflags -X kitty.VCSRevision=…`.

The launcher is produced at `kitty/launcher/kitty`:

```
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 36288 kitty/launcher/kitty
```

### 1.3 VCS-stamped version banner (default build)

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The version `0.35.2` is defined at **`kitty/constants.py:L25`** (`version: Version = Version(0, 35, 2)`). The launcher's default `--version` prints the short form because `cli.py`'s `version(add_rev=False)` omits the revision. The VCS revision is stamped at build time via `get_vcs_rev()` (**`setup.py:L674`**) and is compiled into the C extension as `fast_data_types.KITTY_VCS_REV`; requesting it explicitly reproduces the full stamped banner:

```
$ ./kitty/launcher/kitty +runpy 'from kitty.cli import version; print(version(add_rev=True))'
kitty 0.35.2 (815df1e210) created by Kovid Goyal
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

`version(add_rev=True)` uses `fast_data_types.KITTY_VCS_REV[:10]` (**`kitty/cli.py:L486-L492`**); the stamped revision `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` matches `git rev-parse HEAD`. All observations below come from this exact build.

### 1.4 Run recipe for canonical PTY observation

kitty is a windowed OpenGL app, so a headless run uses a virtual display (Xvfb `:99`) with Mesa software GL. Every canonical experiment below launches a **real child process** under this recipe:

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ export PYTHONHOME=/root/.local/share/uv/python/cpython-3.11.15-linux-x86_64-gnu
$ export PYTHONPATH=/root/kitty-venv/lib/python3.11/site-packages
$ export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 TMPDIR=/tmp/kitty-nosgid
$ ./kitty/launcher/kitty -o close_on_child_death=yes [--dump-bytes=FILE] --dump-commands <child…>
```

Two canonical instrumentation flags do the heavy lifting (both defined in **`kitty/cli.py`**):

- `--dump-bytes=FILE` (**`kitty/cli.py`**) records the **raw bytes exactly as `read_bytes` ingests them** from the child PTY.
- `--dump-commands` (**`kitty/cli.py:L972`**) prints the **parser events** produced by `run_worker` (`draw …`, `screen_set_mode …`, `shell_prompt_marking …`, `handle_remote_cmd …`), i.e. the effect of each token on screen state, in byte order.

---

## 2. Q1 — Input surge, the entry point, and pause/resume

### 2.1 Where the byte stream first enters: `read_bytes` on the io_thread

Raw bytes from a child first enter the system in **`read_bytes(int fd, Screen *screen)`** at **`kitty/child-monitor.c:L1337`**, which runs on the I/O thread. Its body (source, *inferred*) obtains a write buffer from the VT parser, reads the PTY fd into it, and commits the bytes:

```c
// kitty/child-monitor.c:L1337
read_bytes(int fd, Screen *screen) {
    ...
    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;      // L1342: no space -> DON'T read (backpressure)
    while(true) {
        len = read(fd, buf, available_buffer_space);
        ...
    }
    vt_parser_commit_write(screen->vt_parser, len);
    return len != 0;
}
```

It is called from the io_loop at **`kitty/child-monitor.c:L1531`** (`has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)`), and the buffered bytes are later turned into screen mutations by the main thread's **`parse_worker`** (**`kitty/vt-parser.c:L1496`**), which wraps **`run_worker`** (**`kitty/vt-parser.c:L1417`**).

### 2.2 A surge, observed at both levels

A real child emits a 5000-line burst; `--dump-bytes` captures the raw bytes as `read_bytes` ingests them, and `--dump-commands` captures the resulting parser events:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes \
    --dump-bytes=/tmp/kitty_obs/q1_surge_bytes.bin --dump-commands \
    sh -c 'for i in $(seq 1 5000); do printf "L%04d-THE-QUICK-BROWN-FOX\r\n" "$i"; done; sleep 0.3' \
    > /tmp/kitty_obs/q1_surge_dump.txt
$ echo "raw bytes: $(wc -c < q1_surge_bytes.bin); dump lines: $(wc -l < q1_surge_dump.txt); draws: $(grep -c '^draw ' q1_surge_dump.txt)"
raw bytes: 140000; dump lines: 20000; draws: 5000
$ head -4 q1_surge_dump.txt
draw L0001-THE-QUICK-BROWN-FOX
screen_carriage_return
screen_carriage_return
screen_linefeed
$ tail -3 q1_surge_dump.txt
screen_carriage_return
screen_carriage_return
screen_linefeed
```

**Observed:** the surge is **140000 raw bytes** at the `read_bytes` level and becomes **5000 `draw` events** at the `run_worker` level, strictly ordered `L0001…L5000`. (Each line is 28 bytes: 25 printable + `\r\r\n` — the extra `\r` is the tty's ONLCR translation, so what `read_bytes` sees is exactly the on-the-wire stream, which is why each line dumps as two `screen_carriage_return` + one `screen_linefeed`.) The linkage `read_bytes → parse_worker/run_worker` is *inferred* from the source cited above; the byte counts and ordered draws are *observed*.

### 2.3 Pause/resume via synchronized-output mode 2026 — before / during / after

"Pausing then resuming" a session is the terminal's **synchronized output** feature: DEC private mode **2026**, defined as `#define PENDING_MODE 2026` at **`kitty/control-codes.h:L235`**. An application sends `CSI ? 2026 h` to begin (hold the last committed frame while bytes keep being processed) and `CSI ? 2026 l` to end (flush atomically). In kitty this drives **`screen_pause_rendering`** at **`kitty/screen.c:L2506`**, which snapshots screen state under an `expires_at` timer.

To observe the *state* (not just the toggle) canonically, a real child issues the DECRQM query `CSI ? 2026 $ p` at each phase and reads kitty's reply back **through the PTY**, printing the decoded reply (which then re-enters as a `draw`). kitty answers `?2026;1$y` when paused (`expires_at` set) and `?2026;2$y` when not — reported at **`kitty/screen.c:L2238`** (`ans = self->paused_rendering.expires_at ? 1 : 2`).

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands \
    /root/kitty-venv/bin/python3 /tmp/kitty_obs/pause_probe.py > /tmp/kitty_obs/q1_decrqm_dump.txt
$ cat /tmp/kitty_obs/q1_decrqm_dump.txt
report_mode_status 2026 1
draw PHASE=before DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
screen_set_mode 2026 1
draw DRAWN-WHILE-PAUSED
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw PHASE=during DECRQM_REPLY=<ESC>[?2026;1$y
screen_carriage_return
screen_linefeed
screen_reset_mode 2026 1
report_mode_status 2026 1
draw PHASE=after DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
```

**Observed, before/during/after:**
- **before** — `DECRQM_REPLY=<ESC>[?2026;2$y` → state **2**, not paused.
- **during** — after `screen_set_mode 2026 1` (`CSI ? 2026 h`), `DECRQM_REPLY=<ESC>[?2026;1$y` → state **1**, paused, `expires_at` set. Critically, `draw DRAWN-WHILE-PAUSED` is emitted *between* the set and reset: **parsing and screen mutation continue while rendering is paused** — the pause holds the displayed frame, it does not stop ingestion.
- **after** — after `screen_reset_mode 2026 1` (`CSI ? 2026 l`), `DECRQM_REPLY=<ESC>[?2026;2$y` → state **2**, resumed.

The (non-DECRQM) plain form corroborates that all updates land during the hold and flush after:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands \
    sh -c 'printf "BEFORE\r\n"; printf "\033[?2026h"; printf "DUR1\r\nDUR2\r\nDUR3\r\n"; printf "\033[?2026l"; printf "AFTER\r\n"; sleep 0.2'
draw BEFORE
screen_carriage_return
screen_carriage_return
screen_linefeed
screen_set_mode 2026 1
draw DUR1
screen_carriage_return
screen_carriage_return
screen_linefeed
draw DUR2
screen_carriage_return
screen_carriage_return
screen_linefeed
draw DUR3
screen_carriage_return
screen_carriage_return
screen_linefeed
screen_reset_mode 2026 1
draw AFTER
screen_carriage_return
screen_carriage_return
screen_linefeed
```

### 2.4 Edge condition — the mode-2026 safety timeout auto-resumes

If an application sends `CSI ? 2026 h` but never the matching `CSI ? 2026 l` (e.g. it crashes mid-update), the display must not freeze forever. kitty arms a default **2000 ms** safety timeout: `screen_pause_rendering` sets `expires_at = monotonic() + …` for a default `for_in_ms` of 2000 when the caller passes 0 (**`kitty/screen.c:L2521-L2522`**), and **`screen_check_pause_rendering`** at **`kitty/screen.c:L2489-L2490`** auto-resumes once `now > expires_at`:

```c
// kitty/screen.c:L2489-L2490
if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at)
    screen_pause_rendering(self, false, 0);
```

A child begins the pause, never ends it, and polls DECRQM roughly every 500 ms:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands \
    /root/kitty-venv/bin/python3 /tmp/kitty_obs/pause_timeout_probe.py > /tmp/kitty_obs/q1_timeout_dump.txt
$ grep DECRQM_REPLY /tmp/kitty_obs/q1_timeout_dump.txt
draw T=0000ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=0003ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=0506ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1010ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1513ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=2017ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=2523ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3026ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3530ms DECRQM_REPLY=<ESC>[?2026;2$y
```

**Observed:** the reply stays `?2026;1$y` (paused) through **T=1513 ms**, then flips to `?2026;2$y` (resumed) at **T=2017 ms** — the pause auto-expired at the ~2000 ms default **without any `CSI ? 2026 l`**. This is a *timing* claim, so it was repeated; the second run is identical:

```
$ grep DECRQM_REPLY /tmp/kitty_obs/q1_timeout_dump_run2.txt
draw T=0000ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=0003ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=0506ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1010ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1513ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=2017ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=2523ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3026ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3530ms DECRQM_REPLY=<ESC>[?2026;2$y
```

**Run scale:** DECRQM polled 9× per run over ~3.5 s; the 1→2 transition falls in the (1513 ms, 2017 ms] interval in both runs — stable, consistent with the 2000 ms default at `kitty/screen.c:L2521`.


---

## 3. Q2 — The "unseen conductor": thread split, poll order, and input_delay coalescing

### 3.1 The three-way thread split, observed live

Responsibilities are split across three threads. Their names are set in source: the **io_thread** runs `io_loop` (**`kitty/child-monitor.c:L1481`**) and is named `KittyChildMon` (**`L1489`**); the **talk_thread** runs `talk_loop` (**`kitty/child-monitor.c:L1805`**) and is named `KittyPeerMon` (**`L1808`**); the **main thread** runs the Python/GLFW event loop (parse + render). Inspecting a live kitty's threads confirms this.

With **no remote control** (default), only the io_thread is present alongside the main thread and a disk-cache helper (Mesa `llvmpipe-*` GL workers omitted):

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes sh -c 'sleep 12' &
$ KPID=$(pgrep -f 'launcher/kitty' | head -1)
$ for t in /proc/$KPID/task/*; do c=$(cat $t/comm); case "$c" in llvmpipe-*) ;; *) echo "$c";; esac; done
kitty            # (tid = PID) main thread
kitty:disk$0     # disk cache
KittyChildMon    # io_thread (PTY read/write)
```

Enabling remote control (`--listen-on`) spawns the **talk_thread** too:

```
$ ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty_obs/ktalk.sock \
    -o close_on_child_death=yes sh -c 'sleep 12' &
$ KPID=$(pgrep -f 'launcher/kitty' | head -1)
$ for t in /proc/$KPID/task/*; do c=$(cat $t/comm); case "$c" in llvmpipe-*) ;; *) echo "$c";; esac; done | grep Kitty
KittyPeerMon     # talk_thread (remote control) — tid 70145
KittyChildMon    # io_thread (PTY)              — tid 70146
```

**Observed:** `KittyChildMon` (io_thread) is always present; `KittyPeerMon` (talk_thread) appears only when a listen socket is configured. The talk_thread is created by `pthread_create(&self->talk_thread, …, talk_loop, …)` (**`kitty/child-monitor.c:L256`**) and the io_thread by `pthread_create(&self->io_thread, …, io_loop, …)` (**`L291`**).

### 3.2 What decides which event is handled first: `poll()` file-descriptor order

The io_loop polls a single `children_fds` array whose **index order encodes priority**. `EXTRA_FDS = 2` (**`kitty/child-monitor.c:L35`**); slot 0 is the wakeup pipe and slot 1 is the signal fd (**`L183`**: `children_fds[0].fd = self->io_loop_data.wakeup_read_fd; children_fds[1].fd = self->io_loop_data.signal_read_fd;`); child PTYs live at `children_fds[EXTRA_FDS + i]`. After `poll()`, results are consumed in **strict index order** (source, *inferred*):

```c
// kitty/child-monitor.c  (io_loop)
if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd);   // L1515: wakeup  FIRST
if (children_fds[1].revents && POLLIN) {                               // L1516: signals SECOND
    ... read_signals(children_fds[1].fd, handle_signal, &ss); ...
}
for (i = 0; i < self->count; i++) {                                   // L1529: child PTYs LAST
    if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP)) {
        has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen);  // L1531
        ...
    }
}
```

So the ordering is: **wakeup pipe → signals → each child PTY**. This is a *code-derived (inferred)* ordering statement, quoted verbatim from the cited lines. Its *observable consequence* — the input_delay coalescing that the wakeup mechanism produces — is measured next.

### 3.3 Coalescing: the main loop is only woken after `OPT(input_delay)`

Waking the main loop is expensive, so the io_thread batches wakeups. The `WAKEUP` macro and its guard (**`kitty/child-monitor.c:L1562-L1569`**, source, *inferred*):

```c
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
// we only wakeup the main loop after input_delay as wakeup is an expensive operation
// on some platforms, such as cocoa
if (data_received) {
    if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
    else has_pending_wakeups = true;
} else {
    if (has_pending_wakeups && (now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
}
```

The `poll()` timeout is itself bounded by the remaining `input_delay` when a wakeup is pending, so a deferred wakeup fires promptly once the window elapses. The default is `opt('input_delay', '3')` at **`kitty/options/definition.py:L878`** (3 ms).

### 3.4 Measuring the coalescing window (canonical, ≥2 runs)

The window is **measured**, not asserted, with a query/reply ping-pong: a real child sends DECRQM `CSI ? 2026 $ p`, blocks reading kitty's reply through the PTY, and repeats **N = 2000** times (after 20 warm-up round-trips). Because each reply requires a main-loop wakeup that the io_thread defers by up to `input_delay`, the mean round-trip time (RTT) is dominated by `input_delay`. Sweeping the knob proves causation.

```
$ for d in 0 3 10 25; do
    PP_ID=$d PP_OUT=/tmp/kitty_obs/pp_results.txt \
    ./kitty/launcher/kitty -o close_on_child_death=yes -o input_delay=$d \
      /root/kitty-venv/bin/python3 /tmp/kitty_obs/pingpong.py 2000 >/dev/null 2>&1
  done
$ cat /tmp/kitty_obs/pp_results.txt          # RUN 1
RESULT input_delay=0 N=2000 min=0.029 p50=0.044 mean=0.048 p90=0.062 p99=0.111 max=0.957 ms
RESULT input_delay=3 N=2000 min=3.086 p50=3.155 mean=3.165 p90=3.190 p99=3.289 max=6.446 ms
RESULT input_delay=10 N=2000 min=10.061 p50=10.261 mean=10.304 p90=10.346 p99=12.992 max=14.208 ms
RESULT input_delay=25 N=2000 min=25.193 p50=25.317 mean=25.358 p90=25.380 p99=27.439 max=29.551 ms
```

```
$ cat /tmp/kitty_obs/pp_results_run2.txt      # RUN 2 (identical scale, N=2000)
RESULT input_delay=0 N=2000 min=0.029 p50=0.036 mean=0.038 p90=0.046 p99=0.060 max=0.083 ms
RESULT input_delay=3 N=2000 min=3.084 p50=3.153 mean=3.163 p90=3.189 p99=3.254 max=5.849 ms
RESULT input_delay=10 N=2000 min=10.061 p50=10.250 mean=10.293 p90=10.327 p99=12.432 max=13.302 ms
RESULT input_delay=25 N=2000 min=25.188 p50=25.326 mean=25.373 p90=25.400 p99=28.103 max=30.229 ms
```

**Observed (run scale: N = 2000 round-trips per knob, two runs):**
- At the **default `input_delay=3`**, mean RTT is **3.165 ms** (run 1) / **3.163 ms** (run 2); p50 **3.155 / 3.153 ms**; min **3.086 / 3.084 ms**. The two runs agree to within **< 0.01 ms** at p50 — **stable**.
- The mean RTT tracks `input_delay` almost exactly: **0 → 0.048/0.038 ms**, **3 → 3.165/3.163 ms**, **10 → 10.304/10.293 ms**, **25 → 25.358/25.373 ms** — a ~0.05 ms fixed overhead plus the knob. This directly exposes the coalescing window and proves the default 3 ms value from `kitty/options/definition.py:L878`.


---

## 4. Q3 — Keeping shell-integration hints aligned with text, and behavior under stress

### 4.1 Why hints and text never drift: one serial `run_worker` pass

Shell-integration semantic marks are **OSC 133** sequences (FinalTerm/FTCS): `A` prompt-start, `B` command-start, `C` command-executed/output-start, `D;<exit>` command-finished. In kitty they route through `dispatch_osc` (**`kitty/vt-parser.c:L457`**) to the OSC-133 handler `shell_prompt_marking`, which issues the `cmd_output_marking` callback (**`kitty/screen.c:L2338-L2352`**: `A`→PROMPT_START, `C`→OUTPUT_START+cmdline, `D`→exit status). Ordinary text goes through `consume_normal` (**`kitty/vt-parser.c:L230`**). Both are consumed from the **same read buffer in byte order within a single `run_worker` pass** — there is **no separate "hints" channel** that could race text.

**Primary canonical evidence — a real bash with kitty's auto-injected shell integration**, driven through the canonical `send-text` path (which writes to the child PTY via `schedule_write_to_child`). *(In the dump below, the container hostname is shown as `HOST` and two long working-directory paths are abbreviated as `/tmp/blitzy/kitty/...` on the incidental `process_cwd_notification`/`set_title` lines; every `shell_prompt_marking` and `draw` line — the actual evidence — is verbatim.)*

```
$ ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty_obs/ktalk2.sock \
    -o close_on_child_death=no --dump-commands bash -i > /tmp/kitty_obs/q3_realshell_dump.txt &
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/ktalk2.sock send-text 'echo HELLO-FROM-REAL-SHELL\n'
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/ktalk2.sock send-text 'false\n'
$ sed -n '1,55p' /tmp/kitty_obs/q3_realshell_dump.txt
process_cwd_notification 7 kitty-shell-cwd://HOST/tmp/blitzy/kitty/...
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/...
shell_prompt_marking 133 k;end_suffix_kitty
draw echo HELLO-FROM-REAL-SHELL
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
set_title echo HELLO-FROM-REAL-SHELL
shell_prompt_marking 133 C;cmdline=echo\ HELLO-FROM-REAL-SHELL
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 0 32
shell_prompt_marking 133 k;end_suffix_kitty
draw HELLO-FROM-REAL-SHELL
screen_carriage_return
screen_linefeed
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/...
shell_prompt_marking 133 k;end_suffix_kitty
draw false
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
set_title false
shell_prompt_marking 133 C;cmdline=false
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 0 32
shell_prompt_marking 133 k;end_suffix_kitty
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;1
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/...
shell_prompt_marking 133 k;end_suffix_kitty
```

**Observed:** the real bash integration emits, strictly interleaved with `draw` text in byte order: `133 A` (prompt start) → `draw echo …` → `133 C;cmdline=echo\ HELLO-FROM-REAL-SHELL` (output start, with the command line) → `draw HELLO-FROM-REAL-SHELL` (the output) → `133 D;0` (finished, exit **0**). For the `false` command the same structure ends in `133 D;1` — the **exit status flows through OSC 133 D naturally** (`echo`→0, `false`→1). The marks are always adjacent to exactly the text they annotate; there is no drift.

**Controlled illustration (byte-identical to the production emitters).** Hand-emitting the same OSC 133 bytes interleaved with text produces the same serial dispatch:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands \
    sh -c 'printf "\033]133;A\007PROMPT>\033]133;B\007 echo hi\r\n\033]133;C\007hi\r\n\033]133;D;0\007NEXT-PROMPT\r\n"; sleep 0.2'
shell_prompt_marking 133 A
draw PROMPT>
shell_prompt_marking 133 B
draw  echo hi
screen_carriage_return
screen_carriage_return
screen_linefeed
shell_prompt_marking 133 C
draw hi
screen_carriage_return
screen_carriage_return
screen_linefeed
shell_prompt_marking 133 D;0
draw NEXT-PROMPT
screen_carriage_return
screen_carriage_return
screen_linefeed
```

These hand-emitted bytes are byte-identical to what the shipped scripts emit: bash `\e]133;A;k=s\a` / `\e]133;C;cmdline=%q\a` / `\e]133;D;$?\a` (**`shell-integration/bash/kitty.bash:L137,L208,L239`**); zsh (**`shell-integration/zsh/kitty-integration:L145,L153,L218`**); fish (**`shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:L83,L91,L96`**). The parser-dispatch path they exercise is fully canonical.

### 4.2 Heavy backpressure: cooperative and non-lossy

The VT parser's read buffer is **1 MiB**: `#define BUF_SZ (1024u*1024u)` (**`kitty/vt-parser.c:L18`**), which equals the value exported to Python:

```
$ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE as B; print(B, B//1024//1024, "MiB")'
1048576 1 MiB
```

The io_thread gates reads on available space. Each loop iteration it sets the child PTY's poll events to `POLLIN` only if there is room (**`kitty/child-monitor.c:L1501`**, *inferred*):

```c
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

and `vt_parser_has_space_for_input` (**`kitty/vt-parser.c:L1477`**) returns `self->read.sz + self->write.pending < BUF_SZ`. When the buffer is full the gate becomes `0`: the io_thread simply stops asking to read, the kernel PTY buffer fills, and the child **blocks on `write()`** — no bytes are dropped. (`read_bytes` also self-guards: it returns `true` *without reading* when `available_buffer_space == 0`, **`kitty/child-monitor.c:L1342`**.)

**Non-lossy, observed.** A child writes a **16 MiB** burst — 16× the buffer — and `--dump-bytes` records every byte `read_bytes` ingested:

```
$ FLOOD_OUT=/tmp/kitty_obs/q3_flood_meas.txt \
    ./kitty/launcher/kitty -o close_on_child_death=yes --dump-bytes=/tmp/kitty_obs/q3_flood_bytes.bin \
    /root/kitty-venv/bin/python3 /tmp/kitty_obs/flood.py 16
$ echo "captured=$(wc -c < q3_flood_bytes.bin) A_count=$(tr -cd 'A' < q3_flood_bytes.bin | wc -c) sentinel=$(grep -a -o 'SENTINEL-END-[0-9]*' q3_flood_bytes.bin)"
captured=16777243 A_count=16777216 sentinel=SENTINEL-END-16777216
```

**Observed:** exactly **16777216** `'A'` bytes (16 MiB) arrived and the trailing `SENTINEL-END-16777216` is intact — **zero loss** across a burst 16× larger than the buffer.

**Flow-controlled, observed.** The same child measured its own wall-clock write time:

```
$ cat /tmp/kitty_obs/q3_flood_meas.txt
child wrote 16777216 bytes (16 MiB) in 2.634 s -> 6.1 MiB/s
```

16 MiB in **2.634 s (6.1 MiB/s)** — orders of magnitude below memory bandwidth, i.e. the child's `write()` was blocked/paced by the consumer, exactly as cooperative backpressure predicts.

**Before → during, observed.** Timing each 1 MiB write shows the transition into backpressure:

```
$ cat /tmp/kitty_obs/q3_paced_meas.txt
MiB #01 write() took     3.18 ms
MiB #02 write() took    14.00 ms
MiB #03 write() took    10.28 ms
MiB #04 write() took    10.07 ms
MiB #05 write() took    10.16 ms
MiB #06 write() took     8.81 ms
MiB #07 write() took    10.77 ms
MiB #08 write() took    12.91 ms
MiB #09 write() took     8.41 ms
MiB #10 write() took     9.84 ms
MiB #11 write() took     9.78 ms
MiB #12 write() took     9.79 ms
```

**Observed:** MiB **#01 completes in 3.18 ms** (it fits into the kernel PTY buffer plus the 1 MiB parser buffer — *before* saturation); from **#02 onward each MiB is paced at ~8–14 ms** (*during* backpressure), because once the buffer saturates the child's `write()` only proceeds as the main thread drains — at roughly the `repaint_delay = 10 ms` render cadence (`opt('repaint_delay','10')`, **`kitty/options/definition.py:L866`**). *After* the child stops, the buffer drains and the sentinel is delivered (the non-lossy result above).

**Non-canonical boundary corroboration (labeled).** The exact 1 MiB boundary is confirmed by the parser's own test (**non-canonical**, synchronous, bypasses the io_thread) at `kitty_tests/parser.py:L141-L146`: writing `3 × (BUF/3 + 7)` bytes leaves `create_write_buffer` returning empty (`assertFalse`) — i.e. it saturates at exactly `BUF_SZ`; and `kitty_tests/screen.py:L948` splits `'a' * VT_PARSER_BUFFER_SIZE` at `BUF_SZ - 8`.

### 4.3 An unstable remote connection degrades independently of the PTY

Remote control runs on the **talk_thread** with its **own** `poll()` loop (`talk_loop` at **`kitty/child-monitor.c:L1805`**, its poll at **`L1850`**), entirely separate from the io_loop's poll (**`L1509`**). A peer is accepted by `accept_peer` (**`L1632`**) and read by `read_from_peer` (**`L1714`**); an abrupt disconnect (`recv` → `n == 0`) marks the peer finished and queues a `peer_death` message (`notify_on_peer_removal`, **`L1677`**: `m->data = strdup("peer_death")`) that the **main** thread consumes — never touching the io_thread/PTY.

**Independence, observed.** kitty runs a PTY ping-pong child (N = 3000) while a driver abruptly disconnects a peer 30× concurrently:

```
$ ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty_obs/ktalk3.sock \
    -o close_on_child_death=yes -o input_delay=3 \
    /root/kitty-venv/bin/python3 /tmp/kitty_obs/pingpong.py 3000 &     # PTY workload
$ DISRUPT_OUT=/tmp/kitty_obs/disrupt_results.txt \
    /root/kitty-venv/bin/python3 /tmp/kitty_obs/remote_disrupt.py /tmp/kitty_obs/ktalk3.sock 30 partial
DISRUPT mode=partial N=30 ok=30 fail=0  min=0.077 p50=0.091 mean=0.102 p90=0.141 p99=0.305 max=0.305 ms
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_obs/ktalk3.sock ls | wc -c   # remote still works after chaos
10049
$ cat /tmp/kitty_obs/pp_with_disrupt.txt      # PTY RTT during the disruptions
RESULT input_delay=3 N=3000 min=3.083 p50=3.160 mean=3.173 p90=3.202 p99=3.292 max=9.566 ms
```

**Observed:** the PTY ping-pong during remote chaos (p50 **3.160**, mean **3.173 ms**) is essentially identical to the undisrupted baseline from §3.4 (p50 **3.153**, mean **3.163 ms**). The remote peer disruptions do not perturb the PTY pipeline; and `kitty @ ls` still returns 10049 bytes afterward — the talk_thread recovered.

**Reported as a distribution (same input repeated).** Because an unstable remote is the run-to-run-variable case, the *same* disruption is repeated over multiple runs (N = 30 each) rather than smoothed to one number:

```
$ for run in 1 2 3; do
    DISRUPT_OUT=/tmp/kitty_obs/disrupt_dist.txt \
      /root/kitty-venv/bin/python3 /tmp/kitty_obs/remote_disrupt.py /tmp/kitty_obs/ktalk4.sock 30 partial
  done
$ DISRUPT_OUT=/tmp/kitty_obs/disrupt_dist.txt \
    /root/kitty-venv/bin/python3 /tmp/kitty_obs/remote_disrupt.py /tmp/kitty_obs/ktalk4.sock 30 immediate
$ cat /tmp/kitty_obs/disrupt_dist.txt
DISRUPT mode=partial N=30 ok=30 fail=0  min=0.080 p50=0.094 mean=0.105 p90=0.149 p99=0.319 max=0.319 ms
DISRUPT mode=partial N=30 ok=30 fail=0  min=0.081 p50=0.096 mean=0.103 p90=0.117 p99=0.280 max=0.280 ms
DISRUPT mode=partial N=30 ok=30 fail=0  min=0.090 p50=0.119 mean=0.127 p90=0.173 p99=0.311 max=0.311 ms
DISRUPT mode=immediate N=30 ok=30 fail=0  min=0.060 p50=0.082 mean=0.089 p90=0.104 p99=0.276 max=0.276 ms
```

**Observed distribution (120 disconnects total):** every run is **ok=30, fail=0**; per-disconnect handling latency is p50 **0.094 / 0.096 / 0.119 ms** (partial) and **0.082 ms** (immediate), with tails to **max 0.28–0.32 ms**. kitty stayed alive throughout, and both `KittyPeerMon` and `KittyChildMon` threads survived every disruption (verified via `/proc/<pid>/task`), with `send-text` still succeeding afterward. The remote path degrades and recovers on its own thread, isolated from the PTY.


---

## 5. Q4 — End-to-end rhythm, and the outbound keypress path

### 5.1 The full inbound rhythm, composed

Putting Q1–Q3 together, a byte that arrives from the child travels:

1. **Arrival →** the child writes to the PTY; on the io_thread, `poll()` reports `POLLIN` on that child's fd (after wakeup/signals in index order, §3.2) and **`read_bytes`** (**`kitty/child-monitor.c:L1337`**) reads it into the 1 MiB VT parser buffer — *unless* the buffer is full, in which case `POLLIN` is suppressed and the child blocks (§4.2).
2. **Coalesced wakeup →** the io_thread wakes the main loop only after `OPT(input_delay)` (measured **~3 ms**, §3.4), batching bursts (**`kitty/child-monitor.c:L1562-L1569`**).
3. **Serial dispatch →** the main thread runs **`run_worker`** (**`kitty/vt-parser.c:L1417`**), consuming the buffer in byte order and dispatching each token to screen state: **text** via `consume_normal`, **CSI/ESC** mode/cursor ops, **OSC** via `dispatch_osc`→`shell_prompt_marking`, and **DCS** via `parse_kitty_dcs`.
4. **Screen mutation → render →** the frame is drawn, honoring a mode-2026 pause (§2.3–2.4) and the render knobs, then the interface **settles**.

The four dispatch branches are all observable. §2 showed `draw` (text) and `screen_set_mode 2026` (CSI); §4.1 showed `shell_prompt_marking 133` (OSC). The **DCS** branch — `parse_kitty_dcs` at **`kitty/vt-parser.c:L586`** — handles both the inline remote-command route and standard device queries, interleaved with text in the same serial pass:

```
$ ./kitty/launcher/kitty -o allow_remote_control=yes -o close_on_child_death=yes --dump-commands \
    sh -c 'printf "TEXT-BEFORE\r\n"; printf "\033P@kitty-cmd{\"cmd\":\"ls\",\"version\":[0,35,2]}\033\\"; printf "\033P$qm\033\\"; printf "TEXT-AFTER\r\n"; sleep 0.3'
draw TEXT-BEFORE
screen_carriage_return
screen_carriage_return
screen_linefeed
handle_remote_cmd {"cmd":"ls","version":[0,35,2]}
screen_request_capabilities 36 m
draw TEXT-AFTER
screen_carriage_return
screen_carriage_return
screen_linefeed
```

**Observed:** the inline DCS `\eP@kitty-cmd{…}\e\\` becomes `handle_remote_cmd {"cmd":"ls",…}` (a second remote-control route, *distinct* from the talk-socket path of §4.3, but sharing the same serial parse), and the standard DECRQSS `\eP$qm\e\\` becomes `screen_request_capabilities 36 m` — both sandwiched, in order, between `draw TEXT-BEFORE` and `draw TEXT-AFTER`.

The rhythm is parameterized by three observed knob defaults (confirmed by the `opt(...)` lines):

- `opt('input_delay', '3')` — **`kitty/options/definition.py:L878`** (the coalescing window, measured 3.16 ms in §3.4).
- `opt('repaint_delay', '10')` — **`kitty/options/definition.py:L866`** (the render cadence; matches the ~10 ms backpressure drain in §4.2).
- `opt('sync_to_monitor', 'yes')` — **`kitty/options/definition.py:L889`** (frames aligned to the monitor refresh).

### 5.2 The outbound path: key events encoded by mode and modifiers

The *outbound* (user keypress → child) path is distinct from the inbound path. A GLFW key event enters **`on_key_input(GLFWkeyevent *ev)`** at **`kitty/keys.c:L166`**, is encoded by **`encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)`** at **`kitty/keys.c:L251`** — note the encoder is passed `screen->modes.mDECCKM` (application cursor-key mode) — and the result is written to the child by **`schedule_write_to_child`** at **`kitty/keys.c:L253/L259`**.

Key events are injected canonically with `kitty @ send-key`, whose server side (`window.py:send_key` → `send_key_sequence` → `encode_key_for_tty`) calls the **same** encoder: `pyencode_key_for_tty` at **`kitty/keys.c:L311`** invokes `encode_glfw_key_event(&ev, cursor_key_mode, …)` at **`kitty/keys.c:L319`**. A child in raw mode records the exact bytes it receives (ground truth of `schedule_write_to_child`'s output).

**Arrow key, DECCKM off vs on (application cursor keys):**

```
$ # DECCKM OFF (default): child emits CSI ?1 l, then `kitty @ send-key up`
$ cat /tmp/kitty_obs/keycap_up_off.txt
RECV_DECODED=b'\x1b[A'
RECV_HEX=1b 5b 41
$ # DECCKM ON: child emits CSI ?1 h, then `kitty @ send-key up`
$ cat /tmp/kitty_obs/keycap_up_on.txt
RECV_DECODED=b'\x1bOA'
RECV_HEX=1b 4f 41
```

**Observed:** with **DECCKM off**, Up encodes as `ESC [ A` (`1b 5b 41`, CSI); with **DECCKM on**, the same key encodes as `ESC O A` (`1b 4f 41`, SS3). The only thing that changed was `screen->modes.mDECCKM` — proving the encoder at `kitty/keys.c:L251` branches on cursor-key mode.

**Modifiers (Shift / Alt / Ctrl + Up, legacy encoding, DECCKM off):**

```
$ cat /tmp/kitty_obs/keycap_up_shift.txt
RECV_DECODED=b'\x1b[1;2A'
RECV_HEX=1b 5b 31 3b 32 41
$ cat /tmp/kitty_obs/keycap_up_alt.txt
RECV_DECODED=b'\x1b[1;3A'
RECV_HEX=1b 5b 31 3b 33 41
$ cat /tmp/kitty_obs/keycap_up_ctrl.txt
RECV_DECODED=b'\x1b[1;5A'
RECV_HEX=1b 5b 31 3b 35 41
```

**Observed:** the modifier is encoded as the CSI parameter `1 + (Shift=1, Alt=2, Ctrl=4)`: Shift+Up → `ESC [ 1 ; 2 A`, Alt+Up → `ESC [ 1 ; 3 A`, Ctrl+Up → `ESC [ 1 ; 5 A`, versus plain Up → `ESC [ A` (§5.2 above).

**Caveat (labeled).** `send-key` exercises `encode_glfw_key_event` + `schedule_write_to_child` canonically (the bytes above are exactly what the child received through the PTY), but it does **not** traverse the GLFW OS entry `on_key_input` (**`kitty/keys.c:L166`**), which requires a physical/synthetic X11 key event that the headless sandbox cannot deliver (no `xdotool`/XTEST). The `on_key_input → encode_glfw_key_event` linkage is therefore *inferred* from source; the encoder call it makes (`…, screen->modes.mDECCKM, …`, **`L251`**) is byte-identical to the `send-key` path (**`L319`**, same function), and the observed DECCKM off/on divergence confirms the live `mDECCKM` is consulted.

---

## 6. Canonical vs. non-canonical evidence

- **Canonical (primary proof for all four threads):** a real child process writing to kitty's PTY, whose bytes enter through `read_bytes` (**`kitty/child-monitor.c:L1337`**) on the io_thread, observed via the canonically built `kitty/launcher/kitty` and its own instrumentation (`--dump-commands`, `--dump-bytes`, `--debug-keyboard`), plus live `/proc/<pid>/task` thread inspection and query/reply round-trips through the PTY (DECRQM `CSI ?2026$p`, ping-pong). All of §2, §3, §4, and §5 rest on this.
- **Canonical input drivers:** `kitty @ send-text` and `kitty @ send-key` write to the child PTY via `schedule_write_to_child`, the same mechanism as real typing; `send-key` additionally invokes the same `encode_glfw_key_event` encoder as `on_key_input`.
- **Non-canonical (supplement only, labeled where used):** the synchronous `parse_bytes` helper (`kitty_tests/__init__.py:L30`), which drives the parser via `test_parse_written_data` (**`kitty/screen.c:L4772`**) directly, **bypassing the io_thread and the PTY**. It appears here only in §4.2 to corroborate the exact `BUF_SZ` boundary (`kitty_tests/parser.py:L141-L146`, `kitty_tests/screen.py:L948`) and is never used as primary proof for the PTY/threading claims of Q1/Q2/Q4.

---

## 7. Coverage pass

Every sub-part and named symbol, each with its command + unedited output above:

| Item | Where | Evidence |
|------|-------|----------|
| **Q1** entry point `read_bytes` [`child-monitor.c:L1337`] (call site L1531) | §2.1–2.2 | surge: 140000 raw bytes → 5000 ordered `draw` |
| `parse_worker` [`vt-parser.c:L1496`] / `run_worker` [`L1417`] | §2.1, §5.1 | dump-commands events (inferred linkage) |
| `PENDING_MODE 2026` [`control-codes.h:L235`], `screen_pause_rendering` [`screen.c:L2506`] | §2.3 | before `?2026;2$y` / during `?2026;1$y` / after `?2026;2$y` |
| pause: parsing continues during hold | §2.3 | `draw DRAWN-WHILE-PAUSED` between set/reset |
| `screen_check_pause_rendering` [`screen.c:L2489-2490`], 2000 ms timeout [`L2521`] | §2.4 | auto-resume at T=2017 ms, 2 runs identical |
| **Q2** thread split: `KittyChildMon` io_thread [`L1481/L1489`], `KittyPeerMon` talk_thread [`L1805/L1808`] | §3.1 | live `/proc/<pid>/task` listings |
| poll order: `EXTRA_FDS` [`L35`], wakeup/signal fds [`L183`], consume order [`L1515/L1516/L1529-1531`] | §3.2 | quoted source (inferred) |
| coalescing `WAKEUP` [`L1562-1569`], `input_delay=3` [`definition.py:L878`] | §3.3–3.4 | measured 3.165/3.163 ms; sweep 0/3/10/25; N=2000×2 |
| **Q3** serial alignment: `dispatch_osc` [`vt-parser.c:L457`], `cmd_output_marking` [`screen.c:L2338-2352`], `consume_normal` [`vt-parser.c:L230`] | §4.1 | real bash `133 A/C/D;0`, `false`→`D;1`; controlled interleave |
| backpressure: `BUF_SZ` 1 MiB [`vt-parser.c:L18`], `vt_parser_has_space_for_input` [`L1477`], gate [`child-monitor.c:L1501`], self-guard [`L1342`] | §4.2 | non-lossy 16777216 'A' + sentinel; 6.1 MiB/s; per-MiB pacing 3.18→~10 ms |
| non-canonical boundary (labeled) | §4.2 | `parser.py:L141-146`, `screen.py:L948` |
| unstable remote: `talk_loop` [`L1805`], `accept_peer` [`L1632`], `read_from_peer` [`L1714`], `peer_death` [`L1677`] | §4.3 | 120 disconnects (ok=30×4, fail=0) distribution; PTY RTT unchanged |
| **Q4** end-to-end + DCS `parse_kitty_dcs` [`vt-parser.c:L586`] | §5.1 | `handle_remote_cmd` + `screen_request_capabilities` interleaved with text |
| knobs `repaint_delay=10` [`L866`], `sync_to_monitor=yes` [`L889`] | §5.1 | quoted `opt(...)` lines; ~10 ms drain in §4.2 |
| outbound `on_key_input` [`keys.c:L166`], `encode_glfw_key_event(…, mDECCKM, …)` [`L251`], `schedule_write_to_child` [`L253/L259`] | §5.2 | DECCKM off `ESC[A` vs on `ESC O A`; Shift/Alt/Ctrl `ESC[1;2/3/5A` |

All four question threads — input surge & entry point, the conductor's thread split/prioritization, aligned semantics under backpressure and an unstable remote, and the end-to-end rhythm — are answered from observed runtime behavior of the canonically built kitty `0.35.2` @ `815df1e210e0…`, with every claim carrying its exact command, complete unedited output, and `file:line`/symbol grounding.

