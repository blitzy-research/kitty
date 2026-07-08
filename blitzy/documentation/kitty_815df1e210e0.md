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
| Branch | `blitzy-616b2c75-5ea1-415a-9c80-ac1890def288` (== source branch `kitty_815df1e210e0`) |
| HEAD commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
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

The build entry is `setup.py`. The Python version is gated by `check_version_info()` (`setup.py:L30`), which reads `requires-python = ">=3.8"` from `pyproject.toml:L2`. A curiosity worth flagging: the failure message at `setup.py:L44` literally reads *"calibre requires Python …"* — a copy‑paste artifact from Kovid Goyal's other project; the gate still correctly enforces kitty's `>=3.8`. Go is pinned to `go 1.22` by `go.mod:L3` (satisfied by the installed Go 1.24.4).

**Command (bare canonical build — captured to document the honest failure):**

```bash
cd /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3
export PATH=/usr/lib/go-1.24/bin:$PATH
export CI=true
python3 setup.py build
```

**Complete output (tail — the fatal error):**

```
glfw/wl_window.c: In function 'xdgToplevelHandleConfigure':
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT' not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_TOP' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM' not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
```

Exit status: **1**.

**Cause → effect.** The vendored GLFW fork's `xdgToplevelHandleConfigure()` switch in `glfw/wl_window.c:668` enumerates the `XDG_TOPLEVEL_STATE_*` values known at the time of this kitty 0.35.2 snapshot (`RESIZING`, `MAXIMIZED`, `FULLSCREEN`, `ACTIVATED`, the `TILED_*` set, and optionally `SUSPENDED`). The host's newer `wayland-protocols` (1.45) adds four `XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}` enumerators. kitty compiles with `-Werror` (and gcc's default `-Werror=switch`), so an `enum` `switch` that does not handle every enumerator is a **fatal** error. This is purely a build‑environment version drift; the kitty source is unchanged and correct for the protocol version it targets.

**Command (working canonical build — official flag, no source edits):**

```bash
cd /tmp/blitzy/kitty/blitzy-616b2c75-5ea1-415a-9c80-ac1890def288_1d39c3
export PATH=/usr/lib/go-1.24/bin:$PATH
export CI=true
python3 setup.py build --ignore-compiler-warnings
```

`--ignore-compiler-warnings` is an official `setup.py` option that sets the C `werror` flag to empty, so warnings (including the unhandled‑`switch` warning) no longer abort the build. **No source file is modified.** Exit status: **0**.

**Complete output (head + linker tail; the middle is the deterministic compile progression `[1/122] … [122/122]`):**

```
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
   ... [compile progress 7/122 .. 121/122] ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
```

**Command (verify artifacts + they are git‑ignored):**

```bash
ls -la kitty/launcher/kitty kitty/fast_data_types.so
file kitty/launcher/kitty kitty/fast_data_types.so
git check-ignore kitty/launcher/kitty kitty/fast_data_types.so
git status --porcelain
```

**Output (paraphrased for brevity where non‑essential, verbatim for the key facts):**

- `kitty/launcher/kitty` → `ELF 64-bit LSB pie executable, x86-64`
- `kitty/fast_data_types.so` → `ELF 64-bit LSB shared object, x86-64`
- `git check-ignore` prints both paths (⇒ both are git‑ignored)
- `git status --porcelain` → **empty** (clean; the build changed no tracked file)

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

These satisfy the documented minimums in `docs/build.rst:L79-L120` (e.g. HarfBuzz `>= 2.2.0`; observed 10.2.0). `.github/workflows/ci.yml` uses the same canonical `python setup.py build` invocation for CI.

### Headless launch

**Command (start virtual X server + select software GL):**

```bash
nohup Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset >/tmp/xvfb.log 2>&1 &
export DISPLAY=:99
export LIBGL_ALWAYS_SOFTWARE=1   # override: force Mesa software rasterizer (labeled)
echo "WAYLAND_DISPLAY=[$WAYLAND_DISPLAY]"
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
- `Failed to open systemd user bus …` is benign in this container (no systemd user session); it does not affect the startup path being traced.

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

**Command (kitty's own report of the running backend — in‑process, canonical):**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty -o watcher=/tmp/obs_config.py -o confirm_os_window_close=0 sh -c 'sleep 2'
# /tmp/obs_config.py calls kitty.debug_config.debug_config(get_options()) inside the live process
```

**Relevant output line (from kitty's own `debug_config`):**

```
Running under: X11
```

This is kitty's own runtime determination (via `kitty.debug_config.compositor_name()`), so it reflects the backend actually selected, not an assumption.

### GLX vs EGL vs OSMESA — mechanism and evidence

On X11, the vendored GLFW fork chooses the context API in `_glfwCreateContextGLX`/`_glfwInitEGL` dispatch at `glfw/x11_window.c:L1878-L1917`:

```c
if (ctxconfig->source == GLFW_NATIVE_CONTEXT_API) {   // L1878
    if (!_glfwInitGLX()) return GLFW_FALSE;           // L1880
    ...
    if (!_glfwCreateContextGLX(window, ctxconfig, fbconfig)) ...   // L1910/L1912
} else if (ctxconfig->source == GLFW_EGL_CONTEXT_API) {           // L1885
    if (!_glfwInitEGL()) return GLFW_FALSE;           // L1887
    ...
    if (!_glfwCreateContextEGL(window, ctxconfig, fbconfig)) ...   // L1915/L1917
}
```

The source is validated to be one of `GLFW_NATIVE_CONTEXT_API`, `GLFW_EGL_CONTEXT_API`, or `GLFW_OSMESA_CONTEXT_API` at `glfw/context.c:L60-L62`. kitty does not override the X11 context‑creation API, so it keeps GLFW's default of `GLFW_NATIVE_CONTEXT_API` → **GLX**. (Wayland, by contrast, always uses EGL.)

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
gl_version_string(void) {                                  // L42
    ...
    const char *gvs = (const char*)glGetString(GL_VERSION); // L46
    snprintf(buf, sizeof(buf), "'%s' Detected version: %d.%d", gvs, gl_major, gl_minor);
    return buf;
}
```

printed at `kitty/gl.c:L72`:

```c
if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", ..., gl_version_string());
```

**Command + complete output (3 runs, GL version line only shown for brevity — the full run output is in part (a)):**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
for i in 1 2 3; do ./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'true' 2>&1 | grep "GL version string"; done
```

```
[0.183] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[0.184] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[0.182] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

**Stable across 3 runs** (only the timestamp varies).

### Obtaining `GL_VENDOR` / `GL_RENDERER` / `GL_SHADING_LANGUAGE_VERSION` from kitty's real context

kitty does not print these three strings; the C symbols `C(GL_VENDOR)` etc. at `kitty/shaders.c:L1255-L1258` merely expose the GL *enum constants* (integers) to Python, not the strings. To read the strings **from kitty's own context** (not an external tool), I hooked `prerender_function` (`kitty/fonts/render.py:L364`), which kitty invokes from `send_prerendered_sprites()` (`kitty/fonts.c:L1450`) **while the GL context is current** (it uploads glyph sprites to the texture atlas). Inside that callback I called `glGetString` via `ctypes` — same process, same GLX context kitty created, at the exact moment kitty itself is using it. This is a labeled in‑process probe, cross‑validated against kitty's own `GL_VERSION` line above.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty +launch /tmp/obs_glstrings.py -o confirm_os_window_close=0 sh -c 'sleep 2'
# obs_glstrings.py wraps kitty.fonts.render.prerender_function; inside it (GL ctx current):
#   gl = ctypes.CDLL("libGL.so.1"); gl.glGetString.restype = c_char_p
#   print GL_VENDOR(0x1F00) GL_RENDERER(0x1F01) GL_VERSION(0x1F02) GL_SHADING_LANGUAGE_VERSION(0x8B8C)
#   enumerate extensions via glGetStringi to check GL_ARB_texture_storage
```

**Complete, unedited output (run 1; identical on run 2):**

```
GL_VENDOR='Mesa'
GL_RENDERER='llvmpipe (LLVM 20.1.8, 256 bits)'
GL_VERSION='4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2'
GL_SHADING_LANGUAGE_VERSION='4.50'
GL_NUM_EXTENSIONS=229
ARB_texture_storage_present=True
ARB_texture_storage_entry='GL_ARB_texture_storage'
```

The `GL_VERSION` here is **byte‑identical** to the value kitty prints itself (`4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2`), confirming the probe reads the same context kitty uses. Cross‑check against the environment's `glxinfo` (Vendor `Mesa`, Renderer `llvmpipe (LLVM 20.1.8, 256 bits)`, core profile `4.5`, GLSL `4.50`) matches exactly.

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
if (gl_major < OPENGL_REQUIRED_VERSION_MAJOR || (gl_major == OPENGL_REQUIRED_VERSION_MAJOR && gl_minor < OPENGL_REQUIRED_VERSION_MINOR))
    fatal("OpenGL version is %d.%d, version >= %d.%d required for kitty", ...);   // L74
```

The mandatory extension check is **independent of version**, at `kitty/gl.c:L63-L67`:

```c
#define ARB_TEST(name) \                                   // L63
    if (!GLAD_GL_ARB_##name) { \
        fatal("The OpenGL driver on this system is missing the required extension: ARB_%s", #name); }  // L65
ARB_TEST(texture_storage);                                 // L67
```

Because the observed context is 4.5 (≥ 3.1) and exposes `GL_ARB_texture_storage` (confirmed present above, among 229 extensions), both gates pass and startup proceeds — kitty launched successfully, which is itself runtime proof the gates passed.

**Cause → effect.** With no GPU, Mesa resolves GL to its Gallium **llvmpipe** software rasterizer; hence `GL_RENDERER='llvmpipe …'` and `GL_VENDOR='Mesa'`. llvmpipe advertises a 4.5 core profile — far above kitty's Linux floor of 3.1 — so the version gate (`kitty/gl.c:L74`) is satisfied. The `ARB_texture_storage` requirement is checked separately (`kitty/gl.c:L67`) because kitty uses immutable texture storage for its glyph atlas regardless of GL version; llvmpipe provides it, so the extension gate also passes. The correct way to report this environment is therefore the **observed software** renderer (llvmpipe), not a hardware value.


---

## (d) Detected display configuration

### Direct answer

- **Content scale: `1.0`** (both axes). **Logical DPI: `96.0`** (both axes).
- **Initial window size: `640 × 400` *pixels*** (framebuffer `640 × 400`). Note: the defaults are **pixels**, not cells (see below).
- **HiDPI edge:** forcing `Xft.dpi: 192` makes content scale `2.0`, logical DPI `192.0`, and cell metrics double — while the window stays `640 × 400` px.
- **Error/edge:** with no/invalid `DISPLAY`, kitty fails early with a verbatim GLFW init error and exits 1 (captured below).

### Mechanism

Logical DPI is derived from the monitor content scale by `dpi_from_scale()` at `kitty/glfw.c:L812-L819`: `dpi = content_scale × factor`, where `factor = 72.0` on Apple and **`96.0` on Linux**. The content scale itself is obtained (and clamped for invalid/NaN/absurd values → `1.0`) by `get_window_content_scale()` at `kitty/glfw.c:L823-L834`. On X11 the scale comes from the X resource `Xft.dpi`, read by GLFW's `_glfwGetSystemContentScaleX11()` at `glfw/x11_init.c:L462` (`XResourceManagerString` L483 → `XrmGetResource("Xft.dpi", …)` L494 → `atof` L497 → `scale = dpi / 96` L505‑506; default 96 if unset, L467).

The initial window size is produced by `initial_window_size_func()` (`kitty/os_window_size.py:L54`), whose inner `get_window_size(cell_width, cell_height, dpi_x, dpi_y, xscale, yscale)` (`kitty/os_window_size.py:L70`) forces `xscale = yscale = 1` on X11 (`L73-L74`) and then computes width/height. Crucially, the width/height branch depends on the *unit*: only when the unit is `'cells'` does it multiply by cell size and DPI spacing (`L88-L90`); otherwise it uses the raw pixel value (`width = w`, `L92`; `height = h`). The defaults `initial_window_width 640` (`kitty/options/definition.py:L994`) and `initial_window_height 400` (`kitty/options/definition.py:L998`) carry the **`px`** unit, so the window is 640×400 pixels.

### Observed default configuration

Captured in‑process through kitty's own `fast_data_types.get_os_window_size()` (the same accessor kitty uses), via a watcher module fired at window creation:

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty -o watcher=/tmp/obs_watcher.py -o confirm_os_window_close=0 sh -c 'sleep 2'
# obs_watcher.py on_resize -> get_os_window_size(os_window_id)
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

### HiDPI edge condition (non‑1.0 scale)

I set `Xft.dpi: 192` on the X server (the input GLFW reads for content scale), then relaunched. Method labeled: this changes the *display's* advertised DPI; kitty honors it via the GLFW mechanism cited above.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
printf 'Xft.dpi: 192\n' | xrdb -merge
./kitty/launcher/kitty -o watcher=/tmp/obs_watcher.py -o confirm_os_window_close=0 sh -c 'sleep 2'
```

**Complete, unedited output (key fields):**

```json
  "get_os_window_size": {
    "width": 640, "height": 400,
    "framebuffer_width": 640, "framebuffer_height": 400,
    "xscale": 2.0, "yscale": 2.0,
    "xdpi": 192.0, "ydpi": 192.0,
    "cell_width": 18, "cell_height": 36
  }
```

So content scale `2.0` → logical DPI `192.0` (= 2.0 × 96) → cell size doubles to `18 × 36`, while the window stays `640 × 400` px (pixel unit is DPI‑independent). After this, the override was cleared (`xrdb -load /dev/null`) and the default was re‑verified (scale `1.0`, DPI `96`, cell `9 × 18`).

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

Source of these lines: the GLFW error callback `error_callback()` at `kitty/glfw.c:L1412-L1413` (`log_error("[glfw error %d]: %s", …)`); error code `65544` is `GLFW_PLATFORM_ERROR` (`glfw/glfw3.h:L724`, `0x00010008`). `glfwInit()` is invoked at `kitty/glfw.c:L1456`; when it returns false, `init_glfw_module()` raises `SystemExit('GLFW initialization failed')` at `kitty/main.py:L92`.

This failure occurs at **GLFW initialization** (before any window is created). It is distinct from — and earlier than — the window‑creation fatal at `kitty/glfw.c:L1199` (`"Failed to create GLFW temp window! … kitty requires working OpenGL %d.%d drivers."`, citing `OPENGL_REQUIRED_VERSION`), which would fire only if init succeeded but the GL context could not be created (e.g. broken drivers). With an absent/broken display, init fails first, so the L1199 fatal is not reached (that deeper path is *inferred* from code, not triggered here).

**Cause → effect.** The chain is display → content scale → logical DPI → cell metrics → pixel geometry. On X11 the display's `Xft.dpi` sets GLFW's content scale (`glfw/x11_init.c:L462`); kitty multiplies by 96 to get logical DPI (`kitty/glfw.c:L812`); logical DPI scales FreeType face metrics into cell pixels (part (e)). Window pixel dimensions with the default `px` unit are independent of DPI (`kitty/os_window_size.py:L92`), which is why the window stays 640×400 while cells grow under HiDPI. When there is no usable display, X11 platform init raises `GLFW_PLATFORM_ERROR`, kitty logs it and aborts — the GPU path is never reached, exactly as observed.

---

## (e) Font setup + cell metrics — and a correction to the "two‑phase" premise

### Direct answer (leading with the observed correction)

**The commonly assumed "two‑phase" cell‑metric computation does not occur in kitty 0.35.2.** Cell metrics are computed **exactly once**, at the **real detected DPI (96)**, inside `create_os_window()` — **not** once at a default DPI before the window and again at the real DPI. The function often assumed to compute metrics pre‑window, `set_font_family()`, creates **no font group at all** and computes **no** metrics; it only resolves font files and stores descriptors/callbacks.

Observed values at the default DPI (96), stable across runs:

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

Driving kitty over its real PTY and injecting each query, kitty responds with the following **exact bytes** (verified byte‑for‑byte against the source strings; **identical across 3 runs**):

| Query (sent) | Response (received) | Meaning |
|---|---|---|
| Primary DA `\x1b[c` | `\x1b[?62;c` | VT‑220 class terminal |
| Secondary DA `\x1b[>c` | `\x1b[>1;4000;35c` | type 1, firmware 4000, ROM 35 |
| Tertiary DA `\x1b[=c` | *(no response)* | not implemented |
| XTVERSION `\x1b[>q` | `\x1bP>|kitty(0.35.2)\x1b\\` | name+version identity |
| DSR cursor `\x1b[6n` | `\x1b[1;1R` | cursor at row 1, col 1 |
| DECRQM 2026 `\x1b[?2026$p` | `\x1b[?2026;2$y` | synchronized output: recognized, reset |
| DECRQM 1049 `\x1b[?1049$p` | `\x1b[?1049;2$y` | alt‑screen: recognized, reset |
| DECRQM 25 `\x1b[?25$p` | `\x1b[?25;1$y` | cursor visibility: set |
| kitty keyboard `\x1b[?u` | `\x1b[?0u` | kitty keyboard protocol; flags 0 |
| kitty graphics `\x1b_Gi=31,…,a=q,…\x1b\\` | `\x1b_Gi=31;OK\x1b\\` | graphics protocol supported |
| XTGETTCAP "TN" `\x1bP+q544e\x1b\\` | `\x1bP1+r544e=787465726d2d6b69747479\x1b\\` | terminal name = `xterm-kitty` |

### Method

A tiny ephemeral harness (`/tmp/obs_pty_child.py`, removed afterward) was run **as the child of the real launcher**, so its `stdin`/`stdout` *are* kitty's PTY slave. It put the tty in raw mode, wrote each query to stdout (which flows to kitty's terminal parser), and read kitty's response back from stdin. This exercises kitty's real capability‑reporting code (`kitty/screen.c`), not a bypass.

**Command:**

```bash
export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1
./kitty/launcher/kitty -o confirm_os_window_close=0 python3 /tmp/obs_pty_child.py
```

**Complete, unedited output (run 1):**

```
primary_da         QUERY=b'\x1b[c'                                RESP_len=7
    RESP_repr = b'\x1b[?62;c'
    RESP_hex  = 1b 5b 3f 36 32 3b 63
secondary_da       QUERY=b'\x1b[>c'                               RESP_len=13
    RESP_repr = b'\x1b[>1;4000;35c'
    RESP_hex  = 1b 5b 3e 31 3b 34 30 30 30 3b 33 35 63
tertiary_da        QUERY=b'\x1b[=c'                               RESP_len=0
    RESP_repr = b''
    RESP_hex  = 
xtversion          QUERY=b'\x1b[>q'                               RESP_len=19
    RESP_repr = b'\x1bP>|kitty(0.35.2)\x1b\\'
    RESP_hex  = 1b 50 3e 7c 6b 69 74 74 79 28 30 2e 33 35 2e 32 29 1b 5c
dsr_cursor_pos     QUERY=b'\x1b[6n'                               RESP_len=6
    RESP_repr = b'\x1b[1;1R'
    RESP_hex  = 1b 5b 31 3b 31 52
decrqm_2026_sync   QUERY=b'\x1b[?2026$p'                          RESP_len=11
    RESP_repr = b'\x1b[?2026;2$y'
    RESP_hex  = 1b 5b 3f 32 30 32 36 3b 32 24 79
decrqm_1049_alt    QUERY=b'\x1b[?1049$p'                          RESP_len=11
    RESP_repr = b'\x1b[?1049;2$y'
    RESP_hex  = 1b 5b 3f 31 30 34 39 3b 32 24 79
decrqm_25_curvis   QUERY=b'\x1b[?25$p'                            RESP_len=9
    RESP_repr = b'\x1b[?25;1$y'
    RESP_hex  = 1b 5b 3f 32 35 3b 31 24 79
kitty_keyboard_u   QUERY=b'\x1b[?u'                               RESP_len=5
    RESP_repr = b'\x1b[?0u'
    RESP_hex  = 1b 5b 3f 30 75
kitty_graphics_q   QUERY=b'\x1b_Gi=31,s=1,v=1,a=q,t=d,f=24;AAAA\x1b\\' RESP_len=12
    RESP_repr = b'\x1b_Gi=31;OK\x1b\\'
    RESP_hex  = 1b 5f 47 69 3d 33 31 3b 4f 4b 1b 5c
xtgettcap_TN       QUERY=b'\x1bP+q544e\x1b\\'                     RESP_len=34
    RESP_repr = b'\x1bP1+r544e=787465726d2d6b69747479\x1b\\'
    RESP_hex  = 1b 50 31 2b 72 35 34 34 65 3d 37 38 37 34 36 35 37 32 36 64 32 64 36 62 36 39 37 34 37 34 37 39 1b 5c
```

**Stability:** runs 2 and 3 produced byte‑identical output (`diff` of the three captures is empty).

### Byte‑for‑byte verification against source

- **Primary DA.** `report_device_attributes()` at `kitty/screen.c:L2121` responds only when `mode == 0`; `case 0` writes `write_escape_code_to_child(self, ESC_CSI, "?62;c")` at `kitty/screen.c:L2125`. `ESC_CSI` prepends `\x1b[`, giving `\x1b[?62;c` = `1b 5b 3f 36 32 3b 63`. **Matches.** The `62` denotes a VT‑220‑class terminal.
- **Secondary DA.** `case '>'` writes `ESC_CSI, ">1;" xstr(PRIMARY_VERSION) ";" xstr(SECONDARY_VERSION) "c"` at `kitty/screen.c:L2128`. The macros are injected by `setup.py`: `primary_version = version[0] + 4000` (`setup.py:L605`; the `+4000` is so vim enables SGR mouse mode) and `secondary_version = version[1]` (`setup.py:L606`). With `version = (0, 35, 2)` this is `PRIMARY_VERSION = 4000`, `SECONDARY_VERSION = 35`, so the response is `\x1b[>1;4000;35c`. **Matches** the observed bytes exactly (confirming the derived macro values).
- **XTVERSION.** `screen_xtversion()` at `kitty/screen.c:L2135` writes `ESC_DCS, ">|kitty(" XT_VERSION ")"` at `kitty/screen.c:L2137`. `XT_VERSION = "0.35.2"` (`setup.py:L607`, `'.'.join(map(str, version))`). `ESC_DCS` wraps in `\x1bP … \x1b\\`, giving `\x1bP>|kitty(0.35.2)\x1b\\`. **Matches.**
- **Tertiary DA** (`\x1b[=c`): **no response** — `report_device_attributes()` handles only `case 0` and `case '>'` (`kitty/screen.c:L2123-L2129`); there is no tertiary handler, so kitty stays silent. Reported exactly as observed (a true negative).
- **DECRQM.** Values follow the standard: `1` = set, `2` = reset (mode recognized). Synchronized output (2026) and alt‑screen (1049) are **recognized** (value 2 = currently off); cursor visibility (25) is **set** (value 1 = visible).
- **kitty keyboard protocol** (`\x1b[?u` → `\x1b[?0u`): kitty answers with its current progressive‑enhancement flags, `0` at startup — produced by `screen_report_key_encoding_flags()` (`kitty/screen.c:L1212-L1216`). The fact that it answers at all advertises support for the kitty keyboard protocol.
- **kitty graphics protocol** (`a=q` query → `\x1b_Gi=31;OK\x1b\\`): the `OK` for image id 31 advertises graphics‑protocol support.
- **XTGETTCAP "TN".** Query cap `544e` = ASCII `"TN"` (terminal name). Response value `787465726d2d6b69747479` decodes to `xterm-kitty`; the leading `1` after `\x1bP` means "found/valid". So kitty reports `$TERM = xterm-kitty`.

**Cause → effect.** Each query is a control sequence the child writes to its terminal; kitty's VT parser dispatches it to the corresponding handler in `kitty/screen.c`, which writes the answer back onto the PTY via `write_escape_code_to_child`. The Primary/Secondary DA identify kitty as a VT‑220‑class terminal with a version‑stamped identity (the `+4000` primary version is a deliberate signal to editors like vim). XTVERSION and XTGETTCAP give kitty's name/version and `$TERM`. DECRQM lets applications discover which private modes kitty understands. The kitty keyboard and graphics responses advertise kitty's own protocol extensions. Every byte is deterministic (build‑time macros + fixed handlers), which is why the output is byte‑identical across runs.

---

## (g) Reconstructed init‑order sequence + key values

### Ordered subsystem map (grounded in `file:line`, confirmed by observation where noted)

1. **Native launcher** — `kitty/launcher/kitty` → `kitty/launcher/main.c`. For a source build (`FROM_SOURCE` defined, `FOR_BUNDLE` not), `run_embedded()` (`kitty/launcher/main.c:L177`) embeds CPython: `PyConfig_InitPythonConfig` (`L193`) → `PyConfig_SetBytesArgv` (`L196`) → `PyConfig_SetBytesString(executable/run_filename)` (`L198/L200`) → `Py_InitializeFromConfig` (`L211`) → `Py_RunMain` (`L216`).
2. **Python entry** — control reaches `kitty.main.main()` (`kitty/main.py:L524`) → `_main()`. (`kitty/entry_points.py` handles the `kitty +…` subcommands such as `+launch`/`+runpy` used by the observation harnesses; the default GUI launch runs `kitty.main.main`.)
3. **CLI parse + env prep** — `kitty/main.py` `_main()`.
4. **Backend selection** — `init_glfw()` (`kitty/main.py:L96`) picks `x11` (part (b)); `init_glfw_module()` (`kitty/main.py:L90`) calls `glfw_init` and raises `SystemExit('GLFW initialization failed')` at `kitty/main.py:L92` if it fails (part (d)). GLFW itself initializes in `glfw_init` (`kitty/glfw.c`), calling `glfwInit()` at `kitty/glfw.c:L1456`.
5. **AppRunner** — `AppRunner.__call__` (`kitty/main.py:L247`): `set_scale(opts.box_drawing_scale)` (`L248`, box‑drawing scale) → `set_options(...)` (`L249`) → `set_font_family(opts)` (`L251`, **resolves font files + stores descriptors; creates no font group, computes no cell metrics** — part (e)) → `_run_app(...)` (`L252`).
6. **First OS window** — `Boss` calls `create_os_window(initial_window_size_func(...), …)` (`kitty/boss.py:L421-L422`).
7. **GPU bootstrap** — inside `create_os_window()` (`kitty/glfw.c:L1176-L1262`):
   - `glfwWindowHint(GLFW_VISIBLE, false)` (`L1176`); GL version/profile hints at `L1127-L1128` (request the platform minimum, 3.1 on Linux).
   - hidden **640×480 temporary window** `glfwCreateWindow(640, 480, "temp", …)` (`L1198`); fatal at `L1199` (cites `OPENGL_REQUIRED_VERSION`) if it fails.
   - `get_window_content_scale(temp_window, …)` (`L1200`) → **content scale + DPI** (1.0 / 96 observed).
   - `load_fonts_data(OPT(font_size), xdpi, ydpi)` (`L1202`) → `font_group_for` → `initialize_font_group` → **`calc_cell_metrics()`** at the **real DPI** (the single metric computation — part (e)).
   - compute real `width,height` via the `get_window_size` closure (`L1203`; 640×400 px).
   - real `glfwCreateWindow(width, height, …)` (`L1208`); destroy temp window (`L1209`); `glfwMakeContextCurrent` (`L1211`).
   - `gl_init()` on the first window (`L1212`): GLAD load, `ARB_texture_storage` gate (`kitty/gl.c:L67`), version gate (`kitty/gl.c:L73-L74`).
   - `glEnable(GL_FRAMEBUFFER_SRGB)` (`L1214`); load shader programs (`kitty/shaders.c`).
   - `add_os_window()` registers the `OSWindow` (`L1253`).
8. **Child + capabilities answerable** — the child process is spawned and kitty can answer Device‑Attributes/DECRQM/etc. (`kitty/screen.c:L2121` and neighbors — part (f)). Content is displayed only *after* this point (out of scope).

Observationally, the merged debug log for a run is: `GL version string … (t=0.183)` → `OS Window created (t=0.225)` → `Child launched (t=0.239)` (see part (a); note the stdout/stderr ordering caveat).

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
| Initial window size | `640 × 400` px | `kitty/options/definition.py:L994,L998`; `kitty/os_window_size.py:L92` | `get_os_window_size()`; `opts.*=(…, 'px')` |
| Framebuffer size | `640 × 400` | — | `get_os_window_size()` |
| Default font size | `11.0` pts | `kitty/options/definition.py:L59` | resolved `opts.font_size` |
| Font family | DejaVu Sans Mono (+ Bold/Italic/BI) | `kitty/fonts/render.py:L173` | `--debug-font-fallback` |
| Cell width × height | `9 × 18` (DPI 96); `18 × 36` (DPI 192) | `kitty/fonts.c:L373` | `prerender_function` capture |
| Baseline | `14` (DPI 96) | `kitty/fonts.c:L398` | `prerender_function` |
| Underline position / thickness | `15` / `1` (DPI 96) | `kitty/fonts.c:L398` | `prerender_function` |
| Strikethrough position / thickness | `10` / `1` (DPI 96) | `kitty/fonts.c:L398` | `prerender_function` |
| Primary DA | `\x1b[?62;c` | `kitty/screen.c:L2125` | PTY capture |
| Secondary DA | `\x1b[>1;4000;35c` | `kitty/screen.c:L2128` | PTY capture |
| XTVERSION | `\x1bP>|kitty(0.35.2)\x1b\\` | `kitty/screen.c:L2137` | PTY capture |
| `$TERM` (XTGETTCAP TN) | `xterm-kitty` | — | PTY capture |

### kitty's own canonical `debug_config()` (in‑process; ANSI color stripped for readability)

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux ... x86_64
Ubuntu 25.10 ...
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
```

(The raw output contains kitty's own ANSI SGR color codes; they are stripped here for readability. The `815df1e210` in the banner is the short commit hash, matching the checkout.)

**Cause → effect (why this order).** kitty must know the **cell size before the visible window is created**, because window geometry and layout depend on cell size, which depends on DPI. It cannot query real DPI without a GL‑capable window, so it creates a *hidden temporary* window first (`kitty/glfw.c:L1198`) purely to read content scale/DPI (`L1200`), computes cell metrics once at that real DPI (`L1202`), sizes and creates the real window (`L1208`), then destroys the temp. `gl_init()` runs on the *first* window only (`L1212`) because GLAD's function pointers and the version/extension gates are process‑global — they need loading exactly once. Font descriptors are stored earlier (in `set_font_family`) but metrics are deferred to this point so they are computed at the true DPI, not a guessed one — which is precisely why the "two‑phase" premise does not hold.

---

## (h) Consolidated cause → effect reasoning

- **(a) Why build‑from‑source + Xvfb/llvmpipe.** Building this exact commit lets every claim cite the code under test and exercises the real native launcher/extension. kitty mandates an OpenGL context, so a GPU‑less container needs a virtual X server (Xvfb) plus a software GL (Mesa llvmpipe) to reach the GPU path. The bare build fails only because the host `wayland-protocols` added enum values newer than this snapshot's GLFW `switch`, and kitty compiles `-Werror`; the official `--ignore-compiler-warnings` flag resolves it without editing source.
- **(b) env → backend → context source.** No `WAYLAND_DISPLAY` ⇒ `is_wayland()` False ⇒ `x11` backend (`kitty/main.py:L96`). x11 keeps GLFW's default native context API ⇒ **GLX** (`glfw/x11_window.c:L1878`), confirmed by `libGLX_mesa` loaded and `libEGL` absent.
- **(c) driver → GL strings; version gate vs extension gate.** No GPU ⇒ Mesa selects **llvmpipe** ⇒ `GL_RENDERER='llvmpipe …'`, `GL_VENDOR='Mesa'`. llvmpipe's 4.5 core profile clears the Linux **3.1** floor (`kitty/gl.c:L74`); the `ARB_texture_storage` requirement is checked independently (`kitty/gl.c:L67`) because kitty uses immutable texture storage regardless of version. Both gates pass, so startup proceeds.
- **(d) scale → DPI → cell → geometry.** X11 `Xft.dpi` sets GLFW content scale (`glfw/x11_init.c:L462`); kitty multiplies by 96 (`kitty/glfw.c:L812`) to get logical DPI; DPI scales cell metrics. Window pixel size (default `px` unit) is DPI‑independent (`kitty/os_window_size.py:L92`), which is why HiDPI grows cells (9×18 → 18×36) but not the 640×400 window. Absent/broken display ⇒ `GLFW_PLATFORM_ERROR` at init ⇒ early abort (`kitty/main.py:L92`).
- **(e) one computation at real DPI.** `set_font_family` only discovers font files and stores descriptors, then `free_font_groups()` (`kitty/fonts.c:L1442`) — no metrics. `calc_cell_metrics` runs once, via `load_fonts_data` → `font_group_for` → `initialize_font_group` (`kitty/fonts.c:L1511`), inside `create_os_window` at the real DPI (`kitty/glfw.c:L1202`). Hence the "two‑phase" premise is falsified by direct observation.
- **(f) query → handler → response bytes.** Each control sequence is dispatched by kitty's VT parser to a handler in `kitty/screen.c` that writes a deterministic, build‑time‑stamped answer back to the child; hence identical bytes across runs, and a true silence for the unimplemented tertiary DA.
- **(g) ordering rationale.** DPI/cell metrics must precede the visible window; a hidden temp window yields real DPI; `gl_init` runs once on the first window; font descriptors are stored early but metrics deferred to the true DPI.

---

## Appendix — reproducibility, stability, and cleanup

- **Run scale / stability.** GL strings: stable across 3 runs (kitty `--debug-rendering`) and 2 in‑context probes. Cell metrics: stable across 2 runs at DPI 96 (identical) plus 1 HiDPI run at DPI 192. Device‑Attributes / capability bytes: **byte‑identical across 3 runs** (`diff` empty).
- **Real entry point.** All values were obtained by launching the compiled `kitty/launcher/kitty` (which embeds CPython) → `kitty.main.main()`. In‑process probes (`glGetString` inside `prerender_function`; `get_os_window_size()`; `debug_config(get_options())`) run inside kitty's own live process/context and are cross‑validated against kitty's own output. `/proc/pid/maps` is an OS‑level observation of kitty's own process. No value came from a remote‑control or debug bypass.
- **Temporary scripts** (created under `/tmp`, removed after use): `obs_watcher.py`, `obs_prerender.py`, `obs_phase.py`, `obs_glstrings.py`, `obs_config.py`, `obs_pty_child.py`, and their `*_out.txt`/`*.log` captures.
- **Read‑only guarantee.** After the investigation, `git status --porcelain` shows only `blitzy/documentation/kitty_815df1e210e0.md`; all `kitty/` build products are git‑ignored, and no tracked source file was modified.

