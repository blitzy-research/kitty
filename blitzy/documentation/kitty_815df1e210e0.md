# Kitty Scrollback History Buffer Under Stress — Empirical Source Analysis

**Commit**: `815df1e21` ("Wire up applying of font config")
**Source branch**: `kitty_815df1e210e0`
**Scope**: Answers three investigative questions about kitty's scrollback history buffer memory behavior, scroll responsiveness, and buffer boundary transitions, strictly grounded in source-code inspection of commit `815df1e21`.

All claims below cite a specific source file and line range. Numbers are derived from the exact struct sizes, constants, and allocation paths found in the code — not from empirical runs or speculation. Every reader should be able to reproduce every number by opening the cited file at the cited line.

---

## Executive Summary

**Question 1:** *"When hundreds of thousands of lines of output are generated rapidly, what happens to memory consumption — does it grow unbounded, or does something cap it?"*

Memory grows in discrete, predictable steps until the scrollback buffer is fully populated, after which it plateaus. Each step is an `add_segment()` call (`kitty/history.c:17-29`) that performs a single `calloc()` of `xnum * SEGMENT_SIZE * (sizeof(CPUCell) + sizeof(GPUCell)) + SEGMENT_SIZE * sizeof(LineAttrs)` bytes — at 80 columns this is exactly **5 244 928 bytes (≈ 5.002 MiB)** per 2048 lines. Once `num_segments` reaches `ceil(ynum / 2048)`, no further segments are allocated: `historybuf_push()` at `kitty/history.c:275-284` reuses slots via modular arithmetic, overwriting in place. If the optional pager history is enabled, evicted lines are ANSI-serialized into a ring buffer that grows in 1 MiB increments up to its configured ceiling (`kitty/history.c:89-101`). **Bottom line:** memory grows once to a deterministic ceiling of `ceil(scrollback_lines / 2048) × segment_bytes` and then stays flat — the circular buffer overwrites in place and never grows further.

**Question 2:** *"When scrolling back through a very large history while new output is still being generated, is the terminal responsive — or does scrolling cause observable lag?"*

Scroll rendering cost is independent of the scrollback depth: `screen_update_cell_data()` at `kitty/screen.c:2737-2797` iterates exactly `self->lines` times — the visible viewport — regardless of how many millions of lines sit in `historybuf->count`. The GPU upload size is `sizeof(GPUCell) * screen->lines * screen->columns` (`kitty/shaders.c:408-409`), i.e., a constant 20 × rows × columns per dirty frame. During concurrent output, `history_line_added_count` accumulates in `INDEX_UP` (`kitty/screen.c:1559`) and is folded into `scrolled_by` at render time (`kitty/screen.c:2761`), anchoring the user's view so it does not drift. The I/O thread coalesces wakeups by `input_delay` (default 3 ms, `kitty/options/definition.py:878`) and the main thread gates rendering by `repaint_delay` (default 10 ms, `kitty/options/definition.py:866`), yielding a ~100 FPS cap and ~13 ms worst-case scroll-to-display latency. **Bottom line:** rendering is O(visible×columns) not O(history) — scrollback size does not affect scroll responsiveness.

**Question 3:** *"When does the buffer's internal storage grow — at what boundaries, and can those transitions be observed externally?"*

HistoryBuf storage grows in segment-sized increments: the first segment is allocated eagerly in `create_historybuf()` at `kitty/history.c:127`; every subsequent segment is allocated lazily by `segment_for()` (`kitty/history.c:36-42`) the first time a line would be pushed into a not-yet-allocated segment. The transition boundaries at 80 columns are every 2048 newly pushed lines, each triggering a `realloc` of the 24-byte-per-entry segment pointer array plus a single `calloc` of 5 244 928 bytes. The pager history ring buffer grows differently: it starts at `MIN(1 MiB, configured_max)` (`kitty/history.c:67`) and extends in 1 MiB increments via `pagerhist_extend()` (`kitty/history.c:89-101`), which allocates a new ring buffer, copies the existing contents, and frees the old one — a copy-on-extend, not in-place realloc. **Bottom line:** every 2048 lines pushed into history produces one observable ~5 MiB allocation; every 1 MiB of pager history demand produces one observable extend-and-copy.

---

## Source Code Foundation

All claims below cite the file and line number(s) in this table. Line numbers correspond to commit `815df1e21`.

| File | Line Range Read | Relevance Summary |
|------|-----------------|-------------------|
| `kitty/history.c` | 15, 17-29, 31-34, 36-42, 44-48, 67, 70-80, 89-101, 117-133, 218-226, 258-273, 275-284, 577-579 | Core scrollback implementation — segment allocation, circular push, pager-history ring-buffer growth, creation-time eager allocation |
| `kitty/data-types.h` | 221, 228, 231-239, 262-266, 268-272, 282-290 | Cell and segment struct sizes; `static_assert` pins of 20 / 12 / 1 bytes; HistoryBuf/HistoryBufSegment/PagerHistoryBuf layouts |
| `kitty/screen.c` | 130, 1552-1567, 1570-1577, 1589-1598, 1907-1911, 1913-1920, 2505-2544, 2597-2601, 2713-2735, 2737-2797, 2842-2853, 4090-4118 | Screen wiring, scroll+index path, dirty/pause rendering, O(visible) render loops, visual-line mapping, user-facing scroll API |
| `kitty/screen.h` | 88-115 | `Screen` struct fields: `scrolled_by`, `is_dirty`, `scroll_changed`, `history_line_added_count`, `historybuf` |
| `kitty/shaders.c` | 393-418, 608-630 | `cell_prepare_to_render()` GPU upload size formula; `draw_scroll_indicator()` `scrolled_by / historybuf->count` position |
| `kitty/child-monitor.c` | 437-448, 870-896, 1337-1356, 1480-1578 | I/O thread loop, wake-up coalescing by `input_delay`, render gated by `repaint_delay`, `read_bytes` path |
| `kitty/vt-parser.c` | 18, 21, 1416-1446, 1450-1462 | `BUF_SZ = 1 MiB` per-parser input buffer; `run_worker()` flush condition; `vt_parser_create_write_buffer()` remaining-space API |
| `kitty/line-buf.c` | 85-120 | `alloc_linebuf()` five-call allocation pattern for contrast with HistoryBuf's single-call pattern |
| `kitty/options/definition.py` | 372, 406, 866, 878 | Default values: `scrollback_lines=2000`, `scrollback_pager_history_size=0`, `repaint_delay=10`, `input_delay=3` |
| `kitty/options/utils.py` | 557-566 | Parsers: negative `scrollback_lines` → 2³² − 1; `scrollback_pager_history_size` MB-to-bytes capped at 4 GiB − 1 |
| `3rdparty/ringbuf/ringbuf.h` | 40-41, 52, 72-73, 79-80, 86-87 | Ring-buffer API: `ringbuf_new`, `ringbuf_capacity`, `ringbuf_bytes_free`, `ringbuf_bytes_used` |
| `3rdparty/ringbuf/ringbuf.c` | 42-66, 92 | `struct ringbuf_t`, `capacity+1` allocation for full-vs-empty disambiguation, capacity = buffer_size − 1 |
| `kitty_tests/datatypes.py` | 487, 501-510 | 3000-line HistoryBuf stress test — reference validation of circular push order |
| `kitty_tests/screen.py` | 695-741 | `test_pagerhist` — reference validation of ring-buffer overflow, rewrap, multi-byte behavior |
| `kitty_tests/__init__.py` | 184-189, 224, 237-241 | `filled_history_buf()`, default `scrollback_pager_history_size=1024`, `create_screen()` helpers |

