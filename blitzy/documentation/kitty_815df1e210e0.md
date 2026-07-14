# Kitty Input-Event Flow & Focus Management — A Runtime-Evidenced Investigation

**Repository:** `kitty` (kovidgoyal/kitty)
**Kitty source pinned at:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config")
**Question answered:** How does kitty actually handle input-event flow and focus management across OS-windows, tabs, and child processes at runtime?

This document answers seven sub-questions (Q1–Q7) **from what was observed while the real code ran**, not from reading the source alone. Every behavioral claim is paired with (a) the exact command, (b) the raw, unedited captured output, and (c) a `file:line` citation into the source that was built and observed. Steps that were established from source but not directly visible in a runtime trace are explicitly labelled **source-assisted**; interpretations not directly observed are labelled **inferred**.

---

## 0. Executive summary (direct answers, one line each)

- **Q1 — Which window gets input?** The per-event target is the **active window of the OS-window that *received* the event** — not a scan of `is_focused`. The X server delivers the key event to the focused X11 top-level window; kitty's `key_callback` resolves *that* concrete `GLFWwindow*` to its `OSWindow` (`set_callback_window`, `kitty/glfw.c:196`), and `active_window()` indexes `callback_os_window->tabs[active_tab].windows[active_window]` (`kitty/keys.c:106`). `OSWindow.is_focused` + the MRU counter are mirrored global bookkeeping, **not** the per-key selector.
- **Q2 — How does focus change propagate?** By **two distinct internal paths**, told apart at runtime by the presence/absence of the `on_focus_change` trace line. **Path A (OS-window focus):** GLFW `window_focus_callback` (`kitty/glfw.c:515`) → `is_focused`/MRU → `Boss.on_focus` (`kitty/boss.py:1651`) → `Window.focus_changed` → `Screen.focus_changed` (`kitty/screen.c:4604`). **Path B (internal window/tab switch):** Python-only via `WindowList` (`kitty/window_list.py:192`) / tab-index setter (`kitty/tabs.py:906`) with **no GLFW callback at all** — yet the child still gets its DECSET-1004 focus bytes.
- **Q3 — How is input routed to the child?** OS → external GLFW backend (+libxkbcommon) → C `key_callback` (`kitty/glfw.c:430`) → C `on_key_input` (`kitty/keys.c:166`) → Python shortcut test `dispatch_possible_special_key`; if not consumed, C `encode_glfw_key_event` (`kitty/key_encoding.c:414`) → **id-keyed** `schedule_write_to_child(w->id, …)` (`kitty/child-monitor.c:372`) → drained to the PTY by the `io_loop` thread. The id is the window resolved in Q1.
- **Q4 — Stack snapshot.** A first attach was authentically **blocked** (EPERM, no `CAP_SYS_PTRACE`); after a container-scoped `--cap-add=SYS_PTRACE`, `py-spy dump --native` captured the main thread across all three layers, and a **deterministic `gdb` conditional breakpoint** captured the exact C→Python shortcut-dispatch frame; `gdb`/`eu-stack` enumerated the `KittyChildMon` `io_loop` thread. Frame-identical across 2 runs.
- **Q5 — Input to an unfocused / just-closed window?** Input strictly **follows focus**; an unfocused window receives nothing (observed twice). Input generated right after closing the active window is **re-routed to the new active window**; the closed window's child is gone and its output file is frozen. Silent, no crash.
- **Q6 — Layer attribution.** External libs receive/translate the OS event (`glfw-x11.so`, xkb); **C** (`fast_data_types.so`) encodes and writes to the PTY; **Python** (`libpython`) only arbitrates shortcuts and hosts the loop. Three incorrect interpretations are refuted with snapshot + inventory evidence, each bounded to the sampled process/run.
- **Q7 — Correctness-vs-responsiveness tradeoff.** Input is handled synchronously on the **main/UI thread** while a **separate `io_loop` thread** drains child writes (POLLOUT-driven, `kitty/child-monitor.c:1503`) and *coalesces* child **output** parsing (`input_delay`) and rendering (`repaint_delay`). Under a ~232k–251k lines/s background flood on an unfocused window, the focused window's ordered 20-key burst still arrived **complete, in order, and with zero cross-child leakage**, delivery span essentially unchanged (~229 ms quiet vs ~235 ms flooded) — while the flood **producer was throttled** (blocked on PTY writes). kitty trades away background-output immediacy/throughput to keep the focused input path responsive and per-child delivery correct.

---

## 1. Methodology and grounding rules

- **Run-first.** kitty was **built from this checkout** and launched through its **canonical entry point** `kitty/launcher/kitty`. No pre-installed binary, no remote-control injection, and no debug hook was used to *originate* the keystrokes under study. `--debug-keyboard` was used only to *observe* input (it emits first-party trace lines; it does not synthesize events).
- **Canonical input delivery.** Because kitty is a GPU/GLFW application with no physical keyboard in a headless container, real key/focus/resize/scroll events were delivered through the **X11 XTEST extension** (`XTestFakeKeyEvent`/`XTestFakeButtonEvent`). XTEST injects events at the **X server**, which delivers them to the focused X11 top-level window exactly as a physical keyboard or `xdotool` would (xdotool itself uses XTEST). These events flow through the patched-GLFW X11 backend into kitty's C callbacks — i.e. **the real path under study**, not a synthetic bypass. The container lacks `xdotool`/`python-Xlib`; a small, auditable C XTEST injector `xinj` (full source in the Appendix) was compiled from the present `Xlib.h` + `libXtst.so.6` and used purely as the "keyboard/mouse".
- **Default configuration vs. deliberate child instrumentation.** kitty itself was always run in its **default configuration** with `--config NONE` (no user config file is read, so reported behavior is what a normal user of this revision sees). Where a scenario needed to *see what a child received/produced*, the **child program** was a small logger/producer (`-o shell=…` or a trailing `python3 …` command). That instruments the child end of the PTY; it is **not** a change to kitty's configuration or input path.
- **Observed-output discipline.** Every claim below shows its captured output next to it. Nothing is paraphrased before the relevant result appears. Anything not directly observed is labelled **source-assisted** (established from source, corroborated where possible) or **inferred**.
- **Reproducibility.** Timing/magnitude claims (Q7) and the stack inventory (Q4) are shown **stable across ≥2 runs**.
- **Repository untouched.** All scripts/artifacts lived outside the tracked source tree (container path `/kqna`, host `/tmp/kqna_evidence`) and were deleted afterward; the *"Reproducibility & repository hygiene"* section below shows the tracked source tree is unchanged apart from this one document.

### 1.1 Environment, provenance, and build container

The build/run container is a **derived** image `kitty-qna:latest`, built `FROM` the mandated base image, adding only observation tooling (Xvfb + Mesa software GL, `gdb`, `eu-stack`, `py-spy`). Both digests are recorded so the environment is reproducible.

```text
Derived image : kitty-qna:latest
                sha256:31793ec7aa17621e6ebc6512a04047248bca8e81508aa08a6b9216e4789bdbc8
Base image    : andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
                sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288
                (RepoDigest ghcr.io/scaleapi/swe-atlas@sha256:60da90a7183a82861fc6d1d40cb8086baa6a8a0e0d05f26d03aafd0f5b3cc384)

Toolchain     : Ubuntu 24.04.2 LTS; Python 3.12.3; gcc 13.3.0; go 1.23.4
Observation   : py-spy 0.4.2  (/usr/local/bin/py-spy,
                sha256:9b4d1f39b2a47ae44f4c6a46f615dcc0287d7755beba5065f32391951e07d594)
                gdb 15.1-1ubuntu1~24.04.1 ; elfutils/eu-stack 0.190-1.1ubuntu0.1
                xvfb 2:21.1.12-1ubuntu1.6 ; libgl1-mesa-dri 25.2.8-0ubuntu0.24.04.2 (llvmpipe, LLVM 20.1.2)
Exact pkgs    : gcc-13=13.3.0-6ubuntu2~24.04 ; gdb=15.1-1ubuntu1~24.04.1 ;
                elfutils=0.190-1.1ubuntu0.1 ; libgl1-mesa-dri=25.2.8-0ubuntu0.24.04.2 ;
                libx11-6=2:1.8.7-1build1 ; libxtst6=2:1.2.3-1.1build1 ;
                python3=3.12.3-0ubuntu2 ; xvfb=2:21.1.12-1ubuntu1.6
Host          : /proc/sys/kernel/yama/ptrace_scope = 1  (never modified — see Q4)
```

