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

This `git rev-parse HEAD` is run inside the container's `/app` — the Kitty **source under study**, checked out at the baseline commit. This documentation file itself is the *only* addition, committed on top of that baseline on the deliverable branch, so a `git diff` from the baseline contains exactly one new file (`blitzy/documentation/kitty_815df1e210e0.md`) and no source changes (re‑verified in §7).

### 1.2 Build (run‑first, step 1)

Kitty is not pure Python — it has a C core plus a Go tools layer — so a binary must be compiled before anything can be observed. The `Makefile` target `all` runs the build (`Makefile:L12-L13` → `python3 setup.py`). The full `make all` output is 36 compile steps plus 2 link steps; the tail is shown verbatim (the command pipes through `tail -6` so the block is exactly what was printed):

```console
$ make all 2>&1 | tail -6
[35/36] Compiling [wayland] glfw/wl_cursors.c ...
[36/36] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
 done
[1/2] Linking [wayland] kitty/glfw-wayland ...
[2/2] Linking launcher ...
 done
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root 1001 36224 Jul  1 05:05 kitty/launcher/kitty
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

> The build **adds** the compiled launcher `kitty/launcher/kitty`; the C sources `kitty/launcher/{launcher.h,main.c,single-instance.c}` are pre‑existing tracked files and are left untouched. The build outputs (`build/`, `kitty/launcher/kitty`, `*.so`) are all git‑ignored, so building never dirties the working tree — verified in §7.

### 1.3 Headless display + software GL (run‑first, step 2)

Kitty renders through OpenGL and requires a GL context. In source, the required version is assembled from `kitty/data-types.h:L20` `#define OPENGL_REQUIRED_VERSION_MAJOR 3` and a platform‑dependent minor: `kitty/data-types.h:L22` `#define OPENGL_REQUIRED_VERSION_MINOR 3` inside `#ifdef __APPLE__` (`:L21`) versus `kitty/data-types.h:L24` `#define OPENGL_REQUIRED_VERSION_MINOR 1` in the `#else` (`:L23`) branch — so on this **Linux** host the minimum is **OpenGL 3.1** (with `#define GLSL_VERSION 140` at `:L26`). Kitty requests that context via `kitty/glfw.c:L1127-L1128` `glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR/MINOR, OPENGL_REQUIRED_VERSION_MAJOR/MINOR)`.

Because there is no GPU, a virtual X display plus Mesa's software rasterizer (`llvmpipe`) were used:

```console
$ Xvfb :99 -screen 0 1024x768x24 &
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
$ glxinfo -B | grep -E 'OpenGL renderer string|OpenGL core profile version string'
OpenGL renderer string: llvmpipe (LLVM 19.1.1, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1
```

> **Software‑GL caveat.** All rendering observed here proceeds through **Mesa `llvmpipe` software OpenGL**, not a hardware GPU. This is important when interpreting the `--debug-rendering` output in Q3.

### 1.4 Run with tracing + the exact keys pressed (run‑first, step 3)

Two documented debug flags were passed together so all three stages appear in one session — `--debug-input` (a.k.a. `--debug-keyboard`) and `--debug-rendering` (a.k.a. `--debug-gl`), defined in `kitty/cli.py:L996` and `kitty/cli.py:L989`. The default shell (`/bin/bash`) was launched by giving kitty no program argument. Both output streams were captured to a scratch directory **outside** the repository (removed afterward — see §7); the reason for splitting stdout/stderr is explained in §2:

```console
# stdout -> cap.out (GL banner + parsed dump), stderr -> cap.err (key traces)
$ kitty/launcher/kitty --debug-input --debug-rendering \
      --dump-bytes=bytes.log -o repaint_delay=2 \
      > cap.out 2> cap.err &

# focus the window under Xvfb, then press a few simple keys in the shell:
$ WID=$(xdotool search --class kitty | head -1)
$ xdotool windowactivate --sync "$WID"
$ xdotool type --delay 180 'ls'   # printable keys: l, s
$ xdotool key Return              # the Enter key
```

The **exact keys pressed** were: `l`, `s`, then **Enter** (typing `ls` and running it), later followed by `e`,`x`,`i`,`t`, Enter to close the shell. This stimulus is deliberately chosen so that printable keys exercise the *text* path while Enter exercises the *encoded* path (see Q2). A **second** session pressed three default kitty shortcuts (`kitty_mod` = `ctrl+shift`) to exercise the *shortcut‑dispatch* path:

```console
$ kitty/launcher/kitty --debug-input --debug-rendering -o repaint_delay=2 \
      > sc.out 2> sc.err &
$ xdotool key --clearmodifiers ctrl+shift+l       # next_layout
$ xdotool key --clearmodifiers ctrl+shift+equal   # change_font_size
$ xdotool key --clearmodifiers ctrl+shift+c       # copy_to_clipboard
```

Running the `ls`+Enter stimulus a **second** time (`cap2.err`) produced **byte‑for‑byte identical** key‑trace lines after stripping the timestamp/color prefixes, and the raw child round‑trip (`bytes.log`) was identical too — confirming the behavior described here is *consistently* observed at runtime rather than incidental:

```console
# normalize = strip the leading "[secs] " timestamp and the ANSI SGR color codes
$ norm(){ grep -aE 'on_key_input|sent key|ignoring as keyboard' "$1" | sed -E 's/^\[[0-9]+\.[0-9]+\] //; s/\x1b\[[0-9;]*m//g'; }
$ norm cap.err > n1.txt ; norm cap2.err > n2.txt
$ diff n1.txt n2.txt && echo "IDENTICAL: 0 differences"
IDENTICAL: 0 differences
$ md5sum n1.txt n2.txt
5e2d582954ac5d30484266f3ea8a71ee  n1.txt
5e2d582954ac5d30484266f3ea8a71ee  n2.txt
$ md5sum bytes.log bytes2.log   # raw child round-trip, both runs
bef71b189721e5747facd3938e314253  bytes.log
bef71b189721e5747facd3938e314253  bytes2.log
```

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

**Answer.** The operating system / display server (here X11 via Xvfb) hands the key event to Kitty's **GLFW backend callback `key_callback`** (`kitty/glfw.c:L430`). In this repository, `key_callback` is the **first backend callback** in the path to receive the event; for a real, non‑focus‑synthetic event on a ready window it forwards to Kitty's **keyboard‑processing entry point `on_key_input`** (`kitty/keys.c:L166`). The explicit order is therefore: **OS/display server → `key_callback` (`kitty/glfw.c:L430`) → `on_key_input` (`kitty/keys.c:L166`)** — `on_key_input` is the first Kitty code that *processes the keystroke*, immediately downstream of the backend callback.

**Code path.**
- The callback is registered once, at window creation: `kitty/glfw.c:L1292` `glfwSetKeyboardCallback(glfw_window, key_callback);`.
- The hand‑off happens at `kitty/glfw.c:L439`: `if (is_window_ready_for_callbacks() && !ev->fake_event_on_focus_change) on_key_input(ev);`.
- `on_key_input(GLFWkeyevent *ev)` begins at `kitty/keys.c:L166`. When keyboard debugging is on (`kitty/keys.c:L172` `if (OPT(debug_keyboard))`), it prints the event it received at `kitty/keys.c:L176` (the IME‑only variant `on_IME_input` is at `:L174`).

**Observed proof.** Every key produced an `on_key_input:` line on **stderr**. Command that produced it (limited to the first three PRESS events — `l`, `s`, Enter):

```console
$ xdotool type --delay 180 'ls'   # then: xdotool key Return
$ grep -a on_key_input cap.err | grep 'action: PRESS' | head -3 | cat -v   # cat -v renders ESC 0x1B as ^[
```

Verbatim (the `^[` sequences are the literal `\x1b[33m`/`\x1b[m` SGR yellow/reset that `kitty/keys.c:L176` wraps around `on_key_input`):

```text
[2.207] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[2.292] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[2.786] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
```

**Reading the line.** The fields map exactly to the format string at `kitty/keys.c:L176` (`glfw key: 0x%x native_code: 0x%x action: %s %stext: '%s' state: %d`):

- `glfw key: 0x6c` / `native_code: 0x6c` — the `l` key (`0x6c` = ASCII `l`); `0x73` = `s`; the Enter key reports the GLFW functional keycode `glfw key: 0xe001` with X11 `native_code: 0xff0d` (`XK_Return`).
- `action: PRESS` — one of `PRESS`/`RELEASE`/`REPEAT` (`kitty/keys.c:L178`).
- `mods: none` — produced by `format_mods` (`kitty/keys.c:L144`); a modified chord instead reads e.g. `mods: ctrl+shift ` (see Q2).
- `text: 'l'` — the UTF‑8 text GLFW resolved for the key; empty for non‑text keys such as Enter.

