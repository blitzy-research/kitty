# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive, evidence-based analysis document** that answers specific behavioral questions about the Kitty terminal emulator's scrollback history buffer under stress conditions. The user is not requesting source code changes to the Kitty codebase; instead, they require an investigative report rooted in actual code analysis, with the following research goals:

- **Memory Consumption Under Heavy Output** — Determine precisely how memory is consumed as hundreds of thousands of lines are printed rapidly into the terminal. Quantify per-line and per-segment memory costs using the real data structures (`CPUCell`, `GPUCell`, `LineAttrs`, `HistoryBufSegment`) and the segmented allocation scheme in `kitty/history.c`.
- **Scroll Responsiveness During Concurrent Output** — Analyze whether the terminal remains responsive when the user scrolls back through a large history while new output is still being appended. Identify any prioritization between input processing and data ingestion, citing the threading model in `kitty/child-monitor.c` and the render scheduling in `kitty/shaders.c`.
- **Buffer Boundary Behavior and Allocation Transitions** — Document the exact points at which the buffer's internal storage scheme changes as it grows, including segment allocation thresholds, pager history ring buffer expansion, and the ring buffer wrap-around behavior implemented via the `3rdparty/ringbuf/` library.
- **Observable Measurement Methods** — Describe measurement approaches (temporary scripts, external memory monitoring) that the user could employ to observe these behaviors empirically, without modifying the Kitty repository.

The user explicitly states: *"Temporary scripts may be used for observation and measurement, but the repository itself should remain unchanged."*

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — No Repository Modifications**: The Kitty source repository must not be altered in any way. All analysis must be derived from reading existing source code, and any suggested measurement scripts must be external temporary files.
- **Implementation Rule — SWE-AtlasQnA-Repo**: The project's implementation rules require creating a new markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory in the destination repo. This document must comprehensively answer the posed questions with rationale grounded in the code, no assumptions, and no modifications to any existing files.
- **Evidence-Based Answers Only**: Every claim in the output document must reference specific source files and line ranges in the Kitty codebase. The user requests "actual memory measurements, not just the theory," so the document should include concrete calculations and suggested scripts for empirical validation.
- **Preserve User Requirements Exactly**: User Example: *"If I generate a massive amount of terminal output, say, printing hundreds of thousands of lines rapidly, what happens to memory consumption as the history accumulates?"* — This phrasing should be directly addressed.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the memory consumption question**, we will analyze the `HistoryBuf` struct in `kitty/data-types.h` (lines 282–290), the `SEGMENT_SIZE` constant and `add_segment()` function in `kitty/history.c` (lines 15–29), and compute per-line/per-segment costs using the known sizes of `CPUCell` (12 bytes), `GPUCell` (20 bytes), and `LineAttrs` (1 byte). We will also analyze the `PagerHistoryBuf` ring buffer allocation in `kitty/history.c` (lines 66–101).
- To **answer the scroll responsiveness question**, we will analyze the `screen_history_scroll()` function in `kitty/screen.c` (lines 4091–4118), the `screen_update_cell_data()` rendering path (lines 2738–2790), the `screen_pause_rendering()` mechanism (lines 2506–2544), and the threaded rendering architecture documented in `kitty/child-monitor.c`.
- To **answer the buffer boundary question**, we will trace the `segment_for()` function in `kitty/history.c` (lines 36–42) and the `pagerhist_extend()` function (lines 90–101) to identify exact allocation transition points.
- To **deliver the output**, we will create the file `blitzy/documentation/kitty_815df1e210e0.md` containing the complete analysis.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The scrollback history buffer system spans multiple C source files, Python configuration/binding files, a vendored third-party ring buffer library, test suites, and a Go-based benchmark tool. The following files were identified as directly relevant through exhaustive repository inspection.

