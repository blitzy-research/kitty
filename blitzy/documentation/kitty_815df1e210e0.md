# kitty: Where the Work Actually Happens — A Run-First Architecture Investigation

> **The tension.** kitty "calls itself GPU accelerated yet the codebase feels like a tight weave of Python and C." This document resolves that tension **empirically**: instead of reasoning about how the architecture is *meant* to work, every claim below was produced by **running the real code paths and capturing the actual output**, then citing the exact `file:line` that explains it. Four questions are answered — which language does the heavy lifting, what the GLSL files do, why the main entry point fails, and whether kittens are truly independent — followed by a synthesis that ties them together.

## Methodology and environment

- **Run-first discipline.** Each answer leads with a direct conclusion, then shows the **exact command** and its **complete, unedited output**, then the **`file:line` evidence**, then the cause-and-effect rationale. Anything not directly observed is explicitly prefixed **"Inferred:"**.
- **Canonical paths only.** Failures are reproduced through the real entry points — the repository-root `__main__.py` and a standalone kitten via `kittens.runner` — never through a debug hook, mock, or synthetic stand-in.
- **Read-only.** This Markdown file is the only artifact created anywhere. No source file was modified; the working tree stayed byte-for-byte clean (verified in the Appendix).

**Observed environment.** The branch, repository root, interpreters, and build artifacts were captured directly:

```console
$ git rev-parse --show-toplevel
/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770

$ git branch --show-current
blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd

$ git status --porcelain
[empty output above = clean]

$ /opt/kitty311-venv/bin/python3 --version
Python 3.11.15

$ /usr/bin/python3 --version
Python 3.13.7

$ ls -la kitty/fast_data_types*
-rw-r--r-- 1 root root   35795 Jul 14 18:52 kitty/fast_data_types.pyi
-rwxr-xr-x 1 root root 1248952 Jul 14 19:10 kitty/fast_data_types.so

$ ls kitty/__main__.py 2>&1
ls: cannot access 'kitty/__main__.py': No such file or directory
```

Two facts shape everything that follows:

1. **The compiled native extension `kitty/fast_data_types.so` is present** in this environment (1,248,952 bytes), because the project was built canonically by the setup step (`CFLAGS="-Wno-error=switch" /opt/kitty311-venv/bin/python setup.py --verbose`). Alongside it sits only a **type stub** `kitty/fast_data_types.pyi` (35,795 bytes) — a `.pyi` file is **not importable Python code**, it only describes types for static analysis.
2. **The canonical interpreter is the virtualenv `python3` at `/opt/kitty311-venv/bin/python3` (CPython 3.11.15)** — this is the interpreter that built the `.so`, and 3.11 is the highest version kitty documents (see Synthesis). The system `python3` is 3.13.7, **above** kitty's supported ceiling; it is used only to illustrate a version-boundary effect in Q3.

Because the failure signals in Q3 and Q4 are triggered by the **absence** of `fast_data_types.so`, this document observes **both states**:

- the **post-build** state (`.so` present) — kitty actually runs and creates a GPU context; and
- the **pre-build** state — reproduced non-destructively by temporarily moving the *gitignored* `.so` aside (with a guaranteed restore, verified by md5), running the **real** canonical entry points to capture the genuine `ModuleNotFoundError`, then restoring it.

This before/after pairing is what proves the `.so` is the load-bearing dependency: the same command fails without it and progresses with it.

---

## Q1 — Which language does the heavy lifting, and where does performance come from?

**Direct answer.** At runtime the **compiled C core does the heavy lifting**, and performance originates in **C plus GPU offload**. Python is a comparatively thin orchestration/configuration/extensibility shell, and Go is a **separate, self-contained** command-line binary that is not part of the in-process hot path. The per-byte and per-frame work — reading the PTY, parsing VT escape sequences, mutating the screen grid, rasterizing glyphs into a texture atlas, and issuing OpenGL draw calls — all lives in C and on the GPU; Python only dispatches high-level events.

### The language footprint (measured, and stable across two runs)

The counts below use **tracked source** (`git ls-files`), which is the honest measure of the hand-written codebase. The counting command was run **twice** and produced identical numbers:

```console
$ for ext in c h py go glsl; do echo "$ext: files=$(git ls-files "*.$ext"|wc -l) lines=$(git ls-files "*.$ext"|xargs cat|wc -l)"; done
--- RUN 1 ---
.c    files=128  lines=61806
.h    files=84   lines=37939
.py   files=214  lines=62874
.go   files=258  lines=56071
.glsl files=13   lines=696
--- RUN 2 ---
.c    files=128  lines=61806
.h    files=84   lines=37939
.py   files=214  lines=62874
.go   files=258  lines=56071
.glsl files=13   lines=696
```

| Language | Files | Lines | Runtime role |
|----------|-------|-------|--------------|
| C (`.c`) | 128 | 61,806 | Performance core (`fast_data_types`, bundled GLFW) |
| C headers (`.h`) | 84 | 37,939 | C interfaces |
| Python (`.py`) | 214 | 62,874 | Orchestration, configuration, kittens |
| Go (`.go`) | 258 | 56,071 | Standalone CLI tools / `kitten` binary |
| GLSL (`.glsl`) | 13 | 696 | GPU shader programs |

The headline is that **C source (61,806 lines) plus C headers (37,939 lines) = 99,745 lines of C**, comfortably the largest body of code, and it is exactly the code that runs on every keystroke and every frame. Python is similar in raw line count (62,874) but, as Q3/Q4 show, it spends those lines on orchestration and configuration, delegating the hot work to C. GLSL is tiny in size (696 lines) but, as Q2 shows, disproportionately central — it is where every visible pixel is produced.

> **Counting-method note (important).** Do **not** count lines with a `find … -print -exec cat {} +` pipeline: `-print` emits each *filename* as a line **and** `cat` emits the file *contents*, so the pipeline double-emits and inflates each language's line total by exactly its file count (for example GLSL would come out as 709 instead of the correct 696 — the delta of 13 equals the number of `.glsl` files). The correct forms are `git ls-files "*.EXT" | xargs cat | wc -l` (used above) or `find … -print0 | xargs -0 cat | wc -l`.

