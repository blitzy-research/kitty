# Kitty Terminal Emulator — Runtime Architecture Investigation

## Metadata

| Field | Value |
|---|---|
| **Kitty Version** | 0.35.2 (`kitty/constants.py` line 25: `version: Version = Version(0, 35, 2)`) |
| **Source Branch** | `kitty_815df1e210e0` |
| **VCS Revision** | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| **Investigation Date** | 2025 (sandboxed CI environment) |
| **Python Requirement** | ≥ 3.8 (`pyproject.toml` line 2: `requires-python = ">=3.8"`) |
| **Go Version** | 1.22 (`go.mod` line 3: `go 1.22`) |
| **C Standard** | C11 with `-std=c11` (`setup.py` compiler flags) |
| **Investigation Environment** | Ubuntu 24.04.4 LTS, kernel 6.6.113+, x86-64, headless (no display server) |

> **Evidence Classification Convention**: Throughout this document, observations are marked as:
> - **[OBSERVED]** — Actual command output captured during this investigation
> - **[SOURCE-INFORMED]** — Conclusion drawn from source code analysis, corroborated where possible by build artifacts
> - **[BLOCKED]** — Tool or command that could not execute, with error and fallback documented

---

## Table of Contents

1. [Build and Launch Observations](#section-1--build-and-launch-observations)
2. [Loaded Modules and Libraries](#section-2--loaded-modules-and-libraries)
3. [Thread Activity: Idle vs. Stress](#section-3--thread-activity-idle-vs-stress)
4. [Remote Control Interface Queries](#section-4--remote-control-interface-queries)
5. [Kitten Process Relationship](#section-5--kitten-process-relationship)
6. [Kitten Binary Inspection](#section-6--kitten-binary-inspection)
7. [Symbol and Stack Snapshots](#section-7--symbol-and-stack-snapshots)
8. [Language Responsibility Inference](#section-8--language-responsibility-inference)
9. [Two Falsified Interpretations](#section-9--two-falsified-interpretations)
10. [Portability vs. Performance Tradeoff](#section-10--portability-vs-performance-tradeoff)
11. [Full Command Transcripts (Appendix)](#section-11--full-command-transcripts-appendix)

---

## Section 1 — Build and Launch Observations

### 1.1 Build System Architecture

The Kitty build is orchestrated by a single `setup.py` at the repository root. This file performs three distinct compilation phases in a single invocation:

1. **C Extension Compilation** — Compiles 49 `.c` source files from `kitty/` into a single shared object `kitty/fast_data_types.so`, plus two separate GLFW backend modules (`kitty/glfw-x11.so`, `kitty/glfw-wayland.so`).
2. **Go Binary Compilation** — The `build_static_kittens()` function (setup.py lines 1130–1165) invokes `go build -v` targeting `tools/cmd` as the source directory, producing the `kitten` binary placed adjacent to the `kitty` launcher.
3. **C Launcher Compilation** — Compiles `kitty/launcher/main.c` and supporting files into the `kitty` launcher binary, which embeds CPython.

**[OBSERVED]** The build command and its results:

```bash
$ python3 setup.py build --verbose --ignore-compiler-warnings
# (Full build output captured during environment setup — see Appendix A.1)
```

The build produced the following artifacts:

```bash
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so kitty/glfw-x11.so kitty/glfw-wayland.so
-rwxr-xr-x 1 root root  1541408 kitty/fast_data_types.so    # 1.5 MB — monolithic C extension
-rwxr-xr-x 1 root root   464264 kitty/glfw-wayland.so       # 454 KB — Wayland GLFW backend
-rwxr-xr-x 1 root root   377408 kitty/glfw-x11.so           # 369 KB — X11 GLFW backend
-rwxr-xr-x 1 root root 15761668 kitty/launcher/kitten       # 15 MB  — Go static binary
-rwxr-xr-x 1 root root    36224 kitty/launcher/kitty        # 36 KB  — C launcher (thin)
```

> **Key Observation**: The size difference is striking. The `kitty` launcher is only 36 KB — it is a thin C binary whose sole job is to bootstrap CPython. The actual application logic lives in the 1.5 MB `fast_data_types.so` (C) and the Python source files. Meanwhile, the `kitten` binary at 15 MB is a self-contained Go executable carrying its own runtime, garbage collector, and all Go dependencies statically linked.

### 1.2 Binary Format Verification

**[OBSERVED]** Binary type identification:

```bash
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  BuildID[sha1]=e8c64dd649a7e0353b70a45f1defd979b45f8b79,
  for GNU/Linux 3.2.0, not stripped

$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  Go BuildID=hqq4-U2LKlixbsjwYo2Y/n8c9tVmrH955DZP0gtLh/TlErkqS1Lkyxjr-Onw6b/O2yALBSe3QkGVIYlcExQ,
  stripped
```

**Analysis:**

- The `kitty` launcher is a **dynamically-linked PIE executable** (Position Independent Executable). It links against `libpython3.12.so` because its primary function is to initialize and run the CPython interpreter.
- The `kitten` binary is a **Go executable** with a `Go BuildID` embedded in the ELF header. It is listed as "dynamically linked" — on this build, the Go compiler linked against `libc.so.6` (CGO was enabled or the Go toolchain chose external linking). The binary is **stripped** (`-s -w` ldflags in setup.py line 1157), explaining the absence of debug symbols.

### 1.3 Native Launcher Role

The C launcher (`kitty/launcher/main.c`) is the process entry point. Its responsibilities are minimal but critical:

**[SOURCE-INFORMED]** From `kitty/launcher/main.c`:

- **Lines 8–23**: Includes `<Python.h>` along with platform headers. The `RunData` struct (lines 46–50) holds `exe`, `exe_dir`, `lc_ctype`, and `lib_dir` — the minimal data needed to bootstrap CPython.
- **Lines 52–77**: The `set_kitty_run_data()` function creates a Python dictionary (`sys.kitty_run_data`) containing `bundle_exe_dir`, and optionally `from_source` and `lc_ctype_before_python`. This dictionary is how the Python layer discovers the binary locations.
- **Lines ~200–220**: The launcher calls `Py_InitializeFromConfig(&config)` followed by `Py_RunMain()`, transferring control entirely to Python. The launcher's C code never participates in the rendering loop — it exits once Python takes over.

```c
// kitty/launcher/main.c (paraphrased from lines 200-220)
status = Py_InitializeFromConfig(&config);
if (PyStatus_Exception(status)) goto fail;
if (!set_kitty_run_data(run_data, from_source, NULL)) return 1;
PySys_SetObject("frozen", Py_False);
return Py_RunMain();
```

> **Key Insight**: The native C launcher is NOT the performance-critical C layer. It is a ~36 KB bootstrap shim. The actual C hot-path code lives entirely inside `fast_data_types.so`, loaded as a Python extension module.

### 1.4 Python Startup Sequence

**[SOURCE-INFORMED]** From `kitty/main.py` lines 441–521, the `_main()` function executes:

```python
# kitty/main.py _main() — startup sequence (paraphrased)
running_in_kitty(True)                          # Mark this process as kitty
cli_opts, rest = parse_args(args=args, ...)     # Parse command-line options
opts = create_opts(cli_opts, ...)               # Load configuration
setup_environment(opts, cli_opts)               # Set environment variables
set_locale()                                    # Configure locale
sys.setswitchinterval(1000.0)                   # ← CRITICAL: single Python thread
mask_kitty_signals_process_wide()               # Block signals in non-main threads
init_glfw(opts, ...)                            # Initialize GLFW (C call via fast_data_types)
run_app(opts, cli_opts, bad_lines, talk_fd)     # Create Boss → ChildMonitor → main_loop()
glfw_terminate()                                # Cleanup
```

The call `sys.setswitchinterval(1000.0)` at line 504 is **pivotal evidence**: it sets the Python GIL switch interval to 1000 seconds — effectively disabling thread switching. The comment reads: `"we have only a single python thread"`. This confirms that **Python never runs multiple concurrent threads in kitty**. All threading is done in C via `pthread_create`.

### 1.5 Launch Attempt in Headless Environment

**[OBSERVED]** Attempting to launch kitty in the sandboxed environment:

```bash
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ kitty/launcher/kitty --listen-on unix:/tmp/kitty-test.sock
[0.059] [glfw error 65544]: X11: The DISPLAY environment variable is missing
GLFW initialization failed
```

**Analysis**: The `--version` flag succeeds because it only requires the Python layer (entry point dispatch). The full launch fails at GLFW initialization because:

1. The `init_glfw()` call in `kitty/main.py` line 514 calls into C (`fast_data_types`) to initialize the GLFW windowing library
2. GLFW's X11 backend (`glfw/x11_init.c`) requires a valid `DISPLAY` environment variable pointing to an X11 server
3. The headless CI environment has no display server

> **This failure is itself evidence**: It proves that the C layer's GLFW initialization is a mandatory gateway — without a display server, the rendering pipeline cannot start. This is a direct consequence of the C layer owning the platform windowing and GPU rendering responsibilities.

---

## Section 2 — Loaded Modules and Libraries

### 2.1 The `fast_data_types` C Extension Module

The central artifact connecting Python and C is `kitty/fast_data_types.so` — a single shared object that bundles the entire C engine. This module is defined in `kitty/data-types.c`, which includes:

**[SOURCE-INFORMED]** From `kitty/data-types.c` lines 1–37:

```c
#include "data-types.h"
#include "charsets.h"
#include "base64.h"
#include "control-codes.h"
#include "wcwidth-std.h"
#include "wcswidth.h"
#include "modes.h"
#include "monotonic.h"
```

> *Note: The above excerpt shows only the project-internal includes. System headers (`sys/socket.h`, `sys/types.h`, `unistd.h`) and internal utility headers (`cleanup.h`, `safe-wrappers.h`) are omitted for focus.*

These are just the top-level includes. The build system (`setup.py`) compiles **49 `.c` files** from the `kitty/` directory into this single `.so`:

| C Source File | Purpose |
|---|---|
| `child-monitor.c` | Three-thread architecture, main loop, rendering dispatch |
| `vt-parser.c` | VT escape sequence state machine |
| `screen.c` | Screen model (cells, cursor, attributes) |
| `line.c`, `line-buf.c` | Line buffer management |
| `shaders.c` | OpenGL shader program management, sprite maps |
| `gl.c`, `gl-wrapper.c` | GLAD OpenGL function loader |
| `freetype.c` | FreeType glyph rasterization |
| `fontconfig.c` | Fontconfig font discovery (Linux) |
| `fonts.c` | Font subsystem orchestration with HarfBuzz |
| `glyph-cache.c` | GPU texture atlas for glyph caching |
| `graphics.c` | Graphics protocol (inline images) |
| `colors.c` | Color management, sRGB conversion |
| `history.c` | Scrollback ring buffer |
| `keys.c`, `key_encoding.c` | Input handling and key encoding |
| `mouse.c` | Mouse event processing |
| `png-reader.c` | PNG decoding via libpng |
| `crypto.c` | AES-256-GCM and X25519 via OpenSSL |
| `simd-string.c`, `simd-string-128.c`, `simd-string-256.c` | SIMD-accelerated string scanning |
| `state.c` | Global state structure |
| `kittens.c` | Native kitten response parser |
| `glfw.c`, `glfw-wrapper.c` | GLFW integration layer |
| _(and 26 more)_ | Character sets, cursor, disk cache, desktop, hyperlinks, etc. |

**[OBSERVED]** The Python module exposes 581 public attributes:

```bash
$ python3 -c "
import sys; sys.path.insert(0, '.')
import kitty.fast_data_types as fdt
attrs = [a for a in dir(fdt) if not a.startswith('_')]
print(f'Total public attributes: {len(attrs)}')
"
Total public attributes: 581
```

Key attribute categories include:

```
Shader Programs: BGIMAGE_PROGRAM, BORDERS_PROGRAM, CELL_BG_PROGRAM, CELL_FG_PROGRAM,
                 CELL_PROGRAM, CELL_SPECIAL_PROGRAM, GRAPHICS_ALPHA_MASK_PROGRAM,
                 GRAPHICS_PREMULT_PROGRAM, GRAPHICS_PROGRAM, TINT_PROGRAM

Thread/Monitor:  ChildMonitor

Crypto:          AES256GCMDecrypt, AES256GCMEncrypt, EllipticCurveKey

Screen:          Screen

GLFW Constants:  GLFW_ACCUM_ALPHA_BITS, GLFW_CLIENT_API, ... (hundreds)
```

### 2.2 Shared Library Dependencies

#### 2.2.1 `kitty` Launcher Dependencies

**[OBSERVED]** Dynamic library dependencies of the launcher:

```bash
$ ldd kitty/launcher/kitty
  linux-vdso.so.1
  libpython3.12.so.1.0 => /lib/x86_64-linux-gnu/libpython3.12.so.1.0
  libc.so.6             => /lib/x86_64-linux-gnu/libc.so.6
  libm.so.6             => /lib/x86_64-linux-gnu/libm.so.6
  libz.so.1             => /lib/x86_64-linux-gnu/libz.so.1
  libexpat.so.1         => /lib/x86_64-linux-gnu/libexpat.so.1
  /lib64/ld-linux-x86-64.so.2
```

**Analysis**: The launcher links **only** against libpython and standard system libraries. It has zero rendering libraries — confirming the launcher is a pure CPython bootstrapper.

#### 2.2.2 `fast_data_types.so` Dependencies

**[OBSERVED]** This is where the rendering-adjacent libraries appear:

```bash
$ ldd kitty/fast_data_types.so
  libm.so.6             => /lib/x86_64-linux-gnu/libm.so.6
  libpython3.12.so.1.0  => /lib/x86_64-linux-gnu/libpython3.12.so.1.0
  libharfbuzz.so.0       => /lib/x86_64-linux-gnu/libharfbuzz.so.0
  libpng16.so.16         => /lib/x86_64-linux-gnu/libpng16.so.16
  liblcms2.so.2          => /lib/x86_64-linux-gnu/liblcms2.so.2
  libcrypto.so.3         => /lib/x86_64-linux-gnu/libcrypto.so.3
  libz.so.1              => /lib/x86_64-linux-gnu/libz.so.1
  libc.so.6              => /lib/x86_64-linux-gnu/libc.so.6
  libexpat.so.1          => /lib/x86_64-linux-gnu/libexpat.so.1
  libfreetype.so.6       => /lib/x86_64-linux-gnu/libfreetype.so.6
  libglib-2.0.so.0       => /lib/x86_64-linux-gnu/libglib-2.0.so.0
  libgraphite2.so.3      => /lib/x86_64-linux-gnu/libgraphite2.so.3
  libbz2.so.1.0          => /lib/x86_64-linux-gnu/libbz2.so.1.0
  libbrotlidec.so.1      => /lib/x86_64-linux-gnu/libbrotlidec.so.1
  libpcre2-8.so.0        => /lib/x86_64-linux-gnu/libpcre2-8.so.0
  libbrotlicommon.so.1   => /lib/x86_64-linux-gnu/libbrotlicommon.so.1
```

**[OBSERVED]** Confirmed via `readelf -d`:

```bash
$ readelf -d kitty/fast_data_types.so | grep NEEDED
  (NEEDED)  Shared library: [libm.so.6]
  (NEEDED)  Shared library: [libpython3.12.so.1.0]
  (NEEDED)  Shared library: [libharfbuzz.so.0]
  (NEEDED)  Shared library: [libpng16.so.16]
  (NEEDED)  Shared library: [liblcms2.so.2]
  (NEEDED)  Shared library: [libcrypto.so.3]
  (NEEDED)  Shared library: [libz.so.1]
  (NEEDED)  Shared library: [libc.so.6]
```

#### 2.2.3 Library-to-Source Tracing

Each linked library maps to specific C source files in the `fast_data_types` module:

| Library | Version | C Source File(s) | Purpose | Symbol Evidence |
|---|---|---|---|---|
| **libfreetype.so.6** | FreeType 26.1.20 | `kitty/freetype.c` | Glyph rasterization, bitmap rendering | `FT_Init_FreeType`, `FT_Load_Glyph`, `FT_Render_Glyph` |
| **libharfbuzz.so.0** | HarfBuzz 8.3.0 | `kitty/fonts.c` | OpenType text shaping, ligatures | `hb_buffer_create`, `hb_shape`, `hb_ft_font_create` |
| **libpng16.so.16** | libpng 1.6.43 | `kitty/png-reader.c` | PNG image decoding | `png_create_read_struct`, `png_read_image` |
| **liblcms2.so.2** | lcms2 2.14 | `kitty/colors.c` | ICC color profile management | `cmsCreateTransform`, `cmsCreate_sRGBProfile` |
| **libcrypto.so.3** | OpenSSL 3.0.13 | `kitty/crypto.c` | X25519 key exchange, AES-256-GCM encryption | `EVP_EncryptInit_ex`, `EVP_PKEY_derive` |
| **libpython3.12.so** | Python 3.12.3 | All `.c` files | CPython embedding and Python C API | `PyInit_fast_data_types` (module entry) |
| **libz.so.1** | zlib | Various | Compression support | Standard zlib symbols |

> **Security Patch Status — Ubuntu Backported Fixes**: The library versions reported above
> via `pkg-config --modversion` reflect *upstream* base version numbers. On Ubuntu 24.04,
> security patches are backported into stable upstream versions without incrementing the
> upstream version number. Verification via `dpkg -l` and `apt-get changelog` reveals the
> actual patch levels of the runtime libraries linked into `fast_data_types.so`:
>
> - **libssl3t64 3.0.13-0ubuntu3.9** (libcrypto): Contains backported patches for
>   CVE-2025-15467 (stack buffer overflow in CMS AuthEnvelopedData processing) and 20+
>   additional CVEs. Despite `pkg-config` reporting `3.0.13`, the Ubuntu package carries
>   all known security fixes through the `-0ubuntu3.9` patch series.
> - **libpng16-16t64 1.6.43-5ubuntu0.5**: Contains backported patches for CVE-2025-64505,
>   CVE-2025-64506, CVE-2025-64720, CVE-2025-65018, CVE-2025-66293, and 5 additional CVEs
>   across 5 Ubuntu security updates (0ubuntu0.1 through 0ubuntu0.5). All heap overflow and
>   out-of-bounds read vulnerabilities reported against libpng before 1.6.51 are patched.
> - **liblcms2-2 2.14-2build1**: Not affected by CVE-2025-29070 (which targets lcms2 v2.16;
>   the installed v2.14 predates the vulnerable code path).
>
> Thus, despite `pkg-config` reporting upstream versions that appear vulnerable to version-
> number-based scanning, the actual runtime libraries in this build environment carry all
> known security fixes as of the system's package repository state. This is a standard
> consequence of Ubuntu's long-term-support backporting practice and must be accounted for
> when performing dependency vulnerability assessments.

**[OBSERVED]** Symbol-level proof via `nm -D`:

```bash
$ nm -D kitty/fast_data_types.so | grep 'hb_'
  U hb_buffer_add_utf32
  U hb_buffer_create
  U hb_buffer_destroy
  U hb_buffer_get_glyph_infos
  U hb_buffer_get_glyph_positions
  U hb_shape
  U hb_ft_font_create
  # ... (22 HarfBuzz symbols total)

$ nm -D kitty/fast_data_types.so | grep 'FT_'
  U FT_Init_FreeType
  U FT_Done_FreeType
  U FT_Load_Glyph
  U FT_Render_Glyph
  U FT_Bitmap_Convert
  # ... (FreeType symbols)

$ nm -D kitty/fast_data_types.so | grep 'png_'
  U png_create_read_struct
  U png_read_image
  U png_get_image_width
  # ... (25 libpng symbols total)
```

#### 2.2.4 GLFW Backend Dependencies

**[OBSERVED]** The GLFW X11 backend links against platform-specific windowing libraries:

```bash
$ readelf -d kitty/glfw-x11.so | grep NEEDED
  (NEEDED)  Shared library: [libm.so.6]
  (NEEDED)  Shared library: [libX11.so.6]
  (NEEDED)  Shared library: [libXcursor.so.1]
  (NEEDED)  Shared library: [libxkbcommon.so.0]
  (NEEDED)  Shared library: [libxkbcommon-x11.so.0]
  (NEEDED)  Shared library: [libX11-xcb.so.1]
  (NEEDED)  Shared library: [libdbus-1.so.3]
  (NEEDED)  Shared library: [libc.so.6]
```

> **Key Observation**: Note what is **absent** from all these dependency lists — there is **no Go runtime library**, no `libgo.so`, no Go-related shared objects whatsoever. The entire kitty process (launcher + `fast_data_types.so` + GLFW backends) runs exclusively on C and Python. Go never enters this process space.

#### 2.2.5 Notable Absence: OpenGL

OpenGL (`libGL.so`) is **not** listed as a direct NEEDED dependency in `readelf -d` output for `fast_data_types.so`. This is because Kitty uses **GLAD** (the OpenGL loader in `kitty/gl.c` and the `glad/` directory) to dynamically load OpenGL function pointers at runtime. The `gl.c` file includes `glfw-wrapper.h` which provides the GLAD-generated loader. OpenGL symbols are resolved via `dlopen`/`dlsym` at initialization time, meaning they appear in `/proc/PID/maps` at runtime but not in static linkage analysis.

**[SOURCE-INFORMED]** From `kitty/gl.c` lines 1–20:

```c
#include "gl.h"
#include "glfw-wrapper.h"    // GLAD loader
#include "state.h"

static void check_for_gl_error(...) {
    GLenum code = glad_glGetError();  // GLAD-generated OpenGL call
    ...
}
```

This means that in a running kitty process with a display server, `libGL.so`, `libEGL.so`, or Mesa libraries (`libgallium.so`, `swrast_dri.so`) would appear in `/proc/PID/maps`, loaded on-demand by GLAD.

#### 2.2.6 Notable Absence: Fontconfig

Similarly, `libfontconfig.so` does not appear as a direct NEEDED dependency despite Kitty using Fontconfig for font discovery on Linux. This is because `kitty/fontconfig.c` also uses runtime `dlopen` — analogous to the GLAD approach for OpenGL:

**[SOURCE-INFORMED]** From `kitty/fontconfig.c`:

```c
#include <dlfcn.h>                                    // line 12
static void* libfontconfig_handle = NULL;             // line 20
// ...
libfontconfig_handle = dlopen(libnames[i], RTLD_LAZY); // line 91
```

Fontconfig is loaded dynamically at runtime via `dlopen()`, iterating over a list of possible library names until one is found. This means `libfontconfig.so` would appear in `/proc/PID/maps` at runtime (after font subsystem initialization) but not in static `readelf -d` or `ldd` output. The `dlclose()` call at line 134 indicates the library handle is also properly released during cleanup.

### 2.3 Kitten Binary Dependencies

**[OBSERVED]** In stark contrast to the C layer:

```bash
$ ldd kitty/launcher/kitten
  linux-vdso.so.1
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  /lib64/ld-linux-x86-64.so.2
```

The `kitten` binary depends **only** on `libc.so.6`. All Go runtime, cryptographic, image processing, and networking code is statically compiled into the 15 MB binary. This is the hallmark of Go's compilation model.

**[OBSERVED]** Confirming via `readelf`:

```bash
$ readelf -d kitty/launcher/kitten | grep NEEDED
  (NEEDED)  Shared library: [libc.so.6]
```

Only a single `NEEDED` entry. Compare this with `fast_data_types.so`'s 8 NEEDED entries — the Go binary carries its world internally.

> **Known Go Dependency CVE — golang.org/x/image v0.17.0**: The `kitten` binary is compiled
> against `golang.org/x/image v0.17.0` (declared in `go.mod`), which contains
> CVE-2024-24792 — a panic in the TIFF image parser when processing corrupt or malicious
> paletted images with invalid color indices. The vulnerable `tiff.Decode` function is
> imported in `tools/utils/images/formats.go` and exercised by the `icat` kitten when
> displaying TIFF images. The fix requires upgrading to `golang.org/x/image v0.18.0+`.
> Because Go statically compiles all dependencies into the binary, this vulnerability is
> embedded in the `kitten` executable itself — unlike the C libraries linked by
> `fast_data_types.so`, which benefit from Ubuntu's system-level backported security patches.
> An attacker could cause a denial-of-service by providing a maliciously crafted TIFF image
> to `kitten icat`.

---

## Section 3 — Thread Activity: Idle vs. Stress

### 3.1 The Three-Thread Architecture

Kitty's threading model is defined entirely in C, in `kitty/child-monitor.c`. The `ChildMonitor` struct (lines 49–62) is the central data structure:

**[SOURCE-INFORMED]** From `kitty/child-monitor.c` lines 49–62:

```c
typedef struct {
    PyObject_HEAD

    PyObject *dump_callback, *update_screen, *death_notify;
    unsigned int count;
    bool shutting_down;
    pthread_t io_thread, talk_thread;   // ← Two additional thread handles

    int talk_fd, listen_fd;             // ← Remote control socket fds
    Message *messages;
    size_t messages_capacity, messages_count;
    LoopData io_loop_data;
    void (*parse_func)(void*, ParseData*, bool);  // ← VT parser function pointer
} ChildMonitor;
```

The 2–3 threads are (the Talk thread is only created when remote control is enabled via `--listen-on` or single-instance mode):

#### Thread 1: Main Thread (Rendering + Event Loop)

**[SOURCE-INFORMED]** Created implicitly (the process's initial thread). Runs `main_loop()` at line 1258:

```c
// kitty/child-monitor.c lines 1258-1274
static PyObject*
main_loop(ChildMonitor *self, PyObject *a UNUSED) {
    state_check_timer = add_main_loop_timer(1000, true, do_state_check, self, NULL);
    run_main_loop(process_global_state, self);
    // ... cleanup ...
    Py_RETURN_NONE;
}
```

The `run_main_loop()` function drives the GLFW event loop, calling `render()` (lines 870–896) on each iteration:

```c
// kitty/child-monitor.c lines 870-896
static void render(monotonic_t now, bool input_read) {
    for (size_t i = 0; i < global_state.num_os_windows; i++) {
        OSWindow *w = global_state.os_windows + i;
        if (!render_os_window(w, now, false, scan_for_animated_images)) {
            // ...
        }
    }
    last_render_at = now;
}
```

The `render_os_window()` function (lines 832–868) calls:
- `make_os_window_context_current(w)` — Activate the OpenGL context
- `prepare_to_render_os_window(w, ...)` — Update screen data for rendering
- `render_prepared_os_window(w, ...)` — Execute the OpenGL rendering pipeline
  - which internally calls `swap_window_buffers()` (line 810) — Present the frame

**All of these are C functions. The Main thread spends its rendering time in C, not Python.**

#### Thread 2: I/O Thread (PTY Multiplexing + VT Parsing)

**[SOURCE-INFORMED]** Created in `start()` at line 291:

```c
// kitty/child-monitor.c line 291
ret = pthread_create(&self->io_thread, NULL, io_loop, self);
```

The I/O thread runs `io_loop()`, which:
1. Polls child PTY file descriptors via `poll()`
2. Reads PTY output data
3. Dispatches to `self->parse_func` — which is either `parse_worker` or `parse_worker_dump` (line 180–181)
4. The `parse_func` calls into `kitty/vt-parser.c` to process escape sequences

The VT parser (`kitty/vt-parser.c`) includes SIMD acceleration:

```c
// kitty/vt-parser.c line 15
#include "simd-string.h"
```

This means the I/O thread runs **pure C code** for parsing, with SIMD-accelerated string scanning on capable hardware.

#### Thread 3: Talk Thread (Remote Control)

**[SOURCE-INFORMED]** Conditionally created in `start()` at lines 285–289:

```c
// kitty/child-monitor.c lines 285-289
if (self->talk_fd > -1 || self->listen_fd > -1) {
    if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
        return PyErr_Format(PyExc_OSError, "Failed to start talk thread...");
    }
    talk_thread_started = true;
}
```

The Talk thread only starts when remote control is enabled (via `--listen-on` or single-instance mode). It handles UNIX socket communication for `kitty @` commands.

### 3.2 Thread Synchronization

**[SOURCE-INFORMED]** From `kitty/child-monitor.c` lines 76–79:

```c
#define children_mutex(op)  pthread_mutex_##op(&children_lock);
#define talk_mutex(op)      pthread_mutex_##op(&talk_lock);
```

- `children_lock` — Protects the child process list (used in `add_child` at line 307)
- `talk_lock` — Protects the talk/remote-control message queue
- Wakeup pipes — Cross-thread signaling: `wakeup_io_loop(self, false)` (line 300) writes to a pipe that `io_loop`'s `poll()` monitors

### 3.3 Single-Python-Thread Evidence

**[SOURCE-INFORMED]** From `kitty/main.py` line 504:

```python
sys.setswitchinterval(1000.0)  # we have only a single python thread
```

This is **the** evidence that Python never runs concurrent threads. The switch interval is set to 1000 seconds (effectively infinite). This means:

- The GIL never needs to be released for Python thread switching
- All multi-threading happens in C via `pthread_create`
- Python code runs only on the Main thread, between C calls
- When C code runs on the Main thread (rendering), Python is not executing

### 3.4 Idle vs. Stress Comparison

**[BLOCKED]** Direct thread observation via `/proc/PID/task/` was not possible because the kitty process could not be launched (no display server). However, the architecture makes precise predictions:

| Aspect | Idle State | Under Rendering Stress |
|---|---|---|
| **Thread Count** | 2 or 3 (Main + I/O, + Talk if RC enabled) | **Same 2 or 3** — no dynamic thread creation |
| **Main Thread** | Blocked in GLFW `poll_events()` waiting for input/timer | Actively calling `render_os_window()` → OpenGL draw calls each frame |
| **I/O Thread** | Blocked in `poll()` on PTY fds with no data arriving | Continuously reading PTY data, calling `parse_func` (VT parser) at high rate |
| **Talk Thread** | Blocked in `poll()` on UNIX socket with no clients | Active only if `kitty @` commands are being sent during stress |
| **Python Activity** | Minimal — occasional timer callbacks | Minimal — Python only handles high-level events between C frames |
| **C Activity** | Near zero CPU | Dominant — VT parsing + OpenGL rendering consume most CPU time |

> **Reasoning**: The thread count stays constant because kitty creates exactly 2–3 threads at startup and never spawns more. The stress manifests as increased CPU utilization on the **existing** threads, not as additional thread creation. This is a deliberate design choice — thread creation/destruction overhead is avoided, and the fixed thread pool has clear ownership of responsibilities.

### 3.5 Expected `/proc/PID/task/` Output

In a running kitty instance with remote control enabled, one would observe:

```
/proc/<PID>/task/
├── <TID_main>/     — Main thread (rendering + event loop)
├── <TID_io>/       — I/O thread (PTY mux + VT parsing)
└── <TID_talk>/     — Talk thread (remote control socket)
```

Each thread's `stat` file would show CPU time distribution shifting from IO-bound (idle) to CPU-bound (stress) behavior, with the I/O thread showing the largest delta due to VT parsing volume.

---

## Section 4 — Remote Control Interface Queries

### 4.1 Remote Control Protocol Architecture

**[SOURCE-INFORMED]** The remote control system spans all three languages:

**Python Server Side** — `kitty/remote_control.py` (lines 31–40) imports:

```python
from .fast_data_types import (
    AES256GCMDecrypt,      # C-implemented decryption
    AES256GCMEncrypt,      # C-implemented encryption
    EllipticCurveKey,      # C-implemented key exchange
    get_boss,              # Get the Boss singleton
    get_options,           # Get current options
    monotonic,             # High-resolution timer
    read_command_response, # Read RC response
    send_data_to_peer,     # Send data to RC client
)
```

**C Transport Layer** — The Talk thread in `child-monitor.c` handles UNIX socket I/O, reading incoming commands and writing responses. The encryption (AES-256-GCM) and key exchange (X25519) are implemented in C via `kitty/crypto.c` using OpenSSL.

**Python Command Dispatch** — `kitty/rc/base.py` provides the `RemoteCommand` base class. There are **41 command modules** in `kitty/rc/`, each implementing a specific `kitty @` subcommand.

**Go Client Side** — The `kitten` binary includes a Go implementation of the remote control client in `tools/cmd/at/`. When a user runs `kitty @ ls`, it can use either the Go `kitten` binary as the client or the Python-based client.

**Protocol Flow:**

```
Client (Go kitten or Python) ──UNIX socket──→ Talk Thread (C) ──mutex──→ Python dispatch (rc/*.py) → Response
```

### 4.2 Command Examples and Expected Outputs

**[BLOCKED]** The remote control commands could not be executed because launching kitty requires a display server. Below are the commands that **would** be issued and their expected outputs based on source code analysis.

#### 4.2.1 `kitty @ ls`

**Command:**

```bash
$ kitty @ --to unix:/tmp/kitty-test.sock ls
```

**Expected Output** (from `kitty/rc/ls.py` lines 48–57):

The `response_from_kitty` method calls `boss.list_os_windows(window, tab_filter, window_filter)`, which returns a JSON tree:

```json
[
  {
    "id": 1,
    "is_focused": true,
    "platform_window_id": 12345678,
    "tabs": [
      {
        "id": 1,
        "is_focused": true,
        "title": "~",
        "layout": "stack",
        "windows": [
          {
            "id": 1,
            "is_focused": true,
            "is_self": false,
            "title": "zsh",
            "pid": 54321,
            "cwd": "/home/user",
            "cmdline": ["/bin/zsh"],
            "columns": 80,
            "lines": 24,
            "env": {"TERM": "xterm-kitty", "SHELL": "/bin/zsh"}
          }
        ]
      }
    ]
  }
]
```

This reveals the hierarchical model: **OS Windows → Tabs → Windows**, managed by `kitty/boss.py`, with the window state (PID, CWD, command line) tracked per child process.

#### 4.2.2 `kitty @ get-colors`

**Command:**

```bash
$ kitty @ --to unix:/tmp/kitty-test.sock get-colors
```

**Expected Output** (from `kitty/rc/get_colors.py` lines 42–50):

The `response_from_kitty` method iterates over `opts` attributes that are `Color` instances:

```
foreground    #dddddd
background    #000000
cursor        #cccccc
selection_foreground #000000
selection_background #fffacd
color0        #000000
color1        #cc0403
color2        #19cb00
...
color15       #ffffff
```

#### 4.2.3 `kitty @ get-text`

**Command:**

```bash
$ kitty @ --to unix:/tmp/kitty-test.sock get-text --extent screen
```

**Expected Output** (from `kitty/rc/get_text.py`): Returns the current screen contents as plain text (or with ANSI codes if `--ansi` is specified).

### 4.3 Dual-Language Client Implementation

A notable architectural detail: the `kitty @` command interface has **two independent client implementations**:

1. **Python client** — in `kitty/remote_control.py`, used when running from within the kitty process itself
2. **Go client** — in `tools/cmd/at/`, compiled into the `kitten` binary, used when running from any terminal

Both communicate with the same **C+Python server** via the UNIX socket protocol. This is evidence of the three-language boundary: C handles the socket I/O (Talk thread), Python handles command dispatch (rc modules), and Go provides a portable standalone client.

---

## Section 5 — Kitten Process Relationship

### 5.1 The `os.execl` Process Replacement

The most critical observation about the kitten process model is how Go kittens are launched. This is **not** a `fork+exec` creating a child process under the kitty parent — it is an `execl` that **replaces the current process image**.

**[SOURCE-INFORMED]** From `kitty/entry_points.py` lines 10–12:

```python
def icat(args: List[str]) -> None:
    from kitty.constants import kitten_exe
    os.execl(kitten_exe(), "kitten", *args)
```

The `os.execl()` system call replaces the **entire process** — the Python interpreter, all loaded C extensions, everything — with the Go `kitten` binary. After this call, the process is running Go code exclusively.

**[SOURCE-INFORMED]** From `kitty/constants.py` lines 82–84:

```python
@run_once
def kitten_exe() -> str:
    return os.path.join(os.path.dirname(kitty_exe()), 'kitten')
```

The `kitten` binary is a **separate file on disk**, located in the same directory as the `kitty` binary. This is not a symlink or a different invocation of the same binary — it is a completely independent Go executable.

### 5.2 Additional exec Patterns

The `os.execl`/`os.execvp` pattern is used consistently for all Go-implemented functionality:

**[SOURCE-INFORMED]** From `kitty/entry_points.py`:

```python
# Line 27-30: hold() function
def hold(args: List[str]) -> None:
    from kitty.constants import kitten_exe
    args = ['kitten', '__hold_till_enter__'] + args[1:]
    os.execvp(kitten_exe(), args)

# Lines 33-43: complete() function
def complete(args: List[str]) -> None:
    from kitty.constants import kitten_exe
    args = ['kitten', '__complete__'] + args[1:]
    os.execvp(kitten_exe(), args)
```

Every function that delegates to Go uses `os.execl` or `os.execvp` — process replacement, not subprocess creation.

### 5.3 Expected Process Tree Observation

When `kitty +kitten icat <image>` is invoked from within a running kitty terminal:

**[SOURCE-INFORMED]** The process flow is:

1. The kitty main process (PID A) spawns a child shell process (PID B)
2. The user types `kitty +kitten icat photo.png`
3. The kitty launcher dispatches to `entry_points.py`'s `icat()` function
4. `os.execl(kitten_exe(), "kitten", "icat", "photo.png")` is called
5. This **replaces** PID B's process image with the Go `kitten` binary
6. The `kitten` process (now PID B) runs the Go icat code from `kittens/icat/main.go`

What `ps` would show during icat execution:

```bash
$ ps aux | grep -E 'kitty|kitten'
user  PID_A  ... kitty/launcher/kitty --listen-on unix:/tmp/kitty.sock
user  PID_B  ... kitten icat photo.png
```

What `pstree` would show:

```
kitty(PID_A)───zsh(PID_B)───kitten(PID_B')
```

> **Key Insight**: The `kitten` process runs as a **child process** of the shell that kitty spawned, NOT as a child of the kitty process directly. The `os.execl` replaces the shell's child, so the kitten binary communicates with the terminal via standard I/O (escape sequences), not via any in-process API.

### 5.4 Contrast: Python Kittens Run In-Process

**[SOURCE-INFORMED]** From `kittens/runner.py` lines 46–65 (simplified for clarity — the actual source includes a `with preserve_sys_path():` context manager, `sys.path.insert(0, ...)` path manipulation, and explicit `lambda *a, **kw: None` defaults):

```python
def import_kitten_main_module(config_dir: str, kitten: str) -> Dict[str, Any]:
    if kitten.endswith('.py'):
        with preserve_sys_path():
            path = path_to_custom_kitten(config_dir, kitten)
            if os.path.dirname(path):
                sys.path.insert(0, os.path.dirname(path))
            with open(path) as f:
                src = f.read()
            code = compile(src, path, 'exec')
            g = {'__name__': 'kitten'}
            exec(code, g)
            hr = g.get('handle_result', lambda *a, **kw: None)
        return {'start': g['main'], 'end': hr}

    kitten = resolved_kitten(kitten)
    m = importlib.import_module(f'kittens.{kitten}.main')
    return {
        'start': getattr(m, 'main'),
        'end': getattr(m, 'handle_result', lambda *a, **k: None),
    }
```

Python-based kittens (those with Python `main.py` in their package) are loaded via `importlib.import_module()` and run **within the kitty process**. They share the same Python interpreter, the same `fast_data_types` C extension, and the same process memory.

**The key distinction:**

| Aspect | Go Kittens (e.g., icat) | Python Kittens |
|---|---|---|
| **Launch mechanism** | `os.execl()` — process replacement | `importlib.import_module()` — in-process |
| **Process model** | Separate OS process, separate PID | Same process as kitty |
| **Language runtime** | Go runtime (GC, goroutines) | CPython interpreter (shared with kitty) |
| **Communication** | Terminal escape sequences, UNIX sockets | Direct Python function calls |
| **Access to C layer** | None (separate binary) | Full access via `fast_data_types` |

### 5.5 icat Go Implementation

**[SOURCE-INFORMED]** The icat kitten's Go source files in `kittens/icat/`:

```go
// kittens/icat/main.go — imports reveal the Go ecosystem used
import (
    "kitty/tools/cli"           // CLI argument parsing
    "kitty/tools/tty"           // Terminal TTY handling
    "kitty/tools/tui"           // Text UI framework
    "kitty/tools/tui/graphics"  // Graphics protocol implementation
    "kitty/tools/utils/images"  // Image loading and processing
    "kitty/tools/utils/style"   // Terminal styling
    "golang.org/x/sys/unix"     // POSIX system calls
)
```

Supporting files:
- `cli_generated.go` — Auto-generated CLI argument definitions
- `native.go` — Native Go image decoding (PNG, JPEG, GIF, WebP, TIFF, BMP)
- `detect.go` — Terminal graphics capability probing via escape sequences
- `transmit.go` — Graphics protocol data transmission (shared memory, files, or direct)
- `process_images.go` — Image processing pipeline (resize, crop, color adjustment)
- `magick.go` — Optional ImageMagick integration for exotic formats

All image processing, protocol negotiation, and data transmission happens in Go — **none of it requires the C layer or Python interpreter**.

---

## Section 6 — Kitten Binary Inspection

### 6.1 Binary Format Identification

**[OBSERVED]** The `file` command reveals the binary's nature:

```bash
$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  Go BuildID=hqq4-U2LKlixbsjwYo2Y/n8c9tVmrH955DZP0gtLh/TlErkqS1Lkyxjr-Onw6b/O2yALBSe3QkGVIYlcExQ,
  stripped
```

**Analysis:**

- **`ELF 64-bit LSB executable`** — Standard Linux executable format
- **`Go BuildID=hqq4-U2LKlixbsjwYo2Y/...`** — Embedded Go build identifier, definitively proving this is a Go-compiled binary
- **`dynamically linked`** — Links against `libc.so.6` only (CGO or external linking mode)
- **`stripped`** — Debug symbols removed via `-s -w` ldflags (setup.py line 1157)

### 6.2 Dynamic Dependency Analysis

**[OBSERVED]** The kitten binary's minimal dependency footprint:

```bash
$ ldd kitty/launcher/kitten
  linux-vdso.so.1
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  /lib64/ld-linux-x86-64.so.2

$ readelf -d kitty/launcher/kitten | grep NEEDED
  (NEEDED)  Shared library: [libc.so.6]
```

**Only `libc.so.6`** is required. No libpython, no libharfbuzz, no libfreetype, no OpenGL, no libpng — the Go binary is self-contained for all its functionality. Image processing uses pure Go libraries (`github.com/kovidgoyal/imaging`, `golang.org/x/image`), not system libraries.

### 6.3 Go Runtime Presence

**[OBSERVED]** The binary is stripped (`-s -w`), so `nm` and `go tool nm` cannot read symbols:

```bash
$ nm kitty/launcher/kitten
nm: kitty/launcher/kitten: no symbols

$ go tool nm kitty/launcher/kitten
reading kitty/launcher/kitten: no symbol section
```

However, Go runtime evidence is still available via `strings` and the `.go.buildinfo` section:

**[OBSERVED]** A raw `strings | grep 'runtime\.'` produces garbled entries first (due to partial string matches in the stripped binary):

```bash
$ strings kitty/launcher/kitten | grep 'runtime\.' | head -10
runtime.
runtime.H9
runtime.H9
runtime.H9
runtime.H9
runtime.H
runtime.H
runtime.H9
runtime.H92
runtime.1
```

Filtering for clean Go runtime symbols with a tighter pattern reveals the actual Go runtime function names:

```bash
$ strings kitty/launcher/kitten | grep -E '^runtime\.[a-z]' | head -10
runtime.cmpstring
runtime.memequal
runtime.memequal_varlen
runtime.init
runtime.init.func2
runtime.sigdelset
runtime.memhash8
runtime.memhash16
runtime.memhash128
runtime.memhash_varlen
```

These are unmistakably Go runtime internal functions — `runtime.cmpstring`, `runtime.memequal`, `runtime.memhash*` — confirming a Go runtime is embedded in the binary despite symbol stripping.

**[OBSERVED]** The raw `readelf -p .go.buildinfo` output contains `^I` tab characters, `\n` literal escapes, and hash checksums that make it difficult to read directly. Using `go version -m` provides cleaner, structured output:

```bash
$ go version -m kitty/launcher/kitten
kitty/launcher/kitten: go1.22.10
	path	kitty/tools/cmd
	mod	kitty	(devel)
	dep	github.com/ALTree/bigfloat	v0.2.0
	dep	github.com/alecthomas/chroma/v2	v2.14.0
	dep	github.com/bmatcuk/doublestar/v4	v4.6.1
	dep	github.com/disintegration/imaging	v1.6.2
	dep	github.com/dlclark/regexp2	v1.11.0
	dep	github.com/edwvee/exiffix	v0.0.0-20240229113213-0dbb146775be
	dep	github.com/google/uuid	v1.6.0
	dep	github.com/klauspost/cpuid/v2	v2.2.5
	dep	github.com/kovidgoyal/imaging	v1.6.3
	dep	github.com/rwcarlsen/goexif	v0.0.0-20190401172101-9e8deecbddbd
	dep	github.com/seancfoley/bintree	v1.3.1
	dep	github.com/seancfoley/ipaddress-go	v1.6.0
	dep	github.com/shirou/gopsutil/v3	v3.24.5
	dep	github.com/tklauser/go-sysconf	v0.3.12
	dep	github.com/tklauser/numcpus	v0.6.1
	dep	github.com/zeebo/xxh3	v1.0.2
	dep	golang.org/x/exp	v0.0.0-20230801115018-d63ba01acd4b
	dep	golang.org/x/image	v0.17.0
	dep	golang.org/x/sys	v0.21.0
	dep	howett.net/plist	v1.0.1
	build	-buildmode=exe
	build	-compiler=gc
	build	-ldflags="-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w"
	build	CGO_ENABLED=1
```

This confirms:
1. The binary was built from `kitty/tools/cmd` (the Go entry point)
2. Go version `go1.22.10` was used
3. 14 of 15 direct dependencies from `go.mod` are embedded (`github.com/google/go-cmp` is test-only and excluded from the binary), plus 6 indirect dependencies (disintegration/imaging, klauspost/cpuid/v2, rwcarlsen/goexif, seancfoley/bintree, tklauser/go-sysconf, tklauser/numcpus)
4. The ldflags include `-s -w` (strip symbols/DWARF) and the VCS revision for version tracking
5. `CGO_ENABLED=1` confirms external linking mode (explaining the `libc.so.6` dependency)

**[OBSERVED]** Kitty-specific Go packages embedded in the binary:

```bash
$ strings kitty/launcher/kitten | grep '^kitty/' | head -20
kitty/tools/cli
kitty/tools/tty
kitty/tools/tui
kitty/kittens/ssh
kitty/tools/utils
kitty/kittens/ask
kitty/tools/rsync
kitty/tools/config
kitty/tools/themes
kitty/tools/cmd/at
kitty/kittens/diff
kitty/kittens/icat
kitty/kittens/hints
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/wcswidth
kitty/kittens/themes
kitty/tools/utils/shm
kitty/tools/cli/markup
kitty/kittens/show_key
```

This reveals the full scope of Go functionality: CLI tools, kittens (icat, diff, ssh, hints, themes, transfer, show_key, ask), TUI framework (sgr, loop, graphics), shared utilities (wcswidth, shm), crypto, and remote control client (`cmd/at`).

### 6.4 Build System Evidence

**[SOURCE-INFORMED]** From `setup.py` lines 1130–1165, the `build_static_kittens()` function:

```python
def build_static_kittens(args, launcher_dir, destination_dir='', ...):
    go = shutil.which('go')
    cmd = [go, 'build', '-v']
    vcs_rev = args.vcs_rev or get_vcs_rev()
    ld_flags = []
    binary_data_flags = [f"-X kitty.VCSRevision={vcs_rev}"]
    if not args.debug:
        ld_flags.append('-s')   # Strip symbol table
        ld_flags.append('-w')   # Strip DWARF debug info
    cmd += ['-ldflags', ' '.join(binary_data_flags + ld_flags)]
    dest = os.path.join(destination_dir or launcher_dir, 'kitten')
    src = os.path.abspath('tools/cmd')
    # ... execute go build ...
```

The Go entry point describes itself:

```go
// tools/cmd/main.go lines 22-24
root.ShortDescription = "Fast, statically compiled implementations of various kittens
  (command line tools for use with kitty)"
```

> **The description is itself a design statement**: "Fast, statically compiled implementations" — the Go kittens exist to provide **portable, fast CLI tools** that don't require a Python interpreter or the C extension to be present on the target system.

### 6.5 Go Version Confirmation

**[OBSERVED]** Direct Go version string in the binary:

```bash
$ strings kitty/launcher/kitten | grep 'go1\.[0-9]'
go1.22.10

$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

---

## Section 7 — Symbol and Stack Snapshots

### 7.1 Snapshot Attempts and Results

Due to the headless environment (no display server), the kitty process could not be fully launched, making live process inspection impossible. Below is a systematic account of each attempted method and the fallback used.

#### Method 1: `/proc/PID/maps` — Memory Map

**[BLOCKED]** Cannot obtain because the kitty process cannot start:

```bash
$ kitty/launcher/kitty --listen-on unix:/tmp/kitty-test.sock
[0.059] [glfw error 65544]: X11: The DISPLAY environment variable is missing
GLFW initialization failed
```

**Fallback**: We used `ldd` on the compiled binaries to reconstruct what the memory map **would** contain. In a running kitty process, `/proc/PID/maps` would show:

```
# Expected /proc/<PID>/maps entries (based on ldd output):
<addr>  r-xp  .../kitty/launcher/kitty           # C launcher code
<addr>  r-xp  .../libpython3.12.so.1.0           # CPython interpreter
<addr>  r-xp  .../kitty/fast_data_types.so        # 1.5 MB C extension (ALL C code)
<addr>  r-xp  .../kitty/glfw-x11.so              # GLFW X11 backend
<addr>  r-xp  .../libharfbuzz.so.0               # HarfBuzz text shaping
<addr>  r-xp  .../libfreetype.so.6               # FreeType glyph rasterization
<addr>  r-xp  .../libpng16.so.16                 # PNG decoding
<addr>  r-xp  .../liblcms2.so.2                  # ICC color management
<addr>  r-xp  .../libcrypto.so.3                 # OpenSSL encryption
<addr>  r-xp  .../libGL.so.1                     # OpenGL (loaded by GLAD at runtime)
<addr>  r-xp  .../libX11.so.6                    # X11 protocol
<addr>  r-xp  .../libxkbcommon.so.0              # Keyboard layout handling
```

**What would NOT appear**: Any Go runtime library, `libgo.so`, or Go-compiled `.so` files — because Go code runs only in the separate `kitten` binary process.

#### Method 2: `strace` on kitten Binary

**[OBSERVED]** Since the `kitten` binary can run without a display server (for `--version`), we captured a brief trace:

```bash
$ strace -f -e trace=write,read -c kitty/launcher/kitten --version 2>&1
strace: Process 44785 attached
strace: Process 44786 attached
strace: Process 44787 attached
strace: Process 44788 attached
strace: Process 44789 attached
strace: Process 44790 attached
strace: Process 44791 attached
strace: Process 44792 attached
strace: Process 44793 attached
strace: Process 44794 attached
kitten 0.35.2 created by Kovid Goyal
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000223          22        10           read
  0.00    0.000000           0         1           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.000223          20        11           total
```

**Analysis**: The 10 `strace: Process ... attached` messages are notable — they reveal that the Go runtime immediately spawns multiple OS threads (goroutine scheduler threads) even for a trivial `--version` invocation. This is characteristic of the Go runtime, which creates threads for its garbage collector, timer management, and goroutine scheduling. The `read` calls dominate (10 reads vs 1 write) because the Go runtime performs thread setup reads before the single `write` that outputs the version string. In a full icat session, the write calls would dominate (graphics protocol escape sequences being sent to the terminal).

#### Method 3: Symbol Table of `fast_data_types.so`

**[OBSERVED]** The `fast_data_types.so` is **not stripped**, so symbol inspection succeeds:

```bash
$ nm -D kitty/fast_data_types.so | grep 'T ' | head -8
0000000000029860 T PyInit_fast_data_types   # Module initialization entry point
00000000000f5c90 T base64_decode
00000000000f5bf0 T base64_encode
00000000000f5be0 T base64_stream_decode
00000000000f5b90 T base64_stream_decode_init
00000000000f5b10 T base64_stream_encode
00000000000f5b20 T base64_stream_encode_final
00000000000f5ac0 T base64_stream_encode_init
```

The key exported symbol is `PyInit_fast_data_types` at address `0x29860` — this is the CPython module initialization function that runs when Python executes `import kitty.fast_data_types`.

**Imported symbols reveal the C layer's dependencies:**

```bash
$ nm -D kitty/fast_data_types.so | grep 'U ' | grep -E 'hb_|FT_|png_|EVP_|cms' | wc -l
104
```

There are **104 undefined symbols** (imported functions) from HarfBuzz (22), FreeType (25), libpng (25), OpenSSL (26), and lcms2 (6). These are the rendering-adjacent library functions that the C layer calls directly.

### 7.2 SIMD Evidence

**[SOURCE-INFORMED]** The C layer includes SIMD-accelerated string operations:

```c
// kitty/simd-string-128.c — SSE4.2 (128-bit)
#define KITTY_SIMD_LEVEL 128
#include "simd-string-impl.h"

// kitty/simd-string-256.c — AVX2 (256-bit)
#define KITTY_SIMD_LEVEL 256
#include "simd-string-impl.h"
```

The VT parser includes SIMD acceleration:

```c
// kitty/vt-parser.c line 15
#include "simd-string.h"
```

And `kitty/simd-string.c` provides the runtime dispatch coordinator that selects the appropriate implementation based on CPU capabilities.

**These SIMD operations exist exclusively in the C layer.** Neither Python (which has no SIMD intrinsics support) nor Go (whose compiler has limited auto-vectorization) provides equivalent acceleration. This is a deliberate architectural choice: the VT parser's inner loop — where every byte of terminal output passes through — is optimized with platform-specific SIMD instructions compiled into the C extension.

### 7.3 Expected Stack Frames Under Rendering Stress

Based on the source code structure, a `gdb -batch -ex 'thread apply all bt' -p PID` during rendering stress would reveal:

**Main Thread (rendering):**
```
#0  glXSwapBuffers()              — libGL.so (frame presentation)
#1  swap_window_buffers()         — fast_data_types.so (kitty/shaders.c)
#2  render_prepared_os_window()   — fast_data_types.so (kitty/shaders.c)
#3  render_os_window()            — fast_data_types.so (kitty/child-monitor.c:833)
#4  render()                      — fast_data_types.so (kitty/child-monitor.c:871)
#5  run_main_loop()               — fast_data_types.so (GLFW event loop)
#6  main_loop()                   — fast_data_types.so (kitty/child-monitor.c:1262)
#7  <Python frame: boss.child_monitor.main_loop()>  — Python calling into C
```

**I/O Thread (parsing):**
```
#0  simd_find_either_of_two_bytes_128() — fast_data_types.so (kitty/simd-string-128.c)
#1  do_parse_vt()                       — fast_data_types.so (kitty/vt-parser.c)
#2  parse_worker()                      — fast_data_types.so (kitty/child-monitor.c)
#3  io_loop()                           — fast_data_types.so (kitty/child-monitor.c)
#4  start_thread()                      — libpthread.so
```

**Talk Thread (idle or serving RC):**
```
#0  poll()                        — libc.so (waiting for UNIX socket data)
#1  talk_loop()                   — fast_data_types.so (kitty/child-monitor.c)
#2  start_thread()                — libpthread.so
```

> **Observation**: The expected stacks are dominated by C frames. Python frames appear only at the highest level of the Main thread (where Python called `boss.child_monitor.main_loop()`). The I/O thread and Talk thread have **zero** Python frames — they run entirely in C.

---

## Section 8 — Language Responsibility Inference

### 8.1 Synthesis of All Evidence

Based on the runtime artifacts collected — binary inspection, library linkage, symbol analysis, build system analysis, and source code — the following responsibility model emerges:

### 8.2 Python Responsibilities

| Responsibility | Source File(s) | Evidence |
|---|---|---|
| **Application startup and lifecycle** | `kitty/main.py` (lines 441–521) | `_main()` calls `parse_args → create_opts → init_glfw → run_app` |
| **Configuration loading** | `kitty/config.py` | Parses `kitty.conf`, applies options |
| **Window/Tab management** | `kitty/boss.py`, `kitty/window.py`, `kitty/tabs.py` | Boss singleton manages OS windows, tabs, windows |
| **Remote control command dispatch** | `kitty/remote_control.py`, `kitty/rc/*.py` (41 modules) | `RemoteCommand` subclasses handle each `kitty @` command |
| **Python kitten framework** | `kittens/runner.py` (lines 46–65) | `importlib.import_module(f'kittens.{kitten}.main')` |
| **Session management** | `kitty/session.py` | Session layout creation |
| **GLSL shader source loading** | `kitty/shaders.py` | Reads `.glsl` files, injects preprocessor macros |
| **Entry point dispatch** | `kitty/entry_points.py` | Routes `+kitten`, `+hold`, `+complete` to appropriate handler |

**Evidence Chain:**
- `sys.setswitchinterval(1000.0)` at `main.py:504` confirms single Python thread
- Python never participates in the rendering loop (it calls `main_loop()` which enters C)
- Python's role is **orchestration and high-level logic** — it sets up the system, then delegates to C for the event loop

### 8.3 C Responsibilities

| Responsibility | Source File(s) | Evidence |
|---|---|---|
| **VT escape sequence parsing** | `kitty/vt-parser.c` | Hot path with SIMD acceleration (`simd-string.h` include) |
| **Screen model + line buffers** | `kitty/screen.c`, `kitty/line.c`, `kitty/line-buf.c` | Core data structures for terminal content |
| **OpenGL GPU rendering** | `kitty/shaders.c`, `kitty/gl.c`, 13 `.glsl` files | `render_os_window()` → OpenGL draw calls |
| **Font discovery + rasterization** | `kitty/freetype.c`, `kitty/fontconfig.c`, `kitty/fonts.c` | `FT_*`, `hb_*` symbols in `fast_data_types.so` |
| **GPU glyph texture atlas** | `kitty/glyph-cache.c` | `alloc_sprite_map()` with `GL_MAX_TEXTURE_SIZE` query |
| **Graphics protocol (images)** | `kitty/graphics.c` | Inline image buffer management |
| **Child process monitoring** | `kitty/child-monitor.c` | Three-thread architecture, `pthread_create` calls |
| **SIMD string operations** | `kitty/simd-string-128.c`, `kitty/simd-string-256.c` | SSE4.2 / AVX2 for VT parser inner loop |
| **Process entry (launcher)** | `kitty/launcher/main.c` | CPython embedding via `Py_InitializeFromConfig` |
| **Platform windowing (GLFW)** | `glfw/*.c` | X11/Wayland/Cocoa backends, compiled as separate `.so` |
| **Cryptography** | `kitty/crypto.c` | `EVP_*` OpenSSL symbols for RC encryption |
| **PNG decoding** | `kitty/png-reader.c` | `png_*` symbols for image loading |
| **Color management** | `kitty/colors.c` | `cmsCreateTransform` for ICC profiles |

**Evidence Chain:**
- `ldd kitty/fast_data_types.so` shows direct linkage to libharfbuzz, libfreetype, libpng, liblcms2, libcrypto
- `nm -D` reveals 104 imported symbols from these rendering libraries (hb_=22, FT_=25, png_=25, EVP_=26, cms=6)
- Thread creation in C (`pthread_create` at lines 286, 291 of `child-monitor.c`)
- Rendering loop entirely in C (`render()` → `render_os_window()` → OpenGL calls)
- The `.so` file is 1.5 MB — orders of magnitude larger than the 36 KB launcher

### 8.4 Go Responsibilities

| Responsibility | Source File(s) | Evidence |
|---|---|---|
| **`kitten` CLI binary** | `tools/cmd/main.go` | "Fast, statically compiled implementations of various kittens" |
| **Image display (icat)** | `kittens/icat/*.go` | Complete Go implementation: detect, process, transmit |
| **Remote control client** | `tools/cmd/at/` | Go-side `kitty @` command client |
| **Shell completion** | `os.execvp(kitten_exe(), args)` in `entry_points.py:33` | Delegated to Go binary |
| **Diff viewer** | `kittens/diff/` | Go-implemented side-by-side diff |
| **SSH kitten** | `kittens/ssh/` | Go-implemented SSH integration |
| **Various CLI utilities** | `kittens/hints/`, `kittens/themes/`, `kittens/transfer/`, etc. | Go-implemented kittens |
| **TUI framework** | `tools/tui/` | Go text UI library for kitten interfaces |
| **Encryption (client side)** | `tools/crypto/` | Go X25519/AES-GCM for RC client |

**Evidence Chain:**
- `file kitty/launcher/kitten` shows `Go BuildID` — definitively a Go binary
- `ldd` shows only `libc.so.6` — no Python, no rendering libraries
- `strings` reveal `runtime.main`, `kitty/tools/*`, `kitty/kittens/*` Go packages
- `go version -m` shows 14 direct + 6 indirect Go dependencies embedded, built from `kitty/tools/cmd`
- The binary is 15 MB — self-contained with the entire Go runtime
- Kitten process runs as separate PID (via `os.execl`)

### 8.5 The Architecture Boundary

```
┌─────────────────────────────────────────────┐
│           Kitty Process (single PID)         │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │         Python Layer                  │   │
│  │  main.py, boss.py, config.py         │   │
│  │  remote_control.py, rc/*.py          │   │
│  │  entry_points.py, session.py         │   │
│  │  shaders.py (GLSL loading)           │   │
│  └──────────┬───────────────────────────┘   │
│             │ calls via fast_data_types API  │
│  ┌──────────▼───────────────────────────┐   │
│  │         C Layer                       │   │
│  │  fast_data_types.so (49 .c files)     │   │
│  │  ├─ child-monitor.c (2–3 threads)     │   │
│  │  ├─ vt-parser.c (SIMD-accelerated)    │   │
│  │  ├─ screen.c, line.c, line-buf.c      │   │
│  │  ├─ shaders.c + 13 .glsl files        │   │
│  │  ├─ freetype.c, fontconfig.c, fonts.c │   │
│  │  ├─ glyph-cache.c (GPU texture atlas) │   │
│  │  ├─ graphics.c (image protocol)       │   │
│  │  └─ crypto.c (OpenSSL)                │   │
│  │                                       │   │
│  │  glfw-x11.so / glfw-wayland.so        │   │
│  │  (vendored GLFW 3.4 fork)             │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  Linked: libpython3.12, libharfbuzz,        │
│          libfreetype, libpng, liblcms2,     │
│          libcrypto, libGL (GLAD-loaded),    │
│          libX11 / libwayland               │
└─────────────────────────────────────────────┘

            ╔═══════════════════════════╗
            ║  PROCESS BOUNDARY         ║
            ╚═══════════════════════════╝

┌─────────────────────────────────────────────┐
│       Kitten Process (separate PID)          │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │         Go Layer                      │   │
│  │  kitten binary (15 MB, standalone)    │   │
│  │  ├─ tools/cmd/main.go (entry point)   │   │
│  │  ├─ kittens/icat/*.go (image display) │   │
│  │  ├─ tools/cmd/at/ (RC client)         │   │
│  │  ├─ tools/tui/ (TUI framework)        │   │
│  │  ├─ tools/crypto/ (encryption)        │   │
│  │  └─ kittens/{diff,ssh,hints,...}      │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  Links only: libc.so.6                      │
│  Communication: escape sequences,           │
│                 UNIX sockets (RC)           │
└─────────────────────────────────────────────┘
```

**The fundamental architectural insight**: C and Python share one process (C as Python extension modules loaded via `fast_data_types.so`), while Go **always** runs as a separate process communicating via terminal escape sequences or UNIX sockets. This is not a design accident — it is a deliberate separation that allows the Go tools to be distributed independently of the Python+C kitty installation.

---

## Section 9 — Two Falsified Interpretations

### 9.1 Falsified Interpretation #1: "Go Handles the Rendering Pipeline"

#### Why This Is Plausible From Code Reading

A developer browsing the repository might reasonably conclude that Go participates in the rendering pipeline:

- The `tools/tui/graphics/` package implements the kitty graphics protocol in Go
- The `kittens/icat/` directory contains Go code that transmits image data for display
- The `tools/tui/` package provides a complete terminal UI framework with rendering capabilities
- The `kitten` binary is 15 MB — much larger than the 36 KB kitty launcher, suggesting it carries significant functionality

One could plausibly conclude: "The Go code handles image rendering and perhaps assists with the GPU rendering pipeline."

#### Runtime Evidence That Falsifies This

1. **Process map evidence**: `ldd kitty/fast_data_types.so` and `ldd kitty/launcher/kitty` show **zero Go libraries** in the kitty process. There is no `libgo.so`, no Go runtime segments, no Go-compiled shared objects loaded. The rendering pipeline runs entirely within the kitty process, which contains only C and Python code.

2. **Binary separation evidence**: `file kitty/launcher/kitten` shows a Go binary with its own `Go BuildID`, and `readelf -d` shows it as a separate executable requiring only `libc.so.6`. The kitten binary never loads into the kitty process — it is invoked via `os.execl()` which **replaces** the process image entirely (`kitty/entry_points.py` lines 10–12).

3. **Rendering hot path is pure C**: The `render()` function in `kitty/child-monitor.c` (lines 870–896) iterates over OS windows and calls `render_os_window()` which invokes `make_os_window_context_current()` → `prepare_to_render_os_window()` → `render_prepared_os_window()` → `swap_window_buffers()`. All of these are C functions within `fast_data_types.so`. No Go interop exists at any point in this call chain.

4. **Shader compilation is pure C**: `kitty/shaders.c` (line 20) defines the shader programs (`CELL_PROGRAM`, `BORDERS_PROGRAM`, `GRAPHICS_PROGRAM`, etc.) and compiles GLSL shaders. `alloc_sprite_map()` (lines 50–69) queries `GL_MAX_TEXTURE_SIZE` directly via GLAD. No Go code participates in shader management.

5. **Symbol evidence**: `nm -D kitty/fast_data_types.so` shows `FT_*`, `hb_*`, `png_*`, `EVP_*` symbols — the rendering-adjacent libraries are all C. The single exported symbol `PyInit_fast_data_types` is a CPython module init, not a Go function.

#### The Correct Interpretation

Go's `tui/graphics` package implements the **client side** of the kitty graphics protocol — it sends escape sequences containing image data **to** the terminal. The **server side** (receiving and rendering the images on the GPU) is implemented in C (`kitty/graphics.c`). Go generates the protocol messages; C renders them. This is a producer-consumer relationship across a process boundary, not shared rendering responsibility.

### 9.2 Falsified Interpretation #2: "Python Directly Handles VT Parsing and Screen Updates"

#### Why This Is Plausible From Code Reading

Several observations might lead to this conclusion:

- Python files like `kitty/window.py` and `kitty/tabs.py` manage window and tab state
- Python imports `Screen` from `fast_data_types`, suggesting Python manipulates screen objects
- The `kitty/boss.py` module appears to be the central controller that manages all windows and their content
- The `kitty/fast_data_types.pyi` type stub file declares `Screen` as a Python class with methods like `insert_characters`, `cursor_position`, `erase_in_display`
- One might conclude: "Python parses VT escape sequences and calls Screen methods to update the display"

#### Runtime Evidence That Falsifies This

1. **The VT parser is entirely C**: `kitty/vt-parser.c` is a ~1000-line C state machine that processes escape sequences. It includes `simd-string.h` (line 15) for SIMD-accelerated byte scanning — an optimization that only makes sense in C, not Python.

2. **The I/O thread is pure C**: The `parse_func` function pointer in `ChildMonitor` (line 61: `void (*parse_func)(void*, ParseData*, bool)`) is set to either `parse_worker` or `parse_worker_dump` (lines 180–181). Both are C functions. The I/O thread that runs `io_loop()` (created via `pthread_create` at line 291) calls this C function directly — Python's GIL is never involved in the parsing path.

3. **Single Python thread proof**: `sys.setswitchinterval(1000.0)` at `main.py` line 504 with the comment `"we have only a single python thread"` proves that Python never runs concurrent threads. Since the I/O thread runs `io_loop` (C), and the Main thread runs `run_main_loop` (C) — Python code only executes in the gaps between C calls on the Main thread. Python cannot be running a high-frequency parsing loop because it only has one thread and that thread is predominantly in C code.

4. **Screen model updates in C**: `kitty/screen.c` implements the screen operations in C. When the VT parser encounters an escape sequence like "cursor move" or "erase display", it calls C functions like `screen_cursor_position()` or `screen_erase_in_display()` directly — these are C-to-C function calls within the I/O thread, never crossing into Python.

5. **The `Screen` Python type is a C extension type**: `Screen` appears in Python (via `fast_data_types.Screen`) but is actually implemented as a C `PyTypeObject` in `screen.c`. Python can call methods on `Screen` objects, but the actual data manipulation happens in C. The `.pyi` stub file merely provides type annotations for Python static analysis — it does not indicate that the logic is written in Python.

#### The Correct Interpretation

Python **orchestrates** screen management at a high level — creating/destroying screens, associating them with windows, and reading their state for features like remote control. But the **hot-path data flow** — bytes from PTY → VT parser → screen model updates → rendering — is entirely in C, running on the I/O thread and Main thread without Python involvement. Python's role is supervisory, not participatory, in the parsing/rendering pipeline.

---

## Section 10 — Portability vs. Performance Tradeoff

### 10.1 The Core Tradeoff: Go Static Binary (Portable) vs. C SIMD Extensions (Performant)

Kitty's three-language architecture embodies a deliberate tradeoff between portability and performance, directly observable through the binary artifacts.

### 10.2 The Portability Side: Go's Self-Contained Binary

**[OBSERVED]** The `kitten` binary's dependency profile:

```bash
$ file kitty/launcher/kitten
...Go BuildID=..., stripped

$ ldd kitty/launcher/kitten
  libc.so.6   # Only system dependency

$ ls -la kitty/launcher/kitten
-rwxr-xr-x 1 root root 15761668 kitty/launcher/kitten  # 15 MB standalone
```

The Go `kitten` binary requires only `libc.so.6` at runtime. It can be:
- **Copied to any compatible Linux system** and executed immediately, with no installation of Python, FreeType, HarfBuzz, or OpenGL
- **Cross-compiled** for different platforms — `setup.py` line 1203 shows: `build_static_kittens(args, launcher_dir, args.dir_for_static_binaries, for_platform=(os_, arch))`
- **Distributed independently** — a user can download just the `kitten` binary to get CLI tools (icat, diff, ssh kitten, etc.) without installing the full kitty application

The Go runtime embedded in the binary provides:
- Garbage collection (no manual memory management)
- Goroutine scheduler (lightweight concurrency)
- Pure Go image processing (`github.com/kovidgoyal/imaging`, `golang.org/x/image`) — no system libpng/libjpeg needed
- Cross-platform syscall abstraction (`golang.org/x/sys`)

### 10.3 The Performance Side: C's Platform-Optimized Extensions

**[OBSERVED]** The `fast_data_types.so` dependency profile:

```bash
$ ldd kitty/fast_data_types.so
  libharfbuzz.so.0   # Text shaping
  libfreetype.so.6   # Glyph rasterization
  libpng16.so.16     # PNG decoding
  liblcms2.so.2      # Color management
  libcrypto.so.3     # Encryption
  # + libpython3.12, libm, libz, libc, and transitive deps

$ ls -la kitty/fast_data_types.so
-rwxr-xr-x 1 root root 1541408 kitty/fast_data_types.so  # 1.5 MB compiled C
```

The C extension requires **eight shared libraries** at runtime, each of which must be installed on the target system. In return, it provides:

1. **SIMD-accelerated VT parsing**: `kitty/simd-string-128.c` (SSE4.2) and `kitty/simd-string-256.c` (AVX2) provide vectorized byte scanning for the VT parser's inner loop. These intrinsics process 16 or 32 bytes simultaneously, compared to Go's byte-at-a-time processing.

2. **Direct OpenGL access**: `kitty/gl.c` loads OpenGL functions via GLAD, enabling direct GPU draw calls. The shader pipeline (`kitty/shaders.c` + 13 `.glsl` files) compiles and executes GLSL shaders for text cell rendering, border decoration, image compositing, and background effects.

3. **Native library integration**: FreeType for sub-pixel glyph rasterization, HarfBuzz for OpenType shaping (ligatures, kerning), and Fontconfig for system font discovery. These mature C libraries represent decades of optimization that no Go package can replicate.

4. **Platform-specific GLFW backends**: The vendored GLFW fork in `glfw/` provides separate backends for X11 (`glfw/x11_*.c`), Wayland (`glfw/wl_*.c`), and macOS Cocoa (`glfw/cocoa_*.m`). Each backend is compiled C tailored to the platform's windowing API, providing minimal-overhead event handling and OpenGL context management.

### 10.4 The Tradeoff in Practice

**[SOURCE-INFORMED]** Consider the image display flow for `kitty +kitten icat photo.png`:

1. **Go (portable)**: The `kitten` binary loads the image using pure Go libraries (`golang.org/x/image`), processes it (resize, color adjustment), encodes it in the kitty graphics protocol, and sends escape sequences to the terminal. **No system libraries needed** — the same binary works on any Linux system.

2. **C (performant)**: The kitty terminal receives the escape sequences on the I/O thread (`child-monitor.c`), decodes the image data via `kitty/graphics.c` using `libpng`, and uploads it to the GPU as a texture via OpenGL. The rendering pipeline (`kitty/shaders.c`) composites the image onto the terminal using a GPU shader (`kitty/graphics_fragment.glsl`). **This requires libpng, OpenGL, and a GPU** — but achieves hardware-accelerated rendering.

If Go were used for the rendering hot path instead of C:
- SIMD intrinsics (`_mm_cmpistri` for SSE4.2, `_mm256_cmpeq_epi8` for AVX2) would not be available — Go's compiler has limited auto-vectorization
- Direct OpenGL calls via GLAD would require CGO, losing Go's portability advantage
- Native FreeType/HarfBuzz integration would require CGO bridges, adding complexity
- The self-contained binary model would break — the binary would need shared library dependencies

Conversely, if C were used for the CLI kittens:
- Each kitten would need to link against system libraries
- Cross-compilation would be complex (different libraries per target)
- Distribution would require a full build environment on the target system
- The 15 MB standalone binary model would be impossible

### 10.5 Additional Tradeoff: GLFW Backend Complexity

The vendored GLFW fork demonstrates another dimension of this tradeoff:

- **C approach**: Separate backend implementations for X11 (`glfw/x11_init.c`, `glfw/x11_window.c`, `glfw/x11_monitor.c`), Wayland (`glfw/wl_init.c`, `glfw/wl_window.c`), and Cocoa (`glfw/cocoa_init.m`, `glfw/cocoa_window.m`). Each is ~2000–3000 lines of platform-specific C code compiled conditionally. This trades **code complexity** (maintaining three backends) for **platform performance** (each backend uses the native API directly).

- **Go approach**: The `kitten` binary uses `golang.org/x/sys` for cross-platform system calls, abstracting away platform differences into a single codebase. This trades **some performance** (abstraction overhead) for **portability** (one binary, all platforms).

> **The deliberate split**: Performance-critical rendering runs through platform-specific compiled C with SIMD and direct GPU access. Portable utility functionality runs through self-contained Go with cross-platform abstraction. Python orchestrates the boundary, keeping the high-level logic flexible while delegating hot paths to C and portable tools to Go.

---

## Section 11 — Full Command Transcripts (Appendix)

### A.1 Build Commands

```bash
# Build environment setup (performed by setup agent)
$ python3 --version
Python 3.12.3

$ go version
go version go1.22.10 linux/amd64

$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04.2) 13.3.0

# Full build invocation
$ export PATH=/usr/local/go/bin:$PATH
$ export KITTY_NO_LTO=1
$ python3 setup.py build --verbose --ignore-compiler-warnings
# (Output: compiled 49 .c files, built Go kitten binary, built C launcher)
# Build completed successfully

# Verify build artifacts
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x 1 root root  1541408 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 15761668 kitty/launcher/kitten
-rwxr-xr-x 1 root root    36224 kitty/launcher/kitty
```

### A.2 Binary Verification Commands

```bash
# Binary format identification
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  BuildID[sha1]=e8c64dd649a7e0353b70a45f1defd979b45f8b79,
  for GNU/Linux 3.2.0, not stripped

$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  Go BuildID=hqq4-U2LKlixbsjwYo2Y/n8c9tVmrH955DZP0gtLh/TlErkqS1Lkyxjr-Onw6b/O2yALBSe3QkGVIYlcExQ,
  stripped

$ file kitty/fast_data_types.so
kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
  dynamically linked, BuildID[sha1]=847633a229fed560d8483e9fe26d96f881a3c26a,
  not stripped

$ file kitty/glfw-x11.so
kitty/glfw-x11.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
  dynamically linked, BuildID[sha1]=6be30c4e8d075b0108dd2ea67e5e74356a0e0a20,
  not stripped

$ file kitty/glfw-wayland.so
kitty/glfw-wayland.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
  dynamically linked, BuildID[sha1]=618ee49779b013e85c47beadd5d79baf42f6fed8,
  not stripped

# Dynamic dependencies — kitty launcher
$ ldd kitty/launcher/kitty
  linux-vdso.so.1 (0x00007fff251cc000)
  libpython3.12.so.1.0 => /lib/x86_64-linux-gnu/libpython3.12.so.1.0
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6
  libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1
  libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1
  /lib64/ld-linux-x86-64.so.2

# Dynamic dependencies — kitten binary
$ ldd kitty/launcher/kitten
  linux-vdso.so.1 (0x00007fff6ef99000)
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  /lib64/ld-linux-x86-64.so.2

# Dynamic dependencies — fast_data_types.so
$ ldd kitty/fast_data_types.so
  libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6
  libpython3.12.so.1.0 => /lib/x86_64-linux-gnu/libpython3.12.so.1.0
  libharfbuzz.so.0 => /lib/x86_64-linux-gnu/libharfbuzz.so.0
  libpng16.so.16 => /lib/x86_64-linux-gnu/libpng16.so.16
  liblcms2.so.2 => /lib/x86_64-linux-gnu/liblcms2.so.2
  libcrypto.so.3 => /lib/x86_64-linux-gnu/libcrypto.so.3
  libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1
  libfreetype.so.6 => /lib/x86_64-linux-gnu/libfreetype.so.6
  libglib-2.0.so.0 => /lib/x86_64-linux-gnu/libglib-2.0.so.0
  libgraphite2.so.3 => /lib/x86_64-linux-gnu/libgraphite2.so.3
  libbz2.so.1.0 => /lib/x86_64-linux-gnu/libbz2.so.1.0
  libbrotlidec.so.1 => /lib/x86_64-linux-gnu/libbrotlidec.so.1
  libpcre2-8.so.0 => /lib/x86_64-linux-gnu/libpcre2-8.so.0
  libbrotlicommon.so.1 => /lib/x86_64-linux-gnu/libbrotlicommon.so.1

# Dynamic dependencies — GLFW X11 backend
$ ldd kitty/glfw-x11.so
  libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6
  libX11.so.6 => /lib/x86_64-linux-gnu/libX11.so.6
  libXcursor.so.1 => /lib/x86_64-linux-gnu/libXcursor.so.1
  libxkbcommon.so.0 => /lib/x86_64-linux-gnu/libxkbcommon.so.0
  libxkbcommon-x11.so.0 => /lib/x86_64-linux-gnu/libxkbcommon-x11.so.0
  libX11-xcb.so.1 => /lib/x86_64-linux-gnu/libX11-xcb.so.1
  libdbus-1.so.3 => /lib/x86_64-linux-gnu/libdbus-1.so.3
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6

# NEEDED entries from ELF dynamic section
$ readelf -d kitty/fast_data_types.so | grep NEEDED
  (NEEDED) Shared library: [libm.so.6]
  (NEEDED) Shared library: [libpython3.12.so.1.0]
  (NEEDED) Shared library: [libharfbuzz.so.0]
  (NEEDED) Shared library: [libpng16.so.16]
  (NEEDED) Shared library: [liblcms2.so.2]
  (NEEDED) Shared library: [libcrypto.so.3]
  (NEEDED) Shared library: [libz.so.1]
  (NEEDED) Shared library: [libc.so.6]

$ readelf -d kitty/launcher/kitten | grep NEEDED
  (NEEDED) Shared library: [libc.so.6]
```

### A.3 Symbol Inspection Commands

```bash
# Module entry point
$ nm -D kitty/fast_data_types.so | grep 'T PyInit'
0000000000029860 T PyInit_fast_data_types

# HarfBuzz symbols (text shaping)
$ nm -D kitty/fast_data_types.so | grep 'hb_'
  U hb_buffer_add_utf32
  U hb_buffer_create
  U hb_buffer_destroy
  U hb_buffer_get_glyph_infos
  U hb_buffer_get_glyph_positions
  U hb_shape
  U hb_ft_font_create
  # ... (22 total)

# FreeType symbols (glyph rasterization)
$ nm -D kitty/fast_data_types.so | grep 'FT_'
  U FT_Init_FreeType
  U FT_Done_FreeType
  U FT_Load_Glyph
  U FT_Render_Glyph
  U FT_Bitmap_Convert
  U FT_Bitmap_Done
  U FT_Done_Face
  # ...

# PNG symbols (image decoding)
$ nm -D kitty/fast_data_types.so | grep 'png_'
  U png_create_read_struct
  U png_read_image
  U png_get_image_width
  U png_get_image_height
  # ... (25 total)

# OpenSSL symbols (encryption)
$ nm -D kitty/fast_data_types.so | grep 'EVP_'
  U EVP_EncryptInit_ex
  U EVP_EncryptUpdate
  U EVP_EncryptFinal_ex
  U EVP_DecryptInit_ex
  U EVP_PKEY_derive
  U EVP_PKEY_keygen
  # ... (26 total)

# lcms2 symbols (color management)
$ nm -D kitty/fast_data_types.so | grep 'cms'
  U cmsCloseProfile
  U cmsCreateTransform
  U cmsCreate_sRGBProfile
  U cmsDeleteTransform
  U cmsDoTransform
  U cmsOpenProfileFromMem
  # (6 total)

# Kitten binary — stripped, no standard symbols
$ nm kitty/launcher/kitten
nm: kitty/launcher/kitten: no symbols

$ go tool nm kitty/launcher/kitten
reading kitty/launcher/kitten: no symbol section

# Go runtime strings (proves Go binary)
# Raw grep produces garbled entries first due to partial matches in stripped binary:
$ strings kitty/launcher/kitten | grep 'runtime\.' | head -10
runtime.
runtime.H9
runtime.H9
runtime.H9
runtime.H9
runtime.H
runtime.H
runtime.H9
runtime.H92
runtime.1

# Filtered for clean Go runtime function names:
$ strings kitty/launcher/kitten | grep -E '^runtime\.[a-z]' | head -10
runtime.cmpstring
runtime.memequal
runtime.memequal_varlen
runtime.init
runtime.init.func2
runtime.sigdelset
runtime.memhash8
runtime.memhash16
runtime.memhash128
runtime.memhash_varlen

# Kitty Go packages embedded
$ strings kitty/launcher/kitten | grep '^kitty/' | head -20
kitty/tools/cli
kitty/tools/tty
kitty/tools/tui
kitty/kittens/ssh
kitty/tools/utils
kitty/kittens/ask
kitty/tools/rsync
kitty/tools/config
kitty/tools/themes
kitty/tools/cmd/at
kitty/kittens/diff
kitty/kittens/icat
kitty/kittens/hints
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/wcswidth
kitty/kittens/themes
kitty/tools/utils/shm
kitty/tools/cli/markup
kitty/kittens/show_key

# Go build info (using go version -m for readable output; raw readelf -p .go.buildinfo
# contains ^I tab chars and hash checksums that are difficult to read)
$ go version -m kitty/launcher/kitten
kitty/launcher/kitten: go1.22.10
	path	kitty/tools/cmd
	mod	kitty	(devel)
	dep	github.com/ALTree/bigfloat	v0.2.0
	dep	github.com/alecthomas/chroma/v2	v2.14.0
	dep	github.com/bmatcuk/doublestar/v4	v4.6.1
	dep	github.com/disintegration/imaging	v1.6.2
	dep	github.com/dlclark/regexp2	v1.11.0
	dep	github.com/edwvee/exiffix	v0.0.0-20240229113213-0dbb146775be
	dep	github.com/google/uuid	v1.6.0
	dep	github.com/klauspost/cpuid/v2	v2.2.5
	dep	github.com/kovidgoyal/imaging	v1.6.3
	dep	github.com/rwcarlsen/goexif	v0.0.0-20190401172101-9e8deecbddbd
	dep	github.com/seancfoley/bintree	v1.3.1
	dep	github.com/seancfoley/ipaddress-go	v1.6.0
	dep	github.com/shirou/gopsutil/v3	v3.24.5
	dep	github.com/tklauser/go-sysconf	v0.3.12
	dep	github.com/tklauser/numcpus	v0.6.1
	dep	github.com/zeebo/xxh3	v1.0.2
	dep	golang.org/x/exp	v0.0.0-20230801115018-d63ba01acd4b
	dep	golang.org/x/image	v0.17.0
	dep	golang.org/x/sys	v0.21.0
	dep	howett.net/plist	v1.0.1
	build	-buildmode=exe
	build	-compiler=gc
	build	-ldflags="-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w"
	build	CGO_ENABLED=1
  # (14 direct deps + 6 indirect deps; google/go-cmp is test-only, excluded from binary)

# Go version embedded
$ strings kitty/launcher/kitten | grep 'go1\.'
go1.22.10
```

### A.4 Python Runtime Inspection

```bash
# fast_data_types module introspection
$ python3 -c "
import sys; sys.path.insert(0, '.')
import kitty.fast_data_types as fdt
attrs = [a for a in dir(fdt) if not a.startswith('_')]
print(f'Total public attributes: {len(attrs)}')
print('Programs:', [a for a in attrs if 'PROGRAM' in a])
print('Crypto:', [a for a in attrs if 'AES' in a or 'Elliptic' in a])
print('Monitor:', [a for a in attrs if 'Monitor' in a])
"
Total public attributes: 581
Programs: ['BGIMAGE_PROGRAM', 'BORDERS_PROGRAM', 'CELL_BG_PROGRAM', 'CELL_FG_PROGRAM',
           'CELL_PROGRAM', 'CELL_SPECIAL_PROGRAM', 'GRAPHICS_ALPHA_MASK_PROGRAM',
           'GRAPHICS_PREMULT_PROGRAM', 'GRAPHICS_PROGRAM', 'TINT_PROGRAM']
Crypto: ['AES256GCMDecrypt', 'AES256GCMEncrypt', 'EllipticCurveKey']
Monitor: ['ChildMonitor']
```

### A.5 Version Verification

```bash
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

### A.6 Launch Attempt (Blocked)

```bash
$ kitty/launcher/kitty --listen-on unix:/tmp/kitty-test.sock
[0.059] [glfw error 65544]: X11: The DISPLAY environment variable is missing
GLFW initialization failed

# Analysis: Expected failure. The GLFW initialization in the C layer (via init_glfw()
# called from kitty/main.py line 514) requires a display server. The X11 backend
# reads $DISPLAY, which is not set in this headless CI environment. This confirms
# the C layer's dependency on a platform windowing system for the rendering pipeline.
```

### A.7 Strace on Kitten Binary

```bash
$ strace -f -e trace=write,read -c kitty/launcher/kitten --version 2>&1
strace: Process 44785 attached
strace: Process 44786 attached
strace: Process 44787 attached
strace: Process 44788 attached
strace: Process 44789 attached
strace: Process 44790 attached
strace: Process 44791 attached
strace: Process 44792 attached
strace: Process 44793 attached
strace: Process 44794 attached
kitten 0.35.2 created by Kovid Goyal
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000223          22        10           read
  0.00    0.000000           0         1           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.000223          20        11           total
```

> *Note: The 10 "Process ... attached" messages confirm the Go runtime spawns multiple OS threads (goroutine scheduler, GC, timers) even for a trivial invocation. PIDs will vary between runs.*

### A.8 Remote Control Commands (Could Not Execute)

The following commands would be issued in a full environment with a running kitty instance:

```bash
# Would require: kitty running with --listen-on unix:/tmp/kitty-test.sock

# List windows/tabs
$ kitty @ --to unix:/tmp/kitty-test.sock ls
# Expected: JSON tree of OS windows → tabs → windows with PID, title, dimensions

# Get current colors
$ kitty @ --to unix:/tmp/kitty-test.sock get-colors
# Expected: key-value pairs of color names and hex values

# Get screen text
$ kitty @ --to unix:/tmp/kitty-test.sock get-text --extent screen
# Expected: plain text content of the active terminal window

# Blocked because: GLFW initialization failed (no display server)
# Fallback: Source code analysis of kitty/rc/ls.py, get_colors.py, get_text.py
```

### A.9 Process Observation Commands (Could Not Execute)

```bash
# Would require: running kitty process

# Thread enumeration
$ ls /proc/$(pgrep kitty)/task/
# Expected: 3 directories (Main, I/O, Talk threads)

# Memory map
$ cat /proc/$(pgrep kitty)/maps | grep '\.so'
# Expected: fast_data_types.so, glfw-x11.so, libpython, libharfbuzz, libfreetype, ...

# Process tree during icat
$ kitty +kitten icat photo.png &
$ pstree -p $(pgrep kitty)
# Expected: kitty(PID)───zsh(PID2)───kitten(PID3)

# Blocked because: GLFW initialization failed (no display server)
# Fallback: ldd analysis of compiled binaries reconstructs the expected memory map
```

### A.10 Stress Test Commands (Could Not Execute)

```bash
# Would require: running kitty process

# Generate colored output stress
$ for i in $(seq 1 10000); do printf "\033[38;5;$((i % 256))m█"; done

# Scrollback churn
$ seq 1 1000000

# Rapid resize (via remote control)
$ for i in $(seq 1 100); do
    kitty @ --to unix:/tmp/kitty-test.sock resize-window --increment 2
    sleep 0.01
  done

# Blocked because: GLFW initialization failed (no display server)
# Fallback: Source code analysis of rendering hot path in child-monitor.c
```

---

## Summary of Findings

### The Three-Language Division

| Language | Process Model | Responsibilities | Key Evidence |
|---|---|---|---|
| **C** (49 files, 1.5 MB .so) | In-process (extension module) | VT parsing, OpenGL rendering, font rasterization, GPU glyph cache, SIMD string ops, threading, platform windowing | `ldd` shows libharfbuzz/libfreetype/libpng linkage; `nm -D` shows 104 imported rendering symbols; `pthread_create` in source |
| **Python** (orchestration layer) | In-process (interpreter) | Startup, config, window/tab management, RC dispatch, kitten framework, shader loading | `sys.setswitchinterval(1000.0)` proves single thread; `ldd kitty` shows libpython; entry_points.py routes all dispatch |
| **Go** (15 MB binary) | Separate process | CLI tools, kittens (icat, diff, ssh, etc.), RC client, shell completion | `file` shows Go BuildID; `ldd` shows only libc; `strings` reveal Go runtime and kitty packages; `os.execl` in entry_points.py |

### Key Architectural Insights

1. **C and Python share one process** — C runs as Python extension modules (`fast_data_types.so`), called through the CPython C API.
2. **Go always runs as a separate process** — invoked via `os.execl`/`os.execvp`, communicating via escape sequences or UNIX sockets.
3. **The 2–3 C threads (Main/IO, and optionally Talk when RC is enabled) are created in C** — Python never creates threads; `sys.setswitchinterval(1000.0)` confirms single-Python-thread design.
4. **The rendering hot path is entirely C** — from VT parsing (with SIMD) through OpenGL rendering, never touching Python.
5. **The Go layer trades rendering capability for portability** — the 15 MB standalone binary carries its own runtime but has zero rendering libraries.
