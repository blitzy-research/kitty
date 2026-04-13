# Kitty Terminal Emulator — Scrollback History Buffer Analysis

## Introduction

This document presents a comprehensive, evidence-based investigation of the Kitty terminal emulator's scrollback history buffer behavior under stress conditions. It answers four specific behavioral questions:

1. **Memory Consumption Under Heavy Output** — What happens to memory when hundreds of thousands of lines are printed rapidly?
2. **Scroll Responsiveness During Concurrent Output** — Does scrolling remain responsive while new output is being appended?
3. **Buffer Boundary Behavior and Allocation Transitions** — At what exact points does the internal storage scheme change as the buffer grows?
4. **Observable Measurement Methods** — How can these behaviors be observed empirically?

### Methodology

Every conclusion in this document is derived from direct analysis of the Kitty source code. Each claim references specific source files and line numbers. No assumptions are made — the code is the sole source of truth.

**No files in the Kitty repository have been modified.** All measurement scripts described in this document are external and temporary — they are intended to be run outside the repository and discarded after use.

---

## Section 1: Memory Consumption Under Heavy Output

> *"If I generate a massive amount of terminal output, say, printing hundreds of thousands of lines rapidly, what happens to memory consumption as the history accumulates?"*

### 1.1 Data Structure Foundations

The scrollback history buffer's memory footprint is determined by three fundamental cell structures and their enclosing allocation scheme. The exact sizes are verified by compile-time assertions in the source code.

#### Cell Structures

| Structure | Size | Verification | Contents |
|-----------|------|--------------|----------|
| `GPUCell` | **20 bytes** | `static_assert(sizeof(GPUCell) == 20, "Fix the ordering of GPUCell")` at `kitty/data-types.h:221` | Foreground color, background color, decoration foreground color, sprite coordinates (x, y, z), and cell attributes (`CellAttrs`) — defined at `kitty/data-types.h:216-220` |
| `CPUCell` | **12 bytes** | `static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell")` at `kitty/data-types.h:228` | Character codepoint (`char_type ch`), hyperlink ID, and three combining character indices — defined at `kitty/data-types.h:223-227` |
| `LineAttrs` | **1 byte** | `uint8_t val` in union — `kitty/data-types.h:231-239` | Bitfield union containing `is_continued` (1 bit), `has_dirty_text` (1 bit), `has_image_placeholders` (1 bit), and `prompt_kind` (2 bits) |

#### Buffer Structures

The `HistoryBuf` structure (`kitty/data-types.h:282-290`) orchestrates the entire scrollback buffer:

```
HistoryBuf {
    xnum           — number of columns per line
    ynum           — maximum number of lines (scrollback_lines)
    num_segments   — current number of allocated segments
    segments       — pointer to array of HistoryBufSegment structs
    pagerhist      — pointer to PagerHistoryBuf (optional, for pager overflow)
    line           — reusable Line object for iteration
    start_of_data  — circular buffer head index
    count          — number of lines currently stored
}
```

Each `HistoryBufSegment` (`kitty/data-types.h:262-266`) holds contiguous storage for up to 2048 lines:

```
HistoryBufSegment {
    gpu_cells   — pointer to GPUCell array
    cpu_cells   — pointer to CPUCell array
    line_attrs  — pointer to LineAttrs array
}
```

The optional `PagerHistoryBuf` (`kitty/data-types.h:268-272`) provides overflow storage as a ring buffer:

```
PagerHistoryBuf {
    ringbuf        — void pointer to ring buffer (from 3rdparty/ringbuf/)
    maximum_size   — configured maximum capacity in bytes
    rewrap_needed  — flag for pending rewrap after resize
}
```

### 1.2 Per-Line Memory Cost

For any given terminal width, the per-line memory cost in the history buffer is:

```
per_line = columns × sizeof(CPUCell) + columns × sizeof(GPUCell) + sizeof(LineAttrs)
         = columns × 12 + columns × 20 + 1
         = columns × 32 + 1
```

| Terminal Width | Per-Line Cost | Calculation |
|----------------|---------------|-------------|
| 80 columns (default) | **2,561 bytes ≈ 2.5 KB** | `80 × 12 + 80 × 20 + 1 = 960 + 1,600 + 1` |
| 132 columns | **4,225 bytes ≈ 4.1 KB** | `132 × 12 + 132 × 20 + 1 = 1,584 + 2,640 + 1` |
| 200 columns | **6,401 bytes ≈ 6.3 KB** | `200 × 12 + 200 × 20 + 1 = 2,400 + 4,000 + 1` |
| 240 columns | **7,681 bytes ≈ 7.5 KB** | `240 × 12 + 240 × 20 + 1 = 2,880 + 4,800 + 1` |

### 1.3 Segment Allocation Scheme

The history buffer does **not** allocate all memory upfront. Instead, it uses a segmented allocation scheme where memory is allocated in fixed-size segments of 2048 lines each.

**Segment Size Constant:** `SEGMENT_SIZE` is defined as `2048` at `kitty/history.c:15`:

```c
#define SEGMENT_SIZE 2048
```

**Segment Allocation Function:** `add_segment()` at `kitty/history.c:17-29` performs the allocation:

1. Increments `self->num_segments` (line 19)
2. `realloc`s the segment pointer array: `sizeof(HistoryBufSegment) * self->num_segments` (line 20)
3. Computes the data block sizes (lines 23-24):
   - `cpu_cells_size = xnum * SEGMENT_SIZE * sizeof(CPUCell)`
   - `gpu_cells_size = xnum * SEGMENT_SIZE * sizeof(GPUCell)`
4. Performs a single `calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs))` for the entire data block (line 25)
5. Sets up internal pointers within the contiguous block (lines 27-28): `gpu_cells` is placed immediately after `cpu_cells`, and `line_attrs` immediately after `gpu_cells`

