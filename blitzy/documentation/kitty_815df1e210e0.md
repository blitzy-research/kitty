# How kitty handles a child that prints a few lines and exits with status 0

An investigative, runtime-observed answer, grounded in captured output and `file:line` source citations.

## Scope and provenance

- **Investigated software:** `kitty` terminal emulator, repository `kovidgoyal/kitty`.
- **Investigated (source) commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. Every `file:line` citation in this document refers to the tree at that commit.
- **This document lives** on branch `blitzy-4da995f7-9d87-4535-892b-6e7ceff8c1ad`. The branch `HEAD` at authoring time was the commit that first added this file (`9fd0da15ff09436c0c3c40eb7dbdda3556358f34`, *"docs: add investigative answer for kitty child-exit-0 behavior"*); this HEAD is the **documentation** commit, **not** a commit of the investigated source. No source file under `kitty/`, `kittens/`, `tools/`, or `shell-integration/` was modified — the only change on this branch is the creation of this Markdown file.
- **Methodology (per the SWE-AtlasQnA rules):** the relevant code paths were **built and run first**, real output was captured with temporary observation scripts, and only then was this answer written. Every behavioral claim is presented next to the exact command that produced it and the unedited output, and is labeled **[observed]**, **[source-derived]**, **[inferred]**, or **[non-canonical corroboration]**. All temporary scripts live outside the repository (under `/tmp/kwork`) and the repository is left byte-for-byte unchanged apart from this file.

### The nine questions

1. End-to-end flow from "child is running" through "child exits 0, having printed a few lines to stdout".
2. Kitty's own process exit code when the child exits with status 0.
3. The full, exact text of any completion message shown to the user.
4. Which component tracks the child across its lifetime.
5. The single function that turns the child's exit status into the user-facing message.
6. The OS signal kitty listens for to learn a child terminated.
7. The system call kitty uses to retrieve the child's exit status.
8. How the exit status travels from the shell to the message-generating function (protocol / escape sequence).
9. Where the child's printed output actually appears.

## Environment and canonical build

### Honest environment disclosure (finding of provenance)

The project's setup instructions name a private image, `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0...` from `ghcr.io/scaleapi/swe-atlas`. That image is **private and could not be pulled** in this sandbox (no registry credentials). However, **this sandbox host is itself the coding-agent environment** — it contains the same toolchain (Python 3.11, Go 1.22, a C toolchain, and all of kitty's build dependencies) — so the canonical build and all runs below were performed **natively on this host**. This deviation from "run inside the named container" is disclosed explicitly here; it does not affect any observed value, because the build and runs use kitty's real, default entry points.

### Build host and toolchain **[observed]**

```
$ cat /tmp/kwork/evidence/build_provenance.txt
### uname / OS
Linux reverse-code-generator-04182660-cdbt8 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux
PRETTY_NAME="Ubuntu 25.10"
NAME="Ubuntu"
### interpreter + toolchain
python3 -> /usr/local/bin/python3.11 = Python 3.11.13
go version go1.22.12 linux/amd64
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
GNU bash, version 5.2.37(1)-release (x86_64-pc-linux-gnu)
### git provenance
branch=blitzy-4da995f7-9d87-4535-892b-6e7ceff8c1ad
HEAD=9fd0da15ff09436c0c3c40eb7dbdda3556358f34
investigated source commit target = 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

`python3` is a shim pointing at Python 3.11.13 (kitty's runtime interpreter for this build); the system default is 3.13, but the project's highest explicitly-supported CI version is 3.11, so 3.11 is used.

### Canonical default build: `make` **[observed]**

Kitty's canonical build is `make`, whose `all:` target invokes `python3 setup.py` (default, no `--debug`). A full clean rebuild:

```
$ PATH=/usr/local/go/bin:$PATH GOPATH=/root/go CI=true make clean
$ ( set -o pipefail; time make > /tmp/kwork/evidence/make_full.log 2>&1; echo "MAKE_EXIT=${PIPESTATUS[0]}" ) 2> /tmp/kwork/evidence/make_full.time

$ head -16 /tmp/kwork/evidence/make_full.log
python3 setup.py
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
[5/85] Compiling kitty/graphics.c ...
[6/85] Compiling kitty/child-monitor.c ...
[7/85] Compiling kitty/fonts.c ...
[8/85] Compiling kitty/shaders.c ...

$ cat /tmp/kwork/evidence/make_full.time
real	1m10.263s
user	2m31.647s
sys	0m23.974s
```

The build exits **0** (`MAKE_EXIT=0`), compiles 85 native units and links the Go `kitten` tools, and produces the launcher and C extension:

- `kitty/launcher/kitty` and `kitty/launcher/kitten`
- `kitty/fast_data_types.so`

The build auto-disables the Wayland backend because `wayland-protocols` dev headers are absent, and builds **x11-only**. This is a supported default reaction and does **not** affect the child-exit investigation (the process/PTY/signal/parser paths are backend-independent). The resulting version string is:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### How runs were performed (headless GUI) **[observed]**

Because kitty is a GPU terminal, runs use a headless X server plus software GL:

```
Xvfb :99 -screen 0 1280x800x24 -nolisten tcp     # started once; PID captured; torn down via trap
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
```

The canonical reproduction command used throughout is:

```
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
```

`--config NONE` guarantees **default** configuration (no user `kitty.conf` is read), so every observed value reflects what a normal user sees with defaults.

### Safe temporary-file handling **[observed]**

All observation artifacts live under a dedicated directory created with `mktemp -d` semantics (mode `700`), background servers have their PIDs captured and are stopped via shell `trap`, pipelines use `set -o pipefail`, and each capture checks its exit status. The full, self-contained harnesses are reproduced verbatim in the **Reproducibility** section; nothing is written inside the repository.

## Executive summary

When kitty launches a child that prints a few lines and exits 0:

- The child runs on the **slave side of a pseudo-terminal (PTY)**; kitty's `ChildMonitor` owns the **master** side, the child PID, and the window's `Screen`.
- The child's printed bytes are read from the PTY master, fed to kitty's VT parser, written into the `Screen` line buffer, and rendered into kitty's **on-screen terminal grid** by the GPU renderer — they appear **in the kitty window**, not on kitty's own stdout/stderr.
- The child's termination is delivered to kitty as the OS signal **`SIGCHLD`** (observed as `signalfd` siginfo `ssi_signo=17`, `ssi_code=1`=`CLD_EXITED`), and kitty reaps the zombie with **`wait4(-1, &status, WNOHANG)`** (the `waitpid` family).
- **Kitty's own process exits with code `0`** — independently of the child's status (a child exiting 5 or 42 still yields kitty exit 0).
- **Two different "completion" behaviors** must be distinguished:
  - The direct-reap path (`SIGCHLD` → `wait4`) carries **no** exit status into the window callback `on_child_death(window_id)` (it receives only a window id). By **default** the window simply **closes/tears down**; there is **no persistent completion message** in the default case.
  - A **completion message** appears only via **shell integration**: the integrated shell emits an **OSC 133 `D;<status>`** escape sequence, which kitty decodes and — when `notify_on_cmd_finish` is enabled — turns into the notification body **`Command <cmd> finished with status: 0.\nClick to focus.`** via `Window.handle_cmd_end()`.
  - The `--hold` **flag** holds the window open with an **interactive shell** (env `KITTY_HOLD=1`), showing **no banner**; the separate `kitty +hold` / hold-kitten entry point shows the green banner **`Press Enter or Esc to exit`**.
- **Default window teardown** for a child with no lingering PTY writer is driven by **PTY end-of-file (EIO/EOF)**, not by the `SIGCHLD` reap: with `close_on_child_death` at its default (`no`), kitty removes the window when `read()` on the PTY master returns EOF/EIO; with `close_on_child_death=yes`, the reap immediately removes the window.

### One-line answers

1. **End-to-end flow:** `Child.fork()` spawns the program on a PTY slave → child prints to stdout → `ChildMonitor` reads the PTY master (`read_bytes`) → VT parser → `Screen` buffer → GPU render into kitty's grid; on exit, `SIGCHLD` → `wait4` reaps, and the window tears down (default) via PTY EOF. **[observed + source-derived]**
2. **Kitty's own exit code:** `0`, stable across 5 runs and independent of the child's status. **[observed]**
3. **Full completion message:** default direct launch shows **none**; shell-integration notification body is exactly `Command echo hello finished with status: 0.\nClick to focus.`; `kitty +hold` shows `Press Enter or Esc to exit`. **[observed]**
4. **Child-tracking component:** the **`ChildMonitor`** (C core `kitty/child-monitor.c`; created and populated from `kitty/boss.py`), which owns each child's PID, PTY master fd, and `Screen`. **[source-derived]**
5. **Message-generating function:** **`Window.handle_cmd_end()`** [`kitty/window.py:1408`], which builds the `Command … finished with status: …` body. **[observed + source-derived]**
6. **Termination signal:** **`SIGCHLD`** (signal 17), observed via `signalfd` siginfo. **[observed]**
7. **Status-retrieval system call:** **`wait4(-1, &status, WNOHANG)`** — the `waitpid` family — in `reap_children()`. **[observed + source-derived]**
8. **Status transport:** the **shell-reported command status** travels via the **OSC 133 `D;<code>`** escape sequence emitted by the integrated shell → `vt-parser.c` (OSC 133 dispatch) → `screen.c` `shell_prompt_marking()` (case `D`) → `Window.handle_cmd_end()`. **[observed + source-derived]**
9. **Output location:** the child's stdout appears in **kitty's on-screen terminal grid** (the kitty window), read from the PTY master; it does **not** appear on kitty's own stdout/stderr. **[observed]**

## Q1 — End-to-end flow (running → prints → exits 0)

### Command **[observed]**

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -e read=all \
    -o /tmp/kwork/evidence/kitty.strace \
    ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
$ echo "kitty_exit=$?"
kitty_exit=0
```

