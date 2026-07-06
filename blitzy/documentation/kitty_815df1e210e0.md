# How Kitty's Terminal-Interaction Pipeline Behaves While Alive and Running

**A runtime-evidenced walkthrough — from the moment mixed input arrives to the moment the interface settles again, with special attention to a session that is paused and then resumed.**

---

## 0. Scope, Method, and Environment

### 0.1 What this document answers

This is a read-only, runtime-evidenced investigation of the [kitty](https://sw.kovidgoyal.net/kitty/) terminal emulator (kovidgoyal/kitty, version `0.35.2`, commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). It answers five questions about the *live* interaction pipeline:

- **Q1** — Where does raw input first *enter* the system, and how does a byte stream become something the application can react to — especially across a **pause → resume** transition?
- **Q2** — Who is the "unseen conductor" that manages timing, ordering, and state handoffs, how are those responsibilities split, and what decides which event is handled first?
- **Q3** — When shell-integration hints arrive mixed in with ordinary text, how does the system keep screen state, command context, and input meaning aligned without drifting out of sync?
- **Q4** — Does the pipeline behave differently under heavy **backpressure** or an **unstable remote** connection?
- **Q5** — The end-to-end narrative: how all the moving parts keep their rhythm from arrival to settle.

Every factual claim is grounded in a `file:line` citation naming a specific function/struct/macro, and every behavioral claim is backed by **actual captured output** together with the **exact command** that produced it. Claims derived from reading the source (not observed at runtime) are explicitly labeled **[inferred]**. Values reached through a non-canonical path are labeled **[non-canonical]**.

### 0.2 Method: build and run first, then write

All evidence below was captured by **building and running the real system first**, in the canonical Docker build environment (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), then writing this document from what was observed. Temporary observation scripts lived only under `/tmp` and were deleted afterward; the repository is byte-for-byte unchanged except for this one document.

**Exact build and invocation commands used:**

```bash
# Canonical build (Makefile all: -> python3 setup.py $(VVAL); V=1 adds --verbose, Makefile:L1-L13)
python3 setup.py build --verbose

# Event-loop observability build, REQUIRED for Q2 (Makefile debug-event-loop:, L25-L26)
make debug-event-loop        # -> python3 setup.py build --verbose --debug --extra-logging=event-loop

# Version banner
./kitty/launcher/kitty --version

# GPU-less observation harness: run a script as a file through the real launcher
./kitty/launcher/kitty +launch /tmp/kitty_obs/<script>.py

# Full test suite / a single module (test.py shebang "#!./kitty/launcher/kitty +launch", test.py:L1,L8)
CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ./test.py --module <name>

# Full GUI, headless (software GL via llvmpipe under Xvfb)
xvfb-run -a -s "-screen 0 1280x800x24" env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
    ./kitty/launcher/kitty --config NONE -e <cmd>
```

Captured version banner:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**Runtime versions** (with provenance): Python `>=3.8` required (`pyproject.toml:L2` — `requires-python = ">=3.8"`), highest explicitly supported is 3.11; the container's build interpreter is CPython 3.13.7. Go `1.22` (`go.mod:L3` — `go 1.22`), container `go1.22.12`. The C11 core built under `gcc (Ubuntu 15.2.0)`. The canonical build reports `Disabling building of wayland backend` (no `wayland-protocols` present → X11-only, matching canonical CI); this does not affect any pipeline behavior studied here.

### 0.3 The GPU-less observation harness (honest note on headless limits)

Kitty's GPU renderer needs a display. The container is headless, so two observation surfaces were used:

1. **The PTY-driven test harness in `kitty_tests/`** (`kitty_tests/__init__.py`). Its `parse_bytes(screen, data)` (`kitty_tests/__init__.py:L30`) drives the screen through **exactly the same C API the I/O thread uses**:

   ```python
   def parse_bytes(screen, data, dump_callback=None):
       data = memoryview(data)
       while data:
           dest = screen.test_create_write_buffer()     # -> vt_parser_create_write_buffer()
           s = screen.test_commit_write_buffer(data, dest)  # -> vt_parser_commit_write()
           data = data[s:]
           screen.test_parse_written_data(dump_callback)  # -> parse_worker()
   ```

   Those three `test_*` methods (`kitty/screen.c:L4755`, `L4762`, `L4772`) call `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, and `parse_worker()` — the very functions `read_bytes()` uses inside `io_loop()` (see the Q1 walkthrough). So the harness exercises the **real child-output parser path**, not a stand-in. This is the canonical no-display path and is labeled **harness-driven** where used.

2. **A real windowed GUI under Xvfb + software GL** (llvmpipe), used specifically for the pause/resume safety-valve timeout (Q1) and the event-loop debug stream (Q2), because those require a real render loop.

Where a real end-to-end windowed GLFW keystroke capture could not be produced headlessly, the user-input path is exercised through its real encoding function (`encode_key_for_tty`, the same one `on_key_input()` calls) and labeled accordingly; the remote-control interface (`kitty/rc/`) was **never** substituted for input observation.

---

## 1. Q1 — Where input enters, and how it becomes actionable (incl. pause → resume)

**Direct answer:** There are **two** distinct real entry points, and both matter:

1. **Child-output path (PTY → application):** bytes produced by the child process are first read from the PTY master fd by **`read_bytes(int fd, Screen *screen)`** at **`kitty/child-monitor.c:L1337`**, which runs inside the I/O thread's **`io_loop()`** (forward-declared at `kitty/child-monitor.c:L229`, defined at `L1481`, and it calls `read_bytes()` at `L1531`). `read_bytes()` obtains a write buffer from the VT parser, `read()`s directly into it, and commits it for the parse worker.
2. **User-input path (keyboard → PTY):** a key event arrives at the GLFW callback **`key_callback()`** (`kitty/glfw.c:L430`), which dispatches **`on_key_input(ev)`** (`kitty/glfw.c:L439`) into **`on_key_input()`** (`kitty/keys.c:L166`); that function encodes the key and hands the encoded bytes to **`schedule_write_to_child()`** (`kitty/child-monitor.c:L372`) at call sites `kitty/keys.c:L202/L253/L259/L286/L289`.

A byte stream "becomes actionable" when the VT parser classifies each byte and routes it either to the screen as drawn text or to a command handler (SGR attribute, mode change, OSC hint, …). For **pause → resume**, the primary mechanism is **DEC private mode 2026 (synchronized/pending update)**: while paused the parser keeps consuming and applying input to the live screen model, but the renderer keeps showing the last frame; on resume the renderer fetches the latest state (anti-tearing). Two secondary "pause" mechanisms also exist: **read/flow-control throttling** (Q4) and **job-control suspend/resume** (`SIGTSTP`/`SIGCONT`).

### 1.1 Child-output entry point (observed)

The read path itself, quoted unelided from source:

```c
// kitty/child-monitor.c:L1337
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;

    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;              // L1342 — backpressure early-return (see Q4)

    while(true) {
        len = read(fd, buf, available_buffer_space);        // L1345 — read directly into parser buffer
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;                                   // EIO => child gone
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);         // hand bytes to the parse worker
    return len != 0;
```

**Observed, end-to-end, on a real forked child on a real PTY** (harness `os.read(master_fd)` at `kitty_tests/__init__.py:L363` mirrors the `read(fd, …)` above; `parse_bytes` mirrors the create/commit/parse trio):

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q1_child_output.py
=== Q1 CHILD-OUTPUT ENTRY PATH end-to-end: real forked child on a real PTY ===
raw bytes read from child PTY  = b'plain \x1b[1mBOLD\x1b[0m done\r\n'
screen line0 after parsing     = 'plain BOLD done'
  -> note the SGR bold escape \x1b[1m was consumed as an attribute, NOT drawn as text
```

The raw stream contained the SGR sequence `\x1b[1m` (bold on) and `\x1b[0m` (reset); after parsing, the screen line reads `plain BOLD done` — the escape bytes were classified as an attribute command and **not** drawn, while the printable bytes became screen text. That is the byte stream "becoming actionable."

### 1.2 User-input entry point (observed)

`on_key_input()` (`kitty/keys.c:L166`) encodes the key with `encode_key_for_tty` (the C entry `pyencode_key_for_tty`, `kitty/keys.c:L311`) and then calls `schedule_write_to_child()`. The encoding is the exact one used on the live path. Captured encodings for plain, modifier, and special keys, under both the **legacy** encoding and the **Kitty keyboard protocol**:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q1_keys.py
GLFW mods: SHIFT=1 CTRL=4 ALT=2

=== USER-INPUT PATH: key encoding (encode_key_for_tty == encode_glfw_key_event used by on_key_input keys.c:L166) ===
--- LEGACY encoding (key_encoding_flags=0) ---
  'a' plain                    -> 'a'
  'a' + CTRL                   -> '\x01'
  'a' + ALT                    -> '\x1ba'
  'a' + CTRL+ALT               -> '\x1b\x01'
  'a' + SHIFT (shifted 'A')    -> 'A'
  ENTER plain                  -> '\r'
  ENTER + ALT                  -> '\x1b\r'
  TAB plain                    -> '\t'
  TAB + SHIFT                  -> '\x1b[Z'
  ESC plain                    -> '\x1b'
  UP arrow                     -> '\x1b[A'
  UP + CTRL                    -> '\x1b[1;5A'
--- KITTY keyboard protocol (key_encoding_flags=1 = disambiguate) ---
  'a' plain                    -> 'a'
  'a' + CTRL                   -> '\x1b[97;5u'
  'a' + ALT                    -> '\x1b[97;3u'
  'a' + CTRL+ALT               -> '\x1b[97;7u'
  ESC plain                    -> '\x1b[27u'
  ENTER + CTRL                 -> '\x1b[13;5u'
--- KITTY keyboard protocol (key_encoding_flags=15 = all enhancements) ---
  'a' plain                    -> '\x1b[97u'
  'a' + CTRL                   -> '\x1b[97;5u'
  'a' release                  -> '\x1b[97;1:3u'
```

Reading the encodings: legacy `Ctrl+a` collapses to the C0 control byte `\x01`; `Alt+a` prefixes `ESC` (`\x1ba`); arrows and shifted-Tab use CSI forms (`\x1b[A`, `\x1b[Z`); `Ctrl+Up` carries the modifier as a CSI parameter (`\x1b[1;5A`). Under the Kitty protocol every key becomes a disambiguated `CSI unicode ; modifiers u` form (`Ctrl+a` → `\x1b[97;5u`, where 97 = `ord('a')`, 5 = ctrl), `ESC` becomes reportable as `\x1b[27u`, and with all enhancements on, **key releases** are reported too (`'a' release` → `\x1b[97;1:3u`, the `:3` event-type meaning release) — something the legacy encoding cannot express. This is the "input meaning" that the encoded bytes carry to the child.

### 1.3 Resize — the third kind of "input" (observed)

A window resize is delivered to the child as a `TIOCSWINSZ` ioctl by `pty_resize()` (`kitty/child-monitor.c:L577`, the ioctl at `L579`). End-to-end on a real PTY, the child's own `stty size` reflects the new geometry:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q1_child_output.py
=== Q1 RESIZE path end-to-end: ioctl(TIOCSWINSZ) to child PTY (pty_resize child-monitor.c:L577/L579) ===
child `stty size` before and after resize 10x40 -> 20x100:
    '10 40'
    '20 100'
```

### 1.4 Pause → resume, PRIMARY mechanism: DEC private mode 2026 (observed, before/during/after)

Mode 2026 is defined as `PENDING_UPDATE = (2026 << 5)` at **`kitty/modes.h:L86`**. Enabling it (`CSI ? 2026 h`, "begin synchronized update"/BSU) routes through `screen_set_mode()` → `set_mode_from_const()` `case PENDING_MODE << 5:` (`kitty/screen.c:L1174`) → **`screen_pause_rendering(Screen*, bool pause, int for_in_ms)`** (`kitty/screen.c:L2506`). Disabling it (`CSI ? 2026 l`, ESU) resumes.

The transition was observed by querying the live mode with DECRQM (`CSI ? 2026 $ p`), whose reply is `\x1b[?2026;1$y` when paused and `\x1b[?2026;2$y` when not:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q1_pause_resume.py
=== Q1 PAUSE/RESUME via DEC private mode 2026 (canonical child-output escape path) ===
BEFORE pause: live line0='AAAA' cursor.x=4  DECRQM=b'\x1b[?2026;2$y'  (;2 => NOT paused)
DURING pause: live line0='BBBB' cursor.x=4  DECRQM=b'\x1b[?2026;1$y'  (;1 => paused)
AFTER  pause: live line0='BBBB' cursor.x=4  DECRQM=b'\x1b[?2026;2$y'  (;2 => NOT paused)

=== Q1 legacy DCS pending-mode syntax (ESC P = 1 s ESC\ / = 2 s) ===
start    DECRQM=b'\x1b[?2026;2$y'
after =1s DECRQM=b'\x1b[?2026;1$y'  (;1 => paused)
after =2s DECRQM=b'\x1b[?2026;2$y'  (;2 => NOT paused)
```

The critical observation is in the **DURING** line: after entering pending mode, the stream `\rBBBB` was written and the **live screen line changed from `AAAA` to `BBBB`** — i.e., **the parser kept consuming and applying input while rendering was paused**. On resume (`AFTER`) the mode returns to not-paused. Both the canonical DECSET syntax (`CSI ? 2026 h/l`) and the legacy DCS syntax (`ESC P = 1 s ESC \` to begin, `= 2 s` to end; handled around `kitty/vt-parser.c:L637`) drive the same `screen_pause_rendering()` toggle.

That rendering reads from a **snapshot** while paused is confirmed in source: `screen_pause_rendering()` copies the cursor (`kitty/screen.c:L2527`), the color profile (`L2528`), and the line buffer (`L2529-L2537`) into `paused_rendering`, and `screen_update_cell_data()` (`kitty/screen.c:L2738`) renders from `paused_rendering.linebuf` while paused — so the frozen frame is the snapshot, while the live model advances underneath. **[inferred from reading]** for the `screen_update_cell_data` snapshot-read path (the copy itself and the mode transition are observed above).

### 1.5 Pause → resume: the 2-second safety-valve timeout (observed on a real GUI)

Kitty does not trust an application to always end a synchronized update. `screen_pause_rendering()` sets a default expiry of **2000 ms** when none is given (`for_in_ms <= 0 → 2000`, `kitty/screen.c:L2521`; `expires_at` set at `L2522`), and `screen_check_pause_rendering()` (`kitty/screen.c:L2489`) **auto-resumes** once the deadline passes (`L2490`: `if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at) screen_pause_rendering(self, false, 0)`). Because that check runs from the render path (`prepare_to_render_os_window`, invoked from `kitty/child-monitor.c:L729`), it needs a real render loop — so this was captured in a real windowed kitty under Xvfb:

```
$ xvfb-run -a -s "-screen 0 1280x800x24" env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
    ./kitty/launcher/kitty --config NONE -e python3 /tmp/kitty_obs/q1_timeout_child.py
initial banner/pending bytes from kitty at startup: b''
BEFORE pause: DECRQM=b'\x1b[?2026;2$y'
DURING pause (t+0.00s): DECRQM=b'\x1b[?2026;1$y'
AFTER 2.4s wait (t+2.95s): DECRQM=b'\x1b[?2026;2$y'  <-- expect ;2 auto-resumed by safety valve
```

Here **no ESU was ever sent** — the application entered pending mode and simply waited. After 2.4 s the mode had reverted to not-paused (`;2`) on its own: the safety valve fired, exactly as the code specifies. This result was stable across 2 runs. The parser side has a matching guard whose error text (`kitty/vt-parser.c:L646-L648`) explains a pending-mode abort is "caused by a timeout with no data received for too long or by too much data in pending mode."

### 1.6 Byte classification during pause (observed) — bridge to Q3

Dumping the parser's command stream for the input `AA` + `CSI ? 2026 h` + `BB` + `CSI ? 2026 l` + `CC` shows precisely how text and escapes are separated even as the mode toggles mid-stream:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q1_dump.py
=== Parser command classification of  AA <CSI?2026h> BB <CSI?2026l> CC ===
   ('draw', ('A',))
   ('draw', ('A',))
   ('screen_set_mode', (2026, 1))
   ('draw', ('B',))
   ('draw', ('B',))
   ('screen_reset_mode', (2026, 1))
   ('draw', ('C',))
   ('draw', ('C',))
live line0 = 'AABBCC'
```

The printable bytes became `draw` commands; the escape sequences became `screen_set_mode`/`screen_reset_mode` commands. The text and the control meaning never blur — the foundation of Q3.

### 1.7 Paste bursts: bracketed paste, DEC mode 2004 (observed, byte-accurate)

The "paste bursts" dimension is handled by **bracketed paste**, mode `BRACKETED_PASTE = (2004 << 5)` (`kitty/modes.h:L81`, with start marker `"200~"` at `L82` and end marker `"201~"` at `L83`). `paste_()` (`kitty/screen.c:L4573`) emits `BRACKETED_PASTE_START` (`L4586`), the payload via `write_to_child` (`L4587`), then `BRACKETED_PASTE_END` (`L4588`), gated on `self->modes.mBRACKETED_PASTE`:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q1_paste.py
=== Q1 BRACKETED PASTE (DEC private mode 2004) — byte-accurate ===
mode 2004 OFF: paste('hello world') -> child bytes = b'hello world'
mode 2004 ON : paste('hello world') -> child bytes = b'\x1b[200~hello world\x1b[201~'
               (start marker = ESC [ 200~, end marker = ESC [ 201~)
mode 2004 ON : paste_bytes('hello world') -> child bytes = b'hello world'  (never bracketed)
mode 2004 OFF: paste('hello world') -> child bytes = b'hello world'
```

With mode 2004 **on**, the pasted payload is wrapped byte-accurately as `\x1b[200~hello world\x1b[201~`, so the application can tell a paste burst from typed input. `paste_bytes()` (`kitty/screen.c:L4600`) is the deliberately-unbracketed variant (it passes `allow_bracketed_paste=false`).

### 1.8 Pause → resume: SECONDARY candidate — job-control suspend/resume (traced)

A second sense of "pause a session" is Unix job control (Ctrl-Z → `SIGTSTP`, resume via `fg` → `SIGCONT`). This is reached only when the `HANDLE_TERMIOS_SIGNALS` mode (`= (19997 << 5)`, `kitty/modes.h:L89`) is set. In `on_key_input()` after encoding: `if (size==1 && screen->modes.mHANDLE_TERMIOS_SIGNALS) { if (screen_send_signal_for_key(screen, *encoded_key)) return; }` (`kitty/keys.c:L256-L257`), otherwise it falls through to `schedule_write_to_child()` (`L259`). `screen_send_signal_for_key()` (`kitty/screen.c:L2404`) → `send_signal_for_key` (`kitty/window.py:L1116`) → `kitty/child.py:L481`, which maps `VINTR→SIGINT`, `VSUSP` (Ctrl-Z) `→SIGTSTP` (`kitty/child.py:L493`), `VQUIT→SIGQUIT`, delivering via `os.killpg(os.tcgetpgrp(child_fd), sig)`. **[inferred from reading]** at the runtime level — the code path is traced and cited, but a live Ctrl-Z→SIGTSTP delivery was not separately captured; it requires `HANDLE_TERMIOS_SIGNALS` to be enabled by the running program. The third candidate — read/flow-control throttling — is demonstrated in Q4.

---

## 2. Q2 — The "unseen conductor": division of labor and event ordering

**Direct answer:** The conductor is the **Child Monitor's multi-threaded event loop** in `kitty/child-monitor.c`. It runs **three threads**, confirmed by their set thread-names in the running binary:

- **Main thread** — runs the GLFW/platform event loop (`handleEvents`), rendering, and periodic state checks.
- **I/O thread `"KittyChildMon"`** — the `io_loop()` (`kitty/child-monitor.c:L1481`; thread named at `L1489`); it `poll()`s the child PTY fds plus a self-pipe wakeup fd and a signal fd, reads child output, and writes queued input.
- **Talk thread `"KittyPeerMon"`** — the `talk_loop()` (`kitty/child-monitor.c:L1805`; named at `L1808`); it services remote-control peers.

(A short-lived helper thread `"KittyWriteStdin"`, `kitty/child-monitor.c:L967`, is used for large stdin writes.) The Python side starts the whole machine at `boss.child_monitor.main_loop()` (`kitty/main.py:L234`), with `boss.destroy()` in the `finally` (`L236`); the `Boss` singleton (`kitty/boss.py`, `set_boss`/`get_boss`) is the global coordinator for user actions.

**What decides which event is handled first** is the combination of (a) `poll()` readiness on the multiplexed fds, (b) delay-based scheduling via `input_delay`/`repaint_delay`, and (c) a **self-pipe wakeup** that lets out-of-band events (resize, signals, user actions) preempt an otherwise-idle `poll()` by forcing it to return immediately.

### 2.1 The event-loop debug build and a captured session

The event-loop stream is only compiled in under the debug build (`event-loop` maps to `-DDEBUG_EVENT_LOOP`, `setup.py:L489`; `EVDBG(...)` → `timed_debug_print`, `kitty/child-monitor.c:L29-L30`, timestamp format `"[%.3f] "` from `kitty/monotonic.h:L99`):

```bash
$ make debug-event-loop
# -> python3 setup.py build --verbose --debug --extra-logging=event-loop   (exit 0)
```

A representative session was then run under Xvfb, with a child that emits a start banner, periodic ticks, a ~2000-byte burst, a self-resize, and a done marker:

```bash
$ xvfb-run -a -s "-screen 0 1280x800x24" env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
    ./kitty/launcher/kitty --config NONE -o repaint_delay=10 -e python3 /tmp/kitty_obs/q2_child.py
```

Complete, unedited excerpt of the resulting event-loop debug stream (main-thread ticks):

```
[0.159] Failed to open systemd user bus with error: Connection refused
[0.163] starting handleEvents(0.00)
[0.163] pollForEvents final timeout: 0.000
[0.163] State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 1[0.177] display_read_ok: 0
[0.178] other dispatch done
[0.178] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.178] starting handleEvents(0.00)
[0.178] pollForEvents final timeout: 0.000
[0.178] display_read_ok: 0
[0.178] other dispatch done
[0.178] --------- loop tick, wakeups_happened: 0 ----------
[0.178] starting handleEvents(-0.00)
[0.178] pollForEvents final timeout: 0.002
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[0.182] display_read_ok: 0
[0.182] other dispatch done
[0.182] --------- loop tick, wakeups_happened: 0 ----------
[0.182] starting handleEvents(-0.00)
[0.182] pollForEvents final timeout: 0.484
[0.327] display_read_ok: 0
[0.327] other dispatch done
[0.327] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.327] starting handleEvents(-0.00)
[0.327] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[0.332] display_read_ok: 0
[0.332] other dispatch done
[0.332] --------- loop tick, wakeups_happened: 0 ----------
[0.332] starting handleEvents(-0.00)
[0.332] pollForEvents final timeout: 0.333
[0.447] display_read_ok: 0
[0.447] other dispatch done
```

### 2.2 Annotating the log

- **`starting handleEvents(T)`** (`glfw/x11_window.c:L69`) begins a main-loop tick; **`pollForEvents final timeout: T`** (`glfw/backend_utils.c:L298`) is the actual computed `poll()` wait for that tick.
- **`State check timer fired`** → `do_state_check()` (`kitty/child-monitor.c:L1217`); **`Processing global state`** → `process_global_state()` (`kitty/child-monitor.c:L1225`); **`input_read: N, check_for_active_animated_images: M`** is emitted from `render()` (`kitty/child-monitor.c:L872`).
- **`--------- loop tick, wakeups_happened: N ----------`** (`glfw/main_loop.h:L31`) closes each tick and reports how many self-pipe wakeups were coalesced into that tick.

The **poll timeout varies tick to tick** — `0.000`, `0.002`, `0.003`, `0.333`, `0.484` s — which is the delay-based scheduling in action: when there is nothing pending the loop waits longer, and when input/render work is pending it uses a short, `input_delay`/`repaint_delay`-derived wait. The `wakeups_happened` alternates between `0` (a timer-driven tick) and `1` (an I/O-thread-driven wakeup), showing the two ways a tick is triggered.

### 2.3 The self-pipe wakeup (two-sided)

The self-pipe is the mechanism that lets one thread preempt another's `poll()`. Its primitives live in `kitty/loop-utils.h`: `wakeup_fds[2]` (`L33`), `wakeup_read_fd` (`L39`), `signal_read_fd` (`L40`), `wakeup_loop()` (`L48`), `self_pipe()` (`L52`), and `drain_fd()` (`L76`). The documentation comment on `wakeup()` states its purpose literally — "wakeup the ChildMonitor I/O thread, **forcing it to exit from poll()** if it is waiting there" (`kitty/child-monitor.c:L298-L299`).

- **I/O-thread side:** `io_loop()` includes the wakeup fd as `children_fds[0]` and, when it fires, drains it with `drain_fd(children_fds[0].fd)` (`kitty/child-monitor.c:L1513`); OS signals arrive on `children_fds[1]` and are dispatched via `read_signals(children_fds[1].fd, handle_signal)` (`L1519`).
- **Main-thread side:** after receiving child data, the I/O thread calls `wakeup_main_loop()` (the `WAKEUP` macro, `kitty/child-monitor.c:L1562`), **throttled by `input_delay`** (`L1566-L1569`: it only wakes the main loop if `now - last_main_loop_wakeup_at > input_delay`, otherwise it sets `has_pending_wakeups`). The main loop drains this via `check_for_wakeup_events()` (`glfw/backend_utils.c:L253`); the `wakeups_happened: N` counter above is exactly these events.

This throttling is why a *surge* of child output does not translate into a storm of one-render-per-read: reads are coalesced and the main loop is woken at most about once per `input_delay`.

### 2.4 Delay-based scheduling and fd-readiness masks

- The **I/O-thread `poll()` timeout** is `input_delay`-derived when `has_pending_wakeups`, else `-1` (block indefinitely) (`kitty/child-monitor.c:L1505-L1512`).
- The **main-loop poll timeout** is computed by `set_maximum_wait(...)` from `repaint_delay`/`input_delay` (`kitty/child-monitor.c:L445-L446`), and — importantly for Q1 — the **paused-rendering branch** (`L443-L444`) sets the wait to the pause expiry so the safety-valve timeout can fire on schedule.
- Which fd is serviced is also gated by per-child masks: `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0` and `|= POLLOUT` when there is queued input to write (`kitty/child-monitor.c:L1500-L1503`). When the parser has no room, the PTY is not even marked readable — the Q4 backpressure gate, at the scheduling layer.

### 2.5 Out-of-band handlers (cited)

- **Resize:** `process_pending_resizes()` (`kitty/child-monitor.c:L1043`), invoked from the state check at `L1233`; a resize is debounced before the `TIOCSWINSZ` is applied.
- **Child exit / window close:** `process_pending_closes()` (`kitty/child-monitor.c:L1098`), invoked at `L1246`.
- **Signals:** `handle_signal()` (`kitty/child-monitor.c:L1362-L1382`) maps `SIGINT`/`SIGTERM`/`SIGHUP` → kill, `SIGCHLD` → child-died bookkeeping, `SIGUSR1` → reload config, `SIGUSR2` → log; it is invoked via `read_signals(...)` on the I/O thread (`L1519`). (The AAP cited this region as `L1358-L1385`; the running build places the definition at `L1362-L1382`.)

**Stability note:** across 2 runs the structural markers were stable (run 1 = 15 loop ticks, run 2 = 16 — a normal one-tick wall-clock difference; `State check timer fired` appeared 8× in both). Absolute timestamps naturally vary run to run (wall clock); the *sequence and structure* are stable.


---

## 3. Q3 — Keeping screen state, command context, and input meaning aligned

**Direct answer:** Alignment never drifts because the **VT parser classifies every single byte** (`kitty/vt-parser.c`) and routes escape sequences — OSC 133 prompt/command markers, OSC 7 CWD hints — into dedicated Screen handlers, entirely separately from ordinary printable text which becomes drawn cells. Text bytes and control bytes are consumed by the same linear scan but dispatched to different destinations, so a hint embedded *inside* a run of text neither corrupts the text nor is itself mistaken for text. Shell integration is what emits those hints; it is installed by **`modify_shell_environ()`** (`kitty/shell_integration.py:L218`), which resolves the shell via `get_supported_shell_name(argv[0])` (`L219`) and applies per-shell setup (`setup_fish_env` `L16`, `setup_zsh_env` `L49`, `setup_bash_env` `L70`).

### 3.1 Interleaved OSC 133 / OSC 7 with ordinary text (observed, byte-accurate)

A full shell command cycle was assembled with hints interleaved with plain text — OSC 7 (cwd), OSC 133;A (prompt start), the prompt text, OSC 133;B (command start), the command, OSC 133;C (output start), the output lines, OSC 133;D;0 (command end, exit 0) — and pushed through the real parser. The result was identical across 2 runs:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q3_alignment.py
=== Q3 raw interleaved byte stream sent to parser (byte-accurate) ===
b'\x1b]7;file://host/home/user\x1b\\\x1b]133;A\x1b\\user@host:~$ \x1b]133;B\x1b\\ls -la\x1b]133;C\x1b\\\r\ntotal 42\r\nfile1.txt\r\nfile2.txt\x1b]133;D;0\x1b\\'

=== screen.last_reported_cwd after OSC 7 ===
b'file://host/home/user'

=== cmd_output_marking callbacks captured (OSC 133) ===
last_cmd_cmdline = ''  last_cmd_exit_status = 0

=== dump_lines_with_attrs: each line's prompt_kind + text (proves NO DRIFT) ===
0: output dirty
user@host:~$ ls -la
1: dirty
total 42
2: dirty
file1.txt
3: dirty
file2.txt
```

Reading the result: the OSC 7 hint was captured into `screen.last_reported_cwd = b'file://host/home/user'`; the OSC 133;D;0 set `last_cmd_exit_status = 0`; and `dump_lines_with_attrs` shows the output lines (`total 42`, `file1.txt`, `file2.txt`) landed on the correct screen rows with correct text — **no drift**. (Line 0 is tagged `output` rather than `prompt` because OSC 133;C's `OUTPUT_START` overwrote the earlier `PROMPT_START` on that same row — the prompt and the echoed command share one physical line here; the marking is per-line, so the last marker on the row wins. The *text* is still exactly right.)

The relevant Screen handlers: `shell_prompt_marking()` (`kitty/screen.c:L2328`) parses the OSC 133 letter — `A` → `PROMPT_START` and fires `cmd_output_marking(False)` (`L2337-L2338`), `C` → `OUTPUT_START` + the cmdline (`L2341`), `D` → exit status; OSC 7 is handled by `process_cwd_notification()` (`kitty/screen.c:L2393`), which sets `last_reported_cwd` (exposed at `kitty/screen.c:L4897`).

### 3.2 The strongest no-drift proof: an OSC hint split mid-word (observed)

To show the classification is genuinely per-byte and not reliant on hints sitting on clean boundaries, an OSC 7 sequence was injected **in the middle of the word `HELLO`** (between `HEL` and `LO`):

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q3_alignment.py
=== NOW: OSC hint split IN THE MIDDLE of a word, proving text reassembles + hint still handled ===
line0 = 'HELLO'  (expect 'HELLO' - text reassembled across the embedded OSC)
last_reported_cwd = b'file://host/tmp/x'  (OSC 7 still handled)
```

The input was `b"HEL\x1b]7;file://host/tmp/x\x1b\\LO"`. The screen line reads `HELLO` (the text `HEL` and `LO` were reassembled contiguously across the embedded escape) **and** the cwd hint was still captured (`b'file://host/tmp/x'`). The escape was cleanly lifted out of the text run without disturbing it — text meaning and hint meaning stayed perfectly separated.

### 3.3 What a REAL bash shell actually emits (observed, canonical)

The two captures above inject synthetic sequences. To corroborate with a real shell, a **real bash** was launched with `KITTY_SHELL_INTEGRATION=enabled` over a real PTY (via `safe_env_for_running_shell` + `kitty_tests.PTY`), a prompt awaited, and `echo hello` run — capturing the actual OSC bytes bash's kitty integration emits. Both runs read 529 bytes with an identical marker order:

```
$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ./kitty/launcher/kitty +launch /tmp/kitty_obs/q3_real_shell.py
----- RUN 1: total raw bytes read from real bash child = 529 -----
OSC 133/OSC 7 sequences found: 18
  OSC 133 (prompt/cmd markers): 17   OSC 7 (cwd): 1
  --- byte-accurate OSC 133 markers (in order emitted) ---
     \x1b]133;k;start_kitty\a
     \x1b]133;D;0\a
     \x1b]133;A\a
     \x1b]133;k;end_kitty\a
     \x1b]133;k;start_suffix_kitty\a
     \x1b]133;k;end_suffix_kitty\a
     \x1b]133;C;cmdline=echo\ hello\a
     \x1b]133;k;start_kitty\a
     \x1b]133;k;end_kitty\a
     \x1b]133;k;start_suffix_kitty\a
     \x1b]133;k;end_suffix_kitty\a
     \x1b]133;k;start_kitty\a
     \x1b]133;D;0\a
     \x1b]133;A\a
     \x1b]133;k;end_kitty\a
     \x1b]133;k;start_suffix_kitty\a
     \x1b]133;k;end_suffix_kitty\a
  --- byte-accurate OSC 7 cwd notifications ---
     \x1b]7;kitty-shell-cwd://reverse-code-generator-1cf2b534-sngfp/tmp/kitty_obs/q3home_1\a
  screen.last_reported_cwd = b'kitty-shell-cwd://reverse-code-generator-1cf2b534-sngfp/tmp/kitty_obs/q3home_1'
  last_cmd_exit_status     = 0
  OSC 133 marker letters in order: k D A k k k C k k k k k D A k k k