**Per-Segment Memory at 80 Columns:**

```
cpu_cells:  80 × 2048 × 12  = 1,966,080 bytes
gpu_cells:  80 × 2048 × 20  = 3,276,800 bytes
line_attrs: 2048 × 1         =     2,048 bytes
────────────────────────────────────────────────
Total:                        = 5,244,928 bytes ≈ 5.00 MB
```

Plus the segment pointer array overhead: `sizeof(HistoryBufSegment) × num_segments` — this is a negligible 24 bytes per segment (3 pointers × 8 bytes on 64-bit systems).

**Lazy Allocation:** Segments are allocated on-demand by `segment_for()` at `kitty/history.c:36-42`. The function checks whether the requested segment number exceeds the current count:

```c
while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum))
    add_segment(self);
```

This means a new ~5 MB segment is allocated only when a line is first written into a segment that doesn't yet exist. The condition `SEGMENT_SIZE * self->num_segments < self->ynum` ensures no segment is allocated beyond what `ynum` (the configured `scrollback_lines`) requires.

**Initial Allocation:** The constructor `create_historybuf()` at `kitty/history.c:116-133` always allocates exactly one segment at construction time (line 127: `add_segment(self)`). This means the minimum memory footprint for a history buffer is one full segment (~5 MB at 80 columns), regardless of how many lines are actually stored.

### 1.4 Memory Scaling Table

The number of segments required for a given `scrollback_lines` setting is `ceil(scrollback_lines / SEGMENT_SIZE)`. Since segments are lazily allocated, the steady-state memory equals `num_segments × segment_size`.

| `scrollback_lines` | Segments Required | Memory at 80 cols | Memory at 132 cols | Memory at 200 cols |
|--------------------|-------------------|--------------------|--------------------|---------------------|
| 2,000 (default) | 1 (2048 ≥ 2000) | **~5.00 MB** | **~8.25 MB** | **~12.50 MB** |
| 5,000 | 3 (6144 ≥ 5000) | ~15.0 MB | ~24.8 MB | ~37.5 MB |
| 10,000 | 5 (10240 ≥ 10000) | ~25.0 MB | ~41.3 MB | ~62.5 MB |
| 50,000 | 25 (51200 ≥ 50000) | ~125 MB | ~206 MB | ~313 MB |
| 100,000 | 49 (100352 ≥ 100000) | ~245 MB | ~404 MB | ~613 MB |
| 500,000 | 245 (501760 ≥ 500000) | ~1,225 MB ≈ 1.2 GB | ~2.0 GB | ~3.0 GB |

**Negative (Infinite) Scrollback:** The `scrollback_lines()` parser in `kitty/options/utils.py:557-561` converts any negative value to `2^32 - 1` (4,294,967,295):

```python
def scrollback_lines(x: str) -> int:
    ans = int(x)
    if ans < 0:
        ans = 2 ** 32 - 1
    return ans
```

At 2^32 - 1 lines, the theoretical maximum would require 2,097,152 segments ≈ 10 TB of memory at 80 columns. In practice, this is never reached because:
- Allocation is lazy — segments are created only as lines are pushed
- The system will run out of memory long before this limit
- The documentation in `kitty/options/definition.py:376-378` explicitly warns: *"using very large scrollback is not recommended as it can slow down performance of the terminal and also use large amounts of RAM"*

### 1.5 Configuration Defaults and Limits

| Setting | Default | Source | Notes |
|---------|---------|--------|-------|
| `scrollback_lines` | **2000** | `kitty/options/definition.py:372` | Memory allocated on demand; negative → 2^32 - 1 |
| `scrollback_pager_history_size` | **0** (disabled) | `kitty/options/definition.py:406` | In MB; 0 disables pager history entirely |
| Max pager history | **~4 GB** | `kitty/options/utils.py:566`: `min(ans, 4096 * 1024 * 1024 - 1)` | Hard ceiling of 4,294,967,295 bytes |
| `repaint_delay` | **10 ms** | `kitty/options/definition.py:866` | Delay between screen updates |
| `input_delay` | **3 ms** | `kitty/options/definition.py:878` | Delay before processing program input |

**Important:** Changes to `scrollback_lines` only affect newly created windows, not existing ones. This is documented at `kitty/options/definition.py:379-380` and enforced because the `HistoryBuf` is allocated once in the Screen constructor (`kitty/screen.c:130`):

```c
self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));
```

Note the `MAX(scrollback, lines)` — the history buffer is always at least as large as the window height, ensuring that a full screen of content can be scrolled back even with `scrollback_lines=0`.

### 1.6 Pager History (Secondary Ring Buffer)

When `scrollback_pager_history_size > 0`, a secondary storage system captures lines that overflow from the primary history buffer. This allows the pager (invoked via `scrollback_pager` command) to access far more history than the interactive scrollback buffer retains.

**Creation:** `alloc_pagerhist()` at `kitty/history.c:69-80`:
- Allocates a `PagerHistoryBuf` struct
- Creates an initial ring buffer with size `MIN(1 MB, maximum_size)` — from `initial_pagerhist_ringbuf_sz()` at line 67
- The ring buffer implementation is from `3rdparty/ringbuf/ringbuf.h` — a byte-addressable FIFO (public domain, written by Drew Hess in 2011)

**Data Format:** `pagerhist_push()` at `kitty/history.c:258-273`:
- Converts the line to ANSI-encoded UTF-8 text via `line_as_ansi()` (line 265)
- Prefixes each line with an SGR reset: `\x1b[m` (3 bytes) — line 266
- Appends `\r` always, and `\n` only if the line was NOT soft-wrapped (lines 268-271)
- This text-based storage is much more compact than the raw cell data (~80-100 bytes per 80-column ASCII line vs. 2,561 bytes in the primary buffer)

