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

### 0.2 Build

Kitty is a hybrid C / Python / Go application built by its own `setup.py`, not by a package manifest. The canonical build in this environment:

```bash
source /tmp/kitty-venv/bin/activate
cd <repo>
python3 setup.py
```

> **Honesty note on the build command.** The agent plan named `python3 setup.py develop`. In this environment that variant fails before doing any work:
>
> ```
> $ python3 setup.py develop
> KeyError: 'DEVELOP_ROOT'
>   File ".../setup.py", line 1255, in build_launcher
> ```
> The `develop` branch of `build_launcher` reads `os.environ["DEVELOP_ROOT"]` [setup.py:1255], a variable set only by kitty's bundled-Python `dev.sh` flow, which is absent here. The plain `python3 setup.py` build (the `make all` target) produces the identical launcher binary and is the canonical build used throughout. Its tail output:
> ```
> Package wayland-protocols was not found ... Disabling building of wayland backend
> [1/2] Compiling kitty/launcher/main.c ...
> [2/2] Compiling kitty/launcher/single-instance.c ...
> [1/1] Linking launcher ... done
> ```
> (Wayland is auto-disabled → x11-only, which is the canonical CI configuration on Ubuntu 25.10.) Strict flags `-pedantic-errors -Werror -Wall -Wextra` remained on.

The build produces (all git-ignored): `kitty/launcher/kitty` (the launcher), `kitty/launcher/kitten`, and `kitty/fast_data_types.so`. The launcher self-reports:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### 0.3 Canonical run form (headless)

A real GPU/display is unavailable in this container, so kitty runs under a virtual X display (`xvfb`). Mesa's `llvmpipe` provides software GL. This is still the **real** emulator driving a **real** PTY — only the display surface is virtual. The invocation form used for every experiment:

```bash
export TMPDIR=/tmp/kitty-clean-tmp LANG=C.UTF-8 LC_ALL=C.UTF-8
xvfb-run -a -s "-screen 0 1280x800x24" \
  ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes <FLAG> sh -c '<child program>'
```

`--config NONE` guarantees the **canonical default option values** (`input_delay=3`, `repaint_delay=10`, `resize_debounce_time=(0.1, 0.5)` — confirmed in §Q5); `close_on_child_death=yes` makes kitty exit when the child finishes. A harmless container-only line, `Failed to open systemd user bus`, is filtered from captures with `grep`.

### 0.4 Observation instruments (kitty's own, user-facing flags)

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
2. **All parsing and all screen mutation happen on one thread, in byte order.** The I/O thread only *reads*; the main thread *parses and mutates* [kitty/child-monitor.c:1236, kitty/vt-parser.c:1417]. This single-writer design is the root cause of Q3's coherence.
3. **Every tick runs a fixed sequence: resizes → parse → render** [kitty/child-monitor.c:1233-1237]. This ordering is the literal answer to Q2's "what goes first."

The following sections answer each question with captured evidence. Temporary child scripts lived under `/tmp/q1` (outside the repository) and were removed afterward.

---

## Q1 — Where does raw input first enter the system, and what happens on pause → resume?

### 1.1 Direct answer

Raw bytes cross the boundary in **`read_bytes()`** [kitty/child-monitor.c:1337], which performs **`read(fd, buf, available_buffer_space)`** [kitty/child-monitor.c:1345] **directly into the VT parser's shared 1 MiB write buffer**. `buf` is not a private scratch buffer — it is a pointer *into* the parser's own storage, returned by `vt_parser_create_write_buffer()` [kitty/vt-parser.c:1451] and finalized by `vt_parser_commit_write()` [kitty/vt-parser.c:1465, called at kitty/child-monitor.c:1354]. This is a **zero-copy** read: the kernel writes child bytes straight into the buffer the parser will consume.

"**Paused then resumed**" is the terminal's **synchronized-update** feature. It is reached two ways that **converge on the same function** `screen_pause_rendering()` [kitty/screen.c:2506]:

- **Modern — DEC private mode 2026** (`CSI ? 2026 h` to pause / begin, `CSI ? 2026 l` to resume / end). Setting or resetting the mode routes through `case PENDING_MODE << 5:` → `screen_pause_rendering(self, val, 0)` [kitty/screen.c:1174-1175]. The mode bit is `PENDING_UPDATE (2026 << 5)` [kitty/modes.h:86].
- **Legacy — DCS pending sequences** (`DCS =1s ST` to pause, `DCS =2s ST` to resume). Handled at `case '=':` [kitty/vt-parser.c:636]; `=1s` → `screen_start_pending_mode` → `screen_pause_rendering(self->screen, true, 0)` [kitty/vt-parser.c:639-640]; `=2s` → `screen_stop_pending_mode` → `screen_pause_rendering(self->screen, false, 0)` [kitty/vt-parser.c:644-645].