**Core History Buffer Implementation:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/history.c` | HistoryBuf type: segmented allocation, ring buffer pager history, push/pop/rewrap operations | Primary — all memory allocation and data structures |
| `kitty/data-types.h` | Struct definitions for `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `CPUCell`, `GPUCell`, `LineAttrs`, `Line`, `LineBuf` | Primary — defines memory layout and sizes |
| `kitty/data-types.c` | Python type registration and initialization for `HistoryBuf` and related types | Supporting — type bootstrapping |
| `3rdparty/ringbuf/ringbuf.h` | Ring buffer FIFO interface: `ringbuf_new`, `ringbuf_capacity`, `ringbuf_bytes_used`, `ringbuf_bytes_free`, `ringbuf_memcpy_into` | Primary — pager history storage |
| `3rdparty/ringbuf/ringbuf.c` | Ring buffer implementation (394 lines) | Primary — allocation and capacity logic |

**Screen Model and Scroll Logic:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/screen.c` | Screen constructor with `alloc_historybuf()`, scroll functions (`screen_history_scroll`, `screen_scroll`, `screen_index`), cell data update pipeline, pause-rendering mechanism | Primary — scroll interaction, rendering during scroll |
| `kitty/screen.h` | `Screen` struct with `historybuf`, `scrolled_by`, `scroll_changed`, `paused_rendering`, `history_line_added_count` fields | Primary — state tracking for scroll behavior |
| `kitty/line-buf.c` | `LineBuf` management: line mapping, indexing, reallocation during resize | Supporting — active screen buffer |
| `kitty/lineops.h` | Line operation macros and helpers | Supporting — line copy/clear during scroll |
| `kitty/rewrap.h` | Generic rewrap algorithm used by both `LineBuf` and `HistoryBuf` | Supporting — resize behavior |

**Rendering Pipeline (Scroll Visualization):**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/shaders.c` | `cell_prepare_to_render()`: decides when to upload cell data to GPU based on `scroll_changed` / `is_dirty` flags; `draw_scroll_indicator()` for scroll position bar | Primary — rendering responsiveness during scroll |
| `kitty/state.h` | `Options` struct with `scrollback_pager_history_size`, `scrollback_fill_enlarged_window`, `repaint_delay`, `input_delay`, `scrollback_indicator_opacity` | Primary — configuration driving buffer behavior |

**Configuration and Options:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/options/definition.py` | Declarative schema for `scrollback_lines` (default: 2000), `scrollback_pager_history_size` (default: 0), `scrollback_fill_enlarged_window`, `repaint_delay` (10ms), `input_delay` (3ms), `sync_to_monitor` | Primary — user-tunable parameters |
| `kitty/options/types.py` | Typed `Options` class: `scrollback_lines: int = 2000`, `scrollback_pager_history_size: int = 0` | Supporting — typed defaults |
| `kitty/options/utils.py` | Parsing: `scrollback_lines()` converts negative to `2^32 - 1`; `scrollback_pager_history_size()` converts MB to bytes with 4GB cap | Primary — boundary logic |

**User Interaction (Scroll Commands):**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty/window.py` | Python-level scroll actions: `scroll_line_up`, `scroll_line_down`, `scroll_page_up`, `scroll_page_down`, `scroll_home`, `scroll_end`, `show_scrollback` | Supporting — user-facing scroll API |
| `kitty/rc/scroll_window.py` | Remote control `scroll_window` command | Supporting — programmatic scroll access |

**Tests:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `kitty_tests/datatypes.py` | `test_historybuf()`: tests HistoryBuf push, rewrap with 3000-line buffer, segment behavior | Supporting — validates buffer behavior |
| `kitty_tests/screen.py` | Scrollback fill tests, pager history tests, scroll-to-mark tests | Supporting — validates scroll interaction |

**Benchmark Tool:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `tools/cmd/benchmark/main.go` | Terminal throughput benchmark with `--with-scrollback` flag; measures data throughput with/without scrollback enabled | Supporting — existing performance measurement |

### 0.2.2 Web Search Research Conducted

No external web search was required for this analysis. All conclusions are derived directly from the source code of the Kitty terminal emulator repository. The codebase is self-contained with comprehensive inline documentation in `kitty/options/definition.py` and clear code structure in the C implementation files.

### 0.2.3 New File Requirements

- **New documentation file to create:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — Comprehensive Q&A analysis document answering all user questions about scrollback history buffer behavior under heavy load, memory consumption patterns, scroll responsiveness, and buffer boundary behavior

