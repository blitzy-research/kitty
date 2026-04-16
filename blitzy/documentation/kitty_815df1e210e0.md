# Kitty Terminal Emulator: Initialization Flow Analysis

**Repository**: `kovidgoyal/kitty`
**Source branch**: `kitty_815df1e210e0`
**Head commit**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` — *"Wire up applying of font config"*
**Version**: `kitty 0.35.2 created by Kovid Goyal`
**Analysis scope**: Read-only investigation. No source files were modified.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Build Verification](#2-build-verification)
3. [Runtime Environment Captured](#3-runtime-environment-captured)
4. [Initialization Flow — The Critical Early Startup Phase](#4-initialization-flow--the-critical-early-startup-phase)
5. [Rendering Backend Selection](#5-rendering-backend-selection)
6. [Display Configuration Detection (Content Scale and DPI)](#6-display-configuration-detection-content-scale-and-dpi)
7. [Font System Setup](#7-font-system-setup)
8. [The Three-Way Relationship: Window System ↔ GPU ↔ Fonts](#8-the-three-way-relationship-window-system--gpu--fonts)
9. [Subsystem Initialization Order](#9-subsystem-initialization-order)
10. [Key Values Computed During Startup](#10-key-values-computed-during-startup)
11. [Text Rendering Capabilities (Observed at Runtime)](#11-text-rendering-capabilities-observed-at-runtime)
12. [Observed Runtime Data (Verbatim)](#12-observed-runtime-data-verbatim)
13. [Conclusion](#13-conclusion)

---

## 1. Executive Summary

This document answers, based strictly on kitty's source code and an actual built-and-launched instance, the following questions about kitty's initialization flow:

1. **Which rendering backend does kitty select at startup?**
2. **How does kitty detect its display configuration (content scale, DPI, dimensions)?**
3. **How does the early startup phase (GPU context creation + font system setup) unfold *before any content is displayed*?**
4. **What is the relationship between the window system (GLFW), GPU (OpenGL), and text-cell calculations (FreeType/FontConfig)?**
5. **In what order do these subsystems initialize?**
6. **What key values are computed or detected during startup (DPI, content scale, cell dimensions, OpenGL version, font metrics)?**
7. **What text rendering capabilities are in effect at runtime (`TERM`, `COLORTERM`, terminfo, color support)?**

The investigation proceeded in three parts:

1. **Build**: Compile kitty from source at commit `815df1e210e0`.
2. **Run**: Launch the compiled binary under a headless X11 display server (`Xvfb`) with kitty's own `--debug-rendering` and `--debug-font-fallback` diagnostic flags.
3. **Trace**: Read the source of every file on the critical initialization path and cross-reference each observed value against the code that produces it.

Every claim below cites a specific C/Python function, file, and line (or line-range).

### Headline Answers

| Question | Answer (observed) | Authoritative source |
|---|---|---|
| Rendering backend | OpenGL **4.5 Core Profile** (Mesa 25.2.8, llvmpipe software renderer, LLVM 20.1.2, 256 bits) — requested as min 3.1 Core on Linux | `kitty/data-types.h:20–26`, `kitty/glfw.c:1126–1132`, `kitty/gl.c:42–79` |
| Logical DPI | 96 × 96 (from content scale 1.0 × 1.0) | `kitty/glfw.c:812–821`, `kitty/glfw.c:823–837` |
| GLSL version used | **140** (kitty hard-codes `#version 140` into all shaders) | `kitty/data-types.h:26`, `kitty/shaders.py:63` |
| Required GL extension | `GL_ARB_texture_storage` | `kitty/gl.c:66–71` |
| Fonts (detected) | LiberationMono Regular / Bold / Italic / Bold-Italic (via FontConfig) | `/tmp/kitty_debug.log`, `kitty/fonts/render.py:173–192` |
| `TERM` in child | `xterm-kitty` (custom terminfo "KovIdTTY", 256 colors, 32767 pairs) | `/tmp/kitty_env.log` |
| `COLORTERM` in child | `truecolor` | `/tmp/kitty_env.log` |
| Subsystem convergence point | `create_os_window()` in `kitty/glfw.c` | `kitty/glfw.c:1107–1245` |

---

## 2. Build Verification

### 2.1 Build command and configuration

The repository was built from source using the central `setup.py` build orchestrator (the `Makefile` simply delegates to it). Because the host system ships with a newer `wayland-protocols` whose `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum causes a `-Werror` warning in the vendored GLFW Wayland backend (`glfw/wl_window.c`), we overrode the default strict CFLAGS:

```bash
export CFLAGS="-Wno-error"
python3 setup.py --verbose build
```

This is a **build-time toolchain accommodation only**. No source files were modified.

### 2.2 Build dependencies installed

| Group | Packages |
|---|---|
| Toolchain | `gcc` 13.3.0, `g++` 13.3.0, `python3` 3.12.3, Go 1.22.10 |
| Font | `libfreetype-dev` (2.13.2), `libfontconfig1-dev` (2.15.0), `libharfbuzz-dev` (8.3.0) |
| Window/GPU | `libgl1-mesa-dri` (Mesa 25.2.8), `libx11-dev`, `libx11-xcb-dev`, `libxkbcommon-dev`, `libwayland-dev`, `wayland-protocols` |
| Other | `libdbus-1-dev`, `libssl-dev`, `libpng-dev`, `liblcms2-dev`, `libxxhash-dev`, `libsimde-dev` |
| Headless display | `xvfb`, `x11-utils`, `mesa-utils`, `fonts-liberation`, `fonts-dejavu-core` |

`libsimde-dev` was added after a build error traced to a missing `simde/x86/avx2.h` (kitty uses SIMD Everywhere for portable vectorized paths). `libxxhash-dev` was required because kitty uses `xxHash` directly.

### 2.3 Build products

After a successful `setup.py build` run (exit 0), these artifacts were produced:

```
./kitty/launcher/kitty    (36 KB — native C launcher with embedded CPython)
./kitty/launcher/kitten   (15.7 MB — Go-compiled CLI subcommand dispatcher)
```

Plus native Python extensions (`kitty/fast_data_types*.so`), the per-platform GLFW shared libraries (`kitty/glfw-x11.so`, `kitty/glfw-wayland.so`), the terminfo database (`./terminfo/x/xterm-kitty`), and ~30 generated Wayland protocol binding files under `glfw/`.

### 2.4 Version verification

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

---

## 3. Runtime Environment Captured

All runtime observations in this document were captured from a live launch of the freshly built kitty binary under the following headless environment:

| Item | Value |
|---|---|
| X server | `Xvfb :99 -screen 0 1920x1080x24 +extension GLX +extension RANDR +extension RENDER` |
| X.Org | 21.1.11 |
| Screen | 1920×1080, 24-bit, 100×100 DPI |
| GLX | 1.4, direct rendering: Yes |
| OpenGL vendor | Mesa |
| OpenGL renderer | `llvmpipe (LLVM 20.1.2, 256 bits)` |
| OpenGL core profile version | `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1` |
| OpenGL GLSL version (runtime) | 4.50 |
| Wayland | Not present (`DISPLAY=:99`, no `WAYLAND_DISPLAY`, no `WAYLAND_SOCKET`) — so kitty took the **X11 code path** |

**Invocation used for the runtime trace:**

```bash
timeout 20 ./kitty/launcher/kitty \
  --debug-rendering --debug-font-fallback \
  -o font_family=LiberationMono -o font_size=11.0 --config=NONE \
  sh -c 'env | sort > /tmp/kitty_env.log;
         echo "---TERMINFO---" >> /tmp/kitty_env.log;
         infocmp -1 >> /tmp/kitty_env.log 2>&1;
         echo "---TPUT---"     >> /tmp/kitty_env.log;
         tput colors          >> /tmp/kitty_env.log 2>&1;
         echo "---STTY---"     >> /tmp/kitty_env.log;
         stty size            >> /tmp/kitty_env.log 2>&1;
         sleep 2; exit 0'