----- STABILITY across 2 runs -----
  identical marker order across runs: True
```

Notable, byte-accurate real-shell facts: bash terminates its OSC 133/OSC 7 sequences with **BEL (`\a`)** rather than ST (`\x1b\\`) — the parser accepts both terminators. The command-start marker carries the command context inline: `\x1b]133;C;cmdline=echo\ hello\a`. The prompt-start `A`, command-start `C`, and command-end `D;0` are all present, interleaved with kitty-internal region markers (`133;k;start_kitty`/`end_kitty`, used to delimit prompt regions). The Screen correctly captured `last_reported_cwd` and `last_cmd_exit_status = 0` — command context and screen state stayed aligned with a real shell, not just synthetic input.

### 3.4 The parser internals that guarantee this (cited)

The parser is threaded and double-buffered, guarded by a mutex `lock` (`kitty/vt-parser.c:L206`). Incoming bytes are staged in a `write` region and handed to the reader via the struct `{ size_t offset, sz, pending; } write;` (`kitty/vt-parser.c:L210`). `vt_parser_create_write_buffer()` (`L1451`) hands out the free tail of the 1 MiB buffer; a capacity check `read.sz + write.pending < BUF_SZ` inside `vt_parser_has_space_for_input()` (`L1481`) governs whether more can be accepted; and the actual classification/dispatch runs in `parse_worker()` (`L1496`). Because every byte flows through this one classifier, "ordinary text vs. escape meaning" is a single, consistent decision — there is no second, racing interpretation to drift against.

### 3.5 Canonical test-suite corroboration (observed)

The repository's own tests exercise these exact paths and pass on the running build:

```
$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ./test.py --module shell_integration
Ran 6 tests in ... OK        # bash / fish / zsh, both kitten and non-kitten variants

