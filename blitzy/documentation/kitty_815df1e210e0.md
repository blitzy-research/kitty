# Kitty Input-Event Flow & Focus Management — A Runtime-Evidenced Investigation

**Repository:** `kitty` (kovidgoyal/kitty)
**Branch:** `kitty_815df1e210e0`  **HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config")
**Question answered:** How does kitty actually handle input-event flow and focus management across OS-windows, tabs, and child processes at runtime?

This document answers seven sub-questions (Q1–Q7) **from what was observed while the real code ran**, not from reading the source alone. Every behavioral claim is paired with (a) the exact command, (b) the raw, unedited captured output, and (c) a `file:line` citation into the source that was built and observed. Claims that could not be directly observed are explicitly labelled **inferred**.

---

## 0. Executive summary (direct answers, one line each)

- **Q1 — Which window gets input?** A **two-level focus model**: the OS-window with `OSWindow.is_focused == true`, then, inside that window's active tab, the window at index `tab->active_window`. The per-keystroke resolver `active_window()` computes `callback_os_window->tabs[active_tab].windows[active_window]` (`kitty/keys.c:106`).
- **Q2 — How does focus change propagate?** The external windowing layer fires the C callback `window_focus_callback` (`kitty/glfw.c:515`), which sets `is_focused`, stamps an MRU counter `last_focused_counter = ++focus_counter` (`kitty/glfw.c:531`), then calls Python `Boss.on_focus` (`kitty/boss.py:1651`) → `Window.focus_changed` → `Screen.focus_changed`.
- **Q3 — How is input routed to the child?** OS → external GLFW backend (+libxkbcommon/IBus) → C `key_callback` (`kitty/glfw.c:430`) → C `on_key_input` (`kitty/keys.c:166`) → Python shortcut test `dispatch_possible_special_key` (`kitty/boss.py:1408`); if not consumed, C `encode_glfw_key_event` (`kitty/key_encoding.c:414`) → **id-keyed** `schedule_write_to_child(w->id, …)` (`kitty/child-monitor.c:372`) → drained to the PTY by the `io_loop` thread.
- **Q4 — Stack snapshot.** `py-spy dump --native` captured the MainThread across all three layers; `gdb`/`eu-stack` captured the `KittyChildMon` `io_loop` thread. An attach was authentically **blocked** without `CAP_SYS_PTRACE` and then **remediated**. Stable across 2 runs.
- **Q5 — Input to an unfocused / just-closed window?** Input strictly **follows focus**; an unfocused window receives nothing. Input generated right after closing the active window is **re-routed to the new active window**; the closed window's child receives nothing (the id is simply never matched). Silent, no crash.
- **Q6 — Layer attribution.** External libs receive/translate the OS event; **C** encodes and writes to the PTY; **Python** only arbitrates shortcuts and high-level bookkeeping. Three incorrect interpretations are refuted with snapshot evidence.
- **Q7 — Correctness-vs-responsiveness tradeoff.** Input is handled synchronously on the **main/UI thread** while a **separate `io_loop` thread** drains child writes; a ~185k–198k lines/sec background flood on an unfocused window produced **zero** change in focused-window keystroke latency (median 89–90 ms, both runs) and **zero** cross-child leakage.

---

## 1. Methodology and grounding rules

- **Run-first.** kitty was **built from this checkout** and launched through its **canonical entry point** `kitty/launcher/kitty`. No pre-installed binary, no remote-control injection, and no debug hook was used to *originate* the keystrokes under study. `--debug-keyboard` was used only to *observe* input (it emits first-party trace lines; it does not synthesize events).
- **Canonical input delivery.** Because kitty is a GPU/GLFW application with no physical keyboard in a headless container, real key/focus/resize events were delivered through the **X11 XTEST extension** (`XTestFakeKeyEvent`). XTEST injects events at the **X server**, which delivers them to the focused window exactly as a physical keyboard or `xdotool` would (xdotool itself uses XTEST). These events flow through the patched-GLFW X11 backend into kitty's C callbacks — i.e. **the real path under study**, not a synthetic bypass. (The container lacks `xdotool`/`python-Xlib`; a ~150-line C XTEST injector, `xinj`, was compiled from the present `Xlib.h` + `libXtst.so.6` and used purely as the "keyboard".)
- **Observed-output discipline.** Every claim below shows its captured output next to it. Nothing is paraphrased before the relevant result appears. Anything not directly observed is labelled **inferred**.
- **Reproducibility.** Timing/magnitude claims (Q7) and the stack inventory (Q4) are shown **stable across ≥2 runs**.
- **Repository untouched.** All scripts/artifacts lived outside the repo (container path `/kqna`, host `/tmp/kqna_work`) and were deleted afterward; the section *"Repository left byte-for-byte unchanged"* below proves the tree is byte-for-byte unchanged apart from this one document.

### 1.1 Environment (build/run container)

The build/run container is the user-mandated toolchain image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (base `ghcr.io/scaleapi/swe-atlas`). Observed facts:

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ python3 --version   → Python 3.12.3
$ go version          → go1.23.4
$ gcc --version       → gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ pkg-config --modversion harfbuzz   → 8.3.0
$ pkg-config --modversion xkbcommon  → 1.6.0
$ DISPLAY=:99 glxinfo | grep -i 'OpenGL renderer'
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)     # software GL (Mesa 25.2.8)
```

Manifest facts (read, not modified): `pyproject.toml:2` `requires-python = ">=3.8"`; `go.mod:3` `go 1.22`; `setup.py:609` `at_least_version('harfbuzz', 1, 5)`.

### 1.2 Build & launch (exact canonical commands)

kitty is **not** pre-built in the checkout (`.gitignore` excludes `/kitty/launcher/kitt*` and `*.so`), so building is mandatory and satisfies run-first:

```bash
# Canonical build (default configuration) — compiles the C core into the
# kitty/fast_data_types extension and links the launcher kitty/launcher/kitty
python3 setup.py
```

Observed tail of the build:

```
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done

real	0m15.893s
```

Headless launch through the canonical entry point, under Xvfb + software GL, with the first-party trace switch:

```bash
export DISPLAY=:99
Xvfb :99 -screen 0 1280x800x24 -nolisten tcp &        # headless X server
# LIBGL_ALWAYS_SOFTWARE=1, GALLIUM_DRIVER=llvmpipe are baked into the image
./kitty/launcher/kitty --config NONE -o shell=<child> --debug-keyboard [trailing-program]
```

Notes on the launch flags used throughout:
- `--config NONE` → the **default canonical configuration** (no user config), so reported values reflect what a normal user of this revision sees.
- `-o shell=/…/labelsh.sh` → makes **every** kitty window run a tiny logger child that appends its PTY stdin to `win_${KITTY_WINDOW_ID}.txt`, proving at the child level exactly which window's child received which bytes.
- `--debug-keyboard` (alias `--debug-input`) → CLI option at `kitty/cli.py:996` (`dest=debug_keyboard` at `:997`), consumed at `kitty/main.py:514`; it gates the `debug_input(...)` trace macro (`kitty/state.h:15`, `#define debug debug_input` at `kitty/keys.h:16`).

