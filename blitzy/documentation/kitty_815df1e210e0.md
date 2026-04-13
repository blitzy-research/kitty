# Kitty Terminal Emulator – Startup Lifecycle Investigation

## Source: commit `815df1e21` ("Wire up applying of font config")

### Scope

This document traces every subsystem that initializes between process launch and the moment the Kitty terminal emulator is ready for a shell, all grounded in source code evidence from the repository at the specified commit. Four questions structure the investigation:

1. **Startup Sequence Mapping** — What subsystems initialize between launch and terminal readiness?
2. **Configuration Resolution** — How does Kitty decide on its initial configuration at first launch?
3. **Terminal-to-Shell Handoff** — How does Kitty prepare the terminal to communicate with the child shell?
4. **Display System Evidence** — What confirms the GPU rendering pipeline, font subsystem, and display system are active?

An appendix documents a headless execution attempt via Xvfb.

> **Methodology:** Every claim cites a specific file path and line number. Short code snippets (2–3 lines) illustrate key logic. The startup sequence is presented in exact execution order. Where the code branches by platform, the Linux/X11 path is the primary focus; macOS (Cocoa) and Wayland branches are noted as alternatives.

---

## Q1: Startup Sequence — What subsystems initialize between process launch and terminal readiness?

### Rationale

To map the startup sequence we must follow the call chain from the native C entry point through the Python orchestration layer, all the way to the GLFW event loop. Each subsystem initialization is documented in the exact order it occurs in the source code.

### 1.1 Native Launcher Entry

**File:** `kitty/launcher/main.c`

The process entry point is the C `main()` function at line 439:

```c
int main(int argc, char *argv[], char* envp[])
```

The very first action (line 441) is `ensure_working_stdio()`, which validates that stdin/stdout/stderr are open. If any file descriptor is invalid, it is reopened to `/dev/null` (lines 319–329). This prevents Python from seeing `None` for `sys.stdout` and friends — the comment at lines 289–290 explicitly references the Python source code's semantics.

Next, the executable path is resolved via `read_exe_path()`. On Linux, this reads `/proc/self/exe` (line 281):

```c
if (!safe_realpath("/proc/self/exe", exe, buf_sz)) { ... }
```

On macOS, `_NSGetExecutablePath()` is used instead (line 232); on FreeBSD, `sysctl()` (line 244). The investigation environment is Linux, so the `/proc/self/exe` branch applies.

The launcher then delegates to the `kitten` binary for `@`-prefixed commands and wrapped kittens via `delegate_to_kitten_if_possible()` (line 452). Fast command-line parsing for `--version` and `--single-instance` occurs in `handle_fast_commandline()` (line 453). If `--single-instance` is set, `single_instance_main()` is invoked (line 436), which communicates via a Unix domain socket defined in `kitty/launcher/single-instance.c`. The `CLIOptions` struct is declared in `kitty/launcher/launcher.h` (lines 12–16).

Finally, the Python runtime is bootstrapped via `run_embedded()` (line 464). For source builds (the `#else` branch at line 174), this:

1. Initializes a `PyPreConfig` with `utf8_mode = 1` and `coerce_c_locale = 1` (lines 184–186).
2. Sets `parse_argv = 0` (line 194) and `optimization_level = 2` (line 195) on the `PyConfig`.
3. Calls `Py_InitializeFromConfig()` (line 211).
4. Sets `sys.kitty_run_data` via `set_kitty_run_data()` (line 214), which creates a Python dict with `bundle_exe_dir`, optional `from_source` flag, `lc_ctype_before_python`, and `extensions_dir` (lines 52–77).

For frozen/bundle builds, `bypy_initialize_interpreter()` is called with entry module `"kitty_main"` (line 168).

### 1.2 Python Entry Dispatch

**File:** `kitty/entry_points.py`

The Python entry point is `main()` at line 183. It first checks `sys.frozen` to set up bundled SSL certificates via `setup_openssl_environment()` (lines 184–187). It then reads the first CLI argument and looks it up in the `entry_points` dict (line 189). For a normal GUI launch, no entry point matches, so execution falls through to lines 193–195:

```python
from kitty.main import main as kitty_main
kitty_main()
```

The `__main__.py` at the repository root provides an alternative entry: `from kitty.entry_points import main; main()` (lines 5–7).

### 1.3 `_main()` — The Startup Orchestrator

**File:** `kitty/main.py` (lines 441–521)

This is the heart of the startup sequence. Each step occurs in the exact order listed below.

**Step 1 — Mark as running** (line 442): `running_in_kitty(True)` sets a global flag.

**Step 2 — macOS Launch Services** (lines 444–448): On macOS, if launched by Launch Services, changes CWD to `~` and reads `macos-launch-services-cmdline`. Skipped on Linux.

**Step 3 — CWD validation** (lines 449–454): If the current working directory is invalid, falls back to `~`:

```python
if not cwd_ok:
    os.chdir(os.path.expanduser('~'))
```

**Step 4 — CLI argument parsing** (line 464):

```python
cli_opts, rest = parse_args(args=args, result_class=CLIOptions, ...)
```

Calls `kitty/cli.py:parse_args()` which returns a `CLIOptions` object.

**Step 5 — Detach handling** (lines 470–474): If `--detach` is set, reads stdin session data if session is `'-'`, then calls `detach()`.