### Raw output — the contiguous death window from the strace **[observed]**

```
$ cat /tmp/kwork/evidence/q6q7_window.txt
146320 read(8, "hello\r\nworld\r\n", 1048576) = 14
146321 +++ exited with 0 +++
146320 read(8, 0x585866c7c78e, 1048562) = -1 EIO (Input/output error)
146320 read(7, "\21\0\0\0\0\0\0\0\1\0\0\0\221;\2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128
146320 read(7, 0x7c8262b143a0, 4096)    = -1 EAGAIN (Resource temporarily unavailable)
146320 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 146321
146320 wait4(-1, 0x7c82346c2e50, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

Here TID `146320` is kitty's monitor thread, `146321` is the child. Reading top to bottom: the monitor reads the child's `hello\r\nworld\r\n` (14 bytes) from PTY master fd 8; the child exits 0; the next PTY read returns `EIO` (the PTY-EOF condition after the last writer closed); a `signalfd` read on fd 7 delivers the `SIGCHLD` siginfo; and `wait4` reaps the child (returning pid `146321` with `WIFEXITED && WEXITSTATUS==0`), then a second `wait4` returns `ECHILD`.

### The spawn and wiring **[source-derived]**

The child is created by `Child.fork()`, which opens a PTY and calls the native `spawn`:

`kitty/child.py:276-283` (contiguous):

```python
    def fork(self) -> Optional[int]:
        if self.forked:
            return None
        opts = fast_data_types.get_options()
        self.forked = True
        master, slave = openpty()
        stdin, self.stdin = self.stdin, None
        ready_read_fd, ready_write_fd = os.pipe()
```

`kitty/child.py:327-340` (contiguous) — the native `spawn` is called and the resulting PID and PTY **master** fd are stored on the `Child`:

```python
        self.final_exe = final_exe = which(argv[0]) or argv[0]
        self.final_argv0 = argv[0]
        if self.hold:
            argv = cmdline_for_hold(argv)
            final_exe = argv[0]
        env = tuple(f'{k}={v}' for k, v in self.final_env.items())
        pid = fast_data_types.spawn(
            final_exe, cwd, tuple(argv), env, master, slave, stdin_read_fd, stdin_write_fd,
            ready_read_fd, ready_write_fd, tuple(handled_signals), kitten_exe(), opts.forward_stdio)
        os.close(slave)
        self.pid = pid
        self.child_fd = master
        if stdin is not None:
            os.close(stdin_read_fd)
```

Inside the native `spawn`, the child's stdin/stdout/stderr are redirected to the PTY **slave** — so everything the child prints goes down the PTY, `kitty/child.c:136-146` (contiguous):

```c
            }
            // Redirect stdin/stdout/stderr to the pty
            if (safe_dup2(slave, STDOUT_FILENO) == -1) exit_on_err("dup2() failed for fd number 1");
            if (safe_dup2(slave, STDERR_FILENO) == -1) exit_on_err("dup2() failed for fd number 2");
            if (stdin_read_fd > -1) {
                if (safe_dup2(stdin_read_fd, STDIN_FILENO) == -1) exit_on_err("dup2() failed for fd number 0");
                safe_close(stdin_read_fd, __FILE__, __LINE__);
                safe_close(stdin_write_fd, __FILE__, __LINE__);
            } else {
                if (safe_dup2(slave, STDIN_FILENO) == -1) exit_on_err("dup2() failed for fd number 0");
            }
```

### Reading output and detecting death **[source-derived]**

The monitor reads the PTY master into the VT parser's write buffer. `kitty/child-monitor.c:1336-1356` (contiguous):

```c
static bool
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

Two distinct death-detection outcomes live in this one function: an `EIO` read (PTY EOF, last writer gone) commits 0 bytes and **returns `false`** (lines 1346-1350), and a clean zero-length read also **returns `false`** via `len != 0` (line 1355). Either way, `read_bytes` returning `false` signals the child is gone.

The monitor's I/O loop consumes both the signal path and the PTY path. `kitty/child-monitor.c:1516-1537` (contiguous):

```c
            if (children_fds[1].revents && POLLIN) {
                SignalSet ss = {0};
                data_received = true;
                read_signals(children_fds[1].fd, handle_signal, &ss);
                if (ss.kill_signal || ss.reload_config) {
                    children_mutex(lock);
                    if (ss.kill_signal) kill_signal_received = true;
                    if (ss.reload_config) reload_config_signal_received = true;
                    children_mutex(unlock);
                }
                if (ss.child_died) reap_children(self, OPT(close_on_child_death));
            }
            for (i = 0; i < self->count; i++) {
                if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP)) {
                    data_received = true;
                    has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen);
                    if (!has_more) {
                        // child is dead
                        children_mutex(lock);
                        children[i].needs_removal = true;
                        children_mutex(unlock);
                    }
```

Note `reap_children(self, OPT(close_on_child_death))` at line 1526: by **default** `close_on_child_death` is `no`, so the reap does **not** by itself remove the window; window removal happens because `read_bytes` returned `false` and set `children[i].needs_removal = true` at line 1535 (the PTY-EOF path). This is the causal basis for the default-teardown nuance discussed later.

### Parsing, buffering, and rendering **[source-derived]**

`read_bytes` only commits raw bytes into the VT parser's write buffer. The actual parsing into the `Screen` happens in `do_parse()` [`kitty/child-monitor.c:438`], invoked from the parse pass [`kitty/child-monitor.c:521,530`]; the filled `Screen` cells are then uploaded to the GPU in `prepare_to_render_os_window()` [`kitty/child-monitor.c:705`] via `send_cell_data_to_gpu()` [`kitty/child-monitor.c:714`] and drawn. The net effect: child stdout → PTY master → VT parser → `Screen` grid → GPU render into the kitty window (answered concretely in Q9).

### Rationale

The flow is a producer/consumer split across a PTY: `Child.fork()` wires the child's stdio to the PTY slave and hands the master + PID to the `ChildMonitor`; the monitor thread multiplexes a `signalfd` (for `SIGCHLD`) and the PTY master (for output and EOF). Output is transformed byte-stream → parsed grid → pixels; termination is delivered as a signal and finalized by reaping. Whether a persistent message appears depends entirely on the two secondary mechanisms (hold, shell integration) covered in Q3/Q5/Q8.

## Q2 — Kitty's own exit code when the child exits 0

### Command and raw output **[observed]**

```
$ cat /tmp/kwork/evidence/q2_exit.txt
run 1: kitty_exit=0
run 2: kitty_exit=0
run 3: kitty_exit=0
run 4: kitty_exit=0
run 5: kitty_exit=0
child exit 5  -> kitty_exit=0
child exit 42 -> kitty_exit=0
```

Each `run N` executed `./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'; echo "kitty_exit=$?"`. The last two lines re-ran with `exit 5` and `exit 42` respectively.

### Kitty's own exit code is `0` and is **stable** and **independent of the child's status**

Across 5 identical runs the value is `0` every time (distribution `{0: 5}`), satisfying the ≥2-run stability requirement. Crucially, a child exiting `5` or `42` **still** yields kitty exit `0`: kitty does not propagate the child's status as its own.

### Code citation **[source-derived]**

`kitty/main.py:524-531` (contiguous) — kitty's own process exit is governed by `main()`: a normal return exits `0`, and only an internal, unhandled exception raises `SystemExit(1)`:

```python
def main() -> None:
    try:
        _main()
    except Exception:
        import traceback
        tb = traceback.format_exc()
        log_error(tb)
        raise SystemExit(1)
```

### Rationale

`main()` returns normally after the GUI event loop ends (all windows closed), so the process exit code is `0`. The child's exit status lives in a completely different place — it is reaped by the monitor thread's `wait4` and, for the message path, transported by OSC 133 — but it never becomes kitty's own return code. Hence the observed independence: kitty exits `0` regardless of whether the child exited `0`, `5`, or `42`.

## Q3 — The full completion message