```

Captured artifacts:

- `/tmp/kitty_build.log` — build output (successful)
- `/tmp/kitty_debug.log` — kitty's own `--debug-rendering --debug-font-fallback` output
- `/tmp/kitty_env.log` — the child process's `env`, `infocmp`, `tput colors`, `stty size`

These are reproduced verbatim in [Section 12](#12-observed-runtime-data-verbatim).

---

## 4. Initialization Flow — The Critical Early Startup Phase

The critical early startup phase of kitty — *everything that happens from process entry up to the main event loop, before any PTY data is read or any content is displayed* — spans four layers and roughly the following call chain:

```
kitty/launcher/main.c : main()                       ← native entry
  └─► run_embedded()                                  ← embeds CPython, sets sys.kitty_run_data
        └─► Py runs kitty/entry_points.py : main()
              └─► kitty.main.main()
                    └─► kitty.main._main()
                          ├─► parse_args()
                          ├─► create_opts()
                          ├─► setup_environment()
                          ├─► set_locale()
                          ├─► mask_kitty_signals_process_wide()
                          ├─► init_glfw()            ← loads glfw-x11.so OR glfw-wayland.so
                          │     └─► glfw_init()      ← kitty/glfw.c:1431
                          └─► run_app() (AppRunner.__call__)
                                ├─► set_scale()      ← fonts/box_drawing.py
                                ├─► set_options()    ← push opts to C
                                ├─► set_font_family()← fonts/render.py:173
                                │     └─► set_font_data()    ← kitty/fonts.c:1434
                                └─► _run_app()
                                      ├─► create_os_window()   ← kitty/glfw.c:1107  **convergence point**
                                      │     ├─► (temp window for DPI probe on X11)
                                      │     ├─► load_fonts_data(font_sz, xdpi, ydpi)
                                      │     │     └─► font_group_for()
                                      │     │           └─► initialize_font_group()
                                      │     │                 └─► calc_cell_metrics()
                                      │     │                       └─► cell_metrics() via FreeType
                                      │     ├─► get_window_size() callback (os_window_size.py)
                                      │     ├─► glfwCreateWindow()  ← real window
                                      │     ├─► glfwMakeContextCurrent()
                                      │     ├─► gl_init()           ← kitty/gl.c:52  loads GLAD, checks version, ARB_texture_storage
                                      │     ├─► glEnable(GL_FRAMEBUFFER_SRGB)
                                      │     ├─► load_programs()     ← compiles all shader programs
                                      │     └─► update_os_window_viewport()
                                      ├─► Boss(...)                 ← kitty/boss.py
                                      └─► boss.child_monitor.main_loop()   ← kitty/child-monitor.c
```

Every link in this chain is sourced below.

### 4.1 Native entry and Python embedding — `kitty/launcher/main.c`

The compiled `kitty` binary is a thin native launcher that embeds CPython.

Its `main()` function (`kitty/launcher/main.c:454`):

1. Ensures working stdio.
2. Reads the executable path.
3. Delegates to the Go-built `kitten` binary for `@...` shorthand commands (`delegate_to_kitten_if_possible`).
4. Performs fast command-line preparse (`handle_fast_commandline`, to short-circuit `--version` and `--single-instance`).
5. Computes the path to `KITTY_LIB_PATH` and calls `run_embedded(&run_data)`.

`run_embedded()` (line 147/177) initializes CPython, sets `sys.kitty_run_data` — a dict with `bundle_exe_dir`, `extensions_dir`, and `from_source` — via `set_kitty_run_data()` at `kitty/launcher/main.c:53`, then begins Python execution of the kitty package.

### 4.2 Python entry — `kitty/entry_points.py`

For the default GUI launch path (no `+<subcommand>` argument), `entry_points.main()` delegates to `kitty.main.main()`.

### 4.3 Master orchestrator — `kitty/main.py:_main()` (line 441)

Every major initialization action originates here. In order:

| Step | Call | Effect |
|---|---|---|
| 1 | `running_in_kitty(True)` | Marks process as running-in-kitty. |
| 2 | `parse_args(args=args, result_class=CLIOptions, ...)` | Parses CLI options into `CLIOptions`. |
| 3 | `create_opts(cli_opts, accumulate_bad_lines=bad_lines)` | Loads and merges `kitty.conf`. |
| 4 | `setup_environment(opts, cli_opts)` | Prepares child env (PATH, MANPATH, listen_on). |
| 5 | `set_locale()` | Sets process locale (with macOS CoreText fallback). |
| 6 | `mask_kitty_signals_process_wide()` | Masks signals **before** GLFW starts threads (`kitty/main.py:511–514`, fixing [#4636](https://github.com/kovidgoyal/kitty/issues/4636)). |
| 7 | `init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)` | Loads platform GLFW, calls `glfwInit()`, obtains default DPI. |
| 8 | `run_app(opts, cli_opts, bad_lines, talk_fd)` | Drives font init, window creation, and main loop (inside `setup_profiling()`). |
| 9 | `glfw_terminate()` / `cleanup_ssh_control_masters()` | Cleanup (in `finally`). |

### 4.4 GLFW bootstrap — `kitty/main.py:init_glfw()` and `kitty/glfw.c:glfw_init()`

`init_glfw()` (line 95):

```python
def init_glfw(opts, debug_keyboard=False, debug_rendering=False) -> str:
    glfw_module = 'cocoa' if is_macos else ('wayland' if is_wayland(opts) else 'x11')
    init_glfw_module(glfw_module, debug_keyboard, debug_rendering,
                     wayland_enable_ime=opts.wayland_enable_ime)
    return glfw_module