The mere presence of these lines is the runtime proof that `on_key_input` (`kitty/keys.c:L166`, trace at `:L176`) is Kitty's keyboard‑processing entry point, reached **immediately after** the backend callback `key_callback` (`kitty/glfw.c:L430`) — the first repository code in the path — hands off the event at `kitty/glfw.c:L439`. The trailing `sent key as text…` / `sent encoded key…` text on the *same* line is the next stage (Q2), sharing the one timestamp because of the `starting_print` behavior described in §2.


---

## 4. Q2 — Intermediate processing: parsing, encoding, and dispatch

**Answer.** Between ingress and the child, `on_key_input` (`kitty/keys.c:L166`) drives three sub‑stages: (1) it first asks the Python `Boss` whether the chord is a mapped **shortcut** and, if so, dispatches it and sends nothing to the child; (2) otherwise it **encodes** the event into bytes with `encode_glfw_key_event` (`kitty/key_encoding.c:L414`); and (3) it **writes** those bytes to the child PTY via `schedule_write_to_child` (`kitty/child-monitor.c:L372`). The child's reply is then read back and **parsed** by the VT state machine (`kitty/vt-parser.c`) into updates on the screen‑grid model (`kitty/screen.c`). Each of these sub‑stages emits its own trace line, shown below.

After `on_key_input` receives the event, three things can happen to it. The traces let us watch each branch.

### 4.1 Shortcut dispatch (into the Python `Boss`)

For a `PRESS`/`REPEAT` (`kitty/keys.c:L226`), `on_key_input` first asks whether the chord is a mapped shortcut by dispatching into Python: `kitty/keys.c:L228` `dispatch_key_event(dispatch_possible_special_key);`. That reaches `Boss.dispatch_action` (`kitty/boss.py:L1572`), whose inner `report_match` (`kitty/boss.py:L1579`) — gated by `kitty/boss.py:L1580` `if self.args.debug_keyboard:` — prints the matched action at `kitty/boss.py:L1583` (`timed_debug_print(f'{prefix}\x1b[35m{dispatch_type}\x1b[m matched action:', func_name(f), …)`). If the shortcut is consumed, `on_key_input` logs `handled as shortcut` at `kitty/keys.c:L231` **and returns without sending any bytes to the child.**

**Observed proof** (second session). First, each chord arrives at `on_key_input` with `mods: ctrl+shift`:

```console
$ xdotool key --clearmodifiers ctrl+shift+l      # and ctrl+shift+equal, ctrl+shift+c
$ grep -aE "native_code: 0x(6c|3d|63) action: PRESS mods: ctrl\+shift" sc.err | cat -v
```

```text
[2.201] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: ctrl+shift text: '' state: 0 
[2.642] ^[[33mon_key_input^[[m: glfw key: 0x3d native_code: 0x3d action: PRESS mods: ctrl+shift text: '' state: 0 
[3.084] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl+shift text: '' state: 0 
```

Each dispatches into the `Boss`, which prints the matched action (`^[[35m…^[[m` is the literal magenta `\x1b[35m`/`\x1b[m` from `kitty/boss.py:L1583`), followed by `handled as shortcut` (`kitty/keys.c:L231`); the subsequent **release** takes the distinct branch at `kitty/keys.c:L239`:

```console
$ grep -aE "matched action|handled as shortcut|ignoring release event for previous" sc.err | cat -v
```

```text
^[[35mKeyPress^[[m matched action: next_layout, handled as shortcut
[2.220] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring release event for previous press that was handled as shortcut
^[[35mKeyPress^[[m matched action: change_font_size, [2.646] SIGWINCH sent to child in window: 1 with size: (19, 64, 640, 399)
handled as shortcut
[2.661] ^[[33mon_key_input^[[m: glfw key: 0x3d native_code: 0x3d action: RELEASE mods: none text: '' state: 0 ignoring release event for previous press that was handled as shortcut
^[[35mKeyPress^[[m matched action: copy_to_clipboard, handled as shortcut
[3.102] ^[[33mon_key_input^[[m: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: none text: '' state: 0 ignoring release event for previous press that was handled as shortcut
```

