# How Input Moves Through Kitty's Core Components — A Runtime‑Grounded Walkthrough

> **Question.** "If I start Kitty from this repository with whatever debugging or tracing options are available and press a few simple keys inside the default shell, what does the observable runtime behavior reveal about how input moves through Kitty's core components before the screen updates?"

This document answers that question by **building and running the checked‑out code first**, capturing the real trace output, and only then explaining what the pipeline does — with every claim tied either to a quoted, verbatim capture or to an exact `file:line` reference in the source.

The question decomposes into three sub‑questions, each answered explicitly below and re‑checked in a closing coverage pass:

- **Q1 — Input ingress:** which components receive the input *first*.
- **Q2 — Intermediate processing:** which components handle *parsing, encoding, and dispatch*.
- **Q3 — Display production:** how the *updated display is ultimately produced*.

---

## 1. Introduction — what was built, run, and pressed

### 1.1 Commit under study

All observations below come from the repository checked out at commit **`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`** (this is `kitty 0.35.2`). The build and the run were performed inside the provided Docker image `kitty-build-env:ready` (based on `ghcr.io/scaleapi/swe-atlas`), whose `/app` directory is this repository at that exact commit. Confirmed with:

```console
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

### 1.2 Build (run‑first, step 1)

Kitty is not pure Python — it has a C core plus a Go tools layer — so a binary must be compiled before anything can be observed. The `Makefile` target `all` runs the build (`Makefile:L12-L13` → `python3 setup.py`):

```console
$ make all
...
[1/2] Linking [wayland] kitty/glfw-wayland ...
[2/2] Linking launcher ...
 done
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root 1001 36224 ... kitty/launcher/kitty
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

> The build **adds** the compiled launcher `kitty/launcher/kitty`; the C sources `kitty/launcher/{launcher.h,main.c,single-instance.c}` are pre‑existing tracked files and are left untouched. The build outputs (`build/`, `kitty/launcher/kitt*`, `*.so`) are all git‑ignored (`.gitignore`), so building never dirties the working tree.

### 1.3 Headless display + software GL (run‑first, step 2)

Kitty renders through OpenGL and requires a GL context. In source, the required version is assembled from `kitty/data-types.h:L20` `#define OPENGL_REQUIRED_VERSION_MAJOR 3` and a platform‑dependent minor: `kitty/data-types.h:L22` `#define OPENGL_REQUIRED_VERSION_MINOR 3` inside `#ifdef __APPLE__` (`:L21`) versus `kitty/data-types.h:L24` `#define OPENGL_REQUIRED_VERSION_MINOR 1` in the `#else` (`:L23`) branch — so on this **Linux** host the minimum is **OpenGL 3.1** (with `#define GLSL_VERSION 140` at `:L26`). Kitty requests that context via `kitty/glfw.c:L1127-L1128` `glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR/MINOR, OPENGL_REQUIRED_VERSION_MAJOR/MINOR)`.

Because there is no GPU, a virtual X display plus Mesa's software rasterizer (`llvmpipe`) were used:

```console
$ Xvfb :99 -screen 0 1024x768x24 &
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
$ glxinfo -B | grep -i 'OpenGL renderer\|core profile version'
OpenGL renderer string: llvmpipe (LLVM 19.1.1, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1
```

> **Software‑GL caveat.** All rendering observed here proceeds through **Mesa `llvmpipe` software OpenGL**, not a hardware GPU. This is important when interpreting the `--debug-rendering` output in Q3.

### 1.4 Run with tracing + the exact keys pressed (run‑first, step 3)

Two documented debug flags were passed together so all three stages appear in one session — `--debug-input` (a.k.a. `--debug-keyboard`) and `--debug-rendering` (a.k.a. `--debug-gl`), defined in `kitty/cli.py:L996` and `kitty/cli.py:L989`. The default shell (`/bin/bash`) was launched by giving kitty no program argument. Both output streams were captured separately (the reason is explained in §2):

```console
# stdout -> cap.out (GL line), stderr -> cap.err (key traces)
$ kitty/launcher/kitty --debug-input --debug-rendering \
      --dump-bytes=/evidence/bytes.log -o repaint_delay=2 \
      > /evidence/cap.out 2> /evidence/cap.err &

# focus the window under Xvfb, then press a few simple keys in the shell:
$ WID=$(xdotool search --class kitty | head -1)
$ xdotool windowfocus "$WID"
$ xdotool type --delay 150 'ls'   # printable keys: l, s
$ xdotool key Return              # the Enter key
```

