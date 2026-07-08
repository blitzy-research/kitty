# kitty — Critical Early Startup Phase: A Runtime-Observed Trace

**Answer document for branch `kitty_815df1e210e0` (kitty 0.35.2, commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)**

---

## Preamble

### Objective

This document traces kitty's **critical early startup phase** — from native process launch up to (but **not** including) the moment terminal content is first displayed — and reports, **from actual runtime observation**, how the GPU rendering context is created, how the font system is set up, which rendering backend and display configuration kitty *actually* selects at runtime, what the terminal reports about its text-rendering capabilities, and how the window system, GPU initialization, and text-cell calculations interrelate and sequence.

It answers eight parts:

- **(a)** Canonical build + headless launch
- **(b)** Selected GLFW backend + GLX‑vs‑EGL context source
- **(c)** Observed GL strings + extension checks
- **(d)** Detected display configuration (content scale, logical DPI, window size)
- **(e)** Font setup + cell metrics (with a critical correction to the "two‑phase" premise)
- **(f)** Text‑rendering / terminal capabilities (exact response bytes over a PTY)
- **(g)** Reconstructed init‑order sequence + a table of key values
- **(h)** Consolidated cause → effect reasoning

### Read‑only constraint

This is a **read‑only** investigation. Per the user's verbatim instruction — *"Do not make any changes to the repository and leave the actual codebase unchanged."* — no tracked source file was modified, added, or deleted. The **only** file written into the repository is this document (`blitzy/documentation/kitty_815df1e210e0.md`). All build products under `kitty/` (`kitty/launcher/kitty`, `kitty/fast_data_types.so`, `*.o`) are git‑ignored build artifacts, not source changes. Temporary observation scripts were created only under `/tmp` and removed at the end; the final `git status --porcelain` shows only this document.

### Exact environment

| Item | Value |
|---|---|
| Canonical image | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas`) |
| Repository checkout | `/tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3` |
| Implementation branch | `blitzy-616b2c75-5ea1-415a-9c80-ac1890def288` (the working branch this answer document is committed on; current HEAD `29ed48c2b`) |
| Source / deliverable branch | `kitty_815df1e210e0` (names the kitty source snapshot under investigation; the deliverable file is `kitty_815df1e210e0.md`) |
| Source commit (investigated) | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (the kitty 0.35.2 snapshot; parent of the implementation-branch HEAD `29ed48c2b`) |
| kitty version | `0.35.2` (`kitty/constants.py:L25` — `version: Version = Version(0, 35, 2)`) |
| OS | Ubuntu 25.10 (questing) |
| Python | 3.13.7 (system) |
| Go | 1.24.4 |
| Compiler | gcc 15.2.0 |
| GPU | none — headless via **Xvfb** + **Mesa llvmpipe** software GL |
| Display | `DISPLAY=:99`, no `WAYLAND_DISPLAY` |

### How to read this document

Every factual claim carries four things:

1. the **exact command** that produced the evidence,
2. the **complete, unedited output** of that command,
3. a **`file:line`** citation into the source (anchored on the stable function/symbol name; line numbers re‑verified against the post‑build tree at this commit), and
4. the **cause → effect reasoning** explaining *why* the observed behavior occurs.

Values that could vary (GL strings, cell metrics, Device‑Attributes bytes) were captured across **at least two runs** and confirmed stable; the run count is stated. Where an observation **contradicts** a common assumption, the document **leads with the directly observed result** and then explains the mechanism. Any value that could not be obtained through kitty's real entry point is explicitly labeled.

> **Note on the real entry point.** All values below are obtained by launching the compiled native launcher `kitty/launcher/kitty` (which embeds CPython) → the kitty Python package → `kitty.main.main()`. Where kitty's own code does not surface a value (e.g. `GL_VENDOR`), the value is queried **in‑process, inside kitty's own live GL context**, at the exact point kitty itself uses it (the `prerender_function` callback, where the context is current) — not via an external bypass. Such in‑process probes are labeled where used, and are cross‑validated against kitty's own output.

---

## (a) Canonical build + headless launch

### Direct answer

kitty is built from source with **`python setup.py build`**. On this Ubuntu 25.10 image the *bare* canonical command **fails** at compile time (a dependency‑version drift in `wayland-protocols`, not a kitty defect); the official `--ignore-compiler-warnings` flag makes it succeed **without touching any source**. The build produces the launcher `kitty/launcher/kitty` and the C‑extension `kitty/fast_data_types.so` (both git‑ignored). Because the container has no GPU and no display, kitty is run headless under **Xvfb** with **Mesa llvmpipe** software GL.

### Build — Python/Go version gates and the bare‑build failure

The build entry is `setup.py`. The Python version is gated by `check_version_info()` (`setup.py:L30`), which reads `requires-python = ">=3.8"` from `pyproject.toml:L2`. A curiosity worth flagging: the failure message at `setup.py:L44` literally reads `f'calibre requires Python {minver}. Current Python version: {".".join(map(str, sys.version_info[:3]))}'` — a copy‑paste artifact from Kovid Goyal's other project; the gate still correctly enforces kitty's `>=3.8`. Go is pinned to `go 1.22` by `go.mod:L3` (satisfied by the installed Go 1.24.4).

**Command (bare canonical build — captured to document the honest failure):**

```bash
cd /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3
export PATH=/usr/lib/go-1.24/bin:$PATH
export CI=true
python3 setup.py build
```

**Complete, unedited output** (non‑verbose mode prints one `[N/122] Compiling <unit>` progress line per translation unit via `setup.py:L842`; the numbering is parallel‑dispatch assignment order. The parallel compile pool sends each child's stderr to `DEVNULL` (`setup.py:L844`), so the *only* visible copy of the diagnostic is the re‑run of the failing command by `run_tool` (`setup.py:L853` → `L664‑L665`). Because Python's stdout is block‑buffered under redirection while gcc's stderr is unbuffered, the gcc error text lands in the merged stream *before* the buffered ` done` / failing‑target / failing‑command lines, which flush at process exit — so the ordering below is real and complete, not truncated):

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
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
 done
Compiling [wayland] glfw/wl_window.c ...
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
```

Exit status: **1**. The final line is the complete `gcc` command (ending `-c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o`, shown in full above) that `setup.py` re‑ran to surface the error; the `-Werror` flag is what promotes the unhandled‑`switch` warning to a fatal error.