### 1.3 The pipeline at a glance (three ownership layers)

```
   OS key/focus/resize event
        │
   ┌────▼─────────────────────────────────────────────┐
   │ EXTERNAL WINDOWING LIBRARY (patched GLFW)         │  glfw/x11_window.c, glfw/wl_window.c
   │  • keysym/text via libxkbcommon (glfw/xkb_glfw.c) │  glfw/ibus_glfw.c (IME)
   └────┬──────────────────────────────────────────────┘
        │ registered callbacks
   ┌────▼──────────────────────────────────────────────┐
   │ C CORE  (kitty/fast_data_types.so)                 │
   │  key_callback            kitty/glfw.c:430           │
   │  on_key_input            kitty/keys.c:166           │
   │   ├─ active_window()     kitty/keys.c:106  (target) │
   │   ├─ (ask Python: shortcut?)  ──────────────┐       │
   │   ├─ encode_glfw_key_event  kitty/key_encoding.c:414│
   │   └─ schedule_write_to_child(w->id,…)  child-monitor.c:372  (id-keyed)
   │  io_loop (thread KittyChildMon)  child-monitor.c:1481 → drains write_buf to PTY
   │  window_focus_callback   kitty/glfw.c:515  (focus state + MRU)
   └────┬──────────────────────────────────────────────┘
        │ PyObject_CallMethod(global_state.boss, …)
   ┌────▼──────────────────────────────────────────────┐
   │ PYTHON COORDINATION  (kitty/*.py)                  │
   │  Boss.dispatch_possible_special_key  boss.py:1408   │  → keys.py:154 → get_shortcut keys.py:40
   │  Boss.on_focus                       boss.py:1651   │  → Window.focus_changed window.py:1123
   │  Window.write_to_child               window.py:955  │  (paste / remote-control / kitten text path)
   └─────────────────────────────────────────────────────┘
```

The decisive structural fact proved below (Q3): **both** the fast C keyboard path and the high-level Python text path converge on the single id-keyed function `schedule_write_to_child(id, …)`, and the id is the focused window's id.

---

## Q1 — How does kitty decide which window receives input?

**Direct answer.** kitty uses a **two-level focus model**:

1. **Level 1 — OS window.** Each `OSWindow` carries a boolean `is_focused`. The helper `current_focused_os_window_id()` walks the OS windows and returns the id of the one with `is_focused == true` (`kitty/state.c:120`). A separate most-recently-used helper `last_focused_os_window_id()` (`kitty/state.c:108`) uses the MRU counter (see Q2).
2. **Level 2 — window within the active tab.** Inside the focused OS window's **active tab**, the tab stores an `active_window` **index**; the effective target id is `tab->windows[tab->active_window].id` (`kitty/state.c:355`), set by `set_active_window()` (`kitty/state.c:514`).

At **every keystroke**, the C resolver `active_window()` computes the target from the callback OS window:

```c
// kitty/keys.c:106
active_window(void) {
    Tab *t = global_state.callback_os_window->tabs + global_state.callback_os_window->active_tab;
    Window *w = t->windows + t->active_window;
    if (w->render_data.screen) return w;   // only if it has a live screen
    return NULL;
}
```

So the chain is: **focused OS window → its active tab → that tab's `active_window` index → that window's id → that window's child**. The Python mirror is `Boss.active_window` (`kitty/boss.py:1384`), which returns `self.active_tab.active_window`.

**Runtime evidence (scenario S1).** One OS window; two tabs; splits inside tab 1. Every window's child is the labeller, so its file records exactly what that child received. Ordinary text was typed after each navigation, then the tab/window was switched with the standard shortcuts.

Command (representative — driven through the XTEST "keyboard"; navigation shortcuts are the canonical kitty maps):

```bash
# new_window = kitty_mod+enter (definition.py:3696); previous_window = kitty_mod+[ (definition.py:3755)
# new_tab    = kitty_mod+t     (definition.py:3896); previous_tab   = kitty_mod+left (definition.py:3885)
xinj <<'CMDS'
type activea            # into window 1
key ctrl+shift+Return   # new_window -> window 2 becomes active
type activeb
key ctrl+shift+Return   # new_window -> window 3 becomes active
type activec
key ctrl+shift+[        # previous_window (x2, walking back)
...
key ctrl+shift+t        # new_tab -> window 4 in tab 2
type intabtwo
key ctrl+shift+left     # previous_tab -> back to tab 1's active window
type backtabone
CMDS
```

Raw per-child output (verbatim `cat` of each child's log; each file's first line is the child-start banner it prints on launch):

```
$ cat win_1.txt
[child start pid=6459 KITTY_WINDOW_ID=1]
activea
backtabone
$ cat win_2.txt
[child start pid=6468 KITTY_WINDOW_ID=2]
activeb
$ cat win_3.txt
[child start pid=6470 KITTY_WINDOW_ID=3]
activec
$ cat win_4.txt
[child start pid=6481 KITTY_WINDOW_ID=4]
intabtwo
```

Raw trace showing each navigation was consumed as a shortcut (not sent to any child):

```
KeyPress matched action: new_window, handled as shortcut
KeyPress matched action: new_window, handled as shortcut
KeyPress matched action: previous_window, handled as shortcut
KeyPress matched action: previous_window, handled as shortcut
KeyPress matched action: new_tab, handled as shortcut
KeyPress matched action: previous_tab, handled as shortcut
```

**Cause → effect, confirmed:** `activea/b/c` each landed in the child that was active at that moment (win_1/2/3); after `new_tab`, `intabtwo` went to the tab-2 window (win_4); after `previous_tab`, `backtabone` returned to tab 1's active window (win_1). Text always follows the `active_window()` resolution. Note also that creating tabs/splits did **not** create new OS X windows (the X window count stayed at 1) — **tabs and splits live inside one OS window**, which is why Level 2 (the in-tab `active_window` index) is what disambiguates them.

---

## Q2 — How do focus changes propagate internally?

**Direct answer.** On any focus change the external windowing layer invokes the registered C callback `window_focus_callback` (`kitty/glfw.c:515`; registered by `glfwSetWindowFocusCallback(…, window_focus_callback)` at `kitty/glfw.c:1281`). That callback, in order:

1. emits the trace line `on_focus_change: window id: … focused: …` (`kitty/glfw.c:517`),
2. sets `global_state.callback_os_window->is_focused = focused` (`kitty/glfw.c:527`),
3. on focus-**gain**, stamps a monotonic MRU counter: `last_focused_counter = ++focus_counter` (`kitty/glfw.c:531`; `static id_type focus_counter = 0;` at `:512`) — this is what `last_focused_os_window_id()` later reads (`kitty/state.c:108`),
4. calls into Python `WINDOW_CALLBACK(on_focus, "O", …)` (`kitty/glfw.c:538`) → `Boss.on_focus(os_window_id, focused)` (`kitty/boss.py:1651`), which locates the tab manager, takes `tm.active_window`, and calls `w.focus_changed(focused)` (`kitty/window.py:1123`) → `Screen.focus_changed` (`kitty/screen.c:4604`),
5. updates IME via `glfwUpdateIMEState(…)` (`kitty/glfw.c:540`).