A raw on-disk `find` reports higher C/H/Go counts than `git ls-files`, because the build generates additional files:

```console
$ find . -path ./.git -prune -o -type f -name '*.EXT' ...  (on-disk incl build artifacts)
.c    files=142  lines=63225
.h    files=100  lines=48447
.go   files=338  lines=69151
```

The differences are entirely **build-generated, untracked** files, confirmed by differencing on-disk against tracked:

- **+14 `.c`**: `glfw/wayland-*-client-protocol.c` (wayland-scanner output).
- **+16 `.h`**: the 14 matching `glfw/wayland-*-client-protocol.h` plus `kitty/docs_ref_map_generated.h` and `kitty/uniforms_generated.h`.
- **+78 `.go`**: mostly `*_generated.go` (e.g. `constants_generated.go`, `kittens/*/cli_generated.go`).

So the tracked-source table above is the authoritative footprint; the on-disk inflation is generated code, not hand-written source.

### The C core is the hot path

The principal C-core files, by size:

```console
$ wc -l kitty/screen.c kitty/graphics.c kitty/child-monitor.c kitty/vt-parser.c kitty/state.c kitty/shaders.c kitty/gl.c kitty/simd-string-128.c kitty/simd-string-256.c
  4932 kitty/screen.c
  2431 kitty/graphics.c
  2016 kitty/child-monitor.c
  1596 kitty/vt-parser.c
  1492 kitty/state.c
  1285 kitty/shaders.c
   400 kitty/gl.c
     9 kitty/simd-string-128.c
     9 kitty/simd-string-256.c
 14170 total
```

Mapping each to its runtime responsibility (`file:line` = the file; sizes above):

