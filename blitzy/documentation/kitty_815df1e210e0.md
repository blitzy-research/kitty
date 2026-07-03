# How kitty handles keyboard input during normal interactive use

**Repository:** `kovidgoyal/kitty` · **Branch:** `kitty_815df1e210e0` · **Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

This document answers, empirically, what happens inside the kitty terminal emulator when a user
presses **a few simple keys in the default shell**, under kitty's **default (unconfigured)**
configuration. The question has three parts, which form the spine of this document:

- **(a)** which parts of the system **receive the input first**;
- **(b)** which parts perform the **intermediate processing**;
- **(c)** how the **updated display is ultimately produced**.

## Methodology — RUN FIRST, then write

Every behavioral claim below was produced by **actually building and running kitty and capturing its
own diagnostic output** — *not* by reading source alone. kitty ships first-class observability, so
**no source file was modified**; the pipeline was observed through kitty's documented debug/tracing
flags. The investigation drove keystrokes through the **genuine** GLFW → `on_key_input` path
(`kitty/keys.c:166`), the same code path used during ordinary typing — **not** through a bypassing
interface (remote control `kitten @ send-text`, or the Python `send_key`/`write_to_child` path at
`kitty/window.py:917,955`, which are labeled **(non-canonical)** where they appear).

Conventions used throughout:
- Verbatim captured output is shown in fenced code blocks. Where a line contains ANSI SGR color
  codes, they are shown as `\x1b[..m` (they appear as literal escape bytes in the raw capture).
- Statements confirmed by captured runtime output are presented with that exact line next to them
  (**one claim / one evidence**).
- Statements not directly observed at runtime are prefixed **(inferred)**.
- All `file:line` citations are pinned to commit `815df1e210e0`.
- **Stability:** the full observation was repeated across **2 runs** (`run1`, `run2`). Every
  characteristic line reproduced byte-for-byte except the leading monotonic `[seconds]` timestamp.
  Both-run values are noted where relevant.

The environment is the designated container image
(`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, run as `kitty-qna:local`). kitty is
a GPU-accelerated GUI app, so it was given an offscreen X display (`Xvfb :99 -screen 0 1280x800x24`)
and a PTY-backed default shell (`bash`). Keystrokes were injected with `xdotool key` after
`xdotool windowfocus --sync` (the bare Xvfb server has no window manager).

---

## Section 1 — Exact build & invocation commands (verbatim) + observed versions

### 1.1 Observed toolchain versions

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ python3 --version
Python 3.12.3
$ go version
go version go1.23.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ pkg-config --version
1.8.1
```

These satisfy kitty's stated requirements: Python `>= 3.8` (`pyproject.toml:2`
`requires-python = ">=3.8"`), a C11 compiler (`setup.py:492` uses `-std=c11`), and Go `1.22`
(`go.mod:3` `go 1.22`; `setup.py:1148` runs `cmd = [go, 'build', '-v']`). Native libraries are per
`docs/build.rst:84+` (`harfbuzz >= 2.2.0`, etc.).

### 1.2 Build command (default configuration)

The canonical, default build is `python3 setup.py build --verbose` (default `action: str = 'build'`
per `setup.py:175`). It completed with **exit code 0**. Verbatim head of the build log:

```
$ cd /app && python3 setup.py build --verbose
CC: ['gcc'] (13, 0)
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Copyright (C) 2023 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Detected: CompilerType.gcc
Updating Go generated files...
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten /app/tools/cmd
```

The build produces the native launcher `./kitty/launcher/kitty` (source `kitty/launcher/main.c`,
which embeds CPython and bootstraps `kitty/main.py`). Confirming it runs:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**Cause → effect:** the launcher (`kitty/launcher/main.c`) embeds CPython → runs `kitty/main.py` →
constructs the `Boss` singleton (`kitty/boss.py`) which owns the child monitor and windows. This is
the process that later receives key events and drives rendering.

### 1.3 Invocation command actually run (default config + all debug/tracing flags)

```
DISPLAY=:99 XDG_RUNTIME_DIR=/tmp/xdg-0 \
./kitty/launcher/kitty --config NONE --debug-keyboard --dump-commands --debug-rendering \
    --dump-bytes /tmp/kitty_dump_bytes_run1.bin  bash  \
    > /tmp/kitty_stdout_run1.log  2> /tmp/kitty_stderr_run1.log
```

