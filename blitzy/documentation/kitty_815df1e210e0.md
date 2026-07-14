# How kitty handles a child that prints a few lines and exits with status 0

An investigative, runtime-observed answer, grounded in captured output and `file:line` source citations.

## Scope and provenance

- **Investigated software:** `kitty` terminal emulator, repository `kovidgoyal/kitty`.
- **Investigated (source) commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. Every `file:line` citation in this document refers to the tree at that commit.
- **This document lives** on branch `blitzy-4da995f7-9d87-4535-892b-6e7ceff8c1ad`; it is added by documentation-only commits that touch **only** this Markdown file and never the investigated source, so the branch `HEAD` advances but is **not** load-bearing. The durable, HEAD-independent invariant is that the branch's source tree is **byte-identical to the investigated commit** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`: the sole delta versus that commit is the addition of this file — `git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD` reports exactly `A blitzy/documentation/kitty_815df1e210e0.md`. No file under `kitty/`, `kittens/`, `tools/`, `shell-integration/`, or `docs/` was modified.
- **Methodology (per the SWE-AtlasQnA rules):** the relevant code paths were **built and run first inside the approved container** (below), real output was captured with temporary observation scripts, and only then was this answer written. Every behavioral claim is presented next to the exact command that produced it and the unedited output, and is labeled **[observed]**, **[source-derived]**, **[inferred]**, or **[non-canonical corroboration]**. All temporary scripts and evidence live in a per-session `mktemp -d` directory **outside** the repository, and the repository is left byte-for-byte unchanged apart from this file.

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

### Canonical environment: the approved container **[observed]**

Every build, run, and capture in this document was performed **inside the approved Docker image** the project's setup instructions name, `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (the `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0…` image published under that repo). The image is **available and was pulled/started successfully**; its identity is fixed:

```
$ docker image inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0 \
    --format 'ImageID={{.Id}}
RepoDigest={{index .RepoDigests 0}}'
ImageID=sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288
RepoDigest=ghcr.io/scaleapi/swe-atlas@sha256:60da90a7183a82861fc6d1d40cb8086baa6a8a0e0d05f26d03aafd0f5b3cc384
```

The image's `Entrypoint` is `/bin/bash` and its `WorkingDir` is `/app`, which holds the kitty source tree checked out at the investigated commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (verified below). The canonical invocation used for an interactive session was:

```
$ docker run --rm -it ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0
# (inside the container, working directory /app)
```

All observation tooling that the base image does not ship (`strace`, `Xvfb`, software GL, `dbus-x11`, `libxtst6`, ImageMagick, and the `dunst` notification daemon) was installed **into the disposable container only** with the image's own `apt` (the container has network access; the repository under investigation is never modified). Those installs are listed explicitly in the **Reproducibility** section so a reader can recreate the exact environment. Reporting the previously-recorded native build as canonical would have been wrong; that earlier disclosure has been corrected to name the approved image and its actual toolchain (below).

### Build host and toolchain **[observed]**

The approved image's actual OS, interpreter, and toolchain (captured inside the container):

```
$ grep PRETTY_NAME /etc/os-release
PRETTY_NAME="Ubuntu 24.04.2 LTS"
$ python3 --version ; readlink -f "$(command -v python3)"
Python 3.12.3
/usr/bin/python3.12
$ go version
go version go1.23.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ make --version | head -1
GNU Make 4.3
$ bash --version | head -1
GNU bash, version 5.2.21(1)-release (x86_64-pc-linux-gnu)
```

The investigated source lives at `/app` and is checked out at the target commit, with a clean working tree:

```
$ cd /app && git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git status --porcelain
$        # (empty: the source tree in the image is byte-identical to the investigated commit)
```

Every `file:line` citation in this document refers to that tree. The image ships the default system `python3` (3.12.3), which `make`/`setup.py` use directly — no interpreter shim is created and no virtual environment is required for the build.

### Canonical default build: `make` **[observed]**

Kitty's canonical build is `make`, whose `all:` target invokes `python3 setup.py` (default, no `--debug`). A full clean rebuild inside the approved container (`CI=true` matches the image's CI defaults; `go` is already on the image `PATH` at `/usr/local/go/bin`):

```
$ cd /app
$ CI=true make clean            # -> exit 0
$ ( set -o pipefail; make > make_full.log 2>&1; echo "MAKE_EXIT=${PIPESTATUS[0]}" )
MAKE_EXIT=0
$ wc -l make_full.log
381 make_full.log
```

The complete build log is **381 lines**; the block below is a **[observed — clearly-labeled excerpt]** of that full log (the head, the two backend-selection boundaries, the final compile unit, and the link phase). The full 381-line log is produced verbatim by re-running the `make` command above (see Reproducibility §1):

```
python3 setup.py 
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
   … (wayland client-protocol generation, 28 units) …
[28/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
 done
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
   … (compile units 8–121) …
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
   … (Go kitten/kittens/tools link list) …
```

The build exits **0** (`MAKE_EXIT=0`) and compiles **122 native units**, then links the C extension, the two GLFW backends, and the Go `kitten` tools. Unlike a host lacking Wayland headers, the approved image ships `wayland-protocols 1.34` and `wayland-client 1.22.0`, so the default build enables **both the Wayland and X11 backends** — visible in the log as `[wayland] glfw/wl_window.c` / `[x11] glfw/x11_window.c` compile units and the `[wayland] kitty/glfw-wayland` / `[x11] kitty/glfw-x11` link units (the compile phase contains 37 `[wayland]` and 20 `[x11]` units). The backend selection does **not** affect the child-exit investigation (the process/PTY/signal/parser paths are backend-independent), but it is reported accurately here because it is a build-dependent fact. The produced artifacts and version:

```
$ ls kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
kitty/fast_data_types.so
kitty/launcher/kitten
kitty/launcher/kitty
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### How runs were performed (headless GUI) **[observed]**

Because kitty is a GPU terminal, runs use a headless X server plus software GL. A private display number is allocated per session (not a hard-coded one) so parallel runs never collide, and the server is torn down via a shell `trap`:

```
DISP=":$((90 + RANDOM % 100))"                                   # unique per session
Xvfb "$DISP" -screen 0 1280x800x24 -nolisten tcp >/dev/null 2>&1 &
XVFB_PID=$!; trap 'kill "$XVFB_PID" 2>/dev/null' EXIT
export DISPLAY="$DISP" LIBGL_ALWAYS_SOFTWARE=1                    # Mesa llvmpipe software GL
```

The canonical reproduction command used throughout is:

```
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
```

`--config NONE` guarantees **default** configuration (no user `kitty.conf` is read), so every observed value reflects what a normal user sees with defaults. The exact, self-contained harnesses (with prerequisite installation, a `mktemp -d` work directory, `set -euo pipefail`, and per-step assertions) are reproduced verbatim in the **Reproducibility** section.

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
$ export DISPLAY="$DISP" LIBGL_ALWAYS_SOFTWARE=1          # $DISP allocated per session (see "How runs were performed")
$ strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -e read=all \
    -o "$EVID/kitty.strace" \
    ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
$ echo "kitty_exit=$?"
kitty_exit=0
```

Captured in the approved container; `$EVID` is the per-session `mktemp -d` evidence directory established in Reproducibility §0. The trace is large because `-e read=all` dumps every read payload (this run: 226267 lines); the lines below are the contiguous child-death window, extracted verbatim.

### Raw output — the contiguous death window from the strace **[observed]**

```
$ cat "$EVID/q6q7_window.txt"
8890  read(8, "hello\r\nworld\r\n", 1048576) = 14
8891  +++ exited with 0 +++
8890  read(8, 0x5c862675368e, 1048562)  = -1 EIO (Input/output error)
8890  read(7, "\21\0\0\0\0\0\0\0\1\0\0\0\273\"\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128
8890  read(7, 0x7d484c71b3a0, 4096)     = -1 EAGAIN (Resource temporarily unavailable)
8890  wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 8891
8890  wait4(-1, 0x7d4717ffee60, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

Here TID `8890` is kitty's monitor thread, `8891` is the child. Reading top to bottom: the monitor reads the child's `hello\r\nworld\r\n` (14 bytes) from PTY master fd 8; the child exits 0; the next PTY read returns `EIO` (the PTY-EOF condition after the last writer closed); a `signalfd` read on fd 7 delivers the `SIGCHLD` siginfo (followed by an `EAGAIN` read that drains the signal fd); and `wait4` reaps the child (returning pid `8891` with `WIFEXITED && WEXITSTATUS==0`), then a second `wait4` returns `ECHILD`. The numeric identifiers (TIDs `8890`/`8891`, the buffer addresses) are per-run; the **structure** — `read(...)=14`, then EOF/`EIO`, then the `SIGCHLD` siginfo, then the reap returning the child PID with `WEXITSTATUS==0`, then `ECHILD` — is stable across runs (a second run produced monitor TID `9034`/child `9035` with the identical structure; see Q6).

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
$ for i in 1 2 3 4 5; do ./kitty/launcher/kitty --config NONE \
      sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1; echo "run $i: kitty_exit=$?"; done
run 1: kitty_exit=0
run 2: kitty_exit=0
run 3: kitty_exit=0
run 4: kitty_exit=0
run 5: kitty_exit=0
$ for c in 5 42; do ./kitty/launcher/kitty --config NONE \
      sh -c "echo hi; exit $c" >/dev/null 2>&1; printf 'child exit %-2s -> kitty_exit=%s\n' "$c" "$?"; done
child exit 5  -> kitty_exit=0
child exit 42 -> kitty_exit=0
```

