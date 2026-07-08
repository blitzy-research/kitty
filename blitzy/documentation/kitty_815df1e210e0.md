# Kitty Input-Routing & Focus Management — A Runtime Investigation

**Repository:** kitty terminal emulator · **Branch:** `kitty_815df1e210e0` · **HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Question answered:** How does kitty *actually* route keyboard input and manage focus across OS windows, tabs, and child processes at runtime — established by **building and running** this repository, driving real input, and capturing live artifacts, **not** by reading source alone.

---

## How to read this document

- Every factual claim carries a `file:line` citation into this repository and is tagged **[observed]** (proved by a runtime artifact reproduced verbatim below) or **[inferred]** (derived from reading; runtime confirmation was not possible for the stated reason).
- All command output is reproduced **verbatim and unedited** in fenced blocks, with the exact command shown immediately above it. Nothing logic-bearing is paraphrased or elided.
- **Canonical vs non-canonical.** The *only* input route treated as the answer is the real platform path: synthetic X11 key events (via `xdotool`, which uses the X11 `XTEST` extension) that enter the vendored GLFW library at `_glfwInputKeyboard()` `glfw/input.c:306`. Kitty's remote control (`kitten @ send-text`) and the GLFW `Null`/OSMesa backend (`glfw/null_init.c`, `glfw/null_window.c`) **bypass** this path and are **non-canonical**; they appear below only as explicitly labeled contrasts, never as the observed route.
- The observation lens is kitty's own built-in flag `--debug-input` / `--debug-keyboard` `kitty/cli.py:996`, whose C printers are gated by the master switch `#define debug_input(...)` in `kitty/state.h:15`. It required **no** source modification.

---

## (a) Build & launch — exact commands, tool versions, backend

### Tool versions [observed]

Recorded from the running container. `cat /tmp/kitty_probe/versions.txt`:

```
$ python3 --version
Python 3.13.7
$ go version
go version go1.24.4 linux/amd64
$ cc --version | head -1
cc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ py-spy --version
py-spy 0.4.2
$ gdb --version | head -1
GNU gdb (Ubuntu 16.3-1ubuntu2) 16.3
$ xdotool --version
xdotool version 3.20160805.1
$ Xvfb -help 2>&1 | head -1
use: X [:<display>] [option]
```

Go 1.24.4 satisfies the project floor `go 1.22` at `go.mod:3` **[observed]**. The Python that *runs* kitty is not this system Python 3.13.7 but a bundled CPython 3.14.6 fetched by the canonical build (confirmed by the interpreter banner in the py-spy dump in section (d): `Python v3.14.6`) **[observed]**.

### Build command (canonical, default configuration) [observed]

Built from the repository root exactly as a developer would, per `docs/build.rst:19` (`./dev.sh build`); `dev.sh` dispatches to `go run bypy/devenv.go` which fetches major dependencies as prebuilt binaries and compiles the C extension + vendored GLFW + the Go `kitten`:

```
$ ./dev.sh build --ignore-compiler-warnings
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 27776 kitty/launcher/kitty
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The `--ignore-compiler-warnings` flag is runtime-neutral: it only sidesteps a `-Werror` stop in the **Wayland** backend source (`glfw/wl_window.c`) caused by the freshly-fetched dependency bundle; the **X11** backend used at runtime is unaffected **[inferred]** (build-flag semantics, not a runtime observation). A debug variant `./dev.sh build --debug --ignore-compiler-warnings` was also produced for richer symbols per `docs/build.rst:54`; the stack snapshots in section (d) were taken against the default build, whose `.so` files retain a symbol table (function names resolve) though without DWARF line info **[observed]**.

### Display & backend (real X11, NOT Null/OSMesa) [observed]

No physical display exists, so a real X11 server was provided by Xvfb and software GL forced via Mesa `llvmpipe`. This is the **real X11 GLFW backend** (`kitty/glfw-x11.so`), *not* the headless GLFW `Null`/OSMesa backend:

```
$ Xvfb :99 -screen 0 1920x1080x24 +extension GLX +extension RANDR +render -noreset &
$ export DISPLAY=:99
$ export LIBGL_ALWAYS_SOFTWARE=1
```

Proof the X11 backend (not Null) was loaded: every stack in section (d) shows frames from `kitty/glfw-x11.so` (e.g. `glfwRunMainLoop`, `_glfwDispatchX11Events`, `processEvent`), and `/proc/<pid>/maps` (section (f)) maps `libX11`, `libxcb`, and `libxkbcommon-x11` — none of which the Null backend would load **[observed]**.

### Launch & confirm the debug lens is live [observed]

```
$ nohup ./kitty/launcher/kitty --debug-input --config NONE > /tmp/kitty_probe/kitty.log 2>&1 &
$ xdotool search --sync --class kitty
2097164
```

`--config NONE` selects kitty's built-in defaults (no user `kitty.conf`), i.e. the canonical default configuration — so `input_delay=3` and `repaint_delay=10` (`kitty/options/types.py:536,567`) are in force **[observed]**. Critically, `--debug-input` prints to the **launching process's** stdout/stderr (redirected here to `kitty.log`), **not** into the kitty terminal window — verified because `on_focus_change`/`on_key_input` lines appear in `kitty.log` only after a key is injected, and never inside the on-screen shell **[observed]**. This flag is wired at `kitty/cli.py:996` (`--debug-input --debug-keyboard`, `dest=debug_keyboard` at `:997`), threaded through `_main()` `kitty/main.py:441` → `init_glfw(opts, cli_opts.debug_keyboard, ...)` `kitty/main.py:514` → `init_glfw_module(...)` `kitty/main.py:90`, and every C printer below is gated on it via `#define debug_input(...) if (OPT(debug_keyboard)) {...}` `kitty/state.h:15` **[observed]**.

**Input injection method (canonical).** Real key events were synthesized with `xdotool key --clearmodifiers <KEY>` / `xdotool type`, which drive the X server's `XTEST` extension so the events flow through the ordinary X11 event queue into `_glfwInputKeyboard()` `glfw/input.c:306` **[observed]**. `xdotool`'s `--window` (XSendEvent) form was deliberately **not** used, because GLFW ignores synthetic `send_event` events — so `--window` would be a non-canonical stand-in **[inferred]** (GLFW event-filtering behavior).

---

## (b) R1 — Generating overlapping input activity

All scenarios below were driven into the running kitty via `xdotool` (canonical `_glfwInputKeyboard` path) and captured from the `--debug-input` log. Colour SGR escapes that kitty emits into its own debug lines (`\x1b[33m`, `\x1b[35m`, …) have been stripped for legibility with `sed 's/\x1b\[[0-9;]*m//g'`; no content words were altered **[observed]**.

### R1.1 Tabs, OS windows, rapid focus switching (shortcut keys) [observed]