- `--config NONE` forces kitty's **default configuration** (the `--config`/`-c` option is
  `kitty/cli.py:870`; it accepts `NONE`), so no user `kitty.conf` is applied.
- `bash` is the default shell child; launching it provisions the **PTY** (see Section 3).
- **Stream routing (verified by reading the emitters and confirmed by capture):**
  `--debug-keyboard` output goes to **STDERR** — its macro chain ends in `timed_debug_print`, which is
  `fprintf(stderr, …)`/`vfprintf(stderr, …)` (`kitty/monotonic.h:99`). `--dump-commands` output goes
  to **STDOUT** via `safe_print` → `print(...)` (`kitty/utils.py:125`). The `--debug-rendering`
  "GL version string" line goes to **STDOUT** via a direct `printf` (`kitty/gl.c:72`). Both streams
  were captured separately.

After the window appeared, the keys `a`, `Space`, `Return`, and `b` were typed through the real
OS key-event path. Each key thus reached the genuine per-key C callback `on_key_input(GLFWkeyevent *ev)`
(`kitty/keys.c:166`).

---

## Section 2 — (a) Which components receive the input first

Order of arrival: **vendored GLFW platform/XKB backend** (`glfw/` + `kitty/glfw.c`) → the real C
callback **`on_key_input`** (`kitty/keys.c:166`). Two startup lines confirm the surfaces that must
exist before a key can be received — the OS window and the child shell.

### 2.1 The GUI window and PTY child are created (startup preconditions)

The GLFW layer creates the OS window; `kitty/glfw.c:1321` emits `debug("OS Window created\n")`:

```
[0.197] OS Window created
```
(run2: `[0.140] OS Window created`) — **producing anchor:** `kitty/glfw.c:1321`.

The default shell child is launched over a PTY; `kitty/window.py:871` prints it:

```
[0.210] Child launched
```
(run2: `[0.152] Child launched`) — **producing anchor:** `kitty/window.py:871`.

Focus is delivered to the window (a precondition for it to receive key events):

```
[0.211] \x1b[35mon_focus_change\x1b[m: window id: 0x1 focused: 1
```
— **producing anchor:** `kitty/glfw.c:517` (`window_focus_callback` emits `debug_input("\x1b[35mon_focus_change\x1b[m: window id: 0x%llu focused: %d\n", …)`; the format reproduced identically across both runs).

### 2.2 The GLFW/XKB backend receives the raw OS key first

Before kitty's own key handler runs, the **vendored GLFW X11/XKB backend** translates the raw OS
keycode. Under `--debug-keyboard`, `glfw/xkb_glfw.c:875`
(`debug("%s xkb_keycode: 0x%x ", … "\x1b[31mPress\x1b[m", xkb_keycode)`) prints the received key.
For the pressed `a`:

```
[2.989] \x1b[31mPress\x1b[m xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
```
(run2: `[2.995] … xkb_keycode: 0x26 … glfw_key: 97 (a) xkb_key: 97 (a)` — identical) —
**producing anchor:** `glfw/xkb_glfw.c:875`. This is the earliest, platform-level receipt: the
backend has resolved the X11 keycode `0x26` to the GLFW key `97` (`a`).

**Cause → effect:** the debug hint that turns this on is set at GLFW init from the CLI flag —
`kitty/glfw.c:1444` `glfwInitHint(GLFW_DEBUG_KEYBOARD, debug_keyboard);` and
`kitty/glfw.c:1446` `OPT(debug_keyboard) = debug_keyboard != 0;`, which are fed by
`kitty/main.py:514` `init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)`.

### 2.3 kitty's real per-key entry point: `on_key_input`

The GLFW backend then delivers the event to the genuine C entry point
`on_key_input(GLFWkeyevent *ev)` (`kitty/keys.c:166`). Guarded by
`if (OPT(debug_keyboard)) {` (`kitty/keys.c:172`), the receipt is printed with the format at
`kitty/keys.c:176`
(`on_key_input: glfw key: 0x%x native_code: 0x%x action: %s %stext: '%s' state: %d`). For `a`:

```
[2.989] \x1b[33mon_key_input\x1b[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
```
(run2: `[2.995] … glfw key: 0x61 … text: 'a' state: 0 sent key as text to child: a` — identical) —
**producing anchors:** `kitty/keys.c:166` (entry), `kitty/keys.c:172` (guard), `kitty/keys.c:176`
(receipt format). The `glfw key: 0x61` is `97` decimal — the same key the backend reported in §2.2 —
confirming this is the real received `a`.

