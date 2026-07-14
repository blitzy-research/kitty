# kitty Terminal Reflow (Rewrap) Subsystem — End‑to‑End Trace and Edge‑Case Characterization

**Repository:** `kovidgoyal/kitty`
**Commit (frozen for every citation below):** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Source branch this document is named after:** `kitty_815df1e210e0`
**Deliverable:** this single markdown file. **No kitty source, test, build, or docs file was created, modified, or deleted** — this is a read‑only investigation. The candidate defect described in [§6](#6--reproduced-edge-case--root-cause-analysis-r4--r6) is **characterized, not fixed** (a repair is explicitly out of scope).

---

## How to read this document

- Every **behavioral** claim is labeled **`(observed)`** — it is backed by the actual, complete, unedited runtime output block that immediately follows it — or **`(inferred)`** — derived only from reading the code, with a `file:line` citation.
- Every **code** fact is grounded in a `file:line` reference at commit `815df1e210e0…`. All citations were independently re‑verified against the checked‑out source during this investigation (see [§7.3](#73-citation-verification)).
- Runtime output was captured **first‑hand** through the **real, canonical entry point** — `Screen.draw()` then `Screen.resize()` on a real `Screen` built from the compiled extension `kitty/fast_data_types.so`. No remote‑control hook, debug bypass, fallback, or synthetic stand‑in was used.
- Quoted C is quoted **verbatim** (real lines only — no `// ...` elision).

### The question, decomposed

| Req | What it asks | Answered in |
|-----|--------------|-------------|
| **R1** | Trace the rewrap implementation in the C code | [§2](#2--rewrap-c-code-trace-r1) |
| **R2** | Explain reflow across new dimensions, maintaining line continuations *and* cursor positions | [§3](#3--reflow-across-new-dimensions-continuation--cursor-r2) |
| **R3** | Explain the LineBuf (visible) ↔ HistoryBuf (scrollback) interaction during resize | [§4](#4--linebuf--historybuf-interaction-during-resize-r3) |
| **R4** | Identify potential issues with line‑continuation state propagation between the two buffers | [§6](#6--reproduced-edge-case--root-cause-analysis-r4--r6) |
| **R5** | Describe the complete data flow from the resize entry point through the rewrap logic | [§5](#5--complete-data-flow-from-resize-trigger-to-cell-copy-r5) |
| **R6** | Reproduce and explain the observed edge cases where reflow does not preserve logical line boundaries | [§6](#6--reproduced-edge-case--root-cause-analysis-r4--r6) |

### One‑paragraph finding

kitty reflows text with a single shared algorithm, `rewrap_inner()` in `kitty/rewrap.h`, compiled twice by macro specialization — once for the visible buffer (`kitty/line-buf.c`) and once for the scrollback (`kitty/history.c`). On resize, `screen_resize()` runs it in **two independent passes**: scrollback first, then the visible buffer. Because the scrollback pass is invoked **self‑contained** — `rewrap_inner(self, other, self->count, NULL, NULL, as_ansi_buf)` at `kitty/history.c:L611`, with both the overflow target and the cursor tracker `NULL` — a single logical line that *straddles* the scrollback↔screen boundary is **not rejoined** on widening: the scrollback segment is rewrapped alone, loses its soft‑wrap continuation bit, and the visible segment reflows in isolation. The characters survive but the **logical line boundary is not preserved**. This is proven by contrasting Scenario A (straddling the boundary — wrong) against Scenario A2 (identical text wholly inside one buffer — correct).

### Table of contents

1. [Build & run methodology](#1--build--run-methodology)
2. [Rewrap C‑code trace (R1)](#2--rewrap-c-code-trace-r1)
3. [Reflow across new dimensions: continuation & cursor (R2)](#3--reflow-across-new-dimensions-continuation--cursor-r2)
4. [LineBuf ↔ HistoryBuf interaction during resize (R3)](#4--linebuf--historybuf-interaction-during-resize-r3)
5. [Complete data flow from resize trigger to cell copy (R5)](#5--complete-data-flow-from-resize-trigger-to-cell-copy-r5)
6. [Reproduced edge case & root‑cause analysis (R4 + R6)](#6--reproduced-edge-case--root-cause-analysis-r4--r6)
7. [Observed‑vs‑inferred summary & R1–R6 coverage pass](#7--observed-vs-inferred-summary--r1r6-coverage-pass)

---

## 1 — Build & run methodology

The reflow logic lives inside a **compiled C extension**, `kitty/fast_data_types.so`. Reading the code is not enough; the code must be *built and run* so behavior can be observed. This section records the exact, reproducible build and run steps used to capture every `(observed)` block in this document.

### 1.1 Environment (actual, verified)

| Item | Value in this environment `(observed)` | Notes |
|------|----------------------------------------|-------|
| Repo commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` | `git rev-parse HEAD` |
| Python | **3.13.7** | project requires `>= 3.8` — `pyproject.toml:L2` (`requires-python = ">=3.8"`) |
| C compiler | **gcc 15.2.0** | compiles all `kitty/*.c` |
| Go | 1.24.4 | needed only for the standalone `kitten` CLI, **not** for reflow |
| Virtual env | `/opt/kitty-venv` | created **outside** the repo |

Native prerequisites (`apt`), consistent with `docs/build.rst`: `pkg-config`, `libharfbuzz-dev` (harfbuzz `>= 2.2.0` — `docs/build.rst:L84`), `libfreetype-dev`, `libfontconfig-dev`, `libpng-dev`, `liblcms2-dev`, `libxkbcommon-dev`, `libssl-dev`, `libsimde-dev` (simde — `docs/build.rst:L100`, package name at `docs/build.rst:L120`), plus X11/XCB/GL headers and `libxxhash-dev`.

> **Fidelity note (this environment vs. the historical investigation record).** An earlier capture of this same investigation was made on a host running Python 3.12.3 / gcc 13.3.0, where the built `.so` measured `1,213,072` bytes. Here the toolchain is Python 3.13.7 / gcc 15.2.0 and the freshly‑built `.so` measures **`1,253,792` bytes** `(observed)`. The byte size of a build *artifact* is toolchain‑dependent; the **reflow logic is identical** because the C source is pinned to the same commit. Every runtime block in this document was reproduced on the current toolchain and is byte‑for‑byte deterministic (see [§6.5](#65-determinism)).

### 1.2 Canonical build command

The project's canonical build command (AAP §0.8.1) is:

```
CI=true python3 setup.py build
```

**Observed result in this environment `(observed)`:** exit code **0** — the build succeeded and produced/relinked `kitty/fast_data_types.so` (**1,253,792 bytes**). The full command actually run inside the venv was `CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 python3 setup.py build`.

The only notable output was that the **Wayland GUI backend was disabled** because `wayland-protocols` is not installed:

```
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
```

This is a **GUI‑backend detail entirely unrelated to reflow**: the reflow code (`kitty/rewrap.h`, `kitty/screen.c`, `kitty/line-buf.c`, `kitty/history.c`) is compiled regardless of which windowing backend is enabled.

#### 1.2.1 On `-Werror` and `--ignore-compiler-warnings`

By default the build treats warnings as errors: `werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'` at `setup.py:L491`. The command‑line switch that flips this is registered at `setup.py:L2003-2004` (`--ignore-compiler-warnings`, `action='store_true'`).

- `(observed)` In **this** environment the canonical command needed **no** such flag — the build was clean (exit 0). Because the Wayland backend was disabled (§1.2), the GUI file `glfw/wl_window.c` was **never compiled**, so no GUI‑backend warning could be promoted to an error.
- `(inferred, from setup.py:L491 + standard compiler behavior)` On a host where the Wayland backend *is* built against a newer `wayland-protocols`, `glfw/wl_window.c` can fail under the default `-Werror` on an unhandled `switch` case for the newer `XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}` enumerators. In that case `CI=true python3 setup.py build --ignore-compiler-warnings` sets `werror = ''` (`setup.py:L491`) and the build proceeds. This flag only changes *warning* handling; it **does not alter the reflow codegen** in `fast_data_types.so` — warning flags never change C semantics. This contingency is documented for completeness; it did **not** occur here.

The standalone Go `kitten` CLI is unrelated to reflow and out of scope; here Go 1.24.4 is present, so it did not error either.

**Repository stays pristine `(observed)`:** after building, `git status --porcelain` is **empty**. Build outputs are git‑ignored — `.gitignore:L1` (`*.so`) and `.gitignore:L14` (`/build/`); `git check-ignore kitty/fast_data_types.so` confirms the extension is ignored. No tracked file changed.

### 1.3 Headless run harness — the real, canonical entry point

Observations were driven through kitty's headless test harness, which builds a genuine `Screen` backed by the compiled extension (no GPU, no GUI). A **temporary** probe script was placed **outside** the repository (in `/tmp/kitty_reflow_probe/`) and run with `PYTHONPATH=<repo-root>`. It subclasses `kitty_tests.BaseTest` (`kitty_tests/__init__.py:L208`) and uses its `create_screen(...)` factory (`kitty_tests/__init__.py:L237-L241`), then drives the real API `Screen.draw(text)` → `Screen.resize(lines, columns)`.

The factory and the underlying constructor:

```python
def create_screen(self, cols=5, lines=5, scrollback=5, cell_width=10, cell_height=20, options=None):
    self.set_options(options)
    c = Callbacks()
    s = Screen(c, lines, cols, scrollback, cell_width, cell_height, 0, c)
    return s
```
— `kitty_tests/__init__.py:L237-L241`. `Screen`, `HistoryBuf`, and `LineBuf` are imported from `kitty.fast_data_types` at `kitty_tests/__init__.py:L22`.

State was inspected only through public accessors: `str(s.line(y))` (visible row text, trailing blanks trimmed), `s.linebuf.is_continued(y)` (visible row's soft‑wrap flag), `s.historybuf.count`, `str(s.historybuf.line(i))` (scrollback row text), `Line.last_char_has_wrapped_flag()` — the Python getter defined at `kitty/line.c:L427`, which returns whether `gpu_cells[xnum-1].attrs.next_char_was_wrapped` is set (`kitty/line.c:L429`) — and `s.cursor.x` / `s.cursor.y`.

### 1.4 Critical argument‑order facts (a common source of confusion)

- **`Screen.resize(lines, columns)` takes LINES FIRST, COLUMNS SECOND.** The Python caller passes `ynum` (lines) first: `self.screen.resize(max(0, new_geometry.ynum), max(0, new_geometry.xnum))` at `kitty/window.py:L854`. The C binding parses `"|II"` into `(a, b)` and calls `screen_resize(self, a, b)` at `kitty/screen.c:L3932`, where `a` = lines, `b` = columns. `(observed)` confirmed empirically: `resize(2, 6)` applied to a 4‑column screen produced a **6‑column** result (columns became the *second* argument).
- **The harness factory has the OPPOSITE order:** `create_screen(cols=5, lines=5, …)` takes **cols first**, then internally constructs `Screen(c, lines, cols, …)` — i.e. the `Screen` constructor itself is `(callbacks, lines, columns, …)` (`kitty_tests/__init__.py:L237-L241`). Throughout this document, `create_screen(cols=C, lines=L, …)` is written cols‑first, while `resize(L, C)` is written lines‑first. Both match the real code.

### 1.5 Cleanup

All temporary probe scripts live outside the repository and are deleted after use (`rm -rf /tmp/kitty_reflow_probe`), after which `git status --porcelain` is re‑verified to be **empty** so the source tree is left byte‑for‑byte unchanged.

### 1.6 HistoryBuf index convention used in every output block

For scrollback dumps, `s.historybuf.line(0)` is the **newest** scrollback row (the one immediately above the visible area); higher indices are **older**. To read like a real terminal (oldest at the top), the blocks below print scrollback **highest‑index‑first**, so e.g. `HIST[7]` is the oldest/top row and `HIST[0]` is the newest/just‑above‑visible row.

---

## 2 — Rewrap C‑code trace (R1)

**R1 asks:** *trace the rewrap implementation in the C code.* kitty implements reflow with a **single algorithm written once** — the template `rewrap_inner()` in `kitty/rewrap.h` — and **compiled twice** via macro specialization: once for the visible buffer in `kitty/line-buf.c` (with the file's *default* macros) and once for the scrollback in `kitty/history.c` (with *overridden* macros). This is a deliberate single‑source‑of‑truth pattern, and it is *why one algorithm produces two subtly different behaviors*.

### 2.1 The shared template — `kitty/rewrap.h`

`kitty/rewrap.h` is 96 lines. It begins by declaring macros with `#ifndef` guards so a translation unit can override any of them *before* `#include`‑ing the header; when a macro is not overridden, the default applies.

**Default buffer type and the per‑line helpers** (`kitty/rewrap.h:L10-L42`):

```c
#ifndef BufType
#define BufType LineBuf
#endif

#ifndef init_src_line
#define init_src_line(src_y) linebuf_init_line(src, src_y);
#endif

#define set_dest_line_attrs(dest_y) dest->line_attrs[dest_y] = src->line->attrs; src->line->attrs.prompt_kind = UNKNOWN_PROMPT_KIND;

#ifndef first_dest_line
#define first_dest_line linebuf_init_line(dest, 0); set_dest_line_attrs(0)
#endif

#ifndef next_dest_line
#define next_dest_line(continued) \
    linebuf_set_last_char_as_continuation(dest, dest_y, continued); \
    if (dest_y >= dest->ynum - 1) { \
        linebuf_index(dest, 0, dest->ynum - 1); \
        if (historybuf != NULL) { \
            linebuf_init_line(dest, dest->ynum - 1); \
            dest->line->attrs.has_dirty_text = true; \
            historybuf_add_line(historybuf, dest->line, as_ansi_buf); \
        }\
        linebuf_clear_line(dest, dest->ynum - 1, true); \
    } else dest_y++; \
    linebuf_init_line(dest, dest_y); \
    set_dest_line_attrs(dest_y);
#endif

#ifndef is_src_line_continued
#define is_src_line_continued() (src->line->gpu_cells[src->xnum-1].attrs.next_char_was_wrapped)
#endif
```

Three things in this block are load‑bearing for the whole subsystem:

1. **`is_src_line_continued()` (`kitty/rewrap.h:L40-L42`, read at L41)** is *the* soft‑wrap continuation test: it reads the `next_char_was_wrapped` bit of the **last cell** of the source row (`gpu_cells[src->xnum-1]`). That bit is the per‑row "this line is soft‑wrapped into the next" marker (see [§3.1](#31-a-the-continuation-marker-next_char_was_wrapped)).
2. **`next_dest_line(continued)` (`kitty/rewrap.h:L24-L38`)** finishes the current destination row and starts a new one. It first stamps the just‑finished row's continuation flag via `linebuf_set_last_char_as_continuation(dest, dest_y, continued)` (**L26**). If the destination is already on its last row (`dest_y >= dest->ynum - 1`), it scrolls that row off the top and — **only if `historybuf != NULL`** (**L29-L33**) — pushes it into scrollback via `historybuf_add_line(historybuf, dest->line, as_ansi_buf)` (**L32**). This `historybuf != NULL` guard is the hinge the whole cross‑buffer story turns on.
3. **`set_dest_line_attrs` (`kitty/rewrap.h:L18`)** copies the source row's line attributes onto the destination row and resets the source's `prompt_kind`.

**Low‑level cell copy and the cursor tracker** (`kitty/rewrap.h:L44-L53`):

```c
static inline void
copy_range(Line *src, index_type src_at, Line* dest, index_type dest_at, index_type num) {
    memcpy(dest->cpu_cells + dest_at, src->cpu_cells + src_at, num * sizeof(CPUCell));
    memcpy(dest->gpu_cells + dest_at, src->gpu_cells + src_at, num * sizeof(GPUCell));
}

typedef struct TrackCursor {
    index_type x, y;
    bool is_tracked_line, is_sentinel;
} TrackCursor;
```

`copy_range()` (`kitty/rewrap.h:L44-L48`) is the primitive that actually moves characters: two `memcpy`s, one for the CPU cells and one for the GPU cells, of `num` cells. `TrackCursor` (`kitty/rewrap.h:L50-L53`) is the small struct that lets the algorithm follow a coordinate through the transformation (see [§3.2](#32-b-cursor-preservation-the-trackcursor-machinery-r2)).

**The algorithm itself — `rewrap_inner()` (`kitty/rewrap.h:L56-L96`)**, quoted in full:

```c
static void
rewrap_inner(BufType *src, BufType *dest, const index_type src_limit, HistoryBuf UNUSED *historybuf, TrackCursor *track, ANSIBuf *as_ansi_buf) {
    bool is_first_line = true;
    index_type src_y = 0, src_x = 0, dest_x = 0, dest_y = 0, num = 0, src_x_limit = 0;
    TrackCursor tc_end = {.is_sentinel = true };
    if (!track) track = &tc_end;

    do {
        for (TrackCursor *t = track; !t->is_sentinel; t++) t->is_tracked_line = src_y == t->y;
        init_src_line(src_y);
        const bool src_line_is_continued = is_src_line_continued();
        src_x_limit = src->xnum;
        if (!src_line_is_continued) {
            // Trim trailing blanks since there is a hard line break at the end of this line
            while(src_x_limit && (src->line->cpu_cells[src_x_limit - 1].ch) == BLANK_CHAR) src_x_limit--;
        } else {
            src->line->gpu_cells[src->xnum-1].attrs.next_char_was_wrapped = false;
        }
        for (TrackCursor *t = track; !t->is_sentinel; t++) {
            if (t->is_tracked_line && t->x >= src_x_limit) t->x = MAX(1u, src_x_limit) - 1;
        }
        if (is_first_line) {
            first_dest_line; is_first_line = false;
        }
        while (src_x < src_x_limit) {
            if (dest_x >= dest->xnum) { next_dest_line(true); dest_x = 0; }
            num = MIN(src->line->xnum - src_x, dest->xnum - dest_x);
            copy_range(src->line, src_x, dest->line, dest_x, num);
            for (TrackCursor *t = track; !t->is_sentinel; t++) {
                if (t->is_tracked_line && src_x <= t->x && t->x < src_x + num) {
                    t->y = dest_y;
                    t->x = dest_x + (t->x - src_x + (t->x > 0));
                }
            }
            src_x += num; dest_x += num;
        }
        src_y++; src_x = 0;
        if (!src_line_is_continued && src_y < src_limit) { init_src_line(src_y); next_dest_line(false); dest_x = 0; }
    } while (src_y < src_limit);
    dest->line->ynum = dest_y;
}
```

Walking it as cause → effect:

- **Per source row**, it reads the continuation bit into `src_line_is_continued` (**L66**).
- **If the row is *not* continued** (a hard line break ends it), it **trims trailing blank cells** by shrinking `src_x_limit` while the last cell is `BLANK_CHAR` (**L68-L70**). This is why hard‑ended lines do not carry their padding into the reflowed result.
- **If the row *is* continued**, it **clears the source's terminal wrapped bit**: `src->line->gpu_cells[src->xnum-1].attrs.next_char_was_wrapped = false;` (**L71-L73**, the clear at **L72**). The bit is about to be re‑derived by the copy loop, so the source copy is normalized first.
- It clamps any tracked cursor whose `x` is past the trimmed limit (**L74-L76**), and on the very first source row opens the first destination row via `first_dest_line` (**L77-L79**).
- The **copy loop** (**L80-L91**) fills destination rows `dest->xnum` cells at a time. Whenever the destination row is full (`dest_x >= dest->xnum`), it calls **`next_dest_line(true)`** (**L81**) — the `true` marks the row it just finished as *soft‑wrapped* (continued). Each chunk is a `copy_range(...)` (**L83**), and the tracked cursor is remapped to its new `(dest_y, dest_x)` position (**L84-L89**).
- After a *non‑continued* source row (and if more source remains), it emits a **hard break** with **`next_dest_line(false)`** (**L93**) — the `false` leaves the finished row *not* continued.
- Finally it records the number of destination rows produced: `dest->line->ynum = dest_y;` (**L95**).

The crucial asymmetry to keep in mind for R4/R6: the destination row's continuation bit is (re)written **only** by a `next_dest_line(...)` call. `next_dest_line(true)` happens **only when a destination row fills** (L81). So if a continued source row's content **fits without filling** the destination row, `next_dest_line(true)` is never called for it, and the continuation bit is **not re‑applied** — it stays cleared by L72.

### 2.2 Specialization #1 — the visible buffer, `kitty/line-buf.c` (default macros)

`kitty/line-buf.c` includes the template with the **default** macros — it does *not* redefine `BufType`, so `BufType` is `LineBuf` (`kitty/rewrap.h:L10-L12`). The include site:

```c
#include "rewrap.h"
```
— `kitty/line-buf.c:L583`.

The visible‑buffer entry point is `linebuf_rewrap()` (`kitty/line-buf.c:L585-L622`), quoted in full:

```c
void
linebuf_rewrap(LineBuf *self, LineBuf *other, index_type *num_content_lines_before, index_type *num_content_lines_after, HistoryBuf *historybuf, index_type *track_x, index_type *track_y, index_type *track_x2, index_type *track_y2, ANSIBuf *as_ansi_buf) {
    index_type first, i;
    bool is_empty = true;

    // Fast path
    if (other->xnum == self->xnum && other->ynum == self->ynum) {
        memcpy(other->line_map, self->line_map, sizeof(index_type) * self->ynum);
        memcpy(other->line_attrs, self->line_attrs, sizeof(LineAttrs) * self->ynum);
        memcpy(other->cpu_cell_buf, self->cpu_cell_buf, (size_t)self->xnum * self->ynum * sizeof(CPUCell));
        memcpy(other->gpu_cell_buf, self->gpu_cell_buf, (size_t)self->xnum * self->ynum * sizeof(GPUCell));
        *num_content_lines_before = self->ynum; *num_content_lines_after = self->ynum;
        return;
    }

    // Find the first line that contains some content
    first = self->ynum;
    do {
        first--;
        CPUCell *cells = cpu_lineptr(self, self->line_map[first]);
        for(i = 0; i < self->xnum; i++) {
            if ((cells[i].ch) != BLANK_CHAR) { is_empty = false; break; }
        }
    } while(is_empty && first > 0);

    if (is_empty) {  // All lines are empty
        *num_content_lines_after = 0;
        *num_content_lines_before = 0;
        return;
    }
    *num_content_lines_before = first + 1;
    TrackCursor tcarr[3] = {{.x = *track_x, .y = *track_y }, {.x = *track_x2, .y = *track_y2}, {.is_sentinel = true}};
    rewrap_inner(self, other, *num_content_lines_before, historybuf, (TrackCursor*)tcarr, as_ansi_buf);
    *track_x = tcarr[0].x; *track_y = tcarr[0].y;
    *track_x2 = tcarr[1].x; *track_y2 = tcarr[1].y;
    *num_content_lines_after = other->line->ynum + 1;
    for (i = 0; i < *num_content_lines_after; i++) other->line_attrs[i].has_dirty_text = true;
}
```

Key facts:

- **Fast path** at **`kitty/line-buf.c:L591-L598`**: when the new geometry equals the old (`other->xnum == self->xnum && other->ynum == self->ynum`), it `memcpy`s the whole buffer and returns — **no reflow** (see Scenario D, [§3.3](#33-widen-vs-narrow-and-the-fast-path)).
- It finds the last content row (**L600-L608**), sets up a **real** `TrackCursor tcarr[3]` (two tracked coordinates plus a sentinel) at **L616**, and calls **`rewrap_inner(self, other, *num_content_lines_before, historybuf, (TrackCursor*)tcarr, as_ansi_buf)` at L617** — passing a **real `historybuf` overflow target** and a **real cursor tracker**. Consequences: the visible buffer *overflows into scrollback* and *its cursor is tracked*.

The continuation setter used by the template's `next_dest_line` for this buffer is `linebuf_set_last_char_as_continuation()` (`kitty/line-buf.c:L193-L198`):

```c
void
linebuf_set_last_char_as_continuation(LineBuf *self, index_type y, bool continued) {
    if (y < self->ynum) {
        gpu_lineptr(self, self->line_map[y])[self->xnum - 1].attrs.next_char_was_wrapped = continued;
    }
}
```
— it writes `next_char_was_wrapped = continued` on the last cell of row `y` (**L196**).

### 2.3 Specialization #2 — the scrollback, `kitty/history.c` (overridden macros) — the root‑cause file

`kitty/history.c` **redefines** the template macros *before* including it, then includes it (`kitty/history.c:L582-L592`):

```c
#define BufType HistoryBuf

#define map_src_index(y) ((src->start_of_data + y) % src->ynum)

#define init_src_line(src_y) init_line(src, map_src_index(src_y), src->line);

#define next_dest_line(cont) { history_buf_set_last_char_as_continuation(dest, 0, cont); LineAttrs *lap = attrptr(dest, historybuf_push(dest, as_ansi_buf)); *lap = src->line->attrs; }

#define first_dest_line next_dest_line(false);

#include "rewrap.h"
```

- `BufType` becomes `HistoryBuf` (**L582**).
- `map_src_index(y)` (**L584**) is **circular indexing** `((src->start_of_data + y) % src->ynum)` — scrollback is a ring buffer, so source rows are addressed modulo `ynum`.
- `init_src_line` (**L586**) initializes a source line via that circular map.
- `next_dest_line(cont)` (**L588**) is redefined to stamp the continuation flag with `history_buf_set_last_char_as_continuation(dest, 0, cont)` and then **push a new scrollback row** with `historybuf_push(dest, as_ansi_buf)` — there is **no `historybuf != NULL` overflow branch** here, unlike the default definition, because scrollback *is* the terminal sink.
- `first_dest_line` (**L590**) reduces to `next_dest_line(false)`.

The scrollback entry point is `historybuf_rewrap()` (`kitty/history.c:L594-L614`):

```c
void
historybuf_rewrap(HistoryBuf *self, HistoryBuf *other, ANSIBuf *as_ansi_buf) {
    while(other->num_segments < self->num_segments) add_segment(other);
    if (other->xnum == self->xnum && other->ynum == self->ynum) {
        // Fast path
        for (index_type i = 0; i < self->num_segments; i++) {
            memcpy(other->segments[i].cpu_cells, self->segments[i].cpu_cells, SEGMENT_SIZE * self->xnum * sizeof(CPUCell));
            memcpy(other->segments[i].gpu_cells, self->segments[i].gpu_cells, SEGMENT_SIZE * self->xnum * sizeof(GPUCell));
            memcpy(other->segments[i].line_attrs, self->segments[i].line_attrs, SEGMENT_SIZE * sizeof(LineAttrs));
        }
        other->count = self->count; other->start_of_data = self->start_of_data;
        return;
    }
    if (other->pagerhist && other->xnum != self->xnum && ringbuf_bytes_used(other->pagerhist->ringbuf))
        other->pagerhist->rewrap_needed = true;
    other->count = 0; other->start_of_data = 0;
    if (self->count > 0) {
        rewrap_inner(self, other, self->count, NULL, NULL, as_ansi_buf);
        for (index_type i = 0; i < other->count; i++) attrptr(other, (other->start_of_data + i) % other->ynum)->has_dirty_text = true;
    }
}
```

- **Fast path** at **`kitty/history.c:L597-L606`** (same‑geometry `memcpy`).
- The reflow call is **`rewrap_inner(self, other, self->count, NULL, NULL, as_ansi_buf)` at `kitty/history.c:L611`** — **both** the `historybuf` overflow target **and** the `TrackCursor *track` are **`NULL`**. This is the single most important line for R4/R6: the scrollback rewrap is **self‑contained**. With `historybuf == NULL`, the template's `next_dest_line` overflow branch (`kitty/rewrap.h:L29-L33`) is inert — the scrollback rewrap **cannot pull characters up from the visible buffer**, and with `track == NULL` it tracks **no** cursor.

The continuation setter for scrollback is `history_buf_set_last_char_as_continuation()` (`kitty/history.c:L302-L307`):

```c
static void
history_buf_set_last_char_as_continuation(HistoryBuf *self, index_type y, bool wrapped) {
    if (self->count > 0) {
        gpu_lineptr(self, index_of(self, y))[self->xnum-1].attrs.next_char_was_wrapped = wrapped;
    }
}
```
— it writes `next_char_was_wrapped = wrapped` (**L305**).

### 2.4 One algorithm, two behaviors — the summary for R1

| Aspect | Visible buffer (`line-buf.c`) | Scrollback (`history.c`) |
|--------|-------------------------------|---------------------------|
| `BufType` | `LineBuf` (default, `rewrap.h:L10-12`) | `HistoryBuf` (`history.c:L582`) |
| Source indexing | linear (`rewrap.h:L14-16`) | **circular** `((start_of_data+y)%ynum)` (`history.c:L584`) |
| `rewrap_inner` call | `…, historybuf, (TrackCursor*)tcarr, …` — **real** overflow + **real** tracker (`line-buf.c:L617`) | `…, NULL, NULL, …` — **no** overflow, **no** tracker (`history.c:L611`) |
| Overflow of the top row | pushed into scrollback via `historybuf_add_line` (`rewrap.h:L29-33`) | none (scrollback *is* the sink; pushes via `historybuf_push`, `history.c:L588`) |
| Cursor tracking | yes | no |

The identical inner algorithm therefore does two different jobs depending purely on the macro environment and the two `NULL`s at `history.c:L611`. That difference is exactly what produces the boundary defect in [§6](#6--reproduced-edge-case--root-cause-analysis-r4--r6).

---

## 3 — Reflow across new dimensions: continuation & cursor (R2)

**R2 asks:** *how is text redistributed across a new column/row count while (a) maintaining line continuations and (b) maintaining cursor positions?*

### 3.1 (a) The continuation marker `next_char_was_wrapped`

kitty distinguishes **soft wraps** (a logical line too wide for the grid, which *should* be rejoined and reflowed on resize) from **hard wraps** (an explicit newline, a logical boundary that *must* be preserved). The distinction is carried by a **single bit on the last cell of each row**: `next_char_was_wrapped`, a 1‑bit field of the `CellAttrs` union (`kitty/data-types.h:L206`):

```c
uint16_t next_char_was_wrapped : 1;
```

(The same field appears in the `SGR_MASK` definition at `kitty/data-types.h:L214`.) When the last cell of a row has this bit set, the row is *soft‑wrapped* — its logical line continues on the next row.

How reflow reads and rewrites the bit, as cause → effect (all in `rewrap_inner`, [§2.1](#21-the-shared-template--kittyrewraph)):

1. **Read.** `is_src_line_continued()` reads `gpu_cells[src->xnum-1].attrs.next_char_was_wrapped` (`kitty/rewrap.h:L41`), consumed at `kitty/rewrap.h:L66`.
2. **Not continued → trim + hard break.** Trailing blanks are trimmed (`kitty/rewrap.h:L68-L70`) and, once the row is copied, `next_dest_line(false)` emits a hard break (`kitty/rewrap.h:L93`) — the finished destination row is left *not* continued.
3. **Continued → clear source bit, flow through.** The source's terminal bit is cleared (`kitty/rewrap.h:L72`) and the characters flow straight into the next destination row without a boundary.
4. **Re‑stamp on fill.** When a destination row fills mid‑logical‑line, `next_dest_line(true)` (`kitty/rewrap.h:L81`) sets the just‑finished destination row's continuation bit via the buffer's setter (`linebuf_set_last_char_as_continuation`, `kitty/line-buf.c:L196`; or `history_buf_set_last_char_as_continuation`, `kitty/history.c:L305`).

`(observed)` The end‑to‑end effect on **widening** — a single 12‑char logical line drawn into a 4‑wide grid, then widened to 6 — is visible in Scenario A2, where the whole line lives inside the visible buffer. Before, the logical line occupies three soft‑wrapped 4‑wide rows; after, it occupies two soft‑wrapped 6‑wide rows, and the continuation bits track the new boundaries exactly:

```
A2.BEFORE (4x4): cursor=(4,2) hist.count=0 cols=4 lines=4
    VIS[0]  'ABCD'     is_continued=False last_char_wrapped=True
    VIS[1]  'EFGH'     is_continued=True  last_char_wrapped=True
    VIS[2]  'IJKL'     is_continued=True  last_char_wrapped=False
    VIS[3]  ''         is_continued=False last_char_wrapped=False
A2.AFTER resize(4,6): cursor=(5,1) hist.count=0 cols=6 lines=4
    VIS[0]  'ABCDEF'   is_continued=False last_char_wrapped=True
    VIS[1]  'GHIJKL'   is_continued=True  last_char_wrapped=False
    VIS[2]  ''         is_continued=False last_char_wrapped=False
    VIS[3]  ''         is_continued=False last_char_wrapped=False
    joined-across-buffers = 'ABCDEFGHIJKL'
```

Note the reading convention: `is_continued=True` on a row means that row is the *continuation of* the previous one (kitty's per‑row "is this line continued from above" query), while `last_char_wrapped=True` means the row's last cell has `next_char_was_wrapped` set (it wraps *into* the next). Row `VIS[0] 'ABCDEF'` has `last_char_wrapped=True` (it wraps into `GHIJKL`), and `VIS[1] 'GHIJKL'` has `is_continued=True` (it continues `ABCDEF`) — the logical line `ABCDEFGHIJKL` is preserved as one soft‑wrapped unit.

`(observed)` The effect on **narrowing** is Scenario B ([§4.3](#43-observed-narrowing-pushes-overflow-into-scrollback-scenario-b)): each wide row is split into several narrow rows, with continuation bits set on every non‑final piece.

### 3.2 (b) Cursor preservation: the `TrackCursor` machinery (R2)

The cursor (and the saved‑cursor savepoints) are carried through the transformation so they still point at the same logical character afterwards. The mechanism is the `TrackCursor` struct (`kitty/rewrap.h:L50-L53`):

- `linebuf_rewrap` seeds a real `TrackCursor tcarr[3]` with the current and saved cursor coordinates (`kitty/line-buf.c:L616`) and passes it into `rewrap_inner` (`kitty/line-buf.c:L617`).
- Inside `rewrap_inner`, for each source row the tracker's `is_tracked_line` flag is set when `src_y` matches the tracked `y` (`kitty/rewrap.h:L64`); during the copy loop, when the tracked `x` falls inside the copied chunk, the tracker is remapped to its new grid position (`kitty/rewrap.h:L84-L89`):

```c
for (TrackCursor *t = track; !t->is_sentinel; t++) {
    if (t->is_tracked_line && src_x <= t->x && t->x < src_x + num) {
        t->y = dest_y;
        t->x = dest_x + (t->x - src_x + (t->x > 0));
    }
}
```

- Back in `screen_resize`, the tracked results are read into `CursorTrack` (`kitty/screen.c:L226-L232`) via the `setup_cursor` macro (`kitty/screen.c:L366-L370`), then the final cursor is clamped to the new grid by the `S()` macro (`kitty/screen.c:L419-L423`):

```c
#define S(c, w) c->x = MIN(w.after.x, self->columns - 1); c->y = MIN(w.after.y, self->lines - 1);
    S(self->cursor, cursor);
    S((&(self->main_savepoint.cursor)), main_saved_cursor);
    S((&(self->alt_savepoint.cursor)), alt_saved_cursor);
#undef S
```

- If the cursor was **beyond the content** before resize, a separate branch places it just after the reflowed content (`kitty/screen.c:L424-L427`):

```c
    if (cursor.is_beyond_content) {
        self->cursor->y = cursor.num_content_lines;
        if (self->cursor->y >= self->lines) { self->cursor->y = self->lines - 1; screen_index(self); }
    }
```

Note (contrast with [§2.3](#23-specialization-2--the-scrollback-kittyhistoryc-overridden-macros--the-root-cause-file)): the scrollback rewrap passes `track == NULL` (`kitty/history.c:L611`), so **scrollback reflow tracks no cursor** — only the visible buffer does. That is correct, since the interactive cursor lives in the visible grid.

### 3.3 Widen vs. narrow, and the fast path

Two `(observed)` cursor scenarios demonstrate the tracker end‑to‑end.

**Scenario C — cursor preserved on widening.** Drawing `'ab'` leaves the cursor at column 2 of row 0; widening from 5 to 3… — widening the column count (the row is not reflowed) leaves the cursor untouched:

Command: `create_screen(cols=5, lines=5, scrollback=100)` → `draw('ab')` → `resize(5, 3)` (lines=5, columns=3).
```
C.BEFORE cursor=(2,0)
C.AFTER  resize(5,3) cursor=(2,0)
```

**Scenario C2 — cursor remapped on narrowing.** Drawing 15 characters into a 5‑wide grid leaves the cursor at `(5,2)` (end of the third wrapped row); narrowing to 2 columns reflows the content and the tracker remaps the cursor to its new grid coordinate `(1,4)`:

Command: `create_screen(cols=5, lines=5, scrollback=100)` → `draw('12345'*3)` → `resize(5, 2)` (lines=5, columns=2).
```
C2.BEFORE cursor=(5,2)
C2.AFTER  resize(5,2) cursor=(1,4)
```

Together, C and C2 show the `TrackCursor` machinery (`kitty/rewrap.h:L50-L53,L84-L89`; `kitty/screen.c:L419-L423`) doing its job: **unchanged when the row is unaffected, remapped to the new grid position when the row is reflowed** `(observed)`.

**Scenario D — the fast path (no reflow).** When *neither* columns nor lines change, both `linebuf_rewrap` (`kitty/line-buf.c:L591-L598`) and `historybuf_rewrap` (`kitty/history.c:L597-L606`) take the `memcpy` fast path and perform **no reflow**. The content is byte‑identical afterwards; only the cursor `x` is still clamped from 5 to 4 by the `S()` macro (`kitty/screen.c:L419`) because `x=5` is beyond the last valid column index in a 5‑wide grid:

Command: `create_screen(cols=5, lines=5, scrollback=100)` → `draw('12345'*4)` → `resize(5, 5)` (identical dimensions).
```
D.BEFORE lines=['12345', '12345', '12345', '12345', ''] cursor.x=5
D.AFTER  lines=['12345', '12345', '12345', '12345', ''] cursor.x=4
byte-identical content: True
```

This `(observed)` block is what distinguishes "no reflow needed" (fast path) from "reflow performed" (Scenarios A/A2/B/C2).

---

## 4 — LineBuf ↔ HistoryBuf interaction during resize (R3)

**R3 asks:** *how do the visible screen buffer and the scrollback history rewrap, in what order, and how do lines that overflow the top of the visible buffer get pushed into scrollback?*

### 4.1 The order: history first, then the visible buffer — two independent passes

`screen_resize()` (`kitty/screen.c:L345-L463`) orchestrates the resize. It performs reflow in **two passes, in this order**:

**Pass 1 — scrollback.** `realloc_hb()` allocates a new `HistoryBuf` at the new width and rewraps the old one into it (`kitty/screen.c:L216-L223`):

```c
static HistoryBuf*
realloc_hb(HistoryBuf *old, unsigned int lines, unsigned int columns, ANSIBuf *as_ansi_buf) {
    HistoryBuf *ans = alloc_historybuf(lines, columns, 0);
    if (ans == NULL) { PyErr_NoMemory(); return NULL; }
    ans->pagerhist = old->pagerhist; old->pagerhist = NULL;
    historybuf_rewrap(old, ans, as_ansi_buf);
    return ans;
}
```

It is invoked from `screen_resize` at **`kitty/screen.c:L375`**:

```c
    HistoryBuf *nh = realloc_hb(self->historybuf, self->historybuf->ynum, columns, &self->as_ansi_buf);
    if (nh == NULL) return false;
    Py_CLEAR(self->historybuf); self->historybuf = nh;
```

`historybuf_rewrap` (the actual reflow) is called from `realloc_hb` at **`kitty/screen.c:L221`**, and — as established in [§2.3](#23-specialization-2--the-scrollback-kittyhistoryc-overridden-macros--the-root-cause-file) — it internally calls `rewrap_inner(self, other, self->count, NULL, NULL, …)` at `kitty/history.c:L611`, i.e. **self‑contained**, with no overflow target.

**Pass 2 — the visible buffer.** `realloc_lb()` allocates a new `LineBuf` and rewraps the old visible buffer into it, **passing the already‑rewrapped `historybuf` as the overflow target** (`kitty/screen.c:L234-L242`):

```c
static LineBuf*
realloc_lb(LineBuf *old, unsigned int lines, unsigned int columns, index_type *nclb, index_type *ncla, HistoryBuf *hb, CursorTrack *a, CursorTrack *b, ANSIBuf *as_ansi_buf) {
    LineBuf *ans = alloc_linebuf(lines, columns);
    if (ans == NULL) { PyErr_NoMemory(); return NULL; }
    a->temp.x = a->before.x; a->temp.y = a->before.y;
    b->temp.x = b->before.x; b->temp.y = b->before.y;
    linebuf_rewrap(old, ans, nclb, ncla, hb, &a->temp.x, &a->temp.y, &b->temp.x, &b->temp.y, as_ansi_buf);
    return ans;
}
```

It is invoked for the **main** screen at **`kitty/screen.c:L384`**, passing the real `self->historybuf`:

```c
    LineBuf *n = realloc_lb(self->main_linebuf, lines, columns, &num_content_lines_before, &num_content_lines_after, self->historybuf, &cursor, &main_saved_cursor, &self->as_ansi_buf);
```

`linebuf_rewrap` calls `rewrap_inner(self, other, …, historybuf, (TrackCursor*)tcarr, …)` at `kitty/line-buf.c:L617` with that **real** `historybuf` (see [§2.2](#22-specialization-1--the-visible-buffer-kittyline-bufc-default-macros)).

The **alternate** screen is realloc'd with a **`NULL`** overflow target at **`kitty/screen.c:L394`** because the alt screen has no scrollback:

```c
    n = realloc_lb(self->alt_linebuf, lines, columns, &num_content_lines_before, &num_content_lines_after, NULL, &cursor, &alt_saved_cursor, &self->as_ansi_buf);
```

Between the two passes, on the main screen kitty also snapshots the current prompt so the shell can redraw it without flicker — `prevent_current_prompt_from_rewrapping()` (`kitty/screen.c:L302-L343`) is called at **`kitty/screen.c:L382`**, and the saved prompt lines are copied back after reflow at `kitty/screen.c:L444-L461`. This is orthogonal to the continuation‑bit story but is part of the same function.

### 4.2 How top‑overflow rows are pushed into scrollback

During Pass 2, when `rewrap_inner` fills the **last** destination row of the visible buffer and needs another, the template's `next_dest_line` (`kitty/rewrap.h:L24-L38`) scrolls the top row off and pushes it into scrollback — **but only because `historybuf != NULL`** on this pass (`kitty/rewrap.h:L29-L33`):

```c
    if (dest_y >= dest->ynum - 1) { \
        linebuf_index(dest, 0, dest->ynum - 1); \
        if (historybuf != NULL) { \
            linebuf_init_line(dest, dest->ynum - 1); \
            dest->line->attrs.has_dirty_text = true; \
            historybuf_add_line(historybuf, dest->line, as_ansi_buf); \
        }\
        linebuf_clear_line(dest, dest->ynum - 1, true); \
    } else dest_y++; \
```

`historybuf_add_line()` (`kitty/history.c:L286-L291`) is the push:

```c
void
historybuf_add_line(HistoryBuf *self, const Line *line, ANSIBuf *as_ansi_buf) {
    index_type idx = historybuf_push(self, as_ansi_buf);
    copy_line(line, self->line);
    *attrptr(self, idx) = line->attrs;
}
```

— it advances the ring with `historybuf_push()` (`kitty/history.c:L275-L284`, push at L288), then `copy_line(line, self->line)` (`kitty/history.c:L289`) copies **all** cells (including the last cell's continuation bit), then stores the line attributes (`kitty/history.c:L290`). `copy_line` is declared in `kitty/lineops.h:L25`; `linebuf_copy_line_to` at `kitty/lineops.h:L112` is the sibling primitive used by the prompt copy‑back. The reverse operation, `historybuf_pop_line()` (`kitty/history.c:L293-L300`), is used by the enlarged‑window scrollback‑fill path (`kitty/screen.c:L428-L438`).

The important asymmetry, restated for R3: **the visible‑buffer pass has a place to send its overflow (scrollback); the scrollback pass does not** (`historybuf == NULL` at `kitty/history.c:L611`). Overflow flows **only one way — down from the visible buffer into scrollback — never up**.

### 4.3 (observed) Narrowing pushes overflow into scrollback (Scenario B)

Drawing five 5‑char runs into a 5×5 grid fills all five visible rows as one long soft‑wrapped logical line. Narrowing to 2 columns splits each 5‑wide row into 2‑wide pieces; the visible buffer can only hold 5 of the resulting rows, so **8 rows overflow the top and are pushed into scrollback**, each with its continuation bit preserved:

Command: `create_screen(cols=5, lines=5, scrollback=100)` → `draw('0'*5 + '1'*5 + '2'*5 + '3'*5 + '4'*5)` → `resize(5, 2)` (lines=5, columns=2).
```
B.BEFORE (5x5): cursor=(5,4) hist.count=0 cols=5 lines=5
    VIS[0]  '00000'  is_continued=False last_char_wrapped=True
    VIS[1]  '11111'  is_continued=True  last_char_wrapped=True
    VIS[2]  '22222'  is_continued=True  last_char_wrapped=True
    VIS[3]  '33333'  is_continued=True  last_char_wrapped=True
    VIS[4]  '44444'  is_continued=True  last_char_wrapped=False
B.AFTER  resize(5,2): cursor=(1,4) hist.count=8 cols=2 lines=5
    HIST[7] '00'  last_char_wrapped=True
    HIST[6] '00'  last_char_wrapped=True
    HIST[5] '01'  last_char_wrapped=True
    HIST[4] '11'  last_char_wrapped=True
    HIST[3] '11'  last_char_wrapped=True
    HIST[2] '22'  last_char_wrapped=True
    HIST[1] '22'  last_char_wrapped=True
    HIST[0] '23'  last_char_wrapped=True
    VIS[0]  '33'  is_continued=False last_char_wrapped=True
    VIS[1]  '33'  is_continued=True  last_char_wrapped=True
    VIS[2]  '44'  is_continued=True  last_char_wrapped=True
    VIS[3]  '44'  is_continued=True  last_char_wrapped=True
    VIS[4]  '4'   is_continued=True  last_char_wrapped=False
```

(As per [§1.6](#16-historybuf-index-convention-used-in-every-output-block), `HIST[7]` is the oldest/top row and `HIST[0]` the newest/just‑above‑visible.) `hist.count` went from 0 to **8** `(observed)`, proving the overflow push. Every scrollback row and every visible row except the last carries `last_char_wrapped=True`, so the single logical line `0000011111222223333344444` is preserved end‑to‑end across the boundary — because **all of the content originated in the visible buffer and was handled by Pass 2 with a live overflow target**. This is the well‑behaved counterpart to the defect in [§6](#6--reproduced-edge-case--root-cause-analysis-r4--r6), where content that was *already split across the two buffers before the resize* is not rejoined.

---

## 5 — Complete data flow from resize trigger to cell copy (R5)

**R5 asks:** *describe the complete data flow from the resize entry point through the rewrap logic — the full call chain down to the low‑level cell copying.*

### 5.1 The full chain

```mermaid
flowchart TD
    A["GUI/layout resize → boss/tabs propagate<br/>kitty/boss.py tm.resize() → kitty/tabs.py resize → kitty/tab_bar.py s.resize(1, ncells)"]
      --> B["Window.set_geometry()<br/>kitty/window.py:L850-L856"]
    B --> C["self.screen.resize(ynum, xnum)  (LINES first)<br/>kitty/window.py:L854"]
    C --> D["C binding resize(Screen*, args)<br/>parse |II → (a,b)  kitty/screen.c:L3929-L3935"]
    D --> E["screen_resize(self, a=lines, b=columns)<br/>kitty/screen.c:L345-L463 (call at L3932)"]
    E --> F["PASS 1: realloc_hb()  kitty/screen.c:L375 → L216-L223"]
    F --> G["historybuf_rewrap()  kitty/history.c:L594-L614 (called at screen.c:L221)"]
    E --> P["prevent_current_prompt_from_rewrapping()<br/>kitty/screen.c:L382 → L302-L343"]
    E --> H["PASS 2: realloc_lb()  kitty/screen.c:L384 → L234-L242"]
    H --> I["linebuf_rewrap()  kitty/line-buf.c:L585-L622 (called at screen.c:L240)"]
    G --> J["rewrap_inner(...) NULL, NULL  kitty/history.c:L611"]
    I --> K["rewrap_inner(...) real hb + real track  kitty/line-buf.c:L617"]
    J --> L["kitty/rewrap.h:L56-L96"]
    K --> L
    L --> M["reads next_char_was_wrapped  kitty/data-types.h:L206 (via rewrap.h:L41,L66)"]
    L --> N["copy_range() memcpy cpu+gpu cells  kitty/rewrap.h:L44-L48"]
    L --> O["top overflow → historybuf_add_line() → copy_line()<br/>kitty/rewrap.h:L29-L33; kitty/history.c:L286-L291"]
    L --> Q["TrackCursor remap  kitty/rewrap.h:L84-L89 → screen.c S() clamp L419-L423"]
```

Step by step, top to bottom:

1. **Upstream Python propagation `(inferred, from the call sites)`.** A GUI/layout resize propagates through the tab manager and tabs into each window: `tm.resize()` is called from several sites in `kitty/boss.py` (e.g. window‑resize and font‑change handlers); `kitty/tabs.py` exposes `resize`; and the tab bar resizes its own single‑row screen with `s.resize(1, ncells)` in `kitty/tab_bar.py`. These converge on each window's `set_geometry`.
2. **`Window.set_geometry()`** (`kitty/window.py:L850-L856`). After a guard that skips work when nothing changed (`kitty/window.py:L853`), it calls into the screen and then notifies `on_resize` watchers (`kitty/window.py:L856`):

   ```python
   def set_geometry(self, new_geometry: WindowGeometry) -> None:
       if self.destroyed:
           return
       if self.needs_layout or new_geometry.xnum != self.screen.columns or new_geometry.ynum != self.screen.lines:
           self.screen.resize(max(0, new_geometry.ynum), max(0, new_geometry.xnum))
           self.needs_layout = False
           call_watchers(weakref.ref(self), 'on_resize', {'old_geometry': self.geometry, 'new_geometry': new_geometry})
   ```
   — **`self.screen.resize(max(0, new_geometry.ynum), max(0, new_geometry.xnum))` at `kitty/window.py:L854`** passes **lines (`ynum`) first, columns (`xnum`) second**.
3. **The C binding** `resize(Screen *self, PyObject *args)` (`kitty/screen.c:L3929-L3935`):

   ```c
   static PyObject*
   resize(Screen *self, PyObject *args) {
       unsigned int a=1, b=1;
       if(!PyArg_ParseTuple(args, "|II", &a, &b)) return NULL;
       screen_resize(self, a, b);
       if (PyErr_Occurred()) return NULL;
       Py_RETURN_NONE;
   }
   ```
   parses `"|II"` into `(a, b)` and calls **`screen_resize(self, a, b)` at `kitty/screen.c:L3932`** (`a` = lines, `b` = columns). The method is registered as `MND(resize, METH_VARARGS)` at `kitty/screen.c:L4842`.
4. **`screen_resize()`** (`kitty/screen.c:L345-L463`) clamps `lines`/`columns` to at least 1 (`kitty/screen.c:L348`), then runs **Pass 1** (`realloc_hb`, `kitty/screen.c:L375`) and **Pass 2** (`realloc_lb`, `kitty/screen.c:L384`) as detailed in [§4](#4--linebuf--historybuf-interaction-during-resize-r3), plus cursor remap/clamp (`kitty/screen.c:L419-L427`), optional scrollback‑fill of an enlarged window (`kitty/screen.c:L428-L438`), and prompt copy‑back (`kitty/screen.c:L444-L461`).
5. **`historybuf_rewrap()`** (`kitty/history.c:L594-L614`) and **`linebuf_rewrap()`** (`kitty/line-buf.c:L585-L622`) each dispatch to the shared **`rewrap_inner()`** (`kitty/rewrap.h:L56-L96`) — history with `NULL, NULL` (`kitty/history.c:L611`), visible with a real overflow target and tracker (`kitty/line-buf.c:L617`).
6. **Low‑level cell copy.** `rewrap_inner` moves characters with **`copy_range()`** — two `memcpy`s of the CPU and GPU cell arrays (`kitty/rewrap.h:L44-L48`). Top‑overflow rows are copied into scrollback by **`historybuf_add_line()` → `copy_line()`** (`kitty/history.c:L286-L291`, `copy_line` declared at `kitty/lineops.h:L25`). These are the leaves of the call tree — the actual bytes being redistributed.

### 5.2 Contrast: the PTY winsize path is NOT buffer reflow

A resize also has to tell the child process about the new size. That is a **completely separate** path in `kitty/child-monitor.c` and must not be conflated with the in‑memory reflow above. `pty_resize()` issues the `TIOCSWINSZ` ioctl (`kitty/child-monitor.c:L577-L589`):

```c
static bool
pty_resize(int fd, struct winsize *dim) {
    while(true) {
        if (ioctl(fd, TIOCSWINSZ, dim) == -1) {
            if (errno == EINTR) continue;
            if (errno != EBADF && errno != ENOTTY) {
                log_error("Failed to resize tty associated with fd: %d with error: %s", fd, strerror(errno));
                return false;
            }
        }
        break;
    }
    return true;
}
```

The Python‑facing `resize_pty()` (`kitty/child-monitor.c:L592`, registered as `METHOD(resize_pty, METH_VARARGS)` at `kitty/child-monitor.c:L1922`) looks up the child fd and calls `pty_resize(fd, &dim)` at `kitty/child-monitor.c:L609`. This path merely sends the new window dimensions to the child (which the kernel turns into a `SIGWINCH`); it performs **no `rewrap_inner`, no `LineBuf`/`HistoryBuf` reflow, and touches no continuation bit**. In short: `screen.c` reflows kitty's *own* in‑memory grid; `child-monitor.c` tells the *child program* the terminal got bigger/smaller. R5's data flow concerns only the former.

---

## 6 — Reproduced edge case & root‑cause analysis (R4 + R6)

**R4 asks:** *identify potential issues with line‑continuation state propagation between the two buffers.* **R6 asks:** *reproduce and explain the observed edge cases where reflow does not preserve logical line boundaries.* They share one answer, so they are answered together.

> **Scope reminder.** The behavior below is **characterized, not fixed**. This is a read‑only investigation; proposing or applying a patch to the reflow logic is explicitly out of scope. The purpose here is to *surface and explain* the defect with observed evidence, not to repair it.

### 6.1 The reproduction (investigator‑constructed, not a user example)

The scenario used to trigger the defect was **constructed by this investigation** — the user supplied no examples. It places a single 12‑character logical line `ABCDEFGHIJKL` so that it **straddles the scrollback↔screen boundary**: with a 4‑column, 2‑row screen, drawing 12 characters fills both visible rows and scrolls the first wrapped piece (`ABCD`) into scrollback, leaving `EFGH`/`IJKL` visible. Then the screen is **widened** to 6 columns.

Command: `create_screen(cols=4, lines=2, scrollback=100)` → `draw('ABCDEFGHIJKL')` → `resize(2, 6)` (lines=2, columns=6).

### 6.2 (observed) Scenario A — the defect

```
A.BEFORE (4x2): cursor=(4,1) hist.count=1 cols=4 lines=2
    HIST[0] 'ABCD'     last_char_wrapped=True
    VIS[0]  'EFGH'     is_continued=False last_char_wrapped=True
    VIS[1]  'IJKL'     is_continued=True  last_char_wrapped=False
A.AFTER  resize(2,6): cursor=(2,1) hist.count=1 cols=6 lines=2
    HIST[0] 'ABCD'     last_char_wrapped=False    <== continuation bit True -> False (LOST)
    VIS[0]  'EFGHIJ'   is_continued=False last_char_wrapped=True
    VIS[1]  'KL'       is_continued=True  last_char_wrapped=False
    joined-across-buffers = 'ABCDEFGHIJKL'
```

**What this proves `(observed)`:** Before the resize the logical line is `ABCD`(soft‑wrapped, in scrollback) + `EFGH`(soft‑wrapped) + `IJKL`, i.e. `HIST[0]` has `last_char_wrapped=True`. After widening to 6 columns, the *correct* result would rejoin the line and back‑fill scrollback to `ABCDEF` (pulling `EF` up out of the visible buffer), with `HIST[0].last_char_wrapped` remaining `True`. Instead:

- `HIST[0]` stays `'ABCD'` and its continuation bit **flips `True → False`** — the soft‑wrap link is **lost**.
- the visible buffer reflows **in isolation** to `'EFGHIJ'`/`'KL'`.

No characters are lost (`joined-across-buffers = 'ABCDEFGHIJKL'`), but the **logical line boundary is not preserved**: what was one logical line is now two logical segments — `ABCD` (now hard‑ended) and `EFGHIJKL` (soft‑wrapped).

### 6.3 (observed) Scenario A2 — the control that isolates the cause

The identical logical content, drawn so that it lives **entirely inside the visible buffer** (a 4‑column, 4‑row screen, so nothing spills into scrollback), reflows **correctly** on the same widening:

Command: `create_screen(cols=4, lines=4, scrollback=100)` → `draw('ABCDEFGHIJKL')` → `resize(4, 6)`.
```
A2.BEFORE (4x4): cursor=(4,2) hist.count=0 cols=4 lines=4
    VIS[0]  'ABCD'     is_continued=False last_char_wrapped=True
    VIS[1]  'EFGH'     is_continued=True  last_char_wrapped=True
    VIS[2]  'IJKL'     is_continued=True  last_char_wrapped=False
    VIS[3]  ''         is_continued=False last_char_wrapped=False
A2.AFTER resize(4,6): cursor=(5,1) hist.count=0 cols=6 lines=4
    VIS[0]  'ABCDEF'   is_continued=False last_char_wrapped=True
    VIS[1]  'GHIJKL'   is_continued=True  last_char_wrapped=False
    VIS[2]  ''         is_continued=False last_char_wrapped=False
    VIS[3]  ''         is_continued=False last_char_wrapped=False
    joined-across-buffers = 'ABCDEFGHIJKL'
```

**What this proves `(observed)`:** with the whole line inside **one** buffer, `rewrap_inner` rejoins and re‑splits it correctly to `ABCDEF`/`GHIJKL`, continuation bits intact. Therefore **`rewrap_inner` is correct *within* a single buffer**, and the **A‑vs‑A2 difference isolates the defect to the cross‑buffer (Pass 1 / Pass 2) boundary** — not to the inner algorithm. This contrast is the definitive proof for R6.

### 6.4 Root cause, as a cause → effect chain (R4)

The behavior follows directly from the code traced in [§2](#2--rewrap-c-code-trace-r1) and [§4](#4--linebuf--historybuf-interaction-during-resize-r3). Because Pass 1 (`historybuf_rewrap`) calls `rewrap_inner(self, other, self->count, NULL, NULL, as_ansi_buf)` at **`kitty/history.c:L611`** — overflow target and cursor tracker both `NULL` — it is **self‑contained**: it cannot pull characters *up* from the visible buffer. Walking the straddling line `ABCD`(scrollback) + `EFGH`/`IJKL`(visible) on widening to 6:

1. **Pass 1 rewraps scrollback alone.** For source row `ABCD`, `is_src_line_continued()` reads its wrapped bit = `True` (`kitty/rewrap.h:L66`), so `rewrap_inner` **clears** the source's terminal wrapped bit (`kitty/rewrap.h:L72`) and copies `ABCD` into a fresh 6‑wide destination row.
2. **The continuation bit is never re‑applied.** `ABCD` is only 4 chars, so it **fits in the 6‑wide row without filling it**. The copy loop's fill branch never triggers, so **`next_dest_line(true)` (`kitty/rewrap.h:L81`) is never called** for this row, so the destination row's continuation bit is **never re‑set**. Net: scrollback row `ABCD` ends with `next_char_was_wrapped = False` — exactly the `True → False` flip seen in the [§6.2](#62-observed-scenario-a--the-defect) output. And because `historybuf == NULL` on this pass, there is no mechanism to reach into the visible buffer and pull `EF` up to make `ABCDEF`.
3. **Pass 2 rewraps the visible buffer independently.** `linebuf_rewrap` starts fresh at `EFGH` with **no knowledge** that it continues `ABCD`, and reflows `EFGH`+`IJKL` to `EFGHIJ`/`KL` on its own.

**Net effect (R4):** the continuation (wrapped) state does **not** propagate across the scrollback↔screen boundary in the widening direction. The two independent passes each do a locally‑correct job, but neither is responsible for the *seam* between them: Pass 1 finishes before Pass 2 begins and has no overflow target to receive up‑pulled characters, and Pass 2 has no back‑reference to the tail of scrollback. The `True → False` flip on `HIST[0]` is the concrete, observable footprint of that missing propagation.

Why the narrowing case (Scenario B, [§4.3](#43-observed-narrowing-pushes-overflow-into-scrollback-scenario-b)) is *not* affected: there, the entire logical line originates in the **visible** buffer and is handled by **Pass 2**, which *does* have a live overflow target (`self->historybuf`), so overflow flows correctly **down** into scrollback with continuation bits preserved. The defect is specific to content that is **already split** across the boundary *before* the resize and must be rejoined **upward** — precisely the direction Pass 1's `NULL` overflow target cannot serve.

### 6.5 Determinism

The reproduction is fully deterministic. Scenario A was executed five times and every run produced byte‑identical buffer state and cursor coordinates `(observed)`:

```
Scenario A repeated 5×: all 5 runs identical = True
```

(There is no canonical hash: any digest would be over an investigator‑chosen serialization string and is therefore format‑dependent. The canonical, reproducible fact is that all five runs are identical.)

### 6.6 What would need to change (characterization only — not a fix)

For completeness, and strictly as characterization: a corrective design would have to make the two passes aware of the seam — for example, rewrapping the concatenation of the scrollback tail and the visible head as one unit, or giving the scrollback pass a way to receive up‑pulled characters (a non‑`NULL` sink at `kitty/history.c:L611`) so that a straddling logical line is rejoined before each buffer is re‑split. **No such change is made here** — this remains a read‑only investigation and the defect is left exactly as observed.

---

## 7 — Observed‑vs‑inferred summary & R1–R6 coverage pass

### 7.1 R1–R6 coverage table

| Req | Answered in | Primary evidence | Kind | Rationale (why this answers it) |
|-----|-------------|------------------|------|----------------------------------|
| **R1** — Trace the rewrap C code | [§2](#2--rewrap-c-code-trace-r1) | `rewrap_inner` verbatim (`rewrap.h:L56-L96`); `linebuf_rewrap` (`line-buf.c:L585-L622`); `historybuf_rewrap` (`history.c:L594-L614`); macro overrides (`history.c:L582-L592`) | `(inferred)` code trace, `(observed)` behavior in §3/§4/§6 | One template compiled twice via macro specialization; both specializations quoted in full and their two call sites contrasted (`line-buf.c:L617` vs `history.c:L611`). |
| **R2** — Reflow + continuation + cursor | [§3](#3--reflow-across-new-dimensions-continuation--cursor-r2) | continuation bit (`data-types.h:L206`) read (`rewrap.h:L41,L66`)/cleared (`L72`)/re‑stamped (`L81`,`L93`); `TrackCursor` (`rewrap.h:L50-L53,L84-L89`); `S()` clamp (`screen.c:L419-L423`); Scenarios A2, B, **C**, **C2**, **D** | `(observed)` C/C2/D/A2/B + `(inferred)` code | Shows the exact read/rewrite of the soft‑wrap bit and the cursor remap, with cursor unchanged on unaffected rows (C) and remapped on reflow (C2). |
| **R3** — LineBuf ↔ HistoryBuf interaction | [§4](#4--linebuf--historybuf-interaction-during-resize-r3) | order Pass 1 `realloc_hb` (`screen.c:L375`) then Pass 2 `realloc_lb` (`screen.c:L384`); overflow `next_dest_line`→`historybuf_add_line` (`rewrap.h:L29-L33`; `history.c:L286-L291`); Scenario **B** | `(observed)` B + `(inferred)` code | History rewrapped first, visible second; overflow pushed down into scrollback (hist.count 0→8). |
| **R4** — Continuation‑state propagation issue | [§6.4](#64-root-cause-as-a-cause--effect-chain-r4) | self‑contained history rewrap `NULL,NULL` (`history.c:L611`); missing `next_dest_line(true)` re‑stamp; Scenario **A** `True→False` flip | `(observed)` A + `(inferred)` cause chain | The wrapped bit is not propagated across the boundary upward; the flip on `HIST[0]` is the observable footprint. |
| **R5** — Complete data flow | [§5](#5--complete-data-flow-from-resize-trigger-to-cell-copy-r5) | `Window.set_geometry` (`window.py:L854`) → binding (`screen.c:L3929-L3935`) → `screen_resize` (`screen.c:L345-L463`) → `rewrap_inner` → `copy_range`/`copy_line`; PTY winsize contrast (`child-monitor.c:L577-L609`) | `(inferred)` chain, `(observed)` LINES‑first | Full call chain to the cell‑copy leaves, with the PTY path explicitly separated. |
| **R6** — Reproduce edge cases | [§6.2](#62-observed-scenario-a--the-defect)/[§6.3](#63-observed-scenario-a2--the-control-that-isolates-the-cause) | Scenario **A** (defect) vs **A2** (control) | `(observed)` | Identical text; straddling the boundary breaks, wholly‑in‑one‑buffer works — isolates the defect to the seam. |

### 7.2 Named‑entity checklist (every named item addressed)

| Entity | Where | `file:line` |
|--------|-------|-------------|
| `rewrap_inner` | §2.1, §5 | `rewrap.h:L56-L96` |
| `linebuf_rewrap` | §2.2, §4.1 | `line-buf.c:L585-L622` (call `L617`) |
| `historybuf_rewrap` | §2.3, §4.1 | `history.c:L594-L614` (call `L611`) |
| `next_dest_line` | §2.1, §4.2, §6.4 | `rewrap.h:L24-L38` (default), `history.c:L588` (override), fill call `rewrap.h:L81`, hard break `L93` |
| `is_src_line_continued` | §2.1, §3.1 | `rewrap.h:L40-L42` (read `L41`, used `L66`) |
| `copy_range` | §2.1, §5.1 | `rewrap.h:L44-L48` |
| `copy_line` | §4.2, §5.1 | `history.c:L289`; declared `lineops.h:L25` |
| `historybuf_add_line` | §4.2 | `history.c:L286-L291` |
| `historybuf_push` | §4.2 | `history.c:L275-L284` (push `L288`) |
| `TrackCursor` | §3.2 | `rewrap.h:L50-L53,L84-L89` |
| `next_char_was_wrapped` | §3.1 | `data-types.h:L206` (mask `L214`) |
| `screen_resize` | §4.1, §5.1 | `screen.c:L345-L463` |
| `realloc_hb` | §4.1 | `screen.c:L216-L223` (call `L375`, `historybuf_rewrap` `L221`) |
| `realloc_lb` | §4.1 | `screen.c:L234-L242` (main call `L384`, alt/NULL `L394`, `linebuf_rewrap` `L240`) |
| `prevent_current_prompt_from_rewrapping` | §4.1 | `screen.c:L302-L343` (call `L382`) |
| `S()` clamp | §3.2 | `screen.c:L419-L423` |
| `is_beyond_content` | §3.2 | `screen.c:L424-L427` |
| `Window.set_geometry` | §5.1 | `window.py:L850-L856` (resize `L854`) |
| PTY winsize path | §5.2 | `child-monitor.c:L577` (`pty_resize`), `L592` (`resize_pty`), `L609`, `L1922` |
| `map_src_index` (circular) | §2.3 | `history.c:L584` |
| `linebuf_set_last_char_as_continuation` | §2.2 | `line-buf.c:L193-L198` (set `L196`) |
| `history_buf_set_last_char_as_continuation` | §2.3 | `history.c:L302-L307` (set `L305`) |
| `last_char_has_wrapped_flag` (probe getter) | §1.3 | `line.c:L427` (reads bit `L429`) |
| fast path | §2.2/§2.3/§3.3 | `line-buf.c:L591-L598`; `history.c:L597-L606` |

### 7.3 Condition coverage (every implied condition exercised, before/after)

| Condition | Scenario(s) | Before → After | Result |
|-----------|-------------|----------------|--------|
| **Widen (in‑buffer)** | A2, C | 4×4 → 6 cols; `'ab'` widen | correct reflow / cursor preserved `(observed)` |
| **Widen (cross‑buffer)** | A | 4×2 → 6 cols | continuation lost `(observed, defect)` |
| **Narrow** | B, C2 | 5×5 → 2 cols; 15 chars → 2 cols | overflow→scrollback OK / cursor remapped `(observed)` |
| **Cross‑buffer boundary** | A vs A2 | straddling vs in‑buffer | isolates defect to the seam `(observed)` |
| **Cursor** | C, C2 | (2,0)→(2,0); (5,2)→(1,4) | preserved / remapped `(observed)` |
| **Fast path (no reflow)** | D | 5×5 → 5×5 | byte‑identical, cursor.x clamp 5→4 `(observed)` |

### 7.4 Citation verification

Every `file:line` reference in this document was independently re‑verified against the checked‑out source at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` during this investigation; all were exact. One note carried from the plan: `screen_resize`'s body spans `kitty/screen.c:L345-L463` (its closing brace is at L463; some summaries round the end to L465). The described symbols and behaviors are unaffected.

### 7.5 Observed vs. inferred — the short version

- **Observed (runtime‑confirmed, with output blocks):** the two‑pass ordering's effect on state (A, A2, B); continuation‑bit read/rewrite outcomes (A, A2, B); cursor preserve/remap (C, C2); fast path (D); the cross‑buffer continuation loss and its `True → False` footprint (A); determinism (5× identical); the LINES‑first argument order (empirically, `resize(2,6)` → 6 columns); and the **build outcome in this environment** (`CI=true python3 setup.py build` → exit 0, `.so` = 1,253,792 bytes, tree pristine).
- **Inferred (code‑reading only, labeled inline):** the exact upstream `boss.py`/`tabs.py`/`tab_bar.py` propagation into `set_geometry`; that warning flags never change C semantics; and the cross‑environment `-Werror`/`wl_window.c` contingency that the `--ignore-compiler-warnings` flag would neutralize (it did **not** occur here because the Wayland backend was disabled).

---

### Appendix — scope & fidelity attestation

- Exactly **one** file was produced: this document. No kitty `.c`/`.h`/`.py`/`.rst`/build file was created, modified, or deleted; `git status --porcelain` is clean apart from this deliverable. Temporary probe scripts lived outside the repository (`/tmp/kitty_reflow_probe/`) and were removed.
- The candidate defect ([§6](#6--reproduced-edge-case--root-cause-analysis-r4--r6)) is **characterized, not fixed**; no patch is proposed as the answer.
- All runtime evidence came from the **real** `Screen.draw` / `Screen.resize` entry point via the headless `kitty_tests.BaseTest.create_screen` harness — no remote‑control/debug bypass, no fallback or synthetic values.
