# How keyboard input flows through Kitty's core components — a runtime‑grounded walkthrough

**Repository:** `kovidgoyal/kitty` at HEAD commit `815df1e21` ("Wire up applying of font config").
**Method:** Every claim below was produced by **building and running Kitty from this repository**, launching it with its **own** debug/tracing flags, pressing keys through the **real** keyboard path, and copying the **actual, unedited** output next to each claim. Statements that could not be directly observed at runtime are explicitly labelled **`inferred`** and carry a `file:line` citation instead. File:line references point at the source in this checkout.

---

## 1. One‑paragraph summary

When you press a key in the shell running inside Kitty, the keystroke is **received first by the platform/GLFW layer** — on Linux/X11 the XKB keymap turns the hardware scan‑code into a symbol and hands a key event to Kitty's GLFW callback `key_callback` (`kitty/glfw.c:430`), which immediately calls the first Kitty‑owned handler `on_key_input` (`kitty/keys.c:166`). That handler performs the **intermediate processing**: it asks the Python layer whether the key is a shortcut (`dispatch_possible_special_key`, `kitty/keys.py:154`), then **encodes** the key — either as a legacy byte/escape sequence or as a Kitty‑Keyboard‑Protocol *CSI‑u* sequence (`encode_glfw_key_event`, `kitty/key_encoding.c:414`) — and **writes those bytes to the child** through the pseudo‑terminal (`schedule_write_to_child`, `kitty/child-monitor.c:372`). The child shell echoes bytes back; Kitty's **I/O thread** reads them (`io_loop` → `read_bytes` → `do_parse`, `kitty/child-monitor.c:1481/1337/438`), a **VT state machine** classifies each byte (`consume_normal`, `kitty/vt-parser.c:230`), and printable text is written into the **screen model** (`screen_draw_text`, `kitty/screen.c:866`). Finally the **display is produced on the GPU**: the main‑thread render tick (`render`, `kitty/child-monitor.c:871`) composites the cells through the OpenGL shader pipeline and presents the frame with `swap_window_buffers` (`kitty/child-monitor.c:810`). **Important caveat:** the read/parse work and the rendering work run on **separate, concurrent threads** — the "receive → process → display" ordering below is a *logical* description, not a strict serial pipeline (`docs/performance.rst:8`).

---

## 2. Environment & method

### 2.1 Build (canonical, from this repository)