- `ctrl+shift+l` → `matched action: next_layout`; `ctrl+shift+equal` (`0x3d` = `=`) → `matched action: change_font_size` (which additionally resized the child, hence the interleaved `SIGWINCH sent to child …` line — note it prints its own `[2.646]` timestamp because it starts a fresh line); `ctrl+shift+c` (`0x63` = `c`) → `matched action: copy_to_clipboard`. Each is followed by `handled as shortcut` (`kitty/keys.c:L231`).
- The `func_name` printed after `matched action:` is exactly the dispatched action (`next_layout`, `change_font_size`, `copy_to_clipboard`).
- The subsequent key **release** of a consumed shortcut logs `ignoring release event for previous press that was handled as shortcut` (`kitty/keys.c:L239`).

Because these chords were consumed as shortcuts, **no bytes were sent to the child** for them — consistent with the early `return` at `kitty/keys.c:L231`.

### 4.2 Encoding (deciding the bytes)

If the chord is *not* a shortcut, `on_key_input` encodes it: `kitty/keys.c:L251` `int size = encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key);`. The encoder (`kitty/key_encoding.c:L414`) honors DECCKM cursor‑key mode and the active keyboard‑protocol flags (legacy escape sequences vs. the Kitty Keyboard Protocol). For a plain printable key with text, it returns the sentinel `SEND_TEXT_TO_CHILD` (`kitty/key_encoding.c:L437`); for keys like Enter it returns an encoded byte length.

### 4.3 Writing to the child PTY (the two output branches)

The return value selects the branch, and each branch is traced:

- **Text path** — `size == SEND_TEXT_TO_CHILD`: `kitty/keys.c:L253` `schedule_write_to_child(w->id, 1, text, strlen(text));` then `kitty/keys.c:L254` `debug("sent key as text to child: %s\n", text)`.
- **Encoded path** — `size > 0`: `kitty/keys.c:L259` `schedule_write_to_child(w->id, 1, encoded_key, size);` then `kitty/keys.c:L261` `debug("sent encoded key to child: ")` followed by a per‑byte loop (`kitty/keys.c:L263-L266`) that prints `^[ ` for ESC (`27`), `SPC ` for space, a raw `%c ` for printable bytes, or `0x%x ` otherwise.
- **Unsupported** — otherwise: `kitty/keys.c:L271` `debug("ignoring as keyboard mode does not support encoding this event\n")`.

`schedule_write_to_child` itself lives in `kitty/child-monitor.c:L372` and queues the bytes onto the child PTY.

**Observed proof.** From the `ls` stimulus (`cap.err`), the first five key events show both output branches plus the unsupported branch:

```console
$ grep -aE 'sent key as text|sent encoded key|ignoring as keyboard' cap.err | head -5 | cat -v
```

```text
[2.207] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[2.246] ^[[33mon_key_input^[[m: glfw key: 0x6c native_code: 0x6c action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.292] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[2.337] ^[[33mon_key_input^[[m: glfw key: 0x73 native_code: 0x73 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.786] ^[[33mon_key_input^[[m: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
```

- The printable keys `l` and `s` took the **text** path → `sent key as text to child: l` / `… s` (`kitty/keys.c:L254`).
- **Enter** took the **encoded** path → `sent encoded key to child: 0xd ` — i.e. a single byte `0x0d` (ASCII **CR**), printed through the `else { debug("0x%x ", …); }` branch at `kitty/keys.c:L266` because CR is non‑printable. `0xd` is the exact byte handed to the PTY for Enter under the default (legacy) keyboard mode.
- Every **RELEASE** (and the bare modifier keys `ctrl`/`shift`, which report `glfw key: 0xe062`/`0xe061`) fell through to `ignoring as keyboard mode does not support encoding this event` (`kitty/keys.c:L271`) — nothing is sent for them.

A Python‑side counterpart, `KeyboardHandler.debug_print` (`kitty/keys.py:L242-L245`, gated by `b.args.debug_keyboard`), prints additional keyboard diagnostics when the terminal is in a keyboard‑reporting mode; the simple keys here stayed on the C legacy path above.

### 4.4 The child round‑trip → VT parser → screen model ("before the screen updates")