**Runtime evidence (scenario S2) — the C callback fires as paired transitions.** Two OS windows (A id `0x1`, B id `0x2`); focus was switched rapidly. `on_focus_change` fired as **focus-out on the losing window and focus-in on the gaining window at the same timestamp**, keyed by OS-window id:

```
[0.167] on_focus_change: window id: 0x1 focused: 1     # A focused at startup
[0.915] on_focus_change: window id: 0x1 focused: 0     # (2nd OS window created) A loses…
[0.915] on_focus_change: window id: 0x2 focused: 1     # …B gains
[1.610] on_focus_change: window id: 0x2 focused: 0     # rapid switch: B loses…
[1.611] on_focus_change: window id: 0x1 focused: 1     # …A gains
[1.761] on_focus_change: window id: 0x1 focused: 0
[1.761] on_focus_change: window id: 0x2 focused: 1
[1.913] on_focus_change: window id: 0x2 focused: 0
[1.913] on_focus_change: window id: 0x1 focused: 1
```

Input routing followed OS focus exactly — bytes went to whichever OS window was focused at the time (verbatim `cat`):

```
$ cat win_1.txt
[child start pid=6601 KITTY_WINDOW_ID=1]
start1
inosw1
$ cat win_2.txt
[child start pid=6610 KITTY_WINDOW_ID=2]
inosw2
```

**Runtime evidence (scenario S2b) — full OS→C→Python→child propagation proven end-to-end.** Each window's child enabled DECSET 1004 (focus reporting), so the terminal writes `ESC [ I` on focus-in and `ESC [ O` on focus-out **to the child** — and those bytes are written *only* by `Screen.focus_changed` (`kitty/screen.c:4611`: `if (self->modes.mFOCUS_TRACKING) write_escape_code_to_child(self, ESC_CSI, has_focus ? "I" : "O")`). The children actually received them (`od -c`):

```
$ od -c focrep_1.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   1   ]  \n 033   [   O 033   [   I
0000060 033   [   O 033   [   I 033   [   O
0000071
$ od -c focrep_2.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   2   ]  \n 033   [   O 033   [   I
0000060 033   [   O 033   [   I
0000066
```

Because those `ESC[I`/`ESC[O` bytes can only originate from `Screen.focus_changed`, and that is reached solely via `window_focus_callback` → `Boss.on_focus` → `Window.focus_changed` → `Screen.focus_changed`, their appearance in the child proves the **entire propagation chain across all three layers** ran at runtime — the C callback stamped state and called Python, Python resolved the active window and called back into the per-window C screen, and the screen emitted the report to the child.

---

## Q3 — How is an input event routed to the correct child process?

**Direct answer, in the order components see the event:**

1. **First seen by the external windowing layer.** The patched-GLFW X11 backend (`glfw/x11_window.c`) receives the raw X key event; keysym/text translation is delegated to **libxkbcommon** (`glfw/xkb_glfw.c` — `xkb_state_key_get_utf8`), and IME composition to **IBus** (`glfw/ibus_glfw.c`).
2. **GLFW invokes the registered C callback `key_callback`** (`kitty/glfw.c:430`; registered by `glfwSetKeyboardCallback(…, key_callback)` at `:1292`). It calls `on_key_input(ev)` **only if** `is_window_ready_for_callbacks() && !ev->fake_event_on_focus_change` (`kitty/glfw.c:439`).
3. **`on_key_input`** (`kitty/keys.c:166`) resolves the target with `active_window()` (Q1), and if `!w` bails with `no active window, ignoring` (`kitty/keys.c:182`).
4. **Intermediate processing #1 — Python shortcut test.** Via the C→Python macro `dispatch_key_event` (`kitty/keys.c:218`) it calls `PyObject_CallMethod(global_state.boss, "dispatch_possible_special_key", "O", ke)` (`kitty/keys.c:221`, invoked at `:228`) → `Boss.dispatch_possible_special_key` (`kitty/boss.py:1408`) → `Mappings.dispatch_possible_special_key` (`kitty/keys.py:154`) → `get_shortcut` (`kitty/keys.py:40`) / `matching_key_actions` (`kitty/keys.py:117`). If the key is a configured shortcut it is **consumed**, the trace logs `handled as shortcut` (`kitty/keys.c:231`), and **no bytes reach the child**.
5. **Intermediate processing #2 — C encoding.** If not consumed, C encodes the key with `encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)` (`kitty/keys.c:251`; defined `kitty/key_encoding.c:414`) — legacy escape sequences or the Kitty Keyboard Protocol.
6. **Intermediate processing #3 — optional signal conversion.** For a single-byte control key **when the terminal has `mHANDLE_TERMIOS_SIGNALS` enabled**, `screen_send_signal_for_key(screen, *encoded_key)` (`kitty/keys.c:257`; defined `kitty/screen.c:2404`) converts e.g. Ctrl-C → SIGINT **instead of** writing bytes.
7. **Final destination chosen by child id.** The write is `schedule_write_to_child(w->id, …)` — text branch `kitty/keys.c:253`, encoded branch `kitty/keys.c:259`. The variadic entry (`kitty/child-monitor.c:372`) expands the macro `schedule_write_to_child_generic` (`kitty/child-monitor.c:323`), which takes `children_lock`, **loops `if (children[i].id == id)`** (`kitty/child-monitor.c:336`) — this integer-id match **is** the routing — appends to that child's `screen->write_buf`, wakes the io_loop, and `return found` (`kitty/child-monitor.c:369`).
8. **Drained to the PTY by a dedicated thread.** The `io_loop` thread (`kitty/child-monitor.c:1481`, named `KittyChildMon` at `:1489`) later flushes each child's `write_buf` to its PTY fd via `write_to_child(fd, screen)` (`kitty/child-monitor.c:1443`, called on POLLOUT at `:1540`).

**Convergence proof (named explicitly).** The higher-level Python text path `Window.write_to_child` (`kitty/window.py:955`) calls `child_monitor.needs_write(self.id, data)` → C `needs_write` (`kitty/child-monitor.c:412`) → the **same** `schedule_write_to_child` (`kitty/child-monitor.c:417`). So the C fast keyboard path **and** the Python text path (paste / remote-control / kitten) converge on one id-keyed function.

**Runtime evidence — baseline single window.** With `--debug-keyboard`, typing exercised each branch. Text keys (`SEND_TEXT_TO_CHILD`, `kitty/keys.c:253-254`):

```
[0.788] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none  text: 'a' state: 0 sent key as text to child: a
[0.819] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
```

(The `A` is `Shift`+`a`: same physical key `0x61`, `mods: shift`, producing text `'A'`.)

Non-text keys took the **encoded** branch (`kitty/keys.c:259-261`) with a byte-by-byte dump (`kitty/keys.c:263-268`). Verbatim PRESS lines for the arrow keys and Enter (note the RELEASE events log `ignoring as keyboard mode does not support encoding this event` — only PRESS is encoded in legacy mode):

