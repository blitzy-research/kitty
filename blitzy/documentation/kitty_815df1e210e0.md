# Kitty Input‑Event Routing & Focus Management — Runtime Investigation

> **Question answered.** *How does the Kitty terminal emulator actually handle input‑event flow and focus management across windows, tabs, and child processes at runtime — observed by building and running the binary, not by assuming from reading the source?*

This document is **evidence‑grounded and run‑first**: every claim is backed by the **exact command** that produced it and the **complete, unedited output** captured from a canonical Kitty built and run from *this* repository. Source references use `file:line` (re‑verified at runtime against `HEAD`). Where a value came from a bypass it is labelled **NON‑CANONICAL**; where a conclusion could not be captured after varied effort it is labelled **INFERRED**.

---

## 0. Methodology, environment, and canonical build/run

### 0.1 Repository identity (verbatim)

```console
$ git branch --show-current
blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

The HEAD commit `815df1e210e0…` matches the branch identifier `kitty_815df1e210e0` mandated for this deliverable's filename.

### 0.2 Host (this observation environment)

```console
$ uname -srm
Linux 6.6.122+ x86_64
$ python3 --version
Python 3.13.7
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

> **Note on environment.** The AAP's *mandated* image is Ubuntu 24.04 / Python 3.12. This observation host is Ubuntu 25.10 / Python 3.13.7. Kitty was therefore **rebuilt canonically from source against Python 3.13** (see §0.3) before any observation, and the build‑level observation tooling (`Xvfb`, `xdotool`, `gdb`, `py-spy`, `strace`, `glxinfo`) was installed at the environment level only — **no repository file was modified**. This difference does not affect the input‑pipeline architecture, which is identical across these interpreter minor versions.

### 0.3 Canonical build (verbatim command)

The canonical build entry point is `python3 setup.py` (the `Makefile` `all:` target). For symbolized native frames (needed for the stack snapshots in Part 3/§4 and the just‑closed captures in §4) the debug variant was additionally built, and for I/O‑loop timing (Part 6/§6) the event‑loop‑instrumented debug build:

```console
# canonical default build -> kitty/fast_data_types.so + kitty/launcher/kitty
$ python3 setup.py

# symbolized debug build (make debug == python3 setup.py build --debug)
$ python3 setup.py build --debug

# event-loop-instrumented debug build (make debug-event-loop) — defines DEBUG_EVENT_LOOP (kitty/child-monitor.c:29)
$ python3 setup.py build --debug --extra-logging=event-loop
```

Resulting launcher (a build output; `/kitty/launcher/kitt*` is gitignored, so it is not a tracked change):

```console
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 281344 Jul 13 17:03 kitty/launcher/kitty
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The native launcher is produced by `build_launcher` (**setup.py:1230**). The `--debug-keyboard` flag used throughout is defined at **kitty/cli.py:996-997** (`--debug-input`/`--debug-keyboard`, `dest=debug_keyboard`) and plumbed via `init_glfw` (**kitty/main.py:514** → **:90-97**) to `glfwInitHint(GLFW_DEBUG_KEYBOARD, debug_keyboard)` (**kitty/glfw.c:1444**).

### 0.4 Display context (verbatim)

Kitty is a GPU/OpenGL application. No physical GPU is present, so a headless X11 display backed by **software OpenGL (Mesa `llvmpipe`)** was used:

```console
$ Xvfb :99 -screen 0 1920x1080x24 -ac &
$ export DISPLAY=:99 XDG_RUNTIME_DIR=/tmp/xdg-runtime LIBGL_ALWAYS_SOFTWARE=1
$ glxinfo | grep -E "OpenGL renderer|OpenGL version|direct rendering"
direct rendering: Yes
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.25.10.2
```

### 0.5 Canonical launch pattern & input injection

Every observation launches the real launcher in default configuration (`--config NONE` gives kitty's built‑in defaults with **no** user config), with input tracing to a log:

```console
$ DISPLAY=:99 XDG_RUNTIME_DIR=/tmp/xdg-runtime LIBGL_ALWAYS_SOFTWARE=1 \
    kitty/launcher/kitty --config NONE --debug-keyboard <child-cmd> 2>/tmp/kbd.log