**Growth Behavior:** `pagerhist_extend()` at `kitty/history.c:90-101`:
- Triggered when `pagerhist_write_bytes()` (line 223) detects insufficient free space: `if (sz > space_in_ringbuf) pagerhist_extend(ph, sz)`
- Growth increment: `MIN(ph->maximum_size, buffer_size + MAX(1 MB, minsz))` (line 93) — grows by at least 1 MB each time
- Growth stops when `buffer_size >= ph->maximum_size` (line 92) — after this point, the ring buffer silently overwrites the oldest data via FIFO wrap-around (`ringbuf_memcpy_into` in `3rdparty/ringbuf/ringbuf.c`)

**Growth Sequence (for a 100 MB pager history):**
```
Initial:    1 MB
Extension:  2 MB  (1 + MAX(1, needed))
Extension:  3 MB  (2 + 1)
Extension:  4 MB  (3 + 1)
...
Extension: 100 MB (capped at maximum_size)
→ After reaching 100 MB, ring buffer wraps — no further allocation.
```

### 1.7 Step-by-Step Memory Timeline

**Scenario: Printing 100,000 lines at 80 columns with default `scrollback_lines=2000` and `scrollback_pager_history_size=0`:**

| Event | What Happens | Memory Change |
|-------|-------------|---------------|
| **Screen creation** | `create_historybuf()` (line 127) allocates 1 segment via `add_segment()` | +5.00 MB |
| **Lines 1–2000** | Lines pushed via `historybuf_push()` (line 276); `count` increments from 0 to 2000 | None — data fills the pre-allocated segment |
| **Line 2001** | `count == ynum` (line 279 in `historybuf_push`). Buffer is now full. `start_of_data` advances via `(start_of_data + 1) % ynum` (line 281). Oldest line overwritten. | **None** — purely arithmetic wrap-around |
| **Lines 2001–100,000** | Each new line overwrites the oldest. `start_of_data` advances by 1 each time. No pager history (disabled). | **None** — steady state at ~5.00 MB |

**Total steady-state memory: ~5.00 MB** (one segment, because 2048 ≥ 2000 and no additional segments are needed).

**Scenario: Same but with `scrollback_lines=100000`:**

| Event | What Happens | Memory Change |
|-------|-------------|---------------|
| **Screen creation** | 1 segment allocated (~5.00 MB) | +5.00 MB |
| **Lines 1–2048** | First segment fills | None |
| **Line 2049** | `segment_for()` triggers `add_segment()` — segment 2 allocated | +5.00 MB (total ~10 MB) |
| **Line 4097** | Segment 3 allocated | +5.00 MB (total ~15 MB) |
| **Line 6145** | Segment 4 allocated | +5.00 MB (total ~20 MB) |
| ... | Pattern continues every 2048 lines | +5.00 MB each |
| **Line 100,000** | 49 segments allocated (49 × 2048 = 100,352 ≥ 100,000) | Total ~245 MB |
| **Lines 100,001+** | Circular wrap begins: `count == ynum`. No new segments needed. | **None** — steady state at ~245 MB |

**Key Takeaway:** Memory grows in ~5 MB steps every 2048 lines until `scrollback_lines` is reached, then stops growing entirely. The buffer enters a steady state where new lines overwrite old lines with zero allocation.

---

## Section 2: Scroll Responsiveness During Concurrent Output

> *"Is the terminal still responsive for scrolling back through the history while new output is being appended?"*

### 2.1 Threading Architecture

Kitty uses a multi-threaded architecture, but with an important constraint: **parsing and rendering both occur on the Main Thread**.

| Thread | Responsibility | Source |
|--------|---------------|--------|
| **I/O Thread** | Polls PTY file descriptors via `poll()`/`epoll`, reads child process output into buffers | `kitty/child-monitor.c` |
| **Main Thread** | Runs VT parser (`parse_input`), updates screen model (including history buffer), processes user input (including scroll commands), renders frames via OpenGL | `kitty/child-monitor.c`, `kitty/screen.c`, `kitty/shaders.c` |

**Critical Insight:** Both new output processing (VT parsing → screen model updates → history buffer additions) and scroll input processing (`screen_history_scroll()`) happen on the Main Thread. They are **serialized, not truly concurrent**. The I/O Thread's role is limited to reading raw bytes from the PTY — the actual interpretation and rendering of those bytes happens on the Main Thread.

### 2.2 Render Scheduling Parameters

Two key configuration parameters in the `Options` struct (`kitty/state.h:51`) control how the Main Thread time-multiplexes between input processing and rendering:

| Parameter | Default | Source | Effect |
|-----------|---------|--------|--------|
| `repaint_delay` | **10 ms** | `kitty/options/definition.py:866-874` | Minimum delay between screen updates; yields ~100 FPS. Ignored when there is pending input to process. |
| `input_delay` | **3 ms** | `kitty/options/definition.py:878-887` | Delay before input from the child program is processed. Decreasing increases responsiveness but may increase CPU usage. Ignored when input buffer is almost full. |

These are stored as `monotonic_t repaint_delay, input_delay` in the Options struct at `kitty/state.h:51`.

The `input_delay` parameter is particularly important for scroll responsiveness: it ensures that pending user input (including scroll commands) is processed **before** the next render frame, minimizing perceived latency.

### 2.3 Scroll Mechanics — From User Action to Display

**Step 1: User Action (Python Layer)**

Scroll actions are defined in `kitty/window.py:1831-1871`:

```python
def scroll_line_up(self) -> Optional[bool]:       # line 1832
    if self.screen.is_main_linebuf():
        self.screen.scroll(SCROLL_LINE, True)       # line 1834

def scroll_page_up(self) -> Optional[bool]:        # line 1846
    if self.screen.is_main_linebuf():
        self.screen.scroll(SCROLL_PAGE, True)       # line 1848

def scroll_home(self) -> Optional[bool]:           # line 1860
    if self.screen.is_main_linebuf():
        self.screen.scroll(SCROLL_FULL, True)       # line 1862
```

All scroll actions call `self.screen.scroll(amount, upwards)` which invokes the C-level function.

**Step 2: Scroll Computation (C Layer)**

`screen_history_scroll()` at `kitty/screen.c:4091-4118`:

1. Translates symbolic amounts (lines 4092-4104):
   - `SCROLL_LINE` → `1`
   - `SCROLL_PAGE` → `self->lines - 1` (one less than window height)
   - `SCROLL_FULL` → `self->historybuf->count` (entire history)
2. For downward scrolling, caps at current `scrolled_by` (line 4107)
3. Computes new position: `new_scroll = MIN(self->scrolled_by + amt, self->historybuf->count)` (line 4111)
4. Updates `self->scrolled_by = new_scroll` (line 4113)
5. Calls `dirty_scroll(self)` (line 4114) — which sets `scroll_changed = true` and calls `screen_pause_rendering(self, false, 0)` (`kitty/screen.c:1907-1911`)

The `scrolled_by` field is stored in the `Screen` struct at `kitty/screen.h:91`.

**Step 3: Rendering the Scrolled View**

`cell_prepare_to_render()` at `kitty/shaders.c:394-452` is the GPU rendering entry point. At line 418:

```c
if (screen->reload_all_gpu_data || screen->scroll_changed || screen->is_dirty || screen_resized ||
    (disable_ligatures && cursor_pos_changed)) update_cell_data;
```

When `scroll_changed` is true, this triggers `screen_update_cell_data()` (via the `update_cell_data` macro at lines 408-414), which uploads cell data to the GPU.

**Step 4: Visible-Lines-Only Rendering**

`screen_update_cell_data()` at `kitty/screen.c:2738-2797` is **the key to scroll responsiveness**. It iterates only over visible lines:

- Lines 2763-2774: For the top `MIN(self->lines, self->scrolled_by)` rows — fetches from the **history buffer**:
  ```c
  for (index_type y = 0; y < MIN(self->lines, self->scrolled_by); y++) {
      lnum = self->scrolled_by - 1 - y;
      historybuf_init_line(self->historybuf, lnum, self->historybuf->line);
      // ... render and upload
  }
  ```
- Lines 2776-2787: For the remaining rows — fetches from the **live line buffer**

**This is O(visible_lines), NOT O(total_history).** For a typical 24-line terminal, only 24 lines are fetched and rendered, regardless of whether the history contains 2,000 or 2,000,000 lines. This means scroll rendering cost is constant with respect to history size.

For each history line, `historybuf_init_line()` (`kitty/history.c:180-182`) resolves pointers via `index_of()` (lines 152-159) and `init_line()` (lines 161-176). The `index_of()` function performs:

```c
index_type idx = self->count - 1 - MIN(self->count - 1, lnum);
return (self->start_of_data + idx) % self->ynum;
```

This is **O(1)** — a simple modular arithmetic calculation followed by a pointer dereference into the segment array via `segment_for()`.

### 2.4 View Stability During Concurrent Output

A critical behavior when scrolled back and new output arrives simultaneously is handled at `kitty/screen.c:2761`:

```c
if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);
```

This means: when the user is scrolled back (`scrolled_by > 0`) and new lines are added to the history buffer, `scrolled_by` is automatically incremented by the number of new lines added. This **keeps the user's view position stable** — the visible content doesn't shift even as new lines are pushed into the history above the viewport.

The `history_line_added_count` is captured into a local variable at line 2756, representing how many lines were added to history since the last render cycle. The screen's dirty state is then reset via `screen_reset_dirty()` at line 2759 before the `scrolled_by` adjustment at line 2761 — the captured local variable is unaffected by the reset, so both operations work independently.

### 2.5 Pause Rendering Mechanism

`screen_pause_rendering()` at `kitty/screen.c:2506-2544` provides a mechanism for applications to freeze the visual state while performing rapid updates:

- **Activation:** Triggered by the CSI private mode set sequence `\x1b[?2026h` (Synchronized Updates protocol)
- **Behavior when paused** (`paused_rendering.expires_at` is set):
  - A snapshot of the current screen state is captured: line buffer, cursor, color profile, selections (lines 2524-2542)
  - The renderer uses this snapshot instead of the live state (`cell_prepare_to_render()` checks `screen->paused_rendering.expires_at` at line 416)
- **Default timeout:** 2000 ms (line 2521: `if (for_in_ms <= 0) for_in_ms = 2000`) — prevents indefinite freeze if the application fails to send the resume sequence
- **Deactivation:** Triggered by `\x1b[?2026l` or timeout expiry; sets `is_dirty = true` (line 2511) to force a full refresh

The `dirty_scroll()` function (line 1910) calls `screen_pause_rendering(self, false, 0)` — this **deactivates** any active pause when the user scrolls, ensuring that scroll input is immediately visible even if an application had paused rendering.

### 2.6 Scroll Responsiveness Summary