> **Observed detail (reported as-is):** the receipt text and the dispatch text (`sent key as
> text to child: a`, Section 3) appear **on one physical line**. This is expected: the receipt
> format (`kitty/keys.c:176`) ends with a space and **no** newline, and `timed_debug_print` only
> emits the leading `[seconds]` timestamp immediately after a newline (`kitty/monotonic.h:101,106`).
> So the next `debug(...)` continues the same line until a `\n` is printed.

- **IME sibling variant (inferred):** when a key produces IME text with no key code, the same guard
  block instead prints `on_IME_input: text: %s` (`kitty/keys.c:174`). This branch was not triggered
  by plain ASCII typing, so it is labeled **(inferred)** from reading.
- **Print-macro chain (inferred from reading):** in `keys.c`, `debug(...)` is
  `#define debug debug_input` (`kitty/keys.h:16`) → `#define debug_input(...) if (OPT(debug_keyboard))
  { timed_debug_print(__VA_ARGS__); }` (`kitty/state.h:15`); the flag is `bool debug_keyboard;`
  (`kitty/state.h:80`). The observed lines above confirm the net effect of this chain.

---

## Section 3 — (b) Intermediate processing

Once `on_key_input` has the event, kitty performs four sub-stages: **(i) encode**, **(ii) dispatch to
the child PTY**, **(iii) read the child's echoed bytes and parse them**, and **(iv) update the screen
model**. Each is evidenced below.

### 3.1 (i) Encode — legacy text vs. Kitty Keyboard Protocol

`on_key_input` encodes the event by calling, at `kitty/keys.c:251`:

```
int size = encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key);
```

`encode_glfw_key_event` is defined at `kitty/key_encoding.c:414`. It selects between **legacy**
encoding and the **Kitty Keyboard Protocol** (the CSI-u form). Reading the encoder: the CSI-u trailer
is `char csi_trailer = 'u';` (`kitty/key_encoding.c:150`), legacy mode is chosen when
`bool legacy_mode = !ev->report_all_event_types && !ev->disambiguate;` (`kitty/key_encoding.c:152`),
and the bytes are assembled by `serialize(const EncodingData *data, char *output, const char
csi_trailer)` (`kitty/key_encoding.c:65`). The Python shortcut/keymap layer that runs before raw
encoding is `kitty/keys.py` (`get_shortcut` at `kitty/keys.py:40`, `class Mappings` at
`kitty/keys.py:62`).

The return value drives **two sibling branches** at the call site — **both were observed**:

- **Plain-text branch:** the function returns the sentinel `SEND_TEXT_TO_CHILD`
  (`#define SEND_TEXT_TO_CHILD INT_MIN` at `kitty/keys.h:15`). Observed for `a`, Space, and `b` (their
  `text:` field is non-empty and the dispatch says "as text" — see §3.2).
- **Encoded-escape branch:** the function returns a positive `size`. Observed for `Return`, whose
  `text:` field is empty (`text: ''`) and which was dispatched "as encoded key" — see §3.2.

### 3.2 (ii) Dispatch to the child PTY — both branches

**Plain-text branch** — `kitty/keys.c:253` `schedule_write_to_child(w->id, 1, text, strlen(text));`,
logged by `kitty/keys.c:254` `debug("sent key as text to child: %s\n", text);`. For the letter `a`,
the Space, and the letter `b` (three data points; all reproduced in run2):

```
[2.989] \x1b[33mon_key_input\x1b[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[3.601] \x1b[33mon_key_input\x1b[m: glfw key: 0x20 native_code: 0x20 action: PRESS mods: none text: ' ' state: 0 sent key as text to child:  
[4.833] \x1b[33mon_key_input\x1b[m: glfw key: 0x62 native_code: 0x62 action: PRESS mods: none text: 'b' state: 0 sent key as text to child: b
```

**producing anchors:** `kitty/keys.c:253` (write), `kitty/keys.c:254` (log). (The Space line ends
with a literal space after the colon — the space character that was sent. Each line's leading
`[seconds]` timestamp is the only field that varies between runs.)