There is **no single "program completed" message in the default direct-launch case** — by default the window just closes (see Nuance 1). A persistent completion message appears only via **two distinct, unrelated mechanisms**: the **hold** entry point (a fixed banner) and **shell integration** (a status-bearing notification). Both are reported here, verbatim.

### Q3a — The `kitty +hold` / hold-kitten banner

**Terminology (important):** two different things are colloquially called "hold":

- The **`--hold` flag** to `kitty` (e.g. `kitty --hold sh -c '…'`) keeps the window open by running the command **inside an interactive shell** (`kitten run-shell … --env=KITTY_HOLD=1`); it shows **no banner**.
- The **`kitty +hold` / `kitten __hold_till_enter__`** entry point shows the green banner **`Press Enter or Esc to exit`**.

#### Command and raw output — the `+hold` banner **[observed]**

Captured through a PTY (the banner is written to the terminal, so a PTY is required to see it):

```
$ cat /tmp/kwork/evidence/q3a_hold.txt
### kitty +hold  (public entry) run #1 — full capture (Python repr):
b'hello\r\nworld\r\n\x1b[?s\x1b[*x\x1b[4l\x1b[?1l\x1b[?5l\x1b[?2004l\x1b[?1004l\x1b[?1000l\x1b[?1002l\x1b[?1003l\x1b[?1005l\x1b[?1006l\x1b[?8h\x1b[?7h\x1b[?25h\x1b[>29u\x1b[?25l\r\n\x1b[1;32mPress Enter or Esc to exit\x1b[m'

### banner segment isolated (run #1), Python repr then cat -v:
b'\x1b[1;32mPress Enter or Esc to exit\x1b[m'
^[[1;32mPress Enter or Esc to exit^[[m

### kitty +hold run #2 banner segment (stability check), cat -v:
^[[1;32mPress Enter or Esc to exit^[[m

### kitten __hold_till_enter__ (NONCANONICAL corroboration) banner segment, cat -v:
^[[1;32mPress Enter or Esc to exit^[[m

### byte-identical banner across the two +hold runs?
run1==run2 banner: True | banner bytes: b'\x1b[1;32mPress Enter or Esc to exit\x1b[m'
```

The **exact** message text is `Press Enter or Esc to exit`, wrapped in SGR `\x1b[1;32m` (bold, green) and reset `\x1b[m`. It is **byte-identical across two runs** (stability requirement met). The hidden `kitten __hold_till_enter__` helper produces the same bytes and is shown only as **[non-canonical corroboration]** — the canonical value comes from the public `kitty +hold` entry point.

#### Command and raw output — the `--hold` flag shows **no** banner **[observed]**

```
$ cat /tmp/kwork/evidence/q3_hold_flag_env.txt
### any process on the host whose environ contains KITTY_HOLD=1 (the run-shell hold child) ###
pid=148814 cmdline=[/bin/bash --posix ]
    KITTY_SHELL_INTEGRATION=enabled
    KITTY_HOLD=1
### the actual interactive shell descendants (kitten run-shell) ###
```

A screenshot of the `kitty --hold` window (saved to `/tmp/kwork/evidence/q3_hold_flag.png` and inspected during the investigation) shows the two output lines `hello` and `world` in white monospace on black, and **no green `Press Enter or Esc to exit` banner anywhere** — confirming that the `--hold` flag holds via an interactive shell, not via the banner path.

#### Code citations **[source-derived]**

The banner literal, `tools/tui/hold.go:16-42` (contiguous; banner at line 26):

```go
func HoldTillEnter(start_with_newline bool) {
	lp, err := loop.New(loop.NoAlternateScreen, loop.NoRestoreColors, loop.NoMouseTracking)
	if err != nil {
		return
	}
	lp.OnInitialize = func() (string, error) {
		lp.SetCursorVisible(false)
		if start_with_newline {
			lp.QueueWriteString("\r\n")
		}
		lp.QueueWriteString("\x1b[1;32mPress Enter or Esc to exit\x1b[m")
		return "", nil
	}
	lp.OnFinalize = func() string {
		lp.SetCursorVisible(true)
		return ""
	}

	lp.OnKeyEvent = func(event *loop.KeyEvent) error {
		if event.MatchesPressOrRepeat("enter") || event.MatchesPressOrRepeat("kp_enter") || event.MatchesPressOrRepeat("esc") || event.MatchesPressOrRepeat("ctrl+c") || event.MatchesPressOrRepeat("ctrl+d") {
			event.Handled = true
			lp.Quit(0)
		}
		return nil
	}
	lp.Run()
}
```

The `+hold` dispatch, `tools/cmd/tool/main.go:87-93` (contiguous):

```go
	// __hold_till_enter__
	root.AddSubCommand(&cli.Command{
		Name:            "__hold_till_enter__",
		Hidden:          true,
		OnlyArgsAllowed: true,
		Run: func(cmd *cli.Command, args []string) (rc int, err error) {
			tui.ExecAndHoldTillEnter(args)
```

The standalone Python hold entry, `kitty/utils.py:1031-1035` (contiguous):

```python
def hold_till_enter() -> None:
    import subprocess

    from .constants import kitten_exe
    subprocess.Popen([kitten_exe(), '__hold_till_enter__']).wait()
```

The `+hold` argv dispatch, `kitty/entry_points.py:27-30` (contiguous):

```python
def hold(args: List[str]) -> None:
    from kitty.constants import kitten_exe
    args = ['kitten', '__hold_till_enter__'] + args[1:]
    os.execvp(kitten_exe(), args)
```

The `--hold` **flag** path (interactive shell, `KITTY_HOLD=1`, **no** banner), `kitty/utils.py:1192-1202` (contiguous):

```python
def cmdline_for_hold(cmd: Sequence[str] = (), opts: Optional['Options'] = None) -> List[str]:
    if opts is None:
        with suppress(RuntimeError):
            opts = get_options()
    if opts is None:
        from .options.types import defaults
        opts = defaults
    ksi = ' '.join(opts.shell_integration)
    import shlex
    shell = shlex.join(resolved_shell(opts))
    return [kitten_exe(), 'run-shell', f'--shell={shell}', f'--shell-integration={ksi}', '--env=KITTY_HOLD=1'] + list(cmd)
```

#### Rationale

The `+hold` banner is a fixed string with no exit status — it exists to let the user read the output before the window disappears. The `--hold` flag is a different feature: `Child.fork()` rewrites the command line via `cmdline_for_hold()` (see Q1) so the program runs under an interactive shell that stays alive, which is why its window shows a live shell (and `KITTY_HOLD=1`) rather than a banner.

### Q3b — The shell-integration completion notification

When shell integration is active, the shell reports each command's status via OSC 133 `D;<status>` (see Q8). Kitty turns that into a desktop notification whose **body** is the status-bearing message.

#### Command and raw output — the exact notification body **[observed]**

A **real interactive `bash` running inside kitty** was driven by synthesizing real keystrokes with the X11 `XTEST` extension (the canonical user-input path), under `dbus-run-session` so the notification is delivered over D-Bus, with `notify_on_cmd_finish` set to fire:

```
$ cat /tmp/kwork/evidence/q3b_notify_body.txt
method call time=1783966736.493997 sender=:1.2 -> destination=org.freedesktop.Notifications serial=3 path=/org/freedesktop/Notifications; interface=org.freedesktop.Notifications; member=Notify
   string "kitty"
   uint32 0
   string "/tmp/blitzy/kitty/blitzy-4da995f7-9d87-4535-892b-6e7ceff8c1ad_2bb4d2/logo/kitty.png"
   string "kitty"
   string "Command echo hello finished with status: 0.
Click to focus."
   array [
      string "default"
      string "Click to see changes"
   ]
   array [
      dict entry(
         string "urgency"
         variant             byte 1
      )
   ]
   int32 -1
```

The **exact body text** (the fifth argument to `Notify`) is:

```
Command echo hello finished with status: 0.
Click to focus.
```

i.e. `Command echo hello finished with status: 0.\nClick to focus.` with a literal newline between the two sentences. Stability note (per finding on run-to-run identity): across repeated runs the **body is byte-identical**, but the D-Bus envelope fields (`sender`, `serial`, `time=`) legitimately vary and are therefore not claimed to be stable.

#### Command and raw output — the enable/disable matrix **[observed]**

The same XTEST-driven real-bash-through-kitty run was captured three ways; `--dump-bytes` records the OSC 133 markers kitty's parser actually received, and `dbus-monitor` counts `Notify` calls:

```
condition                                          OSC133 C   OSC133 D;0   Notify calls
kitty -o "notify_on_cmd_finish always 0"  (int on)      2          2            1
kitty -o "shell_integration disabled" (+notify)         0          0            0
kitty  (defaults: notify_on_cmd_finish=never)           2          2            0
```

