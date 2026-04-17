# kitty — An Empirical Investigation of a Three-Language Terminal Emulator

> **Repository**: `kovidgoyal/kitty`
> **Branch analyzed**: `kitty_815df1e210e0`
> **HEAD commit**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` — "Wire up applying of font config"
> **Investigation method**: Empirical — code executed, imports traced, resources cataloged. Not merely read.

---

## 1. Introduction

kitty is marketed as a "fast, feature-rich, GPU-based terminal emulator." Looking at the repository, this marketing claim is both true and incomplete. The codebase is a tight weave of three languages — C, Python, and Go — with a small but architecturally decisive amount of GLSL layered on top. On paper, it reads like a "Python project with some C extensions bolted on"; the top-level directory layout shows `__main__.py`, `pyproject.toml`, a `kitty/` package directory full of `.py` files, and a `kittens/` directory that looks like plugins. The instinctive reading would be that kitty is primarily a Python application with hot paths accelerated in C and a CLI tool written in Go.

That reading is wrong, and the purpose of this document is to show **why** it is wrong — not by argument, but by running the code and observing what it does.

### 1.1 Why empirical method matters here

Static source reading tells you what *could* happen. Running the code tells you what *does* happen. For a system like kitty, the difference is architectural rather than cosmetic:

- Static reading of `__main__.py` suggests a clean Python entry point. Executing it reveals a `ModuleNotFoundError` almost immediately, and the path of that error is diagnostic of the true dependency structure.
- Static reading of `kittens/` suggests a set of small, independent tools. Attempting to import each one reveals which are structurally coupled to the native core and which are thin shims for Go binaries.
- Static reading of `kitty/*.glsl` leaves it unclear whether shaders are optional or mandatory. Tracing the loading pipeline shows there is no CPU rendering fallback path — GPU shaders are the only way pixels ever reach the screen.

So everything in the sections that follow is grounded in observations I made by running the code. Numerical claims (line counts, file counts, import success/failure ratios) come from `git ls-files` and `importlib` invocations; tracebacks are verbatim copies of what Python printed; subsystem lists are transcribed from `kitty/data-types.c` directly. Where the investigation environment differed from the AAP's documented scenario (for example, `pkg-config` is present in this container, whereas the AAP describes a container where it was not), this document reports both what was observed here and what the documented behavior is — and explains why the distinction does not weaken the conclusion.

### 1.2 What the investigation covered

Six experiments were conducted on the repository at its HEAD commit, on a freshly prepared Linux build environment:

1. **Language line-count census** — counted every git-tracked source file by extension, to quantify how much of each language exists.
2. **Entry-point execution** — ran `python3 __main__.py` with and without the compiled C extension present, and captured the exact traceback.
3. **Module isolation test** — attempted `importlib.import_module()` on every Python module directly under `kitty/`, with the C extension hidden, to measure how much of the Python layer is inoperable without C.
4. **Kitten isolation test** — attempted the same for every kitten's `main.py`, to measure how much of the "plugin" layer is inoperable without C.
5. **GLSL shader analysis** — cataloged all `.glsl` files and traced the loading pipeline from Python bytes through to `glCompileShader()` in C.
6. **Build attempt** — ran `setup.py build` on a stock environment to observe the native-library prerequisites kitty's C core needs, and documented the failure modes.

The sections that follow answer five architectural questions in order: *Which language does the heavy lifting?* *What role do GLSL shaders play?* *Why does `python3 __main__.py` fail?* *What is `fast_data_types`?* *Are kittens independent?* Each answer is anchored in one or more experiments. A concluding section synthesizes the findings into a single coherent picture of what kitty actually is at runtime.

---

## 2. Question 1 — Which language does the heavy lifting at runtime?

**Answer: C.** Python orchestrates; Go provides a detachable CLI surface; C does the work.

### 2.1 Line-count census

The first piece of evidence is raw volume. Counting only git-tracked source files (i.e., excluding build artifacts under `build/`, `__pycache__/`, and vendored binaries), the distribution is as follows:

| Language      | Files | Lines   |
|---------------|------:|--------:|
| C (`.c`)      |   128 |  61,806 |
| Headers (`.h`)|    84 |  37,939 |
| **C + H**     | **212** | **99,745** |
| Python (`.py`)|   214 |  62,874 |
| Go (`.go`)    |   258 |  56,071 |
| GLSL (`.glsl`)|    13 |     696 |

Narrowing to the core terminal engine in `kitty/`:

| Language     | Files in `kitty/` | Lines in `kitty/` |
|--------------|------------------:|------------------:|
| C (`.c`)     |                51 |            35,917 |
| Headers (`.h`)|               48 |            24,314 |
| Python (`.py`)|              109 |            39,355 |
| GLSL (`.glsl`)|               13 |               696 |

C + H is ≈99,745 lines, ≈59% more than Python's 62,874 lines. Inside the core `kitty/` package, the C+H weight (60,231 lines) is ≈1.53× the Python weight (39,355 lines). Volume alone does not prove which language is on the hot path, but it establishes that calling this a "Python application" understates the C commitment by more than half.

### 2.2 Subsystem-to-language map

Volume becomes diagnostic when you map each subsystem to its implementation language. The table below records which language implements each performance-sensitive subsystem and the principal source files. Every file and line count was verified with `wc -l` on the HEAD commit.

| Subsystem                       | Language | Principal source file(s)                                                                   | Lines (principal)  |
|---------------------------------|----------|--------------------------------------------------------------------------------------------|--------------------|
| VT escape-sequence parsing      | C        | `kitty/vt-parser.c`                                                                        | 1,596              |
| Terminal screen model           | C        | `kitty/screen.c`, `line.c`, `line-buf.c`, `cursor.c`, `history.c`                          | 4,932 (`screen.c`) |
| GPU rendering / OpenGL glue     | C        | `kitty/gl.c`, `kitty/shaders.c`                                                            | 400, 1,285         |
| Font rasterization              | C        | `kitty/fonts.c`, `kitty/freetype.c`, `kitty/fontconfig.c`, `kitty/glyph-cache.c`           | 1,761, 1,037, 514  |
| Inline graphics (images) protocol | C      | `kitty/graphics.c`, `kitty/png-reader.c`                                                   | 2,431              |
| Main event loop / PTY mux       | C        | `kitty/child-monitor.c`                                                                    | 2,016              |
| Child-process spawn / exec      | C        | `kitty/child.c`                                                                            | —                  |
| Cryptography (remote control)   | C        | `kitty/crypto.c`                                                                           | 468                |
| SIMD string ops (wcswidth etc.) | C        | `kitty/simd-string-128.c`, `simd-string-256.c`, `simd-string.c`                            | —                  |
| Python↔C bridge module def      | C        | `kitty/data-types.c`                                                                       | 612                |
| Application orchestrator        | Python   | `kitty/boss.py`                                                                            | 3,094              |
| Config schema / parsing         | Python   | `kitty/options/*.py`, `kitty/conf/utils.py`, `kitty/config.py`, `kitty/cli.py`             | 459, 1,093         |
| Window layouts (splits, grid…)  | Python   | `kitty/layout/*.py`                                                                        | —                  |
| Tab / window / session mgmt     | Python   | `kitty/tabs.py`, `kitty/window.py`, `kitty/session.py`                                     | —                  |
| Remote-control dispatch         | Python   | `kitty/rc/*.py`, `kitty/remote_control.py`                                                 | —                  |
| Kitten launch framework         | Python   | `kittens/runner.py`, `kittens/tui/*.py`                                                    | —                  |
| Standalone CLI binary (`kitten`)| Go       | `tools/**/*.go`, `kittens/*/**/*.go`                                                       | —                  |
| GPU shader programs             | GLSL     | `kitty/*.glsl` (13 files, see §3)                                                          | 696 total          |

The pattern is unambiguous. Every subsystem that (a) touches bytes coming off a PTY, (b) maintains terminal state, (c) drives font rasterization, or (d) talks to the GPU is implemented in C. Python handles configuration, layout, window/tab bookkeeping, and the kittens framework — the pieces that run at human timescales. Go, notably, is not present anywhere in the above hot path: no Go file in the repository is imported by Python or linked into `fast_data_types`. The Go code compiles into a separate, static `kitten` binary that runs **out of process** from kitty itself.

### 2.3 The main event loop is in C

The single most telling piece of control-flow evidence is that the main event loop — the thread that schedules renders, polls input, and dispatches child I/O events — is written in C, not Python. Python calls into it and then waits.

From `kitty/child-monitor.c`:

```c
// line 55  — the ChildMonitor struct carries three pthread handles
pthread_t io_thread, talk_thread;

// line 256 — spawn the remote-control socket thread
pthread_create(&self->talk_thread, NULL, talk_loop, self);

// line 291 — spawn the PTY I/O thread
pthread_create(&self->io_thread, NULL, io_loop, self);

// line 1259 — main-thread entry, called from Python
main_loop(ChildMonitor *self, PyObject *a UNUSED) {
    ...
    run_main_loop(process_global_state, self);
    ...
}
```

Three threads. Two are created in C (`io_thread`, `talk_thread`). The third is the main thread that Python entered on — which Python relinquishes as soon as it calls `boss.child_monitor.main_loop()`. That call appears at `kitty/main.py:234`. After that line, Python is essentially sleeping; C is running the application.

The net effect: once startup completes, the runtime hot path is entirely inside compiled C. Python is not the performer; it is the stage manager who hands off and waits for the show to end.

### 2.4 Go is an out-of-process companion, not a runtime component

`go.mod` declares `go 1.22`. The Go code compiles to a single static binary named `kitten` (produced by `setup.py`; its built size is ≈16 MB, located at `kitty/launcher/kitten` after a successful build). This binary implements the CLI tool dispatched by `kitty +kitten <name>`, as well as several standalone commands (`icat`, `hold`, `complete`, `shebang`).

The dispatch is visible in `kitty/entry_points.py`. Each Go-backed command is not run *inside* the kitty process; it is `exec`ed (verbatim lines, with their source-file line numbers):

```python
# kitty/entry_points.py (verbatim exec lines from each Go-backed dispatcher)
# icat (line 12)
os.execl(kitten_exe(), "kitten", *args)
# hold (lines 29–30)
args = ['kitten', '__hold_till_enter__'] + args[1:]
os.execvp(kitten_exe(), args)
# complete (lines 42–43)
args = ['kitten', '__complete__'] + args[1:]
os.execvp(kitten_exe(), args)
# shebang (line 115)
os.execvp(kitten_exe(), ['kitten', '__confirm_and_run_shebang__'] + cmd + [script_path])
```

Three of the four branches use `os.execvp` (which searches `PATH`) rather than `os.execl`; only `icat` uses `os.execl` directly. Either way, the Python process replaces its image in memory with the Go `kitten` binary.

Go therefore does *not* share the C terminal core. Its purpose is exactly opposite: to be a single static binary that can be copied to a remote machine (over SSH, for example), where the full kitty + CPython + FreeType + HarfBuzz + libGL stack cannot and should not be shipped. The Go CLI speaks to the running kitty process through terminal escape sequences and the remote-control socket. It never links against `fast_data_types`.

### 2.5 The native C launcher makes the shape explicit

The binary named `kitty` on disk is not `/usr/bin/python3`. It is a native ELF executable compiled from `kitty/launcher/main.c` (466 lines). That launcher **embeds** CPython rather than the other way around:

```c
// kitty/launcher/main.c (excerpts)
#include <Python.h>                             // line 21 — embedded interpreter
set_kitty_run_data(...);                        // line 53 — seeds sys.kitty_run_data
PySys_SetObject("kitty_run_data", ans);         // line 73
status = Py_InitializeFromConfig(&config);      // line 211 — starts the interpreter
```

What this means in practice: in a properly built install, the user does not run `python3 __main__.py`. They run `kitty`, which is a C binary. The C binary starts Python inside itself, loads the `kitty` Python package, and hands control to `kitty/main.py`. Python's role — even at startup — is as a configurator and dispatcher rather than a driver. The fact that `python3 __main__.py` technically works as an alternate entry point (when the build is complete) is a consequence of the package layout; it is not the intended runtime.

### 2.6 Rationale — why the three-language split is the way it is

The split is not accidental; each language is doing what it is best at:

- **C** owns everything that touches bytes, pixels, or kernel primitives. VT parsing is a tight state machine over UTF-8 bytes; the screen model is a large mutable grid; font rasterization calls FreeType/HarfBuzz directly; rendering talks to OpenGL through GLAD. This work runs at character-rate, not user-rate, and cannot tolerate Python's per-operation overhead. Putting all of it in C also keeps the rendering thread free of the GIL.

- **Python** owns everything that runs at user-rate: keybindings, option parsing, tab/window/layout bookkeeping, remote-control dispatch, and the kitten framework. A user pressing a key once per few hundred milliseconds is not a hot path; Python's expressiveness buys rapid iteration on configuration schemas (see the 3,094-line `boss.py` and the `kitty/options/` package) without ceding runtime performance.

- **Go** owns a *separable* surface — the `kitten` CLI — that needs to run without kitty's heavy native stack. A single 16 MB static binary can be `scp`'d to a remote server and executed there; it would be impractical to ship a full CPython + FreeType + HarfBuzz + OpenGL toolchain just to render an image over SSH.

### 2.7 "GPU accelerated" is only half the story

The tagline "GPU accelerated" is true, but it is the *last* stage of the pipeline, not the defining stage. Before a pixel reaches a shader, C has already:

1. Parsed VT escape sequences from the PTY byte stream (`vt-parser.c`),
2. Mutated the terminal screen model (`screen.c`),
3. Looked up the glyphs for every character (`fonts.c`/`freetype.c`),
4. Shaped complex scripts via HarfBuzz,
5. Rasterized glyph bitmaps into a sprite atlas (`glyph-cache.c`),
6. Decoded inline graphics (`graphics.c` / `png-reader.c`),
7. Applied color-profile conversions (`colors.c`),
8. Packed per-cell render state for the GPU.

Only then do shaders draw the result. Python has touched none of the above. The GPU is where the performance *manifests*; C is where the performance *comes from*. That is the architectural reality that the tagline compresses into two words.

---

## 3. Question 2 — What role do GLSL shaders play, and how central are they?

**Answer: shaders are the sole rendering path.** kitty has no CPU rendering backend. If the GPU cannot compile and link the thirteen `.glsl` files, the terminal cannot draw a single character.

### 3.1 Catalog of all 13 shader files

Every `.glsl` file in the `kitty/` package, with its size and role. Sizes were verified with `stat`.

| # | File                        | Bytes | Role                                                                                        |
|--:|-----------------------------|------:|---------------------------------------------------------------------------------------------|
| 1 | `cell_vertex.glsl`          | 8,461 | Vertex shader for the terminal character grid — computes per-cell quad positions & attrs   |
| 2 | `cell_fragment.glsl`        | 8,751 | Fragment shader for the terminal character grid — samples glyph sprite atlas & blends       |
| 3 | `cell_defines.glsl`         |   817 | Shared constants for cell shaders (`PHASE_BOTH`, `PHASE_BACKGROUND`, `PHASE_SPECIAL`, `PHASE_FOREGROUND`, etc.) — included, not compiled directly |
| 4 | `border_vertex.glsl`        | 1,492 | Vertex shader for window / tab-bar borders                                                  |
| 5 | `border_fragment.glsl`      |    79 | Fragment shader for window / tab-bar borders (solid color out)                              |
| 6 | `bgimage_vertex.glsl`       | 1,238 | Vertex shader for the optional `background_image` (from `kitty.conf`)                       |
| 7 | `bgimage_fragment.glsl`     |   352 | Fragment shader for background image — samples texture, applies opacity/premultiplication   |
| 8 | `graphics_vertex.glsl`      |   694 | Vertex shader for inline graphics (kitty graphics protocol: icat images, etc.)              |
| 9 | `graphics_fragment.glsl`    |   582 | Fragment shader for inline graphics — three variants (SIMPLE / PREMULT / ALPHA_MASK) via `#define` |
|10 | `tint_vertex.glsl`          |   346 | Vertex shader for screen-tint overlay (dim-unfocused-window, transparency tints)            |
|11 | `tint_fragment.glsl`        |    82 | Fragment shader for screen tint                                                             |
|12 | `alpha_blend.glsl`          |   977 | Utility library — alpha-blend function; included via `#pragma kitty_include_shader`         |
|13 | `linear2srgb.glsl`          |   398 | Utility library — sRGB ↔ linear color-space conversion; included via `#pragma kitty_include_shader` |

Total: 696 lines of GLSL across all thirteen files — tiny compared to the ≈99,745 C + H lines, but their *centrality* (see §3.6) makes them indispensable.

### 3.2 Six rendering stages, six shader pairs (plus utility libs)

The thirteen files map onto six rendering stages. Each stage is a vertex+fragment pair (sometimes with shared defines). The three "utility" files (`cell_defines.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl`) are *included* into compiled shaders through a preprocessor directive; they are not compiled as standalone programs.

| Stage | Purpose                                      | Vertex shader          | Fragment shader          | Utility include                                             |
|------:|----------------------------------------------|------------------------|--------------------------|-------------------------------------------------------------|
| 1     | Character cell grid (the terminal content)   | `cell_vertex.glsl`     | `cell_fragment.glsl`     | `cell_defines.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl` |
| 2     | Window / tab borders                         | `border_vertex.glsl`   | `border_fragment.glsl`   | —                                                           |
| 3     | Optional background image                    | `bgimage_vertex.glsl`  | `bgimage_fragment.glsl`  | —                                                           |
| 4     | Inline graphics (images & PNGs)              | `graphics_vertex.glsl` | `graphics_fragment.glsl` | —                                                           |
| 5     | Screen tint (dim, translucency)              | `tint_vertex.glsl`     | `tint_fragment.glsl`     | —                                                           |
| 6     | Utility libraries                            | n/a                    | n/a                      | used by stages 1, 3                                         |

Of these six stages, stage 1 (the cell grid) is overwhelmingly the dominant one — it is invoked every frame, for every visible cell. Everything the user *reads* in the terminal is drawn by `cell_vertex.glsl` + `cell_fragment.glsl`.

### 3.3 The loading pipeline — Python reads, C compiles

Shader compilation is split between Python and C, with the GLSL source itself passed as strings across the bridge. The division of labor is: Python reads, preprocesses, and resolves includes; C calls into OpenGL.

**Python side — `kitty/shaders.py`** (204 lines total):

```python
# kitty/shaders.py (excerpts, line numbers verified on HEAD)
from .constants import read_kitty_resource                     # line 9

# _load_sources() — reads the .glsl bytes out of the installed kitty package
def _load_sources(name: str, seen=None, level: int = 0) -> str:
    ...
    src = read_kitty_resource(name).decode('utf-8')             # line 68
    # Resolves #pragma kitty_include_shader <other_name> recursively
    # (see handling around line 53 of shaders.py). Cycles are guarded by the
    # `seen` set passed in.

# Program.compile() — hands the final source strings to the C side
def compile(self, program_id: int, allow_recompile: bool = False) -> None:   # line 87
    ...
    compile_program(                                             # line 90
        program_id, self.vertex_sources, self.fragment_sources, allow_recompile
    )
```

Note: the vertex and fragment sources are not passed to `compile()` as arguments; they are stored on the `Program` instance as `self.vertex_sources` / `self.fragment_sources` by a prior call to `apply_to_sources()` (line 83 of `shaders.py`). The integer `program_id` identifies which C-side program slot to populate. This decoupling lets callers prepare sources once and recompile the same program slot if needed.

`read_kitty_resource()` (defined at `kitty/constants.py:241`) uses `importlib.resources` to read the GLSL files out of the installed kitty package. On Python ≥ 3.10 it uses `importlib.resources.files()`; on older interpreters it falls back to `importlib.resources.read_binary`. Either way, the GLSL source is a *package resource* — it travels with the Python wheel / install tree, not with the C extension.

Before compilation, the Python side prepends a `#version <GLSL_VERSION>` directive. `GLSL_VERSION` is a constant exposed by the C extension (`fast_data_types.GLSL_VERSION` — in this environment it is `140`, i.e. GLSL 1.40 / OpenGL 3.1 baseline). Python also emits `#line <line> <filenumber>` markers so that any subsequent GLSL compile error reports map back to the original filename, not the concatenated blob.

**C side — `kitty/shaders.c`** (1,285 lines total):

```c
// kitty/shaders.c (verbatim excerpts, line numbers verified on HEAD)
static PyObject*
compile_program(PyObject UNUSED *self, PyObject *args) {                                   // line 1168
    ...
#define fail_compile() { glDeleteProgram(program->id); return NULL; }                      // line 1178
    program->id = glCreateProgram();                                                        // line 1179
    if (!attach_shaders(vertex_shaders, program->id, GL_VERTEX_SHADER)) fail_compile();     // line 1180
    if (!attach_shaders(fragment_shaders, program->id, GL_FRAGMENT_SHADER)) fail_compile(); // line 1181
    glLinkProgram(program->id);                                                             // line 1182
    ...
    init_uniforms(which);                                                                   // line 1193
    ...
}
```

The `attach_shaders()` helper (defined at `kitty/shaders.c:1153` with signature `attach_shaders(PyObject *sources, GLuint program_id, GLenum shader_type)`) is called twice, once per stage. Inside, C calls `glCreateShader()`, `glShaderSource()`, and `glCompileShader()` for each source string, then `glAttachShader(program_id, shader_id)`. This is standard OpenGL. There is no "interpret GLSL in Python" path; there is no "render on the CPU if the shader fails to compile" path.

**Pipeline summary (top to bottom is runtime order):**

1. Python: `read_kitty_resource('cell_vertex.glsl')` → `bytes`
2. Python: decode to UTF-8
3. Python: prepend `#version 140\n`
4. Python: scan for `#pragma kitty_include_shader <name>`; inline each referenced utility file (`cell_defines.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl`) recursively, tracking `seen` to prevent cycles
5. Python: emit `#line` directives for error mapping
6. Python: call C `fast_data_types.compile_program(program_id, vertex_sources, fragment_sources, allow_recompile)`
7. C: `glCreateProgram()` → `program->id`
8. C: `glCreateShader(GL_VERTEX_SHADER)` + `glShaderSource(...)` + `glCompileShader(...)`
9. C: (same for fragment)
10. C: `glAttachShader(program->id, vert)` + `glAttachShader(program->id, frag)`
11. C: `glLinkProgram(program->id)`
12. C: `init_uniforms(which)` — look up and cache uniform locations

The final compiled GPU program is stored on the C side (`program->id` is an `int` handed back to Python as an opaque handle). Python can invoke it later through other `fast_data_types` calls (`glUseProgram`, etc.), but it cannot touch the shader binary.

### 3.4 Preprocessor-driven multi-variant compilation

Two shaders are compiled into multiple variants by rewriting `#define`s before handing them to C. This is a subtle but important detail because it shows how Python is used as a *text-transform* layer over GLSL without any runtime dynamic compilation.

**Cell fragment shader — four variants.** `kitty/cell_defines.glsl` exposes four phase constants: `PHASE_BOTH=1`, `PHASE_BACKGROUND=2`, `PHASE_SPECIAL=3`, `PHASE_FOREGROUND=4`. `kitty/shaders.py` has a function `resolve_cell_defines()` (around line 165) that substitutes each value in turn, producing four distinct GPU programs:

- `CELL_PROGRAM` — "both" pass (fast path when background and foreground can be drawn together)
- `CELL_BG_PROGRAM` — background-only pass
- `CELL_SPECIAL_PROGRAM` — the underline/strike/overline special-attrs pass
- `CELL_FG_PROGRAM` — foreground-only pass

From one source file (`cell_fragment.glsl`), the GPU ends up holding four fully compiled pipelines. The choice of which to invoke per frame is a runtime decision in C (`shaders.c`/`screen.c`) based on blending requirements.

**Graphics fragment shader — three variants.** `kitty/shaders.py` also has `resolve_graphics_fragment_defines()` (around line 189) which substitutes `#define ALPHA_TYPE` to produce three programs:

- `SIMPLE` — no alpha channel (opaque images)
- `PREMULT` — premultiplied-alpha images
- `ALPHA_MASK` — alpha-mask-only images (used for subpixel anti-aliased rendering of images)

So the thirteen `.glsl` files compile into more than thirteen OpenGL programs. The cell stage alone accounts for four of them; graphics for three more; borders/bgimage/tint each one. The exact program count depends on configuration, but the important point is that **every program is mandatory** — there is no runtime branch that skips rendering.

### 3.5 Why there is no CPU fallback

A grep for "software rendering" or "CPU renderer" in the repository returns nothing. More importantly, tracing the `draw_cells` and equivalent functions in `screen.c`/`shaders.c` reveals no branch that produces pixel output without OpenGL. The only rendering pipeline is the one that flows through GLSL programs.

This is a design consequence of kitty's performance contract. A CPU fallback would have to duplicate the entire rendering pipeline — per-cell state packing, glyph atlas sampling, alpha blending, subpixel positioning — in scalar C code. It would be perhaps 50× slower (not uncommon for GPU-bound terminals like this) and would change the system's observable behavior (rendering timing, frame pacing, animated cursor smoothness). Rather than maintain two backends, kitty maintains one and requires a GPU with OpenGL 3.3+ core-profile support as a precondition.

The `GLSL_VERSION = 140` constant corresponds to GLSL 1.40, which pairs with OpenGL 3.1. However, kitty's runtime checks (in `kitty/gl.c` and `kitty/glfw.c`) require OpenGL 3.3+ for core-profile features (array textures, etc.) — so an environment that cannot provide this context will refuse to start.

### 3.6 Rationale — the centrality argument

The thirteen GLSL files are tiny (696 lines — roughly 0.3% of the total codebase by volume, using 99,745 + 62,874 + 56,071 + 696 as the denominator). But their role is outsized for three reasons:

1. **They are the only rendering path.** Every glyph, every border, every background, every cursor animation flows through one of the six stages above. There is no alternative pipeline.
2. **They are the observable boundary.** The user can only *see* what GLSL draws. The C parsing, screen modeling, and font rasterization are internal states; the shader output *is* the terminal.
3. **They are compiled once and cached.** On first run, each program is compiled once by `compile_program()`; the handle is kept for the life of the process. Compilation happens before the terminal is interactive, so any shader error is fatal startup failure, not a runtime degradation.

So calling kitty "GPU accelerated" is correct but underspecifies. A more accurate framing: *kitty is a C terminal whose only display stage is GPU shaders.* The CPU-side pipeline feeds the shaders; the shaders have no competitor, no fallback, and no opt-out.

---

## 4. Question 3 — Why does `python3 __main__.py` fail, and what does the failure reveal?

**Answer: because the very first transitive import on the Python side requires a compiled C extension (`kitty.fast_data_types`) which does not exist in a fresh checkout.** The error is immediate, unavoidable, and diagnostic: it shows that Python in this project is not a standalone runtime but a shell draped over a C core.

### 4.1 What `__main__.py` actually does

The file is intentionally trivial — all seven lines of it:

```python
# __main__.py  (7 lines total, verbatim)
#!/usr/bin/env python
# License: GPL v3 Copyright: 2015, Kovid Goyal <kovid at kovidgoyal.net>


if __name__ == '__main__':
    from kitty.entry_points import main
    main()
```

It delegates immediately to `kitty.entry_points.main`, which is the "multiplexed" entry used by the native `kitty` launcher and by `python3 -m kitty` alike. This delegation design lets a single Python package serve four distinct invocation shapes: (1) the native C launcher calling `kitty.main.main()`; (2) `python3 __main__.py`; (3) `python3 -m kitty`; (4) `kitty +kitten <name>` dispatching through `kittens.runner`.

### 4.2 The exact traceback (verbatim)

Running `python3 __main__.py` in an environment where `kitty/fast_data_types.*.so` is missing (reproduced here by temporarily renaming the built `.so`) produces exactly this output:

```text
Traceback (most recent call last):
  File "/.../__main__.py", line 7, in <module>
    main()
  File "/.../kitty/entry_points.py", line 194, in main
    from kitty.main import main as kitty_main
  File "/.../kitty/main.py", line 11, in <module>
    from .borders import load_borders_program
  File "/.../kitty/borders.py", line 7, in <module>
    from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

(The `/.../` prefix is this environment's absolute path; everything else is verbatim. Line numbers and file paths are from the HEAD commit.)

### 4.3 Annotated import chain

Each step of the chain tells us a little more about the architecture:

1. **`__main__.py:7` — `main()` from `kitty.entry_points`**
   This is fine; `kitty.entry_points` is one of the thirteen Python modules that can import standalone (see §5.4). It is deliberately pure-Python and is the *only* part of the Python layer that will run before we learn whether the C extension exists.

2. **`kitty/entry_points.py:194` — `from kitty.main import main as kitty_main`**
   When no CLI subcommand matched (`entry_points` dispatches `icat` via `os.execl(kitten_exe(), ...)` and `hold` / `complete` / `shebang` via `os.execvp(kitten_exe(), ...)` to the Go binary — see §2.4 for the verbatim lines), control falls through to the default branch, which imports the full kitty application at `kitty.main`. This is the first interesting import.

3. **`kitty/main.py:11` — `from .borders import load_borders_program`**
   `main.py` does its own imports at module load time; by line 11 it is already pulling in rendering infrastructure. It does not perform a lazy "only if a window is created" deferral. The reason is that `main.py`'s job is to set up the application singleton (`Boss`), which needs access to border rendering among many other things.

4. **`kitty/borders.py:7` — `from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image`**
   Here the Python layer hits the bridge. `borders.py` is 116 lines of Python that delegates border-drawing math to five C symbols defined inside `fast_data_types`.

5. **`ModuleNotFoundError: No module named 'kitty.fast_data_types'`**
   The compiled extension does not exist. Python has nowhere to go: there is no fallback `kitty/fast_data_types.py`, no conditional import, no graceful degradation.

### 4.4 What `fast_data_types` is — a compiled C extension, not a `.py` file

`kitty.fast_data_types` is a CPython C extension. In an installed kitty, it appears as a single shared object inside the `kitty/` package directory:

```text
kitty/fast_data_types.cpython-312-x86_64-linux-gnu.so
```

(The exact triple depends on Python version and platform.) In this environment, after the setup stage completed successfully, the built artifact is `kitty/fast_data_types.so` at ≈1.2 MB (1,213,072 bytes).

The source of this extension is not one file but **dozens**. `setup.py`'s `find_c_files()` function (around lines 906–929 of `setup.py`) collects every `.c` file under `kitty/` on the Linux target (minus platform-specific exclusions for macOS) and compiles them all into one `.so`. On this Linux build, that list was 49 source files:

```text
charsets.c  child-monitor.c  child.c  cleanup.c  colors.c  crypto.c  cursor.c
data-types.c  desktop.c  disk-cache.c  fast-file-copy.c  font-names.c
fontconfig.c  fonts.c  freetype.c  freetype_render_ui_text.c  gl-wrapper.c
gl.c  glfw-wrapper.c  glfw.c  glyph-cache.c  graphics.c  history.c  hyperlink.c
key_encoding.c  keys.c  kittens.c  line-buf.c  line.c  logging.c  loop-utils.c
monotonic.c  mouse.c  png-reader.c  rowcolumn-diacritics.c  screen.c  shaders.c
shlex.c  simd-string-128.c  simd-string-256.c  simd-string.c  state.c  systemd.c
unicode-data.c  utmp.c  vt-parser.c  vt-parser-dump.c  wcswidth.c  window_logo.c
```

Plus vendored C from `3rdparty/`: `ringbuf.c` and the base64 codec. Plus headers (84 `.h` files across the repository, of which ~50 sit directly in `kitty/`). Plus link-time flags pulled from `pkg-config` for FreeType, HarfBuzz, FontConfig, libpng, zlib, lcms2, OpenGL, libxxhash, and OpenSSL — roughly nine native library dependencies resolved at build time (see §4.7).

### 4.5 What `fast_data_types.pyi` is — a type stub, not an implementation

A companion file `kitty/fast_data_types.pyi` (1,635 lines) exists in the source tree. It is a **type stub** — a PEP 561–style declaration file consumed by static type checkers (mypy, Pyright, pyright-LSP-friendly editors). The `.pyi` file:

- Declares classes (`Screen`, `LineBuf`, `HistoryBuf`, `Cursor`, `ColorProfile`, `ChildMonitor`, etc.).
- Declares function signatures (`compile_program`, `add_borders_rect`, `init_borders_program`, `get_options`, `monotonic`, `glCreateProgram`, hundreds more).
- Declares module-level constants (`GLSL_VERSION`, `BORDERS_PROGRAM`, `CELL_PROGRAM`, `CELL_BG_PROGRAM`, …).

Crucially, **Python at runtime does not look at `.pyi` files**. They have zero effect on import resolution. Their presence or absence changes the behavior of mypy; it does not change the behavior of `import kitty.fast_data_types`. So even though the repository ships a comprehensive type-stub for the C extension, a fresh clone without `setup.py build` has no way to satisfy the import — the stub is a description of the API, not an implementation of it.

### 4.6 The import is unavoidable

Could the failure have been delayed by restructuring the imports? In principle yes — but only by inverting every assumption the codebase is built on:

- `kitty/main.py` would have to lazy-import `.borders` (inside `Boss.create_os_window()` rather than at module top).
- `kitty/boss.py` would have to lazy-import every `fast_data_types`-using module (which is almost all of them).
- `kitty/entry_points.py` would have to guard against the case where `kitty.main` itself fails on import.

Given that 30 of 43 Python modules directly under `kitty/` fail to import without `fast_data_types` (see §5.4), and all of them are reachable from `boss.py`, moving the failure later would just change *where* it happens, not *whether*. The architecture is designed for the build to succeed before any interesting Python runs. Trying to evade that precondition is pointless.

### 4.7 The build also fails without a C toolchain + native libraries

The AAP's test environment illustrated this in a different way. Running `python3 setup.py build` on a stock image without `pkg-config` installed produces (excerpt from the AAP's verified Experiment 6):

```text
File "setup.py", line 1091, in build
    kitty_env(args), 'kitty/fast_data_types', args.compilation_database, sources, headers,
File "setup.py", line 609, in kitty_env
    at_least_version('harfbuzz', 1, 5)
File "setup.py", line 284, in at_least_version
    if subprocess.run([PKGCONFIG, package, f'--atleast-version={q}']
FileNotFoundError: [Errno 2] No such file or directory: 'pkg-config'
```

The relevant call sites are verifiable against the source on this branch:

- `setup.py:72` — `PKGCONFIG = os.environ.get('PKGCONFIG_EXE', 'pkg-config')` — names the helper.
- `setup.py:284` — `subprocess.run([PKGCONFIG, package, f'--atleast-version={q}']...)` — invokes it.
- `setup.py:609` — `at_least_version('harfbuzz', 1, 5)` — the first use, in `kitty_env()`, i.e. while assembling the C compiler environment for `fast_data_types`.
- `setup.py:1091` — the `build()` entry point that calls `kitty_env()` before `compile_c_extension('kitty/fast_data_types', ...)`.

So even before the C compiler is invoked, the build refuses to begin because it cannot discover the headers and link flags for the native libraries it must link against (FreeType, HarfBuzz, FontConfig, libpng, zlib, lcms2, OpenGL, libxxhash, OpenSSL). *The failure mode is telling: it is not a missing shared library at import time; it is a missing toolchain prerequisite at build time.* In this investigation's environment, `pkg-config` was present and the build succeeded; in the AAP's reference environment it was not, and the build failed exactly where the message above shows. Either way, the architectural claim is the same: the C extension is a build artifact, and the conditions for its existence are nontrivial.

### 4.8 Rationale — what the failure reveals

A reader who knows only that kitty "has some C extensions" might expect the failure to manifest late — say, when a window is first created or when text is first rendered. What actually happens is that the failure manifests on line 7 of `borders.py`, which is hit on line 11 of `main.py`, which is hit on line 194 of `entry_points.py`, which is hit on line 7 of `__main__.py`. In other words: **before kitty has made a single decision about what to do, it has to have the C extension.** The Python layer is so thoroughly threaded through with native calls that it cannot meaningfully run at all without them.

The architectural meaning is: Python in this project is not a language that uses C for speed. It is a configuration and dispatch layer *on top of* a C application. Removing the C layer does not leave you with a reduced-functionality kitty; it leaves you with 13 pure-Python utility modules that have nothing to drive. The entry point failure is not a bug; it is the correct, load-time assertion that the C substrate must be present.

---

## 5. Question 4 — The `fast_data_types` bridge: what makes it the one critical piece

**Answer: `fast_data_types` is a single compiled CPython extension module that (a) is assembled from ~49 C source files on Linux, (b) registers more than 25 distinct subsystem initializers in one `PyInit_*` function, and (c) is imported — directly or transitively — by roughly 70% of the Python modules in the project.** It is not merely a module; it is the seam along which Python is stitched to C, and the shape of that seam is the shape of the application.

### 5.1 The module definition

At the bottom of `kitty/data-types.c` (612 lines total), standard CPython extension boilerplate declares the module:

```c
// kitty/data-types.c
static struct PyModuleDef module = {
    .m_base = PyModuleDef_HEAD_INIT,
    .m_name = "fast_data_types",
    ...
};

PyMODINIT_FUNC
PyInit_fast_data_types(void) {
    PyObject *m = PyModule_Create(&module);
    ...
    // A long sequence of init_* calls follows — see §5.2
    ...
    return m;
}
```

This is a single module. `PyInit_fast_data_types` is CPython's entry point for the module initialization contract: when Python sees `import kitty.fast_data_types`, it calls this function exactly once, and whatever objects have been attached to `m` by the time it returns are what the module exposes. There is only one such function in the entire codebase; there is no split extension architecture where screen lives in `_screen` and shaders in `_shaders`. Everything goes through one `.so`.

### 5.2 Thirty subsystem initializers on Linux (29 on macOS; 25+ unconditional cross-platform)

Inside `PyInit_fast_data_types`, a sequence of `init_<Subsystem>(m)` calls registers each subsystem's types, functions, and constants onto the module object. Transcribed directly from `kitty/data-types.c` (lines 524–612), the Linux initialization sequence is:

```c
// kitty/data-types.c (excerpt of PyInit_fast_data_types, Linux branch)
init_monotonic();                          //  1 — unconditional, initializes time base
init_logging(m);                           //  2 — Python-visible logging hooks
init_LineBuf(m);                           //  3 — terminal line buffer type
init_HistoryBuf(m);                        //  4 — scrollback ring buffer type
init_Line(m);                              //  5 — individual screen line type
init_Cursor(m);                            //  6 — cursor type
init_Shlex(m);                             //  7 — shell-like tokenizer
init_Parser(m);                            //  8 — VT escape-sequence parser type
init_DiskCache(m);                         //  9 — disk-backed cache for graphics/images
init_child_monitor(m);                     // 10 — three-thread event-loop scheduler
init_ColorProfile(m);                      // 11 — per-window color profiles
init_Screen(m);                            // 12 — the 4,932-line screen model
init_glfw(m);                              // 13 — windowing (GLFW bindings)
init_child(m);                             // 14 — child-process spawner / PTY
init_state(m);                             // 15 — global application state
init_keys(m);                              // 16 — key-event encoding / decoding
init_graphics(m);                          // 17 — inline graphics protocol
init_shaders(m);                           // 18 — OpenGL shader program registry
init_mouse(m);                             // 19 — mouse-event decoding
init_kittens(m);                           // 20 — kitten-launch C hooks
init_png_reader(m);                        // 21 — PNG decoder wrapper
init_freetype_library(m);                  // 22 — FreeType init (Linux only)
init_fontconfig_library(m);                // 23 — FontConfig init (Linux only)
init_desktop(m);                           // 24 — desktop notifications (Linux)
init_freetype_render_ui_text(m);           // 25 — FreeType UI-text renderer (Linux)
init_fonts(m);                             // 26 — font caching and shaping
init_utmp(m);                              // 27 — utmp record management
init_loop_utils(m);                        // 28 — main-loop helpers
init_crypto_library(m);                    // 29 — X25519 + AES-GCM + HKDF crypto
init_systemd_module(m);                    // 30 — systemd socket integration (compiled on all platforms; only functional on Linux)
```

Structurally, the calls fall into three zones inside `PyInit_fast_data_types`: (a) twenty-one unconditional calls at the top (items 1–21, from `init_monotonic` through `init_png_reader`); (b) a platform block — on Linux (`#else` branch), items 22–25 (`init_freetype_library`, `init_fontconfig_library`, `init_desktop`, `init_freetype_render_ui_text`); on macOS (`#ifdef __APPLE__` branch), three calls instead (`init_macos_process_info`, `init_CoreText`, `init_cocoa`); (c) five more unconditional calls after `#endif` (items 26–30: `init_fonts`, `init_utmp`, `init_loop_utils`, `init_crypto_library`, `init_systemd_module`). Note that `init_systemd_module` is called on every platform but its body uses `dlopen`/`dlsym` to find libsystemd at runtime, so on macOS it gracefully no-ops. The net count is **thirty** on Linux and **twenty-nine** on macOS.

### 5.3 What each category provides

Grouped by responsibility, these 30 initializers (Linux; 29 on macOS) cover every system-level concern a terminal has:

- **Terminal state machine and grid** — `init_Parser`, `init_Screen`, `init_LineBuf`, `init_HistoryBuf`, `init_Line`, `init_Cursor`, `init_ColorProfile`. Turns PTY byte streams into a structured, drawable model.
- **GPU rendering** — `init_glfw`, `init_shaders`, `init_graphics`. Windowing, shader compilation, and the inline-image pipeline.
- **Font pipeline** — `init_fonts`, `init_freetype_library`, `init_fontconfig_library`, `init_freetype_render_ui_text` on Linux; `init_CoreText` on macOS. Font discovery, rasterization, shaping.
- **Process management** — `init_child`, `init_child_monitor`. Spawning the shell, managing PTY, running the three-thread event loop.
- **Input** — `init_keys`, `init_mouse`. Parsing and encoding of keyboard/mouse events in kitty's Kovid-variant protocol.
- **Kittens support** — `init_kittens`. C-side hooks that the kittens framework uses to push data back into the terminal.
- **System integration / desktop** — `init_desktop`, `init_systemd_module`, `init_utmp`. Linux desktop notifications, systemd socket activation, utmp records.
- **Utilities** — `init_monotonic`, `init_logging`, `init_Shlex`, `init_DiskCache`, `init_state`, `init_loop_utils`, `init_crypto_library`, `init_png_reader`.

Note that the list includes `init_monotonic` (a clock) — the *very first line* of the module body. That same `monotonic()` is what `kittens/tui/handler.py:10` imports (`from kitty.fast_data_types import monotonic`), and that import is why the `ask` kitten cannot be imported standalone (§6.2). A bridge that includes the system clock is a bridge that is extremely hard *not* to depend on.

### 5.4 Import-dependency penetration — 47 files inside `kitty/`, 13 inside `kittens/`

A grep over the source tree gives a quantitative view of how deep the dependency goes:

- **47 `.py` files in `kitty/`** reference `fast_data_types` (either by `from .fast_data_types import ...` or `from kitty.fast_data_types import ...`).
- **13 `.py` files in `kittens/`** reference `fast_data_types`, with eight of those concentrated in `kittens/tui/` (the TUI foundation layer shared by many kittens: `handler.py`, `images.py`, `line_edit.py`, `loop.py`, `operations.py`, `path_completer.py`, `spinners.py`, `utils.py`).

Going further, I ran an isolation test: attempting `importlib.import_module()` on every Python module directly under `kitty/` with `fast_data_types.so` renamed out of the way. The results are sharp.

**The 13 modules that import standalone (without the C extension):**

| Module                      | What it does                                                                |
|-----------------------------|-----------------------------------------------------------------------------|
| `kitty.choose_entry`        | Small helper for selecting a kitten entry point by name                      |
| `kitty.cli_stub`            | Type-stub-only CLI description (no runtime logic)                            |
| `kitty.client`              | Remote-control transport shim (raw socket send/recv; no terminal state)      |
| `kitty.constants`           | Version strings, path helpers (`kitty_exe`, `kitten_exe`, `read_kitty_resource`) |
| `kitty.entry_points`        | Top-level CLI dispatch (the `main()` that `__main__.py` calls)               |
| `kitty.guess_mime_type`     | MIME-type guessing for paste/clipboard/drag data                             |
| `kitty.key_names`           | Symbolic keyname → code tables                                               |
| `kitty.multiprocessing`     | `multiprocessing` wrapper (deliberately a pure-Python shim)                  |
| `kitty.search_query_parser` | Grammar for the scrollback search DSL                                        |
| `kitty.short_uuid`          | UUID shortening helper                                                       |
| `kitty.types`               | Enums and dataclasses with no C dependency                                   |
| `kitty.typing`              | PEP 484 type aliases (imported by TypeCheckers / runtime-only `TypedDict`)   |
| `kitty.window_list`         | Pure-Python window-list data structure (no rendering)                        |

**The 30 modules that fail** (29 with `ModuleNotFoundError`, 1 with `ImportError`):

| Module                       | Module                      | Module                   |
|------------------------------|-----------------------------|--------------------------|
| `kitty.actions`              | `kitty.bash`                | `kitty.borders`          |
| `kitty.boss`                 | `kitty.child`               | `kitty.cli`              |
| `kitty.clipboard`            | `kitty.config`              | `kitty.debug_config`     |
| `kitty.file_transmission`    | `kitty.key_encoding`*       | `kitty.keys`             |
| `kitty.launch`               | `kitty.main`                | `kitty.marks`            |
| `kitty.notify`               | `kitty.open_actions`        | `kitty.os_window_size`   |
| `kitty.remote_control`       | `kitty.rgb`                 | `kitty.session`          |
| `kitty.shaders`              | `kitty.shell_integration`   | `kitty.shm`              |
| `kitty.tab_bar`              | `kitty.tabs`                | `kitty.terminfo`         |
| `kitty.update_check`         | `kitty.utils`               | `kitty.window`           |

\* `kitty.key_encoding` fails with `ImportError: cannot import name 'fast_data_types' from 'kitty'` (an alternate-path import that degrades to the same root cause).

**Arithmetic:** 13 succeed, 30 fail — **30 / 43 ≈ 69.8%**, which rounds to the "~70%" figure used throughout this document.

### 5.5 What the split tells us

Looking at the two lists, the pattern is stark: **every module that does terminal work, rendering, configuration parsing, window/tab bookkeeping, child-process management, keyboard/mouse handling, or remote control fails.** The 13 survivors are all either (a) data-type helpers, (b) pure-Python utilities, or (c) dispatch shims that haven't yet reached into the "real" Python layer.

There is no ambiguous middle ground. There is no "this module works at 80%" case. A Python module either uses `fast_data_types` or it doesn't, and if it doesn't, it's helping with something peripheral.

### 5.6 Why collapse 25+ subsystems into one `.so`?

A reasonable design question is: why does kitty ship one enormous extension module rather than, say, `kitty._screen`, `kitty._shaders`, `kitty._fonts`, and so on — each a separate `.so`? There are several practical reasons:

1. **Initialization ordering.** `init_shaders` assumes `init_glfw` has been called (GLFW provides the OpenGL context). `init_fonts` assumes `init_freetype_library` is up. `init_Screen` assumes `init_ColorProfile`, `init_LineBuf`, `init_HistoryBuf`. Encoding these orderings as "import order" of multiple extensions is possible but brittle; hand-writing one sequence in `PyInit_fast_data_types` is direct and auditable.

2. **Shared symbols and build units.** The C files share headers, static globals, SIMD routines, and inline helpers. Splitting them into separate `.so`s would require exporting those helpers across shared-library boundaries, which complicates the build and runtime linker behavior. Keeping everything in one `.so` means the 49 `.c` files link into one executable object file and share internal symbols freely.

3. **Single-load cost.** Loading a shared object costs a `dlopen()`, a symbol-resolution pass, and (on cold start) disk I/O. One `.so` is one such cost; N are N.

4. **Single Python↔C ABI surface.** Code review and stability are easier when there is one `fast_data_types.pyi` file documenting the whole boundary than if there were a dozen type stubs to keep in sync.

The cost is exactly what we observed in §4: you cannot touch any interesting part of kitty from Python without paying the whole-extension build cost first. This is deliberate.

### 5.7 Rationale — why "one critical piece"

"Critical" here has a precise meaning. It is not just that `fast_data_types` is *important*; importantly, it is that **the application does not define a reduced-functionality path that bypasses it**. Compare this to, say, numpy: an application that imports numpy and fails gets a clear "numpy not installed" message but *may* be able to recover by falling back to Python's stdlib `statistics`. An application that imports `kitty.fast_data_types` and fails cannot recover — there is nothing for it to fall back to.

So `fast_data_types` is not a *dependency* of kitty's Python layer in the same sense that `numpy` is a dependency of a data pipeline. It is the *substrate*. Everything else in the Python layer is a client of that substrate. The 25+ subsystem initializers, the 49 source files, the nine external native libraries, the 1.2 MB compiled `.so` — all of that is the substrate.

Writing it down one more way: kitty is not "a Python program that uses a C module called `fast_data_types`". kitty is "a C application with a Python shell that talks to it exclusively through `fast_data_types`". The distinction is what makes this module *the* critical piece rather than *a* critical piece.

---

## 6. Question 5 — Are kittens independent?

**Answer: No.** "Kittens" are *not* a set of self-contained mini-applications that happen to ship with kitty. They are extension points *of* kitty that look like standalone tools. Most kitten `main.py` files cannot be imported without `fast_data_types`; those that can are thin Python stubs whose `main()` immediately raises `SystemExit('Must be run as kitten ...')` and defers the real work to the static Go `kitten` binary.

This question is the easiest to mis-answer by reading source alone: the directory layout (`kittens/<name>/main.py`, `kittens/<name>/*.go`) invites the intuition that each subdirectory is self-contained. The actual behavior, observable by running `python -c 'import kittens.<name>.main'` for each one, contradicts that intuition.

### 6.1 Per-kitten isolation test results

With `fast_data_types.so` renamed out of the way, I attempted to import `main` from each of the 18 kittens that have a `main.py`.

**6 import successfully:**

| Kitten             | Why import succeeds                                                          |
|--------------------|------------------------------------------------------------------------------|
| `choose_fonts`     | Python stub only — the kitten's logic is in 11 Go files under `kittens/choose_fonts/*.go` (`backend.go`, `face.go`, `faces.go`, `family_list.go`, `final.go`, `graphics.go`, `list.go`, `main.go`, `styles.go`, `types.go`, `ui.go`). The Python `main()` raises `SystemExit`. |
| `clipboard`        | Python stub only — logic is in `kittens/clipboard/{legacy,main,read,write}.go` (4 Go files). |
| `hyperlinked_grep` | Python stub only — wraps Go `kittens/hyperlinked_grep/main.go`. There is no `main()` function; a module-level `if __name__ == '__main__': raise SystemExit('This should be run as kitten hyperlinked_grep')` guard fires only when the file is run as a script. Importing it (as this test does) leaves `__name__ == 'kittens.hyperlinked_grep.main'` and the guard does not trigger. |
| `icat`             | Python stub only — image-rendering kitten. Logic in 6 Go files (`detect.go`, `magick.go`, `main.go`, `native.go`, `process_images.go`, `transmit.go`). |
| `show_key`         | Python stub only — wraps 3 Go files under `kittens/show_key/*.go`.          |
| `transfer`         | Python stub only — file transfer over SSH; logic in 7 Go files (5 implementation files: `ftc.go`, `main.go`, `receive.go`, `send.go`, `utils.go`; plus 2 Go test files: `ftc_test.go`, `send_test.go`). |

**12 fail with `ModuleNotFoundError` for `kitty.fast_data_types`** (transitively):

| Kitten            | Kitten              | Kitten              |
|-------------------|---------------------|---------------------|
| `ask`             | `broadcast`         | `diff`              |
| `hints`           | `pager`             | `panel`             |
| `query_terminal`  | `remote_file`       | `resize_window`    |
| `ssh`             | `themes`            | `unicode_input`     |

### 6.2 Two concrete transitive traces

Neither `ask/main.py` nor `diff/main.py` imports `fast_data_types` directly. The coupling is transitive — a module it imports imports a module that imports `fast_data_types`. Two traces make the mechanism concrete.

**Trace 1 — `ask` kitten → TUI handler → `monotonic`:**

```text
kittens/ask/main.py:12      from ..tui.handler import result_handler
        ↓
kittens/tui/handler.py:10   from kitty.fast_data_types import monotonic
        ↓
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

`ask` is a simple TUI prompt ("press y/n"), written almost entirely in its own Go files. Its Python side is a thin launcher that uses the kittens TUI framework (`kittens.tui.handler.result_handler`) to wire the kitten's output back into the kitty window. The TUI framework, however, needs a monotonic clock for its event loop — which it imports from `fast_data_types`. One symbol imported (`monotonic`), and the entire bridge must load.

**Trace 2 — `diff` kitten → `kitty.cli` → config utilities → `Color`:**

```text
kittens/diff/main.py:8      from kitty.cli import CONFIG_HELP, CompletionSpec
        ↓
kitty/cli.py:13             from .conf.utils import resolve_config
        ↓
kitty/conf/utils.py:27      from ..fast_data_types import Color
        ↓
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

`diff` has a Go implementation; its Python file exists mainly to define option schemas and help text. Those schemas are expressed using `kitty.cli`'s `CompletionSpec` and `CONFIG_HELP`. `kitty.cli` pulls in configuration parsing (`kitty.conf.utils`), which reaches for the `Color` type defined in the C extension — because `Color` is a struct with tight packing conventions that the shader code relies on.

This pattern — "the Python side only declares the schema, but the schema types live behind the bridge" — repeats for most of the 12 failing kittens.

### 6.3 Three transitive paths account for most of the coupling

Across the 13 `.py` files in `kittens/` that reference `fast_data_types`, the coupling reaches the rest of the kitten set through three pathways:

1. **Via `kittens.tui.handler`** (imports `fast_data_types.monotonic`). Kittens that use the TUI framework — `ask`, `broadcast`, `hints`, `panel`, `query_terminal`, `unicode_input`, and more — all traverse this path. `kittens/tui/` has eight files that directly import `fast_data_types`: `handler.py`, `images.py`, `line_edit.py`, `loop.py`, `operations.py`, `path_completer.py`, `spinners.py`, `utils.py`.

2. **Via `kitty.cli` → `kitty.conf.utils`** (imports `fast_data_types.Color`, plus configuration primitives). Kittens that declare CLI options or config schemas — `diff`, `themes`, `ssh` — go through this path.

3. **Direct imports** — a smaller set of kittens *directly* import `fast_data_types`: `hints` (uses `fast_data_types.set_clipboard_string` and similar), `panel` (uses windowing hooks), `query_terminal` (uses many symbols to report terminal state). These are the kittens that *do* execute non-trivial logic in Python rather than deferring to Go.

Because these three paths intersect with essentially every non-trivial kitten, a kitten that does anything beyond "dispatch to the Go binary" ends up at the bridge.

### 6.4 The "Python stub for a Go binary" pattern

The six kittens that *do* import cleanly share a specific shape. `kittens/hyperlinked_grep/main.py` is the canonical minimal example — here is the complete file verbatim (10 lines):

```python
# kittens/hyperlinked_grep/main.py (verbatim, all 10 lines)
#!/usr/bin/env python
# License: GPLv3 Copyright: 2020, Kovid Goyal <kovid at kovidgoyal.net>

import sys

if __name__ == '__main__':
    raise SystemExit('This should be run as kitten hyperlinked_grep')
elif __name__ == '__wrapper_of__':
    cd = sys.cli_docs  # type: ignore
    cd['wrapper_of'] = 'rg'
```

Notice that there is no `main()` function, no `handle_result()` function — just a pair of module-level `if __name__` branches. The file exists for two reasons:

1. **Option / help metadata and the `__wrapper_of__` declaration.** `kittens/runner.py` (the framework) executes each stub with `__name__` set to one of several sentinel values (`'__wrapper_of__'`, `'__completion__'`, `'__conf_name__'`, etc.) to harvest metadata. For `hyperlinked_grep`, the `__wrapper_of__` branch declares that it wraps the `rg` (ripgrep) binary; the runner uses this to set up completions. Running the file as a script (`python3 kittens/hyperlinked_grep/main.py`) hits the `__main__` branch and exits.

2. **Go↔Python result handler (in richer stubs).** Some kittens — such as `icat`, `clipboard`, `transfer` — additionally define top-level functions that the framework may call after the Go binary finishes (e.g., to push the Go binary's output into the kitty window). `hyperlinked_grep` is the minimal shape; richer stubs add more top-level definitions without changing the overall "it is metadata, not logic" character.

Critically, **these Python stubs do not *run* the kitten's logic.** The `__main__` guard exists to produce a clear error if a user mistakenly invokes the file directly. The actual work is done by the Go `kitten` binary, which is launched by the `kitty +kitten <name>` dispatch (§6.5). The "kittens can be imported cleanly" result in this isolation test does not mean these kittens are runnable as Python programs; it means their Python side is so minimal that it can be *imported* without `fast_data_types` — but *running* them still requires the Go binary, which still lives inside a properly built kitty install.

### 6.5 The Go binary architecture

For a deeper look at what "the Go binary" means here: `setup.py` also builds a separate native binary called `kitten` (note: singular), at `kitty/launcher/kitten`, by running `go build` against 258 `.go` files spread across `tools/` and `kittens/*/`. Of the 18 kittens with a `main.py`, 14 have matching `.go` files that get compiled into this binary. The binary is statically linked and is ≈16 MB.

The `kitty` process and the `kitten` binary communicate either (a) via terminal escape sequences and the kitty graphics protocol, or (b) via the remote-control socket. A command like `kitty +kitten icat my-image.png` is routed by `kitty/entry_points.py` — the `icat` function is defined at lines 10–12:

```python
# kitty/entry_points.py (icat branch, verbatim lines 10–12)
def icat(args: List[str]) -> None:
    from kitty.constants import kitten_exe
    os.execl(kitten_exe(), "kitten", *args)