$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ./test.py --module parser
Ran 16 tests in ... OK       # incl. test_parser_threading, test_osc_codes, test_csi_codes,
                             #       test_dcs_codes, test_utf8_parsing
```


---

## 4. Q4 — Behavior under heavy backpressure / unstable remote

**Direct answer: yes, the pipeline behaves differently under heavy backpressure — by design.** When the VT parser's write buffer is full, `read_bytes()` **returns early without reading**: `if (!available_buffer_space) return true;` at **`kitty/child-monitor.c:L1342`**, where `available_buffer_space` came from `vt_parser_create_write_buffer()` at `L1341`. Stopping the `read()` stops draining the PTY, the OS pipe buffer fills, and **flow control propagates to the child**, throttling its output until the parser catches up. There is a matching gate one layer up: `io_loop()` will not even mark the PTY readable when the parser is full — `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0` (`kitty/child-monitor.c:L1501`). For the **unstable-remote** dimension, the SSH kitten's resilience is at the **data-buffering** level (it captures and replays "leading data"), not connection re-establishment — there is no reconnection logic; a dropped connection is cleaned up by a bootstrap `trap`.

The parser buffer size is a runtime-confirmed **1 MiB**:

```
$ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE as B; print(B, B==1024*1024)'
1048576 True
```

This matches `#define BUF_SZ (1024u*1024u)` (`kitty/vt-parser.c:L18`); the buffer itself is the linear array `uint8_t buf[BUF_SZ + BUF_EXTRA]` (`kitty/vt-parser.c:L194`).