Reading the matrix:
- With shell integration **on** and notification **enabled**, the shell emits OSC 133 `C`/`D;0` and kitty fires exactly **one** notification.
- With shell integration **disabled at the kitty level**, **no** OSC 133 markers are produced and **no** notification fires (negative control).
- With shell integration **on** but `notify_on_cmd_finish` at its **default (`never`)**, the OSC 133 markers still arrive but **no** notification fires — proving the gate is the `when != 'never'` condition, not a duration threshold (see Nuance 2).

#### Code citation **[source-derived]**

The body string is built in `Window.handle_cmd_end()` (full method quoted under Q5). The single line that formats the message, `kitty/window.py:1428-1429` (contiguous):

```python
            s = self.last_cmd_cmdline.replace('\\\n', ' ')
            cmd.body = f'Command {s} finished with status: {exit_status}.\nClick to focus.'
```

#### Rationale

For `echo hello`, `self.last_cmd_cmdline` is `echo hello` and `exit_status` is `'0'`, so the f-string yields `Command echo hello finished with status: 0.\nClick to focus.` — exactly the observed D-Bus body. The message only appears when `notify_on_cmd_finish != 'never'`, which is why the default configuration shows nothing.

## Q4 — The component that tracks the child

### Answer: the `ChildMonitor` **[source-derived]**

The child is tracked across its lifetime by the **`ChildMonitor`** — a C object (implemented in `kitty/child-monitor.c`) created and driven from Python (`kitty/boss.py`). It owns, per child: the **window id**, the **PID**, the **PTY master file descriptor**, and the window's **`Screen`**. It runs the monitor thread that polls the PTY master and the `signalfd`, reads output, handles `SIGCHLD`, reaps the child, and notifies Python when the child dies.

### Code citations **[source-derived]**

The `Boss` creates the single `ChildMonitor` and wires its death callback, `kitty/boss.py:370-374` (contiguous):

```python
        self.child_monitor = ChildMonitor(
            self.on_child_death,
            DumpCommands(args) if args.dump_commands or args.dump_bytes else None,
            talk_fd, listen_fd,
        )
```

Each new child is registered with its PID, PTY master fd, and `Screen`, `kitty/boss.py:585-587` (contiguous):

```python
    def add_child(self, window: Window) -> None:
        assert window.child.pid is not None and window.child.child_fd is not None
        self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)
```

The C side records exactly those fields, `kitty/child-monitor.c:305-306` (contiguous):

```c
add_child(ChildMonitor *self, PyObject *args) {
#define add_child_doc "add_child(id, pid, fd, screen) -> Add a child."
```

### Rationale

`add_child(id, pid, fd, screen)` binds together everything needed to track the child: the `id` maps back to the `Window`, the `pid` is what `wait4` reaps (Q7), the `fd` is the PTY master read in `read_bytes` (Q1/Q9), and the `screen` is where parsed output lands. The monitor thread owns these for the child's whole life, which is why the `ChildMonitor` is *the* tracking component. On death it invokes `on_child_death(window_id)` — note it passes **only the window id, not an exit status** (see Distinction B / Q7).

## Q5 — The function that turns the exit status into the message

### Answer: `Window.handle_cmd_end()` **[observed + source-derived]**

The single function that converts the **shell-reported command status** into the user-facing message is **`Window.handle_cmd_end()`** at `kitty/window.py:1408`. It is reached from `cmd_output_marking()` when the OSC 133 `D` sub-command is decoded (Q8), and it builds the `Command … finished with status: …` notification body.

### Code citation — full method, `kitty/window.py:1408-1451` (contiguous) **[source-derived]**

```python
    def handle_cmd_end(self, exit_status: str = '') -> None:
        if self.last_cmd_output_start_time == 0.:
            return
        self.last_cmd_output_start_time = 0.
        try:
            self.last_cmd_exit_status = int(exit_status)
        except Exception:
            self.last_cmd_exit_status = 0
        end_time = monotonic()
        last_cmd_output_duration = end_time - self.last_cmd_output_start_time

        self.call_watchers(self.watchers.on_cmd_startstop, {
            "is_start": False, "time": end_time, 'cmdline': self.last_cmd_cmdline, 'exit_status': self.last_cmd_exit_status})

        opts = get_options()
        when, duration, action, notify_cmdline = opts.notify_on_cmd_finish

        if last_cmd_output_duration >= duration and when != 'never':
            cmd = NotificationCommand()
            cmd.title = 'kitty'
            s = self.last_cmd_cmdline.replace('\\\n', ' ')
            cmd.body = f'Command {s} finished with status: {exit_status}.\nClick to focus.'
            cmd.actions = 'focus'
            cmd.only_when = OnlyWhen(when)
            if action == 'notify':
                notify_with_command(cmd, self.id)
            elif action == 'bell':
                class Bell(NotifyImplementation):
                    def __init__(self, window_id: int):
                        self.window_id = window_id
                    def __call__(self, title: str, body: str, identifier: str, urgency: Urgency = Urgency.Normal) -> None:
                        w = get_boss().window_id_map.get(self.window_id)
                        if w:
                            w.screen.bell()
                notify_with_command(cmd, self.id, notify_implementation=Bell(self.id))
            elif action == 'command':
                class Run(NotifyImplementation):
                    def __init__(self, last_cmd_cmdline: str):
                        self.last_cmd_cmdline = last_cmd_cmdline
                    def __call__(self, title: str, body: str, identifier: str, urgency: Urgency = Urgency.Normal) -> None:
                        open_cmd([x.replace('%c', self.last_cmd_cmdline).replace('%s', exit_status) for x in notify_cmdline])
                notify_with_command(cmd, self.id, notify_implementation=Run(self.last_cmd_cmdline))
            else:
                raise ValueError(f'Unknown action in option `notify_on_cmd_finish`: {action}')
```

It is invoked from `cmd_output_marking()`, `kitty/window.py:1453-1461` (contiguous):

```python
    def cmd_output_marking(self, is_start: Optional[bool], cmdline: str = '') -> None:
        if is_start:
            start_time = monotonic()
            self.last_cmd_output_start_time = start_time
            cmdline = decode_cmdline(cmdline) if cmdline else ''
            self.last_cmd_cmdline = cmdline
            self.call_watchers(self.watchers.on_cmd_startstop, {"is_start": True, "time": start_time, 'cmdline': cmdline, 'exit_status': 0})
        else:
            self.handle_cmd_end(cmdline)
```

### Observed recorded state **[observed]**

Driving a real command through kitty with `notify_on_cmd_finish` action `command` (which echoes the recorded values) shows `handle_cmd_end` recorded the status and cmdline:

```
$ cat /tmp/kwork/evidence/q5_recorded_state.txt
NOTIFY_FIRED recorded_exit_status=[0] recorded_cmdline=[echo hello]
```

### An apparent pre-existing behavior in the duration gate **[source-derived; documented, not changed]**

The gate at line 1425 is `if last_cmd_output_duration >= duration and when != 'never':`. However, `last_cmd_output_duration` is computed at line 1417 as `end_time - self.last_cmd_output_start_time`, and `self.last_cmd_output_start_time` was **already set to `0.` at line 1411** — *before* the subtraction. Consequently `last_cmd_output_duration == end_time`, i.e. the raw `monotonic()` clock value (thousands of seconds since boot), which is effectively always `>= duration` (default `5.0`). The gate therefore reduces in practice to **`when != 'never'`**: the `duration` threshold has no practical effect.

This is corroborated empirically by the Q3b matrix: with `notify_on_cmd_finish` at its default (`never`) no notification fires, but with `when='always'` a notification fires immediately for `echo hello` (which has essentially zero real output duration) — the duration threshold never gates it out. This is reported as an **apparent pre-existing behavior/defect in the investigated source**; per scope, **no source change is made** — only documentation.

### Rationale

`handle_cmd_end(exit_status)` is the sole place where the shell-reported status string becomes the message: it stores `last_cmd_exit_status`, and, when the gate passes, formats `cmd.body` (line 1429) with the command line and the status. `cmd_output_marking(is_start=False, cmdline=…)` is its only caller in the OSC 133 `D` path, forwarding the decoded status string.


## Q6 — The OS signal kitty listens for

### Answer: `SIGCHLD` (signal 17) **[observed]**

Kitty learns a child terminated via **`SIGCHLD`**, delivered through a **`signalfd`** (Linux) rather than an async signal handler. This was observed directly.

### Raw output — signalfd setup and the SIGCHLD siginfo **[observed]**

During startup, kitty blocks its handled signals and creates a `signalfd` whose mask includes `CHLD`:

```
$ grep -m1 'rt_sigprocmask(SIG_BLOCK, \[' /tmp/kwork/evidence/kitty.strace | grep CHLD
146254 rt_sigprocmask(SIG_BLOCK, [HUP INT USR1 USR2 TERM CHLD], NULL, 8) = 0
$ grep -m1 'signalfd4' /tmp/kwork/evidence/kitty.strace
146254 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
```

When the child exits, the monitor thread reads one `struct signalfd_siginfo` (128 bytes) from fd 7. The `strace -e read=all` payload:

```
247153:146320 read(7, "\21\0\0\0\0\0\0\0\1\0\0\0\221;\2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128
 | 00000  11 00 00 00 00 00 00 00  01 00 00 00 91 3b 02 00  .............;.. |
 | 00010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................ |
 | 00020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................ |
```

Decoding those exact bytes (the first four fields of `struct signalfd_siginfo`, little-endian `<IiiI`) with a reproducible script:

```
$ cat /tmp/kwork/decode_siginfo.py
import struct
# First 16 bytes of the struct signalfd_siginfo read from kitty's signal fd (fd 7),
# captured verbatim by: strace -e read=all ... (the "read(7, ...) = 128" payload).
first16 = bytes([0x11,0,0,0, 0,0,0,0, 1,0,0,0, 0x91,0x3b,0x02,0])
signo, errno_, code, pid = struct.unpack('<IiiI', first16)
names={17:'SIGCHLD'}; codes={1:'CLD_EXITED'}
print(f"ssi_signo = {signo}  -> {names.get(signo,'?')}")
print(f"ssi_errno = {errno_}")
print(f"ssi_code  = {code}   -> {codes.get(code,'?')}")
print(f"ssi_pid   = {pid}   (the child process that exited)")

$ python3.11 /tmp/kwork/decode_siginfo.py
ssi_signo = 17  -> SIGCHLD
ssi_errno = 0
ssi_code  = 1   -> CLD_EXITED
ssi_pid   = 146321   (the child process that exited)
```

`ssi_signo=17` is `SIGCHLD`; `ssi_code=1` is `CLD_EXITED`; `ssi_pid=146321` is exactly the child PID that `+++ exited with 0 +++` in Q1. (The `signalfd`/`sigprocmask` calls run on the setup thread TID `146254`; the siginfo read and reap run on the monitor thread TID `146320`.)

### Code citations **[source-derived]**

`SIGCHLD` is in kitty's handled-signal set, `kitty/child-monitor.c:121`:

```c
#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0
```

The signals are blocked and a `signalfd` is created, `kitty/loop-utils.c:34-44` (contiguous):

```c
static bool
init_signal_handlers(LoopData *ld) {
    ld->signal_read_fd = -1;
    sigemptyset(&ld->signals);
    for (size_t i = 0; i < ld->num_handled_signals; i++) sigaddset(&ld->signals, ld->handled_signals[i]);
#ifdef HAS_SIGNAL_FD
    if (ld->num_handled_signals) {
        if (sigprocmask(SIG_BLOCK, &ld->signals, NULL) == -1) return false;
        ld->signal_read_fd = signalfd(-1, &ld->signals, SFD_NONBLOCK | SFD_CLOEXEC);
        if (ld->signal_read_fd == -1) return false;
    }
```

The siginfo is read and dispatched, `kitty/loop-utils.c:131-159` (contiguous):

```c
read_signals(int fd, handle_signal_func callback, void *data) {
#ifdef HAS_SIGNAL_FD
    static struct signalfd_siginfo fdsi[32];
    siginfo_t si;
    while (true) {
        ssize_t s = read(fd, &fdsi, sizeof(fdsi));
        if (s < 0) {
            if (errno == EINTR) continue;
            if (errno == EAGAIN) break;
            log_error("Call to read() from read_signals() failed with error: %s", strerror(errno));
            break;
        }
        if (s == 0) break;
        size_t num_signals = s / sizeof(struct signalfd_siginfo);
        if (num_signals == 0 || num_signals * sizeof(struct signalfd_siginfo) != (size_t)s) {
            log_error("Incomplete signal read from signalfd");
            break;
        }
        for (size_t i = 0; i < num_signals; i++) {
            si.si_signo = fdsi[i].ssi_signo;
            si.si_code = fdsi[i].ssi_code;
            si.si_pid = fdsi[i].ssi_pid;
            si.si_uid = fdsi[i].ssi_uid;
            si.si_addr = (void*)(uintptr_t)fdsi[i].ssi_addr;
            si.si_status = fdsi[i].ssi_status;
            si.si_value.sival_int = fdsi[i].ssi_int;
            if (!callback(&si, data)) break;
        }
    }
```

The callback flags a child death on `SIGCHLD`, `kitty/child-monitor.c:1361-1383` (contiguous):

```c
static bool
handle_signal(const siginfo_t *siginfo, void *data) {
    SignalSet *ss = data;
    switch(siginfo->si_signo) {
        case SIGINT:
        case SIGTERM:
        case SIGHUP:
            ss->kill_signal = true;
            break;
        case SIGCHLD:
            ss->child_died = true;
            break;
        case SIGUSR1:
            ss->reload_config = true;
            break;
        case SIGUSR2:
            log_error("Received SIGUSR2: %d\n", siginfo->si_value.sival_int);
            break;
        default:
            break;
    }
    return true;
}
```

### Rationale

Kitty converts asynchronous signal delivery into a synchronous, pollable fd: it blocks `SIGCHLD` (among others) process-wide and drains a `signalfd` inside its monitor loop. When a `SIGCHLD` siginfo arrives, `handle_signal` sets `ss->child_died = true`, which triggers the reap in Q7. The observed `ssi_signo=17`/`CLD_EXITED` for the exact child PID confirms `SIGCHLD` is the signal kitty listens for.

## Q7 — The system call to retrieve the child's exit status

### Answer: `wait4(-1, &status, WNOHANG)` — the `waitpid` family **[observed + source-derived]**

Kitty reaps the child with **`waitpid(-1, &status, WNOHANG)`**, which the C library implements via the **`wait4`** syscall (as strace shows). It is called in a loop until no more children are reapable.

### Raw output — the reap **[observed]**

From the Q1 death window:

```
146320 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 146321
146320 wait4(-1, 0x7c82346c2e50, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

The first `wait4` reaps child `146321` with `WIFEXITED && WEXITSTATUS==0`; the second returns `ECHILD` (nothing left to reap), ending the loop.

### Code citations **[source-derived]**

The reap loop, `kitty/child-monitor.c:1412-1426` (contiguous):

```c
static void
reap_children(ChildMonitor *self, bool enable_close_on_child_death) {
    int status;
    pid_t pid;
    (void)self;
    while(true) {
        pid = waitpid(-1, &status, WNOHANG);
        if (pid == -1) {
            if (errno != EINTR) break;
        } else if (pid > 0) {
            if (enable_close_on_child_death) mark_child_for_removal(self, pid);
            mark_monitored_pids(pid, status);
        } else break;
    }
}
```

Two important facts about what happens to the reaped `status`:

1. `mark_child_for_removal(self, pid)` runs **only if `enable_close_on_child_death`** is true — i.e. only when `close_on_child_death=yes`. With the **default** (`no`), the reap does **not** remove the window (window teardown then comes from the PTY-EOF path — see Nuance 1).
2. `mark_monitored_pids(pid, status)` records the status **only for PIDs explicitly registered in the `monitored_pids[]` array** — it is a **no-op for the primary window child**, which is not in that array. `kitty/child-monitor.c:1397-1409` (contiguous):

```c
static void
mark_monitored_pids(pid_t pid, int status) {
    children_mutex(lock);
    for (ssize_t i = monitored_pids_count - 1; i >= 0; i--) {
        if (pid == monitored_pids[i]) {
            if (reaped_pids_count < arraysz(reaped_pids)) {
                reaped_pids[reaped_pids_count].status = status;
                reaped_pids[reaped_pids_count++].pid = pid;
            }
            remove_i_from_array(monitored_pids, (size_t)i, monitored_pids_count);
        }
    }
    children_mutex(unlock);
}
```

Consequently, the direct-child death callback into Python carries **no exit status** — it receives only a window id, `kitty/boss.py:881`:

```python
    def on_child_death(self, window_id: int) -> None:
```

The status-bearing callback `on_monitored_pid_death(self, pid, exit_status)` [`kitty/boss.py:2725`] exists **only** for explicitly monitored background PIDs, not for the primary child.

### Rationale

`wait4`/`waitpid` is how kitty retrieves the child's raw wait-status and clears the zombie. But that status is used for reaping bookkeeping (and, for explicitly monitored PIDs, `on_monitored_pid_death`); it is **not** propagated as an exit status into the terminal window's message path. That is the technical reason the user-facing "finished with status" message comes from the **shell-integration** transport (Q8), not from the `wait4` reap. This is Distinction B.

## Q8 — How the exit status travels from the shell to the message

### Answer: the OSC 133 `D;<code>` escape sequence **[observed + source-derived]**

The **shell-reported command status** travels from the integrated shell to kitty as the **OSC 133 `D;<code>`** ("command finished") escape sequence, in-band on the PTY. Kitty's VT parser routes OSC code 133 to `shell_prompt_marking()`, whose `D` case extracts the status string and calls back into Python, ending at `Window.handle_cmd_end()` (Q5).

### Raw output — the byte-exact sequence and the on-wire timeline **[observed]**

Byte-exact `D;0` as kitty's parser received it (real bash in kitty, XTEST-driven):

```
$ cat /tmp/kwork/evidence/q8_byte_exact.txt
repr: b'\x1b]133;D;0\x07\x1b'
hex : 1b 5d 31 33 33 3b 44 3b 30 07 1b
cat -v form: ^[]133;D;0^G
```

i.e. `ESC ] 1 3 3 ; D ; 0 BEL`. The ordered markers around the `echo hello` command:

```
$ cat /tmp/kwork/evidence/q8_timeline.txt
# ordered OSC 133 markers kitty's parser received (echo hello command), cat -v:
^[]133;D;0^G
^[]133;A^G
^[]133;C;cmdline=echo\ hello^G
^[]133;D;0^G
^[]133;A^G
^[]133;C;cmdline=exit^G
```

Reading it: a `D;0` closes the previous prompt, `A` starts a new prompt, `C;cmdline=echo\ hello` marks command-output start, then `D;0` reports the command finished with status `0`. With shell integration **disabled at the kitty level**, **zero** `133` markers appear on the wire (the negative control from Q3b: `dump_disabled_notify.bin` has `C=0, D;0=0`).

### Code citations **[source-derived]**

The integrated shells emit the sequence. bash, `shell-integration/bash/kitty.bash:208` and `:239`:

```bash
                builtin printf "\e]133;C;cmdline=%q\a" "$last_cmd"
```
```bash
        _ksi_prompt[ps1]+="\[\e]133;D;\$?\a\e]133;A\a\]"
```

zsh, `shell-integration/zsh/kitty-integration:145` (with status) and `:149` (no status):

```zsh
                    builtin print -nu $_ksi_fd '\e]133;D;'$cmd_status'\a'
```
```zsh
                    builtin print -nu $_ksi_fd '\e]133;D\a'
```

fish, `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:96`:

```fish
                echo -en "\e]133;D;$status\a"
```

The VT parser routes OSC code 133 to `shell_prompt_marking()`, `kitty/vt-parser.c:536-546` (contiguous; the `#ifdef DUMP_COMMANDS` reporting block is shown in place so the quote is not spliced):

```c
        case 133:
#ifdef DUMP_COMMANDS
            START_DISPATCH
            REPORT_OSC2(shell_prompt_marking, code, mv);
            END_DISPATCH_WITHOUT_BREAK
#endif
            if (limit > i) {
                buf[limit] = 0; // safe to do as we have 8 extra bytes after PARSER_BUF_SZ
                shell_prompt_marking(self->screen, (char*)buf + i);
            }
            break;
```

`shell_prompt_marking()` handles `A`/`C`/`D`; the `D` case extracts the exit status and calls back into Python, `kitty/screen.c:2327-2356` (contiguous):

```c
void
shell_prompt_marking(Screen *self, char *buf) {
    if (self->cursor->y < self->lines) {
        char ch = buf[0];
        switch (ch) {
            case 'A': {
                PromptKind pk = PROMPT_START;
                self->prompt_settings.redraws_prompts_at_all = 1;
                self->prompt_settings.uses_special_keys_for_cursor_movement = 0;
                parse_prompt_mark(self, buf+1, &pk);
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = pk;
                if (pk == PROMPT_START) CALLBACK("cmd_output_marking", "O", Py_False);
            } break;
            case 'C': {
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = OUTPUT_START;
                const char *cmdline = "";
                if (strstr(buf + 1, ";cmdline") == buf + 1) {
                    cmdline = buf + 2;
                }
                RAII_PyObject(c, PyUnicode_DecodeUTF8(cmdline, strlen(cmdline), "replace"));
                if (c) { CALLBACK("cmd_output_marking", "OO", Py_True, c); }
                else PyErr_Print();
            } break;
            case 'D': {
                const char *exit_status = buf[1] == ';' ? buf + 2 : "";
                CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);
            } break;
        }
    }
}
```

The `D` case (lines 2350-2352) computes `exit_status` from the bytes after `D;` and invokes the Python callback `cmd_output_marking(None, exit_status)`, which forwards to `handle_cmd_end(exit_status)` (Q5). For `D;0`, `exit_status` is `"0"`.

### The protocol is a de-facto cross-vendor convention **[source-derived, corroborated by external references]**

OSC 133 is not a formally standardized escape sequence; it is a **de-facto cross-vendor convention** originating with FinalTerm and popularized by iTerm2. Its sequences <cite index="3-1">mark up a shell's output with semantic information about where the prompt begins</cite>, where the command begins, and where output begins and ends. The command-finished marker is defined by VS Code's documentation as <cite index="6-7">Based on FinalTerm's `OSC 133 ; D [; <ExitCode>] ST`</cite>, with the exit code optional. When present, terminals <cite index="9-2">treat 0 as "success" and anything else as an error.</cite> The markers are <cite index="7-3">now adopted by iTerm2, VS Code, Ghostty, and others</cite>, and kitty is explicitly among the implementers. Its non-standard status is explicit: <cite index="8-20">The OSC command numbers 133 and 1337 are not registered with any standards body.</cite>

### Rationale

Unlike the direct-child path (Q6/Q7), where the OS carries only a raw wait-status with no escape sequence, the shell-integration path is what carries a human-meaningful **command** status: the shell appends `\e]133;D;$?\a` (bash), `\e]133;D;$cmd_status\a` (zsh), or `\e]133;D;$status\a` (fish) after each command; kitty parses it (`vt-parser.c` → `screen.c`) and delivers the status string to `handle_cmd_end()`. This is why questions 5–8 concern the OSC 133 transport, while 6–7 concern the OS `SIGCHLD`/`wait4` mechanism.

## Q9 — Where the child's printed output appears

### Answer: in kitty's on-screen terminal grid (the kitty window) **[observed]**

The child's stdout is read from the **PTY master**, parsed, written into the window's `Screen` grid, and rendered on screen. It does **not** appear on kitty's own stdout/stderr.

### Raw output — the child's bytes reach kitty via the PTY master **[observed]**

From the Q1 strace (monitor thread reading the PTY master fd 8):

```
146320 read(8, "hello\r\nworld\r\n", 1048576) = 14
```

### Raw output — the bytes land in the `Screen` grid **[observed]**

Feeding `hello\r\nworld\r\n` through kitty's **real** VT parser and `Screen` (via `kitty.fast_data_types.Screen` and the test harness `parse_bytes`, run under `kitty +launch`) fills the grid exactly:

```
$ cat /tmp/kwork/evidence/q9_screen_dump.txt
grid line 0: 'hello'
grid line 1: 'world'
grid line 2: ''
```

A screenshot of the live kitty window (saved to `/tmp/kwork/evidence/q9_grid.png` and inspected during the investigation) shows two lines of light-gray monospace text on a black background — `hello` on the first line and `world` on the second — left-aligned at the top of the terminal grid, confirming the rendered result.

### Raw output — the child's text is NOT on kitty's own stdout/stderr **[observed]**

```
$ cat /tmp/kwork/evidence/q9_negative.txt
kitty_exit=0
--- kitty own stdout: byte count ---
0
--- kitty own stdout: contents ---
--- kitty own stderr: contents ---
[0.195] Failed to open systemd user bus with error: Connection refused
--- count of child lines (hello|world) in kitty own stdout+stderr ---
/tmp/kwork/q9_own_stdout.txt:0
/tmp/kwork/q9_own_stderr.txt:0
```

Kitty's own stdout is **empty (0 bytes)**; its stderr contains only a benign, unrelated systemd-bus warning; and grepping both for `hello`/`world` yields **0** matches. The child's output is thus confined to the terminal grid.

### Code citation **[source-derived]**

The PTY master read is `read_bytes()` [`kitty/child-monitor.c:1345`] (full function quoted under Q1), which commits the bytes into the VT parser write buffer; `do_parse()` [`kitty/child-monitor.c:438`] then feeds the parser into the `Screen`, and `prepare_to_render_os_window()`/`send_cell_data_to_gpu()` [`kitty/child-monitor.c:705,714`] upload the grid cells to the GPU for drawing.

### Rationale

`echo hello; echo world` writes to the child's stdout, which is the PTY **slave** (wired by `dup2` in `spawn`, Q1). Kitty reads the **master** end, so the bytes enter kitty's parser and become grid cells rather than propagating to kitty's own standard streams. The empty own-stdout, the `Screen` dump, and the screenshot together confirm the output lands in the kitty window's terminal grid.


## Two distinctions that resolve the question

### Distinction A — There are two different "completion" messages

The prompt's question 3 ("the full message shown indicating the program completed") only has a well-defined answer once two unrelated messages are separated:

| Message | When it appears | Exact text | Carries status? |
|---|---|---|---|
| `+hold` / hold-kitten banner | `kitty +hold …` (or `kitten __hold_till_enter__`) | `Press Enter or Esc to exit` (SGR bold-green) | No |
| Shell-integration notification | interactive shell with integration on **and** `notify_on_cmd_finish != never` | `Command <cmd> finished with status: <n>.\nClick to focus.` | Yes (`<n>`) |

