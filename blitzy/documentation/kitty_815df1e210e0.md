# HistoryBuf Scrollback Buffer Under Extreme-Throughput Stress: A Code-Level Investigation

> **Kitty Terminal Emulator v0.35.2** · Branch `kitty_815df1e210e0` · Analysis derived entirely from source code inspection

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [HistoryBuf Data Model](#2-historybuf-data-model)
   - 2.1 [HistoryBuf and Its Segments](#21-historybuf-and-its-segments)
   - 2.2 [PagerHistoryBuf and the Ring Buffer](#22-pagerhistorybuf-and-the-ring-buffer)
   - 2.3 [Configuration Parameters](#23-configuration-parameters)
3. [Filling and Overflow Under Flood](#3-filling-and-overflow-under-flood)
   - 3.1 [The Data Path: PTY to Scrollback](#31-the-data-path-pty-to-scrollback)
   - 3.2 [Phase A — Filling (count < ynum)](#32-phase-a--filling-count--ynum)
   - 3.3 [Phase B — Full and Wrapping (count == ynum)](#33-phase-b--full-and-wrapping-count--ynum)
4. [Segment Allocation Dynamics](#4-segment-allocation-dynamics)
   - 4.1 [Lazy Allocation via segment_for()](#41-lazy-allocation-via-segment_for)
   - 4.2 [The add_segment() Memory Layout](#42-the-add_segment-memory-layout)
   - 4.3 [Allocation Counts for Common Configurations](#43-allocation-counts-for-common-configurations)
5. [Pager History Ring Buffer Interaction](#5-pager-history-ring-buffer-interaction)
   - 5.1 [The Spillover Mechanism: pagerhist_push()](#51-the-spillover-mechanism-pagerhist_push)
   - 5.2 [Ring Buffer Growth: pagerhist_extend()](#52-ring-buffer-growth-pagerhist_extend)
   - 5.3 [Ring Buffer Overflow Semantics](#53-ring-buffer-overflow-semantics)
   - 5.4 [Flood Behavior Summary](#54-flood-behavior-summary)
6. [Scrolling While Data Arrives](#6-scrolling-while-data-arrives)
   - 6.1 [The scrolled_by Tracking Mechanism](#61-the-scrolled_by-tracking-mechanism)
   - 6.2 [Line Resolution via visual_line_()](#62-line-resolution-via-visual_line_)
   - 6.3 [Threading Model and Safety](#63-threading-model-and-safety)
   - 6.4 [Behavioral Analysis Under Stress](#64-behavioral-analysis-under-stress)
7. [Memory Structure Evolution Under Pressure](#7-memory-structure-evolution-under-pressure)
   - 7.1 [Phase 1 — Initial Fill](#71-phase-1--initial-fill)
   - 7.2 [Phase 2 — Steady-State Wrap with Pager Growth](#72-phase-2--steady-state-wrap-with-pager-growth)
   - 7.3 [Phase 3 — True Steady State](#73-phase-3--true-steady-state)
   - 7.4 [Circular Addressing Arithmetic](#74-circular-addressing-arithmetic)
8. [Observation Approaches](#8-observation-approaches)
9. [Conclusion](#9-conclusion)

---

## 1. Introduction

This investigation examines how the Kitty terminal emulator's scrollback buffer subsystem behaves under extreme-throughput stress conditions — specifically, what happens when an enormous volume of text arrives in a short period of time. The analysis covers four interrelated concerns:

1. **Segment allocation under flood**: How does `HistoryBuf` fill, stretch, and carve out new segments when the buffer goes from empty to full under sustained pressure?

2. **Segmented scrollback ↔ pager ring buffer interaction**: When the main scrollback is full and lines are evicted, how does the `PagerHistoryBuf` ring buffer receive and manage the overflow? Is the transition smooth or does it introduce observable hesitation?

3. **Active scrolling during output flood**: What happens when a user is actively scrolled back while new data is arriving at full speed? Does the concurrent access pattern hold up?

4. **Memory structure evolution at runtime**: How do the underlying memory structures — segment allocation, ring buffer resizing, and circular indexing — evolve as pressure builds?

Every claim in this document is derived from direct inspection of the Kitty 0.35.2 source code. File paths and line numbers reference the exact locations in the codebase where the described behavior is implemented.

---

## 2. HistoryBuf Data Model

### 2.1 HistoryBuf and Its Segments

The scrollback buffer is a **segmented circular buffer** composed of three cooperating C structs defined in `kitty/data-types.h`.

**HistoryBufSegment** (lines 262–266):

```c
typedef struct {
    GPUCell *gpu_cells;
    CPUCell *cpu_cells;
    LineAttrs *line_attrs;
} HistoryBufSegment;
```

Each segment holds storage for exactly `SEGMENT_SIZE` (2048) lines. The three pointers (`gpu_cells`, `cpu_cells`, `line_attrs`) all point into a single contiguous `calloc` block — the GPU and attrs pointers are computed as offsets from the CPU cells pointer, which is the base of the allocation. This layout ensures cache locality for sequential line access within a segment.

**HistoryBuf** (lines 282–290):

```c
typedef struct {
    PyObject_HEAD

    index_type xnum, ynum, num_segments;
    HistoryBufSegment *segments;
    PagerHistoryBuf *pagerhist;
    Line *line;
    index_type start_of_data, count;
} HistoryBuf;
```

Key fields:

| Field | Meaning |
|-------|---------|
| `xnum` | Number of columns (terminal width) |
| `ynum` | Maximum number of lines the buffer can hold (configured via `scrollback_lines`) |
| `num_segments` | Current number of allocated segments |
| `segments` | Dynamic array of `HistoryBufSegment` structs |
| `pagerhist` | Pointer to optional pager ring buffer (NULL when disabled) |
| `line` | Reusable `Line` object for initializing line pointers |
| `start_of_data` | Index (buffer position) of the **oldest** line in the circular buffer |
| `count` | Number of lines currently stored (invariant: `0 ≤ count ≤ ynum`) |

The circular buffer discipline works as follows: new lines are written at position `(start_of_data + count) % ynum`. When the buffer is full (`count == ynum`), pushing a new line first serializes the oldest line (at `start_of_data`) into the pager ring buffer, then advances `start_of_data` modularly. The slot at the computed index is reused for the incoming line.

### 2.2 PagerHistoryBuf and the Ring Buffer

**PagerHistoryBuf** (`kitty/data-types.h`, lines 268–272):

```c
typedef struct {
    void *ringbuf;
    size_t maximum_size;
    bool rewrap_needed;
} PagerHistoryBuf;
```

This struct wraps a byte-level ring buffer (FIFO) from `3rdparty/ringbuf/`. The `ringbuf` field is an opaque handle to the ring buffer, `maximum_size` is the configured upper bound on the ring buffer's capacity, and `rewrap_needed` signals whether the pager content must be reflowed for a new terminal width.

**ringbuf_t** (`3rdparty/ringbuf/ringbuf.c`, lines 42–47):

```c
struct ringbuf_t {
    uint8_t *buf;
    uint8_t *head, *tail;
    size_t size;
};
```

The ring buffer stores raw bytes (ANSI-encoded text) using `head` and `tail` pointers into a flat `buf` array. One sentinel byte is reserved: `size = capacity + 1` (confirmed by `ringbuf_new()` at line 56: `rb->size = capacity + 1`). This sentinel distinguishes the full state from the empty state without requiring a separate flag.

- `head` points to the next write location (data is copied *into* the buffer at `head`).
- `tail` points to the next read location (data is copied *from* the buffer at `tail`).
- When `head == tail`, the buffer is empty.
- When `ringbuf_nextp(rb, head) == tail`, the buffer is full.

### 2.3 Configuration Parameters

Three user-configurable options control scrollback behavior. These are defined in `kitty/options/definition.py` (lines 372–423):

| Option | Default | Description |
|--------|---------|-------------|
| `scrollback_lines` | 2000 | Maximum number of lines in `HistoryBuf`. Negative values mean effectively infinite scrollback. Memory is allocated on demand. *(line 372)* |
| `scrollback_pager_history_size` | 0 (disabled) | Size in MB of the pager ring buffer. Maximum 4 GB. When 0, `pagerhist` is NULL and no pager history is maintained. *(line 406)* |
| `scrollback_fill_enlarged_window` | `no` | Whether to fill new space with scrollback lines after window enlargement. *(line 420)* |

The Screen constructor (`kitty/screen.c`, line 130) allocates the history buffer with:

```c
self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));
```

This ensures `ynum` is at least as large as the visible screen height.

---

## 3. Filling and Overflow Under Flood

### 3.1 The Data Path: PTY to Scrollback

When a child process emits output, the data follows this precise code path from PTY to scrollback storage:

```
Child PTY Output
    │
    ▼
VT Parser (dispatches to screen operations)
    │
    ▼
screen_index()                          [kitty/screen.c:1569-1577]
    │  (when cursor is at bottom margin, main screen, margin_top == 0)
    ▼
INDEX_UP macro                          [kitty/screen.c:1552-1567]
    ├── linebuf_index()                 (shifts line map in screen buffer)
    ├── historybuf_add_line()            [kitty/history.c:286-291]
    │       └── historybuf_push()        [kitty/history.c:275-284]
    │               ├── [count < ynum]: count++, lazy segment alloc if needed
    │               └── [count == ynum]: pagerhist_push() → start_of_data++
    ├── history_line_added_count++       (scroll tracking accumulator)
    └── linebuf_clear_line()            (clears the bottom screen line)
```

**`screen_index()`** (`kitty/screen.c`, lines 1569–1577):

```c
void
screen_index(Screen *self) {
    unsigned int top = self->margin_top, bottom = self->margin_bottom;
    if (self->cursor->y == bottom) {
        const bool add_to_history = self->linebuf == self->main_linebuf && self->margin_top == 0;
        INDEX_UP(add_to_history);
    } else screen_cursor_down(self, 1);
}
```

The guard `self->linebuf == self->main_linebuf && self->margin_top == 0` ensures history is only populated from the main screen buffer when no scroll region top margin is set. The alternate screen buffer (used by full-screen applications like `vim`) does not add to history.

**`INDEX_UP` macro** (`kitty/screen.c`, lines 1552–1567):

```c
#define INDEX_UP(add_to_history) \
    linebuf_index(self->linebuf, top, bottom); \
    INDEX_GRAPHICS(-1) \
    if (add_to_history) { \
        linebuf_init_line(self->linebuf, bottom); \
        historybuf_add_line(self->historybuf, self->linebuf->line, &self->as_ansi_buf); \
        self->history_line_added_count++; \
        if (self->last_visited_prompt.is_set) { \
            if (self->last_visited_prompt.scrolled_by < self->historybuf->count) \
                self->last_visited_prompt.scrolled_by++; \
            else self->last_visited_prompt.is_set = false; \
        } \
    } \
    linebuf_clear_line(self->linebuf, bottom, true); \
    self->is_dirty = true; \
    index_selection(self, &self->selections, true);
```

The critical sequence is:
1. `linebuf_index()` shifts the line map in the screen buffer (conceptually scrolling up).
2. The scrolled-off line (at `bottom`) is initialized and passed to `historybuf_add_line()`.
3. `history_line_added_count` is incremented — this accumulator is consumed once per render frame to adjust the scroll position.
4. The bottom line is cleared for new content.

**`historybuf_add_line()`** (`kitty/history.c`, lines 286–291):

```c
void
historybuf_add_line(HistoryBuf *self, const Line *line, ANSIBuf *as_ansi_buf) {
    index_type idx = historybuf_push(self, as_ansi_buf);
    copy_line(line, self->line);
    *attrptr(self, idx) = line->attrs;
}
```

This function obtains a slot in the circular buffer via `historybuf_push()`, then copies the line's cell data and attributes into that slot.

**`historybuf_push()`** (`kitty/history.c`, lines 275–284):

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

This is the core circular buffer logic. It branches into two distinct operational phases:

### 3.2 Phase A — Filling (count < ynum)

When the buffer is not yet full:

- `idx = (start_of_data + count) % ynum` computes the next write position. Since `start_of_data` is 0 during filling, this simplifies to `idx = count`.
- `init_line(self, idx, self->line)` sets up the line's cell pointers to the storage at position `idx`. This call may trigger **lazy segment allocation** if the computed index falls in a segment that hasn't been allocated yet (see [Section 4](#4-segment-allocation-dynamics)).
- `count` is incremented.
- `pagerhist_push()` is NOT called — no data is evicted.

**Under flood**: Phase A processes `ynum` lines (e.g., 2000 for the default configuration). Each line push is O(1) aside from the occasional segment allocation, which occurs at most `⌈ynum / 2048⌉ - 1` times (since the first segment is allocated at construction). For the default `scrollback_lines=2000`, no additional segments are needed — the single initial segment (capacity 2048) is sufficient.

### 3.3 Phase B — Full and Wrapping (count == ynum)

Once the buffer reaches capacity:

- `idx = (start_of_data + count) % ynum` computes a position that is equal to `start_of_data` — the oldest line in the buffer.
- `pagerhist_push()` serializes the oldest line (at `start_of_data`) as ANSI text into the pager ring buffer before it is overwritten.
- `start_of_data = (start_of_data + 1) % ynum` advances the circular start pointer.
- `count` remains unchanged at `ynum`.

**Under sustained flood, once in Phase B**:
- Every incoming line triggers `pagerhist_push()` followed by a circular overwrite.
- **No HistoryBuf memory allocation occurs** — the buffer operates as a fixed-size ring.
- The pager ring buffer may grow (see [Section 5](#5-pager-history-ring-buffer-interaction)), but HistoryBuf itself is allocation-free.
- The per-line cost is dominated by the ANSI serialization in `pagerhist_push()`, which is proportional to the line width (`xnum`).

---

## 4. Segment Allocation Dynamics

### 4.1 Lazy Allocation via segment_for()

Segments are not pre-allocated for the entire `ynum` capacity. Instead, they are allocated on demand when a line index falls into an unallocated segment. This is managed by `segment_for()` (`kitty/history.c`, lines 36–42):

```c
static index_type
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments
           && SEGMENT_SIZE * self->num_segments < self->ynum))
        add_segment(self);
    if (UNLIKELY(seg_num >= self->num_segments))
        fatal("Out of bounds access to history buffer line number: %u", y);
    return seg_num;
}
```

The function computes which segment a line index `y` belongs to (`seg_num = y / SEGMENT_SIZE`). If that segment doesn't exist yet, the `while` loop calls `add_segment()` until it does — but **only** if `SEGMENT_SIZE * num_segments < ynum`, which ensures we never allocate more segments than the buffer capacity requires.

The `UNLIKELY()` hint tells the compiler this path is rare, enabling branch-prediction optimization. Once all segments are allocated, `segment_for()` degenerates to a simple integer division — zero overhead.

### 4.2 The add_segment() Memory Layout

Each segment allocation (`kitty/history.c`, lines 17–29) performs exactly two heap operations:

```c
static void
add_segment(HistoryBuf *self) {
    self->num_segments += 1;
    self->segments = realloc(self->segments,
                             sizeof(HistoryBufSegment) * self->num_segments);
    if (self->segments == NULL)
        fatal("Out of memory allocating new history buffer segment");
    HistoryBufSegment *s = self->segments + self->num_segments - 1;
    const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
    const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
    s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size
                             + SEGMENT_SIZE * sizeof(LineAttrs));
    if (!s->cpu_cells)
        fatal("Out of memory allocating new history buffer segment");
    s->gpu_cells = (GPUCell*)(((uint8_t*)s->cpu_cells) + cpu_cells_size);
    s->line_attrs = (LineAttrs*)(((uint8_t*)s->gpu_cells) + gpu_cells_size);
}
```

1. **`realloc()`** on the segments pointer array — grows by one `HistoryBufSegment` struct (3 pointers, typically 24 bytes). This may copy the pointer array but never touches cell data.

2. **`calloc()`** for a single contiguous block holding all cell data for 2048 lines:
   - CPU cells: `xnum × 2048 × sizeof(CPUCell)` bytes
   - GPU cells: `xnum × 2048 × sizeof(GPUCell)` bytes
   - Line attributes: `2048 × sizeof(LineAttrs)` bytes

The GPU cells and line attributes pointers are **not** separate allocations — they are computed as offsets within the `calloc` block. This means each segment requires exactly one `calloc` call, and freeing the segment requires exactly one `free(s->cpu_cells)` call (see `free_segment()` at line 32–34).

**`SEGMENT_SIZE`** is defined as a constant at `kitty/history.c`, line 15:

```c
#define SEGMENT_SIZE 2048
```

### 4.3 Allocation Counts for Common Configurations

The total number of segments needed is `⌈ynum / SEGMENT_SIZE⌉`. The first segment is always allocated at construction time (`create_historybuf()`, line 127: `add_segment(self)`). Remaining segments are allocated lazily during Phase A as the buffer fills.

| `scrollback_lines` | `ynum` | Total Segments | Allocated at Construction | Lazy Allocations During Fill |
|--------------------:|-------:|---------------:|--------------------------:|-----------------------------:|
| 2,000 | 2,000 | 1 | 1 | 0 |
| 5,000 | 5,000 | 3 | 1 | 2 |
| 10,000 | 10,000 | 5 | 1 | 4 |
| 50,000 | 50,000 | 25 | 1 | 24 |
| 100,000 | 100,000 | 49 | 1 | 48 |

**Critical insight for the default configuration**: With `scrollback_lines=2000` and `SEGMENT_SIZE=2048`, only one segment is needed. Since the first segment is allocated at construction, **the HistoryBuf arrives fully allocated from the moment the terminal window is created**. No lazy allocation occurs during the flood — the entire fill phase is allocation-free.

For larger configurations (e.g., 10,000 lines), the lazy allocations occur during Phase A as the buffer fills to positions 2048, 4096, 6144, and 8192 (four `add_segment()` calls). Each allocation is a single `calloc` — a brief but bounded operation. Once the buffer is full, no further segment allocations occur regardless of how long the flood continues.

**Allocation is deterministic and finite**: The `while` loop in `segment_for()` terminates because `SEGMENT_SIZE * num_segments` strictly increases with each `add_segment()` call, and the loop condition `SEGMENT_SIZE * num_segments < ynum` guarantees convergence in at most `⌈ynum / SEGMENT_SIZE⌉` iterations total across the buffer's lifetime.

---

## 5. Pager History Ring Buffer Interaction

### 5.1 The Spillover Mechanism: pagerhist_push()

When `HistoryBuf` is full and a new line is pushed, the oldest line must be preserved somewhere before being overwritten. This is handled by `pagerhist_push()` (`kitty/history.c`, lines 258–273):

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

The function's behavior step by step:

1. **Early exit**: If `pagerhist` is NULL (i.e., `scrollback_pager_history_size=0`), the function returns immediately — **zero overhead** for the default configuration.

2. **Line initialization**: A temporary `Line` struct is initialized pointing to the storage at `start_of_data` — the oldest line about to be overwritten.

3. **ANSI serialization**: `line_as_ansi()` converts the line's cell data (CPU cells + GPU cells) into a sequence of ANSI escape codes stored in `as_ansi_buf` (a UCS-4 buffer).

4. **SGR reset prefix**: `\x1b[m` (3 bytes) is written to reset all text attributes before each line's ANSI content.

5. **UTF-8 encoding and write**: `pagerhist_write_ucs4()` (lines 248–256) encodes each UCS-4 character to UTF-8 and writes the result byte-by-byte via `pagerhist_write_bytes()`.

6. **Line termination**: A carriage return (`\r`) is always appended. A newline (`\n`) is appended only if the line does NOT have a soft-wrap continuation (checked via `next_char_was_wrapped` on the last cell's GPU attributes).

### 5.2 Ring Buffer Growth: pagerhist_extend()

When a write would exceed the ring buffer's current capacity, `pagerhist_write_bytes()` (`kitty/history.c`, lines 218–226) attempts to grow it:

```c
static bool
pagerhist_write_bytes(PagerHistoryBuf *ph, const uint8_t *buf, size_t sz) {
    if (sz > ph->maximum_size) return false;
    if (!sz) return true;
    size_t space_in_ringbuf = ringbuf_bytes_free(ph->ringbuf);
    if (sz > space_in_ringbuf) pagerhist_extend(ph, sz);
    ringbuf_memcpy_into(ph->ringbuf, buf, sz);
    return true;
}
```

Note the crucial design: `pagerhist_extend()` is called as a **best-effort** optimization, but `ringbuf_memcpy_into()` is called **unconditionally** afterward. If extension fails (because the buffer is already at maximum size), the ring buffer's built-in overflow handling takes care of discarding the oldest data. This two-layer approach ensures writes never block or fail under any conditions.

**`pagerhist_extend()`** (`kitty/history.c`, lines 89–101):

```c
static bool
pagerhist_extend(PagerHistoryBuf *ph, size_t minsz) {
    size_t buffer_size = ringbuf_capacity(ph->ringbuf);
    if (buffer_size >= ph->maximum_size) return false;
    size_t newsz = MIN(ph->maximum_size,
                       buffer_size + MAX(1024u * 1024u, minsz));
    ringbuf_t newbuf = ringbuf_new(newsz);
    if (!newbuf) return false;
    size_t count = ringbuf_bytes_used(ph->ringbuf);
    if (count) ringbuf_copy(newbuf, ph->ringbuf, count);
    ringbuf_free((ringbuf_t*)&ph->ringbuf);
    ph->ringbuf = newbuf;
    return true;
}
```

The growth strategy:

- **Growth formula**: `new_capacity = MIN(maximum_size, current_capacity + MAX(1 MB, minsz))`
- **Chunked growth**: Each extension adds at least 1 MB, even if the immediate write is smaller. This amortizes the cost of reallocation over many subsequent writes.
- **Hard ceiling**: Once `buffer_size >= maximum_size`, the function returns `false` and no further growth occurs.
- **Copy-and-replace**: A new ring buffer is allocated, existing data is copied via `ringbuf_copy()` (`3rdparty/ringbuf/ringbuf.c`, lines 358–394), the old buffer is freed, and the handle is swapped.

**Initial sizing** (`kitty/history.c`, line 67):

```c
static size_t
initial_pagerhist_ringbuf_sz(size_t pagerhist_sz) {
    return MIN(1024u * 1024u, pagerhist_sz);
}
```

The ring buffer starts at `MIN(1 MB, scrollback_pager_history_size)`. For a 100 MB pager history, the ring buffer begins at 1 MB and grows in 1 MB increments up to 100 MB. This means ~100 extension events over the buffer's lifetime, each involving an allocation + copy + free cycle.

### 5.3 Ring Buffer Overflow Semantics

When the ring buffer is at maximum capacity and a new write exceeds the free space, `ringbuf_memcpy_into()` (`3rdparty/ringbuf/ringbuf.c`, lines 211–238) handles the overflow:

```c
void *
ringbuf_memcpy_into(ringbuf_t dst, const void *src, size_t count)
{
    const uint8_t *u8src = src;
    const uint8_t *bufend = ringbuf_end(dst);
    int overflow = count > ringbuf_bytes_free(dst);
    size_t nread = 0;

    while (nread != count) {
        assert(bufend > dst->head);
        size_t n = size_t_min(bufend - dst->head, count - nread);
        memcpy(dst->head, u8src + nread, n);
        dst->head += n;
        nread += n;

        if (dst->head == bufend)
            dst->head = dst->buf;
    }

    if (overflow) {
        dst->tail = ringbuf_nextp(dst, dst->head);
        assert(ringbuf_is_full(dst));
    }

    return dst->head;
}
```

The overflow behavior:

1. The `overflow` flag is computed before any copying occurs.
2. Data is written into the buffer in up to two `memcpy` operations (before and after wrapping around the end of the contiguous buffer).
3. If overflow occurred, `tail` is advanced to `ringbuf_nextp(dst, dst->head)` — one position past `head`. This effectively discards the oldest bytes in the buffer to make room for the newly written data.

The `ringbuf_nextp()` function (`3rdparty/ringbuf/ringbuf.c`, lines 150–160) computes the next position with modular wrapping:

```c
static uint8_t *
ringbuf_nextp(ringbuf_t rb, const uint8_t *p)
{
    assert((p >= rb->buf) && (p < ringbuf_end(rb)));
    return rb->buf + ((++p - rb->buf) % ringbuf_buffer_size(rb));
}
```

**Key invariant**: After an overflow write, the ring buffer is always in the full state (`ringbuf_is_full(dst)` asserts). The oldest data has been silently and atomically discarded — there is no error condition, no return code to check, and no user-visible failure.

### 5.4 Flood Behavior Summary

Under extreme throughput with the pager ring buffer enabled, the system passes through three distinct phases:

1. **Pager untouched** (during HistoryBuf Phase A): No writes to the ring buffer. Zero pager overhead.

2. **Pager growing** (HistoryBuf full, ring buffer below maximum): Each evicted line triggers ANSI serialization → UTF-8 encoding → ring buffer write. Periodically (approximately every 1 MB of ANSI text), `pagerhist_extend()` allocates a new, larger ring buffer, copies existing content, and frees the old one. This produces a brief allocation + memcpy spike, but the chunked growth strategy (at least 1 MB per extension) ensures these events are rare relative to the volume of data being processed.

3. **Pager at maximum** (ring buffer at `maximum_size`): `pagerhist_extend()` returns `false`. `ringbuf_memcpy_into()` handles overflow by advancing `tail`. Each line write is O(line_length) with no heap allocation. This is the true steady state under sustained flood.

The **transition between phases is seamless** because `pagerhist_write_bytes()` always proceeds to `ringbuf_memcpy_into()` regardless of whether extension succeeded. There is no conditional branch that behaves differently when the buffer is "growing" vs. "at maximum" — the ring buffer's overflow semantics provide a uniform, allocation-free write path once the maximum is reached.

When `scrollback_pager_history_size=0` (the default), `pagerhist` is NULL. `pagerhist_push()` checks `if (!ph) return;` at line 261, making the entire spillover mechanism a single pointer comparison — effectively zero overhead.

---

## 6. Scrolling While Data Arrives

### 6.1 The scrolled_by Tracking Mechanism

When a user scrolls back in the terminal, the `scrolled_by` field of the `Screen` struct (`kitty/screen.h`, line 91) records how many history lines the viewport has moved back. The challenge is maintaining a stable viewport position while new lines are continuously being pushed into history.

The solution is implemented in `screen_update_cell_data()` (`kitty/screen.c`, lines 2737–2797). At the beginning of each render frame:

```c
unsigned int history_line_added_count = self->history_line_added_count;
// ... (line 2756)
if (self->scrolled_by)
    self->scrolled_by = MIN(self->scrolled_by + history_line_added_count,
                            self->historybuf->count);
```

*(Line 2761)*

The `history_line_added_count` accumulator tracks how many lines were pushed into the history buffer since the last render frame. It is incremented by the `INDEX_UP` macro (line 1559: `self->history_line_added_count++`) and reset to zero by `screen_reset_dirty()` (line 2600: `self->history_line_added_count = 0`), which is called at the start of each frame's rendering.

**The key insight**: By adding `history_line_added_count` to `scrolled_by`, the viewed content remains stable. Each new line pushed into history shifts the logical positions of all existing history lines by one, so the scroll position must be incremented by the same amount to continue pointing at the same content.

**Bounding**: `MIN(scrolled_by + history_line_added_count, historybuf->count)` ensures the scroll position never exceeds the number of lines actually stored in the history buffer. This prevents out-of-bounds access when lines are evicted from history (because the buffer is full and wrapping) faster than they accumulate.

### 6.2 Line Resolution via visual_line_()

The `visual_line_()` function (`kitty/screen.c`, lines 2842–2853) resolves which buffer a given display row should be read from:

```c
static Line*
visual_line_(Screen *self, int y_) {
    index_type y = MAX(0, y_);
    if (self->scrolled_by) {
        if (y < self->scrolled_by) {
            historybuf_init_line(self->historybuf,
                                self->scrolled_by - 1 - y,
                                self->historybuf->line);
            return self->historybuf->line;
        }
        y -= self->scrolled_by;
    }
    return init_line(self, y);
}
```

The rendering loop in `screen_update_cell_data()` uses a parallel approach for efficiency (lines 2763–2788):

- **For rows `y < scrolled_by`**: The line is fetched from `historybuf` using reverse indexing (`lnum = scrolled_by - 1 - y`). Line number 0 in `historybuf` is the most recently added line, so `scrolled_by - 1` is the oldest visible line.

- **For rows `y >= scrolled_by`**: The line is fetched from `linebuf` (the active screen buffer) at `lnum = y - scrolled_by`.

Each line is only re-rendered if its `has_dirty_text` flag is set in its `LineAttrs`. This per-line dirty tracking avoids unnecessary re-rendering of stable history lines, which is critical for performance when scrolled back during a flood.

### 6.3 Threading Model and Safety

There is **no locking** on `HistoryBuf`, `scrolled_by`, or any related scroll state. This is safe because Kitty's architecture serializes all buffer mutation and rendering on the main thread:

- The **I/O thread** (`child_monitor`) reads raw bytes from the PTY file descriptor.
- The **main thread** handles VT parsing, screen mutation (including `historybuf_push()`), and rendering (`screen_update_cell_data()`).

Since VT parsing and rendering are serialized on the same thread, there is no race condition between pushing lines into history and reading them for display. The `write_buf_lock` mutex (`kitty/screen.h`, line 116) protects only the write buffer used for data flowing *to* the child process, not the scrollback buffer.

### 6.4 Behavioral Analysis Under Stress

Under extreme throughput with the user scrolled back:

1. **Between render frames**: Multiple VT sequences may be parsed, each potentially calling `screen_index()` → `INDEX_UP` → `historybuf_add_line()`. The `history_line_added_count` accumulator captures the total number of lines pushed. There is no per-line rendering cost during parsing.

2. **At each render frame**: `screen_update_cell_data()` performs a single adjustment: `scrolled_by += history_line_added_count`. This correctly compensates for any number of lines added since the last frame, whether it was 1 line or 10,000 lines.

3. **Viewport stability**: The user sees the same content they were viewing. As new lines push into history and old lines are evicted, the `scrolled_by` counter increases to keep the viewport pinned to the same logical position.

4. **Boundary condition — eviction exceeds scrollback**: If the user is scrolled to the very top of history (`scrolled_by == historybuf->count`) and new lines cause old lines to be evicted (because `count == ynum`), `MIN(scrolled_by + added, historybuf->count)` caps the position. The viewed content may shift — the oldest lines the user was viewing are gone — but the system never accesses memory out of bounds.

5. **Paused rendering**: The `paused_rendering` struct (`kitty/screen.h`, lines 159–168) maintains a separate `scrolled_by` and `cursor` for the paused state. When rendering is paused (e.g., during rapid output), the paused state captures a snapshot, preventing visual glitches while the main buffer continues to be mutated.

6. **Dirty-line optimization**: History lines read from `historybuf` during scrolled-back rendering are individually checked for `has_dirty_text` (line 2769). Only dirty lines are re-rendered via `render_line()`. Under a flood, newly pushed history lines are marked dirty, but lines the user was viewing (which haven't changed) skip re-rendering — reducing GPU overhead.

---

## 7. Memory Structure Evolution Under Pressure

This section traces how the system's memory allocation patterns evolve as a sustained flood progresses through three distinct phases.

### 7.1 Phase 1 — Initial Fill

**Duration**: From the first line pushed until `count == ynum`.

**HistoryBuf state**:
- `count` increments from 0 to `ynum` (e.g., 0 → 2000)
- `start_of_data` remains 0 throughout
- `idx` advances linearly: 0, 1, 2, ..., `ynum - 1`

**Memory events**:
- Segment allocations occur as `idx` crosses segment boundaries (multiples of 2048).
- For `scrollback_lines=2000`: **zero additional allocations** — the single construction-time segment (capacity 2048 lines) suffices.
- For `scrollback_lines=10000`: four additional `add_segment()` calls at `idx = 2048, 4096, 6144, 8192`. Each allocates one contiguous block:
  - For an 80-column terminal: `80 × 2048 × (sizeof(CPUCell) + sizeof(GPUCell)) + 2048 × sizeof(LineAttrs)` bytes per segment.

**Pager ring buffer**: Untouched. No eviction occurs because the buffer is not yet full.

**Observation**: Phase 1 is a burst of monotonically increasing memory usage. The allocation pattern is predictable and finite — once all `⌈ynum / 2048⌉` segments exist, no more allocations happen in this phase.

### 7.2 Phase 2 — Steady-State Wrap with Pager Growth

**Duration**: From `count == ynum` until the pager ring buffer reaches `maximum_size` (or indefinitely if pager is disabled).

**HistoryBuf state**:
- `count` remains fixed at `ynum`
- `start_of_data` advances modularly: `(start_of_data + 1) % ynum` per line
- `idx` equals `start_of_data` (the oldest line, about to be overwritten)
- **Zero HistoryBuf allocations** — pure circular overwrite

**Pager ring buffer** (if enabled):
- Each evicted line is serialized as ANSI text and written to the ring buffer.
- When `ringbuf_bytes_free() < write_size`, `pagerhist_extend()` is called.
- Growth: at least 1 MB per extension, from initial 1 MB up to `maximum_size`.
- Each extension: `ringbuf_new(newsz)` + `ringbuf_copy()` + `ringbuf_free()`.
- For `scrollback_pager_history_size=100` (100 MB), this means approximately 99 extension events. The `ringbuf_copy()` cost increases linearly with the amount of data in the buffer, so later extensions are more expensive.

**Observation**: This phase has **zero HistoryBuf allocation** but **periodic pager allocation spikes**. Each spike involves allocating a new ring buffer (1 MB to 100 MB), copying the existing content, and freeing the old buffer. These spikes are amortized over ~10,000 lines per MB (at ~100 bytes per ANSI-encoded line), making them infrequent relative to the line throughput.

### 7.3 Phase 3 — True Steady State

**Duration**: Indefinite, after the pager ring buffer reaches `maximum_size`.

**HistoryBuf state**: Same as Phase 2 — circular overwrite.

**Pager ring buffer**:
- `pagerhist_extend()` returns `false` (line 92: `if (buffer_size >= ph->maximum_size) return false`).
- `ringbuf_memcpy_into()` overwrites oldest data on overflow (line 232–234: `dst->tail = ringbuf_nextp(dst, dst->head)`).
- **Zero allocations of any kind**. Every operation is in-place.

**Per-line cost in steady state**:
1. `historybuf_push()` — circular index computation: O(1)
2. `pagerhist_push()` — ANSI serialization: O(xnum)
3. `pagerhist_write_bytes()` — UTF-8 encoding + ring buffer write: O(xnum)
4. Cell data copy (`copy_line`): O(xnum)

Total: **O(xnum) per line with zero heap allocation**. This is the optimal operating mode under sustained flood.

### 7.4 Circular Addressing Arithmetic

The circular buffer uses modular arithmetic for all position computations:

**New line position** (`historybuf_push()`, line 277):
```c
index_type idx = (self->start_of_data + self->count) % self->ynum;
```
When the buffer is full (`count == ynum`), this simplifies to `start_of_data % ynum = start_of_data` — the new line overwrites the oldest.

**Reverse lookup** (`index_of()`, `kitty/history.c`, lines 152–159):
```c
static index_type
index_of(HistoryBuf *self, index_type lnum) {
    if (self->count == 0) return 0;
    index_type idx = self->count - 1 - MIN(self->count - 1, lnum);
    return (self->start_of_data + idx) % self->ynum;
}
```
Line number 0 is the most recently added line. For `lnum = 0`: `idx = count - 1`, so the result is `(start_of_data + count - 1) % ynum` — the last written position. For `lnum = count - 1`: `idx = 0`, so the result is `start_of_data` — the oldest line.

**Ring buffer pointer wrapping** (`ringbuf_nextp()`, `3rdparty/ringbuf/ringbuf.c`, lines 150–160):
```c
return rb->buf + ((++p - rb->buf) % ringbuf_buffer_size(rb));
```
Advances a pointer by one byte with modular wrapping. The increment is performed on the difference `(p - buf)` to ensure portability, and `ringbuf_buffer_size()` (which is `capacity + 1`) is used as the modulus to account for the sentinel byte.

**Ring buffer free space** (`ringbuf_bytes_free()`, `3rdparty/ringbuf/ringbuf.c`, lines 106–113):
```c
size_t
ringbuf_bytes_free(const struct ringbuf_t *rb)
{
    if (rb->head >= rb->tail)
        return ringbuf_capacity(rb) - (rb->head - rb->tail);
    else
        return rb->tail - rb->head - 1;
}
```
The `-1` in the else branch accounts for the sentinel byte. When `head` has wrapped around and is behind `tail`, the free space is `tail - head - 1`.

---

## 8. Observation Approaches

The following approaches describe how one could observe the described behaviors at runtime without modifying the repository. These are informational only — no scripts or code are added to the repository.

### 8.1 Python API via fast_data_types

Kitty exposes `HistoryBuf` as a Python type through the `fast_data_types` CPython extension. A custom kitten (Kitty extension) or a `scrollback_pager` command could inspect buffer state:

```python
# Observation script (NOT committed to repository)
# Run as a custom kitten or in Kitty's debug console

from kitty.fast_data_types import HistoryBuf

# Access via window object:
# buf = window.screen.historybuf
# buf.count    -> number of lines stored
# buf.ynum     -> maximum capacity
# buf.xnum     -> columns
#
# Pager history as bytes:
# data = buf.pagerhist_as_bytes()
# len(data)    -> current pager ring buffer used bytes
```

The `pagerhist_as_bytes()` method (`kitty/history.c`, lines 460–483) returns the ring buffer's contents as a Python bytes object. The `pagerhist_as_text()` method (lines 485–494) returns it as a decoded UTF-8 string.

### 8.2 Remote Control Inspection

Kitty's remote control protocol (`kitty @` commands) can be used to inspect window state:

```bash
# List all windows with their properties
kitty @ ls
```

While this doesn't directly expose `HistoryBuf.count` or segment counts, it provides window dimensions and scrollback-related configuration that can be used to infer buffer sizing.

### 8.3 Custom Scrollback Pager

The `scrollback_pager` option (`kitty/options/definition.py`, line 392) accepts a custom command. A pager could be configured to first report buffer size information (via the piped STDIN size) before displaying content:

```bash
# Example pager that reports size before viewing (NOT committed)
# scrollback_pager bash -c 'echo "Piped $(wc -c < /dev/stdin) bytes" | less'
```

The content piped to the pager includes both the HistoryBuf ANSI output and the PagerHistoryBuf content, so the total byte count reflects the combined retention.

### 8.4 C-Level Instrumentation

For deeper observation, temporary `fprintf(stderr, ...)` statements could be inserted in key functions:

- **`add_segment()`** (line 18): Log segment number and allocation sizes.
- **`pagerhist_extend()`** (line 90): Log old capacity, new capacity, and bytes copied.
- **`historybuf_push()`** (line 276): Log `count`, `start_of_data`, and `idx` to trace circular buffer state.
- **`ringbuf_memcpy_into()`** (line 216): Log `overflow` flag to observe when the ring buffer starts discarding data.

### 8.5 Existing Test Suite

The test suite already includes relevant stress scenarios:

- **`test_historybuf()`** (`kitty_tests/datatypes.py`, line 487): Creates a 3000-line `HistoryBuf` (`HistoryBuf(3000, 5)`), pushes 3000 lines, and verifies all lines are correctly stored and retrievable. This exercises the lazy segment allocation path (3000 lines requires `⌈3000/2048⌉ = 2` segments).

- **`test_resize()`** and **`test_scrollback_fill_after_resize()`** (`kitty_tests/screen.py`): Test scrollback behavior during terminal resize, which exercises the rewrap machinery.

These tests can be run with `python3 setup.py test` to verify the behavioral claims in this document.

---

## 9. Conclusion

This investigation reveals a carefully layered design in Kitty's scrollback buffer subsystem that handles extreme throughput gracefully:

### Key Findings

1. **HistoryBuf under flood is predictable and bounded**: The segmented circular buffer fills during an initial phase with a deterministic number of segment allocations (`⌈ynum / 2048⌉`), then transitions to a zero-allocation steady state. For the default 2000-line scrollback, the buffer is fully allocated at construction — the flood never triggers a single additional allocation.

2. **Pager ring buffer transition is smooth**: The spillover from `HistoryBuf` to `PagerHistoryBuf` is mediated by `pagerhist_write_bytes()`, which always succeeds because `ringbuf_memcpy_into()` handles overflow by silently discarding the oldest data. Growth occurs in 1 MB chunks, amortizing allocation costs. The transition from the "growing" phase to the "at maximum" phase requires no code path changes — the same `ringbuf_memcpy_into()` call handles both cases identically.

3. **Scrolled-back display is correctly maintained under stress**: The `scrolled_by` tracking mechanism uses a per-render-frame accumulator (`history_line_added_count`) that correctly compensates for any number of lines pushed between frames. The bounding `MIN(scrolled_by + added, historybuf->count)` prevents out-of-bounds access. The single-threaded architecture eliminates race conditions.

4. **Memory reaches true steady state**: After the initial fill (Phase 1) and pager growth (Phase 2), the system operates with zero ongoing heap allocation (Phase 3). Every line push is O(xnum) with in-place circular overwrite and ring buffer write. This steady state can persist indefinitely.

### Design Virtues

- **Separation of concerns**: The line-level circular buffer (`HistoryBuf`) and byte-level ring buffer (`PagerHistoryBuf`) serve complementary roles — random access vs. sequential pager output — without coupling their internal state.

- **Graceful degradation**: When the pager ring buffer is at maximum capacity, data is silently discarded from the oldest end. There is no error propagation, no user-visible failure, and no performance cliff.

- **Lazy allocation**: Segment allocation is deferred until needed, reducing startup cost for terminals that may never fill their scrollback buffer.

- **Batched scroll compensation**: By accumulating `history_line_added_count` across an entire render frame rather than adjusting `scrolled_by` per-line, the system avoids O(n) rendering work during a burst of n lines within a single frame.

---

*Document generated from Kitty v0.35.2 source code analysis. All file paths and line numbers reference the `kitty_815df1e210e0` branch.*
