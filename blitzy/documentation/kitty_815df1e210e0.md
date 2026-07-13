# How keyboard input flows through Kitty's core components — a runtime‑grounded walkthrough

**Source tree under investigation:** `kovidgoyal/kitty` at commit `815df1e210e0` ("Wire up applying of font config"). That commit is the **source baseline** this document describes; the answer file itself is added on top of that tree on the working branch, so it is *not* the tree's own HEAD commit. Every `file:line` reference below points at the source as it exists in this `815df1e210e0` checkout.

**Method.** Every claim was produced by **building and running Kitty from this repository**, launching it with its **own** debug/tracing flags, driving keys through the **real** keyboard path (X11 `XTEST`, i.e. synthetic X‑server key events that travel the genuine X11 → XKB → GLFW → Kitty software path — see §2.5), and copying the **actual, unedited** output next to each claim. Nearly all runtime captures below come from a **single canonical run** of the observation harness in the Appendix (which builds nothing itself — it only runs the already‑built launcher); a few explicitly‑labelled supplementary captures come from separate, equally‑genuine runs (the two build transcripts in §2.1, and the legacy Ctrl‑`a` trace in §6.4 — recognisable by its distinct timestamp base). Statements that could not be directly observed at runtime are explicitly labelled **`inferred`** and carry a `file:line` citation instead.

---

## 1. One‑paragraph summary

When you press a key in the shell running inside Kitty, the keystroke is **received first by the platform/GLFW layer** — on Linux/X11 the vendored GLFW fork's XKB code turns the hardware keycode into a symbol (`glfw/xkb_glfw.c:875`) and hands a GLFW key event to Kitty's callback `key_callback` (`kitty/glfw.c:430`), which immediately calls the first Kitty‑owned handler `on_key_input` (`kitty/keys.c:166`). That handler performs the **intermediate processing** on Kitty's **main thread**: it asks the Python layer whether the key is a mapped shortcut (`dispatch_possible_special_key`, dispatched only for PRESS/REPEAT at `kitty/keys.c:226`/`228`), then **encodes** the key — either as a legacy byte/escape sequence or as a Kitty‑Keyboard‑Protocol *CSI‑u* sequence (`encode_glfw_key_event`, `kitty/keys.c:251`) — and **writes those bytes to the child** through the pseudo‑terminal (`schedule_write_to_child`, `kitty/keys.c:253` for text and `:259` for encoded). The child shell echoes bytes back; Kitty's **I/O thread** only *reads* those bytes off the PTY (`io_loop` → `read_bytes`, `kitty/child-monitor.c:1481`/`1337`) into a buffer, and the **main thread** then *parses* them (`parse_input`, `kitty/child-monitor.c:451`, called from `process_global_state` at `:1224`) through a **VT state machine** (`consume_normal`, `kitty/vt-parser.c:230`) into the **screen model** (`screen_draw_text`, `kitty/screen.c:866`). Finally the **display is produced on the GPU**: immediately after parsing, the same main‑thread tick calls `render` (`kitty/child-monitor.c:871`, invoked at `:1237`), which composites the cells through the OpenGL shader pipeline and presents the frame with `swap_window_buffers` (`kitty/child-monitor.c:810`). **Caveat:** only the *PTY read/write* happens on a separate thread; parsing and rendering happen **sequentially on the main thread** — so "receive → process → display" is a faithful description of a keystroke's data flow, with the read step decoupled onto the I/O thread (`docs/performance.rst:8`).

---

## 2. Environment & method

### 2.1 Build (canonical, primary attempt shown first)

The canonical developer build documented for this repository is **`./dev.sh build`** (`docs/build.rst:18-19`); `dev.sh` is a one‑line wrapper, `exec go run bypy/devenv.go "$@"` (`dev.sh:9`). Running that command **verbatim, with no extra flags** in this toolchain **fails**. Its 135‑line transcript begins:

```console
$ export PATH=/usr/local/go/bin:$PATH
$ ./dev.sh build
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
```

Compilation proceeds until the vendored GLFW Wayland window code fails to compile; the genuine tail of the transcript (the failing `gcc` invocation line between `cc1:` and the summary is the only line omitted, for width) is:

```console
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
The following build command failed: /tmp/blitzy/kitty/blitzy-4f08959c-7562-4d36-af17-e80ca3f1005a_a77559/dependencies/linux-amd64/bin/python setup.py develop
exit status 1
```

**Exit status `1`.** This is not a source defect: the prebuilt dependency bundle ships a newer `wayland-protocols` header whose `xdg-toplevel` enum has `XDG_TOPLEVEL_STATE_CONSTRAINED_*` values that the pinned vendored GLFW's `switch` does not enumerate, and the build compiles with `-Werror=switch`. The **fallback** is Kitty's own `setup.py` switch `--ignore-compiler-warnings` (it sets `werror=''`; it is a build flag, **not** a source edit), which builds cleanly. Its transcript begins with the same compile steps:

```console
$ ./dev.sh build --ignore-compiler-warnings
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
```

and ends (after all 122 `Compiling` steps and 5 `Linking` steps) with:

```console
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
```