The **exact keys pressed** were: `l`, `s`, then **Enter** (typing `ls` and running it), later followed by `e`,`x`,`i`,`t`, Enter to close the shell. This stimulus is deliberately chosen so that printable keys exercise the *text* path while Enter exercises the *encoded* path (see Q2). A **second** session pressed three default kitty shortcuts (`kitty_mod` = `ctrl+shift`) to exercise the *shortcut‑dispatch* path:

```console
$ kitty/launcher/kitty --debug-input --debug-rendering -o repaint_delay=2 \
      > /evidence/sc.out 2> /evidence/sc.err &
$ xdotool key --clearmodifiers ctrl+shift+l       # next_layout
$ xdotool key --clearmodifiers ctrl+shift+equal   # change_font_size
$ xdotool key --clearmodifiers ctrl+shift+c       # copy_to_clipboard
```

Running the `ls`+Enter stimulus twice produced **byte‑for‑byte identical** key‑trace lines (after stripping the timestamp/color prefixes), confirming the behavior described here is *consistently* observed at runtime rather than incidental.

---

## 2. How the tracing works (foundation for every quoted line)

Understanding two facts about the trace plumbing makes the rest of the evidence unambiguous.

**(a) The flags gate two different trace macros.**

- `--debug-input`/`--debug-keyboard` sets the option `debug_keyboard` (`kitty/cli.py:L997` `dest=debug_keyboard`; help `kitty/cli.py:L999` "Print out key and mouse events as they are received."). That option gates the `debug_input(...)` macro: `kitty/state.h:L15` `#define debug_input(...) if (OPT(debug_keyboard)) { timed_debug_print(__VA_ARGS__); }`. Inside `kitty/keys.c`, the short name `debug` *is* that macro (`kitty/keys.h:L16` `#define debug debug_input`).
- `--debug-rendering`/`--debug-gl` sets `global_state.debug_rendering` (`kitty/cli.py:L989-L993`, help: "…cause all OpenGL calls to check for errors instead of ignoring them. Also prints out miscellaneous debug information."). That gates `kitty/state.h:L14` `#define debug_rendering(...) if (global_state.debug_rendering) { timed_debug_print(__VA_ARGS__); }`.

**(b) Traces go to two different streams, and each line is timestamped once.**

Both macros route through `timed_debug_print` (`kitty/monotonic.h:L99`). At the *start of every new line* it emits a monotonic timestamp — `kitty/monotonic.h:L102` `fprintf(stderr, "[%.3f] ", ...)` — and writes the body with `kitty/monotonic.h:L105` `vfprintf(stderr, fmt, args)`, i.e. to **stderr**. A `starting_print` flag (`:L101`, updated at `:L107`) means the `[%.3f]` prefix is printed only once per newline‑terminated line, so several `debug(...)` fragments that build one logical line share a **single** timestamp.

The one exception is the GL version banner: `kitty/gl.c:L72` uses `printf(...)` → **stdout** (with its own inline `[%.3f]`). That is why the run above redirects stdout and stderr to different files: the key traces land in `cap.err`, the GL line in `cap.out`.


---

## 3. Q1 — Input ingress: which components receive the input first

**Answer.** The operating system / display server (here X11 via Xvfb) hands the key event to Kitty's **GLFW backend callback `key_callback`** (`kitty/glfw.c:L430`), which — for a real, non‑focus‑synthetic event on a ready window — forwards it to Kitty's own entry point **`on_key_input`** (`kitty/keys.c:L166`). `on_key_input` is the first *Kitty‑owned* code to see the keystroke.

**Code path.**
- The callback is registered once, at window creation: `kitty/glfw.c:L1292` `glfwSetKeyboardCallback(glfw_window, key_callback);`.
- The hand‑off happens at `kitty/glfw.c:L439`: `if (is_window_ready_for_callbacks() && !ev->fake_event_on_focus_change) on_key_input(ev);`.
- `on_key_input(GLFWkeyevent *ev)` begins at `kitty/keys.c:L166`. When keyboard debugging is on (`kitty/keys.c:L172` `if (OPT(debug_keyboard))`), it prints the event it received at `kitty/keys.c:L176` (the IME‑only variant `on_IME_input` is at `:L174`).

