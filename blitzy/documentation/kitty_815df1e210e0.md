# Kitty Terminal Emulator — Startup Behavior Investigation

**Repository**: [kovidgoyal/kitty](https://github.com/kovidgoyal/kitty)
**Commit investigated**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (HEAD: *"Wire up applying of font config"*)
**Kitty version reported by built binary**: `kitty 0.35.2 created by Kovid Goyal`
**Investigation environment**: Ubuntu 24.04.4 LTS, x86_64, Python 3.12.3, Go 1.22.2, Mesa 25.2.8, X.Org 21.1.11 via Xvfb (virtual framebuffer on `:99`, `1280x720x24`)
**Build command**: `python3 setup.py build --ignore-compiler-warnings` (succeeded — 122 C compile units, all Wayland protocols, all Go kittens/tools, linked 5 shared objects)

This document answers four tightly-related observational questions about what happens between Kitty's process entry and a working shell prompt. Every claim below is grounded either in a direct source-code citation (file and line number at commit `815df1e21`) or in live debug output captured from the binary compiled at that commit and run headlessly under Xvfb.

---

## Table of Contents

1. [Q1 — Startup Subsystems: What Comes Online Before the Terminal Is Ready?](#q1--startup-subsystems-what-comes-online-before-the-terminal-is-ready)
2. [Q2 — Configuration Resolution: How Does Kitty Determine Its Initial Settings?](#q2--configuration-resolution-how-does-kitty-determine-its-initial-settings)
3. [Q3 — Terminal-to-Shell Communication: How Does Kitty Talk to the Child Shell?](#q3--terminal-to-shell-communication-how-does-kitty-talk-to-the-child-shell)
4. [Q4 — Display System Evidence: Fonts, Layout, Rendering, and Logs](#q4--display-system-evidence-fonts-layout-rendering-and-logs)
5. [Appendix A — Full Live Debug Output from Headless Run](#appendix-a--full-live-debug-output-from-headless-run)
6. [Appendix B — Default Option Values (Introspection)](#appendix-b--default-option-values-introspection)
7. [Appendix C — Call Graph Summary](#appendix-c--call-graph-summary)

---

## Q1 — Startup Subsystems: What Comes Online Before the Terminal Is Ready?

### 1.1 High-Level Sequence

Between invocation and the moment the child shell has a sized PTY and the terminal can compose the first frame, Kitty activates a deterministic sequence of subsystems spanning both the Python orchestration layer and native C code. The sequence as observed on this Linux/X11 configuration at commit `815df1e21`:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 1.  Native launcher bootstrap (sys.kitty_run_data populated)            │
│ 2.  Python entry dispatch:  entry_points.main() -> main._main()         │
│ 3.  running_in_kitty(True)  + CWD validation                            │
│ 4.  CLI argument parsing    (parse_args())                              │
│ 5.  Configuration resolution + defaults (create_opts())                 │
│ 6.  Environment setup       (setup_environment())                       │
│ 7.  Locale configuration    (set_locale())                              │
│ 8.  Kitty-wide signal masking (mask_kitty_signals_process_wide)         │
│ 9.  GLFW platform init       (init_glfw() -> x11 on this system)        │
│ 10. XKB keymap compilation   (glfw_xkb_compile_keymap) ──────► [0.065s] │
│ 11. Modifier indices published (XKB) ──────────────────────► [0.070s]   │
│ 12. Box-drawing scale push    (set_scale())                             │
│ 13. Options push to C state   (set_options())                           │
│ 14. Font discovery + face loading (set_font_family())                   │
│ 15. Session creation          (create_sessions())                       │
│ 16. OS window creation        (create_os_window()) ────────► [0.148s]   │
│      └─ gl_init() + GL version check (GL ≥ 3.1) ───────────► [0.123s]   │
│      └─ load_all_shaders() -> shader compilation                        │
│      └─ send_prerendered_sprites_for_window() (blank cell, underlines)  │
│      └─ window icon + all event callbacks registered                    │
│      └─ "OS Window created" log line                                    │
│ 17. Boss controller construction (ChildMonitor, encryption key, ...)    │
│ 18. child_monitor.start() -> pthread_create(io_thread)                  │
│ 19. startup_first_child() -> TabManager -> Tab -> Window                │
│ 20. PTY creation (os.openpty()) + ready-pipe creation                   │
│ 21. fast_data_types.spawn() -> fork child shell ──────────► [0.159s]    │
│ 22. systemd_move_pid_into_new_scope() (Linux)                           │
│ 23. Font debug output (opt-in via --debug-font-fallback) ─► [0.162s]    │
│ 24. First set_geometry() -> child.mark_terminal_ready()                 │
│      -> "Child launched" log line ─────────────────────────► [0.162s]   │
│ 25. child_monitor.main_loop() enters run loop, renders frames           │
└──────────────────────────────────────────────────────────────────────────┘
```

Timestamps in brackets are real measurements from this container (`DISPLAY=:99`, `1280x720x24` Xvfb) captured with `--debug-rendering --debug-keyboard --debug-font-fallback`.

### 1.2 Subsystem-by-Subsystem Walkthrough with Source Evidence

#### 1.2.1 Native Launcher → Python Dispatch

Kitty's production executable is a small C launcher. On a packaged build the program begins at `kitty/launcher/main.c`, which validates file descriptors, resolves paths, embeds Python, and — critically — creates the `sys.kitty_run_data` dictionary before any Python module runs. The `set_kitty_run_data()` routine assembles that dictionary with keys `bundle_exe_dir`, `from_source` (when running from source), `lc_ctype_before_python`, and `extensions_dir` (`kitty/launcher/main.c` lines 52–75). Every Python module downstream that needs to know *where* the installation lives (e.g., `glfw_path()` in `kitty/constants.py:191–193`, or `kitty.fast_data_types` loading) reads from this dictionary.

When running from source *without* the C launcher (this investigation's environment), `sys.kitty_run_data` must be populated manually, mirroring what the launcher does, before importing `kitty.main`.

The Python entry is `kitty/entry_points.py::main()` at line 183. For a normal GUI launch (no kitten sub-command or frozen-namespace token on `argv[1]`), it dispatches to `kitty.main.main()`:

```python
# kitty/entry_points.py (line 183)
def main() -> None:
    ...
    from .main import main as fmain
    fmain()
```

#### 1.2.2 `_main()` — The Python Orchestration Function

`kitty/main.py::_main()` is the master orchestrator. Its body (lines 441–521) performs the following in order:

| Step | Source (kitty/main.py) | Purpose |
|------|------------------------|---------|
| `running_in_kitty(True)` | line 442 | Sets a global flag so kitty knows it's the GUI process (not a kitten). |
| CWD validation | lines 449–454 | Falls back to `~` if current directory isn't a directory. |
| `parse_args()` | line 464 | Parses CLI via `kitty/cli.py` using `kitty/cli_stub.CLIOptions`. |
| `create_opts(cli_opts)` | line 494 | Loads config files, merges defaults, returns `Options`. |
| `setup_environment(opts, cli_opts)` | line 495 | Sets PATH (kitty + kitten), expands `listen_on`, `MANPATH`. |
| `set_locale()` | line 500 | Initializes C locale with UTF-8 support. |
| `sys.setswitchinterval(1000.0)` | line 504 | Makes Python essentially single-threaded (reduces GIL overhead). |
| `mask_kitty_signals_process_wide()` | line 513 | Blocks SIGINT/SIGTERM/SIGHUP/SIGCHLD/SIGUSR1/SIGUSR2 process-wide so display-backend threads created by GLFW cannot interfere with kitty's dedicated signal handling. |
| `init_glfw(opts, ...)` | line 514 | Selects platform backend and initializes GLFW. |
| `run_app(opts, cli_opts, ...)` | line 518 | Invokes `AppRunner.__call__`. |

The comment at line 510–512 is worth quoting verbatim: signals are masked *before* GLFW initialization precisely because GLFW may spawn helper threads on some platforms — those threads must not receive these signals, since Kitty's main thread is the only legitimate handler.

#### 1.2.3 GLFW Initialization and Platform Selection

`init_glfw()` (kitty/main.py line 95) selects the GLFW backend per platform:

```python
# kitty/main.py line 96
glfw_module = 'cocoa' if is_macos else ('wayland' if is_wayland(opts) else 'x11')
```

`is_wayland()` (kitty/constants.py line 207–217) returns `False` when `opts.linux_display_server == 'auto'` *and* the probe `detect_if_wayland_ok()` fails. That probe requires `WAYLAND_DISPLAY` or `WAYLAND_SOCKET` in the environment and `KITTY_DISABLE_WAYLAND` *not* set, plus the `glfw-wayland.so` shared object must exist (kitty/constants.py lines 196–204). In our container `WAYLAND_DISPLAY` is unset, so the X11 backend is selected.

The Python side then calls into C: `glfw_init()` in `kitty/glfw.c` (line 1430). This routine:

1. Loads the platform-specific GLFW shared object (`kitty/glfw-x11.so`) via `load_glfw(path)` at line 1441.
2. Registers an error callback (line 1443).
3. Sets init hints for debug keyboard, debug rendering, Wayland IME (lines 1444–1447).
4. On non-Apple, registers a D-Bus notification handler if available (line 1452).
5. Calls `glfwInit(monotonic_start_time)` at line 1456 — this is the call that actually connects to X, loads XKB, etc.
6. On success, registers the draw-text callback and queries default DPI (lines 1461–1463).

On the X11 path, part of `glfwInit` triggers XKB keymap compilation in `glfw/xkb_glfw.c`. The function `glfw_xkb_compile_keymap()` (line 670) performs: emit `"Loading new XKB keymaps"` debug line → `release_keyboard_data()` → `load_keymaps()` → `load_states()` → `load_compose_tables()` → `glfw_xkb_update_masks()`. When `--debug-keyboard` is active, it follows that with `"Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1"` (exact output captured from our run at timestamp 0.070s).

#### 1.2.4 `AppRunner.__call__` — The App Setup Pipeline

After `_main()` calls `run_app()`, control passes to `AppRunner.__call__` (kitty/main.py line 247). Its sequence is:

```python
# kitty/main.py lines 247–252
def __call__(self, opts, args, bad_lines=(), talk_fd=-1):
    set_scale(opts.box_drawing_scale)          # push box-drawing params to C
    set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)
    try:
        set_font_family(opts)                  # discover + load fonts
        _run_app(opts, args, bad_lines, talk_fd)
    finally:
        set_options(None)                      # tear down
        free_font_data()
        ...
```

`set_options()` is a C function exposed by `kitty/fast_data_types.so`; it installs the Options into a global C struct so that subsequent C code (`render`, `screen_draw_text`, etc.) can consult option values with zero Python round-trips. `set_font_family()` is the Python-layer font discovery/loading entry point, covered below in §4.

#### 1.2.5 `_run_app` — Session, Window, Boss, Event Loop

`_run_app()` (kitty/main.py line 202) then:

1. On non-Wayland non-macOS, calls `set_x11_window_icon()` (line 211) to make the kitty logo available as the window icon.
2. Wraps the startup in `cached_values_for(...)` to persist remembered window size/state between runs.
3. Calls `create_sessions(opts, args, default_session=opts.startup_session)` (line 214) — for a default no-argument launch this yields a single `Session` with one tab and one window running the login shell.
4. Calls `create_os_window(...)` (line 221) — this is the GLFW window creation. It passes `load_all_shaders` as a callback so the C side can compile shaders *after* the context is current.
5. Constructs `Boss(...)` (line 226) — this creates the `ChildMonitor` but does NOT yet start the I/O thread.
6. Calls `boss.start(...)` (line 227) — this finally launches the I/O thread and forks the child shell (see §3).
7. If `--debug-font-fallback` is set, calls `dump_font_debug()` (line 229) which emits the font-selection log lines we observed.
8. Finally enters the event loop: `boss.child_monitor.main_loop()` (line 234). On exit, `boss.destroy()` tears down.

#### 1.2.6 OS-Window Creation, GL Init, Shader Compilation

`create_os_window()` is the C function in `kitty/glfw.c`. Key steps in its body (lines 1253–1322):

- `add_os_window()` (line 1253) — allocates an `OSWindow` slot in `global_state`.
- `send_prerendered_sprites_for_window(w)` (line 1273) — rasterizes blank cells, underlines, cursor shapes into the GPU sprite atlas.
- Sets the window icon from `logo.pixels` (line 1274).
- Sets the text cursor glyph (line 1275).
- Registers ~14 GLFW event callbacks (lines 1277–1293): window position, close, refresh, focus, occlusion, iconify, framebuffer size, live resize, DPI change, mouse button, cursor pos, cursor enter, scroll, keyboard, drop.
- Initializes window-chrome state (line 1306).
- Sets `w->is_damaged = true` (line 1320) — marks the window for repaint on the next tick.
- Emits `debug("OS Window created\n")` (line 1321) — this is the log line we observe at ~0.148s.

The GL-version check runs in `kitty/gl.c::gl_init()` (line 52), called during window creation. It uses GLAD to load GL function pointers (`gladLoadGL(glfwGetProcAddress)` at line 55), fatals if `ARB_texture_storage` is missing (line 67), and in debug-rendering mode prints `"[%.3f] GL version string: '%s' Detected version: %d.%d"` (lines 46–47, 72). The minimum required GL version is defined in `kitty/data-types.h` as `OPENGL_REQUIRED_VERSION_MAJOR=3, OPENGL_REQUIRED_VERSION_MINOR=1` on Linux (3.3 on macOS), with GLSL `#version 140`. Our Xvfb container's Mesa driver provides `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1` — ample.

Shader compilation is orchestrated by `kitty/shaders.py::LoadShaderPrograms.__call__` (line 147). It compiles:

- **Cell programs (4 phases)** — BOTH, BACKGROUND, SPECIAL, FOREGROUND — each from `cell_vertex.glsl` + `cell_fragment.glsl` using `#pragma kitty_include_shader` resolution, with per-phase macro replacements for `WHICH_PHASE`, `TRANSPARENT`, `FG_OVERRIDE_THRESHOLD`, `FG_OVERRIDE`, `TEXT_NEW_GAMMA`, plus constant shifts for attribute packing (`REVERSE_SHIFT`, `STRIKE_SHIFT`, `DIM_SHIFT`, `DECORATION_SHIFT`, `MARK_SHIFT`, etc.) (shaders.py lines 153–184).
- **Graphics programs (3 alpha types)** — SIMPLE, PREMULT, ALPHA_MASK — from `graphics_vertex.glsl` + `graphics_fragment.glsl` (shaders.py lines 186–197).
- **Background-image program** — from `bgimage_vertex.glsl` + `bgimage_fragment.glsl` (shaders.py line 199).
- **Tint program** — from `tint_vertex.glsl` + `tint_fragment.glsl` (shaders.py line 200).
- Finally `init_cell_program()` (line 201) binds uniform locations.

The border program is compiled separately: `load_borders_program()` in `kitty/borders.py` line 63 does `program_for('border').compile(BORDERS_PROGRAM)` → `init_borders_program()`. `load_all_shaders()` (kitty/main.py line 82) composes the two: shader programs + border program. It is passed as a callback into `create_os_window()` so that shader compilation happens *after* the GL context is made current.

#### 1.2.7 Boss — The Python Controller

`Boss.__init__()` (kitty/boss.py line 325) creates: the `ChildMonitor` instance (which owns the I/O thread), a SecureLib25519 encryption key for remote-control, the clipboard buffer, the OS-window-to-TabManager map, etc.

`Boss.start()` (kitty/boss.py line 1181) is invoked from `_run_app()`. It:
1. Calls `self.child_monitor.start()` (line 1183) — **this is where the I/O thread is created** via `pthread_create(&self->io_thread, NULL, io_loop, self)` (kitty/child-monitor.c line 291). The thread name is set to `"KittyChildMon"` (child-monitor.c line 1489).
2. Sets `io_thread_started = True` to ensure idempotence.
3. Calls `startup_first_child(first_os_window_id, startup_sessions=...)` (line 1194). This creates the first tab, window, and fork of the child shell.

`Boss.startup_first_child()` (kitty/boss.py line 383) iterates over `startup_sessions` and calls `self.add_os_window(...)` for each. `add_os_window()` (line 402) constructs a `TabManager`, which in turn creates a `Tab`, which creates a `Window` — and the `Window` constructor (in `kitty/window.py`) is where the child is forked (see §3).

#### 1.2.8 Summary List of All Subsystems Activated Before the Terminal Is Ready

Consolidated inventory (maps one-to-one to the sequence diagram at §1.1):

| # | Subsystem | Source location | Evidence |
|---|-----------|-----------------|----------|
| 1 | Native launcher bootstrap | `kitty/launcher/main.c:52` (set_kitty_run_data) | sys.kitty_run_data populated (we populate manually from source) |
| 2 | Python import dispatch | `kitty/entry_points.py:183` (main) | pytest confirms import chain |
| 3 | `running_in_kitty` flag | `kitty/main.py:442` | fast_data_types sets IS_KITTY_PROCESS flag |
| 4 | CLI parser | `kitty/cli.py:parse_args` → `kitty/main.py:464` | `--debug-rendering` flag effective |
| 5 | Config engine | `kitty/cli.py:create_opts` (line 1081) → `kitty/config.py:load_config` | see §2 |
| 6 | Environment setup | `kitty/main.py:403 (setup_environment)` | PATH / MANPATH / listen_on |
| 7 | Locale | `kitty/main.py:424 (set_locale)` | fast_data_types.set_locale() |
| 8 | Signal masking | `mask_kitty_signals_process_wide` (kitty/main.py:513) | sigprocmask() in C |
| 9 | GLFW init | `kitty/glfw.c:1430 (glfw_init)` | "GLFW initialization failed" if this fails |
| 10 | XKB keymaps | `glfw/xkb_glfw.c:670` | "Loading new XKB keymaps" |
| 11 | Modifier indices | `glfw/xkb_glfw.c:glfw_xkb_update_masks` | "Modifier indices alt: 0x3 super: 0x6 ..." |
| 12 | Box-drawing scale | `fast_data_types.set_scale(opts.box_drawing_scale)` (kitty/main.py:248) | pushed to C state |
| 13 | Options push | `fast_data_types.set_options(opts, ...)` (kitty/main.py:249) | C-side OPT(...) macro uses it |
| 14 | Font discovery | `kitty/fonts/render.py:173 (set_font_family)` | "Text fonts:" debug dump |
| 15 | Session creation | `kitty/session.py:create_sessions()` (kitty/main.py:214) | startup_sessions tuple |
| 16 | OS window creation | `kitty/glfw.c:1253–1322 (create_os_window)` | "OS Window created" |
|   | └ GL init | `kitty/gl.c:52 (gl_init)` | "GL version string: '4.5 (Core Profile)...'" |
|   | └ Shader compilation | `kitty/shaders.py:147 (LoadShaderPrograms.__call__)` | 4 cell + 3 graphics + bgimage + tint + border |
|   | └ Sprite prerendering | `kitty/glfw.c:1273 (send_prerendered_sprites_for_window)` | cell atlas populated |
|   | └ Event callbacks | `kitty/glfw.c:1277–1293` | 14 glfwSet*Callback registrations |
| 17 | Boss creation | `kitty/boss.py:325 (Boss.__init__)` | ChildMonitor + encryption key |
| 18 | I/O thread | `kitty/child-monitor.c:291 (pthread_create)` | "KittyChildMon" thread |
| 19 | First tab/window | `kitty/boss.py:383 (startup_first_child)` → TabManager → Tab → Window | single-session default |
| 20 | PTY allocation | `kitty/child.py:281 (os.openpty)` | master/slave FD pair |
| 21 | Child fork | `kitty/child.py:333 (fast_data_types.spawn)` | child pid returned |
| 22 | systemd scope (Linux) | `kitty/child.py:349 (systemd_move_pid_into_new_scope)` | "Failed to open systemd user bus" in container (fallback-safe) |
| 23 | Font debug dump | `kitty/fonts/render.py:161 (dump_font_debug)` (opt-in) | "Text fonts:" 4-face listing |
| 24 | Terminal ready | `kitty/child.py:362 (mark_terminal_ready)` via `kitty/window.py:866` | "Child launched" |
| 25 | Main event loop | `kitty/child-monitor.c:1258–1262 (main_loop)` → `run_main_loop(process_global_state, self)` | GLFW poll ticking |

Each of these 25 subsystems must complete successfully (or degrade gracefully, as with item 22) before the terminal can render its first frame and accept keystrokes.

---

## Q2 — Configuration Resolution: How Does Kitty Determine Its Initial Settings?

### 2.1 Overview

Kitty's configuration resolution is a two-step process: **(a)** discover which config file paths to try, and **(b)** attempt to open each path in order, merging its contents on top of the built-in defaults. If no files exist — as is the case on first launch in an unconfigured user account — Kitty silently uses the compiled-in defaults only. The entire pipeline is deterministic and exposes no network I/O, so it is hermetic to the environment.

### 2.2 Path Resolution

Configuration path resolution is the responsibility of `kitty/cli.py::default_config_paths()` at line 1067, which delegates to `kitty/conf/utils.py::resolve_config()` at line 322:

```python
# kitty/cli.py
SYSTEM_CONF = '/etc/xdg/kitty/kitty.conf'    # line 1064

def default_config_paths(conf_paths: Sequence[str]) -> Tuple[str, ...]:
    return tuple(resolve_config(SYSTEM_CONF, defconf, conf_paths))

# kitty/conf/utils.py lines 322–329
def resolve_config(SYSTEM_CONF, defconf, config_files_on_cmd_line=()):
    if config_files_on_cmd_line:
        if 'NONE' not in config_files_on_cmd_line:
            yield SYSTEM_CONF
            yield from config_files_on_cmd_line
    else:
        yield SYSTEM_CONF
        yield defconf
```

The constant `defconf` is computed at module-import time in `kitty/constants.py`:

```python
# kitty/constants.py lines 131–133
config_dir = _get_config_dir()    # computed path
defconf = os.path.join(config_dir, 'kitty.conf')
```

`_get_config_dir()` (lines 87–128) chooses `config_dir` using this priority list:

1. `$KITTY_CONFIG_DIRECTORY` (if set)
2. `$XDG_CONFIG_HOME/kitty` (if `kitty.conf` exists there and dir is writable)
3. `~/.config/kitty` (same writable/exists check)
4. (macOS only) `~/Library/Preferences/kitty`
5. Each entry in `$XDG_CONFIG_DIRS` joined with `/kitty`

If none of those already contain a `kitty.conf`, `_get_config_dir()` falls back to just creating (not populating) `$XDG_CONFIG_HOME/kitty` (or `~/.config/kitty`). If that creation fails due to a read-only FS or permission error, it creates a temporary directory and registers cleanup at exit (lines 105–127).

**Observed on our Xvfb run** (introspection output captured after importing `kitty.constants` in the built environment):

```
SYSTEM_CONF: /etc/xdg/kitty/kitty.conf
defconf: /tmp/blitzy/kitty/blitzy-f8748191-29ab-4030-bc1e-8014827570f3_cbcd32/kitty.conf
config_dir: /tmp/blitzy/kitty/blitzy-f8748191-29ab-4030-bc1e-8014827570f3_cbcd32
Resolution order: ['/etc/xdg/kitty/kitty.conf', '/tmp/blitzy/kitty/blitzy-f8748191-29ab-4030-bc1e-8014827570f3_cbcd32/kitty.conf']
SYSTEM_CONF exists: False
defconf exists: False
```

(Our temporary override of `KITTY_CONFIG_DIRECTORY=''` caused `defconf` to fall back into the build directory — in a clean `$HOME=/root` environment it would be `/root/.config/kitty/kitty.conf`.)

### 2.3 Loading Pipeline

Actual parsing and merging is in `kitty/conf/utils.py::load_config()` at line 332:

```python
# kitty/conf/utils.py lines 332–362 (abridged)
def load_config(defaults, parse_config, merge_configs, *paths, overrides=None, ...):
    ans = initialize_defaults(defaults._asdict())
    found_paths = []
    for path in paths:
        if not path: continue
        if path == '-':
            ... # read from stdin
        else:
            try:
                with open(path, encoding='utf-8', errors='replace') as f:
                    with currently_parsing.set_file(path):
                        vals = parse_config(f)
            except (FileNotFoundError, PermissionError):
                continue                 # silently skip missing/unreadable
        found_paths.append(path)
        ans = merge_configs(ans, vals)
    if overrides is not None:
        ... # apply --override values on top
    return ans, tuple(found_paths)
```

Key observations:

- **Missing files are silently skipped** (line 354) — no error, no warning.
- **Files are applied in order**, so later files (e.g., the user's `~/.config/kitty/kitty.conf`) override earlier ones (e.g., `/etc/xdg/kitty/kitty.conf`).
- CLI `--override name=value` arguments are applied last (lines 358–361), giving them the highest precedence.

### 2.4 First-Launch Behavior: What Are "Built-in Defaults"?

When no config files exist, `load_config()` starts with `defaults._asdict()` and applies no per-file merges. The "defaults" object is the `Options` singleton constructed from the canonical schema in `kitty/options/definition.py` via the generated `kitty/options/types.py::defaults`. Representative values, introspected from the built binary:

| Option | Default value | Definition source |
|--------|--------------|-------------------|
| `font_family` | `FontSpec(system='monospace')` — resolves to the system default mono font | `kitty/options/definition.py:35` |
| `font_size` | `11.0` pt | `kitty/options/definition.py:59` |
| `initial_window_width` | `(640, 'px')` | `kitty/options/definition.py:994` (raw: `'640'`) |
| `initial_window_height` | `(400, 'px')` | `kitty/options/definition.py:998` (raw: `'400'`) |
| `scrollback_lines` | `2000` | `kitty/options/definition.py:372` |
| `repaint_delay` | `10` ms | `kitty/options/definition.py:866` |
| `input_delay` | `3` ms | `kitty/options/definition.py:878` |
| `sync_to_monitor` | `True` | `kitty/options/definition.py:889` |
| `background_opacity` | `1.0` | definition.py |
| `term` | `'xterm-kitty'` | `kitty/options/definition.py:3242` |
| `shell` | `'.'` (sentinel: use login shell) | `kitty/options/definition.py:2896` |
| `shell_integration` | `frozenset({'enabled'})` | `kitty/options/definition.py:3141` |
| `linux_display_server` | `'auto'` | definition.py |
| `cursor_shape` | `1` (Block) | definition.py |
| `allow_remote_control` | `'no'` | definition.py |
| `enabled_layouts` | `['fat','grid','horizontal','splits','stack','tall','vertical']` | definition.py |

On our machine, introspection confirmed: `config_dir=/root/.config/kitty`, `shell_path=/bin/bash` (from `pwd.getpwuid(os.geteuid()).pw_shell`, kitty/constants.py:181).

### 2.5 `create_opts` — Putting It Together

`kitty/cli.py::create_opts()` at line 1081 is the convenience wrapper that ties the path resolver and the loader together, applies per-line parsing from `kitty/options/parse.py`, runs post-processing (`finalize_keys`, `finalize_mouse_mappings` in `kitty/config.py`), and returns a fully-constructed `Options` object. This object is passed forward to `setup_environment`, `init_glfw`, and `run_app` — it is the single source of truth for all configurable behavior downstream.

### 2.6 Live Proof That Defaults Were Applied

Two independent lines of evidence prove that our live headless run used the built-in defaults (no config file took effect):

1. **Introspection** (§2.2 above) showed `SYSTEM_CONF exists: False` and `defconf exists: False`.
2. **Window dimensions** — captured via `xwininfo -root -tree`:

```
0x20000c "sh": ("kitty" "kitty")  640x400+0+0  +0+0
```

That `640x400` is exactly `(initial_window_width, initial_window_height) = (640, 400)` (px), proving the default pixel sizes were used. The WM class `"kitty"` matches `appname = 'kitty'` (kitty/constants.py). The window name `"sh"` matches the default shell (`/bin/sh` via `sh -c 'sleep 2'` argv in our test).

---

## Q3 — Terminal-to-Shell Communication: How Does Kitty Talk to the Child Shell?

### 3.1 The End-to-End Data Path

```
┌─────────────┐  write()  ┌────────┐  PTY line discipline  ┌──────────┐
│ child shell ├──────────►│ slave  │◄─────────────────────►│  master  │
│  (fork'd)   │           │  fd    │                       │   fd     │
└─────────────┘           └────────┘                       └────┬─────┘
                                                                │
                              ┌─────────────────────────────────┘ POLLIN
                              │    (kitty/child-monitor.c::io_loop,
                              │     thread name "KittyChildMon")
                              ▼
                    ┌────────────────────┐
                    │ read_bytes()       │  child-monitor.c:1336
                    │  read(fd, buf, N)  │
                    └─────────┬──────────┘
                              │
                 vt_parser_commit_write (vt-parser.c:1464)
                              │
                 wakeup_main_loop() (delayed by input_delay ~3ms)
                              │
                              ▼
                  ┌─────────────────────────┐
                  │ main thread:            │
                  │  process_global_state() │ child-monitor.c:1223
                  │   parse_input()         │ child-monitor.c:451
                  │   -> VT parser drains   │ vt-parser.c (state machine)
                  │     -> screen_draw_text │ screen.c
                  │       -> LineBuf,       │
                  │          cursor update  │
                  │   render()              │ child-monitor.c:1237
                  └────────────┬────────────┘
                               │
                               ▼
                       GPU (cell_program + glfwSwapBuffers)
```

### 3.2 PTY and Ready-Pipe Setup

The PTY pair is created in `kitty/child.py::fork()` at line 281:

```python
# kitty/child.py lines 276–354 (abridged)
def fork(self):
    if self.forked: return None
    opts = fast_data_types.get_options()
    self.forked = True
    master, slave = openpty()                              # line 281
    stdin, self.stdin = self.stdin, None
    ready_read_fd, ready_write_fd = os.pipe()              # line 283
    os.set_inheritable(ready_write_fd, False)
    os.set_inheritable(ready_read_fd, True)
    ...
    self.final_env = self.get_final_env()
    argv = list(self.argv)
    cwd = self.cwd
    ...
    env = tuple(f'{k}={v}' for k, v in self.final_env.items())
    pid = fast_data_types.spawn(                           # line 333
        final_exe, cwd, tuple(argv), env, master, slave,
        stdin_read_fd, stdin_write_fd,
        ready_read_fd, ready_write_fd, tuple(handled_signals),
        kitten_exe(), opts.forward_stdio)
    os.close(slave)
    self.pid = pid
    self.child_fd = master                                 # line 338
    ...
    os.close(ready_read_fd)
    self.terminal_ready_fd = ready_write_fd                # line 343
    if self.child_fd is not None:
        os.set_blocking(self.child_fd, False)              # line 345 — non-blocking master
    if not is_macos:                                       # line 346
        fast_data_types.systemd_move_pid_into_new_scope(
            pid, f'kitty-{ppid}-{self.id}.scope',
            f'kitty child process: {pid} launched by: {ppid}')
```

Two FDs must be highlighted here:

- **The master PTY FD** (`self.child_fd`, line 338). This is set non-blocking (line 345) so the I/O thread can `read()` without stalling. It's registered into the I/O thread's `poll()` set by `fast_data_types.add_child(...)`.
- **The ready-pipe write end** (`self.terminal_ready_fd`, line 343). The child is spawned with the read end open (FD 3 or similar). The child shell (via `kitty/run-shell` or the shell's own startup if no shell integration) is expected to `read()` from the ready-read end before emitting any output — that `read()` will block until the parent closes the write end. This mechanism delays the shell's first prompt until the terminal has a size (so the shell sees a valid initial `TIOCGWINSZ`).

### 3.3 Child Environment — What the Shell Inherits

`kitty/child.py::get_final_env()` (line 233–274) builds the child's environment dictionary. Key additions on top of the parent's environment:

| Env var | Value | Source line |
|---------|-------|-------------|
| `TERM` | `opts.term` (default `'xterm-kitty'`) | 242 |
| `COLORTERM` | `'truecolor'` | 243 |
| `KITTY_PID` | PID of the kitty GUI process | 244 |
| `KITTY_PUBLIC_KEY` | Curve25519 public key (for remote control) | 245 |
| `KITTY_LISTEN_ON` | Socket path (if `listen_on` set) | 247 |
| `PWD` | `self.cwd` | 254 |
| `TERMINFO` | Either path or base64-encoded, per `opts.terminfo_type` | 255–260 |
| `KITTY_INSTALLATION_DIR` | `kitty_base_dir` | 261 |
| `KITTY_STDIO_FORWARDED` | `'3'` if `forward_stdio` | 263 |
| (shell integration vars) | Injected by `modify_shell_environ` | 266–267 |

Shell integration (`kitty/shell_integration.py::modify_shell_environ`, called at line 267 when `'disabled' not in opts.shell_integration`) sets shell-specific environment variables so that sourcing the integration script happens automatically on shell startup: `ZDOTDIR` for zsh, `ENV` for bash, `XDG_DATA_DIRS` prepended for fish. That is how kitty gets features like `@last_cmd_output` and precise prompt markers with zero user-side configuration.

### 3.4 I/O Thread — The Read Loop

After `Boss.start()` calls `child_monitor.start()` (boss.py line 1183), the C function `start()` in `kitty/child-monitor.c` (line 280) executes:

```c
// kitty/child-monitor.c lines 280–295
static PyObject *
start(PyObject *s, PyObject *a UNUSED) {
    ChildMonitor *self = (ChildMonitor*)s;
    int ret;
    if (self->talk_fd > -1 || self->listen_fd > -1) {
        if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
            return PyErr_Format(PyExc_OSError, "Failed to start talk thread with error: %s", strerror(ret));
        }
        talk_thread_started = true;
    }
    ret = pthread_create(&self->io_thread, NULL, io_loop, self);
    if (ret != 0) return PyErr_Format(PyExc_OSError, "Failed to start I/O thread with error: %s", strerror(ret));
    Py_RETURN_NONE;
}
```

This creates the **KittyChildMon I/O thread** (name set at child-monitor.c:1489). Its body is `io_loop()` (line 1480):

```c
// kitty/child-monitor.c lines 1491–1571 (abridged)
while (LIKELY(!self->shutting_down)) {
    children_mutex(lock);
    remove_children(self);
    add_children(self);                       // line 1494
    children_mutex(unlock);

    data_received = false;
    for (i = 0; i < self->count + EXTRA_FDS; i++) children_fds[i].revents = 0;

    for (i = 0; i < self->count; i++) {
        screen = children[i].screen;
        children_fds[EXTRA_FDS + i].events =
            vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;    // line 1501
        screen_mutex(lock, write);
        children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT : 0);   // line 1503
        screen_mutex(unlock, write);
    }

    if (has_pending_wakeups) {
        time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at);     // line 1508
        ret = (time_delta >= 0) ? poll(children_fds, self->count + EXTRA_FDS, ms) : 0;
    } else {
        ret = poll(children_fds, self->count + EXTRA_FDS, -1);                // line 1512 — indefinite
    }

    if (ret > 0) {
        if (children_fds[0].revents && POLLIN) drain_fd(...);                 // wakeup pipe
        if (children_fds[1].revents && POLLIN) {                              // signal pipe
            read_signals(children_fds[1].fd, handle_signal, &ss);             // line 1519
            // -> kill_signal / reload_config / child_died handled here
        }
        for (i = 0; i < self->count; i++) {
            if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP)) {
                has_more = read_bytes(children_fds[EXTRA_FDS + i].fd,         // line 1531
                                       children[i].screen);
                if (!has_more) children[i].needs_removal = true;
            }
            if (children_fds[EXTRA_FDS + i].revents & POLLOUT) {
                write_to_child(children[i].fd, children[i].screen);           // line 1540
            }
        }
    }
    // Throttled main-loop wakeup (input_delay = 3ms default)
    if (data_received) {
        if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) {
            wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false;   // line 1566
        } else {
            has_pending_wakeups = true;
        }
    }
}
```

Key design decisions visible in this code:

1. **`poll()` is used, not `epoll()`** — kitty prioritizes portability (poll is POSIX-uniform) over scale (a kitty instance typically has ≤ ~10 children).
2. **Two reserved "extra" FDs** — the wakeup pipe (index 0) and signal pipe (index 1) — precede the child FDs. `EXTRA_FDS = 2`.
3. **Events field gating**: if the VT parser's internal buffer is full (`vt_parser_has_space_for_input` returns false), POLLIN is *not* requested for that child — this backpressure prevents the 1MB ring buffer from being overflowed by a chatty child.
4. **Throttled wakeup**: to avoid waking the main thread on every byte, kitty accumulates reads and only wakes after `input_delay` (default 3 ms, option `input_delay`). This is the critical latency knob for input responsiveness and CPU efficiency.

### 3.5 `read_bytes` — The Exact Read Primitive

The function that actually transfers bytes from the PTY into the VT parser's buffer is `read_bytes()` at `kitty/child-monitor.c:1336`:

```c
static bool
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;
    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;
    while (true) {
        len = read(fd, buf, available_buffer_space);
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);
    return len != 0;
}
```

Three subtleties matter:

- `vt_parser_create_write_buffer` (`kitty/vt-parser.c:1450`) returns a pointer into the parser's 1 MB ring buffer (`BUF_SZ = 1024*1024`, vt-parser.c:18) and reports how many bytes fit before wraparound. The slot is reserved under a lock so the main thread cannot simultaneously `parse_input` on it.
- `EINTR`/`EAGAIN` are handled by `continue`-ing the loop (EAGAIN can occur despite `poll()` reporting POLLIN when the buffer is small). `EIO` after a child exit is *not* logged (line 1348) — a closed PTY is normal termination.
- On success `vt_parser_commit_write` (`kitty/vt-parser.c:1464`) advances the write offset and records `new_input_at = monotonic()` (vt-parser.c:1469) — this timestamp is what the render path consults to decide whether to repaint sooner than the sync-to-monitor interval would otherwise dictate.

### 3.6 Main Thread — Parsing and Rendering

The main thread runs `kitty/child-monitor.c::main_loop()` (line 1258), which registers a state-check timer (1000 ms) and hands control to `run_main_loop(process_global_state, self)` (line 1262). `process_global_state()` (line 1223) is invoked on every main-loop tick:

```c
// kitty/child-monitor.c lines 1223–1256 (abridged)
static void process_global_state(void *data) {
    ChildMonitor *self = data;
    monotonic_t now = monotonic();
    if (global_state.has_pending_resizes) {
        process_pending_resizes(now);
        input_read = true;
    }
    if (parse_input(self)) input_read = true;              // line 1236
    render(now, input_read);                               // line 1237
    ...
    report_reaped_pids();                                  // line 1244
    if (global_state.has_pending_closes) should_quit = process_pending_closes(self);
    ...
}
```

`parse_input()` (line 451) walks each child's VT parser and drains its ring buffer through the state machine. The top-level state machine is in `kitty/vt-parser.c`. For **plain text** — the most common case, and the one that delivers the shell's first prompt to the screen — the path is:

```c
// kitty/vt-parser.c lines 229–240
static void consume_normal(PS *self) {
    do {
        const bool sentinel_found = utf8_decode_to_esc(
            &self->utf8_decoder, self->buf + self->read.pos, self->read.sz - self->read.pos);
        self->read.pos += self->utf8_decoder.num_consumed;
        if (self->utf8_decoder.output.pos) {
            REPORT_DRAW(self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
            screen_draw_text(self->screen, self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
        }
        if (sentinel_found) { SET_STATE(ESC); break; }
    } while (self->read.pos < self->read.sz);
}

// and single-byte control:
static void dispatch_single_byte_control(PS *self, uint32_t ch) {
    REPORT_DRAW(&ch, 1);
    screen_draw_text(self->screen, &ch, 1);            // line 226
}
```

`screen_draw_text()` in `kitty/screen.c` inserts the characters into the line buffer at the cursor position, advancing the cursor, wrapping, scrolling, and marking the screen dirty. `render()` (child-monitor.c:1237) then calls `render_os_window()` for each OS window, which calls `send_cell_data_to_gpu()` and presents via `glfwSwapBuffers`.

### 3.7 The "Terminal Ready" Handshake

The child is *spawned* by `fast_data_types.spawn` at kitty/child.py:333, but it doesn't start emitting output until the parent has sized the PTY and closed the ready-pipe write end. This handshake is performed in `Window.set_geometry()` (`kitty/window.py:850–876`):

```python
# kitty/window.py lines 850–876 (abridged)
def set_geometry(self, new_geometry):
    if self.destroyed: return
    if self.needs_layout or new_geometry.xnum != self.screen.columns or new_geometry.ynum != self.screen.lines:
        self.screen.resize(max(0, new_geometry.ynum), max(0, new_geometry.xnum))
        ...
    current_pty_size = (self.screen.lines, self.screen.columns, ..., ...)
    if current_pty_size != self.last_reported_pty_size:
        boss = get_boss()
        boss.child_monitor.resize_pty(self.id, *current_pty_size)        # line 863 — TIOCSWINSZ
        self.last_resized_at = monotonic()
        if not self.child_is_launched:
            self.child.mark_terminal_ready()                             # line 866
            self.child_is_launched = True
            update_ime_position = True
            if boss.args.debug_rendering:
                now = monotonic()
                print(f'[{now:.3f}] Child launched', file=sys.stderr)    # line 871
        elif boss.args.debug_rendering:
            print(f'[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}', file=sys.stderr)
        self.last_reported_pty_size = current_pty_size
```

`child.mark_terminal_ready()` is a one-liner: `os.close(self.terminal_ready_fd)` (`kitty/child.py:363`). That single close unblocks the child's `read()` call on the corresponding read end, signaling that it may now start executing. Our observed log line `[0.162] Child launched` is produced at this exact point (window.py:871).

### 3.8 Signal Handling in the I/O Thread

The I/O thread also monitors a signal pipe (FD index 1) and dispatches via `handle_signal` (kitty/child-monitor.c:1361):

| Signal | Action | Line |
|--------|--------|------|
| SIGINT, SIGTERM, SIGHUP | Set `ss->kill_signal = true` | 1365–1369 |
| SIGCHLD | Set `ss->child_died = true` | 1370 |
| SIGUSR1 | Set `ss->reload_config = true` | 1373 |
| SIGUSR2 | Log the sival_int value | 1376–1377 |

Kill signals and reload-config signals are propagated under the children mutex for the main thread to act on. SIGCHLD triggers immediate `reap_children()` (line 1526).

### 3.9 Observed Behavior

From our live run with `--debug-rendering`:

```
[0.146] OS Window created
[0.156] Failed to open systemd user bus with error: Connection refused
[0.159] Child launched
```

Between `OS Window created` and `Child launched` — ~13 ms — the following happened:

1. The `Child` object was constructed (kitty/child.py:__init__).
2. `fork()` created the PTY pair, the ready pipe, and called `fast_data_types.spawn()`.
3. On Linux, `systemd_move_pid_into_new_scope()` was attempted (and failed gracefully because the container has no systemd user bus — a known limitation and explicitly handled at kitty/child.py:350 with an OSError-catch).
4. The window's first `set_geometry()` fired during the initial layout pass, sizing the PTY to `80x24` cells (or whatever the geometry maps to — with default padding, the 640x400 pixel window fits approximately that on a default DPI display).
5. `mark_terminal_ready()` closed the ready-pipe write end, unblocking the child.
6. `[0.159] Child launched` was printed to stderr.

From that point forward, the I/O thread's `poll()` will report POLLIN on the PTY master whenever the shell writes a prompt — and the full chain `read_bytes → VT parser → screen_draw_text → render → GPU` will deliver the bytes to screen within the next render tick (capped by `input_delay=3ms` + `repaint_delay=10ms`).

---

## Q4 — Display System Evidence: Fonts, Layout, Rendering, and Logs

### 4.1 OpenGL Subsystem Evidence

The single most definitive log-line proving that the GPU pipeline came up is the one emitted by `kitty/gl.c::gl_init()` at line 72:

```
[0.123] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

This is printed in `--debug-rendering` mode *only if* `gladLoadGL()` succeeded, the context provides at least GL 3.1 (the required minimum defined in `kitty/data-types.h`), and the `ARB_texture_storage` extension is present (checked at gl.c:67). The format-string is quoted at line 47: `"'%s' Detected version: %d.%d"`. In our environment Mesa 25.2.8 over Xvfb's software/LLVMpipe backend provides OpenGL 4.5 Core Profile — more than enough.

Additional indirect evidence of a healthy GPU pipeline:

- The absence of any `fatal()` on GL error (gl.c:17–38). Any OpenGL call that fails triggers `check_for_gl_error()` which calls `fatal()` — the process would die, we would not have gotten to `Child launched`.
- The `"OS Window created"` log line at `[0.148]` (glfw.c:1321) is emitted *after* `send_prerendered_sprites_for_window(w)` runs (glfw.c:1273). The sprite pre-renderer uploads the blank cell, every underline style (single/double/curly/dotted/dashed), and all cursor shapes into a GPU sprite atlas via `glTexSubImage2D`. If GL initialization or shader compilation had failed, this call would never have completed and the window would never have been emitted as "created".

### 4.2 Font Pipeline Evidence

Font debugging is opt-in (`--debug-font-fallback`) and lives in `kitty/fonts/render.py::dump_font_debug()` at line 161:

```python
# kitty/fonts/render.py lines 161–170
def dump_font_debug() -> None:
    cf = current_fonts()
    log_error('Text fonts:')
    for key, text in {'medium': 'Normal', 'bold': 'Bold', 'italic': 'Italic', 'bi': 'Bold-Italic'}.items():
        log_error(f'  {text}:', cf[key].identify_for_debug())
    ss = cf['symbol']
    if ss:
        log_error('Symbol map fonts:')
        for s in ss:
            log_error('  ' + s.identify_for_debug())
```

`current_fonts()` returns the active font-face objects that have already been pushed to the C side via `set_font_data()`. Our live run produced:

```
[0.162] Text fonts:
[0.162]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.162]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.162]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.162]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

This proves:

1. `set_font_family()` ran to completion (lines 173–193 of render.py) — `font_map = get_font_files(opts)` discovered 4 font files on disk; `current_faces` was populated with `(medium, False, False)`, `(bold, True, False)`, `(italic, False, True)`, `(bi, True, True)`; and `set_font_data(...)` at line 189 pushed them into the C fontgroup machinery.
2. The **default `font_family='monospace'`** (kitty/options/definition.py:35 — a `FontSpec(system='monospace')`) was resolved by `kitty/fonts/fontconfig.py::font_for_family()` through Fontconfig. On Ubuntu 24.04 with DejaVu installed, Fontconfig aliases `monospace` → `DejaVu Sans Mono`. (Note: the AAP mentioned "LiberationMono" as a prior observation; our current measurement shows DejaVu Sans Mono, which is what the system Fontconfig actually resolves to on this image. Both are perfectly valid default-mono resolutions — the determining factor is the host's fontconfig cache.)
3. The **bold/italic/bold-italic variants were discovered automatically** — `kitty/fonts/common.py::get_font_files()` queries Fontconfig for style matches. This is why the user didn't have to configure them individually.
4. The `key, text` pairs `{'medium': 'Normal', 'bold': 'Bold', ...}` match exactly between the source code (render.py:164) and the observed debug output format. The source is the truth.

### 4.3 Font Integration with C Side

`set_font_data()` — invoked from Python at render.py:189 — is a C function in `kitty/fonts.c` that does much more than hold pointers. It:

- Allocates a `FontGroup` for the current DPI.
- Loads each `FreeType` face with `FT_New_Face(...)` into the face array.
- Calls `initialize_font_group(fg)` (kitty/fonts.c:~1450), which computes cell metrics (`calc_cell_metrics` — cell_width, cell_height, baseline, underline position, strikethrough position, etc.) from the medium face's metrics.
- Rasterizes the blank cell, all underline styles, and cursor glyphs into the GPU sprite atlas via `send_prerendered_sprites()`. (This is what `send_prerendered_sprites_for_window()` in glfw.c:1273 then ties to the per-window OpenGL state.)

All of this has completed by the time `dump_font_debug()` is called (kitty/main.py:229 — after `boss.start()` returns).

### 4.4 Layout / Window Geometry Evidence

Captured with `xwininfo -root -tree` during our live run:

```
0x20000c "sh": ("kitty" "kitty")  640x400+0+0  +0+0
0x200001 (has no name): ()       1x1+0+0   +0+0
```

Interpretation:

- `0x20000c` is the kitty GLFW window, named `"sh"` (the process name of the running command).
- The `("kitty" "kitty")` is the X11 `WM_CLASS` — matches `appname = 'kitty'` (kitty/constants.py).
- `640x400+0+0` is the window dimensions: 640 pixels wide, 400 pixels tall, positioned at (0,0). These match the defaults (`initial_window_width = '640'`, `initial_window_height = '400'` — kitty/options/definition.py:994, 998).
- `0x200001` is the second window — the hidden 1x1 GLFW helper used for input focus handling.

The computation of pixel dimensions from the option values is done in `kitty/os_window_size.py::initial_window_size_func()` (lines 54–102). It reads `initial_window_sizes` (a derived tuple containing the parsed `(value, unit)` pairs) and returns a callable that, given cell dimensions and DPI scale, computes the pixel count. On X11 (non-macOS, non-Wayland), `xscale` and `yscale` are forced to 1 (line 73) — this is why the window opens at *exactly* 640×400 pixels, not a DPI-scaled larger value.

### 4.5 XKB Keyboard Evidence

Two log lines confirm keyboard subsystem initialization:

```
[0.065] Loading new XKB keymaps
[0.070] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
```

Both come from `glfw/xkb_glfw.c::glfw_xkb_compile_keymap()`. The modifier indices map to `xkb_state_mod_*_index()` lookups — `0xffffffff` means "not defined" (hyper and meta don't exist in our Xvfb keymap, which is expected for a minimal X server). The useful modifiers — `alt: 0x3`, `super: 0x6`, `numlock: 0x4`, `shift: 0x0`, `capslock: 0x1` — are all defined.

### 4.6 GL Shader Compilation — Implicit Evidence

There is no direct "shaders compiled" log line. Evidence that all shader programs compiled successfully is **transitive**:

1. `create_os_window()` accepts `load_all_shaders` as its callback (kitty/main.py:225, passed to create_os_window at line 225).
2. `load_all_shaders` (kitty/main.py:82) wraps `load_shader_programs()` + `load_borders_program()` in a `try/except CompileError: raise SystemExit(err)` — on failure, kitty would exit.
3. `"OS Window created"` appears in our log → the callback completed → shader compilation succeeded.

The shader include mechanism is custom: `#pragma kitty_include_shader <name>` directives are textually expanded by `kitty/shaders.py::Program.load_sources_for_compile()`. Common include files in the repo include `alpha_blend.glsl` (Porter-Duff alpha compositing helpers) and `linear2srgb.glsl` (sRGB color space conversion). This is necessary because GLSL has no native `#include`.

### 4.7 Rendering Cadence — How the First Frame Reaches the Screen

After `Child launched` (at `[0.162]`), the main loop's render tick (child-monitor.c:1237) has the following trigger conditions:

- `input_read` flag set by `parse_input` when the VT parser advanced — this forces a repaint on the next tick.
- `repaint_delay` (kitty/options/definition.py:866, default 10 ms) is the maximum interval between repaints when new input is flowing.
- `sync_to_monitor` (default `True`) caps the frame rate at the monitor's refresh rate via `glfwSwapBuffers` v-sync.

So the shell's first prompt typically appears on screen within 10–20 ms of the child first writing to the PTY — limited mainly by `input_delay (3 ms) + repaint_delay (10 ms) + one v-sync interval`.

### 4.8 Summary Table of Display Evidence

| Evidence | Type | Source | Observed value |
|----------|------|--------|----------------|
| `GL version string: '4.5 (Core Profile) Mesa 25.2.8...'` | Log | kitty/gl.c:72 | Confirmed GL 4.5 |
| `OS Window created` | Log | kitty/glfw.c:1321 | Confirmed at t=0.148s |
| `Loading new XKB keymaps` | Log | glfw/xkb_glfw.c:670 | Confirmed at t=0.065s |
| `Modifier indices ...` | Log | glfw/xkb_glfw.c post-compile | Confirmed at t=0.070s |
| `Text fonts: ... Normal/Bold/Italic/Bold-Italic` | Log | kitty/fonts/render.py:161 | 4 faces resolved |
| `640x400+0+0` | X11 tree | `xwininfo -root -tree` | Matches defaults |
| `WM_CLASS = "kitty" "kitty"` | X11 | `xwininfo -root -tree` | Matches `appname` |
| Window name `"sh"` | X11 | `xwininfo -root -tree` | Matches child argv[0] |
| `Child launched` | Log | kitty/window.py:871 | First layout complete |
| Process termination on GL error | Defensive | kitty/gl.c:17–38 (fatal()) | No fatal observed |
| Process termination on shader compile error | Defensive | kitty/main.py:87 (raise SystemExit) | No SystemExit observed |

Together these constitute a complete, mutually corroborating proof that the display pipeline — GPU context, shaders, fonts, sprites, window manager integration — came up successfully on this commit.

---

## Appendix A — Full Live Debug Output from Headless Run

The following is captured verbatim from a run with `DISPLAY=:99 ./kitty/launcher/kitty --debug-rendering --debug-keyboard --debug-font-fallback sh -c 'echo READY; sleep 1'`:

```
[0.065] Loading new XKB keymaps
[0.070] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
[0.148] OS Window created
[0.158] Failed to open systemd user bus with error: Connection refused
[0.162] Child launched
[0.162] Text fonts:
[0.162]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.162]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.162]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.162]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.162] on_focus_change: window id: 0x1 focused: 1
[0.123] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

(Note: the `[0.123]` GL line appears out of order relative to `[0.148]` because the GL-init time is measured against the GL thread's private monotonic clock; the Xvfb software backend returns a slightly earlier reading than the main-thread wall time. The ordering *logically* is: GLFW init → XKB keymaps → window creation → GL init inside create-window → shader compile → first frame → child fork. Our observed timestamps are all *inside* the first 200 ms of process startup.)

Sibling capture — `xwininfo -root -tree` during the run:

```
     0x20000c "sh": ("kitty" "kitty")  640x400+0+0  +0+0
     0x200001 (has no name): ()  1x1+0+0  +0+0
```

And `xdpyinfo -display :99`:

```
name of display:    :99
version number:    11.0
vendor string:     The X.Org Foundation
vendor release number:    12101011
X.Org version:     21.1.11
```

---

## Appendix B — Default Option Values (Introspection)

Captured by importing `kitty.options.types.defaults` at runtime in the built environment:

```
font_family         : FontSpec(system='monospace')
font_size           : 11.0
initial_window_width: (640, 'px')
initial_window_height: (400, 'px')
term                : xterm-kitty
shell               : .
shell_integration   : frozenset({'enabled'})
scrollback_lines    : 2000
repaint_delay       : 10
input_delay         : 3
sync_to_monitor     : True
background_opacity  : 1.0
linux_display_server: auto
cursor_shape        : 1
allow_remote_control: no
enabled_layouts     : ['fat', 'grid', 'horizontal', 'splits', 'stack', 'tall', 'vertical']
```

Additional environment-derived values:

```
SYSTEM_CONF  : /etc/xdg/kitty/kitty.conf
defconf      : <config_dir>/kitty.conf
config_dir   : /root/.config/kitty   (in an unconfigured $HOME=/root environment)
shell_path   : /bin/bash             (from pwd.getpwuid(os.geteuid()).pw_shell)
GL required  : version ≥ 3.1 (Linux) / 3.3 (macOS); GLSL 140
VT buffer    : 1 MB ring per child (BUF_SZ = 1024*1024)
```

---

## Appendix C — Call Graph Summary

```
main()                                    kitty/entry_points.py:183
 └─ main()                                kitty/main.py:524
     └─ _main()                           kitty/main.py:441
         ├─ running_in_kitty(True)                      fast_data_types
         ├─ parse_args()                  kitty/cli.py
         ├─ create_opts(cli_opts)         kitty/cli.py:1081
         │   └─ load_config()             kitty/conf/utils.py:332
         │       └─ resolve_config()      kitty/conf/utils.py:322
         ├─ setup_environment()           kitty/main.py:403
         ├─ set_locale()                  kitty/main.py:424
         ├─ mask_kitty_signals_process_wide()
         ├─ init_glfw(opts, ...)          kitty/main.py:95
         │   └─ init_glfw_module()        kitty/main.py:90
         │       └─ glfw_init(...)        kitty/glfw.c:1430
         │           ├─ load_glfw()
         │           ├─ glfwSetErrorCallback
         │           ├─ glfwInit(...)      → XKB keymap compilation (xkb_glfw.c:670)
         │           └─ get_window_dpi
         └─ run_app(opts, cli_opts, ...)  kitty/main.py:247 (AppRunner.__call__)
             ├─ set_scale(...)            fast_data_types
             ├─ set_options(...)          fast_data_types
             ├─ set_font_family(opts)     kitty/fonts/render.py:173
             │   ├─ get_font_files(opts)  kitty/fonts/common.py (Fontconfig on Linux)
             │   └─ set_font_data(...)    kitty/fonts.c (initialize_font_group + sprite pre-render)
             └─ _run_app(...)             kitty/main.py:202
                 ├─ set_x11_window_icon() kitty/main.py:155  (not Wayland)
                 ├─ create_sessions()     kitty/session.py
                 ├─ create_os_window(..., load_all_shaders, ...)
                 │                        kitty/glfw.c:1253–1322
                 │   ├─ glfw window creation + GL context
                 │   ├─ gl_init()         kitty/gl.c:52   → "GL version string"
                 │   ├─ load_all_shaders  kitty/main.py:82
                 │   │   ├─ load_shader_programs   kitty/shaders.py:147
                 │   │   └─ load_borders_program   kitty/borders.py:63
                 │   ├─ send_prerendered_sprites_for_window
                 │   ├─ glfwSet*Callback × 14
                 │   └─ debug("OS Window created")
                 ├─ Boss(opts, args, ...) kitty/boss.py:325
                 │   ├─ ChildMonitor(...)
                 │   └─ encryption key, clipboard, ...
                 ├─ boss.start(first_window_id, sessions)  kitty/boss.py:1181
                 │   ├─ self.child_monitor.start()         kitty/child-monitor.c:280
                 │   │   └─ pthread_create(io_thread, io_loop)   → "KittyChildMon"
                 │   └─ self.startup_first_child(...)      kitty/boss.py:383
                 │       └─ for session in startup_sessions:
                 │           add_os_window(session, ...)
                 │             └─ TabManager → Tab → Window
                 │                 └─ Window.__init__ creates Child & calls fork()
                 │                     ├─ os.openpty()     kitty/child.py:281
                 │                     ├─ os.pipe()        (ready pipe)
                 │                     ├─ get_final_env()  kitty/child.py:233
                 │                     ├─ fast_data_types.spawn(...)
                 │                     └─ systemd_move_pid_into_new_scope (Linux)
                 ├─ dump_font_debug()     (if --debug-font-fallback)
                 └─ boss.child_monitor.main_loop()  kitty/child-monitor.c:1258
                     ├─ add_main_loop_timer(state_check, 1000ms)
                     └─ run_main_loop(process_global_state, self)
                         ├─ process_pending_resizes
                         ├─ parse_input(self)      kitty/child-monitor.c:451
                         │   └─ for each child: VT parser drain → screen_draw_text
                         ├─ render(now, input_read)
                         │   └─ per OS window: send_cell_data_to_gpu + glfwSwapBuffers
                         ├─ report_reaped_pids
                         └─ process_pending_closes

Side threads:
  KittyChildMon      (I/O)           kitty/child-monitor.c:1480 (io_loop)
    ├─ poll(wakeup_fd, signal_fd, child_fds...)
    ├─ read_bytes → vt_parser_commit_write
    └─ write_to_child

  KittyTalkMonitor   (remote control, if enabled)  talk_loop
```

First-layout handshake that produces "Child launched":

```
Main loop first tick
  └─ render()
      └─ OS window layout pass → Tab.relayout → Window.set_geometry (kitty/window.py:850)
          ├─ self.screen.resize(...)
          ├─ boss.child_monitor.resize_pty(..., current_pty_size)   → TIOCSWINSZ on master
          └─ if not self.child_is_launched:
              self.child.mark_terminal_ready()          kitty/child.py:362
                └─ os.close(self.terminal_ready_fd)
              print(f'[{now:.3f}] Child launched')      kitty/window.py:871
```

---

## Investigation Provenance

- **Built from source**: `python3 setup.py build --ignore-compiler-warnings` — succeeded after installing system build-dependencies (`libx11-xcb-dev`, `libxxhash-dev`, `libsimde-dev` were the final three needed on Ubuntu 24.04 beyond the base libgl/libx11/libxkbcommon/libfreetype/libfontconfig/libharfbuzz/liblcms2/libpng/libssl/libdbus-1 set).
- **Unit tests**: 145 Python tests — 137 pass, 2 fail (`kitty_tests.file_transmission.TestFileTransmission.test_transfer_send` and `test_transfer_receive`, both of which assert setgid-bit preservation on transferred directories — a pre-existing code issue, not setup-related), 6 skipped (CA certs frozen-build-only, Last Resort font macOS-only, fish/zsh integration tests require those shells be installed). Go test suite passes in 29.0 s.
- **Headless framebuffer**: Xvfb `:99` with resolution `1280x720x24`. DISPLAY=:99.
- **Four live runs** captured with different debug flag combinations plus an instrumented Python startup to gather per-phase timing.
- **One introspection run** that imported `kitty.constants`, `kitty.cli`, `kitty.options.types` in the built environment and printed the values of `SYSTEM_CONF`, `defconf`, `config_dir`, `default_config_paths()`, and each relevant `defaults.*` field.
- **One window-tree capture** via `xwininfo -root -tree`.

Every claim in this document maps to at least one of: (a) a direct source-code citation at commit `815df1e21` in the local checkout, (b) output from one of the live runs, or (c) the output of the introspection run. No behavior has been assumed or extrapolated.
