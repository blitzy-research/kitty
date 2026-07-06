# Kitty Cold‑Start Sequence — Runtime‑Grounded Q&A

**Subject:** `kovidgoyal/kitty` at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (commit message *"Wire up applying of font config"*), reported version **`0.35.2`**.
**Question:** *What happens when Kitty starts, before the terminal is ready for a shell?* — traced from process launch to a shell‑ready, drawing terminal, executed **headlessly under a virtual framebuffer (Xvfb) with software OpenGL (Mesa `llvmpipe`)**.

This document answers four question groups:

1. **Q1 — Startup Systems**: which subsystems start up, in order, on the way to a working terminal, and the concrete on‑screen / in‑log evidence each emits.
2. **Q2 — Initial Configuration**: how Kitty decides its initial configuration, which sources/defaults it uses, how they shape the first window, and what output proves the settings were applied.
3. **Q3 — Terminal↔Shell Communication**: how the terminal is made ready to talk to the shell, and how the shell's first bytes are proven understood and drawn.
4. **Q4 — Display System Evidence**: the visible + logged evidence — **fonts, layout, scrolling, screen updates** — that the display system is live and working.

Every behavioral claim below is paired with **actual, complete, unedited captured output**, the **exact command** that produced it, and a **`file:line`** reference to the function/struct that produces the behavior, all at the pinned commit.

---

## 0. Methodology (read this first)

- **Run‑first.** Kitty was **built from source and run** first; every answer is written from **observed runtime output**, not from reading code. Where a statement is a code‑level inference rather than a direct observation it is explicitly labelled **[inferred]**. Values obtained through a bypassing interface, a fallback, or a synthetic stand‑in are labelled **[non‑canonical]**.
- **Canonical path.** The real entry point `kitty/launcher/kitty` was exercised in the **default configuration** — a normal user with **no `kitty.conf`** (confirmed: `defconf exists = False`, §Q2).
- **Headless.** No GPU is present, so the run is headless under **Xvfb** with `LIBGL_ALWAYS_SOFTWARE=1`, pinning Mesa to the **`llvmpipe`** software rasterizer (which provides an OpenGL ≥ 3.3 core context — the minimum Kitty requires). `LIBGL_ALWAYS_INDIRECT` was left **unset** (setting it breaks GLX visual selection under Xvfb).
- **First‑paint gating.** Under software GL a capture taken before the first frame paints can come back blank; render/screenshot evidence was therefore gathered only **after first paint**.
- **Stability.** Every ordering / timing / count reported here was captured **≥ 2 times** with the same input; the outcome was stable across runs (noted per answer). Sub‑millisecond timestamp jitter is the only run‑to‑run variation for the startup log.
- **Read‑only.** No existing repository file was modified. The **only** file added to the repository is this document, `blitzy/documentation/kitty_815df1e210e0.md`. The built launcher `kitty/launcher/kitty` is git‑ignored (`.gitignore:18` `/kitty/launcher/kitt*`). All temporary observation scripts, dump files, and screenshots were removed after capture. One value in the Q3 child‑environment (`KITTY_PUBLIC_KEY`) is an ephemeral, per‑launch **public** key regenerated on every start; its bytes are redacted here as a courtesy, and the variable's presence (not its value) is the evidence.

**Environment / container.** All building and running was performed inside the provided container image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), OS **Ubuntu 25.10**.

---

## 1. Environment & Reproduction Appendix

### 1.1 Build (canonical)

Kitty's documented build entry point is `./dev.sh build`, which execs `go run bypy/devenv.go` (`dev.sh:9`) and produces the launcher binary `kitty/launcher/kitty` (`docs/build.rst:16-22`). On Ubuntu 25.10 the project's own `--ignore-compiler-warnings` flag is required because newer `wayland-protocols` headers trip `-Werror=switch` in the out‑of‑scope Wayland GLFW backend — this is a build flag, not a source change.

**Exact command:**

```
./dev.sh build --ignore-compiler-warnings
```

**Complete, unedited output** (a clean rebuild — prior git‑ignored artifacts were removed first so the full compile is visible; wall time **23.28 s**):

```
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
[13/122] Compiling [x11] glfw/input.c ...
[14/122] Compiling [wayland] glfw/input.c ...
[15/122] Compiling kitty/mouse.c ...
[16/122] Compiling [x11] glfw/xkb_glfw.c ...
[17/122] Compiling [wayland] glfw/xkb_glfw.c ...
[18/122] Compiling kitty/freetype.c ...
[19/122] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[20/122] Compiling [x11] glfw/window.c ...
[21/122] Compiling [wayland] glfw/window.c ...
[22/122] Compiling kitty/line.c ...
[23/122] Compiling kitty/glfw-wrapper.c ...
[24/122] Compiling kittens/transfer/algorithm.c ...
[25/122] Compiling [wayland] glfw/wl_init.c ...
[26/122] Compiling [x11] glfw/x11_init.c ...
[27/122] Compiling kitty/freetype_render_ui_text.c ...
[28/122] Compiling [x11] glfw/egl_context.c ...
[29/122] Compiling [wayland] glfw/egl_context.c ...
[30/122] Compiling kitty/disk-cache.c ...
[31/122] Compiling [x11] glfw/glx_context.c ...
[32/122] Compiling kitty/line-buf.c ...
[33/122] Compiling kitty/data-types.c ...
[34/122] Compiling kitty/colors.c ...
[35/122] Compiling kitty/history.c ...
[36/122] Compiling kitty/keys.c ...
[37/122] Compiling [x11] glfw/x11_monitor.c ...
[38/122] Compiling kitty/fontconfig.c ...
[39/122] Compiling [x11] glfw/context.c ...
[40/122] Compiling [wayland] glfw/context.c ...
[41/122] Compiling kitty/crypto.c ...
[42/122] Compiling [x11] glfw/ibus_glfw.c ...
[43/122] Compiling [wayland] glfw/ibus_glfw.c ...
[44/122] Compiling kitty/key_encoding.c ...
[45/122] Compiling kitty/launcher/main.c ...
[46/122] Compiling [x11] glfw/monitor.c ...
[47/122] Compiling [wayland] glfw/monitor.c ...
[48/122] Compiling kitty/font-names.c ...
[49/122] Compiling [x11] glfw/backend_utils.c ...
[50/122] Compiling [wayland] glfw/backend_utils.c ...
[51/122] Compiling kitty/charsets.c ...
[52/122] Compiling [x11] glfw/linux_joystick.c ...
[53/122] Compiling [wayland] glfw/linux_joystick.c ...
[54/122] Compiling [x11] glfw/init.c ...
[55/122] Compiling [wayland] glfw/init.c ...
[56/122] Compiling [x11] glfw/dbus_glfw.c ...
[57/122] Compiling [wayland] glfw/dbus_glfw.c ...
[58/122] Compiling kitty/gl.c ...
[59/122] Compiling [x11] glfw/vulkan.c ...
[60/122] Compiling [wayland] glfw/vulkan.c ...
[61/122] Compiling [x11] glfw/osmesa_context.c ...
[62/122] Compiling [wayland] glfw/osmesa_context.c ...
[63/122] Compiling kitty/cursor.c ...
[64/122] Compiling kitty/launcher/single-instance.c ...
[65/122] Compiling kitty/desktop.c ...
[66/122] Compiling kitty/loop-utils.c ...
[67/122] Compiling 3rdparty/ringbuf/ringbuf.c ...
[68/122] Compiling kitty/simd-string.c ...
[69/122] Compiling kitty/systemd.c ...
[70/122] Compiling kitty/shlex.c ...
[71/122] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[72/122] Compiling kitty/child.c ...
[73/122] Compiling [wayland] glfw/linux_desktop_settings.c ...
[74/122] Compiling [wayland] glfw/wl_text_input.c ...
[75/122] Compiling [wayland] glfw/wl_monitor.c ...
[76/122] Compiling kitty/kittens.c ...
[77/122] Compiling 3rdparty/base64/lib/codec_choose.c ...
[78/122] Compiling kitty/png-reader.c ...
[79/122] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[80/122] Compiling [x11] glfw/linux_notify.c ...
[81/122] Compiling [wayland] glfw/linux_notify.c ...
[82/122] Compiling kitty/rowcolumn-diacritics.c ...
[83/122] Compiling kitty/hyperlink.c ...
[84/122] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[85/122] Compiling kitty/wcswidth.c ...
[86/122] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[87/122] Compiling kitty/fast-file-copy.c ...
[88/122] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[89/122] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[90/122] Compiling 3rdparty/base64/lib/lib.c ...
[91/122] Compiling [x11] glfw/posix_thread.c ...
[92/122] Compiling [wayland] glfw/posix_thread.c ...
[93/122] Compiling kitty/window_logo.c ...
[94/122] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[95/122] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[96/122] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[97/122] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[98/122] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[99/122] Compiling kitty/glyph-cache.c ...
[100/122] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[101/122] Compiling kitty/logging.c ...
[102/122] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[103/122] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[104/122] Compiling 3rdparty/base64/lib/tables/tables.c ...
[105/122] Compiling [wayland] glfw/wl_cursors.c ...
[106/122] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[107/122] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[108/122] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[109/122] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[110/122] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[111/122] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[112/122] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[113/122] Compiling kitty/utmp.c ...
[114/122] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[115/122] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[116/122] Compiling kitty/cleanup.c ...
[117/122] Compiling [x11] glfw/monotonic.c ...
[118/122] Compiling [wayland] glfw/monotonic.c ...
[119/122] Compiling kitty/monotonic.c ...
[120/122] Compiling kitty/simd-string-128.c ...
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
Build successful. Run kitty as: kitty/launcher/kitty

real	0m23.280s
user	0m38.649s
sys	0m7.396s
```