**Observed proof.** Every key produced an `on_key_input:` line on **stderr**. Command that produced it:

```console
$ xdotool type --delay 150 'ls'   # then: xdotool key Return
$ grep -a on_key_input cap.err | cat -v      # cat -v renders ESC 0x1B as ^[
```

Verbatim (the `^[` sequences are the literal `\x1b[33m`/`\x1b[m` SGR yellow/reset that `kitty/keys.c:L176` wraps around `on_key_input`):

```text
[5.459] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[5.530] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[6.210] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
```

**Reading the line.** The fields map exactly to the format string at `kitty/keys.c:L176` (`glfw key: 0x%x native_code: 0x%x action: %s %stext: '%s' state: %d`):

- `glfw key: 0x6c` / `native_code: 0x6c` — the `l` key (`0x6c` = ASCII `l`); `0x73` = `s`; the Enter key reports the GLFW functional keycode `glfw key: 0xe001` with X11 `native_code: 0xff0d` (`XK_Return`).
- `action: PRESS` — one of `PRESS`/`RELEASE`/`REPEAT` (`kitty/keys.c:L178`).
- `mods: none` — produced by `format_mods` (`kitty/keys.c:L144`); a modified chord instead reads e.g. `mods: ctrl+shift ` (see Q2).
- `text: 'l'` — the UTF‑8 text GLFW resolved for the key; empty for non‑text keys such as Enter.

The mere presence of these lines is the runtime proof that `on_key_input` (`kitty/keys.c:L166`, trace at `:L176`) is where input first enters Kitty, immediately downstream of `key_callback` (`kitty/glfw.c:L430`). The trailing `sent key as text…` / `sent encoded key…` text on the *same* line is the next stage (Q2), sharing the one timestamp because of the `starting_print` behavior described in §2.


---

## 4. Q2 — Intermediate processing: parsing, encoding, and dispatch

**Answer.** Between ingress and the child, `on_key_input` (`kitty/keys.c:L166`) drives three sub‑stages: (1) it first asks the Python `Boss` whether the chord is a mapped **shortcut** and, if so, dispatches it and sends nothing to the child; (2) otherwise it **encodes** the event into bytes with `encode_glfw_key_event` (`kitty/key_encoding.c:L414`); and (3) it **writes** those bytes to the child PTY via `schedule_write_to_child` (`kitty/child-monitor.c:L372`). The child's reply is then read back and **parsed** by the VT state machine (`kitty/vt-parser.c`) into updates on the screen‑grid model (`kitty/screen.c`). Each of these sub‑stages emits its own trace line, shown below.

After `on_key_input` receives the event, three things can happen to it. The traces let us watch each branch.

### 4.1 Shortcut dispatch (into the Python `Boss`)

For a `PRESS`/`REPEAT` (`kitty/keys.c:L226`), `on_key_input` first asks whether the chord is a mapped shortcut by dispatching into Python: `kitty/keys.c:L228` `dispatch_key_event(dispatch_possible_special_key);`. That reaches `Boss.dispatch_action` (`kitty/boss.py:L1572`), whose inner `report_match` (`kitty/boss.py:L1579`) — gated by `kitty/boss.py:L1580` `if self.args.debug_keyboard:` — prints the matched action at `kitty/boss.py:L1583` (`timed_debug_print(f'{prefix}\x1b[35m{dispatch_type}\x1b[m matched action:', func_name(f), …)`). If the shortcut is consumed, `on_key_input` logs `handled as shortcut` at `kitty/keys.c:L231` **and returns without sending any bytes to the child.**

**Observed proof** (second session; `^[[35m…^[[m` is the literal magenta `\x1b[35m`/`\x1b[m` from `kitty/boss.py:L1583`):

```console
$ xdotool key --clearmodifiers ctrl+shift+l      # and ctrl+shift+equal, ctrl+shift+c
$ grep -aE 'on_key_input|matched action|handled as shortcut' sc.err | cat -v
```

```text
[5.486] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: ctrl+shift text: '' state: 0 
^[[35mKeyPress^[[m matched action: next_layout, handled as shortcut
[6.527] ^[[33mon_key_input^[[m: glfw key: 0x3d native_code: 0x3d action: PRESS mods: ctrl+shift text: '' state: 0 
^[[35mKeyPress^[[m matched action: change_font_size, [6.531] SIGWINCH sent to child in window: 1 with size: (19, 64, 640, 399)
handled as shortcut
[7.569] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl+shift text: '' state: 0 
^[[35mKeyPress^[[m matched action: copy_to_clipboard, handled as shortcut
```