Each `run N` line launches kitty and prints only kitty's **own** `$?` (the child's `hello`/`world` go to the terminal grid and are suppressed here with `>/dev/null 2>&1`); the last two lines re-run with the child exiting `5` and `42`. Captured verbatim in the approved container.

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

Captured through a PTY (the banner is written to the terminal, so a PTY is required to see it). The capture records the **complete lifecycle**: the harness reads until the banner is displayed and the kitten goes quiet, then sends a dismiss key so the kitten exits cleanly and emits its terminal-finalization tail, then reads to EOF. Because the kitten pushes the kitty keyboard protocol (`\x1b[>29u`, visible in the capture), the dismiss key must be sent in that protocol's **CSI-u** encoding (`\x1b[13u` for Enter or `\x1b[27u` for Esc); a legacy `\r`/`\x1b` is not recognized and the kitten would keep waiting:

```
$ cat "$EVID/q3a_hold_full.txt"
### kitty +hold (public entry) COMPLETE capture length (bytes): 176
### kitty +hold run #1 COMPLETE capture (Python repr):
b'hello\r\nworld\r\n\x1b[?s\x1b[*x\x1b[4l\x1b[?1l\x1b[?5l\x1b[?2004l\x1b[?1004l\x1b[?1000l\x1b[?1002l\x1b[?1003l\x1b[?1005l\x1b[?1006l\x1b[?8h\x1b[?7h\x1b[?25h\x1b[>29u\x1b[?25l\r\n\x1b[1;32mPress Enter or Esc to exit\x1b[m\x1b[?25h\x1b[<u\x1b7\x1b[?r\x1b8'

### banner segment (run #1), Python repr then cat -v:
b'\x1b[1;32mPress Enter or Esc to exit\x1b[m'
^[[1;32mPress Enter or Esc to exit^[[m

### finalization tail AFTER the banner SGR reset (run #1), Python repr:
b'\x1b[?25h\x1b[<u\x1b7\x1b[?r\x1b8'

### run #2 COMPLETE capture length (bytes): 176
### byte-identical COMPLETE capture across the two +hold runs?
run1==run2: True | lengths: 176 176
```

The complete capture is **176 bytes** and decomposes into four parts: (1) the child's output `hello\r\nworld\r\n`; (2) a block of terminal-setup sequences (mode resets, cursor show/hide, and `\x1b[>29u` pushing the kitty keyboard protocol); (3) the **banner** `\x1b[1;32mPress Enter or Esc to exit\x1b[m`; and (4) the **finalization tail** `\x1b[?25h\x1b[<u\x1b7\x1b[?r\x1b8` (show cursor, pop keyboard flags `\x1b[<u`, save/restore cursor) emitted as the kitten exits. The **exact** message text is `Press Enter or Esc to exit`, wrapped in SGR `\x1b[1;32m` (bold, green) and reset `\x1b[m`. The entire 176-byte capture is **byte-identical across two runs** (stability requirement met). `kitty +hold` is the canonical public entry point; it dispatches internally to the `kitten __hold_till_enter__` helper (confirmed by the process tree: `kitty +hold …` spawns `kitten __hold_till_enter__ …`), so both produce these exact bytes.

#### Command and raw output — the `--hold` flag shows **no** banner **[observed]**

```
$ cat "$EVID/q3_hold_flag_env.txt"
### any process whose environ contains KITTY_HOLD=1 (the run-shell hold child) ###
pid=<pid> cmdline=[/bin/bash --posix ]
    KITTY_SHELL_INTEGRATION=enabled
    KITTY_HOLD=1
```

So the `--hold` flag holds the window open by running an **interactive `bash`** with `KITTY_HOLD=1` — not a banner. The live window is captured externally and confirmed by OCR (no human "inspection" is relied upon):

```
$ import -window root "$EVID/q3_hold_flag.png"
$ identify "$EVID/q3_hold_flag.png"
$EVID/q3_hold_flag.png PNG 1280x800 1280x800+0+0 8-bit Grayscale Gray 256c 3095B 0.000u 0:00.000
$ convert "$EVID/q3_hold_flag.png" -crop 600x200+0+0 +repage "$EVID/q3_hold_flag_crop.png"
$ tesseract "$EVID/q3_hold_flag_crop.png" stdout 2>/dev/null | sed '/^[[:space:]]*$/d'
hello
world
root@<container-id>:/app#
$ grep -c -i 'Press Enter or Esc to exit' "$EVID/q3_hold_flag_ocr.txt"
0
```