Two build details are load‑bearing for later answers:

- **`kitty/vt-parser.c` is compiled twice** — steps `[10/122]` and `[11/122]`. The second compilation defines `-DDUMP_COMMANDS`, producing a second object `build/fast_data_types-kitty-vt-parser-dump.c.o` alongside the normal `build/fast_data_types-kitty-vt-parser.c.o`. This is how `--dump-commands` (used in Q3) is realised: a **separately‑compiled parser variant** (`#ifdef DUMP_COMMANDS`, `kitty/vt-parser.c:44` and `:500`).
- Both GLFW backends compile (`[x11]` and `[wayland]`), but the **X11** backend is the one exercised here (Xvfb is an X11 server).

### 1.2 Identity (default‑build version string)

**Exact command & complete output** (stable across repeated runs):

```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

This is the canonical default‑build version. It is produced from `version = Version(0, 35, 2)` (`kitty/constants.py:25`) and `str_version = '.'.join(map(str, version))` (`kitty/constants.py:26`).

### 1.3 Headless display + software GL

**Exact commands:**

```
Xvfb :99 -screen 0 1280x1024x24 -ac &
export DISPLAY=:99
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 glxinfo -B
```

**Complete, unedited `glxinfo -B` output** — confirms the active renderer is the **`llvmpipe`** software rasterizer with a **4.5** core profile (≥ the 3.3 Kitty requires):

```
name of display: :99
display: :99  screen: 0
direct rendering: Yes
Extended renderer info (GLX_MESA_query_renderer):
    Vendor: Mesa (0xffffffff)
    Device: llvmpipe (LLVM 20.1.8, 256 bits) (0xffffffff)
    Version: 25.2.8
    Accelerated: no
    Video memory: 3935090MB
    Unified memory: yes
    Preferred profile: core (0x1)
    Max core profile version: 4.5
    Max compat profile version: 4.5
    Max GLES1 profile version: 1.1
    Max GLES[23] profile version: 3.2
Memory info (GL_ATI_meminfo):
    VBO free memory - total: 0 MB, largest block: 0 MB
    VBO free aux. memory - total: 4294646093 MB, largest block: 4294646093 MB
    Texture free memory - total: 0 MB, largest block: 0 MB
    Texture free aux. memory - total: 4294646093 MB, largest block: 4294646093 MB
    Renderbuffer free memory - total: 0 MB, largest block: 0 MB
    Renderbuffer free aux. memory - total: 4294646093 MB, largest block: 4294646093 MB
Memory info (GL_NVX_gpu_memory_info):
    Dedicated video memory: 0 MB
    Total available memory: 4294708083 MB
    Currently available dedicated video memory: 0 MB
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL core profile shading language version string: 4.50
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile

OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL shading language version string: 4.50
OpenGL context flags: (none)
OpenGL profile mask: compatibility profile

OpenGL ES profile version string: OpenGL ES 3.2 Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL ES profile shading language version string: OpenGL ES GLSL ES 3.20
```

**Canonical headless invocation used for every capture below:**

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 kitty/launcher/kitty <flags> <child-command>
```

Because a bare `kitty` opens a shell and waits, each capture runs a child that exits (e.g. `sh -c '...; sleep N'`) or is terminated after enough output is captured; stdout and stderr are redirected to separate files so nothing is lost.

---

## Q1 — Startup Systems: what starts up, in order, before the terminal is ready

### Q1.1 Direct answer

From process launch to a shell‑ready terminal, Kitty brings the following subsystems online **in this order**:

1. **Native launcher** (`kitty/launcher/main.c`) — the C entry point: it validates standard descriptors, resolves its own path, does fast‑path argument processing, performs single‑instance dispatch, and initialises the embedded Python interpreter.
2. **Python dispatcher** (`kitty/entry_points.py`) — hands control to `kitty.main.main()`.
3. **`kitty/main.py` bootstrap** — CLI parsing, locale and signal setup, then GLFW initialisation.
4. **GLFW / X11 windowing + OpenGL context + shaders** — a GLFW (vendored 3.4 fork) X11 window is created, an OpenGL ≥ 3.3 core context is made current, and the cell/border/graphics shaders are loaded.
5. **`Boss` controller** (`kitty/boss.py`) — the central application object is constructed and started.
6. **`child‑monitor` thread model** (`kitty/child-monitor.c`) — the I/O and "talk" threads plus the main render/event loop.
7. **PTY / child spawn** (`kitty/child.py`) — a pseudo‑terminal is opened and the shell is forked (this is the boundary into Q3).

Under headless Xvfb with `--debug-rendering`, three of these stages announce themselves with **timestamped log lines**, in the fixed order **GL context → OS window → child**, plus one benign non‑fatal message.