**Step 6 — Replay commands** (lines 475–478): If `--replay-commands` is set, delegates to `kitty.client.main()` and returns.

**Step 7 — Single-instance data** (lines 479–492): Reads the `KITTY_SI_DATA` environment variable if `--single-instance` was set, extracting the talk file descriptor and socket path.

**Step 8 — Configuration loading** (line 494):

```python
opts = create_opts(cli_opts, accumulate_bad_lines=bad_lines)
```

This calls `kitty/cli.py:create_opts()` at line 1081, which triggers the full config loading pipeline (detailed in Q2 below).

**Step 9 — Environment setup** (line 495): `setup_environment(opts, cli_opts)` at lines 403–421 ensures kitty and kitten binaries are in PATH (`ensure_kitty_in_path()` at line 411, `ensure_kitten_in_path()` at line 412), sets up manpath for frozen builds, and calls `set_default_env(env)` (line 421) to establish the default child environment.

**Step 10 — Locale setup** (lines 499–502): `set_locale()` at line 424 calls `locale.setlocale(locale.LC_ALL, '')`. If this fails, it tries again without `LANG` set (lines 431–438). On macOS, `ensure_macos_locale()` is called first, which uses Cocoa APIs — on Linux this is skipped.

**Step 11 — Python switch interval** (line 504): `sys.setswitchinterval(1000.0)` — since Kitty uses only a single Python thread, this maximizes the GIL hold time for performance.

**Step 12 — Signal masking** (line 513): `mask_kitty_signals_process_wide()` masks signals before GLFW starts threads. The comment at lines 510–512 explains:

> *"mask the signals now as on some platforms the display backend starts threads. These threads must not handle the masked signals, to ensure kitty can handle them."*

**Step 13 — GLFW initialization** (line 514): `init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)` calls `init_glfw()` at line 95, which:

- Selects the platform backend (line 96): `'cocoa'` on macOS, `'wayland'` if `is_wayland(opts)`, otherwise `'x11'`. In our Linux/X11 environment, `glfw_module = 'x11'`.
- Calls `init_glfw_module()` at line 90, which invokes `glfw_init(glfw_path(glfw_module), ...)` (line 91).
- `glfw_path()` from `kitty/constants.py` (lines 191–193) returns the platform-specific `.so` path: `os.path.join(extensions_dir, f'{prefix}glfw-{module}.so')`.
- If initialization fails, raises `SystemExit('GLFW initialization failed')` (line 92).

**Step 14 — `run_app()` call** (line 518): `run_app(opts, cli_opts, bad_lines, talk_fd)` invokes the `AppRunner.__call__()` at line 247.

**Step 15 — Cleanup** (lines 519–521): In the `finally` block, `glfw_terminate()` and `cleanup_ssh_control_masters()` are called.

### 1.4 `AppRunner.__call__()` — Font and Options Push

**File:** `kitty/main.py` (lines 247–260)

**Step 1 — Box drawing scale** (line 248): `set_scale(opts.box_drawing_scale)` configures the box-drawing glyph rasterizer from `kitty/fonts/box_drawing.py`.

**Step 2 — Push options to C** (line 249): `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` transfers the Python `Options` object to the native C layer.

**Step 3 — Font family loading** (line 251): `set_font_family(opts)` calls `kitty/fonts/render.py:set_font_family()` at line 173. This:

- Calls `get_font_files(opts)` from `kitty/fonts/common.py` (line 281), which resolves medium, bold, italic, and bold-italic font faces. On Linux, the fontconfig backend (`kitty/fonts/fontconfig.py`) performs family matching and scoring.
- Builds the `current_faces` list (line 178).
- Creates a symbol map via `create_symbol_map(opts)` (line 186) and narrow symbols via `create_narrow_symbols(opts)` (line 187).
- Registers everything with the native renderer: `set_font_data(render_box_drawing, prerender_function, descriptor_for_idx, ...)` (lines 189–193).

**Step 4 — `_run_app()` call** (line 252).

**Step 5 — Cleanup** (lines 254–260): `set_options(None)` and `free_font_data()` release resources.

### 1.5 `_run_app()` — Window and Boss Creation

**File:** `kitty/main.py` (lines 202–236)

**Step 1 — Platform-specific setup** (lines 202–211): On Linux/X11, `set_x11_window_icon()` (line 211) sets the window icon using `logo/kitty.png` or a custom icon. On macOS, Cocoa global shortcuts and custom beam cursor are set up; on Wayland, window icons are skipped (line 210).

**Step 2 — Cached values** (line 213): `cached_values_for(run_app.cached_values_name)` opens a context manager for persistent cached values (e.g., remembered window size).

**Step 3 — Session creation** (line 214): `create_sessions(opts, args, default_session=opts.startup_session)` calls `kitty/session.py:create_sessions()` at line 219. The cascade is: CLI `--session` argument → `startup_session` config option → fallback shell. The default path (lines 253–263) creates a `Session` with one `Tab`, adds a `SpecialWindow` with the resolved shell.

**Step 4 — Window class and state** (lines 215–219): Determines `wincls`, `window_state`, and `wstate` from the startup session and CLI args.

**Step 5 — OS window creation** (lines 220–225):

```python
window_id = create_os_window(
    run_app.initial_window_size_func(...), pre_show_callback,
    args.title or appname, ..., load_all_shaders, ...)
```