| Factor | Impact on Responsiveness |
|--------|-------------------------|
| **Rendering is O(visible_lines)** | Scrolling through 100,000 lines is as fast as scrolling through 100 — only ~24 lines are rendered per frame |
| **History line lookup is O(1)** | `index_of()` + `segment_for()` = modular arithmetic + array indexing |
| **`scrolled_by` auto-adjustment** | View stays stable during concurrent output — no visual jumping |
| **`input_delay` (3ms)** | Pending scroll input is processed before rendering, minimizing latency |
| **Single-threaded parsing+rendering** | Under extreme output load, scroll may feel slightly laggy because the Main Thread is busy parsing VT sequences — this is inherent to the architecture |
| **`repaint_delay` (10ms)** | ~100 FPS cap; lowering to 1ms improves smoothness at cost of CPU |
| **Pause rendering exits on scroll** | `dirty_scroll()` deactivates any application-initiated pause |

**Bottom line:** Yes, the terminal remains responsive during concurrent output. The primary design factor is that scroll rendering cost is proportional to the visible window size, not the history size. Under extreme output rates (millions of bytes/second), there may be brief latency spikes as the Main Thread is saturated with VT parsing, but the `input_delay` mechanism ensures scroll input is prioritized within the event loop.

---

## Section 3: Buffer Boundary Behavior and Allocation Transitions

### 3.1 Segment Boundary Allocation (Every 2048 Lines)

As the history buffer fills, new memory segments are allocated at predictable intervals. The allocation trigger is in `segment_for()` at `kitty/history.c:36-42`:

```c
static index_type
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum))
        add_segment(self);
    if (UNLIKELY(seg_num >= self->num_segments))
        fatal("Out of bounds access to history buffer line number: %u", y);
    return seg_num;
}
```

**When allocation occurs:** The `segment_for()` function is called whenever a line's buffer position falls into a segment that hasn't been allocated yet. Specifically:

- The buffer position `y` is translated to segment number `seg_num = y / SEGMENT_SIZE`
- If `seg_num >= self->num_segments` (segment doesn't exist yet) AND `SEGMENT_SIZE * self->num_segments < self->ynum` (haven't exceeded configured capacity), then `add_segment()` is called

**Observable transitions at 80 columns with `scrollback_lines` > 2048:**

| History Line Count | Segment Allocated | `add_segment()` Called? | System Call | Memory Impact |
|--------------------|-------------------|------------------------|-------------|---------------|
| 0 (construction) | Segment 0 | Yes (from `create_historybuf()` line 127) | `calloc(1, 5,244,928)` | +5.00 MB |
| 2048 | Segment 1 | Yes (from `segment_for()` line 39) | `realloc` + `calloc(1, 5,244,928)` | +5.00 MB |
| 4096 | Segment 2 | Yes | `realloc` + `calloc(1, 5,244,928)` | +5.00 MB |
| 6144 | Segment 3 | Yes | `realloc` + `calloc(1, 5,244,928)` | +5.00 MB |
| ... | ... | ... | ... | ... |

Each `add_segment()` call (`kitty/history.c:17-29`) performs:
1. `realloc(self->segments, sizeof(HistoryBufSegment) * self->num_segments)` — grows the pointer array (line 20)
2. `calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs))` — allocates the data block (line 25)

Both are O(1) operations (the `realloc` of the pointer array is negligible), but the `calloc` involves a kernel memory allocation of ~5 MB which may briefly stall the thread if the system is under memory pressure.

### 3.2 Circular Wrap-Around (Steady-State Buffer)

Once the buffer has filled to its capacity (`count == ynum`), it transitions to a circular overwrite mode. This happens in `historybuf_push()` at `kitty/history.c:275-284`:

```c
static index_type
historybuf_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
    index_type idx = (self->start_of_data + self->count) % self->ynum;   // line 277
    init_line(self, idx, self->line);
    if (self->count == self->ynum) {                                      // line 279
        pagerhist_push(self, as_ansi_buf);                                // line 280
        self->start_of_data = (self->start_of_data + 1) % self->ynum;   // line 281
    } else self->count++;                                                 // line 282
    return idx;
}
```

**The wrap-around transition occurs exactly once:** when `count` first reaches `ynum`. After this point:

- **No memory allocation occurs** — the existing segments are reused
- Line 277: The insertion index is computed as `(start_of_data + count) % ynum` — this wraps around to the beginning of the buffer
- Line 281: `start_of_data` advances by 1 (modulo `ynum`), effectively "forgetting" the oldest line
- Line 280: Before overwriting, the oldest line is pushed to pager history (if enabled) via `pagerhist_push()`

This is **purely arithmetic** — the cost per line in steady state is:
1. One modular division for index computation
2. One `memcpy`-equivalent for copying cell data (via `copy_line` in `historybuf_add_line` at line 289)
3. Optionally, one UTF-8 encoding pass for pager history

### 3.3 Pager History Ring Buffer Expansion

When `scrollback_pager_history_size > 0`, the pager ring buffer undergoes its own growth transitions.

**Initial Allocation:** `alloc_pagerhist()` at `kitty/history.c:69-80`:
- Calls `initial_pagerhist_ringbuf_sz(pagerhist_sz)` (line 75) which returns `MIN(1024 * 1024, pagerhist_sz)` — so the initial size is 1 MB or the configured maximum, whichever is smaller (line 67)
- Creates ring buffer via `ringbuf_new(sz)` (line 76)

**Growth Trigger:** `pagerhist_write_bytes()` at `kitty/history.c:218-226`:
```c
size_t space_in_ringbuf = ringbuf_bytes_free(ph->ringbuf);     // line 222
if (sz > space_in_ringbuf) pagerhist_extend(ph, sz);           // line 223
```

Growth is triggered when the data to be written exceeds the available free space in the ring buffer.

**Growth Mechanics:** `pagerhist_extend()` at `kitty/history.c:90-101`:

1. Checks if already at maximum: `if (buffer_size >= ph->maximum_size) return false` (line 92) — at maximum, no growth; the ring buffer wraps and overwrites oldest data
2. Computes new size: `newsz = MIN(ph->maximum_size, buffer_size + MAX(1024u * 1024u, minsz))` (line 93) — grows by at least 1 MB
3. Creates a new, larger ring buffer: `ringbuf_new(newsz)` (line 94)
4. Copies existing data: `ringbuf_copy(newbuf, ph->ringbuf, count)` (line 97)
5. Frees old ring buffer: `ringbuf_free((ringbuf_t*)&ph->ringbuf)` (line 98)
6. Replaces pointer: `ph->ringbuf = newbuf` (line 99)

**Growth Transition Table (for `scrollback_pager_history_size=100` MB):**

| Transition | Buffer Size | System Call | Impact |
|------------|-------------|-------------|--------|
| Initial | 1 MB | `ringbuf_new(1MB)` — internally `malloc(1MB + 1)` | +1 MB |
| 1st growth | 2 MB | `ringbuf_new(2MB)` + `ringbuf_copy` + `ringbuf_free(old)` | +1 MB net |
| 2nd growth | 3 MB | Same pattern | +1 MB net |
| ... | ... | ... | ... |
| 99th growth | 100 MB | Final growth to maximum | +1 MB net |
| After max | 100 MB (fixed) | No allocation; `ringbuf_memcpy_into` overwrites oldest | None |

**Ring Buffer Internal Details:** From `3rdparty/ringbuf/ringbuf.h:30-80`:
- Type: `ringbuf_t` — opaque struct with head and tail pointers
- Uses one extra byte internally to distinguish "full" from "empty" state (`ringbuf_buffer_size()` returns `capacity + 1`)
- `ringbuf_bytes_free()` returns available space
- `ringbuf_memcpy_into()` writes data, wrapping around the end if necessary

### 3.4 Buffer Clear Behavior

When the scrollback buffer is cleared (e.g., via `CSI 3 J` or `clear_scrollback` action), memory is reclaimed:

**`historybuf_clear()`** at `kitty/history.c:210-216`:
1. Clears pager history: `pagerhist_clear(self)` (line 211)
2. Resets counters: `count = 0`, `start_of_data = 0` (lines 212-213)
3. Frees all segments except the first: `for (i = 1; i < num_segments; i++) free_segment(segments + i)` (line 214)
4. Resets segment count: `num_segments = 1` (line 215)

This means a clear operation releases all memory back to the single initial segment (~5 MB at 80 columns).

**`pagerhist_clear()`** at `kitty/history.c:103-114`:
1. Resets the ring buffer: `ringbuf_reset(ph->ringbuf)` (line 106)
2. Reallocates at initial size: creates a new buffer at `MIN(1MB, maximum_size)` (lines 107-111)
3. Frees the old (potentially large) buffer

**`screen_clear_scrollback()`** at `kitty/screen.c:1914-1920`:
```c
static void screen_clear_scrollback(Screen *self) {
    historybuf_clear(self->historybuf);
    if (self->scrolled_by != 0) {
        self->scrolled_by = 0;
        dirty_scroll(self);
    }
}
```

This resets the scroll position to 0 and marks the display as needing a refresh.

### 3.5 Summary of Allocation Transition Points

| Transition | Trigger | Memory Operation | Frequency |
|------------|---------|------------------|-----------|
| **Initial segment** | `create_historybuf()` | `calloc` ~5 MB | Once per window |
| **New segment** | Every 2048th line pushed | `realloc` (pointer array) + `calloc` ~5 MB | `ceil(ynum/2048) - 1` times total |
| **Circular wrap** | `count` reaches `ynum` | None (arithmetic only) | Once per buffer lifetime |
| **Pager ring buffer creation** | First line overflows when pager enabled | `malloc` 1 MB | Once per window (if enabled) |
| **Pager ring buffer growth** | Ring buffer full, below maximum | `malloc` (new) + `memcpy` + `free` (old) | Up to `(max_size - 1MB) / 1MB` times |
| **Pager ring buffer wrap** | Ring buffer full, at maximum | None (FIFO overwrite) | Continuous after maximum reached |
| **Buffer clear** | `CSI 3 J` or explicit clear | `free` all segments except first; reallocate pager | On demand |

---

## Section 4: Observable Measurement Methods

The following approaches allow empirical observation of the behaviors described above without modifying any files in the Kitty repository. All scripts are external and temporary.

### 4.1 Memory Monitoring

**Script: Monitor Kitty's Resident Set Size (Linux)**

```bash
#!/bin/bash
# Save as: /tmp/monitor_kitty_memory.sh
# Usage: bash /tmp/monitor_kitty_memory.sh <kitty_pid>

PID=${1:-$(pgrep -f 'kitty$' | head -1)}
echo "Monitoring PID $PID — press Ctrl+C to stop"
echo "Timestamp,VmRSS_kB,VmHWM_kB"

while kill -0 "$PID" 2>/dev/null; do
    RSS=$(grep VmRSS /proc/$PID/status | awk '{print $2}')
    HWM=$(grep VmHWM /proc/$PID/status | awk '{print $2}')
    echo "$(date +%s.%N),$RSS,$HWM"
    sleep 0.5
done
```

**Alternative: Python with psutil**

```python
#!/usr/bin/env python3
# Save as: /tmp/monitor_kitty_memory.py
# Usage: python3 /tmp/monitor_kitty_memory.py <kitty_pid>

import psutil, time, sys

pid = int(sys.argv[1]) if len(sys.argv) > 1 else None
if pid is None:
    for proc in psutil.process_iter(['name']):
        if proc.info['name'] == 'kitty':
            pid = proc.pid
            break

if pid is None:
    print("Kitty process not found"); sys.exit(1)

proc = psutil.Process(pid)
print(f"Monitoring PID {pid}")
print("Time(s),RSS_MB,VMS_MB")

start = time.monotonic()
try:
    while True:
        mem = proc.memory_info()
        elapsed = time.monotonic() - start
        print(f"{elapsed:.1f},{mem.rss / 1048576:.1f},{mem.vms / 1048576:.1f}")
        time.sleep(0.5)
except (psutil.NoSuchProcess, KeyboardInterrupt):
    pass
```