OCR of the `kitty --hold` window reads the two output lines `hello` and `world` followed by an **interactive shell prompt** (`root@<container-id>:/app#`), and the banner text `Press Enter or Esc to exit` is **absent** (grep count `0`) — confirming that the `--hold` flag holds via an interactive shell, not via the banner path. (The prompt hostname is the disposable container's id and is redacted here.)

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

A **real interactive `bash` running inside real kitty** was driven by synthesizing real keystrokes with the X11 `XTEST` extension (the canonical user-input path), under `dbus-run-session` so the notification travels over a private session bus, with `notify_on_cmd_finish always 0` set so the notification fires. A real notification daemon — **`dunst 1.9.2`** — was started on that bus as the `org.freedesktop.Notifications` server, so the notification is not merely *sent* but actually *rendered on screen*; this is proven three independent ways below.

**(a) The daemon that rendered it, and that it was live on screen at capture time:**

```
$ gdbus call --session --dest org.freedesktop.Notifications \
      --object-path /org/freedesktop/Notifications \
      --method org.freedesktop.Notifications.GetServerInformation
('dunst', 'knopwob', '1.9.2 (2023-04-20)', '1.2')
$ dunstctl count
              Waiting: 0
  Currently displayed: 1
              History: 0
```

`Currently displayed: 1` means that, at the instant the screenshot below was taken, dunst had exactly one notification visible on the screen — the one kitty emitted.

**(b) The D-Bus `Notify` method call kitty emitted** (captured by `dbus-monitor` eavesdropping on the same session bus; the transient envelope fields `sender`/`serial`/`time=` are elided as `<…>` because they legitimately vary run-to-run, and the icon path is the in-container path `/app/logo/kitty.png`):

```
$ sed -n '/member=Notify/,/int32 -1/p' q3b_notify_body_raw.txt
method call <…> -> destination=org.freedesktop.Notifications; interface=org.freedesktop.Notifications; member=Notify
   string "kitty"
   uint32 0
   string "/app/logo/kitty.png"
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

**(c) The popup pixels on screen.** A single screenshot of the whole X root (`import -window root q3b_notif_render.png`, `PNG 1280x800`) contains both kitty's terminal grid and dunst's popup. OCR of the full root reads the grid (`root@<container-id>:/app# echo hello` / `hello` / prompt); OCR of the cropped top-right region where dunst places the popup reads the notification itself:

```
$ tesseract q3b_notif_crop.png - --psm 6
Command echo hello
finished with status:
0.
Click to focus.
```

(The header line also OCRs the app name `kitty`. The digit `0` is rendered small; two threshold passes read it as `8`/`Q`, so the byte-exact `0` is taken from the D-Bus body in (b), not from OCR.)

The **exact body text** (the fifth argument to `Notify`) is:

```
Command echo hello finished with status: 0.
Click to focus.
```

i.e. `Command echo hello finished with status: 0.\nClick to focus.` with a literal newline between the two sentences. Stability note (per finding on run-to-run identity): across repeated runs the **body is byte-identical**, but the D-Bus envelope fields (`sender`, `serial`, `time=`) legitimately vary and are therefore not claimed to be stable.

#### Command and raw output — the enable/disable matrix **[observed]**

The same XTEST-driven real-bash-through-kitty run was tallied two ways. **Counting note (non-canonical instrumentation):** `--dump-bytes` is kitty's own raw-byte dump diagnostic option (defined at `kitty/cli.py:985-986`: "Path to file in which to store the raw bytes received from the child process") — it is used here **only as a convenient per-run counter** of how many OSC 133 `C`/`D;0` markers kitty's parser received, and is *not* the canonical proof that the sequence exists on the wire. The **canonical, byte-exact proof** that the real shell writes `\e]133;D;0\a` is the `strace` of the shell's `write()` (and the PTY-tee corroboration) shown under **Q8**, which use no debug interface. `dbus-monitor` counts the `Notify` calls:

```
condition                                          OSC133 C   OSC133 D;0   Notify calls
kitty -o "notify_on_cmd_finish always 0"  (int on)      2          2            1
kitty -o "shell_integration disabled" (+notify)         0          0            0
kitty  (defaults: notify_on_cmd_finish=never)           2          2            0
```

Reading the matrix:
- With shell integration **on** and notification **enabled**, the shell emits OSC 133 `C`/`D;0` and kitty fires exactly **one** notification.
- With shell integration **disabled at the kitty level**, **no** OSC 133 markers are produced and **no** notification fires (negative control).
- With shell integration **on** but `notify_on_cmd_finish` at its **default (`never`)**, the OSC 133 markers still arrive but **no** notification fires — isolating the `when != 'never'` condition (this case varies only `when`, so it says nothing about the separate `duration` term, which is examined under Q5 / Nuance 2 and shown there to be a functional kitty-uptime threshold).

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

The C side declares those fields on the per-child `Child` struct, `kitty/child-monitor.c:65-71` (contiguous):

```c
typedef struct {
    Screen *screen;
    bool needs_removal;
    int fd;
    unsigned long id;
    pid_t pid;
} Child;
```

and `add_child()` parses the Python arguments straight into those fields — the `PyArg_ParseTuple` format string `"kiiO"` binds `id`, `pid`, `fd`, and `screen` into a queued `Child`, `kitty/child-monitor.c:305-317` (contiguous):

```c
add_child(ChildMonitor *self, PyObject *args) {
#define add_child_doc "add_child(id, pid, fd, screen) -> Add a child."
    children_mutex(lock);
    if (self->count + add_queue_count >= MAX_CHILDREN) { PyErr_SetString(PyExc_ValueError, "Too many children"); children_mutex(unlock); return NULL; }
    add_queue[add_queue_count] = EMPTY_CHILD;
#define A(attr) &add_queue[add_queue_count].attr
    if (!PyArg_ParseTuple(args, "kiiO", A(id), A(pid), A(fd), A(screen))) {
        children_mutex(unlock);
        return NULL;
    }
#undef A
    INCREF_CHILD(add_queue[add_queue_count]);
    add_queue_count++;
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
$ cat q5_recorded_state.txt
NOTIFY_FIRED recorded_exit_status=[0] recorded_cmdline=[echo hello]
```

### The duration gate is a kitty-uptime threshold (zeroed-before-subtract) **[observed + source-derived; documented, not changed]**

The gate at line 1425 is `if last_cmd_output_duration >= duration and when != 'never':`. `last_cmd_output_duration` is computed at line 1417 as `end_time - self.last_cmd_output_start_time`, where `end_time = monotonic()` (line 1416) — but `self.last_cmd_output_start_time` was **already reset to `0.` at line 1411**, *before* the subtraction. Consequently `last_cmd_output_duration == end_time == monotonic()`: the recorded "duration" is not the command's real runtime but kitty's clock reading at command-finish.

The decisive detail is *which* `monotonic` this is. It is **kitty's own C clock**, imported from `.fast_data_types` (import block at `kitty/window.py:44`; the name `monotonic` at `kitty/window.py:75`) — **not** Python's `time.monotonic`. Per `kitty/monotonic.h:60-68`, `monotonic()` returns `monotonic_() - monotonic_start_time`, and `init_monotonic()` captures `monotonic_start_time` at kitty startup, so kitty's `monotonic()` is **seconds since kitty started**, beginning at ~`0`. This was measured directly through kitty's canonical `+runpy` entry point, stable across three runs:

```
$ for r in 1 2 3; do ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import monotonic as km
from time import monotonic as pm
import time
print("kitty_monotonic_at_startup=%.6f  python_time_monotonic=%.6f" % (km(), pm()))
time.sleep(2)
print("kitty_monotonic_after_2s_sleep=%.6f  python_time_monotonic=%.6f" % (km(), pm()))'; done
kitty_monotonic_at_startup=0.000717  python_time_monotonic=3008972.218154
kitty_monotonic_after_2s_sleep=2.000825  python_time_monotonic=3008974.218268
kitty_monotonic_at_startup=0.000631  python_time_monotonic=3008974.331554
kitty_monotonic_after_2s_sleep=2.000744  python_time_monotonic=3008976.331672
kitty_monotonic_at_startup=0.000779  python_time_monotonic=3008976.449320
kitty_monotonic_after_2s_sleep=2.000888  python_time_monotonic=3008978.449434
```

kitty's `monotonic()` reads ~`0.0007 s` at startup and grows in real time to ~`2.0008 s` after a 2 s sleep; Python's `time.monotonic()` here reads ~`3,008,972 s`, a large since-boot-style counter that kitty does **not** consult on this path. (An earlier draft mislabeled kitty's value as a since-boot figure; that description belongs to Python's clock, not kitty's.)

Because `last_cmd_output_duration == monotonic()` equals **kitty's uptime at the instant the command's OSC 133 `D` arrives**, the gate is a **functional** threshold, not a no-op: a completion notification fires only when `when != 'never'` **and** kitty has been running at least `duration` seconds. With the default `duration = 5.0`, a command that finishes within the first 5 s after kitty launches is **gated out**; one that finishes later fires. The threshold is real and observable — it simply keys off kitty's uptime rather than the command's own runtime, which is the apparent pre-existing defect (documented here, not changed).

#### Empirical corroboration with a **non-zero** duration **[observed]**

The gate must be probed with a non-zero `duration`; probing with `always 0` is circular, because every clock reading trivially satisfies `>= 0`. A real interactive `bash` was driven inside kitty with `-o "notify_on_cmd_finish always 5"` (the default duration), keystrokes synthesized via the X11 `XTEST` extension under `dbus-run-session`, `Notify` calls counted with `dbus-monitor` and OSC 133 markers counted with `--dump-bytes` (the same non-canonical diagnostic counter noted under Q3b — the canonical byte-exact proof of the markers is the `strace write()` capture under Q8), varying only **when** the `echo hello` command finishes relative to kitty start:

```
$ cat dgate_matrix.txt
===== notify_on_cmd_finish always 5 (NON-ZERO, = default duration) =====
--- EARLY: command finishes <5s after kitty start -> expect gated (Notify=0) ---
early_1        inject@2.5  OSC133_D0=2  Notify_total=0
early_2        inject@2.5  OSC133_D0=2  Notify_total=0
--- LATE: command finishes >5s after kitty start -> expect fires (Notify=1) ---
late_1         inject@8    OSC133_D0=2  Notify_total=1
late_2         inject@8    OSC133_D0=2  Notify_total=1
===== CONTROL: always 100 + EARLY -> expect gated (Notify=0) =====
dg100_early    inject@2.5  OSC133_D0=2  Notify_total=0
===== CONTROL: always 0 + EARLY (old circular method) -> Notify=1 =====
a0_early       inject@2.5  OSC133_D0=2  Notify_total=1
```

The same `echo hello` under the same `always 5` config fires **0** notifications when it finishes early (×2) and **1** when it finishes late (×2); `always 100` gates an early command whose uptime has not yet crossed 100 s; and `always 0` (the circular method the original draft used) fires trivially. In every row the OSC 133 `D;0` sequence still arrives (count `2` — the initial-prompt marker plus the `echo hello` marker), so the transport is constant and the **notification decision is the sole differentiator**. The fired notification's body, captured byte-exact over D-Bus in a LATE run, is exactly the Q3b body:

```
$ sed -n '/member=Notify/,/Click to focus/p' dbus_late.log
method call <…> -> destination=org.freedesktop.Notifications; interface=org.freedesktop.Notifications; member=Notify
   string "kitty"
   uint32 0
   string "/app/logo/kitty.png"
   string "kitty"
   string "Command echo hello finished with status: 0.
Click to focus."
```

(transient envelope fields `sender`/`serial`/`time=` elided as `<…>`; icon path is the in-container path `/app/logo/kitty.png`). By contrast the EARLY capture contains **no** `Notify` method call at all:

```
$ grep -c "^method call" dbus_early.log
0
```

This is an **apparent pre-existing behavior** in the investigated source (the "duration" is really kitty's uptime, not the command's runtime); per scope, **no source change is made** — only documentation.

### Rationale

`handle_cmd_end(exit_status)` is the sole place where the shell-reported status string becomes the message: it stores `last_cmd_exit_status`, and, when the gate passes, formats `cmd.body` (line 1429) with the command line and the status. `cmd_output_marking(is_start=False, cmdline=…)` is its only caller in the OSC 133 `D` path, forwarding the decoded status string.


## Q6 — The OS signal kitty listens for

### Answer: `SIGCHLD` (signal 17) **[observed]**

Kitty learns a child terminated via **`SIGCHLD`**, delivered through a **`signalfd`** (Linux) rather than an async signal handler. This was observed directly.

### Raw output — signalfd setup and the SIGCHLD siginfo **[observed]**

During startup, kitty blocks its handled signals and creates a `signalfd` whose mask includes `CHLD`:

```
$ grep -m1 'rt_sigprocmask(SIG_BLOCK, \[' "$EVID/kitty.strace" | grep CHLD
8824  rt_sigprocmask(SIG_BLOCK, [HUP INT USR1 USR2 TERM CHLD], NULL, 8) = 0
$ grep -m1 'signalfd4' "$EVID/kitty.strace"
8824  signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
```

When the child exits, the monitor thread reads one `struct signalfd_siginfo` (128 bytes) from fd 7. The `strace -e read=all` payload:

```
8890  read(7, "\21\0\0\0\0\0\0\0\1\0\0\0\273\"\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128
 | 00000  11 00 00 00 00 00 00 00  01 00 00 00 bb 22 00 00  .............".. |
 | 00010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................ |
 | 00020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  ................ |
```

Decoding those exact bytes (the first four fields of `struct signalfd_siginfo`, little-endian `<IiiI`) with a reproducible script:

```
$ cat "$EVID/decode_siginfo.py"
import struct
# First 16 bytes of the struct signalfd_siginfo read from kitty's signal fd (fd 7),
# captured verbatim by: strace -e read=all ... (the "read(7, ...) = 128" payload).
first16 = bytes([0x11,0,0,0, 0,0,0,0, 1,0,0,0, 0xbb,0x22,0,0])
signo, errno_, code, pid = struct.unpack('<IiiI', first16)
names={17:'SIGCHLD'}; codes={1:'CLD_EXITED'}
print(f"ssi_signo = {signo}  -> {names.get(signo,'?')}")
print(f"ssi_errno = {errno_}")
print(f"ssi_code  = {code}   -> {codes.get(code,'?')}")
print(f"ssi_pid   = {pid}   (the child process that exited)")

$ python3 "$EVID/decode_siginfo.py"
ssi_signo = 17  -> SIGCHLD
ssi_errno = 0
ssi_code  = 1   -> CLD_EXITED
ssi_pid   = 8891   (the child process that exited)
```

`ssi_signo=17` is `SIGCHLD`; `ssi_code=1` is `CLD_EXITED`; `ssi_pid=8891` is exactly the child PID that `+++ exited with 0 +++` in Q1 (its little-endian bytes `bb 22` = `0x22bb` = `8891`). (The `signalfd`/`sigprocmask` calls run on the setup thread TID `8824`; the siginfo read and reap run on the monitor thread TID `8890`.) A second run reproduced the identical first 12 bytes `11 00 00 00  00 00 00 00  01 00 00 00` (`ssi_signo=17`, `ssi_errno=0`, `ssi_code=1`) with only `ssi_pid` differing to track that run's child, confirming the siginfo structure is stable across runs.

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
8890  wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 8891
8890  wait4(-1, 0x7d4717ffee60, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

The first `wait4` reaps child `8891` with `WIFEXITED && WEXITSTATUS==0`; the second returns `ECHILD` (nothing left to reap), ending the loop. `wait4` is the syscall the C library uses to implement `waitpid`; both `wait4` calls appear on the monitor thread TID `8890`.

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
2. `mark_monitored_pids(pid, status)` records the status **only for PIDs explicitly registered in the `monitored_pids[]` array** — it is a **no-op for the primary window child**, which is not in that array. `kitty/child-monitor.c:1397-1410` (contiguous; closing brace at 1410):

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

**Canonical capture (primary).** The sequence was captured at its true source: an `strace -f -e trace=write -e write=all` of the **real `bash` that kitty launched itself** (`./kitty/launcher/kitty --config NONE bash` under a headless X server; `echo hello` then `exit` typed via X11 `XTEST`). No debug or remote-control interface is used — this is the shell's own `write()` syscall. At the prompt drawn **after** `echo hello` returns, the child `bash` writes a 174-byte prompt block; the exit-status marker sits at offset `0x14` inside it:

```
$ sed -n '407,410p' kitty_osc.strace
<bash-pid> write(2, "\33]133;k;start_kitty\7\33]133;D;0\7\33]"..., 174) = 174
 | 00000  1b 5d 31 33 33 3b 6b 3b  73 74 61 72 74 5f 6b 69  .]133;k;start_ki |
 | 00010  74 74 79 07 1b 5d 31 33  33 3b 44 3b 30 07 1b 5d  tty..]133;D;0..] |
 | 00020  31 33 33 3b 41 07 1b 5d  31 33 33 3b 6b 3b 65 6e  133;A..]133;k;en |
```

Isolating the marker, its byte-exact form is **exactly 10 bytes** — `ESC ] 1 3 3 ; D ; 0 BEL`:

```
$ python3 -c "b=b'\x1b]133;D;0\x07'; print('repr       :',repr(b)); \
print('hex        :',' '.join(f'{x:02x}' for x in b)); \
print('cat -v form: ^[]133;D;0^G'); print('len        :',len(b))"
repr       : b'\x1b]133;D;0\x07'
hex        : 1b 5d 31 33 33 3b 44 3b 30 07
cat -v form: ^[]133;D;0^G
len        : 10
```

**Byte-count note (correcting a naive capture):** the D-marker is *immediately* followed by the next prompt's `A` marker — at offset `0x1e` the bytes `1b 5d 31 33 33 3b 41 07` = `\x1b]133;A\x07` begin. bash emits them back-to-back because its `PS1` appends `\e]133;D;$?\a\e]133;A\a` in one string (`shell-integration/bash/kitty.bash:239`, quoted below). A capture that naively reads "up to the next ESC" therefore appends one trailing `\x1b` — the **leading byte of the following `A` marker**, not part of `D;0`. The true `D;0` transport is the 10 bytes above; the stray 11th `\x1b` is the start of `A`.

**Corroboration (also a real shell, PTY tee).** A `bash` started with kitty's exact shell-integration environment (`ENV=/app/shell-integration/bash/kitty.bash KITTY_SHELL_INTEGRATION=enabled KITTY_BASH_INJECT=1 TERM=xterm-kitty bash --posix -i`) was teed to a file; the D-marker it wrote is byte-identical (`repr: b'\x1b]133;D;0\x07'`, `len : 10`), and the bytes immediately after it are again `b'\x1b]133;A\x07'`.

The ordered OSC 133 markers around the `echo hello` command, extracted in file order from the same canonical `write()` capture (C markers written to fd 1, D/A markers embedded in the fd 2 prompt blocks):

```
$ cat q8_timeline_canonical.txt
133;D;0             startup prompt: previous-command status
133;A               startup prompt start
133;C;cmdline=echo\ hello   echo hello command-output start   (write(1,...,28))
133;D;0             echo hello FINISHED, status 0             (in write(2,...,174))
133;A               next prompt start
133;C;cmdline=exit  exit command-output start                 (write(1,...,21))
```

Reading it: a `D;0` closes the previous prompt, `A` starts a new prompt, `C;cmdline=echo\ hello` marks command-output start, then `D;0` reports the command finished with status `0`. With shell integration **disabled at the kitty level**, **zero** `133` markers are written on the wire (the negative control from the Q3b matrix: the disabled case has `C=0, D;0=0`).

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

OSC 133 is not a formally standardized escape sequence; it is a **de-facto cross-vendor convention** that originated with the FinalTerm terminal and was popularized by [iTerm2's shell integration](https://iterm2.com/documentation-shell-integration.html). The sequences mark the semantic structure of a shell session — prompt start (`A`), prompt end / command start (`B`), command-output start (`C`), and command finished (`D`) — as described in [kitty's own shell-integration documentation](https://sw.kovidgoyal.net/kitty/shell-integration/). The command-finished marker carries an optional exit code; [VS Code's terminal shell-integration documentation](https://code.visualstudio.com/docs/terminal/shell-integration) defines it as `OSC 133 ; D [; <exitcode>] ST` (mark execution finished with an optional exit code), and VS Code's shell-integration source records that this marker is "Based on FinalTerm's `OSC 133 ; D [; <ExitCode>] ST`" ([microsoft/vscode#155639](https://github.com/microsoft/vscode/issues/155639)). When the exit code is present, [Windows Terminal's FTCS shell-integration documentation](https://learn.microsoft.com/en-us/windows/terminal/tutorials/shell-integration) specifies that the terminal treats `0` as success and any non-zero value as an error. The convention has been adopted across vendors — including [iTerm2](https://iterm2.com/documentation-shell-integration.html), [VS Code](https://code.visualstudio.com/docs/terminal/shell-integration), and [Ghostty](https://ghostty.org/docs/config/keybind/reference) — and kitty is explicitly among the implementers (its bash/zsh/fish scripts emit the `D;<status>` marker, cited above from the in-repo shell-integration files). The OSC command numbers involved (`133`, together with iTerm2's proprietary `1337`) are vendor-assigned extensions that are **not** registered with any escape-sequence standards body; they remain the subject of the still-open [freedesktop terminal-wg specifications discussion](https://gitlab.freedesktop.org/terminal-wg/specifications/-/issues/28) rather than a ratified standard.

### Rationale

Unlike the direct-child path (Q6/Q7), where the OS carries only a raw wait-status with no escape sequence, the shell-integration path is what carries a human-meaningful **command** status: the shell appends `\e]133;D;$?\a` (bash), `\e]133;D;$cmd_status\a` (zsh), or `\e]133;D;$status\a` (fish) after each command; kitty parses it (`vt-parser.c` → `screen.c`) and delivers the status string to `handle_cmd_end()`. This is why questions 5–8 concern the OSC 133 transport, while 6–7 concern the OS `SIGCHLD`/`wait4` mechanism.

## Q9 — Where the child's printed output appears

### Answer: in kitty's on-screen terminal grid (the kitty window) **[observed]**

The child's stdout is read from the **PTY master**, parsed, written into the window's `Screen` grid, and rendered on screen. It does **not** appear on kitty's own stdout/stderr.

### Raw output — the child's bytes reach kitty via the PTY master **[observed]**

From the Q1 strace (monitor thread TID `8890` reading the PTY master fd 8):

```
8890  read(8, "hello\r\nworld\r\n", 1048576) = 14
```

### Raw output — the live kitty window renders the child's text (PRIMARY, canonical) **[observed]**

Running the **real launcher** under the headless X server, holding the frame with a trailing `sleep` so an **external** screenshot can be taken of the live GPU-rendered window, then reaping it (the child still exits 0):

```
$ ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; sleep 3' &
$ sleep 1.5; import -window root "$EVID/q9_grid.png"      # ImageMagick captures the X root window
$ wait; echo "grid-run kitty_exit=$?"
grid-run kitty_exit=0
$ identify "$EVID/q9_grid.png"
$EVID/q9_grid.png PNG 1280x800 1280x800+0+0 8-bit Grayscale Gray 256c 1772B 0.000u 0:00.000
```

The captured window is objectively confirmed to contain the child's text two independent ways — no human "inspection" is relied upon:

```
$ convert "$EVID/q9_grid.png" -crop 400x120+0+0 +repage "$EVID/q9_grid_crop.png"
$ tesseract "$EVID/q9_grid_crop.png" stdout 2>/dev/null | sed '/^[[:space:]]*$/d'
hello
world
$ convert "$EVID/q9_grid_crop.png" -depth 8 -format '%c' histogram:info:- | sort -t: -k1 -rn | head -3
     47443: (0,0,0) #000000 gray(0)
       163: (204,204,204) #CCCCCC gray(204)
        27: (173,173,173) #ADADAD gray(173)
```

OCR (`tesseract`) reads exactly `hello` and `world` from the top-left grid region, and the crop's gray-level histogram is bimodal — a black background (`#000000`, 47443 px) plus light-gray anti-aliased glyph pixels peaking at `#CCCCCC` (kitty's default foreground, gray 204) — i.e. two lines of light-gray monospace text on black. This is the canonical proof for Q9: real launcher, real PTY, real `Screen`, real GPU render, captured by an external tool from the live window. (The trailing `sleep 3` only holds the rendered frame long enough to screenshot; the child still exits 0, as `grid-run kitty_exit=0` shows.)

### Cross-check (NON-CANONICAL) — the same bytes fill kitty's real `Screen` buffer **[observed, non-canonical]**

As an in-process corroboration — **not** the canonical launch path, because it injects bytes directly into a `Screen` object instead of going through a child spawned on a PTY — feeding `hello\r\nworld\r\n` through kitty's **real** VT parser and `Screen` (via `kitty.fast_data_types.Screen` and the test helper `parse_bytes`, run under `kitty +launch`) fills the grid identically:

```
$ ./kitty/launcher/kitty +launch "$EVID/screen_dump.py"
$ cat "$EVID/q9_screen_dump.txt"
grid line 0: 'hello'
grid line 1: 'world'
grid line 2: ''
```

This confirms the parser/`Screen` machinery maps the bytes to grid cells `hello` / `world`, corroborating the on-screen capture above. It is labeled non-canonical because it bypasses the child/PTY spawn.

### Raw output — the child's text is NOT on kitty's own stdout/stderr **[observed]**

```
$ cat "$EVID/q9_negative.txt"
kitty_exit=0
--- kitty own stdout: byte count ---
0
--- kitty own stderr: contents ---
[0.167] Failed to open systemd user bus with error: No medium found
--- count of child lines (hello|world) in kitty own stdout+stderr ---
own_stdout:0
own_stderr:0
```

Kitty's own stdout is **empty (0 bytes)**; its stderr contains only a benign, unrelated systemd-bus warning; and grepping both for `hello`/`world` yields **0** matches. The child's output is thus confined to the terminal grid.

### Code citation **[source-derived]**

The PTY master read is `read_bytes()` [`kitty/child-monitor.c:1345`] (full function quoted under Q1), which commits the bytes into the VT parser write buffer; `do_parse()` [`kitty/child-monitor.c:438`] then feeds the parser into the `Screen`, and `prepare_to_render_os_window()`/`send_cell_data_to_gpu()` [`kitty/child-monitor.c:705,714`] upload the grid cells to the GPU for drawing.

### Rationale

`echo hello; echo world` writes to the child's stdout, which is the PTY **slave** (wired by `dup2` in `spawn`, Q1). Kitty reads the **master** end, so the bytes enter kitty's parser and become grid cells rather than propagating to kitty's own standard streams. The OCR-and-histogram-verified screenshot of the live window (primary), the empty own-stdout with zero `hello`/`world` matches (negative proof), and the real-`Screen` cross-check together confirm the output lands in the kitty window's terminal grid — and nowhere on kitty's own standard streams.


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
### DISCRIMINATING close_on_child_death test ###
# child 'echo hi; setsid -f sleep 4; exit 0': the foreground sh exits 0 immediately,
# but the detached 'sleep 4' keeps the PTY slave open (PTY master EOF only at ~4 s).

# DEFAULT close_on_child_death=no  ->  window LINGERS until PTY EOF (~4 s):
$ t0=$(date +%s.%N); ./kitty/launcher/kitty --config NONE \
      sh -c 'echo hi; setsid -f sleep 4; exit 0' >/dev/null 2>&1; rc=$?; t1=$(date +%s.%N); \
      printf 'default(no): kitty_exit=%s  wall=%ss\n' "$rc" "$(awk "BEGIN{printf \"%.2f\",$t1-$t0}")"
default(no): kitty_exit=0  wall=4.28s

# close_on_child_death=yes  ->  window closes IMMEDIATELY on the SIGCHLD reap (~0.3 s):
$ t0=$(date +%s.%N); ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
      sh -c 'echo hi; setsid -f sleep 4; exit 0' >/dev/null 2>&1; rc=$?; t1=$(date +%s.%N); \
      printf 'yes:         kitty_exit=%s  wall=%ss\n' "$rc" "$(awk "BEGIN{printf \"%.2f\",$t1-$t0}")"
yes:         kitty_exit=0  wall=0.26s
```

With the default (`no`), kitty lingers **4.28 s** — until the detached writer closes the PTY (EOF) — proving teardown is PTY-EOF-driven. With `close_on_child_death=yes`, kitty closes in **0.26 s** — immediately on the `SIGCHLD` reap via `mark_child_for_removal`. Both exit `0`. (For a child with no lingering writer, EOF coincides with exit, so the difference is invisible; the detached-writer case is what separates the two mechanisms.)

### Nuance 2 — The completion notification is off by default, and the duration term gates on kitty's uptime

`notify_on_cmd_finish` defaults to `never` (`kitty/options/types.py:560` → `NotifyOnCmdFinish(when='never', duration=5.0, …)`; `kitty/options/definition.py:3190` → `opt('notify_on_cmd_finish', 'never', …)`), so **by default no completion notification is shown** even with shell integration active — confirmed by the Q3b matrix (`enabled_default`: OSC 133 received, `Notify=0`). Separately, as documented under Q5, the `duration` term does **not** measure the command's own runtime: `last_cmd_output_start_time` is zeroed (`kitty/window.py:1411`) *before* the subtraction (`kitty/window.py:1417`), so `last_cmd_output_duration == monotonic()` — and kitty's `monotonic()` (`kitty/monotonic.h:60-68`, imported at `kitty/window.py:44`/`:75`) is **seconds since kitty started**. The gate therefore fires only when `when != 'never'` **and** kitty has been running at least `duration` seconds; with the default `duration = 5.0` a command finishing <5 s after launch is gated out (observed: `always 5` EARLY → `Notify=0` ×3; LATE → `Notify=1` ×3; see Q5). This is an **apparent pre-existing behavior** in the investigated source (the term keys off kitty's uptime rather than the command's runtime) and is documented, not changed.

## Observed vs. inferred classification

Every claim in this document is one of: **runtime-observed** (captured from a live run), **source-derived** (read directly from cited source, and consistent with observed behavior), or **corroborated** (external references confirming an in-repo fact). No claim is purely speculative; nothing required an unverifiable inference.

| Claim | Classification | Evidence |
|---|---|---|
| Kitty's own exit code is `0`, stable, child-status-independent | runtime-observed | Q2 (inline 5-run loop + child 5/42); Reproducibility §2 |
| The child exits and delivers `SIGCHLD` (signal 17, `CLD_EXITED`) | runtime-observed | Q6 strace `signalfd` + siginfo decode; Reproducibility §3 |
| Kitty reaps with `wait4(-1,&status,WNOHANG)` | runtime-observed | Q7 strace `wait4(…)` returning the child PID; Reproducibility §3 |
| `+hold` banner is `Press Enter or Esc to exit` (bold-green), stable ×2 | runtime-observed | Q3a PTY capture (176 bytes, ×2 byte-identical); Reproducibility §5 |
| `--hold` flag shows an interactive shell (`KITTY_HOLD=1`), no banner | runtime-observed | Q3 `--hold` subsection (`KITTY_HOLD=1` + OCR screenshot) |
| Notification body `Command echo hello finished with status: 0.\nClick to focus.` | runtime-observed | Q3b D-Bus body; Reproducibility §7 |
| Notification off by default; fires only when `when != never` (and kitty uptime ≥ `duration`) | runtime-observed | Q3b matrix (`enabled_default` Notify=0); Reproducibility §8 |
| OSC 133 `D;0` bytes `\e]133;D;0\a` (10 bytes) written by the real shell | runtime-observed | Q8 strace of bash `write()` (10-byte `D;0`); Reproducibility §6 |
| Child output `hello`/`world` lands in the `Screen` grid | runtime-observed | Q9 screen dump + grid screenshot (OCR); Reproducibility §4 |
| Child output is NOT on kitty's own stdout/stderr | runtime-observed | Q9 negative check (own stdout/stderr 0 bytes); Reproducibility §4 |
| Default teardown is PTY-EOF (lingers with detached writer); `=yes` reaps immediately | runtime-observed | Nuance 1 discriminating test (4.28 s vs 0.26 s) |
| `ChildMonitor` owns id/PID/PTY-fd/Screen | source-derived | `kitty/boss.py:370-374, 585-587`; `kitty/child-monitor.c:65-71` (Child struct), `305-317` (add_child parse/store) |
| `handle_cmd_end()` builds the message | source-derived (+ observed body) | `kitty/window.py:1408-1451`; Q3b observed body |
| `on_child_death(window_id)` carries no status; `mark_monitored_pids` no-op for primary child | source-derived | `kitty/boss.py:881`; `kitty/child-monitor.c:1397-1410` |
| Duration gate is a functional kitty-uptime threshold (`when != never` **and** kitty uptime ≥ `duration`, due to zeroed-before-subtract) | runtime-observed (+ source-derived) | `kitty/window.py:1411,1416,1417,1425`; `kitty/monotonic.h:60-68`; Reproducibility §8 (`always 5` EARLY→Notify=0, LATE→Notify=1) |
| kitty's `monotonic()` is seconds since kitty started (≈0 at startup, +2 s after a 2 s sleep), not since boot | runtime-observed | `+runpy` measurement ×3; `kitty/monotonic.h:60-68` |
| OSC 133 is a de-facto cross-vendor convention (FinalTerm origin, exit code optional) | corroborated | external references — iTerm2, VS Code, kitty, Ghostty (URLs under Q8) |

## Reproducibility (self-contained)


Every command below was executed **verbatim in a clean instance of the approved container** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (Ubuntu 24.04.2 LTS, Python 3.12.3, Go 1.23.4). The suite is **self-contained**: it installs its own observation prerequisites, works entirely inside a private `mktemp -d` directory (never a fixed path), picks a **unique** X display per run, runs under `set -euo pipefail` with explicit assertions, and tears everything down with a cleanup trap — so nothing leaks transient identifiers and no step can report false success. Running the whole suite start-to-finish, every step printed its `[n] OK` marker and exited `0`, and each generated artifact matched the value reported in the answers above. The scripts are complete — no elisions.

### 0. Prerequisites and safe headless environment

```bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive
apt-get update -qq
apt-get install -y --no-install-recommends \
  xvfb libgl1-mesa-dri strace dbus-x11 dunst libnotify-bin \
  x11-utils xauth imagemagick tesseract-ocr libxtst6 >/dev/null