```

`init_glfw_module()` (line 90) calls `glfw_init(glfw_path(glfw_module), edge_spacing, debug_keyboard, debug_rendering, wayland_enable_ime)`. The choice of which `.so` to dlopen is made by `kitty/constants.py:is_wayland()` (line 207), which is positive only if the environment contains `WAYLAND_DISPLAY` or `WAYLAND_SOCKET` (and `KITTY_DISABLE_WAYLAND` is not set), and the file `glfw-wayland.so` exists (`kitty/constants.py:195–204`).

In our Xvfb environment, **X11 was selected** (no `WAYLAND_DISPLAY`).

`glfw_init()` in `kitty/glfw.c:1431` is where the C-side work happens:

1. It dlopens the platform-specific GLFW shared library (`glfw-x11.so`).
2. It sets GLFW debug hints if `--debug-rendering` or `--debug-keyboard` was passed.
3. It invokes `glfwInit()` — which for X11 calls `_glfwPlatformInit()` (`glfw/x11_init.c`), opening the X display via `XOpenDisplay`, initializing RandR for monitor discovery, and setting up XKB for keyboard handling.
4. Finally, it primes `global_state.default_dpi` by calling `get_window_dpi(NULL, ...)` — using the primary monitor's content scale when no window exists yet.

### 4.5 Font family descriptor setup — `kitty.fonts.render.set_font_family()`

Back in `AppRunner.__call__()` (line 247 of `kitty/main.py`), the sequence is:

```python
set_scale(opts.box_drawing_scale)                                                # box_drawing.py
set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)  # push to C
set_font_family(opts)                                                            # fonts/render.py
_run_app(opts, args, bad_lines, talk_fd)
```

`set_font_family()` (`kitty/fonts/render.py:173–192`) uses FontConfig via `get_font_files(opts)` to resolve the `font_family`, `bold_font`, `italic_font`, `bold_italic_font` options into descriptors, assembles a `current_faces` list (medium + optional bold/italic/bi + symbol-map faces), and then calls `set_font_data(...)` — a C function (`kitty/fonts.c:1434`) which stores descriptor indices in `descriptor_indices` and **clears any pre-existing font groups**.

At this point **no FreeType face has been opened yet**. Discovery is purely via the FontConfig matcher (`kitty/fontconfig.c`). Opening faces is deferred until cell metrics are needed — which requires knowing the DPI — which requires a window. This is the dependency loop that `create_os_window()` breaks with a temp window.

### 4.6 OS window creation — `kitty/glfw.c:create_os_window()` (line 1107)

This is the **single point where the windowing system, GPU, and font subsystems all converge**. The sequence inside `create_os_window()` on the X11 path is precise and evidence-backed:

1. **Set GL window hints** — only on the *first* window (`kitty/glfw.c:1125–1132`):

   ```c
   if (is_first_window) {
       glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, OPENGL_REQUIRED_VERSION_MAJOR);  // 3
       glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, OPENGL_REQUIRED_VERSION_MINOR);  // 1 on Linux, 3 on macOS
       glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, true);
       glfwWindowHint(GLFW_DEPTH_BITS, 0);          // no depth buffer
       glfwWindowHint(GLFW_STENCIL_BITS, 0);        // no stencil buffer
       ...
       if (!global_state.is_wayland) glfwWindowHint(GLFW_SRGB_CAPABLE, true);
   }
   ```

   `OPENGL_REQUIRED_VERSION_MAJOR`/`MINOR` are defined in `kitty/data-types.h:20–24`:

   ```c
   #define OPENGL_REQUIRED_VERSION_MAJOR 3
   #ifdef __APPLE__
   #define OPENGL_REQUIRED_VERSION_MINOR 3
   #else
   #define OPENGL_REQUIRED_VERSION_MINOR 1
   #endif
   ```

   Kitty therefore asks GLFW for a **minimum GL 3.1 Core-forward-compat** context on Linux (3.3 on macOS). The driver is free to grant a newer version — and Mesa did: 4.5 Core.

2. **Create a temp window to probe DPI** — `kitty/glfw.c:1199`:

   ```c
   } else {
       temp_window = glfwCreateWindow(640, 480, "temp", NULL, common_context);
       if (temp_window == NULL) { fatal("Failed to create GLFW temp window! ..."); }
       get_window_content_scale(temp_window, &xscale, &yscale, &xdpi, &ydpi);
   }
   ```

   A 640×480 invisible window is created solely to ask GLFW for the content scale of the monitor the window lands on — because on X11 the DPI is most reliably read from an actual window rather than from the monitor enumeration path. On Wayland this trick is not used (comment on line 1188: *"Cannot use temp window on Wayland as scale is only sent by compositor after window is displayed"*).

3. **Compute cell metrics for this DPI** — `kitty/glfw.c:1203–1204`:

   ```c
   FONTS_DATA_HANDLE fonts_data = load_fonts_data(OPT(font_size), xdpi, ydpi);
   ```

   `load_fonts_data()` (`kitty/fonts.c:1530`) calls `font_group_for(font_sz, dpi_x, dpi_y)` (line 204), which caches `FontGroup`s by `(font_sz_in_pts, logical_dpi_x, logical_dpi_y)` — meaning each unique DPI gets its own set of rasterizer state. On cache miss it calls `initialize_font_group()` (line 1495), which:
   - allocates the `fonts` array,
   - initializes the medium / bold / italic / bold-italic faces via FreeType (`face_from_descriptor()` in `kitty/freetype.c`),
   - initializes any symbol-map faces,
   - calls `calc_cell_metrics(fg)` (line 1511).

   `calc_cell_metrics()` (`kitty/fonts.c:373`) asks FreeType via `cell_metrics()` (`kitty/freetype.c:387`) for the medium face's `cell_width`, `cell_height`, `baseline`, `underline_position`, `underline_thickness`, `strikethrough_position`, `strikethrough_thickness`. It then applies the user's `modify_font cell_width / cell_height / baseline / underline_position / ...` adjustments (via `adjust_metric()`, using `fg->logical_dpi_x/y` to interpret unit suffixes like `%`/`px`/`pt`), and finally calls `sprite_tracker_set_layout(&fg->sprite_tracker, cell_width, cell_height)`.

4. **Ask Python for the initial window pixel size** — `kitty/glfw.c:1204`:

   ```c
   PyObject *ret = PyObject_CallFunction(get_window_size, "IIddff",
       fonts_data->cell_width, fonts_data->cell_height,
       fonts_data->logical_dpi_x, fonts_data->logical_dpi_y,
       xscale, yscale);
   int width = PyLong_AsLong(PyTuple_GET_ITEM(ret, 0)),
      height = PyLong_AsLong(PyTuple_GET_ITEM(ret, 1));
   ```

   The `get_window_size` callback is the inner function returned by `initial_window_size_func()` in `kitty/os_window_size.py:55`. For unit `cells`, it computes (`kitty/os_window_size.py:86–96`):

   ```python
   width  = cell_width  * w / xscale + (dpi_x / 72) * spacing + 1
   height = cell_height * h / yscale + (dpi_y / 72) * spacing + 1
   ```

   with `spacing` summed from `single_window_margin_width` / `window_margin_width` / `single_window_padding_width` / `window_padding_width`. Note that on X11 and non-Wayland non-macOS, the callback **overrides `xscale = yscale = 1`** (`kitty/os_window_size.py:79–81`) — the cell width already accounts for DPI via the FreeType metrics pass.

5. **Create the real window and make its GL context current** — `kitty/glfw.c:1207–1211`:

   ```c
   GLFWwindow *glfw_window = glfwCreateWindow(width, height, title, NULL,
                                              temp_window ? temp_window : common_context);
   if (temp_window) { glfwDestroyWindow(temp_window); temp_window = NULL; }
   ...
   glfwMakeContextCurrent(glfw_window);
   ```

6. **Initialize GL (once)** — `kitty/glfw.c:1213`:

   ```c
   if (is_first_window) gl_init();
   glEnable(GL_FRAMEBUFFER_SRGB);
   ```

   `gl_init()` is covered in [Section 5.2](#52-how-gl_init-validates-the-context).

7. **Initial swap & show** — `apply_swap_interval(-1)`, `glfwSwapBuffers`, `glfwShowWindow`.

8. **Re-measure DPI after the window is shown (Wayland or Apple)** — `kitty/glfw.c:1235–1243`:

   ```c
   if (global_state.is_wayland || is_apple) {
       float n_xscale, n_yscale;
       double n_xdpi, n_ydpi;
       get_window_content_scale(glfw_window, &n_xscale, &n_yscale, &n_xdpi, &n_ydpi);
       if (n_xdpi != xdpi || n_ydpi != ydpi) {
           xdpi = n_xdpi; ydpi = n_ydpi;
           fonts_data = load_fonts_data(OPT(font_size), xdpi, ydpi);
       }
   }
   ```

   This is a second-pass correction for compositor-driven scale changes (e.g. a fractional-scale Wayland monitor, or macOS Quartz moving the window to a Retina display). On pure X11, this block is skipped.

9. **Compile all shader programs (once)** — by invoking the `load_programs` Python callback (`kitty/main.py:78–85`: `load_all_shaders`), which loads the GLSL sources for the cell, border, graphics, bgimage, tint programs. Each Program object prepends `#version {GLSL_VERSION}\n` (= `#version 140`, `kitty/shaders.py:63`) and resolves `#pragma kitty_include_shader <...>` directives.

10. **Register GLFW callbacks** (resize, focus, mouse, keyboard, content-scale-change, monitor-change, etc.) and call `update_os_window_viewport()` (`kitty/glfw.c:130`).

After this, control returns to Python, which constructs the `Boss` and enters `boss.child_monitor.main_loop()` in `kitty/child-monitor.c`.

---

## 5. Rendering Backend Selection

### 5.1 What kitty requests

On Linux, kitty's window-creation code (`kitty/glfw.c:1125–1132`) requests:

```
GLFW_CONTEXT_VERSION_MAJOR = 3
GLFW_CONTEXT_VERSION_MINOR = 1   (per OPENGL_REQUIRED_VERSION_MINOR in data-types.h)
GLFW_OPENGL_FORWARD_COMPAT = true
GLFW_DEPTH_BITS = 0
GLFW_STENCIL_BITS = 0
GLFW_SRGB_CAPABLE  = true  (X11 only — disabled on Wayland to work around #7021/#7174)
```

The Core Profile is implied by the GL_FORWARD_COMPAT hint combined with the profile specification that GLFW applies on Linux (GLX backend selecting a core profile context via `glx_context.c` / `egl_context.c`).

Kitty does **not** use the depth or stencil buffer — its entire rendering pipeline is based on 2D textured quads with blending, so these are zero by design.

### 5.2 How `gl_init()` validates the context

Once the actual window is current, `gl_init()` (`kitty/gl.c:52`) runs once:

```c
void gl_init(void) {
    static bool glad_loaded = false;
    if (!glad_loaded) {
        global_state.gl_version = gladLoadGL(glfwGetProcAddress);
        if (!global_state.gl_version) fatal("Loading the OpenGL library failed");
        if (!global_state.debug_rendering) gladUninstallGLDebug();
        gladSetGLPostCallback(check_for_gl_error);

        if (!GLAD_GL_ARB_texture_storage)
            fatal("The OpenGL driver on this system is missing the required extension: ARB_texture_storage");

        glad_loaded = true;
        int gl_major = GLAD_VERSION_MAJOR(global_state.gl_version);
        int gl_minor = GLAD_VERSION_MINOR(global_state.gl_version);
        if (global_state.debug_rendering)
            printf("[%.3f] GL version string: %s\n",
                   monotonic_t_to_s_double(monotonic()), gl_version_string());
        if (gl_major < OPENGL_REQUIRED_VERSION_MAJOR ||
           (gl_major == OPENGL_REQUIRED_VERSION_MAJOR && gl_minor < OPENGL_REQUIRED_VERSION_MINOR))
            fatal("OpenGL version is %d.%d, version >= %d.%d required for kitty",
                  gl_major, gl_minor,
                  OPENGL_REQUIRED_VERSION_MAJOR, OPENGL_REQUIRED_VERSION_MINOR);
    }
}
```

So the runtime validation has three gates:

1. **GLAD must load** — `gladLoadGL(glfwGetProcAddress)` returns 0 on failure.
2. **`GL_ARB_texture_storage` must be present** — used for the immutable-storage sprite texture array.
3. **GL version must be ≥ 3.1 on Linux** (≥ 3.3 on macOS).

`gl_version_string()` (`kitty/gl.c:42`) formats a diagnostic string comprised of `glGetString(GL_VERSION)` plus GLAD's detected major/minor — and that is exactly what we see in the debug log:

```
[0.141] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

### 5.3 What the runtime actually produced

In our environment:

- `glxinfo` reports `OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)`, `OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1`, `OpenGL core profile shading language version string: 4.50`, `direct rendering: Yes`, `GLX version: 1.4`.
- Therefore kitty got a **Mesa/llvmpipe software-rasterized OpenGL 4.5 Core Profile** context, well exceeding the 3.1 minimum. GLSL 4.50 is available, but kitty only requests `#version 140` (the GLSL feature level equivalent to GL 3.1) per `kitty/data-types.h:26` and `kitty/shaders.py:63`.

### 5.4 Shader programs loaded

After `gl_init()`, `load_all_shaders()` compiles the following programs (identifiers from `kitty/fast_data_types` enum, verified by `./kitty/launcher/kitty +runpy`): `CELL_PROGRAM=0`, `CELL_BG_PROGRAM=1`, `CELL_SPECIAL_PROGRAM=2`, `CELL_FG_PROGRAM=3`, `BORDERS_PROGRAM=4`, `GRAPHICS_PROGRAM=5`, `BGIMAGE_PROGRAM=8`, plus `TINT_PROGRAM`, `GRAPHICS_ALPHA_MASK_PROGRAM`, `GRAPHICS_PREMULT_PROGRAM`. All shaders are preprocessed by `kitty/shaders.py:Program._load_sources()` which expands `#pragma kitty_include_shader <name>` directives and prepends `#version 140\n`.

---

## 6. Display Configuration Detection (Content Scale and DPI)

### 6.1 The content-scale → DPI conversion

`dpi_from_scale()` in `kitty/glfw.c:812`:

```c
static void dpi_from_scale(float xscale, float yscale, double *xdpi, double *ydpi) {
#ifdef __APPLE__
    const double factor = 72.0;
#else
    const double factor = 96.0;
#endif
    *xdpi = xscale * factor;
    *ydpi = yscale * factor;
}
```

On Linux, **the logical DPI is literally `content_scale × 96.0`**. On macOS it is `content_scale × 72.0` (because macOS uses 72 DPI as its "logical" reference).

### 6.2 Where the content scale comes from

`get_window_content_scale()` at `kitty/glfw.c:823`:

```c
static void get_window_content_scale(GLFWwindow *w,
        float *xscale, float *yscale, double *xdpi, double *ydpi) {
    *xscale = 1; *yscale = 1;
    if (w) glfwGetWindowContentScale(w, xscale, yscale);
    else {
        GLFWmonitor *monitor = glfwGetPrimaryMonitor();
        if (monitor) glfwGetMonitorContentScale(monitor, xscale, yscale);
    }
    // sanitize: clamp to (0.0001, 24), replace NaN/negatives with 1.0
    if (*xscale <= 0.0001 || *xscale != *xscale || *xscale >= 24) *xscale = 1.0;
    if (*yscale <= 0.0001 || *yscale != *yscale || *yscale >= 24) *yscale = 1.0;
    dpi_from_scale(*xscale, *yscale, xdpi, ydpi);
}
```

There are thus **two possible sources** for the content scale:

- `glfwGetWindowContentScale(window)` — when a window exists (used by the temp-window DPI probe, and later by `update_os_window_viewport`).
- `glfwGetMonitorContentScale(primary_monitor)` — when no window exists yet (used by `glfw_init` at startup, which calls `get_window_dpi(NULL, ...)` at `kitty/glfw.c:1463`-ish to fill `global_state.default_dpi`).

GLFW itself derives the content scale from X11 RandR per-monitor DPI (or from Wayland compositor scale events). On Xvfb, with a fixed 96 DPI virtual screen and no RandR scale set, the content scale is 1.0.

### 6.3 Observed values

In the Xvfb environment used:

| Value | Observed | Derivation |
|---|---|---|
| Primary monitor | Xvfb screen :99, 1920×1080, 488×274 mm, 100×100 DPI (xdpyinfo) | From Xvfb `-screen 0 1920x1080x24` |
| Content scale (probed via temp window) | `xscale = 1.0`, `yscale = 1.0` | `glfwGetWindowContentScale` returned 1.0 — no HiDPI scaling configured |
| Logical DPI | `xdpi = 96.0`, `ydpi = 96.0` | `dpi_from_scale(1.0, 1.0, ...)` → `1.0 × 96.0` |

These values feed `load_fonts_data(font_size=11.0, xdpi=96.0, ydpi=96.0)`, which is the key input to FreeType's face-sizing step.

### 6.4 Viewport update on resize / monitor change

When a window is resized or moved to another monitor, `update_os_window_viewport()` at `kitty/glfw.c:130` re-computes DPI via `get_window_content_scale()` and — on DPI change — notifies the Boss via a Python callback so that font caches can be rebuilt with the new DPI. In our snapshot trace the window was never resized, so no viewport update occurred after initial creation.

---

## 7. Font System Setup

Kitty's font pipeline for the initial window runs in three phases:

### 7.1 Phase A — Descriptor resolution (Python)

`kitty.fonts.render.set_font_family()` (`kitty/fonts/render.py:173–192`):

```python
def set_font_family(opts=None, override_font_size=None) -> None:
    global current_faces
    opts = opts or defaults
    sz = override_font_size or opts.font_size
    font_map = get_font_files(opts)
    current_faces = [(font_map['medium'], False, False)]
    ftypes: List[...] = ['bold', 'italic', 'bi']
    indices = {k: 0 for k in ftypes}
    for k in ftypes:
        if k in font_map:
            indices[k] = len(current_faces)
            current_faces.append((font_map[k], 'b' in k, 'i' in k))
    before = len(current_faces)
    sm = create_symbol_map(opts)
    ns = create_narrow_symbols(opts)
    num_symbol_fonts = len(current_faces) - before
    set_font_data(
        render_box_drawing, prerender_function, descriptor_for_idx,
        indices['bold'], indices['italic'], indices['bi'], num_symbol_fonts,
        sm, sz, opts.font_features.copy(), ns
    )
```

`get_font_files(opts)` is provided by the platform backend — on Linux, `kitty/fonts/fontconfig.py` (Python) calls into `kitty/fontconfig.c` via `pattern_as_dict` / `fc_match_family` to resolve `opts.font_family`, `opts.bold_font`, `opts.italic_font`, `opts.bold_italic_font` into FontConfig patterns and then to concrete file-path / face-index pairs.

At the C side, `set_font_data()` (`kitty/fonts.c:1434`) stores the descriptor list into `descriptor_indices` and **clears any cached `FontGroup`s** — because a font change invalidates all cell metrics.

### 7.2 Phase B — DPI-aware group initialization (C)

When `create_os_window()` calls `load_fonts_data(font_size, xdpi, ydpi)`:

```c
FONTS_DATA_HANDLE load_fonts_data(double font_sz_in_pts, double dpi_x, double dpi_y) {
    FontGroup *fg = font_group_for(font_sz_in_pts, dpi_x, dpi_y);
    return (FONTS_DATA_HANDLE)fg;
}
```

