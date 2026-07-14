# How Kitty Turns a Surge of Raw Child Output into Coherent Screen & Application State

*A runtime-grounded investigation of the terminal-interaction pipeline, with special attention to pause → resume.*

Source branch: `kitty_815df1e210e0` · HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config").

---

## 0. Scope & Method

This document answers five questions about how the [kitty](https://sw.kovidgoyal.net/kitty/) terminal emulator ingests a burst of bytes from a child process and turns it into coherent on-screen and application state:

- **Q1** — Where does raw input *first enter the system*, and what happens when a session is *paused then resumed*?
- **Q2** — The *"unseen conductor"*: how are timing, ordering, and state hand-offs split up, and *what decides which event gets handled first*?
- **Q3** — When shell-integration hints arrive *mixed in with ordinary text*, how does everything stay aligned *without drifting out of sync*?
- **Q4** — Does behavior differ under *heavy backpressure or an unstable remote connection*?
- **Q5** — *What really happens end-to-end from the moment mixed input arrives to the moment the interface settles again, and how do the moving parts keep their rhythm?*

### 0.1 Method: run-first, observe-then-write

Every behavioral claim below is written **from directly observed runtime output** captured through the **real PTY path** — a genuine child process writing into a real pseudo-terminal that kitty reads. Each claim sits **beside the exact command that produced it and that command's unedited output**, and each *structural* claim (how the code is wired) carries a `file:line` citation to the code that manifests it.

**What is deliberately NOT used as evidence.** The C unit-test hook `test_parse_written_data` [kitty/screen.c:4771-4772] and the Python `parse_bytes` harness [kitty_tests/parser.py:20] both feed bytes straight to the parser worker and **bypass the PTY read path**. They are therefore **non-canonical** for these questions and are never used as primary evidence; where a contrast value from them would appear, it is explicitly labeled *non-canonical*.

### 0.2 Build environment and designated image

**Toolchain (observed).** The pinned toolchain reported by the environment:

```
$ python3 --version
Python 3.13.7
$ go version
go version go1.24.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ pkg-config --version
1.8.1
```

**Designated image — attempted, access denied, honest fallback.** The task nominates the Docker image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (mirrored at `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). It is private; a direct pull is denied:

```
$ docker pull andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
Error response from daemon: pull access denied for andrewparkscaleai/coding-agent, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```

The build therefore runs directly against the environment's system libraries (a fully-supported, canonical `setup.py` build path — kitty ships no lockfile and builds from system `-dev` packages). This is disclosed here so no claim below depends on an unavailable image.

### 0.3 Canonical build command and its output

Kitty is a hybrid C / Python / Go application built by its own `setup.py`, not by a package manifest. The default `setup.py` action is `build` [setup.py:175], i.e. the `make all` target. The exact commands and their real output:

```
$ repo=/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e
$ cd "$repo"
$ source /tmp/kitty-venv/bin/activate
$ python3 setup.py
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
```

(An already-built tree recompiles incrementally, as above; a clean tree compiles the full source set. Wayland is auto-disabled because `wayland-protocols` is absent → x11-only, the canonical CI configuration on Ubuntu 25.10. Strict flags `-pedantic-errors -Werror -Wall -Wextra` stay on.) The launcher self-reports:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The launcher's own compile/link step (via the `build-launcher` action) is:

```
$ python3 setup.py build-launcher
[1/2] Compiling kitty/launcher/main.c ...
[2/2] Compiling kitty/launcher/single-instance.c ...
 done
[1/1] Linking launcher ...
 done
```

The build produces (all git-ignored): `kitty/launcher/kitty` (the launcher), `kitty/launcher/kitten`, and `kitty/fast_data_types.so`.

> **Correction — `python3 setup.py develop` semantics (the agent plan named `develop`).** `develop` does **not** "fail before doing any work." Its handler runs `build(args)` first [setup.py:2123] and only then `build_launcher(args, ..., bundle_type='develop')` [setup.py:2124]; the failure is at that *launcher* step. The real, unedited traceback:
>
> ```
> $ python3 setup.py develop
> Disabling building of wayland backend
> Traceback (most recent call last):
>   File ".../setup.py", line 2168, in main
>     do_build(args)
>   File ".../setup.py", line 2124, in do_build
>     build_launcher(args, launcher_dir=launcher_dir, bundle_type='develop')
>   File ".../setup.py", line 1255, in build_launcher
>     ph = os.path.relpath(os.environ["DEVELOP_ROOT"], '.')
> KeyError: 'DEVELOP_ROOT'
> ```
>
> `DEVELOP_ROOT` is set by kitty's Go dev-environment bootstrap `os.Setenv("DEVELOP_ROOT", root)` [bypy/devenv.go:368] — **not** by a `dev.sh` script — and is absent here. Nor are the two launchers identical: the `source` bundle sets only `-DFROM_SOURCE` with `KITTY_LIB_PATH = relpath('.', launcher_dir)` [setup.py:1253, :1263], whereas the `develop` bundle additionally defines `-DSET_PYTHON_HOME`, adds an `$ORIGIN/../../<root>/lib` rpath with `--disable-new-dtags`, and sets `KITTY_LIB_PATH = src_base` [setup.py:1254-1257, :1266-1267]. The canonical `python3 setup.py` (source) build is used for every observation below.

### 0.4 Canonical run form (headless) and the safe reproduction harness

A real GPU/display is unavailable in this container, so kitty runs under a virtual X display (`xvfb`). Mesa's `llvmpipe` provides software GL. This is still the **real** emulator driving a **real** PTY — only the display surface is virtual.

**Reproduction preamble (run once per shell).** Every command block below assumes this exact preamble has run; it fixes the repo root, the build's virtualenv, a clean `TMPDIR`, and a shorthand for the built launcher. No command uses the bare name `kitty` (which is not on `PATH`); the launcher is always `"$KITTY"` = `./kitty/launcher/kitty`.

```bash
repo=/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e
cd "$repo"
source /tmp/kitty-venv/bin/activate
export TMPDIR=/tmp/kitty-clean-tmp; mkdir -p "$TMPDIR"
export LANG=C.UTF-8 LC_ALL=C.UTF-8
KITTY=./kitty/launcher/kitty
```

**Foreground capture form.** For experiments where a child writes a fixed burst and kitty exits when it finishes, the run is bounded by `timeout`, and every temporary file lives in a private `mktemp -d` directory (mode 0700) removed by a `trap`. `--config NONE` guarantees the **canonical default option values** (`input_delay=3`, `repaint_delay=10`, `resize_debounce_time=(0.1, 0.5)` — confirmed in §Q5); `close_on_child_death=yes` makes kitty exit when the child finishes. The `--dump-commands` / `--dump-bytes` evidence lands on **stdout** (or the named dump file); container-only warnings such as `[0.159] Failed to open systemd user bus with error: Connection refused` appear on **stderr** and are discarded with `2>/dev/null` — the captured stdout is never edited. (Experiments that use `--debug-rendering` / `--debug-input` deliberately capture stderr, since that is where those flags print; each block shows exactly which stream it kept.)

```bash
tmpdir=$(mktemp -d); chmod 700 "$tmpdir"
trap 'rm -rf "$tmpdir"' EXIT
timeout 30 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
  -o close_on_child_death=yes --dump-commands sh -c 'printf "hi\n"' 2>/dev/null
```

**Remote-control (background) form.** Experiments that drive a live kitty from a second `kitty @` client launch kitty in the background under `timeout`, capture **only that PID**, and tear it down by that exact PID (never `pkill`). The control socket lives inside the 0700 `mktemp -d` directory and remote control is restricted to **`socket-only`** (no TTY-escape control), so no same-UID process can reach it via a predictable path:

```bash
tmpdir=$(mktemp -d); chmod 700 "$tmpdir"
sock="$tmpdir/rc.sock"
timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
  -o allow_remote_control=socket-only --listen-on "unix:$sock" \
  -o close_on_child_death=yes sh -c 'sleep 20' >/dev/null 2>&1 &
launch_pid=$!
trap 'kill "$launch_pid" 2>/dev/null; wait "$launch_pid" 2>/dev/null; rm -rf "$tmpdir"' EXIT
for i in $(seq 1 20); do [ -S "$sock" ] && break; sleep 0.3; done   # bounded wait for socket
# ... "$KITTY" @ --to "unix:$sock" <command> ...
kill "$launch_pid" 2>/dev/null; wait "$launch_pid" 2>/dev/null      # PID-scoped teardown
```

Every experiment in this document was run inside this harness. The per-experiment blocks below show the distinctive part (the child program, the flag, and the `kitty @` call) and the unedited output; each background block repeats the `mktemp -d` (mode 0700), the PID capture, and the `trap`/PID-scoped teardown so it is self-contained and abort-safe, but the one-time environment preamble (venv activation, `TMPDIR`, `LANG`/`LC_ALL`, `KITTY=…`) is assumed rather than reprinted every time.

### 0.5 Observation instruments (kitty's own, user-facing flags)

All evidence is captured with kitty's built-in instrumentation, verified in `kitty/cli.py`:

| Flag | Line | What it emits |
|------|------|---------------|
| `--dump-commands` | [kitty/cli.py:972] | The parsed command stream to stdout — one line per parser command (draws coalesced). Primary surface for Q1/Q3/Q5. |
| `--dump-bytes <path>` | [kitty/cli.py:985] | The **raw bytes received from the child** to a file. Shows the exact bytes crossing the boundary (Q1). |
| `--debug-rendering` / `--debug-gl` | [kitty/cli.py:989] | Render/GL debug info, plus `SIGWINCH sent to child` (Q1/Q2/Q5). |
| `--debug-input` / `--debug-keyboard` | [kitty/cli.py:996] | Key/mouse events as received (input side of Q1). |

**How `--dump-*` is wired (cite):** `kitty/boss.py` constructs the `ChildMonitor` [kitty/boss.py:370] and passes `DumpCommands(args) if args.dump_commands or args.dump_bytes else None` [kitty/boss.py:372]. `DumpCommands.__call__` [kitty/boss.py:239] buffers `draw` text, writes raw `bytes` to the dump file, and otherwise flushes the pending draw then prints the command name and its arguments — which is why the stream shows command names like `screen_set_mode`, `shell_prompt_marking`, and `draw` **interleaved in byte order**.

---

## 1. Pipeline overview

At a glance, a byte emitted by the child travels this path (each hop cited in the per-question sections):

```
child process (shell / TUI)
      │  writes bytes to PTY slave
      ▼
PTY master fd            child.py openpty:170 · Child.fork:276 · child_fd=master:338
      │  POLLIN in the I/O thread's poll(2) loop
      ▼
read_bytes()             child-monitor.c:1337 — read(fd,…):1345 straight into…
      │  (zero-copy)
      ▼
shared 1 MiB parse buf   vt-parser.c BUF_SZ:18 · create_write_buffer:1451 · commit_write:1465
      │  I/O thread wakes main thread after input_delay (WAKEUP gate :1566)
      ▼
MAIN THREAD, one tick    process_global_state:1224 — FIXED ORDER:
      ├─ process_pending_resizes(now)   :1233   (debounced 0.1/0.5 s)
      ├─ parse_input(self)              :1236 → run_worker (vt-parser.c:1417) parses & mutates screen
      │        │
      │        ├─ ordinary text → grid mutation (screen.c)
      │        ├─ OSC 133 hints → per-line prompt_kind (screen.c:2337)
      │        └─ DECSET 2026 / DCS =1s → screen_pause_rendering() (screen.c:2506) ── snapshot + 2000 ms timeout
      └─ render(now, input_read)        :1237   (throttled by repaint_delay 10 ms)

backpressure branch: buffer full → has_space_for_input()=false (vt-parser.c:1477)
                     → I/O loop masks POLLIN (child-monitor.c:1501) → PTY fills → child write() blocks

remote branch: peer/SSH → talk_loop() (child-monitor.c:1805) → main thread
```

The three structural facts that make everything else fall into place:

1. **The read is zero-copy into a single shared 1 MiB buffer.** `read_bytes()` reads directly into memory owned by the VT parser [kitty/child-monitor.c:1345, kitty/vt-parser.c:1451].
2. **All parsing and all screen mutation happen on one thread, in byte order.** The I/O thread reads child bytes into the buffer (and also drains POLLOUT writes to the child, delivers signals, and reaps dead children — see Q2); but it never touches the screen. *Parsing and screen mutation* happen only on the main thread [kitty/child-monitor.c:1236, kitty/vt-parser.c:1417]. This single-writer-of-the-screen design is the root cause of Q3's coherence.
3. **Every main-thread tick runs a fixed sequence: resizes → parse → render** [kitty/child-monitor.c:1233-1237]. This ordering is the literal answer to Q2's "what goes first."

The following sections answer each question with captured evidence. Temporary child scripts and sockets lived in a private `mktemp -d` directory (mode 0700, outside the repository) removed by a `trap` after each run (§0.4); the final repository proof is in Appendix C.

---

## Q1 — Where does raw input first enter the system, and what happens on pause → resume?

### 1.1 Direct answer

Raw bytes cross the boundary in **`read_bytes()`** [kitty/child-monitor.c:1337], which performs **`read(fd, buf, available_buffer_space)`** [kitty/child-monitor.c:1345] **directly into the VT parser's shared 1 MiB write buffer**. `buf` is not a private scratch buffer — it is a pointer *into* the parser's own storage, returned by `vt_parser_create_write_buffer()` [kitty/vt-parser.c:1451] and finalized by `vt_parser_commit_write()` [kitty/vt-parser.c:1465, called at kitty/child-monitor.c:1354]. This is a **zero-copy** read: the kernel writes child bytes straight into the buffer the parser will consume.

"**Paused then resumed**" is the terminal's **synchronized-update** feature. It is reached two ways that **converge on the same function** `screen_pause_rendering()` [kitty/screen.c:2506]:

- **Modern — DEC private mode 2026** (`CSI ? 2026 h` to pause / begin, `CSI ? 2026 l` to resume / end). Setting or resetting the mode routes through `case PENDING_MODE << 5:` → `screen_pause_rendering(self, val, 0)` [kitty/screen.c:1174-1175]. The mode bit is `PENDING_UPDATE (2026 << 5)` [kitty/modes.h:86].
- **Legacy — DCS pending sequences** (`DCS =1s ST` to pause, `DCS =2s ST` to resume). Handled at `case '=':` [kitty/vt-parser.c:636]; `=1s` → `screen_start_pending_mode` → `screen_pause_rendering(self->screen, true, 0)` [kitty/vt-parser.c:639-640]; `=2s` → `screen_stop_pending_mode` → `screen_pause_rendering(self->screen, false, 0)` [kitty/vt-parser.c:644-645].

Per the source, `screen_pause_rendering()` does three things on **pause**: refuses if already paused (`if (self->paused_rendering.expires_at) return false;` [kitty/screen.c:2518]); arms a **2000 ms default timeout** (`if (for_in_ms <= 0) for_in_ms = 2000;` [kitty/screen.c:2521]; `expires_at = monotonic() + ms_to_monotonic_t(for_in_ms)` [kitty/screen.c:2522]); and copies the visible grid line-by-line into a separate snapshot linebuf along with the cursor, color profile, selections, and graphics [kitty/screen.c:2528-2543]. While paused, `screen_update_cell_data()` renders from that snapshot linebuf rather than the live grid [kitty/screen.c:2737-2760] — so the *displayed* frame is code-designed to stay frozen while the *live grid keeps mutating*. On **resume** it refuses if not paused (`if (!self->paused_rendering.expires_at) return false;` [kitty/screen.c:2508]), clears `expires_at` [kitty/screen.c:2509], and sets `self->is_dirty = true` [kitty/screen.c:2511] so the accumulated live grid is uploaded on the next render. A safety net, `screen_check_pause_rendering()` [kitty/screen.c:2489], force-resumes when `now > expires_at` [kitty/screen.c:2490] so a stalled application can never freeze the display forever.

> **Scope of the runtime evidence (important).** The captures below use `--dump-commands`, which exposes the **parsed command stream** — it does **not** expose the GPU/display layer. So what is *directly observed* is that the parser keeps consuming input across a pause (the draws land between the pause and resume commands, in byte order). The display-side behavior — that the shown frame is the frozen snapshot, and that resume applies the accumulation as one frame with no half-drawn intermediate — is **code-derived** from `screen.c:2506-2543`, `screen.c:2737-2760`, and `screen.c:2511`, not captured as pixels (no framebuffer capture is available under headless software GL). Each such claim is labeled accordingly.

### 1.2 The entry point is zero-copy — the code that proves it

```c
// kitty/child-monitor.c:1337
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;
    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space); // :1341
    if (!available_buffer_space) return true;
    while(true) {
        len = read(fd, buf, available_buffer_space);   // :1345  ← bytes enter here, into the parser's buffer
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);      // :1354  ← commit what was read
    return len != 0;
```

`vt_parser_create_write_buffer()` returns a pointer *into* the shared buffer at the current write offset, and reports the free space as `BUF_SZ - offset`:

```c
// kitty/vt-parser.c:1451
vt_parser_create_write_buffer(Parser *p, size_t *sz) {
    ...
    self->write.offset = self->read.sz + self->write.pending;
    *sz = BUF_SZ - self->write.offset;   // :1457
    ...
    ans = self->buf + self->write.offset;
    return ans;   // pointer INTO self->buf — no intermediate copy
}
```

The buffer is exactly **1 MiB**: `#define BUF_SZ (1024u*1024u)` [kitty/vt-parser.c:18] (with `BUF_EXTRA (512u/8u)` slack at :20 — note :19 is the explanatory comment — and `MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)` at :21). This is confirmed at runtime from the built module:

```
$ timeout 30 ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE as B; print(B)'
1048576
```

`1048576 = 1024 × 1024`, i.e. the 1 MiB `BUF_SZ`. (`VT_PARSER_BUFFER_SIZE` is the same symbol the test-suite imports to size its own buffers — `from kitty.fast_data_types import ... VT_PARSER_BUFFER_SIZE` [kitty_tests/parser.py:10] — so the runtime value and the source `BUF_SZ` agree.)

**Where the PTY fd comes from (Q1 boundary):** the master fd that `read_bytes()` reads is created by `os.openpty()` in `openpty()` [kitty/child.py:170]; the child is spawned by `Child.fork` [kitty/child.py:276]; kitty keeps the master end as `self.child_fd = master` [kitty/child.py:338] and makes it non-blocking with `os.set_blocking(child_fd, False)` [kitty/child.py:345] (the child's slave end stays blocking — the basis for OS flow control in Q4).

### 1.3 Raw bytes at the boundary (observed)

To show the *exact* bytes crossing the boundary, a child emits a synchronized-update wrapper around the text `HELLO`, and `--dump-bytes` captures what kitty read (the dump file lives in the private `$tmpdir`):

```bash
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --dump-bytes "$tmpdir/q1_bytes.bin" \
    sh -c "printf '\033[?2026hHELLO\033[?2026l'" >/dev/null 2>&1
$ od -A d -t x1z "$tmpdir/q1_bytes.bin"
```

Unedited output:

```
0000000 1b 5b 3f 32 30 32 36 68 48 45 4c 4c 4f 1b 5b 3f  >.[?2026hHELLO.[?<
0000016 32 30 32 36 6c                                   >2026l<
0000021
```

Decoded: `1b 5b 3f 32 30 32 36 68` = `ESC [ ? 2 0 2 6 h` (the BSU / pause), then `48 45 4c 4c 4f` = `HELLO`, then `1b 5b 3f 32 30 32 36 6c` = `ESC [ ? 2 0 2 6 l` (the ESU / resume). The precise pause/resume bytes demonstrably crossed the boundary via the real PTY. (The raw-byte callback is emitted from the parser's own read buffer as `dump_callback(window_id, "bytes", …)` [kitty/vt-parser.c:1399] and written to the file by `DumpCommands.__call__` [kitty/boss.py:243-244], so the hexdump faithfully reflects the boundary bytes.)

### 1.4 Before / during / after — the crux of pause → resume

The interesting behavior is *transitional*, so the value is reported **before**, **during**, and **after** the pause.

**Experiment Q1b — a large surge wrapped in DEC 2026.** The child (written to `$tmpdir/child_dec.sh`) sets mode 2026, writes 50 lines, then resets it:

```bash
$ cat "$tmpdir/child_dec.sh"
printf '\033[?2026h'                                # BSU — begin synchronized update
for i in $(seq 1 50); do printf 'SURGE line %02d\n' "$i"; done
printf '\033[?2026l'                                # ESU — end synchronized update
printf 'AFTER-ESU visible line\n'
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --dump-commands sh "$tmpdir/child_dec.sh" 2>/dev/null
```

The full stream is 155 command lines: one `screen_set_mode 2026 1`, then 50 identical `draw`/CR/LF triples, then one `screen_reset_mode 2026 1`, then the trailing line. It is shown here head-and-tail with the identical middle triples collapsed to a parenthetical count (every distinct command is present; only exact repeats of the already-shown triple are summarized):

```
screen_set_mode 2026 1
draw SURGE line 01
screen_carriage_return
screen_linefeed
draw SURGE line 02
screen_carriage_return
screen_linefeed
        (SURGE line 03 … SURGE line 49: each is draw / screen_carriage_return / screen_linefeed — 47 identical triples)
draw SURGE line 50
screen_carriage_return
screen_linefeed
screen_reset_mode 2026 1
draw AFTER-ESU visible line
screen_carriage_return
screen_linefeed
```

Verifiable counts from the same capture (the `run` helper is the exact command above, captured once so the pipes are reproducible):

```bash
$ run() { timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
      -o close_on_child_death=yes --dump-commands sh "$tmpdir/child_dec.sh" 2>/dev/null; }
$ run | wc -l
155
$ run | grep -c '^draw SURGE line'
50
$ run | grep -nE 'screen_set_mode 2026|screen_reset_mode 2026|AFTER-ESU'
1:screen_set_mode 2026 1
152:screen_reset_mode 2026 1
153:draw AFTER-ESU visible line
```

- **BEFORE (observed):** the stream begins with `screen_set_mode 2026 1` at line 1 — before it, rendering is live (the code initializes `expires_at == 0`).
- **DURING (observed at the command layer; display-side code-derived):** `screen_set_mode 2026 1` (DECSET `CSI?2026h`) calls `screen_pause_rendering(self, true, 0)` [kitty/screen.c:1175]. **All 50 `draw SURGE line NN` commands appear between line 1 and line 152, in byte order** — the *directly observed* fact is that the parser keeps consuming and mutating the live grid throughout the pause. That the *displayed* frame is meanwhile the frozen snapshot is **code-derived** (`expires_at` set at [kitty/screen.c:2522]; snapshot linebuf populated at [kitty/screen.c:2528-2543]; paused rendering reads it at [kitty/screen.c:2737-2760]), not visible in this stream.
- **AFTER (observed at the command layer; single-frame atomicity code-derived):** `screen_reset_mode 2026 1` at line 152 (DECRST `CSI?2026l`) calls `screen_pause_rendering(self, false, 0)`, which sets `is_dirty = true` [kitty/screen.c:2511]; `draw AFTER-ESU visible line` then follows at line 153. That the accumulated 50-line grid is applied as *one* frame with no half-drawn intermediate is **code-derived** from the `is_dirty` upload path, not captured as pixels.

**Cause → effect:** DECSET 2026 *causes* `screen_pause_rendering(true)` to set `expires_at` and take the snapshot [kitty/screen.c:2522-2543], which *causes* the render path to keep presenting the snapshot [kitty/screen.c:2737] while the parser keeps mutating the real grid (observed: the 50 interleaved draws); DECRST 2026 *causes* `is_dirty=true` [kitty/screen.c:2511], which *causes* the next `render()` to upload the whole accumulated grid at once (code-derived).

**Experiment Q1c — the legacy DCS path converges on the same function.** Same shape, using `DCS =1s ST` / `DCS =2s ST`:

```bash
$ cat "$tmpdir/child_dcs.sh"
printf '\033P=1s\033\\'                    # legacy pause
printf 'DCS line 1\n'; printf 'DCS line 2\n'
printf '\033P=2s\033\\'                    # legacy resume
printf 'after-dcs\n'
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --dump-commands sh "$tmpdir/child_dcs.sh" 2>/dev/null
```

Unedited output:

```
screen_start_pending_mode
draw DCS line 1
screen_carriage_return
screen_linefeed
draw DCS line 2
screen_carriage_return
screen_linefeed
screen_stop_pending_mode
draw after-dcs
screen_carriage_return
screen_linefeed
```

`screen_start_pending_mode` / `screen_stop_pending_mode` are the legacy `=1s` / `=2s` reports [kitty/vt-parser.c:639, :644]; both call the very same `screen_pause_rendering()` as DEC 2026 [kitty/vt-parser.c:640, :645], confirming convergence.

### 1.5 The 2000 ms timeout and the two refusal paths (observed)

The pause branch and its 2000 ms default are:

```c
// kitty/screen.c:2506
screen_pause_rendering(Screen *self, bool pause, int for_in_ms) {
    if (!pause) {
        if (!self->paused_rendering.expires_at) return false;   // :2508 refuse resume if not paused
        self->paused_rendering.expires_at = 0;                  // :2509
        self->is_dirty = true;                                  // :2511 atomic flush on resume
        ...
        return true;
    }
    if (self->paused_rendering.expires_at) return false;        // :2518 refuse pause if already paused
    ...
    if (for_in_ms <= 0) for_in_ms = 2000;                       // :2521 default 2000 ms
    self->paused_rendering.expires_at = monotonic() + ms_to_monotonic_t(for_in_ms);  // :2522
    ...
```

with the force-resume safety net:

```c
// kitty/screen.c:2489
screen_check_pause_rendering(Screen *self, monotonic_t now) {
    if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at)
        screen_pause_rendering(self, false, 0);   // :2490 force resume past timeout
}
```

**Experiment Q1d — bracketing the 2000 ms boundary, 2 runs each.** A child pauses, sleeps, then resumes; if the sleep exceeds 2000 ms the timeout force-resumes *before* the resume arrives, so the late resume finds `expires_at == 0` and is refused (which the parser reports as an error). Sleeps of 1.5 s and 2.5 s were each run twice. The `=2s` refusal report is a `_report_error` routed to the dump stream, so it is captured by merging stderr into stdout (`2>&1`):

```bash
# < 2000 ms — clean resume (run twice)
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --dump-commands \
    sh -c 'printf "\033P=1s\033\\"; sleep 1.5; printf "\033P=2s\033\\"' 2>&1 \
    | grep -iE 'pending'
# > 2000 ms — timeout force-resume, late resume refused (run twice)
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --dump-commands \
    sh -c 'printf "\033P=1s\033\\"; sleep 2.5; printf "\033P=2s\033\\"' 2>&1 \
    | grep -iE 'pending'
```

Unedited `grep 'pending'` output, both runs of each sleep:

```
# 1.5 s, run 1:                 # 1.5 s, run 2:
screen_start_pending_mode       screen_start_pending_mode
screen_stop_pending_mode        screen_stop_pending_mode

# 2.5 s, run 1:
[2.670] Pending mode stop command issued while not in pending mode, this can be either a bug in the terminal application or caused by a timeout with no data received for too long or by too much data in pending mode
screen_start_pending_mode
screen_stop_pending_mode
# 2.5 s, run 2:
[2.667] Pending mode stop command issued while not in pending mode, this can be either a bug in the terminal application or caused by a timeout with no data received for too long or by too much data in pending mode
screen_start_pending_mode
screen_stop_pending_mode
```

At 1.5 s the pause is still active when the resume arrives, so there is no error (both runs). At 2.5 s the timeout has already force-resumed, so the resume is refused with the `caused by a timeout` report (both runs; the `[2.670]`/`[2.667]` prefix is the wall-clock stamp when the late `=2s` was parsed, ~2.5 s + startup). The error text is the DCS `=2s` refusal report [kitty/vt-parser.c:645-648], fired because `screen_pause_rendering(false)` returned `false` at [kitty/screen.c:2508]. The boundary lands cleanly between 1.5 s and 2.5 s, matching the coded default of 2000 ms [kitty/screen.c:2521] — **stable across two runs each**.

**The other refusal path — double pause.** Pausing while already paused is refused at [kitty/screen.c:2518]:

```bash
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --dump-commands \
    sh -c 'printf "\033P=1s\033\\"; printf "\033P=1s\033\\"' 2>&1 | grep -iE 'pending|already'
```
```
[0.177] Pending mode start requested while already in pending mode. This is most likely an application error.
screen_start_pending_mode
screen_start_pending_mode
```
That is the DCS `=1s` refusal report [kitty/vt-parser.c:640-641], fired because the second pause found `expires_at` already set. *(The DEC-2026 path reaches the analogous refusal through [kitty/screen.c:1174-1176], which emits `log_error` rather than a parser report; both share the same `screen_pause_rendering` return-`false` mechanism.)*

### 1.6 The input side of the boundary, and bracketed paste

**Keystroke ingress — what `--debug-input` actually shows (honest).** `--debug-input` prints on **stderr**. With no injectable keypress, the only events it reports are windowing-layer *initialization*, not a key event:

```bash
$ timeout 30 xvfb-run -a -s "-screen 0 1280x800x24" "$KITTY" --config NONE \
    -o close_on_child_death=yes --debug-input sh -c 'sleep 0.3' 2>&1 | grep -vi systemd
```
```
[0.060] Loading new XKB keymaps
[0.065] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.161] Mouse cursor entered window: 1 at 640.000000x400.000000
[0.161] Move x: 640.0 y: 400.0 grabbed: 0
[0.161] on_focus_change: window id: 0x1 focused: 0
[0.162] on_focus_change: window id: 0x1 focused: 1
```

> **Honesty label — keyboard encoding is NOT observed here.** The lines above are XKB keymap load, modifier-index setup, an xvfb-generated pointer-enter, and focus initialization — **no key press**. No X-event-injection tool exists in this container: `xdotool`, `xte`, `wtype`, `ydotool` are all absent, and `import Xlib` fails in both the venv and system Python, so an XTEST-synthesized keypress is impossible. Therefore the `glfw → keys.c → key_encoding.c` key-to-escape **encoding stage is unobserved / code-derived**, not demonstrated. Remote-control `send-text` (below) exercises the *write transport* to the child, but by design it **bypasses** the glfw key-encoding stage, so it is not a substitute for a real keypress.

**Bracketed paste through the real PTY (observed).** The paste convention wraps pasted text in `ESC[200~ … ESC[201~` so applications can tell pasted input from typed input. Kitty's constants are `BRACKETED_PASTE (2004 << 5)` [kitty/modes.h:81 — corrected from the plan's `:80`], with `BRACKETED_PASTE_START "200~"` [kitty/modes.h:82] and `BRACKETED_PASTE_END "201~"` [kitty/modes.h:83]. To prove the wrapper actually reaches the child, the child puts its PTY slave in raw mode and saves whatever bytes arrive on stdin; the paste is delivered with the supported flag `send-text --bracketed-paste=enable` (an explicitly *non-keyboard* paste path):

```bash
$ cat "$tmpdir/paste_child.py"
import os, sys, tty, select
out = sys.argv[1]
try: tty.setraw(0)
except Exception: pass
data = b''
while b'\x1b[201~' not in data and len(data) < 8192:
    r, _, _ = select.select([0], [], [], 6)
    if not r: break
    chunk = os.read(0, 4096)
    if not chunk: break
    data += chunk
open(out, 'wb').write(data)
# launch (background, socket-only, PID-scoped teardown — see §0.4), child = python3 paste_child.py "$tmpdir/recv.bin"
$ "$KITTY" @ --to "unix:$sock" send-text --bracketed-paste=enable 'PASTED-TEXT'
$ od -A d -t x1z "$tmpdir/recv.bin"
```

Unedited output — the exact bytes the child received on its stdin:

```
0000000 1b 5b 32 30 30 7e 50 41 53 54 45 44 2d 54 45 58  >.[200~PASTED-TEX<
0000016 54 1b 5b 32 30 31 7e                             >T.[201~<
0000023
```

Decoded: `1b 5b 32 30 30 7e` = `ESC [ 2 0 0 ~` (paste **start**), then `50 41 53 54 45 44 2d 54 45 58 54` = `PASTED-TEXT`, then `1b 5b 32 30 31 7e` = `ESC [ 2 0 1 ~` (paste **end**). This is the definitive Q1-input evidence: the `ESC[200~…ESC[201~` wrapper demonstrably crossed the real PTY into the child. Paste is orchestrated on the Python side by `paste_with_actions` [kitty/window.py:1643] → `paste_bytes` [kitty/window.py:1707] / `paste_text` [kitty/window.py:1713]; `in_bracketed_paste_mode` is a `MODE_GETSET` at [kitty/screen.c:3854]; the C paste helper is `paste_()` [kitty/screen.c:4573]. *(Label: this exercises the paste write-transport, not the keyboard-encoding stage.)*

---

## Q2 — The "unseen conductor": timing, ordering, and state hand-offs — *what decides which event gets handled first?*

### 2.1 Direct answer

There is an **"unseen conductor"** and it is remarkably literal: kitty uses a **three-thread model** with a strict division of labor, and the main thread runs a **fixed per-tick sequence**. *What decides which event gets handled first* is not a heuristic or a priority queue — it is **hard-coded order** in `process_global_state()` [kitty/child-monitor.c:1224]: **resizes, then parse, then render**, deterministically, on every tick. And *within* the parse step there is a second hard-coded order: queued remote-control / talk-thread peer messages are dispatched **before** any child's VT output is parsed (§2.3).

The three threads and their true responsibilities:

- **I/O thread — `io_loop()`** [kitty/child-monitor.c:1481] (named `KittyChildMon` at :1489): a `poll(2)` loop that shuttles bytes across the PTY boundary in **both** directions and manages child-fd lifecycle and process signals. It is **not** "read-only." It **reads** child output into the shared buffer (`read_bytes` [:1531]), **writes** queued keystroke/paste bytes back to the child on `POLLOUT` (`write_to_child` [:1540]), drains the wakeup pipe [:1515], dispatches signals — including `SIGCHLD`-driven child reaping [:1519, :1526] — and retires unexpectedly-closed fds on `POLLNVAL` [:1542]. The one thing it never does is **parse escape sequences or mutate the grid**; that is the main thread's exclusive job (full duty list in §2.4).
- **Main thread — `main_loop()`** [kitty/child-monitor.c:1259] → `run_main_loop(process_global_state, self)` [kitty/child-monitor.c:1262]: runs *all* VT parsing and *all* screen mutation, then renders — and, before parsing child output, dispatches queued peer messages (§2.3).
- **Talk thread — `talk_loop()`** [kitty/child-monitor.c:1805] (named `KittyPeerMon` at :1808): accepts remote-control peers over a Unix socket and **queues** their messages for the main thread; it does not itself parse or mutate the screen (Q4).

### 2.2 The three threads, observed at runtime

Each thread names itself via `set_thread_name(...)` [kitty/threading.h:26, calling `pthread_setname_np`], so a running process reveals kitty's model directly through read-only inspection of `/proc/<pid>/task/*/comm`. The harness below launches a headless kitty with a private listen socket (so the talk thread comes up), waits for the pipeline threads to name themselves, samples the thread table, then tears the process down by pid — all inside a `0700` scratch dir that is removed afterward:

```bash
source /tmp/kitty-venv/bin/activate
export TMPDIR=/tmp/kitty-clean-tmp LANG=C.UTF-8 LC_ALL=C.UTF-8
KITTY=./kitty/launcher/kitty

d="$(mktemp -d)"; chmod 0700 "$d"; sock="$d/rc.sock"
timeout 20 xvfb-run -a -s "-screen 0 1280x800x24" \
  "$KITTY" --config NONE -o allow_remote_control=socket-only --listen-on "unix:$sock" \
  -o close_on_child_death=yes sh -c 'sleep 6' >/dev/null 2>&1 &
launch_pid=$!
trap 'kill "$launch_pid" 2>/dev/null; wait "$launch_pid" 2>/dev/null; rm -rf "$d"' EXIT  # abort-safe teardown
# resolve the real kitty pid (comm==kitty) whose cmdline carries our unique socket
pid=""; tries=0
while [ $tries -lt 60 ]; do
  for p in $(pgrep -x kitty); do
    tr '\0' ' ' < /proc/$p/cmdline | grep -q "unix:$sock" && { pid="$p"; break; }
  done
  [ -n "$pid" ] && break; tries=$((tries+1)); sleep 0.1
done
sleep 1   # let io_loop + talk_loop run set_thread_name (reach steady state)
echo "kitty's own pipeline threads:"
cat /proc/$pid/task/*/comm | grep -E '^KittyChildMon$|^KittyPeerMon$' | sort | uniq -c
kill "$launch_pid"; wait "$launch_pid" 2>/dev/null; rm -rf "$d"
```

Observed, identical across three runs (PIDs 134877 / 134988 / 135099):

```
kitty's own pipeline threads:
      1 KittyChildMon
      1 KittyPeerMon
```

`KittyChildMon` (the I/O thread) and `KittyPeerMon` (the talk thread) are present in every run; together with the main thread they are kitty's entire concurrency model — **main + I/O (`KittyChildMon`) + talk (`KittyPeerMon`)**. The talk thread appears only because `--listen-on` brought up a listen socket.

> **Honesty label — the names appear ~0.3 s late, and the rest of the thread table is environment noise.** A *time-series* sample of the same process shows the Kitty-named threads are **not** present the instant the pid becomes visible: at sample 0 only a single `kitty` thread exists and neither `KittyChildMon` nor `KittyPeerMon` is there; from ~0.3 s onward **both** appear and persist for the process lifetime (no `Failed to set thread name` ever prints on stderr, so naming succeeds — the I/O and talk threads simply run `set_thread_name` as their first action a moment after `pthread_create` returns [kitty/child-monitor.c:1489, :1808]). Sampling too early is exactly what makes the names look absent; the harness adds a settle delay to avoid that pitfall. The remainder of the live thread table — **33** threads still named `kitty`, **32** named `llvmpipe-0…31`, and one `kitty:disk$0` — is **not** part of kitty's model: the `llvmpipe-N` and plain `kitty` workers are **Mesa's software-GL rasterizer pool** (a headless software-rendering artifact) and `kitty:disk$0` is kitty's disk-cache worker (unrelated to the input pipeline). All three counts are stable across the three runs.

The pipeline threads are created in the monitor: `pthread_create(..., io_loop, ...)` [kitty/child-monitor.c:291] and the talk thread [kitty/child-monitor.c:256 / :286].

### 2.3 The fixed per-tick order — the literal "who goes first"

The **top-level** order lives in `process_global_state()`, reproduced verbatim (no elision) from [kitty/child-monitor.c:1224-1237]:

```c
process_global_state(void *data) {
    EVDBG("Processing global state");
    ChildMonitor *self = data;
    maximum_wait = -1;
    bool state_check_timer_enabled = false;
    bool input_read = false;

    monotonic_t now = monotonic();                       // :1231
    if (global_state.has_pending_resizes) {
        process_pending_resizes(now);                    // :1233  ← 1) RESIZES first
        input_read = true;
    }
    if (parse_input(self)) input_read = true;            // :1236  ← 2) PARSE second
    render(now, input_read);                             // :1237  ← 3) RENDER third
```

**Every tick, unconditionally in this order: `process_pending_resizes` → `parse_input` → `render`.** Pending resizes are absorbed before parsing, parsing (and thus all screen mutation) completes before rendering, and rendering always sees a fully-parsed, self-consistent grid.

**But "parse" is itself ordered — and that sub-order is the part a bare "resize → parse → render" summary hides.** Inside `parse_input()` [kitty/child-monitor.c:451] the main thread executes this fixed sequence, in source order:

1. **Drain the child-removal queue** into `remove_notify` under the children lock [:457-462].
2. **Handle kill / reload-config signals** that the I/O thread flagged [:465-475].
3. **Snapshot the live child list** into `scratch` [:478-481].
4. **Dispatch queued peer messages** — under `talk_mutex` it copies the talk thread's queue [:487-497], then for each message calls `boss.peer_message_received(...)` and returns a response to the peer [:504] — **before any child output is parsed**.
5. **Final-parse-then-notify-death** for each removed child: `do_parse(..., true)` [:521].
6. **Parse each live child's buffered VT output**: `do_parse(scratch[i].screen, now, false)` [:530] — *this* is the step that consumes the shared 1 MiB buffer and mutates the grid.
7. **Apply a deferred config reload** if one was requested [:536].

So the precise answer to *"what decides which event gets handled first?"* is: **resizes precede everything; then, within the parse step, remote-control / talk-thread peer messages are serviced before child terminal output; child VT output is parsed last, immediately before render.** A remote-control command arriving in the same tick as a burst of child output is therefore acted on *first*, on the main thread, with no possibility of interleaving with the child parse — the structural basis of Q4's remote-control ordering.

> **Honesty label — why there is no per-tick log line.** The natural per-tick trace, `EVDBG("Processing global state")` [kitty/child-monitor.c:1225] and `render`'s `EVDBG("input_read: %d ...")` [kitty/child-monitor.c:872], is gated on the compile-time macro `#ifdef DEBUG_EVENT_LOOP` [kitty/child-monitor.c:29-33], which is **not defined in the canonical build**. So these lines are compiled *out* and never appear at runtime. The order is therefore a **structural code certainty**, not a runtime log; its *observable consequences* are Q1's atomic parse-before-render pause/resume and Q3's in-order coherence, both captured above and below.

### 2.4 Poll-level ordering inside the I/O thread — and its full duty list

The I/O thread's `poll(2)` fd array is deliberately ordered so control fds precede child fds:

- **Setup:** `children_fds[0].fd = wakeup_read_fd`, `children_fds[1].fd = signal_read_fd` [kitty/child-monitor.c:183], both armed for `POLLIN` [:184]; `EXTRA_FDS = 2` [:35]; child fds begin at index `EXTRA_FDS`. Crucially, each child fd's requested events are recomputed *every* iteration: `POLLIN` is requested **only if** the parser still has buffer space — `vt_parser_has_space_for_input(...) ? POLLIN : 0` [:1501] (the Q4 backpressure gate) — and `POLLOUT` is OR-ed in **only if** there are queued bytes to write back — `screen->write_buf_used ? POLLOUT : 0` [:1503].
- **After `poll()` returns**, the thread processes revents in this fixed order:
  1. **Wakeup fd** `children_fds[0]` → `drain_fd` [:1515] (empties the self-pipe other threads use to wake the loop).
  2. **Signal fd** `children_fds[1]` → `read_signals(..., handle_signal, ...)` [:1519]; if the decoded set includes a child death, `reap_children(self, OPT(close_on_child_death))` runs immediately [:1526].
  3. **Child-fd loop** `children_fds[EXTRA_FDS + i]`, and for *each* child fd **three** distinct duties, not one:
     - **Read** on `POLLIN | POLLHUP` → `read_bytes(...)` into the shared buffer [:1531]; a `false` return (EOF) marks the child for removal.
     - **Write** on `POLLOUT` → `write_to_child(children[i].fd, children[i].screen)` [:1540] — this is where queued keystrokes and pasted text are actually pushed to the child, **in the I/O thread**.
     - **Retire** on `POLLNVAL` → mark the child for removal and log that its fd was unexpectedly closed [:1542].

So the I/O thread's real job is **wakeup → signals/reaping → (per child) read, write, and fd-retirement** — it is emphatically *not* "read-only," even though it never parses or mutates the grid. The write path feeding the `POLLOUT` duty is: `Window.write_to_child` [kitty/window.py:955] → `needs_write` [kitty/child-monitor.c:412] → `schedule_write_to_child` [:372], which appends to `write_buf` and wakes the I/O loop [:363]; the loop then arms `POLLOUT` [:1503] and drains the buffer via `write_to_child` [:1540]. (A separate, *transient* `KittyWriteStdin` thread — `thread_write` [:966], spawned by `cm_thread_write` [:992] — exists only for a one-shot bulk stdin write when a child is launched with initial input [kitty/child.py:341, kitty/boss.py:2439]; it is not used for interactive keystrokes and is not one of the three always-on threads.)

Signals are handled via `KITTY_HANDLED_SIGNALS = SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` [kitty/child-monitor.c:121] in `handle_signal` [kitty/child-monitor.c:1362]; `SIGCHLD` [:1370] drives child reaping.

**Observed consequence of the signal path (a consequence, not a proof of tick ordering).** The clean-exit path is directly observable: when the child exits, `SIGCHLD` → reap → kitty exits under `close_on_child_death=yes`. Full command and unedited output (exit code identical across two runs):

```bash
d="$(mktemp -d)"; chmod 0700 "$d"
timeout 20 xvfb-run -a -s "-screen 0 1280x800x24" \
  ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes --debug-rendering \
  sh -c 'printf "hi\n"; sleep 0.2' >"$d/out.log" 2>"$d/err.log"
echo "kitty exit: $?"
grep -v "systemd user bus" "$d/err.log"   # stderr (debug-rendering)
cat "$d/out.log"                            # stdout
rm -rf "$d"
```
```
kitty exit: 0
[0.151] OS Window created
[0.163] Child launched
[0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

(stderr and stdout are separate streams — the `[0.127]` GL line is stdout and simply predates the `[0.151]`/`[0.163]` stderr lines.) This demonstrates the *consequence* (child death → reap → orderly exit code 0); it says nothing about intra-tick ordering, since the per-tick trace is compiled out (§2.3 honesty label). The intra-tick order is a code certainty; this clean exit is one of its observable downstream effects.

> Note: `SIGWINCH` is **not** in `KITTY_HANDLED_SIGNALS` — kitty, *as the terminal*, generates resizes from glfw and **sends** `SIGWINCH` to the child (Q5); it does not receive `SIGWINCH` itself.

### 2.5 Wakeup coalescing — when the main thread wakes

The I/O thread does not wake the main thread on every read. The `WAKEUP` macro fires only after `input_delay` has elapsed since the last wakeup:

```c
// kitty/child-monitor.c:1562
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
        // we only wakeup the main loop after input_delay as wakeup is an expensive operation
        // on some platforms, such as cocoa
        if (data_received) {
            if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP   // :1566
            else has_pending_wakeups = true;
        } else {
            if (has_pending_wakeups && (now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP  // :1569
        }
```

**Cause → effect:** many small child writes arriving within one `input_delay` window (3 ms; §Q5) do *not* each wake the main thread; they are **batched** so that a single main-thread tick parses them together. The poll timeout is itself bounded by `input_delay` when a wakeup is pending [kitty/child-monitor.c:1506-1512], guaranteeing the batch is flushed promptly. This is the mechanism that keeps the "conductor" from being overwhelmed by a fast talker, and it feeds directly into Q5's rhythm.

**Why single-writer + fixed order guarantees no interleaving:** because only the main thread ever mutates the screen, and it does so in the fixed `resize → parse → render` order within a tick, there is no window in which a resize could land mid-parse, or a render could observe a half-applied escape sequence. The I/O and talk threads only *hand data to* the main thread; they never mutate state concurrently. That is the structural guarantee underlying Q1 (atomic pause/resume) and Q3 (coherent interleaving).

---

## Q3 — Shell-integration hints *mixed in with ordinary text*: staying aligned *without drifting out of sync*

### 3.1 Direct answer

They stay aligned because **all parsing and all screen mutation happen on one thread, in exact byte order**, and prompt attribution is stored **per line**. An `OSC 133` marker and the ordinary text around it are processed in precisely the order the child emitted them: `parse_input` [kitty/child-monitor.c:451] (called from the fixed tick at [kitty/child-monitor.c:1236]) runs `run_worker` [kitty/vt-parser.c:1417], which parses *and* mutates the screen — while the I/O thread never parses or mutates the grid (it only hands raw bytes to the shared buffer; §Q2). Because the `A` and `C` markers are written to `line_attrs[cursor->y].prompt_kind` [kitty/screen.c:2341] for **whatever line the cursor is on at that instant**, and the cursor position is *itself* a product of the same in-order parse, the hint **cannot drift** relative to the text. This is the direct mechanism behind the user's phrase *"without drifting out of sync."*

### 3.2 Where the hints come from, and how they are handled

Shell integration is injected by `modify_shell_environ()` [kitty/shell_integration.py:218]; for bash it sets `ENV` to `kitty.bash` and rewrites the child argv to `bash --posix` [kitty/shell_integration.py:134, :146] (confirmed at runtime below), so the script is sourced without touching the user's rcfile. The per-shell scripts (`shell-integration/bash/kitty.bash`, `zsh/kitty-integration`, `fish/vendor_conf.d/kitty-shell-integration.fish`) then emit the markers.

**Which markers exist versus which kitty actually handles and emits — a critical distinction.** The public FTCS / OSC 133 convention defines `A` = prompt start, `B` = prompt end / command-input start, `C` = command-output start, `D;<exit>` = command finished. **Kitty implements only `A`, `C`, and `D` — there is no `case 'B'`** (verbatim switch below), and the bundled bash integration **emits only `A`/`C`/`D`** (plus kitty's own `k;…` region markers), never `B`:

```bash
# which 133;<letter> markers the bundled bash integration actually emits:
grep -oE "133;[A-Za-z]" shell-integration/bash/kitty.bash | sort | uniq -c
```
```
      3 133;A
      1 133;C
      1 133;D
      7 133;k
```

So `B` is a **public convention that kitty neither emits nor handles** in the exercised paths; a claim that a `B` marker was "observed" would be false. (`k=s`, carried on an `A` marker as `A;k=s`, selects the secondary/continuation prompt — §3.5; the `133;k` markers are a kitty-specific prompt-redraw-region extension, not the FTCS `A/B/C/D`.)

On the parser side, `OSC 133` is dispatched in `dispatch_osc` [kitty/vt-parser.c:457], `case 133` [kitty/vt-parser.c:536], which calls `shell_prompt_marking(self->screen, (char*)buf + i)` [kitty/vt-parser.c:544]. The handler, reproduced **verbatim** (no elision) [kitty/screen.c:2328-2356], shows exactly three cases and where each writes per-line state:

```c
shell_prompt_marking(Screen *self, char *buf) {
    if (self->cursor->y < self->lines) {
        char ch = buf[0];
        switch (ch) {
            case 'A': {
                PromptKind pk = PROMPT_START;
                self->prompt_settings.redraws_prompts_at_all = 1;
                self->prompt_settings.uses_special_keys_for_cursor_movement = 0;
                parse_prompt_mark(self, buf+1, &pk);
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = pk;            // :2337  per-line tag
                if (pk == PROMPT_START) CALLBACK("cmd_output_marking", "O", Py_False);   // :2338
            } break;
            case 'C': {
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = OUTPUT_START;   // :2341  per-line tag
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
                CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);              // :2352  NO prompt_kind set
            } break;
        }
    }
}
```

Two facts follow directly from this source:

1. **`A` and `C` set a per-line `prompt_kind`** — `PROMPT_START`/`SECONDARY_PROMPT` and `OUTPUT_START` respectively, on `line_attrs[cursor->y]` [:2337, :2341]; **`D` sets none** — it only fires the `cmd_output_marking(None, exit_status)` callback [:2352]. So the notion that "`D` closes the output range" in the grid is wrong: `D` carries the exit status to Python (`Window.cmd_output_marking` → `handle_cmd_end` [kitty/window.py:1453]) but tags no line.
2. The **command-output range** (what `get-text --extent=last_cmd_output` returns, §3.4) is computed by `find_cmd_output` [kitty/screen.c:3527] purely from the per-line tags: it starts at the `OUTPUT_START` line and **ends at the next `PROMPT_START`** [kitty/screen.c:3539] (or a screen/history boundary), not wherever `D` arrived.

`parse_prompt_mark` [kitty/screen.c:2316] decodes sub-tokens; `k=s` sets `SECONDARY_PROMPT` [kitty/screen.c:2321].

### 3.3 Observed: markers interleaved in byte order with ordinary text

A **real interactive bash** with shell integration active (kitty's default) is driven through the PTY and captured under `--dump-commands`. Two method notes make the capture faithful: (a) `--dump-commands` output is block-buffered to a file, so the child is exited cleanly with **Ctrl-D** (`\x04`) — `close_on_child_death=yes` then makes kitty flush and exit, leaving the full stream intact; (b) `send-text` here is **remote-control text injection through the child's PTY input** — it is *not* keyboard-encoded input (the keyboard-encoding stage is treated under Q1). Full, runnable harness:

```bash
source /tmp/kitty-venv/bin/activate
export TMPDIR=/tmp/kitty-clean-tmp LANG=C.UTF-8 LC_ALL=C.UTF-8
KITTY=./kitty/launcher/kitty

d="$(mktemp -d)"; chmod 0700 "$d"; sock="$d/rc.sock"
timeout 30 xvfb-run -a -s "-screen 0 1280x800x24" \
  "$KITTY" --config NONE -o allow_remote_control=socket-only --listen-on "unix:$sock" \
  --dump-commands -o close_on_child_death=yes bash -i >"$d/dump.log" 2>"$d/err.log" &
launch_pid=$!
trap 'kill "$launch_pid" 2>/dev/null; wait "$launch_pid" 2>/dev/null; rm -rf "$d"' EXIT  # abort-safe teardown
for i in $(seq 1 50); do [ -S "$sock" ] && break; sleep 0.2; done
sleep 2
send() { "$KITTY" @ --to "unix:$sock" send-text "$1"; sleep 0.7; }
send 'unset HISTCONTROL\r'          # silence the HISTCONTROL advisory
send "PS1='PROMPT\$ '\r"            # predictable prompt text
send 'echo hello-world\r'
send '\x04'                          # Ctrl-D: exit bash -> kitty exits cleanly -> dump flushes
wait "$launch_pid"; echo "kitty exit: $?"
sed -n '54,84p' "$d/dump.log"        # the echo hello-world cycle
rm -rf "$d"
```

The command exits `0`; the unedited dump slice for the `echo hello-world` cycle is:

```
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
draw PROMPT$ 
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e
shell_prompt_marking 133 k;end_suffix_kitty
draw echo hello-world
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
set_title echo hello-world
shell_prompt_marking 133 C;cmdline=echo\ hello-world
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 0 32
shell_prompt_marking 133 k;end_suffix_kitty
draw hello-world
screen_carriage_return
screen_linefeed
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
draw PROMPT$ 
```

The markers land exactly where bash emitted them relative to the text, strictly in byte order: `D;0` (previous command finished, exit 0) → `A` (new prompt starts) → `draw PROMPT$` → `draw echo hello-world` (the typed command echoed) → `screen_reset_mode 2004 1` (bracketed paste off as the line is submitted) → `C;cmdline=echo\ hello-world` (output region begins, carrying the command line) → `draw hello-world` (the command's output) → `screen_set_mode 2004 1` (bracketed paste re-armed) → `D;0` → `A` (next cycle). **No `133 B` appears anywhere** — consistent with §3.2. The `133 k;…` lines are kitty's own prompt-redraw region markers, byte-interleaved with the rest.

### 3.4 Observed (definitive): per-line attribution does not drift

The strongest proof that `C` attaches to the *correct line* is to ask kitty which lines it considers "last command output" — a query answered entirely from the per-line `prompt_kind` tags via `find_cmd_output` [kitty/screen.c:3527]. Run a command whose output is three known lines, then — **while the session is still live** (the `get-text` calls must precede the Ctrl-D exit) — query both extents:

```bash
# ... same launch as §3.3, then:
send 'unset HISTCONTROL\r'
send "PS1='PROMPT\$ '\r"
send 'printf "OUT-LINE-1\\nOUT-LINE-2\\nOUT-LINE-3\\n"\r'
"$KITTY" @ --to "unix:$sock" get-text --extent=last_cmd_output
"$KITTY" @ --to "unix:$sock" get-text --extent=screen
send '\x04'; wait "$launch_pid"
```

`get-text --extent=last_cmd_output` — unedited output (exactly the three output lines, nothing else):

```
OUT-LINE-1
OUT-LINE-2
OUT-LINE-3
```

`get-text --extent=screen` — unedited output (the whole screen, for contrast: two setup prompts, the command line, its output, and the current prompt):

```
root@reverse-code-generator-cb697179-zhdvx:/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e# unset HISTCONTROL
root@reverse-code-generator-cb697179-zhdvx:/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e# PS1='PROMPT$ '
PROMPT$ printf "OUT-LINE-1\nOUT-LINE-2\nOUT-LINE-3\n"
OUT-LINE-1
OUT-LINE-2
OUT-LINE-3
PROMPT$ 
```

**Cause → effect:** the `OSC 133 C` marker set `OUTPUT_START` on the exact line where output began [kitty/screen.c:2341]; `find_cmd_output` then walks from that line to the **next `PROMPT_START`** [kitty/screen.c:3539] to bound the range. Isolating exactly the three output lines — and **nothing** from the prompt or command lines — proves the marker did not drift. Note the range is bounded by the next `A`/`PROMPT_START`, **not** by `D` (§3.2). Had parsing been concurrent or reordered, the boundary line would be wrong; it is not.

### 3.5 Observed: the secondary (continuation) prompt

Opening an unterminated quote forces bash to emit its PS2 continuation prompt, which shell integration marks with `A;k=s`. Driving `echo 'one` ⏎ `two'` ⏎ (same harness as §3.3):

```bash
send 'unset HISTCONTROL\r'
send "PS1='PROMPT\$ '\r"
send "echo 'one\r"     # opens a single-quoted string -> bash shows PS2 continuation
send "two'\r"          # closes the quote -> command runs
send '\x04'; wait "$launch_pid"
sed -n '59,96p' "$d/dump.log"   # the secondary-prompt cycle
```

Unedited dump slice for the continuation cycle:

```
draw PROMPT$ 
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e
shell_prompt_marking 133 k;end_suffix_kitty
draw echo 'one
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 A;k=s
shell_prompt_marking 133 k;end_kitty
draw > two'
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
set_title echo 'onetwo'
shell_prompt_marking 133 C;cmdline=$'echo \'one\ntwo\''
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 0 32
shell_prompt_marking 133 k;end_suffix_kitty
draw one
screen_carriage_return
screen_linefeed
draw two
screen_carriage_return
screen_linefeed
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
draw PROMPT$ 
```

The sequence is, in exact byte order: `draw PROMPT$` (primary prompt) → `draw echo 'one` (first line) → **`133 A;k=s`** (the PS2 continuation prompt begins) → `draw > two'` (the `> ` continuation prompt and the second line) → `C;cmdline=$'echo \'one\ntwo\''` (output region, carrying the assembled multi-line command) → `draw one` / `draw two` (output) → `D;0` → `A` (next cycle). `133 A;k=s` is decoded by `parse_prompt_mark` into `SECONDARY_PROMPT` [kitty/screen.c:2321] — so the continuation prompt is attributed **distinctly** from the primary prompt, on its own line, without disturbing the surrounding text.

### 3.6 The cause → effect, stated plainly

Parsing and mutation are single-writer and byte-ordered: `run_worker` [kitty/vt-parser.c:1417] on the main thread consumes the buffer strictly in order, while the I/O thread (`KittyChildMon`) only hands raw bytes to the buffer (and writes queued input back to the child; §Q2) — it never parses or mutates the screen. Therefore an `A`/`C` marker attaches to `line_attrs[cursor->y]` for whatever line the in-order parse has reached [kitty/screen.c:2341]; since `cursor->y` is itself advanced by that same in-order parse (each `screen_linefeed` in the stream), the hint and the text share one clock and **cannot drift out of sync**.

---

## Q4 — Behavior under heavy backpressure or an unstable remote connection

### 4.1 Direct answer

Yes, behavior changes — but the change is a **readiness mask, not a drop**. When the shared 1 MiB buffer cannot accept more input, `vt_parser_has_space_for_input()` [kitty/vt-parser.c:1477] returns `false`, and the I/O loop stops asking the kernel for readable events on that child. Nothing is discarded; instead the PTY fills and the **child's own `write()` blocks** — OS-level flow control. When the main thread drains the buffer, space reopens and reads resume.

**Cause → effect chain:**
1. Buffer saturates → `vt_parser_has_space_for_input()` returns `false` (`ans = self->read.sz + self->write.pending < BUF_SZ` [kitty/vt-parser.c:1481]).
2. The I/O loop sets that child's poll events to `0` — POLLIN **masked** (`children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0` [kitty/child-monitor.c:1501]).
3. kitty stops reading the master → the PTY kernel buffer fills → the child's blocking `write()` to the slave **blocks**.
4. The main thread parses and drains via `run_worker` (gate: `if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16*1024 > BUF_SZ)` [kitty/vt-parser.c:1425] — note it parses *immediately* when nearly full) → space reopens → POLLIN restored → the child's `write()` unblocks.

The POLLOUT side (when *kitty* has data to send the child) is handled by `write_to_child` [kitty/child-monitor.c:1443] with `events |= write_buf_used ? POLLOUT : 0` [kitty/child-monitor.c:1503].

### 4.2 The gate, in code

```c
// kitty/vt-parser.c:1477
vt_parser_has_space_for_input(const Parser *p) {
    ...
    ans = self->read.sz + self->write.pending < BUF_SZ;   // :1481
    ...
}
```
```c
// kitty/child-monitor.c:1501
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT : 0);   // :1503
```

### 4.3 Observed: fast producer vs. slow drain (before / during / after)

**What this method can and cannot show.** Direct, non-mutating tracing of the parser's internal state (`self->read.sz`, `vt_parser_has_space_for_input()`, the per-fd `POLLIN` mask) is **not possible in this container**: `strace` is absent and `ptrace` is disabled (`cat /proc/sys/kernel/yama/ptrace_scope` → `3`), and no built-in flag dumps buffer occupancy. Adding instrumentation to the source is forbidden by the read-only mandate. Backpressure is therefore measured **indirectly**, by timing each individual `write()` from the child: because the child's PTY slave is a blocking fd [kitty/child.py:171], a `write()` that stalls on OS flow control appears as a slow write. This observes **generic producer backpressure** — the producer blocks when its consumer is slow; it does **not**, by itself, prove *which* buffer is the governing window (that attribution is code-derived; see the note after the data).

The producer writes fixed 4096-byte chunks to `stdout` (fd 1 — the timed path) and writes its one-line summary to a **separate result file**, so the stats survive even when fd 1 is the PTY (whose output kitty consumes and renders rather than returning to the shell):

```python
# $work/producer.py   ($work = mktemp -d, mode 0700; child program, removed afterward)
import os, sys, time
total = int(sys.argv[1]) * 1024 * 1024       # MiB to write to fd 1
resfile = sys.argv[2]                         # summary written here, not to the PTY
chunk = b'x' * 4096
n = total // 4096
slow = 0; mx = 0.0; t0 = time.monotonic()
for _ in range(n):
    a = time.monotonic(); os.write(1, chunk); d = time.monotonic() - a
    if d > mx: mx = d
    if d > 0.001: slow += 1
dt = time.monotonic() - t0
line = ("MB=%d dt=%.4f max_write_us=%.1f slow=%d MBps=%.1f\n"
        % (total//(1024*1024), dt, mx*1e6, slow, (total/1e6)/dt))
with open(resfile, "w") as f:
    f.write(line)
```

Every run below is bounded by `timeout`; the kitty runs use `close_on_child_death=yes` so kitty exits cleanly when the producer finishes, and only the spawned PID is torn down.

**BASELINE — no terminal (write to `/dev/null`), repeated 3× at each scale:**
```bash
$ work=$(mktemp -d "${TMPDIR:-/tmp}/kitty_q4.XXXXXX"); chmod 0700 "$work"
$ for r in 1 2 3; do timeout 60 python3 "$work/producer.py" 16 "$work/b16_$r.txt" >/dev/null; cat "$work/b16_$r.txt"; done
MB=16 dt=0.0021 max_write_us=13.1 slow=0 MBps=7856.1
MB=16 dt=0.0022 max_write_us=57.7 slow=0 MBps=7634.0
MB=16 dt=0.0021 max_write_us=11.8 slow=0 MBps=7892.6
$ for r in 1 2 3; do timeout 60 python3 "$work/producer.py" 64 "$work/b64_$r.txt" >/dev/null; cat "$work/b64_$r.txt"; done
MB=64 dt=0.0086 max_write_us=58.2 slow=0 MBps=7801.3
MB=64 dt=0.0091 max_write_us=11.9 slow=0 MBps=7407.9
MB=64 dt=0.0085 max_write_us=11.9 slow=0 MBps=7875.9
```
No backpressure at either scale: **0** slow writes; throughput ≈ 7.4–7.9 GB/s. The maximum single `write()` is **tens of microseconds** — 16 MiB: 11.8–57.7 µs; 64 MiB: 11.9–58.2 µs — with the larger values being scheduler jitter, not flow control. (This corrects an earlier draft's "≈ 3 µs", which cherry-picked one fast 16-MiB sample and ignored the ~49–58 µs maxima at 64 MiB.)

**UNDER KITTY — real PTY, 16 MiB, 3 runs:**
```bash
$ for r in 1 2 3; do \
    timeout 120 xvfb-run -a -s "-screen 0 1280x800x24" \
      ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
      python3 "$work/producer.py" 16 "$work/k16_$r.txt" >/dev/null 2>&1; \
    cat "$work/k16_$r.txt"; done
MB=16 dt=0.1450 max_write_us=10741.2 slow=15 MBps=115.7
MB=16 dt=0.1396 max_write_us=8853.9 slow=15 MBps=120.2
MB=16 dt=0.1415 max_write_us=9759.8 slow=15 MBps=118.6
```

**UNDER KITTY — 64 MiB (larger scale), 3 runs:**
```bash
$ for r in 1 2 3; do \
    timeout 180 xvfb-run -a -s "-screen 0 1280x800x24" \
      ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
      python3 "$work/producer.py" 64 "$work/k64_$r.txt" >/dev/null 2>&1; \
    cat "$work/k64_$r.txt"; done
MB=64 dt=0.5857 max_write_us=11237.0 slow=64 MBps=114.6
MB=64 dt=0.6031 max_write_us=11321.6 slow=64 MBps=111.3
MB=64 dt=0.5733 max_write_us=10512.8 slow=63 MBps=117.0
```

**CONTROL — generic slow consumer (a plain pipe, no terminal, no VT parser).** The same producer is piped to a reader that sleeps 1 ms per 64 KiB read. If the producer blocks here too, the blocking is generic `write()` flow control — not something parser-specific:
```bash
$ cat > "$work/slow_reader.py" <<'EOF'
import os, time
while True:
    b = os.read(0, 65536)
    if not b: break
    time.sleep(0.001)                         # deliberately slow drain
EOF
$ for r in 1 2; do timeout 120 sh -c "python3 '$work/producer.py' 16 '$work/ctl_$r.txt' | python3 '$work/slow_reader.py'"; cat "$work/ctl_$r.txt"; done
MB=16 dt=0.2757 max_write_us=1129.5 slow=255 MBps=60.8
MB=16 dt=0.2759 max_write_us=1107.3 slow=255 MBps=60.8
```

**Interpretation, before / during / after (at write granularity):**
- **BEFORE / AFTER (consumer keeping up):** the vast majority of writes complete in microseconds — the producer never blocks.
- **DURING (consumer behind):** a small number of writes block for milliseconds. Under kitty these are the moments OS flow control stalls the child's `write()` to the PTY slave while the terminal is behind. **Code-derived mechanism** (not directly traced here): when the shared 1 MiB buffer cannot accept more, `vt_parser_has_space_for_input()` returns `false` [kitty/vt-parser.c:1481] and the I/O loop masks `POLLIN` for that child [kitty/child-monitor.c:1501]; the PTY then fills and the child's blocking `write()` stalls until the main thread drains via `run_worker` [kitty/vt-parser.c:1425].

**Magnitude & stability (the runs above are the raw, unedited dataset).**
- **Throughput collapse:** through the terminal, ≈ 111–120 MB/s at both scales, vs. ≈ 7.4–7.9 GB/s to `/dev/null`. Computed per-scale (baseline mean ÷ kitty mean): **≈ 66× at 16 MiB** (range ≈ 63–68×) and **≈ 67× at 64 MiB** (range ≈ 63–71×). The two scales agree this session, but the exact multiplier is **environment-dependent** because the `/dev/null` baseline itself varies run-to-run; a single fixed factor (e.g. an earlier draft's "51×") is selective and is not claimed.
- **Max single write:** jumps from tens of µs (baseline) to **~8.9–11.3 ms** under kitty — stable across all three runs at both scales.

**Why the slow-write count is *not* a "1 MiB buffer fingerprint."** Under kitty the count of blocking writes is 15 (16 MiB) and 63–64 (64 MiB) — about one per MiB. This count is a property of **generic producer/consumer flow control, not a signature of a specific buffer**: the CONTROL run (a plain pipe with a ~64 KiB kernel buffer and a deliberately slow reader) blocks **255** times for the same 16 MiB — ~17× more often — purely because its buffering/drain profile differs. Because child-`write()` timing cannot observe parser occupancy or the `POLLIN` mask (no `ptrace`/`strace`, no built-in dump), it **cannot attribute** the ~1-block-per-MiB rate under kitty to the 1 MiB `BUF_SZ` [kitty/vt-parser.c:18] specifically. That the parser buffer is the governing window is **code-derived** from the gate in §4.2, not measured here.

### 4.4 The unstable remote connection

Remote control reaches kitty through **three distinct transports**, which an earlier draft conflated ("same VT path"). They are separated here, each with its own observed evidence and code path.

#### (a) Socket remote control — does **not** traverse the VT parser

A `kitty @ --to unix:SOCK …` client connects to kitty's listening Unix socket. Those bytes are read by the **talk thread**, never by `read_bytes()`: `talk_loop()` [kitty/child-monitor.c:1805] → `read_from_peer()` uses `recv(peer->fd, …)` into a per-peer buffer capped at 64 KiB [kitty/child-monitor.c:1714, kitty/child-monitor.c:1722, kitty/child-monitor.c:1717] → `dispatch_peer_command()` [kitty/child-monitor.c:1699] → `queue_peer_message()` [kitty/child-monitor.c:1653], which locks `talk_mutex`, appends the command to the **separate** `self->messages` queue, sets `is_remote_control_peer`, and calls `wakeup_main_loop()` [kitty/child-monitor.c:1666, kitty/child-monitor.c:1669]. The main thread then, inside `parse_input()`, copies that queue under `talk_mutex` [kitty/child-monitor.c:485-497] and dispatches each message via `peer_message_received` [kitty/child-monitor.c:504 → kitty/boss.py:776] **before** it parses live child output (`do_parse` at [kitty/child-monitor.c:530]).

Cause → effect: socket RC is executed **on the main thread, serialized with — and ahead of — VT parsing**; it is *not* "answered off the main thread," and it never enters the 1 MiB VT buffer. It is bounded instead by the 64 KiB per-peer `recv` buffer.

Observed round-trip (safe harness: private mode-0700 dir, unique socket, `allow_remote_control=socket-only`, bounded `timeout`, PID-scoped teardown). The `ls` reply includes a per-window `env` block that contains live secrets, so it is removed by the filter shown below; nothing else is edited, and a leak check confirms no secret remains:

```bash
$ rc=$(mktemp -d "${TMPDIR:-/tmp}/kitty_rc.XXXXXX"); chmod 0700 "$rc"; sock="$rc/rc.sock"
$ cat > "$rc/strip_env.py" <<'EOF'
import sys, json
d = json.load(sys.stdin)
for osw in d:
    for tab in osw.get("tabs", []):
        for w in tab.get("windows", []):
            w.pop("env", None)          # drop sensitive env vars only
print(json.dumps(d, indent=2))
EOF
$ timeout 60 xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
      -o allow_remote_control=socket-only --listen-on "unix:$sock" sh -c 'sleep 30' >/dev/null 2>&1 &
$ kpid=$!; trap 'kill "$kpid" 2>/dev/null; wait "$kpid" 2>/dev/null; rm -rf "$rc"' EXIT  # abort-safe teardown
$ for i in $(seq 1 50); do [ -S "$sock" ] && break; sleep 0.1; done
$ timeout 20 ./kitty/launcher/kitty @ --to "unix:$sock" ls 2>/dev/null | python3 "$rc/strip_env.py"
[
  {
    "background_opacity": 1.0,
    "id": 1,
    "is_active": true,
    "is_focused": true,
    "last_focused": true,
    "platform_window_id": 2097164,
    "tabs": [
      {
        "active_window_history": [
          1
        ],
        "enabled_layouts": [
          "fat",
          "grid",
          "horizontal",
          "splits",
          "stack",
          "tall",
          "vertical"
        ],
        "groups": [
          {
            "id": 1,
            "windows": [
              1
            ]
          }
        ],
        "id": 1,
        "is_active": true,
        "is_focused": true,
        "layout": "fat",
        "layout_opts": {
          "bias": 50,
          "full_size": 1,
          "mirrored": false
        },
        "layout_state": {
          "biased_map": {},
          "main_bias": [
            0.5,
            0.5
          ],
          "num_full_size_windows": 1
        },
        "title": "sh",
        "windows": [
          {
            "at_prompt": false,
            "cmdline": [
              "sh",
              "-c",
              "sleep 30"
            ],
            "columns": 90,
            "created_at": 1784065558205513032,
            "cwd": "/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e",
            "foreground_processes": [
              {
                "cmdline": [
                  "sh",
                  "-c",
                  "sleep 30"
                ],
                "cwd": "/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e",
                "pid": 154983
              },
              {
                "cmdline": [
                  "sleep",
                  "30"
                ],
                "cwd": "/tmp/blitzy/kitty/blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8_8fa74e",
                "pid": 154984
              }
            ],
            "id": 1,
            "is_active": true,
            "is_focused": true,
            "is_self": false,
            "last_cmd_exit_status": 0,
            "last_reported_cmdline": "",
            "lines": 24,
            "pid": 154983,
            "title": "sh",
            "user_vars": {}
          }
        ]
      }
    ],
    "wm_class": "kitty",
    "wm_name": "kitty"
  }
]
$ kill "$kpid" 2>/dev/null; wait "$kpid" 2>/dev/null; rm -rf "$rc"   # scoped teardown + cleanup
```

#### (b) TTY-carried remote control — **does** traverse the VT parser

When there is no socket, `kitty @ ls` writes the command as a DCS escape (`\x1bP@kitty-cmd{…}\x1b\\`) to its controlling TTY — i.e. into kitty's ordinary child-output stream. That path goes through `read_bytes()` [kitty/child-monitor.c:1531] into the 1 MiB VT buffer and is dispatched by the parser: `dispatch_dcs()` [kitty/vt-parser.c:620] `case '@'` [kitty/vt-parser.c:654] → `parse_kitty_dcs()` [kitty/vt-parser.c:586] → `dispatch("cmd{", handle_remote_cmd, 1)` [kitty/vt-parser.c:603] → `screen_handle_kitty_dcs()` [kitty/screen.c:2441]. This was observed directly in the `--dump-commands` VT stream (own private 0700 `mktemp -d`, its own `trap`, `close_on_child_death=yes` so kitty exits when the child finishes and the block-buffered dump flushes):

```bash
$ rc=$(mktemp -d "${TMPDIR:-/tmp}/kitty_ttyrc.XXXXXX"); chmod 0700 "$rc"
$ trap 'rm -rf "$rc"' EXIT
$ dump="$rc/tty_rc_dump.log"
$ timeout 60 xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
      -o allow_remote_control=yes -o close_on_child_death=yes --dump-commands \
      sh -c 'kitty @ ls >/dev/null 2>&1; sleep 0.3' >"$dump" 2>/dev/null
$ grep -n 'handle_remote_cmd' "$dump"
16:handle_remote_cmd {"cmd":"ls","version":[0,26,0],"kitty_window_id":1,"payload":{}}
$ rm -rf "$rc"                                                     # explicit cleanup (also via trap)
```

That `handle_remote_cmd` line is the parser dispatching the RC command **out of the VT stream** — the concrete difference from transport (a), whose command never appears in this dump. Both this run and a repeat gave `kitty_exit=0`, a 19-line dump, and `handle_remote_cmd` at line 16.

**Why `allow_remote_control=yes` here (and why it is safe in this block).** Unlike the socket experiments (a), which are locked to `socket-only` on a unique socket inside a 0700 directory, this block opens **no** socket — `--listen-on` is absent — so there is no predictable path or listening endpoint for a co-tenant process to reach, and the socket-hijack concern does not apply. The *only* remote-control ingress is the controlling TTY of the single child kitty spawned itself (`sh -c 'kitty @ ls …'`), which lives and dies inside the private 0700 directory under `timeout`. `socket-only` cannot be used because it explicitly **denies** TTY-carried requests [kitty/options/definition.py:2982-2985], which would suppress the exact dispatch being demonstrated; and `password` [kitty/options/definition.py:2978-2980] did not complete non-interactively in this harness — the in-child `kitty @` client blocked waiting on the TTY password handshake, so kitty exited only on the outer `timeout` (`kitty_exit=124`) and no `handle_remote_cmd` was emitted. `yes` [kitty/options/definition.py:2995-2996] is therefore the minimal setting under which the TTY dispatch is observable here, and its blast radius is bounded to kitty's own short-lived child.

#### (c) SSH child output — ordinary child output

`ssh` is just another child; its stdout is read by `read_bytes()` and parsed exactly like any other program's output (the Q1 path). The SSH **kitten** additionally deploys terminfo + shell integration to the remote host and can carry RC over the TTY via the `kitty-ssh|` DCS, handled by the same `parse_kitty_dcs()` above [kitty/vt-parser.c:608]. Its sources are `kittens/ssh/{main.py, main.go, config.go, askpass.go, utils.go}`; `kitty/remote_control.py` is the command surface:

```bash
$ timeout 30 ./kitty/launcher/kitty +kitten ssh --help | sed -n '1,7p'
Usage: kitten ssh arguments for the ssh command

The ssh kitten is a thin wrapper around the ssh command. It automatically
enables shell integration on the remote host, re-uses existing connections to
reduce latency, makes the kitty terminfo database available, etc. Its invocation
is identical to the ssh command. For details on its usage, see Truly convenient
SSH.
```

#### Does an unstable link behave differently? (real attempt)

A genuine authenticated SSH connection could **not** be established in this container — there is no SSH server (client only):

```bash
$ ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=no localhost 'echo hello-from-remote'; echo "exit=$?"
ssh: connect to host localhost port 22: Cannot assign requested address
exit=255
```

So the specific **network-fault degradation profile is unverified (code-derived)**, and is stated as such rather than asserted. What *is* observed and directly relevant is the safety valve from Q1: if a synchronized-update pause (which a stalled link can trigger) is not resumed within its timeout, `screen_check_pause_rendering()` [kitty/screen.c:2490] force-resumes rendering, so the display never freezes waiting on a hung producer. For the two transports that reach the VT buffer — (b) and (c) — link instability manifests as the **same OS flow control** measured in §4.3; transport (a) is instead bounded by the 64 KiB per-peer `recv` buffer [kitty/child-monitor.c:1717] and drained on the main thread.

---

## Q5 — End-to-end: from the moment mixed input arrives to the moment the interface settles — *how the moving parts keep their rhythm*

### 5.1 Direct answer

Three pacing timers give the pipeline its rhythm and let the interface settle:

| Timer | Default | Line | Role |
|-------|---------|------|------|
| `input_delay` | **3 ms** | [kitty/options/definition.py:878] | Coalesces reads/wakeups — batches many small child writes into one parse+render tick. |
| `repaint_delay` | **10 ms** | [kitty/options/definition.py:866] | Throttles repaints to a ~10 ms cadence **only when idle**; the throttle is *bypassed* while there is pending input to process (§5.2). |
| `resize_debounce_time` | **(0.1, 0.5) s** | [kitty/options/definition.py:1182] | Debounces live-resize redraw. `on_end` = **0.1 s** = `tuple[0]`; `on_pause` = **0.5 s** = `tuple[1]` [kitty/options/to-c.h:349-350] — platform-dependent roles in §5.2. |

The default *values* are confirmed at runtime from the built binary — this establishes the configured numbers, **not** the timing behavior itself (the behavior is analyzed as cause → effect in §5.2, and where it cannot be directly instrumented in the canonical build it is labeled code-derived; see the observability note in §5.4). Stable across two runs:

```bash
$ timeout 30 ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; \
    print("input_delay", d.input_delay); print("repaint_delay", d.repaint_delay); \
    print("resize_debounce_time", d.resize_debounce_time)'
```
```
input_delay 3
repaint_delay 10
resize_debounce_time (0.1, 0.5)
```

These match `opt('input_delay', '3', ...)` [kitty/options/definition.py:878], `opt('repaint_delay', '10', ...)` [kitty/options/definition.py:866], and `opt('resize_debounce_time', '0.1 0.5', ...)` [kitty/options/definition.py:1182].

### 5.2 How each timer wires the rhythm (cause → effect)

- **`input_delay` (3 ms) — coalescing.** The `WAKEUP` gate wakes the main loop only after `input_delay` since the last wakeup [kitty/child-monitor.c:1566], the poll timeout is bounded by it [kitty/child-monitor.c:1506-1512], and `run_worker` consumes only when `time_since_new_input >= OPT(input_delay)` **unless** flushing or nearly full [kitty/vt-parser.c:1425]. Effect: small writes within a 3 ms window collapse into a single parse+render tick. The `input_delay` option's own long-form doc says it "is ignored when the input buffer is almost full" — which is precisely the `self->read.sz + 16*1024 > BUF_SZ` clause of that gate.
- **`repaint_delay` (10 ms) — throttling.** In `render()`:
  ```c
  // kitty/child-monitor.c:871
  render(monotonic_t now, bool input_read) {
      ...
      if (!input_read && time_since_last_render < OPT(repaint_delay)) {   // :875
          set_maximum_wait(OPT(repaint_delay) - time_since_last_render);
          return;
      }
  ```
  Effect: when idle, repaints are throttled to a ~10 ms cadence; but the throttle is **bypassed when `input_read` is true** — matching the option's long-form doc that it "is ignored ... when there is pending input to be processed," so latency stays low exactly when it matters.
- **`resize_debounce_time` ((0.1, 0.5) s) — debouncing.** The tuple is unpacked into a struct by `resize_debounce_time()` [kitty/options/to-c.h:348]: **`on_end = tuple[0] = 0.1 s`** [kitty/options/to-c.h:349] and **`on_pause = tuple[1] = 0.5 s`** [kitty/options/to-c.h:350]. (An earlier draft had these two reversed.) `process_pending_resizes()` [kitty/child-monitor.c:1043] then applies them **platform-dependently**, exactly as the option's own long-form doc describes [kitty/options/definition.py:1182-1192]:
  - **On OS-notification platforms (e.g. macOS)**, where the OS marks the start/end of a live resize (`live_resize.from_os_notification` [kitty/child-monitor.c:1049]): when the OS reports the resize is complete (`os_says_resize_complete` [kitty/child-monitor.c:1050]) the redraw is **immediate** and `on_end` (0.1 s) is *ignored*; while resizing is merely paused (not yet ended), `on_pause` = **0.5 s** is the redraw-after-pause debounce [kitty/child-monitor.c:1055].
  - **On other platforms (e.g. Linux/X11)**, only `on_end` = **0.1 s** is used [kitty/child-monitor.c:1062]: kitty redraws once no new resize event has arrived for 0.1 s, so it becomes "ready" quickly after the drag ends without continuously repainting (to save energy).
  A committed resize calls `resize_pty` [kitty/child-monitor.c:592] → `pty_resize` [kitty/child-monitor.c:577]; the Python side calls `boss.child_monitor.resize_pty(...)` [kitty/window.py:863] and sends `SIGWINCH` to the child [kitty/window.py:873].

### 5.3 Observed: the resize → SIGWINCH hand-off

`--debug-rendering` surfaces the resize hand-off because [kitty/window.py:873] prints `SIGWINCH sent to child`:

```bash
# safe harness: private mode-0700 dir + unique socket + socket-only RC + PID-scoped teardown.
$ rc=$(mktemp -d "${TMPDIR:-/tmp}/kitty_rz.XXXXXX"); chmod 0700 "$rc"; sock="$rc/rc.sock"
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
      -o close_on_child_death=yes -o allow_remote_control=socket-only \
      --listen-on "unix:$sock" --debug-rendering sh -c 'sleep 6' 2>"$rc/err.log" &
$ kpid=$!; trap 'kill "$kpid" 2>/dev/null; wait "$kpid" 2>/dev/null; rm -rf "$rc"' EXIT  # abort-safe teardown
$ for i in $(seq 1 50); do [ -S "$sock" ] && break; sleep 0.1; done; sleep 0.4
$ ./kitty/launcher/kitty @ --to "unix:$sock" resize-os-window --width 100 --height 30
$ sleep 0.4
$ ./kitty/launcher/kitty @ --to "unix:$sock" resize-os-window --width 90 --height 24
$ sleep 0.5; kill "$kpid" 2>/dev/null; wait "$kpid" 2>/dev/null   # scoped teardown
$ grep -E 'SIGWINCH|Child launched' "$rc/err.log"; rm -rf "$rc"
```
Observed (run 1; byte-identical size tuples on run 2, timestamps [0.760]/[1.262]):
```
[0.164] Child launched
[0.761] SIGWINCH sent to child in window: 1 with size: (30, 100, 900, 540)
[1.264] SIGWINCH sent to child in window: 1 with size: (24, 90, 810, 432)
```
The size tuple is `(lines, cols, width_px, height_px)`. Each discrete resize produces exactly one `SIGWINCH` via `resize_pty`, confirming kitty *sends* the resize signal to the child rather than receiving it. The two signals are ~0.50 s apart, matching the two RC commands issued ~0.4 s apart plus dispatch — a real, reproducible timing observation of the committed-resize path (as distinct from the live-drag debounce, which is code-derived below).

> **Honesty label.** The live-drag *debounce coalescing* (many intermediate sizes collapsing within the 0.1–0.5 s window) is not reproducible under bare `xvfb`, which has no window-manager resize-event stream. The timer values and the `process_pending_resizes` mechanism are observed/code-grounded; the intermediate coalescing itself is **inferred (code-derived)**.

### 5.4 Observed: the end-to-end settling trace

To watch *the moving parts keep their rhythm*, a single child emits a realistic mix — ordinary text, an OSC 133 prompt marker, a prompt, an OSC 133 output marker, a synchronized-update block, then a trailing line — and `--dump-commands` records the whole settle in byte order:

```bash
$ work=$(mktemp -d "${TMPDIR:-/tmp}/kitty_e2e.XXXXXX"); chmod 0700 "$work"
$ cat > "$work/child_e2e.sh" <<'EOF'
printf 'ordinary text line 1\n'
printf '\033]133;A\033\\'; printf 'prompt$ '
printf '\033]133;C\033\\'
printf '\033[?2026h'                         # BSU (begin synchronized update)
printf 'sync line 1\n'; printf 'sync line 2\n'
printf '\033[?2026l'                         # ESU (end synchronized update)
printf '\033]133;D;0\033\\'
printf 'settled tail line\n'
EOF
$ timeout 40 xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes --dump-commands sh "$work/child_e2e.sh"; rm -rf "$work"
```

Unedited command stream — this shows the **byte-order** in which the parser consumed the mix (it is *not* a tick/frame trace); **identical across two runs (18 lines, deterministic)**:

```
draw ordinary text line 1
screen_carriage_return
screen_linefeed
shell_prompt_marking 133 A
draw prompt$
shell_prompt_marking 133 C
screen_set_mode 2026 1
draw sync line 1
screen_carriage_return
screen_linefeed
draw sync line 2
screen_carriage_return
screen_linefeed
screen_reset_mode 2026 1
shell_prompt_marking 133 D;0
draw settled tail line
screen_carriage_return
screen_linefeed
```

**Reading the rhythm (cause → effect), from arrival to settle.** The dump above *directly shows* one thing — the **byte-order** of parsing (observed); the timing/coalescing/render steps around it are **code-derived** from the cited lines (they are not visible in this command trace):
1. *(code-derived)* Bytes arrive and are read into the shared buffer by the I/O thread; the `WAKEUP` gate is designed to coalesce wakeups over `input_delay` (3 ms) [kitty/child-monitor.c:1566] so a burst is handed to the main thread together. The dump does not expose this coalescing; it is inferred from the gate.
2. *(observed)* The main thread runs one tick in fixed order: `process_pending_resizes` → `parse_input` → `render` [kitty/child-monitor.c:1233-1237]. `parse_input` consumes the buffer **in byte order**, so ordinary text, the OSC 133 `A`/`C` markers, and the `2026` block land in the exact order emitted — which is precisely what the 18-line stream shows.
3. *(observed set/reset; code-derived effect)* The dump shows `screen_set_mode 2026 1` … `screen_reset_mode 2026 1` bracketing the two `sync line` draws (observed). Per the code, the reset flips `is_dirty = true` [kitty/screen.c:2511] so the block is flushed together; that atomic-flush *effect* is code-derived, not shown by the dump.
4. *(code-derived)* `render()` throttles to the `repaint_delay` (10 ms) cadence only when idle (`if (!input_read && …)` [kitty/child-monitor.c:875]) and **bypasses** the throttle while `input_read` is true — so a burst renders promptly. That the interface **settles** on `settled tail line` "with no half-drawn frame" is a *display* property of the paused-render mechanism; the command dump does not prove it, so it is labeled code-derived (§1 covers the pause snapshot directly).

**Determinism across runs.** The 18-line stream was byte-identical on both runs, so no distribution is needed for the *ordering*. For the *magnitude/timing* values reported elsewhere (§4.3 throughput, §1 2000 ms pause boundary, §5.3 SIGWINCH spacing), stability across ≥ 2 runs was likewise confirmed.

> **Observability honesty note.** The per-tick and per-frame *counts and timings* of `input_delay`/`repaint_delay` are **not directly instrumentable in the canonical build**: `EVDBG` is `#ifdef DEBUG_EVENT_LOOP` (compiled out; §Q2), `--debug-rendering` emits no per-render/frame timestamps (only OS-window/child-launch/GL/SIGWINCH lines), and there is no runtime frame counter — and adding instrumentation is forbidden by the read-only mandate. What is **observed** here: the default timer *values* (from the binary, §5.1), the byte-order of the settle (§5.4), and the resize→SIGWINCH hand-off with real timestamps (§5.3). The 3 ms coalescing and 10 ms repaint *behaviors* themselves are **code-derived** from the cited gates, not measured.

---

## Appendix A — External conventions (framing only; observed code is the source of truth)

These public conventions frame the narrative; every behavioral claim above rests on observed kitty output, not on these:

- **Synchronized output / DEC private mode 2026.** `CSI ? 2026 h` begins a synchronized update (BSU — batch output), `CSI ? 2026 l` ends it (ESU — apply atomically), so the viewer never sees a half-drawn screen. There is no cross-terminal consensus on a timeout, which is why kitty enforces its own (2000 ms, observed in §Q1). Kitty implements this via `PENDING_UPDATE (2026 << 5)` [kitty/modes.h:86].
- **OSC 133 prompt marking (FTCS).** The public convention defines `A` = prompt start, `B` = prompt end / command start, `C` = command output start, `D;<exit>` = command finished, `k=s` = secondary prompt. **Kitty handles only `A`/`C`/`D` — there is no `case 'B'`** (§3.2): `A` and `C` are stored per line as `prompt_kind` [kitty/screen.c:2337, :2341], while `D` fires the `cmd_output_marking` callback with the exit status and stores no per-line tag.
- **Bracketed paste / mode 2004.** Pasted text is wrapped in `ESC[200~ … ESC[201~`, so applications can distinguish pasted from typed input. Kitty: `BRACKETED_PASTE (2004 << 5)` [kitty/modes.h:81], start `"200~"` [:82], end `"201~"` [:83].

---

## Appendix B — Coverage pass

Every sub-question and every named item, with its concrete value, the evidence type, and the citation.

### Q1 — Raw-input entry point + pause/resume
- [x] **Entry point** `read_bytes()` — [kitty/child-monitor.c:1337]; `read(fd, buf, …)` at :1345 — *observed* via `--dump-bytes` (§1.3).
- [x] **Zero-copy into shared buffer** `vt_parser_create_write_buffer` [kitty/vt-parser.c:1451] (`*sz = BUF_SZ - offset` :1457), `vt_parser_commit_write` [:1465] — *code* (§1.2).
- [x] **Buffer size 1 MiB** `BUF_SZ (1024u*1024u)` [kitty/vt-parser.c:18]; `BUF_EXTRA` :20; `MAX_ESCAPE_CODE_LENGTH` :21 — *observed* `VT_PARSER_BUFFER_SIZE = 1048576` (§1.2).
- [x] **Exact boundary bytes** `1b 5b 3f 32 30 32 36 68 … 6c` — *observed* hexdump (§1.3).
- [x] **DEC 2026 pause/resume** `case PENDING_MODE << 5` [kitty/screen.c:1174-1175]; `PENDING_UPDATE (2026<<5)` [kitty/modes.h:86] — *observed* `screen_set_mode/reset_mode 2026 1` (§1.4).
- [x] **Legacy DCS `=1s`/`=2s`** [kitty/vt-parser.c:636-648] — *observed* `screen_start/stop_pending_mode` (§1.4), converges on `screen_pause_rendering`.
- [x] **`screen_pause_rendering()`** [kitty/screen.c:2506]; resume `is_dirty=true` :2511; refuse-if-not-paused :2508; refuse-if-paused :2518 — *code + observed* (§1.4, §1.5).
- [x] **2000 ms default timeout** [kitty/screen.c:2521-2522]; `screen_check_pause_rendering` force-resume [:2489-2490] — *observed* boundary (1.5 s no-error vs 2.5 s timeout, ×2) (§1.5).
- [x] **Before / during / after** — *observed at the command layer*: `screen_set_mode 2026 1` at line 1 → all 50 `draw SURGE line NN` commands parse in byte order during the pause (lines 1–152) → `screen_reset_mode 2026 1` at line 152 then `draw AFTER-ESU visible line` at line 153. The display-side snapshot (`expires_at`, frozen linebuf) and the single atomic on-resume frame are **code-derived** [kitty/screen.c:2522-2543, :2737-2760, :2511], not captured as pixels (§1.4).
- [x] **Both refusal paths** (double-pause; late-resume) — *observed* error strings (§1.5).
- [x] **Input side** `--debug-input` (glfw ingress) — *observed*; key-encoding stage labeled **inferred** (§1.6).
- [x] **PTY fd origin** `openpty` [kitty/child.py:170], `Child.fork` [:276], `child_fd=master` [:338], `set_blocking(…,False)` [:345] — *code* (§1.2).
- [x] **Bracketed paste 2004** [kitty/modes.h:81-83]; `paste_with_actions` [kitty/window.py:1643], `paste_bytes` [:1707], `paste_text` [:1713], `paste_()` [kitty/screen.c:4573], `in_bracketed_paste_mode` [:3854] — *observed* `screen_set/reset_mode 2004 1` (§1.6).

### Q2 — The unseen conductor
- [x] **Three threads** `io_loop` (`KittyChildMon`) [kitty/child-monitor.c:1481/:1489], `main_loop`→`process_global_state` [:1259/:1224], `talk_loop` (`KittyPeerMon`) [:1805/:1808] — *observed* in `/proc/<pid>/task/*/comm`, stable ×2 (§2.2).
- [x] **Fixed per-tick order** `process_pending_resizes` [:1233] → `parse_input` [:1236] → `render` [:1237] — *code* (§2.3); the literal answer to "what goes first." `EVDBG` compile-gated [:29-33] labeled **honesty note**.
- [x] **Poll fd ordering** wakeup [:183]/signal → child [EXTRA_FDS :35]; process order wakeup [:1515] → signals [:1516-1519] → child reads [:1530-1533] — *code* (§2.4).
- [x] **Signals** `KITTY_HANDLED_SIGNALS` [:121], `handle_signal` [:1362], `SIGCHLD` [:1370] — *observed* clean exit on child death (§2.4); `SIGWINCH` not received (sent to child).
- [x] **Wakeup coalescing** `WAKEUP` macro [:1562], `input_delay` gate [:1566/:1569], poll timeout bound [:1506-1512] — *code* (§2.5).

### Q3 — Coherence without drifting out of sync
- [x] **Single-thread in-order parse** `parse_input` [kitty/child-monitor.c:451] → `run_worker` [kitty/vt-parser.c:1417] — *code* (§3.1, §3.6).
- [x] **OSC 133 dispatch** `dispatch_osc` [kitty/vt-parser.c:457], `case 133` [:536], call [:544] — *code* (§3.2).
- [x] **Per-line attribution** `shell_prompt_marking` [kitty/screen.c:2328] (cases `A` :2332, `C` :2340, `D` :2350 — **no `B`**), `line_attrs[cursor->y].prompt_kind` [:2337, :2341], `parse_prompt_mark` [:2316], `SECONDARY_PROMPT` [:2321]; output-range end via `find_cmd_output` at the next `PROMPT_START` [:3527, :3539] (**not** `D`) — *observed* callbacks + `get-text --extent=last_cmd_output` isolating exactly the output lines (§3.3, §3.4).
- [x] **Hint origin** `modify_shell_environ` [kitty/shell_integration.py:218]; `ENV`→`kitty.bash` + argv→`bash --posix` [:134, :146]; bash/zsh/fish scripts — *code + observed* (§3.2).
- [x] **Markers `A` / `C` / `D` (+ `k=s`, + kitty's own `k;…`)** — *observed* in the byte-order dump: `A`/`C` set per-line `prompt_kind`, `D` fires the exit-status callback only; secondary prompt `A;k=s` observed (§3.3, §3.5). **`B` is a public convention kitty neither emits nor handles** (no `case 'B'`, §3.2) — it was **not** observed and is **not** claimed.

### Q4 — Backpressure / unstable remote
- [x] **Space gate** `vt_parser_has_space_for_input` [kitty/vt-parser.c:1477] (`read.sz + write.pending < BUF_SZ` :1481) — *code* (§4.2).
- [x] **POLLIN masking** `events = has_space ? POLLIN : 0` [kitty/child-monitor.c:1501]; POLLOUT [:1503]; `write_to_child` [:1443] — *code* (§4.2).
- [x] **`run_worker` drain gate** [kitty/vt-parser.c:1425] — *code* (§4.1).
- [x] **Observed backpressure (generic producer)** throughput collapse ≈66× @16 MiB / ≈67× @64 MiB (≈111–120 MB/s vs ≈7.4–7.9 GB/s), max single write tens-of-µs → ~8.9–11.3 ms, **stable ×3** at both scales; slow-write count ≈1/MiB, but a CONTROL slow pipe blocks 255×/16 MiB — so the count is *not* a 1 MiB fingerprint (§4.3).
- [x] **Before/during/after** at write granularity — *observed*; parser saturation (`has_space_for_input`→false [kitty/vt-parser.c:1481], POLLIN mask [kitty/child-monitor.c:1501]) is *code-derived*, not traced (no ptrace/strace) (§4.3).
- [x] **Three transports separated** — (a) socket RC `talk_loop` [kitty/child-monitor.c:1805] → `queue_peer_message` [:1653] → `peer_message_received` [:504 → kitty/boss.py:776] on the main thread, *not* the VT buffer; (b) TTY-carried RC via `dispatch_dcs` [kitty/vt-parser.c:620] → `parse_kitty_dcs` [:586] → `handle_remote_cmd` [:603] — *observed* in `--dump-commands`; (c) SSH child output = ordinary path (§4.4).
- [x] **SSH kitten** `kittens/ssh/`; `remote_control.py` surface — *observed* `+kitten ssh --help`; real `ssh localhost` attempt → no server (exit 255) (§4.4).
- [x] **Unstable link** — (b)/(c) share the OS flow control of §4.3; (a) is bounded by the 64 KiB per-peer buffer [kitty/child-monitor.c:1717]; 2000 ms force-resume [kitty/screen.c:2490] *observed*; network-fault degradation labeled **code-derived/unverified** (§4.4).

### Q5 — Settling & rhythm
- [x] **`input_delay` 3 ms** [kitty/options/definition.py:878] — *observed* default from binary; gate [kitty/child-monitor.c:1566], [kitty/vt-parser.c:1425] (§5.1, §5.2).
- [x] **`repaint_delay` 10 ms** [kitty/options/definition.py:866] — *observed* default; throttle `render` [kitty/child-monitor.c:875], bypassed on pending input (§5.1, §5.2).
- [x] **`resize_debounce_time` (0.1, 0.5) s** [kitty/options/definition.py:1182] — *observed* default; tuple→struct **`on_end`=0.1 s=tuple[0]** [kitty/options/to-c.h:349], **`on_pause`=0.5 s=tuple[1]** [:350] (earlier draft reversed); `process_pending_resizes` [kitty/child-monitor.c:1043]: OS-notified complete ⇒ immediate [:1050], else `on_pause` reflow [:1055]; other platforms use `on_end` [:1062]; `resize_pty` [:592], `pty_resize` [:577], `window.py` [:863/:873] (§5.2).
- [x] **Resize→SIGWINCH hand-off** — *observed* `SIGWINCH sent to child … size (lines,cols,w,h)` (§5.3); live-drag coalescing labeled **inferred**.
- [x] **End-to-end settle** (text + OSC 133 + 2026 block) — *observed*, byte-identical **×2** (§5.4).

### Method & honesty
- [x] **Canonical PTY path only** — every primary capture is a real child through a real PTY; `test_parse_written_data` [kitty/screen.c:4771-4772] and `parse_bytes` [kitty_tests/parser.py:20] were **not** used as evidence (§0.1).
- [x] **Actual output beside every claim; file:line for every structural claim** — throughout.
- [x] **Timing rigor** — `input_delay` 3 ms, `repaint_delay` 10 ms, `resize_debounce_time` (0.1, 0.5) s, buffer 1 MiB, pause timeout 2000 ms; scale stated, stability confirmed across ≥ 2 runs (§Q1, §Q4, §Q5).
- [x] **Corrected citations used** — `BRACKETED_PASTE` at [kitty/modes.h:81] (not :80); `test_parse_written_data` at [kitty/screen.c:4771-4772]; `add_child` at [kitty/boss.py:585]; `BUF_EXTRA` at [kitty/vt-parser.c:20] (not :19).
- [x] **Code-derived (not directly observed) behavioral claims** — inferred from source and labeled inline where they appear, because the canonical headless build offers no non-mutating way to capture them (the software-GL framebuffer is not captured; `EVDBG` per-tick tracing is compile-gated out [kitty/child-monitor.c:29-33]; `ptrace_scope=3` and no `strace`). They are: (1) the key→escape **encoding** stage `glfw`→`keys.c`→`key_encoding.c` — only glfw ingress and an RC-injected bracketed paste were observed (§1.6); (2) the **display-side pause effects** — frozen snapshot, single atomic on-resume frame, no half-drawn intermediate [kitty/screen.c:2522-2543, :2737-2760, :2511] — only the command-stream continuation across the pause was observed (§1.4); (3) **parser-buffer saturation gating** under load — `has_space_for_input`→false [kitty/vt-parser.c:1481] and POLLIN masking [kitty/child-monitor.c:1501] — only generic producer `write()` backpressure was observed (§4.3); (4) the millisecond **timer effects** of `input_delay` 3 ms and `repaint_delay` 10 ms — the *values* are observed from the binary, but the per-tick batching/throttle behavior itself is code-derived (§5.1–§5.2); (5) **live-drag resize coalescing** on the `on_end`/`on_pause` debounce — only the committed-resize→SIGWINCH hand-off was observed (§5.3); (6) the **network-fault SSH degradation** profile — there is no SSH server in the container, and the real attempt failed with exit 255 (§4.4). Everything else is either a direct runtime observation or a structural fact cited to file:line.
- [x] **Build reconciled honestly** — `python3 setup.py develop` fails (`KeyError: 'DEVELOP_ROOT'` [setup.py:1255]); canonical `python3 setup.py` build used, producing `kitty/launcher/kitty` (kitty 0.35.2) (§0.2).

*All temporary observation scripts and captures lived in private `mktemp -d` directories (mode 0700, outside the repository) removed by each block's `trap`/`rm -rf`; the only change to the destination repository is this document — verified directly in Appendix C below.*

---

## Appendix C — Repository immutability proof

The read-only mandate requires that the **only** change to the destination repository is this document, and that every temporary observation artifact is removed. Both are verified directly (base commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, branch `blitzy-05dd5e02-071e-4e0e-a941-87e19867a0f8`).

**1. Temporary workspaces removed.** Every experiment used a private `mktemp -d` directory (mode 0700) under `TMPDIR=/tmp/kitty-clean-tmp`, removed by its own `trap … EXIT` and an explicit `rm -rf`. Nothing remains:

```bash
$ rm -rf /tmp/kitty-clean-tmp/kitty_*        # idempotent; each block's trap has already fired
$ ls -d /tmp/kitty-clean-tmp/kitty_* 2>/dev/null || echo "no leftover experiment dirs"
no leftover experiment dirs
```

**2. Exactly one file differs from the base commit** — the deliverable, added (`A`):

```bash
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
A	blitzy/documentation/kitty_815df1e210e0.md
```

**3. No source, test, or manifest file was touched.** The `--stat` of everything *except* the deliverable is empty — the read-only source mandate is satisfied:

```bash
$ git diff --stat 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -- . ':(exclude)blitzy/documentation/kitty_815df1e210e0.md'
$                                            # (no output)
```

**4. Clean working tree after the single commit** of this document:

```bash
$ git status --porcelain
$                                            # (no output — nothing uncommitted, nothing untracked)
```

The `--name-status` in (2) and the empty source-only `--stat` in (3) are the stable, self-consistent immutability proofs (they do not depend on this document's own length); the full `git diff --stat 815df1e21` accordingly reports `1 file changed`, whose insertion count is simply this document's line count.