export KITTY_REPO=/app                        # repo root at the investigated commit
export WORK="$(mktemp -d)"; EV="$WORK/evidence"; mkdir -p "$EV"
export LANG=C.UTF-8 LC_ALL=C.UTF-8            # the only UTF-8 locale in the image
export DISPLAY=":$((90 + RANDOM % 100))" LIBGL_ALWAYS_SOFTWARE=1   # unique display
Xvfb "$DISPLAY" -screen 0 1280x800x24 -nolisten tcp >"$WORK/xvfb.log" 2>&1 &
XVFB_PID=$!
eval "$(dbus-launch --sh-syntax)"             # session bus for the notification steps (7, 8)
DUNST_PID=""
trap 'kill "$XVFB_PID" "${DBUS_SESSION_BUS_PID:-}" "${DUNST_PID:-}" 2>/dev/null || true; rm -rf "$WORK"' EXIT
for _ in $(seq 1 50); do xdpyinfo -display "$DISPLAY" >/dev/null 2>&1 && break; sleep 0.1; done
cd "$KITTY_REPO"
echo "[0] OK  DISPLAY=$DISPLAY"
```

### 1. Canonical default build

```bash
export PATH="/usr/local/go/bin:$PATH" CI=true
python3 --version                             # -> Python 3.12.3
go version                                    # -> go version go1.23.4 linux/amd64
set +e; make clean >/dev/null 2>&1; make >"$EV/make_full.log" 2>&1; MK=$?; set -e
echo "MAKE_EXIT=$MK"
[ "$MK" -eq 0 ] || { echo "ASSERT FAIL: build"; tail -30 "$EV/make_full.log"; exit 1; }
test -x ./kitty/launcher/kitty || { echo "ASSERT FAIL: launcher"; exit 1; }
./kitty/launcher/kitty --version              # -> kitty 0.35.2 created by Kovid Goyal
echo "[1] build OK"
```

### 2. Kitty's own exit code (Q2)

```bash
allzero=1
for i in 1 2 3 4 5; do
  set +e; ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1; rc=$?; set -e
  echo "run $i: kitty_exit=$rc"; [ "$rc" -eq 0 ] || allzero=0