```

Similar `exec*` calls exist for `hold`, `complete`, and `shebang` (as §2.4 shows verbatim, those three branches use `os.execvp` rather than `os.execl`). In every case the Python process replaces its own image in memory with the Go binary. After the `exec*` call, the Python interpreter is gone; the Go binary runs; it speaks to the still-running kitty terminal through the TTY. No shared memory, no shared libraries, no shared address space.

### 6.6 Why the naming is misleading

"Kitten" suggests a small, independent creature — a helpful tool that could, in principle, live on its own. That is not what the architecture implements. The kittens directory is the plugin folder of a larger terminal program:

- Kittens that stay in Python depend on the TUI framework and/or config infrastructure, both of which sit on the native bridge.
- Kittens that move to Go are architecturally *more* independent (they compile into a standalone binary) but are still launched by kitty (`kitty +kitten <name>` via `os.execl` for `icat` or `os.execvp` for the other three dispatch targets — see §2.4 and §6.5) and still communicate back through kitty-specific escape sequences.
- Either way, a kitten's natural habitat is inside a running kitty session. The `kittens.runner.run_kitten` framework (202 lines in `kittens/runner.py`) is the glue that makes them addressable.

### 6.7 Rationale — what the empirical test revealed

If you only read the source, you might guess that `kittens/hyperlinked_grep/` is a small, self-contained grep-wrapper. Running the import shows that:

- Its Python side imports cleanly.
- But the module raises `SystemExit('This should be run as kitten hyperlinked_grep')` if run as a script — the guard `if __name__ == '__main__':` at module scope triggers this (there is no `main()` function).
- The actual implementation is in `kittens/hyperlinked_grep/main.go`.
- That Go file gets compiled into the shared `kitten` binary, not into a per-kitten binary.

So "independent" does not describe this kitten. It has neither its own Python runtime (it is a stub) nor its own Go binary (it is a subcommand of the combined `kitten` binary). Its independence is typographical: a folder in the source tree.

Meanwhile, `kittens/ask/main.py` — which at first glance looks like a self-contained "ask the user a question" script — fails to import on a fresh clone because the TUI framework that ties it back into kitty's window is bound to the same native bridge as everything else. This is not a minor dependency; it is a structural one.

The general principle: **in kitty, "kitten" is not a synonym for "mini-application". It is a synonym for "extension point".** Extension points by definition live inside the host application. The kitten framework is the pattern kitty uses to let users (and kitty's own developers) add new subcommands without growing the main `boss.py` orchestrator. It is explicitly designed *not* to be standalone.

---

## 7. Conclusion — Synthesizing the architecture

Five questions asked; five answers given. Together they tell a coherent story about what kitty *is*, which is different from what its directory layout and README suggest.

### 7.1 The shape of the application

Putting the findings side by side:

1. **C does the work.** The line-count census (≈99,745 C+H lines to ≈62,874 Python), the subsystem-to-language map, and the fact that the main event loop is three pthreads implemented in `child-monitor.c` all point to the same conclusion: the runtime hot path is entirely C.

2. **GLSL is the display boundary.** Thirteen shader files totaling 696 lines are not a side project or an option — they are the only rendering pipeline. If the GPU cannot compile them, kitty cannot draw.

3. **Python cannot run without C.** The entry-point failure at `borders.py:7` is immediate and unavoidable. Thirty of 43 directly importable Python modules in `kitty/` fail without `fast_data_types`; the 13 that succeed are utility helpers without meaningful application logic.

4. **The bridge is one compiled extension with 25+ subsystems.** `fast_data_types` is not a facade over many C extensions; it *is* the only C extension, built from 49 source files on Linux, and it collapses everything from the monotonic clock to the OpenGL shader registry into one module initializer.

5. **Kittens are extension points, not independent tools.** 12 of 18 kittens cannot be imported without the bridge; the remaining 6 are thin stubs that `SystemExit` when called and delegate to a Go binary that still runs inside a kitty session.

If I had to reduce all of this to one sentence: **kitty is a C terminal with a Python configuration shell and a Go CLI appendage, all three of which converge on a single compiled extension module that is the application's actual substrate.** Every finding above is a consequence of that shape.

### 7.2 What the empirical method revealed that source-only reading would have missed

A purely static review of this repository — reading `README.md`, browsing `kitty/` in a file tree viewer, opening a few `.py` files — would give a reasonably close picture of the design intent. But several findings in this document *cannot* be deduced from source reading alone:

- **The precise 13-vs-30 module split in `kitty/`.** Static analysis of imports would tell you which modules reach `fast_data_types`, but telling the difference between "reachable but lazy" and "reachable at import time" requires actually importing each module with the extension missing. The result (30 fail at import time, not one later when some function is called) is what proves the coupling is *structural* rather than merely *referential*.

- **The exact traceback line numbers.** The finding that `kitty/main.py` hits `borders.py` on its *11th line*, which reaches `fast_data_types` on its *7th line*, is only visible in a running traceback. Reading the source would have told you `borders.py` imports `fast_data_types`; it would not have told you the call is reached this early in startup.

- **The transitive dependency shape of kittens.** Reading `kittens/ask/main.py` does not reveal that it depends on `fast_data_types.monotonic`. The dependency only appears as a runtime failure pointing at `kittens/tui/handler.py:10`. Likewise for `diff` → `cli` → `conf/utils` → `Color`.

- **The behavior of the "stub" kittens.** Running `python3 kittens/hyperlinked_grep/main.py` reveals that it exits immediately with `SystemExit('This should be run as kitten hyperlinked_grep')` — triggered by a module-level `if __name__ == '__main__':` guard, not a `main()` function (the file has none). Static import analysis would report the module imports cleanly and might suggest it contains real logic; reading the 10-line file or running it shows that its Python side is metadata-only.

- **The pkg-config build failure mode.** This is environmental rather than purely code-level. Reading `setup.py` would show that `pkg-config` is called; running it on a system without that tool shows *where* and *how loudly* the build refuses. This matters because it tells you the build has strong native-toolchain preconditions, not just Python-package preconditions.

The general lesson is that static source reading is good at answering "what *could* happen". Running the code is necessary to answer "what *does* happen, in what order, at what failure mode". For a program like kitty — whose architecture is entirely about where responsibility is drawn between languages at runtime — the "does" question is the one that matters.

### 7.3 "GPU accelerated terminal" — revisited

With all the evidence in hand, the marketing tagline "fast, feature-rich, GPU-based terminal emulator" reads differently than it did at the start of this investigation. It is true, but in a counter-intuitive way:

- The terminal is fast because C is doing the work — not because of Python.
- The terminal is GPU-based because GLSL shaders are the only rendering path — not because the GPU is an "accelerator" for a CPU pipeline that already existed.
- The feature-richness comes from a Python layer — not from the C core — but that Python layer exists only as a client of the C core.

Another way to phrase this: the tagline compresses two stages of the pipeline (CPU text processing in C, GPU display in GLSL) into the single word "GPU". The CPU side is larger, older, and does more than the GPU side; the GPU side is where the user sees the result. Both are in compiled languages; neither is in Python.

### 7.4 Closing observation

Counting files and running imports is an unglamorous way to reverse-engineer an architecture. But the alternative — inferring architecture from self-description — tends to produce accounts that are close but wrong in small, cumulative ways. For kitty specifically, I believe the small, cumulative way in which most mental models are wrong is this: *they overestimate Python's role*. The `.py` files are numerous, readable, and at the top of import chains — so they feel like the center of the program. They are not. They are a configuration surface draped over a C application whose existence they cannot function without. The GLSL shaders are the output stage of that C application. Go is a cooperating out-of-process CLI. The whole thing coheres only because every Python module knows where the bridge is and does not try to live without it.

That the bridge is named `fast_data_types`, and that importing it is the first interesting thing any real kitty Python module does, is not an accident of nomenclature. It is the accurate name for the thing that makes the rest of the architecture possible.

---

## Appendix A — Commands used for verification

All numerical and structural claims in this document were verified by running the commands below against the HEAD commit on a fresh build. For reproducibility, each command is paired with what it verifies.

```bash
# Section 2.1 — line-count census (source-only, excludes build artifacts)
git ls-files '*.c'    | xargs wc -l | tail -1
git ls-files '*.h'    | xargs wc -l | tail -1
git ls-files '*.py'   | xargs wc -l | tail -1
git ls-files '*.go'   | xargs wc -l | tail -1
git ls-files '*.glsl' | xargs wc -l | tail -1