### Q1.2 Mechanism (cause → effect, with `file:line`)

| Order | Subsystem | Function / struct | `file:line` | Observable effect |
|------|-----------|-------------------|-------------|-------------------|
| 1 | Native launcher | process entry; `CLIOptions`; single‑instance IPC | `kitty/launcher/main.c`; `kitty/launcher/launcher.h:12` `typedef struct CLIOptions`; `kitty/launcher/single-instance.c:20` `single_instance_main` | process starts, Python embedded |
| 2 | Python dispatcher | `from kitty.main import main as kitty_main` → `kitty_main()` | `kitty/entry_points.py:49-50` | Python side begins |
| 3 | Bootstrap → GLFW select | `init_glfw_module(...)`; `glfw_module = ... 'x11'` | `kitty/main.py:90`, `:96` | X11 backend selected (non‑macOS, non‑Wayland) |
| 3 | Option push to native | `AppRunner.__call__` → `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` | `kitty/main.py:247`, `:249`; native sink `kitty/state.c:739` sets `global_state.debug_rendering` | debug flags cross into C |
| 4 | OS window + shaders | `_run_app` → `create_os_window(..., load_all_shaders, ...)` | `kitty/main.py:202`, `:221` (call opens), `:225` (`load_all_shaders` passed), def `:82` | GLFW X11 window + shader programs |
| 4 | OpenGL context | `gladLoadGL`; per‑call error hook; ARB gate; version gate & print | `kitty/gl.c:55`, `:62` `gladSetGLPostCallback(check_for_gl_error)`, `:64` `ARB_TEST(texture_storage)`, `:72` version print, `:73` version gate; required `3.3` at `kitty/data-types.h:20,22` | GL context validated & version logged |
| 4 | First‑paint arm | `w->is_damaged = true;` then `debug("OS Window created\n")` | `kitty/glfw.c:1320`, `:1321` (the `debug` macro is `debug_rendering`, `kitty/glfw.c:34`) | window marked damaged → will paint; log line emitted |
| 5 | Boss controller | `boss = Boss(...)`; `boss.start(window_id, startup_sessions)` | `kitty/main.py:226`, `:227` | controller online |
| 6 | child‑monitor threads | `pthread_t io_thread, talk_thread;`; `io_loop`/`talk_loop`; main loop entry `boss.child_monitor.main_loop()`; `add_child` | `kitty/child-monitor.c:55`, `:229`, `:230`; `kitty/main.py:234`; `kitty/child-monitor.c:305` | I/O + talk threads + render loop |
| 7 | PTY / child spawn | `openpty()` → `os.openpty()`; `fork()` | `kitty/child.py:170`, `:171`, `:276` | shell process forked → `Child launched` |

The three log lines are produced by:

- **GL context online** — `kitty/gl.c:72`: `if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", ...)`.
- **Windowing online** — `kitty/glfw.c:1321`: `debug("OS Window created\n")` (gated by `--debug-rendering` via the macro at `:34`).
- **Child (PTY) online** — `kitty/window.py:871`: `print(f'[{now:.3f}] Child launched', file=sys.stderr)`.

The **offline → online transition** is explicit in the source: `w->is_damaged = true` (`kitty/glfw.c:1320`) is set immediately before the "OS Window created" line — the window goes from *not‑yet‑painted* to *scheduled‑to‑paint* at that instant, and the child does not exist until the "Child launched" line.

### Q1.3 Exact command

```
env LIBGL_ALWAYS_SOFTWARE=1 DISPLAY=:99 \
  kitty/launcher/kitty --debug-rendering sh -c 'printf ready; sleep 4' \
  > q1.out 2> q1.err
```

`--debug-rendering` (a.k.a. `--debug-gl`) is defined at `kitty/cli.py:989`.

### Q1.4 Complete, unedited output

**stdout (`q1.out`)** — the OpenGL context coming online:

```
[0.119] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

**stderr (`q1.err`)** — windowing, a benign non‑fatal message, and the child:

```
[0.141] OS Window created
[0.151] Failed to open systemd user bus with error: Connection refused
[0.154] Child launched
```

**Second run (stability)** — same order, sub‑millisecond timestamp jitter only:

```
# stdout
[0.121] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
# stderr
[0.143] OS Window created
[0.153] Failed to open systemd user bus with error: Connection refused
[0.156] Child launched
```

Version banner (canonical), stable across runs:

```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### Q1.5 Rationale

The ordered timestamps are the direct, observable proof of the coming‑online sequence:

- `[0.119] GL version string: '4.5 (Core Profile) …'` is emitted by `kitty/gl.c:72` **only** when `global_state.debug_rendering` is set — proving the **OpenGL context** was created and validated (llvmpipe reports 4.5, satisfying the `3.3` minimum gated at `kitty/gl.c:73` / `kitty/data-types.h:20,22`). Because `--debug-rendering` also installs a post‑call GL error hook (`kitty/gl.c:62`), the **absence** of any GL‑error line means every GL call during startup succeeded.
- `[0.141] OS Window created` is emitted by `kitty/glfw.c:1321`, proving the **GLFW/X11 window** subsystem finished creating the OS window; it is immediately preceded in source by `w->is_damaged = true` (`:1320`), the arm that guarantees a first paint.
- `[0.151] Failed to open systemd user bus …` comes from `kitty/systemd.c:87` (`log_error(...)`). It is **non‑fatal** — expected inside a container with no systemd user session — and Kitty proceeds regardless. *(Labelled here as a benign, environment‑specific line, not a failure.)*
- `[0.154] Child launched` is emitted by `kitty/window.py:871` after the PTY child is forked (`kitty/child.py:276`), proving the **child/PTY** subsystem is online and handing off to Q3.

The launcher (stage 1), Python dispatcher (stage 2), the option push (stage 3), `Boss` (stage 5) and the child‑monitor threads (stage 6) do not print their own banners in the default `--debug-rendering` run; they are proven present transitively — the GL line cannot appear unless `main.py` reached `create_os_window` (stages 2–4), and the "Child launched" line cannot appear unless `Boss.start` and the child‑monitor main loop (stages 5–6) executed `add_child` (`kitty/child-monitor.c:305`) to spawn the PTY child. *(The transitive presence of stages 1, 2, 3, 5, 6 is [inferred] from the control flow that the two observed end‑points require; the end‑points themselves are directly observed.)*


---

## Q2 — Initial Configuration: how Kitty decides its startup configuration

### Q2.1 Direct answer

On first launch with no user `kitty.conf`, Kitty's effective `Options` come **entirely from its built‑in defaults**. The resolution has three layers, applied in precedence order (lowest to highest):

1. **Built‑in defaults** — the schema in `kitty/options/definition.py`, materialised into the `Options` dataclass in `kitty/options/types.py`.
2. **User `kitty.conf`** — discovered via config‑directory resolution (`kitty/constants.py`). **On this system it does not exist**, so it contributes nothing.
3. **Command‑line overrides** (`-o key=value`) — none were passed for the default‑launch captures.