Note also that the **`--hold` flag** (not `+hold`) is a third, distinct thing: it holds the window open by running the command under an **interactive shell** (`KITTY_HOLD=1`), and shows **neither** message — just a live shell. In the **default** direct-launch case (`kitty sh -c '… exit 0'`), **no** persistent completion message is shown at all; the window simply closes.

### Distinction B — There are two different exit-status transports

| Transport | Mechanism | What it carries | Reaches the message? |
|---|---|---|---|
| OS process reaping | `SIGCHLD` → `wait4(-1,&status,WNOHANG)` | raw wait-status of the **direct child** | **No** — `on_child_death(window_id)` gets only a window id |
| Shell integration | OSC 133 `D;<code>` escape sequence on the PTY | the **shell-reported command status** | **Yes** — decoded to `handle_cmd_end(exit_status)` |

Questions 6 and 7 concern the **OS** transport (`SIGCHLD`/`wait4`); questions 5 and 8 concern the **shell-integration** transport (OSC 133). They are independent: the OS mechanism reaps the child but does not deliver a status to the window's message path, which is exactly why the status-bearing notification is produced by the shell-integration path and phrased as a **shell-reported command status**, not the child's OS wait-status.

## Two nuances

### Nuance 1 — Default window teardown is driven by PTY EOF, not by the `SIGCHLD` reap

With `close_on_child_death` at its **default** value `no` (`kitty/options/types.py:500` → `close_on_child_death: bool = False`; `kitty/options/definition.py:2920` → `opt('close_on_child_death', 'no', …)`), the `SIGCHLD` reap does **not** remove the window (it passes `enable_close_on_child_death=false` into `reap_children`, so `mark_child_for_removal` is skipped — Q7). Instead, the window is removed because `read_bytes` on the PTY master returns `false` at EOF/EIO and sets `children[i].needs_removal = true` (Q1). This was proven with a **discriminating** experiment where the foreground command exits immediately but a detached writer keeps the PTY open:

```
$ cat /tmp/kwork/evidence/q1_close_alternate.txt
### DISCRIMINATING close_on_child_death test ###
# child: 'echo hi; setsid -f sleep 4; exit 0'  -> foreground sh exits 0 immediately,
# but a detached 'sleep 4' keeps the PTY slave open (verified: PTY master EOF at ~4s).

--- DEFAULT close_on_child_death=no : window LINGERS until PTY EOF (~4s) ---
default(no): kitty_exit=0  wall=4.31s

--- close_on_child_death=yes : window closes IMMEDIATELY when foreground sh exits ---
yes:         kitty_exit=0  wall=0.31s
```

With the default (`no`), kitty lingers **4.31 s** — until the detached writer closes the PTY (EOF) — proving teardown is PTY-EOF-driven. With `close_on_child_death=yes`, kitty closes in **0.31 s** — immediately on the `SIGCHLD` reap via `mark_child_for_removal`. Both exit `0`. (For a child with no lingering writer, EOF coincides with exit, so the difference is invisible; the detached-writer case is what separates the two mechanisms.)

### Nuance 2 — The completion notification is off by default, and the duration gate is effectively inert

`notify_on_cmd_finish` defaults to `never` (`kitty/options/types.py:560` → `NotifyOnCmdFinish(when='never', duration=5.0, …)`; `kitty/options/definition.py:3190` → `opt('notify_on_cmd_finish', 'never', …)`), so **by default no completion notification is shown** even with shell integration active — confirmed by the Q3b matrix (`enabled_default`: OSC 133 received, `Notify=0`). Separately, as documented under Q5, the `duration` threshold is effectively inert because `last_cmd_output_start_time` is zeroed (`kitty/window.py:1411`) *before* the duration subtraction (`kitty/window.py:1417`), making `last_cmd_output_duration` equal to the raw monotonic clock and thus always `>= duration`. The gate therefore reduces to `when != 'never'`. This is an **apparent pre-existing behavior** in the investigated source and is documented, not changed.

## Observed vs. inferred classification

Every claim in this document is one of: **runtime-observed** (captured from a live run), **source-derived** (read directly from cited source, and consistent with observed behavior), or **corroborated** (external references confirming an in-repo fact). No claim is purely speculative; nothing required an unverifiable inference.

| Claim | Classification | Evidence |
|---|---|---|
| Kitty's own exit code is `0`, stable, child-status-independent | runtime-observed | `q2_exit.txt` (5 runs + child exit 5/42) |
| The child exits and delivers `SIGCHLD` (signal 17, `CLD_EXITED`) | runtime-observed | strace `signalfd4`, `read(7)=128`, decode script |
| Kitty reaps with `wait4(-1,&status,WNOHANG)` | runtime-observed | strace `wait4(…) = 146321` |
| `+hold` banner is `Press Enter or Esc to exit` (bold-green), stable ×2 | runtime-observed | `q3a_hold.txt` |
| `--hold` flag shows an interactive shell (`KITTY_HOLD=1`), no banner | runtime-observed | `q3_hold_flag_env.txt` + screenshot |
| Notification body `Command echo hello finished with status: 0.\nClick to focus.` | runtime-observed | `q3b_notify_body.txt` (D-Bus) |
| Notification off by default; fires only when `when != never` | runtime-observed | Q3b matrix (`enabled_default` Notify=0) |
| OSC 133 `D;0` bytes `\e]133;D;0\a` on the wire | runtime-observed | `q8_byte_exact.txt`, `q8_timeline.txt` |
| Child output `hello`/`world` lands in the `Screen` grid | runtime-observed | `q9_screen_dump.txt` + `q9_grid.png` |
| Child output is NOT on kitty's own stdout/stderr | runtime-observed | `q9_negative.txt` (0 bytes) |
| Default teardown is PTY-EOF (lingers with detached writer); `=yes` reaps immediately | runtime-observed | `q1_close_alternate.txt` (4.31 s vs 0.31 s) |
| `ChildMonitor` owns id/PID/PTY-fd/Screen | source-derived | `kitty/boss.py:370-374, 585-587`; `kitty/child-monitor.c:305-306` |
| `handle_cmd_end()` builds the message | source-derived (+ observed body) | `kitty/window.py:1408-1451`; `q3b_notify_body.txt` |
| `on_child_death(window_id)` carries no status; `mark_monitored_pids` no-op for primary child | source-derived | `kitty/boss.py:881`; `kitty/child-monitor.c:1397-1409` |
| Duration gate reduces to `when != never` (zeroed-before-subtract) | source-derived (+ observed) | `kitty/window.py:1411,1417,1425`; Q3b matrix |
| OSC 133 is a de-facto cross-vendor convention (FinalTerm origin, exit code optional) | corroborated | external references (iTerm2, VS Code, terminfo.dev) |

## Reproducibility (self-contained)

All commands were run from the repository root on the host described in *Environment and build*. Every temporary artifact lives under `/tmp/kwork` (outside the repo). The scripts below are complete — no elisions.

### 0. Safe environment setup (headless X, captured PID, cleanup trap)

```bash
set -o pipefail
export KITTY_REPO="$PWD"                       # repository root
export WORK=/tmp/kwork; mkdir -p "$WORK/evidence"; chmod 700 "$WORK"
# headless X for the GPU terminal:
Xvfb :99 -screen 0 1280x800x24 -nolisten tcp >/tmp/xvfb.log 2>&1 &
XVFB_PID=$!
trap 'kill "$XVFB_PID" 2>/dev/null' EXIT       # tear down Xvfb on shell exit
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
```

### 1. Canonical default build

```bash
mkdir -p "$WORK/bin"; ln -sf /usr/local/bin/python3.11 "$WORK/bin/python3"
export PATH="$WORK/bin:/usr/local/go/bin:$PATH" GOPATH=/root/go CI=true
make clean
( set -o pipefail; make > "$WORK/evidence/make_full.log" 2>&1; echo "MAKE_EXIT=${PIPESTATUS[0]}" )
./kitty/launcher/kitty --version    # -> kitty 0.35.2 created by Kovid Goyal
```

### 2. Kitty's own exit code (Q2)

```bash
for i in 1 2 3 4 5; do
  ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1
  echo "run $i: kitty_exit=$?"
done
```

### 3. Signal + reap trace (Q6/Q7) and the reproducible siginfo decode

```bash
strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -e read=all \
  -o "$WORK/evidence/kitty.strace" \
  ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1
```

`decode_siginfo.py` (decodes the exact `read(7)` payload from the trace):

```python
import struct
first16 = bytes([0x11,0,0,0, 0,0,0,0, 1,0,0,0, 0x91,0x3b,0x02,0])
signo, errno_, code, pid = struct.unpack('<IiiI', first16)
names={17:'SIGCHLD'}; codes={1:'CLD_EXITED'}
print(f"ssi_signo = {signo}  -> {names.get(signo,'?')}")
print(f"ssi_errno = {errno_}")
print(f"ssi_code  = {code}   -> {codes.get(code,'?')}")
print(f"ssi_pid   = {pid}   (the child process that exited)")
```