**Exit status `0`.** This was a genuine **full rebuild** (the transcript's own step counter reaches `[122/122]` `Compiling` and `[5/5]` `Linking`), not an incremental no‑op. The canonical launcher artifact is **`kitty/launcher/kitty`** (`docs/build.rst:22`). Prerequisites observed in the environment: `go version go1.22.12 linux/amd64` (satisfies `go 1.22`, `go.mod:3`) and `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` (a C11 compiler); Python `>=3.8` is the documented runtime floor (`pyproject.toml:2`).

### 2.2 Version banner (verbatim, byte‑exact)

```console
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

`od -c` confirms the bytes are exactly `kitty 0.35.2 created by Kovid Goyal\n` (octal length `0000044` = 36 bytes) — plain ASCII, **no** ANSI styling and **no** git‑rev when piped to a non‑TTY:

```console
$ ./kitty/launcher/kitty --version | od -c
0000000   k   i   t   t   y       0   .   3   5   .   2       c   r   e
0000020   a   t   e   d       b   y       K   o   v   i   d       G   o
0000040   y   a   l  \n
0000044
```

The banner is assembled by `version()` (`kitty/cli.py:486-492`, format string `'{} {}{} created by {}'`); the `0.35.2` comes from `version: Version = Version(0, 35, 2)` (`kitty/constants.py:25`). (On an interactive TTY the same function adds italic/green SGR styling and a `rev` field; here the output is non‑TTY, so neither appears.)

### 2.3 Headless display

Kitty renders exclusively through OpenGL with **no CPU fallback** (`gl_init` calls `fatal(...)` if a GL context or the required version/extension is unavailable — `kitty/gl.c:53-75`), so a display/GPU is required. The environment is headless, so a virtual X display was used (the harness allocates a *unique, unused* display number per run, e.g. `:200`, and tears it down afterward):

```console
$ Xvfb :200 -screen 0 1280x800x24 +extension GLX +render -noreset &
$ export DISPLAY=:200
```

Mesa's software GL (llvmpipe) provides OpenGL 4.5 here, which comfortably satisfies Kitty's minimum (**GL 3.1 on Linux**, GL 3.3 on Apple, plus the `ARB_texture_storage` extension — `kitty/data-types.h:19-25`, `kitty/gl.c:63-67`).

### 2.4 Launch with Kitty's own tracing flags

```console
$ stdbuf -oL -eL ./kitty/launcher/kitty --debug-input --debug-rendering \
        --dump-bytes "$WORKDIR/kdump.bin" -o cursor_blink_interval=0 \
        > "$WORKDIR/out.log" 2> "$WORKDIR/err.log" &
```

The default shell under Kitty is `/bin/bash`, and there is no `~/.config/kitty/kitty.conf`, so this is Kitty's **default, canonical configuration** (`-o cursor_blink_interval=0` only stops the cursor blink so the framebuffer capture in §5 is deterministic; it does not affect the input path). `stdbuf -oL -eL` is only a *flushing* aid so the block‑buffered GL diagnostic (§5) appears promptly; it does **not** change Kitty's behavior. These flags are Kitty's native, documented tracing hooks (all declared in `kitty/cli.py`), quoted **verbatim** from the source:

| Flag | Help text (verbatim, `kitty/cli.py`) | Declared at | Where its output goes (observed) |
|---|---|---|---|
| `--debug-input` / `--debug-keyboard` | "Print out key and mouse events as they are received." | `kitty/cli.py:996` | **STDERR** |
| `--debug-rendering` / `--debug-gl` | "Debug rendering commands. This will cause all OpenGL calls to check for errors instead of ignoring them. Also prints out miscellaneous debug information. Useful when debugging rendering problems." | `kitty/cli.py:989` | **STDOUT** (GL‑version line only, observed) |
| `--dump-bytes <path>` | "Path to file in which to store the raw bytes received from the child process." | `kitty/cli.py:985` | the **named file** |
| `--dump-commands` | "Output commands received from child process to STDOUT." | `kitty/cli.py:972` | STDOUT (not enabled here) |

**Stream‑attribution mechanics (verified in source and matched by observation).** The `--debug-input` reception traces are emitted through the `debug` macro, which in `kitty/keys.c` resolves to `debug_input` (`kitty/keys.h:16`), and `debug_input` expands to `if (OPT(debug_keyboard)) { timed_debug_print(...); }` (`kitty/state.h:15`). `timed_debug_print` writes to **stderr** — `fprintf(stderr, "[%.3f] ", ...)` then `vfprintf(stderr, ...)` (`kitty/monotonic.h:102`/`105`) — which is why every such line is prefixed with a `[seconds]` timestamp and appears on STDERR. The single GL‑version diagnostic is the exception: it is printed with `printf(...)` → **STDOUT** (`kitty/gl.c:72`). This exactly matches the capture (reception on STDERR, GL line on STDOUT).

> In the raw captures below, `^[` is the ESC byte (`0x1b`) as rendered by `cat -v`; sequences such as `^[[33m … ^[[m` are ANSI SGR color codes that Kitty's debug prints wrap around their labels (e.g. `kitty/keys.c:176` emits a label wrapped in `\x1b[33m … \x1b[m`). They are part of the genuine, unedited output. **Do not confuse** this `cat -v` rendering of *real* ESC bytes with the *literal* `^[ ` token that the encoded‑key debug formatter prints — see §4.3.

### 2.5 Why this is the real path, not a bypass

The debug flags are wired into the *genuine* input pipeline at GLFW‑init time (`init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)`, `kitty/main.py:514`); enabling them only **annotates** the real path, it does not reroute input. Keys were injected with X11 `XTEST` (via `xdotool`), which are **synthetic X‑server key events**, *not* physical hardware presses — but they are delivered to the X server and travel the **complete, genuine software path**: X server → the vendored GLFW fork's XKB decoder (`glfw/xkb_glfw.c`) → GLFW key event → `key_callback` → `on_key_input`. This is emphatically **not** remote control (`kitty @ send-text`) and **not** injection into Kitty's internals, both of which would bypass reception. Where the text below says "the same key," it means the **same logical key** (identical X keycode), reproduced across separate presses.

---

## 3. Stage 1 — Reception: which parts receive the input first

**Answer:** the **platform/GLFW layer receives the keystroke first**. On Linux the XKB code inside the vendored GLFW fork translates the keycode and hands a GLFW key event to Kitty's `key_callback`, which immediately forwards it to `on_key_input`, the first Kitty‑owned function to see the event.

**Observed evidence.** With `--debug-input`, pressing `l`, `s`, Return produced these **consecutive, unfiltered** STDERR lines (the harness prints them straight from the raw error log, so both the platform XKB line and the Kitty line are visible in order):

```text
[1.129] ^[[31mPress^[[m xkb_keycode: 0x2e clean_sym: l composed_sym: l text: l mods: none glfw_key: 108 (l) xkb_key: 108 (l)
[1.129] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[1.130] ^[[32mRelease^[[m xkb_keycode: 0x2e clean_sym: l mods: none glfw_key: 108 (l) xkb_key: 108 (l)
[1.130] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.139] ^[[31mPress^[[m xkb_keycode: 0x27 clean_sym: s composed_sym: s text: s mods: none glfw_key: 115 (s) xkb_key: 115 (s)
[1.139] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[1.153] ^[[31mPress^[[m xkb_keycode: 0x24 clean_sym: Return composed_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[1.153] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
```

- The **XKB line** (`Press xkb_keycode: 0x2e …`) is emitted by `glfw_xkb_handle_key_event` at `glfw/xkb_glfw.c:875` as it decodes the raw keycode (`xkb_keycode: 0x2e`) into a symbol (`clean_sym: l`). This is the Linux/X11 keymap feeder for GLFW, and it prints **before** any Kitty code runs for that key.
- The **`on_key_input` line** immediately follows, emitted by `on_key_input` (`kitty/keys.c:166`) under `if (OPT(debug_keyboard))` (`kitty/keys.c:172`); its label/format string is at `kitty/keys.c:176`.
- Both lines carry the **same `[seconds]` timestamp** (`[1.129]` for `l`, `[1.139]` for `s`, `[1.153]` for Return), with the XKB line first — the empirical proof that the platform/GLFW layer receives the event and *then* calls into Kitty.

**How the event reaches `on_key_input` (the call chain).** GLFW's keyboard callback is registered once at window creation — `glfwSetKeyboardCallback(w, key_callback)` (`kitty/glfw.c:1292`) — and `key_callback` (`kitty/glfw.c:430`) forwards the event with `on_key_input(ev)` (`kitty/glfw.c:439`). The startup trace shows the platform keymap being loaded before any key is pressed:

```text
[0.056] Loading new XKB keymaps
[0.060] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.146] OS Window created
[0.156] Failed to open systemd user bus with error: Connection refused
[0.159] Child launched
[0.160] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
```

("Loading new XKB keymaps" and "Modifier indices" are the XKB keymap loader in `glfw/xkb_glfw.c`; the "Failed to open systemd user bus" line is benign — `DBUS_SESSION_BUS_ADDRESS` is `/dev/null` in this headless box.) On macOS/Cocoa and Wayland the feeder differs (Cocoa, or `glfw/ibus_glfw.c` for IME), but entry into Kitty is the same `key_callback` → `on_key_input` chain — **`inferred`** for the non‑X11 platforms, since only Linux/X11 was exercised here.

**Thread note.** `key_callback`/`on_key_input` run on Kitty's **main thread** (GLFW delivers window/keyboard callbacks on the thread that owns the window and pumps the event loop), so reception, encoding, parsing, and rendering are all main‑thread work; only the PTY byte read is offloaded (see §4.4).

---

## 4. Stage 2 — Intermediate processing: what happens between reception and the screen update

Two byte streams cross the pseudo‑terminal (PTY) boundary, and both are part of the intermediate stage.

### 4.1 Terminal → child (the keystroke is interpreted, encoded, and written to the shell)

Inside `on_key_input`, three things happen in order, all observable on one continued STDERR line per key:

1. **Shortcut check (PRESS/REPEAT only).** For PRESS or REPEAT actions — the guard `if (action == GLFW_PRESS || action == GLFW_REPEAT)` at `kitty/keys.c:226` — `on_key_input` dispatches into the Python layer via `dispatch_possible_special_key` (`kitty/keys.c:228`; the Python side is `kitty/keys.py:154`) to see whether the key is a mapped shortcut. When it is not consumed, processing continues to encoding.
2. **Encoding (legacy vs. protocol).** `encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)` (`kitty/keys.c:251`, implemented in `kitty/key_encoding.c`) decides how the key is serialized. Its return value selects one of three dispositions.
3. **Write to child (through the PTY).** The bytes are queued with `schedule_write_to_child` — at `kitty/keys.c:253` for the literal‑text disposition and `kitty/keys.c:259` for the encoded disposition (both implemented at `kitty/child-monitor.c:372`).

The three dispositions were all observed, each with its own trailing debug string:

```text
# plain letter 'l'  ->  written to the child as literal text  (schedule_write_to_child @ kitty/keys.c:253; debug string @ :254)
[1.129] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l

# Enter  ->  ENCODED to a single carriage-return byte 0x0d  (schedule_write_to_child @ :259; debug string @ :261)
[1.153] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd

# key RELEASE in the shell's legacy mode  ->  not encoded at all  (debug string @ :271)
[1.130] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

So an ordinary letter is delivered to the shell as its literal text byte (`l` = `0x6c`), Enter is *encoded* to `0x0d`, and key releases are dropped in the shell's default/legacy keyboard mode. The strings `sent key as text to child`, `sent encoded key to child`, and `ignoring as keyboard mode does not support encoding this event` are printed at `kitty/keys.c:254`, `:261`, and `:271` respectively.

*(There is also a Python‑level API for the same terminal→child hop — `send_key` → `send_key_sequence` → `write_to_child` (`kitty/window.py:917`/`937`/`955`). For a pressed key the observed path is the C‑side `on_key_input` → `schedule_write_to_child`; the `window.py` chain is the equivalent programmatic entry point — **`inferred`**, not exercised here.)*

### 4.2 Child → terminal (bytes are read back, parsed, and applied to the screen model)

The shell echoes what you typed (and prints command output). Those bytes are exactly what `--dump-bytes` captures, which is why that file is the empirical fingerprint of this hop.

**Before / during / after (the changing value).** The dump file transitioned from **absent** (before launch) to **719 bytes** (after launch, before typing — the shell's kitty‑integration startup prompt) to **2299 bytes** immediately after typing `ls`+Enter:

```text
before launch                : dump  absent
after launch, before typing  : dump  719 bytes   (shell startup prompt)
after typing 'ls'<Enter>     : dump  2299 bytes  (grew by 1580 bytes, beginning with the echo)
```

The file is opened empty with mode `'wb'` by `DumpCommands` (`kitty/boss.py:237`, installed at `kitty/boss.py:372`), and `DumpCommands.__call__` writes and flushes on `what == 'bytes'` (`kitty/boss.py:243-244`). The first 48 bytes of the delta (genuine `hexdump -C`) begin with the echoed characters:

```text
00000000  6c 73 0d 0a 1b 5b 3f 32  30 30 34 6c 0d 1b 5d 32  |ls...[?2004l..]2|
00000010  3b 6c 73 07 1b 5d 31 33  33 3b 43 3b 63 6d 64 6c  |;ls..]133;C;cmdl|
00000020  69 6e 65 3d 6c 73 07 01  1b 5d 31 33 33 3b 6b 3b  |ine=ls...]133;k;|
```

`6c 73 0d 0a` is `l s CR LF` — the shell echoing the two letters and the newline — followed by control sequences (`\x1b[?2004l` bracketed‑paste‑off, an OSC‑2 title `]2;ls`, an OSC‑133 `]133;C;cmdline=ls` shell‑integration mark) and then the directory listing.

### 4.3 A note on byte semantics (two different renderings of ESC)

The captures contain ESC bytes represented **two different ways**, and they must not be conflated:

- In the **dump file** and in raw stderr, `cat -v` renders a *real* ESC byte (`0x1b`) as `^[` with **no** following space; e.g. the protocol opt‑in in §6.4 appears as `^[[>31u` (bytes `1b 5b 3e 33 31 75`).
- In the **`on_key_input` encoded‑key debug line**, the formatter prints a *literal* token `^[ ` (caret, bracket, **space**) for each ESC byte, and each subsequent byte followed by a space — because the loop at `kitty/keys.c:263` is `if (encoded_key[ki] == 27) { debug("^[ "); } … else { debug("%c ", …); }`. That is why the same CSI‑u sequence shows up as space‑separated `^[ [ 9 7 ; ; 9 7 u` on the debug line but as unspaced `^[[97…u` when the *bytes themselves* are rendered by `cat -v`. The spacing is the tell: it is Kitty's bespoke debug formatter, not the wire bytes.

### 4.4 Read → parse → screen (which thread does what)

The child's output is **read** off the PTY by the **I/O thread** loop `io_loop` (`kitty/child-monitor.c:1481`), which calls `read_bytes` (`kitty/child-monitor.c:1337`); `read_bytes` only copies bytes into the VT‑parser's write buffer — it does **not** parse. Parsing happens on the **main thread**: `process_global_state` (`kitty/child-monitor.c:1224`, the main‑loop tick) calls `parse_input` (`kitty/child-monitor.c:451`, whose own comment reads "Parse all available input that was read in the I/O thread") at `:1236`, and *then* calls `render` at `:1237` — sequentially. When byte‑dumping is enabled the parse worker is `parse_worker_dump` (`kitty/child-monitor.c:180`, vs `parse_worker` at `:181`). The VT state machine classifies each byte: printable text through `consume_normal` (`kitty/vt-parser.c:230`), escapes through `consume_esc` (`:261`), CSI sequences through `consume_csi` (`:839`), dispatched by `dispatch` (`:413`). Printable text is written into the screen model by `screen_draw_text` (`kitty/screen.c:866`) / `draw_codepoint` (`kitty/screen.c:872`), backed by the line/cell buffers (`kitty/line.c`, `kitty/line-buf.c`), scrollback (`kitty/history.c`), and cursor (`kitty/cursor.c`). That the dumped bytes (the `ls` echo above) match what a terminal must parse to update its grid is the observed evidence for this read → parse → screen hop; the specific per‑function transitions **inside** the VT parser are **`inferred`** from the source (the parser has no per‑byte trace flag that was exercised here).

---

## 5. Stage 3 — Display production: how the updated display is ultimately produced

**Answer:** the updated screen is produced by a **main‑thread render tick that composites the screen model on the GPU via OpenGL and presents the frame by swapping the window's buffers** — there is no CPU rendering fallback.

**What was directly observed (context creation).** With `--debug-rendering`, exactly **one** rendering diagnostic appeared, on **STDOUT**, at startup, and it reproduced on every launch:

```text
[0.123] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

This is printed by `printf("[%.3f] GL version string: %s\n", …)` inside `gl_init` (`kitty/gl.c:72`, guarded by `global_state.debug_rendering`). Its presence proves Kitty successfully created an **OpenGL context** (here Mesa's software GL 4.5, which meets Kitty's minimum). Crucially, `--debug-rendering` emitted **no per‑frame render/swap/shader traces** in this build — so the observed rendering signal is bounded to **context initialization**, not to per‑frame drawing.

**Direct evidence that the display actually updates.** Because there is no per‑frame trace, display production was confirmed by capturing the **framebuffer** of the running window and measuring its content:

```text
framebuffer distinct colors: 483 (a blank screen would be 1)
non-background bounding box (glyph region): 1280x800+0+4
```

483 distinct colors over a glyph region spanning the window is direct proof that Kitty drew antialiased text to the screen (a blank/failed render would show a single color). Since Kitty has **no CPU rendering fallback** — `gl_init` aborts via `fatal(...)` if the GL context, version, or `ARB_texture_storage` extension is missing (`kitty/gl.c:53-75`) — a populated framebuffer is itself evidence that the **GPU/OpenGL** path ran.

**The render call chain (source‑grounded `inferred`, since no per‑frame trace was emitted).** The main thread runs the loop tick `process_global_state` (`kitty/child-monitor.c:1224`), which calls `render(now, input_read)` at `:1237`; `render` is defined at `kitty/child-monitor.c:871`. It composites each OS window through `render_os_window` (`:833`), preparing the frame with `prepare_to_render_os_window` (`:705`), drawing it with `render_prepared_os_window` (`:788`), and presenting it with **`swap_window_buffers(os_window)`** (`:810`). The pixel work is done by the OpenGL infrastructure in `kitty/gl.c` and the shader programs in `kitty/shaders.c` together with the **13** GLSL sources in `kitty/` (`ls kitty/*.glsl | wc -l` → `13`): `alpha_blend`, `bgimage_fragment`, `bgimage_vertex`, `border_fragment`, `border_vertex`, `cell_defines`, `cell_fragment`, `cell_vertex`, `graphics_fragment`, `graphics_vertex`, `linear2srgb`, `tint_fragment`, `tint_vertex`. This precise call/shader chain is labelled **`inferred`**: what was *observed* is (a) successful GL context creation and (b) a populated, updating framebuffer.

### 5.1 Concurrency (what actually runs in parallel)

The three stages are a **logical** ordering of a keystroke's data. In terms of threads, only the **PTY read/write** is offloaded: the child's bytes are read on an **I/O thread** (`io_loop`, `kitty/child-monitor.c:1481`), while **parsing and rendering both run on the main thread, sequentially** (`parse_input` at `:1236` immediately followed by `render` at `:1237`). The project documentation states the offload directly: "Interaction with child programs takes place in a separate thread from rendering, to improve smoothness." (`docs/performance.rst:8-9`), and the read/parse handoff is decoupled by a small delay — `input_delay` (default **3 ms**, `docs/performance.rst:48`). Consistent with that design, the reception/parse activity (STDERR) and the render diagnostic (STDOUT) were observed as **concurrent streams**, not strictly interleaved per keystroke — but the *parse→render* step for a given batch is sequential on one thread, so this is **not** a "parse thread vs render thread" split.

---

## 6. Conditions exercised (raw output beside each)

All keys were injected with `xdotool` (X `XTEST`) into the focused Kitty window (window id `2097164` in this run). The `xkb_keycode` visible in the reception traces (§3) confirms the genuine X → XKB → GLFW route.

### 6.1 Condition 1 — Unmodified simple letters (`l`, `s`, Enter)

Command: `xdotool key --clearmodifiers l` / `s` / `Return`. Full genuine PRESS traces are in §3; letters go to the child as literal text, Enter (`glfw key: 0xe001`, native X keysym `0xff0d`) is encoded to `0x0d`. Dump delta = 1580 bytes beginning `6c 73 0d 0a` (`ls\r\n`, §4.2).

### 6.2 Condition 2 — Modifier combinations (Ctrl‑C, Shift‑a, Alt‑a), legacy shell

Command: `xdotool key --clearmodifiers ctrl+c` / `shift+a` / `alt+a`. The `mods` field changes and the emitted bytes differ (unedited traces):

```text
# Ctrl-C : the modifier key itself is ignored, then 'c' with mods:ctrl encodes to 0x03 (ETX / SIGINT)
[1.782] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.788] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3

# Shift-a : composed to 'A' and sent as literal text
[2.112] ^[[33mon_key_input^[[m: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.118] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A

# Alt-a : encoded ESC-prefixed; the debug formatter prints the literal token "^[ a" (ESC byte + 'a')
[2.444] ^[[33mon_key_input^[[m: glfw key: 0xe063 native_code: 0xffe9 action: PRESS mods: alt text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.450] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ a
```

So `mods` takes values `ctrl` / `shift` / `alt` (vs `none` in Condition 1), and the emitted bytes are `0x03` (Ctrl‑C), literal `A` (Shift‑a), and ESC + `a` (Alt‑a — printed as the literal token `^[ a` per §4.3, i.e. bytes `1b 61`). In each modifier combo the modifier key *itself* (`0xe062` ctrl, `0xe061` shift, `0xe063` alt) arrives first and is dropped with "ignoring …".

### 6.3 Condition 3 — Press / repeat / release lifecycle (before / during / after of one key)

Command: `xdotool keydown j; sleep 0.8; xdotool keyup j` (with `xset r rate 250 30`). A single held key produced **1 × PRESS**, **18 × REPEAT**, **1 × RELEASE** — 20 events. The first four genuine events:

```text
[2.975] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: PRESS  mods: none text: 'j' state: 0 sent key as text to child: j
[3.225] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[3.258] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[3.291] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
```

The 18 REPEAT lines are byte‑for‑byte identical except for the `[seconds]` timestamp (they arrived ≈33 ms apart, from `[3.225]` through `[3.787]`). The final two genuine events:

```text
[3.787] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[3.794] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

The complete per‑action tally for the held key `0x6a`, and for the whole run, confirms the count exactly:

```text
# held-key 'j' (0x6a) tally            # whole-run tally (all keys)
      1 action: PRESS                        43 action: PRESS
     18 action: REPEAT                        43 action: RELEASE
      1 action: RELEASE                       18 action: REPEAT
```

The `action:` field (`kitty/keys.c:176`) thus takes all three values — the full lifecycle of one key. Each PRESS/REPEAT sends `j`; the RELEASE is dropped in legacy mode.

### 6.4 Condition 4 — Legacy vs. Kitty Keyboard Protocol (same logical key `a`)

The default shell uses **legacy** encoding (the canonical, primary case). To exercise the **Kitty Keyboard Protocol** without a bypass, `kitten show-key -m kitty` was run *inside* the window; it opts in via progressive enhancement. That kitten enables the **full** protocol, flags **31** (`FULL_KEYBOARD_PROTOCOL = DISAMBIGUATE_KEYS(1) | REPORT_KEY_EVENT_TYPES(2) | REPORT_ALTERNATE_KEYS(4) | REPORT_ALL_KEYS_AS_ESCAPE_CODES(8) | REPORT_TEXT_WITH_KEYS(16) = 31`, `tools/tui/loop/terminal-state.go:15-21`; used by `kittens/show_key/kitty.go:20`). The opt‑in and restore bytes were observed in the child→terminal dump (genuine `cat -v`, counted):

```text
      1 [>31u        # push:  ESC[>31u   (opt in, flags 31)   — tools/tui/loop/terminal-state.go:140
      1 [<u          # pop:   ESC[<u     (restore on exit)    — tools/tui/loop/terminal-state.go:159
```

(These render as `^[[>31u` / `^[[<u` with the ESC byte; the `^[` prefix is trimmed by the counting `grep`.) With the protocol active, the **same logical key `a`** encodes completely differently from legacy. Both sets of lines below are genuine `on_key_input` output (note the space‑separated `^[ [ …` is the debug formatter of §4.3); the legacy `a`/Ctrl‑`a` lines are from a supplementary run in the default `/bin/bash` shell, hence their distinct `[10.x]`/`[11.x]` timestamp base:

```text
# LEGACY (default /bin/bash shell) — genuine on_key_input lines for the key 'a'
[10.940] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: none text: 'a' state: 0 sent key as text to child: a
[10.942] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: ''  state: 0 ignoring as keyboard mode does not support encoding this event
[11.310] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: ctrl text: ''  state: 0 sent encoded key to child: 0x1

# KITTY KEYBOARD PROTOCOL (inside `kitten show-key -m kitty`)
[5.441] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: none text: 'a' state: 0 sent encoded key to child: ^[ [ 9 7 ; ; 9 7 u
[5.447] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: ''  state: 0 sent encoded key to child: ^[ [ 9 7 ; 1 : 3 u
[5.858] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: ''  state: 0 sent encoded key to child: ^[ [ 5 7 4 4 2 ; 5 u
[5.864] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS   mods: ctrl text: ''  state: 0 sent encoded key to child: ^[ [ 9 7 ; 5 u
```

| Same logical key `a` | Legacy (default shell) | Kitty Keyboard Protocol (`show-key -m kitty`) |
|---|---|---|
| press | literal `a` (`0x61`) | `ESC[97;;97u` (CSI‑u; `97` = codepoint of `a`, plus associated text `97`) |
| release | **not encoded** (dropped) | `ESC[97;1:3u` (release **is** reported; event‑type `:3`) |
| Ctrl‑`a` | `0x01` (SOH) | `ESC[97;5u` (modifier field `5` = Ctrl) |
| Ctrl key itself | (dropped) | `ESC[57442;5u` (the Control key is reported too) |

The branch is chosen by `encode_glfw_key_event` from `screen_current_key_encoding_flags(screen)` (`kitty/keys.c:251`); the protocol's push/pop grammar is documented at `docs/keyboard-protocol.rst:293-297`. **Legacy is the canonical primary evidence** because it is what the default shell uses.

### 6.5 Condition 5 — `--dump-bytes` before / during / after

```text
before launch                : dump  absent
after launch, before typing  : dump  719 bytes   (shell startup prompt; kitty/boss.py:237 opens it 'wb')
after all legacy typing      : dump  2781 bytes
```

The bytes appended right after typing begin `6c 73 0d 0a` = `ls\r\n` (the shell **echo**), confirming that `--dump-bytes` captures the **child → terminal** stream — precisely the bytes that feed the VT parser and update the screen (mechanism: `kitty/boss.py:243-244`). Full hex of the first delta is in §4.2.

### 6.6 Condition 6 — Edge branches (focus loss observed; no‑active‑window `inferred`)

**6a — Focus loss (observed distribution).** After moving X input focus away from the Kitty window and injecting three keys (`x`, `y`, `z`), Kitty received **none** of them:

```text
on_key_input count before=104; after 3 keys while UNFOCUSED=104; delivered=0 of 3
focus-change events observed:
[0.160] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
[7.879] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 0
```

The `on_key_input` counter did not advance (104 → 104), and the `on_focus_change … focused: 0` line marks the transition — direct evidence that, unfocused, no key events reach Kitty (they go to whatever the X server considers focused).

**6b — No active window (`inferred`).** If a key event arrives with no active window, `on_key_input` logs `no active window, ignoring` and returns (`kitty/keys.c:182`). This branch was **not** triggered — `grep -c "no active window, ignoring"` over the captured stderr returned `0`, because a window was always present in the single‑window run:

```text
'no active window, ignoring' occurrences this run: 0
```

It is therefore reported as **`inferred`** from the source, not observed.

---

## 7. Stability (reproduced across two fresh launches)

The identical input `ls`+Enter was driven in **two separate, freshly launched** Kitty processes. The reception traces reproduced exactly — same `glfw key` codes, same native codes, same dispositions, same `mods` — differing only in the `[seconds]` timestamp (expected, from the monotonic clock):

```text
# RUN 1 (PRESS lines)
[1.129] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[1.139] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[1.153] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd

# RUN 2 (fresh process, PRESS lines)
[1.118] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[1.133] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[1.147] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
```

The harness's own diff of these lines (timestamps stripped) is empty, and it reports:

```text
RESULT: reception traces IDENTICAL across the two fresh runs (timestamps aside)
```

The dump echo reproduced identically (`6c 73 0d 0a` = `ls\r\n`; 719 → 2299 bytes on both runs), and the GL diagnostic matched (`GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`; `[0.123]` on run 1, `[0.126]` on run 2). **Result: reproduced across 2 runs.** (Methodology note: a single Kitty instance must own the X focus; under a WM‑less `Xvfb` the harness explicitly focuses the PID‑verified window before typing and tears each instance down before starting the next.)

---

## 8. Reasoning / rationale (observed signal → responsible function)

- **Reception is the platform/GLFW layer.** The XKB line (`glfw/xkb_glfw.c:875`) printing a real `xkb_keycode` **immediately before**, and with the same timestamp as, the `on_key_input` line (`kitty/keys.c:176`) shows the OS/GLFW layer decodes and receives the key first and then calls Kitty's `on_key_input` (`kitty/keys.c:166`), reached from `key_callback` (`kitty/glfw.c:430`) → `on_key_input` (`kitty/glfw.c:439`).
- **Intermediate processing is `on_key_input` deciding an encoding and writing to the PTY, then the child's echo being read and parsed into the screen model.** The three trailing dispositions (`sent key as text to child` / `sent encoded key to child` / `ignoring …`, `kitty/keys.c:254`/`261`/`271`) show `encode_glfw_key_event` (`kitty/keys.c:251`) choosing text vs. escape vs. drop, then `schedule_write_to_child` (`kitty/keys.c:253`/`259`) queuing the bytes; the `--dump-bytes` echo (`ls\r\n`) is the child's response that the I/O thread reads (`io_loop`/`read_bytes`, `kitty/child-monitor.c:1481`/`1337`) and the main thread parses (`parse_input` at `:1236` → `consume_normal`, `kitty/vt-parser.c:230`) into `screen_draw_text` (`kitty/screen.c:866`).
- **Display is produced on the GPU.** The `GL version string` line (`kitty/gl.c:72`) proves an OpenGL context was created; the populated framebuffer (483 colors) proves pixels were drawn. Because there is no CPU fallback (`kitty/gl.c:53-75`), a visible, updating window is itself evidence the GPU path ran; the specific `render` → shader → `swap_window_buffers` chain (`kitty/child-monitor.c:871`/`810`) is `inferred` from source since `--debug-rendering` emitted no per‑frame trace.
- **Concurrency is a read‑offload, not a parse/render split.** Only the PTY read runs on the I/O thread; `parse_input` (`kitty/child-monitor.c:1236`) and `render` (`:1237`) run sequentially on the main thread. The "separate thread" in `docs/performance.rst:8` refers to the child‑interaction (I/O) thread; the two streams (STDERR reception/parse, STDOUT GL) were observed concurrently.

### Notes on entry points and counts (observed reality is authoritative)

- Kitty's Python entry point is the **repository‑root `__main__.py`**; there is **no** `kitty/__main__.py` in this checkout (verified: `ls kitty/__main__.py` → not found).
- There are **13** GLSL shader files in `kitty/` (`ls kitty/*.glsl | wc -l` → `13`), listed in §5.
- The version is the real `--version` output (`kitty 0.35.2 created by Kovid Goyal`); source constant `Version(0, 35, 2)` (`kitty/constants.py:25`).

---

## 9. Coverage pass (all three named sub‑questions answered)

- [x] **Reception — "which parts appear to receive the input first"** → §3. Answer: the OS/platform + GLFW layer (XKB feeder `glfw/xkb_glfw.c:875`; GLFW callback `key_callback`, `kitty/glfw.c:430`) receive first, then Kitty's `on_key_input` (`kitty/keys.c:166`). Backed by the same‑timestamp XKB‑before‑`on_key_input` capture.
- [x] **Intermediate processing — "which parts handle the intermediate processing"** → §4. Answer: `on_key_input` interprets/encodes the key (`dispatch_possible_special_key` `kitty/keys.c:228`; `encode_glfw_key_event` `kitty/keys.c:251`) and writes it to the child via the PTY (`schedule_write_to_child` `kitty/keys.c:253`/`259`); the echoed bytes are read on the I/O thread (`io_loop`/`read_bytes` `kitty/child-monitor.c:1481`/`1337`) and parsed on the main thread (`parse_input` `:1236` → `consume_normal` `kitty/vt-parser.c:230`) into the screen model (`screen_draw_text` `kitty/screen.c:866`). Backed by the trailing dispositions and the `--dump-bytes` before/during/after echo.
- [x] **Display production — "how the updated display is ultimately produced"** → §5. Answer: the main‑thread render tick (`render` `kitty/child-monitor.c:871`) composites on the GPU (OpenGL, `kitty/gl.c`; shaders in `kitty/shaders.c` + 13 `*.glsl`) and presents via `swap_window_buffers` (`kitty/child-monitor.c:810`). Backed by the captured `GL version string` diagnostic (`kitty/gl.c:72`) and the populated framebuffer.
- [x] **Concurrency caveat** stated (§5.1, §8): only the PTY read is offloaded to the I/O thread; parse and render run sequentially on the main thread (`kitty/child-monitor.c:1236-1237`; `docs/performance.rst:8`, `:48`).

---

### Appendix — the exact observation harness used (temporary; removed after the investigation)

All raw output in this document came from a single run of the script below. It is **read‑only** with respect to Kitty (it only runs the already‑built launcher with Kitty's own trace flags and injects real `XTEST` keys); it is fully self‑cleaning (`set -euo pipefail`, `umask 077`, a private `mktemp -d` working directory, a unique unused X display per run, PID‑verified window ownership, and a `trap` that terminates every spawned process, removes all artifacts, and prints a final `git status`). It passes `bash -n`. Run as `./observe_kitty_input.sh /path/to/kitty-repo`.

```bash
#!/usr/bin/env bash
# observe_kitty_input.sh
# Runtime-grounded observation harness for Kitty's keyboard-input pipeline.
#
# It is READ-ONLY with respect to the Kitty source tree: it only *runs* the
# already-built launcher with Kitty's own trace flags (--debug-input,
# --debug-rendering, --dump-bytes) and injects real X11 XTEST key events with
# xdotool. Those events enter the genuine X11 -> XKB -> GLFW -> Kitty software
# path upstream of GLFW; this is NOT remote control and NOT synthetic injection
# into Kitty's internals. Every artifact lives inside a private mktemp directory
# and a private Xvfb on a unique display; all of it is torn down on exit.
#
# Usage:  ./observe_kitty_input.sh /path/to/kitty-repo
set -euo pipefail
umask 077

REPO="${1:-$PWD}"
LAUNCHER="$REPO/kitty/launcher/kitty"
[ -x "$LAUNCHER" ] || { echo "FATAL: launcher not found/executable at $LAUNCHER" >&2; exit 3; }

WORKDIR="$(mktemp -d "${TMPDIR:-/tmp}/kitty_obs.XXXXXXXX")"

pick_free_display() {
    local n
    for n in $(seq 200 260); do
        if [ ! -e "/tmp/.X${n}-lock" ] && [ ! -e "/tmp/.X11-unix/X${n}" ]; then
            printf '%s\n' "$n"; return 0
        fi
    done
    echo "FATAL: no free X display found" >&2; return 4
}
DISPNUM="$(pick_free_display)"
DISPLAY_STR=":${DISPNUM}"

XVFB_PID=""
KITTY_PID=""     # launcher pid
GUI_PID=""       # window-owning GUI child pid
WIN=""

kill_wait() {    # kill_wait <pid>
    local p="${1:-}" _
    [ -n "$p" ] || return 0
    kill -0 "$p" 2>/dev/null || return 0
    kill "$p" 2>/dev/null || true
    for _ in $(seq 1 30); do kill -0 "$p" 2>/dev/null || break; sleep 0.1; done
    kill -9 "$p" 2>/dev/null || true
    wait "$p" 2>/dev/null || true
}

cleanup() {
    set +e
    if [ -n "${XVFB_PID}" ] && kill -0 "${XVFB_PID}" 2>/dev/null; then
        DISPLAY="${DISPLAY_STR}" xdotool keyup j 2>/dev/null   # release any held key
    fi
    kill_wait "${GUI_PID}"
    kill_wait "${KITTY_PID}"
    kill_wait "${XVFB_PID}"
    rm -f "/tmp/.X${DISPNUM}-lock" "/tmp/.X11-unix/X${DISPNUM}"
    rm -rf "${WORKDIR}"
    echo "=== CLEANUP absence checks ==="
    kill -0 "${KITTY_PID:-0}" 2>/dev/null && echo "kitty launcher STILL ALIVE" || echo "kitty launcher absent: OK"
    kill -0 "${GUI_PID:-0}"   2>/dev/null && echo "kitty GUI STILL ALIVE"      || echo "kitty GUI absent: OK"
    kill -0 "${XVFB_PID:-0}"  2>/dev/null && echo "Xvfb STILL ALIVE"           || echo "Xvfb absent: OK"
    [ -e "/tmp/.X${DISPNUM}-lock" ]     && echo "X lock STILL present"   || echo "X lock absent: OK"
    [ -e "/tmp/.X11-unix/X${DISPNUM}" ] && echo "X socket STILL present" || echo "X socket absent: OK"
    [ -e "${WORKDIR}" ] && echo "WORKDIR STILL present" || echo "WORKDIR removed: OK"
    echo "=== final git status of repo (only the answer doc should ever change) ==="
    ( cd "$REPO" && git status --porcelain ) || true
}
trap cleanup EXIT INT TERM

Xvfb "${DISPLAY_STR}" -screen 0 1280x800x24 +extension GLX +render -noreset \
    >"${WORKDIR}/xvfb.log" 2>&1 &
XVFB_PID=$!
export DISPLAY="${DISPLAY_STR}"
for _ in $(seq 1 50); do xdpyinfo >/dev/null 2>&1 && break; sleep 0.1; done
xdpyinfo >/dev/null 2>&1 || { echo "FATAL: Xvfb ${DISPLAY_STR} not reachable" >&2; exit 5; }
echo "Xvfb up on ${DISPLAY_STR} (pid ${XVFB_PID}); owns lock $(tr -d ' ' < "/tmp/.X${DISPNUM}-lock" 2>/dev/null)"
xset r rate 250 30 >/dev/null 2>&1 || true   # deterministic autorepeat for the burst

owns() {   # owns <winpid> — true if winpid is the launcher or its child
    local wp="${1:-0}" pp
    [ "$wp" = "$KITTY_PID" ] && return 0
    pp="$(ps -o ppid= -p "$wp" 2>/dev/null | tr -d ' ')"
    [ "$pp" = "$KITTY_PID" ] && return 0
    return 1
}

launch_kitty() {   # launch_kitty <dump> <out> <err>
    local dump="$1" out="$2" err="$3" wp cand
    : > "$dump"
    stdbuf -oL -eL "$LAUNCHER" --debug-input --debug-rendering \
        --dump-bytes "$dump" -o cursor_blink_interval=0 \
        >"$out" 2>"$err" &
    KITTY_PID=$!
    for _ in $(seq 1 100); do grep -qa "Child launched" "$err" 2>/dev/null && break; sleep 0.1; done
    WIN=""; GUI_PID=""
    for _ in $(seq 1 80); do
        for cand in $(xdotool search --class kitty 2>/dev/null); do
            wp="$(xdotool getwindowpid "$cand" 2>/dev/null || echo 0)"
            if owns "$wp"; then WIN="$cand"; GUI_PID="$wp"; break; fi
        done
        [ -n "$WIN" ] && break
        sleep 0.1
    done
    [ -n "$WIN" ] || { echo "FATAL: no kitty window owned by launcher $KITTY_PID" >&2; return 6; }
    echo "kitty launcher pid=$KITTY_PID; window=$WIN; _NET_WM_PID=$GUI_PID (owned by launcher): verified"
    focus_window
}

focus_window() {
    xdotool windowfocus --sync "$WIN" 2>/dev/null || true
    xdotool windowactivate --sync "$WIN" 2>/dev/null || true
    sleep 0.4
}

stop_kitty() {   # stop this run's kitty and WAIT until its window is gone
    kill_wait "${GUI_PID}"
    kill_wait "${KITTY_PID}"
    local _ c leftover
    for _ in $(seq 1 50); do
        leftover=""
        for c in $(xdotool search --class kitty 2>/dev/null); do
            [ "$c" = "$WIN" ] && leftover="yes"
        done
        [ -z "$leftover" ] && break
        sleep 0.1
    done
    KITTY_PID=""; GUI_PID=""; WIN=""
}

dsize() { stat -c '%s' "$1" 2>/dev/null || echo absent; }
tally() { { grep -a on_key_input "$1" || true; } | { grep -oa 'action: [A-Z]*' || true; } | sort | uniq -c; }

# ===========================================================================
echo "############################## RUN 1 ##############################"
DUMP1="${WORKDIR}/kdump1.bin"; OUT1="${WORKDIR}/out1.log"; ERR1="${WORKDIR}/err1.log"
echo "dump BEFORE launch: $( [ -e "$DUMP1" ] && dsize "$DUMP1" || echo absent ) bytes"
launch_kitty "$DUMP1" "$OUT1" "$ERR1"
sleep 0.5
grep -qa "focused: 1" "$ERR1" && echo "focus confirmed: on_focus_change focused: 1" || echo "WARN: focus not confirmed"
echo "dump AFTER launch, BEFORE typing: $(dsize "$DUMP1") bytes"

echo "----- Condition 1: unmodified letters l, s, Return -----"
C1_PRE="$(dsize "$DUMP1")"
R1_PRE="$(wc -l < "$ERR1")"
xdotool key --clearmodifiers l
xdotool key --clearmodifiers s
xdotool key --clearmodifiers Return
sleep 0.6
C1_POST="$(dsize "$DUMP1")"
echo "dump grew ${C1_PRE} -> ${C1_POST} bytes after 'ls'<Enter>"
echo "--- hexdump -C of the first 48 bytes of the child echo delta ---"
tail -c +"$((C1_PRE + 1))" "$DUMP1" | head -c 48 | hexdump -C
echo "--- reception adjacency: platform XKB feeder line (glfw/xkb_glfw.c:875) immediately precedes on_key_input (kitty/keys.c:176), UNFILTERED, cat -v ---"
{ tail -n +"$((R1_PRE + 1))" "$ERR1" | grep -aE 'xkb_keycode|on_key_input' || true; } | head -12 | cat -v

echo "----- Condition 2: modifier combinations ctrl+c, shift+a, alt+a -----"
xdotool key --clearmodifiers ctrl+c; sleep 0.3
xdotool key --clearmodifiers shift+a; sleep 0.3
xdotool key --clearmodifiers alt+a; sleep 0.5

echo "----- Condition 3: press/repeat/release lifecycle (hold j ~0.8s) -----"
xdotool keydown j; sleep 0.8; xdotool keyup j; sleep 0.5

echo "----- Condition 5: dump size after all legacy typing -----"
echo "dump AFTER all legacy typing: $(dsize "$DUMP1") bytes"

echo "----- Condition 4: Kitty Keyboard Protocol via 'kitten show-key -m kitty' -----"
focus_window
# earlier conditions (held-j burst, modifiers) left uncommitted characters on the
# shell command line; abandon it with Ctrl-C so the kitten command runs cleanly
xdotool key --clearmodifiers ctrl+c; sleep 0.4
P_PRE="$(dsize "$DUMP1")"
E_PRE="$(wc -l < "$ERR1")"
xdotool type --clearmodifiers "kitten show-key -m kitty"
xdotool key --clearmodifiers Return
optin="no"
for _ in $(seq 1 60); do
    if tail -c +"$((P_PRE + 1))" "$DUMP1" | grep -qa $'\x1b\[>31u'; then optin="yes"; break; fi
    sleep 0.1
done
echo "protocol opt-in ESC[>31u seen in child->terminal dump: ${optin}"
xdotool key --clearmodifiers a; sleep 0.4          # press+release under protocol
xdotool key --clearmodifiers ctrl+a; sleep 0.4     # Ctrl-a under protocol
xdotool key --clearmodifiers ctrl+c; sleep 0.3     # terminate the kitten
xdotool key --clearmodifiers Return; sleep 0.6
echo "--- opt-in / restore markers in the child->terminal dump (cat -v, counted) ---"
{ tail -c +"$((P_PRE + 1))" "$DUMP1" | cat -v | grep -ao '\[>31u\|\[<u' || true; } | sort | uniq -c
echo "--- protocol-mode reception/encoding traces (verbatim on_key_input, cat -v) ---"
{ tail -n +"$((E_PRE + 1))" "$ERR1" | grep -a on_key_input || true; } | cat -v

echo "----- direct display-update evidence: capture framebuffer -----"
FB="${WORKDIR}/frame1.png"
if command -v import >/dev/null 2>&1; then
    if import -window root "$FB" 2>/dev/null; then
        echo "framebuffer distinct colors: $(convert "$FB" -format '%k' info: 2>/dev/null) (a blank screen would be 1)"
        echo "non-background bounding box (glyph region): $(convert "$FB" -fuzz 5% -trim info: 2>/dev/null | grep -oE '[0-9]+x[0-9]+\+[0-9]+\+[0-9]+' | head -1)"
    fi
fi

echo "----- Condition 6a: focus-loss distribution (unfocus, inject 3 keys) — done LAST -----"
before="$( { grep -ac on_key_input "$ERR1" || true; } )"
xdotool windowfocus --sync 0 2>/dev/null || true      # move X input focus away (to None/root)
sleep 0.3
xdotool key --clearmodifiers x 2>/dev/null || true
xdotool key --clearmodifiers y 2>/dev/null || true
xdotool key --clearmodifiers z 2>/dev/null || true
sleep 0.4
after="$( { grep -ac on_key_input "$ERR1" || true; } )"
echo "on_key_input count before=${before}; after 3 keys while UNFOCUSED=${after}; delivered=$((after-before)) of 3"
echo "focus-change events observed (cat -v):"
{ grep -a on_focus_change "$ERR1" || true; } | cat -v

echo "----- Condition 6b: no-active-window C branch (keys.c:182) probe -----"
echo "'no active window, ignoring' occurrences this run: $( { grep -ac 'no active window, ignoring' "$ERR1" || true; } )"

echo "===== RUN 1 legacy reception/encoding traces (verbatim, cat -v) ====="
{ grep -a on_key_input "$ERR1" | grep -av '; 9 7 \|5 7 4 4 2 ' || true; } | cat -v
echo "===== RUN 1 action-field tally over whole run (PRESS/REPEAT/RELEASE) ====="
tally "$ERR1"
echo "===== RUN 1 held-key 'j' burst tally (isolate the lifecycle key 0x6a) ====="
{ grep -a on_key_input "$ERR1" | grep -a 'key: 0x6a ' || true; } | { grep -oa 'action: [A-Z]*' || true; } | sort | uniq -c
echo "===== RUN 1 STDOUT (--debug-rendering) verbatim, cat -v ====="
cat -v "$OUT1"
echo "===== RUN 1 startup/platform stderr (first 6 lines, cat -v) ====="
head -6 "$ERR1" | cat -v

stop_kitty

# ===========================================================================
echo "############################## RUN 2 (fresh process) ##############################"
DUMP2="${WORKDIR}/kdump2.bin"; OUT2="${WORKDIR}/out2.log"; ERR2="${WORKDIR}/err2.log"
launch_kitty "$DUMP2" "$OUT2" "$ERR2"
sleep 0.5
C2_PRE="$(dsize "$DUMP2")"
xdotool key --clearmodifiers l
xdotool key --clearmodifiers s
xdotool key --clearmodifiers Return
sleep 0.6
C2_POST="$(dsize "$DUMP2")"
echo "RUN 2 dump grew ${C2_PRE} -> ${C2_POST} bytes after 'ls'<Enter>"
echo "--- hexdump -C of first 16 bytes of RUN 2 echo delta ---"
tail -c +"$((C2_PRE + 1))" "$DUMP2" | head -c 16 | hexdump -C
echo "===== RUN 2 PRESS reception/encoding traces (verbatim, cat -v) ====="
{ grep -a on_key_input "$ERR2" | grep -a 'action: PRESS' || true; } | cat -v
echo "===== RUN 2 STDOUT (--debug-rendering) verbatim, cat -v ====="
cat -v "$OUT2"

echo "############################## TWO-RUN COMPARISON ##############################"
first3() {   # the first l, first s, first Enter PRESS line, timestamp stripped
    {   { grep -a on_key_input "$1" | grep -a 'action: PRESS' | grep -am1 "text: 'l'"       || true; }
        { grep -a on_key_input "$1" | grep -a 'action: PRESS' | grep -am1 "text: 's'"       || true; }
        { grep -a on_key_input "$1" | grep -a 'action: PRESS' | grep -am1 "native_code: 0xff0d" || true; }
    } | sed -E 's/^\[[0-9]+\.[0-9]+\] //'
}
first3 "$ERR1" > "${WORKDIR}/r1.txt"
first3 "$ERR2" > "${WORKDIR}/r2.txt"
echo "--- diff RUN1 vs RUN2 PRESS lines for l / s / Enter (empty == identical modulo timestamp) ---"
if diff -u "${WORKDIR}/r1.txt" "${WORKDIR}/r2.txt"; then
    echo "RESULT: reception traces IDENTICAL across the two fresh runs (timestamps aside)"
else
    echo "RESULT: differences shown above"
fi

echo "############################## DONE (cleanup runs on EXIT) ##############################"
```

*This document is the sole artifact added to the repository; no Kitty source file was modified, and all temporary scripts, logs, dump files, and the virtual display created during the investigation were removed afterward (verified with a clean `git status`).*