done
[ "$allzero" -eq 1 ] || { echo "ASSERT FAIL: Q2"; exit 1; }
echo "[2] Q2 OK"
```

### 3. Signal + reap trace (Q6/Q7) with a reproducible siginfo decode

```bash
set +e
strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -e read=all \
  -o "$EV/kitty.strace" \
  ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1
set -e
test -s "$EV/kitty.strace" || { echo "ASSERT FAIL: no strace"; exit 1; }
grep -q 'wait4' "$EV/kitty.strace" || { echo "ASSERT FAIL: no wait4"; exit 1; }
grep -Eq 'signalfd|SIGCHLD' "$EV/kitty.strace" || { echo "ASSERT FAIL: no signalfd/SIGCHLD"; exit 1; }
```

`decode_siginfo.py` decodes the exact `struct signalfd_siginfo` that kitty read from its signal fd, by parsing it **out of the trace** (matching the payload whose `ssi_signo` is `17`); it hardcodes no run-specific value, so the per-run child PID is reported, not baked in:

```python
import re, struct, sys
def unescape(s):
    out = bytearray(); i = 0
    while i < len(s):
        c = s[i]
        if c == '\\':
            n = s[i+1]
            if n == 'x':
                out.append(int(s[i+2:i+4], 16)); i += 4; continue
            if n in '01234567':
                j = i+1; o = ''
                while j < len(s) and len(o) < 3 and s[j] in '01234567':
                    o += s[j]; j += 1
                out.append(int(o, 8) & 0xff); i = j; continue
            simple = {'n':10,'r':13,'t':9,'a':7,'b':8,'f':12,'v':11,'\\':92,'"':34,"'":39,'0':0}
            out.append(simple.get(n, ord(n))); i += 2; continue
        out.append(ord(c)); i += 1
    return bytes(out)
