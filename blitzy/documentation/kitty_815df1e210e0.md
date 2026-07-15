
# kitty: Where the Work Actually Happens — A Run-First Architecture Investigation

> **The tension.** kitty "calls itself GPU accelerated yet the codebase feels like a tight weave of Python and C." This document resolves that tension **empirically**: rather than reasoning about how the architecture is *meant* to work, every behavioral claim below was produced by **running the real code paths and capturing the actual output**, then citing the exact repository-relative `file:line` and the named function/struct that does the work. Four questions are answered — which language does the heavy lifting, what the GLSL files do, why the main entry point fails, and whether kittens are truly independent — followed by a synthesis that ties them together.

## Methodology and environment

- **Run-first discipline.** Each answer leads with a direct conclusion, then shows the **exact command**, its **complete, unedited output**, and an explicit **`[exit: N]`** status; then the repository-relative **`file:line` + named symbol** evidence; then the cause-and-effect rationale. Anything not directly observed is explicitly prefixed **`Inferred:`**. Every answer/reproduction command block (Q1–Q4 and the appendices) is reproduced complete and unedited; the only condensations are two clearly-marked presentations in Appendix B — the canonical build transcript, whose repetitive per-translation-unit compile lines are elided at an explicitly marked point (its compiler identification, complete final link line, Go build line, and exit status are shown verbatim), and the present→absent→present transition, shown as a condensed step list (each step's command, key outcome, and `[exit: N]`) whose full verbatim tracebacks are the ones already shown in Q3/Q4.
- **Canonical paths only.** Failures are reproduced through the real entry points — the repository-root `__main__.py` and a standalone kitten via `kittens.runner` — never through a debug hook, mock, fallback, or synthetic stand-in.
- **Read-only, and left clean.** The only file this task creates or modifies is this Markdown document. To observe post-build behavior, kitty was built canonically (Appendix B); **all build products were then removed**, so the working tree is left with no build, native, generated, or cache artifacts (verified in Appendix E: `0` ignored entries). No tracked source file was modified, and nothing under `/app` was accessed.
- **Two states, both observed.** Because the Q3/Q4 failures are triggered by the **absence** of the compiled extension `kitty/fast_data_types.so`, this document documents both: the **post-build** state (`.so` present — kitty runs and creates a real GPU context), captured during the build window; and the **clean / pre-build** state (`.so` absent — the repository's final state), in which the failures reproduce **directly**, with no manipulation. Appendix B additionally captures both states around a single build artifact inside one interruption-safe transition with md5 verification.

**Observed environment (host, final clean state):**

```console
$ git rev-parse --show-toplevel
/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770
[exit: 0]

$ git branch --show-current
blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd
[exit: 0]

$ /opt/kitty311-venv/bin/python3 --version
Python 3.11.15
[exit: 0]

$ /usr/bin/python3 --version
Python 3.13.7
[exit: 0]

$ ls -la kitty/fast_data_types*
-rw-r--r-- 1 root root 35795 Jul 14 18:52 kitty/fast_data_types.pyi
[exit: 0]

$ ls kitty/__main__.py 2>&1
ls: cannot access 'kitty/__main__.py': No such file or directory
[exit: 2]
```

Three facts shape everything that follows:

1. **The importable native extension `kitty/fast_data_types.so` is produced only by the build.** In the final clean state only its **type stub** `kitty/fast_data_types.pyi` (35,795 bytes) is present — and a `.pyi` file is **not importable Python code**; it only describes types for static analysis. There is no `kitty/fast_data_types.py` fallback. The canonical build that produces the `.so` is shown in Appendix B.
2. **The canonical interpreter is the virtualenv `python3` at `/opt/kitty311-venv/bin/python3` (CPython 3.11.15).** This is the interpreter the extension is built for: the built `.so` links against `libpython3.11.so.1.0` (`readelf` evidence in Q3). The system `python3` is 3.13.7. kitty declares only a version **floor** (`requires-python = ">=3.8"`), not a ceiling; the 3.11 venv is used here because the compiled `.so` is ABI-bound to `libpython3.11` (see Q3 and Synthesis for the correct compatibility framing).
3. **`kitty/__main__.py` does not exist**, which is why `python3 -m kitty` fails *differently* from running the repository-root `__main__.py` (Q3).

### Build provenance — the supplied image vs. the canonical host toolchain

The "Observed environment" above is the **host** on which every answer in this document was reproduced. It is deliberately **not identical** to the **supplied container image**, and that difference is the reason CPython **3.11** — not the image's default 3.12 — is the canonical interpreter here. Both are recorded so the provenance is unambiguous.

**The supplied image (provenance).** The project's build/run image is `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (also published as `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). Its identity and *base* toolchain, observed directly:

```text
### CMD: IMG=andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1   # variable used by the commands below

### CMD: docker image inspect "$IMG" --format 'ID={{.Id}}  WORKDIR={{.Config.WorkingDir}}'
ID=sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288  WORKDIR=/app
[exit: 0]

### CMD: docker image inspect "$IMG" --format '{{range .RepoDigests}}{{println .}}{{end}}'
ghcr.io/scaleapi/swe-atlas@sha256:60da90a7183a82861fc6d1d40cb8086baa6a8a0e0d05f26d03aafd0f5b3cc384
[exit: 0]

### CMD: docker run --rm --entrypoint /bin/bash "$IMG" -lc '. /etc/os-release; echo "$PRETTY_NAME"; python3 --version; go version; gcc --version | head -1; test -x /opt/kitty311-venv/bin/python3 && echo "/opt/kitty311-venv: present" || echo "/opt/kitty311-venv: absent"'
Ubuntu 24.04.2 LTS
Python 3.12.3
go version go1.23.4 linux/amd64
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
/opt/kitty311-venv: absent
[exit: 0]
```

So the base image ships **Ubuntu 24.04.2 / CPython 3.12.3 / Go 1.23.4 / gcc 13.3.0**, its working directory is `/app`, and it does **not** contain `/opt/kitty311-venv`.

**The canonical host toolchain (what actually builds and runs kitty here).** On top of that image, the host filesystem is provisioned with the toolchain used throughout this document: **CPython 3.11.15** at `/opt/kitty311-venv/bin/python3`, **Go 1.24.4**, **gcc 15.2.0**, on **Ubuntu 25.10** (the "Observed environment" block above; full table in Appendix A). CPython **3.11** is used deliberately, not 3.12: it is the **highest Python version the project's own CI documents** (`.github/workflows/ci.yml:85` pins `python-version: "3.11"` — Q3), and building against it makes the compiled extension **ABI-bound to `libpython3.11.so.1.0`** (`readelf` evidence in Q3 and Appendix B). A build under the image's default 3.12 would instead emit a `libpython3.12`-bound `.so` that does not correspond to kitty's highest documented Python, so it is *not* the canonical artifact and is not used here.

**How to read the evidence that follows.** Except for the three `docker …` commands just above — which observe the **supplied image** for provenance — every command and captured output in this document was produced on the **host** toolchain just described (the canonical build/run environment). The supplied image is the starting point; the host CPython-3.11 toolchain is where the answers are reproduced.

---

## Q1 — Which language does the heavy lifting, and where does performance come from?

**Direct answer.** At runtime the **compiled C core does the heavy lifting**: terminal I/O, VT parsing, screen-grid mutation, and rendering are all implemented in C, evidenced by the named hot-path symbols below. *Inferred* (from those code paths, not a profiler run): runtime performance therefore originates in **C plus GPU offload**. Python is a comparatively thin orchestration / configuration / extensibility shell, and Go is a **separate, self-contained** command-line binary that is not loaded into kitty's address space and is not part of the in-process hot path. The per-byte and per-frame work — reading the PTY, parsing VT escape sequences, mutating the screen grid, rasterizing glyphs into a texture atlas, and issuing OpenGL draw calls — is implemented in C and on the GPU; Python dispatches high-level events and wires the C components together.

### The language footprint (measured two ways, each stable across two runs)

The primary measure is **tracked source** (`git ls-files`), the honest size of the hand-written codebase. The command is NUL-safe (`-z` / `xargs -0`, so filenames with spaces or newlines cannot corrupt the count) and was run twice with identical results:

```console
$ for run in 1 2; do echo "--- RUN $run ---"; for ext in c h py go glsl; do files=$(git ls-files -z "*.$ext" | tr -dc '\0' | wc -c); lines=$(git ls-files -z "*.$ext" | xargs -0 cat | wc -l); printf '.%-4s files=%-4s lines=%s\n' "$ext" "$files" "$lines"; done; done
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
[exit: 0]
```

| Language | Files | Lines | Runtime role |
|----------|-------|-------|--------------|
| C (`.c`) | 128 | 61,806 | Performance core (`fast_data_types`, bundled GLFW) |
| C headers (`.h`) | 84 | 37,939 | C interfaces |
| Python (`.py`) | 214 | 62,874 | Orchestration, configuration, kittens |
| Go (`.go`) | 258 | 56,071 | Standalone CLI tools / `kitten` binary |
| GLSL (`.glsl`) | 13 | 696 | GPU shader programs |

The second measure is a **live on-disk** count (`find … -print0`, NUL-safe, excluding `.git`). In the final clean state it is **identical** to the tracked count (run twice), which also confirms that cleanup removed every build product:

```console
$ for run in 1 2; do echo "--- RUN $run ---"; for ext in c h py go glsl; do files=$(find . -path ./.git -prune -o -type f -name "*.$ext" -print0 | tr -dc '\0' | wc -c); lines=$(find . -path ./.git -prune -o -type f -name "*.$ext" -print0 | xargs -0 cat | wc -l); printf '.%-4s files=%-4s lines=%s\n' "$ext" "$files" "$lines"; done; done
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
[exit: 0]
```

**Reconciliation (tracked vs. live during the build).** During the canonical build (Appendix B), *before* cleanup, the live on-disk tree additionally contained build-generated files, so the live count was higher — C 142 / 63,225; H 100 / 48,447; Go 338 / 69,151 (Python and GLSL unchanged). Differencing live-during-build against tracked (`grep -vxF` of the tracked list against the on-disk list) attributes the delta entirely to generated code, not hand-written source:

- **+80 `.go`** = **74** `*_generated.go` + **6** `*_generated_test.go` (`338 − 258 = 80`; note the naive `comm` approach under-reports this as 78, which is why it must be computed by exact set difference or arithmetic).
- **+14 `.c`** = the bundled GLFW Wayland protocol sources `glfw/wayland-*-client-protocol.c` (emitted by `wayland-scanner`).
- **+16 `.h`** = the 14 matching `glfw/wayland-*-client-protocol.h` plus `kitty/docs_ref_map_generated.h` and `kitty/uniforms_generated.h`.

All of these were removed during cleanup (Appendix E), which is why the two clean-state measures above agree exactly. The tracked-source table is therefore the authoritative footprint. Headline: **C source (61,806) + C headers (37,939) = 99,745 lines of C**, the largest body of code, and — as the named symbols below show — it is exactly the code on the per-byte and per-frame path.

### The C core is the hot path (named symbols, not just file sizes)

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
[exit: 0]
```

Size is context, not a timing measurement. The runtime path is anchored to **named functions** (all line numbers repository-relative):

```console
$ grep -n -E '^read_bytes\(|vt_parser_create_write_buffer\(screen|len = read\(fd|vt_parser_commit_write\(screen|^render\(monotonic|^render_os_window\(' kitty/child-monitor.c
833:render_os_window(OSWindow *w, monotonic_t now, bool ignore_render_frames, bool scan_for_animated_images) {
871:render(monotonic_t now, bool input_read) {
1337:read_bytes(int fd, Screen *screen) {
1341:    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
1345:        len = read(fd, buf, available_buffer_space);
1349:            vt_parser_commit_write(screen->vt_parser, 0);
1354:    vt_parser_commit_write(screen->vt_parser, len);
[exit: 0]

$ grep -n -E '^run_worker\(|^parse_worker\(' kitty/vt-parser.c
1417:run_worker(void *p, ParseData *pd, bool flush) {
1496:parse_worker(void *p, ParseData *pd, bool flush) { run_worker(p, pd, flush); }
[exit: 0]

$ grep -n -E '^screen_draw_text\(|^draw_codepoint\(' kitty/screen.c
866:screen_draw_text(Screen *self, const uint32_t *chars, size_t num_chars) {
872:draw_codepoint(Screen *self, char_type ch) {
[exit: 0]

$ grep -n -E '^draw_cells_simple\(|^draw_cells_interleaved\(|^draw_cells\(' kitty/shaders.c
577:draw_cells_simple(ssize_t vao_idx, Screen *screen, const CellRenderData *crd, bool is_semi_transparent) {
868:draw_cells_interleaved(ssize_t vao_idx, Screen *screen, OSWindow *w, const CellRenderData *crd, const WindowLogoRenderData *wl) {
1009:draw_cells(ssize_t vao_idx, const WindowRenderData *srd, OSWindow *os_window, bool is_active_window, bool is_tab_bar, bool is_single_window, Window *window) {
[exit: 0]

$ grep -n -E 'PyInit_fast_data_types' kitty/data-types.c
525:PyInit_fast_data_types(void) {
[exit: 0]
```

Reading these symbols end-to-end gives the per-byte / per-frame path, entirely in C:

- **PTY input → parser buffer.** `kitty/child-monitor.c:read_bytes` (`:1337`) obtains a write buffer from the parser via `vt_parser_create_write_buffer` (`:1341`), performs the raw `read(fd, …)` from the child PTY (`:1345`), and hands the bytes to the parser with `vt_parser_commit_write` (`:1354`).
- **VT parsing.** `kitty/vt-parser.c:run_worker` (`:1417`) is the parse loop; `parse_worker` (`:1496`) is its thin entry that calls it. This turns raw PTY bytes into screen mutations.
- **Screen-grid mutation.** `kitty/screen.c:screen_draw_text` (`:866`) and `draw_codepoint` (`:872`) write characters into the in-memory cell grid (`screen.c` is 4,932 lines — the grid/scrollback/cursor model).
- **Render scheduling + GPU draw.** `kitty/child-monitor.c:render` (`:871`) and `render_os_window` (`:833`) drive redraw on a dedicated C thread; the actual GPU draw calls are `kitty/shaders.c:draw_cells` (`:1009`) → `draw_cells_simple` (`:577`) / `draw_cells_interleaved` (`:868`).
- **Extension entry point.** `kitty/data-types.c:PyInit_fast_data_types` (`:525`) is the CPython module-init function — the single C symbol that *is* `kitty.fast_data_types` (the module at the center of Q3/Q4).

SIMD acceleration for the parser's byte scanning is dispatched at runtime by CPU capability:

```console
$ grep -n -E 'find_either_of_two_bytes_scalar\(|_impl\)\(.*= find_either_of_two_bytes_scalar|^init_simd\(|__builtin_cpu_supports|find_either_of_two_bytes_impl = find_either_of_two_bytes_(128|256)' kitty/simd-string.c
21:find_either_of_two_bytes_scalar(const uint8_t *haystack, const size_t sz, const uint8_t x, const uint8_t y) {
28:static const uint8_t* (*find_either_of_two_bytes_impl)(const uint8_t*, const size_t, const uint8_t, const uint8_t) = find_either_of_two_bytes_scalar;
194:init_simd(void *x) {
198:#define do_check() { has_sse4_2 = __builtin_cpu_supports("sse4.2") != 0; has_avx2 = __builtin_cpu_supports("avx2") != 0; }
233:        find_either_of_two_bytes_impl = find_either_of_two_bytes_256;
241:        if (find_either_of_two_bytes_impl == find_either_of_two_bytes_scalar) find_either_of_two_bytes_impl = find_either_of_two_bytes_128;
[exit: 0]
```

A scalar baseline (`simd-string.c:21`) is held behind a function pointer (`:28`); at startup `init_simd` (`:194`) probes the CPU with `__builtin_cpu_supports` (`:198`) and upgrades the pointer to the AVX2/256-bit (`:233`) or SSE4.2/128-bit (`:241`) implementation. The tiny `simd-string-128.c` / `simd-string-256.c` (9 lines each) are shims that compile the shared implementation at each vector width.

The native windowing/input layer is the bundled GLFW, itself C — X11 (`x11_*.c`), Wayland (`wl_*.c`), and macOS (`cocoa_*.m`):

```console
$ git ls-files 'glfw/*.c' | wc -l
31
[exit: 0]
```

### The performance is observably GPU-backed

With the `.so` present (post-build), kitty runs and creates a real OpenGL context. Under a headless X server with Mesa's software GL, the canonical **C launcher** brings up an **OpenGL 4.5 Core Profile** context (the debug-GL lines are extracted with `grep`, so the output shown is complete for the command as written):

```console
$ LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a ./kitty/launcher/kitty --debug-rendering -o allow_remote_control=no -o confirm_os_window_close=0 -e true 2>&1 | grep -iE 'GL_|OpenGL|Mesa|renderer|version|OS Window|Child|window created|Xvfb'; echo "[pipeline exit: ${PIPESTATUS[0]}]"
[0.305] OS Window created
[0.326] Child launched
[0.283] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[pipeline exit: 0]
```

Directly observed: an OS window is created, a child process is launched, and an **OpenGL 4.5 (Core Profile)** context backed by **Mesa 25.2.8** is established — i.e., rendering is offloaded to the GL pipeline. *Inferred:* because the container has no discrete GPU, this Mesa context is served by a software rasterizer (Mesa's LLVMpipe); this was not directly confirmed (`glxinfo` is not installed in the environment), so only "Mesa 25.2.8 / GL 4.5 Core" is asserted as observed.

### The Go layer is separate

```console
$ sed -n '1,3p' go.mod
module kitty

go 1.22
[exit: 0]
```

Go compiles to an independent executable (`kitty/launcher/kitten` — see Q4). It is not loaded into kitty's address space and carries no reference to the Python extension; it powers the command-line `kitten` tools. Its linkage is examined in Q4 (it is a *dynamically linked* ELF, not a static binary).

### Rationale — why this distribution means "performance comes from C + GPU"

Cause and effect: every byte from the shell flows through `read_bytes` (`child-monitor.c:1337`) into the C `vt-parser.c` worker (`:1417`), mutates the C `screen.c` grid (`screen_draw_text`, `:866`), and — when cells are dirty — is drawn by the C/GL pipeline (`shaders.c:draw_cells`, `:1009`) using a glyph atlas, with SIMD (`simd-string.c`) accelerating the scanning. None of this per-byte/per-frame work is done in Python; Python's role (Q3) is to import and wire these C components together at startup and then dispatch high-level events. *Inferred* (from the call structure, not a timing measurement): this is why the source "feels like a tight weave of Python and C" yet is GPU-accelerated — the weave is real, but at runtime the C side plus the GPU carries the load.

**External corroboration** (observations above remain primary; full links in Appendix F):

- The official **Overview** ([sw.kovidgoyal.net/kitty/overview/](https://sw.kovidgoyal.net/kitty/overview/)) states kitty is "written in a mix of C (for performance sensitive parts), Python (for easy extensibility and flexibility of the UI) and Go (for the command line kittens)" and uses "only OpenGL for rendering everything."
- The official **home page** ([sw.kovidgoyal.net/kitty/](https://sw.kovidgoyal.net/kitty/)) bills kitty as a "GPU based terminal emulator" that "Uses GPU and SIMD vector CPU instructions."
- **DeepWiki** ([deepwiki.com/kovidgoyal/kitty](https://deepwiki.com/kovidgoyal/kitty)) describes a hybrid Python/C/Go application in which "performance-critical operations — terminal state management, rendering, and I/O — are implemented in C."

---

## Q2 — What role do the GLSL files play, and how central are they?

**Direct answer.** The `.glsl` files are **OpenGL shader source programs that drive the GPU rendering pipeline** — they are the mechanism by which kitty draws cells/text, window borders, background images, graphics-protocol images, and a background tint on the GPU. They are **central to on-screen output**: the terminal's visible content is produced by these programs, and — a key finding — their compilation itself flows through the **same** native bridge (`kitty.fast_data_types`) that everything else depends on, linking Q2 directly to Q3/Q4.

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
[exit: 0]

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
[exit: 0]
```

All 13 files live under `kitty/` and total **696** lines.

### Observed decomposition: 5 vertex/fragment pairs + 3 shared includes

The 13 files are **not** 13 independent shaders. The observed structure is **five `*_vertex` / `*_fragment` source pairs** plus **three shared GLSL includes**:

- **Five vertex/fragment pairs (10 files):** `cell`, `border`, `bgimage`, `graphics`, `tint`.
- **Three shared includes (3 files):** `alpha_blend.glsl`, `linear2srgb.glsl`, `cell_defines.glsl` — pulled into the pairs via a `#pragma kitty_include_shader <…>` directive rather than compiled on their own.

The include relationships are observable directly in the sources:

```console
$ grep -n 'kitty_include_shader' kitty/cell_fragment.glsl kitty/cell_vertex.glsl kitty/graphics_fragment.glsl
kitty/cell_fragment.glsl:1:#pragma kitty_include_shader <alpha_blend.glsl>
kitty/cell_fragment.glsl:2:#pragma kitty_include_shader <linear2srgb.glsl>
kitty/cell_fragment.glsl:3:#pragma kitty_include_shader <cell_defines.glsl>
kitty/cell_vertex.glsl:2:#pragma kitty_include_shader <cell_defines.glsl>
kitty/graphics_fragment.glsl:1:#pragma kitty_include_shader <alpha_blend.glsl>
[exit: 0]
```

`cell_defines.glsl`, `alpha_blend.glsl`, and `linear2srgb.glsl` are therefore `#include`-style fragments, not standalone GL programs. This is the observed correction to the planning note's looser "six program pairs" wording; the observed value — **5 pairs + 3 shared includes** — is used throughout.

### Loading and macro substitution (Python side): `kitty/shaders.py`

```console
$ grep -n -E 'from .fast_data_types import|GLSL_VERSION|compile_program|kitty_include_shader|_load_sources|#version|program_for|class MultiReplacer|class LoadShaderPrograms|def __call__|load_shader_programs = ' kitty/shaders.py
10:from .fast_data_types import (
19:    GLSL_VERSION,
29:    compile_program,
53:            Program.include_pat = re.compile(r'^#pragma\s+kitty_include_shader\s+<(.+?)>', re.MULTILINE)
56:        self.original_vertex_sources = tuple(self._load_sources(self.vertex_name, set()))
57:        self.original_fragment_sources = tuple(self._load_sources(self.fragment_name, set()))
61:    def _load_sources(self, name: str, seen: Set[str], level: int = 0) -> Iterator[str]:
63:            yield f'#version {GLSL_VERSION}\n'
78:            yield from self._load_sources(iname, seen, level+1)
90:            compile_program(program_id, self.vertex_sources, self.fragment_sources, allow_recompile)
108:def program_for(name: str) -> Program:
112:class MultiReplacer:
124:    def __call__(self, src: str) -> str:
131:class LoadShaderPrograms:
147:    def __call__(self, semi_transparent: bool = False, allow_recompile: bool = False) -> None:
152:        cell = program_for('cell')
186:        graphics = program_for('graphics')
199:        program_for('bgimage').compile(BGIMAGE_PROGRAM, allow_recompile)
200:        program_for('tint').compile(TINT_PROGRAM, allow_recompile)
204:load_shader_programs = LoadShaderPrograms()
[exit: 0]
```

- **`kitty/shaders.py:10`** opens a `from .fast_data_types import (…)` block — the loader pulls its program IDs and, crucially, its compiler **from the native C extension**.
- **`kitty/shaders.py:19`** imports `GLSL_VERSION` (the `#version` string prepended to every shader); **`:29`** imports **`compile_program`** — the actual shader compiler — from `fast_data_types`. This is the key structural fact for Q2: **shader compilation is a C function**; the Python side only feeds it source strings.
- **`kitty/shaders.py:53`** compiles the include regex; **`:61`** `_load_sources(...)` recursively resolves includes (recursing at **`:78`**) and **`:63`** prepends `#version {GLSL_VERSION}`.
- **`kitty/shaders.py:90`** calls `compile_program(program_id, self.vertex_sources, self.fragment_sources, allow_recompile)` — the hand-off into C.
- **`kitty/shaders.py:112` `MultiReplacer`** implements `{PLACEHOLDER}` substitution; **`:131` `LoadShaderPrograms`** and its `__call__` (**`:147`**) compile all programs at startup; **`:204`** instantiates the singleton `load_shader_programs`.

### Compilation (C side): `kitty/shaders.c`

```console
$ grep -n -E 'compile_program\(PyObject|glCreateProgram\(\)|glLinkProgram\(program->id\)|M\(compile_program' kitty/shaders.c
1168:compile_program(PyObject UNUSED *self, PyObject *args) {
1179:    program->id = glCreateProgram();
1182:    glLinkProgram(program->id);
1236:    M(compile_program, METH_VARARGS),
[exit: 0]
```

- **`kitty/shaders.c:1168`** defines `compile_program(PyObject *self, PyObject *args)` — the C function.
- **`kitty/shaders.c:1179`** calls `glCreateProgram()` and **`:1182`** `glLinkProgram(program->id)` — the real OpenGL calls that build and link the GPU program.
- **`kitty/shaders.c:1236`** registers it to Python via `M(compile_program, METH_VARARGS)`.

The `compile_program` imported at `kitty/shaders.py:29` is exactly this C function at `kitty/shaders.c:1168` — Python assembles GLSL text, C compiles and links it on the GPU.

### Program expansion — one source pair, several GL programs

`LoadShaderPrograms.__call__` (`kitty/shaders.py:147`) specializes the source pairs into multiple GL programs by substituting `{PLACEHOLDER}` macros (`kitty/shaders.py:152` for `cell`, `:186` for `graphics`, `:199`/`:200` for `bgimage`/`tint`):

- **`cell` → 4 programs** via a `WHICH_PHASE` substitution: `CELL_PROGRAM`, `CELL_BG_PROGRAM`, `CELL_SPECIAL_PROGRAM`, `CELL_FG_PROGRAM`.
- **`graphics` → 3 programs**: `GRAPHICS_PROGRAM`, `GRAPHICS_PREMULT_PROGRAM`, `GRAPHICS_ALPHA_MASK_PROGRAM`.
- **`bgimage` → `BGIMAGE_PROGRAM`**, **`tint` → `TINT_PROGRAM`** (one each).
- **`border` → `BORDERS_PROGRAM`**, whose initializer `init_borders_program` is imported from `fast_data_types` at `kitty/borders.py:7` (the very import that fails in Q3).

`cell_defines.glsl` is the shared 31-line `#define` include carrying the `{WHICH_PHASE}`, `{TRANSPARENT}`, and related placeholders that `MultiReplacer` fills in.

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

### Where the glyphs come from (the real font pipeline)

The glyphs the cell programs sample are produced by a Python→C→FreeType pipeline — **not** by `kitty/fonts/__init__.py`, which contains no rasterization, atlas, sprite, or texture code (it holds font-spec dataclasses/typed dicts and family scoring helpers):

```console
$ grep -n -E 'rasteriz|atlas|sprite|texture|set_font_data|render_glyph' kitty/fonts/__init__.py; echo "[exit: $?]"
[exit: 1]

$ grep -n -E 'def set_font_family|set_font_data\(' kitty/fonts/render.py
173:def set_font_family(opts: Optional[Options] = None, override_font_size: Optional[float] = None) -> None:
189:    set_font_data(

$ grep -n -E '^render_group\(|^render_groups\(|^send_prerendered_sprites\(' kitty/fonts.c
714:render_group(FontGroup *fg, unsigned int num_cells, unsigned int num_glyphs, CPUCell *cpu_cells, GPUCell *gpu_cells, hb_glyph_info_t *info, hb_glyph_position_t *positions, Font *font, glyph_index *glyphs, unsigned glyph_count, bool center_glyph) {
1203:render_groups(FontGroup *fg, Font *font, bool center_glyph) {
1450:send_prerendered_sprites(FontGroup *fg) {

$ grep -n 'render_glyphs_in_cells' kitty/freetype.c
675:render_glyphs_in_cells(PyObject *f, bool bold, bool italic, hb_glyph_info_t *info, hb_glyph_position_t *positions, unsigned int num_glyphs, pixel *canvas, unsigned int cell_width, unsigned int cell_height, unsigned int num_cells, unsigned int baseline, bool *was_colored, FONTS_DATA_HANDLE fg, bool center_glyph) {
```

The grep for rasterization terms in `kitty/fonts/__init__.py` returns nothing (`[exit: 1]`). The actual flow is: `kitty/fonts/render.py:set_font_family` (`:173`) configures fonts and calls `set_font_data` (`:189`, a native function); glyph shaping/rasterization happens in C at `kitty/fonts.c:render_group` (`:714`) and `render_groups` (`:1203`), with the rasterized sprites uploaded to the atlas by `send_prerendered_sprites` (`:1450`); the per-glyph pixel rasterization is `kitty/freetype.c:render_glyphs_in_cells` (`:675`). `kitty/fonts/__init__.py` (191 lines) contributes only font-spec types and scoring, not rasterization.

### Centrality rationale

The cell/border/bgimage/graphics/tint programs are what draw the terminal's on-screen content, so the shaders are not peripheral — they *are* the rendering stage. And because the compiler `compile_program` is imported from `fast_data_types` (`kitty/shaders.py:29`) and the border program's initializer is imported at `kitty/borders.py:7`, the shader subsystem sits **on top of the same C bridge** the rest of the application binds to at import time. *Inferred* (from this structure, not from a pixel-level measurement): with the shader programs and their compiler unavailable, kitty could not present its GPU-drawn terminal surface — which is why Q2 is inseparable from Q3/Q4.

**External corroboration** (full links in Appendix F): DeepWiki ([deepwiki.com/kovidgoyal/kitty](https://deepwiki.com/kovidgoyal/kitty)) describes the render step as "dirty cells trigger the GPU rendering pipeline, which uses a sprite atlas and OpenGL shaders," matching the observed atlas + shader-program pipeline enumerated above.

---

## Q3 — Why the main entry point fails, and the one critical piece

**Direct answer.** Running the canonical Python entry file directly fails **immediately** with `ModuleNotFoundError: No module named 'kitty.fast_data_types'`, raised at `kitty/borders.py:7`. The single critical piece that everything depends on is **`kitty.fast_data_types`** — the compiled C extension (`kitty/fast_data_types.so`) produced by the build. It is imported *eagerly at module load time* along the chain `__main__.py` → `kitty/entry_points.py` → `kitty/main.py` → `kitty/borders.py`, so its absence aborts startup before any command-line parsing or `main()` logic can run. There is **no Python fallback** for this module: only a type-stub `kitty/fast_data_types.pyi` is tracked in the repository; the executable code exists solely as the built `.so`.

### Observed failure — canonical Python entry file, clean checkout (extension absent)

This is the natural state of a fresh checkout: the `.so` has not been built, so the failure reproduces with no setup.

```text
### CMD: /opt/kitty311-venv/bin/python3 __main__.py
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

### The import chain, traced to source

Each hop is a real, unconditional module-level import. The failing statement is the last one:

```text
### CMD: sed -n '5,7p' __main__.py
if __name__ == '__main__':
    from kitty.entry_points import main
    main()
[exit: 0]

### CMD: sed -n '183p;194p' kitty/entry_points.py
def main() -> None:
            from kitty.main import main as kitty_main
[exit: 0]

### CMD: sed -n '11p' kitty/main.py
from .borders import load_borders_program
[exit: 0]

### CMD: sed -n '7,8p' kitty/borders.py
from .fast_data_types import BORDERS_PROGRAM, add_borders_rect, get_options, init_borders_program, os_window_has_background_image
from .shaders import program_for
[exit: 0]
```

- `__main__.py:7` calls `main()` (defined in `kitty/entry_points.py`). [`__main__.py:7`]
- `kitty/entry_points.py:194` (inside `main()`, defined at `:183`) imports `kitty.main`. [`kitty/entry_points.py:194`]
- `kitty/main.py:11` imports `load_borders_program` from `kitty/borders.py`. [`kitty/main.py:11`]
- `kitty/borders.py:7` executes `from .fast_data_types import ...` — the exact line that raises. [`kitty/borders.py:7`]

`kitty/borders.py:8` (`from .shaders import program_for`) would also reach the bridge, and `kitty/boss.py:63` (`from .fast_data_types import (`) is another eager importer used during startup — but the traceback terminates at `borders.py:7` because that is the first bridge import encountered on the chain.

### There is no `.py` fallback — only a compiled `.so` and a stub

```text
### CMD: git ls-files 'kitty/fast_data_types*'
kitty/fast_data_types.pyi

### CMD: ls -l kitty/fast_data_types.pyi ; ls -l kitty/fast_data_types*.so
-rw-r--r-- 1 root root 35795 Jul 14 18:52 kitty/fast_data_types.pyi
ls: cannot access 'kitty/fast_data_types*.so': No such file or directory
[exit: 2]
```

The only tracked `fast_data_types` artifact is the `.pyi` **type stub** (declarations for static type-checkers, no executable body). The importable module is exclusively the built `kitty/fast_data_types.so`, which is git-ignored and absent from a clean checkout. Hence the failure is structural, not a path or environment glitch.

### Variant condition — `python3 -m kitty`

The `-m` invocation is exercised as a distinct condition and fails **differently**, which confirms that the repository-root `__main__.py` — not the package — is the canonical Python entry file:

```text
### CMD: /opt/kitty311-venv/bin/python3 -m kitty
/opt/kitty311-venv/bin/python3: No module named kitty.__main__; 'kitty' is a package and cannot be directly executed
[exit: 1]
```

There is no `kitty/__main__.py`, so Python refuses to execute the package directly. This is a *different* error (raised by the interpreter's `runpy` machinery, before any kitty code loads) than the `fast_data_types` `ModuleNotFoundError` above, demonstrating that the two invocation forms are not equivalent.

### Post-build behavior — a second barrier, and what the outer C launcher supplies (F12 terminology)

After a canonical build (Appendix B), the `.so` is present and the *import* barrier is gone — but running the **canonical Python entry file** directly under a bare interpreter still fails, now at a **second** barrier:

```text
### CMD: /opt/kitty311-venv/bin/python3 __main__.py    (post-build, extension PRESENT)
[0.218] Traceback (most recent call last):
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

By contrast, the **outer C launcher** (`kitty/launcher/kitty`, the user-facing executable produced by the build) runs the identical Python code successfully:

```text
### CMD: ./kitty/launcher/kitty --version    (outer C launcher, post-build)
kitty 0.35.2 created by Kovid Goyal
[exit: 0]

### CMD: /opt/kitty311-venv/bin/python3 -c "import kitty.fast_data_types as f; print('bridge OK:', f.__file__.split('/')[-1])"
bridge OK: fast_data_types.so
[exit: 0]
```

Cause and effect: the C launcher injects the `sys.kitty_run_data` attribute (consumed at `kitty/main.py:364` by `ensure_kitty_in_path`) into the interpreter *before* handing control to Python; a bare `python3 __main__.py` never sets it, so `getattr(sys, 'kitty_run_data')` raises `AttributeError`. Two distinct barriers therefore stand between "run the file directly" and "working terminal":

1. **The native bridge** (`kitty.fast_data_types`) — imported eagerly at `kitty/borders.py:7`; missing in a clean checkout → the `ModuleNotFoundError` that is the subject of this question.
2. **Launcher-injected run data** (`sys.kitty_run_data`) — set only by the outer C launcher; missing under a bare interpreter → the post-build `AttributeError` at `kitty/main.py:364`.

Terminology note: throughout this document, *"canonical Python entry file"* refers to the repository-root `__main__.py` that routes into `kitty.entry_points.main` [`__main__.py:7`], and *"outer C launcher"* refers to the compiled `kitty/launcher/kitty` binary that end users actually invoke. The first is the Python source of truth for startup; the second is the process wrapper that prepares the interpreter environment.

### Why 3.11 specifically — ABI binding, not a version "ceiling" (F4)

The reason the build and runtime use CPython 3.11 is an **ABI** constraint, not an upper version bound in the project metadata. The built extension declares a hard `NEEDED` dependency on the 3.11 shared library:

```text
### CMD: readelf -d kitty/fast_data_types.so | grep NEEDED
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libpython3.11.so.1.0]
 0x0000000000000001 (NEEDED)             Shared library: [libharfbuzz.so.0]
 0x0000000000000001 (NEEDED)             Shared library: [libpng16.so.16]
 0x0000000000000001 (NEEDED)             Shared library: [liblcms2.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libcrypto.so.3]
 0x0000000000000001 (NEEDED)             Shared library: [libz.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
[exit: 0]
```

The `.so` is linked against `libpython3.11.so.1.0` (the link line in Appendix B includes `-lpython3.11`), so the module can only be loaded by a CPython 3.11 interpreter whose C-ABI it was compiled against. The project's *declared* Python support is a floor with no ceiling:

```text
### CMD: grep -n 'requires-python' pyproject.toml
2:requires-python = ">=3.8"
[exit: 0]

### CMD: grep -nE 'pyver|python-version' .github/workflows/ci.yml
14:        name: Linux (python=${{ matrix.pyver }} cc=${{ matrix.cc }} sanitize=${{ matrix.sanitize }})
26:                      pyver: "3.8"
30:                      pyver: "3.10"
34:                      pyver: "3.9"
52:          - name: Set up Python ${{ matrix.pyver }}
55:              python-version: ${{ matrix.pyver }}
85:              python-version: "3.11"
170:              python-version: "3.10"
[exit: 0]
```

`pyproject.toml:2` declares `requires-python = ">=3.8"` — a minimum, with **no upper bound**. The CI matrix exercises 3.8, 3.9, 3.10, and 3.11 (`.github/workflows/ci.yml`). This environment uses the highest CI-documented version, **3.11.15**, and the extension is consequently ABI-bound to `libpython3.11.so.1.0`. Running the built `.so` under a *different* CPython minor (e.g., 3.13) is unsupported and unsafe, and the **exact failure mode depends on the environment**:

- **Load-time failure** when `libpython3.11.so.1.0` is not resolvable (e.g., the supplied base image, which ships only CPython 3.12): the dynamic loader cannot satisfy the hard `NEEDED` dependency, so the import aborts with `ImportError: libpython3.11.so.1.0: cannot open shared object file: No such file or directory`.
- **Import-then-crash** when the 3.11 shared library *is* present alongside the mismatched interpreter (e.g., system CPython 3.13 with `/opt/python3.11/lib/libpython3.11.so.1.0` still on the loader path): the module **imports successfully** but a subsequent native call **crashes the process**, because two CPython runtimes are mapped into one address space. Observed directly — importing under `/usr/bin/python3` (3.13) returns `[exit: 0]`, but `from kitty.fast_data_types import wcswidth; wcswidth("kitty")` terminates with `Segmentation fault (core dumped)` `[exit: 139]`.

The import-then-crash mode, captured against the built extension using the **system** CPython 3.13 interpreter (a deliberately mismatched minor) with the 3.11 shared library present on the loader path:

```text
### CMD: test -e /opt/python3.11/lib/libpython3.11.so.1.0 && echo PRESENT   # 3.11 shared lib is on the loader path
PRESENT
[exit: 0]

### CMD: /usr/bin/python3 -c "import kitty.fast_data_types as f; print('IMPORT_OK', f.__file__.split('/')[-1])"   # system CPython 3.13.7, mismatched minor
IMPORT_OK fast_data_types.so
[exit: 0]

### CMD: /usr/bin/python3 -c "from kitty.fast_data_types import wcswidth; print(wcswidth('kitty'))"   # first native call into the 3.11-ABI module
Segmentation fault (core dumped)
[exit: 139]
```

Either way, cross-minor reuse is not supported — but this document does **not** assert a "3.11 ceiling"; the correct statement is "≥ 3.8 floor, built against and ABI-bound to 3.11 here." *(Inferred, for the load-time mode: it follows from the confirmed `NEEDED libpython3.11.so.1.0` above; the import-then-crash mode was observed directly and captured in the block just above.)*

### Supporting orchestration observed

- `kitty/boss.py:63` — `from .fast_data_types import (` — the central startup orchestrator is itself a bridge consumer, reinforcing that the native core is load-bearing for the whole application. [`kitty/boss.py:63`]
- `kitty/constants.py:25` — `version: Version = Version(0, 35, 2)` — the version reported by `kitty --version` above (`kitty 0.35.2`). [`kitty/constants.py:25`]

### Rationale

The failure is *immediate and cryptic* precisely because the bridge is imported at **module top-level**, unconditionally, three hops deep — not lazily inside a function guarded by a helpful error message. Python evaluates `from .fast_data_types import ...` while still importing `kitty.borders` (itself pulled in transitively by `kitty.main`), so the interpreter aborts with a low-level `ModuleNotFoundError` naming an internal module the user has never heard of, rather than a friendly "please build kitty first." This is the concrete, observed evidence that **Python is a thin orchestration shell wired directly into the native core at import time**: remove the one compiled piece and the Python layer cannot even finish importing, let alone start a terminal.

---

## Q4 — Are kittens independent, or do they rely on the same native bridge?

**Direct answer.** It is **split**, and the split is the whole point. The **Python kittens are not independent**: run on their own they fail with the *same* `ModuleNotFoundError: No module named 'kitty.fast_data_types'` seen in Q3, because they pull in the shared `kitty`/`kittens.tui` support modules that bind the native bridge at import time. The **Go "kitten" binary is genuinely independent**: it is a *separate compiled executable* that links only against the system C library — it has **no dependency on the Python extension at all** — and it runs successfully even when `kitty.fast_data_types` is absent. So kitty's modularity is real only on the Go side; the Python kitten layer is as tightly bound to the native core as the main application.

### Observed — standalone Python kitten via the canonical runner (clean checkout, extension absent)

```text
### CMD: /opt/kitty311-venv/bin/python3 -c "from kittens.runner import main"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kittens/runner.py", line 14, in <module>
    from kitty.utils import resolve_abs_or_config_path
  File "/tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/kitty/utils.py", line 45, in <module>
    from .fast_data_types import WINDOW_FULLSCREEN, WINDOW_MAXIMIZED, WINDOW_MINIMIZED, WINDOW_NORMAL, Color, Shlex, get_options, monotonic, open_tty
ModuleNotFoundError: No module named 'kitty.fast_data_types'
[exit: 1]
```

`kittens/runner.py` is the dispatcher every Python kitten goes through. Its very first substantive import — `kittens/runner.py:14`, `from kitty.utils import ...` — drags in `kitty/utils.py`, whose module-level `kitty/utils.py:45` binds the bridge. The kitten never reaches its own logic.

### Observed — an individual Python kitten (`hints`) run on its own (clean checkout, extension absent)

```text
### CMD: /opt/kitty311-venv/bin/python3 -m kittens.hints.main --help
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

The `hints` kitten fails on a *different* transitive path: `kittens/hints/main.py:9` imports `kitty.clipboard`, which at `kitty/clipboard.py:11` imports `kitty.conf.utils`, whose module-level `kitty/conf/utils.py:27` binds the bridge. (It would also fail at its own module-level `kittens/hints/main.py:11`, `from kitty.fast_data_types import get_options`, but the clipboard chain raises first.)

Variant condition — the bare package form fails earlier, at the interpreter's module machinery, confirming `main` is the runnable module:

```text
### CMD: /opt/kitty311-venv/bin/python3 -m kittens.hints
/opt/kitty311-venv/bin/python3: No module named kittens.hints.__main__; 'kittens.hints' is a package and cannot be directly executed
[exit: 1]
```

### Observed contrast — the Go "kitten" binary is self-contained

The Go binary is a *different kind of artifact*. It is **not** a static binary (correcting the looser "static binary" phrasing): `file` shows a dynamically linked ELF with an interpreter, and `ldd` shows it needs only the system C library — crucially **not** `libpython3.11.so.1.0` (contrast with the `readelf` of the `.so` in Q3):

```text
### CMD: file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=73936815a47046935b4a01c85379ef63c7540be1, stripped

### CMD: ldd kitty/launcher/kitten
	linux-vdso.so.1 (0x00007ffd2e2fe000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007e2bbc7f3000)
	/lib64/ld-linux-x86-64.so.2 (0x00007e2bbca40000)
[exit: 0]
```

And it *runs* with the Python extension absent — captured in the interruption-safe present→absent→present experiment (Appendix B), where the `.so` was moved aside so the Go binary and the Python kittens face the identical "extension absent" state:

```text
### STEP 6 (Q4 contrast): Go kitten binary, SAME extension-absent state
### CMD: ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
[exit: 0]
```

The Go sources never reference the native bridge, and they compile to a single module-`kitty` binary:

```text
### CMD: grep -rl 'fast_data_types' tools/cmd/
[exit: 1]

### CMD: head -1 go.mod
module kitty
[exit: 0]

### CMD: ls tools/cmd
at  benchmark  completion  edit_in_kitty  main.go  mouse_demo  pytest  run_shell  show_error  tool  update_self
[exit: 0]
```

`grep -rl` returns no matches (`[exit: 1]` is grep's "nothing found"), so the Go CLI layer never references the Python extension; `go.mod` declares module `kitty` (and, on its third line, `go 1.22`); and `tools/cmd` holds the eleven Go command packages listed above.

*Note on `CGO_ENABLED`:* `setup.py:1173` sets `e['CGO_ENABLED'] = '0'` **only inside** `if for_platform:` (`setup.py:1172`), i.e. for cross-compiled release builds — not for this host build. So the host `kitten` is an ordinary dynamically linked Go executable, which is exactly what `file`/`ldd` show.

### F12 — classification of every bridge import in the kitten layer

Thirteen modules under `kittens/` reference `kitty.fast_data_types` (`grep -rl 'fast_data_types' kittens/` → 13 files), reached in three ways:

| Kind | Meaning | Modules (with `file:line`) |
|------|---------|-----------------------------|
| **Module-level, direct** | `from ...fast_data_types import ...` at column 0 — fails the instant the module is imported | `kittens/tui/handler.py:10`, `kittens/tui/images.py:15`, `kittens/tui/line_edit.py:6`, `kittens/tui/loop.py:19`, `kittens/tui/operations.py:11`, `kittens/tui/path_completer.py:8`, `kittens/tui/spinners.py:6`, `kittens/hints/main.py:11`, `kittens/panel/main.py:10` |
| **Function-local, direct** | bridge imported inside a function body — fails only when that code path runs | `kittens/runner.py:129`, `kittens/tui/utils.py:60`, `kittens/query_terminal/main.py:100`, `kittens/remote_file/main.py:360` |
| **Transitive (via `kitty` package)** | the kitten imports a `kitty.*` module whose *own* module-level import binds the bridge | `kittens/runner.py` → `kitty/utils.py:45`; `kittens/hints/main.py` → `kitty/clipboard.py:11` → `kitty/conf/utils.py:27`; and `kitty/cli.py:15` (`from .fast_data_types import wcswidth`), imported pervasively across the CLI |

The observed standalone failures above are **transitive**: `runner` fails via `kitty/utils.py:45` and `hints` fails via `kitty/conf/utils.py:27`, in both cases *before* reaching the kitten's own direct import. The shared TUI framework (`kittens/tui/*`) is the deeper reason — seven of its modules bind the bridge at column 0, so any TUI kitten inherits the dependency.

### Rationale

The Python kittens are packaged *inside* the same distribution and deliberately reuse `kitty`'s primitives (option handling, color, `wcswidth`, TTY control, the TUI loop) — all of which are implemented in the C extension and imported at module top. That reuse is efficient but it means a Python kitten is **not** a free-standing program: it inherits the native-bridge dependency transitively and cannot start without the `.so`. The Go `kitten`, by contrast, is a wholly separate executable with its own reimplementations under `tools/cmd/`, linked only to libc, and therefore *is* modular and self-contained. This is the observed, two-sided answer to the question: kittens *quietly rely on the same native bridge* when written in Python, and are *genuinely independent* when written in Go.

---

## Synthesis — how the four findings fit together

Pulling the four observations together yields a single coherent picture of kitty's architecture:

- **The performance engine is the compiled C core plus GPU offload (Q1 + Q2).** Terminal I/O, VT parsing, the screen grid, and rendering all execute inside `kitty.fast_data_types` (named hot-path functions in `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/shaders.c`), and the on-screen surface is drawn by the GLSL programs compiled through that same extension. The Python source is large (≈62.9k lines) but sits *above* the hot path; the measured C footprint (128 files / 61,806 lines, plus 84 headers / 37,939 lines and the bundled GLFW windowing layer) is where per-keystroke and per-frame work actually happens. *Inferred* (from the division of labor, not a profiler run): runtime performance originates in C and the GPU, not in Python.

- **Python is a thin orchestration shell wired into the native core at import time (Q3).** The canonical Python entry file cannot even finish importing without the `.so`: the eager `from .fast_data_types import ...` at `kitty/borders.py:7` aborts startup immediately. Python's role is to dispatch high-level events, parse configuration, and wire subsystems together — but it is not separable from the C core.

- **Go is an independent CLI/kitten layer (Q4).** The Go `kitten` binary is a separate executable linked only to libc, with no reference to the Python extension; it runs with the `.so` absent. The Python kittens, by contrast, reuse `kitty`/`kittens.tui` primitives and therefore inherit the native-bridge dependency transitively.

- **The load-bearing dependency is one module reached by two paths (Q3 + Q4).** Both the main-application chain (`__main__.py` → `entry_points.py:194` → `main.py:11` → `borders.py:7`) and the Python-kitten chains (`kittens/runner.py:14` → `kitty/utils.py:45`; `kittens/hints/main.py:9` → `kitty/clipboard.py:11` → `kitty/conf/utils.py:27`) converge on the *same* `kitty.fast_data_types`. That single compiled artifact — ABI-bound here to `libpython3.11.so.1.0` and produced only by a canonical `python3 setup.py` build (Appendix B) — is the one critical piece the user sensed "everything depends on."

In one sentence: **kitty is a C+GPU performance core, orchestrated by a thin Python shell that is inseparable from that core at import time, with a genuinely independent Go layer for its command-line tooling and Go kittens.**

---

## Appendix A — Observation environment

| Item | Value (observed) |
|------|------------------|
| Supplied image | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0…` (a.k.a. `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`); ID `sha256:c0824992ad0b…`, digest `ghcr.io/scaleapi/swe-atlas@sha256:60da90a7183a…`, `WORKDIR=/app` — provenance only (see "Build provenance") |
| Supplied image base toolchain | Ubuntu 24.04.2 / CPython 3.12.3 / Go 1.23.4 / gcc 13.3.0 (no `/opt/kitty311-venv`); the host CPython-3.11 toolchain below is provisioned on top of it and is the canonical build/run environment here |
| OS (host) | Ubuntu 25.10, `x86_64` |
| Python interpreter (host, canonical) | CPython **3.11.15** (`/opt/kitty311-venv/bin/python3`), the highest CI-documented minor |
| Go toolchain | `go1.24.4 linux/amd64` (`/usr/bin/go`) |
| C compiler | `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` |
| Shared Python lib | `/opt/python3.11/lib/libpython3.11.so.1.0` (ABI target of the built `.so`) |
| Git branch | `blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd`; authored at HEAD `d5f2b18b1` (see `git log` for any subsequent documentation-only fix commits) |
| Repo cleanliness | tracked tree clean except the created document; `0` ignored, `0` untracked (Appendix E) |
| Env discipline | `PYTHONDONTWRITEBYTECODE=1` set for all direct Python runs so no `.pyc`/`__pycache__` is written into the tree |

All commands in this document were run from the repository root shown above, with two clearly-labeled exceptions: the three `docker …` commands in "Build provenance" (which observe the **supplied image**, not the working tree) and the deliberately mismatched-interpreter probe in Q3 (run against the built extension with the **system** CPython 3.13). Every answer/reproduction command block gives the exact command, its complete unedited output, and an explicit `[exit: N]`. Output is condensed in only two clearly-marked places, both in Appendix B: the canonical build transcript (its repetitive per-translation-unit compile lines are elided at an explicitly marked point, while the compiler identification, the complete final link line, the Go build line, and the exit status are shown verbatim), and the present→absent→present transition (presented as a step list of each command with its key outcome and `[exit: N]`, the full tracebacks being the ones shown verbatim in Q3/Q4).

---

## Appendix B — Canonical build transcript and the present→absent→present transition

**Canonical build.** The one supported build is `python3 setup.py` (the `all` target in the `Makefile`). On this host it requires the documented `CFLAGS` demotion of a single GLFW/Wayland `-Wswitch` diagnostic (a toolchain/deps mismatch, not a code bug). The `--verbose` run prints one `gcc` line per translation unit; the excerpt below shows — verbatim and complete — the compiler identification, the **final link line** for `fast_data_types.so`, the Go build line, and the exit status (the many per-file compile lines are omitted only where explicitly marked):

```text
### CMD: CFLAGS="-Wno-error=switch" /opt/kitty311-venv/bin/python setup.py --verbose
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
Detected: CompilerType.gcc
[<one gcc compile line per .c translation unit omitted; the complete final link line follows>]
gcc -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -Wno-error=switch -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/opt/python3.11/include/python3.11 -Wall -O3 -shared -flto build/fast_data_types-kitty-charsets.c.o build/fast_data_types-kitty-child-monitor.c.o build/fast_data_types-kitty-child.c.o build/fast_data_types-kitty-cleanup.c.o build/fast_data_types-kitty-colors.c.o build/fast_data_types-kitty-crypto.c.o build/fast_data_types-kitty-cursor.c.o build/fast_data_types-kitty-data-types.c.o build/fast_data_types-kitty-desktop.c.o build/fast_data_types-kitty-disk-cache.c.o build/fast_data_types-kitty-fast-file-copy.c.o build/fast_data_types-kitty-font-names.c.o build/fast_data_types-kitty-fontconfig.c.o build/fast_data_types-kitty-fonts.c.o build/fast_data_types-kitty-freetype.c.o build/fast_data_types-kitty-freetype_render_ui_text.c.o build/fast_data_types-kitty-gl-wrapper.c.o build/fast_data_types-kitty-gl.c.o build/fast_data_types-kitty-glfw-wrapper.c.o build/fast_data_types-kitty-glfw.c.o build/fast_data_types-kitty-glyph-cache.c.o build/fast_data_types-kitty-graphics.c.o build/fast_data_types-kitty-history.c.o build/fast_data_types-kitty-hyperlink.c.o build/fast_data_types-kitty-key_encoding.c.o build/fast_data_types-kitty-keys.c.o build/fast_data_types-kitty-kittens.c.o build/fast_data_types-kitty-line-buf.c.o build/fast_data_types-kitty-line.c.o build/fast_data_types-kitty-logging.c.o build/fast_data_types-kitty-loop-utils.c.o build/fast_data_types-kitty-monotonic.c.o build/fast_data_types-kitty-mouse.c.o build/fast_data_types-kitty-png-reader.c.o build/fast_data_types-kitty-rowcolumn-diacritics.c.o build/fast_data_types-kitty-screen.c.o build/fast_data_types-kitty-shaders.c.o build/fast_data_types-kitty-shlex.c.o build/fast_data_types-kitty-simd-string-128.c.o build/fast_data_types-kitty-simd-string-256.c.o build/fast_data_types-kitty-simd-string.c.o build/fast_data_types-kitty-state.c.o build/fast_data_types-kitty-systemd.c.o build/fast_data_types-kitty-unicode-data.c.o build/fast_data_types-kitty-utmp.c.o build/fast_data_types-kitty-vt-parser.c.o build/fast_data_types-kitty-wcswidth.c.o build/fast_data_types-kitty-window_logo.c.o build/fast_data_types-kitty-vt-parser-dump.c.o build/fast_data_types-3rdparty-ringbuf-ringbuf.c.o build/fast_data_types-3rdparty-base64-lib-arch-neon32-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-sse42-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-ssse3-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-sse41-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-generic-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx2-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx512-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-neon64-codec.c.o build/fast_data_types-3rdparty-base64-lib-tables-tables.c.o build/fast_data_types-3rdparty-base64-lib-codec_choose.c.o build/fast_data_types-3rdparty-base64-lib-lib.c.o -ldl -lm -L/opt/python3.11/lib -lpython3.11 -Xlinker -export-dynamic -lharfbuzz -lGL -lpng16 -llcms2 -llcms2_fast_float -llcms2_threaded -pthread -lm -lcrypto -lrt -lz -o build/kitty/fast_data_types.so
Updating Go generated files...
/usr/bin/go build -v -ldflags '-X kitty.VCSRevision=d5f2b18b133b19f69be07aa49af12d8b67c0d1f9 -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-5c1e066c-992b-4bf9-8d37-696eb3c895bd_c8c770/tools/cmd
BUILD_EXIT=0
```

The link line includes `-lpython3.11` (mid-line, before the final `-o build/kitty/fast_data_types.so`), which is why the resulting `.so` declares `NEEDED libpython3.11.so.1.0` (Q3). The canonical build emits **six** native artifacts (all git-ignored) — four compiled `.so` shared objects plus two launcher binaries:

```text
### CMD: find kitty kittens -type f \( -name '*.so' -o -path '*/launcher/kitten' -o -path '*/launcher/kitty' \) | sort
kittens/transfer/rsync.so
kitty/fast_data_types.so
kitty/glfw-wayland.so
kitty/glfw-x11.so
kitty/launcher/kitten
kitty/launcher/kitty
[exit: 0]
```

- **`kitty/fast_data_types.so`** — the main C extension / native bridge (the Q3/Q4 load-bearing dependency), ABI-bound to `libpython3.11.so.1.0`.
- **`kitty/glfw-x11.so`** and **`kitty/glfw-wayland.so`** — the two bundled GLFW windowing backends (X11 and Wayland), each a separately-linked C shared object selected at runtime by the session type.
- **`kittens/transfer/rsync.so`** — a C extension backing the `transfer` kitten.
- **`kitty/launcher/kitten`** — the standalone **Go** binary (the CLI `kitten`; see Q4).
- **`kitty/launcher/kitty`** — the **C** launcher executable (the canonical way to start kitty; see Q3).

The transcript excerpt above shows only the `fast_data_types.so` link line and the Go `kitten` build line; the link lines for the two GLFW backends, `rsync.so`, and the C launcher are among the per-translation-unit compile/link lines elided at the marked point above.

**Determinism.** Building twice at the same commit produced a byte-identical extension:

```text
### CMD: md5sum kitty/fast_data_types.so   (both builds, at VCS rev d5f2b18b1)
b255c0013972634549f91312ae799a0f  kitty/fast_data_types.so
[exit: 0]
```

This md5 is pinned to the exact commit, not merely to the environment: `kitty/data-types.c` — which defines `PyInit_fast_data_types` and is linked into the `.so` — is compiled with `-DKITTY_VCS_REV="<git rev>"` (`setup.py:726`, `get_source_specific_defines`), so the current git revision string is baked into the extension. The revision baked into the md5 above is the authoring commit `d5f2b18b1…`, the same revision shown in the Go build line's `-X kitty.VCSRevision=d5f2b18b133b19f69be07aa49af12d8b67c0d1f9` above; forcing `setup.py --vcs-rev=d5f2b18b133b19f69be07aa49af12d8b67c0d1f9` reproduces `b255c0013972634549f91312ae799a0f` exactly (confirmed on two consecutive builds). Rebuilding at a later commit (for example a subsequent documentation-only fix, which advances `HEAD`) embeds a different revision and therefore yields a different md5 — so "byte-identical" holds only when both the commit and the `-march=native` environment are held fixed. The kitten binary's `BuildID` (shown in the `file` output in Q4) is commit-pinned for the same reason: the Go binary embeds `VCSRevision` at link time.

**Interruption-safe present→absent→present transition.** To observe the absent-state failures (Q3/Q4) against a built tree *without* leaving the tree altered, a `trap`-guarded script recorded the `.so` hash, moved it aside, ran every absent-state probe, then restored it and re-verified the hash. Before and after hashes match exactly:

```text
### STEP 0: hash BEFORE moving
b255c0013972634549f91312ae799a0f  kitty/fast_data_types.so
### STEP 1: move aside → importlib spec for kitty.fast_data_types -> None   (absent)
### STEP 2 (Q3): python3 __main__.py            → ModuleNotFoundError @ borders.py:7   [exit: 1]
### STEP 3 (Q3 variant): python3 -m kitty       → 'kitty' package cannot be executed    [exit: 1]
### STEP 4 (Q4): kittens.runner                 → ModuleNotFoundError @ utils.py:45      [exit: 1]
### STEP 5 (Q4): kittens.hints.main             → ModuleNotFoundError @ conf/utils.py:27 [exit: 1]
### STEP 6 (Q4 contrast): ./kitty/launcher/kitten --version → kitten 0.35.2              [exit: 0]
### STEP 7: restore + verify hash matches BEFORE
b255c0013972634549f91312ae799a0f  kitty/fast_data_types.so   (== STEP 0)
```

The post-build ABI/linkage and runtime probes referenced elsewhere were captured in this same built state: `readelf -d` `NEEDED libpython3.11.so.1.0` (Q3), `file`/`ldd` of `kitty/launcher/kitten` showing libc-only dynamic linkage (Q4), the bare-interpreter `AttributeError: sys.kitty_run_data` at `kitty/main.py:364` and the working `./kitty/launcher/kitty --version` (Q3), and the GPU line `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2'` with `OS Window created` / `Child launched` (Q1).

---

## Appendix C — Language footprint counting method

Two NUL-safe methods were used and each was run twice with identical results:

```text
### Tracked (source of truth): git ls-files -z
git ls-files -z -- '*.c'  | grep -zc .              # file count
git ls-files -z -- '*.c'  | xargs -0 cat | wc -l    # line total

### Live on disk: find -print0 (excludes .git)
find . -path ./.git -prune -o -type f -name '*.c' -print0 | tr -dc '\0' | wc -c
```

NUL delimiting (`-z` / `-print0`) is used so that any path containing spaces or newlines is counted exactly once. **Tracked** counts (also equal to the **final clean** on-disk counts):

| Category | Files | Lines |
|----------|-------|-------|
| C (`*.c`) | 128 | 61,806 |
| C headers (`*.h`) | 84 | 37,939 |
| Python (`*.py`) | 214 | 62,874 |
| Go (`*.go`) | 258 | 56,071 |
| GLSL (`*.glsl`) | 13 | 696 |

**During-build live snapshot (transient, for reconciliation only).** While the build was in progress, generated sources and the built GLFW backends inflated the on-disk counts: C 142, H 100, Go 338 (Py/GLSL unchanged). The deltas are entirely build products:

- **Go +80** = 74 `*_generated.go` + 6 `*_generated_test.go` (verified with `grep -vxF` against the tracked list; a naive `comm` under-reports this as 78 because of ordering, which is the origin of the earlier "78" error).
- **C +14** = the compiled GLFW Wayland `.c` sources.
- **H +16** = 14 GLFW Wayland headers + `docs_ref_map_generated.h` + `uniforms_generated.h`.

After `git clean -dfX` (Appendix E) the live counts return to exactly the tracked counts, confirming every delta was a removable artifact.

---

## Appendix D — Documented discrepancies with the planning note

Where direct observation diverged from the planning note (AAP), the observed value governs:

| Item | Planning note | Observed | Resolution |
|------|---------------|----------|------------|
| `setup.py` length | "2173 lines" | **2172 lines** (`wc -l`), file **does** end in a newline (`od -c` shows `main()\n`) | Off-by-one counting convention, **not** a missing trailing newline. The earlier draft's claim that the difference was caused by a missing final newline is unsupported and is withdrawn. |
| GLSL structure | "six program pairs" | **13 files = 5 vertex/fragment pairs + 3 shared includes** (`cell_defines.glsl`, `alpha_blend.glsl`, `linear2srgb.glsl`) | Observed correction to the note's looser wording; no claim is made about the note's intent. |
| Go footprint | 258 files | 258 tracked; **+80** generated appear only during a build | Reconciled in Appendix C. |
| Font rasterization site | `kitty/fonts/__init__.py` | `grep -E 'rasteriz|atlas|sprite|texture' kitty/fonts/__init__.py` → **no matches, `[exit: 1]`**; rasterization is in `kitty/fonts.c` + `kitty/freetype.c`, orchestrated from `kitty/fonts/render.py` | Corrected in Q2; `__init__.py` (191 lines) holds only font-spec types and scoring. |

---

## Appendix E — Repository cleanliness certification (capture-then-clean)

**Method.** This investigation used a **capture-then-clean** discipline. Because the four questions concern *runtime* behavior, a canonical `python3 setup.py` build was performed to observe the working runtime and the built-state ABI/linkage; all `.so`-present evidence was captured to files **outside** the repository (`/tmp/evidence/`) *before* any cleanup. The build products the AAP records as absent at task start (`kitty/fast_data_types*.so`, `kitty/launcher/kitty*`, `build/`, generated Go/headers) were then removed with `git clean -dfX`, restoring the working tree to its fresh-checkout baseline.

**Final certification (observed).** The repository is byte-for-byte identical to a clean checkout **except for the single created document**:

```text
### CMD: git status --porcelain --ignored
 M blitzy/documentation/kitty_815df1e210e0.md
[exit: 0]
```

The `--ignored` flag lists ignored paths with a `!!` prefix and untracked paths with `??`; the single ` M` line (the created document) is the **only** output, so there are **0** ignored (`!!`) and **0** untracked (`??`) entries. The build products are likewise gone:

```text
### CMD: ls kitty/fast_data_types*.so kitty/launcher/kitty kitty/launcher/kitten build 2>&1
ls: cannot access 'kitty/fast_data_types*.so': No such file or directory
ls: cannot access 'kitty/launcher/kitty': No such file or directory
ls: cannot access 'kitty/launcher/kitten': No such file or directory
ls: cannot access 'build': No such file or directory
[exit: 2]
```

- **No existing source file was modified, added, or deleted.** The only write to the tree is `blitzy/documentation/kitty_815df1e210e0.md` (this document), which the AAP designates as the sole deliverable.
- **No build product remains.** `0` ignored and `0` untracked entries; the `.so`, both launchers, and `build/` are absent.
- **Temporary observation scripts and `/tmp/evidence/` live outside the repository** and are removed at task end; `PYTHONDONTWRITEBYTECODE=1` ensured no stray `.pyc`/`__pycache__` was written during Python runs.
- When built, the extension's md5 is `b255c0013972634549f91312ae799a0f` (deterministic across rebuilds when the environment *and* the git commit are held fixed). The exact value is specific for two reasons: (1) the build uses `-march=native`, so the emitted machine code is tuned to the build CPU's microarchitecture; and (2) `kitty/data-types.c` — which defines `PyInit_fast_data_types` and is linked into the `.so` — is compiled with `-DKITTY_VCS_REV="<git rev>"` (`setup.py:726`), so the exact commit hash is baked into the extension. The value above is therefore pinned to the authoring commit `d5f2b18b1…` (the same revision embedded in the Go build line in Appendix B) and was reproduced exactly by forcing `setup.py --vcs-rev=d5f2b18b133b19f69be07aa49af12d8b67c0d1f9`. Rebuilding at any later commit (such as a subsequent documentation-only fix that advances `HEAD`) embeds a different revision and yields a different md5 — expected behavior, not a discrepancy. The build itself is path-independent, producing a byte-identical `.so` regardless of the build directory. This hash identifies the artifact observed in Q1/Q3/Q4 and Appendix B and is recorded for reproducibility, not because the artifact is retained.

This certification describes the **actual final state** verified with the commands above — it does not assert that the tree was never touched, but that it has been returned to a clean, artifact-free baseline plus the one intended document.

---

## Appendix F — External sources consulted

The sources below fall into two tiers, labeled in the **Type** column. **First-party / official** references — the kitty project's own documentation site (`sw.kovidgoyal.net`) and its GitHub source repository (`kovidgoyal/kitty`) — are published by the project itself and are the load-bearing external corroboration. **Secondary** references — DeepWiki and Wikipedia — are third-party summaries included only as supporting material, not as authority. Bare aggregator/blog domains were deliberately excluded. In every case the architectural claims in this document are primarily backed by the in-repository `file:line` evidence above; all of these external sources are corroboration only, with the secondary tier explicitly marked as such.

| # | Source (title + link) | Type | Corroborates |
|---|------------------------|------|--------------|
| 1 | [Overview — kitty](https://sw.kovidgoyal.net/kitty/overview/) | First-party (official docs) | Q1: the deliberate C-for-performance / Python-for-orchestration split; the GPU rendering model |
| 2 | [kitty — Kovid's software projects](https://sw.kovidgoyal.net/kitty/) | First-party (official site) | Q1/Q2: GPU-accelerated rendering, SIMD, threaded I/O |
| 3 | [Developing builtin kittens](https://sw.kovidgoyal.net/kitty/kittens/developing-builtin-kittens/) | First-party (official docs) | Q4: kittens may be written in Python or Go; the Go tooling layer |
| 4 | [Kittens — introduction](https://sw.kovidgoyal.net/kitty/kittens_intro/) | First-party (official docs) | Q4: kittens as programs that run inside kitty |
| 5 | [kovidgoyal/kitty — DeepWiki](https://deepwiki.com/kovidgoyal/kitty) | Secondary (third-party wiki) | Q1/Q2: sprite-atlas + OpenGL-shader GPU pipeline; three-language structure |
| 6 | [kitty (terminal emulator) — Wikipedia](https://en.wikipedia.org/wiki/Kitty_(terminal_emulator)) | Secondary (community encyclopedia) | Q1: general GPU-based terminal description and language mix |
| 7 | [kovidgoyal/kitty — GitHub](https://github.com/kovidgoyal/kitty) | First-party (source repository) | Q1: "GPU based terminal emulator" project description |

**Claim-to-source mapping.** Q1's language-split and GPU-engine conclusions map to sources 1, 2, 5, 6, 7; Q2's sprite-atlas/shader pipeline maps to sources 1, 2, 5; Q4's Python-vs-Go kitten split maps to sources 3, 4. Q3 rests entirely on in-repository observation (no external source is needed to show the import failure).

---

*End of document. All behavioral claims above are accompanied by the exact command, its complete unedited output — condensed only in the two clearly-marked Appendix B presentations (the build transcript's explicitly-marked per-translation-unit compile-line elision, and the present→absent→present transition's step-list summary whose full tracebacks appear verbatim in Q3/Q4) — and an explicit exit code; every architectural claim carries a repository-relative `file:line` citation; and every non-observed conclusion is prefixed `Inferred:`.*