**Cause → effect.** The vendored GLFW fork's `xdgToplevelHandleConfigure()` switch in `glfw/wl_window.c:668` enumerates the `XDG_TOPLEVEL_STATE_*` values known at the time of this kitty 0.35.2 snapshot (`RESIZING`, `MAXIMIZED`, `FULLSCREEN`, `ACTIVATED`, the `TILED_*` set, and optionally `SUSPENDED`). The host's newer `wayland-protocols` (1.45) adds four `XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}` enumerators. kitty compiles with `-Werror` (and gcc's default `-Werror=switch`), so an `enum` `switch` that does not handle every enumerator is a **fatal** error. This is purely a build‑environment version drift; the kitty source is unchanged and correct for the protocol version it targets.

**Command (working canonical build — official flag, no source edits):**

```bash
cd /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3
export PATH=/usr/lib/go-1.24/bin:$PATH
export CI=true
python3 setup.py build --ignore-compiler-warnings
```

`--ignore-compiler-warnings` is an official `setup.py` option that sets the C `werror` flag to empty, so warnings (including the unhandled‑`switch` warning) no longer abort the build. **No source file is modified.** Exit status: **0**.

**Complete, unedited output** (no elision — every `[N/122]` compile line and every `[N/5]` link line, exactly as emitted):

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
```

The transcript ends at `[5/5] Linking launcher` and ` done`. The Go‑based `kitten` binary is also produced at this point, but it is invisible in non‑verbose mode for two reasons, both in `setup.py`: the `Updating Go generated files...` progress line and the `go build` command echo are emitted only under `--verbose` (`setup.py:L1107‑L1108`, `L1167‑L1168`), and `go build -v` itself prints nothing when the module/build cache is warm (no packages are rebuilt). The `kitten` binary's presence is verified in the artifact listing below.

**Command (verify artifacts exist, their ELF type, and that they are git‑ignored):**

```bash
ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
file kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
git check-ignore -v build kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
git status --porcelain
```

**Complete, unedited output:**

```
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x 1 root root  1253792 Jul  8 05:39 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 16429348 Jul  8 05:39 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  8 05:38 kitty/launcher/kitty

$ file kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
kitty/launcher/kitty:     ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4a693e4304285476522c8ac6a4eef4babf9072b7, for GNU/Linux 3.2.0, not stripped
kitty/launcher/kitten:    ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=95d5ca68f398e7ab24bc496a7f2645efa685fb4e, stripped
kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=2ef096713336548e66bb9f472c62ea4438f511b6, not stripped

$ git check-ignore -v build kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
.gitignore:14:/build/	build
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitty
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitten
.gitignore:1:*.so	kitty/fast_data_types.so

$ git status --porcelain
(empty output above => clean working tree; the build changed no tracked file)
```

The three build products (`kitty` launcher — 40 KB PIE executable; `kitten` — 16 MB stripped Go executable; `fast_data_types.so` — 1.2 MB shared object) exist; `git check-ignore -v` reports the exact `.gitignore` rule that ignores each (`.gitignore:1 *.so`, `.gitignore:14 /build/`, `.gitignore:18 /kitty/launcher/kitt*`); and `git status --porcelain` prints nothing — the build changed **no tracked file**.

### Resolved dependency versions (observed)

**Command:**

```bash
for lib in harfbuzz freetype2 fontconfig lcms2 libpng libcanberra libxxhash openssl zlib xkbcommon wayland-client x11 xcb; do
  printf "%-16s " "$lib:"; pkg-config --modversion "$lib" 2>/dev/null || echo "(not found)";
done
```

**Complete output:**

```
harfbuzz:        10.2.0
freetype2:       26.2.20
fontconfig:      2.15.0
lcms2:           2.16
libpng:          1.6.50
libcanberra:     0.30
libxxhash:       0.8.3
openssl:         3.5.3
zlib:            1.3.1
xkbcommon:       1.7.0
wayland-client:  1.24.0
x11:             1.8.12
xcb:             1.17.0
```

These satisfy the documented minimums in `docs/build.rst:L79-L120` (e.g. HarfBuzz `>= 2.2.0`; observed 10.2.0). `.github/workflows/ci.yml` does not invoke a bare `setup.py` for its main build: the Linux **Build kitty** step (`.github/workflows/ci.yml:L63`) runs `python .github/workflows/ci.py build`, whose `build_kitty()` (`.github/workflows/ci.py:L102-L104`) invokes `python setup.py build --verbose` (appending `--debug` only on macOS); a separate linux-package job's **Build kitty** step (`.github/workflows/ci.yml:L116`) runs `python setup.py build --debug` directly. Both resolve to the same canonical `setup.py build` entry point used here, differing only in the `--verbose`/`--debug` flags.

### Headless launch

**Command (start virtual X server + select software GL):**

```bash
nohup Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset >/tmp/xvfb.log 2>&1 &
export DISPLAY=:99
export LIBGL_ALWAYS_SOFTWARE=1   # override: force Mesa software rasterizer (labeled)
pgrep -a Xvfb
echo "DISPLAY=[$DISPLAY]"
echo "WAYLAND_DISPLAY=[$WAYLAND_DISPLAY]"
echo "LIBGL_ALWAYS_SOFTWARE=[$LIBGL_ALWAYS_SOFTWARE]"
```

**Complete, unedited output:**

```
$ pgrep -a Xvfb
82616 Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset
$ echo "DISPLAY=[$DISPLAY]"
DISPLAY=[:99]
$ echo "WAYLAND_DISPLAY=[$WAYLAND_DISPLAY]"
WAYLAND_DISPLAY=[]
$ echo "LIBGL_ALWAYS_SOFTWARE=[$LIBGL_ALWAYS_SOFTWARE]"
LIBGL_ALWAYS_SOFTWARE=[1]
```

`WAYLAND_DISPLAY` is empty (no Wayland socket), which — as shown in part (b) — forces the **x11** backend. `LIBGL_ALWAYS_SOFTWARE=1` is an explicit override guaranteeing the Mesa **llvmpipe** software rasterizer is used; it is labeled here because it is an environment override, not a kitty setting. (On this image the driver resolves to llvmpipe regardless, since there is no GPU.)

**Command (launch kitty through the real launcher, reach the GPU path, exit deterministically):**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'true'
```

**Complete, unedited output:**

```
[0.225] OS Window created
[0.235] Failed to open systemd user bus with error: Connection refused
[0.239] Child launched
[0.183] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

Notes, reported exactly as observed:
- The `GL version string` line shows a timestamp (`[0.183]`) *earlier* than the lines above it. This is a stream‑ordering artifact: the GL line is written to **stdout** via `printf` (`kitty/gl.c:L72`), while `OS Window created` / `Child launched` are written to **stderr** via `log_error`; the two streams interleave out of monotonic order when merged. The timestamps are the true event order (`gl_init` at `0.183` precedes window creation at `0.225`).
- `Failed to open systemd user bus with error: Connection refused` is benign in this container (no systemd user session); it does not affect the startup path being traced.

**Cause → effect.** Building from source (rather than installing a package) is required so we run *this exact commit's* code and can cite it line‑for‑line, and so the native launcher/extension are the ones under test. Xvfb + llvmpipe are required because kitty is a GPU‑accelerated terminal that **mandates** an OpenGL context (see part (c)); with no GPU/display, a virtual X server plus a software GL implementation is the canonical way to reach kitty's GPU‑initialization path. The `--debug-rendering` flag (`kitty/cli.py:L989`) turns on kitty's own GL logging, so the GL version is emitted by kitty itself rather than by an external tool.


---

## (b) Selected GLFW backend + GLX‑vs‑EGL context source

### Direct answer

- **Backend: `x11`.** With no Wayland socket present, kitty selects the `x11` GLFW backend. Confirmed by kitty's own reporting: `Running under: X11`.
- **Context source: GLX (native), not EGL.** kitty's X11 window creates its OpenGL context via **GLX** (`libGLX_mesa`), not EGL. Confirmed by the GL libraries actually mapped into the live kitty process (`libGLX_mesa.so` present, `libEGL` absent).

### Backend selection — mechanism and evidence

The backend string is chosen in one line, `init_glfw()` at `kitty/main.py:L96`:

```python
glfw_module = 'cocoa' if is_macos else ('wayland' if is_wayland(opts) else 'x11')
```

`is_wayland()` (`kitty/constants.py:L207`) returns `False` on non‑macOS when no Wayland session is detected; `detect_if_wayland_ok()` (`kitty/constants.py:L196`) checks for `WAYLAND_DISPLAY` / `WAYLAND_SOCKET` in the environment (`kitty/constants.py:L197`). Since neither is set here, `is_wayland()` is `False`, `is_macos` is `False`, and the selector evaluates to **`x11`**.

**Command (kitty's own backend report, in-process — the canonical `debug_config()` output):**

The observation attaches a **temporary** watcher module (removed afterward per the read-only constraint) through kitty's own `-o watcher=` hook. Its `on_resize` handler runs **in-process** in the live kitty (GL context current) and calls kitty's OWN `kitty.debug_config.debug_config(get_options())` — the exact function kitty's built-in `debug_config` action invokes at `kitty/boss.py:L3064`. It records both `get_os_window_size(...)` (used in part (d)) and the `debug_config()` text; this is kitty's own reporting path, not a bypass.

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
cat > /tmp/obs/obs_watcher.py <<'PY'
import json, os, re
from kitty.fast_data_types import (get_os_window_size, cell_size_for_window,
    opengl_version_string, current_fonts, get_options)
from kitty.debug_config import debug_config, compositor_name
_done = False
def on_resize(boss, window, data):
    global _done
    if _done:
        return
    _done = True
    osw = window.os_window_id
    opts = get_options()
    rec = {
        "tag": "on_resize", "os_window_id": osw,
        "get_os_window_size": dict(get_os_window_size(osw)),
        "cell_size_for_window": list(cell_size_for_window(osw)),
        "opengl_version_string": opengl_version_string(),
        "compositor_name": compositor_name(),
        "opts.font_size": opts.font_size,
        "opts.initial_window_width": list(opts.initial_window_width),
        "opts.initial_window_height": list(opts.initial_window_height),
        "current_fonts": {k: f.identify_for_debug()
                          for k, f in current_fonts().items()
                          if hasattr(f, "identify_for_debug")},
    }
    json.dump(rec, open(os.environ.get("OBS_OUT", "/tmp/obs/window_obs.json"), "w"), indent=2)
    dc = debug_config(opts)
    open("/tmp/obs/debug_config_stripped.txt", "w").write(re.sub(r"\x1b.+?m", "", dc))
PY
./kitty/launcher/kitty -o watcher=/tmp/obs/obs_watcher.py -o confirm_os_window_close=0 sh -c 'sleep 2'
cat /tmp/obs/debug_config_stripped.txt
```

**Complete, unedited output** (kitty's own `debug_config()`; the ANSI SGR color escapes are removed exactly as kitty itself does for its clipboard copy — `re.sub(r'\x1b.+?m', '', output)` at `kitty/boss.py:L3065`):

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux reverse-code-generator-a9970399-dpchg 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 25.10 reverse-code-generator-a9970399-dpchg /dev/tty

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
  kitty: /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3/kitty/launcher/kitty
  base dir: /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3
  extensions dir: /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3/kitty
  system shell: /bin/bash
Loaded config overrides:
  watcher /tmp/obs/obs_watcher.py

Config options different from defaults:
watcher:
{'/tmp/obs/obs_watcher.py': '/tmp/obs/obs_watcher.py'}

Important environment variables seen by the kitty process:
	PATH                                /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3/kitty/launcher:/usr/lib/go-1.24/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :99
	LC_CTYPE                            C.UTF-8
```

The backend line is **`Running under: X11`** — kitty's own runtime determination via `kitty.debug_config.compositor_name()`, so it reflects the backend actually selected, not an assumption. (The `watcher` entries under “Loaded config overrides” / “Config options different from defaults” are the temporary observation harness itself, removed afterward so the repository is left unchanged.)

### GLX vs EGL vs OSMESA — mechanism and evidence

On X11, the vendored GLFW fork selects the context API inside `_glfwCreateWindowX11()` in two matching `if / else if` chains keyed on `ctxconfig->source` — first for visual/context-API initialization, then for context creation — at `glfw/x11_window.c:L1876-L1925` (shown complete, verbatim, no elision):

```c
    if (ctxconfig->client != GLFW_NO_API)
    {
        if (ctxconfig->source == GLFW_NATIVE_CONTEXT_API)
        {
            if (!_glfwInitGLX())
                return false;
            if (!_glfwChooseVisualGLX(wndconfig, ctxconfig, fbconfig, &visual, &depth))
                return false;
        }
        else if (ctxconfig->source == GLFW_EGL_CONTEXT_API)
        {
            if (!_glfwInitEGL())
                return false;
            if (!_glfwChooseVisualEGL(wndconfig, ctxconfig, fbconfig, &visual, &depth))
                return false;
        }
        else if (ctxconfig->source == GLFW_OSMESA_CONTEXT_API)
        {
            if (!_glfwInitOSMesa())
                return false;
        }
    }

    if (!visual)
    {
        visual = DefaultVisual(_glfw.x11.display, _glfw.x11.screen);
        depth = DefaultDepth(_glfw.x11.display, _glfw.x11.screen);
    }

    if (!createNativeWindow(window, wndconfig, visual, depth))
        return false;

    if (ctxconfig->client != GLFW_NO_API)
    {
        if (ctxconfig->source == GLFW_NATIVE_CONTEXT_API)
        {
            if (!_glfwCreateContextGLX(window, ctxconfig, fbconfig))
                return false;
        }
        else if (ctxconfig->source == GLFW_EGL_CONTEXT_API)
        {
            if (!_glfwCreateContextEGL(window, ctxconfig, fbconfig))
                return false;
        }
        else if (ctxconfig->source == GLFW_OSMESA_CONTEXT_API)
        {
            if (!_glfwCreateContextOSMesa(window, ctxconfig, fbconfig))
                return false;
        }
    }
```

The `source` is validated to be one of `GLFW_NATIVE_CONTEXT_API`, `GLFW_EGL_CONTEXT_API`, or `GLFW_OSMESA_CONTEXT_API` at `glfw/context.c:L60-L62`. kitty does not override the X11 context-creation API, so it keeps GLFW's default `GLFW_NATIVE_CONTEXT_API`, which routes to `_glfwInitGLX()` (`L1880`) then `_glfwCreateContextGLX()` (`L1912`) — i.e. **GLX**. (`GLFW_EGL_CONTEXT_API` would route to `_glfwInitEGL()`/`_glfwCreateContextEGL()` at `L1887`/`L1917`, and `GLFW_OSMESA_CONTEXT_API` to `_glfwInitOSMesa()`/`_glfwCreateContextOSMesa()` at `L1894`/`L1922`. Wayland always uses EGL.)

To confirm which path is *actually* taken at runtime — rather than inferring from code — I inspected the GL client libraries mapped into the live kitty process. `strace`/`ltrace` are absent in this container, so `/proc/<pid>/maps` was used instead. This is an OS‑level observation of kitty's own process (not a bypass of kitty's code path); it reports exactly which libraries kitty's GLFW loaded.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty -o confirm_os_window_close=0 sh -c 'sleep 4' &
sleep 2.5
# pick the kitty pid whose maps contain libGL, then inspect GL-related maps
TARGET=$(for p in $(pgrep -f 'launcher/kitty'); do grep -q 'libGL' /proc/$p/maps 2>/dev/null && echo $p && break; done)
grep -oE '/[^ ]*(libGL[^ ]*|libEGL[^ ]*|libGLX[^ ]*|libGLdispatch[^ ]*)' /proc/$TARGET/maps | sort -u
grep -q 'libGLX_mesa' /proc/$TARGET/maps && echo "libGLX_mesa (Mesa GLX): PRESENT" || echo "libGLX_mesa: ABSENT"
grep -q 'libEGL'      /proc/$TARGET/maps && echo "libEGL: PRESENT" || echo "libEGL: ABSENT (=> not the EGL path)"
```

**Complete, unedited output:**

```
/usr/lib/x86_64-linux-gnu/libGL.so.1.7.0
/usr/lib/x86_64-linux-gnu/libGLX.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLdispatch.so.0.0.0
libGLX_mesa (Mesa GLX): PRESENT
libEGL: ABSENT (=> not the EGL path)
```

**Cause → effect.** The environment (no `WAYLAND_DISPLAY`) drives `is_wayland()` → `False` → the `x11` backend (`kitty/main.py:L96`). On x11, GLFW's default context source is `GLFW_NATIVE_CONTEXT_API`, which routes to `_glfwInitGLX()` (`glfw/x11_window.c:L1880`) and creates the context with `_glfwCreateContextGLX()`. The runtime library map confirms this precisely: the vendor‑neutral `libGLX.so` and Mesa's GLX vendor driver `libGLX_mesa.so` are loaded, while **no** `libEGL` is mapped — so kitty's GL context is a **GLX** context, backed by Mesa. OSMESA is not used (it would require `GLFW_OSMESA_CONTEXT_API`, which kitty does not request).

---

## (c) Observed GL strings + extension checks

### Direct answer

All four GL strings, obtained from **kitty's own live GL context**, stable across runs:

| GL string | Value |
|---|---|
| `GL_VENDOR` | `Mesa` |
| `GL_RENDERER` | `llvmpipe (LLVM 20.1.8, 256 bits)` |
| `GL_VERSION` | `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2` |
| `GL_SHADING_LANGUAGE_VERSION` | `4.50` |

This is **software rasterization** (llvmpipe), *not* hardware acceleration — and llvmpipe is the honest, canonical value for this GPU‑less environment. The mandatory extension **`GL_ARB_texture_storage` is present**, so kitty's fatal extension gate passes.

**Critical distinction — required minimum vs observed version.** kitty's *required minimum* GL on **Linux is 3.1**, not 3.3. The observed *runtime* version is 4.5. These are two different things and are kept separate below.

### What kitty itself prints, and where

kitty prints **only** the `GL_VERSION` line (under `--debug-rendering`), built by `gl_version_string()` at `kitty/gl.c:L41-L49`:

```c
const char*
gl_version_string(void) {
    static char buf[256];
    int gl_major = GLAD_VERSION_MAJOR(global_state.gl_version);
    int gl_minor = GLAD_VERSION_MINOR(global_state.gl_version);
    const char *gvs = (const char*)glGetString(GL_VERSION);
    snprintf(buf, sizeof(buf), "'%s' Detected version: %d.%d", gvs, gl_major, gl_minor);
    return buf;
}
```

printed at `kitty/gl.c:L72` (complete line, verbatim):

```c
        if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", monotonic_t_to_s_double(monotonic()), gl_version_string());
```

**Command (the `GL version string` line from 3 consecutive runs; the complete per-run output is in part (a)):**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
for i in 1 2 3; do ./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'true' 2>&1 | grep "GL version string" > /tmp/obs/gl_run$i.txt; cat /tmp/obs/gl_run$i.txt; done
```

**Complete, unedited output:**

```
[0.133] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[0.133] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[0.126] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

**Stability proof** — the three runs differ only in the leading `[t]` monotonic timestamp; stripping it collapses them to a single unique line:

```bash
for i in 1 2 3; do sed -E 's/^\[[0-9.]+\] //' /tmp/obs/gl_run$i.txt; done | sort -u | wc -l
```

```
1
```

A single unique line ⇒ the GL version string is byte-identical across all 3 runs (only the timestamp varies).

### Obtaining `GL_VENDOR` / `GL_RENDERER` / `GL_SHADING_LANGUAGE_VERSION` from kitty's real context

kitty does not print these three strings; the C symbols `C(GL_VENDOR)` etc. at `kitty/shaders.c:L1255-L1258` merely expose the GL *enum constants* (integers) to Python, not the strings. To read the strings **from kitty's own context** (not an external tool), I attached a **temporary** observation harness (removed afterward): a `sitecustomize.py` on `PYTHONPATH` that wraps `prerender_function` (`kitty/fonts/render.py:L364`) — which kitty invokes from `send_prerendered_sprites()` (`kitty/fonts.c:L1450`) **while the GL context is current** — and, inside the wrapper, reads `glGetString`/`glGetStringi` via `ctypes` against `libGL.so.1` before calling the original unchanged. Same process, same GLX context kitty created, at the exact moment kitty uploads glyph sprites. This is a labeled in-process probe, cross-validated below against kitty's own `GL_VERSION` line and against `glxinfo`.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
# /tmp/obs/ppgl/sitecustomize.py wraps kitty.fonts.render.prerender_function; inside it
# (GL context current): gl=ctypes.CDLL('libGL.so.1'); gl.glGetString.restype=c_char_p;
# read GL_VENDOR/RENDERER/VERSION/SHADING_LANGUAGE_VERSION; count GL_NUM_EXTENSIONS and
# enumerate glGetStringi(GL_EXTENSIONS,i) to test for GL_ARB_texture_storage.
for i in 1 2 3; do
  GLOBS_OUT=/tmp/obs/glstrings_probe_run$i.txt PYTHONPATH=/tmp/obs/ppgl \
    ./kitty/launcher/kitty -o confirm_os_window_close=0 sh -c 'true' >/dev/null 2>&1
done
cat /tmp/obs/glstrings_probe_run1.txt
```

**Complete, unedited output (run 1 of 3):**

```
GL_VENDOR='Mesa'
GL_RENDERER='llvmpipe (LLVM 20.1.8, 256 bits)'
GL_VERSION='4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2'
GL_SHADING_LANGUAGE_VERSION='4.50'
GL_NUM_EXTENSIONS=229
ARB_texture_storage_present=True
ARB_texture_storage_entry='GL_ARB_texture_storage'
```

**Stability proof (byte-identical across all 3 runs):**

```bash
diff /tmp/obs/glstrings_probe_run1.txt /tmp/obs/glstrings_probe_run2.txt \
  && diff /tmp/obs/glstrings_probe_run1.txt /tmp/obs/glstrings_probe_run3.txt \
  && echo IDENTICAL
```

```
IDENTICAL
```

The `GL_VERSION` here is **byte-identical** to the value kitty prints itself (`4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2`), confirming the probe reads the same context kitty uses.

**Cross-check with `glxinfo`** (external tool, corroboration only — not kitty's path):

```bash
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 glxinfo -B
```

**Complete, unedited output:**

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
    VBO free aux. memory - total: 4294596874 MB, largest block: 4294596874 MB
    Texture free memory - total: 0 MB, largest block: 0 MB
    Texture free aux. memory - total: 4294596874 MB, largest block: 4294596874 MB
    Renderbuffer free memory - total: 0 MB, largest block: 0 MB
    Renderbuffer free aux. memory - total: 4294596874 MB, largest block: 4294596874 MB
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

`glxinfo` independently reports Vendor `Mesa`, Device `llvmpipe (LLVM 20.1.8, 256 bits)`, core-profile version `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2`, and GLSL `4.50` — matching kitty's own context exactly — and **`Accelerated: no`** confirms software rasterization.

### The version requirement (3.1 on Linux) and the extension gate

The required minimum is defined in `kitty/data-types.h:L20-L26`:

```c
#define OPENGL_REQUIRED_VERSION_MAJOR 3      // L20
#ifdef __APPLE__                             // L21
#define OPENGL_REQUIRED_VERSION_MINOR 3      // L22  (Apple)
#else                                        // L23
#define OPENGL_REQUIRED_VERSION_MINOR 1      // L24  (Linux ⇒ 3.1)
#endif
#define GLSL_VERSION 140                     // L26  (GL 3.1)
```

So on **Linux the minimum is OpenGL 3.1** (GLSL 140), and on macOS it is 3.3. The window hints request exactly this minimum at `kitty/glfw.c:L1127-L1128` (`glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR/MINOR, OPENGL_REQUIRED_VERSION_MAJOR/MINOR)`). The version gate in `gl_init()` at `kitty/gl.c:L73-L74` fatals only if the obtained version is below the platform minimum:

```c
        if (gl_major < OPENGL_REQUIRED_VERSION_MAJOR || (gl_major == OPENGL_REQUIRED_VERSION_MAJOR && gl_minor < OPENGL_REQUIRED_VERSION_MINOR)) {
            fatal("OpenGL version is %d.%d, version >= %d.%d required for kitty", gl_major, gl_minor, OPENGL_REQUIRED_VERSION_MAJOR, OPENGL_REQUIRED_VERSION_MINOR);
        }
```

The mandatory extension check is **independent of version**, at `kitty/gl.c:L63-L67`:

```c
#define ARB_TEST(name) \                                   // L63
    if (!GLAD_GL_ARB_##name) { \
        fatal("The OpenGL driver on this system is missing the required extension: ARB_%s", #name); }  // L65
ARB_TEST(texture_storage);                                 // L67
```

Because the observed context is 4.5 (≥ 3.1) and exposes `GL_ARB_texture_storage` (confirmed present above, among 229 extensions), both gates pass and startup proceeds — kitty launched successfully, which is itself runtime proof the gates passed.

**Cause → effect.** With no GPU, Mesa resolves GL to its Gallium **llvmpipe** software rasterizer; hence `GL_RENDERER='llvmpipe (LLVM 20.1.8, 256 bits)'` and `GL_VENDOR='Mesa'`. llvmpipe advertises a 4.5 core profile — far above kitty's Linux floor of 3.1 — so the version gate (`kitty/gl.c:L74`) is satisfied. The `ARB_texture_storage` requirement is checked separately (`kitty/gl.c:L67`) because kitty uses immutable texture storage for its glyph atlas regardless of GL version; llvmpipe provides it, so the extension gate also passes. The correct way to report this environment is therefore the **observed software** renderer (llvmpipe), not a hardware value.


---

## (d) Detected display configuration

### Direct answer

- **Content scale: `1.0`** (both axes). **Logical DPI: `96.0`** (both axes).
- **Initial window size: `640 × 400` *pixels*** (framebuffer `640 × 400`). Note: the defaults are **pixels**, not cells (see below).
- **HiDPI edge:** forcing `Xft.dpi: 192` makes content scale `2.0`, logical DPI `192.0`, and cell metrics double — while the window stays `640 × 400` px.
- **Error/edge:** with no/invalid `DISPLAY`, kitty fails early with a verbatim GLFW init error and exits 1 (captured below).

### Mechanism

Logical DPI is derived from the monitor content scale by `dpi_from_scale()` at `kitty/glfw.c:L812-L819`: `dpi = content_scale × factor`, where `factor = 72.0` on Apple and **`96.0` on Linux**. The content scale itself is obtained (and clamped for invalid/NaN/absurd values → `1.0`) by `get_window_content_scale()` at `kitty/glfw.c:L823-L834`. On X11 the scale comes from the X resource `Xft.dpi`, read by GLFW's `_glfwGetSystemContentScaleX11()` at `glfw/x11_init.c:L462` (`XResourceManagerString` L483 → `XrmGetResource(db, "Xft.dpi", "Xft.Dpi", &type, &value)` L494 → `atof` L497 → `scale = dpi / 96` L505‑506; default 96 if unset, L467).

The initial window size is produced by `initial_window_size_func()` (`kitty/os_window_size.py:L54`), whose inner `get_window_size(cell_width, cell_height, dpi_x, dpi_y, xscale, yscale)` (`kitty/os_window_size.py:L70`) forces `xscale = yscale = 1` on X11 (`L73-L74`) and then computes width/height. Crucially, the width/height branch depends on the *unit*: only when the unit is `'cells'` does it multiply by cell size and DPI spacing (`L88-L90`); otherwise it uses the raw pixel value (`width = w`, `L92`; `height = h`). The defaults `initial_window_width 640` (`kitty/options/definition.py:L994`) and `initial_window_height 400` (`kitty/options/definition.py:L998`) carry the **`px`** unit, so the window is 640×400 pixels.

### Observed default configuration

Captured in‑process through kitty's own `fast_data_types.get_os_window_size()` (the same accessor kitty uses), via a watcher module fired at window creation:

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty -o watcher=/tmp/obs/obs_watcher.py -o confirm_os_window_close=0 sh -c 'sleep 2'
# the same temporary watcher shown in part (b); on_resize records get_os_window_size(os_window_id) + the font table
```

**Complete, unedited output:**

```json
{
  "tag": "on_resize",
  "os_window_id": 1,
  "get_os_window_size": {
    "width": 640,
    "height": 400,
    "framebuffer_width": 640,
    "framebuffer_height": 400,
    "xscale": 1.0,
    "yscale": 1.0,
    "xdpi": 96.0,
    "ydpi": 96.0,
    "cell_width": 9,
    "cell_height": 18
  },
  "cell_size_for_window": [9, 18],
  "opengl_version_string": "'4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5",
  "compositor_name": "X11",
  "current_fonts": {
    "medium": "DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0",
    "bold": "DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0",
    "italic": "DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0",
    "bi": "DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0"
  }
}
```

This confirms scale `1.0`, DPI `96.0`, window `640 × 400` px, framebuffer `640 × 400`, and (foreshadowing part (e)) cell `9 × 18`. Kitty's resolved options corroborate the pixel unit:

```
opts.font_size = 11.0
opts.initial_window_width = (640, 'px')
opts.initial_window_height = (400, 'px')
```

The unit is literally `'px'`. (This corrects any description of the initial size as "cells"; on the default path it is pixels.)

### HiDPI edge condition (non-1.0 scale)

I set `Xft.dpi: 192` on the X server (the input GLFW reads for content scale), then relaunched the same watcher (part (b)). Method labeled: this changes the *display's* advertised DPI; kitty honors it via the GLFW mechanism cited above.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
printf 'Xft.dpi: 192\n' | xrdb -merge
./kitty/launcher/kitty -o watcher=/tmp/obs/obs_watcher.py -o confirm_os_window_close=0 sh -c 'sleep 2'
```

**Complete, unedited output:**

```json
{
  "tag": "on_resize",
  "os_window_id": 1,
  "get_os_window_size": {
    "width": 640,
    "height": 400,
    "framebuffer_width": 640,
    "framebuffer_height": 400,
    "xscale": 2.0,
    "yscale": 2.0,
    "xdpi": 192.0,
    "ydpi": 192.0,
    "cell_width": 18,
    "cell_height": 36
  },
  "cell_size_for_window": [
    18,
    36
  ],
  "opengl_version_string": "'4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5",
  "compositor_name": "X11",
  "opts.font_size": 11.0,
  "opts.initial_window_width": [
    640,
    "px"
  ],
  "opts.initial_window_height": [
    400,
    "px"
  ],
  "current_fonts": {
    "medium": "DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0",
    "bold": "DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0",
    "italic": "DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0",
    "bi": "DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0"
  }
}
```

So content scale `2.0` → logical DPI `192.0` (= 2.0 × 96) → cell size doubles to `18 × 36`, while the window stays `640 × 400` px (the `px` unit is DPI-independent).

### Reset / re-verify to the default

To prove the HiDPI change is not sticky and the default is reproducible, the `Xft.dpi` override was cleared and kitty relaunched with the same watcher.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
xrdb -load /dev/null                     # clear all X resources (removes the Xft.dpi override)
./kitty/launcher/kitty -o watcher=/tmp/obs/obs_watcher.py -o confirm_os_window_close=0 sh -c 'sleep 2'
```

**Complete, unedited output:**

```json
{
  "tag": "on_resize",
  "os_window_id": 1,
  "get_os_window_size": {
    "width": 640,
    "height": 400,
    "framebuffer_width": 640,
    "framebuffer_height": 400,
    "xscale": 1.0,
    "yscale": 1.0,
    "xdpi": 96.0,
    "ydpi": 96.0,
    "cell_width": 9,
    "cell_height": 18
  },
  "cell_size_for_window": [
    9,
    18
  ],
  "opengl_version_string": "'4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5",
  "compositor_name": "X11",
  "opts.font_size": 11.0,
  "opts.initial_window_width": [
    640,
    "px"
  ],
  "opts.initial_window_height": [
    400,
    "px"
  ],
  "current_fonts": {
    "medium": "DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0",
    "bold": "DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0",
    "italic": "DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0",
    "bi": "DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0"
  }
}
```

Scale is back to `1.0`, DPI `96.0`, cell `9 × 18` — byte-identical (in the metric fields) to the original default capture above, confirming the default is stable and the HiDPI edge is fully reversible.

### Error / edge path (broken or absent display)

**Command (DISPLAY unset entirely):**

```bash
env -u DISPLAY -u WAYLAND_DISPLAY LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty -o confirm_os_window_close=0 sh -c 'true'
```

**Complete, unedited output (exit 1):**

```
[0.067] [glfw error 65544]: X11: The DISPLAY environment variable is missing
GLFW initialization failed
```

**Command (DISPLAY points at a nonexistent server):**

```bash
DISPLAY=:123 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty -o confirm_os_window_close=0 sh -c 'true'
```

**Complete, unedited output (exit 1):**

```
[0.106] [glfw error 65544]: X11: Failed to open display :123
GLFW initialization failed
```

Source of these lines: the GLFW error callback `error_callback()` at `kitty/glfw.c:L1412-L1413` (`log_error("[glfw error %d]: %s", error, description)`); error code `65544` is `GLFW_PLATFORM_ERROR` (`glfw/glfw3.h:L724`, `0x00010008`). `glfwInit()` is invoked at `kitty/glfw.c:L1456`; when it returns false, `init_glfw_module()` raises `SystemExit('GLFW initialization failed')` at `kitty/main.py:L92`.

This failure occurs at **GLFW initialization** (before any window is created). It is distinct from — and earlier than — the window‑creation fatal at `kitty/glfw.c:L1199` (`"Failed to create GLFW temp window! This usually happens because of old/broken OpenGL drivers. kitty requires working OpenGL %d.%d drivers."`, citing `OPENGL_REQUIRED_VERSION`), which would fire only if init succeeded but the GL context could not be created (e.g. broken drivers). With an absent/broken display, init fails first, so the L1199 fatal is not reached (that deeper path is *inferred* from code, not triggered here).

**Cause → effect.** The chain is display → content scale → logical DPI → cell metrics → pixel geometry. On X11 the display's `Xft.dpi` sets GLFW's content scale (`glfw/x11_init.c:L462`); kitty multiplies by 96 to get logical DPI (`kitty/glfw.c:L812`); logical DPI scales FreeType face metrics into cell pixels (part (e)). Window pixel dimensions with the default `px` unit are independent of DPI (`kitty/os_window_size.py:L92`), which is why the window stays 640×400 while cells grow under HiDPI. When there is no usable display, X11 platform init raises `GLFW_PLATFORM_ERROR`, kitty logs it and aborts — the GPU path is never reached, exactly as observed.

---

## (e) Font setup + cell metrics — and a correction to the "two‑phase" premise

### Direct answer (leading with the observed correction)

**The commonly assumed "two‑phase" cell‑metric computation does not occur in kitty 0.35.2.** Cell metrics are computed **exactly once**, at the **real detected DPI (96)**, inside `create_os_window()` — **not** once at a default DPI before the window and again at the real DPI. The function often assumed to compute metrics pre‑window, `set_font_family()`, creates **no font group at all** and computes **no** metrics; it only resolves font files and stores descriptors/callbacks.

Observed values at the real detected DPI (96, equal to the default X11 baseline), stable across runs:

| Metric | Value (DPI 96) |
|---|---|
| cell_width | 9 |
| cell_height | 18 |
| baseline | 14 |
| underline_position | 15 |
| underline_thickness | 1 |
| strikethrough_position | 10 |
| strikethrough_thickness | 1 |
| cursor_beam_thickness | 1.5 |
| cursor_underline_thickness | 2.0 |
| font_size (pts) | 11.0 |

Font family resolves to **DejaVu Sans Mono** (the system monospace default).

### Font resolution

Default `font_size` is `11.0` (`kitty/options/definition.py:L59`), and the default `font_family` is `FontSpec(system='monospace')`, which resolves via fontconfig to DejaVu Sans Mono on this image.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty --debug-font-fallback -o confirm_os_window_close=0 sh -c 'true'
```

**Complete, unedited output:**

```
[0.231] Text fonts:
[0.231]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.231]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.231]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.231]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