No new source files, test files, or configuration files are required since the repository must remain unmodified per user instructions and the SWE-AtlasQnA-Repo implementation rule.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The scrollback history buffer analysis depends on understanding the following packages and libraries that are part of the existing Kitty codebase. No new dependencies need to be added.

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| Vendored (3rdparty) | ringbuf | Public domain (2011, Drew Hess) | Byte-addressable ring buffer FIFO used by `PagerHistoryBuf` for pager history storage |
| System | Python (CPython) | >=3.8 (per `pyproject.toml`) | Runtime for Screen/HistoryBuf Python type bindings and configuration parsing |
| System | Go | 1.22 (per `go.mod`) | Build-time for `tools/cmd/benchmark/main.go` benchmark tool |
| System | OpenGL | 3.3+ (per `kitty/data-types.h` lines 20–25) | GPU rendering pipeline for cell data display during scroll operations |
| Vendored (glfw) | GLFW | 3.4 (customized fork) | Platform windowing, input handling, and OpenGL context management |
| System | FreeType | (system-provided) | Glyph rasterization for rendering text in scrollback lines |

### 0.3.2 Dependency Updates

No dependency updates are required. This task produces a documentation artifact only. The analysis is a read-only investigation of existing code, and the output is a markdown document placed in `blitzy/documentation/`.

### 0.3.3 Import and External Reference Context

For the generated analysis document, the following internal code relationships are relevant but require no modifications:

- `kitty/history.c` includes `3rdparty/ringbuf/ringbuf.h` for pager history ring buffer operations
- `kitty/screen.c` includes `kitty/state.h` (which includes `kitty/screen.h` and `kitty/data-types.h`) for access to `Options` and `HistoryBuf`
- `kitty/shaders.c` reads `screen->scroll_changed`, `screen->scrolled_by`, and `screen->historybuf->count` to determine rendering behavior during scroll
- `kitty/options/definition.py` defines the configuration schema that drives buffer sizing via `scrollback_lines` and `scrollback_pager_history_size`

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

Since this task produces a read-only analysis document, no code modifications are required. However, the analysis must accurately describe the integration relationships between the following components that form the scrollback history buffer system:

**Data Flow: Output → History Buffer → Rendering**

- `kitty/screen.c` — `INDEX_UP` macro (line 1552): When the cursor is at the bottom margin and new output arrives, `linebuf_index()` rotates the line buffer and `historybuf_add_line()` pushes the evicted line into the history buffer. The `history_line_added_count` counter is incremented to coordinate with the rendering pipeline.
- `kitty/history.c` — `historybuf_push()` (line 276): Implements the circular buffer logic. When the buffer is full (`count == ynum`), the oldest line is first pushed to the pager history ring buffer via `pagerhist_push()`, then `start_of_data` advances to overwrite it.
- `kitty/history.c` — `add_segment()` (line 17): Allocates a new `HistoryBufSegment` of 2048 lines. Segments are allocated on-demand via `segment_for()` (line 37), which triggers allocation when `seg_num >= self->num_segments`.
- `kitty/history.c` — `pagerhist_extend()` (line 90): Extends the pager ring buffer when free space is insufficient. Grows in increments of at least 1 MB up to `maximum_size` (configured via `scrollback_pager_history_size`).

**Data Flow: Scroll Input → Display Update**

- `kitty/window.py` — `scroll_line_up()`, `scroll_page_up()`, etc.: User actions call `self.screen.scroll(SCROLL_LINE, True)` which invokes the C-level `screen_history_scroll()`.
- `kitty/screen.c` — `screen_history_scroll()` (line 4091): Adjusts `self->scrolled_by` and calls `dirty_scroll()`, which sets `scroll_changed = true` and calls `screen_pause_rendering()`.
- `kitty/shaders.c` — `cell_prepare_to_render()` (line 394): Checks `screen->scroll_changed` flag. When true, triggers `screen_update_cell_data()` which iterates over visible lines, fetching from `historybuf` for scrolled-back lines and from `linebuf` for current screen lines.
- `kitty/screen.c` — `screen_update_cell_data()` (line 2738): For the top `scrolled_by` lines, calls `historybuf_init_line()` to resolve pointers into the correct segment, then calls `render_line()` for glyph rasterization and `update_line_data()` for GPU upload.

**Threading Model Integration**