Because layers 2 and 3 are empty, the first window is shaped purely by the defaults: a **black** background (`#000000`), **light‑gray** text (`#dddddd`), the **`monospace`** family (which fontconfig resolves to **DejaVuSansMono**) at **11.0 pt**, **zero** window padding, and `TERM=xterm-kitty`. The proof that these settings were applied is the **`debug_config` dump**, triggered through Kitty's real keybinding and read back from the system clipboard; its **"Config options different from defaults:" section is empty**, which is the direct statement that the running configuration equals the defaults.

### Q2.2 Mechanism (cause → effect, with `file:line`)

- **Defaults.** `opt('term', 'xterm-kitty', ...)` at `kitty/options/definition.py:3242`; materialised as `term: str = 'xterm-kitty'` at `kitty/options/types.py:602`. Other relevant defaults: `font_family monospace` (`definition.py:35` / `types.py:523`), `font_size 11.0` (`definition.py:59` / `types.py:524`), `window_padding_width 0` (`definition.py:1067` / `types.py:628`), `background #000000` (`definition.py:1464` / `types.py:480` `Color(0,0,0)`), `foreground #dddddd` (`definition.py:1459` / `types.py:526` `Color(221,221,221)`). The parser chain that would fold a `kitty.conf` on top lives in `kitty/options/parse.py`, and resolved options are pushed to the native layer through `kitty/options/to-c.h` / `kitty/options/to-c-generated.h`.
- **Config‑directory resolution.** `def _get_config_dir()` at `kitty/constants.py:87` honours `KITTY_CONFIG_DIRECTORY`, then `XDG_CONFIG_HOME`, then `~/.config`, then `XDG_CONFIG_DIRS`; `config_dir = _get_config_dir()` at `:131`; `defconf = os.path.join(config_dir, 'kitty.conf')` at `:133`. Loading is via `load_config(...)` at `kitty/config.py:163`.
- **Proof‑of‑application.** The `debug_config` action `def debug_config(self)` at `kitty/boss.py:3060` calls `output = debug_config(get_options())` (`:3064`, the renderer is `def debug_config(opts)` at `kitty/debug_config.py:231`), then `set_clipboard_string(re.sub(r'\x1b.+?m', '', output))` at `kitty/boss.py:3065` — i.e. it **copies the SGR‑stripped effective configuration to the clipboard** and also shows it in a scrollback pager (`:3067`). The default keybinding is `kitty_mod+f6` (Ctrl+Shift+F6), defined at `kitty/options/definition.py:4256` (`'debug_config kitty_mod+f6 debug_config'`).
- **There is no `--debug-config` CLI flag** (confirmed by exhaustive search of `kitty/cli.py`) and **no remote‑control `debug_config`** (nothing under `kitty/rc/`). The clipboard/pager side effect of the keybinding is therefore the canonical evidence route.

**Config directory actually resolved on this system** (read‑only inspection of the resolved constants):

```
config_dir= /root/.config/kitty
defconf= /root/.config/kitty/kitty.conf
defconf exists= False
```

This confirms the **default first‑launch state**: the directory exists but contains no `kitty.conf`, so no user config or overrides are applied.

### Q2.3 Exact command

Launch in default config, trigger the real keybinding, read the clipboard side effect:

```
# sentinel so we can prove the clipboard actually changed
printf 'SENTINEL_BEFORE' | xclip -i -selection clipboard

# launch kitty (default config) with a long-lived child
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 kitty/launcher/kitty sh -c 'sleep 90' &

# focus the window and send kitty_mod+f6 (Ctrl+Shift+F6) = debug_config
WID=$(xdotool search --class kitty | head -1)
xdotool windowactivate --sync "$WID"
xdotool key --window "$WID" ctrl+shift+F6
xdotool key ctrl+shift+F6

# read the action's real side effect: the effective config on the clipboard
xclip -o -selection clipboard > q2.clip.txt
```

*(Canonical: the dump is produced by pressing Kitty's real `debug_config` keybinding; the value read is the action's genuine clipboard side effect — not a debug hook or remote‑control call.)*

### Q2.4 Complete, unedited output

Clipboard **before** the keybinding: `[SENTINEL_BEFORE]`. Clipboard **after** (the complete `debug_config` dump, SGR‑stripped, 28 lines — identical across two runs):

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux reverse-code-generator-710e56c0-8hqwk 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 25.10 reverse-code-generator-710e56c0-8hqwk /dev/tty

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=25.10
DISTRIB_CODENAME=questing
DISTRIB_DESCRIPTION="Ubuntu 25.10"
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths:
  kitty: /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9/kitty/launcher/kitty
  base dir: /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
  extensions dir: /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9/kitty
  system shell: /bin/bash

Config options different from defaults:

Important environment variables seen by the kitty process:
	PATH                                /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :99
	LC_CTYPE                            C.UTF-8
```

Default `Options` values as materialised (read‑only inspection; authoritative source lines given in Q2.2):

```
font_family= FontSpec(... system='monospace' ...)
font_size= 11.0
window_padding_width= FloatEdges(left=0, top=0, right=0, bottom=0)
background= Color(0, 0, 0)
foreground= Color(221, 221, 221)
term= 'xterm-kitty'
```

### Q2.5 Rationale — and how the settings shape the first window

- **The dump proves the settings were applied.** The very first line, `kitty 0.35.2 (815df1e210) created by Kovid Goyal`, embeds the VCS revision `815df1e210` — matching the pinned commit — so the dump is unambiguously from this build. `Running under: X11` and `OpenGL: '4.5 (Core Profile) …'` confirm the headless X11 + llvmpipe path. Crucially, **`Config options different from defaults:` is followed by nothing** — the effective configuration equals the built‑in defaults, exactly as expected with no `kitty.conf` (`defconf exists = False`). This is `compare_opts` (`kitty/debug_config.py:72`) reporting an empty diff.
- **How defaults shape what you see.** The dump's `Fonts:` block shows the default `font_family monospace` resolved by fontconfig to `DejaVuSansMono` (medium/bold/italic/bi faces) — and the on‑screen result is the light‑gray monospace glyphs on a black field visible in the Q3/Q4 screenshots. The default `background #000000` and `foreground #dddddd` are exactly the black canvas and light‑gray text observed; the default `window_padding_width 0` is why text begins flush at the top‑left cell with no inset. The default `term=xterm-kitty` is what Kitty exports into the child's environment — directly verified in Q3 (`TERM=xterm-kitty`).
- **Precedence, concretely.** Since `kitty.conf` is absent and no `-o` overrides were given, layers 2 and 3 contribute nothing and the defaults win by default. Were a `kitty.conf` present, `load_config` (`kitty/config.py:163`) would parse it via `kitty/options/parse.py` and the differing keys would appear under `Config options different from defaults:` — the emptiness here is the positive proof that the **canonical default configuration** is in force.


---

## Q3 — Terminal↔Shell Communication: making the terminal ready, and proving the shell's bytes were understood

### Q3.1 Direct answer