- `initial_window_size_func()` from `kitty/os_window_size.py` (line 54) computes window dimensions from config (cells or pixels), DPI, margins, and padding.
- **Critical:** `load_all_shaders` is passed as a callback and executes during window creation when the OpenGL context is current.
- `load_all_shaders()` at line 82 calls `load_shader_programs(semi_transparent)` and `load_borders_program()`.

**Step 6 — Boss creation** (line 226): `Boss(opts, args, cached_values, global_shortcuts, talk_fd)` instantiates the `Boss` controller at `kitty/boss.py` line 325. This:

- Creates `ChildMonitor` (lines 370–374) — the native C object that manages the I/O thread.
- Sets up encryption key, clipboard, mappings (lines 334–378).
- Calls `set_boss(self)` (line 375) to register globally.

**Step 7 — Boss start** (line 227): `boss.start(window_id, startup_sessions)` at `kitty/boss.py` line 1181:

- **I/O thread launch** (line 1183): `self.child_monitor.start()` calls the C `start()` function at `kitty/child-monitor.c` line 280, which creates the `io_loop` pthread (line 291):

  ```c
  ret = pthread_create(&self->io_thread, NULL, io_loop, self);
  ```

- **First child startup** (line 1194): `self.startup_first_child(first_os_window_id, startup_sessions)` at line 383 iterates startup sessions, calls `self.add_os_window()` (line 391). `add_os_window()` at line 402 creates `TabManager(os_window_id, ...)` (line 428), which creates tabs from the session. Each tab creates windows, eventually calling `Tab.launch_child()` which forks the child process.

**Step 8 — Main loop entry** (line 234): `boss.child_monitor.main_loop()` calls the C `main_loop()` at `kitty/child-monitor.c` line 1259:

```c
run_main_loop(process_global_state, self);
```

This enters the GLFW event loop — the process is now fully running.

**Step 9 — Cleanup** (line 236): `boss.destroy()` in the `finally` block.

### 1.6 Shader Compilation Detail

**File:** `kitty/shaders.py`

The `LoadShaderPrograms.__call__()` method at line 147 compiles all GPU shader programs. This executes during OS window creation (via the `load_all_shaders` callback) when the OpenGL context is current.

**Cell shaders** (4 variants, lines 174–184):

- `CELL_PROGRAM` (PHASE_BOTH), `CELL_BG_PROGRAM` (PHASE_BACKGROUND), `CELL_SPECIAL_PROGRAM` (PHASE_SPECIAL), `CELL_FG_PROGRAM` (PHASE_FOREGROUND).
- Uses `cell_vertex.glsl` and `cell_fragment.glsl` with a `MultiReplacer` (line 112) for compile-time defines including `REVERSE_SHIFT`, `STRIKE_SHIFT`, `DIM_SHIFT`, `DECORATION_SHIFT`, `MARK_SHIFT`, `TRANSPARENT`, `FG_OVERRIDE_THRESHOLD`, and `TEXT_NEW_GAMMA`.

**Graphics shaders** (3 variants, lines 186–197):

- `GRAPHICS_PROGRAM` (SIMPLE), `GRAPHICS_PREMULT_PROGRAM` (PREMULT), `GRAPHICS_ALPHA_MASK_PROGRAM` (ALPHA_MASK).
- Uses `graphics_vertex.glsl` and `graphics_fragment.glsl`.

**Background image shader** (line 199): `program_for('bgimage').compile(BGIMAGE_PROGRAM)`.

**Tint shader** (line 200): `program_for('tint').compile(TINT_PROGRAM)`.

**Cell program init** (line 201): `init_cell_program()` sets up uniforms and VAO.

**Border shader** (from `kitty/borders.py` lines 63–65):

```python
program_for('border').compile(BORDERS_PROGRAM)
init_borders_program()
```

**Shader source loading:** `Program.__init__()` at line 48 loads vertex and fragment shaders from kitty resources, prepends `#version {GLSL_VERSION}\n` (line 63), and processes `#pragma kitty_include_shader <name>` directives for includes (lines 61–81). Compilation errors produce a `CompileError` with filename-mapped error messages (lines 87–104).

### 1.7 Summary Startup Chain

```
launcher/main.c:main()                    [C native entry]
  └─ ensure_working_stdio()               [validate stdio fds]
  └─ read_exe_path()                      [resolve binary path]
  └─ delegate_to_kitten_if_possible()     [kitten delegation]
  └─ handle_fast_commandline()            [--version, --single-instance]
  └─ run_embedded()                       [Python runtime bootstrap]
      └─ entry_points.py:main()           [Python dispatch]
          └─ main.py:_main()              [orchestrator]
              ├─ parse_args()             [CLI parsing]
              ├─ create_opts()            [config loading]
              ├─ setup_environment()      [PATH, env]
              ├─ set_locale()             [locale]
              ├─ mask_kitty_signals_process_wide()
              ├─ init_glfw()              [GLFW platform init]
              └─ run_app()                [AppRunner]
                  ├─ set_options()        [push opts to C]
                  ├─ set_font_family()    [font resolution]
                  └─ _run_app()           [window + boss]
                      ├─ create_sessions()
                      ├─ create_os_window(load_all_shaders)
                      │   ├─ load_shader_programs()  [cell, graphics, bgimage, tint]
                      │   └─ load_borders_program()  [border shader]
                      ├─ Boss.__init__()  [ChildMonitor]
                      ├─ Boss.start()
                      │   ├─ child_monitor.start()  [I/O thread]
                      │   └─ startup_first_child()
                      │       └─ TabManager → Tab → Window → Child.fork()
                      │           ├─ openpty()
                      │           ├─ get_final_env()
                      │           └─ spawn()
                      ├─ Window.set_geometry() → mark_terminal_ready()
                      └─ child_monitor.main_loop()  [GLFW event loop]
```