### 4.1 Saturating the parser buffer — before / during / after (observed)

Using the exact C functions `read_bytes()` calls (exposed as `Screen.test_create_write_buffer()` / `test_commit_write_buffer()`, whose returned buffer length **is** `available_buffer_space`), 256 KiB chunks were committed **without** draining, watching `available_buffer_space` fall to zero, then a single `parse_worker()` drain restored it. Scale: **5 × 256 KiB = 1280 KiB offered into the 1024 KiB buffer.** Deterministic across 2 runs:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q4_saturate.py
VT_PARSER_BUFFER_SIZE (BUF_SZ) = 1048576 bytes = 1024 KiB = 1 MiB

================= Q4 PARSER-BUFFER SATURATION (scale: 5 x 256 KiB = 1280 KiB offered into a 1024 KiB buffer) =================

----- RUN 1 -----
stage                                                 committed_total  available_space  has_space_for_input   read_bytes_would
BEFORE                                                              0          1048576                 True   READ (drain PTY)
after commit #1 (+262144B)                                     262144           786432                 True   READ (drain PTY)
after commit #2 (+262144B)                                     524288           524288                 True   READ (drain PTY)
after commit #3 (+262144B)                                     786432           262144                 True   READ (drain PTY)
after commit #4 (+262144B)                                    1048576                0                False   EARLY-RETURN (halt PTY, flow-ctrl to child)
after commit #5 (+0B)                                         1048576                0                False   EARLY-RETURN (halt PTY, flow-ctrl to child)
commit attempt while FULL (accepted 0B)                       1048576                0                False   EARLY-RETURN (halt PTY, flow-ctrl to child)
AFTER drain (parse_worker ran)                                1048576          1048576                 True   READ (drain PTY)

