# How Kitty Handles Keyboard Input During Normal Use — A Runtime-Observed Walkthrough

> **Scope of this document.** This answers the question: *"If I start Kitty from this
> repository with whatever debugging or tracing options are available and press a few
> simple keys inside the default shell, what does the observable runtime behavior reveal
> about how input moves through Kitty's core components before the screen updates?"* —
> specifically **(1) which parts receive the input first, (2) which parts handle the
> intermediate processing, and (3) how the updated display is produced.**
>
> **Everything below was produced by actually building Kitty and running it with its
> own built-in tracing flags, pressing keys through the real windowing path, and
> capturing the output.** Each behavioral claim is placed next to the raw captured
> evidence that produced it and a concrete `file:line` reference. Where a link in the
> chain is not directly printed by the running program, it is explicitly labeled
> **(inferred from reading)** and is bracketed by runtime observations on both sides.

---

## 1. Direct answer (the three stages at a glance)

When you press a key in Kitty's default shell, the observable runtime behavior shows the
input travelling through three consistently-appearing stages:

1. **Receipt (who sees it first).** The physical key is delivered by the operating
   system to the **GLFW platform backend** inside Kitty, which resolves it through
   **XKB** and hands a normalized key event to Kitty's C core. The first thing the
   running program prints for every keypress is an **XKB-layer line** from
   `glfw/xkb_glfw.c:875`, immediately followed by Kitty's core input handler
   **`on_key_input`** (`kitty/keys.c:166`, trace at `kitty/keys.c:176`). So the platform
   backend + `kitty/glfw.c`'s `key_callback` (`kitty/glfw.c:430`) → `on_key_input`
   (`kitty/glfw.c:439`) receive the input first.

2. **Intermediate processing (who transforms it).** `on_key_input` **encodes** the key
   (`encode_glfw_key_event`, `kitty/keys.c:251`) and **writes the resulting bytes to the
   child shell's PTY**. Three different observable branches fire depending on the key:
   a *text* branch (`keys.c:252-254`), an *encoded* branch (`keys.c:255-268`), and an
   *unencodable* branch (`keys.c:270-271`). The child echoes bytes back; Kitty's
   **Child Monitor** reads those bytes on its **I/O thread** (`read_bytes`,
   `kitty/child-monitor.c:1337`) and parses them on the **main thread** (`parse_input`,
   `kitty/child-monitor.c:451`) via the **VT parser** (`run_worker`,
   `kitty/vt-parser.c:1417`), which mutates the **Screen model** and marks it dirty
   (`self->is_dirty = true`, `kitty/screen.c:119, :197, :415`).

3. **Display production (how the screen updates).** A dirty screen causes the next
   render cycle to re-upload the changed cells to the GPU and present a new frame:
   `render()` (`kitty/child-monitor.c:871`) → the dirty guard in `kitty/shaders.c:418`
   fires → `send_cell_data_to_gpu` (`kitty/shaders.c:970`) → `draw_cells`
   (`kitty/shaders.c:1009`) → `swap_window_buffers` (`kitty/child-monitor.c:810`). At
   runtime this manifested as the GPU pipeline initializing (`GL version string: '4.5
   (Core Profile) Mesa ...'`, `kitty/gl.c:72`), zero GL errors across all renders, and a
   measured **85-pixel** change in the window's framebuffer the moment the typed
   character appeared on screen.

A useful mental model that the traces confirm: **reading the child's bytes happens on a
separate thread from parsing + rendering** (Kitty's three-thread Child Monitor), which is
why input capture and screen updates are decoupled and why the frame appears a few
milliseconds after the byte is parsed (`input_delay` = 3 ms `kitty/options/definition.py:878`,
`repaint_delay` = 10 ms `:866`).

```mermaid
flowchart LR
    A["Physical keypress"] --> B["GLFW backend + XKB<br/>glfw/x11_window.c, glfw/xkb_glfw.c:875"]
    B --> C["kitty/glfw.c:430 key_callback<br/>-> :439 on_key_input"]
    C --> D["kitty/keys.c:166 on_key_input<br/>:176 trace, :251 encode"]
    D --> E["write encoded bytes to child PTY"]
    E --> F["child (bash) echoes bytes"]
    F --> G["child-monitor.c:1337 read_bytes (I/O thread)"]
    G --> H["child-monitor.c:451 parse_input (main thread)<br/>vt-parser.c:1417 run_worker"]
    H --> I["screen.c model mutation<br/>is_dirty = true (:119,:197,:415)"]
    I --> J["child-monitor.c:871 render()<br/>shaders.c:418 dirty guard"]
    J --> K["shaders.c:970 send_cell_data_to_gpu<br/>:1009 draw_cells"]
    K --> L["child-monitor.c:810 swap_window_buffers<br/>(new frame on screen)"]
```

---

## 2. How Kitty was built and launched (exact commands + environment)

**Environment.** All build-run-observe steps were performed inside the user-provided
container image (`kitty-qna-runtime:local`, derived from
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), which supplies the C11
compiler (gcc 13.3.0), Go 1.23.4, Python 3.12.3, and a headless GUI/OpenGL context. The
repository checkout is at commit `815df1e21` ("Wire up applying of font config"). A
virtual X display was already running:

```text
DISPLAY=:99   (Xvfb :99 -screen 0 1280x800x24 -ac +extension GLX +render)
LIBGL_ALWAYS_SOFTWARE=1  GALLIUM_DRIVER=llvmpipe   (Mesa software OpenGL)
```

**Default, canonical configuration.** No user or system Kitty configuration file exists,
so Kitty runs with its built-in defaults:

```text
$ ls ~/.config/kitty/kitty.conf        -> No such file or directory
$ ls /etc/xdg/kitty/kitty.conf         -> No such file or directory
```

**Build command (canonical debug build).** This is the `debug:` target from the project's
own `Makefile:22-23` (`python3 setup.py build $(VVAL) --debug`):

```text
$ cd /app
$ python3 setup.py clean
$ python3 setup.py build --debug
```

The build compiled every component on the input-to-display path (122 compile units). A
few representative lines from the real build log:

```text
[1/122] Compiling kitty/screen.c ...
[5/122] Compiling kitty/glfw.c ...
[7/122] Compiling kitty/child-monitor.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[15/122] Compiling kitty/mouse.c ...
[36/122] Compiling kitty/keys.c ...
```

It is genuinely a *debug* build — the compile flags recorded in
`build/compile_commands.json` for `kitty/keys.c` are `-DDEBUG -g3 -Og` (debug build uses
`-g3`; a release build would use `-O3`, `setup.py:1246`). The build produced the launcher
binary `kitty/launcher/kitty` (source `kitty/launcher/main.c`) and the compiled
`kitty/fast_data_types.so` C extension. The resulting binary reports:

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**Launch command (the real GUI binary with built-in tracing flags).** All three of
Kitty's own debug/tracing options were enabled (definitions verified at
`kitty/cli.py:985-999`):