raw = None
for ln in open(sys.argv[1], errors='replace'):
    m = re.search(r'read\(\d+, "((?:[^"\\]|\\.)*)"(?:\.\.\.)?, \d+\)\s*= 128', ln)
    if m:
        b = unescape(m.group(1))
        if len(b) >= 16 and b[0] == 17:      # ssi_signo == SIGCHLD (17)
            raw = b[:16]; break
if raw is None:
    sys.exit("SIGCHLD siginfo not found in trace")
signo, errno_, code, pid = struct.unpack('<IiiI', raw)
names = {17: 'SIGCHLD'}; codes = {1: 'CLD_EXITED'}
print(f"ssi_signo = {signo}  -> {names.get(signo,'?')}")
print(f"ssi_errno = {errno_}")
print(f"ssi_code  = {code}   -> {codes.get(code,'?')}")
print(f"ssi_pid   = {pid}   (the child PID; varies per run)")
```

```bash
python3 "$WORK/decode_siginfo.py" "$EV/kitty.strace" | tee "$EV/siginfo_decode.txt"
grep -q 'SIGCHLD'    "$EV/siginfo_decode.txt" || { echo "ASSERT FAIL: decode signo"; exit 1; }
grep -q 'CLD_EXITED' "$EV/siginfo_decode.txt" || { echo "ASSERT FAIL: decode code"; exit 1; }
echo "[3] Q6/Q7 OK"
```

### 4. On-screen grid proof (Q9): primary screenshot + negative proof + non-canonical cross-check

`screen_dump.py` (the **non-canonical** in-process cross-check — it injects bytes straight into a real `Screen`, bypassing the child/PTY spawn):

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
# (a) PRIMARY canonical proof: the real launcher renders the child's output into the
#     on-screen grid; a trailing sleep holds the frame so an EXTERNAL tool can screenshot it.
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; sleep 3' >/dev/null 2>&1 &
KPID=$!
sleep 1.5
import -window root "$EV/q9_grid.png" 2>/dev/null || { echo "ASSERT FAIL: screenshot"; exit 1; }
wait "$KPID"; echo "grid-run kitty_exit=$?"          # child still exits 0
test -s "$EV/q9_grid.png" || { echo "ASSERT FAIL: no PNG"; exit 1; }
identify "$EV/q9_grid.png"
convert "$EV/q9_grid.png" -crop 400x120+0+0 +repage "$EV/q9_grid_crop.png"
tesseract "$EV/q9_grid_crop.png" stdout 2>/dev/null | sed '/^[[:space:]]*$/d' | tee "$EV/q9_ocr.txt"
grep -qi hello "$EV/q9_ocr.txt" || { echo "ASSERT FAIL: OCR !hello"; exit 1; }
grep -qi world "$EV/q9_ocr.txt" || { echo "ASSERT FAIL: OCR !world"; exit 1; }
# (b) NEGATIVE proof: the child's text is NOT on kitty's own stdout/stderr.
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >"$EV/q9_own_stdout.txt" 2>"$EV/q9_own_stderr.txt"
ob=$(wc -c < "$EV/q9_own_stdout.txt"); echo "own_stdout_bytes=$ob"
[ "$ob" -eq 0 ] || { echo "ASSERT FAIL: own stdout not empty"; exit 1; }
set +e; hs=$(grep -c -E 'hello|world' "$EV/q9_own_stdout.txt"); set -e
echo "own_stdout_hello_world=$hs"; [ "$hs" -eq 0 ] || { echo "ASSERT FAIL: child text on own stdout"; exit 1; }
# (c) NON-CANONICAL cross-check: the same bytes fill kitty's real Screen buffer.
./kitty/launcher/kitty +launch "$WORK/screen_dump.py" > "$EV/q9_screen_dump.txt" 2>&1 || true
cat "$EV/q9_screen_dump.txt"
grep -q "grid line 0: 'hello'" "$EV/q9_screen_dump.txt" || { echo "ASSERT FAIL: screen cross-check"; exit 1; }
echo "[4] Q9 OK"
```