Command actually run (the `--ignore-compiler-warnings` flag is Kitty's own `setup.py` switch, not a source edit; it is required in this toolchain because a newer bundled `wayland-protocols` header trips `-Werror=switch` in the vendored GLFW):

```console
$ export PATH=/usr/local/go/bin:$PATH
$ ./dev.sh build --ignore-compiler-warnings
Build successful. Run kitty as: kitty/launcher/kitty
```

Exit status `0`. `dev.sh` is a one‑line wrapper — `exec go run bypy/devenv.go "$@"` (`dev.sh`) — so `./dev.sh build` downloads the prebuilt dependency bundle and compiles the C core, the Python package, and the Go tooling, exactly as documented (`docs/build.rst:16-22`). The canonical launcher artifact is **`kitty/launcher/kitty`** (`docs/build.rst:22`). Prerequisites observed in the environment: `go version go1.22.12 linux/amd64` (satisfies `go 1.22` in `go.mod:3`) and `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` (a C11 compiler); Python `>=3.8` is the documented runtime floor (`pyproject.toml:2`). The build above was **incremental** (the artifact already existed), which is why the transcript is a single success line.

### 2.2 Version banner (verbatim)

```console
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The bytes were confirmed with `od -c` to be exactly `kitty 0.35.2 created by Kovid Goyal\n` — plain ASCII with **no** ANSI styling and **no** git‑rev when the output is piped (non‑TTY). The banner is assembled by `version()` (`kitty/cli.py:486-492`, format string `'{} {}{} created by {}'`); the `0.35.2` comes from `version: Version = Version(0, 35, 2)` (`kitty/constants.py:25`). (On an interactive TTY the same function adds italic/green SGR styling; the `rev` field is empty because `add_rev` defaults to `False`.)

### 2.3 Headless display

Kitty renders exclusively through OpenGL with **no CPU fallback**, so a display/GPU is required. The environment is headless, so a virtual X display was used:

```console
$ Xvfb :99 -screen 0 1280x800x24 +extension GLX +render -noreset &
$ export DISPLAY=:99
```

Mesa's software GL (llvmpipe) provides OpenGL 4.5 here, which satisfies Kitty's GL ≥ 3.3 requirement (observed in §5).

### 2.4 Launch with Kitty's own tracing flags

```console
$ export DISPLAY=:99
$ stdbuf -oL -eL ./kitty/launcher/kitty --debug-input --debug-rendering \
        --dump-bytes /tmp/kdump.bin > /tmp/kitty_stdout.log 2>/tmp/kitty_stderr.log &
```

The default shell under Kitty is `/bin/bash`, and there is no user `~/.config/kitty/kitty.conf`, so this is Kitty's **default, canonical configuration**. `stdbuf -oL -eL` is only a *flushing* aid: the GL diagnostic (§5) is emitted with `printf` and is block‑buffered when stdout is redirected to a file, so line‑buffering makes it appear promptly. It does **not** change Kitty's behavior.

These flags are Kitty's native, documented tracing hooks (all in `kitty/cli.py`):

| Flag | Effect (verbatim intent) | Declared at | Where its output goes |
|---|---|---|---|
| `--debug-input` / `--debug-keyboard` | "Print out key and mouse events as they are received." | `kitty/cli.py:996` | **STDERR** |
| `--debug-rendering` / `--debug-gl` | "Debug rendering commands … check for errors … prints out miscellaneous debug information." | `kitty/cli.py:989` | **STDERR** (macro traces) + **STDOUT** (GL‑version line) |
| `--dump-bytes <path>` | "Path to file in which to store the raw bytes received from the child process." | `kitty/cli.py:985` | the **named file** |
| `--dump-commands` | "Output commands received from child process to STDOUT." | `kitty/cli.py:972` | STDOUT |

**Why these flags are observation, not a bypass.** They are wired into the *genuine* input pipeline at GLFW‑init time: `init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)` (`kitty/main.py:514`) → `init_glfw_module(...)` (`kitty/main.py:90`) → `glfw_init(..., debug_keyboard, debug_rendering, ...)` (`kitty/main.py:91`). Enabling them only **annotates** the real path; it does not reroute input. All keys below were pressed through the real keyboard path using `xdotool` (X `XTEST`, i.e. real X‑server key events delivered to the focused window) — never through remote control (`kitty @ send-text`) or any synthetic injection into Kitty's internals.

**Stream‑attribution mechanics (verified in source).** The `--debug-input` reception traces and the macro‑based `--debug-rendering` traces are printed by `timed_debug_print` (`kitty/monotonic.h:99`), whose body calls `fprintf(stderr, "[%.3f] ", …)` then `vfprintf(stderr, …)` — hence every such line goes to **STDERR** and is prefixed with a `[seconds]` timestamp. The `debug(...)` macro resolves to `debug_input` in `kitty/keys.c` (`kitty/keys.h:16`, gated on `OPT(debug_keyboard)`) and to `debug_rendering` in `kitty/glfw.c` (`kitty/glfw.c:34`, gated on `global_state.debug_rendering`); both expand to `timed_debug_print` (`kitty/state.h:14-15`). The single exception is the GL‑version line, printed with `printf(...)` → **STDOUT** (`kitty/gl.c:72`).

> In the raw captures below, `^[` is the ESC byte (`0x1b`) as rendered by `cat -v`; sequences such as `^[[33m … ^[[m` are ANSI SGR color codes that the debug prints wrap around their labels (e.g. `kitty/keys.c:176` emits `"\x1b[33mon_key_input\x1b[m: …"`). They are part of the genuine, unedited output.

---

## 3. Stage 1 — Reception: which parts receive the input first

**Answer:** the **platform/GLFW layer receives the keystroke first** — concretely, on Linux the XKB code inside the vendored GLFW fork translates the hardware key and hands a GLFW key event to Kitty's `key_callback`, which immediately forwards it to `on_key_input`, the first Kitty‑owned function to see the event.

**Observed evidence.** Pressing the letter `l` in the shell (command: `xdotool key --clearmodifiers l`) produced this pair of consecutive STDERR lines with the **same timestamp** `[107.882]`:

```text
[107.882] ^[[31mPress^[[m xkb_keycode: 0x2e clean_sym: l composed_sym: l text: l mods: none glfw_key: 108 (l) xkb_key: 108 (l)
[107.882] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
```

- The **first** line is emitted by the XKB layer of the platform code — `debug("%s xkb_keycode: 0x%x ", …)` at `glfw/xkb_glfw.c:875` — as it decodes the raw key (`xkb_keycode: 0x2e`) into a symbol (`clean_sym: l`). This is the Linux/X11 keymap feeder for GLFW.
- The **second** line is emitted by `on_key_input` (`kitty/keys.c:166`) under the guard `if (OPT(debug_keyboard))` (`kitty/keys.c:172`); the label/format string is at `kitty/keys.c:176`. That the two lines share a timestamp and appear back‑to‑back, XKB line **before** the Kitty line, is the empirical proof that the platform/GLFW layer receives the event first and *then* calls into Kitty.

**How the event reaches `on_key_input` (the call chain).** GLFW's keyboard callback is registered once at window creation — `glfwSetKeyboardCallback(glfw_window, key_callback)` (`kitty/glfw.c:1292`) — and `key_callback` (`kitty/glfw.c:430`) forwards the event with `on_key_input(ev)` (`kitty/glfw.c:439`, guarded so callbacks only fire once the window is ready). The startup trace also shows the platform keymap being loaded before any key is pressed:

```text
[0.057] Loading new XKB keymaps
[0.062] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.146] OS Window created
[0.160] Child launched
[0.160] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
```

("Loading new XKB keymaps" is `glfw/xkb_glfw.c:672`; "Modifier indices" is `glfw/xkb_glfw.c:376/540`.) On macOS/Cocoa and Wayland the feeder differs (Cocoa, or `glfw/ibus_glfw.c` for IME), but the entry into Kitty is the same `key_callback` → `on_key_input` chain — **`inferred`** for the non‑X11 platforms, since only Linux/X11 was exercised here.

**Real‑path guarantee.** The `xkb_keycode: 0x2e` in the trace is a genuine hardware key‑code produced by the X server for the `l` key; it is present precisely because the event travelled the real OS → XKB → GLFW route (the debug flags only annotate that route — see §2.4).

---

## 4. Stage 2 — Intermediate processing: what happens between reception and the screen update

Two byte streams cross the pseudo‑terminal (PTY) boundary, and both are part of the intermediate stage:

### 4.1 Terminal → child (the keystroke is interpreted, encoded, and written to the shell)

Inside `on_key_input`, three things happen in order, all observable on one continued STDERR line per key:

1. **Shortcut check (Python).** `on_key_input` calls into the Python layer via `dispatch_possible_special_key` (`kitty/keys.py:154`) to see if the key is a mapped shortcut. When it is not consumed as a shortcut, processing continues to encoding. (The Python side has its own debug branch at `kitty/keys.py:244`.)
2. **Encoding (legacy vs. protocol).** `encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)` (`kitty/keys.c:251`, implemented in `kitty/key_encoding.c:414`) decides how the key is serialized. Its return value drives one of three dispositions.
3. **Write to child (through the PTY).** The bytes are queued with `schedule_write_to_child` (`kitty/keys.c:254` / `kitty/keys.c:266`, implemented at `kitty/child-monitor.c:372`).

The three dispositions were all observed, each with its own trailing debug string appended to the reception line:

```text
# plain letter 'l'  ->  sent to the child as literal text  (kitty/keys.c:254)
[107.882] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l

# Enter  ->  ENCODED to a single carriage-return byte 0x0d  (kitty/keys.c:261)
[108.413] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd

# key RELEASE in the shell's legacy mode  ->  not encoded at all  (kitty/keys.c:271)
[107.883] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

So an ordinary letter is delivered to the shell as its literal text byte (`l` = `0x6c`), Enter is *encoded* to `0x0d`, and key releases are dropped (in the shell's default/legacy keyboard mode). The strings `sent key as text to child`, `sent encoded key to child`, and `ignoring as keyboard mode does not support encoding this event` are printed at `kitty/keys.c:254`, `:261`, and `:271` respectively.

*(There is also a Python‑level API for the same terminal→child hop — `send_key` → `send_key_sequence` → `write_to_child` (`kitty/window.py:917/937/955`), which ultimately calls `child_monitor.needs_write`. For a physically pressed key the observed path is the C‑side `on_key_input` → `schedule_write_to_child`; the `window.py` chain is the equivalent programmatic entry point.)*

### 4.2 Child → terminal (bytes are read back, parsed, and applied to the screen model)

The shell echoes what you typed (and prints command output). Those bytes are exactly what `--dump-bytes` captures, which is why that file is the empirical fingerprint of this hop.

**Before / after (the changing value).** Polling the dump file every 50 ms immediately after launch showed it transition from **absent (0 bytes)** to **322 bytes**:

```text
poll 1-4 : /tmp/kdump.bin  ->  absent (0 bytes; kitty has not opened it / no child output yet)
poll 5+  : /tmp/kdump.bin  ->  322 bytes (steady)
```

The file is opened empty with mode `'wb'` by `DumpCommands` (`kitty/boss.py:237`, installed at `kitty/boss.py:372`); the child shell then writes its 322‑byte kitty‑integration startup prompt into it within ~200 ms. After typing `ls` + Enter it grew by **834 bytes**, beginning with the echo of the typed characters:

```text
# first bytes appended after typing 'ls'<Enter>  (hexdump -C of the delta)
00000000  6c 73 0d 0a 1b 5b 3f 32  30 30 34 6c 0d 1b 5d 32  |ls...[?2004l..]2|
00000010  3b 6c 73 07 1b 5d 31 33  33 3b 43 3b 63 6d 64 6c  |;ls..]133;C;cmdl|
00000020  69 6e 65 3d 6c 73 07 ...                          |ine=ls...       |
```

`6c 73 0d 0a` is `l s CR LF` — the shell echoing the two letters and the newline — followed by control sequences (`\x1b[?2004l` bracketed‑paste‑off, an OSC‑2 title `]2;ls`, an OSC‑133 `]133;C;cmdline=ls` shell‑integration mark) and then the actual directory listing. `DumpCommands.__call__` writes and flushes on `what == 'bytes'` (`kitty/boss.py:242-244`).

**Read → parse → screen (the functions doing the work).** The child's output is read by the I/O thread loop `io_loop` (`kitty/child-monitor.c:1481`), which calls `read_bytes` (`kitty/child-monitor.c:1337`) and then `do_parse` (`kitty/child-monitor.c:438`); when byte‑dumping is enabled the parse worker is `parse_worker_dump` (`kitty/child-monitor.c:180`, vs `parse_worker` at `:181`). The VT state machine classifies each byte: printable text goes through `consume_normal` (`kitty/vt-parser.c:230`), escapes through `consume_esc` (`:261`), and CSI sequences through `consume_csi` (`:839`, dispatched by `dispatch_csi` at `:1027`). Printable text is written into the screen model by `screen_draw_text` (`kitty/screen.c:866`) / `draw_codepoint` (`kitty/screen.c:872`), which is backed by the line/cell buffers (`kitty/line.c`, `kitty/line-buf.c`), scrollback (`kitty/history.c`), and cursor (`kitty/cursor.c`). That the dumped bytes (e.g. the `ls` echo above) match what a terminal must parse to update its grid is the observed evidence for this read → parse → screen hop; the specific per‑function transitions inside the VT parser are **`inferred`** from the source (the parser itself has no per‑byte trace flag exercised here).


---

## 5. Stage 3 — Display production: how the updated display is ultimately produced

**Answer:** the updated screen is produced by a **main‑thread render tick that composites the screen model on the GPU via OpenGL and presents the frame by swapping the window's buffers** — there is no CPU rendering fallback.

**Observed evidence.** With `--debug-rendering`, the following line appeared on **STDOUT** at startup and reproduced on every launch:

```text
[0.124] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

This is printed by `printf("[%.3f] GL version string: %s\n", …)` inside `gl_init` (`kitty/gl.c:72`, guarded by `global_state.debug_rendering`). Its presence proves Kitty successfully created an **OpenGL** context (here Mesa's software GL 4.5, which meets Kitty's GL ≥ 3.3 requirement — `gl_init` calls `fatal(...)` if the version is too low, `kitty/gl.c:74`) and that the GPU/OpenGL path — not a CPU path — is what draws the terminal.

**The render call chain (functions doing the work).** The main thread runs `main_loop` (`kitty/child-monitor.c:1259`), which calls `render(now, input_read)` (`kitty/child-monitor.c:1237`); `render` is defined at `kitty/child-monitor.c:871`. It composites each OS window through `render_os_window` (`kitty/child-monitor.c:833`), which prepares the frame with `prepare_to_render_os_window` (`kitty/child-monitor.c:705`) and draws it with `render_prepared_os_window` (`kitty/child-monitor.c:788`), then presents it with **`swap_window_buffers(os_window)`** (`kitty/child-monitor.c:810`). The actual pixel work is done by the OpenGL infrastructure in `kitty/gl.c` and the shader programs in `kitty/shaders.c` together with the **13** GLSL shader sources in `kitty/` (verified with `ls kitty/*.glsl | wc -l` → `13`): `alpha_blend`, `bgimage_fragment`, `bgimage_vertex`, `border_fragment`, `border_vertex`, `cell_defines`, `cell_fragment`, `cell_vertex`, `graphics_fragment`, `graphics_vertex`, `linear2srgb`, `tint_fragment`, `tint_vertex`. The precise sequence of GL draw calls per frame is **`inferred`** from the source; what was *observed* is the successful GL context creation (the line above) plus a live, updating window (the shell prompt, the echoed `ls`, and its output all appeared on screen).

### 5.1 Concurrency caveat (do not read this as a strict serial pipeline)

The three stages above are a **logical** ordering. In reality Kitty's Child Monitor uses multiple threads: the child's bytes are read and parsed on an **I/O thread** (`io_loop`, `kitty/child-monitor.c:1481`) while frames are drawn on the **main thread** (`render`, `kitty/child-monitor.c:871`). The project documentation states this directly: "Interaction with child programs takes place in a separate thread from … rendering" (`docs/performance.rst:8`), and the two are decoupled by small artificial delays — `input_delay` (default **3 ms**, `docs/performance.rst:48`) and `repaint_delay`. Consistent with that design, the reception/parse traces (STDERR, I/O side) and the render diagnostic (STDOUT, main side) were observed as **concurrent**, not strictly interleaved per keystroke. So: receive → process → display describes the *flow of a keystroke's data*, but the components run **concurrently**.

---

## 6. Conditions exercised (raw output beside each)

All keys were injected with `xdotool` (X `XTEST`) into the focused Kitty window `2097164`; the real hardware `xkb_keycode` visible in each trace confirms the genuine OS → GLFW route.

### 6.1 Condition 1 — Unmodified simple letters (`l`, `s`, Enter)

Command: `xdotool key --clearmodifiers l` / `s` / `Return`.

```text
[107.882] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[107.883] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[108.144] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[108.413] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
```

Letters go to the child as literal text; Enter (`glfw key: 0xe001`, native X keysym `0xff0d`) is encoded to `0x0d`. Dump delta = 834 bytes beginning `6c 73 0d 0a` (`ls\r\n`, see §4.2).

### 6.2 Condition 2 — Modifier combinations (Ctrl‑C, Shift‑a, Alt‑a)

Command: `xdotool key --clearmodifiers ctrl+c` / `shift+a` / `alt+a`. The `mods` field changes and the encoded bytes differ:

```text
# Ctrl-C : the modifier key itself is ignored, then 'c' with mods:ctrl encodes to 0x03 (SIGINT / ETX)
[218.535] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[218.541] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3

# Shift-a : composed to 'A' and sent as literal text
[218.971] ^[[31mPress^[[m xkb_keycode: 0x26 clean_sym: a composed_sym: A text: A mods: shift glfw_key: 97 (a) xkb_key: 97 (a) shifted_key: 65 (A)
[218.972] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A

# Alt-a : encoded as ESC-prefixed (0x1b 0x61) — printed "^[ a"
[219.402] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ a
```

So `mods` takes values `none` (Condition 1) vs `ctrl` / `shift` / `alt`, and the emitted bytes are `0x03` (Ctrl‑C), literal `A` (Shift‑a), and `0x1b 0x61` (Alt‑a). The dump showed the shell echoing `^C` (`5e 43`) and `A` (`41`); Alt‑a was consumed by readline and not echoed. *(One capture had the string `ALSA lib confmisc.c:855:(parse_card) cannot find card '0'` interleaved into stderr — that is benign environmental audio‑bell noise from the headless box reacting to a `BEL`, not a Kitty trace.)*

### 6.3 Condition 3 — Press / release / repeat lifecycle (before / during / after of one key)

Command: `xdotool keydown j; sleep 0.8; xdotool keyup j` (with `xset r rate 250 30`). A single held key produced **1 × PRESS**, **18 × REPEAT**, **1 × RELEASE**:

```text
[259.535] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: PRESS  mods: none text: 'j' state: 0 sent key as text to child: j
[259.785] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
   ... (18 REPEAT lines total, ~33 ms apart) ...
[260.346] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[260.352] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

```text
# action-field tally for the burst
      1 action: PRESS
     18 action: REPEAT
      1 action: RELEASE
```

The `action:` field (`kitty/keys.c:177-178`, the `GLFW_RELEASE/GLFW_PRESS/…REPEAT` ternary) thus takes all three values — the full lifecycle of one key. Each PRESS/REPEAT sends `j`; the RELEASE is dropped in legacy mode.

### 6.4 Condition 4 — Legacy vs. Kitty Keyboard Protocol (same physical key `a`)

The default shell uses **legacy** encoding (the canonical, primary case). To exercise the **Kitty Keyboard Protocol** without a bypass, `kitten show-key -m kitty` was run *inside* the window; it opts in via progressive enhancement (`CSI > 1 u` at startup, `docs/keyboard-protocol.rst:67`; `CSI < u` at exit, `:72`; encoded form `CSI number ; modifiers [u~]`, `:80`). Keys were still pressed on the real keyboard. The **same physical key `a`** encodes completely differently:

```text
# LEGACY (default /bin/bash shell)
[341.839] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: none text: 'a' state: 0 sent key as text to child: a
[341.845] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: ''  state: 0 ignoring as keyboard mode does not support encoding this event
[342.163] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: ctrl text: ''  state: 0 sent encoded key to child: 0x1

# KITTY KEYBOARD PROTOCOL (inside `kitten show-key -m kitty`)
[300.019] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: none text: 'a' state: 0 sent encoded key to child: ^[ [ 9 7 ; ; 9 7 u
[300.025] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: ''  state: 0 sent encoded key to child: ^[ [ 9 7 ; 1 : 3 u
[300.543] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: ctrl text: ''  state: 0 sent encoded key to child: ^[ [ 9 7 ; 5 u
```

| Same key `a` | Legacy (default shell) | Kitty Keyboard Protocol (`show-key -m kitty`) |
|---|---|---|
| press | literal `a` (`0x61`) | `\x1b[97;;97u` (CSI‑u; `97` = codepoint of `a`, with associated text `97`) |
| release | **not encoded** (dropped) | `\x1b[97;1:3u` (release **is** reported; event‑type `:3`) |
| Ctrl‑`a` | `0x01` (SOH) | `\x1b[97;5u` (modifier field `5` = Ctrl) |

In protocol mode even the Control key *itself* is reported (`^[ [ 5 7 4 4 2 ; 5 u`). The branch is chosen by `encode_glfw_key_event` from `screen_current_key_encoding_flags(screen)` (`kitty/keys.c:251`, `kitty/key_encoding.c:414`); the protocol spec is `docs/keyboard-protocol.rst`. **Legacy is the canonical primary evidence** because it is what the default shell uses.

### 6.5 Condition 5 — `--dump-bytes` before / during / after

```text
before launch                : /tmp/kdump.bin  absent      (0 bytes)
after launch, before typing  : /tmp/kdump.bin  322 bytes   (shell startup prompt; kitty/boss.py:237 opens it 'wb')
after typing all keys        : /tmp/kdump.bin  3642 bytes
```

The bytes appended right after typing begin `6c 73 0d 0a` = `ls\r\n` (the shell **echo**), confirming that `--dump-bytes` captures the **child → terminal (output/echo) stream** — precisely the bytes that feed the VT parser and update the screen (mechanism: `kitty/boss.py:242-244`). Full hex of the first delta is in §4.2.

### 6.6 Condition 6 — No active window (edge branch) — `inferred`, not observed

If a key event arrives with no active window, `on_key_input` logs `no active window, ignoring` and returns (`kitty/keys.c:182`). This branch was **not** triggered during the investigation — `grep -c "no active window, ignoring"` over the captured stderr returned `0`, because there was always an active window. It is therefore reported as **`inferred`** from the source, not observed.


---

## 7. Stability (reproduced across two fresh launches)

The identical input `ls`+Enter was driven in **two separate, freshly launched** Kitty processes. The reception traces reproduced exactly — same `glfw key` codes, same native codes, same dispositions, same `mods` — differing only in the `[seconds]` timestamp (expected, from the monotonic clock):

```text
# RUN 1
[107.882] ^[[33mon_key_input^[[m: glfw key: 0x6c ... action: PRESS ... text: 'l' ... sent key as text to child: l
[108.144] ^[[33mon_key_input^[[m: glfw key: 0x73 ... action: PRESS ... text: 's' ... sent key as text to child: s
[108.413] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS ... sent encoded key to child: 0xd

# RUN 2 (fresh process)
[31.696] ^[[33mon_key_input^[[m: glfw key: 0x6c ... action: PRESS ... text: 'l' ... sent key as text to child: l
[31.964] ^[[33mon_key_input^[[m: glfw key: 0x73 ... action: PRESS ... text: 's' ... sent key as text to child: s
[32.232] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS ... sent encoded key to child: 0xd
```

The dump echo reproduced identically (`6c 73 0d 0a` = `ls\r\n`, 834‑byte delta), and the GL diagnostic string matched (`GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`) on both launches. **Result: reproduced across 2 runs.** (Methodology note: a single Kitty instance must own the X focus; two live instances under a WM‑less `Xvfb` caused `on_focus_change … focused: 0` and dropped keys until the environment was reduced to exactly one instance and the window was focused with `xdotool windowfocus` + a real `XTEST` click.)

---

## 8. Reasoning / rationale (observed signal → responsible function)

- **Reception is the platform/GLFW layer.** The XKB line (`glfw/xkb_glfw.c:875`) printing a real `xkb_keycode` **immediately before**, and with the same timestamp as, the `on_key_input` line (`kitty/keys.c:176`) shows the OS/GLFW layer decodes and receives the key first and then calls Kitty's `on_key_input` (`kitty/keys.c:166`), reached from `key_callback` (`kitty/glfw.c:430`) → `on_key_input` (`kitty/glfw.c:439`).
- **Intermediate processing is `on_key_input` deciding an encoding and writing to the PTY, then the I/O thread reading it back and updating the screen model.** The three trailing dispositions (`sent key as text to child` / `sent encoded key to child` / `ignoring …`, `kitty/keys.c:254/261/271`) show `encode_glfw_key_event` (`kitty/key_encoding.c:414`) choosing text vs. escape vs. drop, then `schedule_write_to_child` (`kitty/child-monitor.c:372`) queuing the bytes; the `--dump-bytes` echo (`ls\r\n`) is the child's response that `io_loop`/`read_bytes`/`do_parse` (`kitty/child-monitor.c:1481/1337/438`) feed to the VT parser (`consume_normal`, `kitty/vt-parser.c:230`) and thence to `screen_draw_text` (`kitty/screen.c:866`).
- **Display is produced on the GPU.** The `GL version string` line (`kitty/gl.c:72`) proves an OpenGL context was created; the render tick `render` (`kitty/child-monitor.c:871`) composites and `swap_window_buffers` (`kitty/child-monitor.c:810`) presents. No CPU fallback exists, so a visible, updating window is itself evidence the GPU path ran.
- **Concurrency, not a strict pipeline.** Read/parse (I/O thread) and render (main thread) are separate threads (`docs/performance.rst:8`) coupled by `input_delay`/`repaint_delay` (default `input_delay 3 ms`, `docs/performance.rst:48`); the STDERR (I/O‑side) and STDOUT (render‑side) signals were observed concurrently.

### Notes on entry points and counts (observed reality is authoritative)

- Kitty's Python entry point is the **repository‑root `__main__.py`** (which runs `from kitty.entry_points import main`, defined in `kitty/entry_points.py`); there is **no** `kitty/__main__.py` in this checkout (verified: `ls kitty/__main__.py` → not found).
- There are **13** GLSL shader files in `kitty/` (verified `ls kitty/*.glsl | wc -l` → `13`), listed in §5.
- The version is reported from the real `--version` output (`kitty 0.35.2 created by Kovid Goyal`); the source constant is `Version(0, 35, 2)` (`kitty/constants.py:25`).

---

## 9. Coverage pass (all three named sub‑questions answered)

- [x] **Reception — "which parts appear to receive the input first"** → §3. Answer: the OS/platform + GLFW layer (XKB feeder `glfw/xkb_glfw.c:875`; GLFW callback `key_callback`, `kitty/glfw.c:430`) receive first, then Kitty's `on_key_input` (`kitty/keys.c:166`). Backed by the same‑timestamp XKB‑before‑`on_key_input` capture.
- [x] **Intermediate processing — "which parts handle the intermediate processing"** → §4. Answer: `on_key_input` interprets/encodes the key (`dispatch_possible_special_key` `kitty/keys.py:154`; `encode_glfw_key_event` `kitty/key_encoding.c:414`) and writes it to the child via the PTY (`schedule_write_to_child` `kitty/child-monitor.c:372`); the echoed bytes are read back (`io_loop`/`read_bytes`/`do_parse` `kitty/child-monitor.c:1481/1337/438`), parsed (`consume_normal` `kitty/vt-parser.c:230`), and applied to the screen model (`screen_draw_text` `kitty/screen.c:866`). Backed by the trailing dispositions and the `--dump-bytes` before/after echo.
- [x] **Display production — "how the updated display is ultimately produced"** → §5. Answer: the main‑thread render tick (`render` `kitty/child-monitor.c:871`) composites on the GPU (OpenGL, `kitty/gl.c`; shaders in `kitty/shaders.c` + 13 `*.glsl`) and presents via `swap_window_buffers` (`kitty/child-monitor.c:810`). Backed by the captured `GL version string` diagnostic (`kitty/gl.c:72`).
- [x] **Concurrency caveat** stated (§5.1, §8): receive → process → display is the logical data flow; the components run **concurrently** on separate threads (`docs/performance.rst:8`, `:48`).

---

### Appendix — exact commands used for observation (all temporary; removed after the investigation)

```sh
# build
export PATH=/usr/local/go/bin:$PATH
./dev.sh build --ignore-compiler-warnings
./kitty/launcher/kitty --version

# headless display
Xvfb :99 -screen 0 1280x800x24 +extension GLX +render -noreset &
export DISPLAY=:99

# launch with Kitty's own trace flags (default /bin/bash shell)
stdbuf -oL -eL ./kitty/launcher/kitty --debug-input --debug-rendering \
    --dump-bytes /tmp/kdump.bin > /tmp/kitty_stdout.log 2>/tmp/kitty_stderr.log &

# drive REAL keys (X XTEST) into the focused window
xdotool windowfocus --sync <win>; xdotool mousemove --sync 100 100; xdotool click 1
xdotool key --clearmodifiers l ; xdotool key --clearmodifiers s ; xdotool key --clearmodifiers Return
xdotool key --clearmodifiers ctrl+c ; xdotool key --clearmodifiers shift+a ; xdotool key --clearmodifiers alt+a
xdotool keydown j ; sleep 0.8 ; xdotool keyup j          # PRESS/REPEAT/RELEASE
xdotool type "kitten show-key -m kitty" ; xdotool key Return   # opt into the Kitty Keyboard Protocol

# inspect captured signals
cat /tmp/kitty_stdout.log        # GL version diagnostic (STDOUT)
cat /tmp/kitty_stderr.log        # on_key_input reception + encoding traces (STDERR)
hexdump -C /tmp/kdump.bin        # child -> terminal byte stream
```

*This document is the sole artifact added to the repository; no Kitty source file was modified, and all temporary scripts, logs, and dump files created during the investigation were deleted afterward.*