```text
$ DISPLAY=:99 ./kitty/launcher/kitty \
      --debug-input \
      --debug-rendering \
      --dump-bytes /tmp/kitty_dump.bytes \
      --title kitty-dbg  > /tmp/kitty_trace.log 2>&1 &
```

- `--debug-input` / `--debug-keyboard` (`dest=debug_keyboard`, `cli.py:996-999`) — *"Print
  out key and mouse events as they are received."*
- `--debug-rendering` / `--debug-gl` (`cli.py:989-993`) — *"Debug rendering commands. This
  will cause all OpenGL calls to check for errors instead of ignoring them. Also prints
  out miscellaneous debug information."*
- `--dump-bytes /tmp/kitty_dump.bytes` (`cli.py:985-986`) — *"Path to file in which to
  store the raw bytes received from the child process."*

**Where the trace output goes (important for reading the evidence below).** Kitty's
keyboard/render debug lines are emitted by `timed_debug_print` (`kitty/monotonic.h:99-108`),
which writes to **stderr** and prefixes each logical line with a monotonic timestamp
`[seconds]` (re-armed after any line containing a newline, `monotonic.h:107`). The
`--dump-bytes` file, by contrast, is opened in binary mode (`open(args.dump_bytes, 'wb')`,
`kitty/boss.py:237`) and receives the **raw echoed bytes** verbatim (`kitty/boss.py:242-244`).
All byte-sensitive output below is shown as captured with `cat -v`/`od -c`, so an ESC
(0x1b) byte appears as `^[`.

**The default shell that Kitty spawned.** Kitty resolves the child command from the
`shell` option whose default is `.` = *"whatever shell is set as the default shell for the
current user"* (`kitty/options/definition.py:2899`; see also `is_default_shell`
`kitty/child.py:229`). Observed here:

```text
$ echo $SHELL                 -> /bin/bash
$ python3 -c "from kitty.constants import shell_path; print(shell_path)"  -> /bin/bash
# child process spawned by kitty:
14609 /bin/bash --posix
```

**Startup banner actually captured** (`cat -v`; the `^[[35m ... ^[[m` around
`on_focus_change` are literal ANSI color bytes):

```text
[0.065] Loading new XKB keymaps
[0.070] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
[0.156] OS Window created
[0.169] Failed to open systemd user bus with error: No medium found
[0.173] Child launched
[0.174] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
```

The banner already exhibits all three subsystems: the **XKB keyboard layer** (`Loading
new XKB keymaps`, `Modifier indices ...`), the **GPU/rendering layer** (`GL version
string: '4.5 (Core Profile) Mesa ...'`, emitted at `kitty/gl.c:72`), and the **child/PTY
layer** (`Child launched`).

**How keys were injected (the canonical input path).** Keys were sent with `xdotool key`
(X11 XTEST), which generates *real* X key events delivered by the X server to the focused
Kitty window. Those events flow through the genuine
`GLFW → kitty/glfw.c key_callback → on_key_input` path — the same path a physical keyboard
uses. The window was focused first with `xdotool windowfocus --sync`, then, e.g.,
`xdotool key a`. This is the **canonical** path; see §9 for why the `show-key` kitten and
remote-control `send-text` were deliberately *not* used.

---

## 3. Stage 1 — Which components receive the input first (receipt)

**Claim.** The input is received first by the **GLFW platform backend** (the X11 backend
here) together with its **XKB** key resolver, and then by Kitty's C core function
`key_callback` in `kitty/glfw.c`, which immediately calls `on_key_input`.

**Observed evidence (pressing `a`, run 1).** The very first two lines Kitty printed for the
keypress were:

```text
[3.721] ^[[31mPress^[[m xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[3.721] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
```

**Cause → effect.**

- The **first** line (`Press xkb_keycode: 0x26 clean_sym: a ...`) is emitted by the GLFW
  backend's XKB layer at **`glfw/xkb_glfw.c:875`**:
  `debug("%s xkb_keycode: 0x%x ", action == GLFW_RELEASE ? "\x1b[32mRelease\x1b[m" : "\x1b[31mPress\x1b[m", xkb_keycode)`.
  This line only appears because `--debug-input` turned on the GLFW keyboard-debug hint —
  `glfwInitHint(GLFW_DEBUG_KEYBOARD, debug_keyboard)` at **`kitty/glfw.c:1444`** (and
  `OPT(debug_keyboard) = debug_keyboard != 0` at `:1446`). Its presence *before* any
  Kitty-core line is the runtime proof that **the platform/XKB backend physically
  receives and resolves the hardware keycode first** (raw X11 keycode `0x26`) into a clean
  keysym (`a`). On Linux/X11 the native event itself is delivered by
  `glfw/x11_window.c` (Wayland/macOS would use `glfw/wl_window.c` / `glfw/cocoa_window.m`).