---

## Memory Model — Struct Sizes and Per-Line Costs

### Cell Sizes (Static Asserts)

The Cell struct sizes are not assumed — they are pinned at compile time by `static_assert` declarations:

```c
// kitty/data-types.h:221
static_assert(sizeof(GPUCell) == 20, "Fix the ordering of GPUCell");
```

```c
// kitty/data-types.h:228
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
```

`LineAttrs` is a union wrapping a single `uint8_t` with packed bitfields (`kitty/data-types.h:230-239`):

```c
typedef enum { UNKNOWN_PROMPT_KIND = 0, PROMPT_START = 1, SECONDARY_PROMPT = 2, OUTPUT_START = 3 } PromptKind;
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

The `prompt_kind` field uses the `PromptKind` enum typedef (a `uint8_t`-backed enum declared on the immediately preceding line). The union's underlying storage is `uint8_t val` → `sizeof(LineAttrs) == 1` byte.

### Segment Layout

A `HistoryBufSegment` is three pointers that refer to three co-located sub-arrays in one contiguous allocation (`kitty/data-types.h:262-266`):

```c
typedef struct {
    GPUCell *gpu_cells;
    CPUCell *cpu_cells;
    LineAttrs *line_attrs;
} HistoryBufSegment;
```

The allocation itself is a single `calloc()` with manual pointer arithmetic (`kitty/history.c:23-28`):

```c
const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));
if (!s->cpu_cells) fatal("Out of memory allocating new history buffer segment");
s->gpu_cells = (GPUCell*)(((uint8_t*)s->cpu_cells) + cpu_cells_size);
s->line_attrs = (LineAttrs*)(((uint8_t*)s->gpu_cells) + gpu_cells_size);
```

All three sub-arrays share one backing allocation. A single `free(s->cpu_cells)` releases all three (`kitty/history.c:31-34`):

```c
static void
free_segment(HistoryBufSegment *s) {
    free(s->cpu_cells); memset(s, 0, sizeof(HistoryBufSegment));
}
```

The `memset(s, 0, sizeof(HistoryBufSegment))` is equivalent to clearing all three pointer fields (`cpu_cells`, `gpu_cells`, `line_attrs`) since `HistoryBufSegment` is exactly three pointer fields (`kitty/data-types.h:262-266`).

The segment constant is fixed at 2048 lines (`kitty/history.c:15`):

```c
#define SEGMENT_SIZE 2048
```

### Per-Line Cost

| Component | Bytes | Source |
|-----------|-------|--------|
| `sizeof(CPUCell)` | 12 | `kitty/data-types.h:228` |
| `sizeof(GPUCell)` | 20 | `kitty/data-types.h:221` |
| `sizeof(LineAttrs)` | 1 | `kitty/data-types.h:231-239` (union over `uint8_t val`) |
| Per-cell (CPU + GPU) | 32 | 12 + 20 |
| Per-line cells at xnum=80 | 2 560 | 80 × 32 |
| Per-line LineAttrs | 1 | one byte per line |
| **Per-line total (80 cols)** | **2 561** | **2 560 + 1** |

### Per-Segment Cost at Common Widths

Plugging exact numbers into the `history.c:23-28` formula `cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs)`:

| xnum (cols) | `cpu_cells_size` | `gpu_cells_size` | LineAttrs bytes | **Segment total (bytes)** | **MiB** |
|-------------|------------------|------------------|-----------------|---------------------------|---------|
| 80 | 1 966 080 | 3 276 800 | 2 048 | **5 244 928** | **5.002** |
| 132 | 3 244 032 | 5 406 720 | 2 048 | **8 652 800** | **8.252** |
| 200 | 4 915 200 | 8 192 000 | 2 048 | **13 109 248** | **12.502** |
| 256 | 6 291 456 | 10 485 760 | 2 048 | **16 779 264** | **16.002** |

Worked example at xnum = 80:
`cpu_cells_size = 80 × 2048 × 12 = 1 966 080`
`gpu_cells_size = 80 × 2048 × 20 = 3 276 800`
`LineAttrs = 2048 × 1 = 2 048`
`total = 1 966 080 + 3 276 800 + 2 048 = 5 244 928 bytes = 5 244 928 / 1 048 576 ≈ 5.002 MiB`.

### Contrast with Active LineBuf

The visible `LineBuf` — by contrast with the HistoryBuf's single-`calloc` strategy — uses **five separate** allocations via `PyMem_Calloc` (`kitty/line-buf.c:90-98`):

```c
b->cpu_cell_buf = PyMem_Calloc(xnum * ynum, sizeof(CPUCell));
b->gpu_cell_buf = PyMem_Calloc(xnum * ynum, sizeof(GPUCell));
b->line_map = PyMem_Calloc(ynum, sizeof(index_type));
b->scratch = PyMem_Calloc(ynum, sizeof(index_type));
b->line_attrs = PyMem_Calloc(ynum, sizeof(LineAttrs));
```

This is a deliberate design asymmetry: history is read-mostly so a dense, contiguous segment is preferred and one free() clears everything, whereas `LineBuf` needs auxiliary `line_map` and `scratch` arrays to implement scroll-region operations, so keeping these as separately sized PyMem regions simplifies scroll bookkeeping. The HistoryBuf layout trades flexibility for density; the LineBuf layout trades density for flexibility.

---

## Investigative Area 1 — Memory Under Heavy Output

### Allocation Chain

When a terminal application emits a newline that pushes a line off the bottom of the viewport, the allocation path is:

```
screen_scroll()                             kitty/screen.c:1589-1598
  └─> INDEX_UP(add_to_history=true)         kitty/screen.c:1552-1567
        ├─> linebuf_init_line(linebuf, bottom)
        ├─> historybuf_add_line(historybuf, line, as_ansi_buf)   kitty/screen.c:1558
        │     └─> historybuf_push()                               kitty/history.c:275-284
        │           ├─> init_line() → cpu_lineptr/gpu_lineptr/attrptr
        │           │     └─> segment_for(num, y)                 kitty/history.c:36-42
        │           │           └─> [if needed] add_segment()     kitty/history.c:17-29
        │           │                 ├─> realloc(segments, ...)  kitty/history.c:20
        │           │                 └─> calloc(1, cpu+gpu+la)   kitty/history.c:25
        │           └─> [if count == ynum] pagerhist_push()       kitty/history.c:258-273
        │                 └─> pagerhist_write_bytes()             kitty/history.c:218-226
        │                       └─> [if needed] pagerhist_extend() kitty/history.c:89-101
        │                             ├─> ringbuf_new(newsz)
        │                             ├─> ringbuf_copy(newbuf, ph->ringbuf, count)
        │                             └─> ringbuf_free(&ph->ringbuf)
        ├─> history_line_added_count++            kitty/screen.c:1559
        ├─> linebuf_clear_line(linebuf, bottom)
        └─> is_dirty = true
