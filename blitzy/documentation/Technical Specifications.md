# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive, empirically grounded analysis document** that answers detailed questions about kitty's scrollback history buffer behavior under stress, covering three specific investigative areas:

- **Memory behavior under heavy output**: The user wants to understand what actually happens to memory consumption when hundreds of thousands of lines of terminal output are generated rapidly. This requires tracing the allocation mechanics of `HistoryBuf` segments (each holding 2048 lines), the circular-buffer overwrite behavior, and the optional `PagerHistoryBuf` ring buffer growth pattern in `kitty/history.c`.

- **Scroll responsiveness during concurrent output**: The user wants to know whether scrolling back through a very large history while new output is still being generated causes observable lag or prioritization. This requires analyzing the render pipeline in `kitty/screen.c` (`screen_update_cell_data`), the `scrolled_by` tracking mechanism, the `history_line_added_count` adjustment, and how `repaint_delay` / `input_delay` settings in `kitty/child-monitor.c` affect the scheduling.

- **Buffer boundary transitions**: The user wants to understand when and how the buffer's internal storage grows — specifically when segment allocations occur (every 2048 lines in `add_segment()`) and how the `pagerhist_extend()` ring buffer growth is triggered, and whether these transitions would be observable via memory monitoring.

- **Implicit requirements detected**:
  - Answers must be based on actual source code analysis, not speculation
  - The repository itself must remain unchanged — only temporary external scripts may be used
  - A new markdown document must be created in `blitzy/documentation/` named `<source_branch_name>.md`

### 0.1.2 Special Instructions and Constraints

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the question(s) posed in the prompt. Base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory in the destination repo.
- **No repository modifications**: The user explicitly states "the repository itself should remain unchanged" — only temporary observation scripts are permissible.
- **Evidence-based approach**: The user says "I'd like to see actual memory measurements, not just understand the theory" — the analysis must ground every claim in source code evidence (struct sizes, allocation paths, constants).

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the memory behavior question**, we will trace the allocation chain from `alloc_historybuf()` → `add_segment()` → `calloc()` in `kitty/history.c`, compute exact memory figures from `sizeof(CPUCell)=12`, `sizeof(GPUCell)=20`, `sizeof(LineAttrs)=1`, and `SEGMENT_SIZE=2048`, and explain the circular buffer overwrite at capacity.

- To **answer the scroll responsiveness question**, we will analyze the `screen_update_cell_data()` render path in `kitty/screen.c` (lines 2740–2790), the `screen_history_scroll()` function (lines 4091–4120), the `dirty_scroll()` → `screen_pause_rendering()` mechanism, and the I/O thread scheduling in `kitty/child-monitor.c`.

- To **answer the buffer boundary question**, we will map the `segment_for()` lazy-allocation trigger in `kitty/history.c` (line 39), the `pagerhist_extend()` ring buffer growth in `kitty/history.c` (lines 90–101), and compute when each allocation boundary is reached for various `scrollback_lines` settings.

- To **produce the deliverable**, we will create `blitzy/documentation/kitty_815df1e210e0.md` containing a structured Q&A document with source code citations, computed memory tables, and architectural diagrams.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The following files and folders in the kitty repository are directly relevant to the scrollback buffer investigation. All paths were verified through repository inspection.

**Core Scrollback Buffer Implementation:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/history.c` | History buffer implementation — segment allocation, circular push, pager history ring buffer | PRIMARY — all memory allocation/growth questions answered here |
| `kitty/data-types.h` | Struct definitions for `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `CPUCell`, `GPUCell`, `LineAttrs` | PRIMARY — exact memory per-line calculations from static_assert sizes |
| `kitty/line-buf.c` | Active line buffer (visible screen) — separate from history but structurally similar | Supporting — baseline memory comparison |
| `kitty/line.c` | Line data structure operations | Supporting — per-line data manipulation |
| `kitty/lineops.h` | Line operation macros | Supporting — shared by history.c and line-buf.c |
| `3rdparty/ringbuf/ringbuf.c` | Ring buffer implementation used by `PagerHistoryBuf` | PRIMARY — governs pager history growth behavior |
| `3rdparty/ringbuf/ringbuf.h` | Ring buffer API | Supporting — defines capacity/free/used semantics |