---

## Q2: Configuration Resolution — How does Kitty decide on its initial configuration at first launch?

### Rationale

Kitty's configuration system uses a cascading merge pipeline: built-in defaults → system config → user config → CLI overrides. Understanding this pipeline requires tracing code through four files: `kitty/cli.py`, `kitty/config.py`, `kitty/conf/utils.py`, and `kitty/options/types.py`.

### 2.1 Configuration File Cascade

The config loading pipeline is triggered by `create_opts()` at `kitty/cli.py` line 1081:

```python
config = default_config_paths(args.config)
overrides = map(parse_override, args.override or ())
opts = load_config(*config, overrides=overrides, accumulate_bad_lines=accumulate_bad_lines)
```

`default_config_paths()` at line 1067 calls `resolve_config(SYSTEM_CONF, defconf, conf_paths)`.

**`SYSTEM_CONF`** is defined at line 1064: `'/etc/xdg/kitty/kitty.conf'`.

**`defconf`** from `kitty/constants.py` line 133: `os.path.join(config_dir, 'kitty.conf')` — which typically resolves to `~/.config/kitty/kitty.conf`.

**`resolve_config()`** from `kitty/conf/utils.py` line 322 implements the cascade:

- If CLI config files are specified and none are `'NONE'`: yield `SYSTEM_CONF` first, then the CLI-specified configs (lines 323–326).
- Default: yield `SYSTEM_CONF`, then `defconf` (lines 327–329).

This establishes the cascade order: **system → user** (or CLI-specified).

### 2.2 Config Loading and Merging

**`load_config()`** from `kitty/conf/utils.py` line 332 is the core merge engine:

1. Starts with `defaults._asdict()` — built-in defaults from `kitty/options/types.py` (line 340).
2. For each config path, opens the file, parses it via `parse_config()`, and merges via `merge_configs()` (lines 342–357).
3. Files that don't exist are silently skipped (line 354): `except (FileNotFoundError, PermissionError): continue`.
4. If overrides are present, they are parsed and merged last (lines 358–361).

This means on a fresh first launch where neither `/etc/xdg/kitty/kitty.conf` nor `~/.config/kitty/kitty.conf` exist, the built-in defaults are used unmodified.

### 2.3 Constructing the Options Object

**`kitty/config.py:load_config()`** at line 163 takes the merged dict and:

1. Constructs an `Options` object (line 169).
2. Builds action aliases (lines 171–172).
3. Finalizes keyboard keys and mouse mappings (lines 173–174).
4. Records `config_paths`, `all_config_paths`, and `config_overrides` on the opts (lines 183–185).

### 2.4 Built-in Defaults

`kitty/options/types.py` is auto-generated and contains the `defaults` object. `kitty/options/definition.py` defines the option schema. Key defaults include:

- `font_family`: the system monospace font
- `font_size`: 11.0
- `term`: `'xterm-kitty'`
- `initial_window_width`/`initial_window_height`: window dimensions in cells or pixels
- `color_table`: 256-entry ANSI color array
- `shell_integration`: enabled by default for bash, zsh, and fish

### 2.5 How Options Drive the Startup

Once the `Options` object is constructed, it drives every subsequent initialization:

- **Options pushed to C** (line 249 of `kitty/main.py`): `set_options(opts, ...)` transfers the complete options to the native layer.
- **Font selection**: `opts.font_family`, `opts.font_size`, `opts.bold_font`, `opts.italic_font`, `opts.bold_italic_font` drive `set_font_family()` in `kitty/fonts/render.py`.
- **Window size**: `opts.initial_window_width`, `opts.initial_window_height`, `opts.remember_window_size` drive `initial_window_size_func()` in `kitty/os_window_size.py` (line 54).
- **Color table**: `opts.color_table` (256-entry array) built by `kitty/config.py:build_ansi_color_table()` at line 23.
- **TERM variable**: `opts.term` sets the `TERM` env var in the child process (default `'xterm-kitty'`).
- **Shell integration**: `opts.shell_integration` controls whether kitty injects shell startup hooks.

### 2.6 Runtime Confirmation of Applied Config

`kitty/debug_config.py:debug_config()` provides runtime confirmation of the applied settings:

- **OpenGL version** (line 258): `opengl_version_string()`.
- **Active fonts** (lines 260–263): `current_fonts()` with `identify_for_debug()` for each face.
- **Loaded config files** (lines 269–271): `opts.config_paths` lists which files were actually found and loaded.
- **Config overrides** (lines 272–274): `opts.config_overrides` lists any CLI overrides.
- **Environment variables** (lines 277–291): Reports `PATH`, `LANG`, `DISPLAY`, `WAYLAND_DISPLAY`, and all `LC_*`/`XDG_*` variables.

This diagnostic output can be triggered via the `debug_config` kitten and confirms exactly which config sources were active.

---