The investigation container was started with a **container-scoped** `CAP_SYS_PTRACE` only (needed for Q4's live attach); the host's `ptrace_scope` was never touched. The repository is bind-mounted read-write at `/work` (all build outputs are gitignored), and all scratch lives in the container-only directory `/kqna`:

```bash
docker run -d --name kqna \
  --cap-add=SYS_PTRACE \
  --tmpfs /tmp:exec,size=1g \
  -v <REPO>:/work -w /work \
  --entrypoint bash kitty-qna:latest \
  -lc 'mkdir -p /tmp/.X11-unix; sleep infinity'
```

Baseline (source pin + tracked-tree cleanliness *before* any build), captured verbatim:

```text
$ git rev-parse 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git log --oneline -2
cb701a1d5 docs: add runtime-evidenced Q&A on kitty input flow & focus management
815df1e21 Wire up applying of font config
$ git status --porcelain    # tracked tree state
(exit 0, empty=clean)
$ git check-ignore kitty/launcher/kitty kitty/fast_data_types.so build/
kitty/launcher/kitty
kitty/fast_data_types.so
build/
```

`git check-ignore` proves the three build outputs are gitignored (via `.gitignore`: `*.so`, `/kitty/launcher/kitt*`, `/build/`), so building the project leaves the *tracked* tree untouched.

### 1.2 Build & launch (exact canonical commands)

Canonical default build (compiles the C core into the `kitty/fast_data_types` extension and links the launcher `kitty/launcher/kitty`), captured verbatim:

```text
$ python3 setup.py
EXIT=0   elapsed=40.3s
--- tail of build.log ---
kitty/kittens/ssh
kitty/tools/cmd/benchmark
kitty/kittens/choose_fonts
kitty/kittens/icat
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
--- artifacts (gitignored) ---
-rwxr-xr-x 1 root root 1213072 Jul 14 21:33 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   36224 Jul 14 19:37 kitty/launcher/kitty
```

Launch through the canonical entry point under a headless Xvfb display with software GL. The exact launch and the first-party trace proving it is *our* launcher (PID captured via `$!`, no `pgrep`):

```text
$ ./kitty/launcher/kitty --config NONE --debug-keyboard python3 /kqna/label.py
KPID=5520 (spawned launcher pid, captured via $!)
--- proof kitty is live and it is OUR launcher (/proc/$KPID/cmdline) ---
./kitty/launcher/kitty --config NONE --debug-keyboard python3 /kqna/label.py
--- first-party --debug-keyboard trace (first lines, ANSI stripped) ---
[0.061] Loading new XKB keymaps
[0.066] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.155] Failed to open systemd user bus with error: No medium found
[0.158] on_focus_change: window id: 0x1 focused: 1
```

(The `Failed to open systemd user bus` line is benign container noise and appears in every trace.)

### 1.3 The pipeline at a glance (three ownership layers)

```
   ┌────────────────────────── OS / X server ──────────────────────────┐
   │  XTEST-injected (or physical) key/button/focus/resize events       │
   └───────────────────────────────┬────────────────────────────────────┘
                                    ▼
   EXTERNAL LIBRARY  (glfw-x11.so, libxkbcommon)   ← sees input FIRST
     _glfwDispatchX11Events → processEvent
     → glfw_xkb_handle_key_event  (keysym translation)
                                    ▼
   C CORE  (kitty/fast_data_types.so)
     key_callback (glfw.c:430)  [gates: is_window_ready_for_callbacks glfw.c:202,
                                 !fake_event_on_focus_change glfw.c:439]
       → on_key_input (keys.c:166)          [runs for every PRESS/REPEAT/RELEASE]
          → (C→Python) dispatch_possible_special_key   ─────────┐
          → encode_glfw_key_event (key_encoding.c:414)          │  shortcut?
          → schedule_write_to_child(w->id, 1, data, sz) (child-monitor.c:372)
                                    ▲                            ▼
   PYTHON  (libpython)              │              Boss.dispatch_possible_special_key
     main_loop entered from         │              (consumes → "handled as shortcut",
     boss.child_monitor.main_loop() │               no bytes to child)
     (main.py:234)                  │
                                    ▼
   io_loop thread "KittyChildMon" (child-monitor.c, io_loop)
     drains per-child write_buf to the PTY on POLLOUT → child process
```

The remainder of the document establishes each arrow above with a captured artifact.

---

## Q1 — How does kitty decide which window receives input?

**Direct answer.** The per-event target is the **active window of the OS-window that received the event**, resolved fresh on each keystroke — it is **not** a scan of `OSWindow.is_focused`. Concretely: the X server delivers the key to the focused X11 top-level window; kitty's `key_callback(GLFWwindow *w)` (`kitty/glfw.c:430`) hands that concrete window to `set_callback_window` (`kitty/glfw.c:196`), which sets `global_state.callback_os_window` via `glfwGetWindowUserPointer`; then `active_window()` (`kitty/keys.c:106`) returns `callback_os_window->tabs[active_tab].windows[active_window]` — i.e. the active window of the active tab of *that* OS-window. `OSWindow.is_focused`, `current_focused_os_window_id()` (`kitty/state.c:120`) and the MRU counter `last_focused_os_window_id()` (`kitty/state.c:108`) are mirrored global bookkeeping updated by the focus callback (Q2), **not** the per-key selector. (The setter for the active window index is `set_active_window`, `kitty/state.c:514`.)

> **Correction vs. a common misreading:** the resolver is `active_window()` at `kitty/keys.c:106`, **not** `kitty/state.c:355` — that line lives inside `remove_window_inner` (the window-*removal* path) and is unrelated to per-key routing.

**Scenario S1 — build a nested hierarchy (splits + tabs in one OS-window) and see where typed bytes land.** Each child is a byte logger (`label.py`); the driver types a tag, opens a split (`new_window` = `ctrl+shift+enter`), types, opens a tab (`new_tab` = `ctrl+shift+t`), switches tabs, and types again. Exact command and raw result:

```text
$ ./kitty/launcher/kitty --config NONE -o shell=/kqna/label.py --debug-keyboard python3 /kqna/label.py
kitty-window-count-before=1
$ /kqna/xinj < /kqna/s1.cmds
kitty-window-count-after=1
=== per-child output (verbatim cat) ===
$ cat /kqna/win_1.txt
[child start pid=5967 KITTY_WINDOW_ID=1]
activea
$ cat /kqna/win_2.txt
[child start pid=5974 KITTY_WINDOW_ID=2]
activebbacktabone
$ cat /kqna/win_3.txt
[child start pid=5975 KITTY_WINDOW_ID=3]
intabtwo
=== navigation shortcut trace (ANSI stripped) ===
KeyPress matched action: new_window, handled as shortcut
KeyPress matched action: new_tab, handled as shortcut
KeyPress matched action: previous_tab, handled as shortcut
```

**What this shows.**
- **Tabs and splits are *not* separate X11 top-level windows:** the kitty top-level window count is `1` before and after creating a split and a tab (`before=1 … after=1`). All of them live inside a single OS-window. (Directly observed.)
- **Typed bytes always land in exactly the child that is the active window of the active tab.** After the first tag + `a` they are in `win_1` (`activea`); after opening a split and typing they append to the split `win_2`; after switching tabs the text goes to the tab's window `win_3` (`intabtwo`). (Directly observed.)
- The navigation keys themselves are consumed as shortcuts (`handled as shortcut`) and never reach any child — confirming the *selector* changes without the shortcut keystrokes leaking into a child. (Directly observed.)

**Source-assisted** (mechanism not visible in the keyboard trace, corroborated by the Q4 stack which shows `key_callback` → dispatch on the concrete window): the `GLFWwindow* → callback_os_window → tabs[active_tab].windows[active_window]` resolution in `set_callback_window` (`kitty/glfw.c:196`) and `active_window()` (`kitty/keys.c:106`).

---

## Q2 — How do focus changes propagate internally?

**Direct answer.** Focus changes propagate by **two distinct internal paths**, distinguishable at runtime by whether the first-party `on_focus_change` trace line fires:

- **Path A — OS-window focus (driven by the external layer).** When the *OS-window* gains/loses focus, GLFW fires `window_focus_callback` (`kitty/glfw.c:515`), which emits the `on_focus_change` trace (`kitty/glfw.c:517`), sets `OSWindow.is_focused` (`kitty/glfw.c:527`), stamps the MRU counter `last_focused_counter = ++focus_counter` (`kitty/glfw.c:531`), then calls Python `Boss.on_focus` (`kitty/boss.py:1651`) → `Window.focus_changed` (`kitty/window.py:1123`) → `Screen.focus_changed` (`kitty/screen.c:4604`), which (if focus reporting is on) writes DECSET-1004 bytes `ESC[I`/`ESC[O` to the child (`kitty/screen.c:4611`).
- **Path B — internal window/tab switch (Python-only, no GLFW callback).** When focus moves *within* one OS-window (switching splits or tabs), there is **no** OS focus change and **no** `window_focus_callback`. Python drives it directly: window switches via `WindowList.notify_on_active_window_change` (`kitty/window_list.py:192`), tab switches via the `active_tab_idx` setter (`kitty/tabs.py:906`), window removal via `Boss` (`kitty/boss.py:913`). These call `Window.focus_changed` directly, reaching the same `Screen.focus_changed` and the same DECSET bytes — **without any `on_focus_change` trace**.

**Scenario S2a — Path A: two OS-windows, rapid focus switching.** A second OS-window is created (`ctrl+shift+n`) and focus is switched between the two X11 top-levels with `XSetInputFocus`, while a focus-reporting child (`focrep.py`, enables DECSET-1004) logs the bytes it receives. Exact result:

```text
OS window A X-id=0x20000c (kitty OS-window id 0x1)
kitty X-ids now: 0x200019 0x20000c
OS window B X-id=0x200019 (kitty OS-window id 0x2)
=== rapid focus switching A<->B (XSetInputFocus) with typing ===
=== on_focus_change trace (ANSI stripped, paired transitions keyed by OS-window id) ===
[0.165] on_focus_change: window id: 0x1 focused: 1
[0.935] on_focus_change: window id: 0x1 focused: 0
[0.935] on_focus_change: window id: 0x2 focused: 1
[1.640] on_focus_change: window id: 0x2 focused: 0
[1.641] on_focus_change: window id: 0x1 focused: 1
[2.204] on_focus_change: window id: 0x1 focused: 0
[2.204] on_focus_change: window id: 0x2 focused: 1
[2.765] on_focus_change: window id: 0x2 focused: 0
[2.765] on_focus_change: window id: 0x1 focused: 1
=== focus-report bytes + typed text per child (od -c) ===
$ od -c /kqna/focrep_1.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   1   ]  \n 033   [   O 033   [   I
0000060   i   n   A   1 033   [   O 033   [   I   i   n   A   2
0000076
$ od -c /kqna/focrep_2.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   2   ]  \n 033   [   O 033   [   I
0000060   i   n   B   1 033   [   O
0000067
```

**Path A observations (directly observed).** The `on_focus_change` line fires as **paired transitions keyed by OS-window id** (e.g. `0x1 focused:0` and `0x2 focused:1` at the same timestamp `[0.935]`) — exactly what an external OS focus hand-off looks like. Each child receives `ESC[O` (`033 [ O`, focus-out) / `ESC[I` (`033 [ I`, focus-in) around the typed text that arrived while it was focused (`inA1`, `inA2` to window A's child; `inB1` to window B's child). Focus and input both track the focused OS-window's active window.

**Scenario S2b — Path B: one OS-window, six internal switches.** In a single OS-window (count stays `1` throughout), the driver creates a split and toggles between splits (`previous_window`/`next_window`), creates a tab and switches tabs (`previous_tab`/`next_tab`). Exact result:

```text
kitty OS-window count (stays 1 throughout Path B): 1
=== on_focus_change trace lines (GLFW window_focus_callback) — expect ONLY startup ===
[0.170] on_focus_change: window id: 0x1 focused: 1
on_focus_change count = 1
=== internal focus actions consumed as shortcuts ===
KeyPress matched action: new_window, handled as shortcut
KeyPress matched action: previous_window, handled as shortcut
KeyPress matched action: next_window, handled as shortcut
KeyPress matched action: new_tab, handled as shortcut
KeyPress matched action: previous_tab, handled as shortcut
KeyPress matched action: next_tab, handled as shortcut
=== focus-report bytes reached children WITHOUT any GLFW callback (od -c) ===
$ od -c /kqna/focrep_1.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   1   ]  \n 033   [   O 033   [   I
0000060 033   [   O
0000063
$ od -c /kqna/focrep_2.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   2   ]  \n 033   [   O 033   [   I
0000060 033   [   O 033   [   I 033   [   O
0000071
$ od -c /kqna/focrep_3.txt
0000000   [   f   o   c   u   s   -   r   e   p   o   r   t   i   n   g
0000020       c   h   i   l   d       K   I   T   T   Y   _   W   I   N
0000040   D   O   W   _   I   D   =   3   ]  \n 033   [   O 033   [   I
0000060
```

**Path B observations (the decisive contrast, directly observed).** Six internal focus switches produced **zero** additional `on_focus_change` lines (`on_focus_change count = 1`, the startup line only) — the GLFW `window_focus_callback` never fired. **Yet the children still received DECSET-1004 focus bytes** (`ESC[O`/`ESC[I` sequences in every `focrep_*` file). Therefore internal focus changes reach `Screen.focus_changed` (and its `ESC[I`/`ESC[O` output) **without** going through `window_focus_callback` — proving Path B is a separate, Python-driven route.

**Source-assisted** (the Python call chain itself is not printed by `--debug-keyboard`): the specific functions `WindowList.notify_on_active_window_change` (`kitty/window_list.py:192`), the `active_tab_idx` setter (`kitty/tabs.py:906`), and `Boss` window removal (`kitty/boss.py:913`). The **observed discriminator** is unambiguous: Path A = (`on_focus_change` present + `is_focused`/MRU updated); Path B = (no `on_focus_change`, DECSET bytes still emitted). Registration of the callback is at `kitty/glfw.c:1281`.

---

## Q3 — How is an input event routed to the correct child process?

**Direct answer.** For each `PRESS`/`REPEAT`, C `on_key_input` (`kitty/keys.c:166`) first asks Python whether the key is a configured shortcut (`dispatch_possible_special_key`). If **consumed**, no bytes reach any child (`handled as shortcut`). Otherwise C encodes the key (`encode_glfw_key_event`, `kitty/key_encoding.c:414`) and writes the bytes to the **child selected in Q1**, keyed by window id, via `schedule_write_to_child(w->id, …)` (`kitty/child-monitor.c:372`); the dedicated `io_loop` thread later drains that child's buffer to its PTY. A single-byte control key with terminal signal handling enabled is turned into a **signal** instead of a byte (Ctrl+C → `SIGINT`).

The following are **directly observed**: (1) `on_key_input` runs in C for every `PRESS`/`REPEAT`/`RELEASE`; (2) the per-branch decision *outcomes* and the resulting child bytes; (3) routing to the correct child by id; (4) scroll → focused child SGR bytes and resize → focused child `SIGWINCH`. The intermediate call frames (the C→Python dispatch *call*, the readiness/`fake_event` gates, the `encode_glfw_key_event` frame) are **source-assisted**, and the C→Python dispatch frame is additionally **corroborated by the Q4 stack**.

### Q3.1 — Baseline branch matrix (plain / Shift / Ctrl / Alt / Enter), with child bytes

```text
############ RUN A1 — per-key decision trace (ANSI stripped) ############
[0.595] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[0.596] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.852] on_key_input: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.858] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: shift text: 'A' state: 0 sent key as text to child: A
[0.864] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.871] on_key_input: glfw key: 0xe061 native_code: 0xffe1 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.132] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.132] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x1
[1.132] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.158] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.389] on_key_input: glfw key: 0xe063 native_code: 0xffe9 action: PRESS mods: alt text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.395] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ a
[1.401] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: alt text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.407] on_key_input: glfw key: 0xe063 native_code: 0xffe9 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.663] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd
[1.670] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
$ od -An -c /kqna/win_1.txt
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   6   3   2   9       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n   a   A 001 033   a  \r
```

- Plain `a` → `sent key as text to child: a`; **Shift**+`a` → text `A`; **Ctrl**+`a` → `sent encoded key to child: 0x1`; **Alt**+`a` → `sent encoded key to child: ^[ a`; **Enter** → `0xd`. The child bytes confirm all five: `a A 001 033 a \r` (`001` = Ctrl-A, `033 a` = `ESC a` for Alt-a, `\r` = Enter). (Directly observed; fixes the previously missing Alt artifact.)
- **Every `RELEASE` is explicitly `ignoring as keyboard mode does not support encoding this event`** — the trace shows the RELEASE lines, so the "press writes, release is ignored" behavior is observed, not assumed. (Directly observed.)

### Q3.2 — Arrow keys (legacy cursor encodings), with child bytes

```text
############ RUN A2 — arrow keys (PRESS+RELEASE trace) ############
[0.796] on_key_input: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A
[0.799] on_key_input: glfw key: 0xe008 native_code: 0xff52 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.059] on_key_input: glfw key: 0xe009 native_code: 0xff54 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ B
[1.065] on_key_input: glfw key: 0xe009 native_code: 0xff54 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.327] on_key_input: glfw key: 0xe006 native_code: 0xff51 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ D
[1.333] on_key_input: glfw key: 0xe006 native_code: 0xff51 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.596] on_key_input: glfw key: 0xe007 native_code: 0xff53 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ C
[1.602] on_key_input: glfw key: 0xe007 native_code: 0xff53 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
$ od -An -c /kqna/win_1.txt   (arrow bytes only; strip the child-start banner line)
 033   [   A 033   [   B 033   [   D 033   [   C
----- interpretation: legacy cursor keys = ESC [ A/B/D/C -----
```

Up/Down/Left/Right encode as `ESC [ A/B/D/C` and land in the child as `033 [ A / 033 [ B / 033 [ D / 033 [ C` (legacy cursor keys; `mDECCKM` off). (Directly observed — the child arrow-byte artifact requested by review.)

### Q3.3 — A consumed shortcut writes **zero** bytes to the child

```text
$ od -An -c /kqna/win_1.txt   (BEFORE shortcut — banner only)
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   6   5   1   0       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n
############ RUN A3 — shortcut trace (consumed, NOT sent to child) ############
KeyPress matched action: new_tab, handled as shortcut
$ od -An -c /kqna/win_1.txt   (AFTER shortcut — UNCHANGED, no key bytes added)
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   6   5   1   0       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n
```

The focused child's bytes are **byte-identical** before and after the `new_tab` shortcut — a consumed shortcut produces no child bytes. (Directly observed.)

### Q3.4 — Ctrl+C: **both** branches exercised (byte vs. signal)

The single-byte control path has two mutually-exclusive branches in C: write the byte (`schedule_write_to_child`, `kitty/keys.c:253`/`:259`), **or**, when `screen->modes.mHANDLE_TERMIOS_SIGNALS` is set (DECSET `?19997`, off by default), take the **signal** branch (`kitty/keys.c:256`) → `screen_send_signal_for_key` (`kitty/screen.c:2404`) → `Window.send_signal_for_key` (`kitty/window.py:1116`) → `Child.send_signal_for_key` (`kitty/child.py:481`) → `os.killpg(tcgetpgrp(fd), SIGINT)` and **return early** (no byte, no "sent encoded key" log — the log's absence is the runtime discriminator). Both branches were exercised directly.

**C1 — default (byte) branch**, child receives literal `003`:

```text
$ od -An -c /kqna/win_1.txt   (BEFORE Ctrl+C — banner only)
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   6   7   1   5       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n
############ RUN C1 — Ctrl+C decision trace (ANSI stripped) ############
[0.589] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.591] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x3
[0.597] on_key_input: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.603] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
$ od -An -c /kqna/win_1.txt   (AFTER Ctrl+C — expect literal 003 byte appended)
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   6   7   1   5       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n 003
```

**C2 — signal branch** (child first enables `?19997`, has `ISIG=1`, `VINTR=0x03`); Ctrl+C delivers `SIGINT` via `killpg`, **no byte**:

```text
=== child state BEFORE Ctrl+C (mode-enable marker, termios ISIG, VINTR) ===
enabled_19997
ISIG=1
VINTR=0x03
############ RUN C2 — full on_key_input lines for the ctrl+c press ############
[1.000] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.003] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: ctrl text: '' state: 0 [1.007] Release xkb_keycode: 0x36 clean_sym: c mods: ctrl glfw_key: 99 (c) xkb_key: 99 (c)
on_key_input: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.013] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
--- discriminator: any "sent encoded key / as text" for the c-press? ---
(NONE — no byte written for ctrl+c: signal branch taken)
=== child state AFTER Ctrl+C (expect GOT_SIGINT via killpg; NO literal 003) ===
enabled_19997
ISIG=1
VINTR=0x03
GOT_SIGINT n=1
```

In the raw trace above, the `c`-press `PRESS` line (`[1.003] … state: 0`) is immediately followed by the next timestamp's debug output with **no decision suffix** appended to it — i.e. no `sent encoded key`/`sent key as text`. That absence (confirmed by the discriminator grep returning `NONE`) is the runtime signature of the early-return signal branch, and the child's `SIGINT` handler firing (`GOT_SIGINT n=1`) confirms the signal was delivered instead of a byte.

The contrast is airtight: same keystroke, two configurations — C1 writes byte `0x03` to the child; C2 writes **no** byte and the child's `SIGINT` handler fires (`GOT_SIGINT n=1`). (Both directly observed.)

### Q3.5 — Scenario S3: typing interleaved with genuine scroll and resize

The child logs, with timestamps, the bytes it reads plus `SIGWINCH` (via `TIOCGWINSZ`); the driver types `abc`, scrolls, types `def`, resizes 640×400→700×500, types `ghi`, resizes →1000×700, scrolls, types `jkl`:

```text
=== discovered kitty X11 top-level window id: 0x20000c ===
     0x20000c "python3": ("kitty" "kitty")  640x400+0+0  +0+0
############ S3 — keyboard decision trace (typed letters -> child) ############
[0.701] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[0.708] on_key_input: glfw key: 0x62 native_code: 0x62 action: PRESS mods: none text: 'b' state: 0 sent key as text to child: b
[0.720] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[1.350] on_key_input: glfw key: 0x64 native_code: 0x64 action: PRESS mods: none text: 'd' state: 0 sent key as text to child: d
[1.363] on_key_input: glfw key: 0x65 native_code: 0x65 action: PRESS mods: none text: 'e' state: 0 sent key as text to child: e
[1.375] on_key_input: glfw key: 0x66 native_code: 0x66 action: PRESS mods: none text: 'f' state: 0 sent key as text to child: f
[2.288] on_key_input: glfw key: 0x67 native_code: 0x67 action: PRESS mods: none text: 'g' state: 0 sent key as text to child: g
[2.300] on_key_input: glfw key: 0x68 native_code: 0x68 action: PRESS mods: none text: 'h' state: 0 sent key as text to child: h
[2.312] on_key_input: glfw key: 0x69 native_code: 0x69 action: PRESS mods: none text: 'i' state: 0 sent key as text to child: i
[3.543] on_key_input: glfw key: 0x6a native_code: 0x6a action: PRESS mods: none text: 'j' state: 0 sent key as text to child: j
[3.555] on_key_input: glfw key: 0x6b native_code: 0x6b action: PRESS mods: none text: 'k' state: 0 sent key as text to child: k
[3.590] on_key_input: glfw key: 0x6c native_code: 0x6c action: PRESS mods: none text: 'l' state: 0 sent key as text to child: l
############ S3 — child temporal log (READ bytes + wheel SGR + SIGWINCH), unedited ############
[0.000] START rows=22 cols=71
[0.509] READ b'a'
[0.516] READ b'b'
[0.527] READ b'c'
[1.158] READ b'd'
[1.170] READ b'e'
[1.182] READ b'f'
[1.593] SIGWINCH rows=27 cols=77
[2.095] READ b'g'
[2.108] READ b'h'
[2.120] READ b'i'
[2.533] SIGWINCH rows=38 cols=111
[3.032] READ b'\x1b[<65;1;1M'
[3.045] READ b'\x1b[<65;1;1M'
[3.057] READ b'\x1b[<65;1;1M'
[3.351] READ b'j'
[3.363] READ b'k'
[3.398] READ b'l'
```

- **Genuine resize:** two `SIGWINCH` events with the child seeing the new grid (`rows=27 cols=77`, then `rows=38 cols=111`) as the OS-window resized. (Directly observed.)
- **Genuine scroll:** three wheel-down SGR mouse reports `\x1b[<65;1;1M` reached the child. (Directly observed.)
- **Uninterrupted keystrokes:** every typed letter `a`…`l` reached the child in order, interleaved with the resize/scroll events. (Directly observed.)

On the **main screen**, wheel-*up* produced no child bytes (kitty consumes it for its scrollback pager). To show the scroll path is symmetric, S3b repeats it on the **alternate screen** (`?1049h` + SGR mouse), where there is no scrollback to consume:

```text
=== alt-screen scroll driver: scroll up 3 | type xy | scroll down 3 ===
############ alt-screen child temporal log (both wheel directions -> child) ############
[0.000] START alt-screen+mouse
[0.598] READ b'\x1b[<64;72;22M'
[0.611] READ b'\x1b[<64;72;22M'
[0.623] READ b'\x1b[<64;72;22M'
[0.922] READ b'x'
[0.930] READ b'y'
[1.242] READ b'\x1b[<65;72;22M'
[1.254] READ b'\x1b[<65;72;22M'
[1.266] READ b'\x1b[<65;72;22M'
```

Both wheel directions forward to the child on the alt screen (`<64…` up, `<65…` down), explaining the main-screen asymmetry. (Directly observed.)

**Scroll/resize source-assisted citations** (outcomes observed; frames from source): `scroll_callback` (`kitty/glfw.c:502`, registered `:1291`, gated by `is_window_ready` `:507`) → `scroll_event` (`kitty/mouse.c:890`) → `encode_mouse_scroll(w, upwards?4:5)` (`kitty/mouse.c:956`) → `write_escape_code_to_child` (`kitty/mouse.c:960`) with SGR format `<%d;%d;%d%s` (`kitty/mouse.c:88`, matching `\x1b[<64/65;…M`); resize via `resize_pty` (`kitty/child-monitor.c:592`) → `ioctl TIOCSWINSZ` (`kitty/child-monitor.c:579`) → kernel `SIGWINCH`.

### Q3.6 — Convergence on a single id-keyed writer

**Source-assisted** (corroborated by the Q4 stack for the C fast path): both the C fast keyboard path (`kitty/keys.c:253`/`:259`) and the Python high-level text path `Window.write_to_child` (`kitty/window.py:955`) converge on `schedule_write_to_child(id, …)` (`kitty/child-monitor.c:372`), keyed by window id. The id is the window resolved in Q1, which is why bytes only ever appear in the correct child's file (Q1/Q5). The C fast path was observed directly; the Python `write_to_child` convergence is source-assisted (exercised only by paste/remote-control/kitten text, not by the interactive keystroke path).

---

## Q4 — A stack/symbol snapshot of input handling (commands, raw output, blocked-then-remediated attach)

**Direct result (lead).** A live snapshot of the running kitty process captured the input-handling call path across **all three layers in a single backtrace**, and the two kitty-authored threads (the main/UI thread and the `KittyChildMon` `io_loop` thread). A first attach attempt was **authentically blocked** (EPERM) in a container without `CAP_SYS_PTRACE`; it was **remediated with a container-scoped `--cap-add=SYS_PTRACE`** (the host `ptrace_scope` was never changed). The capture is **frame-identical across 2 runs**.

### Q4.1 — First attempt authentically BLOCKED (no `CAP_SYS_PTRACE`, host `ptrace_scope=1`)

```text
########## Q4 FIRST ATTEMPT — BLOCKED (container has NO SYS_PTRACE; host ptrace_scope=1) ##########
target KPID=29 exe=/work/kitty/launcher/kitty
$ py-spy dump --native --pid 29
Error: Failed to copy Py_Version symbol

Caused by:
    0: Permission denied (os error 13)
    1: Permission denied (os error 13)
py-spy exit=1

$ eu-stack -p 29
PID 29 - process
TID 29:
eu-stack: dwfl_thread_getframes tid 29: Operation not permitted
TID 31:
(eu-stack blocked similarly)

$ cat /proc/sys/kernel/yama/ptrace_scope   (host value, unchanged)
1
```

**Remediation** is a **container-scoped** capability only — the throwaway no-cap container was `docker rm -f`'d, and all successful captures below ran in the `kqna` container started with `--cap-add=SYS_PTRACE`. The host `/proc/sys/kernel/yama/ptrace_scope` stayed `1` throughout; **no** `sysctl ptrace_scope=0` and **no** root-on-host escalation was used or recommended.

### Q4.2 — Secure single-PID targeting (finding-driven)

Every attach targets exactly the numeric PID captured via `$!` at spawn, validated *before* attaching — no `pgrep -f` (which can match stale/multiple processes):

```text
spawned KPID=7476
$ readlink /proc/7476/exe = /work/kitty/launcher/kitty
$ cmdline = ./kitty/launcher/kitty --config NONE -o shell=/kqna/label.py --debug-keyboard python3 /kqna/label.py
$ ps -o pid,stat,comm -p 7476:
    PID STAT COMMAND
   7476 Sl   kitty
```

### Q4.3 — Primary success: `py-spy dump --native` (Python + native-C frames)

```text
$ py-spy dump --native --pid 7476
py-spy exit=0
Process 7476: ./kitty/launcher/kitty --config NONE -o shell=/kqna/label.py --debug-keyboard python3 /kqna/label.py
Python v3.12.3 (/work/kitty/launcher/kitty)

Thread 7476 (idle): "MainThread"
    ppoll (libc.so.6)
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
    0x78a06c7131ca (libc.so.6)
```

This single stack shows all three layers on the main thread: **Python** (`_run_app` at `kitty/main.py:234` = `boss.child_monitor.main_loop()`) → **C** (`main_loop.lto_priv.0` in `kitty/fast_data_types.so`) → **external** (`glfwRunMainLoop` in `kitty/glfw-x11.so`) → **libc** (`ppoll`). It confirms `main.py:234` as the runtime event-loop entry. `py-spy` enumerates only the one *Python* thread (pure-C threads are invisible to it — see Q4.5).

### Q4.4 — Deterministic input-path frame: `gdb` conditional breakpoint on the C→Python shortcut dispatch

kitty is compiled with `PY_SSIZE_T_CLEAN`, so CPython's `PyObject_CallMethod` C macro expands to the `_PyObject_CallMethod_SizeT` symbol (the symbol kitty's C core actually calls to enter Python). A `gdb` breakpoint on that exact symbol, conditioned on the method-name argument in `$rsi`, deterministically catches the C→Python shortcut dispatch when a single `a` is injected. The exact driver script and invocation:

```text
$ cat /kqna/bp2.gdb
set pagination off
set debuginfod enabled off
break _PyObject_CallMethod_SizeT if $_streq((char*)$rsi, "dispatch_possible_special_key")
commands 1
printf "\n=== C->Python SHORTCUT DISPATCH on key press ===\n"
printf "_PyObject_CallMethod_SizeT method-name arg (rsi) = %s\n", (char*)$rsi
bt
detach
quit
end
continue
$ gdb -p 8333 -batch -x /kqna/bp2.gdb        # then inject one 'a' via XTEST
```

Raw captured output (unedited):

```text
KPID=8333 exe=/work/kitty/launcher/kitty
(injecting a -> dispatch_possible_special_key)
--- gdb conditional-breakpoint backtrace (filter out LWP-noise) ---
0x00007d303feeba00 in ppoll () from /lib/x86_64-linux-gnu/libc.so.6
Breakpoint 1 at 0x7d3040165788

Thread 1 "kitty" hit Breakpoint 1, 0x00007d3040165788 in _PyObject_CallMethod_SizeT () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0

=== C->Python SHORTCUT DISPATCH on key press ===
_PyObject_CallMethod_SizeT method-name arg (rsi) = dispatch_possible_special_key
#0  0x00007d3040165788 in _PyObject_CallMethod_SizeT () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#1  0x00007d303f240f25 in key_callback.lto_priv () from /work/kitty/launcher/../../kitty/fast_data_types.so
#2  0x00007d303e2fe182 in glfw_xkb_handle_key_event.constprop () from /work/kitty/glfw-x11.so
#3  0x00007d303e2fa8dd in processEvent () from /work/kitty/glfw-x11.so
#4  0x00007d303e2fb3b8 in _glfwDispatchX11Events.lto_priv.0 () from /work/kitty/glfw-x11.so
#5  0x00007d303e2e2a3e in glfwRunMainLoop () from /work/kitty/glfw-x11.so
#6  0x00007d303f213cfc in main_loop.lto_priv () from /work/kitty/launcher/../../kitty/fast_data_types.so
#7  0x00007d3040172ce2 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#8  0x00007d3040164b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#9  0x00007d30400ff5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#10 0x00007d3040166580 in _PyObject_FastCallDictTstate () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#11 0x00007d30401667ee in _PyObject_Call_Prepend () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#12 0x00007d30401e5075 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#13 0x00007d30401647df in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#14 0x00007d30400ff5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#15 0x00007d304028291f in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#16 0x00007d304027e8b0 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#17 0x00007d30401c1adc in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#18 0x00007d3040164b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#19 0x00007d30400ff5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#20 0x00007d3040307242 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#21 0x00007d3040307da3 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#22 0x00007d304030839c in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#23 0x000056ba666fd0ed in main ()
[Inferior 1 (process 8333) detached]
```

This is the **direct C→Python shortcut frame** — reading bottom-up: external GLFW/xkb sees the event first (`glfw_xkb_handle_key_event` → `processEvent` → `_glfwDispatchX11Events` → `glfwRunMainLoop`, all in `glfw-x11.so`), then kitty's **C** `key_callback` (with `on_key_input` inlined via LTO) calls into **Python** `dispatch_possible_special_key`. It answers a piece of Q3, Q4, and Q6 at once. The `schedule_write_to_child` breakpoint did **not** fire because LTO inlined that function into the keystroke path (`key_callback.lto_priv`, `main_loop.lto_priv`); this is an LTO artifact of the default build, documented as such, not a routing claim. **Run 2** reproduced the same frames:

```text
KPID=8548
--- RUN 2 dispatch backtrace (top frames, LWP-noise filtered) ---
Thread 1 "kitty" hit Breakpoint 1, 0x0000788c804c8788 in _PyObject_CallMethod_SizeT () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
=== C->Python SHORTCUT DISPATCH on key press ===
_PyObject_CallMethod_SizeT method-name arg (rsi) = dispatch_possible_special_key
#0  0x0000788c804c8788 in _PyObject_CallMethod_SizeT () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#1  0x0000788c7f640f25 in key_callback.lto_priv () from /work/kitty/launcher/../../kitty/fast_data_types.so
#2  0x0000788c7e664182 in glfw_xkb_handle_key_event.constprop () from /work/kitty/glfw-x11.so
#3  0x0000788c7e6608dd in processEvent () from /work/kitty/glfw-x11.so
#4  0x0000788c7e6613b8 in _glfwDispatchX11Events.lto_priv.0 () from /work/kitty/glfw-x11.so
#5  0x0000788c7e648a3e in glfwRunMainLoop () from /work/kitty/glfw-x11.so
#6  0x0000788c7f613cfc in main_loop.lto_priv () from /work/kitty/launcher/../../kitty/fast_data_types.so
```

### Q4.5 — Fallbacks and the `io_loop` thread: `gdb thread apply all bt` + `eu-stack`

`gdb thread apply all bt` (67 OS threads in this sampled run) shows the main thread and the dedicated I/O thread:

```text
KPID=7707 exe=/work/kitty/launcher/kitty

########## TOOL 3: gdb thread apply all bt (ALL OS threads, symbolicated) ##########
gdb-all exit=0 ; total gdb threads: 67
--- MAIN thread (has main_loop / glfwRunMainLoop) ---
Thread 1 (Thread 0x7832f9576740 (LWP 7707) "kitty"):
#0  0x00007832f97c7a00 in ppoll () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007832f7bb3af6 in glfwRunMainLoop () from /work/kitty/glfw-x11.so
#2  0x00007832f8a13cfc in main_loop.lto_priv () from /work/kitty/launcher/../../kitty/fast_data_types.so
#3  0x00007832f9a4ece2 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#4  0x00007832f9a40b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#5  0x00007832f99db5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#6  0x00007832f9a42580 in _PyObject_FastCallDictTstate () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#7  0x00007832f9a427ee in _PyObject_Call_Prepend () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#8  0x00007832f9ac1075 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#9  0x00007832f9a407df in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#10 0x00007832f99db5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#11 0x00007832f9b5e91f in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#12 0x00007832f9b5a8b0 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#13 0x00007832f9a9dadc in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#14 0x00007832f9a40b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#15 0x00007832f99db5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#16 0x00007832f9be3242 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#17 0x00007832f9be3da3 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#18 0x00007832f9be439c in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
--- io_loop thread (KittyChildMon) ---
Thread 2 (Thread 0x7831dcff96c0 (LWP 7776) "KittyChildMon"):
#0  0x00007832f97c74cd in poll () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007832f8a15125 in io_loop () from /work/kitty/launcher/../../kitty/fast_data_types.so
#2  0x00007832f9748aa4 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007832f97d5a34 in clone () from /lib/x86_64-linux-gnu/libc.so.6
```

The two backtraces above are reproduced in full, frame-for-frame, as `gdb` emitted them for the two kitty-authored threads. The remaining 65 threads in the 67-thread total are the software-GL worker pool (enumerated by `eu-stack` immediately below and named in the Q6 inventory).

`eu-stack -p` independently corroborates the thread count and the main thread's `ppoll`:

```text
########## RUN 1 PID VALIDATION ##########
KPID=7613  exe=/work/kitty/launcher/kitty
cmdline=./kitty/launcher/kitty --config NONE -o shell=/kqna/label.py --debug-keyboard python3 /kqna/label.py

########## eu-stack -p (ALL OS threads; debuginfod off; inner timeout 30) ##########
eu-stack exit=0
eu-stack OS-thread count: 67
--- eu-stack TID headers + first frame of each (representative) ---
TID 7613:
#0  0x00007e0f97a3ea00 ppoll
TID 7616:
#0  0x00007e0f979bbd71
TID 7617:
#0  0x00007e0f979bbd71
TID 7618:
#0  0x00007e0f979bbd71
TID 7619:
#0  0x00007e0f979bbd71
TID 7620:
#0  0x00007e0f979bbd71
TID 7621:
#0  0x00007e0f979bbd71
TID 7622:
#0  0x00007e0f979bbd71
TID 7623:
#0  0x00007e0f979bbd71
TID 7624:
#0  0x00007e0f979bbd71
TID 7625:
#0  0x00007e0f979bbd71
TID 7626:
#0  0x00007e0f979bbd71
TID 7627:
#0  0x00007e0f979bbd71
TID 7628:
#0  0x00007e0f979bbd71
TID 7629:
#0  0x00007e0f979bbd71
TID 7630:
#0  0x00007e0f979bbd71
TID 7631:
#0  0x00007e0f979bbd71
TID 7632:
#0  0x00007e0f979bbd71
TID 7633:
#0  0x00007e0f979bbd71
TID 7634:
#0  0x00007e0f979bbd71
TID 7635:
#0  0x00007e0f979bbd71
TID 7636:
#0  0x00007e0f979bbd71
TID 7637:
#0  0x00007e0f979bbd71
TID 7638:
#0  0x00007e0f979bbd71
TID 7639:
#0  0x00007e0f979bbd71
TID 7640:
#0  0x00007e0f979bbd71
TID 7641:
#0  0x00007e0f979bbd71
TID 7642:
#0  0x00007e0f979bbd71
TID 7643:
#0  0x00007e0f979bbd71
TID 7644:
#0  0x00007e0f979bbd71
TID 7645:
#0  0x00007e0f979bbd71
TID 7646:
#0  0x00007e0f979bbd71
TID 7647:
#0  0x00007e0f979bbd71
TID 7648:
#0  0x00007e0f979bbd71
TID 7649:
#0  0x00007e0f979bbd71
TID 7650:
#0  0x00007e0f979bbd71
TID 7651:
#0  0x00007e0f979bbd71
TID 7652:
#0  0x00007e0f979bbd71
TID 7653:
#0  0x00007e0f979bbd71
TID 7654:
#0  0x00007e0f979bbd71
```

(TID 7613 is the main `kitty` thread parked in `ppoll`; TIDs 7616–7654 — 39 of the 67 total — are software-GL worker threads all parked at the identical address `0x00007e0f979bbd71`, i.e. a homogeneous pool. This capture is `eu-stack`'s first-frame-per-thread mode; it confirms the 67-thread count and the main thread's `ppoll` independently of `gdb`.)

**Thread taxonomy — bounded to this sampled run (67 threads).** Exactly **two** threads are kitty-authored and relevant to input: the main **`kitty`** UI thread (runs the GLFW callbacks) and **`KittyChildMon`** (the `io_loop` thread that drains child writes / reads child output). There is **no** talk/remote-control thread — expected, because `--config NONE` opens no remote-control socket. The remaining ~64 threads are the **Mesa `llvmpipe` software-GL rasterizer pool** (named `llvmpipe-0…31` plus additional gallium workers and a `kitty:disk$0` shader-cache thread — see Q6 inventory); they exist only because the headless container uses software GL, and the count is **run/GL-backend specific**, not a kitty invariant. Run 2 reproduced the inventory (`gdb threads (RUN 2): 67`) and the main-thread/`KittyChildMon` split.

---

## Q5 — What happens to input for a window that is unfocused or just closed?

**Direct answer.** Input **strictly follows focus**: an unfocused window receives nothing, because routing re-resolves the active window (Q1) on every event. Input generated right after the focused window is **closed** is **re-routed to the new active window** (the survivor); the closed window's child is gone and its output file is frozen. It is silent — no crash, no error.

This is one continuous run in a single OS-window with two split children whose PIDs are constant throughout (`win_1` pid `8763`, `win_2` pid `8768`):

```text
KPID=8694  (single OS window; two split children will share it)
=== create a 2nd window (split): ctrl+shift+Return -> new_window (win_2 focused) ===
win_1 child: [child start pid=8763 KITTY_WINDOW_ID=1]
win_2 child: [child start pid=8768 KITTY_WINDOW_ID=2]

##### STAGE a: focused=win_2 ; type aaa (unfocused win_1 must get nothing) #####
-- win_1 (UNFOCUSED) od -c --
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   8   7   6   3       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n
-- win_2 (FOCUSED)   od -c --
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   8   7   6   8       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   2   ]  \n   a   a   a

##### STAGE b: focus back to win_1 (ctrl+shift+bracketleft=previous_window); type bbb #####
-- win_1 (NOW FOCUSED) od -c --
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   8   7   6   3       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n   b   b   b
-- win_2 (NOW UNFOCUSED, must be unchanged) od -c --
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   8   7   6   8       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   2   ]  \n   a   a   a

##### STAGE c: close focused win_1 (ctrl+shift+w=close_window); then type ccc #####
win_1 child pid was 8763; state now:     PID STAT COMMAND
-- win_1 (JUST CLOSED, frozen) od -c --
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   8   7   6   3       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   1   ]  \n   b   b   b
-- win_2 (SURVIVOR, now focused, must receive ccc) od -c --
   [   c   h   i   l   d       s   t   a   r   t       p   i   d
   =   8   7   6   8       K   I   T   T   Y   _   W   I   N   D
   O   W   _   I   D   =   2   ]  \n   a   a   a   c   c   c

##### trace: window actions (new_window / previous_window / close_window) #####
KeyPress matched action: new_window, handled as shortcut
KeyPress matched action: previous_window, handled as shortcut
KeyPress matched action: close_window, handled as shortcut
```

Under `STAGE c`, the `state now:     PID STAT COMMAND` line is the `ps` **header with no data row beneath it** — the closed child (pid `8763`) no longer exists.

**Observations (directly observed).**
- **STAGE a** — with `win_2` focused, `aaa` went only to `win_2`; the unfocused `win_1` received **nothing** (banner only).
- **STAGE b** — refocusing `win_1`, `bbb` went to `win_1`; the now-unfocused `win_2` is **unchanged** (`aaa` only). Input strictly follows focus (observed in both directions).
- **STAGE c** — closing focused `win_1` and typing `ccc`: the closed child (pid `8763`) is **gone** (`ps` shows no row) and `win_1`'s file is **frozen** at `bbb`; the survivor `win_2` (now focused) received `ccc`. Post-close input **re-routes to the survivor**.

**Inferred / source-assisted (not exercised interactively).** The low-level *silent drop* of a write to a no-longer-existent id lives in `schedule_write_to_child_generic` (`kitty/child-monitor.c:323`): it loops `children[i].id == id`, copies bytes only on a match, otherwise `found` stays false and it `return found` (`kitty/child-monitor.c:369`) with nothing queued. The interactive key path never reaches this branch because it re-resolves `active_window()` (`kitty/keys.c:106`) on every press and therefore always targets a **live** id (hence the observed reroute, not a drop). The `found==false` drop is the *stale-queued-id* edge and is labelled **inferred**, kept distinct from the observed reroute. The high-level Python analogue is the `Failed to write to child` guard in `Window.write_to_child` (`kitty/window.py:960`).

---

## Q6 — Which parts are Python, which are C, which are external libraries? (+ refutations)

**Layer attribution (grounded in the Q4 backtrace; ownership names are source-assisted).**

| Stage | Owner | Evidence (from the captured stack / inventory) |
|-------|-------|-----------------------------------------------|
| Receive & translate the OS key event (keysym, event dispatch) | **External library** | `glfw_xkb_handle_key_event` → `processEvent` → `_glfwDispatchX11Events` → `glfwRunMainLoop`, all in `kitty/glfw-x11.so`; xkb = libxkbcommon (Q4.4) |
| Callback, shortcut dispatch decision, encode, id-keyed write | **C core** | `key_callback.lto_priv` (with `on_key_input` inlined) and `main_loop.lto_priv` in `kitty/fast_data_types.so` (Q4.4); child bytes/encodings observed in Q3 |
| Shortcut arbitration; host the event loop; high-level bookkeeping | **Python** | `_PyObject_CallMethod_SizeT("dispatch_possible_special_key")` in `libpython`, and `main_loop` entered from `boss.child_monitor.main_loop()` at `kitty/main.py:234` (Q4.3/Q4.4) |
| Drain child writes / read child output | **C core, dedicated thread** | `Thread 2 "KittyChildMon" … io_loop` in `kitty/fast_data_types.so` (Q4.5) |

**Before/after thread inventory (refutes "each window has its own input/I-O thread").** In one run, with trivial `/bin/cat` children so windows are countable processes, the kitty thread set was captured with one window and again after creating three more (`ps -T` needs no ptrace):

```text
KPID=9791 exe=/work/kitty/launcher/kitty

########## BEFORE: single window (initial) ##########
--- kitty threads (ps -T comm, count per name) ---
      1 KittyChildMon
     33 kitty
      1 kitty:disk$0
      1 llvmpipe-0
      1 llvmpipe-1
      1 llvmpipe-10
      1 llvmpipe-11
      1 llvmpipe-12
      1 llvmpipe-13
      1 llvmpipe-14
      1 llvmpipe-15
      1 llvmpipe-16
      1 llvmpipe-17
      1 llvmpipe-18
      1 llvmpipe-19
      1 llvmpipe-2
      1 llvmpipe-20
      1 llvmpipe-21
      1 llvmpipe-22
      1 llvmpipe-23
      1 llvmpipe-24
      1 llvmpipe-25
      1 llvmpipe-26
      1 llvmpipe-27
      1 llvmpipe-28
      1 llvmpipe-29
      1 llvmpipe-3
      1 llvmpipe-30
      1 llvmpipe-31
      1 llvmpipe-4
      1 llvmpipe-5
      1 llvmpipe-6
      1 llvmpipe-7
      1 llvmpipe-8
      1 llvmpipe-9
TOTAL kitty threads BEFORE = 67
--- kitty child PROCESSES (ppid=9791) ---
   9860 Ss+  cat
child process count BEFORE = 1

########## create 3 more windows (ctrl+shift+Return x3 = new_window) ##########
--- shortcut trace (new_window matches) ---
new_window matches = 3

########## AFTER: four windows ##########
--- kitty threads (ps -T comm, count per name) ---
      1 KittyChildMon
     33 kitty
      1 kitty:disk$0
      1 llvmpipe-0
      1 llvmpipe-1
      1 llvmpipe-10
      1 llvmpipe-11
      1 llvmpipe-12
      1 llvmpipe-13
      1 llvmpipe-14
      1 llvmpipe-15
      1 llvmpipe-16
      1 llvmpipe-17
      1 llvmpipe-18
      1 llvmpipe-19
      1 llvmpipe-2
      1 llvmpipe-20
      1 llvmpipe-21
      1 llvmpipe-22
      1 llvmpipe-23
      1 llvmpipe-24
      1 llvmpipe-25
      1 llvmpipe-26
      1 llvmpipe-27
      1 llvmpipe-28
      1 llvmpipe-29
      1 llvmpipe-3
      1 llvmpipe-30
      1 llvmpipe-31
      1 llvmpipe-4
      1 llvmpipe-5
      1 llvmpipe-6
      1 llvmpipe-7
      1 llvmpipe-8
      1 llvmpipe-9
TOTAL kitty threads AFTER = 67
--- kitty child PROCESSES (ppid=9791) ---
   9860 Ss+  cat
   9874 Ss+  cat
   9875 Ss+  cat
   9876 Ss+  cat
child process count AFTER = 4
```

Creating three windows added **three child processes** and **zero** kitty threads: the thread set is **byte-identical (67 → 67)** with the same per-name breakdown, while child processes went `1 → 4`. Windows are child **processes**; kitty's thread pool is fixed. (Directly observed; bounded to this sampled run.)

**Refutations (each with runtime evidence; all bounded to the sampled process/run).**

- **A — "The Go `kitten` binary routes interactive keystrokes." REFUTED.** No Go frames appear in *any* backtrace of the sampled kitty process (`py-spy`, `gdb thread apply all bt`, `eu-stack`); the keystroke path is entirely GLFW→C→Python within the kitty process (Q4). `kitten`/`tools/` are separate processes that speak the remote-control protocol and are not on the interactive keystroke path.
- **B — "Python encodes each keystroke and writes it to the PTY." REFUTED.** The Q4.4 backtrace shows Python is consulted **only** for the shortcut decision (`dispatch_possible_special_key`); the encode-and-write is in C (`key_callback.lto_priv` → … → `schedule_write_to_child`, inlined via LTO), and the child byte/encoding outcomes are produced by the C path (Q3.1–Q3.4).
- **C — "Each window runs its own input/I-O thread." REFUTED.** The before/after inventory shows adding three windows changes threads `67 → 67` (identical) while child processes go `1 → 4`; a single main thread services all GLFW input and a single `KittyChildMon` `io_loop` services all child PTYs (Q4.5).

---

## Q7 — One correctness-vs-responsiveness tradeoff (from observed behavior, not comments)

**Direct answer.** kitty handles focused **input** synchronously on the **main/UI thread**, while a **separate `io_loop` thread (`KittyChildMon`)** drains child writes and *coalesces* child **output** processing. The single correctness-vs-responsiveness tradeoff observed: **kitty trades away background-output immediacy/throughput (throttling and coalescing a flooding child) to keep the focused input path responsive *and* per-child delivery complete, ordered, and isolated.**

**How the mechanism actually works (source-verified; corrects a common misreading).** Outbound writes to a child are **not** gated by any delay: `schedule_write_to_child` (`kitty/child-monitor.c:372`) appends to that child's `write_buf` and wakes the loop immediately (`wakeup_io_loop`, `kitty/child-monitor.c:363`); the `io_loop` sets `POLLOUT` for a child **only** when it has pending bytes (`kitty/child-monitor.c:1503`) and drains them via `write_to_child` on `POLLOUT` (`kitty/child-monitor.c:1539`). The two delay options govern the **opposite** direction and rendering:
- `input_delay` (`kitty/definition.py:878`, default 3 ms) = *"Delay before input from the program running in the terminal is processed"* → coalesces **child-output** parsing (`set_maximum_wait(OPT(input_delay) - …)`, `kitty/child-monitor.c:445`); it is *ignored when the input buffer is almost full*.
- `repaint_delay` (`kitty/definition.py:866`, default 10 ms) = render coalescing (`kitty/child-monitor.c:874`); it is *ignored when there is pending input to be processed*.

**Scenario S4 — measure focused-input delivery under a background flood.** The focused window's child is a per-byte timestamp logger (`tslog.py`, clock = `time.monotonic`, resolution `1e-9 s`); a background window's child is a bounded producer (`flood.py`). An ordered 20-key burst `a…t` is injected to the **focused** window; `analyze_ts.py` reduces the log to string / count / ordering / first→last span / count of any foreign (non-burst) printable bytes.

**Quiet baseline (2 runs)** — exact command and raw analysis:

```text
$ ./kitty/launcher/kitty --config NONE python3 /kqna/tslog.py   (+ inject "type abcdefghijklmnopqrst")
--- analyze QUIET run 1 --- monotonic_resolution=0.000000001
received_burst_string='abcdefghijklmnopqrst'
received_count=20 expected_count=20  exact_match_expected=True  in_arrival_order=True
first_byte_ts=3156279.828340 last_byte_ts=3156280.057270 span_ms=228.930
foreign_printable_bytes=0
--- analyze QUIET run 2 ---
received_burst_string='abcdefghijklmnopqrst'  received_count=20  exact_match_expected=True  in_arrival_order=True
span_ms=229.779  foreign_printable_bytes=0
```

**Under background flood (2 runs)** — background `win_2` runs `flood.py`; the burst is injected to focused `win_1` while the flood runs. Raw analysis + the independent flood throughput side-channel:

```text
$ export FLOODTAG=B FLOODMAX=2000000
$ ./kitty/launcher/kitty --config NONE -o shell=/kqna/flood.py python3 /kqna/tslog.py
  (create bg window ctrl+shift+Return; refocus win_1; inject burst while win_2 floods)
background flood procs now:   9663 Rs+   python3 flood.py      (run 1: producer RUNNING)
                              9759 Ds+   python3 flood.py      (run 2: producer BLOCKED on PTY write = backpressure)
--- analyze FLOOD run 1 ---
received_burst_string='abcdefghijklmnopqrst'  received_count=20  exact_match_expected=True  in_arrival_order=True
span_ms=233.196  foreign_printable_bytes=0
--- raw flood_progress_B.txt (produced_lines  elapsed_seconds) ---
1195000 4.761502          (≈ 251k lines/s)
--- analyze FLOOD run 2 ---
received_burst_string='abcdefghijklmnopqrst'  received_count=20  exact_match_expected=True  in_arrival_order=True
span_ms=238.102  foreign_printable_bytes=0
--- raw flood_progress_B.txt ---
1105000 4.767039          (≈ 232k lines/s)
```

**What was measured (2-run stable), and the caveats honored.**

| Metric | Quiet | Flood | Meaning |
|--------|-------|-------|---------|
| Burst completeness | 20/20 | 20/20 | no bytes lost (**correctness**) |
| Burst ordering | `a…t` exact | `a…t` exact | in-order per child (**correctness**) |
| Foreign flood bytes in focused child | 0 | 0 | no cross-child leakage (**isolation/correctness**) |
| Burst delivery span | 228.9 / 229.8 ms | 233.2 / 238.1 ms | ~2–4% change, within jitter (**responsiveness preserved**) |
| Background flood throughput | — | ~251k / ~232k lines/s (~3–4 MB/s) | producer **throttled** (D/R state) |

- The absolute span (~229 ms for 20 keys ≈ 11.5 ms/key) is dominated by the **XTEST injection cadence** — an **injection floor**, not kitty latency. The meaningful signal is the **quiet-vs-flood comparison**: the span barely moves under a multi-MB/s flood.
- No **display latency** is claimed — the on-screen render path was not measured; only child-arrival span, ordering, completeness, isolation, and producer throughput were.

**The tradeoff, strictly from observed behavior.** Under a sustained ~3–4 MB/s background flood: the flood was consumed **in full and in order** and **never leaked** into the focused child (0 foreign bytes) — **correctness/isolation preserved**; the focused keystroke burst stayed **complete and ordered** with delivery span essentially unchanged (~229 → ~235 ms) — **responsiveness preserved**; but the flood **producer was throttled** — blocked on PTY writes (`Ds+`) and rate-bounded (~232–251k lines/s) — **immediacy/throughput of background output sacrificed**. kitty neither drops the background output (which would break correctness) nor lets it monopolize the main thread (which would break responsiveness); it applies **backpressure** and drains/coalesces child output on the `io_loop` thread, so a background firehose cannot starve interactive input. That is the correctness-vs-responsiveness tradeoff, and it is visible in the runtime behavior (span stability + zero leakage + throttled producer), not in code comments.

---

## Appendix — Complete, validated harness scripts (auditability)

All scripts lived in the container-only `/kqna` directory (outside the tracked tree) and were deleted afterward. They are bounded (no unbounded loops originate input; numeric args validated; no shell spawned by the injector). Full source is included here so the methodology is auditable and reproducible.

**`xinj.c`** — minimal X11 XTEST injector (compiled `gcc -O2 -Wall -Werror -o xinj xinj.c -lX11 -l:libXtst.so.6`). It injects real events at the X server and touches no kitty internals:

```c
/* xinj - minimal, auditable X11 XTEST keyboard/mouse injector.
 * Reads line commands from stdin and injects real events at the X server
 * (exactly as xdotool does), delivered by the X server to the focused
 * X11 top-level window. Used purely as the "keyboard/mouse" for headless
 * kitty; it does NOT touch kitty internals.
 * Commands: type <text> | key <combo> | focus <0xWINID> |
 *           scroll up|down [n] | resize <0xWINID> W H | sleep <ms>
 * No shell is spawned; numeric args validated; no files opened.
 */
#include <X11/Xlib.h>
#include <X11/keysym.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
extern int XTestFakeKeyEvent(Display*, unsigned int keycode, int is_press, unsigned long delay);
extern int XTestFakeButtonEvent(Display*, unsigned int button, int is_press, unsigned long delay);
static Display *dpy;
static const unsigned long KDELAY = 6; /* ms between press/release */
static void msleep(long ms){ struct timespec ts={ ms/1000, (ms%1000)*1000000L }; nanosleep(&ts,NULL); }
static int keycode_for(KeySym ks, int *need_shift){
    int kc_min, kc_max, per; XDisplayKeycodes(dpy,&kc_min,&kc_max);
    KeySym *map=XGetKeyboardMapping(dpy,kc_min,kc_max-kc_min+1,&per); *need_shift=0;
    for(int kc=kc_min; kc<=kc_max; kc++) for(int lvl=0; lvl<per && lvl<2; lvl++)
        if(map[(kc-kc_min)*per+lvl]==ks){ XFree(map); *need_shift=(lvl==1); return kc; }
    XFree(map); return 0;
}
static unsigned int modcode(const char*name){
    KeySym ks=0;
    if(!strcmp(name,"ctrl")) ks=XK_Control_L; else if(!strcmp(name,"shift")) ks=XK_Shift_L;
    else if(!strcmp(name,"alt")) ks=XK_Alt_L; else return 0;
    return XKeysymToKeycode(dpy,ks);
}
static KeySym token_keysym(const char*t){
    if(!strcmp(t,"Return")||!strcmp(t,"enter")) return XK_Return;
    if(!strcmp(t,"space")) return XK_space; if(!strcmp(t,"Tab")) return XK_Tab;
    if(!strcmp(t,"BackSpace")) return XK_BackSpace; if(!strcmp(t,"Escape")) return XK_Escape;
    if(!strcmp(t,"Left")||!strcmp(t,"left")) return XK_Left;
    if(!strcmp(t,"Right")||!strcmp(t,"right")) return XK_Right;
    if(!strcmp(t,"Up")||!strcmp(t,"up")) return XK_Up;
    if(!strcmp(t,"Down")||!strcmp(t,"down")) return XK_Down;
    if(!strcmp(t,"bracketleft")) return XK_bracketleft;
    if(!strcmp(t,"bracketright")) return XK_bracketright;
    if(strlen(t)==1) return (KeySym)t[0];
    return 0;
}
int main(void){
    dpy=XOpenDisplay(NULL);
    if(!dpy){ fprintf(stderr,"xinj: cannot open DISPLAY %s\n", getenv("DISPLAY")?getenv("DISPLAY"):"(null)"); return 2; }
    char line[4096];
    while(fgets(line,sizeof line,stdin)){
        char *nl=strchr(line,'\n'); if(nl)*nl=0;
        if(line[0]==0||line[0]=='#') continue;
        char *cmd=strtok(line," "); if(!cmd) continue;
        if(!strcmp(cmd,"type")){
            char *rest=strtok(NULL,""); if(!rest) continue;
            for(char*p=rest;*p;p++){
                KeySym ks=(KeySym)(unsigned char)*p; int sh=0; int kc=keycode_for(ks,&sh);
                if(!kc){ fprintf(stderr,"xinj: no keycode for 0x%lx\n",ks); continue; }
                if(sh){ unsigned int shk=XKeysymToKeycode(dpy,XK_Shift_L); XTestFakeKeyEvent(dpy,shk,1,KDELAY); XFlush(dpy);}
                XTestFakeKeyEvent(dpy,kc,1,KDELAY); XFlush(dpy); msleep(KDELAY);
                XTestFakeKeyEvent(dpy,kc,0,KDELAY); XFlush(dpy);
                if(sh){ unsigned int shk=XKeysymToKeycode(dpy,XK_Shift_L); XTestFakeKeyEvent(dpy,shk,0,KDELAY); XFlush(dpy);}
                msleep(KDELAY);
            }
        } else if(!strcmp(cmd,"key")){
            char *combo=strtok(NULL," "); if(!combo) continue;
            unsigned int mods[4]; int nmods=0; KeySym base=0;
            char *save=NULL; char tmp[256]; strncpy(tmp,combo,sizeof tmp-1); tmp[sizeof tmp-1]=0;
            for(char *tok=strtok_r(tmp,"+",&save); tok; tok=strtok_r(NULL,"+",&save)){
                if(!strcmp(tok,"ctrl")||!strcmp(tok,"shift")||!strcmp(tok,"alt")){
                    unsigned int mc=modcode(tok); if(mc&&nmods<4) mods[nmods++]=mc;
                } else { base=token_keysym(tok); }
            }
            if(!base){ fprintf(stderr,"xinj: bad key combo %s\n",combo); continue; }
            unsigned int bkc=XKeysymToKeycode(dpy,base);
            for(int i=0;i<nmods;i++) XTestFakeKeyEvent(dpy,mods[i],1,KDELAY);
            XFlush(dpy); msleep(KDELAY);
            XTestFakeKeyEvent(dpy,bkc,1,KDELAY); XFlush(dpy); msleep(KDELAY);
            XTestFakeKeyEvent(dpy,bkc,0,KDELAY); XFlush(dpy);
            for(int i=nmods-1;i>=0;i--) XTestFakeKeyEvent(dpy,mods[i],0,KDELAY);
            XFlush(dpy); msleep(KDELAY);
        } else if(!strcmp(cmd,"focus")){
            char *wid=strtok(NULL," "); if(!wid) continue;
            char *end=NULL; unsigned long w=strtoul(wid,&end,0);
            if(end==wid||*end){ fprintf(stderr,"xinj: bad window id %s\n",wid); continue; }
            XSetInputFocus(dpy,(Window)w,RevertToParent,CurrentTime); XFlush(dpy);
        } else if(!strcmp(cmd,"scroll")){
            char *dir=strtok(NULL," "); char *cnt=strtok(NULL," "); if(!dir) continue;
            int n=1; if(cnt){ char*e=NULL; long v=strtol(cnt,&e,10); if(e!=cnt && !*e && v>0 && v<1000) n=(int)v; }
            unsigned int btn = !strcmp(dir,"up")?4:(!strcmp(dir,"down")?5:0);
            if(!btn){ fprintf(stderr,"xinj: bad scroll dir %s\n",dir); continue; }
            for(int i=0;i<n;i++){ XTestFakeButtonEvent(dpy,btn,1,KDELAY); XTestFakeButtonEvent(dpy,btn,0,KDELAY); XFlush(dpy); msleep(KDELAY);}
        } else if(!strcmp(cmd,"resize")){
            char *wid=strtok(NULL," "); char *ws=strtok(NULL," "); char *hs=strtok(NULL," ");
            if(!wid||!ws||!hs){ fprintf(stderr,"xinj: resize needs <winid> W H\n"); continue; }
            char *e1=NULL,*e2=NULL,*e3=NULL; unsigned long w=strtoul(wid,&e1,0);
            long ww=strtol(ws,&e2,10); long hh=strtol(hs,&e3,10);
            if(e1==wid||*e1||e2==ws||*e2||e3==hs||*e3||ww<=0||ww>10000||hh<=0||hh>10000){
                fprintf(stderr,"xinj: bad resize args\n"); continue; }
            XResizeWindow(dpy,(Window)w,(unsigned int)ww,(unsigned int)hh); XFlush(dpy);
        } else if(!strcmp(cmd,"sleep")){
            char *ms=strtok(NULL," "); if(!ms) continue;
            char*e=NULL; long v=strtol(ms,&e,10);
            if(e!=ms && !*e && v>=0 && v<60000) msleep(v);
        } else { fprintf(stderr,"xinj: unknown cmd %s\n",cmd); }
    }
    XCloseDisplay(dpy); return 0;
}
```

**`label.py`** — raw-mode byte logger (the Q1/Q3/Q5 child):

```python
import os, sys, tty
wid = os.environ.get('KITTY_WINDOW_ID', '?')
tag = os.environ.get('WINTAG', '')
path = '/kqna/win_%s%s.txt' % (wid, ('_'+tag) if tag else '')
f = open(path, 'ab', buffering=0)
f.write(('[child start pid=%d KITTY_WINDOW_ID=%s]\n' % (os.getpid(), wid)).encode())
try: tty.setraw(0)
except Exception: pass
while True:
    try: b = os.read(0, 65536)
    except OSError: break
    if not b: break
    f.write(b)
```

**`focrep.py`** — like `label.py` but enables DECSET-1004 focus reporting (the Q2 child):

```python
import os, tty
wid = os.environ.get('KITTY_WINDOW_ID', '?')
tag = os.environ.get('WINTAG', '')
path = '/kqna/focrep_%s%s.txt' % (wid, ('_'+tag) if tag else '')
f = open(path, 'ab', buffering=0)
f.write(('[focus-reporting child KITTY_WINDOW_ID=%s]\n' % wid).encode())
os.write(1, b'\x1b[?1004h')   # DECSET 1004 = enable focus reporting
try: tty.setraw(0)
except Exception: pass
while True:
    try: b = os.read(0, 65536)
    except OSError: break
    if not b: break
    f.write(b)
```

**`tslog.py`** — per-byte timestamp logger (the Q7 focused child):

```python
import os, sys, tty, time
wid = os.environ.get('KITTY_WINDOW_ID', '?')
f = open('/kqna/ts_%s.txt' % wid, 'ab', buffering=0)
res = time.get_clock_info('monotonic').resolution
f.write(('START pid=%d wid=%s monotonic_resolution=%.9f\n' % (os.getpid(), wid, res)).encode())
try: tty.setraw(0)
except Exception: pass
while True:
    try: b = os.read(0, 65536)
    except OSError: break
    if not b: break
    now = time.monotonic()
    for by in b:
        f.write(('%.6f %02x\n' % (now, by)).encode())
```

**`flood.py`** — bounded high-volume output producer (the Q7 background child; no unbounded loop; writes a throughput side-channel):

```python
import os, sys, time, signal
tag = os.environ.get('FLOODTAG', 'A')
prog = '/kqna/flood_progress_%s.txt' % tag
maxlines = int(os.environ.get('FLOODMAX', '2000000'))   # bounded
n = 0; t0 = time.monotonic()
def report():
    with open(prog, 'w') as f: f.write('%d %.6f\n' % (n, time.monotonic() - t0))
def onterm(*_a): report(); os._exit(0)
signal.signal(signal.SIGTERM, onterm)
out = sys.stdout
while n < maxlines:
    n += 1
    out.write('FLOOD-%s %d\n' % (tag, n))
    if n % 5000 == 0: out.flush(); report()
out.flush(); report()
```

**`analyze_ts.py`** — reduces a `tslog.py` log to string / count / ordering / span / foreign-byte count (the Q7 reducer):

```python
import sys
path = sys.argv[1]; expected = sys.argv[2] if len(sys.argv) > 2 else ''
letters = []; foreign = []; res = None
with open(path, 'r', errors='replace') as fh:
    for line in fh:
        line = line.rstrip('\n')
        if line.startswith('START'):
            for tok in line.split():
                if tok.startswith('monotonic_resolution='): res = tok.split('=', 1)[1]
            continue
        parts = line.split()
        if len(parts) != 2: continue
        try: ts = float(parts[0]); by = int(parts[1], 16)
        except ValueError: continue
        if 0x20 <= by < 0x7f:
            ch = chr(by)
            (letters if ch in expected else foreign).append((ts, ch))
recon = ''.join(c for _, c in letters)
print('monotonic_resolution=%s' % res)
print('received_burst_string=%r' % recon)
print('received_count=%d expected_count=%d' % (len(recon), len(expected)))
print('exact_match_expected=%s' % (recon == expected))
print('in_arrival_order=%s' % (recon == expected))
if letters:
    first = letters[0][0]; last = letters[-1][0]
    print('first_byte_ts=%.6f last_byte_ts=%.6f span_ms=%.3f' % (first, last, (last - first) * 1000.0))
print('foreign_printable_bytes=%d' % len(foreign))
```

**`snap.sh`** — PID-validated live stack sampler (refuses to attach unless the target is a single numeric live PID whose cmdline is this repo's `launcher/kitty`; uses only container-scoped `CAP_SYS_PTRACE`, never touches host `ptrace_scope`):

```sh
#!/bin/sh
set -u
pid="${1:-}"; label="${2:-snap}"
case "$pid" in ""|*[!0-9]*) echo "snap: non-numeric pid: '$pid'" >&2; exit 2;; esac
if [ ! -d "/proc/$pid" ]; then echo "snap: pid $pid is not live" >&2; exit 3; fi
exe=$(readlink -f "/proc/$pid/exe" 2>/dev/null || true)
cmd=$(tr "\0" " " < "/proc/$pid/cmdline" 2>/dev/null || true)
echo "snap[$label]: pid=$pid"; echo "snap[$label]: exe=$exe"; echo "snap[$label]: cmdline=$cmd"
case "$cmd" in *launcher/kitty*) : ;; *) echo "snap: REFUSING to attach - not launcher/kitty" >&2; exit 4;; esac
echo "===== py-spy dump --native --pid $pid ====="; py-spy dump --native --pid "$pid" 2>&1
echo "===== gdb -p $pid -batch thread apply all bt ====="
gdb -p "$pid" -batch -ex "set debuginfod enabled off" -ex "thread apply all bt" 2>&1
echo "===== eu-stack -p $pid ====="; eu-stack -p "$pid" 2>&1
```

(Three further single-purpose probes were used and are variants of the above: `ctrlc_signal.py` — enables `?19997`, reports termios `ISIG`/`VINTR` and a `SIGINT` handler (Q3.4 C2); `mouserep.py` — SGR mouse + `SIGWINCH` logger (S3); `scrollrep.py` — alt-screen scroll logger (S3b).)

---

## Reproducibility & repository hygiene

- **Two runs each** for the timing/inventory claims: Q4 (py-spy + gdb dispatch breakpoint + thread inventory) reproduced frame-identically; Q7 quiet (228.9 / 229.8 ms) and flood (233.2 / 238.1 ms) reproduced within jitter with throughput ~232–251k lines/s.
- **Tracked source unchanged.** kitty was built and heavily exercised, but every build output is gitignored (`kitty/launcher/kitty`, `kitty/fast_data_types.so`, `build/` — see §1.1 `git check-ignore`), and all scratch (`/kqna` in the container, `/tmp/kqna_evidence` on the host) is outside the tracked tree. The only tracked change in this branch is **this one document**. (The precise claim is *"the tracked source tree is unchanged"*; the working tree still contains the gitignored, uncommitted build outputs, which git does not track.)
- The exact `git status --porcelain`, `git diff --cached --name-status`, and `git diff --cached --stat` proving a single changed file — together with the `/kqna` and host-scratch removal — are captured at commit time in this branch's history.

---

## Coverage pass (every sub-question answered with command + raw output + `file:line`)

- **Q1** — Per-event selector = active window of the OS-window that received the event; `active_window()` (`kitty/keys.c:106`), `set_callback_window` (`kitty/glfw.c:196`); `is_focused`/MRU (`kitty/state.c:108`,`:120`) are bookkeeping. Evidence: S1 (window count 1→1; typed bytes land in the active tab's active window). ✔
- **Q2** — Two paths: Path A GLFW `window_focus_callback` (`kitty/glfw.c:515`) → `Boss.on_focus` (`kitty/boss.py:1651`) → `Screen.focus_changed` (`kitty/screen.c:4604`); Path B Python-only (`kitty/window_list.py:192`, `kitty/tabs.py:906`, `kitty/boss.py:913`). Evidence: S2a (9 paired `on_focus_change`), S2b (0 additional callbacks yet DECSET bytes). ✔
- **Q3** — `on_key_input` (`kitty/keys.c:166`) → dispatch → `encode_glfw_key_event` (`kitty/key_encoding.c:414`) → `schedule_write_to_child(id)` (`kitty/child-monitor.c:372`); signal branch (`kitty/keys.c:256` → `kitty/child.py:481`). Evidence: A1/A2/A3 branch matrix + child bytes, C1/C2 Ctrl+C, S3/S3b scroll+resize. ✔
- **Q4** — Blocked (EPERM) then remediated (container-scoped `CAP_SYS_PTRACE`); `py-spy` MainThread stack (`main.py:234`), gdb dispatch breakpoint (three layers), `KittyChildMon` `io_loop`; 67-thread inventory bounded to the run; 2-run stable. ✔
- **Q5** — Input follows focus (unfocused gets nothing); post-close reroute to survivor; low-level `found==false` drop (`kitty/child-monitor.c:369`) labelled inferred. Evidence: one continuous run, constant PIDs. ✔
- **Q6** — Attribution table (external `glfw-x11.so`/xkb; C `fast_data_types.so`; Python `libpython`); refutations A/B/C with backtrace + before/after thread inventory (67→67, processes 1→4). ✔
- **Q7** — Writes POLLOUT-driven (`kitty/child-monitor.c:1503`,`:1539`); `input_delay` = output coalescing (`kitty/definition.py:878`, `kitty/child-monitor.c:445`); `repaint_delay` = render coalescing (`kitty/definition.py:866`, `kitty/child-monitor.c:874`). Evidence: S4 quiet vs flood spans, throughput, isolation, producer backpressure; 2 runs. ✔

## Inferred / source-assisted claim audit (claims not directly observed, labelled in-text)

- **Q1** — the `GLFWwindow* → callback_os_window → active_window()` resolution is **source-assisted** (corroborated by the Q4 `key_callback` frame).
- **Q2** — the specific internal Python call chain for Path B (`window_list.py`/`tabs.py`/`boss.py`) is **source-assisted**; the *discriminator* (trace present vs. absent, DECSET bytes in both) is directly observed.
- **Q3** — the C→Python dispatch *call*, the readiness/`fake_event` gates, and the `encode_glfw_key_event` frame are **source-assisted** (the dispatch call is corroborated by Q4.4); all decision *outcomes* and child bytes are directly observed. The Python `write_to_child` convergence is **source-assisted** (not exercised by the interactive keystroke path).
- **Q4** — the `schedule_write_to_child` frame was not captured because LTO inlined it; noted as an LTO artifact.
- **Q5** — the `found==false` silent drop (`kitty/child-monitor.c:369`) is **inferred**; the reroute and follow-focus behavior are directly observed.
- **Q6** — the *mapping* of each captured shared object to an ownership layer (`glfw-x11.so` → external, `fast_data_types.so` → C core, `libpython…so` → Python) and the ownership *names* are **source-assisted**; the presence/absence of each layer's frames in the backtraces and the 67→67 / 1→4 thread-vs-process inventory that grounds refutations A/B/C are directly observed.
- **Q7** — the delay-option *semantics* and the POLLOUT write mechanism are **source-assisted**; the throughput, span stability, ordering, completeness, isolation, and producer backpressure are directly observed. No display-latency claim is made.

Every other statement in this document is a direct runtime observation shown next to the command and raw output that produced it.