- `ctrl+shift+l` → `matched action: next_layout`; `ctrl+shift+equal` (`0x3d` = `=`) → `matched action: change_font_size` (which additionally resized the child, hence the interleaved `SIGWINCH sent to child …` line); `ctrl+shift+c` (`0x63` = `c`) → `matched action: copy_to_clipboard`. Each is followed by `handled as shortcut` (`kitty/keys.c:L231`).
- The `func_name` printed after `matched action:` is exactly the dispatched action (`next_layout`, `change_font_size`, `copy_to_clipboard`).
- The subsequent key **release** of a consumed shortcut takes a distinct branch and logs `ignoring release event for previous press that was handled as shortcut` (`kitty/keys.c:L239`):

```text
[5.505] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring release event for previous press that was handled as shortcut
```

Because these chords were consumed as shortcuts, **no bytes were sent to the child** for them — consistent with the early `return` at `kitty/keys.c:L231`.

### 4.2 Encoding (deciding the bytes)

If the chord is *not* a shortcut, `on_key_input` encodes it: `kitty/keys.c:L251` `int size = encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key);`. The encoder (`kitty/key_encoding.c:L414`) honors DECCKM cursor‑key mode and the active keyboard‑protocol flags (legacy escape sequences vs. the Kitty Keyboard Protocol). For a plain printable key with text, it returns the sentinel `SEND_TEXT_TO_CHILD` (`kitty/key_encoding.c:L437`); for keys like Enter it returns an encoded byte length.

### 4.3 Writing to the child PTY (the two output branches)

The return value selects the branch, and each branch is traced:

- **Text path** — `size == SEND_TEXT_TO_CHILD`: `kitty/keys.c:L253` `schedule_write_to_child(w->id, 1, text, strlen(text));` then `kitty/keys.c:L254` `debug("sent key as text to child: %s\n", text)`.
- **Encoded path** — `size > 0`: `kitty/keys.c:L259` `schedule_write_to_child(w->id, 1, encoded_key, size);` then `kitty/keys.c:L261` `debug("sent encoded key to child: ")` followed by a per‑byte loop (`kitty/keys.c:L263-L266`) that prints `^[ ` for ESC (`27`), `SPC ` for space, a raw `%c ` for printable bytes, or `0x%x ` otherwise.
- **Unsupported** — otherwise: `kitty/keys.c:L271` `debug("ignoring as keyboard mode does not support encoding this event\n")`.

`schedule_write_to_child` itself lives in `kitty/child-monitor.c:L372` and queues the bytes onto the child PTY.

**Observed proof.** From the `ls` stimulus (`cap.err`):

