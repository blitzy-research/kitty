# kitty — Architecture Onboarding Q&A (runtime-grounded)

> Repository: `kovidgoyal/kitty` @ commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> Source branch: `kitty_815df1e210e0` (this document's name is derived from it)
> Product version observed at runtime (native build, non-canonical env): **`kitty 0.35.2 created by Kovid Goyal`**

This document answers four onboarding questions about kitty. Every behavioral claim is
grounded either in a specific `file:line` reference **or** in actual, complete, unedited
command output captured from a **running** instance — the code paths were **built and run
first**, then the answers were written from what was observed. Every statement that could not
be observed directly is labelled **(inferred)**; every value obtained from a non-canonical
route (a bypassing interface, a fallback, a synthetic stand-in, or a build performed outside
the nominated Docker image) is labelled **(non-canonical)**.

---

## The four questions (verbatim)

- **Q1.** *kitty calls itself "GPU accelerated" yet the codebase is a weave of Python and C — which language does the heavy lifting once it is running, and what does that say about where the performance actually comes from?*
- **Q2.** *there are GLSL files scattered around; shader code inside a terminal is unexpected — what role do these files play and how central are they to the system?*
- **Q3.** *running the main entry point directly fails almost immediately with a cryptic error — what exactly is missing at that moment, and what does that tell us about how Python is wired into the native core (there is "one critical piece that everything depends on")?*
- **Q4.** *the `kittens/` directory looks like small self-contained tools — are they truly independent or do they quietly rely on the same native bridge, and what happens if you run one standalone?*

## TL;DR

| Q | One-line answer |
|---|-----------------|
| Q1 | The compiled C extension **`kitty.fast_data_types`** runs the hot paths (screen model, VT parsing, font shaping, PTY I/O thread) and a **GPU** OpenGL/GLSL pipeline computes the pixels; Python only orchestrates. Performance comes from the **native core + GPU**, not Python. (On this headless host the GPU role is filled by Mesa `llvmpipe` **software** rasterization — hardware-GPU behavior is inferred.) |
| Q2 | The **13** `kitty/*.glsl` files are the **GPU render programs** that draw every terminal cell, cursor, selection, border, image and tint. They are **load-bearing**: a deliberately broken cell shader makes kitty refuse to start (shown below). |
| Q3 | The first missing piece is the compiled CPython C-extension **`kitty.fast_data_types`** (the `.so`), which is `*.so`-gitignored and produced only by the build; the first `from .fast_data_types import ...` raises `ModuleNotFoundError`. **80** Python files mention the string; **77** actually import it (**59** under production `kitty/`/`kittens/`). After building, the same bare `python3 __main__.py` no longer raises that error but then fails on `sys.kitty_run_data`, which only the native launcher populates (shown below). |
| Q4 | The kittens are **not** independent — each Python `main.py` transitively (and sometimes directly) needs `kitty.fast_data_types`, so a standalone run fails with the same `ModuleNotFoundError`. For the **12** launcher-"wrapped" kittens, `kitty +kitten <name>` is served by a separate **Go `kitten` binary** (observed **dynamically linked**, `CGO_ENABLED=1` — not static); non-wrapped kittens fall back to the Python runner. |

The unifying thread across **Q1, Q3, Q4** is the *native bridge* `kitty.fast_data_types`:
Q1 shows it does the heavy lifting, Q3 shows the Python front-end cannot start without it,
Q4 shows even the "small tools" depend on it. **Q2** is the other half of performance — the
GLSL programs that run on the GPU.

---

## Methodology & environment (how these answers were produced)

**Run-first.** For every question the relevant code path was executed and its exact, complete
output captured (with an explicit exit code). Two repository states are used:

- **Unbuilt state** — a pristine checkout with **no build artifacts**, exported with
  `git archive HEAD | tar -x -C <dir>` (this yields *tracked files only*, exactly what a fresh
  `git clone` gives you before building). Used for the Q3/Q4 `ModuleNotFoundError` observations.
- **Built state** — the same tree after the build, used for the version banner, the launch
  behaviors, the GPU/shader/thread observations, and the Go `kitten` path.

All temporary trees live under `/tmp` (outside the repository) in uniquely-named,
freshly-emptied directories, and are removed afterward. Neutral `/tmp` paths are used
throughout so no host or workspace identifiers appear in the pasted output.

**Build command.** The build is `python3 setup.py` (the `Makefile` `all:` target is
`python3 setup.py $(VVAL)`):

```text
$ sed -n '12,13p' Makefile
all:
	python3 setup.py $(VVAL)
```

> **Environment honesty — the results below are (non-canonical).** The task nominates the
> Docker image
> `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> as the canonical build/run environment. That image is **not reachable** here — the access
> attempt fails:
>
> ```text
> $ docker pull andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
> Error response from daemon: pull access denied for andrewparkscaleai/coding-agent, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
> ; exit=1
> ```
>
> The canonical toolchain was therefore **replicated natively**, and the build command used was
> `CFLAGS=-Wno-error=switch python3 setup.py`. The `CFLAGS` demotes **only** the specific
> `-Werror=switch` promotion that a newer `wayland-protocols` triggers in `glfw/wl_window.c`
> (it is *not* the same as the broader `--ignore-compiler-warnings`, which disables
> warning-as-error entirely); it changes no tracked source and affects only the Wayland GLFW
> backend, not the C extension or any answer here. **Because this is not the nominated image,
> every build-dependent value below (version banner, launch behavior, thread counts, Go
> linkage) is labelled (non-canonical).** No output is fabricated; anything not directly
> observed is labelled (inferred).

**Environment recorded** (only the non-sensitive toolchain fields; no host/kernel identifiers):

```text
$ python3 --version ; go version ; gcc --version | head -1 ; pkg-config --version
Python 3.13.7
go version go1.24.4 linux/amd64
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
1.8.1
```

Runtime versions (Python 3.13.7, Go 1.24.4) satisfy the project minimums declared in the
manifests (`pyproject.toml:L2` `requires-python = ">=3.8"`; `go.mod:L3` `go 1.22`).

**Default-config version banner** (built state, default configuration; **non-canonical** —
produced by the native build, not the nominated Docker image):

```text
$ ./kitty/launcher/kitty --version ; echo "exit=$?"
kitty 0.35.2 created by Kovid Goyal
exit=0
$ python3 __main__.py --version ; echo "exit=$?"
kitty 0.35.2 created by Kovid Goyal
exit=0
```

The Go build stamps the same VCS revision as the branch, confirming the tree identity
(`go version -m`, non-canonical native build): `vcs.revision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`,
`vcs.modified=false` (full output in Q4).

**Repository integrity (tracked vs. filesystem).** The **tracked** git tree is byte-for-byte
unchanged except for this one added file:

```text
$ git status --porcelain
$ git ls-files blitzy/
blitzy/documentation/kitty_815df1e210e0.md
```

`git status --porcelain` prints nothing (no modified/untracked tracked-tree files); the only
tracked addition is this document. The **filesystem** additionally contains gitignored build
artifacts produced by the `setup.py` build during environment setup (`kitty/fast_data_types.so`,
`kitty/launcher/kitty`, `kitty/launcher/kitten`, `__pycache__/`, generated sources) — these are
excluded by `.gitignore` (line 1 `*.so`; line 18 `/kitty/launcher/kitt*`) and are **not** part
of the tracked tree:

```text
$ git check-ignore kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
kitty/fast_data_types.so
kitty/launcher/kitty
kitty/launcher/kitten
```

No task-created scratch file is left inside the repository; every observation tree/script used
below lives under `/tmp`.

### The native bridge at a glance

```mermaid
graph TD
    subgraph PY["Python front-end (orchestration)"]
        M["__main__.py:7  main()"] --> EP["kitty/entry_points.py:194"]
        EP --> MAIN["kitty/main.py:11  import .borders"]
        MAIN --> B["kitty/borders.py:7  from .fast_data_types import ..."]
        K["kittens/hints/main.py:9  import kitty.clipboard"] --> CL["kitty/clipboard.py:11 -> kitty/conf/utils.py:27"]
    end
    subgraph NATIVE["Native core (compiled, the engine)"]
        FDT["kitty.fast_data_types (.so)\ndata-types.c:525 PyInit_fast_data_types\nlinked from 62 extension C sources (setup.find_c_files, Linux)"]
        GLSL["13 GLSL shaders\ncompiled via gl.c:110 glCompileShader"]
    end
    B -->|import| FDT
    CL -->|import| FDT
    K -->|direct import L11| FDT
    FDT --> GLSL
    LAUNCH["kitty/launcher/main.c\nembeds CPython (Py_InitializeFromConfig:211)\nsets sys.kitty_run_data (main.c:214)"] --> PY
    GO["Go 'kitten' binary (dynamically linked)\nkitten_exe() constants.py:83-84"] -. separate artifact, wrapped kittens only .-> KITTENRUN["kitty +kitten <name>"]
%% In an unbuilt tree FDT is absent, so every import edge into it raises ModuleNotFoundError.
%% After building, a bare 'python3 __main__.py' still needs sys.kitty_run_data, set only by the launcher.
```

---

## Q1 — Which language does the heavy lifting, and where does performance come from?

> *kitty calls itself "GPU accelerated" yet the codebase is a weave of Python and C — which language does the heavy lifting once it is running, and what does that say about where the performance actually comes from?*

### Answer

Once kitty is running, the heavy lifting is done by the **compiled C extension
`kitty.fast_data_types`** (the terminal screen model, VT/escape parsing, font shaping,
graphics, PTY I/O) and by an **OpenGL/GLSL GPU pipeline** (every pixel is produced by shader
programs — see Q2). **Python is a thin orchestration front-end**: it parses the command line,
loads config, and manages windows/tabs, then hands the work to the native core. A **third**
language, **Go**, implements the modern kittens (see Q4). So the "weave of Python and C" is
really a layered engine: **Python orchestrates -> C (`fast_data_types`) runs the hot paths ->
the GPU (GLSL) computes the pixels**. Performance therefore comes from the **native core +
GPU**, not from Python.

> On this headless host the GPU stage is served by Mesa **`llvmpipe`**, which is **CPU
> software rasterization** (forced via `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`). The
> render pipeline and the OpenGL context are therefore observed for real, but the claim that
> the pixels run on *hardware* silicon is **(inferred)** — on a machine with a real GPU the
> same GL/GLSL path executes on the GPU instead of on `llvmpipe` threads.

### The Python front-end pulls its machinery from the compiled extension

`kitty/main.py` begins with standard-library imports; its **first project-local /
native-dependent import** is at `kitty/main.py:L11`, and the block at `kitty/main.py:L32`
imports the window/GPU/PNG/font machinery straight out of the C extension:

```text
$ sed -n '11p;32,45p' kitty/main.py
from .borders import load_borders_program
from .fast_data_types import (
    GLFW_MOD_ALT,
    GLFW_MOD_SHIFT,
    SingleKey,
    create_os_window,
    free_font_data,
    glfw_init,
    glfw_terminate,
    load_png_data,
    mask_kitty_signals_process_wide,
    set_custom_cursor,
    set_default_window_icon,
    set_options,
)
```

`load_borders_program` (imported at L11 from `.borders`) is itself native-dependent — see Q3,
where that import is the first hop that reaches `fast_data_types`. The L32 block pulls
`create_os_window` (L36), `glfw_init` (L38), `load_png_data` (L40) and `set_options` (L44)
directly from the C extension.

### The module identity is declared in C

`kitty/data-types.c` is the C translation unit that **declares** the extension module and its
initializer:

```text
$ sed -n '467,469p' kitty/data-types.c ; echo '...' ; sed -n '524,525p' kitty/data-types.c
static struct PyModuleDef module = {
    .m_base = PyModuleDef_HEAD_INIT,
    .m_name = "fast_data_types",   /* name of module */
...
EXPORTED PyMODINIT_FUNC
PyInit_fast_data_types(void) {
```

The function CPython calls to bring the module to life is **`PyInit_fast_data_types`**
(`kitty/data-types.c:L525`); the module name is literally `fast_data_types`
(`kitty/data-types.c:L469`). It aggregates the many C subsystems. Two *distinct* observed
numbers describe that aggregation (they are **not** the same and should not be blurred):

```text
$ sed -n '476,500p' kitty/data-types.c | grep -cE '^extern .*init_'
25
$ grep -cE 'if \(!init_' kitty/data-types.c
32
```

So there are **25 `extern ... init_*` declarations** (`kitty/data-types.c:L476`-`L500`) versus
**32 `if (!init_...)` call sites** inside `PyInit_fast_data_types`. The call-site count is larger
because the block contains **both** the `__APPLE__` branch (`init_CoreText`, `init_cocoa`,
`init_macos_process_info`) and the non-Apple branch (`init_freetype_library`,
`init_fontconfig_library`, `init_desktop`, `init_freetype_render_ui_text`); `grep` counts both
even though only one set compiles on a given platform.

### The extension is one shared object built from many C sources

The single `fast_data_types` shared object is linked from the C sources that
`setup.find_c_files()` returns. On Linux that set is **62** sources (49 under `kitty/` plus 13
under `3rdparty/`), which is distinct from the repository-wide tracked `.c` count of **128**:

```text
$ git ls-files '*.c' | wc -l
128
$ python3 - <<'PY'
import os, glob
# Reproduce setup.find_c_files() for Linux (is_macos == False)
exclude = {'core_text.m','cocoa_window.m','macos_process_info.c'}
ans = []
for x in sorted(os.listdir('kitty')):
    ext = os.path.splitext(x)[1]
    if ext in ('.c','.m') and os.path.basename(x) not in exclude:
        ans.append('kitty/'+x)
ans.append('kitty/vt-parser-dump.c')
ans.append('3rdparty/ringbuf/ringbuf.c')
ans.extend(glob.glob('3rdparty/base64/lib/arch/*/codec.c'))
ans += ['3rdparty/base64/lib/tables/tables.c','3rdparty/base64/lib/codec_choose.c','3rdparty/base64/lib/lib.c']
print('find_c_files() extension sources (Linux):', len(ans))
PY
find_c_files() extension sources (Linux): 62
```

### At runtime the "heavy" functions are native, and the C libraries are linked in

Importing the module in the built tree shows it resolves to a **compiled shared object**, and
the imported "heavy" callables are native C functions (`builtin_function_or_method`), not
Python functions (run from the neutral built tree `/tmp/kitty_built`):

```text
$ python3 -c "
import kitty.fast_data_types as f
print('module file :', f.__file__)
print('module type :', type(f).__name__)
for name in ('create_os_window','glfw_init','load_png_data','set_options'):
    print(f'{name:16s}: {type(getattr(f,name)).__name__}')
"
module file : /tmp/kitty_built/kitty/fast_data_types.so
module type : module
create_os_window: builtin_function_or_method
glfw_init       : builtin_function_or_method
load_png_data   : builtin_function_or_method
set_options     : builtin_function_or_method

$ file kitty/fast_data_types.so
kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=de56de664dde21b23a324685eb08bf0032548704, not stripped

$ nm -D kitty/fast_data_types.so | grep -i PyInit
000000000002a0a0 T PyInit_fast_data_types
```

The exported symbol `PyInit_fast_data_types` is exactly the C function at
`kitty/data-types.c:L525`. Representative **hot-path** functions are compiled into the same
`.so` (the build uses LTO, so some symbols are inlined/renamed):

```text
$ nm kitty/fast_data_types.so | grep -iE ' t (init_glfw|parse_worker|shape_run)$' | sort
0000000000031c20 t shape_run
000000000005b860 t init_glfw
00000000000c43c0 t parse_worker
```

`shape_run` is font shaping (`kitty/fonts.c`), `parse_worker` is VT escape-sequence parsing
(`kitty/vt-parser.c`), `init_glfw` is windowing/GL init. And the native graphics/font/crypto C
libraries are linked **into** the extension — this is where the real work lives:

```text
$ ldd kitty/fast_data_types.so | grep -iE 'python|harfbuzz|freetype|png|lcms|crypto'
	libpython3.13.so.1.0 => /lib/x86_64-linux-gnu/libpython3.13.so.1.0 (0x00007f65aeb56000)
	libharfbuzz.so.0 => /lib/x86_64-linux-gnu/libharfbuzz.so.0 (0x00007f65aea19000)
	libpng16.so.16 => /lib/x86_64-linux-gnu/libpng16.so.16 (0x00007f65af9e2000)
	liblcms2.so.2 => /lib/x86_64-linux-gnu/liblcms2.so.2 (0x00007f65ae9b2000)
	libcrypto.so.3 => /lib/x86_64-linux-gnu/libcrypto.so.3 (0x00007f65ae39f000)
	libfreetype.so.6 => /lib/x86_64-linux-gnu/libfreetype.so.6 (0x00007f65ae03e000)
```

One native library is conspicuously **absent** from that `ldd` output: `fontconfig`. That is
not an omission. `setup.py` discovers fontconfig through `pkg-config` at **build** time, but the
code loads it at **runtime** via `dlopen`, so it is not recorded as a link-time dependency of
the `.so`. The loader is `load_fontconfig_lib` at `kitty/fontconfig.c:L79`-`L94`, which
`dlopen`s `libfontconfig.so` (or `libfontconfig.so.1`) with `RTLD_LAZY` and calls
`fatal('Failed to find and load fontconfig')` if neither can be found:

```text
$ sed -n '79,94p' kitty/fontconfig.c
load_fontconfig_lib(void) {
        const char* libnames[] = {
#if defined(_KITTY_FONTCONFIG_LIBRARY)
            _KITTY_FONTCONFIG_LIBRARY,
#else
            "libfontconfig.so",
            // some installs are missing the .so symlink, so try the full name
            "libfontconfig.so.1",
#endif
            NULL
        };
        for (int i = 0; libnames[i]; i++) {
            libfontconfig_handle = dlopen(libnames[i], RTLD_LAZY);
            if (libfontconfig_handle) break;
        }
        if (libfontconfig_handle == NULL) { fatal("Failed to find and load fontconfig"); }
$ ldd kitty/fast_data_types.so | grep -i fontconfig ; echo "grep_exit=$?"
grep_exit=1
```

So `fontconfig` never appears in `ldd` (the `grep` exits 1, no match) even though the extension
uses it heavily for font discovery - it is a runtime `dlopen`, not a transitive link dependency.

The representative hot-path C translation units (all compiled into `fast_data_types.so`) and
their measured sizes:

```text
$ for f in kitty/screen.c kitty/glfw.c kitty/graphics.c kitty/child-monitor.c kitty/fonts.c kitty/shaders.c kitty/vt-parser.c; do printf "%-24s %s\n" "$f" "$(wc -c < $f)"; done
kitty/screen.c           199784
kitty/glfw.c             102579
kitty/graphics.c         101377
kitty/child-monitor.c    76612
kitty/fonts.c            76153
kitty/shaders.c          62093
kitty/vt-parser.c        55306
```

| C source | bytes | responsibility |
|----------|------:|----------------|
| `kitty/screen.c` | 199,784 | terminal screen model / scrollback (hot path) |
| `kitty/glfw.c` | 102,579 | OS window + OpenGL context |
| `kitty/graphics.c` | 101,377 | graphics-protocol (inline images) |
| `kitty/child-monitor.c` | 76,612 | non-blocking PTY I/O thread |
| `kitty/fonts.c` | 76,153 | font shaping / rasterization / atlas |
| `kitty/shaders.c` | 62,093 | OpenGL program compile/link (Q2) |
| `kitty/vt-parser.c` | 55,306 | escape-sequence parsing (SIMD-accelerated) |

SIMD evidence: `kitty/simd-string-128.c` and `kitty/simd-string-256.c` are present. After the
license header each is a thin wrapper that selects a vector width and includes a shared
template — e.g. the payload of `kitty/simd-string-128.c` is:

```text
$ tail -2 kitty/simd-string-128.c
#define KITTY_SIMD_LEVEL 128
#include "simd-string-impl.h"
```

with the runtime dispatch in `kitty/simd-string.c`. The Python-visible surface of the module,
`kitty/fast_data_types.pyi` (35,795 bytes, tracked), is a **type stub only** — it documents the
API for type-checkers and is present even when the compiled `.so` is not; it is **not** the
implementation.

### Observed at runtime: C threads + the GPU stage do the work, Python coordinates

Launching the built kitty headless (software GL via `llvmpipe`) shows the GL render context
coming up and the child process being spawned by the C core. **Complete, unedited** output
(the child's own stdout goes to the PTY and is rendered inside the window, so it does not
appear on kitty's stdout — itself proof of real terminal emulation rather than byte-piping):

```text
$ LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe xvfb-run -a ./kitty/launcher/kitty --config NONE --debug-rendering sh -c 'echo BUILT_TREE_CHILD_OK' ; echo "exit=$?"
[0.147] OS Window created
[0.156] Failed to open systemd user bus with error: Connection refused
[0.160] Child launched
[0.124] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
exit=0
```

The observed `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2'` is the `llvmpipe` **software** GL
implementation (observed), not a hardware driver. Inspecting the running process shows the
concurrency lives in the **native** layer, not in Python (whose GIL keeps Python code
effectively single-threaded). This host has `nproc=4`; the thread count is **stable across two
measurements** ~8s apart:

```text
$ xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe ./kitty/launcher/kitty --config NONE sh -c 'sleep 90' &
$ KPID=$(ps -eo pid,comm | awk '$2=="kitty"{print $1; exit}')   # the kitty process
# --- measurement 1 (t ~= 8s) ---
$ grep '^Threads:' /proc/$KPID/status
Threads:	67
$ cat /proc/$KPID/task/*/comm | sed -E 's/-?[0-9]+$//' | sort | uniq -c | sort -rn
     33 kitty
     32 llvmpipe
      1 kitty:disk$
      1 KittyChildMon
# --- measurement 2 (t ~= 16s), stability ---
$ grep '^Threads:' /proc/$KPID/status
Threads:	67
```

Interpretation, separating observation from inference:
- **Observed (software GL):** of the 67 threads, **32 are `llvmpipe-*`** — the Mesa **CPU
  software rasterizer**. On this headless host they stand in for the GPU. On real hardware
  that rasterization runs on the GPU and these CPU threads would not exist **(inferred)**.
- **Observed (native, not GPU-related):** `KittyChildMon` is the non-blocking PTY I/O thread
  and `kitty:disk$` is the disk-cache writer — both created in C.

The `KittyChildMon` thread is created and named in C — `kitty/child-monitor.c:L291`
`ret = pthread_create(&self->io_thread, NULL, io_loop, self);` and `kitty/child-monitor.c:L1489`
`set_thread_name("KittyChildMon");` (`set_thread_name` itself is defined at
`kitty/threading.h:L26`).

### Reconciling "GPU accelerated"

The claim is literal at the API level: kitty creates an OpenGL 4.5 core-profile context
(observed above) and draws the terminal with GLSL shader programs (Q2). Python never touches a
pixel. On this host the OpenGL implementation is `llvmpipe` (CPU software); the statement that
the same pipeline runs on dedicated GPU silicon is **(inferred)** for a hardware-GPU machine.

### Cause -> effect

- **Cause:** the hot paths (screen model, VT parsing, shaping, PTY I/O) are compiled C inside
  `fast_data_types.so`, and rendering is delegated to an OpenGL/GLSL pipeline.
- **Effect:** the observable work at runtime is native C threads (`KittyChildMon`, disk cache)
  plus the GL rasterizer (here `llvmpipe`, 32 CPU threads; a hardware GPU on real silicon,
  inferred); Python only orchestrates (`kitty/main.py`, `kitty/boss.py`, `kitty/constants.py`).
  **Performance comes from the native core + GPU, not Python.**

---

## Q2 — What role do the GLSL files play, and how central are they?

> *there are GLSL files scattered around; shader code inside a terminal is unexpected — what role do these files play and how central are they to the system?*

### Answer

The `.glsl` files are the **GPU render programs**: they are the code that actually draws
every terminal cell (text, background, cursor, selection), the window borders/splits, the
kitty graphics-protocol images, the background image, and the dim/tint overlay. They are not
incidental — they are **load-bearing**: with a broken cell shader the terminal does not start
at all (shown below). Python (`kitty/shaders.py`) loads and preprocesses the `.glsl` source as
a package resource; C (`kitty/shaders.c` -> `kitty/gl.c`) hands it to the GPU driver to compile
and link into an OpenGL program.

### There are exactly 13 shader files, all under `kitty/`

```text
$ ls -1 kitty/*.glsl ; echo "count=$(ls -1 kitty/*.glsl | wc -l)"
kitty/alpha_blend.glsl
kitty/bgimage_fragment.glsl
kitty/bgimage_vertex.glsl
kitty/border_fragment.glsl
kitty/border_vertex.glsl
kitty/cell_defines.glsl
kitty/cell_fragment.glsl
kitty/cell_vertex.glsl
kitty/graphics_fragment.glsl
kitty/graphics_vertex.glsl
kitty/linear2srgb.glsl
kitty/tint_fragment.glsl
kitty/tint_vertex.glsl
count=13
```

Grouped by what they draw:

| Shader file(s) | Role |
|----------------|------|
| `cell_vertex.glsl`, `cell_fragment.glsl`, `cell_defines.glsl` | The terminal text grid — the core cell renderer (text, cursor, selection). `cell_defines.glsl` is a shared include. |
| `border_vertex.glsl`, `border_fragment.glsl` | Window borders / split dividers |
| `graphics_vertex.glsl`, `graphics_fragment.glsl` | kitty graphics-protocol images (inline images) |
| `bgimage_vertex.glsl`, `bgimage_fragment.glsl` | Background image |
| `tint_vertex.glsl`, `tint_fragment.glsl` | Dim/tint overlay |
| `alpha_blend.glsl`, `linear2srgb.glsl` | Shared includes (blending, sRGB color conversion) |

### Python side: `kitty/shaders.py` loads and preprocesses the `.glsl` source

Each `Program` derives its file names by convention and loads the source as a **package
resource**. `kitty/shaders.py:L54`-`L55` set the names:

```text
$ sed -n '54,57p' kitty/shaders.py
        self.vertex_name = vertex_name or f'{name}_vertex.glsl'
        self.fragment_name = fragment_name or f'{name}_fragment.glsl'
        self.original_vertex_sources = tuple(self._load_sources(self.vertex_name, set()))
        self.original_fragment_sources = tuple(self._load_sources(self.fragment_name, set()))
```

`_load_sources` (`kitty/shaders.py:L61`) reads each file via `read_kitty_resource(name)`
(`kitty/shaders.py:L68`, defined at `kitty/constants.py:L241`), resolving
`#pragma kitty_include_shader <...>` includes recursively. Loading the `cell` program in the
built tree shows the actual result (the first source chunk is the injected `#version` line, and
the whole assembled vertex source is 9,297 bytes):

```text
$ python3 - <<'PY'
from kitty.shaders import program_for
p = program_for('cell')
v = p.original_vertex_sources
f = p.original_fragment_sources
print('program name     :', p.name)
print('vertex_name      :', p.vertex_name)
print('fragment_name    :', p.fragment_name)
print('vertex   chunks  :', len(v))
print('fragment chunks  :', len(f))
print('assembled vbytes :', len('\n'.join(v)))
print('vertex chunk[0]  :', repr(v[0]))
PY
program name     : cell
vertex_name      : cell_vertex.glsl
fragment_name    : cell_fragment.glsl
vertex   chunks  : 4
fragment chunks  : 7
assembled vbytes : 9297
vertex chunk[0]  : '#version 140\n'
```

The `cell` program is registered at `kitty/shaders.py:L152` (`cell = program_for('cell')`),
confirming this is the real renderer for the terminal grid, not a demo.

### C side: `kitty/shaders.c` -> `kitty/gl.c` compiles and links on the GPU

The preprocessed source is handed to `compile_shaders` at `kitty/shaders.c:L1160`:

```text
$ sed -n '1160p' kitty/shaders.c
    GLuint shader_id = compile_shaders(shader_type, PyTuple_GET_SIZE(sources), c_sources);
```

which is defined in `kitty/gl.c:L107` and issues the actual OpenGL calls
(`glCreateShader` L108, `glShaderSource` L109, `glCompileShader` L110):

```text
$ sed -n '106,110p' kitty/gl.c
GLuint
compile_shaders(GLenum shader_type, GLsizei count, const GLchar * const * source) {
    GLuint shader_id = glCreateShader(shader_type);
    glShaderSource(shader_id, count, source, NULL);
    glCompileShader(shader_id);
```

The GLSL language version the driver is told to use is exported to Python as `GLSL_VERSION`
(`kitty/shaders.c:L1254`, `C(GLSL_VERSION);`).

### Proof they are load-bearing (SYNTHETIC / NON-CANONICAL)

To show the shaders are central rather than decorative, I compiled kitty normally, then in a
**disposable copy** of the tree appended one invalid token to `kitty/cell_vertex.glsl` and
launched again. Modifying a source file makes this run **non-canonical / synthetic** by
construction; it is done in a `mktemp` copy that is deleted afterward, so the real repository is
never touched. The leading `[N.NNN]` on the first line is a relative startup timestamp that
varies slightly per run; the signal is the exit code and the compile error, both stable.

```text
$ WORK="$(mktemp -d /tmp/kitty_shaderproof.XXXXXX)"
$ cp -a . "$WORK"/ ; cd "$WORK"
# CONTROL: unmodified cell_vertex.glsl (child's stdout goes to the PTY, not kitty's stdout)
$ LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe xvfb-run -a ./kitty/launcher/kitty --config NONE sh -c 'printf CONTROL_CHILD_OK' 2>&1 ; echo "control_exit=$?"
[0.192] Failed to open systemd user bus with error: Connection refused
control_exit=0
# BROKEN: one invalid token appended to the cell vertex shader
$ printf '\nthis_is_not_valid_glsl\n' >> kitty/cell_vertex.glsl
$ LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe xvfb-run -a ./kitty/launcher/kitty --config NONE sh -c 'printf SHOULD_NOT_REACH' 2>&1 ; echo "broken_exit=$?"
Failed to compile GLSL vertex shader:
cell_vertex.glsl:235(1): error: syntax error, unexpected NEW_IDENTIFIER, expecting end of file
broken_exit=1
$ cd /tmp ; rm -rf "$WORK"   # disposable copy removed; real repo untouched
```

The unmodified control starts (`control_exit=0`); a single bad token in `cell_vertex.glsl`
aborts startup (`broken_exit=1`) with the driver's own compile error. The cell shader is on the
critical path to a running terminal.

### How the build treats the `.glsl` files (bundled as resources, not embedded)

The `.glsl` files are **not compiled into the executable**. The build uses them two ways:

- **Build-time code generation:** `setup.py:L1040`-`L1049` reads each `kitty/*.glsl` only to
  scan for `uniform` declarations and generate C uniform-accessor stubs
  (`get_uniform_locations_<name>`); it does not embed the shader bodies.
- **Packaging as resources:** when the source tree is copied into the install dir,
  `setup.py:L1712` sets `allowed_extensions = frozenset('py glsl so'.split())`, so the raw
  `.glsl` files are bundled alongside the `.py` and `.so` files as package data.

At runtime they are read back as package resources by `read_kitty_resource`
(`kitty/constants.py:L241`) and only then compiled by the GPU driver (above). So the files ship
as data resources and are compiled on the GPU at startup.

### Cause -> effect

- **Cause:** kitty renders the entire terminal with OpenGL; the drawing logic for cells,
  borders, images, and overlays lives in these 13 `.glsl` programs, loaded by
  `kitty/shaders.py` and compiled by `kitty/shaders.c`/`kitty/gl.c`.
- **Effect:** the shaders are central infrastructure — the concrete realization of the
  "GPU accelerated" claim. Break the cell shader and the terminal will not start; there is no
  CPU text-drawing fallback in this path.

---

## Q3 — Running the entry point fails immediately; what is missing, and how is Python wired to the native core?

> *running the main entry point directly fails almost immediately with a cryptic error — what exactly is missing at that moment, and what does that tell us about how Python is wired into the native core (there is "one critical piece that everything depends on")?*

### Answer

Running the real entry point (`python3 __main__.py`) on an **unbuilt** checkout fails at once
with `ModuleNotFoundError: No module named 'kitty.fast_data_types'`. The one missing piece is
the **compiled C extension `kitty/fast_data_types.so`** — the `.so` is gitignored and is
produced only by the build; nearly the entire front-end imports it. That is the "one critical
piece that everything depends on."

There is a second, subtler layer, which the earlier draft got wrong and is corrected here.
**Building the extension removes the `ModuleNotFoundError`, but a bare `python3 __main__.py`
still does not launch** — it then fails with
`AttributeError: module 'sys' has no attribute 'kitty_run_data'`. That attribute is injected by
the **native launcher** (`kitty/launcher/main.c`) after it embeds CPython; a plain `python3`
never runs the launcher, so
the attribute is absent. The real product is started by that launcher (or, for a quick check,
`--version`, which exits before the launcher-only code path). So Python is wired to the native
core at two points: (1) a hard import dependency on the compiled `fast_data_types`, and (2) a
runtime handshake (`sys.kitty_run_data`) that only the C launcher performs.

### Step 1 — reproduce the failure on the real entry point (unbuilt tree)

The unbuilt tree is created from the repository's tracked files with `git archive` (the `.so`
is gitignored, so an archive of tracked files is unbuilt **by construction**). It is a
disposable `/tmp` tree, removed afterward:

```text
$ git archive HEAD | tar -t | grep -c fast_data_types.so
0
$ mkdir -p /tmp/kitty_unbuilt && git archive HEAD | tar -x -C /tmp/kitty_unbuilt
$ ls /tmp/kitty_unbuilt/kitty/fast_data_types.so ; echo "ls_exit=$?"
ls: cannot access '/tmp/kitty_unbuilt/kitty/fast_data_types.so': No such file or directory
ls_exit=2
```

Running the real entry point there fails immediately and identically across two runs:

```text
$ cd /tmp/kitty_unbuilt && python3 __main__.py ; echo "exit=$?"
Traceback (most recent call last):
  File "/tmp/kitty_unbuilt/__main__.py", line 7, in <module>
    main()
    ~~~~^^
  File "/tmp/kitty_unbuilt/kitty/entry_points.py", line 194, in main
    from kitty.main import main as kitty_main
  File "/tmp/kitty_unbuilt/kitty/main.py", line 11, in <module>
    from .borders import load_borders_program
  File "/tmp/kitty_unbuilt/kitty/borders.py", line 7, in <module>
    from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image
ModuleNotFoundError: No module named 'kitty.fast_data_types'
exit=1

$ cd /tmp/kitty_unbuilt && python3 __main__.py ; echo "exit=$?"
Traceback (most recent call last):
  File "/tmp/kitty_unbuilt/__main__.py", line 7, in <module>
    main()
    ~~~~^^
  File "/tmp/kitty_unbuilt/kitty/entry_points.py", line 194, in main
    from kitty.main import main as kitty_main
  File "/tmp/kitty_unbuilt/kitty/main.py", line 11, in <module>
    from .borders import load_borders_program
  File "/tmp/kitty_unbuilt/kitty/borders.py", line 7, in <module>
    from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image
ModuleNotFoundError: No module named 'kitty.fast_data_types'
exit=1
```

The failure chain is exact and grounded: `__main__.py:L7` calls `main()`, which at
`kitty/entry_points.py:L194` does `from kitty.main import main as kitty_main`; `kitty/main.py:L11`
runs `from .borders import load_borders_program`; and `kitty/borders.py:L7` is the first line to
execute `from .fast_data_types import ...`, which raises. So `kitty/borders.py` is the first
module whose import reaches the native bridge.

### The alternate invocation fails differently

`python3 -m kitty` fails for a different reason — there is no `kitty/__main__.py`, only the
repository-root `__main__.py`:

```text
$ cd /tmp/kitty_unbuilt && python3 -m kitty ; echo "exit=$?"
/usr/bin/python3: No module named kitty.__main__; 'kitty' is a package and cannot be directly executed
exit=1
$ ls /tmp/kitty_unbuilt/kitty/__main__.py ; echo "ls_exit=$?"
ls: cannot access '/tmp/kitty_unbuilt/kitty/__main__.py': No such file or directory
ls_exit=2
```

### What exactly is missing, and how many modules depend on it

The missing artifact is the compiled CPython extension `kitty/fast_data_types.so`. It is not
tracked (it is gitignored) and is emitted only by the build — `setup.py:L883`-`L884` name the
output `f'{module}.so'` and `setup.py:L1091` sets the module to `kitty/fast_data_types`, so the
artifact is `kitty/fast_data_types.so` on **all** platforms (CPython uses `.so` for extension
modules on Linux and macOS alike; there is no `.dylib` produced for this module):

```text
$ sed -n '883,884p' setup.py
    dest = os.path.join(build_dir, f'{module}.so')
    real_dest = f'{module}.so'
$ git check-ignore kitty/fast_data_types.so ; echo "ignored_exit=$?"
kitty/fast_data_types.so
ignored_exit=0
```

The dependency is pervasive but should be stated precisely (three different measurements, not
one):

```text
$ grep -rlE 'fast_data_types' --include='*.py' . | wc -l          # files that MENTION the string
80
$ # files that actually IMPORT it, via AST (ImportFrom/Import), and the production subset:
$ python3 - <<'PY'
import ast, os
importers=set()
for root,_,files in os.walk('.'):
    if '/.git' in root: continue
    for fn in files:
        if not fn.endswith('.py'): continue
        p=os.path.join(root,fn)
        try: tree=ast.parse(open(p,encoding='utf-8').read())
        except Exception: continue
        for n in ast.walk(tree):
            if isinstance(n,ast.ImportFrom):
                m=n.module or ''
                if m=='fast_data_types' or m.endswith('.fast_data_types'):
                    importers.add(p); break
            elif isinstance(n,ast.Import):
                if any(a.name.endswith('fast_data_types') for a in n.names):
                    importers.add(p); break
prod=[p for p in importers if p.startswith('./kitty/') or p.startswith('./kittens/')]
print('AST importers total      :', len(importers))
print('production kitty//kittens:', len(prod))
print('non-production           :', len(importers)-len(prod))
PY
AST importers total      : 77
production kitty//kittens: 59
non-production           : 18
```

So **80** Python files *mention* the string, **77** actually *import* the module, and **59** of
those are production code under `kitty/` or `kittens/` (the remaining 18 are tests,
`docs/conf.py`, and a code generator). The three mention-only-but-not-import files are
`setup.py` (build path string), `kitty/options/definition.py` (a type name in a string), and
`gen/key_constants.py` (a patch target path).

### Step 2 — build the extension, then run the SAME entry point again (the corrected built-state result)

The tree is built once with the canonical-from-source flow (here in a native, NON-CANONICAL
environment; the leading `CFLAGS=-Wno-error=switch` only demotes the single `-Werror=switch`
diagnostic that a newer `wayland-protocols` triggers — see the environment section):

```text
$ mkdir -p /tmp/kitty_built && git archive HEAD | tar -x -C /tmp/kitty_built
$ cd /tmp/kitty_built && CFLAGS=-Wno-error=switch python3 setup.py > /tmp/kitty_build.log 2>&1 ; echo "build_exit=$?"
build_exit=0
$ file /tmp/kitty_built/kitty/fast_data_types.so
/tmp/kitty_built/kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=de56de664dde21b23a324685eb08bf0032548704, not stripped
```

Now the compiled `.so` exists, so the `ModuleNotFoundError` is gone. But running the **same bare
entry point** still does not launch — it fails on the launcher-only `sys.kitty_run_data`,
identically across two runs (the leading `[N.NNN]` is kitty's relative startup timestamp and
varies slightly per run; the traceback body and exit code are stable):

```text
$ cd /tmp/kitty_built && python3 __main__.py ; echo "exit=$?"
[0.052] Traceback (most recent call last):
  File "/tmp/kitty_built/kitty/main.py", line 526, in main
    _main()
    ~~~~~^^
  File "/tmp/kitty_built/kitty/main.py", line 495, in _main
    setup_environment(opts, cli_opts)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^
  File "/tmp/kitty_built/kitty/main.py", line 411, in setup_environment
    ensure_kitty_in_path()
    ~~~~~~~~~~~~~~~~~~~~^^
  File "/tmp/kitty_built/kitty/main.py", line 364, in ensure_kitty_in_path
    krd = getattr(sys, 'kitty_run_data')
AttributeError: module 'sys' has no attribute 'kitty_run_data'
exit=1

$ cd /tmp/kitty_built && python3 __main__.py ; echo "exit=$?"
[0.052] Traceback (most recent call last):
  File "/tmp/kitty_built/kitty/main.py", line 526, in main
    _main()
    ~~~~~^^
  File "/tmp/kitty_built/kitty/main.py", line 495, in _main
    setup_environment(opts, cli_opts)
    ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^
  File "/tmp/kitty_built/kitty/main.py", line 411, in setup_environment
    ensure_kitty_in_path()
    ~~~~~~~~~~~~~~~~~~~~^^
  File "/tmp/kitty_built/kitty/main.py", line 364, in ensure_kitty_in_path
    krd = getattr(sys, 'kitty_run_data')
AttributeError: module 'sys' has no attribute 'kitty_run_data'
exit=1
```

Two invocations that DO succeed on the built tree confirm the diagnosis. `--version` exits
before the launcher-only path, and the native launcher populates `sys.kitty_run_data` itself:

```text
$ cd /tmp/kitty_built && python3 __main__.py --version ; echo "exit=$?"
kitty 0.35.2 created by Kovid Goyal
exit=0

$ cd /tmp/kitty_built && LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe xvfb-run -a ./kitty/launcher/kitty --config NONE --debug-rendering sh -c 'printf CHILD_OK' 2>&1 ; echo "exit=$?"
[0.158] OS Window created
[0.167] Failed to open systemd user bus with error: Connection refused
[0.170] Child launched
[0.132] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
exit=0
```

### How Python is wired into the native core

The consumer of the attribute is `kitty/main.py:L364` inside `ensure_kitty_in_path()`:

```text
$ sed -n '362,366p' kitty/main.py
def ensure_kitty_in_path() -> None:
    # Ensure the correct kitty is in PATH
    krd = getattr(sys, 'kitty_run_data')
    rpath = krd.get('bundle_exe_dir')
    if not rpath:
```

Nothing in Python sets `sys.kitty_run_data`. The native launcher does. `kitty/launcher/main.c`
embeds CPython by calling `Py_InitializeFromConfig` (`L211`) and then immediately calls
`set_kitty_run_data` (`L214`), which builds a dict and installs it with
`PySys_SetObject("kitty_run_data", ans)` (`L73`):

```text
$ sed -n '211p;214p' kitty/launcher/main.c
    status = Py_InitializeFromConfig(&config);
    if (!set_kitty_run_data(run_data, from_source, NULL)) return 1;
$ sed -n '52,53p;73p' kitty/launcher/main.c
static bool
set_kitty_run_data(RunData *run_data, bool from_source, wchar_t *extensions_dir) {
    int ret = PySys_SetObject("kitty_run_data", ans);
```

So the shipped `kitty` executable is a C program that (1) starts an embedded CPython, (2) sets
`sys.kitty_run_data` describing where the binary lives, and only then (3) hands control to the
Python front-end, which imports the compiled `fast_data_types`. A bare `python3 __main__.py`
performs neither the launcher handshake nor guarantees the extension is present.

### Cause -> effect

- **Cause (layer 1):** the front-end's first native-dependent import
  (`kitty/borders.py:L7` -> `from .fast_data_types import ...`) requires the compiled
  `kitty/fast_data_types.so`, which is gitignored and produced only by the build.
- **Effect (layer 1):** on an unbuilt tree the real entry point dies immediately with
  `ModuleNotFoundError: No module named 'kitty.fast_data_types'`. This is the single critical
  missing piece; 59 production modules import it.
- **Cause (layer 2):** the front-end also expects `sys.kitty_run_data`, which is set only by
  the C launcher (`kitty/launcher/main.c:L214` -> `L73`) after it embeds CPython.
- **Effect (layer 2):** even after building, a bare `python3 __main__.py` fails with
  `AttributeError: module 'sys' has no attribute 'kitty_run_data'`; only the native launcher
  (or the `--version` fast path) runs. Python is a thin front-end tightly coupled to, and
  launched by, the native core.

---

## Q4 — Are the kittens independent tools, or do they rely on the same native bridge?

> *the `kittens/` directory looks like small self-contained tools — are they truly independent or do they quietly rely on the same native bridge, and what happens if you run one standalone?*

### Answer

The kittens are **not** independent of the native bridge. Running a Python kitten standalone
fails: as a module (`python3 -m kittens.hints.main`) it dies with the same
`ModuleNotFoundError: No module named 'kitty.fast_data_types'` seen in Q3, because the kitten
imports `kitty` modules that (transitively and, for `hints`, directly) require the compiled
extension; run as a bare script (`python3 kittens/unicode_input/main.py`) it fails even earlier
with `No module named 'kitty'` because the package is not on `sys.path`. A crucial nuance: most
kittens ship **both** a Python `main.py` and a Go `main.go`, and for the 12 "wrapped" kittens
the Python `main()` is only a stub that redirects you to the compiled Go `kitten` binary
(`raise SystemExit('Should be run as kitten hints')`). The canonical modern invocation is that
Go binary, reached through `kitty +kitten <name>`.

### Standalone run 1 — as a module (`python3 -m kittens.hints.main`, unbuilt tree)

```text
$ cd /tmp/kitty_unbuilt && python3 -m kittens.hints.main ; echo "exit=$?"
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/kitty_unbuilt/kittens/hints/main.py", line 9, in <module>
    from kitty.clipboard import set_clipboard_string, set_primary_selection
  File "/tmp/kitty_unbuilt/kitty/clipboard.py", line 11, in <module>
    from .conf.utils import uniq
  File "/tmp/kitty_unbuilt/kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
exit=1
```

The chain is grounded: `kittens/hints/main.py:L9` runs
`from kitty.clipboard import set_clipboard_string, set_primary_selection`; `kitty/clipboard.py:L11`
does `from .conf.utils import uniq`; and `kitty/conf/utils.py:L27` executes
`from ..fast_data_types import Color`, which raises. (`hints/main.py:L11` also imports
`fast_data_types` directly, `from kitty.fast_data_types import get_options` — but the transitive
`clipboard` import on L9 reaches the bridge first.)

### Standalone run 2 — as a bare script (`python3 kittens/unicode_input/main.py`, unbuilt tree)

```text
$ cd /tmp/kitty_unbuilt && python3 kittens/unicode_input/main.py ; echo "exit=$?"
Traceback (most recent call last):
  File "/tmp/kitty_unbuilt/kittens/unicode_input/main.py", line 6, in <module>
    from kitty.typing import BossType
ModuleNotFoundError: No module named 'kitty'
exit=1
```

Here the very first import fails differently: `kittens/unicode_input/main.py:L6` is
`from kitty.typing import BossType`, and because the script is run by path (not as a package),
`kitty` is not importable at all, so it fails with `No module named 'kitty'` before it ever
reaches `fast_data_types`.

### Even after building, the Python entry for a wrapped kitten is only a stub

Building the extension removes the `ModuleNotFoundError`, but running the Python kitten module
still does not do the work — it redirects to the Go binary:

```text
$ cd /tmp/kitty_built && python3 -m kittens.hints.main ; echo "exit=$?"
Should be run as kitten hints
exit=1
```

That message comes from `kittens/hints/main.py:L258`-`L259`:

```text
$ sed -n '258,259p' kittens/hints/main.py
def main(args: List[str]) -> Optional[Dict[str, Any]]:
    raise SystemExit('Should be run as kitten hints')
```

### The canonical path: the compiled Go `kitten` binary

The modern, canonical way to run a kitten is the Go binary, either directly (`kitten <name>`)
or through `kitty +kitten <name>`. It reports its own version and runs the real tool:

```text
$ ./kitty/launcher/kitten --version ; echo "exit=$?"
kitten 0.35.2 created by Kovid Goyal
exit=0
```

`kitty +kitten hints --help` and `kitten hints --help` produce **byte-identical** output. The
complete `kitten hints --help` (148 lines, 6400 bytes) is:

```text
$ ./kitty/launcher/kitten hints --help
Usage: kitten hints 

Select text from the screen using the keyboard. Defaults to searching for URLs.

Options:
  --program
    What program to use to open matched text. Defaults to the default open
    program for the operating system. Various special values are supported:

    -
    paste the match into the terminal window.

    @
    copy the match to the clipboard

    *
    copy the match to the primary selection (on systems that support primary
    selections)

    @NAME
    copy the match to the specified buffer, e.g. @a

    default
    run the default open program. Note that when using the hyperlink --type the
    default is to use the kitty hyperlink handling facilities.

    launch
    run The launch command to open the program in a new kitty tab, window,
    overlay, etc. For example::


    --program "launch --type=tab vim"

    Can be specified multiple times to run multiple programs.

  --type [=url]
    The type of text to search for. A value of linenum is special, it looks for
    error messages using the pattern specified with --regex, which must have the
    named groups: path and line. If not specified, will look for path:line. The
    --linenum-action option controls where to display the selected error
    message, other options are ignored.
    Choices: url, hash, hyperlink, ip, line, linenum, path, regex, word

  --regex [=(?m)^\s*(.+)\s*$]
    The regular expression to use when option --type is set to regex, in Perl 5
    syntax. If you specify a numbered group in the regular expression, only the
    group will be matched. This allow you to match text ignoring a
    prefix/suffix, as needed. The default expression matches lines. To match
    text over multiple lines, things get a little tricky, as line endings are a
    sequence of zero or more null bytes followed by either a carriage return or
    a newline character. To have a pattern match over line endings you will need
    to match the character set ``[\0\r\n]``. The newlines and null bytes are
    automatically stripped from the returned text. If you specify named groups
    and a --program, then the program will be passed arguments corresponding to
    each named group of the form key=value.

  --linenum-action [=self]
    Where to perform the action on matched errors. self means the current
    window, window a new kitty window, tab a new tab, os_window a new OS window
    and background run in the background. The actual action is whatever
    arguments are provided to the kitten, for example: kitten hints
    --type=linenum --linenum-action=tab vim +{line} {path} will open the matched
    path at the matched line number in vim in a new kitty tab. Note that in
    order to use --program to copy or paste the provided arguments, you need to
    use the special value self.
    Choices: self, background, os_window, tab, window

  --url-prefixes [=default]
    Comma separated list of recognized URL prefixes. Defaults to the list of
    prefixes defined by the url_prefixes option in kitty.conf.

  --url-excluded-characters [=default]
    Characters to exclude when matching URLs. Defaults to the list of characters
    defined by the url_excluded_characters option in kitty.conf. The syntax for
    this option is the same as for url_excluded_characters.

  --word-characters
    Characters to consider as part of a word. In addition, all characters marked
    as alphanumeric in the Unicode database will be considered as word
    characters. Defaults to the select_by_word_characters option from
    kitty.conf.

  --minimum-match-length [=3]
    The minimum number of characters to consider a match.

  --multiple
    Select multiple matches and perform the action on all of them together at
    the end. In this mode, press Esc to finish selecting.

  --multiple-joiner [=auto]
    String for joining multiple selections when copying to the clipboard or
    inserting into the terminal. The special values are: space - a space
    character, newline - a newline, empty - an empty joiner, json - a JSON
    serialized list, auto - an automatic choice, based on the type of text being
    selected. In addition, integers are interpreted as zero-based indices into
    the list of selections. You can use 0 for the first selection and -1 for the
    last.

  --add-trailing-space [=auto]
    Add trailing space after matched text. Defaults to auto, which adds the
    space when used together with --multiple.
    Choices: auto, always, never

  --hints-offset [=1]
    The offset (from zero) at which to start hint numbering. Note that only
    numbers greater than or equal to zero are respected.

  --alphabet
    The list of characters to use for hints. The default is to use numbers and
    lowercase English alphabets. Specify your preference as a string of
    characters. Note that you need to specify the --hints-offset as zero to use
    the first character to highlight the first match, otherwise it will start
    with the second character by default.

  --ascending
    Make the hints increase from top to bottom, instead of decreasing from top
    to bottom.

  --hints-foreground-color [=black]
    The foreground color for hints. You can use color names or hex values. For
    the eight basic named terminal colors you can also use the bright- prefix to
    get the bright variant of the color.

  --hints-background-color [=green]
    The background color for hints. You can use color names or hex values. For
    the eight basic named terminal colors you can also use the bright- prefix to
    get the bright variant of the color.

  --hints-text-color [=bright-gray]
    The foreground color for text pointed to by the hints. You can use color
    names or hex values. For the eight basic named terminal colors you can also
    use the bright- prefix to get the bright variant of the color.

  --customize-processing
    Name of a python file in the kitty config directory which will be imported
    to provide custom implementations for pattern finding and performing actions
    on selected matches. You can also specify absolute paths to load the script
    from elsewhere. See https://sw.kovidgoyal.net/kitty/kittens/hints/ for
    details.

  --window-title
    The title for the hints window, default title is based on the type of text
    being hinted.

  --help, -h
    Show help for this command

kitten hints 0.35.2 created by Kovid Goyal
```

and the two invocations are identical (the diff is empty, exit 0):

```text
$ LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a ./kitty/launcher/kitty +kitten hints --help > /tmp/a.txt ; echo "plus_kitten_exit=$?"
plus_kitten_exit=0
$ ./kitty/launcher/kitten hints --help > /tmp/b.txt ; echo "kitten_exit=$?"
kitten_exit=0
$ wc -l /tmp/a.txt /tmp/b.txt
  148 /tmp/a.txt
  148 /tmp/b.txt
  296 total
$ diff /tmp/a.txt /tmp/b.txt ; echo "diff_exit=$?"
diff_exit=0
```

For contrast, the Python shim reached through the interpreter refuses to do the work, which is
how you can tell the two implementations apart:

```text
$ cd /tmp/kitty_built && python3 __main__.py +kitten hints --help ; echo "exit=$?"
Should be run as kitten hints
exit=1
```

### The Go binary is DYNAMICALLY linked (not static), built with CGO

The earlier draft called the binary "statically linked"; the observed linkage is **dynamic**,
and it is built with `CGO_ENABLED=1`. (`setup.py` names the build function
`build_static_kittens`, but that name is aspirational: it forces `CGO_ENABLED=0` only when
cross-compiling a standalone artifact; the launcher-adjacent `kitten` built natively uses the
default `CGO_ENABLED=1` and is dynamically linked.)

```text
$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=edf4dddb2419eacf2f2ae1a6e26789d39d2a3555, stripped
$ ldd kitty/launcher/kitten
	linux-vdso.so.1 (0x00007ffe3309f000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x0000798c9b4ca000)
	/lib64/ld-linux-x86-64.so.2 (0x0000798c9b716000)
$ go version -m kitty/launcher/kitten | grep -E 'CGO_ENABLED|GOOS|GOARCH|-ldflags|vcs\.'
	build	-ldflags="-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w"
	build	CGO_ENABLED=1
	build	GOARCH=amd64
	build	GOOS=linux
	build	vcs.revision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
	build	vcs.time=2024-06-24T02:24:17Z
	build	vcs.modified=false
```

`file` reports `dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2`; `ldd` resolves
real shared objects (`libc.so.6`, the loader); and the recorded build setting is
`CGO_ENABLED=1`. The VCS revision stamped into the binary is
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (the upstream kitty commit, matching the branch and
canonical build). Two fields above are not byte-reproducible and vary per build or per run: the
`BuildID[sha1]` changes whenever the binary is rebuilt, and the `ldd` load addresses
(`0x...`) are randomized by ASLR on every execution. The stable, load-bearing facts are the
linkage type (`dynamically linked`), the interpreter path, the resolved shared-object names
(`linux-vdso.so.1`, `libc.so.6`, `ld-linux-x86-64.so.2`), and the recorded `CGO_ENABLED=1` — a
dynamically linked, CGO-enabled binary, not a static one.

### Dispatch is route-specific (three distinct mechanisms)

"Kitten dispatch" is not one Go call — there are three routes, and only the 12 **wrapped**
kittens use the compiled binary:

- **Route A - native C, before CPython starts.** The launcher intercepts wrapped kittens and
  `execv`s the Go binary directly. `kitty/launcher/main.c:L354` `delegate_to_kitten_if_possible`
  (called at `L452`) checks `is_wrapped_kitten` (`L333`) and, for `@...`, `+kitten <name>`, or
  `+ kitten <name>` (`L355`-`L357`), calls `exec_kitten` (`L340`) which builds
  `"<exe_dir>/kitten"` and `execv`s it. The wrapped set is the C macro `WRAPPED_KITTENS`,
  exposed to Python at `kitty/data-types.c:L252`-`L253` (`wrapped_kitten_names`).
- **Route B - Python, from the running boss via `kitten_exe()`.** When kitty itself launches a
  kitten, `kitty/boss.py:L1902` computes `is_wrapped = kitten in wrapped_kitten_names()` and, if
  wrapped, `kitty/boss.py:L1948`-`L1949` sets `cmd = [kitten_exe(), kitten]`. `kitten_exe()` is
  `kitty/constants.py:L83`-`L84`, `os.path.join(os.path.dirname(kitty_exe()), 'kitten')` - i.e.
  the same Go binary.
- **Route C - pure-Python runner, the fallback for non-wrapped kittens.**
  `kittens/runner.py:L110` `run_kitten` ultimately calls `kittens/runner.py:L116`
  `runpy.run_module(f'kittens.{kitten}.main', ...)` (and `L61` `importlib.import_module(...)`),
  running the Python `main.py` in-process. This is the path for kittens that have no Go
  implementation.

The 12 wrapped kittens (from the built binary's own `wrapped_kitten_names()`):

```text
$ python3 -c "from kitty.fast_data_types import wrapped_kitten_names; w=wrapped_kitten_names(); print('count=',len(w)); print(' '.join(sorted(w)))"
count= 12
ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer unicode_input
```

### Dual Python/Go implementation

Of the 18 kitten directories that contain a `main.py` or a `main.go`, 14 contain **both** - the
Python and Go implementations coexisting. Only 4 are Python-only; none are Go-only:

```text
$ python3 - <<'PY'
import os
base='kittens'
has_py=set(); has_go=set()
for d in sorted(os.listdir(base)):
    p=os.path.join(base,d)
    if not os.path.isdir(p): continue
    if os.path.exists(os.path.join(p,'main.py')): has_py.add(d)
    if os.path.exists(os.path.join(p,'main.go')): has_go.add(d)
either=has_py|has_go; both=has_py&has_go
print('dirs with main.py or main.go :', len(either))
print('dirs with BOTH               :', len(both))
print('py-only                      :', len(has_py-has_go), sorted(has_py-has_go))
print('go-only                      :', len(has_go-has_py), sorted(has_go-has_py))
PY
dirs with main.py or main.go : 18
dirs with BOTH               : 14
py-only                      : 4 ['broadcast', 'panel', 'remote_file', 'resize_window']
go-only                      : 0 []
```

### Cause -> effect

- **Cause:** every Python kitten imports `kitty` modules; those modules (for example
  `kitty/clipboard.py` -> `kitty/conf/utils.py:L27`) import the compiled `fast_data_types`, and
  `hints` imports it directly (`kittens/hints/main.py:L11`).
- **Effect:** a Python kitten cannot run standalone on an unbuilt tree - it fails with
  `ModuleNotFoundError: No module named 'kitty.fast_data_types'` (module form) or
  `No module named 'kitty'` (script form). The kittens are not independent of the native bridge.
- **Cause:** for the 12 wrapped kittens the real implementation is the Go `kitten` binary; the
  Python `main()` is a stub (`raise SystemExit('Should be run as kitten hints')`), and the C
  launcher/boss route wrapped invocations to that binary.
- **Effect:** the canonical path is `kitty +kitten <name>` (or `kitten <name>`), which produces
  output byte-identical to the Go binary; the Go binary is dynamically linked and built with
  `CGO_ENABLED=1`, a separate artifact from `fast_data_types.so` but part of the same native
  story.

---

## Coverage and honesty pass

This section re-reads each question and confirms every named item is addressed, then records the
honest limitations of the environment in which the answers were produced.

### Every named item, per question

- **Q1 (language / "GPU accelerated").** Which language does the heavy lifting: the compiled C
  extension `kitty.fast_data_types`, with the GPU computing pixels; the "GPU accelerated" claim
  is reconciled with the Python/C weave (Python orchestrates, C runs the hot paths, GLSL shaders
  compute pixels). Module identity `fast_data_types` (`kitty/data-types.c:L467`-`L469`,
  `PyInit_fast_data_types` at `:L525`); aggregation of the C subsystems (`init_*` calls;
  `setup.find_c_files()` = 62 extension sources vs 128 repo-wide `.c`); representative hot paths
  (`screen.c`, `vt-parser.c`, `graphics.c`, `fonts.c`, `child-monitor.c`, `glfw.c`, `shaders.c`);
  `fontconfig` loaded at runtime by `dlopen`; SIMD (`simd-string-128.c` / `-256.c`); the `.pyi`
  type stub; the observed 67 threads and the GL stage.
- **Q2 (GLSL role and centrality).** All 13 `.glsl` files enumerated (`cell_*`, `border_*`,
  `graphics_*`, `bgimage_*`, `tint_*`, `alpha_blend`, `linear2srgb`) and their roles (cells /
  cursor / selection, borders, protocol images, background image, tint, shared includes);
  Python-side load in `kitty/shaders.py:L54`-`L55` and `program_for('cell')`; C-side compile in
  `kitty/shaders.c:L1160` through `kitty/gl.c`; `GLSL_VERSION` at `kitty/shaders.c:L1254`; the
  load-bearing proof (a synthetic broken-shader run); how the build bundles `.glsl` as resources.
- **Q3 (entry-point failure and the native bridge).** The exact error on the real entry point
  (`ModuleNotFoundError: No module named 'kitty.fast_data_types'`) with the full chain
  `__main__.py:L7` -> `entry_points.py:L194` -> `main.py:L11` -> `borders.py:L7`; what is missing
  (the compiled `fast_data_types.so`); the "one critical piece that everything depends on" (80
  grep mentions / 77 AST imports / 59 production modules); the alternate `python3 -m kitty` error;
  the corrected built-state result (the bare entry point then fails on `sys.kitty_run_data`); how
  Python is wired to the native core (`kitty/launcher/main.c` embeds CPython and injects
  `sys.kitty_run_data`).
- **Q4 (kittens and the native bridge).** Whether the kittens are independent (they are not); what
  happens running one standalone (module form -> `No module named 'kitty.fast_data_types'`, script
  form -> `No module named 'kitty'`); the shared native bridge; the Go `kitten` binary
  (dynamically linked, `CGO_ENABLED=1`); `kitten_exe()` at `kitty/constants.py:L83`-`L84`; the
  canonical `kitty +kitten <name>` (byte-identical to `kitten <name>`); the dual Python/Go layout
  (14 of 18 kitten directories have both); the three dispatch routes and the 12 `WRAPPED_KITTENS`.

### Environment honesty (what is canonical and what is not)

- **The canonical image is unavailable here.** The task nominates the Docker image
  `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`;
  pulling it fails with a `pull access denied` error (the complete daemon message is quoted in the
  Methodology section above). Every build-dependent value in this document was therefore produced
  by a **native, non-canonical** from-source build and is labelled as such where it appears.
- **Version banner (non-canonical).** `kitty 0.35.2 created by Kovid Goyal`, read from the
  default-configuration build.
- **The built bare entry point does not launch.** After building, `python3 __main__.py` no longer
  raises `ModuleNotFoundError`, but it then fails on
  `AttributeError: module 'sys' has no attribute 'kitty_run_data'`; only the native launcher (or
  `--version`, which exits before that code path) starts the product. The earlier draft's claim
  that the bare entry point launches after building was wrong and is corrected in Q3.
- **The Go `kitten` binary is dynamically linked, not static.** `file` and `ldd` show a dynamic
  ELF with interpreter `/lib64/ld-linux-x86-64.so.2`, and the recorded build flag is
  `CGO_ENABLED=1`. The function name `build_static_kittens` in `setup.py` is aspirational.
- **GPU acceleration is inferred; software GL is what was observed.** The measured GL version
  string is `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2`, served by the Mesa `llvmpipe` CPU
  software rasterizer (forced via `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`); the claim
  that the same code path drives a hardware GPU on a normal desktop is labelled **(inferred)**.

### Repository integrity

No tracked source file is modified; the only tracked addition is this answer document. The
compiled artifacts used for observation (`kitty/fast_data_types.so` and the `kitty` / `kitten`
launchers) are **git-ignored** build outputs produced by `setup.py`, not committed files:

```text
$ git check-ignore kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
kitty/fast_data_types.so
kitty/launcher/kitty
kitty/launcher/kitten
```

All temporary observation scripts and trees created during the investigation are removed on
completion, so the source repository is left byte-for-byte unchanged apart from this document.