- The **second** line (`on_key_input: glfw key: 0x61 ...`) is the first *Kitty-core* trace,
  printed at **`kitty/keys.c:176`** inside `on_key_input` (`kitty/keys.c:166`). `on_key_input`
  is reached because `kitty/glfw.c`'s keyboard callback forwards to it:
  `key_callback` is registered at **`kitty/glfw.c:1292`**
  (`glfwSetKeyboardCallback(glfw_window, key_callback)`), defined at
  **`kitty/glfw.c:430`**, and calls `on_key_input(ev)` at **`kitty/glfw.c:439`**
  (guarded by `is_window_ready_for_callbacks() && !ev->fake_event_on_focus_change`). So
  `key_callback` is the first Kitty-side C function to see the event, and `on_key_input`
  is where core handling begins.

**A value worth reporting exactly.** The AAP's illustrative example showed
`native_code: 0x26`, but the **observed** value at runtime is `native_code: 0x61` for the
`a` key (with the XKB layer separately reporting the hardware keycode `0x26`). This is
exactly the kind of detail that only a runtime observation pins down — the value reported
here is the observed one.

**Focus is a precondition for receipt.** At startup Kitty logged
`^[[35mon_focus_change^[[m: window id: 0x1 focused: 1`, i.e. the window held X input focus,
which is why the injected key events were delivered to it.

---

## 4. Stage 2 — Intermediate processing (encode → PTY → child echo → parse → screen model)

**Claim.** `on_key_input` maps and **encodes** the key, **writes bytes to the child
shell's PTY**, the child echoes bytes back, and Kitty's **Child Monitor** reads those bytes
on its **I/O thread** while the **main thread** parses them through the **VT parser**, which
mutates the **Screen model** and marks it **dirty**.

### 4a. Encoding and the write to the child (`kitty/keys.c`)

For the same `a` keypress, the `on_key_input` line ended with:

```text
... action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
```

**Cause → effect.** After the trace at `keys.c:176`, `on_key_input` calls
`encode_glfw_key_event(...)` at **`kitty/keys.c:251`** (implemented in
`kitty/key_encoding.c`; shortcut/mapping is resolved via `kitty/keys.py`). The return
value selects one of three branches — and the `--debug-input` trace tells us *which one
fired*:

- **Text branch** (`if (size == SEND_TEXT_TO_CHILD)`, `keys.c:252`): the key's Unicode
  text is written to the child (`schedule_write_to_child`, `keys.c:253`) and the program
  prints `sent key as text to child: %s` (**`keys.c:254`**). That is exactly the tail of
  the line above — so `a` took the **text** branch and the byte `a` was written to the PTY.

(The other two branches — *encoded* and *unencodable* — are demonstrated in §6 with
Enter, Ctrl-C, arrows, F-keys, and a bare modifier.)

### 4b. The child echoes, and the Child Monitor reads it (three-thread model)

The byte written to the PTY is received by the child shell (`/bin/bash`), which **echoes**
it back over the PTY. That echoed byte is what `--dump-bytes` captured. For `a` the dump
grew by exactly one byte:

```text
# od -An -c of the bytes the child echoed for the 'a' keypress
   a
```

**Cause → effect (why reading and rendering are decoupled).** The Child Monitor runs a
**three-thread** model (verified in `kitty/child-monitor.c`): a **Main thread** (parse +
render scheduling), an **I/O thread** (PTY poll/read/write, child reaping), and a **Talk
thread** (peer sockets / remote control). The echoed bytes are read on the **I/O thread**
by `read_bytes(int fd, Screen *screen)` at **`kitty/child-monitor.c:1337`** (invoked from
the poll loop at `:1531`), and are parsed on the **Main thread** by
`parse_input(ChildMonitor *self)` at **`kitty/child-monitor.c:451`**. Because the byte
read (I/O thread) and the parse + screen update (main thread) live on different threads,
input capture is decoupled from display update — which is also why the frame appears a
short, bounded time *after* the byte arrives (see §5 timing).

### 4c. The VT parser classifies bytes and mutates the Screen model

**Cause → effect.** `parse_input` drives the VT parser. `--dump-bytes` selects the
*dumping* worker: Kitty's `Boss` wires a `DumpCommands` callback when `--dump-bytes` is
given — `ChildMonitor(self.on_child_death, DumpCommands(args) if args.dump_commands or
args.dump_bytes else None, ...)` at **`kitty/boss.py:370-372`** — and the Child Monitor
therefore selects `self->parse_func = parse_worker_dump` at **`kitty/child-monitor.c:180`**
(vs. `parse_worker` at `:181` in the non-dumping case). Both
`parse_worker_dump` (**`kitty/vt-parser.c:1493`**) and `parse_worker`
(**`kitty/vt-parser.c:1496`**) call `run_worker(...)` (**`kitty/vt-parser.c:1417`**), which
classifies the incoming bytes (printable text vs. control vs. escape sequences) and drives
the corresponding mutations into the **Screen model** in `kitty/screen.c` (with the grid /
line / cursor / history state in `kitty/line.c`, `kitty/line-buf.c`, `kitty/cursor.c`,
`kitty/history.c`). Those mutations set the screen's dirty flag:
`self->is_dirty = true` at **`kitty/screen.c:119, :197, :415`** (among other sites).

> **(Inferred from reading, confirmed at the endpoints.)** The exact interior call
> `run_worker → screen_*` mutation → the specific `is_dirty` assignment is not individually
> printed by the running program (the parser does not emit a per-byte debug line by
> default). However, both **endpoints are observed at runtime**: the child's echoed bytes
> are captured verbatim by `--dump-bytes` (input to the parser), and the screen visibly
> updates immediately afterward (output of the parser → renderer; see §5 and §7). The
> `is_dirty` mechanism is the code path that connects those two observed facts.

### 4d. What the full round-trip looks like (pressing Enter after typing `a`)

Pressing **Enter** submits the line `a` to bash; the child runs it, prints an error, and
redraws its prompt — and *all of that echoed output* was captured by `--dump-bytes`,
demonstrating the complete write→execute→echo→read→parse round-trip. The Enter keypress
trace and the (beginning of the) 352 bytes the child echoed back:

```text
[4.952] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
```

```text
# od -An -c of the first bytes the child echoed after Enter (line executed + prompt redraw)
  \r  \n 033   [   ?   2   0   0   4   l  \r 033   ]   2   ;   a
  \a  ...  b   a   s   h   :       a   :       c   o   m   m   a   n
   d       n   o   t       f   o   u   n   d  \r  \n  ...
```

**Cause → effect.** Enter is a functional key (`glfw key: 0xe001`, X11 `native_code:
0xff0d`) with no directly-printable text, so it takes the **encoded** branch and Kitty
writes a single carriage-return byte `0x0d` to the PTY (`sent encoded key to child: 0xd`,
`keys.c:261-268`). bash receives the `\r`, executes the buffered line `a`, and echoes back
`bash: a: command not found` plus a freshly-drawn prompt (the `\033 ] 133 ; ...` sequences
are bash's shell-integration marks). Those echoed bytes are exactly what the parser
consumes to update the screen model.


---

## 5. Stage 3 — How the updated display is produced

**Claim.** A dirty screen triggers a render cycle that re-uploads the changed cells to the
GPU and presents a new frame. The runtime-observable facts are: the GPU/GL pipeline
initialized successfully, GL calls ran without error across every render, and the window's
framebuffer measurably changed the instant a typed character appeared.

**Observed evidence.**

1. **The GPU pipeline is real and initialized** — captured at startup (from `kitty/gl.c:72`,
   which only prints when `--debug-rendering` is on):

   ```text
   [0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
   ```

2. **Every OpenGL call was error-checked and none failed.** `--debug-rendering` keeps
   GLAD's debug post-callback installed and routes it to `check_for_gl_error`
   (`kitty/gl.c:59-62`); i.e. it *"cause[s] all OpenGL calls to check for errors instead of
   ignoring them"* (`kitty/cli.py:991-992`). Across the entire session (startup + all
   keypresses + all renders) **no GL error line was ever printed**, which means the
   cell-upload and draw calls in the render path executed successfully.

3. **The framebuffer measurably changed when the character appeared.** In a dedicated
   before/after run, the same region of the 1280×800 window was captured before and after
   pressing `a`, and compared:

   ```text
   PIXELS_CHANGED_before_vs_after = 85     # compare -metric AE before.png after.png
   ```

   85 pixels changed — consistent with a single small `a` glyph plus the cursor advancing
   one cell. (The visual before/after is described in §7.)

**Cause → effect (the render chain).** The dirty flag set by the parser (§4c) is what makes
the next render cycle re-upload cell data instead of reusing the previous frame:

- `render(monotonic_t now, bool input_read)` at **`kitty/child-monitor.c:871`** orchestrates
  the per-window render.
- Inside cell preparation, the guard at **`kitty/shaders.c:418`** is
  `... else if (screen->reload_all_gpu_data || screen->scroll_changed || screen->is_dirty ||
  screen_resized || ...) update_cell_data;` — so when `screen->is_dirty` is true, the
  `update_cell_data` macro (defined at **`kitty/shaders.c:408`**) runs and re-computes the
  cell data (calling `screen_update_cell_data`).
- `send_cell_data_to_gpu(...)` at **`kitty/shaders.c:970`** uploads that cell data to the
  GPU; `draw_cells(...)` at **`kitty/shaders.c:1009`** runs the shader programs (glyphs come
  from the sprite atlas built by `kitty/glyph-cache.c` / `kitty/freetype.c` /
  `kitty/fonts.c`; OpenGL submission goes through `kitty/gl.c` / `kitty/gl-wrapper.c`).
- Finally the frame is presented by `swap_window_buffers(os_window)` at
  **`kitty/child-monitor.c:810`** (inside `render_prepared_os_window`, `:788`), i.e. GLFW
  swaps the window's buffers and the new frame becomes visible.

> **(Inferred from reading, confirmed at the endpoints.)** The interior chain
> `is_dirty → update_cell_data → send_cell_data_to_gpu → draw_cells → swap_window_buffers`
> does not print a per-frame debug line in the default `--debug-rendering` output (that flag
> mainly adds the GL-version banner, per-call GL error checking, and frame-timeout
> warnings). It is therefore labeled inferred-from-reading — but it is bracketed by two
> runtime observations: the parser input (dumped bytes) and the parser/render output (the
> 85-pixel framebuffer change and the visible glyph in §7), plus the fact that the GL
> pipeline initialized and never errored.

**Why the frame appears a few milliseconds after the byte — timing.** Kitty intentionally
coalesces input before repainting, governed by two default timing knobs:
`input_delay` = **3 ms** (`kitty/options/definition.py:878`) and `repaint_delay` = **10 ms**
(`kitty/options/definition.py:866`). In the captured traces the keypress and the resulting
screen change occur within a few milliseconds of each other (e.g. the `a` press is stamped
`[3.721]` and the echoed byte + screen update follow immediately), consistent with these
small delays. This is why the display updates *just after* — not exactly simultaneously
with — the keypress.

---

## 6. Secondary and edge conditions (every implied input mode, exercised)

The question's "a few simple keys" was treated as the primary path, and the "such
as / including" variants were all exercised at runtime. Each sub-section shows the raw
captured `on_key_input` line first, then the branch it took. All byte values are shown
exactly as emitted (ESC = `^[`, per the byte-print legend at `kitty/keys.c:262-268`:
`27 → "^[ "`, space `→ "SPC "`, printable `→ "<char> "`, otherwise `→ "0x%x "`).

### 6a. Printable character `a` — text branch (`keys.c:252-254`)

```text
[3.721] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
```
Echoed byte: `a`. The key has printable text, so `encode_glfw_key_event` returns
`SEND_TEXT_TO_CHILD` and the text is written directly (`keys.c:253-254`).

### 6b. Enter (Return) — encoded branch, single byte (`keys.c:255-268`)