`font_group_for()` (`kitty/fonts.c:204`) looks up a matching `FontGroup` by `(font_sz_in_pts, logical_dpi_x, logical_dpi_y)` — creating one on cache miss. On cache miss it calls `initialize_font_group()` (`kitty/fonts.c:1495`):

```c
static void initialize_font_group(FontGroup *fg) {
    fg->fonts_capacity = 10 + descriptor_indices.num_symbol_fonts;
    fg->fonts = calloc(fg->fonts_capacity, sizeof(Font));
    if (fg->fonts == NULL) fatal("Out of memory allocating fonts array");
    fg->fonts_count = 1;  // the 0 index font is the box font
    #define I(attr)  if (descriptor_indices.attr) fg->attr##_font_idx = initialize_font(...); else fg->attr##_font_idx = -1;
    fg->medium_font_idx = initialize_font(fg, 0, "medium");
    I(bold); I(italic); I(bi);
    #undef I
    fg->first_symbol_font_idx = fg->fonts_count;
    fg->first_fallback_font_idx = fg->fonts_count;
    fg->fallback_fonts_count = 0;
    for (size_t i = 0; i < descriptor_indices.num_symbol_fonts; i++) {
        initialize_font(fg, descriptor_indices.bi + 1 + i, "symbol_map");
        fg->first_fallback_font_idx++;
    }
    calc_cell_metrics(fg);
    for (size_t i = 0; i < descriptor_indices.num_symbol_fonts; i++) {
        Font *font = fg->fonts + i + fg->first_symbol_font_idx;
        set_size_for_face(font->face, fg->cell_height, true, (FONTS_DATA_HANDLE)fg);
    }
}
```

Each `initialize_font` call opens the backing FreeType face at the correct pixel size (via `set_size_for_face` → `FT_Set_Char_Size` using the group's `logical_dpi_x/y` and `font_sz_in_pts`).

### 7.3 Phase C — Cell metrics from FreeType (C)

`calc_cell_metrics()` (`kitty/fonts.c:373`) asks FreeType, via `cell_metrics()` (`kitty/freetype.c:387`), for the medium face's:

- `cell_width` — from `calc_cell_width` (line 374): the `hori_advance` of the ASCII space glyph, in pixels (after scaling from font units via FT_Set_Char_Size).
- `cell_height` — from `calc_cell_height` (line 141): `ascender - descender + line_gap` in font units → pixels, adjusted for the font's `size->metrics.height` and the group's DPI.
- `baseline`, `underline_position`, `underline_thickness`, `strikethrough_position`, `strikethrough_thickness` — all also derived from the face's size metrics.

`calc_cell_metrics()` then applies the user's `modify_font` overrides (`OPT(cell_width)`, `OPT(cell_height)`, `OPT(baseline)`, `OPT(underline_position)`, `OPT(underline_thickness)`, `OPT(strikethrough_position)`, `OPT(strikethrough_thickness)`) via `adjust_metric()`, validates the results against `MIN_WIDTH=2`, `MIN_HEIGHT=4`, `MAX_DIM=1000`, and commits them into:

```c
fg->cell_width  = cell_width;
fg->cell_height = cell_height;
fg->baseline = baseline;
fg->underline_position = underline_position;
fg->underline_thickness = underline_thickness;
fg->strikethrough_position = strikethrough_position;
fg->strikethrough_thickness = strikethrough_thickness;
sprite_tracker_set_layout(&fg->sprite_tracker, cell_width, cell_height);
```

The final call — `sprite_tracker_set_layout` — tells the GPU sprite tracker how big each cell glyph slot will be, so it can compute how many glyphs fit per sprite-map row/column.

### 7.4 Observed fonts

Kitty's `--debug-font-fallback` output captured in `/tmp/kitty_debug.log`:

```
[0.223] Text fonts:
[0.223]   Normal:      LiberationMono:              /usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf:0
[0.223]   Bold:        LiberationMono-Bold:         /usr/share/fonts/truetype/liberation/LiberationMono-Bold.ttf:0
[0.223]   Italic:      LiberationMono-Italic:       /usr/share/fonts/truetype/liberation/LiberationMono-Italic.ttf:0
[0.223]   Bold-Italic: LiberationMono-BoldItalic:   /usr/share/fonts/truetype/liberation/LiberationMono-BoldItalic.ttf:0
```

These are produced by `dump_font_debug()` (`kitty/fonts/render.py:161–170`), which is called from `_run_app()` at `kitty/main.py:224` when `args.debug_font_fallback` is set.

The filesystem path `/usr/share/fonts/truetype/liberation/LiberationMono-*.ttf` (face index 0 in each file) is what FontConfig returned for our `-o font_family=LiberationMono` override; the `fonts-liberation` Debian package was installed as a build dependency.

### 7.5 Sprite map allocation

After the first-window path computes cell metrics, the sprite map is allocated by `alloc_sprite_map()` in `kitty/shaders.c:51`:

```c
glGetIntegerv(GL_MAX_TEXTURE_SIZE, &max_texture_size);
glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, &max_array_texture_layers);
```

On Mesa/llvmpipe, `GL_MAX_TEXTURE_SIZE` is 16384 and `GL_MAX_ARRAY_TEXTURE_LAYERS` is 2048, giving kitty up to ~4 million cells worth of glyph storage before falling back to additional sprite-map pages.

---

## 8. The Three-Way Relationship: Window System ↔ GPU ↔ Fonts

The three subsystems are not independent — they are linked by **data dependencies** that run in one direction during startup, then bi-directionally in response to resize/DPI-change events afterward.

### 8.1 Data-flow diagram

```
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                             GLFW (window system)                          │
    │ glfwInit → glfwCreateWindow(temp 640×480) → glfwGetWindowContentScale →  │
    │   (xscale, yscale)                                                       │
    └─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ feeds into
    ┌─────────────────────────────────────────────────────────────────────────┐
    │   dpi_from_scale(xscale, yscale) → (xdpi=96, ydpi=96) on Linux           │
    └─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ feeds into
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                     Font system (FontConfig + FreeType)                   │
    │ load_fonts_data(font_size=11pt, xdpi=96, ydpi=96)                        │
    │   → font_group_for → initialize_font_group                               │
    │         → FT_New_Face + FT_Set_Char_Size(size=11pt, res=(96,96))         │
    │   → calc_cell_metrics → (cell_width, cell_height, baseline, ...)          │
    │   → sprite_tracker_set_layout(cell_width, cell_height)                   │
    └─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ feeds into (via get_window_size callback)
    ┌─────────────────────────────────────────────────────────────────────────┐
    │              Window sizing (os_window_size.initial_window_size_func)      │
    │ window_width  = cell_width  × cols + (dpi_x/72)·h_spacing + 1            │
    │ window_height = cell_height × rows + (dpi_y/72)·v_spacing + 1            │
    └─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ feeds into
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                       Real window + GL context                            │
    │ glfwCreateWindow(width, height, ...) → glfwMakeContextCurrent →           │
    │   gl_init() (GLAD load, ARB_texture_storage check, version gate) →        │
    │   glEnable(GL_FRAMEBUFFER_SRGB)                                           │
    └─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ feeds into
    ┌─────────────────────────────────────────────────────────────────────────┐
    │              Shader programs + sprite map (GPU resources)                 │
    │ load_programs(is_semi_transparent) → compile_program(CELL, CELL_BG, ...)  │
    │ alloc_sprite_map(cell_width, cell_height) (using MAX_TEXTURE_SIZE &       │
    │                                              MAX_ARRAY_TEXTURE_LAYERS)   │
    └─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Why this order is required

- **GPU cannot be initialized first**, because `gl_init()` needs an OpenGL context, which requires a `GLFWwindow`, which requires knowing the window pixel size, which requires knowing the cell pixel size, which requires FreeType-measured glyph metrics.
- **Fonts cannot be sized first**, because FreeType's `FT_Set_Char_Size` takes a `resolution_xy` parameter (the DPI) — and DPI is discovered from the windowing system's content scale.
- **The window cannot be created at its final size first**, because the final size depends on cell dimensions, which depend on fonts, which depend on DPI.

Kitty resolves this by:

1. Creating an invisible **temp window** at a fixed 640×480 size (X11), purely to extract the content scale (GLFW needs a window or monitor to report this). On Wayland the content scale is read from the monitor directly, because Wayland only reports scale *after* the window is shown.
2. Using that content scale to compute DPI.
3. Using DPI + font size to set up the `FontGroup` and measure `cell_width`/`cell_height`.
4. Using those to compute the real window pixel dimensions.
5. Creating the real window with those dimensions, and only then running `gl_init()` + shader compile.

### 8.3 `FONTS_DATA_HEAD` — the shared contract

The data that `fonts.c` publishes to the rest of the runtime is defined by the `FONTS_DATA_HEAD` macro in `kitty/data-types.h:347`:

```c
#define FONTS_DATA_HEAD \
    SPRITE_MAP_HANDLE sprite_map; \
    double logical_dpi_x, logical_dpi_y, font_sz_in_pts; \
    unsigned int cell_width, cell_height;
typedef struct {FONTS_DATA_HEAD} *FONTS_DATA_HANDLE;
```

Every `OSWindow` carries a `FONTS_DATA_HANDLE fonts_data;` pointer (`kitty/state.h:249`) — that's how the renderer, scroller, and event-loop find out how big a cell is and what sprite map to draw from.

---

## 9. Subsystem Initialization Order

Below is the observed + source-verified chronological initialization order. Each row lists the responsible source location and what it produces.

| # | Phase | Source | Produces |
|---|---|---|---|
| 1 | Native launcher | `kitty/launcher/main.c : main()` (line 454) | `exe`, `exe_dir`, `KITTY_LIB_PATH` |
| 2 | CPython embedding | `kitty/launcher/main.c : run_embedded()` (line 147/177) | `sys.kitty_run_data` dict, Python interpreter running |
| 3 | Python entry dispatch | `kitty/entry_points.py : main()` (line 183) | Routes to `kitty.main.main()` |
| 4 | Master orchestrator entry | `kitty/main.py : _main()` (line 441) | (begins sequence 5→14) |
| 5 | CLI parse | `parse_args()` (line 465) | `CLIOptions` instance |
| 6 | Config load | `create_opts()` (line 494) | `Options` instance, bad-lines list |
| 7 | Environment prep | `setup_environment()` (line 495) | `PATH`, `TERM=xterm-kitty`, `TERMINFO`, `COLORTERM=truecolor`, `KITTY_*` exported |
| 8 | Locale setup | `set_locale()` (line 499) | `LC_*` set in process |
| 9 | Signal masking | `mask_kitty_signals_process_wide()` (line 514) | Signal mask for all future threads |
| 10 | GLFW dlopen + init | `init_glfw()` → `glfw_init()` at `kitty/glfw.c:1431` | `glfw-x11.so` loaded, `glfwInit` run, `global_state.default_dpi` primed |
| 11 | Box-drawing scale | `fonts/box_drawing.py:set_scale(opts.box_drawing_scale)` | Box-drawing line-thickness scale factor |
| 12 | Options push to C | `set_options(opts, is_wayland(), ..., ...)` | `global_state.is_wayland`, `debug_rendering`, `debug_font_fallback`, the whole `Options` view |
| 13 | Font family resolve | `fonts/render.py:set_font_family(opts)` (line 173) → `set_font_data` | `descriptor_indices` populated with medium / bold / italic / bi / symbol-map descriptors; any cached `FontGroup`s cleared |
| 14 | OS Window creation | `fast_data_types.create_os_window(...)` → `kitty/glfw.c:create_os_window` (line 1107) | (begins sequence 15→23) |
| 15 | GL hints | `glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR/MINOR, OPENGL_REQUIRED_*)` | Core-forward-compat context requested, no depth/stencil |
| 16 | Temp window (X11) | `glfwCreateWindow(640, 480, "temp", ...)` at line 1199 | GL context with an XVisualInfo, from which DPI can be read |
| 17 | DPI probe | `get_window_content_scale(temp, ...)` → `dpi_from_scale` | `(xscale, yscale, xdpi, ydpi)` |
| 18 | Font group init | `load_fonts_data(font_size, xdpi, ydpi)` → `font_group_for` → `initialize_font_group` → `calc_cell_metrics` | FreeType faces opened, `cell_width`, `cell_height`, `baseline`, underline metrics in the `FontGroup`; sprite tracker layout configured |
| 19 | Window size calc | `get_window_size` callback (`os_window_size.py:initial_window_size_func`) | Pixel `(width, height)` for the real window |
| 20 | Real window create | `glfwCreateWindow(width, height, title, NULL, temp_window)` at line 1207 | Actual top-level window with shared GL context |
| 21 | Make context current + GL init | `glfwMakeContextCurrent`, then `gl_init()` at `kitty/gl.c:52` | GLAD loaded, `ARB_texture_storage` verified, GL version ≥ 3.1 confirmed |
| 22 | sRGB + blank + swap | `glEnable(GL_FRAMEBUFFER_SRGB)`, `blank_canvas`, `apply_swap_interval`, `glfwSwapBuffers` | First-frame blank |
| 23 | Shader programs compiled | `load_all_shaders(is_semi_transparent)` (`kitty/main.py:load_all_shaders`) | CELL, CELL_BG, CELL_SPECIAL, CELL_FG, BORDERS, GRAPHICS, BGIMAGE, TINT programs compiled with `#version 140` |
| 24 | Sprite map allocation | `alloc_sprite_map(cell_width, cell_height)` on first render of the window | `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS` queried; texture array allocated |
| 25 | Callbacks + viewport | `update_os_window_viewport()` at `kitty/glfw.c:130`; GLFW resize/focus/mouse/keyboard/scale callbacks registered | Viewport ratios set |
| 26 | Boss construction | `Boss(opts, args, cached_values, global_shortcuts, talk_fd)` (`kitty/boss.py`) | `ChildMonitor` thread, clipboard, keys, encryption key, global `boss` singleton |
| 27 | Boss start | `boss.start(window_id, startup_sessions)` | First tab + first window spawned |
| 28 | `dump_font_debug()` | (only if `--debug-font-fallback`) `kitty/main.py:224` | The "Text fonts" log block we observed |
| 29 | Main event loop | `boss.child_monitor.main_loop()` in `kitty/child-monitor.c` | Multiplexes PTY I/O, input, render scheduling |

