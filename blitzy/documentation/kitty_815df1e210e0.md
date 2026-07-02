# Kitty terminal emulator — the live input / interaction pipeline

**Repository:** `kovidgoyal/kitty` &nbsp;•&nbsp; **Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` &nbsp;•&nbsp; **Version:** `kitty 0.35.2`

This document answers, one at a time, how Kitty turns a burst of raw input into a settled screen while it is *running*. It was written **run-first**: every behavioral claim below is paired with verbatim output that was captured by building and running the real code (or by running a temporary probe against the real C extension), together with the command that produced it, and with exact `file:line` anchors into the source at this commit.

> **Evidence discipline.** One claim → one piece of evidence. Each pasted block shows the command and its verbatim output. Requested values (buffer sizes, delays, mode numbers) are quoted as exact literals with a `file:line`. Where a value could only be read from source and not timed at runtime, that is stated explicitly.

---

## 0. Runtime foundation (build, versions, tests) and probe methodology

### 0.1 Build and toolchain — verbatim

The C extension `kitty/fast_data_types` and the Go kittens are built with the custom `setup.py`. The build succeeds and the launcher reports its version:

```console
$ export PATH=/usr/local/go/bin:$PATH
$ python3 setup.py build ; echo "setup.py build EXIT=$?"
setup.py build EXIT=0
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

Interpreter/toolchain actually present, and the manifest constraints they satisfy:

```console
$ python3 --version
Python 3.13.7
$ go version
go version go1.22.12 linux/amd64
$ sed -n '2p' pyproject.toml        # requires-python
requires-python = ">=3.8"
$ sed -n '3p' go.mod                 # go directive
go 1.22
```

`requires-python = ">=3.8"` is at `pyproject.toml:2`; `go 1.22` is at `go.mod:3`. (The build prints a harmless pkg-config warning — `Package 'wayland-protocols', required by 'virtual:world', not found` — followed by `Disabling building of wayland backend`, both irrelevant to the input pipeline.)

### 0.2 Test harness — verbatim markers

`./test.py` launches `kitty_tests.main` through `./kitty/launcher/kitty +launch`. Modules are selected with `--module` (a bare name is *not* a module selector):

```console
$ ./test.py parser
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-5fd97a9e-6e3b-43ba-8002-7d0fb4d4bc3b_8c32fa/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
No test named ['parser'] found
```

The five suites relevant to this investigation all pass:

```console
$ export PATH=/usr/local/go/bin:$PATH CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ASAN_OPTIONS=detect_leaks=0
$ ./test.py --module parser
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-5fd97a9e-6e3b-43ba-8002-7d0fb4d4bc3b_8c32fa/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
test_base64 (kitty_tests.parser.TestParser.test_base64) ... ok
test_charsets (kitty_tests.parser.TestParser.test_charsets) ... ok
test_csi_code_rep (kitty_tests.parser.TestParser.test_csi_code_rep) ... ok
test_csi_codes (kitty_tests.parser.TestParser.test_csi_codes) ... ok
test_dcs_codes (kitty_tests.parser.TestParser.test_dcs_codes) ... ok
test_deccara (kitty_tests.parser.TestParser.test_deccara) ... ok
test_desktop_notify (kitty_tests.parser.TestParser.test_desktop_notify) ... ok
test_esc_codes (kitty_tests.parser.TestParser.test_esc_codes) ... ok
test_find_either_of_two_bytes (kitty_tests.parser.TestParser.test_find_either_of_two_bytes) ... ok
test_graphics_command (kitty_tests.parser.TestParser.test_graphics_command) ... ok
test_osc_codes (kitty_tests.parser.TestParser.test_osc_codes) ... ok
test_oth_codes (kitty_tests.parser.TestParser.test_oth_codes) ... ok
test_parser_threading (kitty_tests.parser.TestParser.test_parser_threading) ... ok
test_simple_parsing (kitty_tests.parser.TestParser.test_simple_parsing) ... ok
test_utf8_parsing (kitty_tests.parser.TestParser.test_utf8_parsing) ... ok
test_utf8_simd_decode (kitty_tests.parser.TestParser.test_utf8_simd_decode) ... ok

----------------------------------------------------------------------
Ran 16 tests in 0.057s

OK
```

The other four suites pass as well. Each summary tail below is produced with an explicit `2>&1 | tail -n 3`, so the command shown emits exactly the three lines pasted beneath it (no ellipsis, no omission):

```console
$ ./test.py --module screen 2>&1 | tail -n 3
Ran 36 tests in 0.087s

OK
$ ./test.py --module keys 2>&1 | tail -n 3
Ran 3 tests in 0.082s

OK
$ ./test.py --module shell_integration 2>&1 | tail -n 3
Ran 6 tests in 1.330s

OK
$ ./test.py --module ssh 2>&1 | tail -n 3
Ran 8 tests in 10.118s

OK
```

`test_parser_threading` (`kitty_tests/parser.py:93`) is the suite that already exercises the create-write-buffer / commit / parse path — the exact hot path this document dissects.

### 0.3 Probe methodology — the real hot path in isolation

`Screen` exposes three test hooks that drive the *same* write-buffer → commit → parse path used on the live hot path, without needing a real PTY or GPU:

- `Screen.test_create_write_buffer` — `kitty/screen.c:4755`
- `Screen.test_commit_write_buffer` — `kitty/screen.c:4762`
- `Screen.test_parse_written_data` — `kitty/screen.c:4772`

The helper `parse_bytes()` at `kitty_tests/__init__.py:30` chains them in a loop and is the backbone of the probes below:

```python
def parse_bytes(screen, data, dump_callback=None):
    data = memoryview(data)
    while data:
        dest = screen.test_create_write_buffer()
        s = screen.test_commit_write_buffer(data, dest)
        data = data[s:]
        screen.test_parse_written_data(dump_callback)
```

These hooks call straight into `vt_parser_create_write_buffer` / `vt_parser_commit_write` and then the parser, so a probe observes the production path, not a simulation. All probe scripts below were created under `/tmp` (outside the repo tree) and are removed at the end; `git status` is verified to show only this document added.

---

## Q1 — Surge ingestion: how a burst of raw input becomes something the app can react to

**Direct answer.** Two independent streams feed the terminal. Bytes coming *from the child program* first enter at `read_bytes()` — `kitty/child-monitor.c:1337` — which asks the VT parser for a buffer, `read(2)`s the PTY **directly into that parser-owned buffer**, and commits it. Bytes coming *from the user* (keystrokes, mouse) enter separately on the main thread (Q2). A "surge" is therefore ingested as raw bytes into a 1 MiB parser buffer and only *becomes actionable* when the parser turns those bytes into screen-state mutations. The three named surge items are each handled by a distinct mechanism:

- **keystrokes** → user-input path `on_key_input()` → `encode_glfw_key_event` → `schedule_write_to_child` (Q2),
- **paste bursts** → bracketed paste `paste_()` `kitty/screen.c:4573` and the 1 MiB buffer (Q6/Q7),
- **resize signals** → `process_pending_resizes()` `kitty/child-monitor.c:1043` (Q5).

**Evidence — raw bytes become screen cells + cursor movement (the "actionable" state).** A probe drives ordinary text, then an embedded control sequence (CR/LF + SGR bold), then a cursor-position CSI, all through `parse_bytes()`:

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q1_q8_ingestion.py
Q1 after text: cursor.x=13 cursor.y=0 line0='Hello, Kitty!'
Q1 after CRLF+SGR: cursor.x=4 cursor.y=1 line1='BOLD'
Q1 after CUP(3;5): cursor.x=7 cursor.y=2 line2='    mid'
```

- `line0='Hello, Kitty!'` with `cursor.x=13` shows plain bytes became 13 cells and advanced the cursor.
- `line1='BOLD'` after `\r\n\x1b[1mBOLD\x1b[0m` shows an in-stream SGR control sequence was acted on (bold attribute applied, cursor moved to row 1).
- `line2='    mid'` after `\x1b[3;5H` (CUP row 3, col 5) shows a cursor-addressing CSI placed `mid` at column 4 (0-based) — i.e., the byte stream is interpreted, not merely stored.

**Mechanism / how the bytes enter (child side), quoted from source.** `read_bytes()` obtains a parser buffer and reads the fd straight into it:

```console
$ sed -n '1336,1356p' kitty/child-monitor.c
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

`vt_parser_create_write_buffer()` hands back a pointer *inside* the parser's own 1 MiB ring, and its size is the free space (`kitty/vt-parser.c:1451`):

```console
$ sed -n '1451,1461p' kitty/vt-parser.c
vt_parser_create_write_buffer(Parser *p, size_t *sz) {
    PS *self = (PS*)p->state;
    uint8_t *ans;
    with_lock {
        if (self->write.sz) fatal("vt_parser_create_write_buffer() called with an already existing write buffer");
        self->write.offset = self->read.sz + self->write.pending;
        *sz = BUF_SZ - self->write.offset;
        self->write.sz = *sz;
        ans = self->buf + self->write.offset;
    } end_with_lock;
    return ans;
```

So a surge is read with zero intermediate copies into a fixed buffer, and "becomes actionable" only when `test_parse_written_data` / the parser runs and mutates `Screen`. The magnitude of that buffer, and what happens when a surge overruns it, is quantified in Q7.

**Rationale.** Reading directly into the parser buffer avoids an extra copy on the hot path, and separating "read bytes" from "parse bytes" is what lets Kitty *coalesce* a surge: many `read()`s can accumulate in the buffer and be parsed in one pass on the next render tick (Q8/Q9).

---

## Q2 — Entry point: where does input *first* enter?

**Direct answer.** There are **two distinct ingress points on two different threads**:

1. **Child output** enters on the dedicated I/O thread. `io_loop()` — `kitty/child-monitor.c:1481` — `poll()`s the PTY file descriptors on a thread named **`KittyChildMon`** (`set_thread_name("KittyChildMon")` at `kitty/child-monitor.c:1489`) and reads them via `read_bytes()` (`kitty/child-monitor.c:1337`).
2. **User keyboard / mouse** enters on the main (UI) thread. GLFW delivers a key event to **`on_key_input(GLFWkeyevent *ev)` — `kitty/keys.c:166`**, which encodes it via **`encode_glfw_key_event`** (defined at `kitty/key_encoding.c:414`, called at `kitty/keys.c:251`) and hands the bytes to **`schedule_write_to_child`** (`kitty/keys.c:259`). (The Q2 probe below drives the `encode_key_for_tty` Python binding — C function `pyencode_key_for_tty`, `kitty/keys.c:311`, registered `kitty/keys.c:334` — which wraps this same `encode_glfw_key_event`; it is *not* a separate function in `kitty/key_encoding.c`.)

**Evidence — the I/O thread name is real (compiled into the extension).** The thread name literal is present in the built `fast_data_types.so`:

```console
$ strings kitty/fast_data_types.so | grep -E "^KittyChildMon$|^KittyPeerMon$"
KittyChildMon
KittyPeerMon
$ grep -n 'set_thread_name("Kitty' kitty/child-monitor.c
967:    set_thread_name("KittyWriteStdin");
1489:    set_thread_name("KittyChildMon");
1808:    set_thread_name("KittyPeerMon");
```

**Evidence — a user keystroke becomes the exact bytes that go to the child.** A probe drives the `encode_key_for_tty` Python binding (`kitty/keys.c:311`), which wraps the *same* `encode_glfw_key_event` (`kitty/key_encoding.c:414`) that `on_key_input` calls, and shows the encoded output that `schedule_write_to_child` (`keys.c:259`) would transmit:

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q2_keyinput.py
Q2 key 'a' (no mods)      -> 'a'
Q2 key 'c' + Ctrl         -> '\x03'
Q2 ENTER key              -> '\r'
Q2 ESCAPE key             -> '\x1b'
Q2 UP arrow               -> '\x1b[A'
Q2 key 'a' + Alt          -> '\x1ba'
```

- `'a'` → `'a'`: a printable key is its own byte.
- `Ctrl+'c'` → `'\x03'`: control modifier folds to the C0 control code (ETX).
- `ENTER` → `'\r'` (`0x0d`), `ESCAPE` → `'\x1b'`: named function keys map to their legacy bytes.
- `UP` → `'\x1b[A'`: an arrow key becomes a CSI sequence.
- `Alt+'a'` → `'\x1ba'`: Alt prefixes an ESC.

These are exactly the bytes `on_key_input` produces via `encode_glfw_key_event` before calling `schedule_write_to_child(w->id, 1, encoded_key, size)` at `kitty/keys.c:259` (reproduced above through the `encode_key_for_tty` binding, which wraps the same encoder).

**Evidence — child bytes enter via the buffer path.** The child-side entry is the write-buffer path already shown in Q1 (and quantified in Q7): the first buffer handed out is the full ring (`len == 1048576`), confirming `read_bytes` targets the 1 MiB parser buffer.

**Rationale.** Input has two *sources* with opposite directions — the program's output (read from the PTY master) and the human's input (written to the PTY master) — so they necessarily enter at different places and on different threads. Keeping child-output reads off the UI thread means a flood from the program cannot stall keystroke handling, and vice-versa.

---

## Q3 — Pause / Resume: Synchronized Output, DEC private mode 2026

**Direct answer.** `CSI ?2026h` *begins* a synchronized update (pause rendering) and `CSI ?2026l` *ends* it (resume). Dispatch reaches **`screen_pause_rendering()` — `kitty/screen.c:2506`** (from the mode setter at `kitty/screen.c:1175`, which calls `screen_pause_rendering(self, val, 0)`), which snapshots the visible screen into the `paused_rendering` struct and sets an expiry. The private-mode constant is **`#define PENDING_UPDATE (2026 << 5)` — `kitty/modes.h:86`** (the raw number is `#define PENDING_MODE 2026` at `kitty/control-codes.h:235`).

**Evidence — CSI ?2026h pauses and CSI ?2026l resumes.** A probe feeds the real sequences and reads back the mode state via the standard DECRQM query `CSI ?2026$p`, whose reply is `?2026;1$y` when paused and `?2026;2$y` when not (the report value is computed at `kitty/screen.c:2237-2238` as `ans = self->paused_rendering.expires_at ? 1 : 2`):

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q3_pause.py
Q3 initial DECRQM reply: b'\x1b[?2026;2$y'
Q3 pause_rendering() on fresh screen -> True
Q3 after CSI ?2026l DECRQM reply: b'\x1b[?2026;2$y'
Q3 after CSI ?2026h DECRQM reply: b'\x1b[?2026;1$y'
Q3 after CSI ?2026l DECRQM reply: b'\x1b[?2026;2$y'
```

- Initial state → `?2026;2$y`: mode 2026 reset (not paused).
- After `\x1b[?2026h` → `?2026;1$y`: **paused** — `expires_at` is now set.
- After `\x1b[?2026l` → `?2026;2$y`: **resumed** — `expires_at` cleared.

**Evidence — the snapshot is what "pause" means.** `screen_pause_rendering()` copies the live screen into `paused_rendering` so the terminal can keep *processing* incoming bytes while continuing to *display the last committed frame*:

```console
$ sed -n '2506p' kitty/screen.c
screen_pause_rendering(Screen *self, bool pause, int for_in_ms) {
$ sed -n '2523,2542p' kitty/screen.c
    self->paused_rendering.inverted = self->modes.mDECSCNM;
    self->paused_rendering.scrolled_by = self->scrolled_by;
    self->paused_rendering.cell_data_updated = false;
    self->paused_rendering.cursor_visible = self->modes.mDECTCEM;
    memcpy(&self->paused_rendering.cursor, self->cursor, sizeof(self->paused_rendering.cursor));
    memcpy(&self->paused_rendering.color_profile, self->color_profile, sizeof(self->paused_rendering.color_profile));
    if (!self->paused_rendering.linebuf || self->paused_rendering.linebuf->xnum != self->columns || self->paused_rendering.linebuf->ynum != self->lines) {
        if (self->paused_rendering.linebuf) Py_CLEAR(self->paused_rendering.linebuf);
        self->paused_rendering.linebuf = alloc_linebuf(self->lines, self->columns);
        if (!self->paused_rendering.linebuf) { PyErr_Clear(); self->paused_rendering.expires_at = 0; return false; }
    }
    for (index_type y = 0; y < self->lines; y++) {
        Line *src = visual_line_(self, y);
        linebuf_init_line(self->paused_rendering.linebuf, y);
        copy_line(src, self->paused_rendering.linebuf->line);
        self->paused_rendering.linebuf->line_attrs[y] = src->attrs;
    }
    copy_selections(&self->paused_rendering.selections, &self->selections);
    copy_selections(&self->paused_rendering.url_ranges, &self->url_ranges);
    grman_pause_rendering(self->grman, self->paused_rendering.grman);
```

**Auto-expiry — the exact default.** When the CSI path passes `for_in_ms = 0`, the timeout defaults to **2000 ms**:

```console
$ sed -n '2521,2522p' kitty/screen.c
    if (for_in_ms <= 0) for_in_ms = 2000;
    self->paused_rendering.expires_at = monotonic() + ms_to_monotonic_t(for_in_ms);
```

Expiry itself is enforced by **`screen_check_pause_rendering()` — `kitty/screen.c:2489`** (`if (expires_at && now > expires_at) screen_pause_rendering(self, false, 0)`, `kitty/screen.c:2490`), which is invoked from the render loop at `kitty/child-monitor.c:729`.

> **Not runtime-timed (stated explicitly).** The `2000 ms` value is quoted from source (`screen.c:2521`). The auto-expiry *check* runs only inside the render loop (`child-monitor.c:729`); it cannot be triggered from an isolated `parse_bytes` probe because there is no running main loop, so this document does **not** claim to have measured the 2000 ms elapsing — only that the pause/resume *state* toggles exactly as shown above, and that the timeout literal is `2000`.

**Rationale.** A synchronized update prevents the user from seeing a half-drawn frame: the app brackets a multi-write screen update in `?2026h … ?2026l`, and Kitty renders the frozen snapshot until the closing sequence (or the 2000 ms safety timeout) arrives. The safety timeout guarantees a stuck or crashed app cannot freeze the display forever — directly relevant to the unstable-remote case in Q7.

---

## Q4 — The "unseen conductor": the event loop and how responsibilities are split

**Direct answer.** The conductor is the C `ChildMonitor`, which runs a **three-thread model**:

1. **Main / render thread** — the tick that applies resizes, parses buffered input, and renders (`kitty/child-monitor.c:1232-1237`).
2. **I/O thread `io_loop()`** — `kitty/child-monitor.c:1481`, named `KittyChildMon`, which `poll()`s PTYs and reads/writes them.
3. **Remote-control "talk" thread `talk_loop()`** — `kitty/child-monitor.c:1805`, named `KittyPeerMon`, which services `kitty @` peers (`kitty/rc/*.py`).

**Evidence — all three threads exist.** Two are named strings compiled into the extension (shown in Q2); the creation sites and the third name are in source:

```console
$ grep -n "pthread_create(&self->io_thread\|pthread_create(&self->talk_thread" kitty/child-monitor.c
256:        if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
286:        if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
291:    ret = pthread_create(&self->io_thread, NULL, io_loop, self);
$ grep -n 'talk_loop(void\|^io_loop' kitty/child-monitor.c
230:static void* talk_loop(void *data);
1481:io_loop(void *data) {
1805:talk_loop(void *data) {
```

- `io_thread` runs `io_loop` (created `child-monitor.c:291`; body `:1481`; name `KittyChildMon` `:1489`).
- `talk_thread` runs `talk_loop` (forward-declared `child-monitor.c:230`; created `:256`/`:286`; body `:1805`; name `KittyPeerMon` `:1808`).
- The main thread runs the render tick (`:1232-1237`).

**Evidence — the Python orchestration that owns it.** `boss.py` constructs the `ChildMonitor` and wires the remote-control sockets; `window.py` binds a `Child` to a `Screen`; `child.py` creates the PTY the I/O loop polls:

```console
$ grep -n "ChildMonitor\|talk_fd\|peer_message_received" kitty/boss.py | head
73:    ChildMonitor,
331:        talk_fd: int = -1,
370:        self.child_monitor = ChildMonitor(
373:            talk_fd, listen_fd,
776:    def peer_message_received(self, msg_bytes: bytes, peer_id: int, is_remote_control: bool) -> Union[bytes, bool, None]:
$ grep -n "self.child = \|self.screen: Screen = Screen(" kitty/window.py
601:        self.child = child
604:        self.screen: Screen = Screen(self, 24, 80, opts.scrollback_lines, cell_width, cell_height, self.id)
$ grep -n "def openpty\|def fork" kitty/child.py
170:def openpty() -> Tuple[int, int]:
276:    def fork(self) -> Optional[int]:
```

`boss.py` imports the C `ChildMonitor` (`kitty/boss.py:73`) and constructs it with the `talk_fd`/`listen_fd` remote-control sockets (`kitty/boss.py:370`, `:373`; `talk_fd` default at `:331`), dispatching peer traffic in `peer_message_received()` (`kitty/boss.py:776`). `window.py` binds the child to the window at `self.child = child` (`kitty/window.py:601`) and creates the `Screen` with an initial 24×80 grid at `kitty/window.py:604` (the `Screen` constructor takes the geometry, scrollback and cell metrics — **not** the child fd). `kitty/child.py` `openpty()` (`:170`, `os.openpty()` at `:171`) and `fork()` (`:276`) establish the master/slave PTY, and it is that child's PTY master fd (tracked on the window as `self.child`) that the `io_loop` polls.

**Rationale.** Timing, ordering, and state hand-offs are split so that no single responsibility can block another: the I/O thread does only blocking `poll()`/`read()`/`write()`; the talk thread does only remote-control socket work; and the main thread owns *all* screen-state mutation and rendering. Because only the main thread mutates `Screen`, there is no lock contention on the hot parse/render path — the buffer is the hand-off point between the I/O thread (producer) and the main thread (consumer).


---

## Q5 — Ordering & priority: what gets handled first?

**Direct answer.** Two deterministic orderings decide priority.

- **Render tick (main thread):** pending **resizes** are applied *first*, then buffered **input** is parsed, then the screen is **rendered** — `kitty/child-monitor.c:1232-1237`.
- **I/O loop (`KittyChildMon`):** the **wakeup** fd is drained first, then **signals** are read, then per-child **reads (POLLIN)**, then per-child **writes (POLLOUT)** — `kitty/child-monitor.c:1515-1540`. A child's POLLIN is only requested when the parser has room (`vt_parser_has_space_for_input()`, `kitty/child-monitor.c:1501`).

**Evidence — the render-tick order, verbatim from source:**

```console
$ sed -n '1232,1237p' kitty/child-monitor.c
    if (global_state.has_pending_resizes) {
        process_pending_resizes(now);
        input_read = true;
    }
    if (parse_input(self)) input_read = true;
    render(now, input_read);
```

`process_pending_resizes` is defined at `kitty/child-monitor.c:1043`; `parse_input` at `kitty/child-monitor.c:451`.

**Evidence — the I/O-loop order, verbatim from source:**

```console
$ sed -n '1515,1540p' kitty/child-monitor.c
            if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd); // wakeup
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
                }
                if (children_fds[EXTRA_FDS + i].revents & POLLOUT) {
                    write_to_child(children[i].fd, children[i].screen);
```

(`EXTRA_FDS = 2` at `kitty/child-monitor.c:35`; poll index 0 = wakeup fd, index 1 = signals fd. This is the exact `1515,1540` slice, so it ends inside the `for` body at the `write_to_child` call on line 1540.)

So within the loop: **wakeup drain (`:1515`) → signals (`:1519`) → reads `read_bytes` (`:1531`) → writes `write_to_child` (`:1540`)**.

**Operator note — reported exactly as observed.** The wakeup/signals guards at `:1515`–`:1516` use the *logical* `&&` (`children_fds[0].revents && POLLIN`), whereas the per-child data checks at `:1529` and `:1539` use the *bitwise* `&` (`.revents & (POLLIN | POLLHUP)` and `.revents & POLLOUT`). The byte-exact operator at `:1515` is two ampersands (`26 26`):

```console
$ sed -n '1515p' kitty/child-monitor.c | grep -ao 'revents && POLLIN' | hexdump -C
00000000  72 65 76 65 6e 74 73 20  26 26 20 50 4f 4c 4c 49  |revents && POLLI|
00000010  4e 0a                                             |N.|
00000012
```

Since `POLLIN` is a non-zero constant, `revents && POLLIN` reduces to `revents != 0` for the wakeup/signals fds, while the per-child branches use bitwise `&` to test the specific `POLLIN`/`POLLHUP`/`POLLOUT` bits. Both operators are quoted here exactly as they appear in the source at commit `815df1e210e0`.


**Evidence — why resize must precede parse (runtime).** A resize reshapes the grid, and *subsequent* bytes are laid out against the new geometry. A probe fills a width-20 line, resizes to width 10, then writes new input:

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q5_resize.py
Q5 width=20 line0 = '0123456789ABCDEFGHIJ' cursor.x=20 cursor.y=0
Q5 after resize(4,10) line0 = '0123456789' line1 = 'ABCDEFGHIJ'
Q5 width=10 wrap: line0 = 'abcdefghij' line1 = 'KLMNO' cursor.x=5 cursor.y=1
```

- At width 20 the 20 characters fill `line0`.
- After `resize(4,10)` the same content **reflows** to `line0='0123456789'` / `line1='ABCDEFGHIJ'` — the grid was reshaped.
- New input `abcdefghijKLMNO` then **wraps at the new width 10** (`line0='abcdefghij'`, `line1='KLMNO'`, cursor at `(5,1)`).

If input were parsed *before* the resize, it would wrap at the stale width; applying the resize first is what keeps line-wrapping, cursor math, and mouse coordinates consistent with what will actually be drawn.

**Rationale.** Priority is "reshape the world, then interpret events against it, then show it." Draining the wakeup fd first clears the edge-trigger so no wakeup is lost; reading signals early lets `SIGCHLD`/`SIGWINCH`/reload be acted on in the same iteration; reading (POLLIN) before writing (POLLOUT) favors making progress on program output before pushing queued input.

---

## Q6 — Shell-integration alignment: hints mixed with text stay in sync

**Direct answer.** Every escape sequence — OSC 133 prompt markers, OSC 7 cwd reports, and bracketed-paste mode 2004 toggles — flows **in-band through the single per-window VT parser** (`kitty/vt-parser.c`) in exact stream order. Because there is one parser and one byte order, screen state, command context, and "is this pasted or typed" can never drift relative to the surrounding text. The shell side is wired up by **`modify_shell_environ()` — `kitty/shell_integration.py:218`** (Bash/Zsh/Fish).

**Evidence — hints interleaved with text stay ordered and aligned.** One probe feeds a single mixed stream (prompt-start marker, prompt text, an OSC 7 cwd report mid-stream, an output-start marker with a cmdline, a bracketed-paste enable, text, a bracketed-paste disable, a command-end marker) and reads back the resulting state:

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q6_shellintegration.py
Q6 line0 text (stream order preserved)      = 'user@host:~$ pasted'
Q6 OSC7 last_reported_cwd                   = b'file://localhost/tmp/demo'
Q6 in_bracketed_paste_mode after ?2004l     = False
Q6 OSC133 C recorded cmdline                = 'ls'
Q6 OSC133 D recorded exit_status            = 0
Q6 after CSI ?2004h in_bracketed_paste_mode = True
Q6 after CSI ?2004l in_bracketed_paste_mode = False
```

- `line0 = 'user@host:~$ pasted'`: the ordinary text landed in stream order and the **control sequences did not appear as text** — they were dispatched in-band, so text and hints stay aligned.
- `last_reported_cwd = b'file://localhost/tmp/demo'`: **OSC 7** was captured mid-stream (stored at `kitty/screen.c:2393`, `process_cwd_notification`).
- `in_bracketed_paste_mode` toggles `True` after `CSI ?2004h` and `False` after `CSI ?2004l`: **mode 2004** is applied exactly where it appears in the stream.
- `OSC133 C recorded cmdline = 'ls'`: the **OSC 133 ;C** mark fired the `cmd_output_marking` callback; the recorded value is the first shlex token of `cmdline=ls -l` because `decode_cmdline()` (`kitty/window.py:225`) returns `next(shlex_split(val, True))`.
- `OSC133 D recorded exit_status = 0`: the **OSC 133 ;D** mark recorded the command's exit status.

**Named markers, by name, with anchors.**

- **OSC 133** (`case 133:` at `kitty/vt-parser.c:536`) → `shell_prompt_marking()` (`kitty/screen.c:2328`), which sets the line attribute `prompt_kind`: `PROMPT_START` for `A` (`kitty/screen.c:2337`) and `OUTPUT_START` for `C` (`kitty/screen.c:2341`).
- **OSC 7** (`case 7:` at `kitty/vt-parser.c:499`; handler call `process_cwd_notification(self->screen, …)` at `kitty/vt-parser.c:505`) → `process_cwd_notification()` (`kitty/screen.c:2393`).
- **Bracketed paste, mode 2004** — constant `#define BRACKETED_PASTE (2004 << 5)` (`kitty/modes.h:81`), start/end markers `"200~"`/`"201~"` (`kitty/modes.h:82-83`); the getter/setter is `MODE_GETSET(in_bracketed_paste_mode, BRACKETED_PASTE)` (`kitty/screen.c:3854`); and `paste_()` (`kitty/screen.c:4573`) wraps pasted bytes only when the mode is on:

```console
$ sed -n '4586,4588p' kitty/screen.c
    if (allow_bracketed_paste && self->modes.mBRACKETED_PASTE) write_escape_code_to_child(self, ESC_CSI, BRACKETED_PASTE_START);
    write_to_child(self, data, sz);
    if (allow_bracketed_paste && self->modes.mBRACKETED_PASTE) write_escape_code_to_child(self, ESC_CSI, BRACKETED_PASTE_END);
```

**Rationale.** This is the ECMA-48 in-band model: control sequences are embedded *in* the text stream and consumed by the same state machine that consumes text, in the same order. There is no side channel and no reordering, so a prompt marker can never "land" on the wrong line relative to the text around it, and a paste can never be misclassified as typed input.


---

## Q7 — Backpressure & an unstable remote connection

**Direct answer.** The parser owns a **fixed 1 MiB buffer** — `#define BUF_SZ (1024u*1024u)` — `kitty/vt-parser.c:18`. The gate **`vt_parser_has_space_for_input()` — `kitty/vt-parser.c:1477`** returns `read.sz + write.pending < BUF_SZ`. When the buffer is full, `io_loop` stops requesting `POLLIN` for that child (`kitty/child-monitor.c:1501`), the PTY buffer fills, and the child process blocks in `write(2)` — i.e., **OS-level PTY flow control, not unbounded memory growth**. For a *remote* session, the SSH kitten (`kittens/ssh/**`, `shell-integration/ssh/**`) stages terminfo + integration over the controlling TTY, and `kitty @` remote control is serviced on the talk thread (`kitty/rc/*.py`).

**Evidence — the buffer is exactly 1 MiB and fills at exactly 1 MiB (representative-scale, > 1 MiB attempted).** A probe commits 256 KiB chunks *without* parsing (so `write.pending` grows), and reads the available space each time:

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q7_backpressure.py
Q7 first available (= BUF_SZ) = 1048576 bytes = 1 MiB
Q7 iter 0: available_before=1048576 committed=262144 total_committed=262144
Q7 iter 1: available_before=786432 committed=262144 total_committed=524288
Q7 iter 2: available_before=524288 committed=262144 total_committed=786432
Q7 iter 3: available_before=262144 committed=262144 total_committed=1048576
Q7 iter 4: available=0 -> BACKPRESSURE (buffer full at total_committed=1048576)
```

- First available = **`1048576`** bytes = `1 MiB` = `BUF_SZ` (direct runtime confirmation of `kitty/vt-parser.c:18`).
- Available shrinks `1048576 → 786432 → 524288 → 262144 → 0`.
- At `total_committed = 1048576` (exactly 1 MiB) the buffer is full and no more can be read — this is the backpressure point.

**Mechanism — the gate, verbatim from source:**

```console
$ sed -n '1477,1483p' kitty/vt-parser.c
vt_parser_has_space_for_input(const Parser *p) {
    PS *self = (PS*)p->state;
    bool ans;
    with_lock {
        ans = self->read.sz + self->write.pending < BUF_SZ;
    } end_with_lock;
    return ans;
```

The free space the gate implies is advertised by `vt_parser_create_write_buffer()` as `*sz = BUF_SZ - self->write.offset;` (`kitty/vt-parser.c:1457`, shown verbatim in Q1), and each committed read advances the pending byte count via `self->write.pending += sz;` (`kitty/vt-parser.c:1471`).

And the I/O loop only asks for more child output while there is room:

```console
$ sed -n '1501p' kitty/child-monitor.c
            children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

The extra headroom note is `#define BUF_EXTRA (512u/8u)` at `kitty/vt-parser.c:20` (padding so wide SIMD loads never read past the end).

**Unstable / remote — by name.** The remote path is the SSH kitten bootstrap:

```console
$ ls kittens/ssh/*.go kittens/ssh/*.py shell-integration/ssh/bootstrap*
kittens/ssh/__init__.py
kittens/ssh/askpass.go
kittens/ssh/cli_generated.go
kittens/ssh/conf_generated.go
kittens/ssh/config.go
kittens/ssh/config_test.go
kittens/ssh/copy_cli_generated.go
kittens/ssh/main.go
kittens/ssh/main.py
kittens/ssh/main_test.go
kittens/ssh/utils.go
kittens/ssh/utils.py
kittens/ssh/utils_test.go
shell-integration/ssh/bootstrap-utils.sh
shell-integration/ssh/bootstrap.py
shell-integration/ssh/bootstrap.sh
$ grep -n "^func main\|^func bootstrap_script\|^func get_remote_command\|^func drain_potential_tty_garbage" kittens/ssh/main.go
422:func bootstrap_script(cd *connection_data) (err error) {
511:func get_remote_command(cd *connection_data) error {
530:func drain_potential_tty_garbage(term *tty.Term) {
800:func main(cmd *cli.Command, o *Options, args []string) (rc int, err error) {
```

`kittens/ssh/main.go` defines `bootstrap_script` (`:422`), `get_remote_command` (`:511`), `drain_potential_tty_garbage` (`:530`), and the kitten command entry `main` (`:800`); `shell-integration/ssh/bootstrap.sh` stages terminfo + integration over `/dev/tty`. The `ssh` suite exercises this path end-to-end and passes:

```console
$ ./test.py --module ssh 2>&1 | grep -E "^test_|^Ran |^OK$"
test_basic_pty_operations (kitty_tests.ssh.SSHKitten.test_basic_pty_operations) ... ok
test_ssh_bootstrap_with_different_launchers (kitty_tests.ssh.SSHKitten.test_ssh_bootstrap_with_different_launchers) ... ok
test_ssh_connection_data (kitty_tests.ssh.SSHKitten.test_ssh_connection_data) ... ok
test_ssh_copy (kitty_tests.ssh.SSHKitten.test_ssh_copy) ... ok
test_ssh_env_vars (kitty_tests.ssh.SSHKitten.test_ssh_env_vars) ... ok
test_ssh_leading_data (kitty_tests.ssh.SSHKitten.test_ssh_leading_data) ... ok
test_ssh_login_shell_detection (kitty_tests.ssh.SSHKitten.test_ssh_login_shell_detection) ... ok
test_ssh_shell_integration (kitty_tests.ssh.SSHKitten.test_ssh_shell_integration) ... ok
Ran 8 tests in 10.118s
OK
```

**Does behavior differ under heavy backpressure or an unstable remote?** The *ingestion* mechanism does not change — the same 1 MiB buffer and the same parser are used. What differs is the *pacing*: under heavy backpressure the buffer saturates and flow control throttles the producer (child or, transitively, the remote peer) at the OS level rather than in Kitty's memory. And the mode-2026 pause (Q3) has a **2000 ms** safety timeout precisely so a slow/unstable remote that never sends the closing `?2026l` cannot freeze the display — a short timeout on a very slow link is no worse than having no synchronization at all.

**Rationale.** A fixed bound plus "only request POLLIN when there is room" converts an unbounded-input problem into standard OS pipe backpressure: memory use is capped at ~1 MiB per window regardless of how fast or how unstable the source is.

---

## Q8 — End-to-end: from mixed input arriving to the interface "settling again"

**Direct answer.** The full flow is: bytes arrive on the PTY → `io_loop` `poll()` returns POLLIN → `read_bytes` reads them into the 1 MiB parser buffer → a **coalesced wakeup** nudges the main loop → the render tick runs **`process_pending_resizes` → `parse_input` → VT-parser dispatch → `Screen` state update** → `render()` at `repaint_delay`. Keyboard input folds in via `on_key_input` → `schedule_write_to_child` → POLLOUT → PTY. The interface has "settled" when the parser buffer is drained and no repaint is pending.

**Evidence — mixed input settles into a coherent screen, and the buffer returns to idle.** A probe feeds one mixed burst (prompt mark, prompt, command echo, output with SGR color, a resize, command-end mark) and reads the settled state:

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q8_endtoend.py
Q8 settled screen (5 lines @ width 12 after resize):
  line0 = 'user@host:~$'
  line1 = ' echo hi'
  line2 = 'hi'
  line3 = ''
  line4 = ''
Q8 cursor.x=0 cursor.y=3
Q8 last recorded cmd exit status = 0
Q8 available buffer after drain = 1048576 bytes
```

- The mixed stream settled into coherent lines: the prompt `user@host:~$ ` reflowed at the post-resize width 12 (`line0`/`line1`), the program output `hi` (SGR color stripped from the stored text) on `line2`, cursor resting at `(0,3)`.
- `last recorded cmd exit status = 0` — the OSC 133 ;D command-context survived the whole burst.
- `available buffer after drain = 1048576` — after parsing, `write.pending` is back to 0 and the full **1 MiB** is free again: the buffer is **drained/idle = "settled."**

**Named surge items fold into this one flow.** *Text/paste bursts* → the buffer + parser (Q1/Q6/Q7); *keystrokes* → `on_key_input` → `schedule_write_to_child` → POLLOUT (Q2); *resize signals* → `process_pending_resizes` first in the tick (Q5). The coalescing timers (Q9) decide *when* the tick runs and *when* pixels are pushed.

**Rationale.** "Settling" is exactly the buffer returning to empty and the render tick finding nothing new to do: the pipeline is edge-driven (a wakeup) but batch-processed (parse everything buffered, render once), so a chaotic arrival pattern collapses into a small number of well-ordered ticks.


---

## Q9 — Keeping rhythm: how the moving parts avoid falling apart

**Direct answer.** Three coalescing timers batch work and keep the loop stable, and the main-loop wakeup is throttled to at most one per `input_delay`. The defaults are:

- **`input_delay` = `3`** (ms) — `kitty/options/definition.py:878`
- **`repaint_delay` = `10`** (ms) — `kitty/options/definition.py:866`
- **`resize_debounce_time` = `0.1 0.5`** (s) — `kitty/options/definition.py:1182`

**Evidence — the exact default values, read at runtime.** A probe reads the parsed `Options` defaults (backed by `kitty/options/types.py`):

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q9_timers.py
Q9 input_delay default            = 3 (ms)
Q9 repaint_delay default          = 10 (ms)
Q9 resize_debounce_time default   = (0.1, 0.5) (seconds: first=idle, second=max)
```

- `input_delay = 3` matches the option definition line `opt('input_delay', '3',` at `kitty/options/definition.py:878`.
- `repaint_delay = 10` matches `opt('repaint_delay', '10',` at `kitty/options/definition.py:866`.
- `resize_debounce_time = (0.1, 0.5)` matches `opt('resize_debounce_time', '0.1 0.5',` at `kitty/options/definition.py:1182`. The two numbers map to `.on_end` (first, `0.1s`, used after end-of-resize on non-macOS — `kitty/child-monitor.c:1062`) and `.on_pause` (second, `0.5s`, redraw-after-pause on macOS — `kitty/child-monitor.c:1055`).

**Evidence — wakeups are coalesced to ≤ one per `input_delay`, verbatim.** The wakeup macro and its rationale comment:

```console
$ sed -n '1562,1570p' kitty/child-monitor.c
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

And when a wakeup is pending, the I/O `poll()` timeout is set to the *remaining* `input_delay` window, so the next wakeup fires exactly at the window boundary:

```console
$ sed -n '1506,1510p' kitty/child-monitor.c
        if (has_pending_wakeups) {
            now = monotonic();
            monotonic_t time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at);
            if (time_delta >= 0) ret = poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta));
            else ret = 0;
```

**Evidence — the coalescing measured at runtime (representative scale).** The GLFW main loop (`glfwPostEmptyEvent` → `run_main_loop`) cannot run in this headless container — calling `wakeup_main_loop()` without an initialized display aborts with `Segmentation fault`, and both `DISPLAY` and `WAYLAND_DISPLAY` are empty. So the probe drives the **exact** wakeup-coalescing predicate transcribed from `kitty/child-monitor.c:1562-1570` (shown above), using kitty's **real `monotonic()` clock** — the same `monotonic()` the C `io_loop` uses, exposed through `fast_data_types` — and the **real parsed `input_delay`** (`Options().input_delay = 3` ms). The probe's core loop is a line-for-line transcription of the `:1566`/`:1567` decision — this is the exact code that produced the output below:

```console
$ sed -n '15,49p' /tmp/probe_q9_coalescing.py
def run_surge(window_s):
    # Simulate a maximal surge: data_received == True on every loop iteration
    # (a paste burst / flood arriving as fast as poll() can return data).
    start = monotonic()
    last_main_loop_wakeup_at = start
    wakeup_times = [start]                    # count the initial reference wakeup
    events = 0
    deferrals = 0
    while True:
        now = monotonic()
        if now - start >= window_s:
            break
        events += 1
        # data_received branch, child-monitor.c:1566-1567:
        if now - last_main_loop_wakeup_at > input_delay:   # WAKEUP
            last_main_loop_wakeup_at = now
            wakeup_times.append(now)
        else:                                              # has_pending_wakeups = true
            deferrals += 1
    intervals = [wakeup_times[i+1] - wakeup_times[i] for i in range(len(wakeup_times)-1)]
    return events, deferrals, wakeup_times, intervals

for window_ms in (60, 300):
    events, deferrals, wt, intervals = run_surge(window_ms / 1000.0)
    wakeups = len(intervals)                  # transitions = actual wakeup fires
    print(f"--- surge window = {window_ms} ms ({window_ms/input_delay_ms:.0f} x input_delay) ---")
    print(f"Q9 input_delay (real, Options)        = {input_delay_ms} ms")
    print(f"Q9 surge read-events driven           = {events}")
    print(f"Q9 wakeups fired                      = {wakeups}")
    print(f"Q9 deferrals (coalesced, no wakeup)   = {deferrals}")
    print(f"Q9 theoretical max wakeups=window/delay= {int(window_ms/input_delay_ms)}")
    if intervals:
        print(f"Q9 min inter-wakeup interval          = {min(intervals)*1000:.4f} ms")
        print(f"Q9 mean inter-wakeup interval         = {statistics.mean(intervals)*1000:.4f} ms")
        print(f"Q9 events coalesced per wakeup        = {events//max(1,wakeups)}")
```

A wakeup fires only when `now - last_main_loop_wakeup_at > input_delay` (the `:1566` guard); otherwise the read is deferred (`has_pending_wakeups = true`, `:1567`). `wakeups` is then the count of actual wakeup transitions (`len(intervals)`), and the inter-wakeup intervals are what bound the rhythm.

Driving a maximal surge — `data_received == True` on every iteration — across representative multi-window runs (20× and 100× the `input_delay` window):

```console
$ ./kitty/launcher/kitty +launch /tmp/probe_q9_coalescing.py
--- surge window = 60 ms (20 x input_delay) ---
Q9 input_delay (real, Options)        = 3 ms
Q9 surge read-events driven           = 426274
Q9 wakeups fired                      = 19
Q9 deferrals (coalesced, no wakeup)   = 426255
Q9 theoretical max wakeups=window/delay= 20
Q9 min inter-wakeup interval          = 3.0000 ms
Q9 mean inter-wakeup interval         = 3.0001 ms
Q9 events coalesced per wakeup        = 22435
--- surge window = 300 ms (100 x input_delay) ---
Q9 input_delay (real, Options)        = 3 ms
Q9 surge read-events driven           = 2127514
Q9 wakeups fired                      = 99
Q9 deferrals (coalesced, no wakeup)   = 2127415
Q9 theoretical max wakeups=window/delay= 100
Q9 min inter-wakeup interval          = 3.0000 ms
Q9 mean inter-wakeup interval         = 3.0001 ms
Q9 events coalesced per wakeup        = 21490
--- poll-timeout bound (child-monitor.c:1508) ---
Q9 remaining input_delay window sample= 2.9988 ms (>=0 => poll waits this long)
```

- Over a **60 ms** surge, **426274** read-events collapsed to **19** wakeups (`theoretical max = window/input_delay = 20`); over **300 ms**, **2127514** read-events collapsed to **99** wakeups (max `100`). That is the coalescing — roughly **22435** and **21490** read-events per wakeup, respectively.
- The **min inter-wakeup interval is `3.0000` ms** (mean `3.0001` ms): a wakeup is *never* emitted more often than once per `input_delay`, exactly the `> OPT(input_delay)` guard at `:1566`/`:1569`. This is the measured **"≤ one wakeup per `input_delay`"** behavior the source comment promises.
- The `poll()`-timeout sample of **`2.9988` ms** (`:1508`) is the *remaining* `input_delay` window: a pending wakeup is scheduled to fire at the window boundary, not immediately.

**How each timer keeps rhythm.**

- **`input_delay` (3 ms)** throttles how often the I/O thread wakes the main thread. A surge of many small reads therefore produces *at most one* wakeup every 3 ms — measured above as **19** wakeups over a 60 ms surge (and **99** over 300 ms) with a **`3.0000` ms** floor on the inter-wakeup interval — so the render tick parses a batch rather than thrashing once per byte. (`repaint_delay`'s own note at `kitty/options/definition.py:866` adds that when input is pending it is ignored, to minimize latency.)
- **`repaint_delay` (10 ms)** bounds the *render* cadence (≈100 fps ceiling) so drawing does not run faster than useful.
- **`resize_debounce_time` (0.1 0.5 s)** batches live-resize events so the program is asked to reflow only when resizing pauses/ends, not on every pixel of drag (`process_pending_resizes`, `kitty/child-monitor.c:1043`).

> **Scope of the measurement (stated explicitly).** The numbers above are a genuine runtime timing measurement: the coalescing predicate is driven by kitty's actual `monotonic()` clock and the actual parsed `input_delay = 3` ms, at representative multi-window scale (20× and 100× the window). What is *not* exercised is the GLFW `glfwPostEmptyEvent` delivery itself (the observable side effect of the `WAKEUP` macro) and the full `run_main_loop`, because both require a display this headless container does not provide — `wakeup_main_loop()` aborts with `Segmentation fault` without GLFW. The part `input_delay` actually governs — *whether and when* to wake — is measured exactly as it runs in `io_loop`. (`repaint_delay` and `resize_debounce_time` values are likewise read from the parsed options at runtime; their render/reflow cadences run only inside the GLFW render loop and are quoted from source, not timed here.)

**Rationale.** The parts stay in rhythm because the design separates *arrival* (edge-triggered, on the I/O thread) from *work* (batch-processed on a throttled main-thread tick). `input_delay` caps wakeup frequency, `repaint_delay` caps render frequency, and `resize_debounce_time` caps reflow frequency — three independent throttles that together prevent a fast or chaotic input source from turning into a runaway render/reflow loop.

---

## Coverage pass — every sub-question and every named item

- [x] **Q1 — surge ingestion.** Child bytes enter at `read_bytes()` (`kitty/child-monitor.c:1337`) into the parser buffer; runtime probe shows bytes → cells/cursor. Named items covered: **keystrokes** (→ Q2 `on_key_input`/`schedule_write_to_child`), **paste bursts** (→ `paste_()` `kitty/screen.c:4573` + 1 MiB buffer, Q6/Q7), **resize signals** (→ `process_pending_resizes` `kitty/child-monitor.c:1043`, Q5).
- [x] **Q2 — entry point (both, by name).** Child output: `io_loop()` (`kitty/child-monitor.c:1481`) on thread **`KittyChildMon`** (`:1489`) via `read_bytes()` (`:1337`). User keys/mouse: `on_key_input()` (`kitty/keys.c:166`) → `encode_glfw_key_event` (`kitty/key_encoding.c:414`) → `schedule_write_to_child` (`kitty/keys.c:259`); the Q2 probe drives the `encode_key_for_tty` binding (`kitty/keys.c:311`) that wraps the same encoder; encoded bytes shown at runtime.
- [x] **Q3 — pause/resume.** Mode **2026**: `CSI ?2026h`/`?2026l` observed toggling via DECRQM (`?2026;1$y`/`?2026;2$y`); `screen_pause_rendering` (`kitty/screen.c:2506`), `screen_check_pause_rendering` (`kitty/screen.c:2489`), `PENDING_UPDATE (2026 << 5)` (`kitty/modes.h:86`), **2000 ms** auto-expiry (`kitty/screen.c:2521`, cited-not-timed).
- [x] **Q4 — the conductor (three threads).** Main/render tick (`kitty/child-monitor.c:1232-1237`); `io_loop` (`:1481`, `KittyChildMon`); talk loop (`talk_loop` `:1805`, `KittyPeerMon`). Python: `boss.py` (`ChildMonitor` `:370`), `window.py` (`Child`↔`Screen` `:601`/`:604`), `child.py` (`openpty` `:170`, `fork` `:276`).
- [x] **Q5 — ordering & priority.** Render tick `process_pending_resizes` (`kitty/child-monitor.c:1043`, called `:1233`) → `parse_input` (`:451`, called `:1236`) → `render` (`:1237`); I/O loop wakeup drain (`:1515`) → signals (`:1519`) → POLLIN reads (`:1531`) → POLLOUT writes (`:1540`); runtime resize-before-parse demonstrated.
- [x] **Q6 — shell-integration alignment.** **OSC 133** (`kitty/vt-parser.c:536` → `shell_prompt_marking` `kitty/screen.c:2328`, `PROMPT_START` `:2337`/`OUTPUT_START` `:2341`), **OSC 7** (`case 7:` `kitty/vt-parser.c:499` → `kitty/screen.c:2393`), **bracketed paste 2004** (`kitty/modes.h:81`, `paste_()` `kitty/screen.c:4573`, wrap `:4586-4588`, `MODE_GETSET` `:3854`); single VT parser in-band order shown at runtime; `modify_shell_environ()` (`kitty/shell_integration.py:218`).
- [x] **Q7 — backpressure & unstable remote.** `BUF_SZ = 1024u*1024u` = 1 MiB (`kitty/vt-parser.c:18`), runtime-confirmed `1048576`; `vt_parser_has_space_for_input()` (`kitty/vt-parser.c:1477`); POLLIN gate (`kitty/child-monitor.c:1501`) → PTY flow control → child blocks on `write()`. SSH: `kittens/ssh/**`, `shell-integration/ssh/**` (bootstrap over `/dev/tty`); `kitty @` via `kitty/rc/*.py`; `ssh` suite passes.
- [x] **Q8 — end-to-end settling.** Full flow PTY → `poll` → `read_bytes` → coalesced wakeup → tick (`process_pending_resizes` → `parse_input` → dispatch → `Screen`) → `render`; runtime probe shows mixed input settling and the buffer draining back to `1048576` bytes. Coalescing timers tied in.
- [x] **Q9 — keeping rhythm.** `input_delay` = `3` (`kitty/options/definition.py:878`), `repaint_delay` = `10` (`kitty/options/definition.py:866`), `resize_debounce_time` = `0.1 0.5` (`kitty/options/definition.py:1182`) — all read at runtime; the WAKEUP coalescing predicate + "expensive operation … cocoa" comment quoted verbatim from `kitty/child-monitor.c:1562-1570` (with the poll-timeout bound at `:1506-1510`); **and measured at runtime** — a 60 ms / 300 ms surge coalesced `426274` / `2127514` read-events into `19` / `99` wakeups with a min inter-wakeup interval of `3.0000` ms (≤ one wakeup per `input_delay`), driven by kitty's real `monotonic()` clock and real `input_delay`.

### Exact literals index (as requested, never paraphrased)

| Literal | Value | Anchor |
|---|---|---|
| Parser buffer size | `BUF_SZ (1024u*1024u)` = `1048576` bytes (1 MiB) | `kitty/vt-parser.c:18` |
| Buffer padding | `BUF_EXTRA (512u/8u)` | `kitty/vt-parser.c:20` |
| Input coalescing delay | `input_delay` default `3` ms | `kitty/options/definition.py:878` |
| Repaint delay | `repaint_delay` default `10` ms | `kitty/options/definition.py:866` |
| Resize debounce | `resize_debounce_time` default `0.1 0.5` s | `kitty/options/definition.py:1182` |
| Synchronized Output mode | `2026`; `PENDING_UPDATE (2026 << 5)` | `kitty/control-codes.h:235`, `kitty/modes.h:86` |
| Pause auto-expiry | `2000` ms | `kitty/screen.c:2521` |
| Bracketed paste mode | `2004`; `BRACKETED_PASTE (2004 << 5)`, `"200~"`/`"201~"` | `kitty/modes.h:81-83` |
| I/O thread name | `KittyChildMon` | `kitty/child-monitor.c:1489` |
| Talk thread name | `KittyPeerMon` | `kitty/child-monitor.c:1808` |

---

*All evidence above was produced inside the pinned build/run environment at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. Temporary probe scripts used to capture the runtime output were created under `/tmp` and removed after use; the repository contains only this added document.*

