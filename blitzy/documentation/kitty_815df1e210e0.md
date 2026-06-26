# kitty — Architecture Q&A (commit `815df1e210e0`)

This document answers four onboarding questions about the [kitty](https://sw.kovidgoyal.net/kitty/)
terminal emulator by **exercising the code and reading the source**, never by assuming how the
architecture is "meant" to work. Every factual claim resolves to a specific `path:line` citation at
commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`, kitty `0.35.2`), and
the behavioral claims are backed by the exact command output captured below.

The four questions:

- **O1** — kitty calls itself "GPU accelerated", yet the tree is a tight weave of Python and C.
  Which language actually carries the runtime workload, and where does the performance come from?
- **O2** — What do the scattered `*.glsl` files do, and how central are they?
- **O3** — Running the main entry point directly fails almost immediately with a cryptic error.
  What is the root cause, and what is the single critical piece everything depends on?
- **O4** — Are the small tools under `kittens/` genuinely independent, or do they quietly depend on
  the same native bridge?

---

## Build & observation methodology

Two build/run venues were available and both were exercised; this note states which venue produced
which observation so that no figure is misattributed. The canonical, user-provided `swe-atlas`
Docker image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, Ubuntu 24.04) ships
kitty **pre-built** — `kitty/fast_data_types.so` is already present at `/app` — and runs **CPython
3.12.3** with **go1.23.4**; it was used to confirm that the built application imports and runs, but
it does **not** ship `cloc`. The version counts and the `cloc` size profile quoted below were
therefore taken in the **host-native** venue (Ubuntu 25.10), where the repository is likewise built
in place by its own native build (`python setup.py`, which produces the `kitty/fast_data_types` C
extension at `setup.py:1091`) and which provides **CPython 3.13.7**, **go1.24.4**, and **cloc 2.04**
(the repository itself declares `go 1.22` at `go.mod:3` and requires Python `>=3.8` at
`pyproject.toml:2`). The host-native figures are not attributed to the Docker image; where a specific
runtime version is material below, the venue is identified explicitly.

The two import-time failures (O3 and O4) are **build-independent** — they occur while Python is
*importing* modules, before any compiled code runs — so they were reproduced against a **pristine,
unbuilt checkout** created with `git archive HEAD | tar -x -C /tmp/kitty_unbuilt` (which deliberately
omits the git-ignored `*.so` artifacts; see `.gitignore:1`). The "after-build resolution" half of O3
is the only observation that requires a completed build, and it was confirmed against the in-place
built tree (where `kitty/fast_data_types.so` exists). All observation scaffolding lived outside the
tracked tree (under `/tmp`); the kitty source repository was left byte-for-byte unchanged.

Each section below is structured as **Observed behavior** (command + captured output) → **Code
evidence** (`path:line` citations) → **Rationale / Conclusion**.

---

## O1 — Which language carries the runtime workload, and where does the performance come from?

**Short answer:** The compiled **C core** owns the runtime hot paths — per-byte VT (escape-sequence)
parsing and the screen-model loop run on the **CPU in C** (SIMD-accelerated), while drawing and
compositing are **offloaded to the GPU** through GLSL shaders. **Python performs no hot-path work at
runtime**: it wires up options, windows, fonts, and the event loop, then hands every performance-
critical operation to the native extension. So "GPU based" is accurate for *rendering*, and the
CPU-side *C* is doing the rest of the heavy lifting.

### Observed behavior

A code-size profile is a useful first orientation, but — as shown below — it is a *method-dependent*
and therefore *weak* proxy for "where the work happens". The repository ships its own measurement
helper, `count-lines-of-code`, which runs `cloc` over `git ls-files` while excluding
linguist-generated/vendored files and two large generated `gen/` tables. Running it verbatim:

```text
$ ./count-lines-of-code
github.com/AlDanial/cloc v 2.04  T=0.81 s (809.5 files/s, 218858.8 lines/s)
--------------------------------------------------------------------------------
Language                      files          blank        comment           code
--------------------------------------------------------------------------------
Go                              257           4493            899          47437
Python                          188           9353           5326          46249
C                                47           2923           1033          28898
reStructuredText                 55           4057           1674           7488
Objective-C                       7           1006            534           6459
C/C++ Header                     41            597            554           3508
Bourne Shell                      9             97             71            690
JSON                              3             32              0            556
GLSL                             13             96             94            506
...
--------------------------------------------------------------------------------
SUM:                            655          22985          10495         143605
--------------------------------------------------------------------------------
```

> **Reproducibility note.** The capture above measures kitty's *own* tracked source at the time of
> observation. Two parts of it are not byte-stable across runs *by design*: `cloc`'s header timing/rate
> (`T=0.81 s …`) is recomputed every invocation, and the `SUM` line depends on the tree's exact file
> set. In particular, this very document is itself a tracked Markdown file under `blitzy/documentation/`,
> so a naive re-run of `./count-lines-of-code` in the destination repository also counts *these* lines —
> adding one file to the `Markdown` row and nudging `SUM` upward by however long this file currently is.
> The stable, deliverable-independent figure is kitty's own source: **655 files / 143,605 code**, exactly
> as shown once this document is excluded. Crucially, the per-language **code** figures the analysis
> relies on — Go 47,437, Python 46,249, C 28,898, C/C++ Header 3,508, GLSL 506 — are unaffected and
> reproduce exactly either way.

By this **official `cloc` code-line method** (which *excludes* the vendored GLFW C under `glfw/`),
the ranking is **Go (47,437) > Python (46,249) > C (28,898) + C/C++ Header (3,508) = 32,406 > GLSL
(506)**. Measured this way, C is only the *third* largest body of code. Raw line counts (including
blanks/comments and, depending on the cut, the vendored C) tell a different story:

| Measurement (over `git ls-files`)              | C + headers | Python  | Go      | GLSL |
| ---------------------------------------------- | ----------: | ------: | ------: | ---: |
| `cloc` *code* lines (repo helper, excl. vendored/generated) | **32,406** | 46,249 | 47,437 | 506 |
| raw lines, whole tree (all tracked files)      | **99,745**  | 62,874  | 56,071  | 696  |
| raw lines, excluding vendored `glfw/` + `3rdparty/` | 61,210 | 62,415  | 56,071  | —    |
| raw lines, **`kitty/` package only**           | **60,231**  | 39,355  | —       | —    |

The numbers disagree on the headline depending on what you count and how, which is precisely why a
line count cannot, by itself, answer "where does performance come from". The decisive evidence is
*where the runtime work is located*, not how many lines each language occupies. Two facts settle it:
within the actual terminal core (the `kitty/` package), C is dominant by raw lines (60,231 vs
39,355); and — far more importantly — Python *imports every hot-path operation from the native C
extension* rather than implementing any of it:

```text
# kitty/main.py — the orchestration layer pulls its primitives from the C core
32: from .fast_data_types import (
33:     GLFW_MOD_ALT,
...
36:     create_os_window,
38:     glfw_init,
44:     set_options,
45: )
```

### Code evidence

- `kitty/data-types.c:525` — `PyInit_fast_data_types(void) {` (preceded by `EXPORTED PyMODINIT_FUNC`
  at `:524`): the native module `kitty.fast_data_types` is a **C** CPython extension.
- `setup.py:1090-1091` — `compile_c_extension(kitty_env(args), 'kitty/fast_data_types', ...)`: the
  build system compiles that extension from the collected C sources/headers.
- `kitty/main.py:32-45` — `from .fast_data_types import (...)`: the Python entry layer imports its
  runtime primitives (`create_os_window`, `glfw_init`/`glfw_terminate`, `set_options`,
  `load_png_data`, `free_font_data`, `set_custom_cursor`, …) **from the C core**.
- Hot-path C sources (raw bytes on disk): `kitty/screen.c` ≈ 200 KB (199,784 B, the screen model),
  `kitty/vt-parser.c` ≈ 55 KB (55,306 B, escape-sequence parsing), plus `kitty/line.c` (36,131 B)
  and `kitty/line-buf.c` (23,125 B). The per-byte parsing and grid mutation live here, in C.
- SIMD acceleration: `kitty/simd-string-128.c` and `kitty/simd-string-256.c` are thin 9-line
  translation units that `#define KITTY_SIMD_LEVEL 128|256` and `#include "simd-string-impl.h"`,
  compiling one shared implementation at two SIMD widths; `setup.py:742-744` adds the per-file
  `-fopenmp-simd -DSIMDE_ENABLE_OPENMP` flags. So the same C parsing code is built twice for 128-bit
  and 256-bit SIMD.
- GPU offload (rendering): the shader *program registry* and the GL management live in C —
  `kitty/shaders.c:20` defines `enum { CELL_PROGRAM, … , TINT_PROGRAM, NUM_PROGRAMS }`, and the
  compositing/draw setup (`init_cell_program`, `bind_program`, `glUniform…`) is C (see O2).
- `pyproject.toml:2` (`requires-python = ">=3.8"`) and `go.mod:3` (`go 1.22`) — Python and Go are
  declared toolchains; the Go body is overwhelmingly the standalone `kitten` CLI tooling, not the
  terminal hot path.
- Public framing corroboration: `README.asciidoc:1` ("… GPU based terminal") and `docs/index.rst:22`
  ("Offloads rendering to the GPU"), `docs/index.rst:23` ("Uses threaded rendering").

### Rationale / Conclusion

A naïve "count the lines" approach is inconclusive and even misleading here: by the repository's own
`cloc` helper, Go and Python both out-measure C, because that method excludes the vendored GLFW C and
strips comments/blanks. What *is* unambiguous from the code is the **division of labor at runtime**.
The Python layer (`kitty/main.py` and friends) does configuration, window/event-loop orchestration,
and glue — and it obtains *every* performance-sensitive primitive by importing it from the native
`kitty.fast_data_types` C extension (`kitty/main.py:32-45`). The CPU hot paths — VT parsing
(`kitty/vt-parser.c`) and the screen model (`kitty/screen.c`), SIMD-accelerated
(`kitty/simd-string-*.c`) — are C. The drawing/compositing stage is handed to the GPU through GLSL
shaders whose program registry and compiler entry point are themselves in C (`kitty/shaders.c`).
Therefore performance comes from **C on the CPU (parsing + grid model, SIMD) plus the GPU for
rendering**, with Python doing zero hot-path work — exactly the nuance behind kitty's "GPU based"
self-description.

---

## O2 — What do the scattered `*.glsl` files do, and how central are they?

**Short answer:** The `*.glsl` files are kitty's **entire rendering pipeline**. They are the OpenGL
vertex/fragment shaders that draw the terminal cells (text), borders, images (the graphics
protocol), background image, and tint overlay. There is **no CPU/software text-drawing fallback** —
the GPU shader path is the *only* render path, which is exactly what substantiates the "GPU based"
identity.

### Observed behavior

There are exactly 13 shader files in `kitty/`, which group cleanly by render concern:

```text
$ ls -1 kitty/*.glsl
kitty/alpha_blend.glsl        # color/blend utility (included by others)
kitty/bgimage_fragment.glsl   # background image
kitty/bgimage_vertex.glsl
kitty/border_fragment.glsl    # window borders
kitty/border_vertex.glsl
kitty/cell_defines.glsl       # SHARED #include (not a standalone program)
kitty/cell_fragment.glsl      # terminal cells = text rendering
kitty/cell_vertex.glsl
kitty/graphics_fragment.glsl  # graphics/image protocol
kitty/graphics_vertex.glsl
kitty/linear2srgb.glsl        # color-space utility (included by others)
kitty/tint_fragment.glsl      # tint overlay
kitty/tint_vertex.glsl
```

`cell_defines.glsl` is not a program of its own — it is a shared header pulled in via kitty's custom
`#pragma kitty_include_shader` directive:

```text
$ grep -n "cell_defines" kitty/*.glsl
kitty/cell_fragment.glsl:3:#pragma kitty_include_shader <cell_defines.glsl>
kitty/cell_vertex.glsl:2:#pragma kitty_include_shader <cell_defines.glsl>
```

These are not peripheral assets — they are consumed at **both build time and runtime**. At build
time `setup.py` globs them and code-generates C uniform-location structs from each shader's `uniform`
declarations; at runtime `kitty/shaders.py` reads each shader as a packaged resource, resolves
recursive `#include`s, prepends a `#version` line, and hands the assembled source to the **native**
`compile_program`. Searching for any software/CPU text-drawing fallback returns nothing:

```text
$ grep -rniE 'software[ _-]?(render|rast|draw)|cpu[ _-]?(render|draw|raster)|no[_-]?gpu|fallback[ _-]?render' kitty/*.c kitty/*.py
$            # (empty — no software/CPU text-drawing fallback path exists)
```

### Code evidence

- The 13 `kitty/*.glsl` files above: cell text (`cell_*`), borders (`border_*`), graphics protocol
  (`graphics_*`), background image (`bgimage_*`), tint overlay (`tint_*`), and the shared utilities
  `cell_defines.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl`.
- `kitty/cell_fragment.glsl:3` and `kitty/cell_vertex.glsl:2` — `#pragma kitty_include_shader
  <cell_defines.glsl>`: confirms `cell_defines.glsl` is a shared include, not a standalone program.
- Runtime loader — `kitty/shaders.py`:
  - `:9` — `from .constants import read_kitty_resource` (shaders are read as packaged resources).
  - `:10-32` — `from .fast_data_types import (...)`: the shader **program registry**
    (`CELL_PROGRAM`, `CELL_BG_PROGRAM`, `CELL_FG_PROGRAM`, `CELL_SPECIAL_PROGRAM`, `GRAPHICS_PROGRAM`,
    `GRAPHICS_PREMULT_PROGRAM`, `GRAPHICS_ALPHA_MASK_PROGRAM`, `BGIMAGE_PROGRAM`, `TINT_PROGRAM`),
    plus `GLSL_VERSION` (`:19`) and `compile_program` (`:29`), are imported **from the C core**.
  - `:61` — `def _load_sources(...)`; `:63` — prepends `#version {GLSL_VERSION}`; `:68` —
    `read_kitty_resource(name)`; with recursive `#include` handling (`:72` onward).
  - `:90` — `compile_program(program_id, self.vertex_sources, self.fragment_sources,
    allow_recompile)`: compilation is **delegated to the native `compile_program`**.
- Native (C) side of shader/GL management:
  - `kitty/shaders.c:20` — `enum { CELL_PROGRAM, CELL_BG_PROGRAM, CELL_SPECIAL_PROGRAM,
    CELL_FG_PROGRAM, BORDERS_PROGRAM, GRAPHICS_PROGRAM, GRAPHICS_PREMULT_PROGRAM,
    GRAPHICS_ALPHA_MASK_PROGRAM, BGIMAGE_PROGRAM, TINT_PROGRAM, NUM_PROGRAMS }` (the registry origin).
  - `kitty/shaders.c:217` `init_cell_program`, `:225`/`:578` `bind_program`, `:226` `glUniform1fv`
    (OpenGL program/uniform management in C).
  - `kitty/shaders.c:1168` — `compile_program(PyObject *self, PyObject *args)`; registered in the
    method table at `kitty/shaders.c:1236` (`M(compile_program, METH_VARARGS)`): the Python symbol
    resolves to this C function. `kitty/gl.c` is the OpenGL wrapper used by it.
- Build-time consumption — `setup.py:1040` (`for x in sorted(glob.glob('kitty/*.glsl')):`), `:1045`
  (`{name.capitalize()}Uniforms`), `:1046` (`get_uniform_locations_{name}`), `:1049`
  (`find_uniform_names`): each `*_fragment`/`*_vertex` shader's `uniform`s are turned into C structs.
- No CPU fallback: the fallback grep above returns empty across `kitty/*.c` and `kitty/*.py`.
- Public framing: `README.asciidoc:1`, `docs/index.rst:4` ("GPU based terminal emulator"),
  `docs/index.rst:22-23` ("Offloads rendering to the GPU" / "Uses threaded rendering").

### Rationale / Conclusion

The shaders are as central as it gets: they *are* the renderer. Everything the user sees — text
cells, borders, inline images, background image, tinting — is drawn by one of these GLSL programs.
Their centrality is reinforced from two directions in the source: the **build** treats them as
first-class inputs (globbing every `kitty/*.glsl` and generating C uniform structs at
`setup.py:1040-1049`), and the **runtime** loads/assembles them in `kitty/shaders.py` only to hand
them to the native `compile_program` for GPU compilation (`kitty/shaders.py:90` →
`kitty/shaders.c:1168`). Crucially, the program registry, the GLSL version, and the compiler all
originate in the C extension (`kitty/shaders.py:10-32`, `kitty/shaders.c:20`), so O2 and O1 are the
same story told from the rendering side. And because a search for any software/CPU text-drawing
fallback comes back empty, the GPU shader pipeline is not *a* render path — it is the *only* render
path. That is what makes "GPU based" a literal architectural fact rather than marketing.

---

## O3 — Why does the entry point fail immediately, and what is the one critical dependency?

**Short answer:** Run against a freshly checked-out (unbuilt) tree, the main entry point raises
`ModuleNotFoundError: No module named 'kitty.fast_data_types'`. That native C extension is the
**single indispensable dependency**, and it is wired in at **import time** — so the whole application
is unimportable until the C core is compiled. Building the extension makes the error disappear.

### Observed behavior

Against a pristine, unbuilt checkout (`/tmp/kitty_unbuilt`, produced by `git archive` — no `*.so`
present), running the entry point fails immediately:

```text
$ cd /tmp/kitty_unbuilt && python3 __main__.py
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
```

The traceback *is* the dependency chain: `__main__.py` → `kitty.entry_points.main` → `kitty.main` →
`kitty.borders` → `import kitty.fast_data_types` (which does not exist yet). After building the
extension (`python setup.py`, which compiles `kitty/fast_data_types` at `setup.py:1091`), the same
invocation succeeds — the `ModuleNotFoundError` is gone:

```text
$ cd <repo root, with kitty/fast_data_types.so present> && python3 __main__.py --version
kitty 0.35.2 created by Kovid Goyal
```

### Code evidence

- `__main__.py:5-7` — `if __name__ == '__main__':` / `from kitty.entry_points import main` / `main()`:
  the root entry point delegates into the `kitty` package.
- `kitty/entry_points.py:183` — `def main() -> None:`; `:194` — `from kitty.main import main as
  kitty_main`; `:195` — `kitty_main()`: `main()` imports `kitty.main` on the way to launching.
- `kitty/main.py:11` — `from .borders import load_borders_program`: the **first hop** that drags in
  the native bridge.
- `kitty/borders.py:7` — `from .fast_data_types import BORDERS_PROGRAM, add_borders_rect,
  get_options, init_borders_program, os_window_has_background_image`: **the import that fails** on an
  unbuilt tree.
- Why it is absent from a fresh checkout: `kitty/fast_data_types.pyi` is a **type stub only** (a
  35,795-byte `.pyi`), and there is **no `kitty/fast_data_types.py` twin** — the real module is the
  compiled `.so`. `.gitignore:1` lists `*.so`, so the compiled extension is intentionally never
  committed and is simply not present until you build.
- Built-state resolution: `setup.py:1090-1091` compiles `kitty/fast_data_types`; once `kitty/
  fast_data_types.so` exists, re-running the entry point no longer raises `ModuleNotFoundError`
  (observed above as `kitty 0.35.2`).

### Rationale / Conclusion

The "cryptic error" is simply the Python import machinery hitting a missing native module. What makes
it *the* critical dependency is *when* it is required: `kitty.borders` imports
`kitty.fast_data_types` at **module load** (`kitty/borders.py:7`), not lazily inside a function, so
the import chain from `__main__.py` cannot even finish loading `kitty.main` without it. The Python
layer is therefore hard-wired to the compiled C core at import time. That single artifact —
`kitty/fast_data_types.so` — is the one indispensable piece: present, the app starts; absent, the
entire main-application import chain (`__main__.py` → `entry_points` → `main` → `borders`) and every
`kitty.*` module that imports the native bridge at module scope (for example `kitty.utils`,
`kitty.cli`, `kitty.clipboard`, `kitty.conf.utils`, `kitty.rgb`) cannot load. (A handful of
dependency-light leaf modules — for example `kitty.constants` and `kitty.types` — *do* still import
on an unbuilt tree, because they touch the native bridge only inside functions, if at all; but
nothing that actually drives the terminal can run without the extension.) The fix is not a code
change but a *build*; this document deliberately explains the failure rather than remediating it.

---

## O4 — Are the kittens truly independent, or do they quietly depend on the same native bridge?

**Short answer:** Running a kitten standalone fails **identically** to the main entry point, with the
same `ModuleNotFoundError: No module named 'kitty.fast_data_types'`. The `kittens/` modularity is
**organizational, not runtime-independent**: every `tui`-based kitten transitively imports the same
native bridge at module load.

### Observed behavior

Two representative kittens, run standalone against the unbuilt tree, both fail — and they converge on
the *same* line via *different* paths:

```text
$ cd /tmp/kitty_unbuilt && python3 -m kittens.hints.main
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
```

```text
$ cd /tmp/kitty_unbuilt && python3 -m kittens.diff.main
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/kitty_unbuilt/kittens/diff/main.py", line 8, in <module>
    from kitty.cli import CONFIG_HELP, CompletionSpec
  File "/tmp/kitty_unbuilt/kitty/cli.py", line 13, in <module>
    from .conf.utils import resolve_config
  File "/tmp/kitty_unbuilt/kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

`hints` reaches it via `kitty.clipboard`; `diff` reaches it via `kitty.cli`; **both** terminate at
`kitty/conf/utils.py:27`, `from ..fast_data_types import Color` — the shared convergence point.

### Code evidence

- Representative kittens and their first cross-package import:
  - `kittens/hints/main.py:9` — `from kitty.clipboard import set_clipboard_string,
    set_primary_selection` (this triggers the failure first; `kittens/hints/main.py:11` also does
    `from kitty.fast_data_types import get_options` directly, but line 9 already aborts the import).
  - `kittens/diff/main.py:8` — `from kitty.cli import CONFIG_HELP, CompletionSpec`.
- The intermediate hops:
  - `kitty/clipboard.py:11` — `from .conf.utils import uniq` (the `hints` path).
  - `kitty/cli.py:13` — `from .conf.utils import resolve_config` (the `diff` path).
- The shared convergence choke point: `kitty/conf/utils.py:27` — `from ..fast_data_types import
  Color`. Both kittens reach this exact line, demonstrating the coupling is structural.
- The dispatcher itself depends on `kitty.*`: `kittens/runner.py:12-14` — `from kitty.constants
  import list_kitty_resources` / `from kitty.types import run_once` / `from kitty.utils import
  resolve_abs_or_config_path`.
- The shared TUI framework pulls the native bridge across many modules — 8 files under `kittens/tui/`
  import `fast_data_types`: `handler.py` (`:10`, `from kitty.fast_data_types import monotonic`),
  `utils.py`, `line_edit.py`, `operations.py`, `spinners.py`, `loop.py`, `images.py`,
  `path_completer.py`. Any kitten built on `tui` inherits the dependency.
- Scope of the pattern: 18 kitten subpackages ship a `main.py` — `ask`, `broadcast`, `choose_fonts`,
  `clipboard`, `diff`, `hints`, `hyperlinked_grep`, `icat`, `pager`, `panel`, `query_terminal`,
  `remote_file`, `resize_window`, `show_key`, `ssh`, `themes`, `transfer`, `unicode_input`. These
  reach the native bridge through more than one choke point, not a single shared line. The
  representative `hints` and `diff` paths converge at `kitty/conf/utils.py:27` because
  `kitty/clipboard.py:11` and `kitty/cli.py:13` each import `.conf.utils` *before* their own direct
  `fast_data_types` import. By contrast `kitty/utils.py` imports `fast_data_types` **directly** at
  module scope (`kitty/utils.py:45`), and `kitty/constants.py` reaches it only inside functions
  (`kitty/constants.py:154`, `:170`, `:304`) — so `utils.py` and `constants.py` are *additional*
  native-bridge choke points, not routes through `kitty/conf/utils.py:27`.

### Rationale / Conclusion

The `kittens/` directory is a clean *organizational* decomposition — each tool is a tidy subpackage
with its own `main.py`. But organization is not runtime independence. At load time, a kitten imports
shared `kitty.*` modules (`kitty.clipboard`, `kitty.cli`, the `kittens/tui/*` framework), and those
modules import `kitty.fast_data_types` at module scope — frequently by converging on
`kitty/conf/utils.py:27`, though some (such as `kitty.utils`) import the bridge directly. So a
standalone kitten is no more self-contained than the main app: both
fail with the identical `ModuleNotFoundError` until the native core is built. The modularity is real
at the source-layout level and absent at the runtime-dependency level.

---

## Summary — O3 and O4 are the same root cause

O1 and O2 establish *where the work is*: the C core owns the CPU hot paths (VT parsing + screen
model, SIMD-accelerated), and the GPU owns rendering through GLSL shaders whose registry and compiler
live in C — with Python doing only orchestration. O3 and O4 are then the **same fact observed from
two entry points**. The main application reaches the native bridge through
`kitty/borders.py:7`; an arbitrary kitten reaches it through `kitty/conf/utils.py:27`; both terminate
at `import kitty.fast_data_types`, and both fail identically on an unbuilt tree:

```text
__main__.py ─▶ kitty.entry_points.main ─▶ kitty.main:11 ─▶ kitty.borders:7 ─┐
                                                                            ├─▶ import kitty.fast_data_types
kittens/hints/main.py:9 ─▶ kitty.clipboard:11 ─▶ kitty.conf.utils:27 ───────┤      └─▶ ModuleNotFoundError
kittens/diff/main.py:8  ─▶ kitty.cli:13       ─▶ kitty.conf.utils:27 ───────┘          (until the .so is built)
```

The unifying conclusion: **`kitty.fast_data_types` (the compiled C `.so`) is the one indispensable
dependency, wired in at import time.** Its absence is not a bug to be fixed here but the very
evidence that the Python layer is a thin orchestration shell over a C+GPU core — which is precisely
why kitty is, accurately, "GPU based."