```text
[4.952] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
```
Encoded output: `0xd` (carriage return). No printable text, so it is encoded; `0x0d` is a
non-printable byte, hence the legend prints it as `0xd` (`keys.c:266`).

### 6c. Ctrl-C — encoded branch producing a control byte (`keys.c:255-268`)

Two `on_key_input` lines fire — first the bare Ctrl press (not encodable), then `c` with
the Ctrl modifier:

```text
[6.184] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[6.190] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3
```
Encoded output: `0x3` (ETX, the interrupt character). The child echoed `^C` followed by a
fresh prompt:

```text
# od -An -c of the first echoed bytes after Ctrl-C
   ^   C 033   [   ?   2   0   0   4   l  \r  ...  # new prompt follows
```

**Cause → effect for the signal.** The encoded output is a single byte
(`size == 1`), so `keys.c:256` checks `screen->modes.mHANDLE_TERMIOS_SIGNALS`. That private
mode (mode `19997`, `kitty/modes.h:89`) is **off in the default configuration**, so the
signal route `screen_send_signal_for_key(...)` (`keys.c:257`, `kitty/screen.c:2404`) is
**not** taken — which is exactly why the trace shows the byte-write path
`sent encoded key to child: 0x3` (`keys.c:259-268`) rather than an early return. The
`0x03` byte is written to the PTY, and the kernel's terminal line discipline (termios
`ISIG`) converts it into `SIGINT` for bash's foreground process group. The observable
result — `^C` echoed and a new prompt — confirms the interrupt happened. *(The
`screen->modes.mHANDLE_TERMIOS_SIGNALS`/`screen_send_signal_for_key` branch not being taken
is inferred from reading the mode's default; the byte-write path actually taken is directly
observed in the trace.)*

### 6d. Arrow key (Up) — encoded branch, multi-byte CSI sequence (`keys.c:255-268`)

```text
[7.429] ^[[33mon_key_input^[[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A
```
Encoded output: `^[ [ A` — i.e. the three bytes `ESC` `[` `A` (the classic ANSI cursor-up
sequence, CSI A). The legend prints ESC as `^[ ` (`keys.c:263`) and the printable `[` and
`A` as themselves (`keys.c:265`). This is a **special/functional key** that *is* encodable,
so it produces a multi-byte escape sequence rather than being dropped.

### 6e. Function key (F1) — encoded branch, SS3 sequence (`keys.c:255-268`)

```text
[8.661] ^[[33mon_key_input^[[m: glfw key: 0xe014 native_code: 0xffbe action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ O P
```
Encoded output: `^[ O P` — the three bytes `ESC` `O` `P` (the SS3-style F1 sequence). This
demonstrates a second special-key encoding shape distinct from the CSI form used by arrows.
*(In this run bash rang the terminal bell on the unrecognized sequence, so the child echoed
a single `\a` (BEL) byte; unrelated ALSA-library stderr noise from the bell attempt was
interleaved and is environmental, not part of Kitty's pipeline.)*

### 6f. Bare modifier (Shift alone) — unencodable branch (`keys.c:270-271`)

```text
[9.895] ^[[33mon_key_input^[[m: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```
Echoed bytes: **none (0 bytes)**. A bare modifier press has no text and no encodable
representation in the active (default/legacy) keyboard mode, so `encode_glfw_key_event`
returns a non-positive size and control reaches the `else` at `keys.c:270`, printing
`ignoring as keyboard mode does not support encoding this event` (`keys.c:271`). This is
the honest edge case: **not every key produces child output** — a bare modifier is
consumed with nothing sent, and correspondingly nothing was captured by `--dump-bytes`.

### 6g. Release events are also "ignored" for encoding

Every key's `RELEASE` event reached the same unencodable branch (no text, nothing to
encode), e.g. for `a`:

```text
[3.727] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```
So in the default legacy keyboard mode, it is the **PRESS** (and REPEAT) events that
produce bytes; RELEASE events are received and traced but produce no child output.

**Summary of the branch taken per condition (all runtime-observed):**

| Condition (`xdotool key ...`) | Branch (`kitty/keys.c`) | Encoded/text output | Bytes to child |
|---|---|---|---|
| `a` (printable) | text `:252-254` | `sent key as text to child: a` | `a` |
| `Return` (Enter) | encoded `:255-268` | `sent encoded key to child: 0xd` | `0x0d` |
| `ctrl+c` | encoded `:255-268` | `sent encoded key to child: 0x3` | `0x03` (→ SIGINT) |
| `Up` (arrow) | encoded `:255-268` | `sent encoded key to child: ^[ [ A` | `ESC [ A` |
| `F1` (function) | encoded `:255-268` | `sent encoded key to child: ^[ O P` | `ESC O P` |
| `Shift_L` (bare mod) | unencodable `:270-271` | `ignoring ... does not support encoding this event` | *(none)* |


---

## 7. Before / during / after the state change

Because typing changes state, the value of the target screen cell was observed **before**,
**during**, and **after** a single `a` keypress, in a dedicated run.

- **Before** — the prompt is drawn and the target cell is empty. The dump file held only
  the 383 startup bytes and the trace held only the 9 startup lines. The captured
  framebuffer region showed:

  ```text
  root@63735bac780f:/app#
  ```
  *(cursor positioned right after `# `; no character typed)*

- **During** — the `a` key is pressed. `on_key_input` takes the text branch and writes `a`
  to the PTY; the child echoes it back (the dump grew from 383 → 384 bytes, i.e. exactly
  the one echoed `a`). The parser consumes that byte and mutates the screen model, setting
  `is_dirty` (`kitty/screen.c:119/:197/:415`). Captured trace for this instant:

  ```text
  [3.967] ^[[31mPress^[[m xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
  [3.967] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
  ```
  *(The `is_dirty` assignment itself is inferred from reading — the parser prints no
  per-byte line — but its two endpoints, the echoed byte and the resulting frame, are both
  observed.)*