----- STABILITY across 2 runs (available_space + has_space at each stage) -----
run1 == run2 (deterministic buffer accounting): True
```

Reading the table: **BEFORE**, the full 1048576 bytes are free and `has_space_for_input` is `True`, so `read_bytes()` would read. **DURING** saturation, `available_space` steps down 786432 → 524288 → 262144 → **0**; at 0 (`has_space_for_input = False`) `read_bytes()` hits its `return true` early-exit and the PTY is no longer drained. A further commit while full accepts **0 bytes** — the concrete "stop reading" that pushes back on the child. **AFTER** one `parse_worker()` drain, `available_space` is back to 1048576 and reads resume. The accounting is bit-for-bit identical across runs (the buffer math has no run-to-run variance).

### 4.2 Real-PTY 8 MiB burst — throughput and OS flow control (observed)

A real forked `/usr/bin/python3` child was made to blast an **8 MiB** burst down a real PTY. Two scenarios were measured; both stable across 2 runs:

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q4_real_pty.py
================= Q4 REAL-PTY BURST (scale: 8 MiB from a real forked /usr/bin/python3 child on a real PTY) =================
[drain-immediately RUN 1] {'total_bytes': 8388622, 'MiB': 8.0, 'read_calls': 2535, 'max_chunk': 8192, 'got_end': True, 'elapsed_s': 0.0388, 'throughput_MiBps': 206.4}
[drain-immediately RUN 2] {'total_bytes': 8388622, 'MiB': 8.0, 'read_calls': 2479, 'max_chunk': 8192, 'got_end': True, 'elapsed_s': 0.0417, 'throughput_MiBps': 191.9}

[backpressure(stop-draining 0.6s) RUN 1] {'child_still_running_while_not_draining': True, 'kernel_pty_buffered_before_block': 65536, 'total_after_full_drain': 8388622, 'got_end': True, 'all_bytes_accounted': True}
[backpressure(stop-draining 0.6s) RUN 2] {'child_still_running_while_not_draining': True, 'kernel_pty_buffered_before_block': 1728512, 'total_after_full_drain': 8388622, 'got_end': True, 'all_bytes_accounted': True}
```