**Encoded-escape branch** — `kitty/keys.c:259` `schedule_write_to_child(w->id, 1, encoded_key, size);`,
logged under `if (OPT(debug_keyboard)) {` (`kitty/keys.c:260`) with the prefix `sent encoded key to
child: ` (`kitty/keys.c:261`). Each byte is rendered: ESC as `^[ ` (`kitty/keys.c:263`), Space as
`SPC ` (`kitty/keys.c:264`), other printables as `%c ` (`kitty/keys.c:265`), and remaining bytes as
`0x%x ` (`kitty/keys.c:266`). Pressing **Return** produced:

```
[4.217] \x1b[33mon_key_input\x1b[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
```
(run2: `[4.223] … sent encoded key to child: 0xd` — identical) — **producing anchors:**
`kitty/keys.c:259,260,261,266`.

> **Observed value, explained:** in the **default** configuration with a plain `bash` child, Return
> encodes to a **single carriage-return byte `0x0d`**, printed here as `0xd` because CR is neither
> ESC nor Space nor `isprint()`, so it takes the `else { debug("0x%x ", …); }` branch
> (`kitty/keys.c:266`). No `^[ ` (ESC) appears, because the Kitty Keyboard Protocol progressive
> enhancement is not active for a bare shell — this is the plain legacy encoding. Reported exactly
> as observed.

The dispatch target is the child's PTY master, created earlier: `openpty()` (`kitty/child.py:170`,
which calls `os.openpty()` at `kitty/child.py:171`) and `fork(self)` (`kitty/child.py:276`). The
`Child launched` line in §2.1 (`kitty/window.py:871`) is the observed confirmation this child exists.

### 3.3 (iii) Child echoes → child monitor reads → VT parser classifies bytes

The shell echoes the bytes it received back over the PTY. The child-monitor I/O thread reads that
output and hands it to the VT parser. Because `--dump-commands` (equivalently `--dump-bytes`) is
active, the dump path is selected at `kitty/child-monitor.c:178-181`:

```
if (dump_callback != Py_None) {
    self->dump_callback = dump_callback; Py_INCREF(dump_callback);
    self->parse_func = parse_worker_dump;
} else self->parse_func = parse_worker;
```

and parse state is carried in `ParseData pd = {.dump_callback = self->dump_callback, .now = now};`
(`kitty/child-monitor.c:439`). The VT state machine (`kitty/vt-parser.c`) classifies each byte and,
for each parsed command, calls the Python `dump_callback`. Specifically, printable characters go
through `REPORT_DRAW` (`kitty/vt-parser.c:92`), which for `rd_ch >= ' '` invokes
`PyObject_CallFunction(self->dump_callback, "KsC", self->window_id, "draw", rd_ch)`
(`kitty/vt-parser.c:105`); the control bytes `CR` and `LF` are reported as `screen_carriage_return`
(`kitty/vt-parser.c:102`) and `screen_linefeed` (`kitty/vt-parser.c:101`).

The Python sink is `class DumpCommands` (`kitty/boss.py:232`), whose `__call__` (`kitty/boss.py:239`)
buffers consecutive draws and flushes them with `safe_print('draw', ''.join(self.draw_dump_buf))`
(`kitty/boss.py:249`), and prints other commands with `safe_print(what, *a)` (`kitty/boss.py:252`).
It is constructed at `kitty/boss.py:372`
(`DumpCommands(args) if args.dump_commands or args.dump_bytes else None`). `safe_print` is
`print(...)` (`kitty/utils.py:125`) → **STDOUT**.

Captured `--dump-commands` STDOUT for the echoed keystrokes (identical in both runs):

```
draw a 
screen_carriage_return
screen_linefeed
```

- `draw a ` — the echoed printable characters (`a` and the following Space) drawn — **producing
  anchors:** `kitty/vt-parser.c:92,105` → `kitty/boss.py:239,249` → `kitty/utils.py:125`.
- `screen_carriage_return` — the CR from Return — **producing anchor:** `kitty/vt-parser.c:102`.
- `screen_linefeed` — the LF the shell emitted alongside CR — **producing anchor:**
  `kitty/vt-parser.c:101`.

**Raw-bytes cross-check (`--dump-bytes`):** the raw pre-parse bytes received from the child were
written to `/tmp/kitty_dump_bytes_run{1,2}.bin`. Both dumps measure **741 bytes**, stable across
the two runs, as measured with `wc -c`:

```
$ wc -c /tmp/kitty_dump_bytes_run1.bin /tmp/kitty_dump_bytes_run2.bin
 741 /tmp/kitty_dump_bytes_run1.bin
 741 /tmp/kitty_dump_bytes_run2.bin
1482 total
```

An `od -c` view of that same dump shows the raw PTY stream — the shell-integration OSC sequences
(the `\x1b]7;kitty-shell-cwd://4d14d0b4de52/app` cwd report and the `\x1b]133;…` prompt markers)
and the prompt text `root@4d14d0b4de52:/app#` — as received *before* the VT parser classifies it,
confirming the parser's input is the child's echoed output:

```
$ od -c /tmp/kitty_dump_bytes_run1.bin | sed -n '11,23p'
0000240 033   \ 033   ]   7   ;   k   i   t   t   y   -   s   h   e   l
0000260   l   -   c   w   d   :   /   /   4   d   1   4   d   0   b   4
0000300   d   e   5   2   /   a   p   p  \a 033   [   ?   2   0   0   4
0000320   h 033   ]   1   3   3   ;   k   ;   s   t   a   r   t   _   k
0000340   i   t   t   y  \a 033   ]   1   3   3   ;   D   ;   0  \a 033
0000360   ]   1   3   3   ;   A  \a 033   ]   1   3   3   ;   k   ;   e
0000400   n   d   _   k   i   t   t   y  \a 033   ]   0   ;   r   o   o
0000420   t   @   4   d   1   4   d   0   b   4   d   e   5   2   :    
0000440   /   a   p   p  \a   r   o   o   t   @   4   d   1   4   d   0
0000460   b   4   d   e   5   2   :   /   a   p   p   #     033   ]   1
0000500   3   3   ;   k   ;   s   t   a   r   t   _   s   u   f   f   i
0000520   x   _   k   i   t   t   y  \a 033   [   5       q 033   ]   2
0000540   ;   /   a   p   p  \a 033   ]   1   3   3   ;   k   ;   e   n
```

### 3.4 (iv) Screen-model update

Parsed printable text updates the screen model via the **runtime** entry
`screen_draw_text(Screen *self, const uint32_t *chars, size_t num_chars)` (`kitty/screen.c:866`),
which calls `screen_on_input(self)` (`kitty/screen.c:867`) then `draw_text(self, chars, num_chars)`
(`kitty/screen.c:868`). The parser invokes it at `kitty/vt-parser.c:226`
(`screen_draw_text(self->screen, &ch, 1)`) and `kitty/vt-parser.c:236`.

> **Note (test-only, do not confuse):** `kitty/screen.c:4772` `test_parse_written_data(...)` is a
> **test helper**, *not* the live runtime screen-update path.

**(inferred)** The observed `draw a` dump line is emitted from the same `REPORT_DRAW` site
(`kitty/vt-parser.c:92,105`) that sits alongside the `screen_draw_text` call (`kitty/vt-parser.c:226`)
in the parser; the dump callback is the observable proxy for the model update. The direct
model-mutation call is thus labeled **(inferred)** from reading, corroborated by the observed
`draw a` output.

### 3.5 Why processing hands off to a *scheduled* render (the (b) → (c) bridge)

**(inferred)** `kitty/child-monitor.c` runs a **three-thread** model: a Main thread (input parsing +
render scheduling), an I/O thread (PTY poll/read/write + child reaping), and a Talk thread
(peer/remote-control sockets). Because input handling is separated from rendering, the intermediate
stage does not draw inline; instead it updates the screen model and lets the Main thread schedule a
render on its own cadence. This separation is the mechanism behind the transition to Section 4. It is
labeled **(inferred)** as it is drawn from reading the child-monitor structure rather than a single
captured line.

---

## Section 4 — (c) How the updated display is produced

After the model updates, the display is produced by: **render scheduling off the child monitor** →
**upload cell data to the GPU + shader pipeline** → **OpenGL draw** → **GLFW frame swap**.

### 4.1 Render scheduling off the child monitor

**(inferred, from reading)** The Main thread schedules a render via
`prepare_to_render_os_window(OSWindow *os_window, …)` (`kitty/child-monitor.c:705`), which calls
`send_cell_data_to_gpu(TD.vao_idx, TD.xstart, TD.ystart, TD.dx, TD.dy, TD.screen, os_window)`
(`kitty/child-monitor.c:714`). Note `send_cell_data_to_gpu` is **defined in** `kitty/shaders.c:970`
(the child monitor only *calls* it); `shaders.c` uploads the cell data to the GPU and drives the
shader pipeline. These are labeled **(inferred)** because they are read from source rather than
printed as a per-call log line.