- **After** — the render cycle re-uploads the cell and swaps the buffer; the glyph is now
  on screen. The captured framebuffer region showed:

  ```text
  root@63735bac780f:/app# a
  ```
  *(the `a` is now displayed, cursor advanced one cell)*

  A pixel comparison of the before and after framebuffers reported **85 changed pixels** —
  the observable proof that the display was re-rendered as a direct result of the keypress.

This before → during → after transition is the concrete, observed demonstration of the
whole pipeline: the cell is unchanged until the echoed byte is parsed (`is_dirty` set), and
only after the subsequent render cycle does the character appear on screen.

---

## 8. Stability across identical runs (consistently observed)

The identical keypress sequence (`a`, `Return`, `ctrl+c`, `Up`, `F1`, `Shift_L`) was run
**three** times (run 1, run 2, run 3) with the same build and launch command. After
stripping the per-line `[seconds]` timestamps, the structural trace for **every** condition
was byte-for-byte identical across all three runs, and the encoded outputs matched exactly:

```text
01_printable_a: STABLE (run1 == run2 == run3, structure identical)
02_enter:       STABLE (run1 == run2 == run3, structure identical)
03_ctrl_c:      STABLE (run1 == run2 == run3, structure identical)
04_arrow_up:    STABLE (run1 == run2 == run3, structure identical)
05_fkey_f1:     STABLE (run1 == run2 == run3, structure identical)
06_bare_shift:  STABLE (run1 == run2 == run3, structure identical)

encoded output per run:
  a       -> "sent key as text to child: a"     (run1, run2, run3)
  Enter   -> "sent encoded key to child: 0xd "  (run1, run2, run3)
  Ctrl-C  -> "sent encoded key to child: 0x3 "  (run1, run2, run3)
  Up      -> "sent encoded key to child: ^[ [ A " (run1, run2, run3)
  F1      -> "sent encoded key to child: ^[ O P " (run1, run2, run3)
  Shift_L -> (no encoded output)                (run1, run2, run3)
```

For example, the `a` keypress in **run 2** produced the same two lines as run 1, differing
only in the timestamp (`[3.796]` vs `[3.721]`):

```text
[3.796] ^[[31mPress^[[m xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[3.796] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
```

The only run-to-run variation was the monotonic timestamp on each line (expected). No
lines were dropped, added, or reordered, and no byte value changed. The behavior described
in this document is therefore **consistently observed**, not incidental to one run.

---

## 9. Canonical vs. non-canonical: how the input was driven

All evidence above came from the **canonical** input path. Keys were injected with
`xdotool key` (X11 XTEST), which generates real X key events; the X server delivers them to
the focused Kitty window; and they flow through
`GLFW platform backend → glfw/xkb_glfw.c (XKB) → kitty/glfw.c:430 key_callback →
kitty/glfw.c:439 on_key_input → kitty/keys.c:166` — the exact same path a physical keyboard
uses. The `on_key_input` traces and the `--dump-bytes` capture confirm the bytes originated
from that path.

Two other facilities were deliberately **not** used to drive the measured keys, because
they would bypass the real path and would therefore be **non-canonical** for this question:

- **`kitten show-key`** reads its *own* stdin as a standalone TUI; it does not exercise
  Kitty's internal `on_key_input`, so it cannot represent "input during normal use."
- **Remote control `send-text`** (e.g. `kitten @ send-text`) injects text directly toward
  the child, bypassing the GLFW/`kitty/keys.c` encoding path. It would show bytes at the
  child without any `on_key_input` receipt/encoding — i.e. it skips the very stages the
  question asks about.

They are mentioned only to contrast; every trace and byte in this document is from the
genuine GLFW → `keys.c` → PTY → VT parser → screen → GPU path.

---

## 10. Appendix — complete captured evidence, with the commands that produced it

### 10.1 Build (canonical debug build)

```text
$ cd /app
$ python3 setup.py clean
$ python3 setup.py build --debug        # == Makefile:22-23 debug target
...
[36/122] Compiling kitty/keys.c ...
...
# resulting debug compile flags for keys.c (build/compile_commands.json): -DDEBUG -g3 -Og
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### 10.2 Launch (real GUI binary + built-in tracing flags)

```text
$ DISPLAY=:99 ./kitty/launcher/kitty --debug-input --debug-rendering \
      --dump-bytes /tmp/kitty_dump.bytes --title kitty-dbg > /tmp/kitty_trace.log 2>&1 &