```
[0.856] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd      # Enter
[1.175] on_key_input: glfw key: 0xe006 native_code: 0xff51 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ D    # Left
[1.311] on_key_input: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A    # Up
[1.447] on_key_input: glfw key: 0xe007 native_code: 0xff53 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ C    # Right
[1.584] on_key_input: glfw key: 0xe009 native_code: 0xff54 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ B    # Down
```

The child received exactly the encoded bytes (`win_1.txt` contained the literal `^[[D^[[A^[[C^[[B`). A **shortcut** (`Ctrl+Shift+T`) was consumed with no child write:

```
[1.855] on_key_input: glfw key: 0x74 native_code: 0x74 action: PRESS mods: ctrl+shift text: '' state: 0
KeyPress matched action: new_tab, handled as shortcut
```

The external translation layer is visible too (libxkbcommon, before `on_key_input`):

```
[0.788] Press xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
```

**Ctrl+C — the signal branch, observed with a caveat.** With the default configuration, `Ctrl+C` was observed to take the **encoded byte** path, not `screen_send_signal_for_key`:

```
[2.378] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3
```

i.e. byte `0x03` was written. The gate at `kitty/keys.c:256` (`if (size == 1 && screen->modes.mHANDLE_TERMIOS_SIGNALS)`) was **false** by default, so kitty wrote `0x03` and the **kernel PTY line discipline** (ISIG) raised SIGINT — the running `cat` died and its tab closed. The alternate path where kitty itself sends the signal (`screen_send_signal_for_key` → `Window.send_signal_for_key` `kitty/window.py:1116` → `Child.send_signal_for_key` `kitty/child.py:481` → `os.killpg(pgrp, signal.SIGINT)` `kitty/child.py:499`) is **conditional on `mHANDLE_TERMIOS_SIGNALS`**; it was *not* exercised in the default config (labelled **inferred** for that mode, cited from source).

**Runtime evidence (scenario S3) — routing holds under concurrent resize + scroll.** Typing continuously while the OS window was resized and scrolled: all 30 injected letters reached the focused child **interleaved** with `SIGWINCH` events (each resize drives `resize_pty`, `kitty/child-monitor.c:592`, → `TIOCSWINSZ` → child `SIGWINCH`):

```
[winch child KITTY_WINDOW_ID=1 initial_size=22 71]
LINE:typea
SIGWINCH new_size=18 62
LINE:typeb
SIGWINCH new_size=21 68
LINE:typec
SIGWINCH new_size=23 75
LINE:typed
SIGWINCH new_size=25 82
LINE:typee
SIGWINCH new_size=27 88
LINE:typef
SIGWINCH new_size=30 95
SIGWINCH new_size=22 71
```

Keystroke delivery and window-size changes proceed concurrently without dropping input — the keystrokes are encoded+queued on the main thread while resizes fire, and the io_loop drains them in order.

---

## Q4 — A stack/symbol snapshot of input handling (with commands, raw output, and a blocked-then-remediated attach)

kitty is a single process hosting a CPython interpreter with the native `fast_data_types` extension. The PID was resolved with `pgrep -f launcher/kitty`. During capture, one window ran a continuous output flood (keeping the `io_loop` thread busy) and another focused window was being typed into.

### Q4.1 — First attempt authentically BLOCKED (no `CAP_SYS_PTRACE`, host `ptrace_scope=1`)

Running the sampler in a container **without** the `SYS_PTRACE` capability (the Docker default drops it; `CapEff=00000000a80425fb` decodes to *no* `cap_sys_ptrace`), against a kitty that is **not** a descendant of the sampler, is blocked by the kernel:

```bash
$ py-spy dump --native --pid 48
```
```
Error: Failed to copy Py_Version symbol

Caused by:
    0: Permission denied (os error 13)
    1: Permission denied (os error 13)
```

The same block hits gdb and even a raw `/proc` stack read:

```bash
$ gdb -p 48 -batch -ex 'set debuginfod enabled off' -ex 'thread apply all bt'
ptrace: Inappropriate ioctl for device.
$ cat /proc/48/stack
cat: /proc/48/stack: Permission denied
```

**Remediation (documented and applied):** grant the capability at container start (`docker run --cap-add=SYS_PTRACE …`) — equivalently run as root with the capability, or relax the host knob `sysctl kernel.yama.ptrace_scope=0` (write `/proc/sys/kernel/yama/ptrace_scope`). All subsequent captures were taken in a container started **with** `--cap-add=SYS_PTRACE`.

### Q4.2 — Primary success: `py-spy dump --native` (Python + native-C frames)

```bash
$ py-spy dump --native --pid <kitty_pid>
```

Run 1 — MainThread caught idle in the GLFW event loop. The capture below is the **complete, unedited** py-spy dump (the hex values are the actual per-run pointer addresses):

```
Process 8543: /work/kitty/launcher/kitty --config NONE -o shell=/kqna/labelsh.sh --debug-keyboard /kqna/floodsh.sh
Python v3.12.3 (/work/kitty/launcher/kitty)

Thread 8543 (idle): "MainThread"
    poll (libc.so.6)
    0x7fd0872e38ca (libxcb.so.1.1.0)
    0x7fd0872e3ef0 (libxcb.so.1.1.0)
    xcb_wait_for_reply64 (libxcb.so.1.1.0)
    _XReply (libX11.so.6.4.0)
    _XGetWindowAttributes (libX11.so.6.4.0)
    XGetWindowAttributes (libX11.so.6.4.0)
    glfwGetWindowAttrib (kitty/glfw-x11.so)
    process_global_state (kitty/fast_data_types.so)
    dispatchTimers.part.0.constprop.0.isra.0 (kitty/glfw-x11.so)
    glfwRunMainLoop (kitty/glfw-x11.so)
    main_loop.lto_priv.0 (kitty/fast_data_types.so)
    _run_app (kitty/main.py:234)
    __call__ (kitty/main.py:252)
    _main (kitty/main.py:518)
    main (kitty/main.py:526)
    main (kitty/entry_points.py:195)
    <module> (__main__.py:7)
    _run_code (<frozen runpy>:88)
    _run_module_as_main (<frozen runpy>:198)
    0x7fd0890151ca (libc.so.6)
```

**Layer mapping (reading the stack bottom-to-top).** The deepest frames are the **external windowing library** — `libxcb`/`libX11` reached through `glfwGetWindowAttrib` in `kitty/glfw-x11.so` (the patched GLFW built as a separate shared object). The middle frames are the **C core** — `process_global_state` and `main_loop.lto_priv.0` in `kitty/fast_data_types.so`. The top frames are **Python** — and `_run_app (kitty/main.py:234)` is exactly the `boss.child_monitor.main_loop()` call site (`kitty/main.py:234`). So one stack literally traverses Python → C core → external library, top to bottom.

Run 2 caught the **same backbone** at a different instant — actively rendering on the main thread (complete, unedited):