### 4.2 OpenGL layer — the observed `--debug-rendering` evidence

The OpenGL layer is wrapped by `kitty/gl.c`. Under `--debug-rendering`/`--debug-gl`, GL calls are
checked for errors instead of ignored: the error checker is
`check_for_gl_error(...)` (`kitty/gl.c:16`); it is installed via `gladSetGLPostCallback(check_for_gl_error);`
(`kitty/gl.c:62`) guarded by `if (!global_state.debug_rendering) { … }` (`kitty/gl.c:59`), and the GL
version is printed at `kitty/gl.c:72`
(`if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", …)`). The corresponding
macro is `debug_rendering(...)` (`kitty/state.h:14`). This is the **observed** display evidence — the
GL context was created and driven for our window:

```
[0.131] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```
(run2: `[0.119] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected
version: 4.5` — identical text) — **producing anchor:** `kitty/gl.c:72`. It confirms kitty is
rendering through a real OpenGL 4.5 core-profile context (here the Mesa software rasterizer, because
the offscreen display uses `llvmpipe`).

### 4.3 Frame swap presents the display

**(inferred, from reading)** The rendered frame is presented by
`swap_window_buffers(OSWindow *os_window)` (`kitty/glfw.c:1802`), which performs
`if (glfwAreSwapsAllowed(os_window->handle)) glfwSwapBuffers(os_window->handle);`
(`kitty/glfw.c:1803`) — the GLFW frame swap that puts the updated cells on screen (the initial swap
is at `kitty/glfw.c:1221`). No per-frame swap log line is emitted, so this final step is labeled
**(inferred)** from these citations; the observed `--debug-rendering` GL activity in §4.2 is the
runtime evidence that the display path executed.

---

## Section 5 — Every debug/tracing flag used (named-item enumeration)

All flags come from the `# Debugging options` block (`kitty/cli.py:965`). This enumerates **every**
flag in that block relevant to tracing the pipeline, plus the removed-flag note.

| Flag | Documented effect | `file:line` | Observed here? |
|------|-------------------|-------------|----------------|
| `--debug-input` / `--debug-keyboard` | "Print out key and mouse events as they are received." (`dest=debug_keyboard`, `type=bool-set`) | `kitty/cli.py:996`, dest `kitty/cli.py:997` | **Yes** — §2.2, §2.3, §3.2 (STDERR) |
| `--dump-commands` | "Output commands received from child process to STDOUT." (`type=bool-set`) | `kitty/cli.py:972` | **Yes** — §3.3 (`draw a`, `screen_carriage_return`, `screen_linefeed`) |
| `--dump-bytes` | "Path to file in which to store the raw bytes received from the child process." | `kitty/cli.py:985` | **Yes** — §3.3 (741-byte capture) |
| `--debug-rendering` / `--debug-gl` | "Debug rendering commands. This will cause all OpenGL calls to check for errors instead of ignoring them. Also prints out miscellaneous debug information." (`type=bool-set`) | `kitty/cli.py:989` | **Yes** — §4.2 (`GL version string`) |
| `--replay-commands` | Replays a prior `--dump-commands` dump (e.g. `kitty sh -c "kitty --replay-commands /path/to/dump; read"`). | `kitty/cli.py:977` | Not needed for live typing (documented for completeness) |
| `--debug-config` | **Removed** — **not present** in `kitty/cli.py` at this commit. | (absent) | Deliberately **not used**; a `grep` for it returns nothing (compatibility constraint). |

Supporting flag-plumbing citations: `--config`/`-c` (`kitty/cli.py:870`) accepts `NONE`;
`kitty/main.py:514` forwards the debug flags into `init_glfw`; `kitty/glfw.c:1444-1446` sets the GLFW
init hints and stores `OPT(debug_keyboard)`; `kitty/boss.py:1580` (`if self.args.debug_keyboard:`) is
the Boss-level branch.

---

## Section 6 — Coverage pass (each named item, with evidence + anchor)

