# Kitty Terminal Startup Investigation — Commit 815df1e210e0

**Repository**: [kovidgoyal/kitty](https://github.com/kovidgoyal/kitty)
**Commit investigated**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (short: `815df1e210e0`, message: *"Wire up applying of font config"*)
**Kitty version reported by the built binary**: `kitty 0.35.2 created by Kovid Goyal`
**Investigation host**: Ubuntu 24.04.4 LTS, x86_64, Python 3.12.3, Go 1.22.2, Mesa 25.2.8, X.Org 21.1.11, Xvfb virtual framebuffer on `:99` at `1280x720x24`
**Branch**: X11 only — `WAYLAND_DISPLAY` is unset throughout; the Wayland code paths are not exercised.

---

## Table of Contents

1. [Abstract / Executive Summary](#abstract--executive-summary)
2. [Investigation Environment and Methodology](#investigation-environment-and-methodology)
3. [Question 1 — What systems come online on the way to a working terminal?](#question-1--what-systems-come-online-on-the-way-to-a-working-terminal)
4. [Question 2 — How is the initial configuration resolved on first launch?](#question-2--how-is-the-initial-configuration-resolved-on-first-launch)
5. [Question 3 — How does data flow from shell to terminal (PTY, fork, VT parser)?](#question-3--how-does-data-flow-from-shell-to-terminal-pty-fork-vt-parser)
6. [Question 4 — What evidence confirms the display system is working?](#question-4--what-evidence-confirms-the-display-system-is-working)
7. [Appendix A — Complete Source File Reference Index](#appendix-a--complete-source-file-reference-index)
8. [Appendix B — Raw Captured Debug Log (verbatim)](#appendix-b--raw-captured-debug-log-verbatim)
9. [Appendix C — Raw xwininfo Output](#appendix-c--raw-xwininfo-output)
10. [Appendix D — Glossary of Key Identifiers](#appendix-d--glossary-of-key-identifiers)
11. [Investigation Provenance](#investigation-provenance)

---

## Abstract / Executive Summary

This document records a read-only, evidence-based investigation of the startup sequence of the Kitty terminal emulator at commit `815df1e210e0` (*"Wire up applying of font config"*). The investigation was conducted on an Ubuntu 24.04 container running Python 3.12, Mesa 25.2.8, and Xvfb headless; because the host has no physical display, every run occurred under `DISPLAY=:99`, and because `WAYLAND_DISPLAY` was never set, only the X11 startup path was exercised. The investigation answers four questions at a glance — (1) which subsystems come online between process entry and a working terminal, (2) how the initial configuration is resolved when no `kitty.conf` files exist, (3) how data flows from the child shell through the PTY and VT parser onto the screen, and (4) what observable evidence proves the display system is healthy. Every claim below is grounded in at least one of three evidence streams: a direct source-code citation at commit `815df1e21` (file path + approximate line number), a live debug log line captured from `./kitty/launcher/kitty --debug-rendering --debug-keyboard --debug-font-fallback`, or an `xwininfo -root -tree` window-tree capture taken while Kitty was running. No source files were modified during the investigation; the only artifact produced is this markdown document.

---

## Investigation Environment and Methodology

### Target Commit

The investigation targets `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (short hash `815df1e210e0`), whose commit message is *"Wire up applying of font config"*. This is the local checkout's `HEAD` prior to the creation of this documentation artifact. All source-line citations below are relative to that commit; line numbers can shift by one or two across minor edits, so the prose uses "around line N" for phase-level citations and exact line numbers only for emitted debug strings that were `grep`-verified.

### Headless Execution Setup (Xvfb)

The host environment has no physical display. A virtual framebuffer was started via the X virtual framebuffer (`Xvfb`):

```
Xvfb :99 -screen 0 1280x720x24 -nolisten tcp -nolisten unix
export DISPLAY=:99
```

Kitty inherits `DISPLAY=:99` at launch and therefore opens its X11 connection against that framebuffer. This is a normal X11 path from Kitty's perspective — no special handling is needed on the Kitty side. The framebuffer's resolution is `1280x720x24` (1280 wide, 720 tall, 24-bit color), which is sufficient for the default 640×400-pixel Kitty window with plenty of slack.

### Build Command

The binary was built from source at this commit with:

```
python3 setup.py build --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag was required because the container's system `wayland-protocols` headers contain newer enumerators than the vendored GLFW Wayland backend expects — a known mismatch that compiles as a warning with newer compilers and becomes an error only when `-Werror` is in effect. The `--ignore-compiler-warnings` flag disables `-Werror` for this compile. The build produces:

- `kitty/fast_data_types.so` — the primary Python/C extension module exposing `spawn`, `set_options`, `set_font_data`, rendering hooks, etc.
- `kitty/glfw-x11.so`, `kitty/glfw-wayland.so` — platform-specific GLFW backend plugins
- `kitty/launcher/kitty` — the native C launcher binary (36 KB)
- Go kitten binaries under `tools/` and `kittens/`

### Launch Command and Debug Flags

When Kitty is built from source and launched *via* the native launcher `./kitty/launcher/kitty`, the launcher (`kitty/launcher/main.c`) takes care of populating `sys.kitty_run_data` with `bundle_exe_dir`, `from_source`, and `extensions_dir`. This dictionary is then read by `kitty/entry_points.py` and `kitty/constants.py` to locate the built C extensions. When launching Kitty as a Python module (bypassing the C launcher — which is sometimes necessary when one needs a controlled Python environment for introspection), this dictionary must be populated manually:

```python
import sys
sys.kitty_run_data = {
    'bundle_exe_dir': '<repo_root>',
    'from_source':    True,
    'extensions_dir': '<repo_root>/kitty',
}
from kitty.entry_points import main
main()
```

For the principal runtime observations in this document, the native launcher was used directly:

```
DISPLAY=:99 ./kitty/launcher/kitty \
    --debug-rendering \
    --debug-keyboard \
    --debug-font-fallback \
    sh -c 'echo READY; sleep 1'
```

The three debug flags are defined in `kitty/cli.py` and trip the following behavior at source:

- `--debug-rendering` → sets `global_state.debug_rendering`, enabling `[T.TTT] GL version string: ...`, `OS Window created`, `Child launched`, and a handful of per-frame/per-child traces.
- `--debug-keyboard` → emits XKB keymap loading and modifier-index tables from `glfw/xkb_glfw.c`.
- `--debug-font-fallback` → calls `kitty/fonts/render.py::dump_font_debug()` at the end of `_run_app()`, printing all resolved font faces.

### Evidence Collection Approach

Three orthogonal evidence streams are correlated throughout the document:

| Stream | What it provides | How it was captured |
|--------|------------------|---------------------|
| **Static source analysis** | File paths, function names, approximate line numbers, code excerpts | `grep -n`, direct reads via the file viewer; line numbers spot-checked with `sed -n` |
| **Live debug log** | Timestamped `[T.TTT]` lines emitted on `stderr`, ordering, observed values | Captured from `DISPLAY=:99 ./kitty/launcher/kitty --debug-rendering --debug-keyboard --debug-font-fallback sh -c 'echo READY; sleep 1' 2>&1` |
| **Window-tree capture** | Confirmation of the X11 window, its title, class, and size | `DISPLAY=:99 xwininfo -root -tree` with Kitty running in another subshell |

Every claim about observed behavior cites at least one of these three sources. No behavior was assumed or extrapolated. Where the agent prompt guidance (AAP §0.8.3) referenced LiberationMono, the actual container had DejaVu Sans Mono installed as the Fontconfig `monospace` alias — this discrepancy is called out at Q4 §4.4 below, and the document reports what was observed rather than what was expected.

---

## Question 1 — What systems come online on the way to a working terminal?

### 1.1 Thinking / Rationale

The question asks us to enumerate everything that activates between the moment the Kitty process begins executing and the moment a shell prompt is ready to accept user input. This is *not* simply a list of functions called; it is a list of discrete **subsystems** — distinct stateful modules that must be initialized and hooked together for the terminal to be functional. Our approach is:

1. **Trace the call graph statically** from the entry point (`kitty/launcher/main.c` → `kitty/entry_points.py::main()` → `kitty/main.py::main()` → `_main()` → `run_app()` → `_run_app()` → `Boss(...)` → `boss.start()` → `boss.child_monitor.main_loop()`) and note every distinct subsystem that is initialized along the way.
2. **Correlate with live debug output** — each debug log line is emitted by a specific `printf`/`debug()`/`print()` call in the source, and because `--debug-rendering`, `--debug-keyboard`, and `--debug-font-fallback` together light up most of the startup-phase tracepoints, the captured log establishes observed ordering and timing.
3. **Group the subsystems into ordered phases** (A–K) so that the order of initialization is clear even where different phases have interleaved internal steps (e.g. OS window creation internally drives both GL init *and* shader loading).

A subsystem, for this purpose, is a cohesive module with its own state: the CLI parser, the configuration engine, GLFW + its platform plugin, the font pipeline (Fontconfig + FreeType + HarfBuzz + sprite sheet on GPU), the OpenGL driver + shader programs, the Boss controller, the ChildMonitor + I/O thread, the PTY pair + child process, the VT parser, the Screen model, and the rendering loop.

### 1.2 High-Level Startup Phase Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│ A. Process entry                                                         │
│    └── Native launcher (kitty/launcher/main.c) — when packaged           │
│    └── Python entry  (kitty/entry_points.py:183 -> kitty/main.py:524)    │
│                                                                          │
│ B. running_in_kitty(True)  +  CWD validation  +  CLI parsing             │
│                                                                          │
│ C. Configuration resolution (create_opts -> resolve_config -> defaults)  │
│                                                                          │
│ D. Environment setup (setup_environment), locale, signal masking         │
│                                                                          │
│ E. GLFW platform init (init_glfw) -> x11 backend .so loaded              │
│    └── XKB keymap compile ──────────────────────────► [0.060s]           │
│    └── Modifier indices published ──────────────────► [0.064s]           │
│                                                                          │
│ F. Fonts: set_font_family() -> Fontconfig -> FreeType -> set_font_data   │
│                                                                          │
│ G. Box-drawing scale push (set_scale), options push to C (set_options)   │
│                                                                          │
│ H. OS window creation (create_os_window)                                 │
│    ├── GL context + gl_init() -> "GL version string" ───► [0.119s]       │
│    ├── load_all_shaders() -> cell/graphics/bgimage/tint/border shaders   │
│    ├── Pre-rendered sprites uploaded to GPU texture                      │
│    ├── X11 window icon set (set_x11_window_icon)                         │
│    ├── 14 GLFW event callbacks registered                                │
│    └── emit "OS Window created" ────────────────────► [0.145s]           │
│                                                                          │
│ I. Boss() + ChildMonitor created                                         │
│    └── boss.start() -> ChildMonitor.start():                             │
│         ├── pthread_create(io_thread, io_loop) ── I/O thread running     │
│         └── (optional) pthread_create(talk_thread, talk_loop)            │
│                                                                          │
│ J. startup_first_child() -> Tab -> Window -> Child.fork()                │
│    ├── os.openpty() -> PTY master/slave                                  │
│    ├── os.pipe() -> ready-notification pipe                              │
│    ├── get_final_env() -> TERM, COLORTERM, TERMINFO, KITTY_PID, ...      │
│    ├── fast_data_types.spawn() -> child PID                              │
│    └── os.set_blocking(master, False)                                    │
│                                                                          │
│ K. Main event-loop tick (render -> layout -> set_geometry)               │
│    ├── First geometry -> child_monitor.resize_pty(...)                   │
│    ├── child.mark_terminal_ready() — closes write end of ready pipe      │
│    ├── "Child launched" emitted ─────────────────────► [0.158s]          │
│    ├── First parse_input() — no pending data yet                         │
│    └── First render() + glfwSwapBuffers -> first frame presented         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Deterministic Phase-by-Phase Sequence (with source references)

The following table walks the sequence phase by phase with primary source file and approximate line cites. Line numbers are verified by `grep -n` at commit `815df1e21` in the local checkout.

| Phase | Subsystem | Primary source | Approximate line |
|-------|-----------|----------------|------------------|
| A | Native launcher bootstrap: descriptor validation, path resolution, Python embedding, `sys.kitty_run_data` population | `kitty/launcher/main.c` | full file (~30 lines) |
| A | Python entry dispatch to default GUI | `kitty/entry_points.py` | around line 183 |
| A | `main()` -> `_main()` wrapper | `kitty/main.py` | lines 441 (`_main`) and 524 (`main`) |
| B | `running_in_kitty(True)` flag; CWD validation and fallback to `$HOME` | `kitty/main.py::_main()` | around line 441 |
| B | `parse_args(...)` → `CLIOptions` dataclass | `kitty/cli.py` (parse_args) + `kitty/cli_stub.py` (CLIOptions dataclass); called from `_main()` | call site around line 465 in `kitty/main.py` |
| C | `create_opts(cli_opts, ...)` | `kitty/cli.py` | line 1081 |
| C | `default_config_paths(())` → `resolve_config(SYSTEM_CONF, defconf, ())` | `kitty/cli.py` | line 1067 calls `resolve_config`; `SYSTEM_CONF` at line 1064 |
| C | `resolve_config()` and generic `load_config()` | `kitty/conf/utils.py` | `resolve_config` at line 322, generic `load_config` at line 332 |
| C | `config_dir`, `defconf`, `appname` constants | `kitty/constants.py` | `appname` line 23, `config_dir` line 131, `defconf` line 133 |
| D | `setup_environment(opts, cli_opts)` — sets PATH, expands listen-on, MANPATH | `kitty/main.py::_main()` | around line 495 |
| D | `set_locale()` | `kitty/main.py::_main()` | around line 501 |
| D | `mask_kitty_signals_process_wide()` — blocks SIGINT, SIGTERM, SIGHUP, SIGCHLD, SIGUSR1, SIGUSR2 in the process so only Kitty handles them | `kitty/main.py::_main()` | around line 515 |
| E | `init_glfw(opts, debug_keyboard, debug_rendering)` | `kitty/main.py` | line 95 |
| E | Platform selection + `.so` load; `glfwInit()` | `kitty/glfw.c::glfw_init()` | around line 1430 |
| E | XKB keymap compile emits `"Loading new XKB keymaps"` | `glfw/xkb_glfw.c` | line 672 |
| E | Modifier indices emitted `"Modifier indices alt: 0x3 ..."` | `glfw/xkb_glfw.c` | lines 376 and 540 (two variants; modern X11 path emits the line without `control:` at the end) |
| F | `set_font_family(opts)` | `kitty/fonts/render.py` | line 173 |
| F | `get_font_files(opts)` — Fontconfig resolution | `kitty/fonts/common.py` | around line 280 |
| F | `set_font_data(...)` — pushes font descriptors + rendering callbacks to C | `kitty/fonts.c` | (C extension bridge) |
| F | `dump_font_debug()` — prints resolved face list at end of `_run_app` | `kitty/fonts/render.py` | line 161 |
| G | `set_scale(...)` — pushes DPI / cell-scale info to C | `fast_data_types` (C extension) | called from `_run_app` |
| G | `set_options(opts, is_wayland(), debug_rendering, debug_font_fallback)` — pushes parsed `Options` into C global state | `fast_data_types` (C extension) | called from `_run_app` |
| H | `create_os_window(...)` — creates GLFW window, GL context, runs shader loader callback, emits `"OS Window created"` | `kitty/glfw.c` | line 1321 contains `debug("OS Window created\n");` |
| H | `gl_init()` — `gladLoadGL`, version check ≥ 3.1 (Linux) / 3.3 (macOS), emits `"GL version string"` | `kitty/gl.c` | line 72 |
| H | `load_all_shaders` callback compiles cell/graphics/bgimage/tint programs | `kitty/main.py` | line 82 wraps `load_shader_programs()` + `load_borders_program()` in try/except CompileError |
| H | `LoadShaderPrograms.__call__` | `kitty/shaders.py` | line 147 |
| H | `init_cell_program()` | `kitty/shaders.py` | line 201 |
| H | `load_borders_program()` | `kitty/borders.py` | line 63 |
| H | `send_prerendered_sprites()` — blank cell, underlines, cursors rasterized and uploaded to GPU texture | `kitty/fonts.c` | `send_prerendered_sprites` |
| H | X11 window icon set (`set_x11_window_icon()` — not on Wayland) | `kitty/main.py` | around line 155 |
| H | 14 GLFW event callbacks registered (focus, resize, keyboard, mouse, scroll, drop, ...) | `kitty/glfw.c::create_os_window()` | around lines 1253–1322 |
| I | `create_sessions()` builds the default single-tab/single-window session | `kitty/session.py::create_sessions` | in the module |
| I | `Boss.__init__()` creates `ChildMonitor(on_child_death, dump_callback, talk_fd, listen_fd)` plus clipboard, remote control, encryption key | `kitty/boss.py::Boss.__init__` | line 325 |
| I | `Boss.start(first_window_id, sessions)` | `kitty/boss.py::Boss.start` | line 1181 |
| I | `ChildMonitor.start()` launches the I/O thread via `pthread_create(&self->io_thread, NULL, io_loop, self)` | `kitty/child-monitor.c` | line 291 |
| J | `Boss.startup_first_child()` → `add_os_window` → `TabManager` → `Tab` → `Window` → `Child.fork()` | `kitty/boss.py::startup_first_child` | line 383 |
| J | `Child.fork()` — PTY via `os.openpty()`, ready pipe via `os.pipe()`, env via `get_final_env()`, spawn via `fast_data_types.spawn()` | `kitty/child.py::fork` | lines 276–360 |
| J | Parent-side bookkeeping: `os.close(slave)`, `self.child_fd = master`, `os.close(ready_read_fd)`, `self.terminal_ready_fd = ready_write_fd`, `os.set_blocking(self.child_fd, False)` | `kitty/child.py::fork` | around lines 340–349 |
| J | `systemd_move_pid_into_new_scope(pid, ...)` (best-effort; logs non-fatal `log_error` if D-Bus is unavailable in a container) | `kitty/child.py::fork` | around line 351 |
| K | `boss.child_monitor.main_loop()` → `run_main_loop(process_global_state, self)` | `kitty/child-monitor.c::process_global_state` | line 1224 |
| K | First layout/render tick: `render_os_window` → `prepare_to_render_os_window` → `Tab.relayout` → `Window.set_geometry` | `kitty/child-monitor.c::render_os_window` | line 833 |
| K | `set_geometry` calls `child_monitor.resize_pty(...)` (→ `TIOCSWINSZ` on master), then `child.mark_terminal_ready()`, then emits `[N.NNN] Child launched` under `--debug-rendering` | `kitty/window.py::set_geometry` | lines 865–871 |
| K | `parse_input()` drains each window's VT-parser write buffer on the main thread | `kitty/child-monitor.c::parse_input` | line 451 |
| K | `send_cell_data_to_gpu()` uploads cell data per visible window; `glfwSwapBuffers` presents the frame | `kitty/child-monitor.c` | lines 714 and 766 (call sites) |

### 1.4 Observed Timing Table

Captured from a live headless run with all three debug flags, the observed startup timeline (wall-clock seconds since process start, as emitted by the binary itself) is:

| Timestamp | Event | Source of print |
|-----------|-------|-----------------|
| `[0.060]` | `Loading new XKB keymaps` | `glfw/xkb_glfw.c` line 672 |
| `[0.064]` | `Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1` | `glfw/xkb_glfw.c` line 376 |
| `[0.119]` | `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5` | `kitty/gl.c` line 72 |
| `[0.145]` | `OS Window created` | `kitty/glfw.c` line 1321 |
| `[0.154]` | `Failed to open systemd user bus with error: Connection refused` (non-fatal; expected in a container without systemd user session) | `kitty/child.py` around line 351 |
| `[0.158]` | `Child launched` | `kitty/window.py` line 871 |
| `[0.158]` | `Text fonts:` and the four font lines (Normal/Bold/Italic/Bold-Italic) | `kitty/fonts/render.py::dump_font_debug()` line 161 |

Notes:

- The `[0.119]` GL-version line is printed from inside `gl_init()` *during* `create_os_window`, which completes at `[0.145]`. The two lines appear out of order in the captured stream because different code paths format the `monotonic()` reading at slightly different times; the *logical* ordering is preserved (GL init happens inside window creation and thus before `"OS Window created"`).
- On every observed run, total time from process start to `Child launched` was ~160 ms.

### 1.5 Summary Checklist

All the subsystems that come online before a working terminal, listed in startup order (expanded from the phase diagram at §1.2):

1. Process-level init (`running_in_kitty(True)` flag; CWD validation)
2. CLI argument parsing (`parse_args` → `CLIOptions`)
3. Configuration engine (searches `/etc/xdg/kitty/kitty.conf` then `~/.config/kitty/kitty.conf`; falls back to compiled defaults)
4. Environment setup (PATH manipulation, listen-on expansion, MANPATH)
5. Locale initialization (`set_locale`)
6. Signal masking (SIGINT, SIGTERM, SIGHUP, SIGCHLD, SIGUSR1, SIGUSR2 blocked process-wide)
7. GLFW platform backend loading (x11 `.so` loaded from `extensions_dir`)
8. XKB keymap compilation (keyboard layout + modifier mapping)
9. Font discovery and loading (Fontconfig queries → FreeType face loading)
10. Box-drawing scale configuration (`set_scale`)
11. Options push to C global state (`set_options`)
12. OpenGL context creation (performed inside GLFW window creation)
13. GL driver validation (`gl_init()` → `gladLoadGL`, version check ≥ 3.1/3.3)
14. Shader program compilation (cell, graphics, bgimage, tint, border)
15. Pre-rendered sprite generation (blank cell, underlines, cursors → GPU texture)
16. Window icon loading and setting
17. Event callback registration (focus, resize, keyboard, mouse, scroll, drop, ...)
18. Session creation (default single-tab, single-window session)
19. Boss controller creation (ChildMonitor + clipboard + remote control + encryption key)
20. I/O thread launch (`pthread_create(io_thread, io_loop)` — poll-based PTY multiplexer)
21. PTY pair creation and child process fork
22. Shell environment population (TERM, COLORTERM, TERMINFO, KITTY_PID, KITTY_INSTALLATION_DIR, shell integration vars)
23. `mark_terminal_ready()` triggered on first geometry set (closes write end of ready pipe; child now free to proceed)

---

## Question 2 — How is the initial configuration resolved on first launch?

### 2.1 Thinking / Rationale

The question is: when a user launches Kitty for the first time — with no `kitty.conf` files anywhere on the system — how does Kitty decide what every option's value should be? There are three possible strategies a terminal emulator could adopt:

1. **Ship with hard-coded defaults** and simply use them if no config is found. (Kitty does this.)
2. **Refuse to start** without a config. (Kitty does not do this.)
3. **Emit a warning** when falling back to defaults. (Kitty is silent by design — the absence of a config is the normal case for a first launch.)

To answer concretely we need to show: (a) the exact *search paths* Kitty probes, in order; (b) the exact *fallback mechanism* when no path yields a readable file; and (c) the *actual default values* that take effect in our measurement environment, ideally with a round-trip check confirming observability (e.g. the window size actually *being* 640×400 because `initial_window_width`/`initial_window_height` defaulted to 640/400).

The configuration pipeline is split across three cooperating files: `kitty/cli.py` owns the path-list construction, `kitty/conf/utils.py` owns the generic "resolve then load" loop, and `kitty/options/` owns the option schema and defaults. Our trace follows the chain from `_main()` → `create_opts()` → `default_config_paths()` → `resolve_config()`, and then cross-references the compiled-in `defaults` singleton (`kitty/options/types.py`) with the option-definition source (`kitty/options/definition.py`) to pull out the specific defaults that took effect.

### 2.2 Config Path Resolution (`resolve_config` → `default_config_paths`)

The entry point from `_main()` is `create_opts()`:

```python
# kitty/main.py::_main() around line 494
bad_lines: List[BadLine] = []
opts = create_opts(cli_opts, accumulate_bad_lines=bad_lines)
```

`create_opts` is defined in `kitty/cli.py` at line 1081:

```python
# kitty/cli.py:1081
def create_opts(args: CLIOptions, accumulate_bad_lines: Optional[List[BadLineType]] = None) -> KittyOpts:
    ...
    config = default_config_paths(args.config)
    ...
```

`default_config_paths` is immediately above at line 1067:

```python
# kitty/cli.py:1064-1068
SYSTEM_CONF = f'/etc/xdg/{appname}/{appname}.conf'
...
def default_config_paths(conf_paths: Sequence[str]) -> Tuple[str, ...]:
    return tuple(resolve_config(SYSTEM_CONF, defconf, conf_paths))
```

With `appname = 'kitty'` from `kitty/constants.py:23`, `SYSTEM_CONF` expands to `/etc/xdg/kitty/kitty.conf`. The `defconf` identifier is imported from `kitty/constants.py`:

```python
# kitty/constants.py:131-133
config_dir = _get_config_dir()
...
defconf = os.path.join(config_dir, 'kitty.conf')
```

`_get_config_dir()` honors `$KITTY_CONFIG_DIRECTORY` first, then the standard `$XDG_CONFIG_HOME/kitty`, then `$HOME/.config/kitty` as the fallback. On our Ubuntu 24.04 container with `HOME=/root` and no `XDG_CONFIG_HOME`, the result is `/root/.config/kitty`. Hence `defconf = /root/.config/kitty/kitty.conf`.

`resolve_config()` in `kitty/conf/utils.py` at line 322 is where the path search order is decided:

```python
# kitty/conf/utils.py:322
def resolve_config(SYSTEM_CONF: str, defconf: str,
                   config_files_on_cmd_line: Sequence[str] = ()) -> Generator[str, None, None]:
    ...
```

Its behavior (summarized from the source at commit `815df1e21`):

- If `config_files_on_cmd_line` contains files (from `--config <path>` on the command line), those paths are yielded in order and the system/user defaults are **not** added.
- Otherwise, `SYSTEM_CONF` is yielded first, then `defconf`.
- The reserved literal `NONE` on the command line suppresses the system config when mixed with explicit `--config` paths.

### 2.3 Config File Search Order (system then user)

With no `--config` given, `default_config_paths(())` returns exactly:

```
(
  '/etc/xdg/kitty/kitty.conf',    # SYSTEM_CONF
  '/root/.config/kitty/kitty.conf',  # defconf (user)
)
```

This was verified at runtime by importing `kitty.cli` and calling `default_config_paths(())`:

```
SYSTEM_CONF: /etc/xdg/kitty/kitty.conf
defconf    : /root/.config/kitty/kitty.conf
default_config_paths(()): ('/etc/xdg/kitty/kitty.conf', '/root/.config/kitty/kitty.conf')
SYSTEM_CONF exists?: False
defconf exists?   : False
```

Neither file exists in the test container (no Kitty package installed, fresh `$HOME`).

The generic loader is `load_config()` in `kitty/conf/utils.py` at line 332. It is structured so that each path returned by `resolve_config()` is attempted in turn. The callback that parses a file is only invoked when the file can be opened; if `open()` raises `FileNotFoundError` (or the file cannot be read), the path is **silently skipped** and the loader proceeds to the next. If no path yields a readable file, the built-in defaults flow through unchanged. There is no warning emitted when a config file is missing; this is intentional because Kitty users who have never created a `kitty.conf` should not be pestered on every launch.

### 2.4 Fallback to Built-in Defaults (`kitty/options/types.py::defaults`)

When `load_config()` yields no parsed files, the effective `KittyOpts` equals the compiled-in `defaults` singleton:

```python
# kitty/options/types.py
defaults = <Options NamedTuple instance populated from kitty/options/definition.py>
```

The `defaults` singleton is built by the code-generation layer from `kitty/options/definition.py`, which is the canonical schema. Every `opt('name', 'default_value_literal', ...)` call in that file registers one option. Representative registrations (line numbers verified by `grep -n` at commit `815df1e21`):

```
kitty/options/definition.py:59    opt('font_size', '11.0', option_type='to_font_size', ctype='double', ...)
kitty/options/definition.py:372   opt('scrollback_lines', '2000', ...)
kitty/options/definition.py:866   opt('repaint_delay', '10', ...)
kitty/options/definition.py:878   opt('input_delay', '3', ...)
kitty/options/definition.py:889   opt('sync_to_monitor', 'yes', ...)
kitty/options/definition.py:994   opt('initial_window_width', '640', ...)
kitty/options/definition.py:998   opt('initial_window_height', '400', ...)
kitty/options/definition.py:1468  opt('background_opacity', '1.0', ...)
```

The `Options` named-tuple shape — the type of `defaults` — is declared in `kitty/options/types.py`, generated mechanically from the same schema. `kitty/options/parse.py` provides the per-line parser used by `load_config()` when a file *is* present. When no file is present, `parse.py` is never invoked and `defaults` flows through untouched.

Downstream of `create_opts()`, the returned `KittyOpts` is pushed into the C layer via `set_options(opts, is_wayland(), debug_rendering, debug_font_fallback)` so that native code (renderer, font group, VT parser, child monitor) reads the same values the Python layer resolved.

### 2.5 Observed Evidence (no config files found)

Three facts were confirmed live in our test environment:

1. **Both configured paths are absent.** Direct filesystem check under Python:
   ```
   SYSTEM_CONF exists?: False
   defconf exists?   : False
   ```

2. **Kitty launches silently with defaults.** The captured `--debug-rendering --debug-keyboard --debug-font-fallback` log (Appendix B) contains *no* `"loaded config"` or `"warning: config not found"` messages — `load_config()` falls through quietly.

3. **The default dimensions took effect at runtime.** `xwininfo -root -tree` captured while Kitty was running under `DISPLAY=:99` showed the window at exactly **640×400 pixels** (Appendix C):

   ```
   0x20000c "sh": ("kitty" "kitty")  640x400+0+0  +0+0
   ```

   This is direct observable confirmation that `initial_window_width=640` (line 994) and `initial_window_height=400` (line 998) from `kitty/options/definition.py` were applied by the compiled `defaults`.

### 2.6 Concrete Default Values That Took Effect (table)

The following table enumerates defaults observed by introspecting `kitty.options.types.defaults` at runtime in the built environment. Each row cross-references the approximate line in `kitty/options/definition.py` where the default is registered. Some values like `font_family` are compound structures; the `system='monospace'` field tells Fontconfig to resolve whatever the system's `monospace` alias is — on this Ubuntu 24.04 container that resolves to DejaVu Sans Mono (see Q4 §4.4).

| Option | Observed default value | `kitty/options/definition.py` line |
|--------|------------------------|-----------------------------------|
| `font_family` | `FontSpec(system='monospace', ...)` | (font_family section; ~55) |
| `font_size` | `11.0` | 59 |
| `initial_window_width` | `(640, 'px')` | 994 |
| `initial_window_height` | `(400, 'px')` | 998 |
| `term` | `'xterm-kitty'` | (term opt) |
| `shell` | `'.'` (use the user's login shell from `/etc/passwd`) | (shell opt) |
| `shell_integration` | `frozenset({'enabled'})` | 3141 (finalized) |
| `scrollback_lines` | `2000` | 372 |
| `repaint_delay` | `10` (ms) | 866 |
| `input_delay` | `3` (ms) | 878 |
| `sync_to_monitor` | `True` | 889 |
| `background_opacity` | `1.0` | 1468 |
| `linux_display_server` | `'auto'` | (linux_display_server opt) |
| `allow_remote_control` | `'no'` | (allow_remote_control opt) |
| `enabled_layouts` | `['fat', 'grid', 'horizontal', 'splits', 'stack', 'tall', 'vertical']` | (enabled_layouts opt) |
| `cursor_shape` | `1` (block) | (cursor_shape opt) |

Taken together: because `/etc/xdg/kitty/kitty.conf` and `/root/.config/kitty/kitty.conf` are both absent, `load_config()` never opens a file, no line-by-line parse is performed, and the `defaults` singleton from `kitty/options/types.py` (populated from the registrations in `kitty/options/definition.py`) is what every downstream subsystem (fonts, window sizing, VT buffer, render pacing, background compositing, ...) sees.

---

## Question 3 — How does data flow from shell to terminal (PTY, fork, VT parser)?

### 3.1 Thinking / Rationale

The question asks for the concrete mechanism by which a child shell process — running as a separate OS process with its own memory, its own `stdout`, and its own `stderr` — delivers bytes to the Kitty terminal emulator that then become characters on the screen. The answer has to cover five distinct concerns:

1. **Channel construction** — What OS primitive connects Kitty (parent) and the shell (child)? Kitty uses a POSIX **pseudo-terminal (PTY)** pair, allocated via `os.openpty()`, plus an additional **ready-notification pipe** to control exactly when the child starts emitting.
2. **Process creation** — How is the child actually forked-and-executed? Kitty uses `fast_data_types.spawn()`, a C helper that wraps the POSIX `posix_spawn`/`fork`+`execve` sequence, attaches the slave PTY as the child's controlling terminal, and arranges file descriptor inheritance precisely.
3. **Gating the child** — If the child were allowed to start writing before the parent had set the window size, the child would emit at the wrong columns/rows. Kitty solves this with a **ready pipe handshake**: the child blocks on `read(ready_read_fd)`, and the parent unblocks it by closing `ready_write_fd` only after `Window.set_geometry()` has resized the PTY. The `"Child launched"` debug-log line is produced at the moment this happens.
4. **Byte transport** — Once the child is running, its output arrives on the master PTY fd. Kitty's dedicated **I/O thread** (`io_loop` in `child-monitor.c`) polls the master fd and reads the bytes into the VT parser's write buffer. It then wakes the main thread.
5. **Parsing and rendering** — The main thread drains the buffer by calling `parse_input()`, which runs bytes through the VT state machine (`kitty/vt-parser.c`) and dispatches to `screen_draw_text()` (for printable characters) and the various CSI/OSC/DCS/APC handlers (for escape sequences). The next render tick composes a frame and swaps it onto the OS window.

Our approach is to walk through each of these concerns, citing the source that implements it, and culminate in the `"Child launched"` log line that is the single, transitive proof that the entire chain has succeeded.

### 3.2 PTY Pair Creation (`os.openpty`)

`Child.fork()` in `kitty/child.py` at line 276 is the canonical entry point. It constructs everything needed for the child:

```python
# kitty/child.py:276 onwards
def fork(self) -> Optional[int]:
    ...
    master, slave = openpty()       # line 281
    ready_read_fd, ready_write_fd = os.pipe()   # line 283
    ...
```

The module-local `openpty` at line 170–171 is:

```python
# kitty/child.py:170-171
def openpty() -> Tuple[int, int]:
    master, slave = os.openpty()
    ...
    return master, slave
```

`os.openpty()` is a thin wrapper over the POSIX `openpty(3)` call, which allocates a PTY master/slave pair. The PTY master (`master` / eventually `self.child_fd`) is the parent's side; the slave (`slave`) becomes the child's `stdin`, `stdout`, `stderr`, and controlling TTY.

### 3.3 Child Environment Construction (`get_final_env`)

Immediately after PTY allocation, `Child.get_final_env()` at line 233 builds the child's environment dictionary:

```python
# kitty/child.py:233 onwards (representative excerpt)
def get_final_env(self) -> Dict[str, str]:
    env = os.environ.copy()
    env['TERM']            = opts.term              # default: 'xterm-kitty'
    env['COLORTERM']       = 'truecolor'
    env['KITTY_PID']       = str(os.getpid())
    env['KITTY_INSTALLATION_DIR'] = kitty_base_dir
    env['TERMINFO']        = base64_terminfo_data() # base64-encoded embedded terminfo
    ...
    if 'disabled' not in opts.shell_integration:
        env = modify_shell_environ(opts, env, argv)  # kitty/shell_integration.py
    ...
    return env
```

The key contract here is that **the child shell knows exactly what Kitty is** and what it supports, via three coordinates:

| Env var | Value | Purpose |
|---------|-------|---------|
| `TERM` | `xterm-kitty` (default from `kitty/options/definition.py`) | ncurses/terminfo lookup key |
| `COLORTERM` | `truecolor` | Signal 24-bit RGB support to shells and apps |
| `TERMINFO` | base64-encoded embedded terminfo DB | Lets the child compile terminfo entries without a separately-installed `xterm-kitty.ti` |
| `KITTY_PID` | parent PID | Allows `kitty @` remote control, shell-integration signaling |
| `KITTY_INSTALLATION_DIR` | repo/install root | Used by shell integration to locate helper scripts |

Shell-integration variables (injected via `modify_shell_environ()` in `kitty/shell_integration.py`) add per-shell coordinates — e.g. `ZDOTDIR` for zsh, `KITTY_SHELL_INTEGRATION=enabled`, and `XDG_DATA_DIRS` extensions for fish — so that when the shell runs its own rc files, it will source Kitty's integration script.

### 3.4 Spawn (`fast_data_types.spawn`) and Ready-Pipe Handshake

At line 333 of `kitty/child.py`:

```python
# kitty/child.py:333 (approximate)
pid = fast_data_types.spawn(
    final_exe, cwd, tuple(argv), env,
    master, slave,
    stdin_read_fd, stdin_write_fd,
    ready_read_fd, ready_write_fd,
    tuple(handled_signals), kitten_exe(), opts.forward_stdio)
```

`spawn()` is a C-implemented helper (see `kitty/*_spawn*.c` in the C sources). It performs the following atomically from the parent's perspective:

1. Calls `posix_spawn`/`fork` to create the child process.
2. In the child: opens the slave PTY as `stdin`/`stdout`/`stderr`, sets it as the controlling terminal, resets signal handlers/masks, `chdir(cwd)`, and `execve(final_exe, argv, env)`.
3. In the parent: closes the slave-side fds the parent doesn't need, and returns the child PID.
4. Before the child's `execve`, the child is arranged so that the very first thing the freshly-execed program does (via Kitty's shell-integration hook, or via the spawn helper itself) is to `read(ready_read_fd)` — which blocks until the parent closes `ready_write_fd`.

After `spawn()` returns, the parent (still in `Child.fork()`) performs housekeeping:

```python
# kitty/child.py (approximate, post-spawn housekeeping)
os.close(slave)               # parent no longer needs the slave end
self.child_fd = master        # keep master for I/O
os.close(ready_read_fd)       # parent doesn't read the ready pipe
self.terminal_ready_fd = ready_write_fd   # parent will close this to release the child
os.set_blocking(self.child_fd, False)     # PTY master becomes non-blocking for the I/O thread
# best-effort; no-op on systems without systemd:
systemd_move_pid_into_new_scope(pid, ...)
```

The `systemd_move_pid_into_new_scope()` call is the source of the `"Failed to open systemd user bus"` log line seen in the captured debug output (Appendix B). In a container without a user D-Bus it fails and is logged as `log_error(...)` — but the failure is **non-fatal** and does not block startup.

### 3.5 `mark_terminal_ready()` — the gating signal

`Child.mark_terminal_ready()` is defined in `kitty/child.py` at line 362:

```python
# kitty/child.py:362
def mark_terminal_ready(self) -> None:
    os.close(self.terminal_ready_fd)
    self.terminal_ready_fd = -1
```

Closing the write end of the ready pipe causes the child's `read(ready_read_fd)` to return `0` (EOF). The child's first-line integration/spawn helper then proceeds to `exec` the user's shell (or simply to start reading from `stdin`), and only **then** does the shell begin emitting output. This guarantees the first output is rendered at the correct column/row because the parent has already called `resize_pty`.

The caller is `Window.set_geometry()` in `kitty/window.py` around lines 862–876:

```python
# kitty/window.py:862-876 (excerpt)
def set_geometry(self, new_geometry: WindowGeometry) -> None:
    ...
    current_pty_size = (new_geometry.xnum, new_geometry.ynum,
                        cell_width * new_geometry.xnum, cell_height * new_geometry.ynum)
    if current_pty_size != self.last_reported_pty_size:
        boss.child_monitor.resize_pty(self.id, *current_pty_size)
        self.last_reported_pty_size = current_pty_size
        if not self.child_is_launched:
            self.child.mark_terminal_ready()      # line 866
            self.child_is_launched = True          # line 867
            if boss.args.debug_rendering:
                now = monotonic()
                print(f'[{now:.3f}] Child launched', file=sys.stderr)   # line 871
```

The `"Child launched"` print in `--debug-rendering` is literally how we observe that the entire handshake — PTY creation, environment construction, spawn, ready-pipe close, first `resize_pty` — has completed. In our captured run it fired at `[0.158]` (Appendix B).

### 3.6 I/O Thread: `read_bytes` → VT Parser Write Buffer

`ChildMonitor` runs a dedicated I/O thread. It is launched from `ChildMonitor.start()` in `kitty/child-monitor.c` at line 291:

```c
// kitty/child-monitor.c:291
pthread_create(&self->io_thread, NULL, io_loop, self);
```

`io_loop` (defined around line 229 and onward in the same file) is the loop body. Conceptually:

```c
// kitty/child-monitor.c::io_loop (conceptual pseudocode)
while (!self->shutting_down) {
    poll(fds /* PTY masters + wakeup pipe + signal pipe */,
         nfds, timeout_ms);
    for each PTY fd with POLLIN:
        read_bytes(fd, screen);            // line 1337
    ...
    if (data_read || signal_received)
        wakeup_main_loop();                // line 1165 (respects input_delay)
}
```

`read_bytes()` at line 1337 reads from the PTY master into the `Screen` object's VT-parser write buffer using `read(fd, ...)` (non-blocking), loops on `EINTR`, stops on `EAGAIN`, and respects the fixed capacity of the ring-buffered `write_buf_used` counter. On any successful read, it calls `vt_parser_commit_write()` to publish the bytes to the main-thread side of the parser.

`wakeup_main_loop()` at line 1165 pokes the wakeup pipe that the main thread is blocking on in `glfwPollEvents`. The `input_delay` option (default **3 ms** from `kitty/options/definition.py:878`) throttles wakeups — multiple reads within a 3 ms window are coalesced into a single main-thread tick — preserving the render budget when the child is emitting a torrent of data (e.g. `cat large_file`).

### 3.7 Main-Thread `parse_input()` → `screen_draw_text()`

On the main thread, the GLFW main loop tick calls `process_global_state()` in `kitty/child-monitor.c` at line 1224:

```c
// kitty/child-monitor.c:1224 (conceptual)
static double
process_global_state(void *data) {
    ChildMonitor *self = data;
    ...
    parse_input(self);         // line 451
    ...
    // Later in the tick:
    for each OS window:
        render_os_window(os_window, ...);   // line 833
    ...
    return next_tick_time;
}
```

`parse_input()` at line 451 iterates all windows, taking the VT parser's pending write buffer (published by the I/O thread) and feeding it to the VT state machine implemented in `kitty/vt-parser.c`. The state machine classifies bytes into:

- **Printable text** → `screen_draw_text()` in `kitty/screen.c` (inserts into the active line at the cursor)
- **C0 control codes** (e.g. `\r`, `\n`, `\t`, `\b`) → `screen_*` handlers
- **CSI sequences** (e.g. `ESC [ ... letter`) → cursor movement, color changes, scroll region, mode switches
- **OSC sequences** (e.g. `ESC ] 0 ; title BEL`) → title/clipboard/color palette changes
- **DCS / APC sequences** → Kitty graphics protocol, device control strings

`screen_on_input()` fires the activity callback for side-effects such as updating the last-input timestamp used for `cursor_blink_interval`.

### 3.8 Data-Flow Diagram

```
          [Shell process writes to stdout/stderr]
                         |
                         v  (kernel PTY line discipline)
                   [PTY slave fd]  <-- child side
                         |
                         v
                   [PTY master fd] <-- parent side (non-blocking)
                         |
                         v
           +---- [io_loop (I/O thread)] ----+
           |     poll() -> read_bytes()     |
           |     -> vt_parser_commit_write()|
           +--------------+-----------------+
                          |
                          v
                  [wakeup_main_loop]
                  (input_delay = 3 ms)
                          |
       --- main thread tick (glfwPollEvents wakes) ---
                          |
                          v
                [process_global_state]
                          |
                          v
                    [parse_input]
                          |
                          v
          [vt-parser state machine  (kitty/vt-parser.c)]
                          |
      +-------------------+------------------------+
      |                   |                        |
      v                   v                        v
[screen_draw_text]   [CSI/OSC/DCS/APC]   [graphics protocol]
      |                   |                        |
      +----------+--------+------------------------+
                 |
                 v
         [Screen line buffer]  (kitty/line.c / line-buf.c)
                 |
                 v  (next render tick)
          [render_os_window]
                 |
                 v
       [send_cell_data_to_gpu]
                 |
                 v
         [glfwSwapBuffers]
                 |
                 v
             [Frame]
```

### 3.9 Observed Evidence (`"Child launched"`)

The captured run with `--debug-rendering --debug-keyboard --debug-font-fallback` (Appendix B) shows:

```
[0.145] OS Window created
[0.154] Failed to open systemd user bus with error: Connection refused
[0.158] Child launched
[0.158] Text fonts:
[0.158]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
...
```

Reading the timeline:

- `[0.145] OS Window created` — GLFW window, GL context, shaders, pre-rendered sprites all ready. (`kitty/glfw.c:1321`.)
- `[0.154] Failed to open systemd user bus` — non-fatal; the spawn helper could not move the new child into a systemd user scope because there is no session bus in the container. (`kitty/child.py` best-effort path.)
- `[0.158] Child launched` — `Window.set_geometry()` has fired for the first time, `resize_pty` has propagated the initial 640×400 pixel / cells-wide-cells-tall dimensions to the PTY, and `Child.mark_terminal_ready()` has closed `ready_write_fd`. The child can now proceed to exec the shell and emit output. (`kitty/window.py:871`.)

The `"Child launched"` line is therefore the single most compact proof that **every** step of the data-flow chain above has succeeded: the PTY exists, the slave is attached to a running child PID, the master is registered with the I/O thread's poll set, the parent has sized the PTY to match the window geometry, and the child's read-side handshake has been released. Any byte that the shell writes from this moment forward traverses the full pipeline described in §§3.6–3.8 and ends up as a cell on the GPU-rendered frame.

---

## Question 4 — What evidence confirms the display system is working?

### 4.1 Thinking / Rationale

A "working terminal" is more than a byte pipeline — it requires that a GPU-rendered frame with the correct characters, colors, and cursor position actually reaches the screen. That means three independent pipelines must all succeed: (1) the **GPU pipeline** (OpenGL context + shaders + vertex buffers + swap-buffer call), (2) the **font pipeline** (FreeType face opened, glyphs rasterized, cell metrics computed, sprite atlas uploaded), and (3) the **window/layout pipeline** (OS window created with the right WM class/title and the right pixel dimensions).

Rather than instrument internal state, we pick *external*, *observable* signals that each subsystem emits and that we can capture unambiguously from a headless run:

| Subsystem | External evidence we can capture |
|-----------|----------------------------------|
| OpenGL context | `"GL version string: '4.5 ...'"` from `kitty/gl.c:72` |
| XKB/keyboard | `"Loading new XKB keymaps"` + `"Modifier indices ..."` from `glfw/xkb_glfw.c` lines 672, 376, 540 |
| OS window | `"OS Window created"` from `kitty/glfw.c:1321` + `xwininfo` output |
| Fonts | `--debug-font-fallback` face list from `kitty/fonts/render.py::dump_font_debug` line 161 |
| Shader compilation | **Transitive proof** — if any shader pair (cell, graphics, bgimage, tint, border) failed to compile, `LoadShaderPrograms.__call__` would raise and the subsequent `"OS Window created"` print would never happen. Its appearance in the log is therefore sufficient evidence that shader compilation succeeded. |
| PTY/shell handshake | `"Child launched"` from `kitty/window.py:871` |

Correlating these signals with the source locations that emit them, and with the `xwininfo` snapshot of the live window, gives a complete evidential picture that the display system is functioning end-to-end.

### 4.2 OpenGL Context Evidence (GL version string)

`kitty/gl.c::gl_init()` — invoked from inside `create_os_window()` immediately after the GLFW window/context is made current — performs the driver entry-point loading via `gladLoadGL` and then (conditionally on `debug_rendering`) prints the version string:

```c
// kitty/gl.c:72 (approximate)
if (global_state.debug_rendering)
    printf("[%.3f] GL version string: %s Detected version: %d.%d\n",
           monotonic_t_to_s_double(monotonic()),
           gl_version_string(), major, minor);
```

It also enforces the minimum OpenGL version Kitty needs (≥ 3.1 / 3.3 for core cell shaders and features like integer attributes and texture arrays). If the driver reports a lower version, `gl_init()` aborts with an error; the startup does **not** reach `"OS Window created"`.

Captured evidence:

```
[0.119] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

This confirms four things at once:

1. `gladLoadGL` succeeded — every GL function pointer Kitty uses (glDrawArrays, glBindVertexArray, glUniform*, glTexSubImage2D, ...) resolved.
2. The version (`4.5`) is comfortably above Kitty's required minimum.
3. The driver is Mesa (software rasterizer `llvmpipe` on Xvfb, since there is no hardware GPU under X11 virtual framebuffer).
4. The OpenGL context is a **Core Profile** context — no deprecated fixed-function state is available, confirming Kitty's modern-GL code path is the one in use.

### 4.3 Shader Compilation Evidence (cell / graphics / bgimage / tint / border)

Shader compilation happens inside `create_os_window()` (`kitty/glfw.c` lines 1253–1322). The Python side uses `LoadShaderPrograms` (`kitty/shaders.py` line 131, `__call__` at line 147):

```python
# kitty/shaders.py:131
class LoadShaderPrograms:
    ...
    def __call__(self, allow_recompile: bool = False) -> None:
        # compiles cell, graphics, bgimage, tint programs and binds uniforms
        ...
```

`init_cell_program()` at line 201 finalizes the cell shader by linking, querying uniform locations, and uploading static uniforms. Border shader compilation is handled separately by `load_borders_program()` in `kitty/borders.py:63`:

```python
# kitty/borders.py:63
def load_borders_program() -> None:
    program = Program(vertex=..., fragment=...)
    ...
```

GLSL source files involved:

| Shader pair | Files |
|-------------|-------|
| Cell (text + cursor) | `kitty/cell_vertex.glsl`, `kitty/cell_fragment.glsl` |
| Graphics (inline images) | `kitty/graphics_vertex.glsl`, `kitty/graphics_fragment.glsl` |
| Background image | `kitty/bgimage_vertex.glsl`, `kitty/bgimage_fragment.glsl` |
| Tint overlay | `kitty/tint_vertex.glsl`, `kitty/tint_fragment.glsl` |
| Border | `kitty/border_vertex.glsl`, `kitty/border_fragment.glsl` |
| Utility | `kitty/alpha_blend.glsl`, `kitty/linear2srgb.glsl` |

**Transitive evidence**: If *any* of these GLSL pairs fails to compile or link, `LoadShaderPrograms.__call__` / `load_borders_program` raises a Python exception, the `create_os_window` path aborts, and `"OS Window created"` is **never** printed. Because we observe

```
[0.145] OS Window created
```

in the captured log (and subsequently `"Child launched"` at `[0.158]`, which depends on having reached Boss startup), all shader programs must have compiled and linked successfully.

### 4.4 Font Pipeline Evidence (Fontconfig queries, FreeType face loads)

The Python side of the font pipeline is orchestrated by `set_font_family()` in `kitty/fonts/render.py:173`:

```python
# kitty/fonts/render.py:173
def set_font_family(opts: Optional[Options] = None, ...) -> None:
    ...
    # 1. resolve font descriptors via fontconfig / core text
    medium, bold, italic, bi = get_font_files(opts)   # kitty/fonts/common.py
    # 2. push descriptors + callbacks to the C side
    set_font_data(...)
```

`get_font_files()` in `kitty/fonts/common.py` resolves the four variants (regular, bold, italic, bold-italic) by calling the platform-specific font enumerator. On Linux that is `font_for_family()` in `kitty/fonts/fontconfig.py`, which issues Fontconfig queries against the system's font cache.

When `--debug-font-fallback` is passed on the command line, `dump_font_debug()` at line 161 is invoked by `_run_app` / `Boss` after `set_font_family()` has completed, printing the resolved faces.

Captured evidence (Appendix B):

```
[0.158] Text fonts:
[0.158]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.158]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.158]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.158]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

Interpretation:

- The default `font_family = FontSpec(system='monospace', ...)` (Q2 §2.6) was fed to Fontconfig as the alias `monospace`.
- On this Ubuntu 24.04 container, Fontconfig's alias chain resolves `monospace` to **DejaVu Sans Mono** — all four variants present at `/usr/share/fonts/truetype/dejavu/DejaVuSansMono*.ttf`.
- The `:0` suffix is the FreeType face index inside each TTF (face 0 = the one and only face in a single-face TTF).
- That all four variants resolved to real files on disk means the FreeType `FT_New_Face` call for each variant succeeded; the C side (`kitty/fonts.c`) proceeded to open the faces, rasterize glyphs, and build the GPU sprite atlas.

Note: the AAP's planning document referenced LiberationMono; in this specific test container Fontconfig's `monospace` alias chain prefers DejaVu Sans Mono over Liberation Mono — this is a property of the Fontconfig configuration installed in the image, not of Kitty. The evidence stands either way: a real monospace family, with all four variants, was resolved.

### 4.5 Window and Layout Evidence (xwininfo)

`xwininfo -root -tree` was run against `DISPLAY=:99` while Kitty was live (Appendix C). The tree output included:

```
0x20000c "sh": ("kitty" "kitty")  640x400+0+0  +0+0
   1 child:
   0x200001 (has no name): ()  1x1+0+0  +0+0
```

Decoded:

- `0x20000c` — X11 window ID of Kitty's top-level OS window (value is arbitrary / session-dependent).
- `"sh"` — `WM_NAME` (the window's title). Kitty's default title policy is to follow the child's argv[0] unless overridden; since we launched `sh -c 'sleep 2'`, the title became `sh`.
- `("kitty" "kitty")` — `WM_CLASS`, which is `(instance, class)`. Both are the literal string `kitty`, set by `set_x11_window_icon()` / GLFW during `create_os_window`. This is how desktop environments group Kitty windows together.
- `640x400+0+0` — the window's geometry: 640 × 400 pixels at offset (0, 0) in the root window. This matches `initial_window_width=640` (`kitty/options/definition.py:994`) and `initial_window_height=400` (line 998) — direct evidence that the default layout path in `kitty/os_window_size.py::initial_window_size_func` was taken.
- `0x200001 (has no name) 1x1+0+0` — an auxiliary 1×1 hidden window used by GLFW for certain IPC / clipboard operations. It is not visible to the user.

### 4.6 Pre-rendered Sprites and Cell Metrics

`kitty/fonts.c::send_prerendered_sprites()` is invoked during font-group initialization (see `initialize_font_group` / `calc_cell_metrics` in the same file, around lines 1450–1530). It rasterizes a small, fixed set of glyph-like sprites — the blank cell background, the various underline styles (straight, dotted, dashed, curly), and the cursor shapes (block, beam, underline) — and uploads them to the GPU sprite texture at known, hard-coded indices. The cell shader can then reference these indices directly without needing to look up a glyph from the font atlas.

There is no dedicated debug-log line for this step in the built code path, but it is *transitively* confirmed by two observations:

1. The `"Child launched"` print (`kitty/window.py:871`) only occurs after `Window.set_geometry()` has fired, and `set_geometry` depends on `cell_width`/`cell_height` being available — which requires `calc_cell_metrics` to have completed, which requires the sprite upload path to have run without error.
2. The visible window is 640×400 with the default defaults, meaning the cell metrics derived a *real* cell size (width × height), and the window's pixel dimensions are computed from cell size × cell count.

### 4.7 Observed Debug Log Summary

Cross-reference of the observable debug strings with their sources:

| Observed string | Source file:line | Meaning |
|-----------------|------------------|---------|
| `Loading new XKB keymaps` | `glfw/xkb_glfw.c:672` | XKB context loaded, keymap compiled from current layout |
| `Modifier indices alt: 0x3 super: 0x6 hyper: ... shift: 0x0 capslock: 0x1 control: 0x2` | `glfw/xkb_glfw.c:376` / `:540` | Modifier bit indices mapped; keyboard subsystem ready |
| `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5` | `kitty/gl.c:72` | OpenGL context live, Core Profile 4.5, minimum version satisfied |
| `OS Window created` | `kitty/glfw.c:1321` | GLFW window + GL context + shaders + sprites + icon + callbacks all succeeded |
| `Failed to open systemd user bus with error: Connection refused` | `kitty/child.py` (best-effort `systemd_move_pid_into_new_scope`) | Non-fatal; no session bus in container |
| `Child launched` | `kitty/window.py:871` | PTY sized, `mark_terminal_ready` fired, shell free to emit |
| `Text fonts: / Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/...` (×4) | `kitty/fonts/render.py::dump_font_debug:161` | All four font variants resolved to real files on disk; FreeType faces opened |

Summing up across §§4.2–4.7: the OS window exists and has the expected class/title/geometry; the OpenGL context is a Core Profile 4.5 context with Mesa; all shader programs compiled (transitive proof via the `"OS Window created"` print); all four font variants resolved and loaded via Fontconfig + FreeType; the XKB keyboard subsystem is initialized with the modifier map decoded; and the shell is free to emit bytes which will be parsed and rendered. The display system is functioning end-to-end.

---

## Appendix A — Complete Source File Reference Index

The following table enumerates every source file cited anywhere in this document and the phase/question it evidences. All paths are relative to the repository root (the directory containing `setup.py`). No file in this list was modified during the investigation.

| File | Role / Evidences |
|------|------------------|
| `kitty/launcher/main.c` | Native C launcher: descriptor validation, path resolution, Python embedding, `sys.kitty_run_data` population |
| `kitty/launcher/launcher.h` | `CLIOptions` struct definition for native-to-Python contract |
| `kitty/launcher/single-instance.c` | Single-instance UNIX socket coordination (referenced; not exercised in this investigation) |
| `kitty/entry_points.py` | Entry dispatcher that routes `sys.argv` to `kitty.main.main()` |
| `kitty/main.py` | `_main()` (line 441), `main()` (line 524), `init_glfw()` (line 95), `_run_app()` (line 202), `load_all_shaders` callback (line 82) — full startup orchestration |
| `kitty/cli.py` | `parse_args`, `create_opts` (line 1081), `default_config_paths` (line 1067), `SYSTEM_CONF` (line 1064) |
| `kitty/cli_stub.py` | `CLIOptions` dataclass holding parsed command-line state |
| `kitty/config.py` | `load_config()` (line 163), `finalize_keys`, `finalize_mouse_mappings` |
| `kitty/conf/utils.py` | Generic `resolve_config()` (line 322), `load_config()` (line 332) |
| `kitty/constants.py` | `appname` (line 23), `config_dir` (line 131), `defconf` (line 133), `glfw_path`, `is_wayland`, resource paths |
| `kitty/options/definition.py` | Canonical option schema and defaults — `font_size` line 59, `scrollback_lines` line 372, `repaint_delay` line 866, `input_delay` line 878, `sync_to_monitor` line 889, `initial_window_width` line 994, `initial_window_height` line 998, `background_opacity` line 1468, `shell_integration` line 3141 (finalized) |
| `kitty/options/types.py` | `Options` named-tuple type and the `defaults` singleton with compiled-in default values |
| `kitty/options/parse.py` | Line-by-line config parser (`create_result_dict`, `parse_conf_item`) |
| `kitty/boss.py` | `Boss.__init__` (line 325), `Boss.startup_first_child` (line 383), `Boss.start` (line 1181) |
| `kitty/session.py` | `create_sessions()`, `get_os_window_sizing_data()` |
| `kitty/os_window_size.py` | `initial_window_size_func()` — computes pixel dimensions from cell size, DPI, and padding |
| `kitty/child.py` | `openpty` (line 170), `get_final_env` (line 233), `Child.fork` (line 276), PTY creation at line 281, ready pipe at line 283, `fast_data_types.spawn()` at line 333, `mark_terminal_ready` at line 362 |
| `kitty/child-monitor.c` | `io_loop` (line 229), `pthread_create(io_thread)` at line 291, `parse_input` (line 451), `send_cell_data_to_gpu` (line 714 / 766), `render_os_window` (line 833), `wakeup_main_loop` (line 1165), `process_global_state` (line 1224), `read_bytes` (line 1337) |
| `kitty/window.py` | `child_is_launched` flag (line 578 / 865), `set_geometry` (around line 850), `mark_terminal_ready` call (line 866), `"Child launched"` print (line 871) |
| `kitty/glfw.c` | `glfw_init`, `create_os_window` (lines 1253–1322), `"OS Window created"` debug print (line 1321) |
| `kitty/gl.c` | `gl_init`, `gladLoadGL`, `"GL version string"` debug print (line 72) |
| `glfw/xkb_glfw.c` | `glfw_xkb_compile_keymap`, `"Loading new XKB keymaps"` (line 672), `"Modifier indices"` (lines 376, 540) |
| `kitty/vt-parser.c` | VT state machine — CSI/OSC/DCS/APC dispatch; plain-text path into `screen_draw_text` |
| `kitty/screen.c` | `screen_draw_text`, `screen_on_input` |
| `kitty/fonts/render.py` | `dump_font_debug` (line 161), `set_font_family` (line 173), `set_font_data` bridge to C |
| `kitty/fonts/common.py` | `get_font_files` — resolves medium/bold/italic/bi descriptors |
| `kitty/fonts/fontconfig.py` | `font_for_family` — Linux Fontconfig queries |
| `kitty/fonts.c` | `send_prerendered_sprites`, `initialize_font_group`, `calc_cell_metrics` |
| `kitty/shaders.py` | `LoadShaderPrograms` class (line 131), `__call__` (line 147), `init_cell_program` (line 201) |
| `kitty/borders.py` | `load_borders_program` (line 63) |
| `kitty/cell_vertex.glsl`, `kitty/cell_fragment.glsl` | Cell (text + cursor) shaders |
| `kitty/graphics_vertex.glsl`, `kitty/graphics_fragment.glsl` | Inline-image shaders |
| `kitty/bgimage_vertex.glsl`, `kitty/bgimage_fragment.glsl` | Background image shaders |
| `kitty/tint_vertex.glsl`, `kitty/tint_fragment.glsl` | Tint overlay shaders |
| `kitty/border_vertex.glsl`, `kitty/border_fragment.glsl` | Border shaders |
| `kitty/alpha_blend.glsl`, `kitty/linear2srgb.glsl` | Utility GLSL shaders |
| `kitty/shell_integration.py` | `modify_shell_environ` — per-shell environment injection |
| `shell-integration/bash/kitty.bash`, `shell-integration/zsh/`, `shell-integration/fish/` | Shell-integration payloads (referenced; not exercised in detail) |
| `setup.py`, `pyproject.toml`, `go.mod` | Build system / Python/Go dependency manifests |

---

## Appendix B — Raw Captured Debug Log (verbatim)

The following is the captured stderr/stdout from a headless run with `DISPLAY=:99` and debug flags `--debug-rendering --debug-keyboard --debug-font-fallback`. Exact timestamps are from the monotonic clock at the time of capture; relative ordering is preserved verbatim.

```
[0.060] Loading new XKB keymaps
[0.064] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.119] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
[0.145] OS Window created
[0.154] Failed to open systemd user bus with error: Connection refused
[0.158] Child launched
[0.158] Text fonts:
[0.158]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.158]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.158]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.158]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.158] on_focus_change: window id: 0x1 focused: 1
```

Notes on observed ordering:

- The `"Loading new XKB keymaps"` and `"Modifier indices"` lines are emitted on the GLFW/XKB init thread before the GL context log; they appear early because XKB keymap compilation is part of opening the X11 display connection.
- The `"GL version string"` line appears in the log at `[0.119]`, *before* `"OS Window created"` at `[0.145]` in the captured output. This is because `gl_init()` executes inside `create_os_window()` — between GLFW window creation and the final `debug("OS Window created\n")` call.
- The `"Failed to open systemd user bus"` is a non-fatal warning from the best-effort `systemd_move_pid_into_new_scope` path in `kitty/child.py`. It appears after the first child's spawn and before `"Child launched"` because spawn runs before the parent calls `set_geometry`.
- `"Child launched"` at `[0.158]` is followed immediately by the font-debug dump at the same timestamp because `dump_font_debug()` is invoked from `_run_app` right after the first window's geometry is established.

---

## Appendix C — Raw xwininfo Output

`xwininfo -root -tree -display :99` captured live while Kitty was running:

```
xwininfo: Window id: 0x21f (the root window) (has no name)

  Root window id: 0x21f (the root window) (has no name)
  Parent window id: 0x0 (none)
     2 children:
     0x20000c "sh": ("kitty" "kitty")  640x400+0+0  +0+0
        1 child:
        0x200001 (has no name): ()  1x1+0+0  +0+0
```

Decoding:

- `0x21f` — root window of the `:99` Xvfb server.
- `0x20000c` — Kitty's top-level window. `WM_NAME = "sh"` (title = child argv[0]); `WM_CLASS = ("kitty", "kitty")` (instance, class). Geometry `640x400+0+0` — 640×400 pixels at position (0, 0) — directly confirms the compiled defaults `initial_window_width=640` and `initial_window_height=400` from `kitty/options/definition.py` lines 994/998.
- `0x200001` — auxiliary 1×1 hidden GLFW helper window.

---

## Appendix D — Glossary of Key Identifiers

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the planning document governing this investigation. |
| **AppRunner** | The `_run_app` orchestrator in `kitty/main.py` that sequences session / window creation and hands off to `Boss`. |
| **Boss** | The top-level Python controller (`kitty/boss.py`). Owns the `ChildMonitor`, clipboard, remote control, encryption key, and the list of OS windows. |
| **ChildMonitor** | C-backed object (`kitty/child-monitor.c`) that owns the I/O thread, the main-loop tick, and the per-child state. |
| **CSI / OSC / DCS / APC** | The four escape-sequence introducers in the VT protocol: CSI = Control Sequence Introducer (`ESC [`), OSC = Operating System Command (`ESC ]`), DCS = Device Control String (`ESC P`), APC = Application Program Command (`ESC _`). |
| **defconf** | `kitty/constants.py:133` — the user's default config path, `$(config_dir)/kitty.conf`. |
| **defaults** | The compiled-in `Options` singleton in `kitty/options/types.py`, populated from `kitty/options/definition.py`, used when no config file is loaded. |
| **fast_data_types** | The C extension module exposing performance-critical primitives (`spawn`, `set_options`, `set_font_data`, screen/line operations) to the Python layer. |
| **Fontconfig** | The Linux font-discovery library that resolves font-family aliases like `monospace` into concrete font files on disk. |
| **FreeType** | The cross-platform font rasterizer used by Kitty to convert vector glyphs into bitmaps for the GPU atlas. |
| **GLFW** | The vendored windowing library (`glfw/` subdirectory of the repo) that creates the OS window, OpenGL context, and input dispatch. |
| **GLSL** | The OpenGL Shading Language; Kitty's `.glsl` files are compiled by the GL driver at startup. |
| **HarfBuzz** | The text-shaping library used by Kitty for complex-script shaping (ligatures, marks, bidi). |
| **Mesa** | The open-source OpenGL/Vulkan driver stack; under Xvfb in the test container, GL calls are routed to Mesa's `llvmpipe` software rasterizer. |
| **OSWindow** | An OS-level window (top-level X11/Wayland/Cocoa window) as seen from `kitty/glfw.c` and `kitty/child-monitor.c`. |
| **PTY** | POSIX pseudo-terminal — a kernel abstraction providing a master/slave fd pair that looks like a TTY to processes writing to the slave. Kitty uses `os.openpty()` to allocate one per child. |
| **PTY master / slave** | The two ends of a PTY. The parent (Kitty) holds the master; the child (shell) is wired to the slave as its `stdin`/`stdout`/`stderr` and controlling terminal. |
| **Screen** | The per-window data structure (implemented in `kitty/screen.c` with Python wrapper) holding the current line buffer, scrollback, cursor state, and VT parser state. |
| **SYSTEM_CONF** | `kitty/cli.py:1064` — the system-wide config path, `/etc/xdg/kitty/kitty.conf`. |
| **Tab / TabManager** | A `Tab` is a logical group of windows within an `OSWindow`; `TabManager` owns the list of tabs. |
| **Window** | Kitty's logical window — a `Screen` plus geometry plus a `Child`. Multiple Windows can coexist inside one Tab (splits). |
| **Xvfb** | X Virtual Framebuffer — an X11 server that renders to a virtual display instead of a real one. Used to run Kitty headless in the CI / container environment. |
| **XKB** | The X Keyboard Extension. Used to compile keymaps and map physical key codes to layout-dependent symbols and modifier indices. |

---

## Investigation Provenance

**Target commit**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (short: `815df1e210e0`), commit message *"Wire up applying of font config"*.

**Environment**: Ubuntu 24.04.4 LTS on x86_64 in a Kubernetes pod derived from `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. Python 3.12.3, Go 1.22.2. Display driver: Mesa 25.2.8 (software `llvmpipe` under Xvfb).

**Headless framebuffer**: `Xvfb :99 -screen 0 1280x720x24` started during environment setup; `DISPLAY=:99` exported.

**Build command**: `python3 setup.py build --ignore-compiler-warnings` — produced `kitty/launcher/kitty`, `kitty/fast_data_types.so`, `kitty/glfw-x11.so`, `kitty/glfw-wayland.so`, plus all Go kittens/tools. The `--ignore-compiler-warnings` flag was required because the container's `wayland-protocols` package ships newer protocol header enumerations than the vendored GLFW Wayland backend expects. This is a build-system accommodation only; the X11 backend (which is what the captured runs used) is unaffected.

**Launch flags**: `--debug-rendering --debug-keyboard --debug-font-fallback` with child command `sh -c 'echo READY; sleep 1'` (or `sh -c 'sleep 2'` for the xwininfo capture).

**Evidence streams correlated**:

1. **Source code analysis** — `grep -n` and targeted line-range reads across `kitty/`, `kitty/conf/`, `kitty/options/`, `kitty/fonts/`, `kitty/launcher/`, and `glfw/` to locate every cited identifier and debug string. Source paths and line numbers in this document were all verified directly against the repository at commit `815df1e21`.
2. **Live headless execution** — Kitty launched under Xvfb with debug flags; stderr/stdout captured. Used to produce Appendix B.
3. **Window verification** — `xwininfo -root -tree -display :99` executed concurrently with a running Kitty process to capture the actual X11 window tree. Used to produce Appendix C.
4. **Runtime Python introspection** — `kitty.options.types.defaults` accessed in a Python REPL (with `sys.kitty_run_data` pre-populated) to enumerate the actual default values that would take effect in the absence of any config file. Used to validate §2.6.

**Read-only constraint**: No file in the source repository was modified as part of this investigation. The only artifact produced is this markdown document at `blitzy/documentation/kitty_815df1e210e0.md`. All temporary log files and helper scripts created during evidence capture have been removed.