**Drain-immediately:** the full 8 MiB (`8388622` bytes = 8388608 + a 14-byte end marker) arrived both runs, read in `io.DEFAULT_BUFFER_SIZE`-sized chunks (`max_chunk: 8192`), at roughly 190–210 MiB/s. **Backpressure (stop draining for 0.6 s):** in both runs the child was **still running** while the reader was idle (`child_still_running_while_not_draining: True`) — i.e., it had **blocked on `write()`** because the kernel PTY buffer filled: OS-level flow control, the exact consequence of `read_bytes()` refusing to read. Once draining resumed, **all 8 MiB were accounted for** (`all_bytes_accounted: True`, `got_end: True`) — throttling never loses data.

(An implementation note discovered while building this: under `kitty +launch`, `sys.executable` is the **kitty launcher, not python3**, so the child must be spawned with `shutil.which('python3')`; and an `EIO` on the master read is the normal "slave closed / child exited" signal, which `read_bytes()` treats as child-gone at `kitty/child-monitor.c:L1347-L1349`.)

### 4.3 Run-to-run distribution (reproduced honestly, not stabilized)

The two backpressure runs above reported **different** `kernel_pty_buffered_before_block` values — `65536` (64 KiB) vs `1728512` (1.65 MiB). Rather than stabilize that away, the identical scenario was repeated 6 times, isolating the pure kernel high-water mark by doing a **single** read after the stall (removing the drain/refill race that produced the spread):

```
$ ./kitty/launcher/kitty +launch /tmp/kitty_obs/q4_distribution.py
Q4 DISTRIBUTION: 6 identical runs, 8 MiB burst, /usr/bin/python3 child, stop-draining 0.5s then ONE read
 run  child_blocked  single_read_bytes
   1           True               4095
   2           True               4095
   3           True               4095
   4           True               4095
   5           True               4095
   6           True               4095

distribution of single-read high-water (bytes): [4095, 4095, 4095, 4095, 4095, 4095]
min=4095  max=4095  child_blocked_every_run=yes
```

**Interpretation.** The *invariant* is rock-stable: across all 6 identical runs the child **always blocked** when draining stopped, and the pure single-read kernel PTY high-water mark was a deterministic **4095 bytes** (the Linux PTY line-discipline output limit; min = max = 4095, zero variance). The earlier 64 KiB↔1.65 MiB spread was therefore a **measurement artifact** of a multi-read drain loop letting the blocked child resume and refill concurrently — *not* variance in the backpressure mechanism itself. The mechanism is deterministic; only the racy way of measuring it produced a distribution.

### 4.4 The `3rdparty/ringbuf/` FIFO — what it actually is (observed finding)

The AAP flagged `3rdparty/ringbuf/` as backpressure-relevant. Grepping the running source shows its **only** consumers are in `kitty/history.c` — it backs the **pager scrollback history** (`pagerhist`), *not* the hot input/backpressure path:

```
$ grep -rn "ringbuf" --include=*.c --include=*.h kitty/ | grep -v 3rdparty | head
kitty/history.c:76:    ph->ringbuf = ringbuf_new(sz);
kitty/history.c:96:    size_t count = ringbuf_bytes_used(ph->ringbuf);
...
```

So the accurate picture is: the buffer that gates child-output flow control is the VT parser's **linear** `uint8_t buf[BUF_SZ]` (`kitty/vt-parser.c:L194`), a distinct structure from the ring buffer. The `ringbuf` is a classic wrap-around byte-FIFO used for scrollback pager history (initial size `MIN(1 MiB, pagerhist_sz)`, `kitty/history.c:L67`); it keeps one or more bytes unusable "to distinguish the 'buffer full' state from the 'buffer empty' state" (`3rdparty/ringbuf/ringbuf.h:L44-L46`), with `ringbuf_is_full` (`ringbuf.c:L122`, free==0), `ringbuf_is_empty` (`L128`), and fd I/O via `ringbuf_read` (`L241`) / `ringbuf_write` (`L334`). This is a correction of the AAP's framing, grounded in the observed usage sites; the ringbuf's role in backpressure is **[inferred: none in the input path]** — it participates only in scrollback storage.

### 4.5 Unstable remote — the SSH kitten path (observed)

The SSH kitten's canonical GPU-less tests all pass on the running build (stable across 2 invocations):

```
$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ./test.py --module ssh
test_basic_pty_operations ... ok
test_ssh_bootstrap_with_different_launchers ... ok
test_ssh_connection_data ... ok
test_ssh_copy ... ok
test_ssh_env_vars ... ok
test_ssh_leading_data ... ok
test_ssh_login_shell_detection ... ok
test_ssh_shell_integration ... ok
----------------------------------------------------------------------
Ran 8 tests in 17.705s
OK
```

**Direct observed behavior:** the bootstrap (`shell-integration/ssh/bootstrap.sh`) reads the incoming payload with **line-oriented, incremental read loops** (`while IFS= read -r line`, `bootstrap.sh:L98` and `L139`) — so it tolerates the stream arriving in arbitrary partial chunks. Crucially it maintains a **`leading_data`** buffer (`bootstrap.sh:L86`, accumulated at `L147`): any input that arrives *before* the base64/tar payload has finished transferring is captured and later replayed to the shell, so early keystrokes on a slow/laggy link are **not lost**. `test_ssh_leading_data` confirms this end-to-end — with `pre_data='before_tarfile'` the post-bootstrap screen shows `UNTAR_DONE\nld:before_tarfile` (`kitty_tests/ssh.py:L158-L169`), i.e. the leading data survived the bootstrap. If the connection or transfer breaks, the `cleanup_on_bootstrap_exit()` trap (`bootstrap.sh:L10`, armed `trap ... EXIT` at `L91`) restores `stty echo` and removes the temp dir, and `die()` (`bootstrap.sh:L16`) reports transfer failure and exits `1`.

**Honest caveat [inferred / not exercised]:** the harness uses a **local PTY** as the SSH transport stand-in; a *real* TCP disconnect / network flakiness was **not** exercised, and the bootstrap contains **no reconnection logic** (it is a one-shot bootstrap). So the observed resilience is specifically leading-data buffering and clean teardown — not session reconnection. The claim "no reconnection behavior exists" is inferred from reading the bootstrap, which has no such code path.


---

## 5. Q5 — End-to-end narrative: from a surge of mixed input to the interface settling

Here is the whole rhythm, tying the observations together. The user's metaphors map onto concrete components: the **busy junction** is the multiplexed `poll()` loop; the **unseen conductor** is the Child Monitor's three-thread event loop; **keeping rhythm under a surge** is the backpressure and pending/synchronized-update machinery.