kitty's own `debug_config` corroborates the same four faces and `font_size = 11.0` (see part (g)).

> A note on `--debug-config`: it is **not** a launch flag in 0.35.2 (`./kitty/launcher/kitty --debug-config` → `Unknown option: --debug-config`, exit 1). It is a runtime action (`Boss.debug_config`, `kitty/boss.py:L3060-L3064`) that calls `debug_config(get_options())`. I therefore captured that same canonical output in‑process (part (g)).

### Cell metrics — captured at the real DPI

Cell metrics are computed by `calc_cell_metrics()` at `kitty/fonts.c:L373-L403`, which calls `cell_metrics(...)` on the medium font face (`L375`), fatals if the width is zero (`L376`), and applies DPI‑adjusted overrides via `adjust_metric(...)` (`L379-L380`) and the `A(which, dpi)` macro (`L398`) for baseline/underline/strikethrough. The values flow to Python through `prerender_function` (`kitty/fonts/render.py:L364`), invoked by `send_prerendered_sprites()` (`kitty/fonts.c:L1450`) — which runs with the GL context current, i.e. during window creation.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty +launch /tmp/obs_prerender.py -o confirm_os_window_close=0 sh -c 'sleep 2'
# obs_prerender.py wraps kitty.fonts.render.prerender_function and records its arguments
```

**Complete, unedited output:**

```
PRERENDER {"cell_width": 9, "cell_height": 18, "baseline": 14, "underline_position": 15, "underline_thickness": 1, "strikethrough_position": 10, "strikethrough_thickness": 1, "cursor_beam_thickness": 1.5, "cursor_underline_thickness": 2.0, "dpi_x": 96.0, "dpi_y": 96.0}
```

### Proof that there is only one computation phase

I instrumented `kitty.main.set_font_family` (the alleged pre‑window "Phase 1") and `prerender_function` (fires whenever a font group is initialized, i.e. when `calc_cell_metrics` runs), and probed for the existence of a font group before and after `set_font_family`.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty +launch /tmp/obs_phase.py -o confirm_os_window_close=0 sh -c 'sleep 2'
```