```
Process 8929: /work/kitty/launcher/kitty --config NONE -o shell=/kqna/labelsh.sh --debug-keyboard /kqna/floodsh.sh
Python v3.12.3 (/work/kitty/launcher/kitty)

Thread 8929 (active+gil): "MainThread"
    0x7e039536cb3c (libc.so.6)
    0x7e039536d43a (libc.so.6)
    free (libc.so.6)
    0x7e0390c2a574 (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e0390c2a976 (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e0390bbad32 (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e0390bb3af2 (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e0390bb3ec0 (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e0390bb438d (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e0390cefc4d (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    0x7e03907a1ca8 (libgallium-25.2.8-0ubuntu0.24.04.2.so)
    draw_cells_simple.lto_priv.0 (kitty/fast_data_types.so)
    draw_cells (kitty/fast_data_types.so)
    process_global_state (kitty/fast_data_types.so)
    dispatchTimers.part.0.constprop.0.isra.0 (kitty/glfw-x11.so)
    glfwRunMainLoop (kitty/glfw-x11.so)
    main_loop.lto_priv.0 (kitty/fast_data_types.so)
    _run_app (kitty/main.py:234)
    __call__ (kitty/main.py:252)
    _main (kitty/main.py:518)
    main (kitty/main.py:526)
    main (kitty/entry_points.py:195)
    <module> (__main__.py:7)
    _run_code (<frozen runpy>:88)
    _run_module_as_main (<frozen runpy>:198)
    0x7e03952ec1ca (libc.so.6)
```

Here the same `main_loop.lto_priv.0 → glfwRunMainLoop → process_global_state` backbone is present, but the top is inside `draw_cells`/`draw_cells_simple.lto_priv.0` (C core) calling down into Mesa `libgallium` (software GL / llvmpipe) — i.e. the main thread was mid-render.

**Note (methodological).** py-spy reported **only** the MainThread. That is expected: kitty's I/O worker is a **pure C pthread** created by the extension (`pthread_create(&self->io_thread, NULL, io_loop, self)`, `kitty/child-monitor.c:291`) and holds no `PyThreadState`, so a Python-centric sampler does not enumerate it. gdb/eu-stack (below) are the complementary tools that reveal it.

### Q4.3 — Fallback A: `gdb thread apply all bt` (reveals the `io_loop` thread)

```bash
$ gdb -p <kitty_pid> -batch -ex 'set debuginfod enabled off' -ex 'thread apply all bt'
```

The full command prints all 67 threads; the two decisive ones (Run 1), quoted **verbatim** as an excerpt of that dump, are Thread 2 (the io worker) and Thread 1 (the main thread):

```
Thread 2 (Thread 0x7fcf53fff6c0 (LWP 8615) "KittyChildMon"):
#0  0x00007fd089106a9a in read () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007fd08841526e in io_loop () from /work/kitty/launcher/../../kitty/fast_data_types.so
#2  0x00007fd089087aa4 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007fd089114a34 in clone () from /lib/x86_64-linux-gnu/libc.so.6

Thread 1 (Thread 0x7fd088eb5740 (LWP 8543) "kitty"):
#0  0x00007fd089106a00 in ppoll () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007fd0874fbaf6 in glfwRunMainLoop () from /work/kitty/glfw-x11.so
#2  0x00007fd088413cfc in main_loop.lto_priv () from /work/kitty/launcher/../../kitty/fast_data_types.so
#3  0x00007fd08938dce2 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#4  0x00007fd08937fb2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#5  0x00007fd08931a5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#6  0x00007fd089381580 in _PyObject_FastCallDictTstate () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#7  0x00007fd0893817ee in _PyObject_Call_Prepend () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#8  0x00007fd089400075 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#9  0x00007fd08937f7df in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#10 0x00007fd08931a5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#11 0x00007fd08949d91f in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#12 0x00007fd0894998b0 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#13 0x00007fd0893dcadc in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#14 0x00007fd08937fb2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#15 0x00007fd08931a5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#16 0x00007fd089522242 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#17 0x00007fd089522da3 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#18 0x00007fd08952339c in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#19 0x000055c842c380ed in main ()
```

Run 2 (same structure; the io thread happened to be in `poll` this time), verbatim:

```
Thread 2 (Thread 0x7e025ffff6c0 (LWP 9001) "KittyChildMon"):
#0  0x00007e03953dd4cd in poll () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007e0394615008 in io_loop () from /work/kitty/launcher/../../kitty/fast_data_types.so
#2  0x00007e039535eaa4 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007e03953eba34 in clone () from /lib/x86_64-linux-gnu/libc.so.6
```

So the **main thread** owns the GLFW event loop (`glfwRunMainLoop` called from the C `main_loop`, entered from `kitty/main.py:234` — matching the py-spy backbone exactly), and a **dedicated `KittyChildMon` thread** runs `io_loop`, caught reading/polling a child PTY (the flood). This is the io thread py-spy could not see.

### Q4.4 — Fallback B: `eu-stack` (independent corroboration)

```bash
$ eu-stack -p <kitty_pid>
```