**1. Mixed input arrives at two doors.** Keystrokes, paste bursts, and resize signals enter through the **user-input door**: GLFW's `key_callback()` (`kitty/glfw.c:L430`) → `on_key_input()` (`kitty/keys.c:L166`), which encodes the key (legacy or Kitty protocol — §1.2) and calls `schedule_write_to_child()` (`kitty/child-monitor.c:L372`); a paste becomes a bracketed `\x1b[200~ … \x1b[201~` payload (§1.7); a resize becomes a debounced `TIOCSWINSZ` ioctl (§1.3, `kitty/child-monitor.c:L577`). Meanwhile the program's output enters through the **child-output door**: the I/O thread's `io_loop()` `poll()`s the PTY and `read_bytes()` (`kitty/child-monitor.c:L1337`) reads directly into the parser buffer.

**2. The conductor decides order.** The three threads (Main / I/O `"KittyChildMon"` / Talk `"KittyPeerMon"`, §2) are coordinated by `poll()` readiness plus delay-based scheduling (`input_delay`/`repaint_delay`, `kitty/child-monitor.c:L445-L446`). Out-of-band events don't wait their turn: the **self-pipe wakeup** (`wakeup()` "forcing it to exit from poll()", `kitty/child-monitor.c:L298-L299`; primitives in `kitty/loop-utils.h`) makes an idle `poll()` return immediately so resizes, signals, and queued writes are serviced promptly. A surge of child output does not become a render storm, because the I/O thread coalesces reads and only wakes the main loop about once per `input_delay` (`kitty/child-monitor.c:L1566-L1569`) — visible as the alternating `wakeups_happened: 0/1` ticks in the §2.1 log.

**3. Every byte is classified.** Read bytes flow to `parse_worker()` (`kitty/vt-parser.c:L1496`), which walks the stream once and dispatches each byte to the right destination: printable text → drawn cells; SGR/mode escapes → attribute/mode changes; OSC 133/OSC 7 → prompt-marking and cwd handlers on the Screen (§1.6, §3). Because there is exactly one classifier, text meaning and control meaning never drift apart — even when a hint is split mid-word (§3.2).

**4. State stays aligned.** As the parser applies commands, the Screen model updates cursor, lines, and command context together; OSC 133;A/C/D keep prompt/command/output regions marked and record the last command's exit status, while OSC 7 records the cwd (§3.1, §3.3). A real bash shell's actual markers (`\x1b]133;C;cmdline=echo\ hello\a`, `\x1b]7;kitty-shell-cwd://…\a`) land in `last_reported_cwd` and `last_cmd_exit_status` with no misalignment.

**5. Rendering settles — with anti-tearing.** Rendering is decoupled from parsing. Normally the main loop repaints on its `repaint_delay` cadence. When an application wants an atomic multi-write update, it enters **synchronized update / pending mode 2026**: the parser keeps consuming and applying input to the live model while the renderer keeps showing the last frame from a **snapshot** (§1.4), and on ESU — or after the **2 s safety-valve timeout** (§1.5, `kitty/screen.c:L2489-L2490,L2521-L2522`) if the app forgets — the renderer fetches the latest state in one go. That is how the interface "settles again" without tearing.

**6. Under a surge, rhythm is kept by pushing back.** If output outruns the parser, the buffer fills and the pipeline throttles rather than falling apart: `io_loop()` stops marking the PTY readable (`kitty/child-monitor.c:L1501`) and `read_bytes()` early-returns (`L1342`), so the child blocks on `write()` (§4.1–4.3) until the parser drains — then reads resume, with **no data lost** (all 8 MiB always accounted for). Over an SSH link, early keystrokes that arrive before the remote shell is ready are buffered as `leading_data` and replayed (§4.5), so the rhythm survives a laggy start.

The net effect: two entry doors, one conductor scheduling by readiness and delay, one per-byte classifier keeping text and meaning aligned, a snapshot-based synchronized-update mechanism for tear-free settling, and backpressure that throttles instead of dropping — together they turn a chaotic surge into a coherent flow.

---

## Appendix A — Entry-point walkthroughs (annotated call chains)

### A.1 Child-output path (PTY → screen)

```
io_loop()                             kitty/child-monitor.c:L1481   (I/O thread "KittyChildMon", L1489)
  poll(children_fds, ...)             kitty/child-monitor.c:L1505-L1512  (timeout = input_delay-derived or -1)
    events gated by parser space      kitty/child-monitor.c:L1501   vt_parser_has_space_for_input()? POLLIN:0
  read_bytes(fd, screen)              kitty/child-monitor.c:L1337   (called at L1531)
    vt_parser_create_write_buffer()   kitty/child-monitor.c:L1341   -> kitty/vt-parser.c:L1451
    if (!available_buffer_space) ...   kitty/child-monitor.c:L1342   (backpressure early-return)
    read(fd, buf, available_...)      kitty/child-monitor.c:L1345
    vt_parser_commit_write(...)       kitty/child-monitor.c (post-read)  -> kitty/vt-parser.c:L1465
  parse_worker()                      kitty/vt-parser.c:L1496       (classifies every byte)
    -> draw text | set/reset mode | OSC 133 prompt-marking (screen.c:L2328) | OSC 7 cwd (screen.c:L2393)
    -> pending mode 2026 -> screen_pause_rendering()  kitty/screen.c:L2506
```

### A.2 User-input path (keyboard → PTY)

```
key_callback(window, key, ...)        kitty/glfw.c:L430
  on_key_input(ev)                    kitty/glfw.c:L439
    on_key_input(w, ev)               kitty/keys.c:L166
      encode_key_for_tty (pyencode_key_for_tty)   kitty/keys.c:L311   (legacy or Kitty protocol; §1.2)
      if HANDLE_TERMIOS_SIGNALS && size==1:
          screen_send_signal_for_key  kitty/keys.c:L256-L257 -> kitty/screen.c:L2404
            -> window.py:L1116 -> child.py:L481 (VINTR/VSUSP/VQUIT -> SIGINT/SIGTSTP/SIGQUIT)
      else schedule_write_to_child()  kitty/keys.c:L259 (also L202/L253/L286/L289) -> kitty/child-monitor.c:L372
```

---

## Appendix B — Citations and caveats

### B.1 Every `file:line` used in this document