### 4. On-screen grid proof (Q9) via kitty's real `Screen`

`screen_dump.py`:

```python
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes
cb = Callbacks()
screen = Screen(cb, 5, 20, 100, 10, 20, 0, cb)   # 5 rows x 20 cols
parse_bytes(screen, b"hello\r\nworld\r\n")         # exactly what the child wrote to the PTY slave
for i in range(3):
    print(f"grid line {i}: {str(screen.line(i))!r}")
```

```bash
./kitty/launcher/kitty +launch "$WORK/screen_dump.py"   # -> grid line 0: 'hello' / 1: 'world'
```

### 5. `+hold` banner (Q3a) via a PTY

`pty_capture.py` (captures a program's terminal output through a PTY):

```python
import os, pty, sys, select, time
argv = sys.argv[2:]
outpath = sys.argv[1]
pid, mfd = pty.fork()
if pid == 0:
    os.execvp(argv[0], argv)
buf = bytearray()
deadline = time.time() + 12
sent_enter = False
while True:
    r,_,_ = select.select([mfd], [], [], 0.3)
    if mfd in r:
        try:
            d = os.read(mfd, 65536)
        except OSError:
            break
        if not d: break
        buf += d
    if not sent_enter and (b'Press Enter' in buf or time.time() > deadline-8):
        time.sleep(0.3); os.write(mfd, b'\r'); sent_enter = True
    if time.time() > deadline: break
    try:
        wpid,_ = os.waitpid(pid, os.WNOHANG)
        if wpid==pid:
            time.sleep(0.2)
            while True:
                rr,_,_=select.select([mfd],[],[],0.3)
                if mfd in rr:
                    try: d=os.read(mfd,65536)
                    except OSError: break
                    if not d: break
                    buf+=d
                else: break
            break
    except ChildProcessError:
        break
open(outpath,'wb').write(bytes(buf))
```

```bash
python3.11 "$WORK/pty_capture.py" "$WORK/evidence/hold1.bin" \
  ./kitty/launcher/kitty +hold sh -c 'echo hello; echo world; exit 0'
```

### 6. Real shell integration → notification (Q3b/Q5/Q8) driven by synthetic keystrokes

`xtype.py` (injects real keystrokes via the X11 XTEST extension — the canonical user-input path):

```python
import ctypes, ctypes.util, sys, time
X11 = ctypes.CDLL("libX11.so.6")
XTST = ctypes.CDLL("libXtst.so.6")
X11.XOpenDisplay.restype = ctypes.c_void_p
X11.XKeysymToKeycode.restype = ctypes.c_uint
X11.XStringToKeysym.restype = ctypes.c_ulong
dpy = X11.XOpenDisplay(None)
if not dpy: sys.exit("no display")
def keysym(s): return X11.XStringToKeysym(s.encode())
SHIFT_KC = X11.XKeysymToKeycode(ctypes.c_void_p(dpy), ctypes.c_ulong(keysym("Shift_L")))
def tapsym(ks, shift=False):
    kc = X11.XKeysymToKeycode(ctypes.c_void_p(dpy), ctypes.c_ulong(ks))
    if kc == 0: return
    if shift: XTST.XTestFakeKeyEvent(ctypes.c_void_p(dpy), ctypes.c_uint(SHIFT_KC), 1, ctypes.c_ulong(0))
    XTST.XTestFakeKeyEvent(ctypes.c_void_p(dpy), ctypes.c_uint(kc), 1, ctypes.c_ulong(0))
    XTST.XTestFakeKeyEvent(ctypes.c_void_p(dpy), ctypes.c_uint(kc), 0, ctypes.c_ulong(0))
    if shift: XTST.XTestFakeKeyEvent(ctypes.c_void_p(dpy), ctypes.c_uint(SHIFT_KC), 0, ctypes.c_ulong(0))
    X11.XFlush(ctypes.c_void_p(dpy))
    time.sleep(0.02)
SPECIAL={' ':'space','\n':'Return','\t':'Tab','-':'minus','.':'period',';':'semicolon','/':'slash','_':('minus',True)}
def typ(s):
    for ch in s:
        if ch in SPECIAL:
            v=SPECIAL[ch]
            if isinstance(v,tuple): tapsym(keysym(v[0]),shift=v[1])
            else: tapsym(keysym(v))
        elif ch.isupper():
            tapsym(keysym(ch), shift=True)
        else:
            tapsym(keysym(ch))
if __name__=="__main__":
    time.sleep(float(sys.argv[1]))
    typ(sys.argv[2])
```

`si_matrix.sh` (runs the enabled/disabled/default matrix; each case counts OSC 133 markers via `--dump-bytes` and `Notify` calls via `dbus-monitor`):

```bash
#!/bin/bash
set -u
cd "$KITTY_REPO"
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
EV=/tmp/kwork/evidence
run_case() {  # $1=label $2..=extra kitty opts
  local label="$1"; shift
  local dbuslog="$EV/dbus_${label}.log" dump="$EV/dump_${label}.bin"
  dbus-monitor "interface='org.freedesktop.Notifications',member='Notify'" >"$dbuslog" 2>/dev/null &
  local dpid=$!
  timeout 30 ./kitty/launcher/kitty --config NONE "$@" --dump-bytes "$dump" bash -i >/dev/null 2>&1 &
  local kpid=$!
  /opt/kitty-venv/bin/python /tmp/kwork/xtype.py 4 "echo hello"$'\n'
  sleep 2
  /opt/kitty-venv/bin/python /tmp/kwork/xtype.py 0 "exit"$'\n'
  wait $kpid 2>/dev/null
  sleep 1; kill $dpid 2>/dev/null
  echo "[$label] kitty done"
}
run_case enabled_notify -o "notify_on_cmd_finish always 0"
run_case disabled_notify -o "shell_integration disabled" -o "notify_on_cmd_finish always 0"
run_case enabled_default
```

The whole matrix must be run under a session bus so notifications are deliverable:

```bash
dbus-run-session -- bash "$WORK/si_matrix.sh"
```

### 7. Cleanup and repository-unchanged verification

```bash
rm -rf /tmp/kwork          # remove ALL temporary artifacts (outside the repo)
git status --porcelain     # only the answer document should appear
```

Actual final status (the only change on the branch is this document):

```
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md
```

## Coverage checklist

Every question and every named item is answered, and every implied condition was exercised.

### Questions

- [x] Q1 end-to-end flow — spawn → PTY → read → parse → Screen → GPU render; exit → `SIGCHLD` → reap → teardown.
- [x] Q2 kitty's own exit code — `0`, stable ×5, child-status-independent.
- [x] Q3 full completion message — none by default; `+hold` banner `Press Enter or Esc to exit`; shell notification `Command … finished with status: 0.\nClick to focus.`
- [x] Q4 tracking component — `ChildMonitor`.
- [x] Q5 message-generating function — `Window.handle_cmd_end()`.
- [x] Q6 signal — `SIGCHLD` (17), via `signalfd`.
- [x] Q7 system call — `wait4(-1,&status,WNOHANG)` (`waitpid` family).
- [x] Q8 status transport — OSC 133 `D;<code>` → `vt-parser.c` → `screen.c` → `handle_cmd_end()`.
- [x] Q9 output location — kitty's on-screen terminal grid (not kitty's own stdout/stderr).

### Named items

- [x] `SIGCHLD` (Q6) — observed via `signalfd` siginfo decode.
- [x] `waitpid`/`wait4` (Q7) — observed in strace.
- [x] shell integration / OSC escape codes (Q8) — real OSC 133 `D;0` captured and traced through the parser.
- [x] `ChildMonitor` (Q4) — cited and its `add_child(id,pid,fd,screen)` registration shown.
- [x] `handle_cmd_end()` (Q5) — full method quoted; observed message body.
- [x] terminal window vs. elsewhere (Q9) — grid dump + screenshot + empty own-stdout.

### Conditions exercised

- [x] Default direct launch (`kitty sh -c '… exit 0'`) — window closes, no message.
- [x] `close_on_child_death=yes` — immediate close (discriminating 0.31 s vs 4.31 s).
- [x] `kitty +hold` — green banner, ×2 byte-identical (stability).
- [x] `kitten __hold_till_enter__` — same banner (non-canonical corroboration).
- [x] `kitty --hold` flag — interactive shell, `KITTY_HOLD=1`, no banner.
- [x] Shell integration enabled + `notify_on_cmd_finish always` — OSC 133 + one notification.
- [x] Shell integration disabled (kitty level) — no OSC 133, no notification (negative control).
- [x] Shell integration enabled + default `notify_on_cmd_finish=never` — OSC 133 present, no notification (gate control).
- [x] Nonzero child status controls — child exit 5 and 42 → kitty exit 0.
- [x] Before/during/after markers — the OSC 133 `D→A→C→D` timeline.
- [x] Kitty's own stdout/stderr negative check — 0 bytes / benign only.