Creating tabs (`ctrl+shift+t`) and switching windows (`ctrl+shift+]` / `ctrl+shift+[`). `grep -aE "matched action|ignoring release event" /tmp/kitty_probe/r1_s1_tabs.log | sed 's/\x1b\[[0-9;]*m//g'`:

```
KeyPress matched action: new_tab, handled as shortcut
KeyPress matched action: new_tab, handled as shortcut
KeyPress matched action: next_window, handled as shortcut
[229.935] on_key_input: glfw key: 0x5d native_code: 0x5d action: RELEASE mods: none text: '' state: 0 ignoring release event for previous press that was handled as shortcut
KeyPress matched action: previous_window, handled as shortcut
[230.382] on_key_input: glfw key: 0x5b native_code: 0x5b action: RELEASE mods: none text: '' state: 0 ignoring release event for previous press that was handled as shortcut
```

The `matched action: … , handled as shortcut` line is printed by kitty's mapping dispatcher `kitty/boss.py:1583`; the paired `ignoring release event for previous press that was handled as shortcut` is `kitty/keys.c:239` **[observed]**. A consumed shortcut writes **nothing** to any child (there is no `sent … to child` line for these presses) — the recipient-selection and shortcut layer sit *above* the PTY write **[observed]**.

### R1.2 Plain + modifier/alternate keys → legacy encodings [observed]

Letters, arrows, `ctrl+c`, `alt+b`, and `F1` typed into a shell. `grep` of `/tmp/kitty_probe/r1_s2_modifiers.log` (stripped):

```
[268.870] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[269.448] on_key_input: glfw key: 0xe006 native_code: 0xff51 (Left) action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ D
[269.448] on_key_input: glfw key: 0xe007 native_code: 0xff53 (Right) action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ C
[269.448] on_key_input: glfw key: 0xe008 native_code: 0xff52 (Up) action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A
[269.448] on_key_input: glfw key: 0xe009 native_code: 0xff54 (Down) action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ B
[269.998] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3
[270.396] on_key_input: glfw key: 0x62 native_code: 0x62 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ b
[270.829] on_key_input: glfw key: 0xe014 native_code: 0xffbe (F1) action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ O P
```

A plain letter is delivered as UTF-8 text (`sent key as text to child: c`, `kitty/keys.c:254`); arrows/`ctrl+c`/`alt+b`/`F1` are turned into escape sequences (`sent encoded key to child:`, `kitty/keys.c:261`) by `encode_glfw_key_event()` `kitty/key_encoding.c:414` — cursor keys → `CSI D/C/A/B`, `ctrl+c` → `0x3` (ETX), `alt+b` → `ESC b`, `F1` → SS3 `ESC O P` **[observed]**.

### R1.3 Kitty Keyboard Protocol (CSI-u) — same key, different encoding [observed]

When the child program enables the Kitty Keyboard Protocol, the per-screen encoding flags change and the *same* physical `a` is encoded differently. Flag transitions are logged by `kitty/screen.c:1244` (Pushed) / `:1252` (Popped). `grep` across `/tmp/kitty_probe/r1_s2b_csiu.log` and `/tmp/kitty_probe/r1_s2c_clean.log` (stripped):

```
[297.465] Pushed key encoding flags to: 1
[298.087] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[330.932] Popped key encoding flags to: 0
[397.151] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent encoded key to child: ^[ [ 9 7 u
```

Under flags `1` (disambiguate only) the letter still goes as text `a`; under the fuller flag set the same key becomes CSI-u `ESC [ 9 7 u` (97 = ASCII `a`) — proving the encoder consults the live per-screen flags from `screen_current_key_encoding_flags()` `kitty/screen.c:1204` at the moment of the keypress, not a fixed table **[observed]**.

### R1.4 Typing during resize & scroll (transitional states) [observed]

Font-resize maps and scrollback keys exercised while typing. `grep -aE "change_font_size|scroll_line_up|scroll|matched action" /tmp/kitty_probe/r1_s3_resize_scroll.log | sed 's/\x1b\[[0-9;]*m//g'` (representative):

```
KeyPress matched action: change_font_size, handled as shortcut
KeyPress matched action: change_font_size, handled as shortcut
KeyPress matched action: scroll_line_up, handled as shortcut
[482.113] on_key_input: glfw key: 0xe004 native_code: 0xff56 (Page Down) action: PRESS mods: shift text: '' state: 0 sent encoded key to child: ^[ [ 5 ; 2 ~
```

Resizing and scrolling do not perturb routing: resize/scroll shortcuts are consumed as actions (`handled as shortcut`, `kitty/keys.c:231`) while an ordinary key pressed in the same period is still encoded and sent to the focused child (`kitty/keys.c:259`) — the recipient decision is independent of transient window geometry/scroll state **[observed]**.

### R1.5 Background output while a *different* window is focused (decisive) [observed]

A continuous emitter (`yes`) was started in window A; window B was then focused and typed into. Full `/tmp/kitty_probe/r1_s4_bgoutput.log` (stripped):

```
[494.334] on_focus_change: window id: 0x1 focused: 0
[494.341] on_focus_change: window id: 0x2 focused: 1
[494.949] Press xkb_keycode: 0x1a clean_sym: e composed_sym: e text: e mods: none glfw_key: 101 (e) xkb_key: 101 (e)
[494.949] on_key_input: glfw key: 0x65 native_code: 0x65 action: PRESS mods: none text: 'e' state: 0 sent key as text to child: e
[494.955] Release xkb_keycode: 0x1a clean_sym: e composed_sym: e text: e mods: none glfw_key: 101 (e) xkb_key: 101 (e)
[494.955] on_key_input: glfw key: 0x65 native_code: 0x65 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[494.955] Press xkb_keycode: 0x36 clean_sym: c composed_sym: c text: c mods: none glfw_key: 99 (c) xkb_key: 99 (c)
[494.955] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[494.958] Release xkb_keycode: 0x36 clean_sym: c composed_sym: c text: c mods: none glfw_key: 99 (c) xkb_key: 99 (c)
```

While window A's `yes` kept producing output (its capture file grew to hundreds of MB in the background), keystrokes typed after the `focused: 1` transition landed in window B's child (`sent key as text to child: e`, `… c`) **[observed]**. Output production and input routing are independent concerns: focus (`on_focus_change`, `kitty/glfw.c:517`) selects the *input* recipient, while the busy background child keeps writing on its own PTY handled by the I/O thread (section (d)/(g)) **[observed]**.

### R1.6 Key auto-repeat (held key) [observed]

Holding `j` for ~0.9 s. Counts and head of `/tmp/kitty_probe/r1_s5_repeat.log` (stripped):