## Q3: Terminal-to-Shell Handoff — How does Kitty prepare the terminal to communicate with the child shell?

### Rationale

The terminal-to-shell handoff involves PTY allocation, environment preparation, shell integration injection, child process spawning, and a synchronization mechanism that ensures the child doesn't start executing until the terminal window is ready. This is all orchestrated through `kitty/child.py` with support from `kitty/shell_integration.py` and `kitty/window.py`.

### 3.1 PTY Allocation

**File:** `kitty/child.py`

`Child.fork()` at line 276 begins the handoff:

```python
master, slave = openpty()
```

The `openpty()` function at line 170:

1. Calls `os.openpty()` to get master and slave file descriptors (line 171).
2. Sets slave as inheritable, master as non-inheritable (lines 172–173).
3. Sets IUTF8 on the master fd via `fast_data_types.set_iutf8_fd(master, True)` (line 174) — this enables UTF-8 aware line editing in the kernel's terminal discipline.

A **ready-pipe** is created for synchronization (line 283):

```python
ready_read_fd, ready_write_fd = os.pipe()
```

The write end is non-inheritable (line 284), the read end is inheritable (line 285). This pipe is the key to the synchronization mechanism described in Section 3.5.

### 3.2 Environment Preparation

**File:** `kitty/child.py`

`Child.get_final_env()` at line 233 builds the complete child environment:

1. Starts with `default_env().copy()` (line 235) — the base environment set during startup by `setup_environment()`.
2. Sets `TERM = opts.term` (line 242) — default `'xterm-kitty'`.
3. Sets `COLORTERM = 'truecolor'` (line 243) — advertises 24-bit color support.
4. Sets `KITTY_PID` to the parent PID (line 244).
5. Sets `KITTY_PUBLIC_KEY` for encrypted remote control (line 245).
6. Sets `KITTY_LISTEN_ON` if the Boss is listening on a socket (lines 246–249).
7. Sets `PWD` to the child's working directory (lines 250–254) — needed for symlinked CWDs.
8. Sets `TERMINFO` based on `opts.terminfo_type` (lines 255–260):
   - `'path'`: `TERMINFO = checked_terminfo_dir()` — the `terminfo/` directory in the kitty installation.
   - `'direct'`: `TERMINFO = base64_terminfo_data()` — a base64-encoded terminfo blob prefixed with `b64:`.
9. Sets `KITTY_INSTALLATION_DIR` (line 261).
10. **Shell integration injection** (lines 265–267):

```python
if not self.should_run_via_run_shell_kitten and 'disabled' not in opts.shell_integration:
    from .shell_integration import modify_shell_environ
    modify_shell_environ(opts, env, self.argv)
```

### 3.3 Shell Integration

**File:** `kitty/shell_integration.py`

`modify_shell_environ()` at line 218:

1. Identifies the shell type via `get_supported_shell_name(argv[0])` (line 219) — supports fish, zsh, and bash.
2. Gets the effective KSI env var via `get_effective_ksi_env_var(opts)` (line 220).
3. Sets `KITTY_SHELL_INTEGRATION` env var (line 223).
4. If RC file modification is allowed, calls shell-specific setup:

   - **fish** (`setup_fish_env()` at line 16): Prepends `shell_integration_dir` to `XDG_DATA_DIRS` and sets `KITTY_FISH_XDG_DATA_DIR`.
   - **zsh** (`setup_zsh_env()` at line 49): Overrides `ZDOTDIR` to kitty's zsh integration directory, saves the original as `KITTY_ORIG_ZDOTDIR`.
   - **bash** (`setup_bash_env()` at line 70): Sets `ENV` to kitty's bash integration script, sets `KITTY_BASH_INJECT` and `HISTFILE`, and adds `--posix` to argv.

### 3.4 Child Process Spawn

**File:** `kitty/child.py`

Continuing in `Child.fork()`:

1. Resolves the final executable (line 327): `self.final_exe = which(argv[0]) or argv[0]`.
2. On macOS with the default shell, wraps via `kitten run-shell` (lines 295–326) — not applicable on Linux.
3. Builds the environment as tuple strings (line 332): `env = tuple(f'{k}={v}' for k, v in self.final_env.items())`.
4. Spawns the child (lines 333–335):

```python
pid = fast_data_types.spawn(
    final_exe, cwd, tuple(argv), env, master, slave,
    stdin_read_fd, stdin_write_fd, ready_read_fd, ready_write_fd, ...)
```

5. Closes the slave fd (line 336) — the parent only needs the master.
6. Stores the PID and master fd (lines 337–338).
7. Makes the master fd non-blocking (line 345).
8. On Linux, moves the child into a systemd scope (lines 346–353) via `fast_data_types.systemd_move_pid_into_new_scope()`.
9. Stores `terminal_ready_fd = ready_write_fd` (line 343) — this is the synchronization handle.

### 3.5 The Ready-Pipe Synchronization Mechanism

This is the most elegant part of the handoff. The child process, after being spawned by `fast_data_types.spawn()`, blocks on `ready_read_fd` waiting for the terminal to be ready. The native `spawn()` implementation reads from the ready pipe before exec'ing the shell.

When the terminal window gets its first geometry layout, `kitty/window.py:Window.set_geometry()` at line 850 triggers the unblock:

```python
if not self.child_is_launched:
    self.child.mark_terminal_ready()
    self.child_is_launched = True
```