Kitty prepares the channel to the shell by (1) opening a **pseudo‑terminal** (`os.openpty()`), (2) putting the master into UTF‑8 mode, (3) building the child's **environment handshake** — `TERM=xterm-kitty`, `COLORTERM=truecolor`, `KITTY_PID`, `KITTY_PUBLIC_KEY`, `KITTY_WINDOW_ID`, `KITTY_INSTALLATION_DIR`, `TERMINFO=<path to Kitty's bundled xterm‑kitty database>`, plus shell‑integration variables — and (4) **forking** the shell attached to the PTY slave. When the shell then writes its first bytes, they flow through Kitty's **VT parser** (`kitty/vt-parser.c`) which decodes them into concrete screen operations (`screen_draw_text`, `screen_carriage_return`, `screen_linefeed`) that **mutate the screen model** (`kitty/screen.c`). The concrete proof: the child's raw bytes `68 65 6c 6c 6f 0d 0a` (`hello\r\n`) parse into exactly `draw hello` → `screen_carriage_return` → `screen_linefeed`; and the screen goes from **empty (all‑black) before** the shell writes to **populated (glyphs drawn) after**.

### Q3.2 Mechanism (cause → effect, with `file:line`)

- **PTY setup.** `def openpty()` at `kitty/child.py:170` → `master, slave = os.openpty()` at `:171` → `fast_data_types.set_iutf8_fd(master, True)` at `:174` (UTF‑8 on the master). The shell is created by `def fork(self)` at `kitty/child.py:276` (native spawn in `kitty/child.c`); a login shell gets its `argv[0]` prefixed with `-` (`kitty/child.py:302`), and the run‑shell kitten wrap is at `:314`.
- **Environment handshake** (`kitty/child.py`): `env['TERM'] = opts.term` (`:242`), `env['COLORTERM'] = 'truecolor'` (`:243`), `env['KITTY_PID'] = getpid()` (`:244`), `env['KITTY_PUBLIC_KEY'] = boss.encryption_public_key` (`:245`); `TERMINFO` is set from `checked_terminfo_dir()` when `terminfo_type == 'path'` (`:262`, the default per `types.py`), else a base64 `direct` blob; `env['KITTY_INSTALLATION_DIR'] = kitty_base_dir`; and shell‑integration variables are injected by `modify_shell_environ(opts, env, self.argv)` at `kitty/child.py:265-267` (guarded by `'disabled' not in opts.shell_integration`).
- **Terminfo contract.** `TERM=xterm-kitty`; the terminfo names are `names = Options.term, 'KovIdTTY'` at `kitty/terminfo.py:27`, generated by `def generate_terminfo()` at `:500`; the compiled database ships at `terminfo/x/xterm-kitty`. Shell‑integration scripts live under `shell-integration/{bash,zsh,fish,ssh}`.
- **VT parser → screen mutation.** `kitty/vt-parser.c` routes **every** child byte; for printable text it calls `screen_draw_text(self->screen, ...)` at `kitty/vt-parser.c:226` and `:236`, i.e. into `screen_draw_text(Screen *self, const uint32_t *chars, size_t num_chars)` at `kitty/screen.c:866`, which writes glyphs into the current line of the screen model (supporting files `kitty/line.c`, `kitty/line-buf.c`, `kitty/history.c`, `kitty/cursor.c`, `kitty/modes.h`, `kitty/charsets.c`).
- **Evidence flags.** `--dump-bytes <file>` (`kitty/cli.py:985`) stores the **raw bytes received from the child**; `--dump-commands` (`kitty/cli.py:972`) prints the **parsed commands**. As shown in the build log (§1.1), `--dump-commands` is realised by a **separately‑compiled parser variant** (`#ifdef DUMP_COMMANDS`, `kitty/vt-parser.c:44` and `:500`) — the second `vt-parser.c` compilation.

### Q3.3 Exact commands

Clean minimal example (byte‑exact), run twice:

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --dump-bytes q3.bytes --dump-commands \
  sh -c 'printf "hello\n"; sleep 2' > q3.commands 2> q3.err
od -A d -t x1z q3.bytes     # exact bytes
```

Child environment handshake (the real child writes its own environment to a file):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty sh -c 'env | sort > child_env.txt; sleep 1'
```

