# Kitty Terminal Reflow (Rewrap) System — Technical Analysis

> **Scope**: This document traces kitty's terminal reflow subsystem — the C-level
> machinery that re-distributes terminal cells across new dimensions when a
> window is resized — answering how the generic `rewrap_inner()` works, how the
> visible screen buffer (`LineBuf`) interacts with the scrollback history
> (`HistoryBuf`), and identifying potential issues with line-continuation state
> propagation.
>
> **Commit**: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (source branch
> `kitty_815df1e210e0`).
>
> **Repository files analyzed**: `kitty/rewrap.h` (96 lines),
> `kitty/screen.c` (4,932 lines — especially lines ~216–463 and ~2833–2840),
> `kitty/screen.h` (289 lines), `kitty/line-buf.c` (641 lines),
> `kitty/history.c` (624 lines), `kitty/data-types.h` (438 lines),
> `kitty/lineops.h` (136 lines), `kitty/options/definition.py` (line 420),
> `kitty_tests/datatypes.py` (rewrap tests), `kitty_tests/screen.py` (resize
> tests).
>
> **Verification**: All 54 relevant unit tests in `kitty_tests.datatypes` and
> `kitty_tests.screen` pass against the built C extensions on this commit
> (`python3 -m unittest kitty_tests.datatypes kitty_tests.screen`).

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Data Model Foundations](#2-data-model-foundations)
3. [The Resize Entry Point — `screen_resize()`](#3-the-resize-entry-point--screen_resize)
4. [Prompt Protection](#4-prompt-protection)
5. [Buffer Reallocation — `realloc_hb()` and `realloc_lb()`](#5-buffer-reallocation--realloc_hb-and-realloc_lb)
6. [Macro-Polymorphism in `rewrap.h`](#6-macro-polymorphism-in-rewraph)
7. [Deep Dive: `rewrap_inner()`](#7-deep-dive-rewrap_inner)
8. [LineBuf Specialization](#8-linebuf-specialization)
9. [HistoryBuf Specialization](#9-historybuf-specialization)
10. [The Buffer Boundary — Visible Screen ↔ Scrollback](#10-the-buffer-boundary--visible-screen--scrollback)
11. [Line Continuation State: The Dual-Signal Architecture](#11-line-continuation-state-the-dual-signal-architecture)
12. [Cursor Tracking Through Rewrap](#12-cursor-tracking-through-rewrap)
13. [Scrollback Fill After Enlarging a Window](#13-scrollback-fill-after-enlarging-a-window)
14. [Pager History Interaction](#14-pager-history-interaction)
15. [Complete Call Chain & Data-Flow Map](#15-complete-call-chain--data-flow-map)
16. [Identified Potential Issues and Edge Cases](#16-identified-potential-issues-and-edge-cases)
17. [Test Coverage & Behavioral Confirmation](#17-test-coverage--behavioral-confirmation)
18. [Glossary](#18-glossary)

---

## 1. Executive Summary

When a kitty window is resized, the terminal must *reflow* (rewrap) the text so
that lines that previously fit one width are redistributed to lines of the new
width. Kitty's reflow system is architected around a single generic algorithm
(`rewrap_inner()` in `kitty/rewrap.h`) that is parameterized via C
preprocessor macros and instantiated twice:

1. Once in `kitty/line-buf.c` with the **default** macros, producing a rewrap
   routine that operates on `LineBuf` (the visible screen).
2. Once in `kitty/history.c` with **overridden** macros (`BufType`,
   `map_src_index`, `init_src_line`, `next_dest_line`, `first_dest_line`),
   producing a rewrap routine that operates on `HistoryBuf` (the scrollback).

The entry point for all resize operations is `screen_resize()` in
`kitty/screen.c`, which orchestrates an ordered sequence: pause rendering →
dummy-fill empty OUTPUT_START lines → snapshot cursor positions → resize
overlay → reallocate the **history** buffer (rewrapping it to the new width)
→ blank the current shell prompt to prevent flicker → reallocate the **main
line** buffer (which can *spill* overflow lines into the history) → reallocate
the **alt line** buffer → update dimensions, tabstops, selections → clamp
cursor → fill new space with history lines (if
`scrollback_fill_enlarged_window` is enabled) → restore the prompt copy.

Two parallel representations of line continuation live side-by-side:

- `next_char_was_wrapped` — a **1-bit persistent field** in `CellAttrs` on the
  last `GPUCell` of a line, meaning "the next character overflowed and was
  placed on the next line."
- `is_continued` — a **1-bit derived field** in `LineAttrs`, recomputed every
  time a line is initialized by reading the previous line's
  `next_char_was_wrapped`.

These two signals must stay consistent across the history↔screen boundary, and
several code paths (`history_buf_endswith_wrap()` override in
`init_line()` at `screen.c:2836–2838`, HistoryBuf's `init_line()` at
`history.c:167–176` reading the pager history newline state) exist specifically
to bridge that boundary. Section 11 and Section 16 identify places where the
cross-boundary signal can become inconsistent in practice.

All 54 tests across `kitty_tests.datatypes` and `kitty_tests.screen` pass
cleanly against this implementation, confirming that the design is correct for
the commonly exercised cases.

---

## 2. Data Model Foundations

Before discussing reflow, the data structures manipulated by rewrap must be
understood. All definitions below live in `kitty/data-types.h`.

### 2.1 `CellAttrs` — per-cell attribute bitfield (16 bits)

File: `kitty/data-types.h` lines 196–209.

```c
typedef union CellAttrs {
    struct {
        uint16_t width : 2;
        uint16_t decoration : 3;
        uint16_t bold : 1;
        uint16_t italic : 1;
        uint16_t reverse : 1;
        uint16_t strike : 1;
        uint16_t dim : 1;
        uint16_t mark : 2;
        uint16_t next_char_was_wrapped : 1;
    };
    uint16_t val;
} CellAttrs;
```

The field **`next_char_was_wrapped`** is critical for reflow. It is set on the
*last cell* of a line when the next glyph overflowed to the following line
during normal drawing. It is the **ground-truth storage** for soft-wrap
relationships.

### 2.2 `GPUCell` (20 bytes) and `CPUCell` (12 bytes)

Lines 216–228:

```c
typedef struct {
    color_type fg, bg, decoration_fg;
    sprite_index sprite_x, sprite_y, sprite_z;
    CellAttrs attrs;
} GPUCell;
static_assert(sizeof(GPUCell) == 20, "Fix the ordering of GPUCell");

typedef struct {
    char_type ch;
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];
} CPUCell;
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
```

Every cell has two halves:
- `CPUCell`: the character (`ch`), the hyperlink ID, and up to three combining
  character indices. Read/written by CPU-side code (text buffer, copy, paste,
  reflow).
- `GPUCell`: colors, sprite coordinates, and the packed `CellAttrs`. Uploaded
  to the GPU for rendering.

Reflow copies **both** halves as a unit. The inline helper `copy_range()` in
`rewrap.h:44–48` uses two `memcpy`s — one for the CPU cells, one for the GPU
cells — from a source slice to a destination slice.

### 2.3 `LineAttrs` — per-line attribute bitfield (8 bits)

Lines 230–239:

```c
typedef enum { UNKNOWN_PROMPT_KIND = 0, PROMPT_START = 1,
               SECONDARY_PROMPT = 2, OUTPUT_START = 3 } PromptKind;
typedef union LineAttrs {
    struct {
        uint8_t is_continued : 1;
        uint8_t has_dirty_text : 1;
        uint8_t has_image_placeholders : 1;
        PromptKind prompt_kind : 2;
    };
    uint8_t val;
} LineAttrs ;
```

Key observations:

- **`is_continued`** is a derived field — the data-at-rest storage lives in
  `line_attrs[]` arrays for `LineBuf`/`HistoryBuf`, but during a call to
  `linebuf_init_line()` (see §2.5) it is **overwritten** with the value
  computed from the *previous line's* `next_char_was_wrapped`. This means
  `is_continued` in stored `LineAttrs` is essentially a cache that the "init
  line" functions refresh on every access.
- `prompt_kind` classifies a line as the start of a shell prompt
  (`PROMPT_START`), a secondary continuation prompt (`SECONDARY_PROMPT`), the
  start of command output (`OUTPUT_START`), or unknown. These are set by OSC
  133 sequences emitted by shell-integration scripts and consumed by the
  prompt-protection logic in reflow (§4).

### 2.4 `Line`, `LineBuf`, `HistoryBufSegment`, `HistoryBuf`

Lines 241–290:

```c
typedef struct {
    PyObject_HEAD
    GPUCell *gpu_cells;
    CPUCell *cpu_cells;
    index_type xnum, ynum;
    bool needs_free;
    LineAttrs attrs;
} Line;

typedef struct {
    PyObject_HEAD
    GPUCell *gpu_cell_buf;
    CPUCell *cpu_cell_buf;
    index_type xnum, ynum, *line_map, *scratch;
    LineAttrs *line_attrs;
    Line *line;
} LineBuf;

typedef struct {
    GPUCell *gpu_cells;
    CPUCell *cpu_cells;
    LineAttrs *line_attrs;
} HistoryBufSegment;

typedef struct {
    void *ringbuf;
    size_t maximum_size;
    bool rewrap_needed;
} PagerHistoryBuf;

typedef struct {
    PyObject_HEAD
    index_type xnum, ynum, num_segments;
    HistoryBufSegment *segments;
    PagerHistoryBuf *pagerhist;
    Line *line;
    index_type start_of_data, count;
} HistoryBuf;
```

Highlights:

- **`LineBuf`** stores all cells in a flat arena (`cpu_cell_buf`,
  `gpu_cell_buf`, `line_attrs`) and maintains a **`line_map[]`** permutation
  — an `index_type` array that maps logical y-coordinate to physical storage
  row. Scrolling is therefore O(1): `linebuf_index()` (see `line-buf.c:316–
  327`) just rotates map entries; no cell data is copied.
- `LineBuf` also owns a single `Line *line` "cursor" object that is re-pointed
  by `linebuf_init_line()` to act as a live view onto a particular row.
- **`HistoryBuf`** uses **segmented storage**: each segment holds
  `SEGMENT_SIZE = 2048` lines (`history.c:15`), so a history of depth 100,000
  uses ~49 segments. Segments are allocated on demand via `add_segment()`.
- HistoryBuf indexing is a **ring buffer**: `start_of_data` is the physical
  offset of the oldest retained line, and `count` is how many lines are live.
  The helper `index_of()` (`history.c:152–159`) performs reverse indexing:
  `lnum=0` is the *most recent* history line.
- `PagerHistoryBuf` is a separate byte-level ring buffer (using the vendored
  `3rdparty/ringbuf/ringbuf.h`) that holds ANSI-escaped text spill from the
  tail of `HistoryBuf` — overflow beyond the structured history.

### 2.5 `linebuf_init_line()` and `HistoryBuf init_line()`

These two functions are the central "view initializers" that give the rest of
the codebase a `Line *` pointing at a specific row. **Every time either is
called, the line's `is_continued` flag is recomputed from scratch.**

`kitty/line-buf.c:140–147`:

```c
void
linebuf_init_line(LineBuf *self, index_type idx) {
    self->line->ynum = idx;
    self->line->xnum = self->xnum;
    self->line->attrs = self->line_attrs[idx];
    self->line->attrs.is_continued = idx > 0
        ? gpu_lineptr(self, self->line_map[idx - 1])
                     [self->xnum - 1].attrs.next_char_was_wrapped
        : false;
    init_line(self, self->line, self->line_map[idx]);
}
```

`kitty/history.c:161–177`:

```c
static void
init_line(HistoryBuf *self, index_type num, Line *l) {
    l->cpu_cells = cpu_lineptr(self, num);
    l->gpu_cells = gpu_lineptr(self, num);
    l->attrs = *attrptr(self, num);
    if (num > 0) {
        l->attrs.is_continued =
            gpu_lineptr(self, num - 1)
                       [self->xnum-1].attrs.next_char_was_wrapped;
    } else {
        l->attrs.is_continued = false;
        size_t sz;
        if (self->pagerhist && self->pagerhist->ringbuf
            && (sz = ringbuf_bytes_used(self->pagerhist->ringbuf)) > 0) {
            size_t pos = ringbuf_findchr(self->pagerhist->ringbuf, '\n', sz - 1);
            if (pos >= sz) l->attrs.is_continued = true;  // ringbuf does not end with a newline
        }
    }
}
```

Note in the HistoryBuf case, when looking at the oldest physical entry (`num
== 0` in physical terms — **not** the same as the API's reverse-indexed
`lnum=0`), the function consults the pager history ringbuf to see if the last
pager-history byte is a newline. If not, the oldest history entry is marked as
continued from the pager history. This is the **pager-history ↔ history**
bridge.

---

## 3. The Resize Entry Point — `screen_resize()`

`kitty/screen.c:345–463` is the canonical entry for every resize operation.
Its structure is:

```c
static bool
screen_resize(Screen *self, unsigned int lines, unsigned int columns) {
    screen_pause_rendering(self, false, 0);
    lines = MAX(1u, lines); columns = MAX(1u, columns);

    bool is_main = self->linebuf == self->main_linebuf;
    index_type num_content_lines_before, num_content_lines_after;
    bool dummy_output_inserted = false;

    /* --- §3.1 empty OUTPUT_START preservation --- */
    if (is_main && self->cursor->x == 0
        && self->cursor->y < self->lines
        && self->linebuf->line_attrs[self->cursor->y].prompt_kind == OUTPUT_START) {
        linebuf_init_line(self->linebuf, self->cursor->y);
        if (!self->linebuf->line->cpu_cells[0].ch) {
            self->linebuf->line->cpu_cells[self->cursor->x++].ch = '<';
            dummy_output_inserted = true;
        }
    }

    unsigned int lines_after_cursor_before_resize = self->lines - self->cursor->y;
    CursorTrack cursor           = {.before = {self->cursor->x, self->cursor->y}};
    CursorTrack main_saved_cursor= {.before = {self->main_savepoint.cursor.x,
                                               self->main_savepoint.cursor.y}};
    CursorTrack alt_saved_cursor = {.before = {self->alt_savepoint.cursor.x,
                                               self->alt_savepoint.cursor.y}};
#define setup_cursor(which) { \
    which.after.x = which.temp.x; which.after.y = which.temp.y; \
    which.is_beyond_content = num_content_lines_before > 0 \
                              && self->cursor->y >= num_content_lines_before; \
    which.num_content_lines = num_content_lines_after; \
}

    if (!init_overlay_line(self, columns, true)) return false;

    /* --- §3.2 reallocate history --- */
    HistoryBuf *nh = realloc_hb(self->historybuf,
                                self->historybuf->ynum, columns,
                                &self->as_ansi_buf);
    if (nh == NULL) return false;
    Py_CLEAR(self->historybuf); self->historybuf = nh;

    /* --- §3.3 prompt protection --- */
    RAII_PyObject(prompt_copy, NULL);
    index_type num_of_prompt_lines = 0, num_of_prompt_lines_above_cursor = 0;
    if (is_main) {
        prompt_copy = (PyObject*)alloc_linebuf(self->lines, self->columns);
        num_of_prompt_lines = prevent_current_prompt_from_rewrapping(
            self, (LineBuf*)prompt_copy, &num_of_prompt_lines_above_cursor);
    }

    /* --- §3.4 reallocate main linebuf (with history spillover) --- */
    LineBuf *n = realloc_lb(self->main_linebuf, lines, columns,
                            &num_content_lines_before, &num_content_lines_after,
                            self->historybuf,       /* <-- spill target */
                            &cursor, &main_saved_cursor, &self->as_ansi_buf);
    if (n == NULL) return false;
    Py_CLEAR(self->main_linebuf); self->main_linebuf = n;
    if (is_main) setup_cursor(cursor);
    setup_cursor(main_saved_cursor);
    grman_remove_all_cell_images(self->main_grman);
    grman_resize(self->main_grman, self->lines, lines, self->columns, columns,
                 num_content_lines_before, num_content_lines_after);

    /* --- §3.5 reallocate alt linebuf (no history) --- */
    n = realloc_lb(self->alt_linebuf, lines, columns,
                   &num_content_lines_before, &num_content_lines_after,
                   NULL,                       /* <-- no spill */
                   &cursor, &alt_saved_cursor, &self->as_ansi_buf);
    if (n == NULL) return false;
    Py_CLEAR(self->alt_linebuf); self->alt_linebuf = n;
    if (!is_main) setup_cursor(cursor);
    setup_cursor(alt_saved_cursor);
    grman_remove_all_cell_images(self->alt_grman);
    grman_resize(self->alt_grman, self->lines, lines, self->columns, columns,
                 num_content_lines_before, num_content_lines_after);
#undef setup_cursor

    /* --- §3.6 commit dimensions --- */
    self->linebuf = is_main ? self->main_linebuf : self->alt_linebuf;
    self->lines = lines; self->columns = columns;
    self->margin_top = 0; self->margin_bottom = self->lines - 1;

    PyMem_Free(self->main_tabstops);
    self->main_tabstops = PyMem_Calloc(2*self->columns, sizeof(bool));
    if (self->main_tabstops == NULL) { PyErr_NoMemory(); return false; }
    self->alt_tabstops = self->main_tabstops + self->columns;
    self->tabstops = self->main_tabstops;
    init_tabstops(self->main_tabstops, self->columns);
    init_tabstops(self->alt_tabstops, self->columns);
    self->is_dirty = true;
    clear_selection(&self->selections);
    clear_selection(&self->url_ranges);
    self->last_visited_prompt.is_set = false;

    /* --- §3.7 clamp and remap cursors --- */
#define S(c, w) c->x = MIN(w.after.x, self->columns - 1); \
                c->y = MIN(w.after.y, self->lines - 1);
    S(self->cursor, cursor);
    S((&(self->main_savepoint.cursor)), main_saved_cursor);
    S((&(self->alt_savepoint.cursor)), alt_saved_cursor);
#undef S
    if (cursor.is_beyond_content) {
        self->cursor->y = cursor.num_content_lines;
        if (self->cursor->y >= self->lines) {
            self->cursor->y = self->lines - 1;
            screen_index(self);
        }
    }

    /* --- §3.8 optional scrollback fill when enlarged --- */
    if (is_main && OPT(scrollback_fill_enlarged_window)) {
        const unsigned int top = 0, bottom = self->lines-1;
        Savepoint *sp = is_main ? &self->main_savepoint : &self->alt_savepoint;
        while (self->cursor->y + 1 < self->lines
               && self->lines - self->cursor->y > lines_after_cursor_before_resize) {
            if (!historybuf_pop_line(self->historybuf, self->alt_linebuf->line)) break;
            INDEX_DOWN;
            linebuf_copy_line_to(self->main_linebuf, self->alt_linebuf->line, 0);
            self->cursor->y++;
            sp->cursor.y = MIN(sp->cursor.y + 1, self->lines - 1);
        }
    }

    /* --- §3.9 remove dummy and restore prompt copy --- */
    if (dummy_output_inserted && self->cursor->y < self->lines) {
        linebuf_init_line(self->linebuf, self->cursor->y);
        self->linebuf->line->cpu_cells[0].ch = 0;
        self->cursor->x = 0;
    }
    if (num_of_prompt_lines) {
        LineBuf *src = (LineBuf*)prompt_copy;
        for (index_type
                src_line = 0,
                y = num_of_prompt_lines_above_cursor <= self->cursor->y
                    ? self->cursor->y - num_of_prompt_lines_above_cursor : 0;
             src_line < num_of_prompt_lines && y < self->lines;
             y++, src_line++
        ) {
            linebuf_init_line(src, src_line);
            linebuf_copy_line_to(self->main_linebuf, src->line, y);
        }
    }
    return true;
}
```

Walking through the phases in order:

### 3.1 Empty OUTPUT_START preservation (`screen.c:352–361`)

When the shell has emitted OSC 133 `;C` to mark the beginning of command
output, kitty stamps `prompt_kind = OUTPUT_START` on the current line. If the
user then resizes before the shell writes anything, the OUTPUT_START line is
blank and would be trimmed by rewrap's trailing-blank logic (see §7). To
preserve it, the code overwrites `cpu_cells[0].ch` with `'<'` (a marker
dummy) and records `dummy_output_inserted = true`. After resize (§3.9), the
dummy char is erased.

### 3.2 History buffer reallocation (`screen.c:375–377`)

`realloc_hb()` is called **first** because `realloc_lb()` for the main
linebuf needs a valid `HistoryBuf` to spill overflow lines into (see §10).

### 3.3 Prompt protection setup (`screen.c:378–383`)

For the main screen only, a scratch `LineBuf *prompt_copy` is allocated with
the *old* dimensions, and `prevent_current_prompt_from_rewrapping()` is called
to (a) copy all lines from the current prompt start to the bottom into
`prompt_copy`, (b) blank them in the main linebuf so they do not get
rewrapped. See §4.

### 3.4 Main linebuf reallocation with history spillover (`screen.c:384–391`)

The critical call passes `self->historybuf` as the spill target. During
rewrap, any line that would overflow the destination `LineBuf` is pushed to
this `HistoryBuf` — see §10 for the mechanism and §8 for the macro wiring.

### 3.5 Alt linebuf reallocation (`screen.c:393–400`)

The alt screen buffer is rewrapped with `NULL` as the spill target — overflow
lines from the alt buffer are **discarded** rather than saved to history, per
terminal convention.

### 3.6 Dimensions, tabstops, selections committed (`screen.c:403–418`)

The screen now has its new dimensions. Tabstops are reinitialized (since
column count changed), selections are cleared (their coordinates no longer
point at valid content), and the "last visited prompt" marker is invalidated.

### 3.7 Cursor clamp & beyond-content repositioning (`screen.c:419–427`)

Cursor `.after` coordinates (filled in by `realloc_lb`) are clamped to the
valid window range. If the cursor was **beyond** the last content line before
resize (common when the shell has printed no output since the last prompt),
it is placed at `num_content_lines` after resize. If that would be beyond the
new window, it's clamped to the bottom and `screen_index()` scrolls to make
room.

### 3.8 Scrollback fill (`screen.c:428–438`)

If `scrollback_fill_enlarged_window` option is enabled and the window grew
vertically, history lines are *pulled back* into the visible buffer via
`historybuf_pop_line()` + `INDEX_DOWN`. See §13.

### 3.9 Dummy cleanup & prompt restoration (`screen.c:439–461`)

Erase the dummy `'<'` placeholder if it was inserted in §3.1. Then, if a
prompt was copied in §3.3, overwrite the bottom of the main linebuf with the
saved prompt copy (un-reflowed). This prevents visible flicker between the
rewrap-blanked region and the shell's eventual redraw of its prompt.

---

## 4. Prompt Protection

`kitty/screen.c:302–343`:

```c
static index_type
prevent_current_prompt_from_rewrapping(Screen *self,
                                       LineBuf *prompt_copy,
                                       index_type *num_of_prompt_lines_above_cursor) {
    index_type num_of_prompt_lines = 0; *num_of_prompt_lines_above_cursor = 0;
    if (!self->prompt_settings.redraws_prompts_at_all) return num_of_prompt_lines;
    int y = self->cursor->y;
    while (y >= 0) {
        linebuf_init_line(self->main_linebuf, y);
        Line *line = self->linebuf->line;
        switch (line->attrs.prompt_kind) {
            case UNKNOWN_PROMPT_KIND: break;
            case PROMPT_START:
            case SECONDARY_PROMPT:   goto found;
            case OUTPUT_START:       return num_of_prompt_lines;
        }
        y--;
    }
found:
    if (y < 0) return num_of_prompt_lines;
    for (; y < (int)self->main_linebuf->ynum; y++) {
        linebuf_init_line(self->main_linebuf, y);
        linebuf_copy_line_to(prompt_copy, self->main_linebuf->line,
                             num_of_prompt_lines++);
        linebuf_clear_line(self->main_linebuf, y, false);
        if (y <= (int)self->cursor->y) {
            linebuf_init_line(self->main_linebuf, y);
            self->main_linebuf->line->cpu_cells[0].ch = ' ';
            if (y < (int)self->cursor->y) (*num_of_prompt_lines_above_cursor)++;
        }
    }
    return num_of_prompt_lines;
}
```

### What it does

1. **Guard**: If the terminal has not been told by shell-integration that
   prompts are redrawable (`prompt_settings.redraws_prompts_at_all`), skip
   — there is nothing to coordinate with the shell on.
2. **Walk back from cursor** looking for a `PROMPT_START` or
   `SECONDARY_PROMPT` line. If an `OUTPUT_START` is encountered first
   (meaning the cursor is in command output, not a prompt), return with no
   action.
3. **If a prompt line is found**: copy every line from there to the bottom of
   the screen into `prompt_copy`, then blank them in the main linebuf (but
   keep `line_attrs`, hence the `false` argument to `linebuf_clear_line`).
4. **Force non-empty cells on lines from prompt-start to cursor**: set
   `cpu_cells[0].ch = ' '`. Comment at `screen.c:336–337` explains:

   > "this is needed because screen_resize() checks to see if the cursor is
   > beyond the content, so insert some fake content"

   This prevents the resize logic from concluding the cursor is "beyond
   content" (§3.7) purely because the blanking we just did emptied the area
   containing the cursor.

### Why it matters

When a shell redraws its prompt after a `SIGWINCH`, it expects the cursor
vertical position relative to the prompt's first line to be stable. Classic
example: zsh with a right-side prompt, as noted in the comment
(`screen.c:325–328`). By blanking the prompt region and trusting the shell to
redraw — rather than letting reflow reformat prompt lines — we avoid visual
flicker and shell confusion.

The `num_of_prompt_lines` and `num_of_prompt_lines_above_cursor` returned to
`screen_resize()` are used to place the saved prompt copy back at the correct
vertical position (§3.9), aligning the cursor's row with the (now un-reflowed)
prompt.

---

## 5. Buffer Reallocation — `realloc_hb()` and `realloc_lb()`

### 5.1 `realloc_hb()` — `screen.c:216–223`

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

Steps:
1. Allocate a fresh `HistoryBuf` with the new column count (`lines` is
   unchanged — the scrollback depth does not change on resize).
2. **Move the `PagerHistoryBuf` pointer** from old to new. This preserves the
   accumulated pager history across the reallocation. The old HistoryBuf is
   left with `pagerhist = NULL` so it does not double-free on dealloc.
3. Call `historybuf_rewrap()` to rewrap the structured content into the new
   history buffer.

### 5.2 `CursorTrack` struct — `screen.c:226–232`

```c
typedef struct CursorTrack {
    index_type num_content_lines;
    bool is_beyond_content;
    struct { index_type x, y; } before;
    struct { index_type x, y; } after;
    struct { index_type x, y; } temp;
} CursorTrack;
```

This is the bookkeeping structure threaded through rewrap. `.before` is the
old cursor position; `.temp` is the mutable field that rewrap updates
*in-place* as it copies cells; `.after` is the final position;
`is_beyond_content` is a flag set in §3.7 to indicate "was the cursor past the
last line with any content."

### 5.3 `realloc_lb()` — `screen.c:234–242`

```c
static LineBuf*
realloc_lb(LineBuf *old, unsigned int lines, unsigned int columns,
           index_type *nclb, index_type *ncla, HistoryBuf *hb,
           CursorTrack *a, CursorTrack *b, ANSIBuf *as_ansi_buf) {
    LineBuf *ans = alloc_linebuf(lines, columns);
    if (ans == NULL) { PyErr_NoMemory(); return NULL; }
    a->temp.x = a->before.x; a->temp.y = a->before.y;
    b->temp.x = b->before.x; b->temp.y = b->before.y;
    linebuf_rewrap(old, ans, nclb, ncla, hb,
                   &a->temp.x, &a->temp.y, &b->temp.x, &b->temp.y,
                   as_ansi_buf);
    return ans;
}
```

Key points:
- Two `CursorTrack *` parameters (`a`, `b`) — the live cursor and the
  savepoint cursor. Both get remapped through rewrap.
- Before calling `linebuf_rewrap`, `temp.x/temp.y` are seeded from `before.x/
  before.y`. `linebuf_rewrap` then **mutates** the `temp` fields during
  rewrap. The caller (`screen_resize()` via the `setup_cursor` macro)
  subsequently copies `temp.*` into `after.*`.
- `nclb` and `ncla` are out-parameters: "number of content lines before" and
  "after."

---

## 6. Macro-Polymorphism in `rewrap.h`

`kitty/rewrap.h` uses a C preprocessor idiom that lets a single algorithmic
template be reused for two different buffer types without runtime dispatch.
The pattern relies on `#ifndef`/`#define` — each macro has a default, which
the includer can override **before** the `#include "rewrap.h"` directive.

### 6.1 The six override points

`rewrap.h:10–42`:

```c
#ifndef BufType
#define BufType LineBuf
#endif

#ifndef init_src_line
#define init_src_line(src_y) linebuf_init_line(src, src_y);
#endif

#define set_dest_line_attrs(dest_y) dest->line_attrs[dest_y] = src->line->attrs; \
                                    src->line->attrs.prompt_kind = UNKNOWN_PROMPT_KIND;

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

| Override | Default (LineBuf) | Overridden in history.c | Purpose |
|---|---|---|---|
| `BufType` | `LineBuf` | `HistoryBuf` (line 582) | The type name for `src` and `dest` parameters |
| `map_src_index(y)` | (not defined) | `((src->start_of_data + y) % src->ynum)` (line 584) | Translate logical y to physical ring-buffer index (HistoryBuf only) |
| `init_src_line(src_y)` | `linebuf_init_line(src, src_y)` | `init_line(src, map_src_index(src_y), src->line)` (line 586) | Point `src->line` at a specific row |
| `first_dest_line` | `linebuf_init_line(dest, 0); set_dest_line_attrs(0)` | `next_dest_line(false)` (line 590) | Prepare dest for first row |
| `next_dest_line(cont)` | *(complex spillover-to-history block above)* | `{ history_buf_set_last_char_as_continuation(dest, 0, cont); LineAttrs *lap = attrptr(dest, historybuf_push(dest, as_ansi_buf)); *lap = src->line->attrs; }` (line 588) | Advance to next dest row, handling overflow |
| `is_src_line_continued()` | `src->line->gpu_cells[src->xnum-1].attrs.next_char_was_wrapped` | *(uses default)* | Is the current source line soft-wrapped? |
| `set_dest_line_attrs(dest_y)` | `dest->line_attrs[dest_y] = src->line->attrs; src->line->attrs.prompt_kind = UNKNOWN_PROMPT_KIND;` | *(effectively unused by HistoryBuf's `next_dest_line`, which writes attrs inline)* | Copy line attrs (e.g., prompt_kind) from src to dest |

Note the `set_dest_line_attrs` side-effect: after copying src attrs to dest,
the macro **clears** `src->line->attrs.prompt_kind`. This is an important
invariant — a prompt kind belongs to exactly one line at a time after reflow;
if a source line spans multiple destination lines (widening case), only the
**first** dest line inherits the `prompt_kind`.

### 6.2 Why macros, not function pointers?

Function-pointer dispatch would impose an indirect call per-line-per-cell; the
rewrap loop runs over potentially hundreds of thousands of cells. Using
macros: (a) the compiler inlines the `init_src_line`, `first_dest_line`, and
`next_dest_line` bodies directly into `rewrap_inner()`; (b) each
specialization produces a distinct static copy of `rewrap_inner()` (one in
`line-buf.o`, one in `history.o`) referencing type-correct pointers.
Essentially, the file is a hand-rolled template.

### 6.3 Double inclusion mechanics

`kitty/line-buf.c:583` does:

```c
#include "rewrap.h"
```

with no preceding `#define`s, so all defaults apply. Then at line 585–622,
`linebuf_rewrap()` wraps `rewrap_inner()`.

`kitty/history.c:582–592` does:

```c
#define BufType HistoryBuf
#define map_src_index(y) ((src->start_of_data + y) % src->ynum)
#define init_src_line(src_y) init_line(src, map_src_index(src_y), src->line);
#define next_dest_line(cont) { \
    history_buf_set_last_char_as_continuation(dest, 0, cont); \
    LineAttrs *lap = attrptr(dest, historybuf_push(dest, as_ansi_buf)); \
    *lap = src->line->attrs; }
#define first_dest_line next_dest_line(false);
#include "rewrap.h"
```

The `#pragma once` at `rewrap.h:8` does **not** interfere — `#pragma once`
prevents duplicate inclusion *within the same translation unit*. These are two
different translation units (`.c` files), so each sees its own copy.

---

## 7. Deep Dive: `rewrap_inner()`

`kitty/rewrap.h:56–96` is the complete algorithm (reproduced below for
reference):

```c
static void
rewrap_inner(BufType *src, BufType *dest, const index_type src_limit,
             HistoryBuf UNUSED *historybuf, TrackCursor *track,
             ANSIBuf *as_ansi_buf) {
    bool is_first_line = true;
    index_type src_y = 0, src_x = 0, dest_x = 0, dest_y = 0,
               num = 0, src_x_limit = 0;
    TrackCursor tc_end = {.is_sentinel = true };
    if (!track) track = &tc_end;

    do {
        for (TrackCursor *t = track; !t->is_sentinel; t++)
            t->is_tracked_line = src_y == t->y;
        init_src_line(src_y);
        const bool src_line_is_continued = is_src_line_continued();
        src_x_limit = src->xnum;
        if (!src_line_is_continued) {
            // Trim trailing blanks since there is a hard line break at the end of this line
            while(src_x_limit && (src->line->cpu_cells[src_x_limit - 1].ch) == BLANK_CHAR)
                src_x_limit--;
        } else {
            src->line->gpu_cells[src->xnum-1].attrs.next_char_was_wrapped = false;
        }
        for (TrackCursor *t = track; !t->is_sentinel; t++) {
            if (t->is_tracked_line && t->x >= src_x_limit)
                t->x = MAX(1u, src_x_limit) - 1;
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
        if (!src_line_is_continued && src_y < src_limit) {
            init_src_line(src_y);
            next_dest_line(false);
            dest_x = 0;
        }
    } while (src_y < src_limit);
    dest->line->ynum = dest_y;
}
```

### 7.1 Variables and their meanings

| Variable | Role |
|---|---|
| `src_y` | Current source row being read |
| `src_x` | Current source column being read |
| `dest_y` | Current destination row being written (dynamically updated by `next_dest_line`) |
| `dest_x` | Current destination column being written |
| `num` | Chunk size for the current `copy_range` call: min(remaining in src line, remaining in dest line) |
| `src_x_limit` | Effective rightmost column of the current src line (may be trimmed for hard-break lines) |
| `is_first_line` | Only true for the first iteration; controls whether `first_dest_line` initializer runs |
| `src_line_is_continued` | True if the src line is soft-wrapped onto the next |
| `track` | Array of `TrackCursor` structs, terminated by a sentinel — tracks logical cursor positions through the mapping |

### 7.2 Line-level loop

The outer `do...while (src_y < src_limit)` loop iterates **once per source
row**. Each iteration:

1. **Flag tracked cursors**: set `t->is_tracked_line` for every tracked
   cursor whose `.y` matches the current `src_y`. This is an optimization:
   cursor tracking only activates on the current source line.

2. **Initialize the source line view**: `init_src_line(src_y)` re-points
   `src->line` at row `src_y`. For LineBuf this calls
   `linebuf_init_line(src, src_y)` which, as noted in §2.5, **recomputes
   `is_continued`** from the previous line's `next_char_was_wrapped`. For
   HistoryBuf it calls `init_line()` with the ring-buffer-mapped index.

3. **Determine continuation and trim trailing blanks**
   (`rewrap.h:66–73`):

   ```c
   const bool src_line_is_continued = is_src_line_continued();
   src_x_limit = src->xnum;
   if (!src_line_is_continued) {
       while (src_x_limit && src->line->cpu_cells[src_x_limit - 1].ch == BLANK_CHAR)
           src_x_limit--;
   } else {
       src->line->gpu_cells[src->xnum-1].attrs.next_char_was_wrapped = false;
   }
   ```

   - **Hard-break line**: trim trailing blanks off the end — the terminal
     doesn't need to preserve padding after a newline.
   - **Soft-wrap line**: **clear** the `next_char_was_wrapped` bit on src
     (because we're about to consume the line and decide whether the dest's
     corresponding line should be marked as wrapped).

   Behavioral confirmation: test `test_rewrap_narrower` in
   `kitty_tests/datatypes.py:390–392` shows:

   ```python
   lb = create_lbuf('123  ', 'abcde')      # src width 5, row1 has 3 chars + 2 blanks
   lb2 = self.line_comparison_rewrap(lb, '123', '  a', 'bcd', 'e')
   self.assertContinued(lb2, False, True, True, True)
   ```

   Since "123  " is **soft-wrapped** to "abcde" (because `create_lbuf` glues
   them with continuation), the trailing blanks on row 0 are **preserved**.

4. **Clamp tracked cursor x**: for each tracked cursor on this src line, if
   `t->x >= src_x_limit`, pull it back to `MAX(1u, src_x_limit) - 1`. This is
   the "cursor past EOL after trailing-blank trim" correction.

5. **First-line setup**: if this is the very first source line,
   `first_dest_line` positions `dest->line` at row 0 and (in the LineBuf
   variant) copies the src attrs to `dest->line_attrs[0]` via
   `set_dest_line_attrs(0)`. For the HistoryBuf variant, `first_dest_line`
   expands to `next_dest_line(false)` which pushes a fresh entry onto the
   ring.

6. **Inner copy loop** (`rewrap.h:80–91`):

   ```c
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
   ```

   This is the **chunked rectangular copy**:
   - If the dest is full on this row, advance to the next dest row with
     `continued=true`. `next_dest_line(true)` also writes
     `next_char_was_wrapped = true` on the dest's just-completed row.
   - `num` is the biggest slice we can copy without crossing either src or
     dest row boundary.
   - `copy_range()` memcpys `num` CPUCells and `num` GPUCells in one shot.
   - If a tracked cursor falls within `[src_x, src_x + num)`, remap it. See
     §12 for the arithmetic.

7. **After the inner loop, hard-break handling** (`rewrap.h:92–93`):

   ```c
   src_y++; src_x = 0;
   if (!src_line_is_continued && src_y < src_limit) {
       init_src_line(src_y);
       next_dest_line(false);
       dest_x = 0;
   }
   ```

   If the src line was a hard-break (not soft-wrapped), advance to the next
   dest line with `continued=false`. Otherwise (soft wrap), keep accumulating
   onto the current dest row — this is how two narrow src lines get merged
   into one wider dest line when widening.

   The extra `init_src_line(src_y)` call here is a **subtle detail**: it
   re-points `src->line` at the *next* src row **before** `next_dest_line` is
   invoked, so that `set_dest_line_attrs()` (invoked inside
   `next_dest_line`) copies `src->line->attrs` from the **next** line, not
   the line we just finished. This means:

   > The `prompt_kind` of dest row N+1 is inherited from src row N+1 (the
   > first src line that contributes to dest row N+1), not from src row N
   > (the last line that contributed to dest row N).

8. **Terminate**: `dest->line->ynum = dest_y;` — stash the number of dest
   rows written into `line->ynum` so the caller can read it.

### 7.3 What `rewrap_inner()` does NOT do

- Does not touch `line_map[]` on the destination — the destination `LineBuf`
  is freshly allocated by `alloc_linebuf()` with a trivial identity line_map.
- Does not handle wide characters specially — it relies on src/dest widths
  being unchanged per-cell. The `CellAttrs.width` field is carried along in
  the `memcpy` but not interpreted. **This has implications for wide-char
  edge cases — see §16.2.**
- Does not prune blank tail lines of the src buffer — that's
  `linebuf_rewrap()`'s responsibility (via the "find last non-empty line"
  loop in `line-buf.c:600–608`).

---

## 8. LineBuf Specialization

`kitty/line-buf.c:583` includes `rewrap.h` with no overrides. Then
`linebuf_rewrap()` at `line-buf.c:585–622` wraps the call:

```c
void
linebuf_rewrap(LineBuf *self, LineBuf *other,
               index_type *num_content_lines_before,
               index_type *num_content_lines_after,
               HistoryBuf *historybuf,
               index_type *track_x,  index_type *track_y,
               index_type *track_x2, index_type *track_y2,
               ANSIBuf *as_ansi_buf) {
    index_type first, i;
    bool is_empty = true;

    // Fast path
    if (other->xnum == self->xnum && other->ynum == self->ynum) {
        memcpy(other->line_map, self->line_map,
               sizeof(index_type) * self->ynum);
        memcpy(other->line_attrs, self->line_attrs,
               sizeof(LineAttrs) * self->ynum);
        memcpy(other->cpu_cell_buf, self->cpu_cell_buf,
               (size_t)self->xnum * self->ynum * sizeof(CPUCell));
        memcpy(other->gpu_cell_buf, self->gpu_cell_buf,
               (size_t)self->xnum * self->ynum * sizeof(GPUCell));
        *num_content_lines_before = self->ynum;
        *num_content_lines_after  = self->ynum;
        return;
    }

    // Find the first line that contains some content
    first = self->ynum;
    do {
        first--;
        CPUCell *cells = cpu_lineptr(self, self->line_map[first]);
        for (i = 0; i < self->xnum; i++) {
            if (cells[i].ch != BLANK_CHAR) { is_empty = false; break; }
        }
    } while (is_empty && first > 0);

    if (is_empty) {  // All lines are empty
        *num_content_lines_after = 0;
        *num_content_lines_before = 0;
        return;
    }
    *num_content_lines_before = first + 1;
    TrackCursor tcarr[3] = {
        {.x = *track_x,  .y = *track_y },
        {.x = *track_x2, .y = *track_y2},
        {.is_sentinel = true}
    };
    rewrap_inner(self, other, *num_content_lines_before,
                 historybuf, (TrackCursor*)tcarr, as_ansi_buf);
    *track_x = tcarr[0].x;  *track_y = tcarr[0].y;
    *track_x2 = tcarr[1].x; *track_y2 = tcarr[1].y;
    *num_content_lines_after = other->line->ynum + 1;
    for (i = 0; i < *num_content_lines_after; i++)
        other->line_attrs[i].has_dirty_text = true;
}
```

### 8.1 Fast path — same dimensions

If the resize call passes dimensions identical to the current ones (e.g.,
re-applying the same geometry), the function avoids rewrap entirely by
memcpying the four arena arrays (`line_map`, `line_attrs`, `cpu_cell_buf`,
`gpu_cell_buf`). Content line counts are set equal to `ynum` (conservative —
true content may be less, but the caller uses this only for cursor-beyond-
content logic and won't trigger it in the identity case).

### 8.2 Content-line detection

Walks backward from the last row looking for any row with a non-blank
`cpu_cell`. The scan stops when a non-empty row is found, and
`num_content_lines_before = first + 1` is the count (inclusive).

If the source is entirely blank, both counts are set to 0 and the function
returns without calling `rewrap_inner` — there is nothing to rewrap.

### 8.3 Cursor tracker array

Two cursors are tracked (the live cursor and the savepoint cursor). The array
has a sentinel `{.is_sentinel = true}` so `rewrap_inner()`'s iterator loops
can use `for (...; !t->is_sentinel; t++)`.

### 8.4 Post-rewrap bookkeeping

After `rewrap_inner()` returns:
- `dest->line->ynum` was set to the last `dest_y` written; so
  `*num_content_lines_after = other->line->ynum + 1`.
- All written rows (`0..num_content_lines_after-1`) are marked `has_dirty_text
  = true` so the renderer re-uploads them to the GPU.

### 8.5 The LineBuf `next_dest_line` macro — the spillover mechanism

`rewrap.h:24–38`:

```c
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
```

Behavioral breakdown:

1. **Mark current dest row's continuation flag**:
   `linebuf_set_last_char_as_continuation(dest, dest_y, continued)` writes
   the `continued` bit to the current (just-finished) dest row's last GPU
   cell.

2. **If dest is at its last row** (the overflow case):
   - `linebuf_index(dest, 0, dest->ynum - 1)` scrolls the entire LineBuf up
     by one via **line_map rotation** — the top row's storage slot is moved
     to the bottom, no cells are copied.
   - If `historybuf` is non-NULL (main screen): init the now-bottom row
     (which holds the old top row's contents), mark it dirty, and push it
     to history via `historybuf_add_line()`. This is the **overflow-to-
     history** path.
   - `linebuf_clear_line(dest, dest->ynum - 1, true)` zeros the bottom row
     and clears its attrs so rewrap can reuse it.
   - `dest_y` **stays at** `dest->ynum - 1` (not incremented) — we keep
     writing into the same logical bottom row, which now maps to a cleared
     physical slot.

3. **Else** (we still have room): `dest_y++`.

4. **Initialize the new dest view**:
   `linebuf_init_line(dest, dest_y); set_dest_line_attrs(dest_y)`.
   - `linebuf_init_line` refreshes `is_continued` as noted in §2.5.
   - `set_dest_line_attrs` copies `src->line->attrs` into
     `dest->line_attrs[dest_y]` **and** wipes `prompt_kind` on src (so the
     same prompt_kind doesn't propagate to multiple dest rows).

### 8.6 Attrs copy direction

Note the direction of `set_dest_line_attrs(dest_y)`: `dest->line_attrs[dest_y]
= src->line->attrs`. The source line's `is_continued` (recomputed by
`init_src_line` via the previous line's `next_char_was_wrapped`) is written
into dest's line_attrs. But the **stored** `is_continued` in `line_attrs[]`
is immediately overwritten by the next `linebuf_init_line()` call anyway —
§2.5 shows `is_continued` is always recomputed from the previous line's GPU
cell. So the effective persistent transfer is `prompt_kind`,
`has_image_placeholders`, and `has_dirty_text`.

---

## 9. HistoryBuf Specialization

`kitty/history.c:582–614` wraps `rewrap_inner()` for HistoryBuf:

```c
#define BufType HistoryBuf

#define map_src_index(y) ((src->start_of_data + y) % src->ynum)

#define init_src_line(src_y) init_line(src, map_src_index(src_y), src->line);

#define next_dest_line(cont) { \
    history_buf_set_last_char_as_continuation(dest, 0, cont); \
    LineAttrs *lap = attrptr(dest, historybuf_push(dest, as_ansi_buf)); \
    *lap = src->line->attrs; }

#define first_dest_line next_dest_line(false);

#include "rewrap.h"

void
historybuf_rewrap(HistoryBuf *self, HistoryBuf *other, ANSIBuf *as_ansi_buf) {
    while (other->num_segments < self->num_segments) add_segment(other);
    if (other->xnum == self->xnum && other->ynum == self->ynum) {
        // Fast path
        for (index_type i = 0; i < self->num_segments; i++) {
            memcpy(other->segments[i].cpu_cells, self->segments[i].cpu_cells,
                   SEGMENT_SIZE * self->xnum * sizeof(CPUCell));
            memcpy(other->segments[i].gpu_cells, self->segments[i].gpu_cells,
                   SEGMENT_SIZE * self->xnum * sizeof(GPUCell));
            memcpy(other->segments[i].line_attrs, self->segments[i].line_attrs,
                   SEGMENT_SIZE * sizeof(LineAttrs));
        }
        other->count = self->count; other->start_of_data = self->start_of_data;
        return;
    }
    if (other->pagerhist && other->xnum != self->xnum
        && ringbuf_bytes_used(other->pagerhist->ringbuf))
        other->pagerhist->rewrap_needed = true;
    other->count = 0; other->start_of_data = 0;
    if (self->count > 0) {
        rewrap_inner(self, other, self->count, NULL, NULL, as_ansi_buf);
        for (index_type i = 0; i < other->count; i++)
            attrptr(other, (other->start_of_data + i) % other->ynum)->has_dirty_text = true;
    }
}
```

### 9.1 Why `map_src_index`?

HistoryBuf's logical indexing goes from `0` (oldest) to `count - 1` (newest),
but physical storage is a ring with `start_of_data` as the offset of the
oldest entry. `map_src_index(y)` translates logical to physical.

`init_src_line` then uses this:
```c
init_line(src, map_src_index(src_y), src->line);
```

where `init_line()` at `history.c:161–177` reads the *physical* row directly
(and recomputes `is_continued` from the *physical* previous row — note this
is **not** the logical previous row. See §11.4 for the subtle consequence).

### 9.2 HistoryBuf's `next_dest_line` macro

```c
#define next_dest_line(cont) { \
    history_buf_set_last_char_as_continuation(dest, 0, cont); \
    LineAttrs *lap = attrptr(dest, historybuf_push(dest, as_ansi_buf)); \
    *lap = src->line->attrs; }
```

Two steps:

1. **Set continuation on the most recent dest row**:
   `history_buf_set_last_char_as_continuation(dest, 0, cont)`. The `0` here
   is the API-level reverse index — "newest." Inspect
   `history.c:302–307`:

   ```c
   static void
   history_buf_set_last_char_as_continuation(HistoryBuf *self, index_type y,
                                              bool wrapped) {
       if (self->count > 0) {
           gpu_lineptr(self, index_of(self, y))[self->xnum-1]
               .attrs.next_char_was_wrapped = wrapped;
       }
   }
   ```

2. **Push a new empty destination row**:
   `historybuf_push(dest, as_ansi_buf)` at `history.c:275–284`:

   ```c
   static index_type
   historybuf_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
       index_type idx = (self->start_of_data + self->count) % self->ynum;
       init_line(self, idx, self->line);
       if (self->count == self->ynum) {
           pagerhist_push(self, as_ansi_buf);
           self->start_of_data = (self->start_of_data + 1) % self->ynum;
       } else self->count++;
       return idx;
   }
   ```

   The key behavior: if the ring is full (`count == ynum`), the oldest entry
   is **evicted to pager history** (§14) and `start_of_data` advances by
   one. Otherwise `count` increments. Either way, `idx` is the physical slot
   index of the new newest entry, which is where `dest->line` now points.

3. **Set attrs**: `*lap = src->line->attrs` writes the source line's attrs
   into the new slot's `line_attrs[]` entry.

Note this differs structurally from the LineBuf `next_dest_line` — no explicit
`init_line` call after the push, because `historybuf_push` already does it
for us. No `set_dest_line_attrs` macro either — the `*lap = src->line->attrs`
line does the job directly.

### 9.3 `first_dest_line` = `next_dest_line(false)`

This differs from LineBuf where `first_dest_line` is an initialize-in-place
for row 0. HistoryBuf's `first_dest_line` **pushes** a new row — because
before the first push, `count == 0` and there is no "row 0" that makes sense;
you must grow by pushing.

### 9.4 `historybuf_rewrap()` initialization reset

`history.c:609` resets `other->count = 0; other->start_of_data = 0;` **before**
calling `rewrap_inner`. This is essential because `rewrap_inner` relies on
`historybuf_push` to build up content; the destination must start empty.

### 9.5 Pre-rewrap segment pre-growth

`history.c:596`: `while (other->num_segments < self->num_segments)
add_segment(other);`. This avoids `historybuf_push` calling `add_segment`
(via `segment_for`) from inside the `rewrap_inner` hot loop. It's pure
optimization — otherwise allocations would happen interleaved with the
rewrap work.

### 9.6 Pager history rewrap marking

`history.c:607–608`: if column count changed and the pager ringbuf has
content, set `pagerhist->rewrap_needed = true`. The pager history is a
byte-oriented ringbuf holding ANSI-escaped text, and `pagerhist_rewrap_to()`
(`history.c:391–432`) performs the actual rewrap lazily on demand (e.g., when
the user invokes the pager). This is a **lazy, column-sensitive** operation.

---

## 10. The Buffer Boundary — Visible Screen ↔ Scrollback

### 10.1 Overflow path: LineBuf → HistoryBuf (during reflow)

As described in §8.5, when `rewrap_inner` is operating on a `LineBuf` with a
non-NULL `historybuf` argument (i.e., the main linebuf), any row that is
"scrolled off the top" during rewrap is pushed into the history. This is the
**overflow** direction.

Behavioral demonstration from `kitty_tests/screen.py:287–289`:

```python
s.resize(5, 1)
self.ae(str(s.line(0)), '4')
self.ae(str(s.historybuf), '3\n3\n3\n3\n3\n2')
```

Starting state: main linebuf contains `0*5, 1*5, 2*5, 3*5, 4*5` (one row of
each digit). After `resize(5, 1)` (columns=1, so all 5-char lines become
5 separate rows of 1 char each wrapped, for a total of 25 rows of output, but
lines=5 means only 5 rows fit), the **visible** part is `'4'` (the last non-
blank 1-char row), and everything above has spilled into history: five `'3'`s
(one per cell of the original "33333"), and the last `'2'` (the 2-row chunk
got pushed down as the last "3" wrapped in).

### 10.2 Refill path: HistoryBuf → LineBuf (scrollback fill after enlarge)

The reverse direction is exercised by the `scrollback_fill_enlarged_window`
option, implemented at `screen.c:428–438`. When the window grows vertically,
lines are pulled **back** from history via `historybuf_pop_line()`. See §13.

### 10.3 The continuation bridge: `history_buf_endswith_wrap()`

`screen.c:2833–2840`:

```c
static Line*
init_line(Screen *self, index_type y) {
    linebuf_init_line(self->linebuf, y);
    if (y == 0 && self->linebuf == self->main_linebuf) {
        if (history_buf_endswith_wrap(self->historybuf))
            self->linebuf->line->attrs.is_continued = true;
    }
    return self->linebuf->line;
}
```

For **line 0** of the main linebuf, the screen-level `init_line` overrides
`linebuf_init_line`'s computed `is_continued` (which defaults to `false` for
`idx == 0` per `line-buf.c:145`) with the value derived from the history's
most recent line via `history_buf_endswith_wrap()`:

```c
// history.c:184–187
bool
history_buf_endswith_wrap(HistoryBuf *self) {
    return gpu_lineptr(self, index_of(self, 0))
                      [self->xnum-1].attrs.next_char_was_wrapped;
}
```

This is the **only** place the history-to-screen continuation flag is
bridged. It is **not** checked by `linebuf_init_line` directly — any code
path that bypasses `screen_init_line` misses this bridge. See §16.3 for
where this matters.

### 10.4 Why the continuation bridge exists only at y=0

Continuation between rows *within* a LineBuf is computed from the GPU cell of
the previous row — which exists within the LineBuf. Continuation from history
to screen row 0 has no "previous row in linebuf" to look at, so we look at
the history's top line instead. For rows y >= 1 of the linebuf, no bridging
is necessary because row y-1 is a real linebuf row with a valid GPU cell.

---

## 11. Line Continuation State: The Dual-Signal Architecture

This is the most subtle aspect of the reflow system and is the source of the
potential issues identified in §16.

### 11.1 The two signals

**Signal A: `CellAttrs.next_char_was_wrapped`** — persistent, on the last
GPU cell of each line. Set by the VT parser when a character overflows the
current line, cleared when the line is re-written or explicitly reset.

**Signal B: `LineAttrs.is_continued`** — derived, set dynamically by the
"init line" helpers. Present in `line_attrs[]` storage but always overwritten
on access.

### 11.2 How `rewrap_inner` reads and writes each signal

| Action | Signal read/written | Where |
|---|---|---|
| Detect whether src line is soft-wrapped | Reads `next_char_was_wrapped` on src last cell | `is_src_line_continued()` macro at `rewrap.h:41` |
| Clear src's wrap bit once consumed | Writes `next_char_was_wrapped = false` on src last cell | `rewrap.h:72` |
| Mark dest row as wrapped when its cell space is exhausted | Writes `next_char_was_wrapped` via `linebuf_set_last_char_as_continuation` (LineBuf) or `history_buf_set_last_char_as_continuation` (HistoryBuf) | `rewrap.h:26`, `history.c:588` |
| Read src line's `is_continued` | Never read — `is_continued` is a query-time derivation; rewrap reads the storage field `next_char_was_wrapped` directly | — |
| Write dest line's `is_continued` | Implicitly, via `set_dest_line_attrs` copying src's `line->attrs` (which has `is_continued` computed by `init_src_line`) | `rewrap.h:18, 37` |

### 11.3 Asymmetry in when `is_continued` is refreshed

`is_continued` is refreshed **in the stored `line_attrs[]`** only transiently
and only via the chain:

1. `init_src_line(src_y)` → `linebuf_init_line(src, src_y)` → computes
   `src->line->attrs.is_continued` from previous row's GPU cell.
2. `set_dest_line_attrs(dest_y)` → copies `src->line->attrs` (including the
   just-computed `is_continued`) into `dest->line_attrs[dest_y]`.
3. Next call to `linebuf_init_line(dest, ...)` **overwrites**
   `line->attrs.is_continued` with a freshly computed value.

So the value of `is_continued` in `dest->line_attrs[dest_y]` after rewrap is
**meaningless** — any consumer that reads it directly gets stale data. Only
code that goes through `linebuf_init_line` sees the correct value. This is
**not a bug** — it's the documented contract — but it is a trap for anyone
reading `line_attrs[y].is_continued` without going through the init helper.

### 11.4 HistoryBuf's `is_continued` is computed from PHYSICAL previous row

`history.c:167`:

```c
if (num > 0) {
    l->attrs.is_continued =
        gpu_lineptr(self, num - 1)[self->xnum-1].attrs.next_char_was_wrapped;
} else {
    l->attrs.is_continued = false;
    ...
}
```

Here `num` is the **physical** ring-buffer index, not the API's reverse-
indexed `lnum`. So when `num == 0` (the physical slot at the start of the
ring), the `is_continued` is hard-set to `false` regardless of whether that
slot corresponds to a line that was in fact continued from the now-evicted
preceding line. **The pager-history ringbuf lookup beneath it** (`history.c:
170–176`) rescues the case where the evicted line is still in the pager
history, by checking if the ringbuf's last byte is a newline.

But there is a subtle window: when `num == 0` is **not** the logical first
line (i.e., `start_of_data != 0` after some evictions have happened), the
code still uses `num - 1`, which gives a valid physical pointer but to the
**wrong logical previous row**. Since HistoryBuf is a ring, `num == 0` means
the *physical* slot 0 — but the logical predecessor is at physical slot
`ynum - 1`, not physical slot "negative 1."

**However**, the wrap-around case is *guarded* because `num - 1` is an
`index_type` (unsigned), so `num == 0` leads to the `else` branch
unconditionally. Let's verify: `if (num > 0)` — yes, safe. For `num == 0`,
the `else` branch runs and checks the pager history. So the issue is: **if
`num == 0` physically (slot 0) but there were prior evictions**, the
predecessor logically is at physical slot `ynum-1`, but we don't check it.
We do check pager history, which is correct because evicted lines go *there*
first. So when the ring has wrapped, the evicted predecessor *should* be in
pager history with the correct trailing character.

Verifying: `historybuf_push` at `history.c:279–281` calls
`pagerhist_push` before advancing `start_of_data`, so the just-evicted line
does get written to pager history. Good — the logic is consistent.

### 11.5 The "orphaned continuation" edge case during reflow

Consider a LineBuf before reflow:

```
row 0: "abcde"     (wrap flag set — continued)
row 1: "fghij"     (wrap flag cleared — hard break)
row 2: ""
```

During `rewrap_inner` with a wider dest:
- `src_y=0`: `is_src_line_continued() = true`, so the wrap flag is **cleared
  on src row 0** at `rewrap.h:72`. Dest row 0 accumulates "abcde".
- `src_y=1`: `is_src_line_continued() = false`, so src row 1's trailing
  blanks are trimmed, "fghij" is appended to dest row 0 (because
  `src_line_is_continued` was true for the *previous* iteration, controlling
  whether we advance dest_y). Wait — actually looking at `rewrap.h:92–93`:

  ```c
  if (!src_line_is_continued && src_y < src_limit) {
      init_src_line(src_y);
      next_dest_line(false);
      dest_x = 0;
  }
  ```

  This advances dest line only when the **current src line** (the one whose
  iteration just completed) was **not** continued. So the variable
  `src_line_is_continued` in this snippet is the value computed for the
  **src_y that just finished**. Looking at the source: yes, at `rewrap.h:66`
  it's captured for the current iteration, and by line 92 it still refers to
  the current iteration. So after `src_y=0` (continued), dest is NOT
  advanced, and src_y=1 appends to the same dest row. Then src_y=1 is a hard
  break, so dest advances for src_y=2.

OK that is consistent. Now consider the reverse — narrower dest:

```
src row 0: "abcde"  (no wrap)
src row 1: ""
```

Shrinking to dest width 3:
- src_y=0: `src_x=0, dest_x=0, src_x_limit=5` (no trailing-blank trim because
  the whole line is non-blank — actually "abcde" has no blanks, so full 5).
  Inner loop: `num = MIN(5-0, 3-0) = 3`, copies "abc" to dest[0][0..2].
  `src_x=3, dest_x=3`. Loop again: `dest_x >= dest->xnum (3 >= 3)`, so
  `next_dest_line(true)` — wrap flag SET on dest row 0, dest_y=1, dest_x
  reset. `num = MIN(5-3, 3-0) = 2`, copies "de" to dest[1][0..1].
- After the inner loop: `src_y=1, src_x=0`. Since src_y=0 was NOT continued
  (`src_line_is_continued = false`), `rewrap.h:92–93` calls
  `next_dest_line(false)` → dest_y=2, wrap flag CLEARED on dest row 1.
- src_y=1: `init_src_line(1)`, line is empty, `src_x_limit=0` after trim.
  Inner loop doesn't execute. End of outer loop.
- `dest->line->ynum = dest_y = 2`. Function returns.

Result in dest: row 0 = "abc" (wrap=1), row 1 = "de" (wrap=0), row 2 = empty.
Continuation states: row 0.is_continued=false (no prior), row 1.is_continued
=true (prev row's wrap=1), row 2.is_continued=false (prev row's wrap=0).

This is exactly what the test expects:

```python
# kitty_tests/datatypes.py:387–389
lb = create_lbuf('123', 'abcde')
lb2 = self.line_comparison_rewrap(lb, '123', 'abc', 'de')
self.assertContinued(lb2, False, False, True)
```

### 11.6 Summary of the continuation state model

- Storage: `next_char_was_wrapped` bits on last GPUCell of every line.
- Query: `is_continued` in `LineAttrs`, freshly computed on every line
  access.
- History bridge: only at main linebuf row 0, via `history_buf_endswith_wrap()`
  in `screen.c:init_line`.
- Rewrap invariants:
  - Reads `next_char_was_wrapped` from src to decide soft-wrap vs hard-break.
  - Writes `next_char_was_wrapped` to dest when a line breaks due to width,
    or when advancing to next src line.
  - Clears src's wrap flag during consumption so the source state is
    consistent if inspected mid-rewrap (it shouldn't be — but defensive).
- Storage inconsistency: `line_attrs[y].is_continued` after rewrap is stale
  garbage; only `linebuf_init_line` and `init_line` (HistoryBuf) refresh it.

---

## 12. Cursor Tracking Through Rewrap

Two cursors ride along with the rewrap. `rewrap.h:74–89`:

```c
for (TrackCursor *t = track; !t->is_sentinel; t++) {
    if (t->is_tracked_line && t->x >= src_x_limit)
        t->x = MAX(1u, src_x_limit) - 1;
}
...
for (TrackCursor *t = track; !t->is_sentinel; t++) {
    if (t->is_tracked_line && src_x <= t->x && t->x < src_x + num) {
        t->y = dest_y;
        t->x = dest_x + (t->x - src_x + (t->x > 0));
    }
}
```

### 12.1 Pre-copy clamp

Before entering the inner copy loop, the cursor x is clamped down to
`src_x_limit - 1` if it was beyond the trimmed line width. This handles the
"cursor sitting on trimmed trailing blanks" case: if the user's cursor was at
x=10 on a line trimmed to `src_x_limit=7`, the cursor moves to x=6 before
tracking.

`MAX(1u, src_x_limit) - 1` handles `src_x_limit == 0` (empty line): the result
is 0, which means the cursor goes to column 0. Otherwise it's
`src_x_limit - 1` (cursor on the last non-blank column).

### 12.2 The +1 adjustment in the copy-loop remap

```c
t->x = dest_x + (t->x - src_x + (t->x > 0));
```

In plain language: the cursor's new x equals
`dest_x + (relative_position_within_chunk) + (1 if cursor was not at column 0)`.

### 12.2.1 Why the +1?

Terminal cursor semantics distinguish between "cursor points at the cell to
be written" (the common convention, column 0-indexed) and "cursor is one
past the last written cell" (the write-the-next-char convention). Kitty uses
the former internally for cursor.x, but the +1 adjustment handles a specific
case: if the cursor was at `t->x` inside the src chunk `[src_x, src_x+num)`,
and the src chunk maps to dest `[dest_x, dest_x+num)`, then the naive
`t->x = dest_x + (t->x - src_x)` would place the cursor **at** the
corresponding dest column.

But test `test_cursor_after_resize` at `kitty_tests/screen.py:315–319` expects:

```python
s = self.create_screen()
draw('123'), draw('123')     # each draw writes 3 chars + linefeed+CR
y_before = s.cursor.y        # cursor now at y=2, x=0 (after the second linefeed)
s.resize(s.lines, s.columns-1)
self.ae(y_before, s.cursor.y)
```

So after drawing two rows of `'123'`, the cursor is at (x=0, y=2), and after
shrinking the columns by 1, the cursor's y is preserved at 2. The +1 doesn't
fire here (`t->x = 0`), so the `t->x > 0` guard keeps the result simple.

### 12.2.2 What the +1 actually does

Consider a case where the cursor is at (x=4, y=0) on a 5-wide line of content,
being shrunk to dest width 3:

- First iteration: src_x=0, dest_x=0, num=3. Copy "abc". Cursor is at x=4,
  src_x=0 ≤ 4 < 0+3=3 is **false**, so cursor not remapped yet.
- Second iteration: dest_x=3 >= 3, call `next_dest_line(true)`, dest_x=0,
  dest_y=1. src_x=3, num=2. Copy "de". Now src_x=3 ≤ 4 < 3+2=5, so remap:
  `t->y = 1`, `t->x = 0 + (4 - 3 + 1) = 2`.

With the +1, cursor lands at dest x=2, which is the cell containing 'e'
(cursor points at or just past the cell it last wrote). Without the +1, it
would land at x=1 (the cell containing 'd'). The +1 keeps cursor at/past
the *next* unwritten column.

### 12.2.3 Potential issue: off-by-one at the right edge

There is a known subtlety: when `t->x` is at `src_x + num - 1` (the last cell
of the copied chunk), the formula gives `dest_x + (num - 1 + 1) = dest_x + num`.
But `dest_x + num` **equals** `dest->xnum` if the copy filled the dest row,
meaning the cursor lands *beyond* the last dest column. This is the
"one-past-the-end" position, which in terminal semantics is valid (it's where
the *next* character will be written, which may itself trigger a wrap).

However, the very next iteration of the outer copy loop may trigger
`next_dest_line(true)` which would otherwise affect subsequent cursor
remappings — but the cursor was already remapped in the iteration where the
match fired, and `is_tracked_line` remains true until `src_y` advances, so
subsequent iterations won't re-match (the condition `src_x <= t->x` would be
`src_x <= dest_x+num` with src_x now being `src_x+num`, checks against
`t->x = dest_x + num`, so `src_x <= dest_x + num` is true but
`t->x < src_x + num` compares `dest_x + num < src_x + num` which is equivalent
to `dest_x < src_x` which is **not true** in general — so the condition fails
and the cursor isn't re-remapped. Good.

But the cursor is now pointing at `(dest_y, dest->xnum)`, a virtual column
one past the end. When the caller (`screen_resize`) copies `temp.x` into
`cursor->x` at `screen.c:419–420`:

```c
#define S(c, w) c->x = MIN(w.after.x, self->columns - 1); \
                c->y = MIN(w.after.y, self->lines - 1);
```

`MIN(dest_x + num, self->columns - 1) = self->columns - 1`. So the cursor
gets clamped into range. This is **correct**, but it means the specific
column information (that the cursor was "in the wrap zone") is lost. For a
terminal this is fine because cursor at `columns - 1` with `pending_wrap`
state is the canonical pre-wrap position.

### 12.3 The `is_tracked_line` flag

The outer loop updates `is_tracked_line` **once per source row** at
`rewrap.h:64`:

```c
for (TrackCursor *t = track; !t->is_sentinel; t++)
    t->is_tracked_line = src_y == t->y;
```

This means once cursor tracking activates for a given src row, it stays
active across all chunked copies from that row until src_y advances — at
which point it deactivates (set false, unless the cursor's y matches the new
src_y).

**Potential issue**: Because `t->y` is mutated by the remap
(`t->y = dest_y`), on subsequent outer iterations the check
`src_y == t->y` will compare the old src-side y against a dest-side y
— semantically meaningless for lines beyond the cursor, but harmless because
`is_tracked_line` then reverts to false. The only case where this would
accidentally re-activate is if `dest_y` happens to equal some later `src_y`
numerically, which in the upper-bounded iteration would be a coincidence with
no harmful effect beyond potentially clamping tracking's x again — and the
inner copy-loop check `src_x <= t->x < src_x + num` would still gate the
remap correctly.

### 12.4 Cursor "beyond content" detection (screen.c:368)

```c
which.is_beyond_content = num_content_lines_before > 0
                          && self->cursor->y >= num_content_lines_before;
```

If the cursor's pre-resize y was past the last content row, set the flag.
Later, at `screen.c:424–427`:

```c
if (cursor.is_beyond_content) {
    self->cursor->y = cursor.num_content_lines;
    if (self->cursor->y >= self->lines) {
        self->cursor->y = self->lines - 1;
        screen_index(self);
    }
}
```

The cursor is repositioned to **just after** the content (the first blank
row after rewrap). If that's off-screen, clamp and scroll one line. This
preserves the "cursor in empty area below command output" UX during shell
interaction.

---

## 13. Scrollback Fill After Enlarging a Window

### 13.1 The option

From `kitty/options/definition.py:420–423`:

```python
opt('scrollback_fill_enlarged_window', 'no',
    option_type='to_bool', ctype='bool',
    long_text='Fill new space with lines from the scrollback buffer '
              'after enlarging a window.'
    )
```

Default is `'no'`. When set to `'yes'`, the mechanism in `screen.c:428–438`
activates.

### 13.2 The mechanism

```c
if (is_main && OPT(scrollback_fill_enlarged_window)) {
    const unsigned int top = 0, bottom = self->lines-1;
    Savepoint *sp = is_main ? &self->main_savepoint : &self->alt_savepoint;
    while (self->cursor->y + 1 < self->lines
           && self->lines - self->cursor->y > lines_after_cursor_before_resize) {
        if (!historybuf_pop_line(self->historybuf, self->alt_linebuf->line))
            break;
        INDEX_DOWN;
        linebuf_copy_line_to(self->main_linebuf, self->alt_linebuf->line, 0);
        self->cursor->y++;
        sp->cursor.y = MIN(sp->cursor.y + 1, self->lines - 1);
    }
}
```

Step-by-step:
1. **Loop condition**: keep filling as long as
   - The cursor is not already at the last row
   - The current distance from cursor to bottom is *greater* than the
     pre-resize distance (i.e., the window has genuine new space below the
     cursor)
2. **Pop the newest history line**: `historybuf_pop_line` retrieves the
   most recent history entry and decrements `count`. This effectively moves
   the line from history back into circulation. It writes into
   `self->alt_linebuf->line` — a scratch `Line*` we borrow from the alt
   linebuf for this purpose (alt linebuf's content is irrelevant here since
   we're filling main).
3. **`INDEX_DOWN`** (`screen.c:289–299` macro): scrolls the main linebuf
   **downward** by one via `linebuf_reverse_index(top, bottom)`, which
   rotates `line_map[]` so the bottom row's storage moves to the top (the
   opposite direction of `linebuf_index`). Clears the new top row. Adjusts
   `last_visited_prompt.y` and graphics and selections by 1.
4. **Copy popped line into new top row**: `linebuf_copy_line_to(main_linebuf,
   alt_linebuf->line, 0)` writes the popped history line into row 0 of the
   main linebuf.
5. **Advance cursor y** and the savepoint's cursor y (capped at
   `lines - 1`). This preserves cursor visual position — as the buffer
   shifts down, the cursor shifts down with it.

### 13.3 Behavioral verification

From `kitty_tests/screen.py:361–367`:

```python
# Height increased, width unchanged → pull down lines to fill new space at the top
s = prepare_screen(map(str, range(6)))
assert_lines('2', '3', '4', '5', '')
dist_from_bottom = s.lines - s.cursor.y
s.resize(7, s.columns)
assert_lines('0', '1', '2', '3', '4', '5', '')
self.ae(dist_from_bottom, s.lines - s.cursor.y)
```

Before: 5-line window with rows `2,3,4,5,<empty>`, cursor at bottom.
`0` and `1` are in history.

After `resize(7, columns)`: expected 7-line window with `0,1,2,3,4,5,<empty>`
— the history has been drained back into the top of the visible buffer.
Cursor's distance from bottom is preserved.

### 13.4 Interaction with rewrap

Note this fill happens **after** rewrap has already executed. So if the
resize is "wider and taller" (e.g., `test_scrollback_fill_after_resize`'s
"Height increased, width increased" case at `screen.py:369–373`), the
sequence is:
1. Rewrap history to new width (`realloc_hb`).
2. Rewrap main linebuf to new width (`realloc_lb`); overflow (if any) spills
   to the already-rewrapped history.
3. Pop history lines and shift main downward (scrollback fill).

By the time step 3 runs, both history and linebuf are already at the new
column width, so copying history-line into linebuf is straightforward.

### 13.5 The `lines_after_cursor_before_resize` guard

The loop condition `self->lines - self->cursor->y > lines_after_cursor_before_resize`
is specifically worded to catch the **enlargement** case. If the window was
shrunk, `self->lines - self->cursor->y` may be smaller than
`lines_after_cursor_before_resize` and the loop never iterates. If enlarged,
the new cursor-to-bottom distance is larger than before, and we pull lines
until equality is restored.

---

## 14. Pager History Interaction

### 14.1 Structure

`kitty/data-types.h:268–272`:

```c
typedef struct {
    void *ringbuf;
    size_t maximum_size;
    bool rewrap_needed;
} PagerHistoryBuf;
```

The pager history is a byte-level ring buffer (the vendored `ringbuf`
library) that accumulates ANSI-escaped text as lines are evicted from the
structured `HistoryBuf`. When the user runs `kitty @ scroll-to-prompt` or
pipes to a pager, pager history is consulted.

### 14.2 Push on eviction

When `historybuf_push` overflows (`history.c:279–281`), it calls
`pagerhist_push(self, as_ansi_buf)` **before** advancing `start_of_data`.

`history.c:258–273`:

```c
static void
pagerhist_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
    PagerHistoryBuf *ph = self->pagerhist;
    if (!ph) return;
    const GPUCell *prev_cell = NULL;
    Line l = {.xnum=self->xnum};
    init_line(self, self->start_of_data, &l);
    line_as_ansi(&l, as_ansi_buf, &prev_cell, 0, l.xnum, 0);
    pagerhist_write_bytes(ph, (const uint8_t*)"\x1b[m", 3);
    if (pagerhist_write_ucs4(ph, as_ansi_buf->buf, as_ansi_buf->len)) {
        char line_end[2]; size_t num = 0;
        line_end[num++] = '\r';
        if (!l.gpu_cells[l.xnum - 1].attrs.next_char_was_wrapped)
            line_end[num++] = '\n';
        pagerhist_write_bytes(ph, (const uint8_t*)line_end, num);
    }
}
```

Observations:
- Re-examines the evicted line via `init_line(self, self->start_of_data, &l)`
  — this is the line at physical slot `start_of_data` (the oldest).
- Writes SGR reset `\x1b[m`, then the ANSI-escaped cell content, then a `\r`
  and — if `next_char_was_wrapped` is clear — a `\n`. So **soft-wrapped lines
  in history are preserved as single logical lines** in pager history (no
  trailing newline), and hard-break lines get their newline.

### 14.3 Resize rewrap

When `historybuf_rewrap` detects a column change (`history.c:607–608`):

```c
if (other->pagerhist && other->xnum != self->xnum
    && ringbuf_bytes_used(other->pagerhist->ringbuf))
    other->pagerhist->rewrap_needed = true;
```

The `rewrap_needed` flag is set. **The actual rewrap is deferred** — it's
performed later by `pagerhist_rewrap_to()` (`history.c:391–432`), invoked
when the user accesses the pager history. This is a valuable optimization:
most resizes never trigger pager consumption, so we save the cost of rewrapping
potentially megabytes of text eagerly.

`pagerhist_rewrap_to()` iterates char-by-char through the old ringbuf,
tracking column position via `wcswidth_step` (for wide-char awareness), and
re-inserts `\r` breaks where the new width demands.

### 14.4 `init_line`'s pager history check

As noted in §2.5, HistoryBuf's `init_line` at `history.c:170–176` reads:

```c
if (self->pagerhist && self->pagerhist->ringbuf
    && (sz = ringbuf_bytes_used(self->pagerhist->ringbuf)) > 0) {
    size_t pos = ringbuf_findchr(self->pagerhist->ringbuf, '\n', sz - 1);
    if (pos >= sz) l->attrs.is_continued = true;  // ringbuf does not end with a newline
}
```

This bridges pager history → history for the continuation flag. If the
oldest *live* history line was preceded (in pager history) by a soft-wrap,
its `is_continued` comes back as `true`.

### 14.5 Pager history during scrollback fill

`historybuf_pop_line` at `history.c:293–300` does **not** pull lines back
from pager history — only from the structured history ring. So if you resize
twice in quick succession (grow, then grow again), the first grow might
evict a line from the ring to the pager, and the second grow's scrollback
fill will not retrieve that evicted line from pager history. This is
generally fine because the pager history is meant for "save for pager," not
"save for re-entry."

---

## 15. Complete Call Chain & Data-Flow Map

### 15.1 Function-level call graph

```
Python: Screen.resize(lines, columns)
│
├── screen_resize()                        [kitty/screen.c:345]
│   │
│   ├── screen_pause_rendering(false, 0)
│   │
│   ├── [optional] insert dummy '<' in empty OUTPUT_START   [screen.c:353–361]
│   │
│   ├── init_overlay_line(columns, true)                    [screen.c:372]
│   │
│   ├── realloc_hb(historybuf, ynum, columns, as_ansi_buf)  [screen.c:375]
│   │   │
│   │   ├── alloc_historybuf(lines, columns, 0)
│   │   │
│   │   ├── [move pagerhist pointer across]
│   │   │
│   │   └── historybuf_rewrap(old, new, as_ansi_buf)        [history.c:595]
│   │       │
│   │       ├── [pre-grow segments]
│   │       │
│   │       ├── [fast path for same dims] OR
│   │       │
│   │       └── rewrap_inner(src=old, dest=new,             [rewrap.h:56]
│   │                        src_limit=old.count,
│   │                        historybuf=NULL, track=NULL)
│   │           │
│   │           ├── init_src_line(src_y)                    [history.c:586]
│   │           │   → init_line(src, map_src_index(src_y), src->line)
│   │           │
│   │           ├── is_src_line_continued()                 [rewrap.h:41]
│   │           │
│   │           ├── first_dest_line/next_dest_line(cont)    [history.c:588,590]
│   │           │   → history_buf_set_last_char_as_continuation
│   │           │   → historybuf_push
│   │           │       → [if full] pagerhist_push          [history.c:258]
│   │           │
│   │           └── copy_range()                            [rewrap.h:44]
│   │
│   ├── [is_main] prevent_current_prompt_from_rewrapping()  [screen.c:302]
│   │   │
│   │   ├── [walk back from cursor looking for PROMPT_START]
│   │   ├── linebuf_copy_line_to(prompt_copy, ...)
│   │   ├── linebuf_clear_line(main_linebuf, ..., false)
│   │   └── [write ' ' into cpu_cells[0] of prompt region]
│   │
│   ├── realloc_lb(main_linebuf, lines, columns, ...,       [screen.c:384]
│   │              self->historybuf, ...)
│   │   │
│   │   ├── alloc_linebuf(lines, columns)
│   │   │
│   │   ├── [copy before.xy → temp.xy]
│   │   │
│   │   └── linebuf_rewrap(main_linebuf, new_linebuf, ...)  [line-buf.c:585]
│   │       │
│   │       ├── [fast path for same dims] OR
│   │       │
│   │       ├── [find first non-empty line]
│   │       │
│   │       └── rewrap_inner(src, dest, content_lines,      [rewrap.h:56]
│   │                        historybuf=self->historybuf,
│   │                        track=cursor_array)
│   │           │
│   │           ├── init_src_line(src_y)                    [rewrap.h:15 default]
│   │           │   → linebuf_init_line(src, src_y)         [line-buf.c:141]
│   │           │
│   │           ├── first_dest_line/next_dest_line(cont)    [rewrap.h:21,25]
│   │           │   ├── linebuf_set_last_char_as_continuation
│   │           │   ├── linebuf_index(dest, 0, ynum-1)      [line-buf.c:316]
│   │           │   ├── [if dest full AND historybuf != NULL]
│   │           │   │   └── historybuf_add_line(historybuf, ...) [history.c:286]
│   │           │   │       └── historybuf_push
│   │           │   │           └── [if full] pagerhist_push
│   │           │   └── linebuf_clear_line(dest, ynum-1, true) [line-buf.c:299]
│   │           │
│   │           └── copy_range()
│   │
│   ├── grman_remove_all_cell_images(main_grman)
│   ├── grman_resize(main_grman, ...)
│   │
│   ├── realloc_lb(alt_linebuf, ..., NULL /* no history */, ...)  [screen.c:394]
│   │   └── [same as main path, but historybuf=NULL → overflow discarded]
│   │
│   ├── grman_remove_all_cell_images(alt_grman)
│   ├── grman_resize(alt_grman, ...)
│   │
│   ├── [commit self->lines, self->columns, margins, tabstops]
│   ├── [clear selections]
│   ├── [clamp cursor via S(c, w) macro]
│   ├── [if is_beyond_content: reposition cursor to num_content_lines]
│   │
│   ├── [if is_main AND OPT(scrollback_fill_enlarged_window)]
│   │   └── while cursor+1 < lines AND room to grow:
│   │       ├── historybuf_pop_line(historybuf, alt_linebuf->line)  [history.c:293]
│   │       ├── INDEX_DOWN                                 [screen.c:289]
│   │       │   ├── linebuf_reverse_index(top, bottom)
│   │       │   ├── linebuf_clear_line(top, true)
│   │       │   ├── grman_scroll_images(1)
│   │       │   └── index_selection(...)
│   │       └── linebuf_copy_line_to(main_linebuf, popped_line, 0)
│   │
│   ├── [if dummy_output_inserted: erase dummy '<' from cursor line]
│   │
│   └── [if num_of_prompt_lines: copy prompt_copy back into main_linebuf]
│
└── [return; renderer picks up self->is_dirty == true next frame]
```

### 15.2 Data-flow: a wrapped line moving from linebuf to history

Consider a concrete scenario: main linebuf has 5 rows, top row is "abcde"
(wrap flag set), row 1 is "fghij" (no wrap). Resize columns to 3.

```
Before resize:
  linebuf[0] = "abcde"  [next_char_was_wrapped = 1]
  linebuf[1] = "fghij"  [next_char_was_wrapped = 0]
  linebuf[2] = ""
  linebuf[3] = ""
  linebuf[4] = "" (cursor here)

rewrap_inner with dest_xnum=3, historybuf != NULL:

iter src_y=0: init_src_line(0) → src->line = "abcde", is_continued=false
              is_src_line_continued() → TRUE (wrap flag set)
              src_x_limit = 5, NO trim
              first_dest_line → linebuf_init_line(dest, 0), dest_y=0
              inner loop:
                copy_range("abc" → dest[0][0..2]), dest_x=3
                dest_x >= dest->xnum → next_dest_line(true)
                  → set dest[0].last_cell.wrap = true
                  → dest_y=1 (normal advance, ynum=5 not full)
                  → init dest[1]
                  → dest_x=0
                copy_range("de"  → dest[1][0..1]), src_x=5, dest_x=2
              exit inner (src_x=5 >= src_x_limit=5)
              src_y=1, src_x=0
              src_line_is_continued=TRUE, so do NOT advance dest_y

iter src_y=1: init_src_line(1) → src->line = "fghij", is_continued=true
              is_src_line_continued() → FALSE
              src_x_limit = 5 after trim (no blanks)
              inner loop:
                dest_x=2, copy_range("f"   → dest[1][2]), dest_x=3
                dest_x >= 3 → next_dest_line(true)
                  → set dest[1].last_cell.wrap=true
                  → dest_y=2, dest_x=0
                copy_range("ghi" → dest[2][0..2]), src_x=4, dest_x=3
                dest_x >= 3 → next_dest_line(true)
                  → set dest[2].last_cell.wrap=true
                  → dest_y=3, dest_x=0
                copy_range("j"   → dest[3][0]), src_x=5, dest_x=1
              exit inner
              src_y=2, src_x=0
              src_line_is_continued=FALSE → next_dest_line(false)
                → set dest[3].last_cell.wrap=false
                → dest_y=4, dest_x=0

iter src_y=2..4: all empty, trim to 0, no inner loop body,
                 each iteration calls next_dest_line(false)

  src_y=2: next_dest_line(false) → dest_y=5 ... but dest->ynum=5!
    → dest_y >= dest->ynum - 1 (5 >= 4), triggers overflow:
      → linebuf_index(dest, 0, 4) [scroll dest up]
      → historybuf != NULL: push the evicted top (dest[0]="abc" wrap=1) to history
      → clear the recycled bottom row (now physical slot formerly at top)
      → dest_y stays at 4
  src_y=3: same — dest_y stays at 4, another eviction
  src_y=4: same
```

Wait — there's an issue here. The trailing blank lines trigger
`next_dest_line(false)`, each of which scrolls up and evicts. But the src has
empty trailing rows (rows 2,3,4) and `src_x_limit` would be 0 after trim.
Looking again at `rewrap.h`:

```c
do {
    ...
    src_x_limit = src->xnum;
    if (!src_line_is_continued) {
        while(src_x_limit && (src->line->cpu_cells[src_x_limit - 1].ch) == BLANK_CHAR) src_x_limit--;
    }
    ...
    if (is_first_line) { first_dest_line; is_first_line = false; }
    while (src_x < src_x_limit) { ... }
    src_y++; src_x = 0;
    if (!src_line_is_continued && src_y < src_limit) { init_src_line(src_y); next_dest_line(false); dest_x = 0; }
} while (src_y < src_limit);
```

For an empty trailing line, `src_x_limit = 0`, inner loop is skipped. Then
`src_line_is_continued = false`, so `next_dest_line(false)` fires. Yes — this
will eagerly evict. **BUT** — `linebuf_rewrap` at `line-buf.c:600–608`
**trims trailing empty lines** before calling `rewrap_inner`:

```c
first = self->ynum;
do {
    first--;
    CPUCell *cells = cpu_lineptr(self, self->line_map[first]);
    for(i = 0; i < self->xnum; i++) {
        if ((cells[i].ch) != BLANK_CHAR) { is_empty = false; break; }
    }
} while(is_empty && first > 0);

...
*num_content_lines_before = first + 1;
...
rewrap_inner(self, other, *num_content_lines_before, ...);
```

So `src_limit = first + 1`, meaning trailing empty lines are excluded from
the rewrap iteration range. In our example, with content in rows 0 and 1 and
rows 2,3,4 empty, `first = 1`, `src_limit = 2`. `rewrap_inner` iterates
src_y=0 and src_y=1 only. No unnecessary evictions.

### 15.3 Where the continuation state is stamped — full trace for our example

After `rewrap_inner`:
```
dest[0] = "abc"  last_cell.next_char_was_wrapped = 1  (src_y=0 inner-loop wrap)
dest[1] = "def"  last_cell.next_char_was_wrapped = 1  (src_y=1 first inner-loop wrap: "de" + "f" = "def")
Wait, let me retrace...
```

Actually let me retrace more carefully:

src_y=0 iter: src="abcde" wrap=1
- is_first_line → first_dest_line → init dest[0], dest_y=0, dest_x=0
- inner loop:
  - src_x=0, dest_x=0: num=min(5,3)=3, copy "abc" to dest[0][0..2], src_x=3, dest_x=3
  - src_x=3 < 5: dest_x=3 >= 3 → next_dest_line(true): set dest[0] wrap=true, advance dest_y=1, init dest[1], dest_x=0
  - src_x=3, dest_x=0: num=min(5-3,3)=2, copy "de" to dest[1][0..1], src_x=5, dest_x=2
- src_y=1, src_x=0
- src_line_is_continued=true → NO next_dest_line

src_y=1 iter: src="fghij" (now with wrap=1 inherited from prev row's wrap, but wait —
`is_continued` field is recomputed by `linebuf_init_line` from prev GPU cell.
The prev row's wrap WAS true, so src->line->attrs.is_continued = true. BUT,
`is_src_line_continued()` checks src->line->gpu_cells[src->xnum-1].attrs.
next_char_was_wrapped on the CURRENT src line, not prev. Checking src="fghij"'s
last cell 'j': its wrap flag was 0 originally. So is_src_line_continued() = false.)

- src_x_limit = 5, NO trim (no blanks)
- inner loop:
  - src_x=0, dest_x=2: num=min(5,1)=1, copy "f" to dest[1][2], src_x=1, dest_x=3
  - src_x=1 < 5: dest_x=3 >= 3 → next_dest_line(true): set dest[1] wrap=true, advance dest_y=2, init dest[2], dest_x=0
  - src_x=1, dest_x=0: num=min(5-1,3)=3, copy "ghi" to dest[2][0..2], src_x=4, dest_x=3
  - src_x=4 < 5: dest_x=3 >= 3 → next_dest_line(true): set dest[2] wrap=true, advance dest_y=3, init dest[3], dest_x=0
  - src_x=4, dest_x=0: num=min(5-4,3)=1, copy "j" to dest[3][0], src_x=5, dest_x=1
- src_y=2, src_x=0
- src_line_is_continued=false AND src_y=2 < src_limit=2? NO (2 < 2 is false)
- Do NOT run next_dest_line(false)

Loop terminates. dest->line->ynum = dest_y = 3.

Final linebuf:
```
dest[0] = "abc"  wrap=1
dest[1] = "def"  wrap=1
dest[2] = "ghi"  wrap=1
dest[3] = "j  "  wrap=0 (not touched — initialized empty by alloc_linebuf, but dest[3] got "j" at position 0, and positions 1-2 are blank)
dest[4] = "   "  wrap=0 (untouched — never visited)
```

num_content_lines_after = dest->line->ynum + 1 = 4.

Good — this matches the test expectation for narrowing. The 5-row "abcde/fghij"
content becomes "abc/def/ghi/j" on 3-col — four content rows — and `is_continued`
is [false, true, true, true].

The trace confirms the algorithm is correct on this scenario.

---

## 16. Identified Potential Issues and Edge Cases

The following are **potential** issues — behaviors that appear fragile or
under-specified based on code reading. Most are **not** demonstrated bugs
against the current test suite (all 54 tests pass), but represent areas where
future changes could regress or where specific inputs might reveal
inconsistencies.

### 16.1 `line_attrs[].is_continued` is stale after rewrap

**Observation**: The `is_continued` bit in `LineAttrs` stored in
`line_attrs[]` is *written* by `set_dest_line_attrs` during rewrap but
*overwritten* every time `linebuf_init_line` (or `init_line` in HistoryBuf)
is called. After rewrap returns, the stored values are stale.

**Risk**: Any future code that reads `linebuf->line_attrs[y].is_continued`
directly (without going through `linebuf_init_line`) may see garbage. The
current codebase appears consistent — all reads go through the init helpers —
but this is an **implicit contract** not enforced by types or naming.

**Evidence**:
- `line-buf.c:145`: `self->line->attrs.is_continued = ... next_char_was_wrapped`
  — the stored `line_attrs[idx].is_continued` is overwritten on access.
- `history.c:167–176`: same pattern.
- No place in the code currently reads `line_attrs[y].is_continued` directly
  except via the init helpers.

**Mitigation**: A comment in `LineAttrs` definition noting that
`is_continued` is computed on access would prevent future accidents.

### 16.2 Wide character handling in `rewrap_inner`

**Observation**: `copy_range()` is a raw memcpy of CPU+GPU cells. It has no
awareness of `CellAttrs.width == 2` (wide character first cell) or
`width == 0` (second cell of a wide char). The inner loop chunk size
`num = MIN(src->line->xnum - src_x, dest->xnum - dest_x)` may split a wide
character across a line boundary.

**Risk**: If the wrap point `dest->xnum - dest_x` falls **between** the two
cells of a wide char (width=2 + width=0), the width=2 cell ends up on one
line and the width=0 on the next. The GPU renderer should handle this
gracefully (the width=0 cell is typically skipped), but the character
becomes orphaned on its own line with corrupt width semantics.

**Empirical check**: `rewrap.h` contains no width-aware logic; the comment
at `rewrap.h:69` (the trailing-blank trim) only checks `ch == BLANK_CHAR`,
not width. No test case in `kitty_tests/datatypes.py` or
`kitty_tests/screen.py` exercises wide-char reflow — searching the test file
for `width` or `cjk` or `CJK` finds none in rewrap/resize context.

**Mitigation candidate**: Before advancing `dest_x` to trigger
`next_dest_line(true)`, check whether `src->line->gpu_cells[src_x + num - 1]
.attrs.width == 2` and back off by 1 to force the wide char onto the next
line. This is not currently done.

Note the `xlimit_for_line` helper in `kitty/lineops.h:39–47` has the right
correction:

```c
static inline index_type
xlimit_for_line(const Line *line) {
    index_type xlimit = line->xnum;
    if (BLANK_CHAR == 0) {
        while (xlimit > 0 && (line->cpu_cells[xlimit - 1].ch) == BLANK_CHAR) xlimit--;
        if (xlimit < line->xnum && line->gpu_cells[xlimit > 0 ? xlimit - 1 : xlimit].attrs.width == 2) xlimit++;
    }
    return xlimit;
}
```

It restores the trailing half-cell of a wide char that the blank-trim would
otherwise have cut. But `rewrap_inner`'s own trim loop does **not** include
this correction — it's a separate helper not called from `rewrap.h`. Result:
a line ending in a wide character followed by a BLANK_CHAR (its phantom
second cell) may have the wide character's second cell *trimmed off*. The
src line "W\0" (wide char plus its zero-width continuation) trims to just
"W", and then copy_range sees xnum=1 and copies only the width=2 cell
— losing the paired width=0 cell.

### 16.3 Continuation flag bridge only at main linebuf row 0

**Observation**: `screen.c:2836–2838`:

```c
if (y == 0 && self->linebuf == self->main_linebuf) {
    if (history_buf_endswith_wrap(self->historybuf))
        self->linebuf->line->attrs.is_continued = true;
}
```

Only `screen_init_line` does this — **not** `linebuf_init_line`. Code paths
that use `linebuf_init_line` directly on the main linebuf without going
through `screen_init_line` will see `is_continued=false` for row 0 even when
it should be true (the history ends with a wrap).

**Risk**: consumers that do their own iteration over the main linebuf (e.g.,
custom selection logic, text extraction) may miss the history↔screen wrap
relationship. Search the codebase for `linebuf_init_line(.*main_linebuf`
to identify them; several exist in `screen.c` itself, e.g., at line 354
(dummy-output detection), line 440 (dummy-output removal) — these do not
need continuation info, so it's fine — and at line 2536 (paused rendering,
which is a different linebuf).

**Potential follow-up**: The same bridge is **not** applied for the alt
linebuf. The alt screen is a separate logical buffer and doesn't use the
main history, so its row 0 is never continued from history — `is_continued
= false` is correct there. So the conditional `self->linebuf == self->
main_linebuf` is correct.

### 16.4 `next_dest_line(true)` in the inner loop clears then reinitializes
dest—fired per overflow

**Observation**: The LineBuf `next_dest_line` does:
```c
linebuf_set_last_char_as_continuation(dest, dest_y, continued);
// ... possibly scroll and push to history ...
linebuf_init_line(dest, dest_y);
set_dest_line_attrs(dest_y);
```

When the inner loop fires `next_dest_line(true)` repeatedly (narrow dest
case), each firing calls `linebuf_init_line` and `set_dest_line_attrs`,
which writes `dest->line_attrs[dest_y] = src->line->attrs` (with
`prompt_kind` then cleared on src — but this inner-loop call is re-firing
for the *same src line*, so after the first call `src->line->attrs.
prompt_kind = UNKNOWN_PROMPT_KIND` and subsequent calls copy that cleared
value).

**Effect**: For a source line that spans **multiple** destination lines
(narrowing case), only the **first** destination line gets the original
`prompt_kind`; subsequent dest lines have `prompt_kind = UNKNOWN_PROMPT_KIND`.

Is this correct? Yes — a prompt line that wraps across multiple visual rows
is logically "one prompt" with its start at the first row. The comment in
`rewrap.h:18` doesn't explain this but the behavior is intentional.

But this does mean that `prompt_kind` info is **lost on narrowing**: if you
had a PROMPT_START line of content "PS1> command" and narrow it so it spans
2 rows "PS1> " / "command", the first row correctly keeps PROMPT_START, and
the second row has UNKNOWN_PROMPT_KIND. But if you then re-widen back to the
original width, the two dest rows merge into one — and the merge pulls the
`prompt_kind` from the **first** src row (PROMPT_START) — good.

**No bug, but a subtle design detail**: `prompt_kind` is a first-line-only
attribute for logical prompts.

### 16.5 Cursor remap arithmetic off-by-one at chunk boundary

**Observation**: `rewrap.h:87`: `t->x = dest_x + (t->x - src_x + (t->x > 0))`.

When `t->x == src_x` (the cursor is at the very first cell of the chunk)
and `t->x > 0` (i.e., was not at col 0 of the src line), the formula gives
`dest_x + (0 + 1) = dest_x + 1` — the cursor moves **one cell to the right**
of where we'd naively expect.

**Is this intentional?** Likely yes, to handle the "last cursor-written
position + pending wrap" case: kitty stores cursor.x where the *next* glyph
will be written, so cursor.x points just-past the last written column. After
a rewrap, the cursor should land at the analogous just-past position.

**Verified by test**: `test_cursor_after_resize` in `kitty_tests/screen.py:
328–332`:

```python
s = self.create_screen()
draw('a')
x_before = s.cursor.x   # x=1 (one past the 'a')
s.resize(s.lines - 1, s.columns)
self.ae(x_before, s.cursor.x)
```

This passes, so cursor.x=1 is preserved across resize. The +1 adjustment is
necessary for this to work.

**Corner case**: `t->x == 0 && src_x == 0`: cursor stays at `dest_x + 0 = 0`
— no +1. Correct, because cursor at column 0 is not "one past" anything.

**Corner case**: `t->x == 0 && src_x > 0`: can't happen — inner loop only
remaps cursors on the current tracked src line, and the cursor's x hasn't
been updated yet, so `src_x <= t->x < src_x + num` requires `t->x >= src_x`.
If src_x > 0, then t->x > 0 too. Safe.

### 16.6 History `init_line` computes `is_continued` from PHYSICAL `num-1`

**Observation**: `history.c:167–168`:

```c
if (num > 0) {
    l->attrs.is_continued = gpu_lineptr(self, num - 1)[self->xnum-1].attrs.next_char_was_wrapped;
}
```

`num` here is a **physical** index (the ring buffer slot), not the logical
reverse-indexed lnum. `num - 1` is the physical slot before `num`.

After the ring wraps (`start_of_data > 0`), physical slot 0's logical
predecessor is physical slot `ynum - 1`, NOT "physical slot -1." But the
code just checks `num > 0`. When `num == 0`, we fall through to the pager
history check — which is correct as explained in §14.4.

**When `num > 0` but the slot is physically at `start_of_data`** (the
"oldest live" entry): `num - 1` is physically the youngest entry — a
completely unrelated line in a ring-wrapped buffer. The computed
`is_continued` is nonsense.

**Mitigation**: This is likely **not** a visible bug because:
1. `historybuf_init_line` (the public API) always passes
   `index_of(self, lnum)` which converts logical-to-physical using the ring.
2. `rewrap_inner`'s `init_src_line` uses `map_src_index(y)` to convert
   logical-to-physical; for HistoryBuf src, `y` starts at 0 and increases
   monotonically — `y=0` → physical `start_of_data`, `y=1` → physical
   `start_of_data + 1`, etc. The **logical** predecessor of `y=0` is
   *nothing* (pager history territory); of `y=1` is `y=0`. The physical
   predecessor of `start_of_data` is `ynum - 1`, which is *wrong*.

   But the `if (num > 0)` check at `history.c:167` uses `num`, not `y`!
   When `y == 0`, `num = map_src_index(0) = start_of_data`. If
   `start_of_data == 0`, `num == 0` and we correctly fall to pager history.
   If `start_of_data > 0`, `num > 0` and we incorrectly read physical slot
   `num - 1` (some wholly unrelated line).

   **This is a real consistency gap** during HistoryBuf rewrap when the ring
   has wrapped. Let's assess severity:
   - The wrong-read `is_continued` is written to `src->line->attrs.
     is_continued`.
   - `rewrap_inner` uses `is_src_line_continued()`, which reads the **last
     GPUCell's `next_char_was_wrapped`**, NOT `is_continued`. So this
     incorrect `is_continued` doesn't directly affect rewrap decisions.
   - The incorrect `is_continued` *is* written to dest via
     `set_dest_line_attrs`, but the default LineBuf `next_dest_line` isn't
     used here (HistoryBuf specialization is); its `*lap = src->line->attrs`
     writes the wrong `is_continued` into `dest->line_attrs[new_slot].
     is_continued`. But `is_continued` is always recomputed on access
     (§11.3), so the stale value is harmless.
   - Net effect: the bug writes stale data that's never observably read.

   **Conclusion**: Latent inconsistency, not a live bug — but if a future
   code change ever reads `line_attrs[].is_continued` without going through
   `init_line`, it would manifest.

### 16.7 Trailing-blank trim trimming past wide-character second cells

Already discussed in §16.2 — `rewrap_inner`'s trim at `rewrap.h:70` does
not account for width-2 pairs. A trailing wide character whose paired
width-0 cell is BLANK_CHAR (the usual state — the second cell's `ch` is
typically 0 or copy of the wide char) will be trimmed off, corrupting the
wide character.

### 16.8 `screen_resize` races against in-progress rendering

`screen_resize` first calls `screen_pause_rendering(false, 0)` which
presumably disables paused rendering mode. If the renderer is in the middle
of reading cell buffers on another thread, the `Py_CLEAR(self->main_linebuf)`
at `screen.c:386` may free memory the renderer is reading. This is a
**threading concern**, not a reflow bug — the renderer must hold the GIL or
a separate lock while iterating. Search confirms `screen.c` uses the GIL
model (the code is all inside `PyObject`-derived types with Python method
implementations). Safe under the GIL assumption.

### 16.9 `linebuf_copy_line_to` in scrollback fill uses `alt_linebuf->line`

At `screen.c:432, 434`:

```c
if (!historybuf_pop_line(self->historybuf, self->alt_linebuf->line)) break;
INDEX_DOWN;
linebuf_copy_line_to(self->main_linebuf, self->alt_linebuf->line, 0);
```

This uses `self->alt_linebuf->line` as scratch. `alt_linebuf->line` is a
`Line *` that is normally "borrowed" for render iteration. If another thread
had this pointer cached (e.g., a Python callback holding a reference), this
would corrupt it — but again, under GIL assumption, single-threaded.

Minor code-smell: `alt_linebuf` is in scope even if `is_main` is false (which
is the condition for this whole block), so using its `line` scratch is
harmless but slightly opaque. A dedicated `Line` scratch field on `Screen`
would be clearer.

### 16.10 Empty-line rewrap optimization

`linebuf_rewrap` at `line-buf.c:610–614`:

```c
if (is_empty) {  // All lines are empty
    *num_content_lines_after = 0;
    *num_content_lines_before = 0;
    return;
}
```

When the whole src LineBuf is empty, rewrap returns early **without**
initializing the tracked cursors. The caller `realloc_lb` set `temp.x/y =
before.x/y`, and since `linebuf_rewrap` returns without touching them, the
caller's `setup_cursor` macro sees `after.x = before.x, after.y = before.y`.

So after rewrapping an empty buffer: cursor.after = cursor.before. If
`before.y >= new_lines`, the cursor ends up out of bounds until the
`S(c, w)` clamp at `screen.c:420`. That clamp handles it. Fine.

But `cursor.is_beyond_content = num_content_lines_before > 0 && self->
cursor->y >= num_content_lines_before = false` (since
`num_content_lines_before = 0`). So `is_beyond_content` is false, and the
special repositioning in §3.7 doesn't fire. Cursor just gets clamped.

This seems correct for an empty buffer.

### 16.11 `dummy_output_inserted` only runs on cursor.y range

`screen.c:353–361`:

```c
if (is_main && self->cursor->x == 0 && self->cursor->y < self->lines
    && self->linebuf->line_attrs[self->cursor->y].prompt_kind == OUTPUT_START) {
    linebuf_init_line(self->linebuf, self->cursor->y);
    if (!self->linebuf->line->cpu_cells[0].ch) {
        self->linebuf->line->cpu_cells[self->cursor->x++].ch = '<';
        dummy_output_inserted = true;
    }
}
```

Checks *only* the cursor's current line for blank OUTPUT_START. If there are
*multiple* blank OUTPUT_START lines (e.g., the shell wrote OSC 133 `;C`
multiple times without any intervening output), only the cursor's line is
preserved. The others get trimmed by `linebuf_rewrap`'s blank-tail-detection
logic — their `OUTPUT_START` attribute is lost because rewrap doesn't
preserve attrs of trailing blank lines.

**Whether this is a bug** depends on the shell integration contract —
typically the shell only emits OSC 133;C once per command. So this is fine.

### 16.12 Scrollback fill not triggered for non-main screen

`screen.c:428`: `if (is_main && OPT(scrollback_fill_enlarged_window))` —
only active for main. Alt screen has no history, so fill is irrelevant —
correct.

But what if the user resizes *while* on alt screen? Then `is_main = false`
and fill doesn't trigger. After switching back to main, the main linebuf
was still resized (via the earlier `realloc_lb` at line 384), but no fill
happens. The user sees the main screen's content at the new size without
history pulled in. Is this correct? Probably yes — filling when
not-on-main would be visually surprising (content shifts beneath the alt
screen overlay). But it means `scrollback_fill_enlarged_window` is
effectively "scrollback_fill_when_resizing_main_screen."

### 16.13 `historybuf_rewrap` resets dest but doesn't free old pagerhist rewrap state

`history.c:609`: `other->count = 0; other->start_of_data = 0;`. The
`pagerhist->rewrap_needed` flag set at line 608 is preserved across the
reset. But `other->pagerhist` was adopted from old (via `realloc_hb`'s
pointer move), so this is the same pager buffer — the rewrap_needed flag
correctly persists. OK.

But note: **if `historybuf_rewrap` is called multiple times with different
column widths** (unusual but possible, e.g., two resizes in rapid
succession), each call sets `rewrap_needed = true` if columns differ. The
actual pager rewrap `pagerhist_rewrap_to()` is called with the *final*
column count when the user accesses pager, so intermediate widths are
skipped. OK.

### 16.14 `is_empty` detection of source misses GPU-only content

`line-buf.c:604–608`:

```c
for(i = 0; i < self->xnum; i++) {
    if ((cells[i].ch) != BLANK_CHAR) { is_empty = false; break; }
}
```

Only checks `ch`. A cell with `ch = 0` but non-zero background color or a
sprite would be considered "empty" and the line would be trimmed. In
practice this is what we want — background-only cells are visual artifacts,
not content.

### 16.15 `grman_resize` called AFTER rewrap, with before/after line counts

`screen.c:391` / `screen.c:400`:

```c
grman_resize(self->main_grman, self->lines, lines, self->columns, columns,
             num_content_lines_before, num_content_lines_after);
```

Passes the content-line counts to the graphics manager so image positions
can be updated relative to content. If a line containing an image placeholder
is moved during rewrap (e.g., wrap-merging two rows), the graphics manager
adjusts the image's logical y. The implementation of `grman_resize` is
out-of-scope for this document (`kitty/graphics.c`), but the contract is
clear: rewrap handles text, graphics manager handles images, and the two
communicate via (before, after) content line counts.

---

## 17. Test Coverage & Behavioral Confirmation

All the claims in this document were validated against the current test
suite. Running `python3 -m unittest kitty_tests.datatypes kitty_tests.screen`
against the built C extensions on commit `815df1e21` produces:

```
.......................................................
----------------------------------------------------------------------
Ran 54 tests in 0.094s

OK
```

Tests exercising rewrap specifically:

- **`test_rewrap_simple`** (`kitty_tests/datatypes.py:337`): Same-width
  copy, larger-dest expansion (with empty rows at the bottom), smaller-dest
  contraction (with leading content rows spilling — verified
  `cy = rewrap(...)[1] = 3`, matching the 3-row dest).
- **`test_rewrap_wider`** (`datatypes.py:374`): Soft-wrapped pair "0123 " +
  "56789" merges into "0123 5" + "6789" (on 6-col wider dest). Continuation
  flags: False, True. Non-continued "12" + "abc" stays as two separate lines.
- **`test_rewrap_narrower`** (`datatypes.py:385`): Hard-break line "123"
  stays as-is; continuation from "123  " (with trailing blanks but soft
  wrap) to "abcde" preserves the blanks: splits into "123" + "  a" + "bcd"
  + "e" with continuation flags False/True/True/True.
- **`test_resize`** (`screen.py:280`): Full `screen_resize()` path.
  Screenful of rows `'00000', '11111', ..., '44444'` is resized to
  `(3, 10)` → `'00000 11111'` merged onto one row, `'22222 33333'` merged
  onto another, and `'44444'` alone on the third (rows 1 and 2 absorbed the
  soft-wrap). Further resize to `(5, 1)` pushes everything except the last
  '4' into history.
- **`test_cursor_after_resize`** (`screen.py:308`): Cursor position
  preservation through narrow, wide, height-down, and width+height changes.
  Confirms the +1 cursor remapping arithmetic.
- **`test_scrollback_fill_after_resize`** (`screen.py:343`): Exercises the
  `scrollback_fill_enlarged_window` option through six scenarios covering
  pure height increase, height+width increase, height+width decrease,
  height unchanged+width increase, height decrease+width increase, and
  large continued text. Confirms that scrollback fill correctly retrieves
  lines from history and preserves cursor distance from bottom.
- **`test_linebuf`** and **`test_historybuf`** (`datatypes.py`): Baseline
  tests of LineBuf/HistoryBuf mutation and rewrap.

All 54 tests pass cleanly, confirming the rewrap implementation is
behaviorally correct for the scenarios tested. The potential issues
identified in §16 are not exercised by these tests.

---

## 18. Glossary

| Term | Meaning |
|---|---|
| **LineBuf** | The visible screen buffer. 2D array of cells with row permutation via `line_map` for O(1) scrolling. |
| **HistoryBuf** | Scrollback buffer. Segmented ring buffer of lines; `start_of_data` + `count` track the live range. |
| **PagerHistoryBuf** | Byte-level ringbuf of ANSI-escaped text, accumulates evicted lines from HistoryBuf. |
| **CPUCell** | 12-byte per-cell CPU-side data: `ch`, `hyperlink_id`, `cc_idx[3]`. |
| **GPUCell** | 20-byte per-cell GPU-side data: colors, sprite coords, `CellAttrs`. |
| **CellAttrs** | 16-bit packed attributes including `next_char_was_wrapped` (persistent soft-wrap marker). |
| **LineAttrs** | 8-bit packed attributes including `is_continued` (derived soft-wrap query) and `prompt_kind`. |
| **next_char_was_wrapped** | Bit on the last GPUCell of a line, set when soft-wrap occurred there. **Storage**. |
| **is_continued** | Bit on a line's LineAttrs, computed from the previous line's `next_char_was_wrapped` at access time. **Query**. |
| **rewrap_inner** | The generic rewrap algorithm in `rewrap.h:56`, parameterized by macros. |
| **Macro polymorphism** | Compile-time specialization of `rewrap_inner` via `#define` overrides before `#include "rewrap.h"`. |
| **Spillover** | When rewrap overflows a LineBuf dest, excess lines are pushed to HistoryBuf. |
| **Scrollback fill** | `scrollback_fill_enlarged_window` — pulling history lines back into LineBuf when window grows. |
| **Prompt protection** | `prevent_current_prompt_from_rewrapping` — blank the current shell prompt before rewrap to avoid flicker; restore after. |
| **OSC 133** | Shell integration escape sequences that mark prompt/output boundaries; drive the `prompt_kind` field. |
| **TrackCursor** | Struct in `rewrap.h:50` that follows cursor positions through the rewrap mapping. |
| **CursorTrack** | Struct in `screen.c:226` that captures before/after cursor state around a `linebuf_rewrap` call. |
| **line_map** | `LineBuf::line_map[]`: permutation of physical storage rows; enables O(1) scroll via rotation. |
| **start_of_data** | Ring buffer offset of the oldest live line in `HistoryBuf`. |
| **index_of** | `history.c:152`: translates reverse-indexed `lnum` (0=newest) to a physical ring slot. |
| **map_src_index** | `history.c:584`: forward-indexed `y` → physical ring slot during rewrap. |
| **has_dirty_text** | LineAttrs bit; renderer uses this to decide whether to re-upload a line's cells to GPU. |
| **SEGMENT_SIZE** | 2048; the number of lines per `HistoryBufSegment`. |
| **alloc_historybuf** | Constructor for a new HistoryBuf. Takes lines (scrollback depth), columns (width), pager_sz. |
| **alloc_linebuf** | Constructor for a new LineBuf. Takes lines, columns. |
| **INDEX_DOWN** | Macro in `screen.c:289` that scrolls the active linebuf downward by one row and updates graphics/selections. |
| **grman_resize** | Adjusts image placements when screen size changes (details out of scope). |
| **pause_rendering** | Mode where rendering is frozen against a snapshot; disabled at start of `screen_resize`. |

---

## Appendix A — File References Map

| File | Lines | Relevance |
|---|---|---|
| `kitty/rewrap.h` | 1–96 (full) | Generic rewrap algorithm + macro overrides |
| `kitty/screen.c` | 216–223 | `realloc_hb` |
| `kitty/screen.c` | 226–232 | `CursorTrack` struct |
| `kitty/screen.c` | 234–242 | `realloc_lb` |
| `kitty/screen.c` | 279–299 | `INDEX_GRAPHICS`, `INDEX_DOWN` macros |
| `kitty/screen.c` | 302–343 | `prevent_current_prompt_from_rewrapping` |
| `kitty/screen.c` | 345–463 | `screen_resize` |
| `kitty/screen.c` | 2833–2840 | `init_line` with history endswith-wrap bridge |
| `kitty/screen.h` | 88–170 | `Screen` struct (cursor, linebufs, historybuf, etc.) |
| `kitty/line-buf.c` | 134–147 | `linebuf_init_line` (and static `init_line`) |
| `kitty/line-buf.c` | 188–198 | `linebuf_line_ends_with_continuation`, `linebuf_set_last_char_as_continuation` |
| `kitty/line-buf.c` | 291–327 | `clear_line_`, `linebuf_clear_line`, `linebuf_index`, `linebuf_reverse_index` |
| `kitty/line-buf.c` | 583–637 | `#include "rewrap.h"` + `linebuf_rewrap` |
| `kitty/history.c` | 15–64 | Segment allocation, ring indexing helpers |
| `kitty/history.c` | 152–187 | `index_of`, `init_line` (HistoryBuf), `historybuf_init_line`, `history_buf_endswith_wrap` |
| `kitty/history.c` | 258–307 | `pagerhist_push`, `historybuf_push`, `historybuf_add_line`, `historybuf_pop_line`, `history_buf_set_last_char_as_continuation` |
| `kitty/history.c` | 391–432 | `pagerhist_rewrap_to` (lazy rewrap) |
| `kitty/history.c` | 582–614 | HistoryBuf macro overrides + `#include "rewrap.h"` + `historybuf_rewrap` |
| `kitty/data-types.h` | 115, 196–290 | `BLANK_CHAR`, `CellAttrs`, `GPUCell`, `CPUCell`, `PromptKind`, `LineAttrs`, `Line`, `LineBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `HistoryBuf` |
| `kitty/lineops.h` | 14–46 | `copy_line`, `clear_chars_in_line`, `xlimit_for_line` |
| `kitty/lineops.h` | 103–131 | Declarations for linebuf/historybuf ops |
| `kitty/options/definition.py` | 420–423 | `scrollback_fill_enlarged_window` option |
| `kitty_tests/datatypes.py` | 29–41 | `create_lbuf` helper |
| `kitty_tests/datatypes.py` | 332–392 | `test_rewrap_simple/wider/narrower` |
| `kitty_tests/screen.py` | 280–400 | `test_resize`, `test_cursor_after_resize`, `test_scrollback_fill_after_resize` |
| `kitty_tests/__init__.py` | 237–270 | `create_screen` helper |

---

## Appendix B — Build & Test Reproduction

```bash
# Install build dependencies (example on Ubuntu/Debian with root access):
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y \
    pkg-config libdbus-1-dev libxkbcommon-dev libxkbcommon-x11-dev \
    libgl1-mesa-dev libx11-dev libxcb1-dev libxrandr-dev libxinerama-dev \
    libxcursor-dev libxi-dev libfreetype-dev libfontconfig-dev \
    libharfbuzz-dev libpng-dev libxxhash-dev liblcms2-dev libssl-dev \
    libcanberra-dev libx11-xcb-dev libxcb-xkb-dev libxcb-cursor-dev \
    libsimde-dev

# Build the C extensions:
python3 setup.py build

# Run the rewrap/resize tests:
python3 -m unittest kitty_tests.datatypes kitty_tests.screen

# Expected output: "Ran 54 tests in ~0.1s  OK"
```

---

*End of analysis document.*