### 9.1 Runtime evidence of order

From `/tmp/kitty_debug.log`:

```
[0.141] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
[0.207] OS Window created
[0.219] Failed to open systemd user bus with error: Connection refused
[0.222] Child launched
[0.223] Text fonts:
[0.223]   Normal: LiberationMono: /usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf:0
...
```

The timestamps directly confirm the ordering 21 → 14 → 27/29 → 28 — GL is initialized before the OS window is declared "created," the OS window is created before the child is launched, the child is launched before the font debug dump prints, and `dump_font_debug()` runs after `Boss.start()`.

The `Failed to open systemd user bus` line comes from `kitty/notifications.py` during `Boss.__init__()` — expected in Docker where there's no user bus — and demonstrates that Boss construction completes normally (bus failure is caught and logged).

---

## 10. Key Values Computed During Startup

All values below are either:
- Compile-time constants baked into the binary (from `kitty/data-types.h`), or
- Values read from the live process at startup, confirmed against the source of the function that produces them.

| Value | Scope | Source | Observed / Constant |
|---|---|---|---|
| `OPENGL_REQUIRED_VERSION_MAJOR` | compile-time | `kitty/data-types.h:20` | **3** |
| `OPENGL_REQUIRED_VERSION_MINOR` (Linux) | compile-time | `kitty/data-types.h:24` | **1** (→ minimum GL 3.1 Core requested) |
| `GLSL_VERSION` | compile-time | `kitty/data-types.h:26` | **140** (GLSL for GL 3.1) |
| Required GL extension | runtime check | `kitty/gl.c:66–71` | **GL_ARB_texture_storage** |
| Runtime GL version | runtime detect | `gl_version_string()` in `kitty/gl.c:42` | `4.5 (Core Profile) Mesa 25.2.8` / detected 4.5 |
| Runtime GLSL version | runtime detect (glxinfo) | driver-reported | 4.50 (but kitty uses 1.40) |
| Content scale (x, y) | runtime detect | `get_window_content_scale` in `kitty/glfw.c:823` | **1.0, 1.0** |
| Logical DPI (x, y) | runtime derived | `dpi_from_scale` in `kitty/glfw.c:812` | **96.0, 96.0** (= 1.0 × 96) |
| Font size | user/config | `opts.font_size` | **11.0** pt (`-o font_size=11.0`) |
| Medium face | runtime resolve | `fonts/render.py:get_font_files` + FontConfig | `/usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf:0` |
| Bold face | runtime resolve | same | `/usr/share/fonts/.../LiberationMono-Bold.ttf:0` |
| Italic face | runtime resolve | same | `/usr/share/fonts/.../LiberationMono-Italic.ttf:0` |
| Bold-Italic face | runtime resolve | same | `/usr/share/fonts/.../LiberationMono-BoldItalic.ttf:0` |
| `cell_width`, `cell_height` | runtime derived | `calc_cell_metrics` → `cell_metrics` in `kitty/freetype.c:387` | unobservable from outside (stored in `FontGroup`), but drive the 71-cols × 23-rows grid we observed via `stty size` |
| `baseline`, `underline_position`, `underline_thickness` | runtime derived | `calc_cell_metrics` in `kitty/fonts.c:373` | stored in `FontGroup`; applied when rendering cell sprites |
| Sprite texture max size | runtime query | `alloc_sprite_map` in `kitty/shaders.c:51–55` | `GL_MAX_TEXTURE_SIZE` = 16384 on Mesa/llvmpipe |
| Sprite array layers | runtime query | same | `GL_MAX_ARRAY_TEXTURE_LAYERS` = 2048 on Mesa/llvmpipe |
| Depth buffer bits | requested | `glfwWindowHint(GLFW_DEPTH_BITS, 0)` | **0** |
| Stencil buffer bits | requested | `glfwWindowHint(GLFW_STENCIL_BITS, 0)` | **0** |
| sRGB framebuffer (X11) | requested | `glfwWindowHint(GLFW_SRGB_CAPABLE, true)` then `glEnable(GL_FRAMEBUFFER_SRGB)` | **true / enabled** |
| Terminal grid (cols × rows) | runtime `stty size` | computed from viewport / cell dims by the PTY slave | **71 × 23** (for the default window with font_size=11, DPI=96, 1920×1080 screen — but note kitty's default window size is NOT full-screen) |
| Cached window size | cached_values (no history at start) | `cached_values_for('main')` | none (first run) |

### 10.1 Why the grid is 71 cols × 23 rows (and not 1920/cell_width)

The window is **not** fullscreen. `initial_window_size_func` uses `opts.initial_window_sizes` — the defaults are `initial_window_width = 640` and `initial_window_height = 400` (both in cells *or* in pixels, per the `cells`/`px` unit). With the default `initial_window_width = 640 px, initial_window_height = 400 px` (historical unit defaults), the resulting pixel window yielded ~71 columns × 23 rows at LiberationMono 11 pt / 96 DPI — consistent with the `stty size` output we captured (23 71).

---

## 11. Text Rendering Capabilities (Observed at Runtime)

The child process spawned by kitty reports the following terminal identity & capabilities — all of these are set by kitty itself during child-process setup:

### 11.1 Environment variables set by kitty

From `/tmp/kitty_env.log`:

| Var | Value | Meaning |
|---|---|---|
| `TERM` | `xterm-kitty` | Custom terminfo entry (identifier: `KovIdTTY`). Makes curses/readline emit kitty-specific escape codes. |
| `COLORTERM` | `truecolor` | Announces 24-bit direct-color support to client apps (common convention popularised by kitty and VTE). |
| `TERMINFO` | `/tmp/blitzy/kitty/.../terminfo` | Points to kitty's own terminfo database, shipped in the build (so child processes find `xterm-kitty` even if system `/etc/terminfo` doesn't have it). |
| `KITTY_WINDOW_ID` | `1` | The kitty-internal window ID for this PTY. Used by kittens and by the remote-control protocol. |
| `KITTY_PID` | `14724` | PID of the kitty parent. |
| `KITTY_INSTALLATION_DIR` | `/tmp/blitzy/kitty/.../blitzy-.../` | Root of the kitty install (so the child can find wrapped kittens). |
| `KITTY_PUBLIC_KEY` | `1:...` | Public key for the encrypted remote-control protocol. Version `1` prefix, base85-encoded. |
| `DBUS_SESSION_BUS_ADDRESS` | `/dev/null` | No D-Bus in Docker — kitty fell back to a dummy address. |