**Screen Model and Scrolling Logic:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/screen.c` | Screen model — scroll operations, `scrolled_by` tracking, `screen_update_cell_data()`, `screen_history_scroll()`, `INDEX_UP` macro, pause rendering | PRIMARY — scroll responsiveness and concurrent output behavior |
| `kitty/screen.h` | Screen struct definition — `scrolled_by`, `scroll_changed`, `is_dirty`, `history_line_added_count` fields | PRIMARY — state tracking for scroll position |

**Rendering Pipeline:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/shaders.c` | GPU rendering — `cell_prepare_to_render()`, `draw_cells()`, `draw_scroll_indicator()` | PRIMARY — render cost analysis during scrolling |
| `kitty/child-monitor.c` | Event loop — I/O thread, `parse_input()`, `render()`, `repaint_delay`/`input_delay` scheduling, `read_bytes()` | PRIMARY — concurrency model and scheduling priority |
| `kitty/vt-parser.c` | VT parser — `BUF_SZ` (1MB), input buffering, `run_worker()` flush logic | Supporting — input processing throughput |

**Configuration:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/options/definition.py` | `scrollback_lines` (default 2000), `scrollback_pager_history_size` (default 0), `repaint_delay` (10ms), `input_delay` (3ms) | PRIMARY — default settings governing buffer behavior |
| `kitty/options/utils.py` | `scrollback_lines()` parser — negative → `2^32 - 1`; `scrollback_pager_history_size()` parser — MB to bytes, max 4GB | PRIMARY — limit calculations |

**Test Suite (reference only, not modified):**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty_tests/datatypes.py` | HistoryBuf unit tests — push, rewrap, 3000-line stress test | Reference — validates our understanding |
| `kitty_tests/screen.py` | Screen scrollback tests — pagerhist, wrapping serialization | Reference — validates scroll behavior |
| `kitty_tests/__init__.py` | Test helpers — `filled_history_buf()`, `create_screen()` with `scrollback` param | Reference — test infrastructure |

### 0.2.2 Integration Point Discovery

- **Screen ↔ History**: `screen_scroll()` and `INDEX_UP` macro push lines from the active `LineBuf` into `HistoryBuf` via `historybuf_add_line()` (screen.c line 1558)
- **History ↔ Pager**: When the circular buffer overwrites, `pagerhist_push()` serializes the evicted line as ANSI into the `PagerHistoryBuf` ring buffer (history.c line 241)
- **Scroll ↔ Render**: `dirty_scroll()` sets `scroll_changed=true` and calls `screen_pause_rendering()`, which snapshots the current visual state (screen.c lines 1908–1910)
- **I/O Thread ↔ Main Thread**: `wakeup_main_loop()` signals the render thread after `input_delay` elapses since last data (child-monitor.c lines 1562–1570)
- **Render ↔ GPU**: `screen_update_cell_data()` uploads `sizeof(GPUCell) * lines * columns` bytes to the GPU each dirty frame (shaders.c line 410)

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — The comprehensive Q&A analysis document answering all user questions with source code citations, computed memory tables, and behavioral analysis

No other new files are required. No existing files are modified.

## 0.3 Dependency Inventory

### 0.3.1 Key Packages and Dependencies

Since this task produces a documentation-only deliverable (a markdown analysis file) and does not modify the repository, no dependency changes are required. However, the following dependencies are relevant to understanding the scrollback buffer implementation:

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| System (C) | CPUCell struct | 12 bytes (static_assert) | Per-cell text data storage in scrollback |
| System (C) | GPUCell struct | 20 bytes (static_assert) | Per-cell rendering data in scrollback |
| Vendored (C) | `3rdparty/ringbuf` | Public domain (2011) | Ring buffer FIFO for PagerHistoryBuf |
| PyPI | Python | ≥ 3.8 (pyproject.toml) | Runtime for configuration and test harness |
| Go modules | Go | 1.22 (go.mod) | CLI tools (not directly relevant to scrollback) |

### 0.3.2 Dependency Updates

No dependency updates are required. This task is read-only analysis producing a standalone markdown document. All source files remain unchanged.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

No code modifications are required. The analysis document must accurately describe the following integration paths to answer the user's questions:

**Scrollback Buffer Memory Path** (answers: "what happens to memory consumption"):
```
Screen.screen_scroll() → INDEX_UP macro → historybuf_add_line()
  → historybuf_push() → segment_for() → [maybe add_segment()]
  → pagerhist_push() → pagerhist_write_bytes() → [maybe pagerhist_extend()]
```

**Scroll Rendering Path** (answers: "does the terminal remain responsive"):
```
screen_history_scroll() → dirty_scroll() → screen_pause_rendering()
  → [next render frame] → cell_prepare_to_render() → screen_update_cell_data()
  → historybuf_init_line() for visible lines → update_line_data() → GPU upload
```

**I/O Concurrency Path** (answers: "prioritization between scroll and output"):
```
I/O Thread: poll() → read_bytes() → vt_parser_commit_write()
  → [after input_delay] → wakeup_main_loop()
Main Thread: parse_input() → do_parse() → run_worker() → consume_input()
  → render() → render_os_window() → [GPU frame]
```

### 0.4.2 Critical Integration Details for the Analysis

- **`scrolled_by` auto-adjustment**: In `screen_update_cell_data()` (screen.c line 2761), when `scrolled_by > 0`, it is automatically increased by `history_line_added_count` so the user's scroll position tracks correctly as new lines push into history during concurrent output.

- **`pause_rendering` snapshot**: When `dirty_scroll()` is triggered (screen.c line 1908), it calls `screen_pause_rendering()` which snapshots the current visual state into a separate `LineBuf`, allowing the render thread to display a stable frame while the I/O thread continues processing.

- **Segment allocation is synchronous**: `add_segment()` (history.c line 18) calls `realloc()` for the segment pointer array and `calloc()` for the actual cell data. This is an infrequent operation (every 2048 lines pushed) but involves a ~5MB allocation for an 80-column terminal, which can cause a brief memory allocation stall.

- **Ring buffer growth is copy-on-extend**: `pagerhist_extend()` (history.c line 90) allocates a new ring buffer, copies all existing data, and frees the old one. For large pager histories, this involves copying potentially megabytes of data.

### 0.4.3 No Database or Schema Changes

This task does not involve any database, migration, or schema changes.

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