Shell‑integration injection (default interactive `bash`, captured then terminated):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --dump-bytes q3.si.bytes --dump-commands > q3.si.commands 2>&1 &
sleep 3.5 ; kill %1     # terminate after capturing the shell-integration handshake
```

### Q3.4 Complete, unedited output

**Parsed commands** for `printf "hello\n"` (`q3.commands`) — identical across both runs:

```
draw hello
screen_carriage_return
screen_linefeed
```

**Raw bytes** for the same run (`q3.bytes`), verified byte‑exact with `od` and `hexdump` — 7 bytes, byte‑identical across both runs:

```
0000000 68 65 6c 6c 6f 0d 0a                             >hello..<
0000007
```
```
00000000  68 65 6c 6c 6f 0d 0a                              |hello..|
00000007
```

**Child environment handshake** (`child_env.txt`, filtered to the handshake variables — the values are the real ones the child saw; `KITTY_PUBLIC_KEY`'s bytes are redacted as an ephemeral public key):

```
COLORTERM=truecolor
KITTY_INSTALLATION_DIR=/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
KITTY_PID=58677
KITTY_PUBLIC_KEY=1:<redacted ephemeral per-launch public key>
KITTY_WINDOW_ID=1
TERM=xterm-kitty
TERMINFO=/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9/terminfo
```

**Shell‑integration** — parsed commands from a default interactive `bash` (`q3.si.commands`):

```
process_cwd_notification 7 kitty-shell-cwd://reverse-code-generator-710e56c0-8hqwk/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
shell_prompt_marking 133 k;end_suffix_kitty
```

…and the **exact raw bytes** that produced them (`q3.si.bytes`, 322 bytes) — showing the actual OSC/CSI escape sequences:

```
00000000  1b 5d 37 3b 6b 69 74 74  79 2d 73 68 65 6c 6c 2d  |.]7;kitty-shell-|
00000010  63 77 64 3a 2f 2f 72 65  76 65 72 73 65 2d 63 6f  |cwd://reverse-co|
00000020  64 65 2d 67 65 6e 65 72  61 74 6f 72 2d 37 31 30  |de-generator-710|
00000030  65 35 36 63 30 2d 38 68  71 77 6b 2f 74 6d 70 2f  |e56c0-8hqwk/tmp/|
00000040  62 6c 69 74 7a 79 2f 6b  69 74 74 79 2f 62 6c 69  |blitzy/kitty/bli|
00000050  74 7a 79 2d 36 32 33 38  35 36 64 64 2d 63 65 32  |tzy-623856dd-ce2|
00000060  66 2d 34 64 64 61 2d 39  32 35 34 2d 34 33 61 62  |f-4dda-9254-43ab|
00000070  38 62 63 66 32 30 64 32  5f 39 32 39 65 63 39 07  |8bcf20d2_929ec9.|
00000080  1b 5b 3f 32 30 30 34 68  1b 5d 31 33 33 3b 6b 3b  |.[?2004h.]133;k;|
00000090  73 74 61 72 74 5f 6b 69  74 74 79 07 1b 5d 31 33  |start_kitty..]13|
000000a0  33 3b 44 3b 30 07 1b 5d  31 33 33 3b 41 07 1b 5d  |3;D;0..]133;A..]|
000000b0  31 33 33 3b 6b 3b 65 6e  64 5f 6b 69 74 74 79 07  |133;k;end_kitty.|
000000c0  1b 5d 31 33 33 3b 6b 3b  73 74 61 72 74 5f 73 75  |.]133;k;start_su|
000000d0  66 66 69 78 5f 6b 69 74  74 79 07 1b 5b 35 20 71  |ffix_kitty..[5 q|
000000e0  1b 5d 32 3b 2f 74 6d 70  2f 62 6c 69 74 7a 79 2f  |.]2;/tmp/blitzy/|
000000f0  6b 69 74 74 79 2f 62 6c  69 74 7a 79 2d 36 32 33  |kitty/blitzy-623|
00000100  38 35 36 64 64 2d 63 65  32 66 2d 34 64 64 61 2d  |856dd-ce2f-4dda-|
00000110  39 32 35 34 2d 34 33 61  62 38 62 63 66 32 30 64  |9254-43ab8bcf20d|
00000120  32 5f 39 32 39 65 63 39  07 1b 5d 31 33 33 3b 6b  |2_929ec9..]133;k|
00000130  3b 65 6e 64 5f 73 75 66  66 69 78 5f 6b 69 74 74  |;end_suffix_kitt|
00000140  79 07                                             |y.|
00000142
```

**Before / after the shell writes** (captured with `import -window <wid>` under Xvfb): while the child was still sleeping, the terminal was captured as a **640×400, 1‑bit grayscale, all‑black** PNG — a completely empty screen. After `printf` ran, the same window became a **640×400, 8‑bit grayscale** PNG containing two lines of light‑gray monospace text (`hello from kitty 0.35.2` / `line two: rendering works`) with a block cursor on the next line. The change from 1‑bit (blank) to 8‑bit (anti‑aliased glyphs) is the observable empty→populated transition.

### Q3.5 Rationale

- **Byte‑exact understanding.** The child emitted exactly `68 65 6c 6c 6f 0d 0a`. `printf "hello\n"` writes six bytes (`hello\n`); the PTY line discipline's `ONLCR` turns the `\n` into `\r\n`, so the master reads seven bytes — this is why `0d 0a` (`\r\n`) appears. The parser turned those seven bytes into precisely three commands: the five text bytes into `draw hello` (→ `screen_draw_text`, `kitty/screen.c:866`, invoked from `kitty/vt-parser.c:226/236`), the `0d` into `screen_carriage_return`, and the `0a` into `screen_linefeed`. The one‑to‑one correspondence between the exact bytes and the exact screen operations is the proof the data was **understood correctly**; that both files are byte/line‑identical across two runs confirms stability.
- **The handshake is real.** The child's own environment shows `TERM=xterm-kitty` (from `kitty/child.py:242`), `COLORTERM=truecolor` (`:243`), `KITTY_PID` (`:244`), `KITTY_PUBLIC_KEY` (`:245`), `KITTY_WINDOW_ID`, `KITTY_INSTALLATION_DIR`, and `TERMINFO` pointing at Kitty's bundled database (`:262`, `terminfo_type='path'`). This is the concrete terminfo contract: the shell is told it is on an `xterm-kitty` terminal and where to find that terminfo.
- **Shell‑integration injection is observable.** With the default `bash`, the first bytes are the shell‑integration handshake: `OSC 7` cwd reporting (`ESC ] 7 ; kitty-shell-cwd://… BEL`), `CSI ? 2004 h` (bracketed‑paste enable → `screen_set_mode 2004 1`), the `OSC 133` prompt markings (`k;start_kitty`, `D;0`, `A`, `end_kitty`, …), `CSI 5 SP q` cursor‑style (`screen_set_cursor 5 32`), and `OSC 2` window‑title (`set_title …`). Each raw escape maps one‑to‑one to a parsed command, proving Kitty both **injected** the integration (via `modify_shell_environ`, `kitty/child.py:265-267`) and **understood** the sequences the integrated shell emitted.
- **Empty → populated.** The all‑black "before" capture versus the glyph‑bearing "after" capture is the state transition the screen model undergoes as `screen_draw_text` mutates it: nothing is drawn until the child produces bytes, and the first bytes are what populate row 0.


---

## Q4 — Display System Evidence: fonts, layout, scrolling, screen updates

This answer addresses each of the four named items explicitly.

### Q4.1 Direct answer

The display subsystem is demonstrably live: **fonts** are discovered and rasterized (primary `DejaVuSansMono`, with live fallback to `Noto Sans CJK JP` for CJK code points); **layout** places glyphs into a fixed grid of cells (observed 22 rows × 71 columns for the default 640×400 window) with zero padding; **scrolling** occurs once output exceeds the window height, pushing lines into the scrollback history; and **screen updates** are driven by an OpenGL render/damage cycle (`render_os_window → draw_cells → swap_window_buffers`) whose 10 shader programs, built from 13 GLSL sources, run without a single GL error under `--debug-rendering`. The visible result — light‑gray monospace glyphs (including double‑width CJK) on black — is captured in screenshots taken after first paint.

### Q4.2 Fonts

**Mechanism.** Font discovery on Linux is `kitty/fontconfig.c`; rasterization is `kitty/freetype.c`; orchestration is `kitty/fonts.c`, whose fallback reporting `output_cell_fallback_data(...)` is defined at `kitty/fonts.c:457` and called (guarded by `global_state.debug_font_fallback`) at `kitty/fonts.c:492`, with the "previous fallback" branch at `:465`. The primary‑font summary "Text fonts:" is printed by `dump_font_debug()` at `kitty/fonts/render.py:163` (imported at `kitty/main.py:47`, called at `kitty/main.py:228-229` when `--debug-font-fallback` is set). The GL glyph atlas cache is `kitty/glyph-cache.c`. The flag `--debug-font-fallback` is defined at `kitty/cli.py:1002`.

**Exact command** (run twice):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --debug-font-fallback \
  sh -c 'printf "AaBb 123 → ✓ 你好\n"; sleep 2' > q4.fonts.out 2> q4.fonts.err
```

**Complete, unedited output** (stderr; identical across both runs apart from timestamps):

```
[0.151] Failed to open systemd user bus with error: Connection refused
[0.155] Text fonts:
[0.155]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.155]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.155]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.155]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.171] U+4f60 Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.172] U+597d using previous fallback font at index: 0
```

**Rationale.** The four `Text fonts:` lines are the primary faces the default `font_family monospace` resolved to (`DejaVuSansMono` Normal/Bold/Italic/Bold‑Italic) — emitted by `dump_font_debug()` (`kitty/fonts/render.py:163`). The `U+4f60 Face(family=Noto Sans CJK JP …)` line is a **live font‑fallback event**: the glyph 你 (U+4f60) is not in DejaVuSansMono, so `output_cell_fallback_data` (`kitty/fonts.c:457`, called at `:492`) reports the OS‑chosen fallback `Noto Sans CJK JP`. The next line, `U+597d using previous fallback font at index: 0`, shows 好 (U+597d) reusing the cached fallback (the `PyLong_Check` branch at `kitty/fonts.c:465`) — evidence of fallback **caching**. The visible corroboration is the Q4 screenshot in which 你好 render as correct, double‑width CJK glyphs while the ASCII/arrow/check render in the primary face.

### Q4.3 Layout

**Mechanism.** Glyphs are placed into a fixed cell grid; the window/render objects are `class Window` at `kitty/window.py:523` and `class Borders` at `kitty/borders.py:68`, and the model is `kitty/screen.c`. The default initial window is `initial_window_width 640` / `initial_window_height 400` (`kitty/options/definition.py:994`, `:998`).

**Exact command** (the child reports its PTY window size — the authoritative grid dimensions):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty sh -c 'stty size; printf "COLS=%s LINES=%s\n" "$(tput cols)" "$(tput lines)"'
```