### 4.2 Output Generation Scripts

These scripts generate controlled volumes of terminal output to fill the scrollback buffer:

**Fast Numeric Output:**
```bash
# Generates 100,000 short lines
seq 1 100000
```

**Controlled Line Length (80 columns):**
```bash
# Generates 500,000 lines of exactly 80 characters each
python3 -c "
for i in range(500000):
    print(f'Line {i:>6d}: ' + 'x' * 70)
"
```

**Variable-Width Unicode Output:**
```bash
# Generates lines with varying widths including multi-byte characters
python3 -c "
import random, string
for i in range(200000):
    width = random.randint(40, 120)
    line = ''.join(random.choices(string.ascii_letters + string.digits, k=width))
    print(line)
"
```

**Suggested Experiment Protocol:**

1. Start Kitty with a specific `scrollback_lines` setting:
   ```bash
   kitty -o scrollback_lines=50000
   ```
2. In a separate terminal, start the memory monitor:
   ```bash
   bash /tmp/monitor_kitty_memory.sh $(pgrep -f 'kitty$' | head -1)
   ```
3. In the Kitty window, run the output generator:
   ```bash
   seq 1 200000
   ```
4. Observe memory growth in ~5 MB steps every 2048 lines, stopping at `ceil(50000/2048) = 25` segments ≈ 125 MB

### 4.3 Built-in Benchmark Tool

Kitty includes a throughput benchmark tool at `tools/cmd/benchmark/main.go`:

**Usage:**
```bash
kitten @ benchmark --with-scrollback
```

**Key flags** (from `tools/cmd/benchmark/main.go:25-29`):

| Flag | Type | Purpose |
|------|------|---------|
| `Repetitions` | `int` | Number of benchmark iterations |
| `WithScrollback` | `bool` | When true, keeps the main screen buffer (line 59: `Alternate_screen: !opts.WithScrollback`); when false, uses the alternate screen which has no scrollback |
| `Render` | `bool` | When false (default), uses pause/resume rendering protocol to measure raw parsing throughput |

**How it works:**

1. Opens the controlling terminal in raw mode (line 51)
2. Sends `\x1b[?2026h` to pause rendering (line 68: `const pause_rendering = "\x1b[?2026h"`)
3. Writes test data in a loop for the configured number of repetitions (lines 81-90)
4. Sends `\x1b[?2026l` to resume rendering (line 69: `const resume_rendering = "\x1b[?2026l"`)
5. Sends `\x1b[5n` (Device Status Report) multiple times and waits for `\x1b[0n` responses to confirm all data has been processed (lines 95-99)
6. Reports throughput in MB/s

**Measuring with vs. without scrollback:**
```bash
# Without scrollback (uses alternate screen — no history buffer overhead):
kitten @ benchmark

# With scrollback (stays in main screen — exercises history buffer):
kitten @ benchmark --with-scrollback
```

The difference in throughput between these two modes indicates the overhead of history buffer operations (line copy, segment allocation, optional pager history encoding).

### 4.4 Scroll Latency Measurement

**DSR Round-Trip Technique:**

The benchmark tool's technique of sending `\x1b[5n` and timing the `\x1b[0n` response (as seen at `tools/cmd/benchmark/main.go:95`) can be adapted to measure scroll-related latency:

```python
#!/usr/bin/env python3
# Save as: /tmp/measure_scroll_latency.py
# Run INSIDE a Kitty terminal window
# Measures processing latency via DSR round-trip

import sys, os, time, termios, tty, select

fd = sys.stdin.fileno()
old_settings = termios.tcgetattr(fd)

try:
    tty.setraw(fd)

    # Measure baseline latency (no scrollback load)
    latencies = []
    for _ in range(20):
        os.write(sys.stdout.fileno(), b'\x1b[5n')
        start = time.monotonic()
        while True:
            r, _, _ = select.select([fd], [], [], 1.0)
            if r:
                data = os.read(fd, 100)
                if b'\x1b[0n' in data:
                    break
        latencies.append((time.monotonic() - start) * 1000)

    os.write(sys.stdout.fileno(), b'\r\n\x1b[?25h')  # show cursor
    avg = sum(latencies) / len(latencies)
    p99 = sorted(latencies)[int(0.99 * len(latencies))]
    print(f"\r\nDSR round-trip: avg={avg:.2f}ms, p99={p99:.2f}ms\r\n")

finally:
    termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
```

### 4.5 Configuration Experiments

The following `kitty.conf` adjustments can be used to observe different behaviors:

**Experiment 1: Memory Growth with Different Scrollback Sizes**
```conf
# Test with various values and monitor RSS:
scrollback_lines 2000      # Default: ~5 MB steady state
scrollback_lines 10000     # ~25 MB steady state
scrollback_lines 100000    # ~245 MB steady state
```

**Experiment 2: Pager History Growth**
```conf
# Enable pager history and observe ring buffer growth:
scrollback_pager_history_size 100
# → Initial 1 MB ring buffer, grows in 1 MB increments to 100 MB
# Use: Ctrl+Shift+H to view in pager after filling scrollback
```

**Experiment 3: Scroll Responsiveness Tuning**
```conf
# Faster rendering (more CPU, smoother scroll):
repaint_delay 1
# vs. default:
repaint_delay 10

# Faster input processing (more responsive scroll):
input_delay 1
# vs. default:
input_delay 3

# Disable vsync for uncapped rendering:
sync_to_monitor no
```