- The **I/O Thread** (in `kitty/child-monitor.c`) polls PTY file descriptors, reads child output, and feeds bytes to the VT parser.
- The **Main Thread** runs `parse_input` which processes the byte stream through the VT parser state machine, updating the screen model (including history buffer additions).
- **Frame rendering** occurs on the Main Thread via `cell_prepare_to_render()`, gated by `repaint_delay` (default 10ms) and `input_delay` (default 3ms).
- When scroll input arrives while output is being generated, both operations compete for the Main Thread. The `input_delay` parameter controls batching: pending input is processed before rendering, minimizing latency.

### 0.4.2 Key Interaction Boundaries

| Component A | Component B | Interface | Behavior Under Load |
|-------------|-------------|-----------|---------------------|
| VT Parser (vt-parser.c) | Screen Model (screen.c) | `screen_index()` → `historybuf_add_line()` | Each new line at margin bottom pushes one line to history |
| History Buffer (history.c) | Segment Allocator | `segment_for()` → `add_segment()` | New segment allocated every 2048 lines |
| History Buffer (history.c) | Pager Ring Buffer | `pagerhist_push()` → `pagerhist_write_bytes()` | Oldest lines overflow from HistoryBuf to ring buffer |
| Screen Model (screen.c) | Renderer (shaders.c) | `scroll_changed` flag → `cell_prepare_to_render()` | Scroll triggers GPU data re-upload for visible region |
| Scroll Input (window.py) | Screen Model (screen.c) | `screen_history_scroll()` | Adjusts `scrolled_by`, marks dirty |
| Pause Rendering (screen.c) | Renderer (shaders.c) | `paused_rendering.expires_at` check | Freeze display while batching rapid updates |

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since the repository must remain unchanged per user instructions and the SWE-AtlasQnA-Repo rule, the implementation consists of creating a single new markdown document. No existing files are modified.

**Group 1 — Output Document:**

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — Comprehensive analysis document answering all four user questions with evidence drawn from the source code

**The document must contain the following analytical sections:**

- **Section 1: Memory Consumption Under Heavy Load** — Quantitative analysis of per-line costs, segment allocation thresholds, and total memory for various scrollback sizes
- **Section 2: Scroll Responsiveness During Concurrent Output** — Analysis of threading model, rendering pipeline, and latency characteristics when scrolling during active output
- **Section 3: Buffer Boundary Behavior** — Documentation of allocation transitions at segment boundaries and pager ring buffer expansion events
- **Section 4: Measurement Approaches** — Suggested external scripts and tools for empirical observation

### 0.5.2 Implementation Approach — Analysis Content

The document will be structured to answer each user question with the following evidence-based methodology:

**Memory Consumption Analysis:**

The analysis will compute exact memory costs from the struct definitions in `kitty/data-types.h`:

- `CPUCell`: 12 bytes (verified by `static_assert` at line 228)
- `GPUCell`: 20 bytes (verified by `static_assert` at line 221)
- `LineAttrs`: 1 byte (uint8_t union)
- Per-line cost at 80 columns: `80 × (12 + 20) + 1 = 2,561 bytes ≈ 2.5 KB`
- `SEGMENT_SIZE` is 2048 lines (defined at `kitty/history.c` line 15)
- Per-segment allocation at 80 columns: `80 × 2048 × 12 + 80 × 2048 × 20 + 2048 × 1 = 5,244,928 bytes ≈ 5.00 MB`
- Segments are allocated on-demand: the first segment is created at `HistoryBuf` construction (`create_historybuf()` line 127), and subsequent segments are lazily created by `segment_for()` (line 39) when a line index exceeds the current segment count
- For the default `scrollback_lines=2000`, only 1 segment is needed (2048 ≥ 2000), consuming ~5 MB
- For 100,000 lines (49 segments): ~245 MB
- For negative (infinite) scrollback, `scrollback_lines()` in `kitty/options/utils.py` converts to `2^32 - 1`

**Scroll Responsiveness Analysis:**

The document will explain the following mechanisms:

- Kitty uses a multi-threaded architecture: I/O polling on a dedicated thread, input parsing and rendering on the Main Thread
- `repaint_delay` (default 10ms) and `input_delay` (default 3ms) govern render scheduling
- When `scroll_changed` is set by `dirty_scroll()`, the next render cycle in `cell_prepare_to_render()` (line 418 of `kitty/shaders.c`) re-uploads cell data to the GPU
- The `screen_pause_rendering()` mechanism (line 2506 of `kitty/screen.c`) can freeze the visual state for up to 2000ms when activated by DCS `?2026h`, allowing batched updates
- During scrollback viewing, `screen_update_cell_data()` iterates only over the visible lines (typically 24–50), not the entire history, so scroll rendering cost is O(visible_lines), not O(total_history)
- When scrolled back (`scrolled_by > 0`) and new output arrives, `self->scrolled_by` is adjusted upward (line 2761: `self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count)`) to maintain the user's view position

**Buffer Boundary Analysis:**

The document will describe observable transitions:

- **Segment boundary**: Every 2048th line pushed to history triggers `add_segment()` — a `realloc` of the segment pointer array followed by a single `calloc` of approximately 5 MB (at 80 columns). This is an O(1) operation but involves a system `calloc` call
- **Circular wrap-around**: When `count == ynum`, the buffer wraps via `start_of_data = (start_of_data + 1) % ynum` (line 281 of `kitty/history.c`). This is purely arithmetic — no allocation occurs
- **Pager ring buffer expansion**: When `scrollback_pager_history_size > 0` and the ring buffer runs out of space, `pagerhist_extend()` allocates a new, larger ring buffer (growth increment: at least 1 MB), copies existing data, and frees the old buffer. This occurs at `pagerhist_write_bytes()` (line 223) when `sz > space_in_ringbuf`
- **Pager ring buffer initial allocation**: Starts at `MIN(1MB, maximum_size)` per `initial_pagerhist_ringbuf_sz()` (line 67)

### 0.5.3 Suggested Measurement Scripts (External, Temporary)

The document will include suggested temporary Python and shell scripts for empirical observation, such as:

- A script using `psutil` or `/proc/[pid]/status` to monitor RSS (Resident Set Size) of the Kitty process while a heavy output generator runs
- A script that generates controlled output volumes (e.g., `seq 1 100000` or `python3 -c "for i in range(500000): print(f'Line {i}')"`) to fill the scrollback buffer
- Timing scripts using `time` or monotonic clock measurements to assess scroll input-to-display latency
- Instructions for using the built-in `kitten @ benchmark --with-scrollback` tool for throughput measurement

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Documentation Output:**

- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive Q&A analysis document

**Source Files Analyzed (read-only, no modifications):**