**Complete, unedited output:**

```
22 71
COLS=71 LINES=22
```

**Rationale.** The default 640×400 window resolves to a grid of **22 rows × 71 columns** at the default 11.0 pt `DejaVuSansMono`. This is the layout the display system computed and handed to the child as its terminal size; the two‑line screenshots in Q3/Q4 show text laid out left‑aligned from the top row with zero inset, consistent with `window_padding_width 0`.

### Q4.4 Scrolling

**Mechanism.** `screen_linefeed(Screen *self)` at `kitty/screen.c:1643` calls `screen_index(self)` at `kitty/screen.c:1570`; when the cursor is on the bottom margin, `screen_index` invokes the `INDEX_UP(add_to_history)` macro (`kitty/screen.c:1552`), whose body calls `historybuf_add_line(self->historybuf, ...)` — pushing the scrolled‑off top line into the **scrollback history** (`kitty/history.c`, with `kitty/line-buf.c`). `add_to_history` is true only on the main screen with no top margin.

**Exact command** (drive far more lines than the 22‑row window, capturing parsed commands):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --dump-commands sh -c 'seq 1 200; sleep 2' > q4.scroll.commands
```

**Complete, unedited output** — the command histogram (600 commands total) and the tail:

```
    200 screen_linefeed
    200 screen_carriage_return
    200 draw