The bytes written to the PTY are consumed by the child (the default `bash`). Its response is read back by Kitty's I/O loop, whose `parse_func` is `parse_worker` (`kitty/child-monitor.c:L181`) — or `parse_worker_dump` (`kitty/child-monitor.c:L180`) when a dump callback is installed. Adding `--dump-bytes=…` installs exactly such a callback: `kitty/boss.py:L372` passes `DumpCommands(args)` to the `ChildMonitor` when `args.dump_commands or args.dump_bytes`. `DumpCommands` writes the **raw** inbound bytes to the file (`kitty/boss.py:L242-L244`) and prints the **parsed command names** to stdout (`kitty/boss.py:L249,L252`). The parser that turns bytes into those commands is the VT state machine in `kitty/vt-parser.c` (e.g. `dispatch_single_byte_control` `:L224` → `screen_draw_text` `:L226`; control bytes map at `:L96-L102`, where CR → `screen_carriage_return` `:L102` and LF → `screen_linefeed` `:L101`), which mutates the grid model in `kitty/screen.c`.

**Observed proof of the round‑trip.** The raw inbound bytes (`bytes.log`, rendered with `cat -v`, so CR shows as `^M` and ESC as `^[`) show the shell setting up its prompt, echoing the typed command, and then emitting the directory listing:

```console
$ cat -v bytes.log
```

```text
^[P@kitty-print|aWdub3JlYm90aCBvciBpZ25vcmVzcGFjZSBwcmVzZW50IGluIGJhc2ggSElTVENPTlRST0wgc2V0dGluZywgc2hvd2luZyBydW5uaW5nIGNvbW1hbmQgd2lsbCBub3QgYmUgcm9idXN0Cg==}^[\^[]7;kitty-shell-cwd://44dd5f519fa4/app^G^[[?2004h^[]133;k;start_kitty^G^[]133;D;0^G^[]133;A^G^[]133;k;end_kitty^G^[]0;root@44dd5f519fa4: /app^Groot@44dd5f519fa4:/app# ^[]133;k;start_suffix_kitty^G^[[5 q^[]2;/app^G^[]133;k;end_suffix_kitty^Gls^M
^[[?2004l^M^[]2;ls^G^[]133;C;cmdline=ls^G^A^[]133;k;start_kitty^G^B^A^[]133;k;end_kitty^G^B^A^[]133;k;start_suffix_kitty^G^B^A^[[0 q^B^A^[]133;k;end_suffix_kitty^G^B^[[0m^[[01;34m3rdparty^[[0m                ^[[01;34mglfw^[[0m               ^[[01;34m__pycache__^[[0m^M
Brewfile                go.mod             pyproject.toml^M
^[[01;34mbuild^[[0m                   go.sum             README.asciidoc^M
^[[01;32mbuild-terminfo^[[0m          INSTALL.md         SECURITY.md^M
^[[01;34mbypy^[[0m                    key_encoding.json  session.vim^M
CHANGELOG.rst           ^[[01;34mkittens^[[0m            ^[[01;32msetup.py^[[0m^M
constants_generated.go  ^[[01;34mkitty^[[0m              ^[[01;34mshell-integration^[[0m^M
CONTRIBUTING.md         ^[[01;34mkitty_tests^[[0m        shell.nix^M
^[[01;32mcount-lines-of-code^[[0m     LICENSE            staticcheck.conf^M
^[[01;32mdev.sh^[[0m                  ^[[01;34mlogo^[[0m               ^[[01;34mterminfo^[[0m^M
^[[01;34mdocs^[[0m                    __main__.py        ^[[01;32mtest.py^[[0m^M
^[[01;34mgen^[[0m                     Makefile           ^[[01;34mtools^[[0m^M
^[[01;34mglad^[[0m                    ^[[01;32mpublish.py^[[0m         ^[[01;32mupdate-on-ox^[[0m^M
^[[?2004h^[]133;k;start_kitty^G^[]133;D;0^G^[]133;A^G^[]133;k;end_kitty^G^[]0;root@44dd5f519fa4: /app^Groot@44dd5f519fa4:/app# ^[]133;k;start_suffix_kitty^G^[[5 q^[]2;/app^G^[]133;k;end_suffix_kitty^Gexit^M
^[[?2004l^M^[]2;exit^G^[]133;C;cmdline=exit^G^A^[]133;k;start_kitty^G^B^A^[]133;k;end_kitty^G^B^A^[]133;k;start_suffix_kitty^G^B^A^[[0 q^B^A^[]133;k;end_suffix_kitty^G^Bexit^M
```