- `kitty/history.c` — Complete file (624 lines): segment allocation, ring buffer pager, push/pop, rewrap, clear
- `kitty/data-types.h` — Complete file (438 lines): `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `CPUCell`, `GPUCell`, `LineAttrs` struct definitions
- `kitty/screen.c` — Key sections: constructor (lines 95–145), `INDEX_UP` macro (lines 1552–1567), `screen_scroll` (lines 1590–1598), `screen_history_scroll` (lines 4091–4118), `screen_update_cell_data` (lines 2738–2790), `screen_pause_rendering` (lines 2506–2544), `screen_clear_scrollback` (lines 1914–1919)
- `kitty/screen.h` — Complete file (289 lines): `Screen` struct with `scrolled_by`, `scroll_changed`, `historybuf`, `paused_rendering`
- `kitty/shaders.c` — Key sections: `cell_prepare_to_render` (lines 394–452), `draw_scroll_indicator` (lines 608–630)
- `kitty/state.h` — `Options` struct (lines 36–109): `scrollback_pager_history_size`, `repaint_delay`, `input_delay`, `scrollback_indicator_opacity`
- `kitty/options/definition.py` — Scrollback section (lines 369–423): `scrollback_lines`, `scrollback_pager_history_size`, `scrollback_fill_enlarged_window`
- `kitty/options/utils.py` — Parsing functions (lines 557–566): `scrollback_lines()`, `scrollback_pager_history_size()`
- `kitty/options/types.py` — Typed defaults (lines 572–574)
- `kitty/line-buf.c` — Line buffer management (lines 1–60): `linebuf_clear`, `linebuf_mark_line_dirty`
- `kitty/rewrap.h` — Generic rewrap algorithm (lines 1–80)
- `kitty/window.py` — Scroll action methods (lines 1831–1870): `scroll_line_up`, `scroll_page_up`, `scroll_home`
- `kitty/rc/scroll_window.py` — Remote control scroll command
- `3rdparty/ringbuf/ringbuf.h` — Ring buffer interface (252 lines)
- `3rdparty/ringbuf/ringbuf.c` — Ring buffer implementation (394 lines)
- `kitty_tests/datatypes.py` — `test_historybuf()` (lines 487–564)
- `kitty_tests/screen.py` — Scrollback and pager history tests (lines 281–904)
- `tools/cmd/benchmark/main.go` — Benchmark tool with `--with-scrollback` option

**Analysis Topics In Scope:**

- Quantitative memory footprint calculations for `HistoryBuf` at various `scrollback_lines` settings
- Per-segment allocation costs and thresholds
- Pager ring buffer sizing, growth, and maximum capacity
- Rendering pipeline behavior when `scroll_changed` is true
- Threading model impact on scroll responsiveness
- `repaint_delay` and `input_delay` effects on perceived latency
- `scrolled_by` adjustment during concurrent output
- Observable allocation events at segment boundaries
- Circular buffer wrap-around behavior
- Suggested external measurement methodologies

### 0.6.2 Explicitly Out of Scope

- **Any modification to existing Kitty source files** — explicitly prohibited by user instructions
- **Any new source code files in the Kitty repository** — only the analysis document in `blitzy/documentation/` is created
- **GPU rendering internals beyond scroll-related paths** — shader details for non-scroll rendering (background images, borders, tint) are not analyzed
- **Font pipeline and glyph rasterization performance** — outside the scrollback buffer scope
- **Platform-specific GLFW behavior** — windowing backend differences not relevant to buffer behavior
- **Network/remote control performance** — `scroll_window` RC command is noted but not performance-analyzed
- **Alternative terminal emulator comparisons** — only Kitty's implementation is analyzed
- **Shell integration and prompt marking** — OSC 133 markers mentioned only where they intersect with scrollback (scroll-to-prompt)
- **Graphics protocol interaction with scrollback** — image placeholder handling during scroll is noted but not deeply analyzed
- **Configuration live-reload impact on scrollback** — the docs note that `scrollback_lines` changes only affect new windows, not existing ones

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules are explicitly emphasized by the user and the project configuration:

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the question(s) posed in the prompt. Provide thinking and rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Do not add any other code in the source repository besides the requested document. Place the generated document in the `blitzy/documentation` directory.

- **No Repository Modifications**: The user states *"the repository itself should remain unchanged."* This means zero modifications to any file under the Kitty source tree. Only the `blitzy/documentation/kitty_815df1e210e0.md` file is created.

- **Evidence-Based Analysis**: All claims about buffer behavior must be traceable to specific source code locations. The user requests "actual memory measurements, not just understand the theory" — the document must include concrete numeric calculations derived from struct sizes and allocation constants found in the code.

- **Temporary Scripts Are Acceptable**: The user permits "temporary scripts for observation and measurement." These should be described in the document as suggested external tools but must not be committed to the repository.

### 0.7.2 Code-Derived Constraints

From analysis of the codebase, the following constraints apply to the accuracy of the analysis:

- Memory calculations must use the verified struct sizes: `CPUCell` = 12 bytes (static_assert at `kitty/data-types.h:228`), `GPUCell` = 20 bytes (static_assert at `kitty/data-types.h:221`), `LineAttrs` = 1 byte
- Segment size is fixed at compile time: `SEGMENT_SIZE = 2048` (`kitty/history.c:15`)
- Scrollback lines default is 2000 (`kitty/options/types.py:572`), and negative values convert to `2^32 - 1` (`kitty/options/utils.py:560`)
- Pager history default is 0 (disabled) and maximum is `4096 * 1024 * 1024 - 1` bytes (~4GB) (`kitty/options/utils.py:566`)
- The ring buffer initial allocation is `MIN(1MB, maximum_size)` (`kitty/history.c:67`)
- Ring buffer growth increment is at least 1 MB (`kitty/history.c:93`)
- The `scrollback_lines` option only affects newly created windows, not existing ones (per documentation in `kitty/options/definition.py:379–380`)

## 0.8 References

### 0.8.1 Codebase Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Root-Level Exploration:**

- Repository root (`""`) — folder contents retrieved, establishing project structure as the Kitty terminal emulator
- `pyproject.toml` — Confirmed `requires-python = ">=3.8"`, mypy/ruff configuration
- `go.mod` — Confirmed Go 1.22 requirement
- `setup.py` — Build system entry point, Python version validation

**Core History Buffer Files (Primary Analysis):**

- `kitty/history.c` — Complete file (624 lines): segmented HistoryBuf implementation, PagerHistoryBuf ring buffer, segment allocation (`add_segment`, `segment_for`), push/pop operations, pager history extension, rewrap logic
- `kitty/data-types.h` — Complete file (438 lines): Struct definitions for `HistoryBuf` (lines 282–290), `HistoryBufSegment` (lines 262–266), `PagerHistoryBuf` (lines 268–272), `CPUCell` (lines 223–228), `GPUCell` (lines 216–221), `LineAttrs` (lines 231–239), `LineBuf` (lines 252–260), `Line` (lines 241–249)
- `3rdparty/ringbuf/ringbuf.h` — Ring buffer FIFO interface (252 lines)
- `3rdparty/ringbuf/ringbuf.c` — Confirmed 394 lines of implementation

**Screen and Scroll Logic Files:**

- `kitty/screen.c` — Multiple sections retrieved: constructor (lines 95–145), INDEX_UP macro (lines 1552–1567), screen_scroll (lines 1590–1634), screen_clear_scrollback (lines 1914–1920), screen_move_into_scrollback (lines 1925–1941), screen_history_scroll (lines 4091–4134), screen_update_cell_data (lines 2738–2790), screen_pause_rendering (lines 2506–2544), screen_update_only_line_graphics_data (lines 2709–2735)
- `kitty/screen.h` — Complete file (289 lines): Screen struct definition, ScrollType enum, Selection types
- `kitty/line-buf.c` — Opening section (lines 1–60): LineBuf management functions

**Rendering Pipeline Files:**

- `kitty/shaders.c` — Key sections: cell_prepare_to_render (lines 394–452), draw_scroll_indicator (lines 608–630)
- `kitty/state.h` — Complete file (401 lines): Options struct, GlobalState, OSWindow, Window, Tab definitions

**Configuration Files:**

- `kitty/options/definition.py` — Scrollback section (lines 369–423), performance section (lines 863–899)
- `kitty/options/utils.py` — Parsing functions (lines 555–566)
- `kitty/options/types.py` — Grep confirmed defaults at lines 572, 574

**User Interface and Scroll Commands:**

- `kitty/window.py` — Opening section (lines 1–50) and scroll methods via grep (lines 1831–1870)
- `kitty/rc/scroll_window.py` — Grep confirmed scroll command at lines 62, 79
- `kitty/rewrap.h` — Opening section (lines 1–80): Generic rewrap algorithm

**Test Files:**

- `kitty_tests/datatypes.py` — Grep confirmed test_historybuf at line 487 with large buffer tests
- `kitty_tests/screen.py` — Grep confirmed scrollback tests across lines 281–904

**Benchmark Tool:**

- `tools/cmd/benchmark/main.go` — Opening section (lines 1–100): Benchmark with `--with-scrollback` flag, pause/resume rendering protocol

**Technical Specification Sections Retrieved:**

- Section 2.1 — Feature Catalog: Confirmed F-018 (Scrollback Buffer) as High-priority completed feature
- Section 4.3 — Terminal I/O Pipeline: VT parser dispatch, GPU rendering pipeline, performance architecture
- Section 5.2 — Component Details: VT Parser/Screen Model, GPU Rendering Pipeline, Child Monitor threading, Configuration System

### 0.8.2 Attachments and External Resources

- **No attachments** were provided by the user
- **No Figma URLs** were provided
- **No external URLs** were referenced
- **No environment files** were provided in `/tmp/environments_files/`
- **Branch name**: `kitty_815df1e210e0` — used for the output document filename per the SWE-AtlasQnA-Repo rule