`screen_pause_rendering()` does three things on **pause**: refuses if already paused (`if (self->paused_rendering.expires_at) return false;` [kitty/screen.c:2518]); arms a **2000 ms default timeout** (`if (for_in_ms <= 0) for_in_ms = 2000;` [kitty/screen.c:2521]; `expires_at = monotonic() + ms_to_monotonic_t(for_in_ms)` [kitty/screen.c:2522]); and snapshots the visible grid, cursor, colors, selections, and graphics so the *display* is frozen while the *live grid keeps mutating*. On **resume** it refuses if not paused (`if (!self->paused_rendering.expires_at) return false;` [kitty/screen.c:2508]), clears `expires_at` [kitty/screen.c:2509], and sets `self->is_dirty = true` [kitty/screen.c:2511] so the accumulated live grid is flushed to the GPU **atomically**. A safety net, `screen_check_pause_rendering()` [kitty/screen.c:2489], force-resumes when `now > expires_at` [kitty/screen.c:2490] so a stalled application can never freeze the display forever.

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
$ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE as B; print(B)'
1048576
```

`1048576 = 1024 × 1024`, i.e. the 1 MiB `BUF_SZ`.

**Where the PTY fd comes from (Q1 boundary):** the master fd that `read_bytes()` reads is created by `os.openpty()` in `openpty()` [kitty/child.py:170]; the child is spawned by `Child.fork` [kitty/child.py:276]; kitty keeps the master end as `self.child_fd = master` [kitty/child.py:338] and makes it non-blocking with `os.set_blocking(child_fd, False)` [kitty/child.py:345] (the child's slave end stays blocking — the basis for OS flow control in Q4).

### 1.3 Raw bytes at the boundary (observed)

To show the *exact* bytes crossing the boundary, a child emits a synchronized-update wrapper around the text `HELLO`, and `--dump-bytes` captures what kitty read:

```bash
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes --dump-bytes /tmp/q1/q1_bytes.bin \
    sh -c "printf '\033[?2026hHELLO\033[?2026l'"