**Complete, unedited output (two runs shown — identical, confirming stability):**

```
CALL kitty.main.set_font_family (Phase-1 candidate, pre-window)
  before set_font_family: NO font group -> RuntimeError('must create font group first')
  after set_font_family: NO font group -> RuntimeError('must create font group first')
PRERENDER#1 (font-group init -> calc_cell_metrics ran) dpi=(96.0000,96.0000) cell=9x18 baseline=14 ul_pos=15 ul_th=1 st_pos=10 st_th=1
CALL kitty.main.set_font_family (Phase-1 candidate, pre-window)
  before set_font_family: NO font group -> RuntimeError('must create font group first')
  after set_font_family: NO font group -> RuntimeError('must create font group first')
PRERENDER#1 (font-group init -> calc_cell_metrics ran) dpi=(96.0000,96.0000) cell=9x18 baseline=14 ul_pos=15 ul_th=1 st_pos=10 st_th=1
```

`calc_cell_metrics` fires exactly **once** (`PRERENDER#1`), at DPI `(96, 96)` — the real detected DPI — and never at a separate default DPI beforehand.

### Why — the mechanism

`set_font_family()` (Python, `kitty/fonts/render.py:L173`, called from `AppRunner.__call__` at `kitty/main.py:L251`) does two things: (1) resolves font *files* via `get_font_files(opts)` (fontconfig discovery — no metrics), and (2) calls the C function `set_font_data` (`kitty/fonts.c:L1434`). That C function stores the callbacks (`box_drawing_function`, `prerender_function`, `descriptor_for_idx`) and parses `OPT(font_size)`, then calls **`free_font_groups()`** at `kitty/fonts.c:L1442` — it *destroys* font groups rather than creating one, and computes no metrics. That is exactly why the probe reports "NO font group" both before and after.