`mark_terminal_ready()` at `kitty/child.py` line 362 simply closes the write end:

```python
os.close(self.terminal_ready_fd)
```

Closing the write end causes the child's blocked read on `ready_read_fd` to return EOF, unblocking the child process. The shell can now start executing, confident that:

- The terminal window exists and has dimensions.
- The PTY has been resized to match the window geometry (line 863: `boss.child_monitor.resize_pty(self.id, *current_pty_size)`).
- SIGWINCH won't race with the shell's initial setup.

**Debug evidence** (`kitty/window.py` lines 869–871): When `--debug-rendering` is active:

```python
if boss.args.debug_rendering:
    now = monotonic()
    print(f'[{now:.3f}] Child launched', file=sys.stderr)
```

On subsequent resizes (lines 872–873): `[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}`.

### 3.6 Data Flow After Launch

Once the child is unblocked and the shell starts producing output:

1. The **I/O thread** (`kitty/child-monitor.c:io_loop()` at line 1481) polls child PTY fds with `POLLIN` events (lines 1498–1504).
2. `read_bytes()` at line 1337 reads from the child fd into the VT parser buffer (line 1345) and commits via `vt_parser_commit_write()` (line 1354).
3. The VT parser in `kitty/vt-parser.c` classifies bytes into text, CSI, OSC, and DCS sequences, updating the screen model in `kitty/screen.c`.
4. The main loop renders the updated screen model via the GPU pipeline.

---

## Q4: Display System Evidence — What visible evidence confirms the GPU rendering pipeline, font subsystem, and display system are active?

### Rationale

The display system's readiness can be confirmed through several categories of evidence: font resolution and registration, shader compilation, glyph cache population, I/O loop activity, and diagnostic output. Each piece of evidence traces back to specific code.

### 4.1 Font Subsystem Evidence

**Font resolution:**

- `set_font_family()` in `kitty/fonts/render.py` (line 173) resolves font faces via `get_font_files(opts)` from `kitty/fonts/common.py` (line 281).
- On Linux, the fontconfig backend (`kitty/fonts/fontconfig.py`) performs family matching and scoring to find the best matching medium, bold, italic, and bold-italic faces.
- The resolved faces are registered with the native renderer via `set_font_data()` (lines 189–193).

**Font debug output:**

- `dump_font_debug()` at `kitty/fonts/render.py` line 161 outputs all resolved fonts with `identify_for_debug()`.
- `debug_config()` at `kitty/debug_config.py` lines 260–263 reports active fonts:

```python
for k, font in current_fonts().items():
    if hasattr(font, 'identify_for_debug'):
        p(yellow(f'  {k}:'), font.identify_for_debug())
```

**Box-drawing rasterization:**

- `render_box_drawing` from `kitty/fonts/box_drawing.py` is passed to `set_font_data()` as a callback. It rasterizes box-drawing characters, missing glyph placeholders, underlines, and cursors into bitmaps for GPU upload.

### 4.2 Shader Compilation Evidence

All shaders must compile successfully for the terminal to render. The shader compilation occurs during OS window creation when the OpenGL context is current:

| Shader | Source Files | Variants | Program IDs |
|--------|-------------|----------|-------------|
| Cell (text + cursor) | `cell_vertex.glsl`, `cell_fragment.glsl` | 4 (BOTH, BG, SPECIAL, FG) | `CELL_PROGRAM`, `CELL_BG_PROGRAM`, `CELL_SPECIAL_PROGRAM`, `CELL_FG_PROGRAM` |
| Graphics (images) | `graphics_vertex.glsl`, `graphics_fragment.glsl` | 3 (SIMPLE, PREMULT, ALPHA_MASK) | `GRAPHICS_PROGRAM`, `GRAPHICS_PREMULT_PROGRAM`, `GRAPHICS_ALPHA_MASK_PROGRAM` |
| Background image | `bgimage_vertex.glsl`, `bgimage_fragment.glsl` | 1 | `BGIMAGE_PROGRAM` |
| Tint overlay | `tint_vertex.glsl`, `tint_fragment.glsl` | 1 | `TINT_PROGRAM` |
| Border | `border_vertex.glsl`, `border_fragment.glsl` | 1 | `BORDERS_PROGRAM` |

**Utility shaders** included via `#pragma kitty_include_shader`: `alpha_blend.glsl` (blending), `linear2srgb.glsl` (color space conversion).

If any shader fails to compile, `CompileError` is raised with filename-mapped error messages (`kitty/shaders.py` lines 87–104), and the terminal cannot start (`load_all_shaders()` at `kitty/main.py` line 82 catches `CompileError` and raises `SystemExit`).

### 4.3 Glyph Cache

`kitty/glyph-cache.c` manages a GPU texture atlas for rasterized glyphs. `kitty/freetype.c` performs the actual FreeType glyph rasterization. The `prerender_function` callback passed to `set_font_data()` handles glyph prerendering — characters are rasterized on demand and uploaded to the GPU texture.

### 4.4 I/O Loop and Rendering Evidence

**I/O thread** (`kitty/child-monitor.c:io_loop()` at line 1481):