$ od -A d -t x1z /tmp/q1/q1_bytes.bin
```

Unedited output:

```
0000000  1b 5b 3f 32 30 32 36 68 48 45 4c 4c 4f 1b 5b 3f  >.[?2026hHELLO.[?<
0000016  32 30 32 36 6c                                   >2026l<
0000021
```

Decoded: `1b 5b 3f 32 30 32 36 68` = `ESC [ ? 2 0 2 6 h` (the BSU / pause), then `48 45 4c 4c 4f` = `HELLO`, then `1b 5b 3f 32 30 32 36 6c` = `ESC [ ? 2 0 2 6 l` (the ESU / resume). The precise pause/resume bytes demonstrably crossed the boundary via the real PTY. (`--dump-bytes` is emitted at kitty/vt-parser.c:1399 from the same buffer the parser consumes, so the hexdump faithfully reflects the boundary bytes.)

### 1.4 Before / during / after — the crux of pause → resume

The interesting behavior is *transitional*, so the value is reported **before**, **during**, and **after** the pause.

**Experiment Q1a — a large surge wrapped in DEC 2026.** The child sets mode 2026, writes 50 lines, then resets it:

```bash
$ cat /tmp/q1/child_dec2026.sh
printf '\033[?2026h'                      # BSU — pause the display
for i in $(seq 1 50); do printf 'SURGE line %02d\r\n' "$i"; done
printf '\033[?2026l'                      # ESU — resume, atomic flush
printf 'AFTER-ESU visible line\r\n'
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes --dump-commands sh /tmp/q1/child_dec2026.sh
```

Unedited output (abridged **only** in the count of identical `draw`/CR/LF triples, which are shown in full for the first and last line; nothing in the pause/resume logic is elided):

```
screen_set_mode 2026 1
draw SURGE line 01
screen_carriage_return
screen_linefeed
draw SURGE line 02
screen_carriage_return
screen_linefeed
        ... (SURGE line 03 … SURGE line 49, each: draw / screen_carriage_return / screen_linefeed) ...
draw SURGE line 50
screen_carriage_return
screen_linefeed
screen_reset_mode 2026 1
draw AFTER-ESU visible line
screen_carriage_return
screen_linefeed
```

- **BEFORE:** the display renders live (not paused; `expires_at == 0`).
- **DURING:** `screen_set_mode 2026 1` (DECSET `CSI?2026h`) calls `screen_pause_rendering(self, true, 0)` [kitty/screen.c:1175], which sets `expires_at` [kitty/screen.c:2522] and snapshots the grid. **All 50 `draw` commands are still parsed and mutate the live grid, in byte order, while the display shows the frozen snapshot** — the stream proves the parser never stopped.
- **AFTER:** `screen_reset_mode 2026 1` (DECRST `CSI?2026l`) calls `screen_pause_rendering(self, false, 0)`, which sets `is_dirty = true` [kitty/screen.c:2511] so the accumulated 50-line grid is flushed **atomically** in one render; `AFTER-ESU visible line` then draws normally.

**Cause → effect:** DECSET 2026 *causes* `expires_at` to be set and a grid snapshot to be taken, which *causes* the render path to keep displaying the snapshot; the parser keeps mutating the real grid; DECRST 2026 *causes* `is_dirty=true`, which *causes* the next `render()` to upload the whole accumulated grid at once — the viewer never sees a half-drawn screen.

**Experiment Q1c — the legacy DCS path converges on the same function.** Same shape, using `DCS =1s ST` / `DCS =2s ST`:

```bash
$ cat /tmp/q1/child_dcs.sh
printf '\033P=1s\033\\'                    # legacy pause
printf 'DCS line 1\r\n'; printf 'DCS line 2\r\n'
printf '\033P=2s\033\\'                    # legacy resume
printf 'after-dcs\r\n'
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes --dump-commands sh /tmp/q1/child_dcs.sh
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

**Experiment Q1d — bracketing the 2000 ms boundary, 2 runs each.** A child pauses, sleeps, then resumes; if the sleep exceeds 2000 ms the timeout force-resumes *before* the resume arrives, so the late resume finds `expires_at == 0` and is refused (which the parser reports as an error). Sleeps of 1.5 s and 2.5 s were each run twice:

```bash
# child: printf '\033P=1s\033\\'; sleep <T>; printf '\033P=2s\033\\'
$ ... --dump-commands sh -c 'printf "\033P=1s\033\\"; sleep 1.5; printf "\033P=2s\033\\"'   # ×2
$ ... --dump-commands sh -c 'printf "\033P=1s\033\\"; sleep 2.5; printf "\033P=2s\033\\"'   # ×2
```

Observed (stable across both runs of each):

| Sleep before resume | Run 1 | Run 2 | Interpretation |
|---|---|---|---|
| **1.5 s** (< 2000 ms) | no error | no error | pause still active at resume → clean resume |
| **2.5 s** (> 2000 ms) | `[2.672] Pending mode stop command issued while not in pending mode ... caused by a timeout` | `[2.668] Pending mode stop command issued while not in pending mode ... caused by a timeout` | timeout force-resumed at ~2.0 s; the 2.5 s resume was refused |

The error text is the DCS `=2s` refusal report [kitty/vt-parser.c:645-648], fired because `screen_pause_rendering(false)` returned `false` at [kitty/screen.c:2508]. The boundary lands cleanly between 1.5 s and 2.5 s, matching the coded default of 2000 ms [kitty/screen.c:2521] — **stable across two runs each**.

**The other refusal path — double pause.** Pausing while already paused is refused at [kitty/screen.c:2518]:

```bash
$ ... --dump-commands sh -c 'printf "\033P=1s\033\\"; printf "\033P=1s\033\\"'
```
```
[0.176] Pending mode start requested while already in pending mode. This is most likely an application error.
```
That is the DCS `=1s` refusal report [kitty/vt-parser.c:640-641], fired because the second pause found `expires_at` already set. *(The DEC-2026 path reaches the analogous refusal through [kitty/screen.c:1174-1176], which emits `log_error` rather than a parser report; both share the same `screen_pause_rendering` return-`false` mechanism.)*

### 1.6 The input side of the boundary, and bracketed paste

**Keystroke ingress (observed).** `--debug-input` shows the canonical windowing-layer (glfw) event path that feeds keystrokes toward the child:

```bash
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes --debug-input sh -c 'sleep 0.2'
```
```
[0.060] Loading new XKB keymaps:
[0.065] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
on_focus_change: window id: 0x1 focused: 1
```

> **Honesty label.** No X-event-injection tool exists in this container (`xdotool`, `xte`, `wtype`, `ydotool`, and Python-Xlib are all absent), so a *physical* keypress driving the `glfw → keys.c → key_encoding.c` encoding stage could not be synthesized. The glfw ingress above is observed; the exact key→escape *encoding* is therefore **code-derived (inferred, not observed)**.

**The write transport to the child (observed via a real PTY).** Sending text through kitty's remote control exercises the write path `write_to_child` [kitty/window.py:955] → `schedule_write_to_child` [kitty/child-monitor.c:372] → `write_to_child` [kitty/child-monitor.c:1443] → PTY master → child slave:

```bash
# kitty launched with -o allow_remote_control=yes --listen-on unix:/tmp/q1/rc.sock, child echoes stdin
$ kitty @ --to unix:/tmp/q1/rc.sock send-text 'hello-from-write-path\n'
```
Dumped stream:
```
draw hello-from-write-path
draw CHILD-READ:[hello-from-write-path]
```
The first `draw` is the PTY echo of the injected bytes; the second is the child reading them back from stdin — proving the bytes traversed a real PTY. *(Label: `send-text` exercises the write transport; it does not exercise the glfw key-encoding stage.)*

**Bracketed paste (mode 2004).** The paste path is wrapped by `BRACKETED_PASTE (2004 << 5)` [kitty/modes.h:81 — corrected from the plan's `:80`], with `BRACKETED_PASTE_START "200~"` [kitty/modes.h:82] and `BRACKETED_PASTE_END "201~"` [kitty/modes.h:83]. A child toggling the mode is observed as:

```bash
$ ... --dump-commands sh -c "printf '\033[?2004h'; printf 'x'; printf '\033[?2004l'"
```
```
screen_set_mode 2004 1
draw x
screen_reset_mode 2004 1
```

Paste is orchestrated on the Python side by `paste_with_actions` [kitty/window.py:1643] → `paste_bytes` [kitty/window.py:1707] / `paste_text` [kitty/window.py:1713]; `in_bracketed_paste_mode` is a `MODE_GETSET` at [kitty/screen.c:3854]; the C paste helper is `paste_()` [kitty/screen.c:4573].

---

## Q2 — The "unseen conductor": timing, ordering, and state hand-offs — *what decides which event gets handled first?*

### 2.1 Direct answer

There is an **"unseen conductor"** and it is remarkably literal: kitty uses a **three-thread model** with a strict division of labor, and the main thread runs a **fixed per-tick sequence**. *What decides which event gets handled first* is not a heuristic or a priority queue — it is **hard-coded order** in `process_global_state()` [kitty/child-monitor.c:1224]: **resizes, then parse, then render**, deterministically, on every tick.

The three threads:

- **I/O thread — `io_loop()`** [kitty/child-monitor.c:1481] (named `KittyChildMon` at :1489): a `poll(2)` loop that *only reads* child output into the shared buffer. It never touches the screen.
- **Main thread — `main_loop()`** [kitty/child-monitor.c:1259] → `run_main_loop(process_global_state, self)` [kitty/child-monitor.c:1262]: runs *all* VT parsing and *all* screen mutation, then renders.
- **Talk thread — `talk_loop()`** [kitty/child-monitor.c:1805] (named `KittyPeerMon` at :1808): services remote-control peers over a Unix socket, off the main thread (Q4).

### 2.2 The three threads, observed at runtime

Thread names are set with `set_thread_name(...)`, so the live process reveals the model directly (read-only inspection of `/proc/<pid>/task/*/comm`):

```bash
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o allow_remote_control=yes --listen-on unix:/tmp/q1/rc.sock \
    -o close_on_child_death=yes sh -c 'sleep 5' &
$ pid=$(pgrep -f 'launcher/kitty --config NONE' | head -1)
$ cut -d' ' -f1 /proc/$pid/task/*/comm | sort | uniq -c | sort -rn
```

Observed (Run 1, PID 78820) — and confirmed present again in Run 2 (PID 79716):

```
     33 kitty
     32 llvmpipe-0        (…llvmpipe-1 … llvmpipe-31: Mesa software-GL rasterizer workers)
      1 KittyChildMon
      1 KittyPeerMon
```

`KittyChildMon` (the I/O thread) and `KittyPeerMon` (the talk thread) are present in both runs. The 32 `llvmpipe-N` threads and the many `kitty` worker threads are **Mesa's software-GL rasterizer pool** — an artifact of headless software rendering, *not* part of kitty's concurrency model. Kitty's own pipeline is **main + I/O (`KittyChildMon`) + talk (`KittyPeerMon`)**. The talk thread appears only because `--listen-on` brought up a listen socket.

The threads are created in the monitor: `pthread_create(..., io_loop, ...)` and the talk thread at [kitty/child-monitor.c:291 / :256 / :286].

### 2.3 The fixed per-tick order — the literal "who goes first"

```c
// kitty/child-monitor.c:1224
process_global_state(void *data) {
    ...
    monotonic_t now = monotonic();                       // :1231
    if (global_state.has_pending_resizes) {
        process_pending_resizes(now);                    // :1233  ← 1) RESIZES first
        input_read = true;
    }
    if (parse_input(self)) input_read = true;            // :1236  ← 2) PARSE second
    render(now, input_read);                             // :1237  ← 3) RENDER third
```

**Every tick, unconditionally in this order: `process_pending_resizes` → `parse_input` → `render`.** That is the answer to "what decides which event gets handled first": pending resizes are absorbed before parsing, parsing (and thus screen mutation) completes before rendering, and rendering always sees a fully-parsed, self-consistent grid.

> **Honesty label — why there is no per-tick log line.** The natural per-tick trace, `EVDBG("Processing global state")` [kitty/child-monitor.c:1225] and `render`'s `EVDBG("input_read: %d ...")` [kitty/child-monitor.c:872], is gated on the compile-time macro `#ifdef DEBUG_EVENT_LOOP` [kitty/child-monitor.c:29-33], which is **not defined in the canonical build**. So these lines are compiled *out* and never appear at runtime. The order is therefore a **structural code certainty**, not a runtime log; its *observable consequences* are Q1's atomic parse-before-render pause/resume and Q3's in-order coherence, both captured above and below.

### 2.4 Poll-level ordering inside the I/O thread

Within the I/O thread, the poll fd array is deliberately ordered so control fds precede child fds:

- Setup: `children_fds[0].fd = wakeup_read_fd`, `children_fds[1].fd = signal_read_fd` [kitty/child-monitor.c:183], both `POLLIN` [:184]; `EXTRA_FDS = 2` [:35]; child fds begin at index `EXTRA_FDS`.
- After `poll()` returns, the processing order is: **(1)** the wakeup fd `children_fds[0]` → `drain_fd` [kitty/child-monitor.c:1515]; **(2)** the signal fd `children_fds[1]` → `read_signals` → `handle_signal` [kitty/child-monitor.c:1516-1519]; **(3)** the child fds loop `children_fds[EXTRA_FDS + i]` → `read_bytes` [kitty/child-monitor.c:1530-1533].

So *wakeup, then signals, then child output*. Signals are handled via `KITTY_HANDLED_SIGNALS = SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2` [kitty/child-monitor.c:121] in `handle_signal` [kitty/child-monitor.c:1362]; `SIGCHLD` [:1370] drives child reaping. **Observed consequence of the signal path in every run:** when the child exits, `SIGCHLD` → reap → kitty exits under `close_on_child_death=yes`:

```bash
$ xvfb-run -a ... --debug-rendering sh -c 'printf hi; sleep 0.2' ; echo "kitty exit: $?"
```
```
[0.164] Child launched
...
kitty exit: 0
```

> Note: `SIGWINCH` is **not** in `KITTY_HANDLED_SIGNALS` — kitty, *as the terminal*, generates resizes from glfw and **sends** `SIGWINCH` to the child (Q5); it does not receive `SIGWINCH` itself.

### 2.5 Wakeup coalescing — when the main thread wakes

The I/O thread does not wake the main thread on every read. The `WAKEUP` macro fires only after `input_delay` has elapsed since the last wakeup:

```c
// kitty/child-monitor.c:1562
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
        // we only wakeup the main loop after input_delay as wakeup is an expensive operation
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

They stay aligned because **all parsing and all screen mutation happen on one thread, in exact byte order**, and prompt attribution is stored **per line**. An `OSC 133` marker and the ordinary text around it are processed in precisely the order the child emitted them: `parse_input` [kitty/child-monitor.c:451] (called from the fixed tick at [kitty/child-monitor.c:1236]) runs `run_worker` [kitty/vt-parser.c:1417], which parses *and* mutates the screen — while the I/O thread only ever *reads* bytes into the buffer. Because a marker is written to `line_attrs[cursor->y].prompt_kind` [kitty/screen.c:2337] for **whatever line the cursor is on at that instant**, and the cursor position is *itself* a product of the same in-order parse, the hint **cannot drift** relative to the text. This is the direct mechanism behind the user's phrase *"without drifting out of sync."*

### 3.2 Where the hints come from, and how they are handled

Shell integration is injected by `modify_shell_environ()` [kitty/shell_integration.py:218], which sets `KITTY_SHELL_INTEGRATION` in the child's environment; the per-shell scripts then emit the OSC 133 markers (`shell-integration/bash/kitty.bash`, `zsh/kitty-integration`, `fish/vendor_conf.d/kitty-shell-integration.fish`). Marker meanings (public FTCS / OSC 133 convention — external framing only): `A` = prompt start, `B` = prompt end / command start, `C` = command output start, `D;<exit>` = command finished, `k=s` = secondary/continuation prompt.

On the parser side, `OSC 133` is dispatched in `dispatch_osc` [kitty/vt-parser.c:457], `case 133` [kitty/vt-parser.c:536], which calls `shell_prompt_marking(self->screen, buf+i)` [kitty/vt-parser.c:544]. In the screen, per-line attribution is keyed on `cursor->y`:

```c
// kitty/screen.c:2327
shell_prompt_marking(Screen *self, char *buf) {
    if (self->cursor->y < self->lines) {
        char ch = buf[0];
        switch (ch) {
            case 'A': {
                PromptKind pk = PROMPT_START;                                   // :2333
                ...
                parse_prompt_mark(self, buf+1, &pk);                            // :2336
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = pk;    // :2337  ← per-line tag
                if (pk == PROMPT_START) CALLBACK("cmd_output_marking", "O", Py_False);  // :2338
            } break;
            case 'C': {
                self->linebuf->line_attrs[self->cursor->y].prompt_kind = OUTPUT_START;  // :2341
                ...
```

`parse_prompt_mark` [kitty/screen.c:2316] decodes sub-tokens; `k=s` sets `SECONDARY_PROMPT` [kitty/screen.c:2321].

### 3.3 Observed: markers interleaved in byte order with ordinary text

A **real interactive bash** with shell integration active (default) is driven with remote-control `send-text`, captured under `--dump-commands`:

```bash
# kitty launched: -o allow_remote_control=yes --listen-on unix:/tmp/q1/rc.sock --dump-commands  bash -i
$ kitty @ --to unix:/tmp/q1/rc.sock send-text 'echo hello-world\n'
```

Unedited stream (the interesting commands, in byte order — no logic elided):

```
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
draw prompt$
draw echo hello-world
shell_prompt_marking 133 C;cmdline=echo\ hello-world
draw hello-world
```

The markers land exactly where bash emitted them relative to the text: `D;0` (previous command finished, exit 0) → `A` (new prompt starts) → the prompt draws → the typed command draws → `C;cmdline=...` (output begins) → the command's output draws. (kitty also emits its own `OSC 133 k;...` sub-region markers — `k;start_kitty`, `k;end_kitty` — around prompt regions; these are byte-interleaved too.)

### 3.4 Observed (definitive): per-line attribution does not drift

The strongest proof that a marker attaches to the *correct line* is to ask kitty which lines it considers "last command output." Run a command whose output is three known lines, then query by prompt-kind extent:

```bash
$ kitty @ --to unix:/tmp/q1/rc.sock send-text 'printf "OUT-LINE-1\nOUT-LINE-2\nOUT-LINE-3\n"\n'
$ kitty @ --to unix:/tmp/q1/rc.sock get-text --extent=last_cmd_output
```
Unedited output:
```
OUT-LINE-1
OUT-LINE-2
OUT-LINE-3
```

For contrast, `--extent=screen` showed the prompt+command on lines 1–3 and the output on lines 5–7; `--extent=last_cmd_output` isolated **exactly** the output (lines 5–7) and **nothing** from the prompt/command lines:

```bash
$ kitty @ --to unix:/tmp/q1/rc.sock get-text --extent=screen
```
```
prompt$ printf "OUT-LINE-1\nOUT-LINE-2\nOUT-LINE-3\n"

OUT-LINE-1
OUT-LINE-2
OUT-LINE-3
```

**Cause → effect:** the `OSC 133 C` marker set `OUTPUT_START` on the exact line where output began [kitty/screen.c:2341], and `D` closed it; the `last_cmd_output` extent is computed purely from those per-line `prompt_kind` tags — so isolating exactly the three output lines proves the marker did **not** drift onto the prompt or command lines. Had parsing been concurrent or reordered, the boundary line would be wrong; it is not.

### 3.5 Observed: the secondary (continuation) prompt

Entering a line-continuation forces bash to emit its PS2 prompt, which shell integration marks with `A;k=s`:

```bash
# send: echo one\   <enter>   two   <enter>   (a PS2 continuation)
$ kitty @ --to unix:/tmp/q1/rc.sock send-text 'echo one\\\ntwo\n'
```
```
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 A;k=s
shell_prompt_marking 133 C;cmdline=echo\ onetwo
draw one two
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
```

`133 A;k=s` is decoded by `parse_prompt_mark` into `SECONDARY_PROMPT` [kitty/screen.c:2321] — the secondary prompt is attributed distinctly from the primary one, again on the correct line.

### 3.6 The cause → effect, stated plainly

Parsing and mutation are single-writer and byte-ordered: `run_worker` [kitty/vt-parser.c:1417] on the main thread consumes the buffer strictly in order, and the I/O thread (`KittyChildMon`) only fills the buffer — it never mutates the screen. Therefore an OSC 133 marker attaches to `line_attrs[cursor->y]` for whatever line the in-order parse has reached [kitty/screen.c:2337]; since `cursor->y` is itself advanced by that same in-order parse (each `screen_linefeed` in the stream), the hint and the text share one clock and **cannot drift out of sync**.

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

`strace` is unavailable (no `ptrace` in this container), so backpressure is quantified by **timing each individual `write()`** from the child. Since the child's PTY slave is blocking [kitty/child.py:171], a `write()` that blocks on flow control shows up as a slow write. The producer writes fixed 4096-byte chunks and records the duration of each `os.write`:

```python
# /tmp/q1/producer.py  (child program — lived outside the repo, removed afterward)
import os, sys, time
total = int(sys.argv[1]) * 1024 * 1024      # MiB to write
chunk = b'x' * 4096
n = total // 4096
slow = 0; mx = 0.0; t0 = time.monotonic()
for _ in range(n):
    a = time.monotonic(); os.write(1, chunk); d = time.monotonic() - a
    mx = max(mx, d)
    if d > 0.001: slow += 1
dt = time.monotonic() - t0
sys.stderr.write("MB=%d dt=%.4f max_write=%.6f slow=%d MBps=%.1f\n"
                 % (total/1e6, dt, mx, slow, (total/1e6)/dt))
```

**BASELINE — no terminal (write to `/dev/null`):**
```bash
$ python3 /tmp/q1/producer.py 16 > /dev/null
MB=16 dt=0.0028 max_write=0.000003 slow=0 MBps=5989.0
$ python3 /tmp/q1/producer.py 64 > /dev/null
MB=64 dt=0.0146 max_write=0.000049 slow=0 MBps=4607.0
```
No backpressure: 4600–6000 MB/s, **0** slow writes, max single write ≈ 3 µs.

**UNDER KITTY — real PTY, 16 MiB, two runs:**
```bash
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes python3 /tmp/q1/producer.py 16
# run 1
MB=16 dt=0.1451 max_write=0.009281 slow=15 MBps=115.6
# run 2
MB=16 dt=0.1432 max_write=0.009292 slow=15 MBps=117.1
```

**UNDER KITTY — 64 MiB (larger scale), two runs:**
```bash
$ xvfb-run -a ... ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes python3 /tmp/q1/producer.py 64
# run 1
MB=64 dt=0.5827 max_write=0.009173 slow=65 MBps=115.2
# run 2
MB=64 dt=0.5892 max_write=0.009569 slow=63 MBps=113.9
```

**Interpretation, before / during / after (at write granularity):**
- **BEFORE / AFTER (buffer has space):** the vast majority of writes complete in microseconds — kitty is reading, `has_space_for_input()` is true, POLLIN is requested.
- **DURING (buffer full):** a small number of writes block for ~9.3 ms — these are the moments the 1 MiB buffer is full, POLLIN is masked [kitty/child-monitor.c:1501], the PTY fills, and the child's `write()` blocks until the main thread drains via `run_worker` [kitty/vt-parser.c:1425].

**Magnitude & stability.** Throughput drops ~**51×** (≈115 MB/s through the terminal vs. ≈5000–6000 MB/s to `/dev/null`), and the max single `write()` jumps from ~3 µs to ~9.3 ms — **stable across two runs at both 16 MiB and 64 MiB**.

**Quantitative fingerprint of the 1 MiB buffer.** The count of slow (blocking) writes is **≈ 1 per MiB written**: 15 for 16 MiB, and 63–65 for 64 MiB. That rate is exactly what a **1 MiB** flow-control window predicts (`16 MiB / 1 MiB ≈ 16`, `64 MiB / 1 MiB ≈ 64`). A pure PTY kernel buffer (typically 8–64 KiB) would force blocking ~16× more often; the observed ~1/MiB rate confirms that **kitty's 1 MiB `BUF_SZ` [kitty/vt-parser.c:18] is the governing flow-control window**, not the small kernel PTY buffer.

### 4.4 The unstable remote connection

**Remote-control transport (observed).** Remote peers are serviced by the talk thread `talk_loop()` [kitty/child-monitor.c:1805], off the main thread: `read_from_peer` [kitty/child-monitor.c:1714] and `dispatch_peer_command` [kitty/child-monitor.c:1699] hand work to the main thread, where `peer_message_received` [kitty/boss.py:776] parses the remote-control DCS envelope (prefix `\x1bP@kitty-cmd`, terminator `\x1b\\`). A round-trip was observed:

```bash
$ kitty @ --to unix:/tmp/q1/rc4.sock ls
```
```json
[{"id": 1, "platform_window_id": ..., "is_focused": true, "tabs": [{"windows": [
   {"id": 1, "pid": 84404, "cmdline": ["sh", "-c", "sleep 5"], ...}
]}]}]
```
The query was answered by the talk thread and marshaled to the main thread — not blocking the parse/render pipeline.

**SSH kitten (observed).** The SSH kitten is the remote path that deploys terminfo + shell integration to remote hosts:

```bash
$ ./kitty/launcher/kitty +kitten ssh --help | head
```
> a thin wrapper around the `ssh` command; it automatically enables shell integration on the remote host, re-uses existing connections to reduce latency, and makes the kitty terminfo database available on the remote host.

Its sources are `kittens/ssh/{main.py, main.go, config.go, askpass.go, utils.go}`, and `kitty/remote_control.py` is the command surface.

**Does an unstable link behave differently?** The remote-control payload is itself a **DCS escape sequence**, so once it reaches the terminal it traverses the *same* VT read path → 1 MiB buffer → backpressure gate as ordinary child output. Link instability therefore manifests as the **same flow control** already measured in §4.3. The concrete, *observed* slow-link safety behavior is the pause timeout from Q1: a synchronized-update pause whose resume is delayed past 2000 ms (exactly what a stalled/unstable link would cause) is **force-resumed** by `screen_check_pause_rendering()` [kitty/screen.c:2490], so the display never freezes waiting for a hung remote.

> **Honesty label.** A genuine network-fault-injected SSH link could not be reproduced headlessly (no reachable SSH server or fault-injection tooling in this container). The peer loop, the DCS transport, and the flow-control mechanism are **observed**; the specific *degradation profile* of a faulty network link is **inferred (code-derived)** from those observed pieces plus the observed 2000 ms force-resume.

---

## Q5 — End-to-end: from the moment mixed input arrives to the moment the interface settles — *how the moving parts keep their rhythm*

### 5.1 Direct answer

Three pacing timers give the pipeline its rhythm and let the interface settle:

| Timer | Default | Line | Role |
|-------|---------|------|------|
| `input_delay` | **3 ms** | [kitty/options/definition.py:878] | Coalesces reads/wakeups — batches many small child writes into one parse+render tick. |
| `repaint_delay` | **10 ms** | [kitty/options/definition.py:866] | Throttles repaints so the screen redraws at most ~every 10 ms under sustained output. |
| `resize_debounce_time` | **(0.1, 0.5) s** | [kitty/options/definition.py:1182] | Debounces resizes — 0.1 s settle during a live drag, 0.5 s hard cap. |

The defaults are confirmed at runtime from the built binary (not read from source alone):

```bash
$ ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; \
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
- **`resize_debounce_time` ((0.1, 0.5) s) — debouncing.** `process_pending_resizes()` [kitty/child-monitor.c:1043] applies pending resizes only while a live resize is in progress, using `.on_pause = 0.1 s` [kitty/child-monitor.c:1055] to settle during a drag pause and `.on_end = 0.5 s` [kitty/child-monitor.c:1066] as the cap. A committed resize calls `resize_pty` [kitty/child-monitor.c:592] → `pty_resize` [kitty/child-monitor.c:577], and the Python side calls `boss.child_monitor.resize_pty(...)` [kitty/window.py:863] and sends `SIGWINCH` to the child [kitty/window.py:873].

### 5.3 Observed: the resize → SIGWINCH hand-off

`--debug-rendering` surfaces the resize hand-off because [kitty/window.py:873] prints `SIGWINCH sent to child`:

```bash
# kitty launched with --debug-rendering; two remote-control resizes issued 0.4 s apart
$ kitty @ --to unix:/tmp/q1/rc.sock resize-os-window --width 100 --height 30
$ kitty @ --to unix:/tmp/q1/rc.sock resize-os-window --width 90  --height 24
```
Observed:
```
[0.165] Child launched
[1.212] SIGWINCH sent to child in window: 1 with size: (30, 100, 900, 540)
[1.597] SIGWINCH sent to child in window: 1 with size: (24, 90, 810, 432)
```
The size tuple is `(lines, cols, width_px, height_px)`. Each discrete resize produces exactly one `SIGWINCH` via `resize_pty`, confirming kitty *sends* the resize signal to the child rather than receiving it.

> **Honesty label.** The live-drag *debounce coalescing* (many intermediate sizes collapsing within the 0.1–0.5 s window) is not reproducible under bare `xvfb`, which has no window-manager resize-event stream. The timer values and the `process_pending_resizes` mechanism are observed/code-grounded; the intermediate coalescing itself is **inferred (code-derived)**.

### 5.4 Observed: the end-to-end settling trace

To watch *the moving parts keep their rhythm*, a single child emits a realistic mix — ordinary text, an OSC 133 prompt marker, a prompt, an OSC 133 output marker, a synchronized-update block, then a trailing line — and `--dump-commands` records the whole settle in byte order:

```bash
$ cat /tmp/q1/child_e2e.sh
printf 'ordinary text line 1\r\n'
printf '\033]133;A\033\\'; printf 'prompt$ '
printf '\033]133;C\033\\'
printf '\033[?2026h'                         # BSU
printf 'sync line 1\r\n'; printf 'sync line 2\r\n'
printf '\033[?2026l'                         # ESU — atomic flush
printf '\033]133;D;0\033\\'
printf 'settled tail line\r\n'
$ xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --config NONE \
    -o close_on_child_death=yes --dump-commands sh /tmp/q1/child_e2e.sh
```

Unedited stream — **identical across two runs (18 lines, deterministic)**:

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

**Reading the rhythm (cause → effect), from arrival to settle:**
1. Bytes arrive and are read into the shared buffer by the I/O thread; the `WAKEUP` gate coalesces them over `input_delay` (3 ms) [kitty/child-monitor.c:1566] so the whole mix is handed to the main thread together.
2. The main thread runs one tick: `process_pending_resizes` (nothing pending) → `parse_input` → `render` [kitty/child-monitor.c:1233-1237]. `parse_input` consumes the buffer **in byte order**, so ordinary text, the OSC 133 `A`/`C` markers, and the `2026` block all land in the exact order emitted.
3. The `2026` block pauses the *display* while the two `sync line` draws mutate the *live grid*; `screen_reset_mode 2026 1` flips `is_dirty=true` [kitty/screen.c:2511] so `render()` flushes the whole block atomically.
4. `render()` throttles to the `repaint_delay` (10 ms) cadence when idle [kitty/child-monitor.c:875] but is bypassed while input is pending — so the burst renders promptly and the interface **settles** on `settled tail line` with no half-drawn frame.

**Determinism across runs.** The 18-line stream was byte-identical on both runs, so no distribution needed to be reported for the *ordering*. For the *magnitude/timing* values (§4.3 throughput, §Q1 2000 ms boundary), stability across ≥ 2 runs was likewise confirmed.

> **Observability honesty note.** Per-tick and per-frame *counts* are not directly instrumentable in the canonical build — `EVDBG` is `#ifdef DEBUG_EVENT_LOOP` (compiled out; §Q2) and there is no runtime frame counter. The `input_delay` coalescing cadence is visible *indirectly* in the Q4 backpressure profile (a steady ~115 MB/s drain with ~9.3 ms max blocking waits ≈ a few `input_delay` cycles). The timer values (observed), the mechanisms (cited), the resize hand-off (observed), and the end-to-end settle (observed) are all solid; fine-grained coalescing/frame counts are labeled **code-derived** where they cannot be counted directly.

---

## Appendix A — External conventions (framing only; observed code is the source of truth)

These public conventions frame the narrative; every behavioral claim above rests on observed kitty output, not on these:

- **Synchronized output / DEC private mode 2026.** `CSI ? 2026 h` begins a synchronized update (BSU — batch output), `CSI ? 2026 l` ends it (ESU — apply atomically), so the viewer never sees a half-drawn screen. There is no cross-terminal consensus on a timeout, which is why kitty enforces its own (2000 ms, observed in §Q1). Kitty implements this via `PENDING_UPDATE (2026 << 5)` [kitty/modes.h:86].
- **OSC 133 prompt marking (FTCS).** `A` = prompt start, `B` = prompt end / command start, `C` = command output start, `D;<exit>` = command finished, `k=s` = secondary prompt. Kitty stores these per line as `prompt_kind` [kitty/screen.c:2337, :2341].
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
- [x] **Before / during / after** — *observed*: live → snapshot+`expires_at` while 50 lines still parse → atomic flush on resume (§1.4).
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
- [x] **Per-line attribution** `shell_prompt_marking` [kitty/screen.c:2327], `PROMPT_START` [:2333], `line_attrs[cursor->y].prompt_kind` [:2337], `OUTPUT_START` [:2341], `parse_prompt_mark` [:2316], `SECONDARY_PROMPT` [:2321] — *observed* callbacks + `get-text --extent=last_cmd_output` isolating exactly the output lines (§3.3, §3.4).
- [x] **Hint origin** `modify_shell_environ` [kitty/shell_integration.py:218]; bash/zsh/fish scripts — *code* (§3.2).
- [x] **A / B / C / D / k=s** markers — *observed*, including secondary prompt `A;k=s` (§3.3, §3.5).

### Q4 — Backpressure / unstable remote
- [x] **Space gate** `vt_parser_has_space_for_input` [kitty/vt-parser.c:1477] (`read.sz + write.pending < BUF_SZ` :1481) — *code* (§4.2).
- [x] **POLLIN masking** `events = has_space ? POLLIN : 0` [kitty/child-monitor.c:1501]; POLLOUT [:1503]; `write_to_child` [:1443] — *code* (§4.2).
- [x] **`run_worker` drain gate** [kitty/vt-parser.c:1425] — *code* (§4.1).
- [x] **Observed backpressure** ~51× throughput drop (≈115 MB/s vs ≈5000 MB/s), max write 3 µs→9.3 ms, **stable ×2** at 16 & 64 MiB; **~1 blocking write per MiB** = 1 MiB buffer fingerprint (§4.3).
- [x] **Before/during/after** buffer occupancy — *observed* at write granularity (§4.3).
- [x] **Remote path** `talk_loop` [kitty/child-monitor.c:1805], `read_from_peer` [:1714], `dispatch_peer_command` [:1699], `peer_message_received` [kitty/boss.py:776] — *observed* `kitty @ ls` round-trip (§4.4).
- [x] **SSH kitten** `kittens/ssh/`; `remote_control.py` surface — *observed* `+kitten ssh --help` (§4.4).
- [x] **Unstable link** — *observed* mechanism (DCS over same VT path; 2000 ms force-resume [kitty/screen.c:2490]); network-fault degradation labeled **inferred** (§4.4).

### Q5 — Settling & rhythm
- [x] **`input_delay` 3 ms** [kitty/options/definition.py:878] — *observed* default from binary; gate [kitty/child-monitor.c:1566], [kitty/vt-parser.c:1425] (§5.1, §5.2).
- [x] **`repaint_delay` 10 ms** [kitty/options/definition.py:866] — *observed* default; throttle `render` [kitty/child-monitor.c:875], bypassed on pending input (§5.1, §5.2).
- [x] **`resize_debounce_time` (0.1, 0.5) s** [kitty/options/definition.py:1182] — *observed* default; `process_pending_resizes` [kitty/child-monitor.c:1043], on_pause [:1055], on_end [:1066], `resize_pty` [:592], `pty_resize` [:577], `window.py` [:863/:873] (§5.2).
- [x] **Resize→SIGWINCH hand-off** — *observed* `SIGWINCH sent to child … size (lines,cols,w,h)` (§5.3); live-drag coalescing labeled **inferred**.
- [x] **End-to-end settle** (text + OSC 133 + 2026 block) — *observed*, byte-identical **×2** (§5.4).

### Method & honesty
- [x] **Canonical PTY path only** — every primary capture is a real child through a real PTY; `test_parse_written_data` [kitty/screen.c:4771-4772] and `parse_bytes` [kitty_tests/parser.py:20] were **not** used as evidence (§0.1).
- [x] **Actual output beside every claim; file:line for every structural claim** — throughout.
- [x] **Timing rigor** — `input_delay` 3 ms, `repaint_delay` 10 ms, `resize_debounce_time` (0.1, 0.5) s, buffer 1 MiB, pause timeout 2000 ms; scale stated, stability confirmed across ≥ 2 runs (§Q1, §Q4, §Q5).
- [x] **Corrected citations used** — `BRACKETED_PASTE` at [kitty/modes.h:81] (not :80); `test_parse_written_data` at [kitty/screen.c:4771-4772]; `add_child` at [kitty/boss.py:585]; `BUF_EXTRA` at [kitty/vt-parser.c:20] (not :19).
- [x] **Inferred labels** — key-encoding stage, live-drag resize coalescing, and network-fault SSH degradation are the only code-derived (not directly observed) items, each labeled inline.
- [x] **Build reconciled honestly** — `python3 setup.py develop` fails (`KeyError: 'DEVELOP_ROOT'` [setup.py:1255]); canonical `python3 setup.py` build used, producing `kitty/launcher/kitty` (kitty 0.35.2) (§0.2).

*All temporary observation scripts lived under `/tmp/q1` (outside the repository) and were removed; the only change to the destination repository is this document.*