67 TIDs; the MainThread (TID 8543) and the io thread (TID 8615) agree with gdb. Verbatim excerpts of those two TID blocks (the main thread's frames `#12`–`#17` are libpython interpreter internals, elided **after** the relevant `main_loop.lto_priv.0`/`glfwRunMainLoop` frames have appeared):

```
PID 8543 - process
TID 8543:
#0  0x00007fd089083d71
#1  0x00007fd0890867ed pthread_cond_wait
#2  0x00007fd08474dedd
#3  0x00007fd084a1717b
#4  0x00007fd084a123a8
#5  0x00007fd0842ad274
#6  0x00007fd086d05b5c
#7  0x00007fd086d08ccf
#8  0x00007fd0884172f1 process_global_state
#9  0x00007fd087518493 dispatchTimers.part.0.constprop.0.isra.0
#10 0x00007fd0874fbb1e glfwRunMainLoop
#11 0x00007fd088413cfc main_loop.lto_priv.0
```

…and the io thread (TID 8615), complete:

```
TID 8615:
#0  0x00007fd0891064cd __poll
#1  0x00007fd088415008 io_loop
#2  0x00007fd089087aa4
#3  0x00007fd089114a34 __clone
```

`eu-stack` independently resolves the same two symbols that matter — `main_loop.lto_priv.0`/`glfwRunMainLoop` on the main thread and `io_loop` on TID 8615 — with a third, unrelated tool, confirming the gdb result was not a debugger artifact.

### Q4.5 — Thread inventory and reproducibility

Of the **67** threads, exactly **two are kitty's own**: `MainThread` (GLFW event+render loop) and `KittyChildMon` (`io_loop`, child-PTY read/write). The other **65 are Mesa `libgallium`/`llvmpipe` software-GL workers** — proven by sampling one (verbatim gdb block for Thread 32, a representative worker; all 65 share this exact `pthread_cond_wait → libgallium → clone` shape):

```
Thread 32 (Thread 0x7fcff17fa6c0 (LWP 8585) "kitty"):
#0  0x00007fd089083d71 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007fd0890867ed in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007fd08474dedd in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.24.04.2.so
#3  0x00007fd084a1653b in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.24.04.2.so
#4  0x00007fd08474de0c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.24.04.2.so
#5  0x00007fd089087aa4 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#6  0x00007fd089114a34 in clone () from /lib/x86_64-linux-gnu/libc.so.6
```

These worker threads never enter kitty code (no `fast_data_types.so` frame), so they are irrelevant to input handling — they exist only because software GL parallelizes rasterization.

No `KittyPeerMon` (talk/remote-control thread) and no `KittyWriteStdin` appeared, which the source explains: `io_thread` is **always** created (`kitty/child-monitor.c:291`), but `talk_thread` is created **only** with remote control enabled (`kitty/child-monitor.c:254-259`/`:286-289`, guarded by `talk_thread_started`), and `KittyWriteStdin` only on an explicit stdin write (`set_thread_name` sites: `KittyChildMon` `:1489`, `KittyPeerMon` `:1808`, `KittyWriteStdin` `:967`). With `--config NONE` (no `listen_on`), no talk thread is spawned.

**Reproducibility:** captured on **2 runs**; the thread inventory (MainThread + KittyChildMon + Mesa pool) and the call-path backbone (`main.py:234` → `main_loop` → `glfwRunMainLoop`; `io_loop` on `KittyChildMon`) were identical both times — only the instantaneous top frame differed (idle-in-`ppoll` vs actively rendering; `read` vs `poll`).

---

## Q5 — What happens to input for a window that is unfocused or just closed?

**Direct answer.**
- **Unfocused window:** it receives **nothing**. Input is routed to whatever `active_window()` resolves to (the focused window), so an unfocused window is simply never selected as the target.
- **Just-closed window:** input generated immediately after closing the active window is **re-routed to the new active (surviving) window**; the closed window's child receives nothing further. There is no crash and no misdelivery. Mechanically, once the child is gone its id no longer matches in the id-keyed loop of `schedule_write_to_child` (`kitty/child-monitor.c:336`), so a write to that id would `return found == false` (`kitty/child-monitor.c:369`) and be **silently dropped**; but in practice the C keystroke path re-resolves to the live active window before that can happen.

### Q5.1 — Input to an unfocused window (scenario S5b)

Two OS windows, A (kitty OS-window id `0x1`, child `KITTY_WINDOW_ID=1`) and B (id `0x2`, child `KITTY_WINDOW_ID=2`), were created with `new_os_window` and focus was steered with the injector's `focus <x11-window-id>` command. The `--debug-keyboard` focus trace (verbatim, ANSI colour codes stripped) shows A focused at startup, B gaining focus when created, then focus returning to A:

```
[0.158] on_focus_change: window id: 0x1 focused: 1
[0.916] on_focus_change: window id: 0x1 focused: 0
[0.916] on_focus_change: window id: 0x2 focused: 1
[1.812] on_focus_change: window id: 0x2 focused: 0
[1.812] on_focus_change: window id: 0x1 focused: 1
```

**STEP 1 — with focus on A (last line above, `0x1 focused: 1`), the string `AAAfocusA` was typed.** Every keystroke is routed to the child as text (verbatim excerpt; the trailing `A` is the shifted `a`, the rest are `f o c u s`):

```
[2.113] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
[2.125] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
[2.137] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
[2.149] on_key_input: glfw key: 0x66 native_code: 0x66 action: PRESS mods: none text: 'f' state: 0 sent key as text to child: f
[2.161] on_key_input: glfw key: 0x6f native_code: 0x6f action: PRESS mods: none text: 'o' state: 0 sent key as text to child: o
[2.173] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[2.185] on_key_input: glfw key: 0x75 native_code: 0x75 action: PRESS mods: none text: 'u' state: 0 sent key as text to child: u
[2.197] on_key_input: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[2.210] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
```

The per-child output files at this point (verbatim `cat`) show **A received the text and the unfocused B received nothing**:

```
$ cat win_1_focusB_test.txt          # A — was focused
[child start pid=8236 KITTY_WINDOW_ID=1]
AAAfocusA
$ cat win_2_focusB_test.txt          # B — was UNFOCUSED
[child start pid=8242 KITTY_WINDOW_ID=2]
```

**STEP 2 — focus was then moved to B and `BBBfocusB` typed.** The final per-child snapshot (verbatim `cat`) shows A's child **frozen** at its earlier content (it was now the unfocused one) while B's child received the new text:

```
$ cat win_1_unfocused_test.txt       # A — now UNFOCUSED, unchanged
[child start pid=8351 KITTY_WINDOW_ID=1]
AAAfocusA
$ cat win_2_unfocused_test.txt       # B — now focused
[child start pid=8357 KITTY_WINDOW_ID=2]
BBBfocusB
```

**Cause → effect:** while A was focused, only A's child grew (`AAAfocusA`) and B stayed empty; after focus moved to B, only B's child grew (`BBBfocusB`) and A stayed frozen. Input strictly follows OS focus — an unfocused window is never selected by `active_window()` and therefore receives nothing. (The differing child pids between the two snapshot pairs reflect that the intermediate and final states were captured from separate invocations of the S5b driver; the routing behaviour was identical in both.)

### Q5.2 — Input to a just-closed window (scenario S5a)

Two windows in one tab; the active one (`win_2`, holding `activeB`) was **closed** with `close_window` (`Ctrl+Shift+W`, `kitty/options/definition.py:3743`) and `postclose` was typed **immediately** (no delay):

```bash
xinj <<'CMDS'
key ctrl+shift+w      # close the ACTIVE window (win_2)
type postclose        # sent with zero settle time
key Return
CMDS
```

Trace — the close is consumed as a shortcut, then all nine letters of `postclose` are sent as text to a child (verbatim, ANSI colour codes stripped):

```
KeyPress matched action: close_window, handled as shortcut
[2.040] on_key_input: glfw key: 0x70 native_code: 0x70 action: PRESS mods: none text: 'p' state: 0 sent key as text to child: p
[2.052] on_key_input: glfw key: 0x6f native_code: 0x6f action: PRESS mods: none text: 'o' state: 0 sent key as text to child: o
[2.064] on_key_input: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[2.077] on_key_input: glfw key: 0x74 native_code: 0x74 action: PRESS mods: none text: 't' state: 0 sent key as text to child: t
[2.089] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[2.101] on_key_input: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
[2.113] on_key_input: glfw key: 0x6f native_code: 0x6f action: PRESS mods: none text: 'o' state: 0 sent key as text to child: o
[2.125] on_key_input: glfw key: 0x73 native_code: 0x73 action: PRESS mods: none text: 's' state: 0 sent key as text to child: s
[2.137] on_key_input: glfw key: 0x65 native_code: 0x65 action: PRESS mods: none text: 'e' state: 0 sent key as text to child: e
```

Raw per-child output (verbatim `cat`) — the survivor got it; the closed window's child is frozen at its pre-close content:

```
$ cat win_1_afterclose.txt          # survivor
[child start pid=8112 KITTY_WINDOW_ID=1]
postclose
$ cat win_2_closed.txt              # just-closed window's child
[child start pid=8118 KITTY_WINDOW_ID=2]
activeB
```

So the bytes were **re-routed to the new active window** (`win_1`), and the just-closed window's child (`win_2`) received nothing after `activeB` (its content before the close). Closing the **last** remaining window ended the OS window and the process — the driver's `kill -0 "$KPID"` liveness probe reported (verbatim driver output):

```
KITTY EXITED after last-window close
```

This ties to the close path `mark_for_close` (`kitty/child-monitor.c:568`) and the io_loop's removal of dead children at the top of its loop (`remove_children`, defined `kitty/child-monitor.c:1313`, called inside `io_loop` at `kitty/child-monitor.c:1493`).

### Q5.3 — The `no active window, ignoring` guard is defensive/rare (observed negative result)

The guard `if (!w) { debug("no active window, ignoring\n"); return; }` (`kitty/keys.c:182`) fires only when `active_window()` returns `NULL`. An attempt was made to trigger it with **6 rapid create→type→close cycles** plus a last-window-close key burst:

```bash
$ grep -c "no active window, ignoring" trace_s5a.log trace_s5a2.log
trace_s5a.log:0
trace_s5a2.log:0
```

**Count = 0.** Observed reason: `is_window_ready_for_callbacks()` (`kitty/glfw.c:202`) blocks `on_key_input` **entirely** when the active tab has `num_windows == 0`, and a window's screen is attached synchronously with creation, so `active_window()` (which requires `w->render_data.screen`, `kitty/keys.c:106`) essentially always resolves to a live window. The `NULL` guard is therefore a **defensive** path for transient teardown races and was not reachable through normal interactive close/create. (The specific "input arriving while `active_window()==NULL`" scenario is thus labelled **inferred** — the guard exists in source but did not fire at runtime.)

### Q5.4 — The Python-path silent drop (`Failed to write to child`)

The high-level Python write path `Window.write_to_child` logs `Failed to write to child {self.id} as it does not exist` (`kitty/window.py:960`) when `child_monitor.needs_write(self.id, data)` (`:959`) returns non-`True` — i.e. when C `needs_write` (`kitty/child-monitor.c:412`) → `schedule_write_to_child` finds no matching id (`return found == false`, `:369`). This is the guard for **stale references on the paste / remote-control / kitten path**, not for interactive keystrokes (which re-route or bail before reaching a dead id). It was **not** triggered by the interactive path in these runs; the id-keyed silent-drop **mechanism** is nonetheless demonstrated by Q5.2 (the closed window's id `win_2` was never matched again — its child stayed frozen). The exact `window.py:960` log line is therefore cited from source and labelled **inferred / not-triggered-interactively**.

---

## Q6 — Which parts are Python, which are C, which are external libraries? (+ refutations)

**Attribution table**, each row backed by an observed artifact from the runs above:

| Pipeline stage | Owning layer | Observed evidence |
|---|---|---|
| Receive raw OS key/focus/resize event | **External library** — patched GLFW X11 backend (`glfw/x11_window.c`) | Q4 stacks show deepest frames in `libX11`/`libxcb` under `kitty/glfw-x11.so` |
| Keysym → text translation | **External library** — libxkbcommon (`glfw/xkb_glfw.c`) | Baseline trace `xkb_keycode: 0x26 … xkb_key: 97 (a)` before `on_key_input` |
| IME composition | **External library** — IBus (`glfw/ibus_glfw.c`) | (present in build; not exercised — no IME configured; **inferred** from source) |
| GLFW event loop / dispatch | **External library** — `glfwRunMainLoop` (`kitty/glfw-x11.so`) | Q4 MainThread frame `glfwRunMainLoop (kitty/glfw-x11.so)` |
| `key_callback` / `on_key_input` / focus state | **C core** (`kitty/glfw.c`, `kitty/keys.c`, `kitty/state.c`) | `--debug-keyboard` lines `on_key_input`, `on_focus_change`; frames in `fast_data_types.so` |
| Key **encoding** | **C core** (`kitty/key_encoding.c:414`) | Trace `sent encoded key to child: ^[ [ D` (bytes generated in C) |
| **Write** to child / io loop | **C core** (`kitty/child-monitor.c`: `schedule_write_to_child`, `io_loop`) | gdb/eu-stack `#1 io_loop (fast_data_types.so)` on `KittyChildMon` |
| Shortcut arbitration | **Python** (`kitty/boss.py:1408` → `kitty/keys.py:154` → `:40`) | Trace `matched action: … handled as shortcut`; Q4 Python frames |
| Focus bookkeeping / high-level routing | **Python** (`kitty/boss.py:1651`, `kitty/window.py`) | S2b focus-report bytes reached child via `Boss.on_focus`→`Window.focus_changed` |

The C↔Python boundary is explicit in both directions: C→Python via `PyObject_CallMethod(global_state.boss, "dispatch_possible_special_key", …)` (`kitty/keys.c:221`) and the `WINDOW_CALLBACK(on_focus, …)` macro (`kitty/glfw.c:538`); Python→C via extension functions `needs_write`/`schedule_write_to_child`/`main_loop`.

### Refutations (each with runtime evidence)

- **Refute A — "the Go `kitten` binary routes interactive keystrokes."** **False.** The `tools/` Go tree builds a **separate** process that talks to a running kitty via remote control. Across every py-spy/gdb/eu-stack snapshot, **no Go frames appear**; the entire keystroke path is inside the kitty process's MainThread (GLFW→C→Python) and `KittyChildMon` (`io_loop`) threads. Evidence: Q4 §Q4.2–Q4.5 stacks (frames are `libX11`/`glfw-x11.so`/`fast_data_types.so`/`libpython3.12.so`, never any `*.go`/Go runtime).
- **Refute B — "Python encodes each keystroke and writes it to the PTY."** **False.** The encode+write is observed in **C**: `encode_glfw_key_event` (`kitty/keys.c:251`, defined `kitty/key_encoding.c:414`) → `schedule_write_to_child(w->id, …)` (`kitty/keys.c:259`, `kitty/child-monitor.c:372`), drained by the C `io_loop`. The `--debug-keyboard` line `sent encoded key to child:` is emitted by C (`kitty/keys.c:261`) with **no** Python write frame for ordinary keys; Python is consulted **only** for the shortcut decision (`dispatch_possible_special_key`). Evidence: baseline encoded-key trace + Q4 gdb `io_loop` frame in `fast_data_types.so`.
- **Refute C — "each window runs its own input thread."** **False.** A **single** main thread services all GLFW input callbacks and a **single** `io_loop` thread (`KittyChildMon`) services all child PTYs. The Q4 inventory shows exactly one MainThread + one `KittyChildMon`; the other 65 threads are Mesa software-GL workers (verified: their stacks bottom out in `libgallium`), **not** per-window input threads. Adding a second window/tab did not add an input thread.

---

## Q7 — One correctness-vs-responsiveness tradeoff (from observed behavior, not comments)

**Direct answer.** kitty handles input **synchronously on the main/UI thread** (GLFW callback → `on_key_input` → encode → enqueue into the target child's `write_buf`), while a **separate `io_loop` thread** (`KittyChildMon`, `kitty/child-monitor.c:1481`; declared `pthread_t io_thread, talk_thread;` at `:55`) drains those buffers to the PTYs. The two-thread split buys **responsiveness** — a flood of output on one child cannot stall input handling for another — and the id-keyed per-child `write_buf` preserves **correctness** (per-child write ordering, no cross-child leakage). The cost paid on the correctness side is that visible echo of typed input is **decoupled** from the keystroke and can be deferred by the io_loop's coalescing/throttle windows (`OPT(input_delay)` default `3` ms, `kitty/options/definition.py:878`, applied in the io_loop at `kitty/child-monitor.c:445`/`:1508`/`:1566`; `OPT(repaint_delay)` default `10` ms, `:866`).

**Observed evidence (scenario S4), stable across 2 runs.** One window (A) ran a busy output flood; a **different** split window (B) was focused and typed into. Per-keystroke end-to-end latency (wall time from XTEST injection of a marker to that marker appearing in B's child file) was measured with the flood **off** ("quiet") and **on** ("flood"). The flood throughput into A's PTY was measured from A's own progress counter.

```
RUN 1:  quiet per-keystroke ms: 90 91 90 90 91 89 90 89   → MEDIAN 90
        flood                                              → MEDIAN 90   (identical to quiet)
        flood throughput ≈ 198,000 lines/sec into window A's PTY
        isolation: 0 FLOOD-A lines leaked into B (B contained only mark1..mark8)

RUN 2:  quiet per-keystroke ms: 89 90 90 90 89 91 89 89   → MEDIAN 89
        flood per-keystroke ms: 89 90 89 90 90 91 90 89   → MEDIAN 90
        flood throughput = 185,186 lines/sec  (520000 lines / 2.808 s) into window A's PTY
        isolation: 0 FLOOD-A lines leaked into B
        flood-run trace: 102 on_key_input events; 40 "sent key as text to child" (8 markers × 5 chars)
```

Focused-window B's child file, both conditions (verbatim `cat`, RUN 2) — **only** the eight injected markers, **zero** flood leakage from window A:

```
$ cat win_B_quiet.txt
[child start pid=7382 KITTY_WINDOW_ID=2]
mark1
mark2
mark3
mark4
mark5
mark6
mark7
mark8
$ cat win_B_flood.txt
[child start pid=7561 KITTY_WINDOW_ID=2]
mark1
mark2
mark3
mark4
mark5
mark6
mark7
mark8
```

**The tradeoff, read strictly from the numbers:**
- **Responsiveness (won):** a background flood of **~185k–198k lines/sec** on an **unfocused** window produced **no measurable change** in focused-window keystroke latency (median **89–90 ms** whether quiet or flooded, in both runs). The main/UI thread kept servicing input while the `io_loop` thread absorbed the flood on another child. (The ~90 ms floor is the injector's per-word typing cadence, not kitty's handling cost; the decisive signal is its **invariance** under load.)
- **Correctness (preserved, and the cost):** every keystroke landed in the correct child in order, with **zero** cross-child leakage — because writes are serialized through the id-keyed per-child `write_buf` (`kitty/child-monitor.c:336`) and flushed by a single `io_loop`. The price is that echo is not synchronous with the keypress: input is enqueued on the UI thread and only later drained/coalesced by the io_loop under the `input_delay`/`repaint_delay` windows, so under heavy load the *display* of what you typed can lag the keypress even though the *routing* is exact. This is derived from the observed thread split (Q4: input on MainThread, draining on `KittyChildMon`) and the observed latency invariance under flood, **not** from any code comment.

---

## Repository left byte-for-byte unchanged

All observation scripts/artifacts lived **outside** the repository (container path `/kqna`, host `/tmp/kqna_work`) and were deleted after use. The only new tracked path in the entire repository is this one document. Verified at the repo root:

```bash
$ git status --porcelain
?? blitzy/                       # only the new documentation directory (this file)

$ git diff --stat                # tracked files — empty (no source file modified)

$ git check-ignore kitty/fast_data_types.so kitty/launcher/kitty
kitty/fast_data_types.so         # build outputs are .gitignore'd (not added)
kitty/launcher/kitty

$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

After `git add blitzy/documentation/kitty_815df1e210e0.md`, the staged set is exactly this single file; the build outputs (`*.so`, `kitty/launcher/kitty`) are excluded by the repo's existing `.gitignore` and were never added.

---

## Coverage pass (every sub-question answered with command + raw output + `file:line`)

| # | Sub-question | Answered by | Key `file:line` |
|---|---|---|---|
| Q1 | Which window receives input | §Q1 (S1 child files + shortcut trace) | `keys.c:106`, `state.c:355`, `boss.py:1384` |
| Q2 | How focus changes propagate | §Q2 (S2 `on_focus_change` pairs + S2b focus-report bytes) | `glfw.c:515,527,531,538`, `boss.py:1651`, `window.py:1123`, `screen.c:4611` |
| Q3 | Input routing to the child | §Q3 (baseline text/encoded/shortcut/Ctrl-C traces + S3) | `glfw.c:430,439`, `keys.c:166,182,221,251,257,259`, `key_encoding.c:414`, `child-monitor.c:323,336,369,372,412` |
| Q4 | Stack/symbol snapshot | §Q4 (py-spy `--native` + blocked-error + gdb + eu-stack, 2 runs) | `main.py:234`, `child-monitor.c:291,1481,1489` |
| Q5 | Unfocused / just-closed window | §Q5 (S5b follows-focus; S5a re-route/frozen child; NULL-guard=0) | `keys.c:106,182`, `child-monitor.c:336,369,568`, `window.py:960` |
| Q6 | Layer attribution + ≥2 refutations | §Q6 (attribution table + refutes A/B/C, snapshot-backed) | `keys.c:221,251,259`, `key_encoding.c:414`, `child-monitor.c:1481` |
| Q7 | One correctness-vs-responsiveness tradeoff | §Q7 (S4 latency invariance + isolation, 2 runs) | `child-monitor.c:55,336,445,1481`, `definition.py:866,878` |

Q6 provides **three** evidence-backed refutations (A, B, C). Q7's timing/magnitude claim is shown **stable across 2 runs**.

## Inferred-claim audit (claims not directly observed, labelled in-text)

1. **IBus/IME stage** (Q6 table): built in but not exercised (no IME configured) — **inferred** from source.
2. **`screen_send_signal_for_key` (kitty-sent SIGINT) branch** (Q3): only fires with `mHANDLE_TERMIOS_SIGNALS`; in the default config Ctrl-C was observed to write byte `0x03` and the **kernel** raised SIGINT, so the kitty-sent-signal branch is **inferred** from source (`keys.c:257`, `window.py:1116`, `child.py:499`).
3. **`no active window, ignoring` guard fires** (Q5.3): the guard exists (`keys.c:182`) but fired **0 times** in testing; the "input arrives while `active_window()==NULL`" trigger is **inferred** (defensive path).
4. **Python-path `Failed to write to child` log** (Q5.4): the interactive keystroke path did not trigger it; the exact `window.py:960` line is cited from source and labelled **inferred / not-triggered-interactively** (the id-keyed silent-drop *mechanism* is observed via the frozen closed-window child in Q5.2).

Everything else in this document is a direct runtime observation with its command and raw output shown adjacent.