```
PRESS  count: 1
REPEAT count: 22
RELEASE count: 1
[546.866] on_key_input: glfw key: 0x6a native_code: 0x6a action: PRESS  mods: none text: 'j' state: 0 sent key as text to child: j
[547.066] on_key_input: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[547.099] on_key_input: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
[547.132] on_key_input: glfw key: 0x6a native_code: 0x6a action: REPEAT mods: none text: 'j' state: 0 sent key as text to child: j
```

A single hold produces one `PRESS`, a stream of `REPEAT` events (~33 ms apart), and one `RELEASE`; each `PRESS`/`REPEAT` is independently routed and written to the child, driven by `on_key_input()` `kitty/keys.c:166` **[observed]**.

---


## (c) R2 — How a keystroke is routed: first-seen → intermediate → final destination

The pipeline, as reconstructed from the logs above and the live stacks in section (d):

```
OS/X11 key event
  → [external lib, C]  _glfwInputKeyboard()            glfw/input.c:306      (normalizes; delivers to callback)
  → [external lib, C]  glfw_xkb_handle_key_event()      glfw/xkb_glfw.c       (XKB decode; "Press/Release xkb_keycode …")
  → [kitty C ext]      key_callback()                   kitty/glfw.c:430      (GLFW callback registered by kitty)
  → [kitty C ext]      on_key_input()                   kitty/keys.c:166      (RECIPIENT DECISION + dispatch + encode)
       ├─ active_window()                               kitty/keys.c:106      (picks the single recipient window)
       ├─ boss.dispatch_possible_special_key()          kitty/boss.py:1408    (shortcut? if so, consume; write nothing)
       ├─ encode_glfw_key_event()                       kitty/key_encoding.c:414 (legacy vs CSI-u)
       └─ schedule_write_to_child(w->id, …)             kitty/keys.c:259
  → [kitty C ext]      schedule_write_to_child_generic  kitty/child-monitor.c:323 (match child by window id; append to write_buf)
  → PTY write on the I/O thread (KittyChildMon)         kitty/child-monitor.c:io_loop → child process
```

### R2a — Which component sees the input *first* [observed]

The first code to hold a normalized key event is the vendored GLFW library. In the single-keystroke trace (`/tmp/kitty_probe/r1_s4_bgoutput.log`), the XKB-layer line prints **before** kitty's `on_key_input`, at the **same timestamp**:

```
[494.949] Press xkb_keycode: 0x1a clean_sym: e composed_sym: e text: e mods: none glfw_key: 101 (e) xkb_key: 101 (e)
[494.949] on_key_input: glfw key: 0x65 native_code: 0x65 action: PRESS mods: none text: 'e' state: 0 sent key as text to child: e
```

The `Press xkb_keycode …` line is emitted inside GLFW's XKB handler `glfw/xkb_glfw.c` (external library); `_glfwInputKeyboard()` `glfw/input.c:306` is the normalized entry that hands the event to the registered callback `key_callback()` `kitty/glfw.c:430` **[observed]**. The identical timestamp shows the two happen in one synchronous call chain — corroborated at the symbol level in section (d), where `key_callback` sits directly above `glfw_xkb_handle_key_event` in the same stack **[observed]**.

### R2b — How the recipient is decided, and the intermediate processing [observed]

`on_key_input()` calls `active_window()` **first** `kitty/keys.c:167`. `active_window()` returns `global_state.callback_os_window->tabs[…].windows[…]`, or `NULL` if that window has no screen `kitty/keys.c:106`. So the recipient is deterministically *the current active window, of the active tab, of the OS window that received the event* — chosen in C, never by the child **[observed]**. If there is no such window the code logs `no active window, ignoring` and returns `kitty/keys.c:182` **[inferred that this exact branch fires — I could not force a no-active-window instant externally; the string and guard are at `kitty/keys.c:182`]**.

Intermediate processing is shortcut resolution. For press/repeat, `on_key_input` calls Python `boss.dispatch_possible_special_key(ev)` `kitty/boss.py:1408` (which delegates to the mappings dispatcher). If the key matches a mapping it is consumed — kitty prints `matched action: …` `kitty/boss.py:1583` then `handled as shortcut` `kitty/keys.c:231` and returns **without** writing to a child (proved in R1.1: `ctrl+shift+t` yielded `new_tab, handled as shortcut` and no `sent … to child` line) **[observed]**. After the Python call the window is re-fetched by id and guarded by `if (!w) return;` `kitty/keys.c:236`, so a handler that closed the window cannot cause a stale write **[inferred — the guard at `kitty/keys.c:236`; not separately forced]**.

### R2c — How the final destination is chosen [observed]

A non-shortcut key is encoded (`encode_glfw_key_event()` `kitty/key_encoding.c:414`) and written by `schedule_write_to_child(w->id, 1, …)` `kitty/keys.c:259`. Delivery is by **window id**: `schedule_write_to_child_generic` loops kitty's child table and matches `children[i].id == id` `kitty/child-monitor.c:336`, appends the bytes to that child's screen `write_buf`, and wakes the I/O loop `kitty/child-monitor.c:323`–`369` **[observed]**.