The metric computation happens only when a font group is *initialized*: `initialize_font_group()` (`kitty/fonts.c:L1495`) calls `calc_cell_metrics()` at `kitty/fonts.c:L1511`; it is reached via `font_group_for()` (`kitty/fonts.c:L204`, which calls `initialize_font_group` at `L216`), which is reached via `load_fonts_data()` (`kitty/fonts.c:L1530`, calling `font_group_for` at `L1531`). And `load_fonts_data()` is first called inside `create_os_window()` at `kitty/glfw.c:L1202`, using the **real** DPI obtained from the temporary window. Hence a single computation, at the real DPI.

For completeness: the `set_scale()` call at `kitty/main.py:L248` (defined in `kitty/fonts/box_drawing.py:L20`) sets the box‑drawing scale from `opts.box_drawing_scale` *before* `set_font_family` — it configures box‑drawing geometry, not cell metrics. The optional post‑show recompute path exists at `kitty/glfw.c:L1232-L1240` (a second `get_window_content_scale` at `L1235` + `load_fonts_data` at `L1239`), but it is **guarded by `if (global_state.is_wayland || is_apple)`** at `kitty/glfw.c:L1232` — so on **X11 it is skipped entirely** (never reached, regardless of scale). This is confirmed by observation: only one `PRERENDER` fires. The recompute exists because Wayland/fractional‑scale and macOS multi‑monitor moves can change DPI *after* the surface is shown.

**Cause → effect.** The real DPI must be known before the *visible* window is sized, because cell size (which depends on DPI) determines how content will lay out. kitty achieves this by (i) storing font descriptors early in `set_font_family`/`set_font_data` (cheap, no GL, no metrics) and (ii) deferring the actual metric computation to `create_os_window`, where a hidden temporary window yields the true content scale/DPI. At that single point `calc_cell_metrics` scales the DejaVu Sans Mono face metrics by DPI 96 to yield cell `9 × 18` with baseline 14 and the underline/strikethrough geometry above. Under HiDPI (DPI 192) the same computation yields `18 × 36` (part (d)) — linear in DPI, exactly as the `adjust_metric`/`A()` scaling predicts.


---

## (f) Text‑rendering / terminal capabilities — exact response bytes over a PTY

### Direct answer

Driving kitty over its real PTY and injecting each query, kitty responds with the following **exact bytes** for all **17 queries** — Primary/Secondary/Tertiary Device Attributes, XTVERSION, DSR, three DECRQM mode reports, the kitty keyboard and graphics protocols, **text‑sizing (CSI 14/16/18 t)**, and **underline‑capability (XTGETTCAP Su/Smulx/Setulc)** queries. Every response is **byte‑identical across 3 runs** and verified byte‑for‑byte against the source strings below:

| Query (sent) | Response (received) | Meaning |
|---|---|---|
| Primary DA `\x1b[c` | `\x1b[?62;c` | VT‑220‑class terminal (`?62`) |
| Secondary DA `\x1b[>c` | `\x1b[>1;4000;35c` | type 1, version 4000, ROM 35 |
| Tertiary DA `\x1b[=c` | `*(no response)*` | not implemented |
| XTVERSION `\x1b[>q` | `\x1bP>\|kitty(0.35.2)\x1b\\` | name+version identity `kitty(0.35.2)` |
| DSR cursor `\x1b[6n` | `\x1b[1;1R` | cursor at row 1, col 1 |
| DECRQM 2026 `\x1b[?2026$p` | `\x1b[?2026;2$y` | synchronized‑output mode recognized, currently reset |
| DECRQM 1049 `\x1b[?1049$p` | `\x1b[?1049;2$y` | alt‑screen mode recognized, currently reset |
| DECRQM 25 `\x1b[?25$p` | `\x1b[?25;1$y` | cursor‑visibility mode set (visible) |
| kitty keyboard `\x1b[?u` | `\x1b[?0u` | kitty keyboard protocol; flags 0 at startup |
| Text size (CSI 14 t) `\x1b[14t` | `\x1b[4;396;639t` | text‑area size in px (code 4): height 396 × width 639 |
| Cell size (CSI 16 t) `\x1b[16t` | `\x1b[6;18;9t` | cell size in px (code 6): height 18 × width 9 |
| Screen size (CSI 18 t) `\x1b[18t` | `\x1b[8;22;71t` | screen size in cells (code 8): 22 lines × 71 cols |
| XTGETTCAP TN `\x1bP+q544e\x1b\\` | `\x1bP1+r544e=787465726d2d6b69747479\x1b\\` | terminal name = `xterm-kitty` |
| XTGETTCAP Su `\x1bP+q5375\x1b\\` | `\x1bP1+r5375\x1b\\` | styled/colored underline supported (boolean, no value) |
| XTGETTCAP Smulx `\x1bP+q536d756c78\x1b\\` | `\x1bP1+r536d756c78=5c455b343a25703125646d\x1b\\` | underline‑style SGR = `\E[4:%p1%dm` |
| XTGETTCAP Setulc `\x1bP+q536574756c63\x1b\\` | `\x1bP1+r536574756c63=5c455b35383a323a257031257b36353533367d252f25643a257031257b3235367d252f257b3235357d252625643a257031257b3235357d25262564253b6d\x1b\\` | underline‑color SGR (SGR 58:2 truecolor) |
| kitty graphics `\x1b_Gi=31,s=1,v=1,a=q,t=d,f=24;AAAA\x1b\\` | `\x1b_Gi=31;OK\x1b\\` | graphics protocol supported (`OK` for image id 31) |

### Method

An ephemeral harness (`/tmp/obs/pty_child.py`, removed afterward) is run **as the child of the real launcher**, so its `stdin`/`stdout` *are* kitty's PTY slave. It puts the tty in raw mode, writes each query byte‑sequence to stdout (which flows into kitty's terminal parser), reads kitty's response back from stdin, and records the exact bytes to a JSON file. This exercises kitty's real capability‑reporting code (`kitty/screen.c`, `kitty/vt-parser.c`, `kitty/terminfo.py`), not a bypass. The byte‑sensitive query list is shown complete:

```python
#!/usr/bin/env python3
# Ephemeral OBSERVATION child (removed after use). Runs INSIDE kitty as the child
# process on the real PTY. Writes terminal queries to stdout (-> kitty parses them
# as terminal input on the real vt-parser path) and reads kitty's real response
# bytes back from stdin (the PTY slave). Records exact bytes to OBS_PTY_OUT.
import os
import sys
import termios
import tty
import select
import json

OUT = os.environ.get("OBS_PTY_OUT", "/tmp/obs/pty_responses.json")
fd = sys.stdin.fileno()
old = termios.tcgetattr(fd)
tty.setraw(fd)

# (label, query-bytes). Byte-sensitive; keep exact.
QUERIES = [
    ("primary_DA",        b"\x1b[c"),
    ("secondary_DA",      b"\x1b[>c"),
    ("tertiary_DA",       b"\x1b[=c"),
    ("XTVERSION",         b"\x1b[>q"),
    ("DSR_cursor_pos",    b"\x1b[6n"),
    ("DECRQM_2026",       b"\x1b[?2026$p"),
    ("DECRQM_1049",       b"\x1b[?1049$p"),
    ("DECRQM_25",         b"\x1b[?25$p"),
    ("kbd_flags",         b"\x1b[?u"),
    # text sizing (CSI 14/16/18 t) -> screen_report_size
    ("textsize_CSI_14_t", b"\x1b[14t"),
    ("textsize_CSI_16_t", b"\x1b[16t"),
    ("textsize_CSI_18_t", b"\x1b[18t"),
    # XTGETTCAP underline caps + TN
    ("xtgettcap_TN",      b"\x1bP+q544e\x1b\\"),
    ("xtgettcap_Su",      b"\x1bP+q5375\x1b\\"),
    ("xtgettcap_Smulx",   b"\x1bP+q536d756c78\x1b\\"),
    ("xtgettcap_Setulc",  b"\x1bP+q536574756c63\x1b\\"),
    # kitty graphics query (APC _G ... ST) — 1x1 RGB, direct
    ("kitty_graphics",    b"\x1b_Gi=31,s=1,v=1,a=q,t=d,f=24;AAAA\x1b\\"),
]


def read_response(timeout=0.6):
    buf = b""
    while True:
        r, _, _ = select.select([fd], [], [], timeout)
        if not r:
            break
        chunk = os.read(fd, 4096)
        if not chunk:
            break
        buf += chunk
    return buf


results = {}
try:
    # small settle read to drain any startup chatter
    read_response(0.3)
    for label, q in QUERIES:
        os.write(1, q)
        resp = read_response(0.6)
        results[label] = {
            "query_bytes": q.decode("latin1"),
            "query_hex": q.hex(),
            "response_bytes": resp.decode("latin1"),
            "response_hex": resp.hex(),
            "response_repr": repr(resp),
        }
finally:
    termios.tcsetattr(fd, termios.TCSADRAIN, old)
    with open(OUT, "w") as f:
        json.dump(results, f, indent=2)
```

A trivial formatter (`/tmp/obs/pty_fmt.py`) renders the JSON kitty's child wrote into a complete, readable block (a *display* tool over kitty's own recorded bytes, not a second source):

```python
import json, sys
d = json.load(open(sys.argv[1]))
for label, e in d.items():
    q = e["query_bytes"].encode("latin1")
    r = e["response_bytes"].encode("latin1")
    hexsp = " ".join(r.hex()[i:i+2] for i in range(0, len(r.hex()), 2))
    print("%-18s QUERY=%-40r RESP_len=%d" % (label, q, len(r)))
    print("    RESP_repr = %r" % r)
    print("    RESP_hex  = %s" % hexsp)
```

**Command (create harness, run 3× writing one JSON per run, format run 1):**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
# /tmp/obs/pty_child.py is the harness shown above; /tmp/obs/pty_fmt.py the formatter above
for i in 1 2 3; do
  OBS_PTY_OUT=/tmp/obs/pty_responses_run$i.json \
    ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 /tmp/obs/pty_child.py
done
python3 /tmp/obs/pty_fmt.py /tmp/obs/pty_responses_run1.json
```

**Complete, unedited output (run 1 of 3 — all 17 queries, no elision):**

```
primary_DA         QUERY=b'\x1b[c'                                RESP_len=7
    RESP_repr = b'\x1b[?62;c'
    RESP_hex  = 1b 5b 3f 36 32 3b 63
secondary_DA       QUERY=b'\x1b[>c'                               RESP_len=13
    RESP_repr = b'\x1b[>1;4000;35c'
    RESP_hex  = 1b 5b 3e 31 3b 34 30 30 30 3b 33 35 63
tertiary_DA        QUERY=b'\x1b[=c'                               RESP_len=0
    RESP_repr = b''
    RESP_hex  = 
XTVERSION          QUERY=b'\x1b[>q'                               RESP_len=19
    RESP_repr = b'\x1bP>|kitty(0.35.2)\x1b\\'
    RESP_hex  = 1b 50 3e 7c 6b 69 74 74 79 28 30 2e 33 35 2e 32 29 1b 5c
DSR_cursor_pos     QUERY=b'\x1b[6n'                               RESP_len=6
    RESP_repr = b'\x1b[1;1R'
    RESP_hex  = 1b 5b 31 3b 31 52
DECRQM_2026        QUERY=b'\x1b[?2026$p'                          RESP_len=11
    RESP_repr = b'\x1b[?2026;2$y'
    RESP_hex  = 1b 5b 3f 32 30 32 36 3b 32 24 79
DECRQM_1049        QUERY=b'\x1b[?1049$p'                          RESP_len=11
    RESP_repr = b'\x1b[?1049;2$y'
    RESP_hex  = 1b 5b 3f 31 30 34 39 3b 32 24 79
DECRQM_25          QUERY=b'\x1b[?25$p'                            RESP_len=9
    RESP_repr = b'\x1b[?25;1$y'
    RESP_hex  = 1b 5b 3f 32 35 3b 31 24 79