```

Two conditions gate each step:

- **`historybuf_add_line` only runs when `add_to_history` is true.** `INDEX_UP`'s caller `screen_index()` (`kitty/screen.c:1573-1577`) computes `add_to_history = linebuf == main_linebuf && margin_top == 0` — so lines scrolled out of the alternate screen or out of a scroll region are discarded, not captured to history.
- **`pagerhist_push` only runs when the history buffer is full.** `historybuf_push` at `kitty/history.c:279` tests `if (self->count == self->ynum)` before calling `pagerhist_push`. In a young history buffer (still filling for the first time), no pager-history writes occur.

### Capacity Formula

At Screen creation, `kitty/screen.c:130` calls:

```c
self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));
```

`alloc_historybuf(lines, columns, pagerhist_sz)` maps the first two arguments to `create_historybuf(type, columns, lines, pagerhist_sz)` (`kitty/history.c:577-579`) — so inside `history.c`, `xnum` is the column count and `ynum` is the (possibly inflated) scrollback line count. The `MAX(scrollback, lines)` guarantee at `screen.c:130` means even `scrollback_lines=0` produces a HistoryBuf with `ynum = screen_lines`.

Critically, `create_historybuf()` **eagerly allocates one segment at initialization** by calling `add_segment(self)` at `kitty/history.c:127`. So the minimum HistoryBuf memory at Screen creation is already **one segment** — at 80 cols that is 5 244 928 bytes, not zero. This contradicts a naive reading of "lazy allocation": the first segment is eager, and only the second-and-onwards segments are lazy.

Segment count is bounded by:

```
num_segments_max = ceil(ynum / SEGMENT_SIZE) = ceil(max(scrollback_lines, screen_lines) / 2048)
```

which is enforced by the guard `SEGMENT_SIZE * self->num_segments < self->ynum` in `segment_for()` at `kitty/history.c:36-42`.

### Steady-State Memory Table (80 columns)

The following values are the **maximum** resident bytes consumed by the HistoryBuf (not counting pager history). They are reached once the buffer first fills and `num_segments` stops growing; thereafter memory is flat.

| `scrollback_lines` | Effective `ynum` | `num_segments = ceil(ynum/2048)` | Steady-state bytes | Steady-state MiB |
|--------------------|------------------|----------------------------------|--------------------|------------------|
| 2 000 (default) | max(2000, screen_lines) → 2 000 | 1 | 5 244 928 | ≈ 5.002 |
| 10 000 | 10 000 | 5 | 26 224 640 | ≈ 25.010 |
| 50 000 | 50 000 | 25 | 131 123 200 | ≈ 125.049 |
| 100 000 | 100 000 | 49 | 257 001 472 | ≈ 245.096 |
| -1 (parsed → 2³² − 1) | 4 294 967 295 | 2 097 152 | 10 999 411 245 056 | ≈ 10.004 TiB |

Derivations:
- `num_segments = ceil(ynum / 2048)`; e.g. `ceil(10000/2048) = ceil(4.8828…) = 5`; `ceil(100000/2048) = ceil(48.828…) = 49`.
- `steady_state_bytes = num_segments × 5 244 928` (per-segment value from the Memory Model section).
- For 100 000: `49 × 5 244 928 = 257 001 472 ≈ 245.096 MiB`. Verify: `49 × 2048 = 100 352 ≥ 100 000` ✓.
- For 10 000: `5 × 5 244 928 = 26 224 640`. Verify: `5 × 2048 = 10 240 ≥ 10 000` ✓.
- For -1: `scrollback_lines(x)` in `kitty/options/utils.py:557-561` parses a negative value to `2**32 - 1 = 4 294 967 295`; `ceil(4 294 967 295 / 2048) = 2 097 152`; `2 097 152 × 5 244 928 = 10 999 411 245 056 bytes ≈ 10.999 TB (decimal) ≈ 10.004 TiB (binary)`.

The last row is "nominal" — no realistic system has ~11 TB of RAM. Because `create_historybuf` only eagerly allocates the **first** segment and `segment_for()` is lazy thereafter, in practice the process RSS grows only as far as the user's actual output volume; the 2³² − 1 figure is a ceiling, not a commitment. Still, the implementation does allow RSS growth without a soft cap until OOM occurs.

### Circular Buffer — the Critical Plateau

Once `num_segments == ceil(ynum/2048)` and every slot is populated, memory stops growing. The key is `historybuf_push()` (`kitty/history.c:275-284`):

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

Observations grounded in the quoted lines:

- `idx = (start_of_data + count) % ynum` — a pure modular index. Line slots are **reused in place**; the backing memory (the segment-allocated `cpu_cells`/`gpu_cells`/`line_attrs` arrays) is never freed.
- When the buffer is full (`count == ynum`), the oldest line is written into the pager history (if enabled) via `pagerhist_push`, and `start_of_data` advances to shift the "oldest" pointer by one.
- **No `realloc`, no `free`, no new allocation occurs on the hot path of the full-buffer push.** The only allocation pathway under heavy output after the buffer has reached `num_segments_max` is pager history extension (next subsection).

This is why heavy output into an already-full scrollback buffer produces **flat** memory: RSS stabilizes, with no allocator activity except for pager-history growth.

### Pager History Behavior

The pager history is an optional second-tier storage that serializes evicted lines as ANSI-annotated bytes into a ring buffer. It is a separate allocation from the segment-backed HistoryBuf, and it has its own growth dynamics.

- **Default is disabled.** `scrollback_pager_history_size` defaults to `0` (`kitty/options/definition.py:406`). When zero, `alloc_pagerhist(0)` returns `NULL` and no ring buffer is allocated.
- **Parsing (non-zero case).** `kitty/options/utils.py:564-566` computes `int(max(0, float(x)) * 1024 * 1024)` (MB-to-bytes) and caps the result at `4096 * 1024 * 1024 - 1 = 4 294 967 295` bytes. So the configured maximum is at most 4 GiB − 1.
- **Initial allocation.** The initial ring-buffer capacity is `MIN(1024u*1024u, pagerhist_sz)` (`kitty/history.c:67`) — at most 1 MiB, regardless of how high the configured maximum is. Requesting 100 MiB still starts with a 1 MiB ring buffer.
- **Extension is copy-on-extend.** When a write does not fit, `pagerhist_extend()` (`kitty/history.c:89-101`) computes `newsz = MIN(ph->maximum_size, buffer_size + MAX(1 MiB, minsz))`, then:
  1. `newbuf = ringbuf_new(newsz)` — new allocation of size `newsz + 1` bytes (`3rdparty/ringbuf/ringbuf.c:56` — `+1` for full/empty disambiguation).
  2. `ringbuf_copy(newbuf, ph->ringbuf, count)` — copies every byte currently in the ring buffer.
  3. `ringbuf_free(&ph->ringbuf)` — frees the old buffer.
  4. `ph->ringbuf = newbuf`.
- **Transient peak during extend.** Between steps 1 and 3, both the old and new ring buffers are resident. For a 4 GiB pager history, the final extend step briefly holds ~8 GiB of pager-history memory.
- **Ceiling.** When `buffer_size >= ph->maximum_size` (`kitty/history.c:92`), `pagerhist_extend()` returns `false`. `pagerhist_write_bytes()` then calls `ringbuf_memcpy_into` (`kitty/history.c:224`) directly, and per ring-buffer semantics the oldest bytes are overwritten (the tail chases the head).
- **What gets written.** `pagerhist_push()` (`kitty/history.c:258-273`) writes three parts per evicted line: a `\x1b[m` SGR reset, the ANSI-formatted line bytes, and either a `\r` (if the line soft-wrapped) or `\r\n` (if it ended naturally). So the pager history stores a replayable ANSI stream, not raw cell data.

### Worst-Case Peak vs Steady-State

At a HistoryBuf segment boundary, two allocator events occur:

1. `self->segments = realloc(self->segments, sizeof(HistoryBufSegment) * self->num_segments)` (`kitty/history.c:20`) — the pointer-array allocation. Each `HistoryBufSegment` is three pointers = 24 bytes on x86-64. For the largest realistic case (`num_segments = 49` at `scrollback_lines=100 000`), this array is only 1 176 bytes — negligible.
2. `calloc(1, 5 244 928)` (at 80 cols) for the new segment body — a single ~5 MiB calloc. Because this size exceeds glibc's default `M_MMAP_THRESHOLD` (128 KiB), the allocation is typically backed by an anonymous `mmap()` rather than a heap extension, and is returned to the OS on free.

In practice, crossing a segment boundary momentarily holds both the old and new segments in RSS (during the realloc of `segments[]` specifically, not of the per-segment data). The per-segment `calloc` is a fresh allocation; there is no old-segment copy to free, so the transient peak is just `previous_steady_state + 5 244 928 bytes`.

Pager-history extends are costlier transients because both the full old ring buffer and the full new ring buffer are held simultaneously during the copy (see Investigative Area 3 for the full boundary table).

---

## Investigative Area 2 — Scroll Responsiveness During Concurrent Output

### Scroll Rendering Path (source-annotated diagram)

```
Scroll Rendering Path (kitty/screen.c, kitty/shaders.c):
  screen_history_scroll(amt, upwards)            screen.c:4090-4118
    ├─> switch(amt): SCROLL_LINE=1, SCROLL_PAGE=lines-1, SCROLL_FULL=historybuf->count
    ├─> new_scroll = MIN(scrolled_by + amt, historybuf->count)   screen.c:4111
    ├─> if (new_scroll != scrolled_by):
    │     scrolled_by = new_scroll
    │     dirty_scroll(self)                     screen.c:4114
    │       ├─> scroll_changed = true            screen.c:1909
    │       └─> screen_pause_rendering(self, false, 0)  screen.c:1910 → 2505
    │             └─> (pause=false branch) clears any active snapshot; is_dirty=true; expires_at=0

  → on next render frame (main thread):
    cell_prepare_to_render()                      shaders.c:393-418
      ├─> if screen->paused_rendering.expires_at:
      │     [use stale snapshot — not triggered by dirty_scroll]
      └─> else if reload_all_gpu_data || scroll_changed || is_dirty || resized || ...:
            update_cell_data { ... }              shaders.c:408-414
              ├─> sz = sizeof(GPUCell) * screen->lines * screen->columns;
              ├─> address = alloc_and_map_vao_buffer(...)
              ├─> screen_update_cell_data(screen, address, fonts_data, ...)   screen.c:2737
              │     ├─> scrolled_by += history_line_added_count              screen.c:2761
              │     ├─> for y in [0, MIN(lines, scrolled_by)):
              │     │      historybuf_init_line(historybuf, scrolled_by-1-y, line)
              │     │      render_line(...); update_line_data(...)          screen.c:2763-2775
              │     └─> for y in [scrolled_by, lines):
              │            linebuf_init_line(linebuf, y - scrolled_by)
              │            if dirty: render_line + screen_render_line_graphics
              │            update_line_data(...)                             screen.c:2776-2788
              └─> unmap_vao_buffer(vao_idx, cell_data_buffer)
```

### I/O Concurrency Path (source-annotated diagram)

```
I/O Concurrency Path (kitty/child-monitor.c, kitty/vt-parser.c):
  I/O Thread:
    io_loop()                                    child-monitor.c:1480-1578
      ├─> poll(fds, n, has_pending_wakeups ? input_delay - elapsed : -1)   child-monitor.c:1506-1513
      ├─> if POLLIN on child fd: read_bytes(fd, screen)                    child-monitor.c:1337-1356
      │     ├─> buf = vt_parser_create_write_buffer(parser, &avail)        vt-parser.c:1450-1462
      │     │     └─> *sz = BUF_SZ - self->write.offset;   (BUF_SZ = 1 MiB)
      │     ├─> read(fd, buf, available_buffer_space)
      │     └─> vt_parser_commit_write(parser, len)
      └─> WAKEUP if (now - last_main_loop_wakeup_at > OPT(input_delay))    child-monitor.c:1562-1570
            ├─> wakeup_main_loop();
            ├─> last_main_loop_wakeup_at = now;
            └─> has_pending_wakeups = false;

  Main Thread:
    parse_input() → do_parse(self, screen, now, flush)                     child-monitor.c:437-448
      ├─> self->parse_func(screen, &pd, flush)
      │     └─> run_worker()                                                vt-parser.c:1416-1446
      │           └─> while consumer available && (flush ||
      │                 time_since_new_input >= input_delay ||
      │                 read.sz + 16*1024 > BUF_SZ):
      │                 consume_input() → screen_scroll() etc.              vt-parser.c:1425
      ├─> if input_read:
      │     set_maximum_wait(OPT(input_delay) - pd.time_since_new_input)   child-monitor.c:445
      └─> render(now, input_read)                                           child-monitor.c:870-896
            ├─> if !input_read && time_since_last_render < repaint_delay:
            │     set_maximum_wait(repaint_delay - time_since_last_render)  child-monitor.c:876
            │     return;                                                    child-monitor.c:877
            └─> for each OSWindow: render_os_window(w, now, ...)
                  └─> render_prepared_os_window(w, ...)
                        └─> cell_prepare_to_render → draw_cells → swap_buffers
```

### Key Properties of Rendering

**O(visible) cost, independent of history depth.** The two loops inside `screen_update_cell_data()` iterate exactly `self->lines` times, split between history-backed visible rows and live-linebuf rows. Quoting the loop splits at `kitty/screen.c:2763-2788`:

- **History half:** `for (index_type y = 0; y < MIN(self->lines, self->scrolled_by); y++) { ... historybuf_init_line(self->historybuf, self->scrolled_by - 1 - y, ...); render_line(...); update_line_data(...); }`
- **Live half:** `for (index_type y = self->scrolled_by; y < self->lines; y++) { linebuf_init_line(self->linebuf, y - self->scrolled_by); ... }`

Combined, these loops touch exactly `self->lines` rows. There is **no loop over `historybuf->count`**. This is the architectural guarantee that scrolling back in a 100 000-line history has the same render cost per frame as a 2 000-line history.

**GPU upload size scales only with the viewport.** `kitty/shaders.c:408-409` defines:

```c
sz = sizeof(GPUCell) * screen->lines * screen->columns;
```

For a 24 × 80 terminal, this is `20 × 24 × 80 = 38 400` bytes per dirty frame. Independent of `historybuf->count`.

**Auto-tracking scroll position.** When new output arrives while the user is scrolled back, `scrolled_by` is automatically advanced to keep the visible content stable. The key line is at `kitty/screen.c:2761` inside `screen_update_cell_data`:

```c
if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + self->history_line_added_count, self->historybuf->count);
```

An equivalent hook exists in `screen_update_only_line_graphics_data` at `kitty/screen.c:2716` for the graphics-only path.

Walk-through: suppose the user scrolls back 100 lines (`scrolled_by = 100`). In the next render tick, the I/O thread has pushed 10 new lines (`history_line_added_count = 10`). Before this hook runs, the 100 visible "history lines" map to history indices `[99, 98, …, 0]` from top to bottom. If `scrolled_by` were not updated, the new lines would push the old content out of view and the user's anchor would drift. With the hook, `scrolled_by` becomes `MIN(100 + 10, historybuf->count) = 110`, and the same content remains visible — lines that were at history indices `[99…0]` are now at indices `[109…10]`, and `scrolled_by-1-y` for `y ∈ [0, 24)` resolves to the correct indices.

`history_line_added_count` is accumulated in `INDEX_UP` (`kitty/screen.c:1559`) and cleared in `screen_reset_dirty()` (`kitty/screen.c:2597-2601`):

```c
static void
screen_reset_dirty(Screen *self) {
    self->is_dirty = false;
    self->history_line_added_count = 0;
}
```

So the counter is **per-frame** (not cumulative) — each render frame sees only the lines added since the last frame, preventing `scrolled_by` from over-advancing.

**Snapshot isolation during pause.** The `screen_pause_rendering()` function (`kitty/screen.c:2505-2544`) implements a pause-and-snapshot mechanism. When `pause=true` is passed (not the scroll-induced case), a fresh `LineBuf` of `lines × columns` is allocated at `kitty/screen.c:2531` and a loop at `kitty/screen.c:2534-2539` copies each visual line via `copy_line(src, dest)`. Subsequent frames render from the snapshot until the pause expires.

Crucially, `dirty_scroll()` at `kitty/screen.c:1907-1911` passes `pause=false`, which clears any existing pause. So a user scroll action **always takes an unpaused render path** — it does not display a stale snapshot. The pause/snapshot mechanism exists for large text pastes and similar flows, not for user scrolling.

**Scroll indicator is constant-time.** `draw_scroll_indicator()` at `kitty/shaders.c:608-630` computes `frac = (float)screen->scrolled_by / (float)screen->historybuf->count;` — an O(1) arithmetic operation. The indicator bar is drawn via `glDrawArrays(GL_TRIANGLE_FAN, 0, 4)` (`kitty/shaders.c:627`) — a single four-vertex triangle fan. No iteration over history is involved.

### I/O Thread Characteristics

**Per-child 1 MiB input buffer.** `kitty/vt-parser.c:18` fixes `BUF_SZ` at `1024u*1024u` bytes. Each `Screen` has one `vt_parser` with one such buffer.

**Near-full early flush.** `run_worker()` (`kitty/vt-parser.c:1416-1446`) flushes the parser's consumer loop when `flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16*1024 > BUF_SZ`. The last term is key for backpressure prevention: when the input buffer is within 16 KiB of full, processing starts immediately, bypassing the `input_delay` coalescing. This guarantees the I/O thread never wedges the pipeline waiting for the delay timer.

**Wake-up coalescing by `input_delay`.** The `WAKEUP` macro in `io_loop` (`kitty/child-monitor.c:1562-1570`) only fires `wakeup_main_loop()` when `now - last_main_loop_wakeup_at > OPT(input_delay)`:

```c
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
if (data_received) {
    if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
    else has_pending_wakeups = true;
} else {
    if (has_pending_wakeups && (now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
}
```

With `input_delay = 3 ms` (`kitty/options/definition.py:878`), wake-ups are at most ~333 Hz. At high output rates where the I/O thread is continuously reading, this batches main-thread work into ~3 ms bundles rather than one wake-up per `read()`.

**Frame rate cap via `repaint_delay`.** `render()` (`kitty/child-monitor.c:870-896`) early-returns if no new input arrived and `time_since_last_render < OPT(repaint_delay)`:

```c
if (!input_read && time_since_last_render < OPT(repaint_delay)) {
    set_maximum_wait(OPT(repaint_delay) - time_since_last_render);
    return;
}
```

With `repaint_delay = 10 ms` (`kitty/options/definition.py:866`), this caps the render rate at ~100 FPS.

### Lag Characterization

Combining the properties above, observable scroll-to-display latency is bounded by a small sum of per-frame budgets — not by history depth. The sources of lag are:

1. **GPU upload bandwidth** for a fixed `20 × lines × columns` byte block per dirty frame. For 24 × 80, this is 38 400 bytes per frame; for 60 × 200, this is 240 000 bytes per frame. At any typical PCIe bandwidth this is sub-millisecond.
2. **Scheduler budget:** one `input_delay` (3 ms) + one `repaint_delay` (10 ms) = 13 ms worst-case from "I/O thread queued a wakeup" → "GPU has swapped buffers for the next frame".
3. **Segment-boundary allocator stalls.** A `calloc(1, 5 244 928)` on the I/O thread during `screen_scroll → INDEX_UP → historybuf_add_line → historybuf_push → segment_for → add_segment` — typically served from `mmap(MAP_ANONYMOUS)` and fast, but could stall briefly on a heavily loaded kernel.

**None of these sources scale with `historybuf->count`.** The user's perceived scroll responsiveness is therefore independent of how much scrollback has accumulated.

---

## Investigative Area 3 — Buffer Boundary Transitions

### Lazy Trigger

The lazy allocation of HistoryBuf segments is implemented by `segment_for()` at `kitty/history.c:36-42`:

```c
static index_type
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
    if (UNLIKELY(seg_num >= self->num_segments)) fatal("Out of bounds access to history buffer line number: %u", y);
    return seg_num;
}
```

Note the return type is `index_type` (an unsigned integer, the segment number) — not `HistoryBufSegment*`. Dereferencing to the actual `HistoryBufSegment` is performed by the `seg_ptr` macro at `kitty/history.c:44-48`, which consumes `segment_for`'s returned `seg_num` to index into `self->segments[]`:

```c
#define seg_ptr(which, stride) { \
    index_type seg_num = segment_for(self, y); \
    y -= seg_num * SEGMENT_SIZE; \
    return self->segments[seg_num].which + y * stride; \
}
```

Three guards apply in `segment_for`:

- `seg_num >= self->num_segments` — only add segments if the requested segment does not yet exist.
- `SEGMENT_SIZE * self->num_segments < self->ynum` — never over-allocate beyond `ynum`. When the ceiling is reached, this loop exits and the next call to `segment_for` simply returns the existing `seg_num`.
- `if (UNLIKELY(seg_num >= self->num_segments)) fatal(...)` (line 40) — a belt-and-braces bounds check: if the `while` loop exited without reaching the requested segment (because the `ynum` ceiling was hit first), the process aborts rather than returning an invalid index.

The `UNLIKELY(...)` macros are compiler branch-prediction hints (`__builtin_expect(..., 0)` on GCC/Clang) indicating the allocator-trigger and fatal paths are cold; the hot path is "segment already exists, return immediately."

Note the `while` rather than `if`: if `segment_for(y)` is invoked with a `y` several segments beyond the current frontier (e.g., after a `historybuf_rewrap` or non-sequential init), the loop adds as many segments as needed in one call. In the normal push hot path, the frontier advances one line at a time, so at most one `add_segment` happens per push.

### `add_segment()` — the Allocation Sequence

Full sequence at `kitty/history.c:17-29`:

```c
static void
add_segment(HistoryBuf *self) {
    self->num_segments += 1;
    self->segments = realloc(self->segments, sizeof(HistoryBufSegment) * self->num_segments);
    if (self->segments == NULL) fatal("Out of memory allocating new history buffer segment");
    HistoryBufSegment *s = self->segments + self->num_segments - 1;
    const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
    const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
    s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));
    if (!s->cpu_cells) fatal("Out of memory allocating new history buffer segment");
    s->gpu_cells = (GPUCell*)(((uint8_t*)s->cpu_cells) + cpu_cells_size);
    s->line_attrs = (LineAttrs*)(((uint8_t*)s->gpu_cells) + gpu_cells_size);
}
```

Step-by-step:

1. `num_segments += 1` (line 19).
2. `segments = realloc(segments, 24 × num_segments)` (line 20) — three pointers of 8 bytes each on x86-64 = 24 bytes per entry.
3. Fatal on OOM for the segments-pointer realloc (line 21) — the process terminates, there is no graceful back-off. Note that both fatal messages in `add_segment` are textually identical (`"Out of memory allocating new history buffer segment"`); the two call sites differ only in which allocation failed (the pointer-array realloc at line 20 vs the per-segment `calloc` at line 25).
4. `calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs))` (line 25) — one call, one allocation. At 80 cols this is 5 244 928 bytes.
5. Fatal on OOM for the per-segment calloc (line 26) — same semantics, same fatal message.
6. Pointer arithmetic (lines 27–28) partitions the single allocation into `cpu_cells` | `gpu_cells` | `line_attrs` sub-regions.

Because `calloc` zero-fills, the allocation is memory-safe against uninitialized reads but incurs an O(segment_bytes) memset on each new segment (5 244 928 byte zero-fill at 80 cols).

### Boundary Timeline (80 columns)

The following table shows when each segment allocation occurs as lines are pushed into an initially empty HistoryBuf with large `ynum` (so no ceiling is reached). The "pushed" column counts logical pushes; the "seg #" column shows the segment allocated (where seg 0 is pre-allocated eagerly by `create_historybuf()` at `kitty/history.c:127`).

| After N lines pushed | Triggered segment # | `num_segments` | Cumulative bytes | Cumulative MiB |
|----------------------|---------------------|----------------|------------------|----------------|
| 0 (Screen creation) | seg 0 (eager) | 1 | 5 244 928 | 5.002 |
| 2 048 | — (still filling seg 0) | 1 | 5 244 928 | 5.002 |
| 2 049 | seg 1 (lazy) | 2 | 10 489 856 | 10.003 |
| 4 097 | seg 2 (lazy) | 3 | 15 734 784 | 15.005 |
| 6 145 | seg 3 (lazy) | 4 | 20 979 712 | 20.006 |
| 8 193 | seg 4 (lazy) | 5 | 26 224 640 | 25.008 |

For `scrollback_lines = 10 000`, `ynum = 10 000`, so the loop guard `SEGMENT_SIZE * num_segments < ynum` stops at `num_segments = 5` (since `5 × 2048 = 10 240 ≥ 10 000`). Lines 8 192 through 10 239 reside in segment 4 even though only 10 000 are logically valid (the remainder stay zeroed until overwritten).

For `scrollback_lines = 2 000` with 24 visible lines, `ynum = MAX(2000, 24) = 2000`; `ceil(2000/2048) = 1` segment, so only the eagerly allocated seg 0 is ever used. A 3 000-line burst into such a buffer would fill seg 0, overwrite from the start, and never allocate seg 1 — the ceiling check in `segment_for` never fires because `num_segments = 1` already satisfies `1 × 2048 = 2048 ≥ 2000`. To observe a segment boundary experimentally, configure `scrollback_lines ≥ 2049` (e.g., 4000) and stream > 2048 lines.

### Pager History Boundaries

Separate from the HistoryBuf segment dynamics, the pager history ring buffer has its own growth pattern:

- **First allocation.** If `scrollback_pager_history_size > 0`, `alloc_pagerhist(pagerhist_sz)` (`kitty/history.c:70-80`) creates a `PagerHistoryBuf` struct and calls `ringbuf_new(initial_pagerhist_ringbuf_sz(pagerhist_sz))` — the initial capacity is `MIN(1 MiB, pagerhist_sz)` per `kitty/history.c:67`. `ringbuf_new(capacity)` internally `malloc`s `capacity + 1` bytes for the data buffer (`3rdparty/ringbuf/ringbuf.c:56`) — the `+1` byte is reserved for the full-versus-empty distinction since the ring buffer uses the head-equals-tail-means-empty convention. The `struct ringbuf_t` bookkeeping itself is a separate, small allocation.
- **First write.** A line is evicted from the HistoryBuf (buffer full at `kitty/history.c:279`) → `pagerhist_push()` (line 280) → `pagerhist_write_bytes()` (`kitty/history.c:218-226`). If `sz > ringbuf_bytes_free(ph->ringbuf)`, the write path calls `pagerhist_extend(ph, sz)` before `ringbuf_memcpy_into`.
- **Extend step.** `pagerhist_extend()` at `kitty/history.c:89-101`:
  - `newsz = MIN(ph->maximum_size, buffer_size + MAX(1024u*1024u, minsz))` — so the minimum growth increment is 1 MiB (larger only if a single write exceeds 1 MiB, which is unusual for a terminal line).
  - `newbuf = ringbuf_new(newsz)` — a fresh allocation.
  - `ringbuf_copy(newbuf, ph->ringbuf, count)` — copies all existing bytes (up to `maximum_size` on the final extend).
  - `ringbuf_free(&ph->ringbuf)` — frees the old.
  - `ph->ringbuf = newbuf`.
- **Ceiling and overwrite.** When `buffer_size >= ph->maximum_size` (`kitty/history.c:92`), `pagerhist_extend` returns `false`. `pagerhist_write_bytes` then proceeds to `ringbuf_memcpy_into` unconditionally; per ring-buffer semantics, writes beyond the head position advance the tail (the oldest bytes are lost).

### Observable Boundary Signatures

Summary of what an external monitor attached to the kitty process would observe:

- **HistoryBuf segment allocations.** A step-function increase in RSS of 5 244 928 bytes (at 80 cols) every ~2048 new lines pushed into main-screen scrollback, until `num_segments = ceil(ynum / 2048)` is reached. Thereafter, no further HistoryBuf growth — RSS from HistoryBuf plateaus permanently.
- **Pager-history extends.** A 1 MiB step in RSS on the first pager-history overflow, another 1 MiB step when the next overflow hits, and so on, until `ph->maximum_size` is reached. Each extend is preceded by a brief spike equal to the current ring-buffer capacity (because old + new coexist during the copy), then the old allocation is returned to the allocator.
- **Final plateau.** Steady-state memory is `ceil(ynum / 2048) × segment_bytes` (HistoryBuf) + `ph->maximum_size + sizeof(ringbuf_t)` (pager history) + per-frame LineBuf and auxiliary structures. Once reached, heavy output causes no further allocator activity on the scrollback hot path.

---

## Observability Methods (Non-Invasive)

These techniques allow an engineer to externally observe the allocation behaviors described above **without modifying the kitty repository**. Temporary shell or Python scripts that live outside the kitty source tree are acceptable; edits to kitty's C, Python, or Go sources are not.

### 1. `/proc/<pid>/status` polling (Linux)

Poll the `VmRSS`, `VmSize`, and `VmPeak` fields at regular intervals while the subject process produces output. A single-file helper script like the following suffices:

```sh
#!/bin/sh
# sample_rss.sh <pid>  (save outside the kitty repo)
while kill -0 "$1" 2>/dev/null; do
    awk '/VmRSS|VmSize|VmPeak/' /proc/"$1"/status
    echo '---'
    sleep 0.1