- `kitty/screen.c` (4,932) — the terminal grid / screen model: the in-memory representation of every cell, scrollback, and cursor state.
- `kitty/graphics.c` (2,431) — the graphics-protocol image handling (kitty's image display).
- `kitty/child-monitor.c` (2,016) — the threaded PTY I/O and render loop: a dedicated C thread polls the child process's file descriptors and drives rendering.
- `kitty/vt-parser.c` (1,596) — the VT escape-sequence parser that turns raw bytes from the PTY into screen mutations.
- `kitty/state.c` (1,492) — global state management.
- `kitty/shaders.c` (1,285) — OpenGL shader compilation and linking (see Q2).
- `kitty/gl.c` (400) — the thin OpenGL wrapper.
- `kitty/simd-string-128.c` / `kitty/simd-string-256.c` (9 each) — tiny SIMD-dispatch shims over a shared implementation, i.e. hand-vectorized string scanning for the parser hot path.

The native windowing layer is the bundled GLFW, itself C:

```console
$ git ls-files 'glfw/*.c' | wc -l
31

$ git ls-files 'glfw/*.m' 'glfw/*.c' | grep -E 'cocoa|null|init|wl_|x11_'
glfw/cocoa_init.m
glfw/cocoa_joystick.m
glfw/cocoa_monitor.m
glfw/cocoa_window.m
glfw/init.c
glfw/null_init.c
glfw/null_joystick.c
glfw/null_monitor.c
glfw/null_window.c
glfw/wl_client_side_decorations.c
glfw/wl_cursors.c
glfw/wl_init.c
glfw/wl_monitor.c
glfw/wl_text_input.c
glfw/wl_window.c
glfw/x11_init.c
glfw/x11_monitor.c
glfw/x11_window.c
```

That is 31 tracked `.c` files plus platform sources — Cocoa (`.m`) for macOS, and X11 (`x11_*.c`) and Wayland (`wl_*.c`) backends for Linux — the entire cross-platform window/input layer, in C.

The project states its own GPU intent on the first line of its README:

```console
$ head -1 README.asciidoc
= kitty - the fast, feature-rich, cross-platform, GPU based terminal
```

### The performance is observably GPU-backed

With the `.so` present, kitty actually runs. Under a headless X server with Mesa's software GL, the canonical **C launcher** creates a real **OpenGL 4.5 Core Profile** context:

```console
$ LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a ./kitty/launcher/kitty --debug-gl -o confirm_os_window_close=0 -o enable_audio_bell=no sh -c 'echo RENDERED_OK; sleep 1; exit 0' 2>&1; echo "[exit: $?]"
[0.142] OS Window created
[0.151] Failed to open systemd user bus with error: Connection refused
[0.155] Child launched
[0.120] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[exit: 0]
```

The window is created, a child is launched, and an OpenGL 4.5 context is established — direct evidence that rendering is offloaded to the GL pipeline (here backed by Mesa's LLVMpipe because there is no physical GPU in the container). Note that the `echo RENDERED_OK` text does **not** appear on stdout: it was written into the child's PTY and rendered into the GPU-backed terminal surface, which is itself confirmation that terminal output is drawn by the GL pipeline rather than echoed to the controlling terminal.

### The Go layer is separate

```console
$ sed -n '1,3p' go.mod
module kitty

go 1.22
```

Go compiles to an independent static binary (`kitty/launcher/kitten`, 16 MB — see Q4). It is not loaded into kitty's address space and is not part of the per-frame hot path; it powers the command-line `kitten` tools.

### Rationale — why this distribution means "performance comes from C + GPU"

Cause and effect: every time a byte arrives from the shell, it flows through the C `vt-parser.c`, mutates the C `screen.c` grid, and — when cells are dirty — is drawn by the C/GL pipeline in `shaders.c`/`gl.c` using a glyph atlas, with SIMD (`simd-string-*.c`) accelerating the byte scanning. None of this per-byte/per-frame work is done in Python; Python's role (Q3) is to import and wire these C components together at startup and then dispatch high-level events. That is precisely why the source "feels like a tight weave of Python and C" yet is GPU-accelerated: the weave is real, but at runtime the C side (plus the GPU) carries the load.

**External corroboration** (observations above remain primary):

- DeepWiki's architectural overview (deepwiki.com/kovidgoyal/kitty) describes kitty as a hybrid Python/C/Go application in which performance-critical operations — terminal state management, rendering, and I/O — are implemented in C, while application logic, configuration, and extensibility are in Python, and a growing set of kittens/tools are in Go.
- Wikipedia's entry (en.wikipedia.org/wiki/Kitty_(terminal_emulator)) describes kitty as GPU-accelerated and "written in a mix of C, Python and Go."
- The Geeks3D writeup summarizes the split as C "for performance sensitive parts," Python for extensibility/UI, and Go "for the command line kittens."
- The official site (sw.kovidgoyal.net/kitty) bills kitty as a "GPU based terminal emulator" that uses the GPU and SIMD vector CPU instructions.

---

## Q2 — What role do the GLSL files play, and how central are they?

**Direct answer.** The `.glsl` files are **OpenGL shader source programs that drive the GPU rendering pipeline** — they are the mechanism by which kitty draws cells/text, window borders, background images, graphics-protocol images, and background tint on the GPU. They are **highly central**: without them there is no on-screen output, and — a key finding — their compilation itself flows through the same native bridge (`kitty.fast_data_types`) that everything else depends on (linking Q2 directly to Q3/Q4).

### Enumerating the shaders

```console
$ ls -1 kitty/*.glsl
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

$ wc -l kitty/*.glsl
   22 kitty/alpha_blend.glsl
   14 kitty/bgimage_fragment.glsl
   48 kitty/bgimage_vertex.glsl
    6 kitty/border_fragment.glsl
   48 kitty/border_vertex.glsl
   31 kitty/cell_defines.glsl
  204 kitty/cell_fragment.glsl
  233 kitty/cell_vertex.glsl
   27 kitty/graphics_fragment.glsl
   24 kitty/graphics_vertex.glsl
   15 kitty/linear2srgb.glsl
    6 kitty/tint_fragment.glsl
   18 kitty/tint_vertex.glsl
  696 total
```

All 13 files live under `kitty/` and total **696** lines.

### Observed decomposition: 5 vertex/fragment pairs + 3 shared includes

The 13 files are **not** 13 independent shaders. The observed structure is **five true vertex/fragment source pairs** plus **three shared GLSL includes**:

- **Five `*_vertex` / `*_fragment` pairs (10 files):** `cell`, `border`, `bgimage`, `graphics`, `tint`.
- **Three shared includes (3 files):** `alpha_blend.glsl`, `linear2srgb.glsl`, `cell_defines.glsl` — pulled into the pairs via a `#pragma kitty_include_shader <…>` directive rather than compiled on their own.

The include relationships are observable directly in the sources:

```console
$ grep -n 'kitty_include_shader' kitty/cell_fragment.glsl kitty/cell_vertex.glsl kitty/graphics_fragment.glsl
kitty/cell_fragment.glsl:1:#pragma kitty_include_shader <alpha_blend.glsl>
kitty/cell_fragment.glsl:2:#pragma kitty_include_shader <linear2srgb.glsl>
kitty/cell_fragment.glsl:3:#pragma kitty_include_shader <cell_defines.glsl>
kitty/cell_vertex.glsl:2:#pragma kitty_include_shader <cell_defines.glsl>
kitty/graphics_fragment.glsl:1:#pragma kitty_include_shader <alpha_blend.glsl>
```

> **Reconciliation with the plan's "6 program pairs."** The upstream analysis loosely counted the `alpha_blend`+`linear2srgb` helpers as a sixth "pair." The **observed reality is 5 vertex/fragment pairs + 3 shared includes**; `cell_defines.glsl`, `alpha_blend.glsl`, and `linear2srgb.glsl` are `#include`-style fragments, not standalone programs. The observed value is used here.

### Loading and macro substitution (Python side): `kitty/shaders.py`

```console
$ grep -n -E 'from .fast_data_types import|GLSL_VERSION|compile_program|kitty_include_shader|_load_sources|#version|program_for|class MultiReplacer|class LoadShaderPrograms|def __call__|load_shader_programs = ' kitty/shaders.py
10:from .fast_data_types import (
19:    GLSL_VERSION,
29:    compile_program,
53:            Program.include_pat = re.compile(r'^#pragma\s+kitty_include_shader\s+<(.+?)>', re.MULTILINE)
61:    def _load_sources(self, name: str, seen: Set[str], level: int = 0) -> Iterator[str]:
63:            yield f'#version {GLSL_VERSION}\n'
90:            compile_program(program_id, self.vertex_sources, self.fragment_sources, allow_recompile)
108:def program_for(name: str) -> Program:
112:class MultiReplacer:
124:    def __call__(self, src: str) -> str:
131:class LoadShaderPrograms:
147:    def __call__(self, semi_transparent: bool = False, allow_recompile: bool = False) -> None:
204:load_shader_programs = LoadShaderPrograms()
```

Reading this trace:

- **`kitty/shaders.py:10`** opens a `from .fast_data_types import (…)` block — the Python shader loader pulls its program IDs and, crucially, its compiler **from the native C extension**.
- **`kitty/shaders.py:19`** imports `GLSL_VERSION` (the `#version` string prepended to every shader).
- **`kitty/shaders.py:29`** imports **`compile_program`** — the actual shader compiler — **from `fast_data_types`**. This is the key structural fact for Q2: **shader compilation is a C function**, and the Python side merely feeds it source strings.
- **`kitty/shaders.py:53`** compiles the include regex `^#pragma\s+kitty_include_shader\s+<(.+?)>`.
- **`kitty/shaders.py:61`** `def _load_sources(...)` recursively resolves those includes, and **`:63`** prepends `#version {GLSL_VERSION}` to the assembled source.
- **`kitty/shaders.py:90`** calls `compile_program(program_id, self.vertex_sources, self.fragment_sources, allow_recompile)` — the hand-off into C.
- **`kitty/shaders.py:108`** `program_for(name)` returns a (cached) `Program`.
- **`kitty/shaders.py:112–127`** `class MultiReplacer` implements `{PLACEHOLDER}` substitution (its pattern is `\{([A-Z_]+)\}`), used to specialize a single GLSL source into multiple GL programs.
- **`kitty/shaders.py:131` / `:147`** `class LoadShaderPrograms` and its `__call__` do the actual compilation of all programs at startup.
- **`kitty/shaders.py:204`** instantiates the singleton `load_shader_programs = LoadShaderPrograms()`.

The imported program IDs make the shader→program mapping explicit. The native import block reads:

```console
--- shaders.py L10-31 (native import block) ---
from .fast_data_types import (
    BGIMAGE_PROGRAM,
    CELL_BG_PROGRAM,
    CELL_FG_PROGRAM,
    CELL_PROGRAM,
    CELL_SPECIAL_PROGRAM,
    DECORATION,
    DECORATION_MASK,
    DIM,
    GLSL_VERSION,
    GRAPHICS_ALPHA_MASK_PROGRAM,
    GRAPHICS_PREMULT_PROGRAM,
    GRAPHICS_PROGRAM,
    MARK,
    MARK_MASK,
    NUM_UNDERLINE_STYLES,
    REVERSE,
    STRIKETHROUGH,
    TINT_PROGRAM,
    compile_program,
    get_options,
    init_cell_program,
```

### Compilation (C side): `kitty/shaders.c`

```console
$ grep -n -E 'compile_program\(PyObject|glCreateProgram\(\)|glLinkProgram\(program->id\)|M\(compile_program' kitty/shaders.c
1168:compile_program(PyObject UNUSED *self, PyObject *args) {
1179:    program->id = glCreateProgram();
1182:    glLinkProgram(program->id);
1236:    M(compile_program, METH_VARARGS),
```

This is a clean Python→C bridge:

- **`kitty/shaders.c:1168`** defines `compile_program(PyObject *self, PyObject *args)` — the C function.
- **`kitty/shaders.c:1179`** calls `glCreateProgram()` and **`:1182`** `glLinkProgram(program->id)` — the real OpenGL calls that build and link the GPU program.
- **`kitty/shaders.c:1236`** registers it to Python via `M(compile_program, METH_VARARGS)`.

The `compile_program` imported at `shaders.py:29` is exactly this C function at `shaders.c:1168` — Python assembles GLSL text, C compiles and links it on the GPU.

### Program expansion — one source pair, several GL programs

`LoadShaderPrograms.__call__` (`shaders.py:147`) specializes the source pairs into multiple GL programs by substituting `{PLACEHOLDER}` macros:

- **`cell` → 4 programs** via a `WHICH_PHASE` substitution: `CELL_PROGRAM` (BOTH), `CELL_BG_PROGRAM` (BACKGROUND), `CELL_SPECIAL_PROGRAM` (SPECIAL), `CELL_FG_PROGRAM` (FOREGROUND).
- **`graphics` → 3 programs** by rewriting `#define ALPHA_TYPE` into a concrete `#define`: `GRAPHICS_PROGRAM` (SIMPLE), `GRAPHICS_PREMULT_PROGRAM` (PREMULT), `GRAPHICS_ALPHA_MASK_PROGRAM` (ALPHA_MASK).
- **`bgimage` → `BGIMAGE_PROGRAM`**, **`tint` → `TINT_PROGRAM`** (one each).
- **`border` → `BORDERS_PROGRAM`**, compiled via `kitty/borders.py` whose `init_borders_program` is imported from `fast_data_types` at `kitty/borders.py:7` (the very import that fails in Q3).

`cell_defines.glsl` is the shared 31-line `#define` include carrying the `{WHICH_PHASE}`, `{TRANSPARENT}`, `{FG_OVERRIDE}`, `{FG_OVERRIDE_THRESHOLD}`, `{TEXT_NEW_GAMMA}`, `{*_SHIFT}`, and `{MARK_MASK}` placeholders that `MultiReplacer` fills in.

### Each shader mapped to a rendering stage

| Shader pair / include | Rendering stage |
|-----------------------|-----------------|
| `cell_{vertex,fragment}` | Text and the cell grid (the terminal's core output) |
| `border_{vertex,fragment}` | Window borders between panes |
| `bgimage_{vertex,fragment}` | Background images |
| `graphics_{vertex,fragment}` | kitty graphics-protocol images (`icat`, image display) |
| `tint_{vertex,fragment}` | Background tint overlay |
| `alpha_blend.glsl`, `linear2srgb.glsl` | Shared color-space / alpha-blending helper includes |
| `cell_defines.glsl` | Shared `#define`/placeholder include for the cell programs |

The glyphs consumed by the cell programs are produced by the font pipeline in `kitty/fonts/__init__.py` (191 lines), which rasterizes glyphs into the sprite/texture atlas that the cell shaders sample — the atlas model that makes GPU text rendering cheap.

### Centrality rationale

Every visible pixel kitty draws is produced by one of these GL programs, so the shaders are not peripheral — they *are* the rendering. And because the compiler (`compile_program`) is imported from `fast_data_types` (`shaders.py:29`) and the border program's initializer is imported at `borders.py:7`, the shader subsystem sits **on top of the same C bridge** that the rest of the application binds to at import time. That is why Q2 is inseparable from Q3/Q4: the GPU pipeline is central, and it is reached through the one native module everything depends on.

**External corroboration.** Independent descriptions of the GPU model match the observed pipeline: sources describe kitty as rendering text with OpenGL by rasterizing each glyph once into a texture atlas so that cost scales with the number of *unique* glyphs rather than characters on screen (petronellatech.com; datadaily.news), and DeepWiki notes that dirty cells trigger a GPU pipeline using a sprite atlas and OpenGL shaders. These corroborate the centrality of the shader programs enumerated above.


---

## Q3 — Why does the main entry point fail immediately, and what is the one critical piece?

**Direct answer.** Running the repository-root `__main__.py` halts almost immediately because the native module **`kitty.fast_data_types`** — a compiled `.so` extension with **no `.py` fallback** — is imported at module-load time, and when it is absent the import raises `ModuleNotFoundError` before any application logic executes. That single compiled extension is the **"one critical piece that everything depends on."**

### The failure (pre-build state, real canonical entry point)

Reproduced by temporarily moving the gitignored `kitty/fast_data_types.so` aside (fully restored afterward — see Appendix) and running the **real** entry point under the canonical interpreter:

```console
$ /opt/kitty311-venv/bin/python3 __main__.py 2>&1; echo "[exit: $?]"
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/__main__.py", line 7, in <module>
    main()
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/entry_points.py", line 194, in main
    from kitty.main import main as kitty_main
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 11, in <module>
    from .borders import load_borders_program
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/borders.py", line 7, in <module>
    from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image
ModuleNotFoundError: No module named 'kitty.fast_data_types'
[exit: 1]
```

### The import chain, line by line

The traceback *is* the chain; each hop is confirmed in source:

```console
$ sed -n '5,7p' __main__.py
if __name__ == '__main__':
    from kitty.entry_points import main
    main()

$ sed -n '183p;194p' kitty/entry_points.py
def main() -> None:
            from kitty.main import main as kitty_main

$ sed -n '11p' kitty/main.py
from .borders import load_borders_program

$ sed -n '7,8p' kitty/borders.py
from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image
from .shaders import program_for
```

- **`__main__.py:7`** — inside `if __name__ == '__main__':` (`:5`), after `from kitty.entry_points import main` (`:6`), it calls `main()`.
- **`kitty/entry_points.py:183`** defines `def main()`; the executed branch at **`:194`** does `from kitty.main import main as kitty_main`.
- **`kitty/main.py:11`** does `from .borders import load_borders_program`, pulling in `borders`.
- **`kitty/borders.py:7`** executes `from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image` — the module-level import of the native bridge. (Note `:8` immediately also does `from .shaders import program_for`, which — per Q2 — would itself require the same bridge.)

### There is no `.py` fallback — only a type stub

```console
$ ls -la kitty/fast_data_types*
-rw-r--r-- 1 root root   35795 Jul 14 18:52 kitty/fast_data_types.pyi
-rwxr-xr-x 1 root root 1248952 Jul 14 19:10 kitty/fast_data_types.so
```

The only non-`.so` file is `kitty/fast_data_types.pyi`, a **type stub** used by static type checkers — it is not importable at runtime. There is no `kitty/fast_data_types.py`. The module is produced exclusively by the build: `setup.py` defines `compile_c_extension` (`setup.py:856`) and invokes it for the `fast_data_types` target at `setup.py:1090`. So when the `.so` is absent, the import has nothing to resolve to, and startup dies at the first hop into `kitty.main`.

### Variant: `python3 -m kitty` fails *differently* (confirming the canonical path)

```console
$ /opt/kitty311-venv/bin/python3 -m kitty 2>&1; echo "[exit: $?]"
/opt/kitty311-venv/bin/python3: No module named kitty.__main__; 'kitty' is a package and cannot be directly executed
[exit: 1]
```

This is a **different** failure with a different cause. `python -m kitty` asks Python to execute the `kitty` *package*, which requires a `kitty/__main__.py` — and there is none:

```console
$ ls kitty/__main__.py 2>&1
ls: cannot access 'kitty/__main__.py': No such file or directory
```

So the `-m kitty` form never even reaches the `fast_data_types` import; it is rejected earlier by the interpreter. This confirms that the **repository-root `__main__.py` is the canonical entry point**, not `-m kitty`. (The `/opt/kitty311-venv/bin/python3` prefix reflects the canonical venv interpreter used here; a different interpreter would only change that prefix, not the message.)

### Post-build: the import succeeds, and the truly canonical launcher is the C launcher

With the `.so` present, the `fast_data_types` import **succeeds** and execution proceeds further. Running the entry point directly under the canonical venv now fails at a *later*, different point:

```console
$ /opt/kitty311-venv/bin/python3 __main__.py 2>&1; echo "[exit: $?]"
[0.222] Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 526, in main
    _main()
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 495, in _main
    setup_environment(opts, cli_opts)
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 411, in setup_environment
    ensure_kitty_in_path()
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 364, in ensure_kitty_in_path
    krd = getattr(sys, 'kitty_run_data')
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: module 'sys' has no attribute 'kitty_run_data'

[exit: 1]
```

The failure has moved from the `borders.py:7` import all the way to `kitty/main.py:364`, where `ensure_kitty_in_path()` reads `getattr(sys, 'kitty_run_data')`. That attribute is injected by kitty's **C launcher**, not by plain `python`. This is why the canonical run uses `./kitty/launcher/kitty` (Q1), which starts successfully:

```console
$ ./kitty/launcher/kitty --version ; echo "[exit: $?]"
kitty 0.35.2 created by Kovid Goyal
[exit: 0]
```

Two things are proven at once: (a) the `.so` is exactly what unblocks the import — with it present, startup gets past `borders.py:7`; and (b) the C launcher is the outermost canonical wrapper, injecting `sys.kitty_run_data` before Python runs.

(Inferred: the earlier version-specific failure observed under the *system* interpreter 3.13.7 — a `ValueError: not enough values to unpack` at `kitty/cli.py:49` during option parsing — is a consequence of running above kitty's documented Python ceiling of 3.11, not a defect in kitty. It is noted only to justify using the 3.11.15 venv for canonical observations; the complete captured output is included in the Appendix.)

### Supporting orchestration context

```console
$ sed -n '63p' kitty/boss.py
from .fast_data_types import (

$ sed -n '23p;25p;66p' kitty/constants.py
appname: str = 'kitty'
version: Version = Version(0, 35, 2)
def kitty_exe() -> str:
```

`kitty/boss.py` (3,094 lines) is the central startup orchestrator and itself imports the bridge at `:63`; `kitty/constants.py` holds `appname='kitty'` (`:23`), `version = Version(0, 35, 2)` — i.e. kitty 0.35.2 (`:25`) — and `kitty_exe()` (`:66`).

### Rationale — cause and effect

Python resolves `import` statements **eagerly at module load**. The very first hop, `from kitty.main import main` (`entry_points.py:194`), transitively imports `kitty.borders`, whose module-level `from .fast_data_types import …` (`borders.py:7`) executes immediately as the module body runs. Because the compiled extension is the only provider of that module and it was absent, the import raises `ModuleNotFoundError` **before any terminal, window, or event-loop logic runs** — hence the program "fails almost immediately." This demonstrates that Python is wired into the native core **at import time**, not lazily: the C extension is not an optional accelerator but a hard, load-bearing dependency of the Python layer.


---

## Q4 — Are kittens independent, or do they rely on the same native bridge?

**Direct answer.** The **Python kittens are not independent** — they quietly depend on the **same** native bridge, `kitty.fast_data_types`. Running a kitten standalone fails with the identical `ModuleNotFoundError` when the `.so` is absent. In contrast, the **Go reimplementations under `tools/cmd/` are self-contained** and carry no dependency on the Python extension. So the system is modular *across* the language boundary (Go) but tightly coupled *within* the Python layer.

### Standalone kitten via the canonical launcher `kittens/runner.py`

```console
$ /opt/kitty311-venv/bin/python3 -c "import kittens.runner" 2>&1; echo "[exit: $?]"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kittens/runner.py", line 14, in <module>
    from kitty.utils import resolve_abs_or_config_path
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/utils.py", line 45, in <module>
    from .fast_data_types import WINDOW_FULLSCREEN, WINDOW_MAXIMIZED, WINDOW_MINIMIZED, WINDOW_NORMAL, Color, Shlex, get_options, monotonic, open_tty
ModuleNotFoundError: No module named 'kitty.fast_data_types'
[exit: 1]
```

Importing the kitten runner reaches the bridge through `kitty.utils`: `kittens/runner.py:14` (`from kitty.utils import resolve_abs_or_config_path`) → `kitty/utils.py:45` (`from .fast_data_types import …`) → the same `ModuleNotFoundError`.

### An individual kitten too (edge path, same root cause)

```console
$ /opt/kitty311-venv/bin/python3 -m kittens.hints.main 2>&1; echo "[exit: $?]"
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kittens/hints/main.py", line 9, in <module>
    from kitty.clipboard import set_clipboard_string, set_primary_selection
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/clipboard.py", line 11, in <module>
    from .conf.utils import uniq
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
[exit: 1]
```

The `hints` kitten reaches the bridge by a **different sub-path** — `kittens/hints/main.py:9` (`from kitty.clipboard import …`) → `kitty/clipboard.py:11` (`from .conf.utils import uniq`) → `kitty/conf/utils.py:27` (`from ..fast_data_types import Color`) — but the terminal cause is **identical**. Different roads, same bridge.

The relevant source lines confirm both sub-paths and the shared consumers:

```console
$ sed -n '14p' kittens/runner.py ; sed -n '45p' kitty/utils.py ; sed -n '15p' kitty/cli.py
from kitty.utils import resolve_abs_or_config_path
from .fast_data_types import WINDOW_FULLSCREEN, WINDOW_MAXIMIZED, WINDOW_MINIMIZED, WINDOW_NORMAL, Color, Shlex, get_options, monotonic, open_tty
from .fast_data_types import wcswidth

$ sed -n '9p;11p' kittens/hints/main.py ; sed -n '11p' kitty/clipboard.py ; sed -n '27p' kitty/conf/utils.py
from kitty.clipboard import set_clipboard_string, set_primary_selection
from kitty.fast_data_types import get_options
from .conf.utils import uniq
from ..fast_data_types import Color
```

Note that `kitty/cli.py:15` (`from .fast_data_types import wcswidth`) means even the shared **CLI option parser** used by kittens needs the bridge, and `kitty/utils.py:45` is the common shared consumer that both the runner path and many kittens transit.

### Every Python kitten module that imports the bridge

```console
$ grep -rln fast_data_types kittens/ | sort
kittens/hints/main.py
kittens/panel/main.py
kittens/query_terminal/main.py
kittens/remote_file/main.py
kittens/runner.py
kittens/tui/handler.py
kittens/tui/images.py
kittens/tui/line_edit.py
kittens/tui/loop.py
kittens/tui/operations.py
kittens/tui/path_completer.py
kittens/tui/spinners.py
kittens/tui/utils.py
```

That is **13 source modules** (all `.py`; no `.pyc` cache files surfaced here because these runs used `PYTHONDONTWRITEBYTECODE=1`). Their exact import sites:

```console
$ grep -n 'fast_data_types' kittens/runner.py kittens/hints/main.py kittens/panel/main.py kittens/remote_file/main.py kittens/tui/handler.py kittens/tui/images.py kittens/tui/line_edit.py kittens/tui/loop.py kittens/tui/operations.py kittens/tui/path_completer.py kittens/tui/spinners.py
kittens/runner.py:129:    from kitty.fast_data_types import set_options
kittens/hints/main.py:11:from kitty.fast_data_types import get_options
kittens/panel/main.py:10:from kitty.fast_data_types import (
kittens/remote_file/main.py:360:        from kitty.fast_data_types import get_options
kittens/tui/handler.py:10:from kitty.fast_data_types import monotonic
kittens/tui/images.py:15:from kitty.fast_data_types import create_canvas
kittens/tui/line_edit.py:6:from kitty.fast_data_types import truncate_point_for_length, wcswidth
kittens/tui/loop.py:19:from kitty.fast_data_types import FILE_TRANSFER_CODE, close_tty, normal_tty, open_tty, parse_input_from_terminal, raw_tty
kittens/tui/operations.py:11:from kitty.fast_data_types import Color
kittens/tui/operations.py:464:        'from kitty.fast_data_types import Color',
kittens/tui/path_completer.py:8:from kitty.fast_data_types import wcswidth
kittens/tui/spinners.py:6:from kitty.fast_data_types import monotonic

$ grep -n 'fast_data_types' kittens/query_terminal/main.py kittens/tui/utils.py | head
kittens/query_terminal/main.py:100:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:112:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:124:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:136:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:148:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:159:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:170:        from kitty.fast_data_types import current_fonts
kittens/query_terminal/main.py:182:        from kitty.fast_data_types import Color, get_boss
kittens/query_terminal/main.py:199:        from kitty.fast_data_types import Color, get_boss
kittens/query_terminal/main.py:221:    from kitty.fast_data_types import get_options
```

| # | Kitten module | Representative import site |
|---|---------------|----------------------------|
| 1 | `kittens/runner.py` | `:14` via `kitty.utils`; `:129` `from kitty.fast_data_types import set_options` |
| 2 | `kittens/hints/main.py` | `:11` `get_options` (also transitively via `kitty.clipboard` → `conf/utils:27`) |
| 3 | `kittens/panel/main.py` | `:10` `from kitty.fast_data_types import (…)` |
| 4 | `kittens/query_terminal/main.py` | `:100+` `current_fonts`; `:182/199` `Color, get_boss`; `:221` `get_options` |
| 5 | `kittens/remote_file/main.py` | `:360` `get_options` |
| 6 | `kittens/tui/handler.py` | `:10` `monotonic` |
| 7 | `kittens/tui/images.py` | `:15` `create_canvas` |
| 8 | `kittens/tui/line_edit.py` | `:6` `truncate_point_for_length, wcswidth` |
| 9 | `kittens/tui/loop.py` | `:19` `FILE_TRANSFER_CODE, close_tty, normal_tty, open_tty, parse_input_from_terminal, raw_tty` |
| 10 | `kittens/tui/operations.py` | `:11` `Color` |
| 11 | `kittens/tui/path_completer.py` | `:8` `wcswidth` |
| 12 | `kittens/tui/spinners.py` | `:6` `monotonic` |
| 13 | `kittens/tui/utils.py` | `:60/:76` `get_options` / `set_options` |

The concentration in `kittens/tui/*` is telling: the shared TUI framework that virtually every Python kitten builds on imports the bridge at module load, so importing *any* kitten that uses the TUI transitively loads `fast_data_types`.

### The Go contrast — self-contained tooling

```console
$ grep -rl fast_data_types tools/ ; echo "[exit: $?]"
[exit: 1]
```

No file under `tools/` references `fast_data_types` (grep found nothing; exit code 1 = no matches). The Go reimplementations live under `tools/cmd/`:

```console
$ ls -1 tools/cmd/
at
benchmark
completion
edit_in_kitty
main.go
mouse_demo
pytest
run_shell
show_error
tool
update_self

$ sed -n '1,3p' go.mod
module kitty

go 1.22
```

And the Go `kitten` binary runs standalone, with no Python `.so` involved:

```console
$ ./kitty/launcher/kitten --version ; echo "[exit: $?]"
kitten 0.35.2 created by Kovid Goyal
[exit: 0]
```

This is the before/after boundary that makes the modularity conclusion rest on **both** states: the Python kittens **fail** without the bridge, while the Go binary **succeeds** without it.

### Rationale

The Python kittens all funnel through a small set of shared modules — `kitty/utils.py` (`:45`), `kitty/cli.py` (`:15`), and the `kittens/tui/*` framework — each of which imports `fast_data_types` at module load. Because Python executes those imports eagerly, importing any Python kitten transitively loads the bridge and therefore fails without the `.so` (exactly as Q3's main entry point does). The Go tools, by contrast, are compiled into an independent static binary (module `kitty`, `go 1.22`) that never references the Python extension, so they run with no dependency on it. Hence kittens are **not** independent within the Python layer — they share the same native bridge — but kitty's tooling *is* genuinely modular where it crosses into Go.

**External corroboration.** Independent sources describe kittens as an extension/plugin system for kitty written in Python (and, increasingly, Go), consistent with the observed split between Python kittens that depend on the native bridge and self-contained Go reimplementations (Wikipedia; DeepWiki; sw.kovidgoyal.net/kitty).


---

## Synthesis — how the four findings fit together

The four observations converge on one coherent architecture:

- **C + GPU is the performance engine.** The largest body of code is C (Q1: 128 `.c` files / 61,806 lines plus 84 headers / 37,939 lines), and it is exactly the code on the per-byte and per-frame path — VT parsing, screen-grid mutation, PTY I/O, SIMD string scanning, and OpenGL rendering. A real OpenGL 4.5 context is created at runtime (Q1). The GLSL shader programs (Q2) are where every visible pixel is produced.
- **Python is an orchestration shell bound to the native core at import time.** Python's line count rivals C's, but Q3 shows those lines depend on the C extension the moment a module body runs: the main entry point dies at `borders.py:7` without the `.so`.
- **Go is an independent CLI layer.** The Go tooling under `tools/cmd/` references none of the Python extension (Q4) and runs as a standalone static binary.

### Q3 and Q4 are two paths to the *same* load-bearing dependency

The single most important structural fact is that the main-entry failure and the standalone-kitten failure are **the same failure reached by different routes** — both terminate at `kitty.fast_data_types`:

```
Q3 (main entry point):
  __main__.py:7  →  entry_points.py:194  →  main.py:11  →  borders.py:7  ─┐
                                                                          ├─→  kitty.fast_data_types (.so)
Q4 (standalone kitten):                                                   │        └─ absent ⇒ ModuleNotFoundError
  kittens/runner.py:14  →  kitty/utils.py:45  ───────────────────────────┤
  kittens/hints/main.py:9 → kitty/clipboard.py:11 → kitty/conf/utils.py:27┤
  kitty/cli.py:15  ────────────────────────────────────────────────────-─┘
```

Whether you start kitty proper or a lone kitten, Python's eager, module-load-time imports drive you into the one compiled extension. That is why the codebase "feels like a tight weave of Python and C": the weave is literally load-bearing, stitched at import time.

### Build and version context

```console
$ wc -l setup.py
2172 setup.py

$ sed -n '492p;856p;1090p;1138p' setup.py
    std = '' if is_openbsd else '-std=c11'
def compile_c_extension(
    compile_c_extension(
        raise SystemExit('The go tool was not found on this system. Install Go')

$ sed -n '12,13p' Makefile
all:
	python3 setup.py $(VVAL)

$ sed -n '1,2p' pyproject.toml
[project]
requires-python = ">=3.8"

$ sed -n '85p' .github/workflows/ci.yml
              python-version: "3.11"

$ sed -n '27,28p' CONTRIBUTING.md
tests as well (see the `kitty_tests/` sub-directory for existing tests, which
can be run with `./test.py`).
```

- The canonical build is `python3 setup.py` (`Makefile:12–13`), which compiles the C extension via `compile_c_extension` (defined at `setup.py:856`, invoked for `fast_data_types` at `setup.py:1090`) using `-std=c11` (`setup.py:492`), and **hard-requires the Go tool** — `raise SystemExit('The go tool was not found on this system. Install Go')` (`setup.py:1138`).
- Version floors: `pyproject.toml:2` `requires-python = ">=3.8"`; the CI matrix's highest single documented interpreter is `"3.11"` (`.github/workflows/ci.yml:85`); Go is `1.22` (`go.mod:3`). This is why the 3.11.15 venv was used for canonical observations.
- Tests live in `kitty_tests/` and run via `./test.py` (`CONTRIBUTING.md:27–28`).

### Resolving the user's tension

The "tight weave of Python and C" is real at the **source** level — Python and C modules are interleaved throughout `kitty/`. But at **runtime** the weave is **asymmetric**: C (plus the GPU, via the GLSL programs) carries the per-byte and per-frame work, while Python only orchestrates and configures. The weave is load-bearing precisely because Python binds to `fast_data_types` at import time, so nothing in the Python layer — not the main app, not a lone kitten — runs without the compiled core. Go sits outside this weave entirely, as an independent CLI layer. So kitty is legitimately "GPU accelerated": the acceleration lives in the C core and the shader pipeline, and the Python you see in the tree is the thin shell that wires that core together.

---

## Appendix

### A. Environment

| Item | Value |
|------|-------|
| Repository root | `/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770` |
| Git branch | `blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd` (document name derives from the source branch `kitty_815df1e210e0`) |
| Canonical interpreter | `/opt/kitty311-venv/bin/python3` — CPython **3.11.15** (built the `.so`; matches kitty's documented ceiling) |
| System interpreter | `/usr/bin/python3` — CPython 3.13.7 (above ceiling; used only for the version-boundary illustration) |
| kitty version | 0.35.2 (`kitty/constants.py:25`; `./kitty/launcher/kitty --version`) |
| Native extension | `kitty/fast_data_types.so` present (1,248,952 bytes); only stub `kitty/fast_data_types.pyi` otherwise |

**Both states observed.** Because the built `.so` was present in this environment, the *post-build* success signals (OpenGL 4.5 context; `./kitty/launcher/kitty --version`) were observed directly, and the *pre-build* failure signals (Q3/Q4 `ModuleNotFoundError`) were reproduced by temporarily moving the gitignored `.so` aside and restoring it. This before/after pairing is stronger evidence than either state alone: the same commands fail without the `.so` and progress with it.

### B. Version-boundary illustration (system Python 3.13.7, above the 3.11 ceiling)

Running the entry point under the *system* interpreter (post-build, `.so` present) fails earlier than under the venv, at CLI option parsing — a consequence of running above kitty's documented Python ceiling, not a kitty defect:

```console
$ /usr/bin/python3 __main__.py 2>&1; echo "[exit: $?]"
[0.183] Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 526, in main
    _main()
    ~~~~~^^
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/main.py", line 464, in _main
    cli_opts, rest = parse_args(args=args, result_class=CLIOptions, usage=usage, message=msg, appname=appname)
                     ~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/cli.py", line 1054, in parse_args
    options = parse_option_spec(ospec())
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/cli.py", line 442, in parse_option_spec
    current_cmd['completion'] = CompletionSpec.from_string(v)
                                ~~~~~~~~~~~~~~~~~~~~~~~~~~^^^
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/cli.py", line 49, in from_string
    ck, vv = x.split(':', 1)
    ^^^^^^
ValueError: not enough values to unpack (expected 2, got 1)

[exit: 1]
```

Note this interpreter still imported `fast_data_types` successfully (it reached CLI parsing), confirming the `.so` loads under both interpreters; only the canonical 3.11.15 venv is used for the primary observations.

### C. Counting-method pitfall

Language line counts must avoid the `find … -print -exec cat {} +` anti-pattern, which double-emits (filename via `-print` **and** contents via `cat`) and inflates each language's line total by exactly its file count (e.g. GLSL 709 vs. the correct 696; the delta 13 = number of `.glsl` files). The counts in Q1 use `git ls-files "*.EXT" | xargs cat | wc -l` and were confirmed identical across two runs.

### D. Observed-vs-plan discrepancies (observed values win)

| Item | Plan (AAP) | Observed | Note |
|------|-----------|----------|------|
| `setup.py` length | 2173 lines | **2172 lines** (`wc -l`) | Off-by-one from trailing-newline / `wc` semantics |
| Shader decomposition | "6 program pairs" | **5 vertex/fragment pairs + 3 shared includes** | `alpha_blend`/`linear2srgb`/`cell_defines` are `#include`s, not standalone pairs |
| Kitten grep hits | 13 `.py` + 2 `.pyc` | **13 `.py`, 0 `.pyc`** | This run set `PYTHONDONTWRITEBYTECODE=1`, so no bytecode caches were created |
| `-m kitty` prefix | `/usr/bin/python3` | `/opt/kitty311-venv/bin/python3` | Only the interpreter prefix differs; the message is identical |
| Pre-build baseline | `.so` absent | `.so` **present** (post-build) | Pre-build failures reproduced by temporarily moving the `.so` aside, then restoring |

### E. Cleanliness verification

All observation scripts wrote their output **outside** the repository (under `/tmp`). The temporarily-moved `.so` was restored byte-for-byte (md5 `948052a62ac184d65defb3079fbf8161` before and after), and `*.so`, `*.pyc`, and `__pycache__/` are gitignored (`.gitignore:1,2,20`), so runtime artifacts never dirty the tree. Nothing under `/app` was accessed. The only change introduced anywhere is this document. The final working-tree state:

```console
$ git status --porcelain
?? blitzy/documentation/kitty_815df1e210e0.md
```

Apart from this newly created answer document, the working tree is byte-for-byte unchanged.

### F. External sources consulted (corroboration only; observations remain authoritative)

- DeepWiki — `kovidgoyal/kitty` architectural overview (deepwiki.com/kovidgoyal/kitty).
- Wikipedia — "kitty (terminal emulator)" (en.wikipedia.org/wiki/Kitty_(terminal_emulator)).
- Official project site (sw.kovidgoyal.net/kitty).
- Geeks3D project writeup (geeks3d.com).
- GPU glyph-atlas model background (petronellatech.com; datadaily.news).