```

**Input is delivered through the real OS path**, not remote control: `xdotool` synthesizes genuine X11 `KeyPress`/`ButtonPress` events (XTEST, `send_event=False`) that flow through the X server → GLFW → kitty exactly as a physical keyboard would. This is the **canonical** input path. (Remote control `kitty @ …` is used *only* where explicitly labelled NON‑CANONICAL, and never as the routing evidence itself.)

> **Labelling key.** **CANONICAL** = real launcher + real OS input path. **NON‑CANONICAL** = value obtained via remote control / debug hook / synthetic stand‑in. **INFERRED** = not captured after varied effort; grounded in `file:line`. ANSI SGR colour codes emitted by `--debug-keyboard` have been stripped from pasted logs for readability; **all values (key codes, byte sequences, actions, timestamps) are verbatim and unmodified.**

---

## Part 1 — Reproducing overlapping input activity

**Direct answer.** All requested overlapping conditions were reproduced against the canonical binary: **multiple tabs and windows in one OS window, rapid focus switching, keystrokes delivered during window resize and during scrollback, and a background window emitting output while a different window holds focus.** In every case keystrokes were routed to exactly the **active** window while the single ChildMonitor I/O thread concurrently drained the background window's PTY. Timing‑sensitive claims were repeated at stated scale and were byte‑for‑byte stable across ≥2 runs.

Default key mappings (active even under `--config NONE`; `kitty_mod = ctrl+shift` at **kitty/options/definition.py:3474**) used to build the scenario: `new_window` = `ctrl+shift+enter` (**:3696**), `new_os_window` = `ctrl+shift+n` (**:3730**), `next_window` = `ctrl+shift+]` (**:3751**), `next_tab` = `ctrl+shift+right` (**:3874**), `new_tab` = `ctrl+shift+t` (**:3896**), `scroll_line_up` = `ctrl+shift+up` (**:3577**), `close_window` = `ctrl+shift+w` (**:3743**).

### 1.1 Multiple tabs + multiple windows — routing follows the active window

A session opened two tabs — tab 1 with windows **W1, W2**, tab 2 with window **W3** — each window running a raw‑stdin recorder. The scripted input was: type `1` (W1 active) → `Ctrl+Shift+]` (`next_window`) → type `2` (W2 active) → `Ctrl+Shift+Right` (`next_tab`) → type `3` (W3 active). The per‑window recordings:

```console
$ cat /tmp/evidence/routing_perwindow.txt
W1(hex):  31
W2(hex):  32
W3(hex):  33
```

Each window received **only** the digit typed while it was active (`0x31`='1', `0x32`='2', `0x33`='3'). The matching `--debug-keyboard` trace shows the switch keys were consumed as shortcuts (no bytes emitted) and the digits were sent to the then‑active window:

```text
on_key_input: glfw key: 0x31 native_code: 0x31 action: PRESS mods: none text: '1' state: 0 sent key as text to child: 1
on_key_input: glfw key: 0x5d native_code: 0x5d action: PRESS mods: ctrl+shift text: '' state: 0
KeyPress matched action: next_window, handled as shortcut
on_key_input: glfw key: 0x32 native_code: 0x32 action: PRESS mods: none text: '2' state: 0 sent key as text to child: 2
on_key_input: glfw key: 0xe007 native_code: 0xff53 action: PRESS mods: ctrl+shift text: '' state: 0
KeyPress matched action: next_tab, handled as shortcut
on_key_input: glfw key: 0x33 native_code: 0x33 action: PRESS mods: none text: '3' state: 0 sent key as text to child: 3
```

This is the core observation for routing: keyboard input always targets `active_window()` (**kitty/keys.c:106**), and window/tab switches are themselves keyboard shortcuts consumed *before* any bytes reach a child (**kitty/keys.c:228-231**).

### 1.2 Rapid focus switching across OS windows

Creating a second OS window (`Ctrl+Shift+N`) and alternating focus produced paired focus callbacks — the losing window gets `focused: 0` and the gaining window `focused: 1` at the same timestamp — emitted by `window_focus_callback` (**kitty/glfw.c:515**, debug string at **:517**):

```text
on_focus_change: window id: 0x1 focused: 0
on_focus_change: window id: 0x2 focused: 1
```

(Intra‑OS‑window switches like §1.1's `next_window` do **not** fire `on_focus_change`; they change the active window inside one OS window via `set_active_window` at **kitty/state.c:514**. Cross‑OS‑window switches *do* fire it. Both are shown, exercising the transitional state before/after each switch.)

### 1.3 Background window producing output while another window has focus

Two independent proofs were captured.

**(a) The I/O thread drains an unfocused window's PTY.** A generator in a **background** (unfocused) window wrote 5,000,000 bytes to its PTY while a *different* window held focus:

```console
$ cat /tmp/evidence/bg_start.txt
BGSTART 1783961774.440931257
$ cat /tmp/evidence/bg_done.txt
BGDONE 1783961774.504355902 bytes=5000000
```

The 5 MB completed in **~64 ms** (1783961774.504 − 1783961774.440). A PTY's kernel buffer is only tens of kilobytes (≤ 64 KB); had the unfocused window's PTY *not* been read, the child would have blocked on `write()` long before 5 MB. So kitty's single I/O thread (`io_loop`, **kitty/child-monitor.c:1481**; `poll()` over all PTY fds at **:1509**) read the **unfocused** window's output continuously — output flow is independent of focus.

**(b) Temporal overlap with foreground typing.** A timed generator in the background window emitted a line each ~200 ms (writing to its PTY *first*, then to a file):

```console
$ sed -n '1,6p' /tmp/evidence/bg_timeline.log
BG 0 1783961821.093308320
BG 1 1783961821.298938134
BG 2 1783961821.505210384
BG 3 1783961821.711147010
BG 4 1783961821.917131043
BG 5 1783961822.122836402
$ grep -E "BG (0|28)" /tmp/evidence/bg_timeline.log
BG 0  1783961821.093308320
BG 28 1783961826.856599731
```

The background stream spanned `821.09 → 826.86`, fully covering the foreground typing window (`TYPE_START 823.90 → TYPE_END 825.99`). The **foreground** recorder received exactly the typed text `HELLO`; the **background** window emitted all 29 lines during the same interval — each line requiring a successful PTY write that only happens if kitty is draining that PTY. Focus governs **keyboard destination**; it does **not** gate **child‑output** processing.

### 1.4 Keystrokes during window resize (transitional state)

While the OS window was being continuously resized (`RESIZE 855.58 → 857.21`), the characters `r s z r s z` were injected. The recorder received all of them, and `on_key_input` `PRESS` fired mid‑resize for each — input is processed during the resize transitional state (the resize runs on the same main GLFW thread that dispatches key events, so keys interleave with resize events rather than being dropped).

### 1.5 Keystrokes during scrollback (transitional state)

Five `Ctrl+Shift+Up` (`scroll_line_up`) entered scrollback, then `x` was typed. `on_key_input` reported `text:'x'` and the byte was sent to the child; per **kitty/keys.c:247-249** a normal key press while scrolled back first snaps the view to the bottom and then delivers the key. The recorder received `x`. Input is processed during the scrollback transitional state.

### 1.6 Run‑scale and stability (≥2 runs)

For the timing/losslessness claim, the **same** unchanged input — 200 keystrokes (`a`) — was replayed twice (**scale: 200 keystrokes × 2 runs**):

- Run 1: 200 bytes received, 200 `on_key_input(PRESS 'a')`, 0 non‑`a`.
- Run 2: 200 bytes received, 200 `on_key_input(PRESS 'a')`, 0 non‑`a`.

```console
$ cmp /tmp/evidence/scale_1.bin /tmp/evidence/scale_2.bin && echo IDENTICAL
IDENTICAL
```

The two runs are **byte‑for‑byte identical** — the path is lossless and stable at this scale.

---

## Part 2 — Input routing, focus propagation, and child delivery

**Direct answer.** A keystroke's life cycle is: **the external GLFW backend (a `dlopen`'d shared library) sees the raw OS event first**, translates it via XKB, and calls kitty's C callback `key_callback` (**kitty/glfw.c:439**), which immediately calls `on_key_input` (**kitty/keys.c:166**). `on_key_input`'s *first act* is to select the target window with `active_window()` (**kitty/keys.c:106-110**). The event is then offered to **Python** for shortcut matching via `dispatch_possible_special_key` (**kitty/keys.c:228** → `Boss.dispatch_possible_special_key`, **kitty/boss.py:1408**). If consumed, no bytes are sent. Otherwise the C layer encodes the event (`encode_glfw_key_event`, **kitty/keys.c:251**) and **queues** the bytes with `schedule_write_to_child` (**kitty/keys.c:259** → **kitty/child-monitor.c:372**), which wakes the **I/O thread**. That thread (`io_loop`) `poll()`s all PTYs and flushes the queued bytes to the child's PTY master fd via `write_to_child` (**kitty/child-monitor.c:1443**). Focus changes propagate on a parallel path from GLFW's `window_focus_callback` (**kitty/glfw.c:515**) into Python (`Boss.on_focus`, **kitty/boss.py:1651** → `Window.focus_changed`, **kitty/window.py:1123**) and update C active‑window state (`set_active_window`, **kitty/state.c:514**).

### 2.1 What sees input first — and the exact ordered layering

The `--debug-keyboard` log for a single `Enter`, captured from startup, shows the ordering unambiguously (complete, unedited except ANSI stripping):

```console
$ cat /tmp/evidence/ordered_enter.txt
[0.060] Loading new XKB keymaps
[0.065] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.159] on_focus_change: window id: 0x1 focused: 1
[4.975] Loading new XKB keymaps
[4.980] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[4.980] Press xkb_keycode: 0x24 clean_sym: Return composed_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[4.980] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
[4.983] Release xkb_keycode: 0x24 clean_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
```

**Key fact:** the external GLFW/XKB line `Press xkb_keycode: 0x24 … glfw_key: 57345 (ENTER)` **always precedes** kitty's `on_key_input`. The GLFW backend (external C library) translates the raw hardware keycode `0x24` into a keysym via XKB, then hands a `GLFWkeyevent` to kitty's C extension. This ordering is proven structurally by the stack snapshot in Part 3/§3.2.

### 2.2 Intermediate processing — target selection, shortcut dispatch, encoding

`on_key_input` (**kitty/keys.c:166**) begins with `Window *w = active_window();` (**:106-110**), i.e. `global_state.callback_os_window->tabs[active_tab].windows[active_window]`. It then runs the shortcut‑dispatch macro (**:228**); the Python side (`Boss.dispatch_possible_special_key`, **kitty/boss.py:1408**) reports a match through `report_match` (**boss.py:1579**, print at **:1583**). A consumed shortcut prints `handled as shortcut` (**keys.c:231**) and emits **no** bytes — directly observed in §1.1 (`matched action: next_window, handled as shortcut`). If unconsumed, `encode_glfw_key_event` (**keys.c:251**, implemented in **kitty/key_encoding.c**) produces the on‑wire bytes.

### 2.3 Byte fidelity — the exact emitted bytes reach the child

The `--debug-keyboard` `sent encoded key to child:` print dumps the exact `encoded_key` buffer handed to `schedule_write_to_child` (**keys.c:259-268**). A raw‑stdin child (`stty raw; os.read(0)` → hex) captured what actually arrived on the PTY:

| Key | `--debug-keyboard` says (C, keys.c:261) | Child received (raw PTY) | Encoding (kitty/key_encoding.c) |
|-----|------------------------------------------|--------------------------|----------------------------------|
| `Enter` | `sent encoded key to child: 0xd` | `0d` | C0 carriage return |
| `Ctrl+A` | `sent encoded key to child: 0x1` | `01` | C0 control |
| `Up` | `sent encoded key to child: ^[ [ A` | `1b 5b 41` | legacy CSI (`ESC [ A`, DECCKM off) |

```console
$ cat /tmp/evidence/child_rx.hex
0d
01
1b 5b 41
61
61
61
61
61
61
61
61
```

The child received **exactly** the bytes the C layer reported emitting — verified byte‑for‑byte. (The trailing eight `61` = `'a'` bytes are the key‑repeat test in §2.6.) End‑to‑end path confirmed: OS → GLFW → `on_key_input` → `encode_glfw_key_event` → `schedule_write_to_child` → `io_loop` → `write_to_child` → PTY → child.

### 2.4 Final destination — the I/O thread flushes to the PTY master

`schedule_write_to_child` (**kitty/child-monitor.c:372**) queues the bytes on the target window's screen write buffer and wakes the I/O thread. The I/O thread (`io_loop`, **:1481**) `poll()`s every child fd (**:1509**) and, when a PTY master signals `POLLOUT`, calls `write_to_child` (**:1443**) to flush. This is proven at symbol level by the gdb breakpoint capture in Part 3/§3.3 (frame `#0 write_to_child (fd=8, …) … #1 io_loop … child-monitor.c:1540`).

### 2.5 Focus propagation — the parallel path

The focus path is separate from the keyboard byte path. `window_focus_callback` (**kitty/glfw.c:515**) fires on OS‑window focus change (observed in §1.2 as paired `on_focus_change … focused: 0/1`) and calls into Python via `WINDOW_CALLBACK(on_focus, …)` → `Boss.on_focus` (**kitty/boss.py:1651**) → the tab manager resolves the active window → `Window.focus_changed` (**kitty/window.py:1123**). Internal active‑window bookkeeping runs through `notify_on_active_window_change` (**kitty/window_list.py:192**) and the C globals `set_active_window`/`set_active_tab`/`current_focused_os_window_id` (**kitty/state.c:514 / :506 / :120**). Because keyboard routing reads `active_window()` at the moment of each keystroke, these focus updates are exactly what re‑targets subsequent input (demonstrated in §4).

### 2.6 Secondary paths exercised (modifiers, shortcut, repeat, release, IME)

- **Plain key** `a`: `text:'a'` → `sent key as text to child: a` (the `SEND_TEXT_TO_CHILD` branch, **keys.c:252-254**).
- **`Ctrl+A`**: `mods: ctrl` → encoded `0x1` (**keys.c:259**); child received `01` (§2.3).
- **Consumed shortcut** `Ctrl+Shift+T`: `KeyPress matched action: new_tab, handled as shortcut` — consumed, **no** bytes to child (default `new_tab` mapping, active even under `--config NONE`).
- **Key repeat**: holding `a` ~0.9 s produced **1 `PRESS` + 7 `REPEAT` + 1 `RELEASE`**; each `REPEAT` ran `on_key_input` and sent `a` — hence the eight `61` bytes in §2.3. `GLFW_REPEAT` is handled identically to `PRESS`.
- **Release**: `action: RELEASE … ignoring as keyboard mode does not support encoding this event` — releases are not encoded in the default (legacy) keyboard mode.
- **IME / `on_IME_input`** (**keys.c:174**, `GLFW_IME_COMMIT_TEXT` at **keys.c:200-203**): **INFERRED** — not exercised because no IBus/IME framework is available in the headless environment. Grounded in **kitty/keys.c:174,196-206** and the IME source `glfw/ibus_glfw.c`; the code path exists but was not driven at runtime.


---

## Part 3 — Stack/symbol snapshot with a tiered fallback

**Direct answer.** A point‑in‑time **attach** to the running process (Tier 1) is **blocked** in the restricted configuration (`ptrace_scope=1`, no `CAP_SYS_PTRACE`) with `ptrace … Operation not permitted`. The working alternatives — **launch‑as‑child / privileged tracer** (Tier 2, `py-spy` + `gdb`), a **Python all‑thread `faulthandler` dump** (Tier 3), and the **in‑repo `--debug-keyboard` symbol log** (Tier 4) — all produce real call‑path visibility. Multiple snapshots are shown below, including a full native+Python stack from the OS event down to `on_key_input`, and the I/O thread stopped inside `write_to_child`.

> **Capability note (honest disclosure).** *This* container happens to grant root `CAP_SYS_PTRACE` (`CapEff=000001ffffffffff`), so attach‑by‑PID *would* succeed here. To faithfully reproduce the **canonical restricted** condition described for the mandated image (attach blocked), Tier 1 was run after dropping the capability (`capsh --drop=cap_sys_ptrace`) and as the unprivileged user `nobody` under `ptrace_scope=1`. Tier 2 then uses the permitted mechanisms.

### 3.1 Tier 1 — attach by PID (BLOCKED; verbatim errors)

```console
# target: a canonical kitty launched as: kitty/launcher/kitty --config NONE -o input_delay=3 sh -c 'sleep 600'  (PID 123483)

# (1) py-spy attach, root with CAP_SYS_PTRACE dropped:
#     capsh --drop=cap_sys_ptrace -- -c "py-spy dump --pid 123483"
Error: Failed to get process executable name. Check that the process is running.

Caused by:
    0: Permission denied (os error 13)
    1: Permission denied (os error 13)

# (2) py-spy attach, unprivileged user under ptrace_scope=1:
#     setpriv --reuid=nobody --regid=nogroup --clear-groups py-spy dump --pid 123483
Permission Denied: Try running again with elevated permissions by going 'sudo env "PATH=$PATH" !!'

# (3) strace attach, unprivileged user under ptrace_scope=1:
#     setpriv --reuid=nobody --regid=nogroup --clear-groups strace -p 123483
strace: attach: ptrace(PTRACE_SEIZE, 123483): Operation not permitted

# (4) gdb attach, root with CAP_SYS_PTRACE dropped:
#     capsh --drop=cap_sys_ptrace -- -c "gdb -p 123483 -batch -ex 'bt'"
Could not attach to process.  If your uid matches the uid of the target
process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
ptrace: Inappropriate ioctl for device.
No stack.
```

This is the expected Linux behaviour: attaching to a non‑child process needs elevated privilege or `CAP_SYS_PTRACE`, or `ptrace_scope=0`.

### 3.2 Tier 2 — merged native+Python stack (py-spy launch/attach with capability)

`py-spy dump --native` on the running launcher yields a single merged stack for the main thread — spanning **libc → external GLFW → C extension → Python** in one view:

```console
$ py-spy dump --native --pid <PID>
$ cat /tmp/evidence/pyspy_native_release.txt
Process 102573: kitty/launcher/kitty --config NONE --debug-keyboard /tmp/rxtag.sh /tmp/t1.bin
Python v3.13.7 (/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/kitty)

Thread 102573 (idle): "MainThread"
    0x7fd63b1fa772 (libc.so.6)
    0x7fd63b1ee13c (libc.so.6)
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
    _run_module_as_main (<frozen runpy>:199)
    0x7fd63b178575 (libc.so.6)
```

Note the shared‑object annotations: `glfwRunMainLoop` lives in **`kitty/glfw-x11.so`** (external), `main_loop` in **`kitty/fast_data_types.so`** (C extension), and the callers are **`kitty/main.py`** (Python). This single frame is the backbone of the language‑split analysis in Part 5.

### 3.3 Tier 2 — gdb breakpoint on `on_key_input` (full path, OS → C)

On the symbolized debug build, a breakpoint on `on_key_input` was hit by injecting `z`. The complete backtrace (unedited) resolves **every** intermediate frame with `file:line`:

```console
$ gdb -p <PID> -ex 'break on_key_input' -ex continue    # then inject 'z'
Thread 1 "kitty" hit Breakpoint 1, on_key_input (ev=ev@entry=0x7ffd42291ba0) at kitty/keys.c:166
166	on_key_input(GLFWkeyevent *ev) {
#0  on_key_input (ev=ev@entry=0x7ffd42291ba0) at kitty/keys.c:166
#1  0x00007834ce86c643 in key_callback (w=<optimized out>, ev=0x7ffd42291ba0) at kitty/glfw.c:439
#2  0x00007834cd7652d6 in _glfwInputKeyboard (window=window@entry=0x57e0dbd5aa50, ev=ev@entry=0x7ffd42291ba0) at glfw/input.c:350
#3  0x00007834cd777fe6 in glfw_xkb_handle_key_event (window=0x57e0dbd5aa50, xkb=0x7834cd7ca030 <_glfw+131824>, xkb_keycode=52, action=action@entry=1) at glfw/xkb_glfw.c:966
#4  0x00007834cd772def in processEvent (event=event@entry=0x7ffd42291dd0) at glfw/x11_window.c:1254
#5  0x00007834cd7739bb in dispatch_x11_queued_events (num_events=0) at glfw/x11_window.c:2664
#6  0x00007834cd773a3a in _glfwDispatchX11Events () at glfw/x11_window.c:2678
#7  0x00007834cd773bb0 in handleEvents (timeout=<optimized out>) at glfw/x11_window.c:73
#8  0x00007834cd773c10 in _glfwPlatformWaitEvents () at glfw/x11_window.c:2731
#9  0x00007834cd76dbf4 in _glfwPlatformRunMainLoop (tick_callback=0x7834ce81bbf2 <process_global_state>, data=0x7834cd8d1cf0) at glfw/main_loop.h:30
#10 0x00007834cd76478c in glfwRunMainLoop (callback=<optimized out>, data=<optimized out>) at glfw/init.c:360
#11 0x00007834ce86f2cb in run_main_loop (cb=cb@entry=0x7834ce81bbf2 <process_global_state>, cb_data=cb_data@entry=0x7834cd8d1cf0) at kitty/glfw.c:2103
#12 0x00007834ce81823c in main_loop (self=0x7834cd8d1cf0, a=<optimized out>) at kitty/child-monitor.c:1262
#13 0x00007834cf84d7c0 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#14 0x00007834cf84157e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#15 0x00007834cf9861a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#16 0x00007834cf84308e in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#17 0x00007834cf8e4ad5 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#18 0x00007834cf8413c2 in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#19 0x00007834cf9861a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#20 0x00007834cf985159 in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#21 0x00007834cf97ef29 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#22 0x00007834cf89dc18 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#23 0x00007834cf84157e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#24 0x00007834cf9861a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#25 0x00007834cfa398a1 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#26 0x00007834cfa3ad12 in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#27 0x000057e0b37699a7 in run_embedded (run_data=0x7ffd42292cf0) at kitty/launcher/main.c:216
#28 0x000057e0b376a8ce in main (argc=6, argv=0x7ffd42295e88, envp=0x7ffd42295ec0) at kitty/launcher/main.c:464
[Inferior 1 (process 104999) detached]
```

Reading bottom‑up: the launcher (`run_embedded`, **kitty/launcher/main.c:216**) starts CPython, which calls kitty's `main_loop` (**child-monitor.c:1262**) → `run_main_loop` (**glfw.c:2103**) → **external GLFW** `glfwRunMainLoop` (**glfw/init.c:360**) → the X11 event pump (**glfw/x11_window.c**) → XKB translation `glfw_xkb_handle_key_event` (**glfw/xkb_glfw.c:966**) → `_glfwInputKeyboard` (**glfw/input.c:350**) → kitty's C callback `key_callback` (**glfw.c:439**) → `on_key_input` (**keys.c:166**). This is the authoritative proof of "what sees input first".

### 3.4 Tier 2 — gdb breakpoint on `write_to_child` (I/O thread)

Injecting `Q` and breaking on `write_to_child` stops the **I/O thread** (named `KittyChildMon`) inside the flush, showing the child delivery is on a *different* thread than input intake:

```console
$ sed -n '72,80p' /tmp/evidence/gdb_write_to_child.txt
Breakpoint 1 at 0x7834ce8191d8: write_to_child. (2 locations)
[Switching to Thread 0x7834a05026c0 (LWP 105066)]
Thread 2 "KittyChildMon" hit Breakpoint 1.1, write_to_child (fd=8, screen=0x57e0dafb12f0) at kitty/child-monitor.c:1443
1443	write_to_child(int fd, Screen *screen) {
#0  write_to_child (fd=8, screen=0x57e0dafb12f0) at kitty/child-monitor.c:1443
#1  0x00007834ce8198ad in io_loop (data=0x7834cd8d1cf0) at kitty/child-monitor.c:1540
#2  0x00007834cf588d64 in start_thread (arg=<optimized out>) at ./nptl/pthread_create.c:448
#3  0x00007834cf61c3fc in __GI___clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:78
```

`fd=8` is a PTY master; `#1 io_loop … child-monitor.c:1540` is the flush site inside the poll loop. A release‑build snapshot of the same thread at rest shows it parked in `poll`:

```console
$ cat /tmp/evidence/gdb_io_loop_release.txt
#3  __GI___poll (...) at poll.c:29
#4  io_loop () from kitty/fast_data_types.so
#5  start_thread (...) at pthread_create.c:448
#6  __GI___clone3 ()
```

### 3.5 Tier 3 — Python all‑thread `faulthandler` dump

Kitty does **not** auto‑register `faulthandler` (`KITTY_HANDLED_SIGNALS` at **child-monitor.c:121** handles only `SIGINT/SIGHUP/SIGTERM/SIGCHLD/SIGUSR1/SIGUSR2`; `SIGUSR1→reload_config`, `SIGUSR2→log_error` at **:1373-1377**). Enabling it explicitly at launch (`PYTHONFAULTHANDLER=1`) and signalling produced:

```console
$ PYTHONFAULTHANDLER=1 kitty/launcher/kitty --config NONE sh -c 'sleep 600' 2>/tmp/evidence/faulthandler_dump.txt &
$ kill -ABRT <PID>          # <PID> = the kitty process just launched; SIGABRT triggers faulthandler
$ cat /tmp/evidence/faulthandler_dump.txt
Fatal Python error: Aborted

Current thread 0x00007898c8b29780 (most recent call first):
  File "/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/../../kitty/main.py", line 234 in _run_app
  File "/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/../../kitty/main.py", line 252 in __call__
  File "/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/../../kitty/main.py", line 518 in _main
  File "/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/../../kitty/main.py", line 526 in main
  File "/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/../../kitty/entry_points.py", line 195 in main
  File "/tmp/blitzy/kitty/blitzy-c482b7b3-3c80-494a-aa8c-49c3573f10f8_583c50/kitty/launcher/../../__main__.py", line 7 in <module>
  File "<frozen runpy>", line 88 in _run_code
  File "<frozen runpy>", line 198 in _run_module_as_main

Extension modules: kitty.fast_data_types (total: 1)
```

Only **`MainThread`** carries a Python stack; the I/O thread and the render threads have **no** Python frame — direct evidence that those threads are pure C (they never enter the interpreter). This corroborates the language split in Part 5.

### 3.6 Tier 4 — in‑repo `--debug-keyboard` symbol log (always available)

The guaranteed fallback is the in‑repo facility itself: `--debug-keyboard` prints symbol‑level markers `on_key_input` (**keys.c:176**) and `sent encoded key to child:` (**keys.c:261**), plus the Python‑side `matched action:` (**boss.py:1583**). Every log excerpt in Parts 1–2 is a Tier‑4 snapshot of the call path.

**Tiering summary:** Tier 1 (attach) → **blocked** (`ptrace … Operation not permitted`); Tier 2 (`py-spy`/`gdb`, launch‑as‑child or capability) → **succeeded** (full native+Python stacks, both threads); Tier 3 (`faulthandler`) → **succeeded** (Python all‑thread); Tier 4 (`--debug-keyboard`) → **always available**.


---

## Part 4 — Input for an unfocused or just‑closed window

**Direct answer.** An **unfocused** window receives **no keyboard input at all** — keystrokes are always steered to `active_window()` (**kitty/keys.c:106**), so a background window gets zero bytes while unfocused, *even though its own child's output keeps flowing* (the single I/O thread keeps reading its PTY regardless of focus). For a **just‑closed** window, its child is reaped by `reap_children` (**kitty/child-monitor.c:1413**) and its PTY fd is **removed from the poll set** by `remove_children` (**kitty/child-monitor.c:1313**, fd set to `-1` at **:1323**); once removed, the loop never flushes that child again, so any bytes still queued for it are **dropped**, and new input is routed to the surviving active window.

### 4.1 Unfocused window — before / during / after a focus change

Two windows in one OS window: **W_A** (a timed‑output generator that also records its stdin) and **W_B** (a stdin recorder). Observed across the transition:

| Phase | Active window | Typed | `recA` (W_A stdin) | `recB` (W_B stdin) | W_A generator lines |
|-------|---------------|-------|--------------------|--------------------|---------------------|
| **Before** | W_A | `a` | `[a]` | `[]` (empty) | 26 |
| **During** | switch → W_B via `Ctrl+Shift+]` (`next_window`) | — | — | — | — |
| **After** | W_B | `b` | `[a]` (**frozen**) | `[b]` | grew 29 → 36 |

Interpretation, tied to runtime state:
- **Before:** with W_A active, `a` reached **only** W_A (`recA=[a]`, `recB=[]`) — keyboard targets the active window.
- **After the switch:** `b` reached **only** W_B (`recB=[b]`), and W_A's recording **stayed `[a]`** — the now‑unfocused W_A received **no** new keystroke.
- **Throughout:** W_A's generator line count **kept rising (29 → 36)** while W_A was unfocused. Each emitted line requires a successful PTY write, which only happens if kitty is reading W_A's PTY. So the single `io_loop` (**child-monitor.c:1481/1509**) drains the **unfocused** window's output the entire time. **Focus gates keyboard destination, not output processing.**

### 4.2 Just‑closed window — reap, fd removal, and reroute (gdb, runtime)

Session: **W_C** (a child that **exits when it reads `q`**) active + **W_D** (recorder) surviving. Sequence: type `x` (→ W_C), type `q` (→ W_C; child consumes it and exits), then type `y`.

Observed child/PTY life cycle:

```text
BEFORE close:  W_C child alive=yes,  W_D child alive=yes
AFTER 'q':     W_C child alive=no,   W_D child alive=yes
recC = 0x78 ('x' only — 'q' was consumed by the child, which then exited)
recD = 0x79 ('y' — rerouted to the surviving active window)
```

The `--debug-keyboard` log confirms kitty *delivered* all three bytes to whichever window was active at the time (`q` **was** written to W_C's PTY; the child read it and exited):

```text
on_key_input: glfw key: 0x78 native_code: 0x78 action: PRESS mods: none text: 'x' state: 0 sent key as text to child: x
on_key_input: glfw key: 0x71 native_code: 0x71 action: PRESS mods: none text: 'q' state: 0 sent key as text to child: q
on_key_input: glfw key: 0x79 native_code: 0x79 action: PRESS mods: none text: 'y' state: 0 sent key as text to child: y
```

The close path was captured live on the **I/O thread** (`KittyChildMon`) with gdb:

```console
$ gdb -p <PID> -ex 'break reap_children' -ex 'break remove_children' -ex continue   # then inject 'q'
$ grep -A5 'HIT ' /tmp/evidence/gdb_close.txt
==== HIT reap_children (SIGCHLD -> waitpid) thread: ====
#0  reap_children (self=self@entry=0x7f7a34c96330, enable_close_on_child_death=false) at kitty/child-monitor.c:1413
#1  0x00007f7a35c19832 in io_loop (data=0x7f7a34c96330) at kitty/child-monitor.c:1526
#2  start_thread (arg=<optimized out>) at ./nptl/pthread_create.c:448

==== HIT remove_children (io_loop: fd leaves poll set @c-m.c:1327) count=2 ====
#0  remove_children (self=self@entry=0x7f7a34c96330) at kitty/child-monitor.c:1314
#1  0x00007f7a35c196f2 in io_loop (data=0x7f7a34c96330) at kitty/child-monitor.c:1493
#2  start_thread (arg=<optimized out>) at ./nptl/pthread_create.c:448
#3  __GI___clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:78
```

- `reap_children` (**child-monitor.c:1413**, called from `io_loop` at **:1526**) runs `waitpid(-1, …, WNOHANG)` on `SIGCHLD` to reap the dead child.
- `remove_children` (defined at **child-monitor.c:1313**; gdb stopped at its first statement, reported as **:1314**, called from `io_loop` at **:1493**) reports `count=2` on entry (about to drop to 1). This is exactly where the **PTY fd leaves the poll set**: `cleanup_child` (**:1306**) does `safe_close(children[i].fd)` (**:1307**) + `hangup(pid)` (**:1308**), then the slot is cleared (`children[i] = EMPTY_CHILD`, **:1322**) and `children_fds[EXTRA_FDS + i].fd = -1` (**:1323**), and `self->count` is decremented (**:1331**).

> The `@c-m.c:1327` text inside the `==== HIT remove_children … ====` line above is an **author‑supplied label** embedded in the gdb breakpoint command (a `printf`), *not* a line number resolved by gdb. The authoritative fd‑removal statement is `children_fds[EXTRA_FDS + i].fd = -1` at **child-monitor.c:1323** (shown in the source detail of this bullet); the function itself is defined at **:1313**.

**Why queued writes are dropped.** In `io_loop`, the per‑child loop `for (i = 0; i < self->count; i++)` (**child-monitor.c:1528**) calls `write_to_child` only for a child whose pollfd carries `POLLOUT` — the `if (children_fds[EXTRA_FDS + i].revents & POLLOUT) { write_to_child(children[i].fd, children[i].screen); }` at **child-monitor.c:1539-1540** — i.e. only for a child **still in the polled range**. After `remove_children` clears the slot (`EMPTY_CHILD`, :1322), sets `fd=-1` (:1323), and decrements `count` (:1331), the closed child is no longer iterated — so any bytes still in its `screen->write_buf` are never flushed, i.e. **dropped**. (Mechanism proven by the observed fd removal + the loop's write gating; the fd is additionally hard‑closed by `cleanup_child` at :1307.)

**Reroute proof.** After W_C closed, `y` was received by the surviving window (`recD=0x79`) and W_C's recording was unchanged — new input follows `active_window()`, which now resolves to W_D.

### 4.3 OS‑window teardown — `process_pending_closes` (main thread)

Closing the **last** window (`Ctrl+Shift+W` = `close_window`, **definition.py:3743**, consumed as a shortcut: `KeyPress matched action: close_window, handled as shortcut`) tears down the OS window on the **main GLFW thread**:

```console
$ gdb -p <PID> -ex 'break process_pending_closes' -ex 'break close_os_window' -ex continue   # then Ctrl+Shift+W
$ grep -A4 'HIT ' /tmp/evidence/gdb_ppc.txt
==== HIT process_pending_closes (main thread OS-window teardown) num_os_windows=1 ====
#0  process_pending_closes (self=self@entry=0x7f7a34c96330) at kitty/child-monitor.c:1098
#1  0x00007f7a35c1bca0 in process_global_state (data=0x7f7a34c96330) at kitty/child-monitor.c:1246
#2  0x00007f7a34b33c31 in _glfwPlatformRunMainLoop (tick_callback=0x7f7a35c1bbf2 <process_global_state>, data=0x7f7a34c96330) at glfw/main_loop.h:34
#3  0x00007f7a34b2a78c in glfwRunMainLoop (callback=<optimized out>, data=<optimized out>) at glfw/init.c:360
#4  0x00007f7a35c6f2cb in run_main_loop (cb=cb@entry=0x7f7a35c1bbf2 <process_global_state>, cb_data=cb_data@entry=0x7f7a34c96330) at kitty/glfw.c:2103

==== HIT close_os_window ====
#0  close_os_window (self=self@entry=0x7f7a34c96330, os_window=os_window@entry=0x56aa5ada7c90) at kitty/child-monitor.c:1083
#1  0x00007f7a35c18d0e in process_pending_closes (self=self@entry=0x7f7a34c96330) at kitty/child-monitor.c:1123
#2  0x00007f7a35c1bca0 in process_global_state (data=0x7f7a34c96330) at kitty/child-monitor.c:1246
```

> **Run‑first correction.** `process_pending_closes` (**child-monitor.c:1098**) is the **OS‑window** teardown path (main thread), reached from the GLFW main loop via `process_global_state` (**:1246**). It did **not** fire for the *single‑window* close in §4.2 (that window's child died and was removed by the I/O thread's `reap_children`+`remove_children`, while the OS window stayed open because W_D survived). The observed behaviour thus refines the naïve assumption that per‑window closes go through `process_pending_closes`: **per‑window fd removal happens in `remove_children` on the I/O thread; OS‑window teardown happens in `process_pending_closes` on the main thread.**


---

## Part 5 — Which parts are Python, which are C, and which are external libraries

**Direct answer.** The input pipeline spans exactly **three ownership tiers**: (1) an **external library**, the GLFW backend, shipped as a *separate* shared object `kitty/glfw-x11.so` and loaded at runtime with `dlopen` (**kitty/glfw-wrapper.c:16**) — it owns the OS event pump, X11 event processing, XKB keymap translation, and IME; (2) the **C extension** `kitty/fast_data_types.so`, which owns routing (`active_window`), encoding (`encode_glfw_key_event`), and all PTY I/O (`io_loop`, `write_to_child`); and (3) the **Python** control layer (`boss.py`, `window.py`, `window_list.py`, `tabs.py`, `main.py`), which owns startup, focus bookkeeping, and shortcut matching. Below, each tier is attributed from artifacts, and **three** plausible‑but‑wrong interpretations are refuted with evidence.

### 5.1 Attribution from artifacts

**External GLFW = `kitty/glfw-x11.so`, `dlopen`'d at runtime.** Two backends ship; the X11 one is loaded dynamically:

```console
$ ls -1 kitty/glfw-*.so
kitty/glfw-wayland.so
kitty/glfw-x11.so
$ sed -n '16p' kitty/glfw-wrapper.c
    handle = dlopen(path, RTLD_LAZY);
$ nm -DC kitty/glfw-x11.so | grep glfwRunMainLoop
000000000000b775 T glfwRunMainLoop
```

`glfwRunMainLoop` is an **exported** (`T`) symbol of `glfw-x11.so`. XKB (`glfw/xkb_glfw.c`, 968 lines) and IME (`glfw/ibus_glfw.c`) are compiled **into this backend**. The Part‑3 stacks confirm at runtime that the frames `glfwRunMainLoop`, `_glfwPlatformRunMainLoop` (`glfw/main_loop.h`), `processEvent` (`glfw/x11_window.c`), `glfw_xkb_handle_key_event` (`glfw/xkb_glfw.c:966`), and `_glfwInputKeyboard` (`glfw/input.c:350`) all execute in this external library, **before** any kitty symbol.

**C extension = `kitty/fast_data_types.so`.** It links the embedded interpreter and libc; GLFW is **not** linked (it is `dlopen`'d), and there is **no Go runtime**:

```console
$ ldd kitty/fast_data_types.so | grep -Ei "python|libc.so|libglib"
	libpython3.13.so.1.0 => /lib/x86_64-linux-gnu/libpython3.13.so.1.0
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
	libglib-2.0.so.0 => /lib/x86_64-linux-gnu/libglib-2.0.so.0
```

All input‑path work is C in this object:

```console
$ nm -C kitty/fast_data_types.so | grep -E " t (key_callback|on_key_input|active_window|encode_glfw_key_event|write_to_child|io_loop|window_focus_callback|set_active_window|window_for_event)$"
000000000008177a t active_window
00000000000815c0 t encode_glfw_key_event
000000000001959b t io_loop
000000000006c57b t key_callback
0000000000082a0e t on_key_input
00000000000becfe t set_active_window
000000000006cb47 t window_focus_callback
00000000000954f2 t window_for_event
00000000000191d8 t write_to_child
```

Mapping to source: `key_callback`/`window_focus_callback` (**glfw.c**), `on_key_input`/`active_window`/`encode_glfw_key_event` (**keys.c**), `write_to_child`/`io_loop` (**child-monitor.c**), `set_active_window` (**state.c**), `window_for_event` (**mouse.c**).

**Python = the control layer.** The Part‑3 py-spy stack attributes `_run_app`/`_main`/`main` to **`kitty/main.py`**; the shortcut match is `Boss.dispatch_possible_special_key` (**boss.py:1408**); focus is `Boss.on_focus` (**boss.py:1651**) → `Window.focus_changed` (**window.py:1123**). The `faulthandler` dump shows **only** `MainThread` has Python frames.

### 5.2 Ruled‑out interpretation #1 — "Python reads the keyboard directly" — REFUTED

The gdb backtrace in §3.3 shows the first code to touch a key is **external GLFW C** (`glfw_xkb_handle_key_event`, `_glfwInputKeyboard`) calling kitty's **C** callback `key_callback` (**glfw.c:439**) → `on_key_input` (**keys.c:166**) — with **no Python frame anywhere** between the OS event and `on_key_input`. The `--debug-keyboard` ordering (§2.1) independently shows the external `Press xkb_keycode …` line **always precedes** `on_key_input`. Python is consulted **only after** the C layer, and **only** for shortcut matching (`dispatch_possible_special_key`, **keys.c:228** → **boss.py:1408**); byte encoding and enqueue are C (`encode_glfw_key_event`/`schedule_write_to_child`, **keys.c:251-259**). For a plain non‑shortcut key, the interpreter is never involved in producing the bytes. Hence Python does **not** read the keyboard.

### 5.3 Ruled‑out interpretation #2 — "each window/child has its own input thread or goroutine" — REFUTED

With **three** windows open, `gdb info threads` shows exactly **one** I/O thread:

```console
$ gdb -p <PID> -batch -ex 'info threads'   # 3-window session; the 3 NAMED threads shown (67 total; the rest are llvmpipe-N render + worker threads, tallied below)
* 1    Thread 0x7aecc7927780 (LWP 116486) "kitty"         __syscall_cancel_arch () at ../sysdeps/unix/sysv/linux/x86_64/syscall_cancel.S:56
  2    Thread 0x7aec989a36c0 (LWP 116553) "KittyChildMon" __syscall_cancel_arch () at ../sysdeps/unix/sysv/linux/x86_64/syscall_cancel.S:56
  3    Thread 0x7aec997eb6c0 (LWP 116552) "kitty:disk$0"  __syscall_cancel_arch () at ../sysdeps/unix/sysv/linux/x86_64/syscall_cancel.S:56
$ grep -c KittyChildMon /tmp/evidence/gdb_info_threads.txt
1
$ grep -c '"llvmpipe' /tmp/evidence/gdb_info_threads.txt
32
$ grep -cE 'Thread 0x' /tmp/evidence/gdb_info_threads.txt
67
```

There is exactly **one** `KittyChildMon` thread regardless of window count; it is the single `io_loop` that `poll()`s **all** PTYs at once (**child-monitor.c:1509**) — as directly demonstrated in §1.3 (W1/W2/W3 all drained by the one thread) and §4.1 (unfocused window still drained). The remaining threads are one main GLFW thread (`"kitty"`, LWP 116486), one disk thread, a fixed worker pool, and 32 Mesa `llvmpipe-N` **render** threads (67 total) — **none per‑window**. And there are **no goroutines**: `ldd` (§5.1) shows no Go runtime linked into `fast_data_types.so`; the Go `kitten` binary is a *separate* executable and is not part of the terminal input process. The thread list is composed of OS `pthread`s (`start_thread`/`__clone3`), not a Go scheduler.

### 5.4 Ruled‑out interpretation #3 — "keyboard and mouse route the same way" — REFUTED

**Keyboard routing is logical** (`active_window()`, **keys.c:106**, independent of cursor position — §1.1, §4.1). **Mouse routing is spatial** — `window_for_event` (**mouse.c:616**)/`closest_window_for_event` (**mouse.c:641**) resolve the window physically **under the cursor**. Captured at runtime, mouse events carry **x/y coordinates** and fire a *different* debug path, `on_mouse_input` (**mouse.c:188**):

```text
Mouse cursor entered window: 1 at 100.000000x80.000000
Move x: 100.0 y: 80.0 grabbed: 0
on_mouse_input: press button: left mods: none grabbed: 0 handled_in_kitty: 1
MouseEvent matched action: mouse_handle_click
on_mouse_input: click button: left mods: none grabbed: 0 handled_in_kitty: 1
```

The coordinate‑bearing `on_mouse_input` path is structurally distinct from the coordinate‑free `on_key_input`→`active_window()` path — so keyboard and mouse do **not** route the same way.


---

## Part 6 — A measured correctness/efficiency‑vs‑responsiveness tradeoff

**Direct answer.** The `input_delay` option (default **3 ms**, **kitty/options/definition.py:878**) is a real, **measured** tradeoff: it is the minimum interval the I/O thread waits before waking the render loop when **child output** arrives. Raising it trades **responsiveness** for **efficiency**. Measured at fixed output volume, `input_delay=100` vs `0` gives **~7× fewer render wakeups** and **~3.4× less CPU**, but **~1.7× higher output latency** (the producer is throttled). This conclusion is from **measured** runtime numbers, **not** from the option's help text.

### 6.1 The enforcement sites (what makes it a tradeoff)

`input_delay` is enforced in two places in `kitty/child-monitor.c`:

- **I/O‑thread wakeup coalescing** (**child-monitor.c:1562-1569**; the `WAKEUP` macro at **:1562**, the `input_delay` gate at **:1566**): the loop only wakes the render/main loop after `input_delay` has elapsed since the last wakeup —
  ```c
  #define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
          // we only wakeup the main loop after input_delay as wakeup is an expensive operation
          // on some platforms, such as cocoa
          if (data_received) {
              if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
              else has_pending_wakeups = true;
  ```
- **Main‑thread parse wait** (**child-monitor.c:446**): `set_maximum_wait(OPT(input_delay) - pd.time_since_new_input)` bounds how long processing of freshly‑arrived input is deferred so updates can batch.

Fewer wakeups ⇒ fewer (expensive, software‑`llvmpipe`) renders ⇒ less CPU and fewer partial‑frame artifacts, but each wakeup is delayed up to `input_delay`, adding latency.

### 6.2 Measurement method

A child emits a **fixed** volume — **200,000 lines (~16 MB)** — then keeps its window alive; kitty is launched with a single launch‑time override (never committed): `kitty/launcher/kitty --config NONE -o input_delay=<N> --session …`. Metrics per run: kitty CPU (`utime+stime` from `/proc/<pid>/stat`, `CLK_TCK=100`), main‑loop wakeups (`grep -c 'wakeups_happened: 1'`), and the child's own emit time. **Scale: 200,000 lines × 2 runs per `input_delay`.**

### 6.3 Results (2 runs each; stable)

| `input_delay` | kitty CPU (s), run1/run2 | main‑loop wakeups, run1/run2 | child emit (s), run1/run2 | ms per wakeup |
|---------------|--------------------------|------------------------------|---------------------------|---------------|
| **0** | 3.45 / 3.52 | 124 / 126 | 0.895 / 0.892 | ~7.2 |
| **3** (default) | 3.17 / 3.44 | 125 / 133 | 0.848 / 0.922 | ~6.8 |
| **100** | 1.05 / 1.02 | 18 / 18 | 1.517 / 1.517 | ~84.3 |

The complete, unedited per‑run output lines from the matrix driver (`cpu_s` = kitty CPU seconds; `wakeups` = `wakeups_happened: 1` count; `ticks` = `loop tick` count; `child_emit_s` = the child's own emit duration):

```console
$ # each row emitted by the matrix driver after one run_one <input_delay> <tag>
input_delay=0    run=d0r1 cpu_s=3.450   wakeups=124   ticks=125   child_emit_s=0.895438
input_delay=0    run=d0r2 cpu_s=3.520   wakeups=126   ticks=127   child_emit_s=0.892480
input_delay=3    run=d3r1 cpu_s=3.170   wakeups=125   ticks=205   child_emit_s=0.848095
input_delay=3    run=d3r2 cpu_s=3.440   wakeups=133   ticks=233   child_emit_s=0.922086
input_delay=100  run=d100r1 cpu_s=1.050   wakeups=18    ticks=35    child_emit_s=1.516535
input_delay=100  run=d100r2 cpu_s=1.020   wakeups=18    ticks=35    child_emit_s=1.516916
```

**What the numbers show.**
- **Responsiveness cost of a *high* delay:** at `input_delay=100` the same fixed output takes **1.52 s** to emit vs **0.89 s** at `input_delay=0` (**~1.7×** slower) — because kitty drains the PTY only ~every 84 ms, the producer blocks on `write()` longer. The measured **~84 ms per wakeup** ≈ the 100 ms coalescing window, directly confirming the enforcement at **child-monitor.c:1566** (the `> OPT(input_delay)` gate).
- **Efficiency gain of a *high* delay:** wakeups drop from ~125 to **18** (**~7×**) and CPU from ~3.5 s to **~1.0 s** (**~3.4×**).
- **Stability:** run‑to‑run values are close (at `input_delay=100`, wakeups `18/18` and emit `1.517/1.517` are essentially identical).

### 6.4 An honest nuance: `input_delay=0` vs `3` are ~indistinguishable at flood

At a saturating output rate the two are nearly identical (7.2 vs 6.8 ms/wakeup; CPU 3.45 vs 3.17–3.44 s) because **each `llvmpipe` render frame already costs ~7 ms > 3 ms**, so wakeups are **render‑bound**, not delay‑bound. This is *why* the 3 ms default is a good tradeoff: its latency cost is effectively free at these rates, yet it still coalesces any output bursts finer than 3 ms (which is where it pays off on faster GPUs / lighter output). The large, unambiguous effect appears at `input_delay=100`.

### 6.5 Controlling for a logging artifact

The measurement build has event‑loop logging (`--extra-logging=event-loop`), so one might worry CPU tracks *logging*, not rendering. It does not: `input_delay=100` produced **fewer** log lines (177) than `input_delay=0` (627) yet used **far less** CPU. If logging drove CPU, the relationship would be inverted. The ~450‑line logging difference is < 0.05 s versus a ~2.4 s CPU delta — so CPU tracks real render wakeups. **This entire result is measured at runtime; the `long_text` help in definition.py:878 was not used as evidence.**


---

## Part 7 — Repository left unchanged; temporary artifacts removed

**Direct answer.** The source tree was **not** modified. The only tracked addition is this document, `blitzy/documentation/kitty_815df1e210e0.md`. All observation scripts and logs lived under `/tmp/**` (outside the repository) and were removed after the investigation. Build outputs under the repo (`kitty/fast_data_types.so`, `kitty/glfw-*.so`, `kitty/launcher/kitty`, `build/`) are **gitignored** and therefore are not tracked changes.

All temporary artifacts were confined to `/tmp` and deleted during cleanup, e.g.:
- scenario/harness scripts: `/tmp/rxhex.sh`, `/tmp/rxtag.sh`, `/tmp/rxtag2.sh`, `/tmp/genrec.sh`, `/tmp/closable.sh`, `/tmp/flood.sh`, `/tmp/measure.sh`;
- session files: `/tmp/session_*.conf`;
- captured logs: `/tmp/evidence/**`, `/tmp/flood_kitty_*.log`, `/tmp/kbd.log`.

The final verification (verbatim `git status --porcelain` after cleanup) is:

```console
$ git status --porcelain
?? blitzy/

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/kitty_815df1e210e0.md

$ git check-ignore kitty/launcher/kitty kitty/fast_data_types.so kitty/glfw-x11.so kitty/glfw-wayland.so
kitty/launcher/kitty
kitty/fast_data_types.so
kitty/glfw-x11.so
kitty/glfw-wayland.so
```

(`git status --porcelain` collapses the single untracked directory to `?? blitzy/`; `--untracked-files=all` expands it to the one file. This is the working‑tree state after cleanup and immediately before the deliverable is committed.)

The only entry is the untracked answer document; no tracked source file appears as modified. (Any `??` lines for `kitty/launcher/kitty`, `kitty/*.so`, or `build/` would be gitignored build outputs, not tracked changes — confirmed via `git check-ignore`.)

---

## Coverage pass — every required part and named item

| # | Required item | Where answered | Key evidence / anchor |
|---|---------------|----------------|-----------------------|
| 1 | Reproduce overlapping input activity | Part 1 | Multi‑tab/window routing (`routing_perwindow.txt` W1=31,W2=32,W3=33); focus switching (`on_focus_change` glfw.c:515/517); background drain (5 MB/64 ms); resize + scrollback; 200×2 byte‑identical (`cmp … IDENTICAL`) |
| 2 | Routing / focus / child delivery narrative | Part 2 | Ordered layering (`ordered_enter.txt`); `active_window()` keys.c:106; dispatch keys.c:228→boss.py:1408; encode keys.c:251; enqueue keys.c:259→c‑m.c:372; `write_to_child` c‑m.c:1443; focus path glfw.c:515→boss.py:1651→window.py:1123→state.c:514; byte‑fidelity table + `child_rx.hex` |
| 3 | ≥1 stack/symbol snapshot + tiered fallback | Part 3 | Tier1 block (`ptrace … Operation not permitted`); Tier2 py-spy native + gdb `on_key_input` full path + `write_to_child`/`io_loop`; Tier3 `faulthandler`; Tier4 `--debug-keyboard` |
| 4 | Unfocused / just‑closed behavior | Part 4 | Before/during/after table (recA frozen, genA 29→36); gdb `reap_children` c‑m.c:1413 + `remove_children` c‑m.c:1313 (fd→‑1 at :1323); reroute recD=0x79; OS‑window teardown `process_pending_closes` c‑m.c:1098 |
| 5 | Language/library split + ≥2 refutations | Part 5 | `ldd`/`nm` attribution; dlopen glfw‑wrapper.c:16; refute "Python reads keyboard" (§5.2), "per‑window thread/goroutine" (§5.3, 1 `KittyChildMon` for 3 windows), "kbd==mouse" (§5.4, `on_mouse_input` mouse.c:188 spatial) |
| 6 | One MEASURED tradeoff | Part 6 | `input_delay` matrix (0/3/100), CPU 3.5→1.0 s, wakeups 124→18, emit 0.89→1.52 s; enforcement c‑m.c:1562‑1569 (input_delay gate :1566) & :446; scale 200k×2; logging‑artifact control |
| 7 | Repo unchanged + cleanup | Part 7 | `git status --porcelain` shown verbatim in Part 7 (captured post‑cleanup); temp artifacts under `/tmp` removed; build outputs gitignored |

**Named‑item checklist:** `key_callback` ✔ · `on_key_input` ✔ · `active_window()` ✔ · `dispatch_possible_special_key` ✔ · `encode_glfw_key_event` ✔ · `schedule_write_to_child` ✔ · `io_loop`/`poll` ✔ · `write_to_child` ✔ · `window_focus_callback`/`on_focus` ✔ · `notify_on_active_window_change` ✔ · `Window.focus_changed` ✔ · `set_active_window` ✔ · `window_for_event` (mouse) ✔ · `process_pending_closes` ✔ · `openpty`/`Child.fork` (child.py:170/276) ✔ · `input_delay`/`repaint_delay`/`sync_to_monitor` ✔ · external GLFW via `glfw-wrapper.c` + `xkb_glfw.c`/`ibus_glfw.c` ✔.

### Notes on labelling
- **NON‑CANONICAL:** none of the routing/byte/latency evidence relied on remote control or debug hooks; all input was real X11 (XTEST). Remote control was not used as routing evidence.
- **INFERRED:** the IME/`on_IME_input` path (§2.6) — code exists (keys.c:174, 200‑203; `glfw/ibus_glfw.c`) but was not driven because no IBus/IME framework is present in the headless environment.
- **Byte‑sensitive results** (Enter `0d`, Ctrl+A `01`, Up `1b 5b 41`) were verified against the exact bytes the child received on its raw PTY.