> The leading `^[P@kitty-print|…}^[\` (a DCS) and the `^[]133;…^G` sequences are bash **shell‑integration** markers; the round‑trip of interest is the echoed `…ls^M` near the end of the first line, followed by the directory‑listing rows (each ending `^M`), and finally the echoed `…exit^M`.

And the **parsed** command stream on stdout (`cap.out`) shows those bytes becoming screen operations — first the echo of `ls`, then the directory listing being **drawn** (note these dump lines carry no `[%.3f]` timestamp; only the GL banner does — see §6/§7):

```console
$ grep -aE '^draw |^screen_carriage_return|^screen_linefeed|^screen_set_mode' cap.out | head -12
```

```text
screen_set_mode 2004 1
draw root@44dd5f519fa4:/app# 
draw ls
screen_carriage_return
screen_linefeed
screen_carriage_return
draw 3rdparty
draw                 
draw glfw
draw                
draw __pycache__
screen_carriage_return
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
- The finished frame is presented by swapping buffers **from the render loop**: `render_prepared_os_window` calls `swap_window_buffers(os_window)` at `kitty/child-monitor.c:L810`, and that function performs the actual swap at `kitty/glfw.c:L1802-L1803` (`swap_window_buffers(OSWindow *os_window)` → `if (glfwAreSwapsAllowed(os_window->handle)) glfwSwapBuffers(os_window->handle);`). (The separate `glfwSwapBuffers` at `kitty/glfw.c:L1221` is *only* the one‑time initial blank‑canvas swap during window creation — not the per‑frame presentation path.)

**Observed proof.** The single most direct, consistently‑appearing rendering signal is the GL version banner, emitted on **stdout** by `kitty/gl.c:L72` (`printf("[%.3f] GL version string: %s\n", …)`, where the string is built by `snprintf(..., "'%s' Detected version: %d.%d", …)` at `kitty/gl.c:L47`). Command and output across all three runs:

```console
$ grep -a 'GL version string' cap.out cap2.out sc.out
cap.out:[0.119] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
cap2.out:[0.118] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
sc.out:[0.123] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```

- The reported renderer/version is `'4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1'` with `Detected version: 4.5` — i.e., the context created under `LIBGL_ALWAYS_SOFTWARE=1` is Mesa `llvmpipe`'s OpenGL 4.5 core profile, comfortably above the Linux minimum of 3.1 (`kitty/data-types.h:L20,L24`). The same line appeared in every run (`[0.119]`, `[0.118]`, `[0.123]`), varying only in its timestamp — evidence that context creation and the GL path are exercised consistently.
- Under `--debug-rendering`'s per‑call error checking (`kitty/gl.c:L62`), the **absence** of any error/fatal line is itself observable evidence that the render path ran to completion. No `OpenGL error …` (`kitty/gl.c:L17`), no `Loading the OpenGL library failed` (`kitty/gl.c:L57`), no version‑too‑low `fatal` (`kitty/gl.c:L74`), and no `Failed to create GLFW temp window!` (`kitty/glfw.c:L1199`) appeared on any stream:

```console
$ grep -acE 'OpenGL error|fatal|Failed to create GLFW|Loading the OpenGL library failed' cap.err cap.out sc.err sc.out
cap.err:0
cap.out:0
sc.err:0
sc.out:0
```

- The screen‑content evidence from Q2 (`draw ls`, then `draw 3rdparty` … the drawn directory listing) is the model that this loop uploads via `send_cell_data_to_gpu` (`kitty/child-monitor.c:L714,L766`) and composites through the shaders before the render‑loop buffer swap (`kitty/child-monitor.c:L810` → `kitty/glfw.c:L1802-L1803`).

> Note (see §7): `--debug-rendering` does not print a line *per frame*; its observable footprint is the one‑time GL banner plus the installed error‑checking callback. Under `llvmpipe`, the compositing is done in software rather than on a hardware GPU.

---

## 6. End‑to‑end summary

Putting the observed signals in order, a single keystroke travels like this (timestamps are the real `[%.3f]` values from the `l` press and the GL banner):