kbd_flags          QUERY=b'\x1b[?u'                               RESP_len=5
    RESP_repr = b'\x1b[?0u'
    RESP_hex  = 1b 5b 3f 30 75
textsize_CSI_14_t  QUERY=b'\x1b[14t'                              RESP_len=12
    RESP_repr = b'\x1b[4;396;639t'
    RESP_hex  = 1b 5b 34 3b 33 39 36 3b 36 33 39 74
textsize_CSI_16_t  QUERY=b'\x1b[16t'                              RESP_len=9
    RESP_repr = b'\x1b[6;18;9t'
    RESP_hex  = 1b 5b 36 3b 31 38 3b 39 74
textsize_CSI_18_t  QUERY=b'\x1b[18t'                              RESP_len=10
    RESP_repr = b'\x1b[8;22;71t'
    RESP_hex  = 1b 5b 38 3b 32 32 3b 37 31 74
xtgettcap_TN       QUERY=b'\x1bP+q544e\x1b\\'                     RESP_len=34
    RESP_repr = b'\x1bP1+r544e=787465726d2d6b69747479\x1b\\'
    RESP_hex  = 1b 50 31 2b 72 35 34 34 65 3d 37 38 37 34 36 35 37 32 36 64 32 64 36 62 36 39 37 34 37 34 37 39 1b 5c
xtgettcap_Su       QUERY=b'\x1bP+q5375\x1b\\'                     RESP_len=11
    RESP_repr = b'\x1bP1+r5375\x1b\\'
    RESP_hex  = 1b 50 31 2b 72 35 33 37 35 1b 5c
xtgettcap_Smulx    QUERY=b'\x1bP+q536d756c78\x1b\\'               RESP_len=40
    RESP_repr = b'\x1bP1+r536d756c78=5c455b343a25703125646d\x1b\\'
    RESP_hex  = 1b 50 31 2b 72 35 33 36 64 37 35 36 63 37 38 3d 35 63 34 35 35 62 33 34 33 61 32 35 37 30 33 31 32 35 36 34 36 64 1b 5c
xtgettcap_Setulc   QUERY=b'\x1bP+q536574756c63\x1b\\'             RESP_len=144
    RESP_repr = b'\x1bP1+r536574756c63=5c455b35383a323a257031257b36353533367d252f25643a257031257b3235367d252f257b3235357d252625643a257031257b3235357d25262564253b6d\x1b\\'
    RESP_hex  = 1b 50 31 2b 72 35 33 36 35 37 34 37 35 36 63 36 33 3d 35 63 34 35 35 62 33 35 33 38 33 61 33 32 33 61 32 35 37 30 33 31 32 35 37 62 33 36 33 35 33 35 33 33 33 36 37 64 32 35 32 66 32 35 36 34 33 61 32 35 37 30 33 31 32 35 37 62 33 32 33 35 33 36 37 64 32 35 32 66 32 35 37 62 33 32 33 35 33 35 37 64 32 35 32 36 32 35 36 34 33 61 32 35 37 30 33 31 32 35 37 62 33 32 33 35 33 35 37 64 32 35 32 36 32 35 36 34 32 35 33 62 36 64 1b 5c
kitty_graphics     QUERY=b'\x1b_Gi=31,s=1,v=1,a=q,t=d,f=24;AAAA\x1b\\' RESP_len=12
    RESP_repr = b'\x1b_Gi=31;OK\x1b\\'
    RESP_hex  = 1b 5f 47 69 3d 33 31 3b 4f 4b 1b 5c
```

**Stability proof (byte-identical across all 3 runs — explicit diff of both the raw JSON and the formatted render):**

```bash
diff /tmp/obs/pty_responses_run1.json /tmp/obs/pty_responses_run2.json \
  && diff /tmp/obs/pty_responses_run1.json /tmp/obs/pty_responses_run3.json \
  && echo JSON_IDENTICAL
for i in 1 2 3; do python3 /tmp/obs/pty_fmt.py /tmp/obs/pty_responses_run$i.json > /tmp/obs/pty_fmt_run$i.txt; done
diff /tmp/obs/pty_fmt_run1.txt /tmp/obs/pty_fmt_run2.txt \
  && diff /tmp/obs/pty_fmt_run1.txt /tmp/obs/pty_fmt_run3.txt \
  && echo FORMATTED_IDENTICAL