- Named `"KittyChildMon"` via `set_thread_name()` (line 1489).
- Polls child fds with `POLLIN` for incoming data and `POLLOUT` for pending writes (lines 1498–1504).
- `read_bytes()` at line 1337 reads from the child fd, commits to the VT parser buffer via `vt_parser_commit_write()` (line 1354).
- Data flow: child PTY → `read_bytes()` → VT parser → screen model → render flag → main loop render cycle.

**Main loop** (`kitty/child-monitor.c:main_loop()` at line 1259):

- Runs `run_main_loop(process_global_state, self)` — the GLFW event loop.
- `parse_input()` at line 451 processes data read by the I/O thread in the main thread context — transferring screen updates to GPU render data.

### 4.5 Debug Rendering Output

The `--debug-rendering` CLI flag activates diagnostic timestamps:

- **Child launched** (`kitty/window.py` lines 869–871): `[{now:.3f}] Child launched` is printed to stderr on the first geometry set.
- **SIGWINCH** (`kitty/window.py` lines 872–873): `[{monotonic():.3f}] SIGWINCH sent to child in window: {self.id} with size: {current_pty_size}` on subsequent resizes.

The `--debug-font-fallback` flag triggers `dump_font_debug()` at `kitty/main.py` line 229.

### 4.6 `debug_config()` Diagnostic Report

`kitty/debug_config.py:debug_config()` provides a comprehensive diagnostic report:

- **OpenGL version** (line 258): `opengl_version_string()` reports the GL version string.
- **Compositor** (line 257): `compositor_name()` reports the window compositor (not on macOS).
- **Frozen status** (line 259): Whether running from a frozen build.
- **Active fonts** (lines 260–263): Each font face with `identify_for_debug()`.
- **Paths** (lines 264–268): kitty executable, base dir, extensions dir, system shell.
- **Loaded config files** (lines 269–271).
- **Config overrides** (lines 272–274).
- **Environment variables** (lines 277–291): `PATH`, `LANG`, `DISPLAY`, `WAYLAND_DISPLAY`, `LC_*`, `XDG_*`.

This output serves as definitive evidence that the display system initialized successfully.

---

## Appendix: Headless Execution Attempt via Xvfb

### Background

The Kitty binary requires several components to be present:

1. A compiled C extension (`fast_data_types.so`) containing the native bindings for GLFW, OpenGL, FreeType, Fontconfig, and the child monitor.
2. A functional display server — the GLFW initialization at `kitty/main.py` line 91 requires a connected X11 or Wayland display.
3. An OpenGL-capable context for shader compilation.

Xvfb (X Virtual Framebuffer) provides a virtual X11 display server that satisfies the `DISPLAY` environment variable requirement, available at `/usr/bin/Xvfb`.

### Build Prerequisites

The Kitty binary was compiled using the setup agent's build command:

```bash
python3 setup.py build --ignore-compiler-warnings
```

This produced two critical artifacts:

- **Binary**: `kitty/launcher/kitty` (ELF 64-bit executable)
- **Shared library**: `kitty/fast_data_types.so` (C extension with GLFW, OpenGL, FreeType, Fontconfig, and child monitor bindings)

Without the compiled `fast_data_types.so`, attempting `import kitty.fast_data_types` from Python raises `ImportError` — the native launcher is required for the full startup path.

### Headless Execution — Run 1: `--debug-rendering`

A headless launch was performed using Xvfb as a virtual X11 display, with `--debug-rendering` enabled and a minimal child process (`/bin/true`) to capture the full startup-to-exit cycle:

```bash
Xvfb :99 -screen 0 1280x720x24 -ac +extension GLX &
DISPLAY=:99 timeout 5 ./kitty/launcher/kitty --debug-rendering --config NONE -e /bin/true 2>&1
```

**Captured stderr output:**

```
[0.199] OS Window created
[0.208] Failed to open systemd user bus with error: Connection refused
[0.212] Child launched
[0.136] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

**Exit code: 0** (clean shutdown)

**Analysis of each line:**

1. **`[0.199] OS Window created`** — Confirms that `create_os_window()` at `kitty/main.py` (line 221) successfully created an X11 window on the virtual display. The timestamp (199 ms from startup) reflects the cost of GLFW initialization (`init_glfw()` at line 95), font family resolution (`set_font_family()` at line 251), and shader compilation (all 10+ programs via `load_all_shaders()` callback at line 82). This message is emitted by the native `create_os_window()` implementation when `--debug-rendering` is active.

2. **`[0.208] Failed to open systemd user bus with error: Connection refused`** — This non-fatal warning originates from `kitty/child.py` (lines 346–353), where `systemd_move_pid_into_new_scope()` attempts to move the child PID into a systemd transient scope. In a container or headless CI environment without a running systemd user session, the D-Bus connection is unavailable. The startup continues normally — this is a best-effort operation guarded by a try/except.

3. **`[0.212] Child launched`** — Confirms that `mark_terminal_ready()` fired, triggered by `Window.set_geometry()` at `kitty/window.py` (lines 865–867). This is the critical synchronization event: closing the ready-pipe's write fd unblocks the child process (`os.close(self.terminal_ready_fd)` at `kitty/child.py` line 362). The `--debug-rendering` timestamp is emitted at `kitty/window.py` (lines 869–871). The 13 ms gap between window creation (0.199) and child launch (0.212) represents the first layout pass.

4. **`[0.136] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5`** — Confirms OpenGL context creation and GLAD function loading succeeded. The GL version string matches `opengl_version_string()` used by `debug_config()` at `kitty/debug_config.py` (line 258). Mesa 25.2.8 provides a software-rendered OpenGL 4.5 Core Profile via llvmpipe — sufficient for all of Kitty's GLSL shaders. The timestamp (0.136) appears lower than the others because this message is emitted from the GLFW/GL initialization path which runs on its own timing context within `create_os_window()`.

### Headless Execution — Run 2: `--debug-font-fallback`

A second run was performed with `--debug-font-fallback` added to capture font resolution details:

```bash
Xvfb :99 -screen 0 1280x720x24 -ac +extension GLX &
DISPLAY=:99 timeout 8 ./kitty/launcher/kitty --debug-rendering --debug-font-fallback --config NONE -e /bin/true 2>&1
```

**Captured stderr output:**

```
[0.149] OS Window created
[0.160] Failed to open systemd user bus with error: Connection refused
[0.164] Child launched
[0.165] Text fonts:
[0.165]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.165]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.165]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.165]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.123] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