```text
[5.459] ...action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[5.530] ...action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[6.210] ...glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
[5.492] ...action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

- The printable keys `l` and `s` took the **text** path → `sent key as text to child: l` / `… s` (`kitty/keys.c:L254`).
- **Enter** took the **encoded** path → `sent encoded key to child: 0xd ` — i.e. a single byte `0x0d` (ASCII **CR**), printed through the `else { debug("0x%x ", …); }` branch at `kitty/keys.c:L266` because CR is non‑printable. `0xd` is the exact byte handed to the PTY for Enter under the default (legacy) keyboard mode.
- Every **RELEASE** (and the bare modifier keys `ctrl`/`shift`, which report `glfw key: 0xe062`/`0xe061`) fell through to `ignoring as keyboard mode does not support encoding this event` (`kitty/keys.c:L271`) — nothing is sent for them.

A Python‑side counterpart, `KeyboardHandler.debug_print` (`kitty/keys.py:L242-L245`, gated by `b.args.debug_keyboard`), prints additional keyboard diagnostics when the terminal is in a keyboard‑reporting mode; the simple keys here stayed on the C legacy path above.

### 4.4 The child round‑trip → VT parser → screen model ("before the screen updates")

The bytes written to the PTY are consumed by the child (the default `bash`). Its response is read back by Kitty's I/O loop, whose `parse_func` is `parse_worker` (`kitty/child-monitor.c:L181`) — or `parse_worker_dump` (`kitty/child-monitor.c:L180`) when a dump callback is installed. Adding `--dump-bytes=…` installs exactly such a callback: `kitty/boss.py:L372` passes `DumpCommands(args)` to the `ChildMonitor` when `args.dump_commands or args.dump_bytes`. `DumpCommands` writes the **raw** inbound bytes to the file (`kitty/boss.py:L242-L244`) and prints the **parsed command names** to stdout (`kitty/boss.py:L249,L252`). The parser that turns bytes into those commands is the VT state machine in `kitty/vt-parser.c` (e.g. `dispatch_single_byte_control` `:L224` → `screen_draw_text` `:L226`; control bytes map at `:L96-L102`, where CR → `screen_carriage_return` `:L102` and LF → `screen_linefeed` `:L101`), which mutates the grid model in `kitty/screen.c`.

**Observed proof of the round‑trip.** The raw inbound bytes (`/evidence/bytes.log`, `cat -v`, CR shown as `^M`) show the shell echoing the typed command and then emitting its output:

```text
...root@9f90e42a4223:/app# ...ls^M
...3rdparty ... glfw ... __pycache__^M
gen ... Makefile ... tools^M
...exit^M
```

And the **parsed** command stream on stdout (`cap.out`) shows those bytes becoming screen operations — first the echo of `ls`, then the directory listing being **drawn**:

```console
$ grep -aE '^draw |screen_carriage_return|screen_linefeed|screen_set_mode' cap.out | head
```

```text
screen_set_mode 2004 1
draw root@9f90e42a4223:/app# 
draw ls
screen_carriage_return
screen_linefeed
...
draw 3rdparty
draw Brewfile                go.mod             pyproject.toml
```

This is precisely the "before the screen updates" round‑trip: the outbound `sent key as text/encoded key to child` bytes (`kitty/keys.c:L254`/`:L261`) go to the PTY, the child answers, and the inbound bytes are parsed (`kitty/vt-parser.c`) into `draw …`, `screen_carriage_return`, `screen_linefeed`, … operations that update the screen model (`kitty/screen.c`) — the state the renderer will later push to the GPU (Q3).


---

## 5. Q3 — Display production: how the updated display is produced

**Answer.** Once the screen model has been updated (Q2), a dedicated render/event loop pushes the changed cells to the GPU and composites them with the OpenGL shader stages, then swaps buffers to present the frame.

**Code path.**
- The loop is entered from Python: `kitty/main.py:L234` `boss.child_monitor.main_loop()`, which calls the C `main_loop` (`kitty/child-monitor.c:L1259`) → `kitty/child-monitor.c:L1262` `run_main_loop(process_global_state, self);`.
- Changed cell data is uploaded to the GPU by `send_cell_data_to_gpu(...)` — `kitty/child-monitor.c:L714` (tab bar) and `kitty/child-monitor.c:L766` (window cells) — each of which flags `needs_render = true`.
- Compositing runs through the shader stages in `kitty/shaders.c`: `cell_prepare_to_render` (`:L394`), `draw_background_image` (`:L455`), `draw_graphics` (`:L509`), `draw_cells_simple` (`:L577`), `draw_tint` (`:L596`), and `draw_scroll_indicator` (`:L609`), orchestrated by `draw_cells` (`:L1009`), using the GLSL programs (`cell_*.glsl`, `border_*.glsl`, `bgimage_*.glsl`, `graphics_*.glsl`, `tint_*.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl`, `cell_defines.glsl`).
- Under `--debug-rendering`, GL error checking is installed with `kitty/gl.c:L62` `gladSetGLPostCallback(check_for_gl_error)`; `check_for_gl_error` (`kitty/gl.c:L16`) calls `glad_glGetError()` (`:L18`) after every GL call and aborts with `fatal("OpenGL error: %s (calling function: %s)", …)` (`:L17`) if any error is seen.
- The finished frame is presented by swapping buffers: `kitty/glfw.c:L1221` `glfwSwapBuffers(glfw_window)`.

**Observed proof.** The single most direct, consistently‑appearing rendering signal is the GL version banner, emitted on **stdout** by `kitty/gl.c:L72` (`printf("[%.3f] GL version string: %s\n", …)`, where the string is built by `snprintf(..., "'%s' Detected version: %d.%d", …)` at `kitty/gl.c:L47`). Command and output:

```console
$ grep -a 'GL version string' cap.out
[0.142] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```

- The reported renderer/version is `'4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1'` with `Detected version: 4.5` — i.e., the context created under `LIBGL_ALWAYS_SOFTWARE=1` is Mesa `llvmpipe`'s OpenGL 4.5 core profile, comfortably above the Linux minimum of 3.1 (`kitty/data-types.h:L20,L24`). The same line appeared in every run (`[0.142]`, `[0.126]`, `[0.121]`), varying only in its timestamp — evidence that context creation and the GL path are exercised consistently.
- **No** `OpenGL error …` abort (`kitty/gl.c:L17`), **no** `Loading the OpenGL library failed` (`kitty/gl.c:L57`), **no** `OpenGL version is %d.%d, version >= %d.%d required for kitty` (`kitty/gl.c:L74`), and **no** `Failed to create GLFW temp window!` (`kitty/glfw.c:L1199`) appeared on either stream. Under `--debug-rendering`'s per‑call error checking (`kitty/gl.c:L62`), that silence is the observable evidence that the GL context was created and the render path ran to completion without error.
- The screen‑content evidence from Q2 (`draw ls`, then `draw 3rdparty` … the drawn directory listing) is the model that this loop uploads via `send_cell_data_to_gpu` (`kitty/child-monitor.c:L714,L766`) and composites through the shaders before the `glfwSwapBuffers` present (`kitty/glfw.c:L1221`).

> Note (see §7): `--debug-rendering` does not print a line *per frame*; its observable footprint is the one‑time GL banner plus the installed error‑checking callback. Under `llvmpipe`, the compositing is done in software rather than on a hardware GPU.

---

## 6. End‑to‑end summary

Putting the observed signals in order, a single keystroke travels like this (timestamps are the real `[%.3f]` values from the `l` press and the GL banner):

| # | Stage | Component (`file:line`) | Observable signal |
|---|-------|-------------------------|-------------------|
| 0 | GL context ready (startup) | `kitty/gl.c:L72` | `[0.142] GL version string: '4.5 … Mesa …' Detected version: 4.5` (stdout) |
| 1 | OS/display server → backend callback | `kitty/glfw.c:L430`, registered `:L1292` | (delivers the X11 event) |
| 2 | Backend → Kitty entry point | `kitty/glfw.c:L439` → `kitty/keys.c:L166` | `[5.459] on_key_input: … text: 'l' …` (stderr) |
| 3a | Shortcut? dispatch to Boss | `kitty/keys.c:L228` → `kitty/boss.py:L1583` | `KeyPress matched action: next_layout` + `handled as shortcut` (`kitty/keys.c:L231`) |
| 3b | Else encode | `kitty/keys.c:L251` → `kitty/key_encoding.c:L414` | (returns `SEND_TEXT_TO_CHILD` or a byte length) |
| 4 | Write to child PTY | `kitty/keys.c:L253/L259` → `kitty/child-monitor.c:L372` | `sent key as text to child: l` / `sent encoded key to child: 0xd ` |
| 5 | Child answers; read + parse | `kitty/child-monitor.c:L181` → `kitty/vt-parser.c` | raw `ls^M` (bytes.log); `draw ls`, `screen_carriage_return`, `screen_linefeed` (cap.out) |
| 6 | Update grid model | `kitty/screen.c` | `draw 3rdparty` … (the drawn `ls` output) |
| 7 | Render loop | `kitty/main.py:L234` → `kitty/child-monitor.c:L1259,L1262` | (drives frames) |
| 8 | Upload cells to GPU | `kitty/child-monitor.c:L714,L766` | sets `needs_render = true` |
| 9 | Composite via shaders | `kitty/shaders.c:L394…L1009` (+ `*.glsl`) | (no GL error under `--debug-rendering`) |
| 10 | Present frame | `kitty/glfw.c:L1221` | buffers swapped → display |

The `[%.3f]` timestamps also give a rough input‑to‑effect latency: the `l` press was traced at `[5.459]` and its outbound write on the same line; the whole `ls`+Enter interaction completed within a few hundred milliseconds of trace time under software GL.


---

## 7. What could not be verified (and honest caveats)

- **Software GL, not a hardware GPU.** All rendering used Mesa `llvmpipe` (`OpenGL renderer string: llvmpipe (LLVM 19.1.1, 256 bits)`) under `Xvfb` with `LIBGL_ALWAYS_SOFTWARE=1`. The GPU‑specific timing and any driver‑specific behavior of `send_cell_data_to_gpu`/the shader stages could not be observed; the observed `Detected version: 4.5` and the *absence* of GL errors are what confirm the render path ran. Real‑GPU frame timing is therefore **not** verified here.
- **No per‑frame render log exists to quote.** `--debug-rendering` installs GL error checking (`kitty/gl.c:L62`) and prints the one‑time GL banner (`kitty/gl.c:L72`); it does not emit a line for each composited frame. So Q3's frame production is evidenced *indirectly* (the GL banner + the drawn `ls` output + no `fatal(...)`), not by a literal "frame N rendered" trace.
- **No pixel screenshot.** The image tools `import`, `xwd`, `convert`, `scrot`, and `ffmpeg` are absent from the container and could not be installed offline, so a framebuffer screenshot of the rendered window was not captured. The "screen updated" claim rests on the parsed `draw …` command stream (`cap.out`) and the raw child bytes (`bytes.log`), not on a captured image.
- **`glfwSwapBuffers` is not individually traced.** The buffer‑swap at `kitty/glfw.c:L1221` has no debug print; it is cited from source, and its precondition (a valid GL context) is what the GL banner demonstrates at runtime.
- **The AAP's "OpenGL 3.3+" is Apple‑only.** On this Linux host the compiled minimum is **3.1** (`kitty/data-types.h:L24`, `GLSL_VERSION 140` at `:L26`); `3.3` (`:L22`) applies under `#ifdef __APPLE__`. This document quotes the runtime‑detected `4.5` rather than asserting a fixed minimum.
- **Encoding internals inferred from the outcome.** `encode_glfw_key_event` (`kitty/key_encoding.c:L414`) is not itself traced; its effect is inferred from the branch taken (`sent key as text …` vs `sent encoded key … 0xd`). The default legacy mode was in force for the simple keys pressed; the Kitty Keyboard Protocol path was not exercised by this stimulus.