# child spawned:  /bin/bash --posix
```

### 10.3 Full startup trace (run 1, `cat -v`)

```text
[0.065] Loading new XKB keymaps
[0.070] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
[0.156] OS Window created
[0.169] Failed to open systemd user bus with error: No medium found
[0.173] Child launched
[0.174] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
```

### 10.4 Per-condition captures (run 1)

Each block is the exact stderr-trace delta (`cat -v`) for the keypress, followed by the
exact bytes the child echoed (`od -An -c`), produced by `xdotool key <spec>` after focusing
the window.

**`xdotool key a`**
```text
[3.721] ^[[31mPress^[[m xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[3.721] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[3.727] ^[[32mRelease^[[m xkb_keycode: 0x26 clean_sym: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[3.727] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
--- echoed bytes ---
   a
```

**`xdotool key Return`**
```text
[4.952] ^[[31mPress^[[m xkb_keycode: 0x24 clean_sym: Return composed_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[4.952] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
[4.959] ^[[32mRelease^[[m xkb_keycode: 0x24 clean_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[4.959] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
--- echoed bytes (352 total; beginning shown) ---
  \r  \n 033   [   ?   2   0   0   4   l  \r 033   ]   2   ;   a
  \a  ...  b   a   s   h   :       a   :       c   o   m   m   a   n
   d       n   o   t       f   o   u   n   d  \r  \n  ...
```

**`xdotool key ctrl+c`**
```text
[6.183] ^[[31mPress^[[m xkb_keycode: 0x25 clean_sym: Control_L composed_sym: Control_L mods: none glfw_key: 57442 (LEFT_CONTROL) xkb_key: 65507 (Control_L)
[6.184] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[6.190] ^[[31mPress^[[m xkb_keycode: 0x36 clean_sym: c composed_sym: c mods: ctrl glfw_key: 99 (c) xkb_key: 99 (c)
[6.190] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3
[6.198] ^[[32mRelease^[[m xkb_keycode: 0x25 clean_sym: Control_L mods: ctrl glfw_key: 57442 (LEFT_CONTROL) xkb_key: 65507 (Control_L)
[6.202] ^[[32mRelease^[[m xkb_keycode: 0x36 clean_sym: c mods: none glfw_key: 99 (c) xkb_key: 99 (c)
--- echoed bytes (214 total; beginning shown) ---
   ^   C 033   [   ?   2   0   0   4   l  \r  ...  # new prompt follows
```

**`xdotool key Up`**
```text
[7.429] ^[[31mPress^[[m xkb_keycode: 0x6f clean_sym: Up composed_sym: Up mods: none glfw_key: 57352 (UP) xkb_key: 65362 (Up)
[7.429] ^[[33mon_key_input^[[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A
[7.436] ^[[32mRelease^[[m xkb_keycode: 0x6f clean_sym: Up mods: none glfw_key: 57352 (UP) xkb_key: 65362 (Up)
--- echoed bytes ---
   a
```

**`xdotool key F1`**
```text
[8.661] ^[[31mPress^[[m xkb_keycode: 0x43 clean_sym: F1 composed_sym: F1 mods: none glfw_key: 57364 (F1) xkb_key: 65470 (F1)
[8.661] ^[[33mon_key_input^[[m: glfw key: 0xe014 native_code: 0xffbe action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ O P
[8.669] ^[[32mRelease^[[m xkb_keycode: 0x43 clean_sym: F1 mods: none glfw_key: 57364 (F1) xkb_key: 65470 (F1)
--- echoed bytes ---
  \a
```

**`xdotool key Shift_L`**
```text
[9.895] ^[[31mPress^[[m xkb_keycode: 0x32 clean_sym: Shift_L composed_sym: Shift_L mods: none glfw_key: 57441 (LEFT_SHIFT) xkb_key: 65505 (Shift_L)
[9.895] ^[[33mon_key_input^[[m: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[9.901] ^[[32mRelease^[[m xkb_keycode: 0x32 clean_sym: Shift_L mods: shift glfw_key: 57441 (LEFT_SHIFT) xkb_key: 65505 (Shift_L)
--- echoed bytes ---
(0 bytes)
```

### 10.4a Complete (un-truncated) echoed-byte dumps for the multi-byte conditions

For completeness, the two conditions whose echoed output spans many bytes (Enter and
Ctrl-C) are reproduced here **in full, unedited** — every byte the child echoed, exactly as
`od -An -c` rendered it (ESC shown as its octal escape `033`, `\r`/`\n`/`\a` as C escapes,
control bytes such as `001`/`002` as octal). These are the same captures whose beginnings
appear abbreviated in §4 and §10.4; nothing is elided below. The large volume is the shell's
prompt-redraw / shell-integration output (the `\033 ] 133 ; ...` OSC 133 sequences), not the
keypress encoding itself — the keypress encoding (`0xd` for Enter, `0x3` for Ctrl-C) is the
single byte shown in the corresponding trace line.

**`xdotool key Return` — complete echoed bytes (352 total):**
```text
# od -An -c /tmp/kitty_capture/run1/02_enter.bytes
  \r  \n 033   [   ?   2   0   0   4   l  \r 033   ]   2   ;   a
  \a 033   ]   1   3   3   ;   C   ;   c   m   d   l   i   n   e
   =   a  \a 001 033   ]   1   3   3   ;   k   ;   s   t   a   r
   t   _   k   i   t   t   y  \a 002 001 033   ]   1   3   3   ;
   k   ;   e   n   d   _   k   i   t   t   y  \a 002 001 033   ]
   1   3   3   ;   k   ;   s   t   a   r   t   _   s   u   f   f
   i   x   _   k   i   t   t   y  \a 002 001 033   [   0       q
 002 001 033   ]   1   3   3   ;   k   ;   e   n   d   _   s   u
   f   f   i   x   _   k   i   t   t   y  \a 002   b   a   s   h
   :       a   :       c   o   m   m   a   n   d       n   o   t
       f   o   u   n   d  \r  \n 033   [   ?   2   0   0   4   h
 033   ]   1   3   3   ;   k   ;   s   t   a   r   t   _   k   i
   t   t   y  \a 033   ]   1   3   3   ;   D   ;   1   2   7  \a
 033   ]   1   3   3   ;   A  \a 033   ]   1   3   3   ;   k   ;
   e   n   d   _   k   i   t   t   y  \a 033   ]   0   ;   r   o
   o   t   @   6   3   7   3   5   b   a   c   7   8   0   f   :
       /   a   p   p  \a   r   o   o   t   @   6   3   7   3   5
   b   a   c   7   8   0   f   :   /   a   p   p   #     033   ]
   1   3   3   ;   k   ;   s   t   a   r   t   _   s   u   f   f
   i   x   _   k   i   t   t   y  \a 033   [   5       q 033   ]
   2   ;   /   a   p   p  \a 033   ]   1   3   3   ;   k   ;   e
   n   d   _   s   u   f   f   i   x   _   k   i   t   t   y  \a
```

**`xdotool key ctrl+c` — complete echoed bytes (214 total):**
```text
# od -An -c /tmp/kitty_capture/run1/03_ctrl_c.bytes
   ^   C 033   [   ?   2   0   0   4   l  \r 033   [   ?   2   0
   0   4   h 033   [   ?   2   0   0   4   l  \r  \r  \n 033   [
   ?   2   0   0   4   h 033   ]   1   3   3   ;   k   ;   s   t
   a   r   t   _   k   i   t   t   y  \a 033   ]   1   3   3   ;
   D   ;   1   3   0  \a 033   ]   1   3   3   ;   A  \a 033   ]
   1   3   3   ;   k   ;   e   n   d   _   k   i   t   t   y  \a
 033   ]   0   ;   r   o   o   t   @   6   3   7   3   5   b   a
   c   7   8   0   f   :       /   a   p   p  \a   r   o   o   t
   @   6   3   7   3   5   b   a   c   7   8   0   f   :   /   a
   p   p   #     033   ]   1   3   3   ;   k   ;   s   t   a   r
   t   _   s   u   f   f   i   x   _   k   i   t   t   y  \a 033
   [   5       q 033   ]   2   ;   /   a   p   p  \a 033   ]   1
   3   3   ;   k   ;   e   n   d   _   s   u   f   f   i   x   _
   k   i   t   t   y  \a
```

The first two bytes of the Ctrl-C dump (`^ C`) are the terminal's echo of the interrupt
character, confirming the `0x3` byte from the trace reached the child; everything after is
the shell's post-`SIGINT` prompt redraw.

### 10.5 Component → source reference map (all consulted read-only)

| Pipeline role | Function / site | `file:line` |
|---|---|---|
| Debug flag: `--dump-bytes` | option definition | `kitty/cli.py:985-986` |
| Debug flag: `--debug-rendering`/`--debug-gl` | option definition | `kitty/cli.py:989-993` |
| Debug flag: `--debug-input`/`--debug-keyboard` | option definition | `kitty/cli.py:996-999` |
| Receipt (platform, X11) | native key delivery | `glfw/x11_window.c` (Wayland `glfw/wl_window.c`, macOS `glfw/cocoa_window.m`) |
| Receipt (XKB trace) | `debug("%s xkb_keycode ...")` | `glfw/xkb_glfw.c:875` |
| Receipt (kitty core) | `key_callback` | `kitty/glfw.c:430` (registered `:1292`) |
| Receipt (dispatch) | `on_key_input(ev)` call | `kitty/glfw.c:439` |
| Debug wiring | `GLFW_DEBUG_KEYBOARD` hint | `kitty/glfw.c:1444` / `:1446` |
| Intermediate (handler) | `on_key_input` | `kitty/keys.c:166` (trace `:176`) |
| Intermediate (encode) | `encode_glfw_key_event` | `kitty/keys.c:251` (impl `kitty/key_encoding.c`, mapping `kitty/keys.py`) |
| Branch: text | `sent key as text to child` | `kitty/keys.c:252-254` |
| Branch: encoded + byte legend | `sent encoded key to child` | `kitty/keys.c:255-268` |
| Branch: signal route (default off) | `screen_send_signal_for_key` | `kitty/keys.c:256-257`, `kitty/screen.c:2404`, mode `kitty/modes.h:89` |
| Branch: unencodable | `ignoring ... does not support encoding` | `kitty/keys.c:270-271` |
| Transport (worker select) | `parse_worker_dump` / `parse_worker` | `kitty/child-monitor.c:180-181` |
| Output (I/O thread read) | `read_bytes` | `kitty/child-monitor.c:1337` (poll `:1531`) |
| Output (main thread parse) | `parse_input` | `kitty/child-monitor.c:451` |
| Parser | `run_worker` (via `parse_worker(_dump)`) | `kitty/vt-parser.c:1417` (`:1493`/`:1496`) |
| Screen model dirty flag | `self->is_dirty = true` | `kitty/screen.c:119, :197, :415` |
| Display (render orchestration) | `render()` | `kitty/child-monitor.c:871` |
| Display (dirty guard + upload) | `update_cell_data` guard | `kitty/shaders.c:418` (macro `:408`) |
| Display (GPU upload) | `send_cell_data_to_gpu` | `kitty/shaders.c:970` |
| Display (draw) | `draw_cells` | `kitty/shaders.c:1009` |
| Display (present) | `swap_window_buffers` | `kitty/child-monitor.c:810` (`render_prepared_os_window` `:788`) |
| Display (GL init/version) | `gl_init` / GL version print | `kitty/gl.c:52` / `:72` (error check `:59-62`) |
| Timing knobs | `input_delay` / `repaint_delay` | `kitty/options/definition.py:878` / `:866` |
| Debug output primitive | `timed_debug_print` (→ stderr, `[secs]`) | `kitty/monotonic.h:99-108` |
| `--dump-bytes` sink | `DumpCommands` (binary, raw) | `kitty/boss.py:237, :242-244` (wired `:370-372`) |
| Orchestration / entry | `main_loop()` / `ChildMonitor` created | `kitty/main.py:234` / `kitty/boss.py:370` |
| Default shell resolution | `is_default_shell` / shell default `.` | `kitty/child.py:229` / `kitty/options/definition.py:2899` |
| Docs corroboration | `--debug-input` prints per-key text | `docs/mapping.rst:115, :338` |

### 10.6 Notes on methodology and integrity

- **Runtime-first.** The code paths were built and run before this answer was written; every
  behavioral claim is backed by the captured output shown next to it.
- **Byte-exact.** Byte-sensitive values are shown as emitted (`^[` for ESC, `SPC` for space,
  `0x%x` for other non-printables — the legend at `kitty/keys.c:262-268`), taken directly
  from the trace and from `od -c` of the `--dump-bytes` file.
- **Inferred vs. observed.** The only links labeled *(inferred from reading)* are the
  parser's internal `is_dirty` assignment and the interior render chain, neither of which
  prints a per-event debug line by default; both are bracketed by observed endpoints
  (dumped bytes on one side, the 85-pixel framebuffer change / visible glyph on the other).
- **Read-only + cleanup.** No Kitty source file was modified. All tracing used Kitty's own
  built-in flags; all helper scripts, the trace log, the `--dump-bytes` file, and the
  before/after screenshots lived outside the repository (or were deleted), leaving the
  checkout unchanged except for this document.