```

```
JSON_IDENTICAL
FORMATTED_IDENTICAL
```

### Byte‑for‑byte verification against source

- **Primary DA.** `report_device_attributes()` at `kitty/screen.c:L2121` responds only when `mode == 0`; `case 0` writes `write_escape_code_to_child(self, ESC_CSI, "?62;c")` at `kitty/screen.c:L2125`. `ESC_CSI` prepends `\x1b[`, giving `\x1b[?62;c` = `1b 5b 3f 36 32 3b 63`. **Matches.** The `62` denotes a VT‑220‑class terminal.
- **Secondary DA.** `case '>'` writes `ESC_CSI, ">1;" xstr(PRIMARY_VERSION) ";" xstr(SECONDARY_VERSION) "c"` at `kitty/screen.c:L2128`. The macros are injected by `setup.py`: `primary_version = version[0] + 4000` (`setup.py:L605`; the `+4000` is so vim enables SGR mouse mode) and `secondary_version = version[1]` (`setup.py:L606`). With `version = (0, 35, 2)` this is `PRIMARY_VERSION = 4000`, `SECONDARY_VERSION = 35`, so the response is `\x1b[>1;4000;35c`. **Matches** the observed bytes exactly (confirming the derived macro values).
- **XTVERSION.** `screen_xtversion()` at `kitty/screen.c:L2135` writes `ESC_DCS, ">|kitty(" XT_VERSION ")"` at `kitty/screen.c:L2137`. `XT_VERSION = "0.35.2"` (`setup.py:L607`, `'.'.join(map(str, version))`). `ESC_DCS` prepends `\x1bP` and appends the ST `\x1b\\`, giving the complete response `\x1bP>|kitty(0.35.2)\x1b\\`. **Matches.**
- **Tertiary DA** (`\x1b[=c`): **no response** — `report_device_attributes()` handles only `case 0` and `case '>'` (`kitty/screen.c:L2123-L2129`); there is no tertiary handler, so kitty stays silent. Reported exactly as observed (a true negative).
- **DECRQM.** Values follow the standard: `1` = set, `2` = reset (mode recognized). Synchronized output (2026) and alt‑screen (1049) are **recognized** (value 2 = currently off); cursor visibility (25) is **set** (value 1 = visible).
- **kitty keyboard protocol** (`\x1b[?u` → `\x1b[?0u`): kitty answers with its current progressive‑enhancement flags, `0` at startup — produced by `screen_report_key_encoding_flags()` (`kitty/screen.c:L1212-L1216`). The fact that it answers at all advertises support for the kitty keyboard protocol.
- **kitty graphics protocol** (`a=q` query → `\x1b_Gi=31;OK\x1b\\`): the `OK` for image id 31 advertises graphics‑protocol support.
- **XTGETTCAP "TN".** Query cap `544e` = ASCII `"TN"` (terminal name). Response value `787465726d2d6b69747479` decodes to `xterm-kitty`; the leading `1` after `\x1bP` means "found/valid". So kitty reports `$TERM = xterm-kitty`.

- **Text sizing (CSI 14 t / 16 t / 18 t).** The CSI `t` handler in the parser dispatches on the first parameter at `kitty/vt-parser.c:L1197`: `case 14/16/18` (`L1202-L1204`) call `CALL_CSI_HANDLER1(screen_report_size, 0)` (`L1205`), while the text‑area *resize* requests `case 4/8` are explicitly rejected with `REPORT_ERROR("Escape codes to resize text area are not supported")` (`L1198-L1201`). `screen_report_size()` at `kitty/screen.c:L2142-L2167` builds the reply `snprintf(buf, sizeof(buf), "%u;%u;%ut", code, height, width)` (`L2164`) and writes it with `ESC_CSI` (`L2165`), i.e. `\x1b[code;height;widtht`:
  - `\x1b[14t` → `case 14` sets `code=4`, `width = cell_size.width * columns` (`L2149`), `height = cell_size.height * lines` (`L2150`) — with cell `9×18`, 71 cols, 22 lines this is `4;396;639` = `\x1b[4;396;639t`. **Matches** (`396 = 18×22`, `639 = 9×71`). This is the **text area in pixels**.
  - `\x1b[16t` → `case 16` sets `code=6`, `width = cell_size.width` (`L2154`), `height = cell_size.height` (`L2155`) — `6;18;9` = `\x1b[6;18;9t`. **Matches.** This is the **single cell size in pixels** (height 18, width 9) — the exact cell metrics from part (e).
  - `\x1b[18t` → `case 18` sets `code=8`, `width = columns` (`L2159`), `height = lines` (`L2160`) — `8;22;71` = `\x1b[8;22;71t`. **Matches.** This is the **screen size in cells** (22 lines, 71 columns).
- **Underline‑style / underline‑color capabilities (XTGETTCAP `Su`, `Smulx`, `Setulc`).** These are answered by `kitty/terminfo.py:get_capabilities()` (`L520`); its inner `result()` (`L523-L527`) returns `f'{valid}+r{name}'` for a value‑less capability and `f'1+r{name}={hexlify(value)}'` for a capability with a value. The query name is the hex of the terminfo name; the reply is wrapped in DCS (`\x1bP` — `\x1b\\`).
  - `Su` (query hex `5375` = `"Su"`) is a **boolean** capability listed at `kitty/terminfo.py:L56`; for booleans `result()` is called with `''` (`L544-L545`), taking the `not x` branch with `valid=1`, so the reply is `1+r5375` with **no `=value`** — `\x1bP1+r5375\x1b\\`. **Matches.** kitty thereby advertises that it supports styled/colored underlines.
  - `Smulx` (query hex `536d756c78`) is a **string** capability `r'\E[4:%p1%dm'` at `kitty/terminfo.py:L244`; `result()` takes the value branch, so the reply value is `hexlify('\E[4:%p1%dm')` = `5c455b343a25703125646d`, giving `\x1bP1+r536d756c78=5c455b343a25703125646d\x1b\\`. Decoding the value hex reproduces exactly `\E[4:%p1%dm`. **Matches** — this is the SGR that selects the underline *style* (`4:1` straight, `4:2` double, `4:3` curly).
  - `Setulc` (query hex `536574756c63`) is a **string** capability `r'\E[58:2:%p1%{65536}%/%d:%p1%{256}%/%{255}%&%d:%p1%{255}%&%d%;m'` at `kitty/terminfo.py:L288` (a comment at `L286` notes it is the parameterized equivalent of the `Su` boolean); `result()` takes the value branch. Its complete response bytes are shown in full in the table above and the output block; decoding the response's value hex byte‑for‑byte reproduces exactly that terminfo string. **Matches.** This is the SGR that selects the underline *color* (SGR 58, truecolor sub‑parameter `2`).

**Cause → effect.** Each query is a control sequence the child writes to its terminal; kitty's VT parser (`kitty/vt-parser.c`) dispatches it to the corresponding handler in `kitty/screen.c` (or, for XTGETTCAP, to `kitty/terminfo.py`), which writes the answer back onto the PTY via `write_escape_code_to_child`. The Primary/Secondary DA identify kitty as a VT‑220‑class terminal with a version‑stamped identity (the `+4000` primary version is a deliberate signal to editors like vim). XTVERSION and XTGETTCAP `TN` give kitty's name/version and `$TERM`. DECRQM lets applications discover which private modes kitty understands. The **text‑sizing** replies (CSI 14/16/18 t) expose the pixel geometry computed during startup — the cell size (`16 t` → `18×9`) is literally the cell metric from part (e), and the text area (`14 t` → `396×639`) is that cell size times the grid — so an application can recover kitty's cell dimensions directly. The **underline** capabilities (`Su`/`Smulx`/`Setulc`) advertise kitty's styled‑ and colored‑underline support and hand back the exact SGR strings to use. The kitty keyboard and graphics responses advertise kitty's own protocol extensions. Every byte is deterministic (build‑time macros + fixed handlers + fixed terminfo strings + startup‑fixed cell metrics), which is why the output is byte‑identical across runs.

---

## (g) Reconstructed init‑order sequence + key values

### Ordered subsystem map (grounded in `file:line`, confirmed by observation where noted)

1. **Native launcher** — `kitty/launcher/kitty` → `kitty/launcher/main.c`. For a source build (`FROM_SOURCE` defined, `FOR_BUNDLE` not), `run_embedded()` (`kitty/launcher/main.c:L177`) embeds CPython: `PyConfig_InitPythonConfig` (`L193`) → `PyConfig_SetBytesArgv` (`L196`) → `PyConfig_SetBytesString(executable/run_filename)` (`L198/L200`) → `Py_InitializeFromConfig` (`L211`) → `Py_RunMain` (`L216`).
2. **Python entry** — control reaches `kitty.main.main()` (`kitty/main.py:L524`) → `_main()`. (`kitty/entry_points.py` handles the `kitty +<subcommand>` dispatch such as `+launch`/`+runpy` used by the observation harnesses; the default GUI launch runs `kitty.main.main`.)
3. **CLI parse + env prep** — `kitty/main.py` `_main()`.
4. **Backend selection** — `init_glfw()` (`kitty/main.py:L96`) picks `x11` (part (b)); `init_glfw_module()` (`kitty/main.py:L90`) calls `glfw_init` and raises `SystemExit('GLFW initialization failed')` at `kitty/main.py:L92` if it fails (part (d)). GLFW itself initializes in `glfw_init` (`kitty/glfw.c`), calling `glfwInit()` at `kitty/glfw.c:L1456`.
5. **AppRunner** — `AppRunner.__call__` (`kitty/main.py:L247`): `set_scale(opts.box_drawing_scale)` (`L248`, box‑drawing scale) → `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` (`L249`) → `set_font_family(opts)` (`L251`, **resolves font files + stores descriptors; creates no font group, computes no cell metrics** — part (e)) → `_run_app(opts, args, bad_lines, talk_fd)` (`L252`).
6. **First OS window** — `Boss` calls `create_os_window()` (`kitty/boss.py:L421-L424`), passing `initial_window_size_func(size_data, self.cached_values)` (the initial‑size closure) as the first argument, then `pre_show_callback` and the window title/name/class/state.
7. **GPU bootstrap** — inside `create_os_window()` (`kitty/glfw.c:L1176-L1262`):
   - `glfwWindowHint(GLFW_VISIBLE, false)` (`L1176`); GL version/profile hints at `L1127-L1128` (request the platform minimum, 3.1 on Linux).
   - hidden **640×480 temporary window** `glfwCreateWindow(640, 480, "temp", NULL, common_context)` (`L1198`); fatal at `L1199` (cites `OPENGL_REQUIRED_VERSION`) if it fails.
   - `get_window_content_scale(temp_window, &xscale, &yscale, &xdpi, &ydpi)` (`L1200`) → **content scale + DPI** (1.0 / 96 observed).
   - `load_fonts_data(OPT(font_size), xdpi, ydpi)` (`L1202`) → `font_group_for` → `initialize_font_group` → **`calc_cell_metrics()`** at the **real DPI** (the single metric computation — part (e)).
   - compute real `width,height` via the `get_window_size` closure (`L1203`; 640×400 px).
   - real `glfwCreateWindow(width, height, title, NULL, temp_window ? temp_window : common_context)` (`L1208`); destroy temp window (`L1209`); `glfwMakeContextCurrent` (`L1211`).
   - `gl_init()` on the first window (`L1212`): GLAD load, `ARB_texture_storage` gate (`kitty/gl.c:L67`), version gate (`kitty/gl.c:L73-L74`).
   - `glEnable(GL_FRAMEBUFFER_SRGB)` (`L1214`); load shader programs (`kitty/shaders.c`).
   - `add_os_window()` registers the `OSWindow` (`L1253`).
8. **Child + capabilities answerable** — the child process is spawned and kitty can answer Device‑Attributes/DECRQM/etc. (`kitty/screen.c:L2121` and neighbors — part (f)). Content is displayed only *after* this point (out of scope).

Observationally, the merged debug log for a run orders these events as: the **`GL version string`** line (part (c)) → **`OS Window created`** → **`Child launched`**. The complete lines and their monotonic timestamps are shown in part (a); only the timestamps vary run‑to‑run while the order is stable (note the stdout/stderr ordering caveat).

### Key values (observed) — value → result → `file:line` → how observed

| Key value | Observed result | `file:line` | How observed |
|---|---|---|---|
| Backend | `x11` | `kitty/main.py:L96`; `kitty/constants.py:L207` | kitty `debug_config` "Running under: X11" |
| Context source | GLX (Mesa) | `glfw/x11_window.c:L1878` | `/proc/pid/maps`: `libGLX_mesa` present, `libEGL` absent |
| `GL_VENDOR` | `Mesa` | `kitty/shaders.c:L1256` | in‑context `glGetString` (prerender) |
| `GL_RENDERER` | `llvmpipe (LLVM 20.1.8, 256 bits)` | `kitty/shaders.c:L1258` | in‑context `glGetString` |
| `GL_VERSION` | `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2` | `kitty/gl.c:L46`, `kitty/shaders.c:L1255` | kitty `--debug-rendering` + in‑context |
| `GL_SHADING_LANGUAGE_VERSION` | `4.50` | `kitty/shaders.c:L1257` | in‑context `glGetString` |
| Required GL minimum (Linux) | `3.1` | `kitty/data-types.h:L20-L24` | source (gate `kitty/gl.c:L73-L74`) |
| `ARB_texture_storage` | present | `kitty/gl.c:L67` | in‑context extension enumeration |
| Content scale | `1.0` (x,y) | `kitty/glfw.c:L823` | `get_os_window_size()` |
| Logical DPI | `96.0` (x,y) | `kitty/glfw.c:L812` | `get_os_window_size()` |
| Initial window size | `640 × 400` px | `kitty/options/definition.py:L994,L998`; `kitty/os_window_size.py:L92` | `get_os_window_size()`; `opts.initial_window_width=(640,'px')`, `initial_window_height=(400,'px')` |
| Framebuffer size | `640 × 400` | — | `get_os_window_size()` |
| Default font size | `11.0` pts | `kitty/options/definition.py:L59` | resolved `opts.font_size` |
| Font family | DejaVu Sans Mono (+ Bold/Italic/BI) | `kitty/fonts/render.py:L173` | `--debug-font-fallback` |
| Cell width × height | `9 × 18` (DPI 96); `18 × 36` (DPI 192) | `kitty/fonts.c:L373` | `prerender_function` capture |
| Baseline | `14` (DPI 96) | `kitty/fonts.c:L398` | `prerender_function` |
| Underline position / thickness | `15` / `1` (DPI 96) | `kitty/fonts.c:L398` | `prerender_function` |
| Strikethrough position / thickness | `10` / `1` (DPI 96) | `kitty/fonts.c:L398` | `prerender_function` |
| Primary DA | `\x1b[?62;c` | `kitty/screen.c:L2125` | PTY capture |
| Secondary DA | `\x1b[>1;4000;35c` | `kitty/screen.c:L2128` | PTY capture |
| XTVERSION | `\x1bP>\|kitty(0.35.2)\x1b\\` | `kitty/screen.c:L2137` | PTY capture |
| `$TERM` (XTGETTCAP TN) | `xterm-kitty` | — | PTY capture |

### kitty's own canonical `debug_config()` (in‑process; ANSI color stripped)

The complete output of kitty's own `debug_config()` — the exact function kitty's built‑in `debug_config` action calls at `kitty/boss.py:L3064` — is captured in‑process by the watcher shown in **part (b)** (the command is given there). It is reproduced **complete** here (the ANSI SGR color escapes are removed exactly as kitty itself does for its clipboard copy — `re.sub(r'\x1b.+?m', '', output)` at `kitty/boss.py:L3065`):

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux reverse-code-generator-a9970399-dpchg 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 25.10 reverse-code-generator-a9970399-dpchg /dev/tty

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
  kitty: /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3/kitty/launcher/kitty
  base dir: /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3
  extensions dir: /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3/kitty
  system shell: /bin/bash
Loaded config overrides:
  watcher /tmp/obs/obs_watcher.py

Config options different from defaults:
watcher:
{'/tmp/obs/obs_watcher.py': '/tmp/obs/obs_watcher.py'}

Important environment variables seen by the kitty process:
	PATH                                /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3/kitty/launcher:/usr/lib/go-1.24/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :99
	LC_CTYPE                            C.UTF-8
```

The `815df1e210` in the banner is the short commit hash, matching the checkout; `Running under: X11` is kitty's own backend determination (via `kitty.debug_config.compositor_name()`); the `OpenGL:` line is the renderer. The `watcher` entries under “Loaded config overrides” / “Config options different from defaults” are the temporary observation harness itself, removed afterward so the repository is left unchanged.

**Cause → effect (why this order).** kitty must know the **cell size before the visible window is created**, because window geometry and layout depend on cell size, which depends on DPI. It cannot query real DPI without a GL‑capable window, so it creates a *hidden temporary* window first (`kitty/glfw.c:L1198`) purely to read content scale/DPI (`L1200`), computes cell metrics once at that real DPI (`L1202`), sizes and creates the real window (`L1208`), then destroys the temp. `gl_init()` runs on the *first* window only (`L1212`) because GLAD's function pointers and the version/extension gates are process‑global — they need loading exactly once. Font descriptors are stored earlier (in `set_font_family`) but metrics are deferred to this point so they are computed at the true DPI, not a guessed one — which is precisely why the "two‑phase" premise does not hold.

---

## (h) Consolidated cause → effect reasoning

- **(a) Why build‑from‑source + Xvfb/llvmpipe.** Building this exact commit lets every claim cite the code under test and exercises the real native launcher/extension. kitty mandates an OpenGL context, so a GPU‑less container needs a virtual X server (Xvfb) plus a software GL (Mesa llvmpipe) to reach the GPU path. The bare build fails only because the host `wayland-protocols` added enum values newer than this snapshot's GLFW `switch`, and kitty compiles `-Werror`; the official `--ignore-compiler-warnings` flag resolves it without editing source.
- **(b) env → backend → context source.** No `WAYLAND_DISPLAY` ⇒ `is_wayland()` False ⇒ `x11` backend (`kitty/main.py:L96`). x11 keeps GLFW's default native context API ⇒ **GLX** (`glfw/x11_window.c:L1878`), confirmed by `libGLX_mesa` loaded and `libEGL` absent.
- **(c) driver → GL strings; version gate vs extension gate.** No GPU ⇒ Mesa selects **llvmpipe** ⇒ `GL_RENDERER='llvmpipe (LLVM 20.1.8, 256 bits)'`, `GL_VENDOR='Mesa'`. llvmpipe's 4.5 core profile clears the Linux **3.1** floor (`kitty/gl.c:L74`); the `ARB_texture_storage` requirement is checked independently (`kitty/gl.c:L67`) because kitty uses immutable texture storage regardless of version. Both gates pass, so startup proceeds.
- **(d) scale → DPI → cell → geometry.** X11 `Xft.dpi` sets GLFW content scale (`glfw/x11_init.c:L462`); kitty multiplies by 96 (`kitty/glfw.c:L812`) to get logical DPI; DPI scales cell metrics. Window pixel size (default `px` unit) is DPI‑independent (`kitty/os_window_size.py:L92`), which is why HiDPI grows cells (9×18 → 18×36) but not the 640×400 window. Absent/broken display ⇒ `GLFW_PLATFORM_ERROR` at init ⇒ early abort (`kitty/main.py:L92`).
- **(e) one computation at real DPI.** `set_font_family` only discovers font files and stores descriptors, then `free_font_groups()` (`kitty/fonts.c:L1442`) — no metrics. `calc_cell_metrics` runs once, via `load_fonts_data` → `font_group_for` → `initialize_font_group` (`kitty/fonts.c:L1511`), inside `create_os_window` at the real DPI (`kitty/glfw.c:L1202`). Hence the "two‑phase" premise is falsified by direct observation.
- **(f) query → handler → response bytes.** Each control sequence is dispatched by kitty's VT parser to a handler in `kitty/screen.c` that writes a deterministic, build‑time‑stamped answer back to the child; hence identical bytes across runs, and a true silence for the unimplemented tertiary DA.
- **(g) ordering rationale.** DPI/cell metrics must precede the visible window; a hidden temp window yields real DPI; `gl_init` runs once on the first window; font descriptors are stored early but metrics deferred to the true DPI.

## Coverage matrix — every named item in the question

Per Rule R15 (coverage pass), the question is decomposed into every distinct named item it asks for — each mechanism, capability, value, and every “e.g./such as/including” example. Each row gives the **observed value**, the **`file:line`** of the function that actually performs the work, the **evidence** (which observation/part), the **sibling variants** covered, and the **causal reason**. Every value below is grounded in observed output or a code reference elsewhere in this document.

| # | Named item (as asked) | Observed value | `file:line` (does the work) | How observed / part | Sibling variants covered | Causal reason |
|---|---|---|---|---|---|---|
| 1 | Windowing backend selected | `x11` | `kitty/main.py:L96`; `kitty/constants.py:L207` (`is_wayland`) | kitty `debug_config` “Running under: X11” — part (b) | `cocoa` (macOS) and `wayland` (WAYLAND_DISPLAY set) not selected | no `WAYLAND_DISPLAY` and not Apple → x11 |
| 2 | GL context source (GLX/EGL/OSMESA) | GLX (native) | `glfw/x11_window.c:L1880` (`_glfwInitGLX`), L1912 (`_glfwCreateContextGLX`); `glfw/context.c:L60` | `/proc/pid/maps`: `libGLX_mesa` present, `libEGL` absent — part (b) | EGL (Wayland/forced) and OSMESA (headless) siblings both discussed in (b) | X11 default context API = NATIVE → GLX |
| 3 | `GL_VENDOR` | `Mesa` | `kitty/shaders.c:L1256` | in-context `glGetString` — part (c) | — | Mesa is the GL implementation |
| 4 | `GL_RENDERER` | `llvmpipe (LLVM 20.1.8, 256 bits)` | `kitty/shaders.c:L1258` | in-context `glGetString`; `glxinfo` cross-check — part (c) | hardware renderer not present | no GPU → llvmpipe software rasterizer |
| 5 | `GL_VERSION` | `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2` | `kitty/gl.c:L46` (`gl_version_string`); `kitty/shaders.c:L1255` | `--debug-rendering` + in-context; 3-run stable — part (c) | — | llvmpipe advertises 4.5 core |
| 6 | `GL_SHADING_LANGUAGE_VERSION` | `4.50` | `kitty/shaders.c:L1257` | in-context `glGetString` — part (c) | — | matches GL 4.5 core |
| 7 | Hardware vs software rasterization | **software** (`Accelerated: no`) | `kitty/shaders.c:L1258`; `glxinfo` `Accelerated: no` | `glxinfo` cross-check — part (c) | hardware-accelerated path not taken | no GPU in the container |
| 8 | `ARB_texture_storage` extension | present | `kitty/gl.c:L67` (`ARB_TEST(texture_storage)`) | in-context extension enumeration — part (c) | absent → fatal (gate) | kitty uses immutable texture storage |
| 9 | Required GL minimum | `3.1` (Linux); `3.3` (Apple) | `kitty/data-types.h:L20-L24`; gate `kitty/gl.c:L73-L74` | source; observed 4.5 ≥ 3.1 — part (c) | Apple 3.3 sibling | version gate independent of extension gate |
| 10 | Monitor content scale | `1.0` (HiDPI `2.0`) | `kitty/glfw.c:L823` (`get_window_content_scale`) | `get_os_window_size()`; HiDPI via `Xft.dpi 192`; reset → 1.0 re-verified — part (d) | HiDPI 2.0 covered | X11 `Xft.dpi`/96 → scale |
| 11 | Logical DPI derived | `96.0` (HiDPI `192.0`) | `kitty/glfw.c:L812` (`dpi_from_scale`, ×96) | `get_os_window_size()` — part (d) | HiDPI 192 covered | scale × 96 |
| 12 | Initial window / viewport size | `640 × 400` px | `kitty/options/definition.py:L994,L998`; `kitty/os_window_size.py:L92` | `get_os_window_size()` — part (d) | HiDPI keeps 640×400 (px unit is DPI-independent) | `px` unit → raw pixels |
| 13 | Font family resolution | DejaVu Sans Mono (+ Bold/Italic/BI) | `kitty/fonts/render.py:L173` (`set_font_family`), L161 (`dump_font_debug`) | `--debug-font-fallback`; `debug_config` — part (e) | bold/italic/bold-italic resolved separately | fontconfig default monospace |
| 14 | Default font size | `11.0` pts | `kitty/options/definition.py:L59` | resolved `opts.font_size` — part (e) | — | kitty built-in default |
| 15 | Cell width × height | `9 × 18` (HiDPI `18 × 36`) | `kitty/fonts.c:L373` (`calc_cell_metrics` → `cell_metrics` L375) | `prerender_function` capture — part (e) | HiDPI 18×36 covered | face metrics scaled by DPI |
| 16 | Baseline | `14` (HiDPI `28`) | `kitty/fonts.c:L375` (`cell_metrics`) | `prerender_function` — part (e) | HiDPI 28 | from FreeType face metrics |
| 17 | Underline position / thickness | `15` / `1` (HiDPI `29` / `2`) | `kitty/fonts.c:L375` (`cell_metrics`) | `prerender_function` — part (e) | HiDPI 29/2 | from face metrics |
| 18 | Strikethrough position / thickness | `10` / `1` (HiDPI `20` / `2`) | `kitty/fonts.c:L375` (`cell_metrics`) | `prerender_function` — part (e) | HiDPI 20/2 | from face metrics |
| 19 | Primary Device Attributes | `\x1b[?62;c` | `kitty/screen.c:L2125` (`report_device_attributes`, case 0) | PTY capture, 3-run byte-identical — part (f) | secondary/tertiary siblings | VT-220-class terminal |
| 20 | Secondary Device Attributes | `\x1b[>1;4000;35c` | `kitty/screen.c:L2128` (case `>`) | PTY capture, 3-run — part (f) | — | type 1; version 4000; ROM 35 |
| 21 | Tertiary Device Attributes | *(no response)* | `kitty/screen.c:L2121-L2130` (no `=` case in switch) | PTY capture (true silence) — part (f) | — | unimplemented → silence |
| 22 | Mode report (DECRQM) | `\x1b[?2026;2$y`, `\x1b[?1049;2$y`, `\x1b[?25;1$y` | `kitty/screen.c:L2203` (`report_mode_status`), response L2240 | PTY capture, 3-run — part (f) | 2026 & 1049 reset(2); 25 set(1) | reports queried-mode state |
| 23 | Text-sizing (CSI 14/16/18 t) | `\x1b[4;396;639t`, `\x1b[6;18;9t`, `\x1b[8;22;71t` | `kitty/screen.c:L2142` (`screen_report_size`); dispatch `kitty/vt-parser.c:L1205` | PTY capture, 3-run — part (f) | codes 4/6/8 all covered | cell × grid arithmetic |
| 24 | Underline caps (Su/Smulx/Setulc) | `Su` boolean; `Smulx`/`Setulc` SGR strings | `kitty/terminfo.py:L56/L244/L288`; `screen_request_capabilities` `kitty/screen.c:L2446` | PTY XTGETTCAP, 3-run — part (f) | boolean vs string cap paths both covered | terminfo capability DB |
| 25 | Graphics capability | `\x1b_Gi=31;OK\x1b\\` | `kitty/graphics.c:L768` (`OK`); dispatch `kitty/screen.c:L1047` | PTY capture, 3-run — part (f) | — | image id 31 accepted |
| 26 | kitty keyboard protocol | `\x1b[?0u` | `kitty/screen.c:L1212` (`screen_report_key_encoding_flags`) | PTY capture, 3-run — part (f) | flags 0 at startup | progressive-enhancement flags |
| 27 | XTVERSION | `\x1bP>\|kitty(0.35.2)\x1b\\` | `kitty/screen.c:L2137` (`ESC_DCS ">\|kitty(" XT_VERSION ")"`) | PTY capture, 3-run — part (f) | — | name + version identity |
| 28 | DSR (cursor position) | `\x1b[1;1R` | `kitty/screen.c:L2179` (`report_device_status`), response L2196 | PTY capture, 3-run — part (f) | — | cursor at row 1, col 1 |
| 29 | XTGETTCAP TN (terminal name) | `xterm-kitty` (`\x1bP1+r544e=787465726d2d6b69747479\x1b\\`) | `kitty/screen.c:L2446` (`screen_request_capabilities`) | PTY capture, 3-run — part (f) | — | `$TERM` name |
| 30 | Init order / subsystem sequence | 8-step map (native launcher through child) | `kitty/glfw.c:L1176-L1262`; `kitty/main.py:L247-L252`; `kitty/boss.py:L421-L424` | reconstructed + merged debug log — part (g) | — | DPI must precede the visible window |
| 31 | Key values enumerated | 22-row table | part (g) key-values table | in-process + PTY — part (g) | — | consolidated observed values |
| 32 | Window-system → GPU → cell-calc relationship | scale → DPI → cell → geometry | `kitty/glfw.c:L1200-L1208`; `kitty/fonts.c:L373`; `kitty/os_window_size.py:L92` | parts (d)/(e)/(g) | — | cell size is needed before the window is created |

---

## Appendix — reproducibility, stability, and cleanup

- **Run scale / stability.** GL strings: stable across 3 runs (kitty `--debug-rendering`) and 2 in‑context probes. Cell metrics: stable across 2 runs at DPI 96 (identical) plus 1 HiDPI run at DPI 192. Device‑Attributes / capability bytes: **byte‑identical across 3 runs** (`diff` empty).
- **Real entry point.** All values were obtained by launching the compiled `kitty/launcher/kitty` (which embeds CPython) → `kitty.main.main()`. In‑process probes (`glGetString` inside `prerender_function`; `get_os_window_size()`; `debug_config(get_options())`) run inside kitty's own live process/context and are cross‑validated against kitty's own output. `/proc/pid/maps` is an OS‑level observation of kitty's own process. No value came from a remote‑control or debug bypass.
- **Temporary scripts** (created under `/tmp`, removed after use): `obs_watcher.py`, `obs_prerender.py`, `obs_phase.py`, `obs_glstrings.py`, `obs_config.py`, `obs_pty_child.py`, and their `*_out.txt`/`*.log` captures.
- **Read‑only guarantee.** After the investigation, `git status --porcelain` shows only `blitzy/documentation/kitty_815df1e210e0.md`; all `kitty/` build products are git‑ignored, and no tracked source file was modified.