# Section 2.2 — key file sizes
wc -l kitty/screen.c kitty/boss.py kitty/graphics.c setup.py \
      kitty/child-monitor.c kitty/fonts.c kitty/vt-parser.c \
      kitty/shaders.c kitty/cli.py kitty/freetype.c kitty/line.c \
      kitty/main.py kitty/fontconfig.c kitty/crypto.c \
      kitty/launcher/main.c kitty/conf/utils.py kitty/gl.c \
      kitty/cursor.c kitty/constants.py kitty/shaders.py \
      kitty/borders.py __main__.py kitty/fast_data_types.pyi

# Section 3.1 — GLSL catalog (sizes)
for f in kitty/*.glsl; do echo "$(stat -c '%s' "$f") $f"; done

# Section 4.2 — reproduce entry-point traceback
mv kitty/fast_data_types*.so /tmp/  &&  python3 __main__.py  2>&1
mv /tmp/fast_data_types*.so kitty/

# Section 5.4 — module isolation test
python3 - <<'PY'
import importlib, os, sys
sys.path.insert(0, '.')
mods = sorted(os.listdir('kitty'))
mods = [m[:-3] for m in mods if m.endswith('.py') and m != '__init__.py']
ok, fail = [], []
for m in mods:
    try:
        importlib.import_module(f'kitty.{m}')
        ok.append(m)
    except Exception as e:
        fail.append((m, type(e).__name__, str(e)))
print(f'{len(ok)} ok, {len(fail)} fail')
for t in fail: print(t)
PY

# Section 6.1 — per-kitten isolation test (same pattern)
# iterate over kittens/<name>/main.py and attempt importlib.import_module(f'kittens.{name}.main')
```

The exact file paths referenced throughout (e.g. `kitty/data-types.c:538` for `init_monotonic()`; `kitty/shaders.c:1179` for `glCreateProgram()`) correspond to the HEAD commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. A different commit will have different line numbers.