That the bytes truly reach the intended child's PTY was confirmed with a round-trip: a `printf '\033[c'` (Primary Device Attributes query) typed into the focused window produced kitty's DA reply, captured raw from that child. `od -c /tmp/kitty_probe/da.txt`:

```
0000000 033   [   ?   6   2   ;   c
0000007
```

The child received `ESC [ ? 62 ; c` — kitty's DA response — proving the input path terminates at the correct child PTY and that kitty (main loop), not the child, produced the reply **[observed]**.

### R2d — Focus propagation (C → Python fan-out) [observed]

A platform focus change enters GLFW at `_glfwInputWindowFocus()` `glfw/window.c:45`, surfaces in kitty at `window_focus_callback()` `kitty/glfw.c:515` which logs `on_focus_change` `kitty/glfw.c:517`, sets `global_state.callback_os_window->is_focused` `kitty/glfw.c:527`, and fans out to Python via `WINDOW_CALLBACK(on_focus, "O", …)` `kitty/glfw.c:538`. Python `boss.on_focus()` `kitty/boss.py:1651` forwards to the window's `focus_changed()` `kitty/window.py:1123`, which sets `self.is_focused` `kitty/window.py:1126`, calls watchers with `on_focus_change` `kitty/window.py:1127`, and finally `self.screen.focus_changed(focused)` `kitty/window.py:1134`. At the C screen level `focus_changed()` `kitty/screen.c:4604` writes `ESC[I`/`ESC[O` to the child **iff** focus-tracking mode is on `kitty/screen.c:4611`.

This full chain was confirmed end-to-end. A child in focus-reporting mode captured exactly one `I`/`O` pair per transition across four focus flips. `od -c /tmp/kitty_probe/focus_py.txt`:

```
0000000 033   [   O 033   [   I 033   [   O 033   [   I
0000014
```

`ESC[O` (blur) and `ESC[I` (focus) alternate, matching the four `on_focus_change` transitions in the log — the C-side `is_focused` flip and the Python fan-out both fire, and the child observes the result **[observed]**.

---


## (d) R3 — Stack / symbol snapshots of input handling

Three complementary live snapshots were taken against the running kitty (pid 55798): a merged Python+native dump (`py-spy`), and two `gdb` breakpoints that caught the input path *in the act* at its entry and its delivery point.

### R3.0 The ptrace attach was NOT blocked — and here is the concrete reason [observed]

The prompt anticipates a ptrace block. In this environment the attach **succeeded**, and the reason is concrete and proven, not hand-waved. `cat /tmp/kitty_probe/r3_ptrace_env.txt`:

```
$ id
uid=0(root) gid=0(root) groups=0(root)

$ cat /proc/sys/kernel/yama/ptrace_scope
1

$ capsh --print | grep -iE 'current|bounding'
Current: =ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read,cap_perfmon,cap_bpf,cap_checkpoint_restore
```

`kernel.yama.ptrace_scope` is `1` (restricted), which normally forbids attaching to a non-descendant process. But the caller is **root** (`uid=0`) with **`cap_sys_ptrace`** present and effective (`Current: =ep`, and `cap_sys_ptrace` in the bounding set) — Yama's `ptrace_scope=1` is explicitly bypassed for a tracer holding `CAP_SYS_PTRACE`, so `py-spy --native` and `gdb -p` both attach cleanly **[observed]**. This is the concrete mechanism the attach depends on; had root/`CAP_SYS_PTRACE` been absent, the same `ptrace_scope=1` would have produced an `Operation not permitted` failure, at which point the `gdb`/`pstack` fallbacks documented here would still work under the same privilege.

### R3.1 Merged Python + native dump (`py-spy dump --native`) [observed]

```
$ py-spy dump --native --pid 55798
```
```
Process 55798: ./kitty/launcher/kitty --debug-input --config NONE
Python v3.14.6 (/tmp/blitzy/kitty/blitzy-d0e37078-3a77-49bc-afff-f7686163bf62_725416/kitty/launcher/kitty)

Thread 55798 (idle)
    0x7fb865269772 (libc.so.6)
    0x7fb86525d13c (libc.so.6)
    poll (libc.so.6)
    glfwRunMainLoop (kitty/glfw-x11.so)
    main_loop.lto_priv.0 (kitty/fast_data_types.so)
    _run_app (kitty/main.py:234)
    __call__ (kitty/main.py:252)
    _main (kitty/main.py:518)
    main (kitty/main.py:526)
    main (kitty/entry_points.py:195)
    <module> (__main__.py:7)
    _run_code (<frozen runpy>:88)
    _run_module_as_main (<frozen runpy>:204)
    0x7fb8651e7575 (libc.so.6)
```

At idle, the single Python thread is parked in `poll` inside `glfwRunMainLoop` (`kitty/glfw-x11.so`), reached from the C `main_loop` (`kitty/fast_data_types.so`), reached from Python `_run_app` `kitty/main.py:234` → `_main` `kitty/main.py:518` → `main` `kitty/main.py:526` → `kitty/entry_points.py:195` **[observed]**. This proves kitty is embedded CPython 3.14.6 whose main thread lives in the GLFW/X11 event loop — the input path is driven from *this* thread, not a Python-level select loop **[observed]**.

> Note: `py-spy dump` reports only threads that currently hold Python state (the main thread here). Its `--threads` option is a `record` flag, not a `dump` flag — `py-spy dump --native --threads` fails with `unexpected argument '--threads' found` (exit 2), confirmed against `py-spy dump --help`. Full OS-thread structure is therefore taken from `gdb` below **[observed]**.

### R3.2 Live input-ENTRY stack (`gdb` break on `key_callback`) [observed]

A breakpoint on `key_callback`, then real keys injected; it fired on the third injection. Command and the resulting Thread-1 backtrace, `cat /tmp/kitty_probe/gdb_keycb_bt.txt` (native frames; Python interpreter frames #6–#32 collapse to `main()`):

```
$ gdb -p 55798 -batch -ex "set pagination off" -ex "break key_callback" \
      -ex "continue" -ex "backtrace" -ex "detach" -ex "quit"
```
```
Thread 1 "kitty" hit Breakpoint 1, 0x00007fb86444b440 in key_callback () from …/kitty/fast_data_types.so
#0  0x00007fb86444b440 in key_callback () from …/kitty/fast_data_types.so
#1  0x00007fb8633bc5e6 in glfw_xkb_handle_key_event.constprop () from …/kitty/glfw-x11.so
#2  0x00007fb8633bfcb2 in processEvent () from …/kitty/glfw-x11.so
#3  0x00007fb8633c0cb0 in _glfwDispatchX11Events.lto_priv.0 () from …/kitty/glfw-x11.so
#4  0x00007fb86339fb3a in glfwRunMainLoop () from …/kitty/glfw-x11.so
#5  0x00007fb8644140cc in main_loop.lto_priv () from …/kitty/fast_data_types.so
#6  0x00007fb8655e91a5 in _PyObject_VectorcallTstate (…) at ./Include/internal/pycore_call.h:177
…  [Python interpreter frames #7–#31]
#32 0x000055db0c2ba1e1 in main ()
```

Reading top-down: the X11 dispatcher `_glfwDispatchX11Events` → `processEvent` → `glfw_xkb_handle_key_event` (this is where `_glfwInputKeyboard` `glfw/input.c:306` is **inlined** by LTO) → kitty's `key_callback` `kitty/glfw.c:430`. This is the *input entry*, running on the main thread inside the GLFW/X11 loop `kitty/glfw.c:430` **[observed]**. (`on_key_input` `kitty/keys.c:166` does not appear as its own frame because LTO inlined it into `key_callback`; the symbol table has no DWARF line info for these `.so`s, so frames resolve to function+library only.)

### R3.3 Live DELIVERY stack (`gdb` break on `schedule_write_to_child`) [observed]

A breakpoint on `schedule_write_to_child.constprop.0`, keys injected; hit on the second injection. `cat /tmp/kitty_probe/gdb_swc_bt.txt`:

```
$ gdb -p 55798 -batch -ex "set pagination off" \
      -ex "break schedule_write_to_child" -ex "break schedule_write_to_child.constprop.0" \
      -ex "continue" -ex "backtrace" -ex "detach" -ex "quit"
```
```
Thread 1 "kitty" hit Breakpoint 2, 0x00007fb86441ed50 in schedule_write_to_child.constprop () from …/kitty/fast_data_types.so
#0  0x00007fb86441ed50 in schedule_write_to_child.constprop () from …/kitty/fast_data_types.so
#1  0x00007fb86444c363 in key_callback () from …/kitty/fast_data_types.so
#2  0x00007fb8633bc5e6 in glfw_xkb_handle_key_event.constprop () from …/kitty/glfw-x11.so
#3  0x00007fb8633bfcb2 in processEvent () from …/kitty/glfw-x11.so
#4  0x00007fb8633c0cb0 in _glfwDispatchX11Events.lto_priv.0 () from …/kitty/glfw-x11.so
#5  0x00007fb86339fb3a in glfwRunMainLoop () from …/kitty/glfw-x11.so
#6  0x00007fb8644140cc in main_loop.lto_priv () from …/kitty/fast_data_types.so
#7  0x00007fb8655e91a5 in _PyObject_VectorcallTstate (…) at ./Include/internal/pycore_call.h:177
…  [Python interpreter frames #8–#32]
#33 0x000055db0c2ba1e1 in main ()
```

The delivery call `schedule_write_to_child` `kitty/keys.c:259` → `kitty/child-monitor.c:372` sits **directly above** `key_callback` in the *same* stack as R3.2 **[observed]**. This is the decisive structural fact: recipient selection (`active_window()`), shortcut dispatch, encoding, and the child-write scheduling all execute **synchronously on one call chain on the single main thread** — there is no queue or worker between "key arrives" and "bytes scheduled to the child" **[observed]**.

### R3.4 Full OS-thread structure (`gdb thread apply all bt`) [observed]

`gdb -p 55798 -batch -ex "set pagination off" -ex "thread apply all bt"` captured 68 threads (`/tmp/kitty_probe/gdb_all_bt.txt`). Thread-name histogram:

```
Total threads: 68
     33  "kitty"            (1 main/input thread + Mesa gallium workers)
     32  "llvmpipe-0".."llvmpipe-31"   (Mesa software rasterizer pool)
      1  "KittyChildMon"    (PTY I/O multiplexer)
      1  "kitty:disk$0"     (disk cache)
      1  "LinuxAudioSucks"  (audio)
```

The PTY I/O thread is parked in `poll` inside `io_loop`, never in `key_callback`:

```
Thread 3 (LWP 55867) "KittyChildMon":
#0  __syscall_cancel_arch () at …/syscall_cancel.S:56
#3  0x00007fb8652e4a8e in __GI___poll (…) at …/poll.c:29
#4  0x00007fb8644157e5 in io_loop () from …/kitty/fast_data_types.so
#5  0x00007fb865260d64 in start_thread (…) at ./nptl/pthread_create.c:448
#6  0x00007fb8652f43fc in __GI___clone3 () at …/clone3.S:78
```

So input selection happens only on Thread 1, while `KittyChildMon` (`io_loop`, `kitty/child-monitor.c`) merely multiplexes PTY file descriptors — a clean producer/consumer split: the main thread *schedules* bytes into a child's `write_buf`, and `KittyChildMon` performs the actual PTY `write()` **[observed]**. The many `llvmpipe`/gallium threads are Mesa's software-GL rasterizer, unrelated to input **[observed]**.

> Methodology note (honest hiccup): the *first* gdb breakpoint attempt timed out (exit 124). Attaching to 68 threads is slow, and keys were injected before gdb had issued `continue`, so kitty was still frozen and the breakpoint never fired (`Breakpoint 1 at 0x7fb86441e930` was set but not hit). The fix was to insert a `sleep` after launching gdb so `continue` was in effect before injection — after which both R3.2 and R3.3 fired reliably **[observed]**.

---


## (e) R4 — Input for an unfocused, or just-closed, window

These scenarios used a **fresh** kitty instance (pid 70936; OS windows `0x1` / `0x2`) because a stale earlier instance had a child left in CSI-u mode. A raw-mode stdin witness (`reader.py`, using `tty.setraw` + `select` + `os.read`) was armed inside a child to log every byte it *actually* received; this avoids a cooked-TTY line-buffering artifact that a naïve `cat` reader would show.

### R4a — Input while a *different* window is focused: unfocused window gets nothing [observed]

Window W1 (`0x1`) was left unfocused with its reader armed; W2 (`0x2`) was focused and typed into. `cat /tmp/kitty_probe/r4a_unfocused_evidence.txt`:

```
### R4a UNFOCUSED — raw-stdin negative control on unfocused window W1 (id 0x1)
# childW1_raw.log = every byte delivered to W1's child while a raw reader ran; W1 was UNFOCUSED while typing occurred in W2:
READER_STARTED 1783485954.870
READER_ENDED 1783485979.895

# W2 (focused) marker file created by W2's child:
-rw-r--r-- 1 root root 0 Jul  8 04:45 /tmp/kitty_probe/W2_focused_marker

# Focus boundaries during the experiment (from kitty_r4.log):
[0.190] on_focus_change: window id: 0x1 focused: 1
[14.542] on_focus_change: window id: 0x1 focused: 0
[14.542] on_focus_change: window id: 0x2 focused: 1
[33.199] on_focus_change: window id: 0x2 focused: 0
[33.241] on_focus_change: window id: 0x1 focused: 1
[35.856] on_focus_change: window id: 0x1 focused: 0
```

W1's reader ran for 25 s (`READER_STARTED` → `READER_ENDED`) and logged **zero** `BYTES` lines while unfocused, whereas W2's child executed the typed command (the `W2_focused_marker` file was created) **[observed]**. Input is delivered only to the current `active_window()` `kitty/keys.c:106`; an unfocused window receives nothing because it is simply never selected as the recipient **[observed]**.

### R4a (cont.) — The synthetic focus-loss RELEASE is deliberately skipped [observed]

GLFW, on focus loss, sets `_glfw.focusedWindowId = 0` `glfw/window.c:52` and **synthesizes fake key-RELEASE events** with `.fake_event_on_focus_change = true` for any keys still held `glfw/window.c:59`. Kitty ignores these: `key_callback` only calls `on_key_input` when `is_window_ready_for_callbacks() && !ev->fake_event_on_focus_change` `kitty/glfw.c:439`. Tested by holding `j` across a focus change (autorepeat off). `cat /tmp/kitty_probe/r4a_fakerelease_evidence.txt`:

```
### R4a FAKE-RELEASE SKIP — held 'j' (0x6a) across a focus change (autorepeat off)
[96.805] on_key_input: glfw key: 0x6a native_code: 0x6a action: PRESS mods: none text: 'j' state: 0 sent key as text to child: j
[97.425] on_focus_change: window id: 0x2 focused: 0
[97.425] on_focus_change: window id: 0x1 focused: 1
[98.031] Release xkb_keycode: 0x2c clean_sym: j mods: none glfw_key: 106 (j) xkb_key: 106 (j)

PRESS   j (0x6a) count: 1
RELEASE j (0x6a) count: 0
```

There is exactly one `on_key_input … PRESS … 'j'` and **zero** `on_key_input … RELEASE … 'j'` across the focus flip **[observed]**. The XKB layer still logs a low-level `Release xkb_keycode … j` line (from `glfw/xkb_glfw.c`), but no `on_key_input` was generated for it — the synthetic focus-loss release was filtered at `kitty/glfw.c:439`, exactly as the `fake_event_on_focus_change` guard specifies **[observed]**. This prevents a spurious key-up from being delivered to a child at the moment focus leaves a window **[observed]**.

### R4b — Input after a window has just been closed [observed]+[inferred]

A third window W3 (child pid 72545, a live `/bin/bash --posix`) was closed with `ctrl+shift+w`, then a key was injected. `cat /tmp/kitty_probe/r4b_justclosed_evidence.txt`:

```
### R4b JUST-CLOSED — window W3 (WID 4194354) closed via ctrl+shift+w
W3 child pid before close: 72545 (was ALIVE: /bin/bash --posix)
W3 child pid after close : 72545 -> TERMINATED (ps finds nothing)
Windows after close: 4194316 4194332   (W3 4194354 GONE)
Post-close keystrokes (after explicitly focusing survivor W1, since no WM auto-focuses):
  survivor.txt => SURVIVOR=71004  (a LIVE kitty child, ppid 70936)
  71004 != 72545  => input routed to a DIFFERENT, SURVIVING window's child; never to the destroyed W3
```

After close, W3 is gone from the window list and its child (72545) is terminated (`kitty/window.py:888` `close()` → `kitty/window.py:1560` `destroy()`; tab-side `kitty/tabs.py:580` `remove_window()`) **[observed]**. A subsequently-injected key was received by a **surviving** window's child (`SURVIVOR=71004`, ≠ the dead 72545) — post-close input targets the *new* current `active_window()`, never the destroyed window **[observed]**.

An important environment fact was surfaced honestly: under Xvfb with **no window manager**, X does not auto-reassign input focus when a window is destroyed, so the survivor W1 had to be focused explicitly before typing — this is an X/WM behavior, not kitty's, and does not affect the routing conclusion **[observed]**.

The *internal* drop mechanism for a genuinely stale window id is **[inferred]** because it cannot be forced from outside a synchronous keypress: were bytes ever scheduled to a vanished id, `schedule_write_to_child_generic` would find no child (`children[i].id == id` never true `kitty/child-monitor.c:336`), leave `found = false` `kitty/child-monitor.c:325`, and `return found` `kitty/child-monitor.c:369` — i.e. the write is silently dropped; and the `if (!w) return;` guard `kitty/keys.c:236` after the Python dispatch already prevents even reaching that call once the active window has gone **[inferred]**.

---


## (f) R5 — Which parts are Python, which are C, which are external libraries; and ruled-out misreadings

### Ownership, attributed from runtime artifacts [observed]

Derived from the loaded objects in `/proc/70936/maps` (which shared objects the live input process actually maps) cross-referenced with the stack frames in section (d). `cat /tmp/kitty_probe/kitty_maps_libs.txt` (categorized):

| Pipeline stage | Owner | Evidence (loaded object / stack frame / `file:line`) |
|---|---|---|
| X11 transport (event delivery) | **External library** (C) | `libX11.so.6`, `libxcb.so.1` mapped; frames `_glfwDispatchX11Events`, `processEvent` in `glfw-x11.so` |
| Key decode / keymap / compose | **External library** (C) | `libxkbcommon.so`, `libxkbcommon-x11.so` mapped; `Press/Release xkb_keycode …` from `glfw/xkb_glfw.c`; IME via `glfw/ibus_glfw.c` |
| Normalized key event entry | **Vendored GLFW** (C, in-repo) | `_glfwInputKeyboard()` `glfw/input.c:306`; focus entry `_glfwInputWindowFocus()` `glfw/window.c:45` — inlined into `glfw_xkb_handle_key_event` frame in `glfw-x11.so` |
| Callback glue, recipient decision, encode, child-write scheduling | **Kitty C extension** (in-repo) | `kitty/fast_data_types.so` frames `key_callback` `kitty/glfw.c:430`, `on_key_input` `kitty/keys.c:166`, `active_window` `kitty/keys.c:106`, `encode_glfw_key_event` `kitty/key_encoding.c:414`, `schedule_write_to_child` `kitty/child-monitor.c:372` |
| Shortcut resolution & focus fan-out | **Python** (embedded CPython 3.14.6) | `libpython3.14.so` mapped; Python interpreter frames above `main_loop`; `boss.dispatch_possible_special_key` `kitty/boss.py:1408`, `boss.on_focus` `kitty/boss.py:1651`, `window.focus_changed` `kitty/window.py:1123` |
| GPU/software rendering | **External library** (C) | `libGL`, `libGLX_mesa`, `libgallium-25.2.8`, `libLLVM.so.20.1` mapped; `llvmpipe-*` threads (section (d)) — not on the input path |

The division is clear from what is mapped and what appears in the stacks: **external C libraries** own transport (X11) and key decoding (xkbcommon); the **vendored GLFW C** (in-repo) owns the normalized event entry; the **kitty C extension** owns recipient selection, encoding, and child-write scheduling; **Python** owns shortcut policy and focus fan-out **[observed]**.

### Ruled-out interpretation (1): "Go participates in the core keyboard input path" — REFUTED [observed]

`cat /tmp/kitty_probe/r5_attribution_evidence.txt` (relevant slice):

```
## Rule-out (1) Go NOT in input path:
go.mod:3 => go 1.22 ; kitten = separate Go binary (go1.24.4, 16MB, stripped) at kitty/launcher/kitten
NO Go runtime mapped in /proc/70936/maps ; NO kitten process running during input ; NO Go frames in any R3 stack
```

The Go component under `tools/` (`go.mod:3`) builds only the standalone `kitten` CLI, which is a **separate** stripped Go ELF binary at `kitty/launcher/kitten` — not linked into the kitty process **[observed]**. During input there is no `kitten` process running, no Go runtime object in `/proc/70936/maps`, and zero Go frames in any of the section-(d) stacks **[observed]**. Go is therefore not on the core input path — it is a separate remote-control/CLI process **[observed]**.

### Ruled-out interpretation (2): "The shell child, or the X server, reads the keyboard directly" — REFUTED [observed]

If the child (or X) read the keyboard directly, the *same* physical key would always reach the *same* consumer regardless of which kitty window is focused. It does not. The identical physical key `a` was injected four times, alternating focus W1/W2/W1/W2, with a raw reader in each child:

```
## Rule-out (2) neither shell child nor X server reads keyboard directly:
SAME physical 'a' -> W1 child got 2 bytes (while W1 focused), W2 child got 2 bytes (while W2 focused). r5_W1.log / r5_W2.log.
```

`r5_W1.log` shows two `BYTES b'a'` (the two presses made while W1 was focused) and `r5_W2.log` shows two `BYTES b'a'` (the two while W2 was focused) **[observed]**. The same physical key reaches *different* children depending purely on kitty's focus state — so the byte only reaches a child *after* kitty's `active_window()` selection `kitty/keys.c:106` and `schedule_write_to_child` `kitty/keys.c:259`; neither the child nor the X server is doing the reading **[observed]**.

### Ruled-out interpretation (3): "Each window/tab runs its own input-reading thread" — REFUTED [observed]

The OS-thread histogram (section (d), reconfirmed with 2 windows + 2 child shells) is invariant in the input-relevant threads:

```
## Rule-out (3) no per-window input threads:
     33 kitty            (1 main input thread + Mesa gallium workers)
     32 llvmpipe-0..31   (Mesa rasterizer pool)
      1 KittyChildMon    (single PTY multiplexer)
      1 kitty:disk$0
Exactly 1 main 'kitty' input thread + exactly 1 'KittyChildMon' PTY thread regardless of window/child count.
```

There is exactly **one** input-driving thread (Thread 1, running `key_callback`/`on_key_input`) and exactly **one** PTY multiplexer (`KittyChildMon`, running `io_loop`), no matter how many windows/tabs/children exist **[observed]**. The remaining threads are Mesa's software-GL pool. So input is not read per-window; a single main thread selects the recipient and the single `KittyChildMon` thread services all PTYs `kitty/child-monitor.c` **[observed]**.

---


## (g) R6 — One correctness-vs-responsiveness tradeoff, measured at runtime

**The tradeoff (stated from the data, not from comments):** kitty's default `input_delay=3` (`kitty/options/types.py:536`) deliberately **delays reflecting terminal *output* by up to ~3 ms in order to coalesce it** — buying lower CPU and flicker-free, consistent frames (a correctness/efficiency win) at the cost of ~3 ms of output-reflection latency (a responsiveness cost). Keyboard *input* is not delayed inbound (it runs on the synchronous path proved in section (d)), and `repaint_delay` is bypassed whenever input is pending, so keystroke responsiveness is preserved while background output is batched.

### Measurement 1 — terminal round-trip latency (primary, sensitive metric) [observed]

A child emits a DSR cursor-position query `ESC[6n`; kitty's **main loop** parses it and replies `ESC[…R`; the child times the round trip in raw mode. 500 samples per run, 2 runs per setting. `cat /tmp/kitty_probe/rt_d3_r1.txt rt_d3_r2.txt rt_d0_r1.txt rt_d0_r2.txt`:

```
# input_delay=3  (DEFAULT)                # input_delay=0  (-o input_delay=0)
--- run1 ---                              --- run1 ---
samples=500                               samples=500
min=3.118   median=3.156                  min=0.057   median=0.094
mean=3.162  p90=3.189   max=3.457         mean=0.099  p90=0.122   max=1.054
--- run2 ---                              --- run2 ---
samples=500                               samples=500
min=3.105   median=3.167                  min=0.061   median=0.093
mean=3.173  p90=3.208   max=3.356         mean=0.099  p90=0.122   max=0.945
```

Scale/stability: 500 samples × 2 runs per setting. At the default, the median round-trip is **~3.16 ms** (runs: 3.156 / 3.167); at `input_delay=0` it is **~0.094 ms** (runs: 0.094 / 0.093) **[observed]**. The distributions are tight and reproducible across both runs, so this is a stable effect, not noise **[observed]**. The **median delta ≈ 3.16 − 0.094 ≈ 3.06 ms**, which equals the `input_delay` value — i.e. the extra latency *is* the coalescing window **[observed]**.

### Measurement 2 — CPU under a fast background emitter [observed]

Whole-process CPU (from `/proc/<pid>/stat`) while a background `yes` firehose ran; 3×4 s samples, 2 runs per setting. `grep` of `/tmp/kitty_probe/r6_tradeoff_evidence.txt`:

```
input_delay=3: run1 326.5/323.2/327.0 ; run2 325.2/326.8/326.5   (mean ~325.9%)
input_delay=0: run1 337.5/337.5/329.8 ; run2 336.5/336.2/338.2   (mean ~335.6%)
```

Disabling coalescing (`input_delay=0`) costs **~+10 CPU-points** under load (more main-loop wakes) **[observed]**. The gap is modest because a `yes` firehose keeps the input buffer almost full, and `input_delay` is by design "ignored when the input buffer is almost full" `kitty/options/definition.py:885` — so under saturation both settings wake frequently **[observed]**. A moderate ~1000-writes/s emitter drove both settings to ~394% CPU (render-bound under llvmpipe software GL), which is why **latency (Measurement 1), not CPU, is the sensitive discriminator here** **[observed]**.

### Mechanism (corroboration only — the claim rests on the numbers above) [observed]

The measured 3 ms is produced by the output-coalescing gate in the I/O thread. `kitty/child-monitor.c:1562`–`1567`:

```
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
        // we only wakeup the main loop after input_delay as wakeup is an expensive operation
        // on some platforms, such as cocoa
        if (data_received) {
            if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
            else has_pending_wakeups = true;
```

When a child produces output, the `KittyChildMon` thread wakes the main loop only if more than `OPT(input_delay)` has elapsed since the last wake; otherwise it defers (`has_pending_wakeups = true`) — so bursts of child output are batched into at most one main-loop wake per `input_delay` `kitty/child-monitor.c:1565` **[observed]**. Conversely, the render throttle `repaint_delay` (default 10 ms, `kitty/options/types.py:567`) is skipped when input is pending: `render()` defers only `if (!input_read && time_since_last_render < OPT(repaint_delay))` `kitty/child-monitor.c:875`, and the option's own docs say the delay "is ignored" while there is pending input `kitty/options/definition.py:874` **[observed]**. The definition also warns that a low `input_delay` "might cause flicker in full screen programs that redraw the entire screen" `kitty/options/definition.py:883` — the correctness/quality side of the same knob **[observed]**. These citations only corroborate; the tradeoff itself is established by the measured 3.16 ms vs 0.094 ms round-trip and the ~+10 CPU-points **[observed]**.

---


## (h) Coverage pass — every requirement and sub-part

Each item lists where it is answered, its evidence, and an **[observed]**/**[inferred]** tag with a `file:line` anchor.

| Req | Sub-part | Where | Tag | Anchor |
|---|---|---|---|---|
| **R1** | Multiple tabs + OS windows, rapid focus switching | (b) R1.1 | [observed] | `kitty/boss.py:1583`, `kitty/keys.c:239` |
| R1 | Plain keys + modifiers/alternate (arrows, ctrl+c, alt+b, F1) | (b) R1.2 | [observed] | `kitty/keys.c:254,261`, `kitty/key_encoding.c:414` |
| R1 | Alternate encoding mode (CSI-u vs legacy) | (b) R1.3 | [observed] | `kitty/screen.c:1204,1244,1252` |
| R1 | Typing during resize & scroll (transitional) | (b) R1.4 | [observed] | `kitty/keys.c:231,259` |
| R1 | Background output while a different window is focused | (b) R1.5 | [observed] | `kitty/glfw.c:517`, `kitty/keys.c:254` |
| R1 | Key auto-repeat (held key) | (b) R1.6 | [observed] | `kitty/keys.c:166` |
| **R2a** | Component that sees input first | (c) R2a | [observed] | `glfw/input.c:306`, `kitty/glfw.c:430` |
| **R2b** | Recipient decision (`active_window`) | (c) R2b | [observed] | `kitty/keys.c:106,167` |
| R2b | "no active window" branch | (c) R2b | [inferred] | `kitty/keys.c:182` |
| R2b | Intermediate: shortcut resolution consumes key | (c) R2b | [observed] | `kitty/boss.py:1408,1583`, `kitty/keys.c:231` |
| R2b | Re-fetch + `if(!w) return` after Python dispatch | (c) R2b | [inferred] | `kitty/keys.c:236` |
| **R2c** | Final destination: encode + write by window id | (c) R2c | [observed] | `kitty/key_encoding.c:414`, `kitty/keys.c:259`, `kitty/child-monitor.c:336` |
| R2c | Bytes reach correct child PTY (DA round-trip) | (c) R2c | [observed] | `kitty/child-monitor.c:323`–`369` |
| **R2d** | Focus propagation C→Python fan-out | (c) R2d | [observed] | `glfw/window.c:45`, `kitty/glfw.c:515,527,538`, `kitty/boss.py:1651`, `kitty/window.py:1123,1134`, `kitty/screen.c:4604,4611` |
| **R3** | ptrace not blocked — concrete mechanism proven | (d) R3.0 | [observed] | `/proc/sys/kernel/yama/ptrace_scope`=1 + `CAP_SYS_PTRACE` |
| R3 | Merged Python+native dump (`py-spy --native`) | (d) R3.1 | [observed] | `kitty/main.py:234,518,526` |
| R3 | Live input-ENTRY stack (`gdb` break `key_callback`) | (d) R3.2 | [observed] | `kitty/glfw.c:430`, `glfw/input.c:306` |
| R3 | Live DELIVERY stack (`gdb` break `schedule_write_to_child`) | (d) R3.3 | [observed] | `kitty/keys.c:259`, `kitty/child-monitor.c:372` |
| R3 | Full OS-thread structure; PTY thread in `io_loop` | (d) R3.4 | [observed] | `kitty/child-monitor.c` (`io_loop`) |
| R3 | Blocked-then-fallback chain (`--threads` reject; first gdb timeout) | (d) R3.1, R3.4 | [observed] | — |
| **R4** | Unfocused window receives nothing | (e) R4a | [observed] | `kitty/keys.c:106` |
| R4 | Synthetic focus-loss RELEASE skipped | (e) R4a | [observed] | `glfw/window.c:52,59`, `kitty/glfw.c:439` |
| R4 | Just-closed: input targets new active window, not the dead one | (e) R4b | [observed] | `kitty/window.py:888,1560`, `kitty/tabs.py:580` |
| R4 | Internal stale-id drop mechanism | (e) R4b | [inferred] | `kitty/child-monitor.c:325,336,369`, `kitty/keys.c:236` |
| **R5** | Python/C/external ownership attribution | (f) table | [observed] | `/proc/maps` + section (d) frames |
| R5 | Rule-out (1): Go not in input path | (f) | [observed] | `go.mod:3` |
| R5 | Rule-out (2): child/X server don't read keyboard directly | (f) | [observed] | `kitty/keys.c:106,259` |
| R5 | Rule-out (3): no per-window input threads | (f) | [observed] | `kitty/child-monitor.c` |
| **R6** | One tradeoff, measured (≥2 runs, stability stated) | (g) M1, M2 | [observed] | `kitty/options/types.py:536,567`, `kitty/child-monitor.c:1565,875` |
| **R7** | Repository left unchanged; temp artifacts removed | (i) below | [observed] | `git status --porcelain` |

**Non-canonical items** (shown only as labeled contrasts, never as the observed route): kitty remote control `kitten @ send-text`; GLFW `Null`/OSMesa backend `glfw/null_init.c`; `xdotool --window` (XSendEvent). None underpins any conclusion here **[observed]**.

---

## (i) R7 — Repository left unchanged (verification)

All instrumentation lived under `/tmp/kitty_probe/` and was deleted; the spawned `Xvfb`/kitty processes were killed. From the repository root, `git status --porcelain` shows exactly one new file — this document — and nothing else (build outputs are gitignored):

```
$ git status --porcelain
?? blitzy/
```

`blitzy/` is untracked and contains only `blitzy/documentation/kitty_815df1e210e0.md`. No existing repository file was modified, added, deleted, renamed, or moved; no code was added beyond this Markdown document **[observed]**.

---

## Appendix — raw artifact inventory

Captured under `/tmp/kitty_probe/` during the investigation (all ephemeral; removed in R7). Listed for provenance:

- **Versions:** `versions.txt` · ptrace/caps `r3_ptrace_env.txt`
- **R1:** `kitty_R1_full.log`, `r1_s1_tabs.log`, `r1_s2_modifiers.log`, `r1_s2b_csiu.log`, `r1_s2c_clean.log`, `r1_s2c_csiu8.log`, `r1_s3_resize_scroll.log`, `r1_s4_bgoutput.log`, `r1_s5_repeat.log`
- **R2:** `r2_single_key.log`, `r2_focus_change.log`, `focus_py.txt`, `da.txt`, `winB.txt`
- **R3:** `pyspy_native.txt`, `gdb_keycb_bt.txt`, `gdb_swc_bt.txt`, `gdb_all_bt.txt`
- **R4:** `r4a_unfocused_evidence.txt`, `childW1_raw.log`, `r4a_fakerelease_evidence.txt`, `r4b_justclosed_evidence.txt`, `survivor.txt`
- **R5:** `r5_attribution_evidence.txt`, `kitty_maps_libs.txt`, `r5_W1.log`, `r5_W2.log`
- **R6:** `r6_tradeoff_evidence.txt`, `rt_d3_r1.txt`, `rt_d3_r2.txt`, `rt_d0_r1.txt`, `rt_d0_r2.txt`
- **Harnesses (temporary):** `reader.py` (raw-stdin witness), `emitter.py` (moderate output), `rt_dsr.py` (DSR round-trip timer)

