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
- **Headless.** No GPU is present, so the run is headless under **Xvfb** with `LIBGL_ALWAYS_SOFTWARE=1`, pinning Mesa to the **`llvmpipe`** software rasterizer (which provides an OpenGL **4.5** core context — far exceeding the minimum Kitty requires on this Linux/X11 path, **OpenGL 3.1**, defined at `kitty/data-types.h:20` (major `3`) and `:24` (minor `1`, the non‑Apple `#else` branch); macOS requires `3.3` via the `#ifdef __APPLE__` branch at `:22`). `LIBGL_ALWAYS_INDIRECT` was left **unset** (setting it breaks GLX visual selection under Xvfb).
- **First‑paint gating.** Under software GL a capture taken before the first frame paints can come back blank; render/screenshot evidence was therefore gathered only **after first paint**.
- **Stability.** Every ordering / timing / count reported here was captured **≥ 2 times** with the same input; the outcome was stable across runs (noted per answer). Sub‑millisecond timestamp jitter is the only run‑to‑run variation for the startup log.
- **Read‑only.** No existing repository file was modified. The **only** file added to the repository is this document, `blitzy/documentation/kitty_815df1e210e0.md`. The built launcher `kitty/launcher/kitty` is git‑ignored (`.gitignore:18` `/kitty/launcher/kitt*`). All temporary observation scripts, dump files, and raw `.png` captures were removed after capture; the three evidence screenshots are **embedded inline** in this document as base64 `data:` URIs (see Q2.5, Q3.4, Q4.6), so the document is self-contained and the repository still gains only this one file. One value in the Q3 child‑environment (`KITTY_PUBLIC_KEY`) is an ephemeral, per‑launch **public** key regenerated on every start; because it is a public (not secret) key, its real value **is shown** in Q3.4, and its change across two runs is itself reported as evidence of per-launch regeneration.

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

**Complete, unedited `glxinfo -B` output** — confirms the active renderer is the **`llvmpipe`** software rasterizer with a **4.5** core profile (far above the **3.1** Kitty requires on this Linux/X11 path; see `kitty/data-types.h:20,24`):

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
4. **GLFW / X11 windowing + OpenGL context + shaders** — a GLFW (vendored 3.4 fork) X11 window is created, an OpenGL core context (Linux/X11 minimum **3.1**; `llvmpipe` supplies **4.5**) is made current, and the cell/border/graphics shaders are loaded.
5. **`Boss` controller** (`kitty/boss.py`) — the central application object is constructed and started.
6. **`child‑monitor` thread model** (`kitty/child-monitor.c`) — the I/O and "talk" threads plus the main render/event loop.
7. **PTY / child spawn** (`kitty/child.py`) — a pseudo‑terminal is opened and the shell is forked (this is the boundary into Q3).

Under headless Xvfb with `--debug-rendering`, three of these stages announce themselves with **timestamped log lines**, in the fixed order **GL context → OS window → child**, plus one benign non‑fatal message. The remaining stages emit no banner of their own, so they are observed **directly from live process state** (§Q1.4, "Direct process & thread evidence"): the native launcher as the ELF process with the bundled Python interpreter (`libpython3.14.so.1.0`) mapped in‑process, and the `Boss`/`child‑monitor` layer as the named `KittyChildMon` I/O thread.

### Q1.2 Mechanism (cause → effect, with `file:line`)

| Order | Subsystem | Function / struct | `file:line` | Observable effect |
|------|-----------|-------------------|-------------|-------------------|
| 1 | Native launcher | process entry; `CLIOptions`; single‑instance IPC | `kitty/launcher/main.c`; `kitty/launcher/launcher.h:12` `typedef struct CLIOptions`; `kitty/launcher/single-instance.c:284` `single_instance_main` | process starts, Python embedded |
| 2 | Python dispatcher | `from kitty.main import main as kitty_main` → `kitty_main()` | `kitty/entry_points.py:49-50` | Python side begins |
| 3 | Bootstrap → GLFW select | `init_glfw_module(...)`; `glfw_module = ... 'x11'` | `kitty/main.py:90`, `:96` | X11 backend selected (non‑macOS, non‑Wayland) |
| 3 | Option push to native | `AppRunner.__call__` → `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` | `kitty/main.py:247`, `:249`; native sink `kitty/state.c:739` sets `global_state.debug_rendering` | debug flags cross into C |
| 4 | OS window + shaders | `_run_app` → `create_os_window(..., load_all_shaders, ...)` | `kitty/main.py:202`, `:221` (call opens), `:225` (`load_all_shaders` passed), def `:82` | GLFW X11 window + shader programs |
| 4 | OpenGL context | `gladLoadGL`; per‑call error hook; ARB gate; version gate & print | `kitty/gl.c:55`, `:62` `gladSetGLPostCallback(check_for_gl_error)`, `:63-67` `ARB_TEST(texture_storage)`, `:72` version print, `:73` version gate; required minimum **3.1** on Linux/X11 at `kitty/data-types.h:20` (major `3`) + `:24` (minor `1`), macOS `3.3` at `:22` (`#ifdef __APPLE__`) | GL context validated & version logged |
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

**Direct process & thread evidence (the otherwise‑silent subsystems).** The native launcher, the embedded Python interpreter, the `Boss` controller, and the `child‑monitor` threads do **not** emit their own stderr banners in the default run, so — rather than leaving them to inference — they are observed directly from live process state while a `sleep`‑ing child keeps the process alive. **Exact commands:**

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 kitty/launcher/kitty sh -c 'sleep 9' &
sleep 3 ; GUI=$(pgrep -f 'launcher/kitty' | head -1)
file kitty/launcher/kitty
cat /proc/$GUI/comm
grep -oE 'libpython[0-9.]+[a-z]*\.so[^ ]*' /proc/$GUI/maps | sort -u
for t in /proc/$GUI/task/*/comm; do cat "$t"; done | sort | uniq -c
```

**Complete, unedited output** (one representative run; the `<pid>` differs per launch, and only the `llvmpipe-*` worker count tracks the CPU count — everything else is identical across two runs):

```
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=2256b865ccba8a061a80a015bdb5abecd4c5f059, for GNU/Linux 3.2.0, not stripped
$ cat /proc/97414/comm
kitty
$ grep -oE 'libpython[0-9.]+[a-z]*\.so[^ ]*' /proc/97414/maps | sort -u
libpython3.14.so.1.0
$ for t in /proc/97414/task/*/comm; do cat "$t"; done | sort | uniq -c
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
```

This is the direct observation that:

- **Native launcher (stage 1)** — the process image is the compiled C launcher (`file` → *ELF … executable*; `/proc/<pid>/comm` → `kitty`), and it has **embedded and initialised the bundled Python interpreter in‑process**: `libpython3.14.so.1.0` is mapped into the launcher's own address space (`/proc/<pid>/maps`).
- **Python dispatcher / bootstrap (stages 2–3)** — that mapped `libpython3.14.so.1.0` is the interpreter that runs `kitty/entry_points.py:49-50` → `kitty.main.main()`; its execution is independently visible as the Python `print(... 'Child launched' ...)` at `kitty/window.py:871`.
- **`Boss` (stage 5) + `child‑monitor` threads (stage 6)** — the named thread **`KittyChildMon`** (the I/O thread; `set_thread_name("KittyChildMon")` at `kitty/child-monitor.c:1489`, created by `start()` at `kitty/child-monitor.c:291`) is present, which can only exist after `Boss.start` constructed the `ChildMonitor` and called its `start()`. Also visible: the disk‑cache thread `kitty:disk$0` and the 32 Mesa `llvmpipe-0…31` software‑GL worker threads (proof the `llvmpipe` renderer is live). The **talk** thread `KittyPeerMon` (`talk_loop`; `set_thread_name("KittyPeerMon")` at `kitty/child-monitor.c:1808`) is **absent** — `start()` creates it only when `self->talk_fd > -1 || self->listen_fd > -1` (`kitty/child-monitor.c:285`), i.e. only with remote‑control listening enabled, which the default launch does not use. *(This corrects the naive expectation that both I/O and talk threads always run: by default only `KittyChildMon` exists.)*

### Q1.5 Rationale

The ordered timestamps are the direct, observable proof of the coming‑online sequence:

- `[0.119] GL version string: '4.5 (Core Profile) …'` is emitted by `kitty/gl.c:72` **only** when `global_state.debug_rendering` is set — proving the **OpenGL context** was created and validated (llvmpipe reports 4.5, far exceeding the Linux/X11 minimum of **3.1** gated at `kitty/gl.c:73` against `kitty/data-types.h:20` major `3` + `:24` minor `1`; the `3.3` at `:22` is the macOS‑only `#ifdef __APPLE__` branch). Because `--debug-rendering` also installs a post‑call GL error hook (`kitty/gl.c:62`), the **absence** of any GL‑error line means every GL call during startup succeeded.
- `[0.141] OS Window created` is emitted by `kitty/glfw.c:1321`, proving the **GLFW/X11 window** subsystem finished creating the OS window; it is immediately preceded in source by `w->is_damaged = true` (`:1320`), the arm that guarantees a first paint.
- `[0.151] Failed to open systemd user bus …` comes from `kitty/systemd.c:87` (`log_error(...)`). It is **non‑fatal** — expected inside a container with no systemd user session — and Kitty proceeds regardless. *(Labelled here as a benign, environment‑specific line, not a failure.)*
- `[0.154] Child launched` is emitted by `kitty/window.py:871` after the PTY child is forked (`kitty/child.py:276`), proving the **child/PTY** subsystem is online and handing off to Q3.