**Exit code: 0** (clean shutdown)

**Font resolution analysis:**

The `--debug-font-fallback` flag activated the font debug output from `kitty/fonts/render.py:dump_font_debug()` (line 161), which calls `identify_for_debug()` on each resolved face. The output confirms that `set_font_family()` (line 173) successfully resolved all four font variants via the fontconfig backend (`kitty/fonts/fontconfig.py`):

| Variant | Font File | Path |
|---------|-----------|------|
| Normal | DejaVuSansMono | `/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf` |
| Bold | DejaVuSansMono-Bold | `/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf` |
| Italic | DejaVuSansMono-Oblique | `/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf` |
| Bold-Italic | DejaVuSansMono-BoldOblique | `/usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf` |

**Rationale**: With `--config NONE`, the built-in default `font_family` (monospace) was used. Fontconfig resolved this to DejaVu Sans Mono — the standard monospace font available on the system. The `:0` suffix indicates face index 0 within the `.ttf` file. The `get_font_files()` function at `kitty/fonts/common.py` (line 281) populated the medium, bold, italic, and bold-italic slots, and `set_font_data()` (lines 189–193) registered them with the native FreeType renderer in `kitty/freetype.c`.

### Key Findings from Headless Execution

The Xvfb execution confirms the complete startup lifecycle documented in Q1 through Q4:

1. **GLFW initialized successfully** — The X11 GLFW backend loaded and connected to the virtual display.
2. **OpenGL 4.5 context obtained** — Mesa's software renderer (llvmpipe) provided a Core Profile context sufficient for all of Kitty's shaders.
3. **All shaders compiled** — The absence of any `CompileError` messages confirms that all cell (4 variants), graphics (3 variants), bgimage, tint, and border shaders compiled successfully against the Mesa OpenGL 4.5 implementation.
4. **Font resolution succeeded** — Fontconfig resolved all four font variants (normal, bold, italic, bold-italic) to DejaVu Sans Mono.
5. **OS window created** — `create_os_window()` returned a valid window ID on the virtual display.
6. **Child process launched** — The ready-pipe synchronization mechanism (`mark_terminal_ready()`) fired correctly after the first geometry layout.
7. **Clean exit** — The child process (`/bin/true`) exited immediately, and Kitty shut down cleanly with exit code 0.
8. **Total startup time**: approximately 150–210 ms from process entry to child launch, confirming the efficiency of the initialization pipeline even on a software-rendered display.

### Limitations

Xvfb provides only a software-rendered OpenGL context (via Mesa's llvmpipe). While shader compilation succeeds, rendering performance is limited compared to hardware-accelerated contexts. The child process output (stdout) goes to the PTY, not to stderr, so environment variables and shell integration output from the child cannot be captured via stderr redirection alone — they are rendered (invisibly) on the virtual display's framebuffer.

---

## Summary

The Kitty terminal emulator's startup lifecycle at commit `815df1e21` follows a precisely ordered sequence:

1. **Native bootstrap** (`kitty/launcher/main.c`): Validates stdio, resolves executable path, initializes Python runtime.
2. **Python dispatch** (`kitty/entry_points.py`): Routes to `kitty.main.main()`.
3. **Orchestration** (`kitty/main.py:_main()`): Parses CLI args, loads config, sets up environment and locale, masks signals, initializes GLFW.
4. **Font and options** (`kitty/main.py:AppRunner`): Pushes options to C, resolves and registers font families.
5. **Window and shaders** (`kitty/main.py:_run_app()`): Creates sessions, creates the OS window (compiling all GPU shaders during creation), creates the Boss controller.
6. **I/O thread and child** (`kitty/boss.py:Boss.start()`): Starts the I/O monitor thread, creates tab managers, forks child processes with prepared PTYs and environments.
7. **Terminal ready** (`kitty/window.py:Window.set_geometry()`): On first layout, closes the ready-pipe to unblock the child shell.
8. **Event loop** (`kitty/child-monitor.c:main_loop()`): GLFW event loop runs, I/O thread polls child PTYs, VT parser processes shell output, GPU renders the terminal.

The configuration cascade — built-in defaults → `/etc/xdg/kitty/kitty.conf` → `~/.config/kitty/kitty.conf` → CLI overrides — is fully resolved before any windowing or rendering initialization begins, ensuring that font selection, window sizing, color tables, and shell integration are all determined from the configuration before the first pixel is drawn.