done
```

Expected signature: at 80 columns with `scrollback_lines=50000`, VmRSS advances in ~5 MiB steps each time ~2048 lines are pushed into main-screen history. After 25 such steps, VmRSS stops advancing regardless of further output.

### 2. `/proc/<pid>/smaps`

Large anonymous mappings correspond to HistoryBuf segments. Each `mmap`-backed segment at 80 cols is ~5 246 976 bytes (rounded up from 5 244 928 for allocator bookkeeping and alignment — glibc typically uses `M_MMAP_THRESHOLD = 128 KiB`, so any calloc over 128 KiB is served by `mmap(MAP_ANONYMOUS)`). Filtering `/proc/<pid>/smaps` for anonymous regions of ~5 MiB reveals the segment count directly.

### 3. `mallinfo2()` / `malloc_stats()`

On glibc, an external `gdb` or `pyrasite`-style sidecar attached to kitty can call `malloc_stats()` (which prints arena statistics to stderr) without changing kitty code. This reveals the mix of `mmap`-backed vs heap-backed allocations. Alternatively, `mallinfo2()` returns structured counters. Both are read-only observability — the attached process merely invokes existing libc functions.

### 4. `strace` on `mmap`/`brk`/`munmap`

Run kitty under `strace -e trace=mmap,mmap2,munmap,brk -p <pid>` and correlate `mmap(NULL, 5246976, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` calls with segment boundaries. Each new HistoryBuf segment at 80 cols produces exactly one such `mmap` of ~5 MiB.

### 5. `bpftrace` / eBPF uprobes

If debug symbols are present on the kitty binary, attach uprobes:

```
bpftrace -e 'uprobe:/path/to/kitty:add_segment { printf("add_segment t=%llu\n", nsecs); }'
bpftrace -e 'uprobe:/path/to/kitty:pagerhist_extend { printf("extend t=%llu arg2=%lu\n", nsecs, arg2); }'
```

These produce a timeline of every segment and pager-history growth event with nanosecond precision. Again, no kitty source change.

### 6. Reproduction Workload

To force HistoryBuf segment boundaries in a controlled way:

- Configure `scrollback_lines=4000` (so `ceil(4000/2048)=2` segments may be allocated).
- Start kitty, record the baseline VmRSS.
- Stream more than 2048 lines on the main screen (e.g., `yes | head -n 2100`) and observe a ~5 MiB step.
- Stream further to force full scrollback and observe memory plateau.

To force pager-history extends:

- Configure `scrollback_lines=2000` (small history) and `scrollback_pager_history_size=10` (10 MB pager history).
- Stream enough output that lines get evicted into the pager history (> 2000 lines), then continue to accumulate until the initial 1 MiB pager ring buffer overflows.
- Observe ~1 MiB VmRSS steps until the 10 MiB ceiling is reached.

### 7. Test Suite as Ground Truth

The in-repo tests validate the behaviors described above without requiring any new code:

- `kitty_tests/datatypes.py:501-510` constructs `HistoryBuf(3000, 5)` and pushes 3000 lines, verifying LIFO order. With `ynum=3000`, the guard in `segment_for()` permits up to `ceil(3000/2048) = 2` segments — so this test crosses the seg 0 → seg 1 boundary at push 2049. It is the most concise in-tree demonstration of the boundary behavior.
- `kitty_tests/screen.py:695-740` is the authoritative proof of pager-history extend and overwrite semantics. It constructs a Screen with `scrollback_pager_history_size` set to a very small value (via the `hsz=8` parameter), forces overflow, and asserts that the oldest bytes are trimmed as expected.
- `kitty_tests/__init__.py:184-189` provides `filled_history_buf()` — the test-infrastructure helper that populates a HistoryBuf for any test. `BaseTest.set_options` (line 224) sets `scrollback_pager_history_size=1024` by default for tests that need pager history active.

None of these methods require modifying kitty source files. Any observer script or debugger attachment is external to the repository.

---

## Key Findings & Conclusions

- **Segment size is exactly 2048 lines.** `#define SEGMENT_SIZE 2048` at `kitty/history.c:15` is the granularity of every HistoryBuf memory event.
- **Per-line cost at 80 columns is 2 561 bytes** (2 560 cell bytes + 1 LineAttrs byte; `sizeof(GPUCell)=20`, `sizeof(CPUCell)=12` per `kitty/data-types.h:221, 228`; `sizeof(LineAttrs)=1` per the union over `uint8_t val` at `kitty/data-types.h:231-239`). Per-segment at 80 cols is exactly **5 244 928 bytes (≈ 5.002 MiB)** from the `history.c:23-25` formula.
- **The first segment is allocated eagerly at Screen creation** (`kitty/history.c:127` inside `create_historybuf`). So the minimum HistoryBuf memory immediately after opening a kitty window is one segment (~5 MiB at 80 cols), not zero — the "lazy allocation" description applies only to segments 1 through N−1.
- **HistoryBuf memory never shrinks once allocated.** `historybuf_push()` (`kitty/history.c:275-284`) reuses slots via `idx = (start_of_data + count) % ynum` and, when full, overwrites in place by advancing `start_of_data`. No `free()` occurs on the hot path.
- **`alloc_historybuf` uses `MAX(scrollback_lines, screen_lines)`** (`kitty/screen.c:130`). Even `scrollback_lines=0` still produces a HistoryBuf of `ynum = screen_lines`, so every Screen has at least `screen_lines` worth of scrollback infrastructure.
- **Scroll responsiveness is O(screen→lines × screen→columns) per frame** (`kitty/screen.c:2763-2788`, `kitty/shaders.c:409`), independent of `historybuf->count`. The GPU upload of the dirty frame is exactly `20 × lines × columns` bytes.
- **`scrolled_by` is auto-adjusted each frame** by `history_line_added_count` (`kitty/screen.c:2761`, mirrored at `kitty/screen.c:2716`). The counter is incremented in `INDEX_UP` (`kitty/screen.c:1559`) and cleared in `screen_reset_dirty()` (`kitty/screen.c:2597-2601`) — ensuring per-frame, non-cumulative updates.
- **`dirty_scroll()` clears any active pause snapshot** because it invokes `screen_pause_rendering(self, false, 0)` (`kitty/screen.c:1910`). User scroll actions therefore render fresh content, not a stale snapshot — confirming that the snapshot path is for large-paste and animation coalescing, not for user scrolling.
- **I/O wake-ups are batched by `input_delay`** (default 3 ms, `kitty/options/definition.py:878`) via the `WAKEUP` macro at `kitty/child-monitor.c:1562-1570`. Rendering is capped by `repaint_delay` (default 10 ms, `kitty/options/definition.py:866`) via the early-return at `kitty/child-monitor.c:875-877`.
- **`run_worker` bypasses `input_delay` when the input buffer is near-full.** The flush condition `read.sz + 16*1024 > BUF_SZ` at `kitty/vt-parser.c:1425` guarantees the input pipeline never back-pressures the I/O thread for long — critical for scroll responsiveness under sustained heavy output.
- **`scrollback_lines=-1` parses to 2³² − 1 = 4 294 967 295** (`kitty/options/utils.py:557-561`). The resulting ~10.999 TB (decimal) / ~10.004 TiB (binary) ceiling is nominal only — segments are allocated lazily, so actual memory is bounded by output volume and OS limits rather than by this cap.
- **Pager history starts at `MIN(1 MiB, configured_max)`** (`kitty/history.c:67`) regardless of the configured ceiling up to 4 GiB − 1 (`kitty/options/utils.py:564-566`). It grows by 1 MiB increments via the copy-on-extend path at `kitty/history.c:89-101`, producing a brief memory peak equal to `old_capacity + new_capacity` during each extend.
- **HistoryBuf uses one `calloc` per segment; LineBuf uses five `PyMem_Calloc` calls.** Compare `kitty/history.c:23-28` (single allocation for cpu_cells + gpu_cells + line_attrs) with `kitty/line-buf.c:90-98` (separate buffers for `cpu_cell_buf`, `gpu_cell_buf`, `line_map`, `scratch`, `line_attrs`). This is a deliberate density-vs-flexibility trade-off: history is read-mostly (density wins), the visible LineBuf needs auxiliary maps for scroll-region operations (flexibility wins).
- **Scrollback indicator position is O(1).** `frac = scrolled_by / historybuf->count` at `kitty/shaders.c:616`, controlled by the `scrollback_indicator_opacity` option. Drawing the bar is a single 4-vertex triangle fan with no iteration over history.
- **Observable allocation boundaries are predictable.** Every 2048 newly captured main-screen lines produces one HistoryBuf segment allocation (~5 MiB at 80 cols); every 1 MiB of pager-history demand produces one ring-buffer extend. Both can be observed externally via `/proc/<pid>/status`, `strace` on `mmap`, or eBPF uprobes — **no kitty source modification is required** to measure either boundary.

---

## Citations Index

| File | Lines Cited |
|------|-------------|
| `kitty/history.c` | 15 (SEGMENT_SIZE); 17–29 (add_segment); 20 (realloc segments); 23–28 (calloc formula); 31–34 (free_segment); 36–42 (segment_for); 44–48 (seg_ptr macro); 67 (initial_pagerhist_ringbuf_sz); 70–80 (alloc_pagerhist); 89–101 (pagerhist_extend); 92 (ceiling check); 117–133 (create_historybuf); 127 (eager add_segment); 218–226 (pagerhist_write_bytes); 224 (ringbuf_memcpy_into); 258–273 (pagerhist_push); 275–284 (historybuf_push); 279 (full-check before pagerhist_push); 280 (pagerhist_push call); 281 (start_of_data advance); 577–579 (alloc_historybuf) |
| `kitty/data-types.h` | 221 (sizeof(GPUCell)==20); 228 (sizeof(CPUCell)==12); 231–239 (LineAttrs union); 262–266 (HistoryBufSegment); 268–272 (PagerHistoryBuf); 282–290 (HistoryBuf) |
| `kitty/screen.c` | 130 (alloc_historybuf wiring); 1552–1567 (INDEX_UP); 1558 (historybuf_add_line); 1559 (history_line_added_count++); 1570–1577 (screen_index / add_to_history); 1574 (main_linebuf + margin_top==0); 1589–1598 (screen_scroll); 1907–1911 (dirty_scroll); 1909 (scroll_changed=true); 1910 (pause_rendering(false)); 1913–1920 (screen_clear_scrollback); 2505–2544 (screen_pause_rendering); 2531 (snapshot LineBuf); 2534–2539 (copy_line loop); 2597–2601 (screen_reset_dirty); 2713–2735 (screen_update_only_line_graphics_data); 2716 (graphics-path scrolled_by hook); 2737–2797 (screen_update_cell_data); 2761 (main scrolled_by auto-adjust); 2763–2775 (history-half render loop); 2776–2788 (live-half render loop); 2842–2853 (visual_line_); 4090–4118 (screen_history_scroll); 4111 (new_scroll MIN); 4114 (dirty_scroll call) |
| `kitty/screen.h` | 88–115 (Screen struct: scrolled_by, is_dirty, scroll_changed, history_line_added_count, historybuf) |
| `kitty/shaders.c` | 393–418 (cell_prepare_to_render); 408–409 (update_cell_data macro / sz formula); 608–630 (draw_scroll_indicator); 616 (frac = scrolled_by / historybuf->count); 627 (glDrawArrays 4-vertex) |
| `kitty/child-monitor.c` | 437–448 (do_parse); 445 (set_maximum_wait input_delay); 870–896 (render); 874–877 (repaint_delay gate); 1337–1356 (read_bytes); 1480–1578 (io_loop); 1506–1513 (poll with timeout); 1562–1570 (WAKEUP / input_delay coalesce) |
| `kitty/vt-parser.c` | 18 (BUF_SZ = 1 MiB); 21 (MAX_ESCAPE_CODE_LENGTH = BUF_SZ/4); 1416–1446 (run_worker); 1425 (flush condition); 1450–1462 (vt_parser_create_write_buffer) |
| `kitty/line-buf.c` | 85–120 (alloc_linebuf); 90–98 (five PyMem_Calloc calls) |
| `kitty/options/definition.py` | 372 (scrollback_lines default 2000); 406 (scrollback_pager_history_size default 0); 866 (repaint_delay default 10); 878 (input_delay default 3) |
| `kitty/options/utils.py` | 557–561 (scrollback_lines parser → 2³²−1); 564–566 (scrollback_pager_history_size MB-to-bytes, 4 GiB−1 cap) |
| `3rdparty/ringbuf/ringbuf.h` | 40–41 (ringbuf_new); 52 (ringbuf_buffer_size); 72–73 (ringbuf_capacity); 79–80 (ringbuf_bytes_free); 86–87 (ringbuf_bytes_used) |
| `3rdparty/ringbuf/ringbuf.c` | 42–47 (struct ringbuf_t); 55–57 (malloc capacity+1); 92 (capacity = buffer_size − 1); public-domain implementation by Drew Hess, 2011, see `3rdparty/ringbuf/ringbuf.c` header comment |
| `kitty_tests/datatypes.py` | 487 (test_historybuf); 501–510 (3000-line stress test) |
| `kitty_tests/screen.py` | 695–740 (test_pagerhist); 695 (hsz=8 construction); 728–733 (pagerhist_rewrap); 735–741 (multi-byte overflow) |
| `kitty_tests/__init__.py` | 184–189 (filled_history_buf); 224 (default scrollback_pager_history_size=1024); 237–241 (create_screen) |