## 8. Coverage checklist (closing pass)

- [x] **Verbatim question reproduced** — at the top of the document as a blockquote.
- [x] **Q1 — Input ingress — answered explicitly (§3):** OS/display server → `key_callback` (`kitty/glfw.c:L430`, registered `:L1292`) → `on_key_input` (`kitty/keys.c:L166`), proven by the verbatim `on_key_input:` traces (`kitty/keys.c:L176`) each paired with the `xdotool` command that produced them.
- [x] **Q2 — Intermediate processing — answered explicitly (§4):** shortcut dispatch (`kitty/boss.py:L1583` `matched action:` + `kitty/keys.c:L231` `handled as shortcut`), encoding (`kitty/keys.c:L251` → `kitty/key_encoding.c:L414`), PTY write with both branches (`sent key as text to child: l/s` `kitty/keys.c:L254`; `sent encoded key to child: 0xd ` `kitty/keys.c:L261,L266`), the ignore branches (`:L271`, `:L239`), and the child round‑trip through `parse_worker`/`vt-parser.c` into `screen.c` (raw `ls^M` + `draw …` commands) — all quoted verbatim.
- [x] **Q3 — Display production — answered explicitly (§5):** render loop (`kitty/main.py:L234` → `kitty/child-monitor.c:L1259,L1262`), GPU upload (`kitty/child-monitor.c:L714,L766`), shader compositing (`kitty/shaders.c`), buffer swap (`kitty/glfw.c:L1221`), proven by the verbatim `GL version string:` banner (`kitty/gl.c:L72`) and the absence of GL `fatal(...)`.
- [x] **End‑to‑end ordering (§6)** — a single ordered table from OS event to buffer swap, keyed to the observed timestamps.
- [x] **Consistency** — the `ls`+Enter key traces were byte‑for‑byte identical across two runs; the GL banner appeared in all runs.
- [x] **What could not be verified (§7)** — software‑GL caveat, no per‑frame log, no pixel screenshot, and the 3.1‑vs‑3.3 correction, all stated honestly.

### Reproduction note

The build/run/capture was performed inside the provided Docker image (`kitty-build-env:ready`, from `ghcr.io/scaleapi/swe-atlas`) because the analysis host lacks the C/Go toolchain and native GL/font libraries; a bare `python3 setup.py` there fails with `FileNotFoundError: [Errno 2] No such file or directory: 'pkg-config'`. All scratch files used to gather this evidence lived outside the repository and were removed afterward, leaving this document as the only change.