The launcher (stage 1), Python dispatcher (stage 2), the option push (stage 3), `Boss` (stage 5) and the `child‑monitor` threads (stage 6) do **not** print their own stderr banners in the default `--debug-rendering` run — but they are **not left to inference**: they are directly observed in live process state (the "Direct process & thread evidence" capture above) as the ELF launcher process carrying `libpython3.14.so.1.0` in its address space (stages 1–2) and the named `KittyChildMon` I/O thread (stages 5–6). The control‑flow relationship corroborates the ordering: the GL line cannot appear unless `main.py` reached `create_os_window` (stages 2–4), and the `Child launched` line cannot appear unless `Boss.start` and the child‑monitor main loop (stages 5–6) executed `add_child` (`kitty/child-monitor.c:305`) to spawn the PTY child. *(The only genuinely **[inferred]** element is the exact internal micro‑ordering of stages 2–4 that precedes the first log line; every subsystem's presence is directly observed — via its own log line, or via its live process image / named thread.)*


---

## Q2 — Initial Configuration: how Kitty decides its startup configuration

### Q2.1 Direct answer

On first launch with no user `kitty.conf`, Kitty's effective `Options` come **entirely from its built‑in defaults**. The resolution has three layers, applied in precedence order (lowest to highest):

1. **Built‑in defaults** — the schema in `kitty/options/definition.py`, materialised into the `Options` dataclass in `kitty/options/types.py`.
2. **User `kitty.conf`** — discovered via config‑directory resolution (`kitty/constants.py`). **On this system it does not exist**, so it contributes nothing.
3. **Command‑line overrides** (`-o key=value`) — none were passed for the default‑launch captures.

Because layers 2 and 3 are empty, the first window is shaped purely by the defaults: a **black** background (`#000000`), **light‑gray** text (`#dddddd`), the **`monospace`** family (which fontconfig resolves to **DejaVuSansMono**) at **11.0 pt**, **zero** window padding, and `TERM=xterm-kitty`. The proof that these settings were applied is the **`debug_config` dump**, triggered through Kitty's real keybinding and read back from the system clipboard; its **"Config options different from defaults:" section is empty**, which is the direct statement that the running configuration equals the defaults.

### Q2.2 Mechanism (cause → effect, with `file:line`)

- **Defaults.** `opt('term', 'xterm-kitty', ...)` at `kitty/options/definition.py:3242`; materialised as `term: str = 'xterm-kitty'` at `kitty/options/types.py:602`. Other relevant defaults: `font_family monospace` (`kitty/options/definition.py:35` / `kitty/options/types.py:523`), `font_size 11.0` (`kitty/options/definition.py:59` / `kitty/options/types.py:524`), `window_padding_width 0` (`kitty/options/definition.py:1067` / `kitty/options/types.py:628`), `background #000000` (`kitty/options/definition.py:1464` / `kitty/options/types.py:480` `Color(0,0,0)`), `foreground #dddddd` (`kitty/options/definition.py:1459` / `kitty/options/types.py:526` `Color(221,221,221)`). The parser chain that would fold a `kitty.conf` on top lives in `kitty/options/parse.py`, and resolved options are pushed to the native layer through `kitty/options/to-c.h` / `kitty/options/to-c-generated.h`.
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

- **Visible confirmation (screenshot).** The default window attributes above are directly visible in an after-first-paint capture of the default window (same `import -window <wid>` method, embedded as a `data:` URI): a pure-black canvas (`background #000000`) carrying light-gray (`foreground #dddddd`) monospace glyphs that begin flush at the top-left cell (`window_padding_width 0`). `identify` reports `640x400 Gray colors=157`. This is the same after-first-paint frame used for the state transition in Q3.4:

![Q2 default window: light-gray monospace text on a black canvas with zero padding](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAoAAAAGQCAAAAAAbmI75AAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAACYktHRAD/h4/MvwAAB+NJREFUeNrt3HuQ1WUZB/BnuRQoZGgJKSq1zaSWFyZApEVlYCRIkjQxxxFTU/OS5DVzzEs44AUSlNG8ESIoKQqIio5jKyreaLm6i4g/A0EFCY9xEUTb7Y89e87v7B6XVVNZ/Xz+YN59z/N73pfzvjNn5uwXIgAAAD5XM8rrz8xPkvsLB8V0mF6VzGjaEsOTbnXDwclZ3nLyWhWZ6xqz6w2K+f2+Yxa9+7HXe6fq7dpBZfnZ9QYNjDm0XWbayNTELQfuVLPukRERkw6OiM0/Sr10ffddYu2M0Q70y3ABm2a3TTd9gqeeeqrJpZcNenrOoJNen5yf+WDOmzV9Tlp7W0TmtogP0sUHvvbo1v6nrZvgRJudGeUTKisnR0SMmvvKvNs7RETMrvvkzQ7ajl+4rOLagqeSJEmSGRExrvLiF5YtOjtXM+zVh6rufHzJ+FTx8KRb/GrB/W3jlCRJzoqIsiRJkiQZmx/0XTYiIkYu659/qvypiM6Lp9TbbpdXb46Y9Hyxv0mvV29znM1Nq4jvrLu+e7/zR8fY/jPn/mDIiDOKVI0se6ii39GrbsjPHBnju3bNtjjirmcPW5+vKVmwus/MxT/rkEl3GHpB5XERd9wxeHRExDOl2U/e3OCJpGdEHPTaY/lHOv0zYtXqzoU76fbbeDkiOlRVr5k8vt4uO5Wsd6DN8AJuPTVz+8J9IvrMuTBiz27Fqg5++byY/NxPbyjeYsq4mJeuub1/n4d3PLJH6i5F2W8qTmx8I7NPGTRz4J535ie6fG3DNT+/ceMu6aKTLyl5f+LYiOSNlbsefuHamYUtTs7c6kCb4QXMZCI27xD9dzwsiYgPi1XtNC8i1nQq3qJ6cmFNzfItkamO9uma01sv38ZGxgz5xcyjNo3LT7SMlps3bG5ZUDRnZIdDfvnapLg8ImbfdFThBZy414ilDrQZXsDqiIiSKIm7//SRZTW5PxramilaU5Kueezbx8yf3uhGNi84MA5YkPrYTra2P/2KOP69dNHSpTHqxSGTIiLiiY3fKmgw6YBR9zjPZqdFbvTopn3zd6p14eA/nSOiY6bRTo3WTPrDuxd0LJipbllv8MA3/tqh4I6u3iui825vRESUHXNA/mJn93ZYu/Xpl6bsf/WdjrP5SX0N8+SA8bPbdG1/fESs7H7Gitdfyg+eH/CXin67/r3RTo3XrLrhilEnpCcye/dfuW5NajBzWN+V09IV5Sf+bc6gVg9GRJz2k7sXRnScuOiNtr12fixi1uI3d+5XMyP/Uty339R2p8ea6U60+V7Ac0b1Lvvw309HRIy7aljrWWfnBxfv1G/QhuljGu20jZop3QZfdG35nhHnnfdO94iYMvTGlg8NSw/mfPfFgif+vPOhZZkJqa8B12/tu2N15t5LI0oGtHn/rVvvTRXv0/q4iFjkAvKJjava15vwldNqe9lIj/17L6pyHnxRHl/2eDfvAgAAfBWkYs+fc1x5/vji859qG6n0Nc1Ai+hauqruh1xcuRGV4z7zPTVlG3xJFHwP+DHiyp+l7WQbfB7yYYRcXDlmPDn1pfoZ6VplSdJmQJKMjYU3RkTE/f9Ih6V7Dq73q4y6sHSuT65z6QNLqmqzA7kl6oobbqPz1CVVE9Kf18VWz60VUZu+jlvmv7L4ESfcfC7gHaXnZ0ed3+p9V8/zI8Ye8eSl0w4akS95prR0y6zS0mHxZpeIiNh9eYwse+KKRUefExFx5ujj6nVvdcRdx05Yn+pT1/nyve8dtUf7KFgiW9xwG8N/eN91u7dLtS26evbxiBh6yZKjN8cf+8664Ka1Tng7V/RXcet/F1cPaTQj/a8e8etjL/5gl8omhaXzfeo6/7ji8shcFwVLZIsbbqP73Mvi7bHbWj33eDZ9vduHU+fFzU64WV7AiNjaaEZ6weG9Dv3ekRtjVjosPbRBWW1YOtUn27mszYqIaVcVvFSXrG6wjV5tV0Q8fM22Vs89nk1fz+o9ccXKB30GN8cLWBOxjYz0fef2+f7z+2fermpKWDrVJ9u5Ov4bEdUFL+WS1fW30cTVc49n09ePLD12nx69qx91xtu1FpHKP9eTzkjn1AaYM6v32+GWvfZ4vSAInc4tN97n2S2dInq1/Ygl6hdv7hIxsE1qpujqOXXp62TECWd+vbcj3r61ilT+ub5URjp/+LUB5uU9q559r8sLBUHobDi5KX0qug965tySj1iivrk9r1x2QnV6ptjqebXp6xHfrHhnYI1/ptQMLmA29pyKK2elMtI52QBz5SELY+nu5dGksHTDPlded82WpRuKvtRwG1eOHlLzXMf0f4SwjdWndBt80bUbD+nTInPPREfM/0PnpcO9CV9G200iuhFDepZvOnXrNIflAn5Be+w1sGbNmHkOCwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJqh/wGgxSIdbxgDxAAAAABJRU5ErkJggg==)


---

## Q3 — Terminal↔Shell Communication: making the terminal ready, and proving the shell's bytes were understood

### Q3.1 Direct answer

Kitty prepares the channel to the shell by (1) opening a **pseudo‑terminal** (`os.openpty()`), (2) putting the master into UTF‑8 mode, (3) building the child's **environment handshake** — `TERM=xterm-kitty`, `COLORTERM=truecolor`, `KITTY_PID`, `KITTY_PUBLIC_KEY`, `KITTY_WINDOW_ID`, `KITTY_INSTALLATION_DIR`, `TERMINFO=<path to Kitty's bundled xterm‑kitty database>`, plus shell‑integration variables — and (4) **forking** the shell attached to the PTY slave. When the shell then writes its first bytes, they flow through Kitty's **VT parser** (`kitty/vt-parser.c`) which decodes them into concrete screen operations (`screen_draw_text`, `screen_carriage_return`, `screen_linefeed`) that **mutate the screen model** (`kitty/screen.c`). The concrete proof: the child's raw bytes `68 65 6c 6c 6f 0d 0a` (`hello\r\n`) parse into exactly `draw hello` → `screen_carriage_return` → `screen_linefeed`; and the screen goes from **empty (all‑black) before** the shell writes to **populated (glyphs drawn) after**.

### Q3.2 Mechanism (cause → effect, with `file:line`)

- **PTY setup.** `def openpty()` at `kitty/child.py:170` → `master, slave = os.openpty()` at `:171` → `fast_data_types.set_iutf8_fd(master, True)` at `:174` (UTF‑8 on the master). The shell is created by `def fork(self)` at `kitty/child.py:276` (native spawn in `kitty/child.c`). **On this Linux/Xvfb run the child is forked directly, with no login‑shell prefix and no run‑shell wrap.** The run‑shell kitten wrap — `argv = [kitten_exe(), 'run-shell', '--shell', …]` at `kitty/child.py:314` — is guarded by `if self.should_run_via_run_shell_kitten:` (`kitty/child.py:295`), and `should_run_via_run_shell_kitten = is_macos and self.is_default_shell` (`kitty/child.py:230`) is therefore **`False` on Linux**. The login‑shell `argv[0]` hyphen‑prefix is applied only *inside* that kitten, at `tools/tui/run.go:170` — `shell_cmd[0] = "-" + filepath.Base(shell_cmd[0])` — inside `if runtime.GOOS == "darwin"` (`tools/tui/run.go:167`), i.e. **macOS/Darwin‑only and not exercised here**. (The comment block at `kitty/child.py:296‑309` explains the rationale for that macOS behavior; it is documentation, not the implementation.)
- **Environment handshake** (`kitty/child.py`): `env['TERM'] = opts.term` (`:242`), `env['COLORTERM'] = 'truecolor'` (`:243`), `env['KITTY_PID'] = getpid()` (`:244`), `env['KITTY_PUBLIC_KEY'] = boss.encryption_public_key` (`:245`); `TERMINFO` is set from `checked_terminfo_dir()` when `terminfo_type == 'path'` (the default per `kitty/options/types.py`) at `kitty/child.py:255‑258` (`if opts.terminfo_type == 'path'` → `tdir = checked_terminfo_dir()` → `env['TERMINFO'] = tdir`), else the base64 `direct` blob at `kitty/child.py:259‑260` (`elif opts.terminfo_type == 'direct'` → `env['TERMINFO'] = base64_terminfo_data()`); `env['KITTY_INSTALLATION_DIR'] = kitty_base_dir` (`:261`); (note `:262` is the unrelated `if opts.forward_stdio:` guard, not `TERMINFO`); and shell‑integration variables are injected by `modify_shell_environ(opts, env, self.argv)` at `kitty/child.py:265-267` (guarded by `'disabled' not in opts.shell_integration`).
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

Child environment handshake (the real child writes its own environment to a file, which is then filtered on the host), run twice to expose per‑launch variability:

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty sh -c 'env | sort > child_env.txt; sleep 2'
grep -E '^(TERM|COLORTERM|TERMINFO|KITTY_)' child_env.txt   # -> the handshake vars shown in Q3.4
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

**Child environment handshake** — the real child writes its own environment with `env | sort > child_env.txt`, which is then filtered to the handshake variables on the host with `grep -E '^(TERM|COLORTERM|TERMINFO|KITTY_)' child_env.txt`. Below is the **complete, unedited** output of that `grep` for **run 1**. `KITTY_PUBLIC_KEY` is Kitty's **public** encryption key — a per‑launch ephemeral value that is safe to display (see the *Secret handling* note in §Notes); it and `KITTY_PID` are the only two variables that change between launches:

```
COLORTERM=truecolor
KITTY_INSTALLATION_DIR=/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
KITTY_PID=101933
KITTY_PUBLIC_KEY=1:G;!_?+qIC+mo_k@Rfr5ML3=Ztot^C!L-PDSVOAeC
KITTY_WINDOW_ID=1
TERM=xterm-kitty
TERMINFO=/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9/terminfo
```

**Per‑launch variability — run 2 of the identical command.** Re‑running produces byte‑identical *static* variables (`COLORTERM`, `KITTY_INSTALLATION_DIR`, `KITTY_WINDOW_ID`, `TERM`, `TERMINFO`); only `KITTY_PID` (the GUI process id, set at `kitty/child.py:244`) and the ephemeral `KITTY_PUBLIC_KEY` (`:245`) differ between the two runs:

```
KITTY_PID=102013
KITTY_PUBLIC_KEY=1:Km|*tkvW%L&`w8c7g=K(VZ*2k%o>QjY)}qoPABLY
```

A host‑side `diff` of the two captures with those two per‑launch lines excluded (`diff <(grep -vE '^(KITTY_PID|KITTY_PUBLIC_KEY)=' run1) <(grep -vE '^(KITTY_PID|KITTY_PUBLIC_KEY)=' run2)`) prints **nothing** and the script reports `STATIC VARS IDENTICAL ACROSS RUNS` — i.e. the `TERM`/`COLORTERM`/`TERMINFO` contract is **stable across runs**, while the two identity/security values are freshly generated each launch.

**Shell‑integration** — parsed commands from a default interactive `bash` (`q3.si.commands`):

```
handle_remote_print aWdub3JlYm90aCBvciBpZ25vcmVzcGFjZSBwcmVzZW50IGluIGJhc2ggSElTVENPTlRST0wgc2V0dGluZywgc2hvd2luZyBydW5uaW5nIGNvbW1hbmQgd2lsbCBub3QgYmUgcm9idXN0Cg==}
ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
process_cwd_notification 7 kitty-shell-cwd://reverse-code-generator-710e56c0-8hqwk/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
screen_set_mode 2004 1
screen_delete_characters 59
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
set_title root@reverse-code-generator-710e56c0-8hqwk: /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
set_icon root@reverse-code-generator-710e56c0-8hqwk: /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
draw root@reverse-code-generator-710e56c0-8hqwk:/tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9# 
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/blitzy-623856dd-ce2f-4dda-9254-43ab8bcf20d2_929ec9
shell_prompt_marking 133 k;end_suffix_kitty
```

…and the **exact raw bytes** that produced them (`q3.si.bytes`, **719 bytes**, byte‑identical across two runs) — showing the actual OSC/CSI/DCS escape sequences. The capture opens with a `DCS @kitty-print … ST` diagnostic carrying the base64 of *“ignoreboth or ignorespace present in bash HISTCONTROL setting…”* — emitted by Kitty's own bash integration at `shell-integration/bash/kitty.bash:230-231` because the container's `/root/.bashrc:10` sets `HISTCONTROL=ignoredups:ignorespace` — and then the handshake proper (`OSC 7`, `CSI ?2004h`, the `OSC 133` prompt marks, `OSC 0/2` title/icon) interleaved with the real default `bash` prompt draw (`root@…:/…# `, from `/root/.bashrc`'s `PS1`):

```
00000000  1b 50 40 6b 69 74 74 79  2d 70 72 69 6e 74 7c 61  |.P@kitty-print|a|
00000010  57 64 75 62 33 4a 6c 59  6d 39 30 61 43 42 76 63  |Wdub3JlYm90aCBvc|
00000020  69 42 70 5a 32 35 76 63  6d 56 7a 63 47 46 6a 5a  |iBpZ25vcmVzcGFjZ|
00000030  53 42 77 63 6d 56 7a 5a  57 35 30 49 47 6c 75 49  |SBwcmVzZW50IGluI|
00000040  47 4a 68 63 32 67 67 53  45 6c 54 56 45 4e 50 54  |GJhc2ggSElTVENPT|
00000050  6c 52 53 54 30 77 67 63  32 56 30 64 47 6c 75 5a  |lRST0wgc2V0dGluZ|
00000060  79 77 67 63 32 68 76 64  32 6c 75 5a 79 42 79 64  |ywgc2hvd2luZyByd|
00000070  57 35 75 61 57 35 6e 49  47 4e 76 62 57 31 68 62  |W5uaW5nIGNvbW1hb|
00000080  6d 51 67 64 32 6c 73 62  43 42 75 62 33 51 67 59  |mQgd2lsbCBub3QgY|
00000090  6d 55 67 63 6d 39 69 64  58 4e 30 43 67 3d 3d 7d  |mUgcm9idXN0Cg==}|
000000a0  1b 5c 1b 5d 37 3b 6b 69  74 74 79 2d 73 68 65 6c  |.\.]7;kitty-shel|
000000b0  6c 2d 63 77 64 3a 2f 2f  72 65 76 65 72 73 65 2d  |l-cwd://reverse-|
000000c0  63 6f 64 65 2d 67 65 6e  65 72 61 74 6f 72 2d 37  |code-generator-7|
000000d0  31 30 65 35 36 63 30 2d  38 68 71 77 6b 2f 74 6d  |10e56c0-8hqwk/tm|
000000e0  70 2f 62 6c 69 74 7a 79  2f 6b 69 74 74 79 2f 62  |p/blitzy/kitty/b|
000000f0  6c 69 74 7a 79 2d 36 32  33 38 35 36 64 64 2d 63  |litzy-623856dd-c|
00000100  65 32 66 2d 34 64 64 61  2d 39 32 35 34 2d 34 33  |e2f-4dda-9254-43|
00000110  61 62 38 62 63 66 32 30  64 32 5f 39 32 39 65 63  |ab8bcf20d2_929ec|
00000120  39 07 1b 5b 3f 32 30 30  34 68 1b 5b 35 39 50 1b  |9..[?2004h.[59P.|
00000130  5d 31 33 33 3b 6b 3b 73  74 61 72 74 5f 6b 69 74  |]133;k;start_kit|
00000140  74 79 07 1b 5d 31 33 33  3b 44 3b 30 07 1b 5d 31  |ty..]133;D;0..]1|
00000150  33 33 3b 41 07 1b 5d 31  33 33 3b 6b 3b 65 6e 64  |33;A..]133;k;end|
00000160  5f 6b 69 74 74 79 07 1b  5d 30 3b 72 6f 6f 74 40  |_kitty..]0;root@|
00000170  72 65 76 65 72 73 65 2d  63 6f 64 65 2d 67 65 6e  |reverse-code-gen|
00000180  65 72 61 74 6f 72 2d 37  31 30 65 35 36 63 30 2d  |erator-710e56c0-|
00000190  38 68 71 77 6b 3a 20 2f  74 6d 70 2f 62 6c 69 74  |8hqwk: /tmp/blit|
000001a0  7a 79 2f 6b 69 74 74 79  2f 62 6c 69 74 7a 79 2d  |zy/kitty/blitzy-|
000001b0  36 32 33 38 35 36 64 64  2d 63 65 32 66 2d 34 64  |623856dd-ce2f-4d|
000001c0  64 61 2d 39 32 35 34 2d  34 33 61 62 38 62 63 66  |da-9254-43ab8bcf|
000001d0  32 30 64 32 5f 39 32 39  65 63 39 07 72 6f 6f 74  |20d2_929ec9.root|
000001e0  40 72 65 76 65 72 73 65  2d 63 6f 64 65 2d 67 65  |@reverse-code-ge|
000001f0  6e 65 72 61 74 6f 72 2d  37 31 30 65 35 36 63 30  |nerator-710e56c0|
00000200  2d 38 68 71 77 6b 3a 2f  74 6d 70 2f 62 6c 69 74  |-8hqwk:/tmp/blit|
00000210  7a 79 2f 6b 69 74 74 79  2f 62 6c 69 74 7a 79 2d  |zy/kitty/blitzy-|
00000220  36 32 33 38 35 36 64 64  2d 63 65 32 66 2d 34 64  |623856dd-ce2f-4d|
00000230  64 61 2d 39 32 35 34 2d  34 33 61 62 38 62 63 66  |da-9254-43ab8bcf|
00000240  32 30 64 32 5f 39 32 39  65 63 39 23 20 1b 5d 31  |20d2_929ec9# .]1|
00000250  33 33 3b 6b 3b 73 74 61  72 74 5f 73 75 66 66 69  |33;k;start_suffi|
00000260  78 5f 6b 69 74 74 79 07  1b 5b 35 20 71 1b 5d 32  |x_kitty..[5 q.]2|
00000270  3b 2f 74 6d 70 2f 62 6c  69 74 7a 79 2f 6b 69 74  |;/tmp/blitzy/kit|
00000280  74 79 2f 62 6c 69 74 7a  79 2d 36 32 33 38 35 36  |ty/blitzy-623856|
00000290  64 64 2d 63 65 32 66 2d  34 64 64 61 2d 39 32 35  |dd-ce2f-4dda-925|
000002a0  34 2d 34 33 61 62 38 62  63 66 32 30 64 32 5f 39  |4-43ab8bcf20d2_9|
000002b0  32 39 65 63 39 07 1b 5d  31 33 33 3b 6b 3b 65 6e  |29ec9..]133;k;en|
000002c0  64 5f 73 75 66 66 69 78  5f 6b 69 74 74 79 07     |d_suffix_kitty.|
000002cf
```

**Before / after the shell writes.** The state transition was captured with `import -window <wid>` under Xvfb, gated to **after first paint**. While the child was still sleeping (no output yet) the window is a completely empty frame; after `printf` ran, the same window carries two rows of glyphs. Both PNGs are embedded below as `data:` URIs (each is a 640x400 grayscale PNG of only a few hundred bytes to ~2 KiB, so the document stays self-contained).

**Before** - child still sleeping, no bytes written. `identify` reports `640x400 Gray colors=1`; a per-row luminance projection (PIL) finds **0 non-black pixels** - the screen model has nothing drawn yet:

![Q3 before: empty all-black terminal before the shell writes](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAoAAAAGQAQAAAAAWiOyIAAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAACYktHRAAB3YoTpAAAADZJREFUeNrtwTEBAAAAwqD1T20ND6AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAeDR+kAABdezbqAAAAABJRU5ErkJggg==)

**After** - the child ran `printf "hello from kitty 0.35.2\n"; printf "line two: rendering works\n"`. `identify` reports `640x400 Gray colors=157`; the per-row projection finds two text bands at **rows 5-18** (`hello from kitty 0.35.2`) and **rows 23-36** (`line two: rendering works`) - two cell rows populated with anti-aliased light-gray monospace glyphs:

![Q3 after: two lines of light-gray monospace text on black](data:image/png;base64,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)

The jump from **1 unique color (blank)** to **157 unique colors (anti-aliased glyphs)** is the observable empty->populated transition of the screen model as `screen_draw_text` (`kitty/screen.c:866`) mutates it.

### Q3.5 Rationale

- **Byte‑exact understanding.** The child emitted exactly `68 65 6c 6c 6f 0d 0a`. `printf "hello\n"` writes six bytes (`hello\n`); the PTY line discipline's `ONLCR` turns the `\n` into `\r\n`, so the master reads seven bytes — this is why `0d 0a` (`\r\n`) appears. The parser turned those seven bytes into precisely three commands: the five text bytes into `draw hello` (→ `screen_draw_text`, `kitty/screen.c:866`, invoked from `kitty/vt-parser.c:226/236`), the `0d` into `screen_carriage_return`, and the `0a` into `screen_linefeed`. The one‑to‑one correspondence between the exact bytes and the exact screen operations is the proof the data was **understood correctly**; that both files are byte/line‑identical across two runs confirms stability.
- **The handshake is real.** The child's own environment shows `TERM=xterm-kitty` (from `kitty/child.py:242`), `COLORTERM=truecolor` (`:243`), `KITTY_PID` (`:244`), `KITTY_PUBLIC_KEY` (`:245`), `KITTY_WINDOW_ID`, `KITTY_INSTALLATION_DIR`, and `TERMINFO` pointing at Kitty's bundled database (`kitty/child.py:255‑258`, the `terminfo_type == 'path'` branch). This is the concrete terminfo contract: the shell is told it is on an `xterm-kitty` terminal and where to find that terminfo.
- **Shell‑integration injection is observable.** With the default `bash`, the first bytes are a `DCS @kitty-print … ST` diagnostic (Kitty's bash integration warning about `HISTCONTROL=ignoredups:ignorespace`, emitted at `shell-integration/bash/kitty.bash:230-231` → parsed as `handle_remote_print`), immediately followed by the handshake proper: `OSC 7` cwd reporting (`ESC ] 7 ; kitty-shell-cwd://… BEL` → `process_cwd_notification`), `CSI ? 2004 h` (bracketed‑paste enable → `screen_set_mode 2004 1`), a `CSI 59 P` line clear (`screen_delete_characters 59`), the `OSC 133` prompt markings (`k;start_kitty`, `D;0`, `A`, `end_kitty`, …), the real `bash` prompt itself (`OSC 0` title+icon then the raw `draw root@…:/…# ` from `/root/.bashrc`'s `PS1`), then `k;start_suffix_kitty`, `CSI 5 SP q` cursor‑style (`screen_set_cursor 5 32`), `OSC 2` window‑title (`set_title …`), and `k;end_suffix_kitty`. Each raw escape maps one‑to‑one to a parsed command, proving Kitty both **injected** the integration (via `modify_shell_environ`, `kitty/child.py:265-267`) and **understood** the sequences the integrated shell emitted. (The `HISTCONTROL` warning and the `PS1` prompt draw are the shell's own behaviour driven by the container's default `/root/.bashrc`, not Kitty's injection — they appear because this is the genuine default‑user launch.)
- **Empty → populated.** The all‑black "before" capture versus the glyph‑bearing "after" capture is the state transition the screen model undergoes as `screen_draw_text` mutates it: nothing is drawn until the child produces bytes, and the first bytes are what populate row 0.


---

## Q4 — Display System Evidence: fonts, layout, scrolling, screen updates

This answer addresses each of the four named items explicitly.

### Q4.1 Direct answer

The display subsystem is demonstrably live: **fonts** are discovered and rasterized (primary `DejaVuSansMono`, with live fallback to `Noto Sans CJK JP` for CJK code points); **layout** places glyphs into a fixed grid of cells (observed 22 rows × 71 columns for the default 640×400 window) with zero padding; **scrolling** occurs once output exceeds the window height, pushing lines into the scrollback history; and **screen updates** are driven by an OpenGL render/damage cycle (`render_os_window → draw_cells → swap_window_buffers`) whose 10 shader programs, built from 13 GLSL sources, run without a single GL error under `--debug-rendering`. The visible result — light‑gray monospace glyphs (including double‑width CJK) on black — is captured in screenshots taken after first paint.

Because Q4 names four items — **fonts, layout, scrolling, and screen updates** — each is answered below as its own **five-part mini-answer** in the mandated order: **Direct answer → Mechanism (`file:line`) → Exact command → Complete, unedited output → Rationale**. (Q4.5 additionally splits its output into *Evidence A*, the `--debug-rendering` context/error check, and *Evidence B*, the per-frame instrumentation log; Q4.4 splits its output into the post-processing transcript and the complete raw artifact.) §Q4.6 gathers the embedded screenshots that corroborate all four items.

### Q4.2 Fonts

**Direct answer.** The primary monospace face resolves to `DejaVuSansMono` (Normal/Bold/Italic/Bold-Italic); code points it lacks — such as CJK — trigger **live fallback** to `Noto Sans CJK JP`, and both facts are printed verbatim by Kitty's own `--debug-font-fallback` dump.

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

**Direct answer.** The default 640×400 window is laid out as a fixed grid of **22 rows × 71 columns** with zero padding, and that grid size is exactly what Kitty hands the child as its PTY window size.

**Mechanism.** Glyphs are placed into a fixed cell grid; the window/render objects are `class Window` at `kitty/window.py:523` and `class Borders` at `kitty/borders.py:68`, and the model is `kitty/screen.c`. The default initial window is `initial_window_width 640` / `initial_window_height 400` (`kitty/options/definition.py:994`, `:998`).

**Exact command** — the capture mechanism is explicit: the child (running on the PTY slave) writes its PTY window size into a file with `stty size` and `tput`, and the host then reads that file back with an unfiltered `cat`, so the shown output is the complete file contents rather than text trapped inside the on‑screen window. Run twice:

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty sh -c 'stty size > /tmp/q4.layout.txt; \
    printf "COLS=%s LINES=%s\n" "$(tput cols)" "$(tput lines)" >> /tmp/q4.layout.txt; sleep 1'
cat /tmp/q4.layout.txt
```

**Complete, unedited output** (the `cat` of the child‑written file; byte‑identical on both runs — a `diff` of the two prints nothing and reports `LAYOUT IDENTICAL ACROSS RUNS`):

```
22 71
COLS=71 LINES=22
```

**Rationale.** The default 640×400 window resolves to a grid of **22 rows × 71 columns** at the default 11.0 pt `DejaVuSansMono`. This is the layout the display system computed and handed to the child as its terminal size; the two‑line screenshots in Q3/Q4 show text laid out left‑aligned from the top row with zero inset, consistent with `window_padding_width 0`.

### Q4.4 Scrolling

**Direct answer.** When the child prints more lines than the 22-row window holds, each linefeed at the bottom margin scrolls the viewport up and pushes the displaced top line into the scrollback history; driving 200 lines produces exactly 200 `draw` + 200 `screen_carriage_return` + 200 `screen_linefeed` parsed commands.

**Mechanism.** `screen_linefeed(Screen *self)` at `kitty/screen.c:1643` calls `screen_index(self)` at `kitty/screen.c:1570`; when the cursor is on the bottom margin, `screen_index` invokes the `INDEX_UP(add_to_history)` macro (`kitty/screen.c:1552`), whose body calls `historybuf_add_line(self->historybuf, ...)` — pushing the scrolled‑off top line into the **scrollback history** (`kitty/history.c`, with `kitty/line-buf.c`). `add_to_history` is true only on the main screen with no top margin.

**Exact command** — drive far more lines than the 22‑row window, dumping every parsed command to a file, **run twice** (`q4.scroll.commands.run1`, `q4.scroll.commands.run2`):

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --dump-commands sh -c 'seq 1 200; sleep 2' > q4.scroll.commands
```

**Post‑processing commands** (run on the captured artifact so the histogram and head/tail are fully reproducible, and the two runs are compared with `diff`):

```
wc -l q4.scroll.commands
awk '{print $1}' q4.scroll.commands | sort | uniq -c
head -6 q4.scroll.commands
tail -6 q4.scroll.commands
diff q4.scroll.commands.run1 q4.scroll.commands.run2 && echo "SCROLL IDENTICAL ACROSS RUNS"
```

**Post‑processing output** (shell transcript; `$` lines are the commands above, everything else is their verbatim output):

```
$ wc -l q4.scroll.commands
600 q4.scroll.commands
$ awk '{print $1}' q4.scroll.commands | sort | uniq -c
    200 draw
    200 screen_carriage_return
    200 screen_linefeed
$ head -6 q4.scroll.commands
draw 1
screen_carriage_return
screen_linefeed
draw 2
screen_carriage_return
screen_linefeed
$ tail -6 q4.scroll.commands
draw 199
screen_carriage_return
screen_linefeed
draw 200
screen_carriage_return
screen_linefeed
$ diff q4.scroll.commands.run1 q4.scroll.commands.run2 && echo "SCROLL IDENTICAL ACROSS RUNS"
SCROLL IDENTICAL ACROSS RUNS
```

**Complete, unedited output** — the entire **600‑line** `q4.scroll.commands` from run 1 (run 2 is byte‑identical, as the `diff` above confirms). This is the full raw artifact, not a histogram or tail:

```
draw 1
screen_carriage_return
screen_linefeed
draw 2
screen_carriage_return
screen_linefeed
draw 3
screen_carriage_return
screen_linefeed
draw 4
screen_carriage_return
screen_linefeed
draw 5
screen_carriage_return
screen_linefeed
draw 6
screen_carriage_return
screen_linefeed
draw 7
screen_carriage_return
screen_linefeed
draw 8
screen_carriage_return
screen_linefeed
draw 9
screen_carriage_return
screen_linefeed
draw 10
screen_carriage_return
screen_linefeed
draw 11
screen_carriage_return
screen_linefeed
draw 12
screen_carriage_return
screen_linefeed
draw 13
screen_carriage_return
screen_linefeed
draw 14
screen_carriage_return
screen_linefeed
draw 15
screen_carriage_return
screen_linefeed
draw 16
screen_carriage_return
screen_linefeed
draw 17
screen_carriage_return
screen_linefeed
draw 18
screen_carriage_return
screen_linefeed
draw 19
screen_carriage_return
screen_linefeed
draw 20
screen_carriage_return
screen_linefeed
draw 21
screen_carriage_return
screen_linefeed
draw 22
screen_carriage_return
screen_linefeed
draw 23
screen_carriage_return
screen_linefeed
draw 24
screen_carriage_return
screen_linefeed
draw 25
screen_carriage_return
screen_linefeed
draw 26
screen_carriage_return
screen_linefeed
draw 27
screen_carriage_return
screen_linefeed
draw 28
screen_carriage_return
screen_linefeed
draw 29
screen_carriage_return
screen_linefeed
draw 30
screen_carriage_return
screen_linefeed
draw 31
screen_carriage_return
screen_linefeed
draw 32
screen_carriage_return
screen_linefeed
draw 33
screen_carriage_return
screen_linefeed
draw 34
screen_carriage_return
screen_linefeed
draw 35
screen_carriage_return
screen_linefeed
draw 36
screen_carriage_return
screen_linefeed
draw 37
screen_carriage_return
screen_linefeed
draw 38
screen_carriage_return
screen_linefeed
draw 39
screen_carriage_return
screen_linefeed
draw 40
screen_carriage_return
screen_linefeed
draw 41
screen_carriage_return
screen_linefeed
draw 42
screen_carriage_return
screen_linefeed
draw 43
screen_carriage_return
screen_linefeed
draw 44
screen_carriage_return
screen_linefeed
draw 45
screen_carriage_return
screen_linefeed
draw 46
screen_carriage_return
screen_linefeed
draw 47
screen_carriage_return
screen_linefeed
draw 48
screen_carriage_return
screen_linefeed
draw 49
screen_carriage_return
screen_linefeed
draw 50
screen_carriage_return
screen_linefeed
draw 51
screen_carriage_return
screen_linefeed
draw 52
screen_carriage_return
screen_linefeed
draw 53
screen_carriage_return
screen_linefeed
draw 54
screen_carriage_return
screen_linefeed
draw 55
screen_carriage_return
screen_linefeed
draw 56
screen_carriage_return
screen_linefeed
draw 57
screen_carriage_return
screen_linefeed
draw 58
screen_carriage_return
screen_linefeed
draw 59
screen_carriage_return
screen_linefeed
draw 60
screen_carriage_return
screen_linefeed
draw 61
screen_carriage_return
screen_linefeed
draw 62
screen_carriage_return
screen_linefeed
draw 63
screen_carriage_return
screen_linefeed
draw 64
screen_carriage_return
screen_linefeed
draw 65
screen_carriage_return
screen_linefeed
draw 66
screen_carriage_return
screen_linefeed
draw 67
screen_carriage_return
screen_linefeed
draw 68
screen_carriage_return
screen_linefeed
draw 69
screen_carriage_return
screen_linefeed
draw 70
screen_carriage_return
screen_linefeed
draw 71
screen_carriage_return
screen_linefeed
draw 72
screen_carriage_return
screen_linefeed
draw 73
screen_carriage_return
screen_linefeed
draw 74
screen_carriage_return
screen_linefeed
draw 75
screen_carriage_return
screen_linefeed
draw 76
screen_carriage_return
screen_linefeed
draw 77
screen_carriage_return
screen_linefeed
draw 78
screen_carriage_return
screen_linefeed
draw 79
screen_carriage_return
screen_linefeed
draw 80
screen_carriage_return
screen_linefeed
draw 81
screen_carriage_return
screen_linefeed
draw 82
screen_carriage_return
screen_linefeed
draw 83
screen_carriage_return
screen_linefeed
draw 84
screen_carriage_return
screen_linefeed
draw 85
screen_carriage_return
screen_linefeed
draw 86
screen_carriage_return
screen_linefeed
draw 87
screen_carriage_return
screen_linefeed
draw 88
screen_carriage_return
screen_linefeed
draw 89
screen_carriage_return
screen_linefeed
draw 90
screen_carriage_return
screen_linefeed
draw 91
screen_carriage_return
screen_linefeed
draw 92
screen_carriage_return
screen_linefeed
draw 93
screen_carriage_return
screen_linefeed
draw 94
screen_carriage_return
screen_linefeed
draw 95
screen_carriage_return
screen_linefeed
draw 96
screen_carriage_return
screen_linefeed
draw 97
screen_carriage_return
screen_linefeed
draw 98
screen_carriage_return
screen_linefeed
draw 99
screen_carriage_return
screen_linefeed
draw 100
screen_carriage_return
screen_linefeed
draw 101
screen_carriage_return
screen_linefeed
draw 102
screen_carriage_return
screen_linefeed
draw 103
screen_carriage_return
screen_linefeed
draw 104
screen_carriage_return
screen_linefeed
draw 105
screen_carriage_return
screen_linefeed
draw 106
screen_carriage_return
screen_linefeed
draw 107
screen_carriage_return
screen_linefeed
draw 108
screen_carriage_return
screen_linefeed
draw 109
screen_carriage_return
screen_linefeed
draw 110
screen_carriage_return
screen_linefeed
draw 111
screen_carriage_return
screen_linefeed
draw 112
screen_carriage_return
screen_linefeed
draw 113
screen_carriage_return
screen_linefeed
draw 114
screen_carriage_return
screen_linefeed
draw 115
screen_carriage_return
screen_linefeed
draw 116
screen_carriage_return
screen_linefeed
draw 117
screen_carriage_return
screen_linefeed
draw 118
screen_carriage_return
screen_linefeed
draw 119
screen_carriage_return
screen_linefeed
draw 120
screen_carriage_return
screen_linefeed
draw 121
screen_carriage_return
screen_linefeed
draw 122
screen_carriage_return
screen_linefeed
draw 123
screen_carriage_return
screen_linefeed
draw 124
screen_carriage_return
screen_linefeed
draw 125
screen_carriage_return
screen_linefeed
draw 126
screen_carriage_return
screen_linefeed
draw 127
screen_carriage_return
screen_linefeed
draw 128
screen_carriage_return
screen_linefeed
draw 129
screen_carriage_return
screen_linefeed
draw 130
screen_carriage_return
screen_linefeed
draw 131
screen_carriage_return
screen_linefeed
draw 132
screen_carriage_return
screen_linefeed
draw 133
screen_carriage_return
screen_linefeed
draw 134
screen_carriage_return
screen_linefeed
draw 135
screen_carriage_return
screen_linefeed
draw 136
screen_carriage_return
screen_linefeed
draw 137
screen_carriage_return
screen_linefeed
draw 138
screen_carriage_return
screen_linefeed
draw 139
screen_carriage_return
screen_linefeed
draw 140
screen_carriage_return
screen_linefeed
draw 141
screen_carriage_return
screen_linefeed
draw 142
screen_carriage_return
screen_linefeed
draw 143
screen_carriage_return
screen_linefeed
draw 144
screen_carriage_return
screen_linefeed
draw 145
screen_carriage_return
screen_linefeed
draw 146
screen_carriage_return
screen_linefeed
draw 147
screen_carriage_return
screen_linefeed
draw 148
screen_carriage_return
screen_linefeed
draw 149
screen_carriage_return
screen_linefeed
draw 150
screen_carriage_return
screen_linefeed
draw 151
screen_carriage_return
screen_linefeed
draw 152
screen_carriage_return
screen_linefeed
draw 153
screen_carriage_return
screen_linefeed
draw 154
screen_carriage_return
screen_linefeed
draw 155
screen_carriage_return
screen_linefeed
draw 156
screen_carriage_return
screen_linefeed
draw 157
screen_carriage_return
screen_linefeed
draw 158
screen_carriage_return
screen_linefeed
draw 159
screen_carriage_return
screen_linefeed
draw 160
screen_carriage_return
screen_linefeed
draw 161
screen_carriage_return
screen_linefeed
draw 162
screen_carriage_return
screen_linefeed
draw 163
screen_carriage_return
screen_linefeed
draw 164
screen_carriage_return
screen_linefeed
draw 165
screen_carriage_return
screen_linefeed
draw 166
screen_carriage_return
screen_linefeed
draw 167
screen_carriage_return
screen_linefeed
draw 168
screen_carriage_return
screen_linefeed
draw 169
screen_carriage_return
screen_linefeed
draw 170
screen_carriage_return
screen_linefeed
draw 171
screen_carriage_return
screen_linefeed
draw 172
screen_carriage_return
screen_linefeed
draw 173
screen_carriage_return
screen_linefeed
draw 174
screen_carriage_return
screen_linefeed
draw 175
screen_carriage_return
screen_linefeed
draw 176
screen_carriage_return
screen_linefeed
draw 177
screen_carriage_return
screen_linefeed
draw 178
screen_carriage_return
screen_linefeed
draw 179
screen_carriage_return
screen_linefeed
draw 180
screen_carriage_return
screen_linefeed
draw 181
screen_carriage_return
screen_linefeed
draw 182
screen_carriage_return
screen_linefeed
draw 183
screen_carriage_return
screen_linefeed
draw 184
screen_carriage_return
screen_linefeed
draw 185
screen_carriage_return
screen_linefeed
draw 186
screen_carriage_return
screen_linefeed
draw 187
screen_carriage_return
screen_linefeed
draw 188
screen_carriage_return
screen_linefeed
draw 189
screen_carriage_return
screen_linefeed
draw 190
screen_carriage_return
screen_linefeed
draw 191
screen_carriage_return
screen_linefeed
draw 192
screen_carriage_return
screen_linefeed
draw 193
screen_carriage_return
screen_linefeed
draw 194
screen_carriage_return
screen_linefeed
draw 195
screen_carriage_return
screen_linefeed
draw 196
screen_carriage_return
screen_linefeed
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

**Rationale.** 200 printed lines produced exactly 200 `draw` + 200 `screen_carriage_return` + 200 `screen_linefeed` commands (600 total — the complete artifact above). The window holds only 22 rows, so from the 23rd line onward each `screen_linefeed` (`kitty/screen.c:1643`) drives `screen_index` (`:1570`) into `INDEX_UP` (`:1552`), scrolling the viewport and appending the displaced top line to the scrollback via `historybuf_add_line`. The visible end state is the last rows (…`draw 200`) on screen with the earlier numbers scrolled into history — the concrete scrolling behavior. **Stability:** the run was executed **twice** with identical `seq 1 200` input; the two 600‑line captures are byte‑identical (`diff` prints nothing → `SCROLL IDENTICAL ACROSS RUNS`), so the 200/200/200 distribution is stable, not a one‑shot observation.

### Q4.5 Screen updates (render / damage cycle)

**Direct answer.** Screen updates are driven by an OpenGL render/damage cycle (`render_os_window → draw_cells → swap_window_buffers`) that runs per frame with **zero GL errors**; an instrumentation build shows the `render()` entry point is reached on every event-loop cycle (17 cycles in the sample run, of which 5 are driven by child writes).

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

**Rationale.** `--debug-rendering` sets `global_state.debug_rendering` (`kitty/state.c:739`), which (a) prints the GL version once (`kitty/gl.c:72`, shown above → the render context is a real 4.5 core profile on llvmpipe) and (b) installs a post‑call GL error hook (`gladSetGLPostCallback(check_for_gl_error)`, `kitty/gl.c:62`) that would print on **any** failed GL call. Across the whole run **no GL‑error line appears**, so every draw in the `render_os_window → draw_cells → swap_window_buffers` cycle (`kitty/child-monitor.c:833/795/802/810`) succeeded. That the frames were actually produced is confirmed by the screenshots (blank → glyphs), and the per‑character screen mutations that feed each frame are exactly the `draw`/`screen_linefeed` commands captured in Q3/Q4.4. *(Note: in normal operation `--debug-rendering` does not emit a per‑frame log line; the "No render frame received…" line at `kitty/child-monitor.c:822` only appears on a stall, which did not occur here. A richer per‑frame event log is available via a debug build — `./dev.sh build --debug`, `docs/build.rst:54` — and is captured verbatim as **Evidence B** below.)*

**Per‑frame render cycle — instrumentation build (Evidence B).** `--debug-rendering` (Evidence A) proves the context is real and error‑free but, in normal operation, does not print a line per frame. To surface the render cycle itself, the binary was rebuilt **once** as an instrumentation build that only *adds logging* — it does not change the render logic:

```
./dev.sh build --debug --extra-logging=event-loop --ignore-compiler-warnings
kitty/launcher/kitty --version        # -> kitty 0.35.2  (same version; only logging added)
```

`--extra-logging=event-loop` defines `-DDEBUG_EVENT_LOOP` (`setup.py:488-489`), which activates the `EVDBG(...)` macro (`kitty/child-monitor.c:29‑30` → `#define EVDBG(...) timed_debug_print(__VA_ARGS__)`). The render entry point logs on **every** cycle at `kitty/child-monitor.c:872` (`EVDBG("input_read: %d, check_for_active_animated_images: %d", ...)` — the first statement of `render()`), and the event‑loop driver logs at `:1225` (`Processing global state`) and `:1217` (`State check timer fired`). The **exact run** (a child that writes five lines spaced 0.4 s apart, forcing repeated input‑driven frames), run twice:

```
env DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
  kitty/launcher/kitty --debug-rendering \
  sh -c 'for i in 1 2 3 4 5; do printf "frame line %s\n" "$i"; sleep 0.4; done; sleep 1' \
  > q4.evdbg.out 2> q4.evdbg.err
```

**Complete, unedited output** (`q4.evdbg.err`, run 1 — 89 lines; the GL‑version line went to stdout, as in Evidence A). `timed_debug_print` emits each `[timestamp] message` *without* a trailing newline, so successive messages visibly concatenate (e.g. `State check timer firedProcessing global stateinput_read: 1,…`) — this is the raw, unedited behaviour, not a formatting artefact of this document:

```
[0.140] OS Window created
[0.150] Failed to open systemd user bus with error: Connection refused
[0.154] Child launched
[0.154] starting handleEvents(0.00)
[0.154] pollForEvents final timeout: 0.000
[0.154] State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 1[0.167] display_read_ok: 0
[0.167] other dispatch done
[0.167] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 1, check_for_active_animated_images: 0[0.170] starting handleEvents(0.00)
[0.170] pollForEvents final timeout: 0.000
[0.170] display_read_ok: 0
[0.170] other dispatch done
[0.170] --------- loop tick, wakeups_happened: 0 ----------
[0.170] starting handleEvents(-0.00)
[0.170] pollForEvents final timeout: 0.487
[0.559] display_read_ok: 0
[0.559] other dispatch done
[0.559] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.559] starting handleEvents(-0.00)
[0.559] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[0.564] display_read_ok: 0
[0.564] other dispatch done
[0.564] --------- loop tick, wakeups_happened: 0 ----------
[0.564] starting handleEvents(-0.00)
[0.564] pollForEvents final timeout: 0.092
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0[0.659] display_read_ok: 0
[0.659] other dispatch done
[0.659] --------- loop tick, wakeups_happened: 0 ----------
[0.659] starting handleEvents(-0.00)
[0.659] pollForEvents final timeout: 0.498
[0.961] display_read_ok: 0
[0.961] other dispatch done
[0.961] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.961] starting handleEvents(-0.00)
[0.961] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[0.967] display_read_ok: 0
[0.967] other dispatch done
[0.967] --------- loop tick, wakeups_happened: 0 ----------
[0.967] starting handleEvents(-0.00)
[0.968] pollForEvents final timeout: 0.190
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0[1.159] display_read_ok: 0
[1.159] other dispatch done
[1.159] --------- loop tick, wakeups_happened: 0 ----------
[1.159] starting handleEvents(-0.00)
[1.159] pollForEvents final timeout: 0.497
[1.364] display_read_ok: 0
[1.364] other dispatch done
[1.364] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[1.364] starting handleEvents(-0.00)
[1.364] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[1.369] display_read_ok: 0
[1.370] other dispatch done
[1.370] --------- loop tick, wakeups_happened: 0 ----------
[1.370] starting handleEvents(-0.00)
[1.370] pollForEvents final timeout: 0.287
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0[1.659] display_read_ok: 0
[1.659] other dispatch done
[1.659] --------- loop tick, wakeups_happened: 0 ----------
[1.659] starting handleEvents(-0.00)
[1.659] pollForEvents final timeout: 0.497
[1.767] display_read_ok: 0
[1.767] other dispatch done
[1.767] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[1.767] starting handleEvents(-0.00)
[1.767] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[1.772] display_read_ok: 0
[1.772] other dispatch done
[1.772] --------- loop tick, wakeups_happened: 0 ----------
[1.772] starting handleEvents(-0.00)
[1.772] pollForEvents final timeout: 0.384
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0[2.159] display_read_ok: 0
[2.159] other dispatch done
[2.159] --------- loop tick, wakeups_happened: 0 ----------
[2.159] starting handleEvents(-0.00)
[2.159] pollForEvents final timeout: 0.497
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0[2.659] display_read_ok: 0
[2.659] other dispatch done
[2.659] --------- loop tick, wakeups_happened: 0 ----------
[2.659] starting handleEvents(-0.00)
[2.659] pollForEvents final timeout: 0.497
State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 0[3.159] display_read_ok: 0
[3.159] other dispatch done
[3.159] --------- loop tick, wakeups_happened: 0 ----------
[3.159] starting handleEvents(-0.00)
[3.159] pollForEvents final timeout: 0.497
[3.172] display_read_ok: 0
[3.172] other dispatch done
[3.172] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[3.180] main loop exiting
```

**Stability (run 2 of the identical command).** Both runs are **89 lines** with an identical message‑kind distribution: **17** `Processing global state` (= 17 `render()` cycles), of which exactly **5** report `input_read: 1` — one per child write (`frame line 1`…`5`) — and **12** report `input_read: 0` (idle repaint‑delay cycles); the event loop performs **17** `loop tick` iterations (6 with `wakeups_happened: 1`, 11 with `0`) and **11** `State check timer fired`. Timestamps differ between runs; the cycle structure and counts do not.

**Rationale (Evidence B).** Each `Processing global state` → `input_read` pair is one entry into `render()` (`kitty/child-monitor.c:871‑872`), the function that drives `render_os_window → draw_cells → swap_window_buffers`. The five `input_read: 1` cycles are the five child writes being turned into repaints; the `input_read: 0` cycles are idle repaint‑delay ticks. Combined with Evidence A (a real 4.5 core context and **zero** GL errors) and the painted screenshots (Q3.4 blank→glyphs, Q4.6 fallback), this is direct, per‑frame confirmation that the screen‑update cycle is live — not inferred. This is strictly an instrumentation build: `-DDEBUG_EVENT_LOOP` only adds `timed_debug_print` calls; the render code path is the production path, and the **canonical binary is rebuilt and restored immediately after this capture** (see Notes).

### Q4.6 Visible evidence (screenshots)

Screenshots were captured with `import -window <wid>` under the Xvfb display, **after** the `--debug-rendering` log confirmed the context was up (first-paint gating), and are **embedded below** as `data:` URIs so the document stays self-contained. Each is a 640x400 grayscale PNG; a per-row luminance projection (PIL) is quoted to ground the description in pixel data. The empty->populated **before/after** pair is embedded in the state-transition evidence of Q3.4 above.

**Unicode / font-fallback** - so the embedded screenshot corresponds exactly to the `--debug-font-fallback` dump above, the child ran the **same command** as Q4.2: `printf "AaBb 123 → ✓ 你好\n"`. `identify` reports `640x400 Gray colors=154`; the per-row projection finds a single populated text band at **rows 3-17**. On that row `AaBb 123`, the arrow → (U+2192) and the check ✓ (U+2713) render in the primary `DejaVuSansMono` face (they are in its coverage, so the dump above shows no fallback for them), while the CJK pair 你 (U+4F60) / 好 (U+597D) render as double-width cells via the `Noto Sans CJK JP` fallback the dump names:

![Q4 unicode/fallback: AaBb 123, arrow, check and CJK 你好 rendered under Xvfb](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAoAAAAGQCAAAAAAbmI75AAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAACYktHRAD/h4/MvwAABTNJREFUeNrt2WtslmcZB/B/KwXKiLMYGTHdBmHYrcNhDKkZWeYyJJss4CZ4mAmH4QEjcRJRpHPDBLNRdCJqFtPBwqGbY27KsTsEhWUBMwrTjIOltC+g6whLVotkwQEB/VDXlgIufph+eH+/T891Pfd9X8lzXXmeN3kTAKCY1P4q3/lN78Q3NnZflns8RaPf/6vwxL25sbR34oaSZPmtSZLdX9AYA5hk06Dx71XduUPqc03XG7D8H0mSqwrJ/R9IsuCGen0pKptbV5wXNxUKrX9qqEw2/b7v0qnP/rGwIUl+ur25+aV5Sep3Hmjeft9/XXNbQ+YVujRXJFlaKBQKreOT4U++OlVLikdpMnzkK9efnzy8aMmWmgcvtnzwya1/T5J87NATK898bWZyZseKFcfv+eq7Vxr7xV7BL67qyOSmidMyZ2L9W51Jlt57dtqMkkLywOglz2hLUVm457a2u5LMe3F/664fJ2lqTPLitmTTtvUt+1b1Wd604Z2rcW3Luy6Gt/3yXat8bu/inmBGy5+X1bbemarCuCzdnCSZsyuTmpM8tklHikm/ZOzhF968fV3yodanO26+69hPkqR85tBXklzZ8eCYyXULLrF3WMmJJMnYr+dAn1vTK352fqL6u821PdGUI+dy7uVjFUkm/G1nkmRkR0aeqEreV1KVtGhM8Qxg5cj1ab0+yYIka18em6TqYGnJvnuTvD23PdfddKm9szofTTLrvpJTa/qMWwbdU7LsvMQPT327V9TYXJsl2fDWoqS2eU6SpPL1DO/cUJbk2eR7vsLFM4DT+jdm1413NOaW2SMGlQ44muRIQ/9Rn37kS0lne3LsyktsXXP1Qy1JdiyuuHnqoce705uGJUnZ7P1bulO3Hr/t2vntvbbWpzbJ8IYku27pSn24MR9pvz15bOgkXSmqAawpXZWU3NGYH5T/9uDpB0qSnFqdDJ4wel/++R92Pj7m4SeTpKUlDzd9vmcAd1QkyafebO5ZuuTINRsbLzhgVv+1lyUNn5ldn6Rq6KTXrn5eP4pvACtG/eF3yd2jU1H51I8y8v3H/n3jdOnQZEhle4Ydv+jGtdV1T3QHJWU9N+qSpK7jy73eeEc/frD2whMmt7VXJfv+UlOfZOD2/gtPLdeP4hvAr5RvXZ1UT5mw5UR1xeVLBiTJgBn9Rnzyja3JwGUbx4zq/XvsijvTr3z226vz9EefGTw7b6y/Ys2e18vHDXmhz7ELKnt/cdef/fmFlT9b/VCS5NC1SfLqrIrnP7h4roYU3QB+4uTKJM9NmbTl0ekvnd1/NElGLDx38vDyJK+Vff/M9t4vr1Hzk8vnH1+d68ruTrJn/YnT4y871/nr+/ue23v+snLlRSoP3L0qSbK7oiuuK1s5/a9LdYT/ieeWJRk/s1DzTuKRA9/M3Mqamqc2eDi89zYvTbKupfu/vpt2fitJ1rXtXeThAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUOz+BVGxQEeY1Tx2AAAAAElFTkSuQmCC)

The before/after frames in Q3.4 (blank -> glyphs) and this fallback frame together are the visible confirmation that font discovery, rasterization, cell layout, and the render/damage cycle are all live. *(The PNGs are embedded here as evidence; the transient capture scripts and the raw `.png`/dump files under `/tmp` were removed after encoding, leaving the repository unchanged apart from this document - see the Notes section.)*

---

## Coverage Checklist

Every distinct thing each question asks — including each "e.g./such as/including/like" item — with its concrete value, `file:line`, and observed evidence.

| Q | Item | Concrete value | `file:line` | Observed evidence |
|---|------|----------------|-------------|-------------------|
| Q1 | Native launcher | C entry + `CLIOptions` + single‑instance | `kitty/launcher/main.c`; `kitty/launcher/launcher.h:12`; `kitty/launcher/single-instance.c:284` | ELF process `/proc/<pid>/comm=kitty`; `libpython3.14.so.1.0` mapped |
| Q1 | Python dispatcher | `main()` handoff | `kitty/entry_points.py:49-50` | `libpython3.14.so.1.0` in maps; `Child launched` (a Python print) |
| Q1 | Bootstrap → GLFW/X11 | backend `x11` | `kitty/main.py:96` | `OS Window created` |
| Q1 | OpenGL context + shaders | `4.5 (Core Profile)` llvmpipe; **3.1** required on Linux/X11 (macOS 3.3) | `kitty/gl.c:72,73`; `kitty/data-types.h:20,24` (Linux) / `:22` (macOS); `kitty/main.py:82,221,225` | stdout GL line; no GL errors |
| Q1 | Boss controller | `Boss` + `boss.start` | `kitty/main.py:226,227`; `kitty/boss.py` | `KittyChildMon` thread present (Boss started the child‑monitor); `Child launched` |
| Q1 | child‑monitor threads | `io_thread`=`KittyChildMon` (always); `talk_thread`=`KittyPeerMon` (only if listening) | `kitty/child-monitor.c:281,285,291,305,1489,1808`; `kitty/main.py:234` | `KittyChildMon` observed in `/proc/<pid>/task/*/comm`; `add_child` → `Child launched` |
| Q1 | PTY / child spawn | `openpty` + `fork` | `kitty/child.py:170,171,276` | `Child launched` (`kitty/window.py:871`) |
| Q1 | Offline→online transition | `is_damaged=true` then window created | `kitty/glfw.c:1320,1321` | ordered timestamps `0.119→0.141→0.154` |
| Q1 | Version banner | `kitty 0.35.2 created by Kovid Goyal` | `kitty/constants.py:25,26` | `--version` output |
| Q2 | Built‑in defaults precedence | defaults win (no conf/overrides) | `kitty/options/definition.py`; `kitty/options/types.py` | empty "different from defaults" |
| Q2 | Config‑dir resolution | `/root/.config/kitty`, `defconf exists=False` | `kitty/constants.py:87,131,133` | resolved‑constants output |
| Q2 | CLI overrides | none applied | `kitty/config.py:163`; `kitty/options/parse.py` | dump shows no overrides |
| Q2 | debug_config proof | full dump → clipboard (SGR‑stripped) | `kitty/boss.py:3060,3064,3065`; `kitty/debug_config.py:231,72`; keybinding `kitty/options/definition.py:4256` | 28‑line clipboard dump, ×2 identical |
| Q2 | Window‑attribute correlation | bg `#000000`, fg `#dddddd`, `monospace`/`DejaVuSansMono` 11.0, pad 0 | `kitty/options/definition.py:35,59,1067,1459,1464`; `kitty/options/types.py:480,523,524,526,628` | dump `Fonts:` + embedded screenshot (§Q2.5) |
| Q2 | Version `0.35.2` / VCS | `kitty 0.35.2 (815df1e210)` | `kitty/constants.py:25,26` | dump line 1 |
| Q2 | `term=xterm-kitty` | default term | `kitty/options/definition.py:3242`; `kitty/options/types.py:602` | child `TERM=xterm-kitty` (Q3) |
| Q3 | `openpty` | `os.openpty()` + iutf8 | `kitty/child.py:170,171,174` | `hello\r\n` bytes flow |
| Q3 | `fork` | shell forked on PTY | `kitty/child.py:276` | `Child launched`; child env captured |
| Q3 | `TERM` / terminfo | `xterm-kitty`; DB `terminfo/x/xterm-kitty` | `kitty/child.py:242`; `kitty/terminfo.py:27,500` | child env `TERM=xterm-kitty`, `TERMINFO=…` |
| Q3 | `KITTY_*`/`COLORTERM`/`TERMINFO` env | `COLORTERM=truecolor`, `KITTY_PID`, `KITTY_PUBLIC_KEY`, `KITTY_WINDOW_ID`, `KITTY_INSTALLATION_DIR`, `TERMINFO` | `kitty/child.py:242‑245,255‑261` | complete `child_env` capture (§Q3.4) |
| Q3 | Shell‑integration injection | OSC 7/133/2, bracketed paste, cursor style | `kitty/child.py:265‑267`; `shell-integration/{bash,zsh,fish,ssh}` | `q3.si.commands` + 719‑byte hexdump |
| Q3 | VT‑parser → screen mutation | `draw hello`/CR/LF | `kitty/vt-parser.c:226,236`; `kitty/screen.c:866` | parsed commands vs bytes `68 65 6c 6c 6f 0d 0a` |
| Q3 | before/after screen | blank (1 color) → glyphs (157 colors) | `kitty/screen.c:866` | embedded before/after screenshots + per‑row projections (§Q3.4) |
| Q4 | **Fonts** | `DejaVuSansMono` + `Noto Sans CJK JP` fallback | `kitty/fonts.c:457,465,492`; `kitty/fonts/render.py:163`; `kitty/fontconfig.c`; `kitty/freetype.c` | `--debug-font-fallback` dump + embedded screenshot (§Q4.6) |
| Q4 | **Layout** | 22 rows × 71 cols, pad 0, 640×400 | `kitty/window.py:523`; `kitty/borders.py:68`; `kitty/options/definition.py:994,998` | `stty size`=`22 71` via child‑written file + `cat` (§Q4.3) |
| Q4 | **Scrolling** | 200 lines → history scrollback | `kitty/screen.c:1643,1570,1552`; `kitty/history.c` | 200×`screen_linefeed`, complete output ×2 runs (§Q4.4) |
| Q4 | **Screen updates** | render→draw_cells→swap; 10 programs / 13 GLSL | `kitty/child-monitor.c:833,705,795,802,810`; `kitty/shaders.c:20`; `kitty/main.py:82` | per‑frame EVDBG render log (§Q4.5) + zero GL errors; embedded screenshot |

---

## Notes on labelled evidence

- **[non‑canonical] alternatives not used for canonical claims.** The GLFW **null**, **OSMesa**, and **EGL/ANGLE** headless backends (`glfw/null_init.c`, `glfw/osmesa_context.c`, `glfw/egl_context.c`) exist in‑repo and are compiled (see build log), but the **Xvfb + X11** path is the canonical one exercised here; the alternatives were not used to produce any observation above.
- **[inferred] items.** These subsystems are now backed by **direct observation**, not inference: in Q1 the native launcher, embedded Python interpreter, `Boss`, and `child‑monitor` threads do not emit their own stderr banners under the default `--debug-rendering` run, so they are observed from live process state instead — the ELF launcher process with `libpython3.14.so.1.0` mapped in‑process, and the named `KittyChildMon` I/O thread (§Q1.4, "Direct process & thread evidence"). The single remaining **[inferred]** element anywhere in Q1 is the exact internal micro‑ordering of the Python‑bootstrap → GLFW‑init → shader‑load steps *before* the first timestamped log line; every subsystem's *presence* is directly observed. All other claims across Q1–Q4 are backed by direct captured output.
- **Read‑only confirmation.** No existing repository file was modified, added to, or deleted. The only added tracked path is `blitzy/documentation/kitty_815df1e210e0.md`; the built `kitty/launcher/kitty` is git‑ignored (`.gitignore:18`). All temporary scripts, dump files (`q1.*`, `q3.bytes`, …), and raw `.png` captures were removed after encoding (the three evidence screenshots are embedded inline as base64 `data:` URIs), leaving the repository byte‑for‑byte unchanged apart from this document. `git status --porcelain` at completion shows only the added document under `blitzy/`.
- **Secret handling.** `KITTY_PUBLIC_KEY` (Q3) is an ephemeral, per‑launch **public** key that changes on every start; because it is a **public** key (not a secret), its real value is shown in Q3.4; it changes on every launch, and that per-run change is itself reported as evidence (§Q3.4). No private key or secret is reproduced here.

## Stability summary

| Capture | Runs | Result |
|---------|------|--------|
| Q1 startup order/log | 2 | Same order (GL → window → child); sub‑ms timestamp jitter only |
| `--version` banner | 2+ | `kitty 0.35.2 created by Kovid Goyal` (identical) |
| Q2 `debug_config` dump | 2 | 28 lines, byte‑identical (`diff` empty) |
| Q3 `hello` bytes & commands | 2 | Byte‑identical (`cmp`) and command‑identical (`diff`) |
| Q4 `--debug-font-fallback` | 2 | Identical apart from timestamps |
| Q4 scrolling counts | 2 (n=200) | 200 draw / 200 CR / 200 LF both runs; two 600‑line captures byte‑identical (`diff` empty) |