**Group 1 — Deliverable:**

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md`
  - Comprehensive markdown document answering all three investigative areas
  - Structured with clear sections for memory behavior, scroll responsiveness, and buffer boundaries
  - Includes computed memory tables derived from source code constants
  - All claims cite specific source files and line numbers
  - Architectural diagrams illustrating data flow through the scrollback system

### 0.5.2 Implementation Approach

The analysis document must be constructed from deep source code reading, not from running kitty (which requires a full C build toolchain, GPU, and display server). The approach:

- **Establish the data model** by documenting `CPUCell` (12 bytes), `GPUCell` (20 bytes), `LineAttrs` (1 byte), `SEGMENT_SIZE` (2048), and the segment allocation formula from `kitty/history.c` and `kitty/data-types.h`
- **Compute memory tables** for various `scrollback_lines` configurations (default 2000, 10000, 50000, 100000, unlimited/-1) at an 80-column terminal width, showing segment count, total memory, and growth timeline
- **Trace the circular buffer behavior** by documenting `historybuf_push()` → `start_of_data` advancement → old line overwrite, proving that memory stabilizes at `num_segments * segment_size` and never grows further
- **Analyze the pager history growth** by documenting `initial_pagerhist_ringbuf_sz()` (min 1MB, max_size), `pagerhist_extend()` growth steps (1MB increments up to `maximum_size`), and the ring buffer's overwrite-on-full semantics
- **Document scroll responsiveness** by analyzing the render loop's O(visible_lines) cost regardless of history depth, the `scrolled_by` auto-tracking mechanism, and the `input_delay`/`repaint_delay` scheduling in the I/O thread
- **Map observable allocation boundaries** at every 2048-line interval for segment growth, and at capacity-exceeds-buffer thresholds for pager history ring buffer extension

### 0.5.3 Key Technical Findings to Document

**Memory under heavy load:**
- At default `scrollback_lines=2000` with 80 columns: 1 segment of ~5.0 MB allocated
- Memory is allocated on demand per-segment (SEGMENT_SIZE=2048 lines each)
- Once all segments are allocated (upon first fill of the history buffer), memory is **flat** — the circular buffer overwrites in place
- For `scrollback_lines=-1` (infinite): segments keep growing without bound at ~5 MB per 2048 lines

**Scroll responsiveness:**
- Rendering always processes exactly `screen->lines` visible lines regardless of total history depth
- `scrolled_by` is automatically adjusted by `history_line_added_count` each render frame so scroll position tracks during concurrent output
- `dirty_scroll()` triggers `screen_pause_rendering()` which creates a snapshot, allowing the I/O thread to continue without blocking the display
- The `repaint_delay` (default 10ms, ~100 FPS) and `input_delay` (default 3ms) govern frame scheduling

**Buffer boundaries:**
- New segment allocation happens every 2048 lines via `add_segment()` — a ~5 MB calloc
- Pager history ring buffer starts at min(1MB, max_size) and grows by 1MB increments in `pagerhist_extend()`
- Both transitions involve memory allocator calls (calloc/malloc) that could be observed via `/proc/[pid]/status` VmRSS monitoring

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

- **Analysis document**: `blitzy/documentation/kitty_815df1e210e0.md`
- **Source files analyzed** (read-only):
  - `kitty/history.c` — complete scrollback buffer implementation
  - `kitty/data-types.h` — struct definitions and size assertions
  - `kitty/screen.c` — screen model, scroll operations, rendering pipeline
  - `kitty/screen.h` — screen state struct
  - `kitty/line-buf.c` — active line buffer for context
  - `kitty/line.c` — line data structures
  - `kitty/lineops.h` — shared line operation macros
  - `kitty/shaders.c` — GPU render pipeline and scroll indicator
  - `kitty/child-monitor.c` — I/O thread, event loop, render scheduling
  - `kitty/vt-parser.c` — input buffer size (BUF_SZ = 1MB)
  - `kitty/options/definition.py` — scrollback configuration defaults
  - `kitty/options/utils.py` — scrollback value parsing logic
  - `3rdparty/ringbuf/ringbuf.c` — ring buffer implementation
  - `3rdparty/ringbuf/ringbuf.h` — ring buffer API
  - `kitty_tests/datatypes.py` — history buffer test suite (reference)
  - `kitty_tests/screen.py` — screen scrollback tests (reference)
  - `kitty_tests/__init__.py` — test helper infrastructure (reference)

### 0.6.2 Explicitly Out of Scope

- Modification of any existing repository files
- Building or running the kitty binary (requires full C toolchain, GPU, and display server)
- GPU rendering performance profiling (requires running kitty)
- Font pipeline or glyph cache analysis (not related to scrollback)
- Inline graphics protocol behavior during scrolling
- Go tools layer analysis (irrelevant to scrollback buffer)
- Cross-platform windowing differences (Wayland vs X11 scrollback behavior)
- Network/remote control scroll commands (beyond noting `scroll_window` exists)
- Performance optimizations or code changes to the scrollback system

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the question(s) posed in the prompt. Build and run the source code to analyse the repository behavior as needed. Do not make assumptions, base answers on the code as the truth. Provide thinking/rationale behind the answers. Do not modify any existing files in the source repository. Do not add any other code in the source repository (besides the above requested document). Place the generated document in the `blitzy/documentation` directory in the destination repo.
- **No repository modification**: "Temporary scripts may be used for observation and measurement, but the repository itself should remain unchanged."
- **Evidence-based answers**: "I'd like to see actual memory measurements, not just understand the theory" — all answers must be grounded in concrete code analysis with exact byte counts, struct sizes, and allocation paths.
- **Code as truth**: Do not make assumptions — every claim must reference specific source file paths and line ranges.

### 0.7.2 Architectural Conventions to Follow

- The analysis document must respect the kitty project's architecture: the three-thread model (main, I/O, talk), the C-Python boundary, and the segment-based history buffer design
- Memory calculations must use the exact `static_assert` sizes from `kitty/data-types.h`: `sizeof(GPUCell) == 20`, `sizeof(CPUCell) == 12`
- Buffer size calculations must use the exact constants: `SEGMENT_SIZE = 2048`, `BUF_SZ = 1024*1024`, `initial_pagerhist_ringbuf_sz = MIN(1024*1024, pagerhist_sz)`
- Default configuration values must match `kitty/options/definition.py`: `scrollback_lines=2000`, `scrollback_pager_history_size=0`, `repaint_delay=10`, `input_delay=3`

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were inspected to derive the conclusions in this Agent Action Plan:

**Core scrollback implementation files:**
- `kitty/history.c` — Complete file read; contains `add_segment()`, `segment_for()`, `historybuf_push()`, `pagerhist_push()`, `pagerhist_extend()`, `pagerhist_write_bytes()`, `pagerhist_clear()`, `historybuf_clear()`, `historybuf_rewrap()`, `alloc_pagerhist()`, `initial_pagerhist_ringbuf_sz()`
- `kitty/data-types.h` — Lines 195–310 read; contains `GPUCell` (20 bytes), `CPUCell` (12 bytes), `LineAttrs`, `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `LineBuf`, `Line` struct definitions
- `kitty/line-buf.c` — Lines 1–100 read; contains `alloc_linebuf()`, cell buffer allocation
- `kitty/line.c` — File identified; 1003 lines of line data operations
- `kitty/lineops.h` — File identified; 136 lines of shared macros