**Experiment 4: Observe Segment Allocation Under Load**

Combine the memory monitor script with the output generator, using `scrollback_lines=50000`:

1. Expected memory pattern: RSS grows in ~5 MB steps
2. Each step corresponds to one `add_segment()` call
3. After reaching 25 segments (51,200 lines capacity), RSS plateaus
4. Generating additional output beyond 50,000 lines causes no further memory growth

**Experiment 5: Scroll During Heavy Output**

```bash
# In one Kitty window, start generating output:
python3 -c "
import time
for i in range(1000000):
    print(f'Line {i}: ' + 'data ' * 14)
    if i % 10000 == 0:
        time.sleep(0.01)  # Brief pause every 10k lines
"

# While this runs, use Shift+PageUp/PageDown to scroll
# Observe: the view should remain responsive, with the viewed
# content staying stable (scrolled_by auto-adjusts per line 2761
# of kitty/screen.c)
```

---

## Appendix: Key Source File Reference Index

| File | Lines | Description |
|------|-------|-------------|
| `kitty/data-types.h` | 216-221 | `GPUCell` struct (20 bytes) |
| `kitty/data-types.h` | 223-228 | `CPUCell` struct (12 bytes) |
| `kitty/data-types.h` | 231-239 | `LineAttrs` union (1 byte) |
| `kitty/data-types.h` | 262-266 | `HistoryBufSegment` struct |
| `kitty/data-types.h` | 268-272 | `PagerHistoryBuf` struct |
| `kitty/data-types.h` | 282-290 | `HistoryBuf` struct |
| `kitty/history.c` | 15 | `SEGMENT_SIZE = 2048` |
| `kitty/history.c` | 17-29 | `add_segment()` — segment allocation |
| `kitty/history.c` | 36-42 | `segment_for()` — lazy allocation trigger |
| `kitty/history.c` | 66-67 | `initial_pagerhist_ringbuf_sz()` — initial ring buffer size |
| `kitty/history.c` | 69-80 | `alloc_pagerhist()` — pager ring buffer creation |
| `kitty/history.c` | 90-101 | `pagerhist_extend()` — ring buffer growth |
| `kitty/history.c` | 103-114 | `pagerhist_clear()` — ring buffer reset |
| `kitty/history.c` | 116-133 | `create_historybuf()` — constructor |
| `kitty/history.c` | 152-159 | `index_of()` — circular buffer index calculation |
| `kitty/history.c` | 161-176 | `init_line()` — line pointer initialization |
| `kitty/history.c` | 180-182 | `historybuf_init_line()` — public line initialization |
| `kitty/history.c` | 210-216 | `historybuf_clear()` — buffer clear |
| `kitty/history.c` | 218-226 | `pagerhist_write_bytes()` — ring buffer write with growth trigger |
| `kitty/history.c` | 258-273 | `pagerhist_push()` — encode line as UTF-8 ANSI |
| `kitty/history.c` | 275-284 | `historybuf_push()` — circular buffer push logic |
| `kitty/screen.c` | 93-148 | `new_screen_object()` — Screen constructor |
| `kitty/screen.c` | 130 | `alloc_historybuf()` call with `MAX(scrollback, lines)` |
| `kitty/screen.c` | 1552-1567 | `INDEX_UP` macro — line eviction to history |
| `kitty/screen.c` | 1569-1577 | `screen_index()` — cursor-at-bottom scrolling |
| `kitty/screen.c` | 1907-1911 | `dirty_scroll()` — sets `scroll_changed`, deactivates pause |
| `kitty/screen.c` | 1914-1920 | `screen_clear_scrollback()` — clears history buffer |
| `kitty/screen.c` | 2506-2544 | `screen_pause_rendering()` — synchronized update mechanism |
| `kitty/screen.c` | 2738-2797 | `screen_update_cell_data()` — visible-lines-only GPU upload |
| `kitty/screen.c` | 2761 | `scrolled_by` auto-adjustment during concurrent output |
| `kitty/screen.c` | 4091-4118 | `screen_history_scroll()` — scroll amount computation |
| `kitty/screen.h` | 88-170 | `Screen` struct definition |
| `kitty/screen.h` | 91 | `scrolled_by` field |
| `kitty/screen.h` | 101 | `scroll_changed` field |
| `kitty/screen.h` | 159-168 | `paused_rendering` struct |
| `kitty/shaders.c` | 394-452 | `cell_prepare_to_render()` — rendering entry point |
| `kitty/shaders.c` | 418 | `scroll_changed` trigger for GPU data re-upload |
| `kitty/shaders.c` | 608-630 | `draw_scroll_indicator()` — scroll position bar |
| `kitty/state.h` | 36-109 | `Options` struct |
| `kitty/state.h` | 45 | `scrollback_pager_history_size` option |
| `kitty/state.h` | 51 | `repaint_delay, input_delay` timing options |
| `kitty/options/definition.py` | 372 | `scrollback_lines` default: 2000 |
| `kitty/options/definition.py` | 406 | `scrollback_pager_history_size` default: 0 |
| `kitty/options/definition.py` | 866-874 | `repaint_delay` default: 10ms |
| `kitty/options/definition.py` | 878-887 | `input_delay` default: 3ms |
| `kitty/options/utils.py` | 557-561 | `scrollback_lines()` parser — negative → 2^32-1 |
| `kitty/options/utils.py` | 564-566 | `scrollback_pager_history_size()` parser — MB to bytes, max 4GB |
| `kitty/window.py` | 1832-1871 | Scroll action methods |
| `3rdparty/ringbuf/ringbuf.h` | 1-80 | Ring buffer FIFO interface |
| `tools/cmd/benchmark/main.go` | 25-99 | Benchmark tool with `--with-scrollback` flag |