| # | Stage | Component (`file:line`) | Observable signal |
|---|-------|-------------------------|-------------------|
| 0 | GL context ready (startup) | `kitty/gl.c:L72` | `[0.119] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5` (stdout) |
| 1 | OS/display server → backend callback | `kitty/glfw.c:L430`, registered `:L1292` | (delivers the X11 event) |
| 2 | Backend → keyboard entry point | `kitty/glfw.c:L439` → `kitty/keys.c:L166` | `[2.207] on_key_input: … text: 'l' …` (stderr) |
| 3a | Shortcut? dispatch to Boss | `kitty/keys.c:L228` → `kitty/boss.py:L1583` | `KeyPress matched action: next_layout` + `handled as shortcut` (`kitty/keys.c:L231`) |
| 3b | Else encode | `kitty/keys.c:L251` → `kitty/key_encoding.c:L414` | (returns `SEND_TEXT_TO_CHILD` or a byte length) |
| 4 | Write to child PTY | `kitty/keys.c:L253/L259` → `kitty/child-monitor.c:L372` | `sent key as text to child: l` / `sent encoded key to child: 0xd ` |
| 5 | Child answers; read + parse | `kitty/child-monitor.c:L181` → `kitty/vt-parser.c` | raw `ls^M` (bytes.log); `draw ls`, `screen_carriage_return`, `screen_linefeed` (cap.out) |
| 6 | Update grid model | `kitty/screen.c` | `draw 3rdparty` … (the drawn `ls` output) |
| 7 | Render loop | `kitty/main.py:L234` → `kitty/child-monitor.c:L1259,L1262` | (drives frames) |
| 8 | Upload cells to GPU | `kitty/child-monitor.c:L714,L766` | sets `needs_render = true` |
| 9 | Composite via shaders | `kitty/shaders.c:L394…L1009` (+ `*.glsl`) | (no GL error under `--debug-rendering`) |
| 10 | Present frame (render‑loop swap) | `kitty/child-monitor.c:L810` → `kitty/glfw.c:L1802-L1803` | buffers swapped → display |

On ordering and timing: the outbound write shares its keypress's timestamp on a single line (e.g. the `l` press **and** its `sent key as text to child: l` are both stamped `[2.207]`; Enter and its `sent encoded key to child: 0xd ` are both `[2.786]`), which is the "before the screen updates" moment captured directly. An exact input‑to‑screen *latency* is **not** computed here because the parsed/drawn dump lines (`cap.out`) carry no `[%.3f]` timestamp (only the key traces and the GL banner are timestamped), so there is no captured completion timestamp to subtract from — this is noted honestly in §7 rather than estimated.


---

## 7. What could not be verified (and honest caveats)

- **Software GL, not a hardware GPU.** All rendering used Mesa `llvmpipe` (`OpenGL renderer string: llvmpipe (LLVM 19.1.1, 256 bits)`) under `Xvfb` with `LIBGL_ALWAYS_SOFTWARE=1`. The GPU‑specific timing and any driver‑specific behavior of `send_cell_data_to_gpu`/the shader stages could not be observed; the observed `Detected version: 4.5` and the *absence* of GL errors are what confirm the render path ran. Real‑GPU frame timing is therefore **not** verified here.
- **No per‑frame render log exists to quote.** `--debug-rendering` installs GL error checking (`kitty/gl.c:L62`) and prints the one‑time GL banner (`kitty/gl.c:L72`); it does not emit a line for each composited frame. So Q3's frame production is evidenced *indirectly* (the GL banner + the drawn `ls` output + the verified absence of any `fatal(...)`), not by a literal "frame N rendered" trace.
- **Input‑to‑screen latency is not measurable from these captures.** The parsed/drawn dump lines (`cap.out`) and the raw bytes (`bytes.log`) carry no `[%.3f]` timestamp — only the key traces (`cap.err`) and the GL banner (stdout) are timestamped. There is therefore no captured "screen updated" timestamp to subtract from the keypress timestamp, so no numeric latency is asserted. (The `[2.207]`→`[2.786]` gap between the `l` and Enter presses is merely the `xdotool --delay 180` typing cadence, not a pipeline latency.)
- **No pixel screenshot.** The image tools `import`, `xwd`, `convert`, `scrot`, and `ffmpeg` are absent from the container, so a framebuffer screenshot of the rendered window was not captured. The "screen updated" claim rests on the parsed `draw …` command stream (`cap.out`) and the raw child bytes (`bytes.log`), not on a captured image.
- **The buffer swap is not individually traced.** The render‑loop swap at `kitty/child-monitor.c:L810` → `kitty/glfw.c:L1802-L1803` has no debug print; it is cited from source, and its precondition (a valid GL context) is what the GL banner demonstrates at runtime.
- **The AAP's "OpenGL 3.3+" is Apple‑only.** On this Linux host the compiled minimum is **3.1** (`kitty/data-types.h:L24`, `GLSL_VERSION 140` at `:L26`); `3.3` (`:L22`) applies under `#ifdef __APPLE__`. This document quotes the runtime‑detected `4.5` rather than asserting a fixed minimum.
- **Encoding internals inferred from the outcome.** `encode_glfw_key_event` (`kitty/key_encoding.c:L414`) is not itself traced; its effect is inferred from the branch taken (`sent key as text …` vs `sent encoded key … 0xd`). The default legacy mode was in force for the simple keys pressed; the Kitty Keyboard Protocol path was not exercised by this stimulus.
- **Read‑only observation (verified).** The build and run never modified the tracked source tree. After `make all` and the traced sessions, the source repository was clean; all capture files lived in a scratch directory outside the repository:

```console
$ git -C /app status --porcelain | wc -l    # 0 = source tree untouched by observation
0
```

## 8. Coverage checklist (closing pass)

- [x] **Verbatim question reproduced** — at the top of the document as a blockquote.
- [x] **Q1 — Input ingress — answered explicitly (§3):** OS/display server → `key_callback` (`kitty/glfw.c:L430`, registered `:L1292`, first backend callback) → `on_key_input` (`kitty/keys.c:L166`, keyboard‑processing entry point), proven by the verbatim `on_key_input:` PRESS traces (`kitty/keys.c:L176`) each paired with the command that produced them.
- [x] **Q2 — Intermediate processing — answered explicitly (§4):** shortcut dispatch (`kitty/boss.py:L1583` `matched action:` + `kitty/keys.c:L231` `handled as shortcut`), encoding (`kitty/keys.c:L251` → `kitty/key_encoding.c:L414`), PTY write with both branches (`sent key as text to child: l/s` `kitty/keys.c:L254`; `sent encoded key to child: 0xd ` `kitty/keys.c:L261,L266`), the ignore branches (`:L271`, `:L239`), and the child round‑trip through `parse_worker`/`vt-parser.c` into `screen.c` (raw `ls^M` + `draw …` commands) — all quoted verbatim with their producing commands.
- [x] **Q3 — Display production — answered explicitly (§5):** render loop (`kitty/main.py:L234` → `kitty/child-monitor.c:L1259,L1262`), GPU upload (`kitty/child-monitor.c:L714,L766`), shader compositing (`kitty/shaders.c`), render‑loop buffer swap (`kitty/child-monitor.c:L810` → `kitty/glfw.c:L1802-L1803`), proven by the verbatim `GL version string:` banner (`kitty/gl.c:L72`) and the verified absence of GL `fatal(...)`.
- [x] **End‑to‑end ordering (§6)** — a single ordered table from OS event to the render‑loop buffer swap, keyed to the observed timestamps.
- [x] **Consistency (§1.4)** — the `ls`+Enter key traces were byte‑for‑byte identical across two runs (`md5sum` match), the raw child round‑trip was identical (`md5sum` match), and the GL banner appeared in all runs; each shown with its exact command and output.
- [x] **What could not be verified (§7)** — software‑GL caveat, no per‑frame log, no measurable input‑to‑screen latency, no pixel screenshot, the untraced buffer swap, and the 3.1‑vs‑3.3 correction, all stated honestly with the read‑only proof.

### Reproduction note

The build/run/capture was performed inside the provided Docker image (`kitty-build-env:ready`, from `ghcr.io/scaleapi/swe-atlas`) because the analysis host lacks the C/Go toolchain and native GL/font libraries; a bare `python3 setup.py` there fails with `FileNotFoundError: [Errno 2] No such file or directory: 'pkg-config'`. All scratch files used to gather this evidence lived outside the repository and were removed afterward, leaving this document as the only change (the read‑only proof `git -C /app status --porcelain | wc -l` → `0` is shown in §7).