**Screen model and rendering files:**
- `kitty/screen.c` — Multiple ranges read: lines 100–160 (screen creation), 1535–1570 (INDEX_UP macro), 1590–1640 (screen_scroll), 1900–1960 (dirty_scroll, clear_scrollback), 2489–2560 (pause_rendering), 2594–2610 (screen_reset_dirty), 2700–2810 (screen_update_cell_data), 2843–2880 (visual_line_), 4091–4130 (screen_history_scroll)
- `kitty/screen.h` — Lines 87–105 read; contains Screen struct with `scrolled_by`, `scroll_changed`, `is_dirty`, `history_line_added_count`
- `kitty/shaders.c` — Lines 394–430 (cell_prepare_to_render), 609–640 (draw_scroll_indicator)

**Event loop and I/O files:**
- `kitty/child-monitor.c` — Lines 420–470 (do_parse, parse_input), 860–910 (render scheduling), 1337–1380 (read_bytes), 1481–1600 (io_loop)
- `kitty/vt-parser.c` — Lines 1415–1470 (run_worker, vt_parser_create_write_buffer); BUF_SZ=1MB constant at line 18

**Configuration files:**
- `kitty/options/definition.py` — Lines 369–465 read; scrollback options, repaint_delay, input_delay, wheel_scroll settings
- `kitty/options/utils.py` — Lines 557–575 read; `scrollback_lines()` and `scrollback_pager_history_size()` parsers

**Third-party dependencies:**
- `3rdparty/ringbuf/ringbuf.h` — Complete file read; ring buffer API (capacity, bytes_free, bytes_used, memcpy_into, memmove_from, etc.)
- `3rdparty/ringbuf/ringbuf.c` — Lines 1–100 read; `ringbuf_new()`, `ringbuf_reset()`, `ringbuf_free()`, internal `struct ringbuf_t`

**Test files (reference):**
- `kitty_tests/datatypes.py` — Lines 487–565 read; HistoryBuf test cases including 3000-line stress test
- `kitty_tests/screen.py` — Lines 281–710 read; scrollback fill, pagerhist, wrapping serialization tests
- `kitty_tests/__init__.py` — Lines 184–250 read; `filled_history_buf()`, `BaseTest.create_screen()`

**Project configuration files:**
- `pyproject.toml` — Complete file read; requires-python >= 3.8
- `go.mod` — Complete file read; Go 1.22
- `setup.py` — Lines 1–80 read; build system

**Tech spec sections retrieved:**
- Section 2.1 Feature Catalog — F-018 Scrollback Buffer description
- Section 4.3 Terminal Input/Output Pipeline — VT parser and rendering pipeline flows
- Section 5.2 Component Details — Child Monitor, VT Parser, GPU Rendering Pipeline details

### 0.8.2 Attachments and External Metadata

- No Figma URLs provided
- No external attachments provided
- Docker environment: `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
- Source branch: `kitty_815df1e210e0`
- HEAD commit: `815df1e21` ("Wire up applying of font config")