All line numbers below were re-confirmed against the running build (commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`).

| Area | Symbol | Location |
|------|--------|----------|
| Child-output read | `read_bytes()` | `kitty/child-monitor.c:L1337` |
| Backpressure early-return | `if (!available_buffer_space) return true;` | `kitty/child-monitor.c:L1342` |
| Read into parser buffer | `read(fd, buf, available_buffer_space)` | `kitty/child-monitor.c:L1345` |
| EIO = child gone | `errno != EIO` branch | `kitty/child-monitor.c:L1347-L1349` |
| I/O thread loop | `io_loop()` (decl L229) | `kitty/child-monitor.c:L1481` |
| I/O thread name | `"KittyChildMon"` | `kitty/child-monitor.c:L1489` |
| `read_bytes` call site | inside `io_loop` | `kitty/child-monitor.c:L1531` |
| Poll fd-readiness gate | `vt_parser_has_space_for_input()? POLLIN:0` | `kitty/child-monitor.c:L1501` |
| I/O poll timeout | input_delay-derived / -1 | `kitty/child-monitor.c:L1505-L1512` |
| Self-pipe drain (I/O) | `drain_fd(children_fds[0].fd)` | `kitty/child-monitor.c:L1513` |
| Signal dispatch | `read_signals(..., handle_signal)` | `kitty/child-monitor.c:L1519` |
| Wakeup doc | "forcing it to exit from poll()" | `kitty/child-monitor.c:L298-L299` |
| Input write hand-off | `schedule_write_to_child()` | `kitty/child-monitor.c:L372` |
| Main-loop wait | `set_maximum_wait(input_delay/repaint_delay)` | `kitty/child-monitor.c:L445-L446` |
| Paused-render wait branch | wait = pause expiry | `kitty/child-monitor.c:L443-L444` |
| Main-loop wakeup throttle | `wakeup_main_loop()` gated by input_delay | `kitty/child-monitor.c:L1562-L1569` |
| Resize handler | `process_pending_resizes()` (call L1233) | `kitty/child-monitor.c:L1043` |
| Close handler | `process_pending_closes()` (call L1246) | `kitty/child-monitor.c:L1098` |
| Signal handler | `handle_signal()` | `kitty/child-monitor.c:L1362-L1382` |
| PTY resize ioctl | `pty_resize()` / `ioctl(TIOCSWINSZ)` | `kitty/child-monitor.c:L577/L579` |
| State check / global state | `do_state_check` / `process_global_state` | `kitty/child-monitor.c:L1217/L1225` |
| Render debug line | `render()` | `kitty/child-monitor.c:L872` |
| Talk thread | `talk_loop()` / `"KittyPeerMon"` | `kitty/child-monitor.c:L1805/L1808` |
| Debug timestamp | `timed_debug_print` | `kitty/child-monitor.c:L29-L30` |
| Self-pipe primitives | `wakeup_fds`/`self_pipe`/`drain_fd` | `kitty/loop-utils.h:L33,L39,L40,L48,L52,L76` |
| Parser buffer size | `BUF_SZ (1024u*1024u)` | `kitty/vt-parser.c:L18` |
| Parser buffer array | `uint8_t buf[BUF_SZ + BUF_EXTRA]` | `kitty/vt-parser.c:L194` |
| Parser mutex | `lock` | `kitty/vt-parser.c:L206` |
| Write double-buffer | `{offset, sz, pending} write` | `kitty/vt-parser.c:L210` |
| Pending-mode start/stop | `screen_start/stop_pending_mode` | `kitty/vt-parser.c:L639/L644` |
| Pending-mode safety text | timeout/too-much-data | `kitty/vt-parser.c:L646-L648` |
| Create write buffer | `vt_parser_create_write_buffer()` (`*sz=BUF_SZ-offset` L1457) | `kitty/vt-parser.c:L1451` |
| Space check | `read.sz+write.pending<BUF_SZ` | `kitty/vt-parser.c:L1481` |
| Parse worker | `parse_worker()` | `kitty/vt-parser.c:L1496` |
| Parser API | create/commit/has_space | `kitty/vt-parser.h:L34/L35/L36` |
| Pause rendering | `screen_pause_rendering()` | `kitty/screen.c:L2506` |
| Pause check / auto-resume | `screen_check_pause_rendering()` | `kitty/screen.c:L2489-L2490` |
| Default 2000 ms / expires_at | pending timeout | `kitty/screen.c:L2521-L2522` |
| Pause snapshot | cursor/color_profile/linebuf copy | `kitty/screen.c:L2527-L2537` |
| Mode 2026 dispatch | `case PENDING_MODE << 5` | `kitty/screen.c:L1174` |
| Render-from-snapshot | `screen_update_cell_data()` | `kitty/screen.c:L2738` |
| OSC 133 marking | `shell_prompt_marking()` | `kitty/screen.c:L2328` (A `L2337-L2338`, C `L2341`) |
| OSC 7 cwd | `process_cwd_notification()` | `kitty/screen.c:L2393` (exposed `L4897`) |
| Bracketed paste | `paste_()` START/payload/END | `kitty/screen.c:L4573,L4586,L4587,L4588` |
| Unbracketed paste | `paste_bytes()` | `kitty/screen.c:L4600` |
| Signal-for-key | `screen_send_signal_for_key()` | `kitty/screen.c:L2404` |
| Modes: paste/pending/signals | 2004 / 2026 / termios-signals | `kitty/modes.h:L81-L83,L86,L89` |
| Pending mode const | `PENDING_MODE 2026` | `kitty/control-codes.h:L235` |
| Key entry | `key_callback` / `on_key_input(ev)` | `kitty/glfw.c:L430/L439` |
| Framebuffer resize | `framebuffer_size_callback` | `kitty/glfw.c:L330` |
| Key processing | `on_key_input()` (write sites L202/L253/L259/L286/L289) | `kitty/keys.c:L166` |
| Signal-key gate | `HANDLE_TERMIOS_SIGNALS` check | `kitty/keys.c:L256-L257` |
| Key encode | `pyencode_key_for_tty` | `kitty/keys.c:L311` |
| Shell integration install | `modify_shell_environ()` (`get_supported_shell_name` L219) | `kitty/shell_integration.py:L218` |
| Per-shell setup | fish/zsh/bash | `kitty/shell_integration.py:L16/L49/L70` |
| Native loop start | `boss.child_monitor.main_loop()` | `kitty/main.py:L234` |
| Window resize / SIGWINCH | `resize_pty` / debug print | `kitty/window.py:L863/L873`; `send_signal_for_key` `L1116` |
| Job-control signals | `send_signal_for_key` / SIGTSTP | `kitty/child.py:L481/L493` |
| Ring buffer FIFO | full/empty/read/write | `3rdparty/ringbuf/ringbuf.h:L44-L46`; `ringbuf.c:L122/L128/L241/L334` |
| Pager-hist ringbuf use | `initial_pagerhist_ringbuf_sz` | `kitty/history.c:L67,L76` |
| SSH bootstrap | leading_data / read loop / trap / die | `shell-integration/ssh/bootstrap.sh:L10,L16,L86,L91,L98,L139,L147` |
| SSH leading-data test | `test_ssh_leading_data` | `kitty_tests/ssh.py:L158-L169` |
| Harness parse path | `parse_bytes()` / `test_*` methods | `kitty_tests/__init__.py:L30`; `kitty/screen.c:L4755/L4762/L4772` |
| Build targets | `all:` / `debug-event-loop:` | `Makefile:L12-L13/L25-L26` |
| Test launcher | shebang / import | `test.py:L1/L8` |
| Runtime versions | Python `>=3.8` / Go `1.22` | `pyproject.toml:L2`; `go.mod:L3` |

### B.2 Claims labeled inferred (read from source, not directly observed at runtime)

- **§1.4** — that `screen_update_cell_data()` (`kitty/screen.c:L2738`) renders from `paused_rendering.linebuf` while paused (the snapshot *copy* and the mode transition are observed; the render-read of the snapshot is inferred from reading).
- **§1.8** — job-control suspend/resume (`SIGTSTP`/`SIGCONT`) is traced through `keys.c` → `screen.c` → `window.py` → `child.py` and cited, but a live signal delivery was not separately captured (requires `HANDLE_TERMIOS_SIGNALS` enabled by the running program).
- **§4.4** — the `3rdparty/ringbuf/` FIFO plays **no** role in the input/backpressure path (its only usage sites, in `kitty/history.c`, are observed via grep; "no role in input path" is the inferred conclusion).
- **§4.5** — the SSH kitten has **no reconnection logic** and a real TCP disconnect was **not** exercised (the harness uses a local PTY as the transport); leading-data buffering and clean teardown *are* observed via `test_ssh_leading_data`.

### B.3 Non-canonical values

- **None.** No value reported above was obtained through the remote-control interface (`kitty/rc/`) or a debug hook substituted for the real entry point. The user-input encodings (§1.2) were produced by `encode_key_for_tty` — the same function `on_key_input()` calls — and the child-output, pause/resume, paste, resize, backpressure, and shell-integration captures all flow through the real parser/PTY paths (or a real windowed GUI under Xvfb for the render-dependent safety-valve timeout). The `kitty/rc/` remote-control path is *described* only where relevant and never used as an observation source.

### B.4 Method caveats

- Render-dependent behavior (the 2 s pause safety valve in §1.5 and the event-loop stream in §2) was captured in a **real windowed kitty under Xvfb + software GL (llvmpipe)**; all other captures use the PTY-driven `kitty_tests` harness, which drives the same C parser API the I/O thread uses (§0.3).
- Magnitude/timing values were run at stated scale (8 MiB burst; 5×256 KiB into a 1 MiB buffer) and confirmed across ≥2 runs; the one genuine run-to-run spread (§4.3) was reproduced and explained as a measurement artifact rather than stabilized away.
- This investigation was strictly read-only: the only change to the repository is this single document; all temporary observation scripts lived under `/tmp` and were removed afterward.