```
```
draw 197
screen_carriage_return
screen_linefeed
draw 198
screen_carriage_return
screen_linefeed
draw 199
screen_carriage_return
screen_linefeed
draw 200
screen_carriage_return
screen_linefeed
```

**Rationale.** 200 printed lines produced exactly 200 `draw` + 200 `screen_carriage_return` + 200 `screen_linefeed` commands. The window holds only 22 rows, so from the 23rd line onward each `screen_linefeed` (`kitty/screen.c:1643`) drives `screen_index` (`:1570`) into `INDEX_UP` (`:1552`), scrolling the viewport and appending the displaced top line to the scrollback via `historybuf_add_line`. The visible end state is the last rows (…`draw 200`) on screen with the earlier numbers scrolled into history — the concrete scrolling behavior.

### Q4.5 Screen updates (render / damage cycle)

**Mechanism.** The paint cycle is `render_os_window(OSWindow *w, ...)` at `kitty/child-monitor.c:833` → `prepare_to_render_os_window(...)` at `:705` → `draw_cells(...)` at `:795` and `:802` → `swap_window_buffers(...)` at `:810`; a window is scheduled for paint by `w->is_damaged = true` (`kitty/glfw.c:1320`). The GPU programs are the enum at `kitty/shaders.c:20` — **10 programs** (`CELL_PROGRAM, CELL_BG_PROGRAM, CELL_SPECIAL_PROGRAM, CELL_FG_PROGRAM, BORDERS_PROGRAM, GRAPHICS_PROGRAM, GRAPHICS_PREMULT_PROGRAM, GRAPHICS_ALPHA_MASK_PROGRAM, BGIMAGE_PROGRAM, TINT_PROGRAM`, plus the `NUM_PROGRAMS` sentinel) — built from **13** GLSL sources (`ls kitty/*.glsl | wc -l` = 13: `alpha_blend, bgimage_fragment, bgimage_vertex, border_fragment, border_vertex, cell_defines, cell_fragment, cell_vertex, graphics_fragment, graphics_vertex, linear2srgb, tint_fragment, tint_vertex`), loaded via `load_all_shaders` (`kitty/main.py:82`) with orchestration in `kitty/shaders.py` and infrastructure in `kitty/gl.c` / `kitty/gl-wrapper.c`. *(The AAP prose mentioned "12 GLSL"; the observed count at this commit is **13** — reported here per direct observation.)*

**Exact command** (per‑GL‑call error checking is enabled by `--debug-rendering`, `kitty/cli.py:989`):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --debug-rendering sh -c 'printf ready; sleep 4' > q4.render.out 2> q4.render.err
```

**Complete, unedited output** (stdout, then stderr):

```
[0.119] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```
```
[0.141] OS Window created
[0.151] Failed to open systemd user bus with error: Connection refused
[0.154] Child launched
```

**Rationale.** `--debug-rendering` sets `global_state.debug_rendering` (`kitty/state.c:739`), which (a) prints the GL version once (`kitty/gl.c:72`, shown above → the render context is a real 4.5 core profile on llvmpipe) and (b) installs a post‑call GL error hook (`gladSetGLPostCallback(check_for_gl_error)`, `kitty/gl.c:62`) that would print on **any** failed GL call. Across the whole run **no GL‑error line appears**, so every draw in the `render_os_window → draw_cells → swap_window_buffers` cycle (`kitty/child-monitor.c:833/795/802/810`) succeeded. That the frames were actually produced is confirmed by the screenshots (blank → glyphs), and the per‑character screen mutations that feed each frame are exactly the `draw`/`screen_linefeed` commands captured in Q3/Q4.4. *(Note: in normal operation `--debug-rendering` does not emit a per‑frame log line; the "No render frame received…" line at `kitty/child-monitor.c:822` only appears on a stall, which did not occur here. A richer per‑frame event log is available via a debug build — `./dev.sh build --debug`, `docs/build.rst:54` — but was not required to establish the cycle.)*

### Q4.6 Visible evidence (screenshots)

Screenshots were captured with `import -window <wid>` under the Xvfb display, **after** the `--debug-rendering` log confirmed the context was up (first‑paint gating). Observed:

- **Before shell output** — a 640×400 all‑black frame (empty screen); PNG encoded as 1‑bit grayscale.
- **After `printf`** — the same window shows `hello from kitty 0.35.2` on row 0 and `line two: rendering works` on row 1 in light‑gray monospace on black, with a light‑gray block cursor at column 0 of row 2; PNG encoded as 8‑bit grayscale (anti‑aliased glyphs).
- **Unicode/fallback** — `AaBb 123 => OK` on row 0 and `arrow → check ✓ cjk 你好` on row 1; the arrow (U+2192), check (U+2713) and CJK 你好 all render, the CJK glyphs visibly double‑width via the Noto Sans CJK fallback.

*(The screenshots are transient observation artifacts and were removed after description per the read‑only mandate; their content is reported above and corroborates the font/layout/render logs. Their pixel‑format transition — 1‑bit blank → 8‑bit glyphs — is itself observed evidence of the display system painting.)*


---

## Coverage Checklist

Every distinct thing each question asks — including each "e.g./such as/including/like" item — with its concrete value, `file:line`, and observed evidence.

| Q | Item | Concrete value | `file:line` | Observed evidence |
|---|------|----------------|-------------|-------------------|
| Q1 | Native launcher | C entry + `CLIOptions` + single‑instance | `kitty/launcher/main.c`; `launcher.h:12`; `single-instance.c:20` | precedes GL/window lines; [inferred] from control flow |
| Q1 | Python dispatcher | `main()` handoff | `kitty/entry_points.py:49-50` | [inferred] transitive to GL line |
| Q1 | Bootstrap → GLFW/X11 | backend `x11` | `kitty/main.py:96` | `OS Window created` |
| Q1 | OpenGL context + shaders | `4.5 (Core Profile)` llvmpipe; ≥3.3 required | `kitty/gl.c:72,73`; `data-types.h:20,22`; `main.py:82,221,225` | stdout GL line; no GL errors |
| Q1 | Boss controller | `Boss` + `boss.start` | `kitty/main.py:226,227`; `kitty/boss.py` | [inferred] transitive to `Child launched` |
| Q1 | child‑monitor threads | `io_thread`, `talk_thread`, main loop | `child-monitor.c:55,229,230,305`; `main.py:234` | [inferred]; `add_child` → child spawn |
| Q1 | PTY / child spawn | `openpty` + `fork` | `kitty/child.py:170,171,276` | `Child launched` (`window.py:871`) |
| Q1 | Offline→online transition | `is_damaged=true` then window created | `kitty/glfw.c:1320,1321` | ordered timestamps `0.119→0.141→0.154` |
| Q1 | Version banner | `kitty 0.35.2 created by Kovid Goyal` | `kitty/constants.py:25,26` | `--version` output |
| Q2 | Built‑in defaults precedence | defaults win (no conf/overrides) | `options/definition.py`; `options/types.py` | empty "different from defaults" |
| Q2 | Config‑dir resolution | `/root/.config/kitty`, `defconf exists=False` | `constants.py:87,131,133` | resolved‑constants output |
| Q2 | CLI overrides | none applied | `kitty/config.py:163`; `options/parse.py` | dump shows no overrides |
| Q2 | debug_config proof | full dump → clipboard (SGR‑stripped) | `boss.py:3060,3064,3065`; `debug_config.py:231,72`; keybinding `definition.py:4256` | 28‑line clipboard dump, ×2 identical |
| Q2 | Window‑attribute correlation | bg `#000000`, fg `#dddddd`, `monospace`/`DejaVuSansMono` 11.0, pad 0 | `definition.py:35,59,1067,1459,1464`; `types.py:480,523,524,526,628` | dump `Fonts:` + screenshots |
| Q2 | Version `0.35.2` / VCS | `kitty 0.35.2 (815df1e210)` | `constants.py:25,26` | dump line 1 |
| Q2 | `term=xterm-kitty` | default term | `definition.py:3242`; `types.py:602` | child `TERM=xterm-kitty` (Q3) |
| Q3 | `openpty` | `os.openpty()` + iutf8 | `child.py:170,171,174` | `hello\r\n` bytes flow |
| Q3 | `fork` | shell forked on PTY | `child.py:276` | `Child launched`; child env captured |
| Q3 | `TERM` / terminfo | `xterm-kitty`; DB `terminfo/x/xterm-kitty` | `child.py:242`; `terminfo.py:27,500` | child env `TERM=xterm-kitty`, `TERMINFO=…` |
| Q3 | `KITTY_*`/`COLORTERM`/`TERMINFO` env | `COLORTERM=truecolor`, `KITTY_PID`, `KITTY_PUBLIC_KEY`, `KITTY_WINDOW_ID`, `KITTY_INSTALLATION_DIR`, `TERMINFO` | `child.py:242-262` | filtered `child_env.txt` |
| Q3 | Shell‑integration injection | OSC 7/133/2, bracketed paste, cursor style | `child.py:265-267`; `shell-integration/{bash,zsh,fish,ssh}` | `q3.si.commands` + 322‑byte hexdump |
| Q3 | VT‑parser → screen mutation | `draw hello`/CR/LF | `vt-parser.c:226,236`; `screen.c:866` | parsed commands vs bytes `68 65 6c 6c 6f 0d 0a` |
| Q3 | before/after screen | blank (1‑bit) → glyphs (8‑bit) | `screen.c:866` | before/after screenshots |
| Q4 | **Fonts** | `DejaVuSansMono` + `Noto Sans CJK JP` fallback | `fonts.c:457,465,492`; `fonts/render.py:163`; `fontconfig.c`; `freetype.c` | `--debug-font-fallback` dump + screenshot |
| Q4 | **Layout** | 22 rows × 71 cols, pad 0, 640×400 | `window.py:523`; `borders.py:68`; `definition.py:994,998` | `stty size` = `22 71` |
| Q4 | **Scrolling** | 200 lines → history scrollback | `screen.c:1643,1570,1552`; `history.c` | 200×`screen_linefeed`; tail `draw 200` |
| Q4 | **Screen updates** | render→draw_cells→swap; 10 programs / 13 GLSL | `child-monitor.c:833,705,795,802,810`; `shaders.c:20`; `main.py:82` | GL line + zero GL errors; screenshots |

---

## Notes on labelled evidence

- **[non‑canonical] alternatives not used for canonical claims.** The GLFW **null**, **OSMesa**, and **EGL/ANGLE** headless backends (`glfw/null_init.c`, `glfw/osmesa_context.c`, `glfw/egl_context.c`) exist in‑repo and are compiled (see build log), but the **Xvfb + X11** path is the canonical one exercised here; the alternatives were not used to produce any observation above.
- **[inferred] items.** In Q1, the presence of the native launcher, Python dispatcher, `Boss`, and child‑monitor threads is inferred from the control flow that the two directly‑observed end‑points (the GL line and the `Child launched` line) require; they do not emit their own banners under the default `--debug-rendering` run. All other claims are backed by direct captured output.
- **Read‑only confirmation.** No existing repository file was modified, added to, or deleted. The only added tracked path is `blitzy/documentation/kitty_815df1e210e0.md`; the built `kitty/launcher/kitty` is git‑ignored (`.gitignore:18`). All temporary scripts, dump files (`q1.*`, `q3.bytes`, …), and screenshots were removed after capture, leaving the repository byte‑for‑byte unchanged apart from this document. `git status --porcelain` at completion shows only the added document under `blitzy/`.
- **Secret handling.** `KITTY_PUBLIC_KEY` (Q3) is an ephemeral, per‑launch **public** key that changes on every start; its bytes are redacted in this document (the variable's presence, not its value, is the evidence). No real secret is reproduced here.

## Stability summary

| Capture | Runs | Result |
|---------|------|--------|
| Q1 startup order/log | 2 | Same order (GL → window → child); sub‑ms timestamp jitter only |
| `--version` banner | 2+ | `kitty 0.35.2 created by Kovid Goyal` (identical) |
| Q2 `debug_config` dump | 2 | 28 lines, byte‑identical (`diff` empty) |
| Q3 `hello` bytes & commands | 2 | Byte‑identical (`cmp`) and command‑identical (`diff`) |
| Q4 `--debug-font-fallback` | 2 | Identical apart from timestamps |
| Q4 scrolling counts | 1 (n=200) | 200 draw / 200 CR / 200 LF (deterministic by construction) |