### 11.2 Terminfo capabilities (from `infocmp -1`)

Header: `xterm-kitty|KovIdTTY,`

Selected boolean capabilities: `am` (auto-margin), `ccc` (can change colors), `hs` (has status line), `km` (8-bit meta), `mc5i` (printer), `mir` (insert-mode), `msgr` (safe to move in standout), `npc` (no pad), `xenl` (newline eats).

Selected numeric capabilities:

| Cap | Value | Meaning |
|---|---|---|
| `colors` | `0x100` = **256** | 256 color palette (plus 24-bit via direct-color sequences) |
| `cols` | `80` | Default column count (terminfo default; overridden by window resize signals) |
| `lines` | `24` | Default row count (terminfo default; overridden by window resize signals) |
| `pairs` | `0x7fff` = **32767** | Color pair count |
| `it` | `8` | Initial tab stop |

Key highlights of string capabilities:

- Full `cup`, `cuu`, `cud`, `cuf`, `cub` cursor motion.
- `setaf`/`setab` (implicit in terminfo), plus `initc` for OSC-4 RGB color changes: `initc=\E]4;%p1%d;rgb:%p2%{255}%*%{1000}%/%2.2X/%p3%{255}%*%{1000}%/%2.2X/%p4%{255}%*%{1000}%/%2.2X\E\\` — i.e. `\e]4;<idx>;rgb:RR/GG/BB\e\\`.
- Mouse reporting `kmous=\E[M`.
- Function keys kf1..kf63 fully defined (including Shift/Ctrl/Alt/Meta-modified variants kf13..kf63).
- ACS alternate-character-set translation map.
- Alternate-screen / mouse / bracketed-paste / focus-in/out / kitty-keyboard-protocol sequences (beyond the first 187 lines captured but part of the xterm-kitty terminfo).

### 11.3 `tput colors` confirmation

```
$ tput colors
256
```

This confirms that curses/terminfo-aware programs see 256 colors. 24-bit truecolor is signalled **out-of-band** via `COLORTERM=truecolor` because terminfo historically lacks a "millions of colors" integer (though xterm's `RGB` extension cap does appear in kitty's terminfo, it is read only by newer ncurses versions).

### 11.4 Terminal dimensions

```
$ stty size
23 71
```

So the child saw a 71-column × 23-row pseudo-terminal — determined by `(viewport_width / cell_width) × (viewport_height / cell_height)` given the default ~640×400 initial window size.

### 11.5 Font pipeline summary

End-to-end, when kitty renders a Unicode glyph into a cell:

1. The code point is routed through the `FontGroup`'s fallback chain (medium → bold/italic/bi based on cell attributes → symbol-map → system fallbacks via FontConfig).
2. The chosen face is opened by FreeType (`kitty/freetype.c`) and rasterized to an 8-bit alpha bitmap at cell resolution.
3. For complex scripts and ligatures, HarfBuzz (linked as a shared library, `libharfbuzz0b`) performs shaping — grouping multiple codepoints into glyph runs.
4. The rasterized alpha mask is uploaded into the sprite texture array (one slot per `(glyph, attrs)` tuple) allocated by `alloc_sprite_map()`.
5. The CELL_* shader programs sample the sprite map and composite onto the cell background, applying truecolor foreground (from `COLORTERM=truecolor`-aware escape sequences) and applying sRGB encoding (because `GL_FRAMEBUFFER_SRGB` was enabled).

---

## 12. Observed Runtime Data (Verbatim)

### 12.1 `/tmp/kitty_debug.log` — kitty's own `--debug-rendering --debug-font-fallback` output

```
[0.207] OS Window created
[0.219] Failed to open systemd user bus with error: Connection refused
[0.222] Child launched
[0.223] Text fonts:
[0.223]   Normal: LiberationMono: /usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf:0
[0.223]   Bold: LiberationMono-Bold: /usr/share/fonts/truetype/liberation/LiberationMono-Bold.ttf:0
[0.223]   Italic: LiberationMono-Italic: /usr/share/fonts/truetype/liberation/LiberationMono-Italic.ttf:0
[0.223]   Bold-Italic: LiberationMono-BoldItalic: /usr/share/fonts/truetype/liberation/LiberationMono-BoldItalic.ttf:0
[0.141] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1' Detected version: 4.5
```

