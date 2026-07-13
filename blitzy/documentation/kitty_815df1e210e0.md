# How keyboard input flows through Kitty's core components — a runtime‑grounded walkthrough

**Source tree under investigation:** `kovidgoyal/kitty` at commit `815df1e210e0` ("Wire up applying of font config"). That commit is the **source baseline** this document describes; the answer file itself is added on top of that tree on the working branch, so it is *not* the tree's own HEAD commit. Every `file:line` reference below points at the source as it exists in this `815df1e210e0` checkout.

**Method (build and run first, then write).** Every runtime claim was produced by **building and running Kitty from this repository**, launching it with its **own** debug/tracing flags, driving keys through the **real** keyboard path (X11 `XTEST`, i.e. synthetic X‑server key events that travel the genuine X11 → XKB → GLFW → Kitty software path — see §2.5), and copying the **actual, unedited** output next to each claim.

**Observed vs. `inferred` (a convention used throughout).** A statement is treated as **observed** only when a captured line, byte, or measured value in this document demonstrates it directly. A statement that could only be read from the source (a function's *name*, the *internal order* of steps that emit no separate trace, a per‑byte parser transition, the render call chain, a flag arithmetic, a non‑Linux platform path) is explicitly labelled **`inferred`** and carries a `file:line` citation. Where a paragraph mixes the two, the observed part is shown with its capture and the interpreted part is marked `inferred` in place.

**Privacy.** `--debug-input` records the *text* of every keystroke to the (stderr) debug log, and `--dump-bytes` together with the parsed‑command output on stdout (see §2.4) records the child's echo of what you typed. **Do not type real secrets while tracing.** The harness in the Appendix uses only harmless markers, runs under `umask 077`, keeps every artifact in a private directory **outside** the repository, and deletes all of it on exit.

**Provenance.** All runtime captures in §§3–7 (reception, encoding, `--dump-bytes`, framebuffer, legacy and Kitty‑Keyboard‑Protocol traces, the `kitten show-key -m normal` helper, the Ctrl‑D case, and both stability comparisons) come from **one** execution of the single observation harness in the Appendix (it builds nothing — it only runs the already‑built launcher six times). The only captures that come from **separate, explicitly‑labelled** runs are the two **build transcripts** and the **version banner** in §2.1–§2.2, because the harness does not build Kitty. A full per‑section origin table is given in the "Provenance inventory" section just before the Appendix.

---

## 1. One‑paragraph summary

When you press a key in the shell running inside Kitty, the keystroke is **received first by the platform/GLFW layer** — on Linux/X11 the vendored GLFW fork's XKB code turns the hardware keycode into a symbol and hands a GLFW key event to Kitty's callback `key_callback` (`kitty/glfw.c:430`), which immediately calls the first Kitty‑owned handler `on_key_input` (`kitty/keys.c:166`). *(Observed: an XKB decode line prints immediately before, and at the same timestamp as, the `on_key_input` line — §3.)* That handler performs the **intermediate processing**: it asks the Python layer whether the key is a mapped shortcut (`dispatch_possible_special_key`, dispatched for PRESS/REPEAT at `kitty/keys.c:226`/`228`), then **encodes** the key — either as a legacy byte/escape sequence or as a Kitty‑Keyboard‑Protocol *CSI‑u* sequence (`encode_glfw_key_event`, `kitty/keys.c:251`) — and **writes those bytes to the child** through the pseudo‑terminal (`schedule_write_to_child`, `kitty/keys.c:253` for text and `:259` for encoded). *(Observed: the final disposition of each key — text / encoded / dropped — via the trailing debug strings in §4.1. The internal ordering of shortcut‑check → encode → write emits no separate trace and is `inferred` from `kitty/keys.c:226`–`271`.)* The child shell echoes bytes back; Kitty's **I/O thread** reads those bytes off the PTY (`io_loop` → `read_bytes`, `kitty/child-monitor.c:1481`/`1337`) into a buffer, and the **main thread** then *parses* them (`parse_input`, `kitty/child-monitor.c:451`, called from the main‑loop tick at `:1236`) through a **VT state machine** (`consume_normal`, `kitty/vt-parser.c:230`) into the **screen model** (`screen_draw_text`, `kitty/screen.c:866`). *(Observed: the exact echoed bytes via `--dump-bytes`, §4.2. The read‑vs‑parse thread assignment and the per‑byte parser transitions are `inferred` from the cited source.)* Finally the **display is produced on the GPU**: the same main‑thread tick calls `render` (`kitty/child-monitor.c:871`, invoked at `:1237`), which composites the cells through the OpenGL shader pipeline and presents the frame with `swap_window_buffers` (`kitty/child-monitor.c:810`). *(Observed: OpenGL‑context creation and a before/after framebuffer change caused by typing, §5. The `render`→shader→`swap` chain is `inferred`.)* **Caveat:** only the PTY read/write is offloaded to the I/O thread; parsing and rendering run **sequentially on the main thread** — so "receive → process → display" is a faithful description of a keystroke's data flow (`inferred` from `kitty/child-monitor.c:1236`–`1237`; `docs/performance.rst:8`).

---

## 2. Environment & method

### 2.1 Build — canonical attempt, then the required non‑canonical build‑flag fallback

The canonical developer build documented for this repository is **`./dev.sh build`** (`docs/build.rst:18-19`); `dev.sh` is a one‑line wrapper, `exec go run bypy/devenv.go "$@"` (`dev.sh:9`). Running that **canonical** command **verbatim, with no extra flags** in this toolchain **fails** — so the values below were obtained from a **non‑canonical build‑flag fallback**, which is called out explicitly. This is a *build‑flag* fallback (`setup.py`'s own `--ignore-compiler-warnings`), **not** a Kitty source edit; no source file was changed.

The bare‑command transcript is **135 lines**; because the parallel build interleaves per‑file `Compiling …` progress lines whose ordering varies from run to run, the excerpt below is the **head** and **tail** of that transcript (labelled as an excerpt), not a line‑for‑line reproduction. Head (the command, immediately followed by its first output lines):

```console
$ export PATH=/usr/local/go/bin:$PATH
$ ./dev.sh build
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
```

Tail of the same 135‑line transcript (the failing compile and the process exit; the final `gcc` invocation line and interleaved parallel‑progress lines are part of the full transcript and vary between runs — this is an excerpt of the diagnostic and exit lines):

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

**Exit status `1`.** This is not a source defect: the prebuilt dependency bundle ships a newer `wayland-protocols` header whose `xdg-toplevel` enum has `XDG_TOPLEVEL_STATE_CONSTRAINED_*` values that the pinned vendored GLFW's `switch` does not enumerate, and the build compiles with `-Werror=switch`. The **non‑canonical fallback** used for every subsequent step is Kitty's own `setup.py` switch **`--ignore-compiler-warnings`** (it sets `werror=''`; a build flag, **not** a source edit). Its transcript is **130 lines**; head (command immediately followed by output):

```console
$ ./dev.sh build --ignore-compiler-warnings
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
```

Tail (the link steps and success line):

```console
[122/122] Compiling kitty/gl-wrapper.c ...
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
Build successful. Run kitty as: kitty/launcher/kitty
```

**Exit status `0`.** This was a genuine **full rebuild** (the transcript's own counter reaches `[122/122]` `Compiling` and `[5/5]` `Linking`), not an incremental no‑op. The canonical launcher artifact is **`kitty/launcher/kitty`** (`docs/build.rst:22`). Prerequisites observed in the environment: `go version go1.22.12 linux/amd64` (satisfies `go 1.22`, `go.mod:3`) and `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` (a C11 compiler); Python `>=3.8` is the documented runtime floor (`pyproject.toml:2`).

> **Canonicality note.** Because the canonical `./dev.sh build` fails in this toolchain, the binary under test was produced by the non‑canonical `--ignore-compiler-warnings` build flag. The *runtime behavior and version banner* reported below are nonetheless those of a default build of this source (the flag only relaxes `-Werror`, changing no code path); this is stated so the reader is not misled into thinking the canonical command succeeded here.

### 2.2 Version banner (verbatim, byte‑exact — supplementary run)

Command immediately followed by its output:

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

The banner is assembled by `version()` (`kitty/cli.py:486-492`, format string `'{} {}{} created by {}'`); the `0.35.2` comes from `version: Version = Version(0, 35, 2)` (`kitty/constants.py:25`). *(On an interactive TTY the same function adds italic/green SGR styling and a `rev` field — `inferred` from `kitty/cli.py:486-492`; here the output is non‑TTY, so neither appears, which is what the `od -c` bytes confirm.)*

### 2.3 Headless display

Kitty renders exclusively through OpenGL with **no CPU fallback** (`gl_init` calls `fatal(...)` if a GL context or the required version/extension is unavailable — `kitty/gl.c:53-75`), so a display/GPU is required. The environment is headless, so a virtual X display was used (the harness allocates a *unique, unused* display number per run, e.g. `:200`, and tears it down afterward). Command immediately followed by its effect:

```console
$ Xvfb :200 -screen 0 1280x800x24 +extension GLX +render -noreset &
$ export DISPLAY=:200
Xvfb up on :200 (pid 237369)
```

Mesa's software GL (llvmpipe) provides OpenGL 4.5 here, which comfortably satisfies Kitty's minimum (**GL 3.1 on Linux**, GL 3.3 on Apple, plus the `ARB_texture_storage` extension — `kitty/data-types.h:19-25`, `kitty/gl.c:63-67`).

### 2.4 Launch with Kitty's own tracing flags, and where each stream goes

The harness launches the already‑built launcher like this (command shown immediately before the stream it produces is discussed):

```console
$ stdbuf -oL -eL ./kitty/launcher/kitty --debug-input --debug-rendering \
        --dump-bytes "$WORKDIR/kdump.bin" -o cursor_blink_interval=0 \
        > "$WORKDIR/out.log" 2> "$WORKDIR/err.log" &
```

**Configuration labelling (canonical vs. one display‑only override).** The default shell under Kitty is `/bin/bash`, and there is no `~/.config/kitty/kitty.conf`, so this runs Kitty's **default shell and default configuration** — with exactly **one non‑default, display‑only override**: `-o cursor_blink_interval=0`. That override only stops the cursor blink so the before/after framebuffer capture in §5 is deterministic; it does **not** touch the input path. `stdbuf -oL -eL` is only a *flushing* aid; it does not change Kitty's behavior. So this is *not* a wholly untouched canonical launch — it is the default shell + default config **plus one display‑only cursor‑blink override**, stated here so the reader is not misled.

> **Privacy reminder (repeat of the note above, at the point of launch).** With these flags, everything you type is recorded: the *key text* to `err.log` (stderr), the child's *echo bytes* to the `--dump-bytes` file, and the *parsed commands* to `out.log` (stdout — see below). Never type passwords or other secrets during a traced session; keep the logs outside any repository (the harness uses `umask 077` and a private `mktemp -d`) and delete them afterward.

These flags are Kitty's native, documented tracing hooks (all declared in `kitty/cli.py`), quoted **verbatim** from the source. The final column reports the stream(s) on which each flag's output was **actually observed** in this run:

| Flag | Help text (verbatim, `kitty/cli.py`) | Declared at | Stream(s) where output was observed |
|---|---|---|---|
| `--debug-input` / `--debug-keyboard` | "Print out key and mouse events as they are received." | `kitty/cli.py:996` | **STDERR** (timestamped reception/encoding traces) |
| `--debug-rendering` / `--debug-gl` | "Debug rendering commands. This will cause all OpenGL calls to check for errors instead of ignoring them. Also prints out miscellaneous debug information. Useful when debugging rendering problems." | `kitty/cli.py:989` | **STDOUT** (one GL‑version line) |
| `--dump-bytes <path>` | "Path to file in which to store the raw bytes received from the child process." | `kitty/cli.py:985` | the **named file** (raw child bytes) **and STDOUT** (parsed commands — see the correction below) |
| `--dump-commands` | "Output commands received from child process to STDOUT." | `kitty/cli.py:972` | STDOUT (this flag was **not** on our command line — but its effect is present anyway; see below) |

**Correction — `--dump-bytes` alone also prints parsed commands to STDOUT.** The command dumper is installed whenever **either** dump flag is set: the `ChildMonitor(...)` constructor is passed `DumpCommands(args) if args.dump_commands or args.dump_bytes else None` as its command-dumper argument (`kitty/boss.py:372`, the argument spanning `kitty/boss.py:370-374`). `DumpCommands.__call__` writes the raw bytes to the file **only** when `what == 'bytes'`, and for **every other** parsed command it calls `safe_print(...)` — i.e. it prints the parsed command name and arguments to **STDOUT** (`kitty/boss.py:239-252`). `safe_print` wraps Python's `print()` onto `sys.stdout` (`kitty/utils.py`), which is **block‑buffered** when stdout is redirected to a file, so these lines appear **only after Kitty flushes/exits**. That is exactly what the captured `out.log` shows after the process is stopped — **190** total lines = **1** GL‑version line + **134** parsed‑command lines (the remainder are blanks/continuations):

```console
$ wc -l out.log ; then classify each line
out1.log total lines=190; GL-version lines=1; parsed-command lines=134
```

The single GL line on STDOUT (verbatim, `cat -v`):

```text
[0.128] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

The first twelve **non‑GL** STDOUT lines — the parsed commands emitted by `DumpCommands` (`kitty/boss.py:248-252`), verbatim `cat -v`:

```text
handle_remote_print aWdub3JlYm90aCBvciBpZ25vcmVzcGFjZSBwcmVzZW50IGluIGJhc2ggSElTVENPTlRST0wgc2V0dGluZywgc2hvd2luZyBydW5uaW5nIGNvbW1hbmQgd2lsbCBub3QgYmUgcm9idXN0Cg==}
ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
Got XkbNewKeyboardNotify event with changes: key codes: 1 geometry: 1 device id: 0
process_cwd_notification 7 kitty-shell-cwd://reverse-code-generator-f2f188bd-pbq24/tmp/blitzy/kitty/blitzy-4f08959c-7562-4d36-af17-e80ca3f1005a_a77559
screen_set_mode 2004 1
screen_delete_characters 59
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
set_title root@reverse-code-generator-f2f188bd-pbq24: /tmp/blitzy/kitty/blitzy-4f08959c-7562-4d36-af17-e80ca3f1005a_a77559
set_icon root@reverse-code-generator-f2f188bd-pbq24: /tmp/blitzy/kitty/blitzy-4f08959c-7562-4d36-af17-e80ca3f1005a_a77559
```

So the earlier intuition that "stdout only carries the GL line" is wrong: `--dump-bytes` turns on the same `DumpCommands` object, and its parsed‑command output shares STDOUT with the GL line (the GL line is a C `printf` and is line‑buffered via `stdbuf -oL`; the command lines are Python `print`/`safe_print` and are block‑buffered, hence visible only after flush/exit).

**Stream‑attribution mechanics for the reception traces (verified in source and matched by observation).** The `--debug-input` reception traces are emitted through the `debug` macro, which in `kitty/keys.c` resolves to `debug_input` (`kitty/keys.h:16`), and `debug_input` expands to `if (OPT(debug_keyboard)) { timed_debug_print(...); }` (`kitty/state.h:15`). `timed_debug_print` writes to **stderr** — `fprintf(stderr, "[%.3f] ", ...)` then `vfprintf(stderr, ...)` (`kitty/monotonic.h:102`/`105`) — which is why every such line is prefixed with a `[seconds]` timestamp and appears on STDERR. The single GL‑version diagnostic is printed with `printf(...)` → **STDOUT** (`kitty/gl.c:72`). This matches the capture exactly (reception on STDERR, GL line + parsed commands on STDOUT).

> In the raw captures below, `^[` is the ESC byte (`0x1b`) as rendered by `cat -v`; sequences such as `^[[33m … ^[[m` are ANSI SGR color codes that Kitty's debug prints wrap around their labels (e.g. `kitty/keys.c:176` emits a label wrapped in `\x1b[33m … \x1b[m`). They are part of the genuine, unedited output. **Do not confuse** this `cat -v` rendering of *real* ESC bytes with the *literal* `^[ ` token that the encoded‑key debug formatter prints — see §4.3.

### 2.5 Why this is the real path, not a bypass

The debug flags are wired into the *genuine* input pipeline at GLFW‑init time (`init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)`, `kitty/main.py:514`); enabling them only **annotates** the real path, it does not reroute input. Keys were injected with X11 `XTEST` (via `xdotool`), which are **synthetic X‑server key events**, *not* physical hardware presses — but they are delivered to the X server and travel the **complete, genuine software path**: X server → the vendored GLFW fork's XKB decoder (`glfw/xkb_glfw.c`) → GLFW key event → `key_callback` → `on_key_input`. This is emphatically **not** remote control (`kitty @ send-text`) and **not** injection into Kitty's internals, both of which would bypass reception. Where the text below says "the same key," it means the **same logical key** (identical X keycode), reproduced across separate presses.

---

## 3. Stage 1 — Reception: which parts receive the input first

**Answer:** the **platform/GLFW layer receives the keystroke first**. On Linux the XKB code inside the vendored GLFW fork translates the keycode and hands a GLFW key event to Kitty's `key_callback`, which immediately forwards it to `on_key_input`, the first Kitty‑owned function to see the event.

**Observed evidence.** With `--debug-input`, pressing `l`, `s`, Return produced the **reception‑related** STDERR lines below — the platform XKB feeder line and the Kitty `on_key_input` line — shown **in emission order** and **byte‑verbatim** (`cat -v`; no field within any shown line is altered). The harness selects these from the raw STDERR log with `grep -aE 'xkb_keycode|on_key_input'` (see Appendix), so unrelated startup/render lines are excluded while the XKB and `on_key_input` lines stay interleaved in the exact order they were logged. For brevity the excerpt keeps the full PRESS+RELEASE pair only for the first key (`l`); for `s` and Return only the PRESS pair is shown. The elided `s` and Return **RELEASE** pairs are identical in form to the shown `l` RELEASE pair (`… action: RELEASE … ignoring as keyboard mode does not support encoding this event`) and add no new information — the harness's own `head -12` step emits all twelve lines:

```text
[1.707] ^[[31mPress^[[m xkb_keycode: 0x2e clean_sym: l composed_sym: l text: l mods: none glfw_key: 108 (l) xkb_key: 108 (l)
[1.707] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[1.708] ^[[32mRelease^[[m xkb_keycode: 0x2e clean_sym: l mods: none glfw_key: 108 (l) xkb_key: 108 (l)
[1.708] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.717] ^[[31mPress^[[m xkb_keycode: 0x27 clean_sym: s composed_sym: s text: s mods: none glfw_key: 115 (s) xkb_key: 115 (s)
[1.717] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[1.731] ^[[31mPress^[[m xkb_keycode: 0x24 clean_sym: Return composed_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[1.731] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
```

- The **XKB line** (`Press xkb_keycode: 0x2e …`) decodes the raw keycode (`xkb_keycode: 0x2e`) into a symbol (`clean_sym: l`); it is the Linux/X11 keymap feeder for GLFW and prints **before** any Kitty code runs for that key. *(That this specific line is emitted by `glfw_xkb_handle_key_event` at `glfw/xkb_glfw.c:875` is `inferred` from source; what is **observed** is that an XKB decode line for the keycode prints first.)*
- The **`on_key_input` line** immediately follows, emitted by `on_key_input` (`kitty/keys.c:166`) under `if (OPT(debug_keyboard))` (`kitty/keys.c:172`); its label/format string is at `kitty/keys.c:176`.
- Both lines carry the **same `[seconds]` timestamp** (`[1.707]` for `l`, `[1.717]` for `s`, `[1.731]` for Return), with the XKB line first — the empirical proof that the platform/GLFW layer receives the event and *then* calls into Kitty.

**How the event reaches `on_key_input` (the call chain — `inferred` from source).** GLFW's keyboard callback is registered once at window creation — `glfwSetKeyboardCallback(glfw_window, key_callback)` (`kitty/glfw.c:1292`) — and `key_callback` (`kitty/glfw.c:430`) forwards the event with `on_key_input(ev)` (`kitty/glfw.c:439`). *(These specific call sites are `inferred` from source; what is observed is the same‑timestamp ordering above and the startup trace below.)* The startup trace (verbatim `cat -v`) shows the platform keymap being loaded before any key is pressed:

```text
[0.057] Loading new XKB keymaps
[0.065] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.150] OS Window created
[0.174] Failed to open systemd user bus with error: Connection refused
[0.178] Child launched
[0.178] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
```

("Loading new XKB keymaps" and "Modifier indices" are the XKB keymap loader in `glfw/xkb_glfw.c`; the "Failed to open systemd user bus" line is benign — `DBUS_SESSION_BUS_ADDRESS` is `/dev/null` in this headless box.) On macOS/Cocoa and Wayland the feeder differs (Cocoa, or `glfw/ibus_glfw.c` for IME), but entry into Kitty is the same `key_callback` → `on_key_input` chain — **`inferred`** for the non‑X11 platforms, since only Linux/X11 was exercised here.

**Thread note (`inferred`).** `key_callback`/`on_key_input` run on Kitty's **main thread** (GLFW delivers window/keyboard callbacks on the thread that owns the window and pumps the event loop), so reception, encoding, parsing, and rendering are all main‑thread work; only the PTY byte read is offloaded (see §4.4). This thread attribution is `inferred` from the GLFW event‑loop design; what is observed is the trace ordering.

---

## 4. Stage 2 — Intermediate processing: what happens between reception and the screen update

Two byte streams cross the pseudo‑terminal (PTY) boundary, and both are part of the intermediate stage.

### 4.1 Terminal → child (the keystroke is interpreted, encoded, and written to the shell)

Inside `on_key_input`, three things happen; the first two emit no separate trace, so their **order** is `inferred` from `kitty/keys.c:226`–`271`, while the **final disposition** of each key is **observed** on its one continued STDERR line:

1. **Shortcut check (PRESS/REPEAT only) — `inferred`.** For PRESS or REPEAT actions — the guard `if (action == GLFW_PRESS || action == GLFW_REPEAT)` at `kitty/keys.c:226` — `on_key_input` dispatches into the Python layer via `dispatch_possible_special_key` (`kitty/keys.c:228`; the Python side is `kitty/keys.py:154`) to see whether the key is a mapped shortcut. When it is not consumed, processing continues to encoding.
2. **Encoding (legacy vs. protocol) — `inferred`.** `encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)` (`kitty/keys.c:251`, implemented in `kitty/key_encoding.c`) decides how the key is serialized. Its return value selects one of three dispositions.
3. **Write to child (through the PTY) — `inferred` mechanism, observed result.** The bytes are queued with `schedule_write_to_child` — at `kitty/keys.c:253` for the literal‑text disposition and `kitty/keys.c:259` for the encoded disposition (both implemented at `kitty/child-monitor.c:372`).

The three dispositions were all **observed**, each with its own trailing debug string (verbatim `cat -v`; note the encoded‑key formatter prints each byte followed by a space, so a trailing space is genuine):

```text
# plain letter 'l'  ->  written to the child as literal text  (schedule_write_to_child @ kitty/keys.c:253; debug string @ :254)
[1.707] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l

# Enter  ->  ENCODED to a single carriage-return byte 0x0d  (schedule_write_to_child @ :259; debug string @ :261)
[1.731] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 

# key RELEASE in the shell's legacy mode  ->  not encoded at all  (debug string @ :271)
[1.708] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
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

The file is opened empty with mode `'wb'` by `DumpCommands` (`kitty/boss.py:237`, installed at `kitty/boss.py:372`), and `DumpCommands.__call__` writes and flushes on `what == 'bytes'` (`kitty/boss.py:243-244`). The first 48 bytes of the delta (genuine `hexdump -C`, command shown first) begin with the echoed characters:

```console
$ tail -c +720 kdump.bin | head -c 48 | hexdump -C
00000000  6c 73 0d 0a 1b 5b 3f 32  30 30 34 6c 0d 1b 5d 32  |ls...[?2004l..]2|
00000010  3b 6c 73 07 1b 5d 31 33  33 3b 43 3b 63 6d 64 6c  |;ls..]133;C;cmdl|
00000020  69 6e 65 3d 6c 73 07 01  1b 5d 31 33 33 3b 6b 3b  |ine=ls...]133;k;|
```

`6c 73 0d 0a` is `l s CR LF` — the shell echoing the two letters and the newline — followed by control sequences (`\x1b[?2004l` bracketed‑paste‑off, an OSC‑2 title `]2;ls`, an OSC‑133 `]133;C;cmdline=ls` shell‑integration mark) and then the directory listing. This confirms `--dump-bytes` captures the **child → terminal** stream — precisely the bytes that feed the VT parser.

### 4.3 A note on byte semantics (two different renderings of ESC)

The captures contain ESC bytes represented **two different ways**, and they must not be conflated:

- In the **dump file** and in raw stderr, `cat -v` renders a *real* ESC byte (`0x1b`) as `^[` with **no** following space; e.g. the protocol opt‑in in §6.4 appears as `^[[>31u` (bytes `1b 5b 3e 33 31 75`).
- In the **`on_key_input` encoded‑key debug line**, the formatter prints a *literal* token `^[ ` (caret, bracket, **space**) for each ESC byte, and each subsequent byte followed by a space — because the loop at `kitty/keys.c:263` is `if (encoded_key[ki] == 27) { debug("^[ "); } … else { debug("%c ", …); }`. That is why the same CSI‑u sequence shows up as space‑separated `^[ [ 9 7 ; ; 9 7 u ` on the debug line but as unspaced `^[[97…u` when the *bytes themselves* are rendered by `cat -v`. The spacing (including the trailing space) is the tell: it is Kitty's bespoke debug formatter, not the wire bytes.

### 4.4 Read → parse → screen (which thread does what)

The child's output is **read** off the PTY by the **I/O thread** loop `io_loop` (`kitty/child-monitor.c:1481`), which calls `read_bytes` (`kitty/child-monitor.c:1337`); `read_bytes` only copies bytes into the VT‑parser's write buffer — it does **not** parse. Parsing happens on the **main thread**: `process_global_state` (`kitty/child-monitor.c:1224`, the main‑loop tick) calls `parse_input` (`kitty/child-monitor.c:451`, whose own comment reads "Parse all available input that was read in the I/O thread") at `:1236`, and *then* calls `render` at `:1237` — sequentially. When byte‑dumping is enabled the parse worker is `parse_worker_dump` (`kitty/child-monitor.c:180`, vs `parse_worker` at `:181`). The VT state machine classifies each byte: printable text through `consume_normal` (`kitty/vt-parser.c:230`), escapes through `consume_esc` (`:261`), CSI sequences through `consume_csi` (`:839`), dispatched by `dispatch` (`:413`). Printable text is written into the screen model by `screen_draw_text` (`kitty/screen.c:866`) / `draw_codepoint` (`kitty/screen.c:872`), backed by the line/cell buffers (`kitty/line.c`, `kitty/line-buf.c`), scrollback (`kitty/history.c`), and cursor (`kitty/cursor.c`).

**What is observed vs `inferred` here.** *Observed:* the dumped bytes (the `ls` echo in §4.2) are exactly what a terminal must parse to update its grid. *`inferred` from source:* the read‑on‑I/O‑thread vs parse‑on‑main‑thread split, and the specific per‑function transitions **inside** the VT parser and screen model — the parser exposes no per‑byte trace flag that was exercised here, so those function‑level attributions rest on the cited source, not on a captured trace.

---

## 5. Stage 3 — Display production: how the updated display is ultimately produced

**Answer:** the updated screen is produced by a **main‑thread render tick that composites the screen model on the GPU via OpenGL and presents the frame by swapping the window's buffers** — there is no CPU rendering fallback.

**What was directly observed (context creation).** With `--debug-rendering`, exactly **one** rendering diagnostic appeared, on **STDOUT**, at startup, and it reproduced on every launch (verbatim `cat -v`):

```text
[0.128] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

This is printed by `printf("[%.3f] GL version string: %s\n", …)` inside `gl_init` (`kitty/gl.c:72`, guarded by `global_state.debug_rendering`). Its presence proves Kitty successfully created an **OpenGL context** (here Mesa's software GL 4.5, which meets Kitty's minimum). Crucially, `--debug-rendering` emitted **no per‑frame render/swap/shader traces** in this build — so the observed rendering signal is bounded to **context initialization**, not to per‑frame drawing.

**Direct evidence that the display update is caused by the input (before / after of the *same* window).** Because there is no per‑frame trace, display production was confirmed by capturing the framebuffer of the **same** running window **before** and **after** typing `ls`+Enter and measuring the change. Commands shown immediately before their output:

```console
$ import -window root frame_before.png     # clean prompt, before typing
distinct colors BEFORE: 164
sha256 BEFORE:          d6f866ace4146e355e059bb5cfd5fbe7d2e668afb88a9a1ddefb2e297cd25b16

$ xdotool key --clearmodifiers l ; s ; Return      # drive the real keystrokes
$ import -window root frame_after.png      # same window, after typing
distinct colors AFTER:  487 (a blank screen would be 1)
sha256 AFTER:           2b5e30037a6497f93192e0de8a18ce5cc931df069024a54969d47c1b191fcab1
non-background bounding box (glyph region) AFTER: 1280x800+0+5

$ compare -metric AE frame_before.png frame_after.png : changed pixels = 22420 (0.0218945)
```

The framebuffer **changed as a direct result of the keystrokes**: the distinct‑color count rose from **164 → 487**, the `sha256` of the whole framebuffer changed (`d6f866ac… → 2b5e3003…`), and ImageMagick's `compare` counted **22 420** changed pixels. A blank or non‑updating window would show a single color and a zero pixel delta. Since Kitty has **no CPU rendering fallback** — `gl_init` aborts via `fatal(...)` if the GL context, version, or `ARB_texture_storage` extension is missing (`kitty/gl.c:53-75`) — a populated framebuffer that *transitions in response to typing* is itself evidence that the **GPU/OpenGL** path ran and produced the update.

**The render call chain (source‑grounded `inferred`, since no per‑frame trace was emitted).** The main thread runs the loop tick `process_global_state` (`kitty/child-monitor.c:1224`), which calls `render(now, input_read)` at `:1237`; `render` is defined at `kitty/child-monitor.c:871`. It composites each OS window through `render_os_window` (`:833`), preparing the frame with `prepare_to_render_os_window` (`:705`), drawing it with `render_prepared_os_window` (`:788`), and presenting it with **`swap_window_buffers(os_window)`** (`:810`). The pixel work is done by the OpenGL infrastructure in `kitty/gl.c` and the shader programs in `kitty/shaders.c` together with the **13** GLSL sources in `kitty/` (`ls kitty/*.glsl | wc -l` → `13`): `alpha_blend`, `bgimage_fragment`, `bgimage_vertex`, `border_fragment`, `border_vertex`, `cell_defines`, `cell_fragment`, `cell_vertex`, `graphics_fragment`, `graphics_vertex`, `linear2srgb`, `tint_fragment`, `tint_vertex`. This precise call/shader chain is labelled **`inferred`**: what was *observed* is (a) successful GL context creation and (b) a framebuffer that transitions in response to real input.

### 5.1 Concurrency (what actually runs in parallel)

The three stages are a **logical** ordering of a keystroke's data. In terms of threads, only the **PTY read/write** is offloaded: the child's bytes are read on an **I/O thread** (`io_loop`, `kitty/child-monitor.c:1481`), while **parsing and rendering both run on the main thread, sequentially** (`parse_input` at `:1236` immediately followed by `render` at `:1237`) — this sequential‑on‑one‑thread ordering is `inferred` from those two adjacent call sites. The project documentation states the offload directly: "Interaction with child programs takes place in a separate thread from rendering, to improve smoothness." (`docs/performance.rst:8-9`), and the read/parse handoff is decoupled by a small delay — `input_delay` (default **3 ms**, `docs/performance.rst:48`). Consistent with that design, the reception/parse activity (STDERR) and the render diagnostic (STDOUT) were **observed** as concurrent streams, not strictly interleaved per keystroke — but the *parse→render* step for a given batch is sequential on one thread, so this is **not** a "parse thread vs render thread" split.

---

## 6. Conditions exercised (raw output beside each)

All keys were injected with `xdotool` (X `XTEST`) into the focused Kitty window (its **X11** window id `2097164` in this run, used as the `xdotool` target and PID‑verified before typing — this is distinct from Kitty's **internal** window id `0x1` shown in the `on_focus_change` trace of §3). The `xkb_keycode` visible in the reception traces (§3) confirms the genuine X → XKB → GLFW route. Traces below are **byte‑verbatim** (`cat -v`, single space between fields, exactly as captured); a separate readability‑normalized table is offered only in §6.4 and is labelled as such.

### 6.1 Condition 1 — Unmodified simple letters (`l`, `s`, Enter)

Command: `xdotool key --clearmodifiers l ; s ; Return`. Full genuine PRESS traces are in §3; letters go to the child as literal text, Enter (`glfw key: 0xe001`, native X keysym `0xff0d`) is encoded to `0x0d`. Dump delta = 1580 bytes beginning `6c 73 0d 0a` (`ls\r\n`, §4.2).

### 6.2 Condition 2 — Modifier combinations (Ctrl‑C, Shift‑a, Alt‑a), legacy shell

Command: `xdotool key --clearmodifiers ctrl+c ; shift+a ; alt+a`. The full unedited block (byte‑verbatim `cat -v`; the modifier key itself arrives first and is dropped, then the letter carries the `mods` field):

```text
[3.151] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.157] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3 
[3.163] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.169] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.482] ^[[33mon_key_input^[[m: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.488] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
[3.494] ^[[33mon_key_input^[[m: glfw key: 0xe061 native_code: 0xffe1 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.501] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.814] ^[[33mon_key_input^[[m: glfw key: 0xe063 native_code: 0xffe9 action: PRESS mods: alt text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.820] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ a 
[3.826] ^[[33mon_key_input^[[m: glfw key: 0xe063 native_code: 0xffe9 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.832] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

So `mods` takes values `ctrl` / `shift` / `alt` (vs `none` in Condition 1), and the emitted bytes are `0x3` (Ctrl‑C, i.e. ETX/SIGINT), literal `A` (Shift‑a, composed to uppercase and sent as text), and ESC + `a` (Alt‑a — printed as the literal token `^[ a ` per §4.3, i.e. bytes `1b 61`). In each modifier combo the modifier key *itself* (`0xe062` ctrl, `0xe061` shift, `0xe063` alt) arrives first and is dropped with "ignoring …".

### 6.3 Condition 3 — Press / repeat / release lifecycle (before / during / after of one key)

Command: `xdotool keydown j; sleep 0.8; xdotool keyup j` (with `xset r rate 250 30`). A single held key produced **1 × PRESS**, **18 × REPEAT**, **1 × RELEASE**. The per‑action tally for the held key `0x6a` (byte‑verbatim):

```text
      1 action: PRESS
      1 action: RELEASE
     18 action: REPEAT
```

The first four genuine events (byte‑verbatim `cat -v`):

```text
[4.349] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: PRESS mods: none text: 'j' state: 0 sent key as text to child: j
[4.600] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[4.633] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[4.666] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
```

The 18 REPEAT lines are byte‑for‑byte identical except for the `[seconds]` timestamp (≈33 ms apart, `[4.600]` → `[5.163]`). The final two genuine events:

```text
[5.163] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[5.169] ^[[33mon_key_input^[[m: glfw key: 0x6a native_code: 0x6a action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

The whole‑run action‑field tally (all keys typed in the main run) is consistent — 10 discrete key presses and releases plus the 18 repeats of the held key:

```text
     10 action: PRESS
     10 action: RELEASE
     18 action: REPEAT
```

The `action:` field (`kitty/keys.c:176`) thus takes all three values — the full lifecycle of one key. Each PRESS/REPEAT sends `j`; the RELEASE is dropped in legacy mode.

### 6.4 Condition 4 — Legacy vs. Kitty Keyboard Protocol, plus the `-m normal` helper (same logical key `a`)

The default shell uses **legacy** encoding (the canonical, primary case). To exercise the encodings without a bypass, the `kitten show-key` helper was run *inside* the window in two modes: **`-m normal`** (a normal/legacy‑mode helper) and **`-m kitty`** (which opts into the Kitty Keyboard Protocol via progressive enhancement). Both are real keystrokes into a real child program.

**Helper A — `kitten show-key -m normal` (normal/legacy mode).** Kitty's send‑side `on_key_input` traces for `a` and Ctrl‑`a` (byte‑verbatim `cat -v`):

```text
[3.336] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[3.357] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.755] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.761] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x1 
```

and the **child → terminal** dump under `-m normal`, printable rendering of the received bytes (`cat -v`) — `a` arrives as literal `a`, Ctrl‑`a` as `^A` (`0x01`):

```text
a a a ^A a ^A ^A a ^A ^A a a a a a ^A 
```

**Helper B — `kitten show-key -m kitty` (Kitty Keyboard Protocol).** This kitten enables the **full** protocol, flags **31** (`FULL_KEYBOARD_PROTOCOL = DISAMBIGUATE_KEYS(1) | REPORT_KEY_EVENT_TYPES(2) | REPORT_ALTERNATE_KEYS(4) | REPORT_ALL_KEYS_AS_ESCAPE_CODES(8) | REPORT_TEXT_WITH_KEYS(16) = 31` — this flag arithmetic is `inferred` from `tools/tui/loop/terminal-state.go:15-21`; used by `kittens/show_key/kitty.go:20`). The opt‑in and restore bytes were **observed** in the child→terminal dump (genuine `cat -v`, counted):

```text
      1 [>31u        # push:  ESC[>31u   (opt in, flags 31)   — tools/tui/loop/terminal-state.go:140
      1 [<u          # pop:   ESC[<u     (restore on exit)    — tools/tui/loop/terminal-state.go:160
```

(These render as `^[[>31u` / `^[[<u` with the ESC byte; the `^[` prefix is trimmed by the counting `grep`.) With the protocol active, the **same logical key `a`** encodes completely differently. Byte‑verbatim `on_key_input` output under the protocol (note the space‑separated `^[ [ …` and trailing space are the debug formatter of §4.3):

```text
[2.239] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent encoded key to child: ^[ [ 9 7 ; ; 9 7 u 
[2.246] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 1 : 3 u 
[2.657] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: ^[ [ 5 7 4 4 2 ; 5 u 
[2.663] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 5 u 
[2.669] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 sent encoded key to child: ^[ [ 5 7 4 4 2 ; 1 : 3 u 
[2.676] ^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 1 : 3 u 
[3.088] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: ^[ [ 5 7 4 4 2 ; 5 u 
[3.094] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: ^[ [ 9 9 ; 5 u 
[3.100] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.106] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

**Readability‑normalized comparison (secondary — derived from the byte‑verbatim lines above; not itself a capture):**

| Same logical key `a` | `-m normal` / legacy (default shell) | Kitty Keyboard Protocol (`show-key -m kitty`) |
|---|---|---|
| press | literal `a` (`0x61`) | `ESC[97;;97u` (CSI‑u; `97` = codepoint of `a`, plus associated text `97`) |
| release | **not encoded** (dropped) | `ESC[97;1:3u` (release **is** reported; event‑type `:3`) |
| Ctrl‑`a` | `0x1` (SOH) | `ESC[97;5u` (modifier field `5` = Ctrl) |
| Ctrl‑`c` | `0x3` (§6.2) | `ESC[99;5u` (`99` = codepoint of `c`, modifier `5` = Ctrl) |
| Ctrl key itself | (dropped) | `ESC[57442;5u` (the Control key is reported too) |

The branch is chosen by `encode_glfw_key_event` from `screen_current_key_encoding_flags(screen)` (`kitty/keys.c:251`); the protocol's push/pop grammar is documented at `docs/keyboard-protocol.rst:293-297` (`inferred` grammar reference). **Legacy is the canonical primary evidence** because it is what the default shell uses; the two‑run stability of the protocol traces is shown in §7.

### 6.5 Condition 5 — `--dump-bytes` before / during / after

```text
before launch                : dump  absent
after launch, before typing  : dump  719 bytes   (shell startup prompt; kitty/boss.py:237 opens it 'wb')
after all legacy typing      : dump  2781 bytes
```

The bytes appended right after typing begin `6c 73 0d 0a` = `ls\r\n` (the shell **echo**), confirming that `--dump-bytes` captures the **child → terminal** stream — precisely the bytes that feed the VT parser and update the screen (mechanism: `kitty/boss.py:243-244`). Full hex of the first delta is in §4.2.

### 6.6 Condition 6 — Ctrl‑D / EOT on an empty command line (error/exit edge)

Command: `xdotool key --clearmodifiers ctrl+d` on an empty command line, in a **fresh** Kitty instance. Ctrl‑D sends the End‑Of‑Transmission byte `0x04`, which `bash` reads as end‑of‑file on an empty line and exits, closing the window. Byte‑verbatim `on_key_input` (the Ctrl key itself is dropped, then `d`+ctrl encodes to `0x4`):

```text
[1.226] ^[[33mon_key_input^[[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.232] ^[[33mon_key_input^[[m: glfw key: 0x64 native_code: 0x64 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x4 
```

Before / during / after of the changing values (dump size and process state), commands shown before output:

```text
dump size BEFORE Ctrl-D: 719 bytes
dump size AFTER Ctrl-D: 736 bytes (delta 17 bytes)
launcher exit status after Ctrl-D EOF: 0
kitty window still present after Ctrl-D: no
```

The child→terminal dump tail after Ctrl‑D shows the shell's logout output (`cat -v`) — bracketed‑paste‑off then the shell's `exit`:

```text
^[[?2004l^M^M
exit^M
```

So Ctrl‑D encodes to a single `0x4` byte, the shell reaches EOF and prints `exit`, the launcher exits cleanly with status **0**, and the GUI window is gone — the full receive → encode → child‑exit → window‑close lifecycle of the terminating keystroke.

### 6.7 Condition 7 — Edge branches (focus loss observed; no‑active‑window `inferred`)

**7a — Focus loss (observed distribution).** After moving X input focus away from the Kitty window and injecting three keys (`x`, `y`, `z`), Kitty received **none** of them (commands and output):

```text
$ xdotool windowfocus --sync 0 ; xdotool key x ; xdotool key y ; xdotool key z
on_key_input count before=38; after 3 keys while UNFOCUSED=38; delivered=0 of 3
focus-change events observed:
[0.178] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 1
[5.705] ^[[35mon_focus_change^[[m: window id: 0x1 focused: 0
```

The `on_key_input` counter did not advance (38 → 38), and the `on_focus_change … focused: 0` line marks the transition — direct evidence that, unfocused, no key events reach Kitty (they go to whatever the X server considers focused).

**7b — No active window (`inferred`).** If a key event arrives with no active window, `on_key_input` logs `no active window, ignoring` and returns (`kitty/keys.c:182`). This branch was **not** triggered — `grep -c "no active window, ignoring"` over the captured stderr returned `0`, because a window was always present in the single‑window run:

```text
'no active window, ignoring' occurrences this run: 0
```

It is therefore reported as **`inferred`** from the source, not observed.

---

## 7. Stability (reproduced across fresh launches — legacy and protocol)

**Legacy (two fresh processes).** The identical input `ls`+Enter was driven in **two separate, freshly launched** Kitty processes. The reception traces reproduced exactly — same `glfw key` codes, same native codes, same dispositions, same `mods` — differing only in the `[seconds]` timestamp:

```text
# RUN 1 (PRESS lines)
[1.707] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[1.717] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[1.731] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 

# RUN 2 (fresh process, PRESS lines)
[1.117] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[1.132] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[1.147] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
```

The harness's own diff of these lines (timestamps stripped) is empty, and it reports:

```text
RESULT: legacy reception traces IDENTICAL across the two fresh runs (timestamps aside)
```

The dump echo reproduced identically (`6c 73 0d 0a` = `ls\r\n`; 719 → 2299 bytes on both runs), and the GL diagnostic matched (`GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`; `[0.128]` on run 1, `[0.125]` on run 2).

**Kitty Keyboard Protocol (two fresh processes).** The `kitten show-key -m kitty` opt‑in and the CSI‑u encodings for `a` were also driven in **two independent, freshly launched** Kitty processes; both opted in (`ESC[>31u` seen) and produced byte‑identical CSI‑u encodings. The harness's timestamp‑stripped diff of the protocol `a` lines reports:

```text
$ RUN4 protocol CSI-u 'a' lines (timestamp-stripped):
^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent encoded key to child: ^[ [ 9 7 ; ; 9 7 u 
^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 1 : 3 u 
^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 5 u 
^[[33mon_key_input^[[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 1 : 3 u 
$ RUN5 protocol CSI-u 'a' lines (timestamp-stripped):   (identical to RUN4)
RESULT: Kitty-protocol traces IDENTICAL across the two fresh runs (timestamps aside)
```

**Result: reproduced across 2 runs for both encodings.** (Methodology note: a single Kitty instance must own the X focus; under a WM‑less `Xvfb` the harness explicitly focuses the PID‑verified window before typing and tears each instance down before starting the next.)

---

## 8. Reasoning / rationale (observed signal → responsible function)

- **Reception is the platform/GLFW layer.** *Observed:* an XKB decode line printing a real `xkb_keycode` **immediately before**, and with the same timestamp as, the `on_key_input` line (`kitty/keys.c:176`) — proving the OS/GLFW layer receives the key first and then calls Kitty. *`inferred` from source:* that the feeder is `glfw_xkb_handle_key_event` (`glfw/xkb_glfw.c:875`) and that entry is via `key_callback` (`kitty/glfw.c:430`) → `on_key_input` (`kitty/glfw.c:439`; handler defined `kitty/keys.c:166`).
- **Intermediate processing is `on_key_input` choosing an encoding and writing to the PTY, then the child's echo being read and parsed into the screen model.** *Observed:* the three trailing dispositions (`sent key as text to child` / `sent encoded key to child` / `ignoring …`, `kitty/keys.c:254`/`261`/`271`) and the `--dump-bytes` echo (`ls\r\n`). *`inferred` from source:* that the disposition is decided by `encode_glfw_key_event` (`kitty/keys.c:251`) after `dispatch_possible_special_key` (`:228`), queued by `schedule_write_to_child` (`:253`/`259`), read by `io_loop`/`read_bytes` (`kitty/child-monitor.c:1481`/`1337`), and parsed by `parse_input` (`:1236`) → `consume_normal` (`kitty/vt-parser.c:230`) → `screen_draw_text` (`kitty/screen.c:866`).
- **Display is produced on the GPU.** *Observed:* the `GL version string` line (`kitty/gl.c:72`) proving an OpenGL context was created, and a framebuffer that **transitions** (164 → 487 colors, 22 420 changed pixels, different `sha256`) in response to typing. *`inferred` from source:* the specific `render` → shader → `swap_window_buffers` chain (`kitty/child-monitor.c:871`/`810`), since `--debug-rendering` emitted no per‑frame trace. Because there is no CPU fallback (`kitty/gl.c:53-75`), a window that updates in response to input is itself evidence the GPU path ran.
- **Concurrency is a read‑offload, not a parse/render split.** *Observed:* the two streams (STDERR reception/parse, STDOUT GL) appeared concurrently. *`inferred` from source:* only the PTY read runs on the I/O thread while `parse_input` (`kitty/child-monitor.c:1236`) and `render` (`:1237`) run sequentially on the main thread; the "separate thread" in `docs/performance.rst:8` refers to the child‑interaction (I/O) thread.

### Notes on entry points and counts (observed reality is authoritative)

- Kitty's Python entry point is the **repository‑root `__main__.py`**; there is **no** `kitty/__main__.py` in this checkout (verified: `ls kitty/__main__.py` → not found).
- There are **13** GLSL shader files in `kitty/` (`ls kitty/*.glsl | wc -l` → `13`), listed in §5.
- The version is the real `--version` output (`kitty 0.35.2 created by Kovid Goyal`); source constant `Version(0, 35, 2)` (`kitty/constants.py:25`).

---

## 9. Coverage pass (all three named sub‑questions answered)

Each item states the answer and separates the **observed** basis from any **`inferred`** attribution.

- [x] **Reception — "which parts appear to receive the input first"** → §3. **Answer:** the OS/platform + GLFW layer receive first, then Kitty's `on_key_input` (`kitty/keys.c:166`). **Observed basis:** the same‑timestamp XKB‑decode‑before‑`on_key_input` capture. **`inferred`:** the specific feeder/callback function names (`glfw/xkb_glfw.c:875`, `kitty/glfw.c:430`).
- [x] **Intermediate processing — "which parts handle the intermediate processing"** → §4. **Answer:** `on_key_input` interprets/encodes the key and writes it to the child via the PTY; the echoed bytes are read on the I/O thread and parsed on the main thread into the screen model. **Observed basis:** the trailing dispositions and the `--dump-bytes` before/during/after echo. **`inferred`:** the internal step ordering, the read‑vs‑parse thread split, and the per‑function VT/screen transitions.
- [x] **Display production — "how the updated display is ultimately produced"** → §5. **Answer:** the main‑thread render tick composites on the GPU (OpenGL) and presents via `swap_window_buffers` (`kitty/child-monitor.c:810`). **Observed basis:** the captured `GL version string` diagnostic (`kitty/gl.c:72`) and the before/after framebuffer transition caused by typing. **`inferred`:** the `render`→shader→`swap` call chain and the 13‑shader composition.
- [x] **Concurrency caveat** stated (§5.1, §8): **observed** that reception/parse (STDERR) and the GL diagnostic (STDOUT) are concurrent; **`inferred`** that only the PTY read is offloaded while parse and render run sequentially on the main thread (`kitty/child-monitor.c:1236-1237`; `docs/performance.rst:8`, `:48`).

---

## Provenance inventory (which run produced each capture)

To remove any ambiguity about where the raw output came from:

| Section / capture | Source run |
|---|---|
| §2.1 bare `./dev.sh build` transcript (135 lines) | **Supplementary build run** (the harness does not build) |
| §2.1 `--ignore-compiler-warnings` transcript (130 lines) | **Supplementary build run** |
| §2.2 `--version` banner + `od -c` (36 bytes) | **Supplementary run** of the built launcher |
| §2.4 `out.log` stream attribution (190/1/134), GL line, parsed commands | Single harness run — RUN 1 (stdout captured after stop) |
| §3 reception adjacency, startup stderr | Single harness run — RUN 1 |
| §4.1 dispositions; §4.2 dump before/during/after + hexdump | Single harness run — RUN 1 |
| §5 GL line; before/after framebuffer (164→487, 22 420 px) | Single harness run — RUN 1 |
| §6.2 modifiers; §6.3 held‑`j` lifecycle; §6.5 dump sizes; §6.7 focus loss / no‑active‑window | Single harness run — RUN 1 |
| §6.4 `-m normal` helper | Single harness run — RUN 3 |
| §6.4 `-m kitty` protocol traces + opt‑in/restore | Single harness run — RUN 4 |
| §6.6 Ctrl‑D / EOT | Single harness run — RUN 6 |
| §7 legacy stability (RUN 1 vs RUN 2) | Single harness run — RUN 1 & RUN 2 |
| §7 protocol stability (RUN 4 vs RUN 5) | Single harness run — RUN 4 & RUN 5 |

Everything in §§3–7 is from **one** execution of the Appendix harness (six fresh Kitty instances, RUN 1–RUN 6, in that single execution). Only the build transcripts and the version banner (§2.1–§2.2) are from separate, explicitly‑labelled supplementary runs, because the harness does not build Kitty.

---

### Appendix — the exact observation harness used (temporary; removed after the investigation)

The script below is the **exact** harness whose single execution produced every §§3–7 capture in this document. It is **read‑only** with respect to Kitty (it only runs the already‑built launcher with Kitty's own trace flags and injects real `XTEST` keys). It is fully self‑cleaning: `set -euo pipefail`, `umask 077`, a private `mktemp -d` working directory, a unique unused X display per run, PID‑verified window ownership, and — importantly — the cleanup `trap` is armed **immediately after the working directory is created and before** the failure‑capable display selection, so no artifact is leaked on any early‑exit path (including "no free display", `exit 4`). It passes `bash -n`. **Privacy:** because it enables `--debug-input` and `--dump-bytes`, the script types only harmless markers and keeps every artifact mode‑`0600` under the private `umask 077` directory outside the repository, deleting them all on exit. Run as `./observe_kitty_input.sh /path/to/kitty-repo`.

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
# and a private Xvfb on a unique display; all of it is torn down on exit -- on
# the success path AND on every early-failure path (the cleanup trap is armed
# immediately after the working directory is created, before any step that can
# fail).
#
# PRIVACY: --debug-input records every keystroke's text to the (stderr) debug
# log, and --dump-bytes plus the command-dump on stdout record the child's
# echo of what you typed. Do NOT type real secrets while tracing. This harness
# uses only harmless markers, keeps every artifact mode 0600 under a private
# umask 077 directory outside the repository, and deletes them all on exit.
#
# Usage:  ./observe_kitty_input.sh /path/to/kitty-repo
set -euo pipefail
umask 077

REPO="${1:-$PWD}"
LAUNCHER="$REPO/kitty/launcher/kitty"
[ -x "$LAUNCHER" ] || { echo "FATAL: launcher not found/executable at $LAUNCHER" >&2; exit 3; }

# --- Create the working dir and ARM CLEANUP FIRST, before anything that can
# --- fail (e.g. display selection). This guarantees no artifact is leaked on
# --- any early-exit path.
WORKDIR="$(mktemp -d "${TMPDIR:-/tmp}/kitty_obs.XXXXXXXX")"
XVFB_PID=""; KITTY_PID=""; GUI_PID=""; WIN=""
DISPNUM=""; DISPLAY_STR=""

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
    if [ -n "${DISPLAY_STR}" ] && [ -n "${XVFB_PID}" ] && kill -0 "${XVFB_PID}" 2>/dev/null; then
        DISPLAY="${DISPLAY_STR}" xdotool keyup j 2>/dev/null   # release any held key
    fi
    kill_wait "${GUI_PID}"
    kill_wait "${KITTY_PID}"
    kill_wait "${XVFB_PID}"
    if [ -n "${DISPNUM}" ]; then
        rm -f "/tmp/.X${DISPNUM}-lock" "/tmp/.X11-unix/X${DISPNUM}"
    fi
    rm -rf "${WORKDIR}"
    echo "=== CLEANUP absence checks ==="
    { [ -n "${KITTY_PID}" ] && kill -0 "${KITTY_PID}" 2>/dev/null; } && echo "kitty launcher STILL ALIVE" || echo "kitty launcher absent: OK"
    { [ -n "${GUI_PID}" ]   && kill -0 "${GUI_PID}"   2>/dev/null; } && echo "kitty GUI STILL ALIVE"      || echo "kitty GUI absent: OK"
    { [ -n "${XVFB_PID}" ]  && kill -0 "${XVFB_PID}"  2>/dev/null; } && echo "Xvfb STILL ALIVE"           || echo "Xvfb absent: OK"
    [ -n "${DISPNUM}" ] && [ -e "/tmp/.X${DISPNUM}-lock" ]     && echo "X lock STILL present"   || echo "X lock absent: OK"
    [ -n "${DISPNUM}" ] && [ -e "/tmp/.X11-unix/X${DISPNUM}" ] && echo "X socket STILL present" || echo "X socket absent: OK"
    [ -e "${WORKDIR}" ] && echo "WORKDIR STILL present" || echo "WORKDIR removed: OK"
    echo "=== final git status of repo (only the answer doc should ever change) ==="
    ( cd "$REPO" && git status --porcelain ) || true
}
trap cleanup EXIT INT TERM

# --- Only now do the failure-capable display selection. ---
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

Xvfb "${DISPLAY_STR}" -screen 0 1280x800x24 +extension GLX +render -noreset \
    >"${WORKDIR}/xvfb.log" 2>&1 &
XVFB_PID=$!
export DISPLAY="${DISPLAY_STR}"
for _ in $(seq 1 50); do xdpyinfo >/dev/null 2>&1 && break; sleep 0.1; done
xdpyinfo >/dev/null 2>&1 || { echo "FATAL: Xvfb ${DISPLAY_STR} not reachable" >&2; exit 5; }
echo "Xvfb up on ${DISPLAY_STR} (pid ${XVFB_PID})"
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

stop_kitty() {   # stop this run's kitty and WAIT until its window is gone (flushes stdout)
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
strip_ts() { sed -E 's/^\[[0-9]+\.[0-9]+\] //'; }

# ===========================================================================
echo "############################## RUN 1 — legacy /bin/bash (main run) ##############################"
DUMP1="${WORKDIR}/kdump1.bin"; OUT1="${WORKDIR}/out1.log"; ERR1="${WORKDIR}/err1.log"
echo "\$ dump size BEFORE launch: $( [ -e "$DUMP1" ] && dsize "$DUMP1" || echo absent )"
launch_kitty "$DUMP1" "$OUT1" "$ERR1"
sleep 0.5
grep -qa "focused: 1" "$ERR1" && echo "focus confirmed: on_focus_change focused: 1" || echo "WARN: focus not confirmed"
echo "\$ dump size AFTER launch, BEFORE typing: $(dsize "$DUMP1") bytes"

echo "----- framebuffer BEFORE typing (clean prompt) -----"
FB_BEFORE="${WORKDIR}/frame_before.png"
echo "\$ import -window root frame_before.png"
import -window root "$FB_BEFORE" 2>/dev/null || true
echo "distinct colors BEFORE: $(convert "$FB_BEFORE" -format '%k' info: 2>/dev/null)"
echo "sha256 BEFORE:          $(sha256sum "$FB_BEFORE" 2>/dev/null | cut -d' ' -f1)"

echo "----- Condition 1: unmodified letters l, s, Return -----"
C1_PRE="$(dsize "$DUMP1")"
R1_PRE="$(wc -l < "$ERR1")"
echo "\$ xdotool key --clearmodifiers l ; xdotool key --clearmodifiers s ; xdotool key --clearmodifiers Return"
xdotool key --clearmodifiers l
xdotool key --clearmodifiers s
xdotool key --clearmodifiers Return
sleep 0.7
C1_POST="$(dsize "$DUMP1")"
echo "dump grew ${C1_PRE} -> ${C1_POST} bytes after 'ls'<Enter> (delta $((C1_POST-C1_PRE)) bytes)"
echo "--- hexdump -C of the first 48 bytes of the child->terminal echo delta ---"
tail -c +"$((C1_PRE + 1))" "$DUMP1" | head -c 48 | hexdump -C
echo "--- reception adjacency (UNFILTERED, cat -v): platform XKB feeder (glfw/xkb_glfw.c:875) then on_key_input (kitty/keys.c:176) ---"
{ tail -n +"$((R1_PRE + 1))" "$ERR1" | grep -aE 'xkb_keycode|on_key_input' || true; } | head -12 | cat -v

echo "----- framebuffer AFTER typing 'ls'<Enter> + input-caused transition metric (F2) -----"
FB_AFTER="${WORKDIR}/frame_after.png"
echo "\$ import -window root frame_after.png"
import -window root "$FB_AFTER" 2>/dev/null || true
echo "distinct colors AFTER:  $(convert "$FB_AFTER" -format '%k' info: 2>/dev/null) (a blank screen would be 1)"
echo "sha256 AFTER:           $(sha256sum "$FB_AFTER" 2>/dev/null | cut -d' ' -f1)"
echo "non-background bounding box (glyph region) AFTER: $(convert "$FB_AFTER" -fuzz 5% -trim info: 2>/dev/null | grep -oE '[0-9]+x[0-9]+\+[0-9]+\+[0-9]+' | head -1)"
echo -n "\$ compare -metric AE frame_before.png frame_after.png : changed pixels = "
compare -metric AE "$FB_BEFORE" "$FB_AFTER" null: 2>&1 || true
echo ""

echo "----- Condition 2: modifier combinations ctrl+c, shift+a, alt+a (legacy) -----"
M_PRE="$(wc -l < "$ERR1")"
echo "\$ xdotool key --clearmodifiers ctrl+c ; xdotool key --clearmodifiers shift+a ; xdotool key --clearmodifiers alt+a"
xdotool key --clearmodifiers ctrl+c; sleep 0.3
xdotool key --clearmodifiers shift+a; sleep 0.3
xdotool key --clearmodifiers alt+a; sleep 0.5
echo "--- on_key_input traces for the modifier conditions (verbatim, cat -v) ---"
{ tail -n +"$((M_PRE + 1))" "$ERR1" | grep -a on_key_input || true; } | cat -v

echo "----- Condition 3: press/repeat/release lifecycle (hold j ~0.8s) -----"
echo "\$ xdotool keydown j ; sleep 0.8 ; xdotool keyup j"
xdotool keydown j; sleep 0.8; xdotool keyup j; sleep 0.5
echo "--- held-key 'j' (0x6a) per-action tally ---"
{ grep -a on_key_input "$ERR1" | grep -a 'key: 0x6a ' || true; } | { grep -oa 'action: [A-Z]*' || true; } | sort | uniq -c
echo "--- first 4 and last 2 held-'j' events (verbatim, cat -v) ---"
{ grep -a on_key_input "$ERR1" | grep -a 'key: 0x6a ' || true; } | head -4 | cat -v
echo "   ... (identical REPEAT lines elided; count shown in tally) ..."
{ grep -a on_key_input "$ERR1" | grep -a 'key: 0x6a ' || true; } | tail -2 | cat -v

echo "----- Condition 5: dump size after all legacy typing -----"
echo "dump AFTER all legacy typing: $(dsize "$DUMP1") bytes"

echo "----- Condition 6a: focus-loss distribution (unfocus, inject 3 keys) -----"
before="$( { grep -ac on_key_input "$ERR1" || true; } )"
echo "\$ xdotool windowfocus --sync 0 ; xdotool key x ; xdotool key y ; xdotool key z"
xdotool windowfocus --sync 0 2>/dev/null || true
sleep 0.3
xdotool key --clearmodifiers x 2>/dev/null || true
xdotool key --clearmodifiers y 2>/dev/null || true
xdotool key --clearmodifiers z 2>/dev/null || true
sleep 0.4
after="$( { grep -ac on_key_input "$ERR1" || true; } )"
echo "on_key_input count before=${before}; after 3 keys while UNFOCUSED=${after}; delivered=$((after-before)) of 3"
echo "--- focus-change events observed (cat -v) ---"
{ grep -a on_focus_change "$ERR1" || true; } | cat -v

echo "----- Condition 6b: no-active-window C branch (keys.c:182) probe -----"
echo "'no active window, ignoring' occurrences this run: $( { grep -ac 'no active window, ignoring' "$ERR1" || true; } )"

echo "===== RUN 1 whole-run action-field tally (PRESS/REPEAT/RELEASE) ====="
tally "$ERR1"
echo "===== RUN 1 startup/platform stderr (first 6 lines, cat -v) ====="
head -6 "$ERR1" | cat -v

stop_kitty   # flushes the child's block-buffered stdout

echo "===== RUN 1 STDOUT AFTER EXIT — stream attribution for --dump-bytes (F1) ====="
echo "\$ wc -l out1.log ; then classify each line"
TOTAL_OUT="$(wc -l < "$OUT1")"
GL_OUT="$( { grep -ac 'GL version string' "$OUT1" || true; } )"
CMD_OUT="$( { grep -acE '^(draw|set_title|screen_|process_|shell_|report_|desktop_|active_hyperlink|first_key|clipboard_|cmd_|dcs_|osc_)' "$OUT1" || true; } )"
echo "out1.log total lines=${TOTAL_OUT}; GL-version lines=${GL_OUT}; parsed-command lines=${CMD_OUT}"
echo "--- the single GL-version line on STDOUT (cat -v) ---"
{ grep -a 'GL version string' "$OUT1" || true; } | cat -v
echo "--- first 12 NON-GL lines on STDOUT = parsed commands emitted by DumpCommands (boss.py:248-252), verbatim cat -v ---"
{ grep -av 'GL version string' "$OUT1" || true; } | head -12 | cat -v

echo "===== RUN 1 legacy reception/encoding traces (verbatim, cat -v; protocol lines excluded) ====="
{ grep -a on_key_input "$ERR1" | grep -av '; 9 7 \|5 7 4 4 2 ' || true; } | cat -v

# ===========================================================================
echo "############################## RUN 2 — legacy stability (fresh process) ##############################"
DUMP2="${WORKDIR}/kdump2.bin"; OUT2="${WORKDIR}/out2.log"; ERR2="${WORKDIR}/err2.log"
launch_kitty "$DUMP2" "$OUT2" "$ERR2"
sleep 0.5
C2_PRE="$(dsize "$DUMP2")"
echo "\$ xdotool key --clearmodifiers l ; s ; Return"
xdotool key --clearmodifiers l
xdotool key --clearmodifiers s
xdotool key --clearmodifiers Return
sleep 0.7
C2_POST="$(dsize "$DUMP2")"
echo "RUN 2 dump grew ${C2_PRE} -> ${C2_POST} bytes after 'ls'<Enter>"
echo "--- hexdump -C of first 16 bytes of RUN 2 echo delta ---"
tail -c +"$((C2_PRE + 1))" "$DUMP2" | head -c 16 | hexdump -C
echo "===== RUN 2 PRESS reception/encoding traces (verbatim, cat -v) ====="
{ grep -a on_key_input "$ERR2" | grep -a 'action: PRESS' || true; } | cat -v
echo "===== RUN 2 GL diagnostic on STDOUT (verbatim, cat -v) ====="
{ grep -a 'GL version string' "$OUT2" || true; } | cat -v
stop_kitty

echo "############################## TWO-RUN COMPARISON — legacy (timestamps stripped) ##############################"
first3() {   # the first l, first s, first Enter PRESS line, timestamp stripped
    {   { grep -a on_key_input "$1" | grep -a 'action: PRESS' | grep -am1 "text: 'l'"       || true; }
        { grep -a on_key_input "$1" | grep -a 'action: PRESS' | grep -am1 "text: 's'"       || true; }
        { grep -a on_key_input "$1" | grep -a 'action: PRESS' | grep -am1 "native_code: 0xff0d" || true; }
    } | strip_ts
}
first3 "$ERR1" > "${WORKDIR}/r1.txt"
first3 "$ERR2" > "${WORKDIR}/r2.txt"
echo "\$ diff RUN1 vs RUN2 PRESS lines for l / s / Enter (empty == identical modulo timestamp)"
if diff -u "${WORKDIR}/r1.txt" "${WORKDIR}/r2.txt"; then
    echo "RESULT: legacy reception traces IDENTICAL across the two fresh runs (timestamps aside)"
else
    echo "RESULT: differences shown above"
fi

# ===========================================================================
echo "############################## RUN 3 — protocol helper: kitten show-key -m normal (fresh) ##############################"
DUMP3="${WORKDIR}/kdump3.bin"; OUT3="${WORKDIR}/out3.log"; ERR3="${WORKDIR}/err3.log"
launch_kitty "$DUMP3" "$OUT3" "$ERR3"
sleep 0.5
focus_window
xdotool key --clearmodifiers ctrl+c; sleep 0.4    # clear any pending command line

echo "----- Condition 4a: kitten show-key -m normal (legacy/normal mode helper) -----"
En_PRE="$(wc -l < "$ERR3")"; Pn_PRE="$(dsize "$DUMP3")"
echo "\$ (typed into the shell) kitten show-key -m normal <Enter>, then press a, ctrl+a, ctrl+c"
xdotool type --clearmodifiers "kitten show-key -m normal"
xdotool key --clearmodifiers Return; sleep 1.2
xdotool key --clearmodifiers a; sleep 0.4
xdotool key --clearmodifiers ctrl+a; sleep 0.4
xdotool key --clearmodifiers ctrl+c; sleep 0.3    # exit the kitten
xdotool key --clearmodifiers Return; sleep 0.5
echo "--- Kitty send-side on_key_input traces under -m normal (verbatim, cat -v; a and ctrl+a) ---"
{ tail -n +"$((En_PRE + 1))" "$ERR3" | grep -a on_key_input | grep -a 'native_code: 0x61\|native_code: 0xffe3' || true; } | cat -v
echo "--- child->terminal dump delta under -m normal: printable rendering of received bytes (cat -v) ---"
tail -c +"$((Pn_PRE + 1))" "$DUMP3" | cat -v | grep -aoE 'a|\^A|0x[0-9a-f]+' | head -20 | tr '\n' ' '; echo ""
stop_kitty

# ---- reusable clean protocol run: fresh instance, opt-in via show-key -m kitty, press a/ctrl+a/ctrl+c ----
run_protocol() {   # run_protocol <label> <dump> <out> <err>
    local label="$1" dump="$2" out="$3" err="$4" ep pp _
    launch_kitty "$dump" "$out" "$err"
    sleep 0.5
    focus_window
    xdotool key --clearmodifiers ctrl+c; sleep 0.4
    ep="$(wc -l < "$err")"; pp="$(dsize "$dump")"
    echo "\$ (typed into the shell) kitten show-key -m kitty <Enter>, wait for ESC[>31u opt-in, then press a, ctrl+a, ctrl+c"
    xdotool type --clearmodifiers "kitten show-key -m kitty"
    xdotool key --clearmodifiers Return
    PROTO_OPTIN="no"
    for _ in $(seq 1 80); do
        if tail -c +"$((pp + 1))" "$dump" | grep -qa $'\x1b\[>31u'; then PROTO_OPTIN="yes"; break; fi
        sleep 0.1
    done
    echo "protocol opt-in ESC[>31u seen in child->terminal dump: ${PROTO_OPTIN}"
    xdotool key --clearmodifiers a; sleep 0.4          # press+release under protocol
    xdotool key --clearmodifiers ctrl+a; sleep 0.4     # Ctrl-a under protocol
    xdotool key --clearmodifiers ctrl+c; sleep 0.3     # terminate the kitten
    xdotool key --clearmodifiers Return; sleep 0.6
    echo "--- ${label} protocol-mode reception/encoding traces (verbatim on_key_input, cat -v) ---"
    { tail -n +"$((ep + 1))" "$err" | grep -a on_key_input || true; } | cat -v
    echo "--- ${label} opt-in / restore markers in the child->terminal dump (cat -v, counted) ---"
    { tail -c +"$((pp + 1))" "$dump" | cat -v | grep -ao '\[>31u\|\[<u' || true; } | sort | uniq -c
    stop_kitty
}

# ===========================================================================
echo "############################## RUN 4 — Kitty protocol run #1 (fresh) ##############################"
DUMP4="${WORKDIR}/kdump4.bin"; OUT4="${WORKDIR}/out4.log"; ERR4="${WORKDIR}/err4.log"
run_protocol "RUN 4" "$DUMP4" "$OUT4" "$ERR4"

# ===========================================================================
echo "############################## RUN 5 — Kitty protocol run #2 (fresh; F5 stability) ##############################"
DUMP5="${WORKDIR}/kdump5.bin"; OUT5="${WORKDIR}/out5.log"; ERR5="${WORKDIR}/err5.log"
run_protocol "RUN 5" "$DUMP5" "$OUT5" "$ERR5"

# ===========================================================================
echo "############################## TWO-RUN COMPARISON — Kitty protocol (timestamps stripped) ##############################"
protolines() {   # protocol CSI-u 'a' encodings, timestamp-stripped; crash-safe (every stage guarded)
    { grep -a on_key_input "$1" 2>/dev/null || true; } \
      | { grep -a 'native_code: 0x61' || true; } \
      | { grep -a 'sent encoded key to child: \^\[ \[' || true; } \
      | strip_ts
}
protolines "$ERR4" > "${WORKDIR}/p4.txt" || true
protolines "$ERR5" > "${WORKDIR}/p5.txt" || true
echo "\$ RUN4 protocol CSI-u 'a' lines (timestamp-stripped):"
cat "${WORKDIR}/p4.txt"
echo "\$ RUN5 protocol CSI-u 'a' lines (timestamp-stripped):"
cat "${WORKDIR}/p5.txt"
echo "\$ diff RUN4 vs RUN5 protocol CSI-u 'a' lines (empty == identical modulo timestamp)"
if diff -u "${WORKDIR}/p4.txt" "${WORKDIR}/p5.txt"; then
    echo "RESULT: Kitty-protocol traces IDENTICAL across the two fresh runs (timestamps aside)"
else
    echo "RESULT: differences shown above"
fi

# ===========================================================================
echo "############################## RUN 6 — Ctrl-D / EOT on empty command line (fresh; exits) (F3) ##############################"
DUMP6="${WORKDIR}/kdump6.bin"; OUT6="${WORKDIR}/out6.log"; ERR6="${WORKDIR}/err6.log"
launch_kitty "$DUMP6" "$OUT6" "$ERR6"
sleep 0.5
D_PRE="$(dsize "$DUMP6")"
Ed_PRE="$(wc -l < "$ERR6")"
echo "dump size BEFORE Ctrl-D: ${D_PRE} bytes"
echo "\$ xdotool key --clearmodifiers ctrl+d   (on an empty command line -> bash EOF)"
xdotool key --clearmodifiers ctrl+d
# wait for the launcher to exit on its own (shell EOF -> window closes); set -e safe
DRC="timeout"
for _ in $(seq 1 100); do
    if ! kill -0 "$KITTY_PID" 2>/dev/null; then
        wait "$KITTY_PID" 2>/dev/null && DRC=0 || DRC=$?
        break
    fi
    sleep 0.1
done
D_POST="$(dsize "$DUMP6")"
echo "dump size AFTER Ctrl-D: ${D_POST} bytes (delta $((D_POST-D_PRE)) bytes)"
echo "--- on_key_input traces for Ctrl-D (verbatim, cat -v; ctrl modifier then 'd') ---"
{ tail -n +"$((Ed_PRE + 1))" "$ERR6" | grep -a on_key_input || true; } | cat -v
echo "--- child->terminal dump tail after Ctrl-D (cat -v; shell logout bytes) ---"
tail -c +"$((D_PRE + 1))" "$DUMP6" | tail -c 64 | cat -v; echo ""
echo "launcher exit status after Ctrl-D EOF: ${DRC}"
leftover=""
for c in $(xdotool search --class kitty 2>/dev/null); do [ "$c" = "$WIN" ] && leftover="yes"; done
echo "kitty window still present after Ctrl-D: $([ -n "$leftover" ] && echo yes || echo no)"
KITTY_PID=""; GUI_PID=""; WIN=""   # already exited

echo "############################## DONE (cleanup runs on EXIT) ##############################"
```

*End of document. The harness above was executed once (producing RUN 1–RUN 6 in a single run); all temporary artifacts it created were removed on exit, and the only file added to the repository is this document at `blitzy/documentation/kitty_815df1e210e0.md`.*