### 5. `+hold` banner (Q3a) via a PTY

`hold_capture.py` runs `kitty +hold` twice under a PTY, captures the **complete** lifecycle output (dismissing the kitten with the keyboard-protocol CSI-u encoding so it emits its finalization tail), and reports the byte length, banner presence, and byte-for-byte stability across the two runs:

```python
import os, pty, select, time, sys
BANNER = b"\x1b[1;32mPress Enter or Esc to exit\x1b[m"
def capture(argv, dismiss=b"\x1b[13u", hard_cap=10.0):
    pid, fd = pty.fork()
    if pid == 0:
        os.execvp(argv[0], argv); os._exit(127)
    buf = b""; start = time.time(); dismissed = False; last = time.time()
    while time.time() - start < hard_cap:
        r, _, _ = select.select([fd], [], [], 0.4)
        if r:
            try: d = os.read(fd, 65536)
            except OSError: break
            if not d: break
            buf += d; last = time.time()
        else:
            if not dismissed and buf:
                os.write(fd, dismiss); dismissed = True; last = time.time()
            elif dismissed and time.time() - last > 1.0:
                break
    try: os.waitpid(pid, 0)
    except OSError: pass
    return buf
argv = ["./kitty/launcher/kitty", "+hold", "sh", "-c", "echo hello; echo world; exit 0"]
r1 = capture(list(argv)); r2 = capture(list(argv))
open(sys.argv[1], "wb").write(r1)
print("hold_len_run1=%d" % len(r1))
print("hold_len_run2=%d" % len(r2))
print("banner_present=%s" % (BANNER in r1))
print("byte_identical_run1_run2=%s" % (r1 == r2))
```

```bash
python3 "$WORK/hold_capture.py" "$EV/hold1.bin" | tee "$EV/hold_summary.txt"
grep -q 'banner_present=True'            "$EV/hold_summary.txt" || { echo "ASSERT FAIL: hold banner"; exit 1; }
grep -q 'byte_identical_run1_run2=True'  "$EV/hold_summary.txt" || { echo "ASSERT FAIL: hold not stable"; exit 1; }
grep -q 'hold_len_run1=176'              "$EV/hold_summary.txt" || { echo "ASSERT FAIL: hold length != 176"; exit 1; }
echo "[5] Q3a OK"
```

### 6. Canonical OSC 133 `D;0` on the wire (Q8)

`xtype.py` injects real keystrokes via the X11 `XTEST` extension — the canonical user-input path, driving the integrated shell exactly as a user's keyboard would:

```python
import ctypes, sys, time
X11 = ctypes.CDLL("libX11.so.6"); XTST = ctypes.CDLL("libXtst.so.6")
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
    X11.XFlush(ctypes.c_void_p(dpy)); time.sleep(0.03)
SPECIAL = {" ": "space", "\n": "Return", "-": "minus"}
def typ(s):
    for ch in s:
        if ch in SPECIAL: tapsym(keysym(SPECIAL[ch]))
        elif ch.isupper(): tapsym(keysym(ch), shift=True)
        else: tapsym(keysym(ch))
if __name__ == "__main__":
    time.sleep(float(sys.argv[1])); typ(sys.argv[2])
```

The canonical proof of the transport is the shell's own `write()` of the `D;<status>` bytes, captured with `strace`:

```bash
strace -f -e trace=write -e write=all -o "$EV/kitty_osc.strace" \
  ./kitty/launcher/kitty --config NONE bash >"$EV/kitty_osc_run.log" 2>&1 &
KP=$!
sleep 3.5
python3 "$WORK/xtype.py" 0 $'echo hello\n'      # real keystrokes drive the integrated shell
sleep 1.5
python3 "$WORK/xtype.py" 0 $'exit\n'
sleep 2
kill -9 "$KP" 2>/dev/null || true
grep -aq ']133;D;0' "$EV/kitty_osc.strace" || { echo "ASSERT FAIL: no OSC D;0 write"; exit 1; }
grep -a ']133;D;0' "$EV/kitty_osc.strace" | head -1 || true
echo "[6] Q8 OSC D;0 OK"
```

### 7. Visible completion notification (Q3b/Q5) via a real notification daemon

`dunst` is started as the `org.freedesktop.Notifications` server on the session bus; a real interactive `bash` is driven inside kitty with `notify_on_cmd_finish always 0`; the fired notification is confirmed three independent ways — the daemon's own `GetServerInformation`, an external screenshot plus `dunstctl count` (`Currently displayed: 1`), and the byte-exact `Notify` body captured over D-Bus:

```bash
dunst >"$EV/dunst.log" 2>&1 &
DUNST_PID=$!
sleep 1.2
gdbus call --session --dest org.freedesktop.Notifications \
  --object-path /org/freedesktop/Notifications \
  --method org.freedesktop.Notifications.GetServerInformation 2>&1 | tee "$EV/notif_server.txt" || true
dbus-monitor "interface='org.freedesktop.Notifications',member='Notify'" >"$EV/notify_body.txt" 2>/dev/null &
DBMON=$!
./kitty/launcher/kitty --config NONE -o "notify_on_cmd_finish always 0" bash >"$EV/kitty_notif_run.log" 2>&1 &
KP=$!
sleep 3.5
python3 "$WORK/xtype.py" 0 $'echo hello\n'
sleep 2.5
import -window root "$EV/q3b_notif_render.png" 2>/dev/null && echo "notif shot ok" || true
dunstctl count | tee "$EV/dunstctl_count.txt" || true
python3 "$WORK/xtype.py" 0 $'exit\n'
sleep 1
kill -9 "$KP" "$DBMON" 2>/dev/null || true
grep -aq 'finished with status: 0' "$EV/notify_body.txt" || { echo "ASSERT FAIL: no Notify body"; exit 1; }
sed -n '/member=Notify/,/Click to focus/p' "$EV/notify_body.txt" | head -20 || true
kill -9 "$DUNST_PID" 2>/dev/null || true; DUNST_PID=""
echo "[7] Q3b/Q5 notification OK"
```

### 8. Duration-gate matrix (Q5 / Nuance 2)

`dgate_case.sh` is the per-case worker (run inside a per-case `dbus-run-session`, `DISPLAY` already exported). It counts `Notify` D-Bus calls with `dbus-monitor` and OSC 133 `D;0` markers with `--dump-bytes` — the latter used **only** as a non-canonical per-run marker counter (the canonical byte-exact proof of the markers is the `strace write()` capture in step 6):

```bash
cat > "$WORK/dgate_case.sh" <<'EOS'
#!/usr/bin/env bash
set -uo pipefail
cd /app
dunst >/dev/null 2>&1 & DN=$!
mon="$(mktemp)"; dump="$(mktemp)"
dbus-monitor "interface='org.freedesktop.Notifications',member='Notify'" >"$mon" 2>/dev/null & MON=$!
sleep 0.8
timeout 45 ./kitty/launcher/kitty --config NONE -o "notify_on_cmd_finish $WHEN" --dump-bytes "$dump" bash >/dev/null 2>&1 & KP=$!
sleep "$INJECT"
python3 "$WORK/xtype.py" 0 $'echo hello\n'
sleep 3
d0=$(grep -c '133;D;0' "$dump" 2>/dev/null || true); d0="${d0:-0}"
nt=$(grep -c 'member=Notify' "$mon" 2>/dev/null || true); nt="${nt:-0}"
printf '%-14s inject@%-4s OSC133_D0=%s  Notify_total=%s\n' "$LABEL" "$INJECT" "$d0" "$nt" >> "$EV/dgate_matrix.txt"
if [ -n "$BODYOUT" ] && [ "$nt" -ge 1 ]; then sed -n '/member=Notify/,/int32 -1/p' "$mon" > "$EV/$BODYOUT"; fi
if [ "$LABEL" = "early_1" ]; then cp "$mon" "$EV/dbus_early.log"; fi
kill -9 "$KP" "$MON" "$DN" 2>/dev/null || true
EOS
```

The orchestrator gives each case its own `Xvfb` and session bus, varying only **when** `echo hello` finishes relative to kitty start:

```bash
MATRIX="$EV/dgate_matrix.txt"; : > "$MATRIX"
run_dgate () {  # label  when  inject_delay  [bodyout]
  local label="$1" when="$2" inject="$3" bodyout="${4:-}"
  local dpy=":$((90 + RANDOM % 100))"
  Xvfb "$dpy" -screen 0 1280x800x24 -nolisten tcp >/dev/null 2>&1 &
  local xv=$!; sleep 1.0
  DISPLAY="$dpy" WORK="$WORK" EV="$EV" LABEL="$label" WHEN="$when" INJECT="$inject" BODYOUT="$bodyout" \
    dbus-run-session -- bash "$WORK/dgate_case.sh"
  kill -9 "$xv" 2>/dev/null || true; sleep 0.4
}
echo "===== notify_on_cmd_finish always 5 (NON-ZERO, = default duration) =====" >> "$MATRIX"
echo "--- EARLY: command finishes <5s after kitty start -> expect gated (Notify=0) ---" >> "$MATRIX"
run_dgate early_1 "always 5" 2.5 ""
run_dgate early_2 "always 5" 2.5 ""
echo "--- LATE: command finishes >5s after kitty start -> expect fires (Notify=1) ---" >> "$MATRIX"
run_dgate late_1 "always 5" 8 "dbus_late.log"
run_dgate late_2 "always 5" 8 ""
echo "===== CONTROL: always 100 + EARLY -> expect gated (Notify=0) =====" >> "$MATRIX"
run_dgate dg100_early "always 100" 2.5 ""
echo "===== CONTROL: always 0 + EARLY (old circular method) -> Notify=1 =====" >> "$MATRIX"
run_dgate a0_early "always 0" 2.5 ""
cat "$MATRIX"
grep -q 'early_1        inject@2.5  OSC133_D0=2  Notify_total=0' "$MATRIX" || { echo "ASSERT FAIL: EARLY not gated"; exit 1; }
grep -q 'late_1         inject@8    OSC133_D0=2  Notify_total=1' "$MATRIX" || { echo "ASSERT FAIL: LATE not fired"; exit 1; }
echo "[8] Q5 duration-gate OK"
```

### 9. Cleanup and repository-unchanged verification

```bash
rm -rf "$WORK"             # remove the private temp dir (and only it)
git -C "$KITTY_REPO" status --porcelain    # clean once the answer document is committed
git -C "$KITTY_REPO" diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD   # sole delta vs the investigated source
```

Once the answer document is committed, the working tree is clean and the only difference from the investigated source commit is this one file (the durable, HEAD-independent check; while the document is still being edited, `git status --porcelain` instead shows it as modified/untracked):

```
$ git status --porcelain
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md
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
- [x] `close_on_child_death=yes` — immediate close (discriminating 0.26 s vs 4.28 s).
- [x] `kitty +hold` — green banner, ×2 byte-identical (stability).
- [x] `kitten __hold_till_enter__` — same banner (non-canonical corroboration).
- [x] `kitty --hold` flag — interactive shell, `KITTY_HOLD=1`, no banner.
- [x] Shell integration enabled + `notify_on_cmd_finish always` — OSC 133 + one notification.
- [x] Shell integration disabled (kitty level) — no OSC 133, no notification (negative control).
- [x] Shell integration enabled + default `notify_on_cmd_finish=never` — OSC 133 present, no notification (gate control).
- [x] Duration gate with non-zero `duration` (`always 5`) — EARLY finish `Notify=0` ×3, LATE finish `Notify=1` ×3; `always 100`+EARLY gated; `always 0`+EARLY fires (circular) — proves a functional kitty-uptime threshold.
- [x] Nonzero child status controls — child exit 5 and 42 → kitty exit 0.
- [x] Before/during/after markers — the OSC 133 `D→A→C→D` timeline.
- [x] Kitty's own stdout/stderr negative check — 0 bytes / benign only.