- [x] **(a) Receive first** — real `on_key_input:` line captured for `a` (`glfw key: 0x61 … text:
  'a'`) → `kitty/keys.c:166,172,176`; earliest platform receipt `Press xkb_keycode: 0x26 … glfw_key:
  97 (a)` → `glfw/xkb_glfw.c:875`; GLFW hint plumbing `kitty/glfw.c:1444`; window/child preconditions
  `kitty/glfw.c:1321`, `kitty/window.py:871`.
- [x] **(b) Encode — both branches** — plain-text `SEND_TEXT_TO_CHILD` (`kitty/keys.h:15`) for
  `a`/Space/`b`, and encoded positive-`size` for Return → `kitty/keys.c:251` +
  `kitty/key_encoding.c:414,150,152,65`.
- [x] **(b) Dispatch to PTY — both branches** — `sent key as text to child: a` (and Space, `b`) →
  `kitty/keys.c:253,254`; `sent encoded key to child: 0xd` for Return → `kitty/keys.c:259,260,261,266`;
  PTY via `kitty/child.py:170,171,276`.
- [x] **(b) Child echo + parse** — `--dump-commands` STDOUT `draw a`, `screen_carriage_return`,
  `screen_linefeed` → `kitty/child-monitor.c:178-181,439`; `kitty/vt-parser.c:92,101,102,105`;
  `kitty/boss.py:232,239,249,252,372`; `kitty/utils.py:125`. Raw bytes via `--dump-bytes` (741 B).
- [x] **(b) Screen model** — runtime update `screen_draw_text` (`kitty/screen.c:866,867,868`) driven
  by `kitty/vt-parser.c:226,236`; `kitty/screen.c:4772` noted as **test-only**. Model-mutation linkage
  labeled **(inferred)**.
- [x] **(c) Display produced** — observed `GL version string: '4.5 (Core Profile) Mesa …'` →
  `kitty/gl.c:72` (with `--debug-rendering`); render scheduling `kitty/child-monitor.c:705,714`;
  cell upload `kitty/shaders.c:970`; GL error hook `kitty/gl.c:16,59,62`; frame swap
  `kitty/glfw.c:1802,1803` **(inferred)**.
- [x] **Every debug/tracing flag enumerated** (Section 5): `--debug-input`/`--debug-keyboard`,
  `--dump-commands`, `--dump-bytes`, `--debug-rendering`/`--debug-gl`, `--replay-commands`, plus the
  removed `--debug-config` note → `kitty/cli.py:965-1005`.
- [x] **Exact build & invocation commands** stated verbatim (Section 1) → `setup.py:175`,
  `kitty/launcher/main.c`, `pyproject.toml:2`, `go.mod:3`, `docs/build.rst:84`.
- [x] **Stability** — the observation was repeated across **2 runs**; every characteristic line
  reproduced except the monotonic `[seconds]` prefix.
- [x] **Non-canonical contrast** labeled — the Python `send_key`/`write_to_child` and remote-control
  paths (`kitty/window.py:917,955`) are **(non-canonical)** and were **not** used for evidence; only
  the genuine `on_key_input` path (`kitty/keys.c:166`) was exercised.
- [x] **Inferred vs observed** — every non-observed statement is prefixed **(inferred)**; observed
  values are quoted verbatim next to their claim.

### End-to-end summary (cause → effect)

A key press in the default shell is first resolved by the **vendored GLFW/XKB backend**
(`glfw/xkb_glfw.c:875`) and delivered to kitty's real callback **`on_key_input`** (`kitty/keys.c:166`);
kitty **encodes** it (`kitty/keys.c:251` → `kitty/key_encoding.c:414`) — as plain text
(`SEND_TEXT_TO_CHILD`) for `a`/Space/`b` or as a single CR byte for Return — and **writes it to the
child PTY** (`kitty/keys.c:253`/`259`; `kitty/child.py:170,276`). The shell **echoes** the bytes; the
**child monitor** reads them (`kitty/child-monitor.c:178-181`) and the **VT parser**
(`kitty/vt-parser.c`) classifies them, which `--dump-commands` surfaces as `draw a`,
`screen_carriage_return`, `screen_linefeed`; these **update the screen model**
(`kitty/screen.c:866`). Finally the Main thread **schedules a render**
(`kitty/child-monitor.c:705,714` → `kitty/shaders.c:970`), the **OpenGL** pipeline runs
(`kitty/gl.c`, observed `GL version string` at `kitty/gl.c:72`), and a **GLFW frame swap**
(`kitty/glfw.c:1802,1803`) presents the updated display.