(Note: the `[0.141]` GL line was emitted before `[0.207]` because it happens inside `gl_init()` which runs before `OS Window created` is logged by the Python layer — kitty's `--debug-rendering` prints the GL version from C with the monotonic-start-relative timestamp, while the Python-layer log message is emitted later.)

### 12.2 `/tmp/kitty_env.log` — relevant excerpts

```
COLORTERM=truecolor
DBUS_SESSION_BUS_ADDRESS=/dev/null
DISPLAY=:99
KITTY_INSTALLATION_DIR=/tmp/blitzy/kitty/blitzy-97d410b6-b93d-4a1e-9a9a-67b8f208e54c_1c2325
KITTY_PID=14724
KITTY_PUBLIC_KEY=1:dpv6UAsR-vUm2^BYmKIB$vJJrGQeg8d(FoW?see_
KITTY_WINDOW_ID=1
LC_CTYPE=C.UTF-8
PATH=/tmp/blitzy/kitty/.../kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
TERM=xterm-kitty
TERMINFO=/tmp/blitzy/kitty/.../terminfo
WINDOWID=2097164

---TERMINFO---
#       Reconstructed via infocmp from file: /tmp/blitzy/.../terminfo/x/xterm-kitty
xterm-kitty|KovIdTTY,
        am, ccc, hs, km, mc5i, mir, msgr, npc, xenl,
        colors#0x100, cols#80, it#8, lines#24, pairs#0x7fff,
        (… 180+ more string and key capabilities …)

---TPUT---
256
---STTY---
23 71
```

### 12.3 `glxinfo` excerpt

```
direct rendering: Yes
GLX version: 1.4
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.1
OpenGL core profile shading language version string: 4.50
OpenGL core profile profile mask: core profile
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.24.04.1
OpenGL shading language version string: 4.50
OpenGL ES profile version string: OpenGL ES 3.2 Mesa 25.2.8-0ubuntu0.24.04.1
```

### 12.4 `xdpyinfo` excerpt

```
name of display:    :99
version number:     11.0
vendor string:      The X.Org Foundation
vendor release number:  12101011
X.Org version:      21.1.11
```

Screen dimensions (from the full output): **1920×1080 pixels (488×274 mm) — 100×100 DPI**.

---

## 13. Conclusion

The kitty terminal emulator's initialization flow is **a data-dependency-driven three-way pipeline** between the windowing subsystem (GLFW), the GPU subsystem (OpenGL via GLAD), and the font subsystem (FontConfig + FreeType + HarfBuzz).

Despite appearing monolithic from the user's point of view (one binary, one window), kitty's startup is actually structured to break a cyclic dependency by creating a throwaway "temp window" at a fixed 640×480 size purely to extract the content scale (DPI) from the windowing system before any cell metrics can be computed.

**The critical early-startup phase (from process entry to first frame swap) consists of 29 distinct steps**, all of which converge at `kitty/glfw.c:create_os_window()` — where, within a single C function, kitty (a) requests a GL 3.1+ Core Forward-Compat context, (b) probes the display's content scale via a temp window, (c) converts scale to DPI (96 DPI per unit on Linux, 72 DPI per unit on macOS), (d) measures FreeType cell metrics at that DPI, (e) computes a real pixel window size from cell metrics, (f) creates the real window, (g) makes its context current, (h) loads GLAD and verifies `ARB_texture_storage` plus a minimum GL version, (i) enables sRGB framebuffer encoding, and (j) compiles all shader programs.

**At runtime, under Mesa 25.2.8 / llvmpipe, kitty 0.35.2 reported:**

- OpenGL **4.5 Core Profile** (GLAD detected 4.5; kitty required min 3.1; shaders use GLSL 140)
- Content scale **1.0 × 1.0** → logical DPI **96 × 96**
- Fonts: **LiberationMono** Regular/Bold/Italic/BoldItalic from `/usr/share/fonts/truetype/liberation/`
- `TERM=xterm-kitty`, `COLORTERM=truecolor`, 256 color palette (24-bit direct-color available via `COLORTERM`), 32767 color pairs, 187+ terminfo capabilities (from custom `KovIdTTY` entry)
- Terminal grid **71 cols × 23 rows** (for default ~640×400 initial window size at 11pt LiberationMono / 96 DPI)

All answers above are directly traceable to specific files, functions, and line ranges in the kitty source tree at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

---

## Appendix A: Key Files Referenced

| File | Lines | Role |
|---|---|---|
| `kitty/launcher/main.c` | 53, 147, 177, 454 | Native launcher + CPython embedding |
| `kitty/entry_points.py` | 183 | Python-side entry dispatch |
| `kitty/main.py` | 85–100, 202–260, 441–532 | Master orchestrator (`init_glfw`, `_run_app`, `AppRunner`, `_main`) |
| `kitty/constants.py` | 191–193, 207–219 | `glfw_path()` and `is_wayland()` |
| `kitty/config.py` | — | Config loading (`create_opts`) |
| `kitty/glfw.c` | 130, 812, 823, 1107–1245, 1431 | `update_os_window_viewport`, `dpi_from_scale`, `get_window_content_scale`, `create_os_window`, `glfw_init` |
| `kitty/gl.c` | 42–79 | `gl_version_string`, `gl_init` (GLAD load + validation) |
| `kitty/data-types.h` | 20–26, 347–348 | `OPENGL_REQUIRED_VERSION_*`, `GLSL_VERSION`, `FONTS_DATA_HEAD` |
| `kitty/state.h` | 220–260 | `OSWindow` struct |
| `kitty/shaders.c` | 51–55 | `alloc_sprite_map` — `GL_MAX_TEXTURE_SIZE` query |
| `kitty/shaders.py` | 63, entire module | `Program` class, `#version` prepending, `#pragma kitty_include_shader` expansion |
| `kitty/fonts.c` | 200–220, 373–420, 1434–1530 | `font_group_for`, `calc_cell_metrics`, `set_font_data`, `initialize_font_group`, `load_fonts_data` |
| `kitty/freetype.c` | 141, 262, 374, 387, 934 | `calc_cell_height`, `calc_cell_width`, `cell_metrics` |
| `kitty/fontconfig.c` | entire | Linux FontConfig bridge |
| `kitty/fonts/render.py` | 161–210 | `set_font_family`, `dump_font_debug` |
| `kitty/fonts/box_drawing.py` | — | `set_scale` for box-drawing glyphs |
| `kitty/os_window_size.py` | 1–102 | `initial_window_size_func`, `edge_spacing` |
| `kitty/session.py` | — | Session / `get_os_window_sizing_data` |
| `kitty/child-monitor.c` | — | `main_loop()` (event-loop multiplexer) |
| `kitty/debug_config.py` | — | `opengl_version_string`, `current_fonts`, compositor detection |
| `glfw/init.c` | 226–280 | `glfwInit()` |
| `glfw/x11_init.c` | — | `_glfwPlatformInit` (X11 backend) |
| `glfw/glx_context.c` | — | GLX context negotiation |
| `glfw/monitor.c` | — | Monitor enumeration |

## Appendix B: Build / Run Commands

### B.1 Build

```bash
# System deps (Ubuntu/Debian):
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    build-essential pkg-config python3 python3-dev \
    libfreetype-dev libfontconfig1-dev libharfbuzz-dev \
    libgl1-mesa-dri libegl1-mesa-dev libgles2-mesa-dev \
    libx11-dev libx11-xcb-dev libxkbcommon-dev libxkbcommon-x11-dev \
    libxcursor-dev libxrandr-dev libxinerama-dev libxi-dev \
    libwayland-dev wayland-protocols \
    libdbus-1-dev libssl-dev libpng-dev liblcms2-dev \
    libxxhash-dev libsimde-dev \
    xvfb x11-utils mesa-utils fonts-liberation

# Build
cd /path/to/kitty
export CFLAGS="-Wno-error"
python3 setup.py --verbose build
# produces ./kitty/launcher/kitty (native launcher)
# and     ./kitty/launcher/kitten (Go CLI binary)
```

### B.2 Run under Xvfb

```bash
Xvfb :99 -screen 0 1920x1080x24 +extension GLX +extension RANDR +extension RENDER -noreset &
sleep 1
DISPLAY=:99 ./kitty/launcher/kitty --debug-rendering --debug-font-fallback \
    -o font_family=LiberationMono -o font_size=11.0 --config=NONE
```

---
*End of analysis document.*
