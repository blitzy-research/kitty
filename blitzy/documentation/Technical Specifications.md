# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative technical document** that comprehensively answers four interrelated questions about the Kitty terminal emulator's scrollback history buffer behavior under heavy load conditions. The request falls into the category **Create new documentation** and the documentation type is **Technical Investigation / Architecture Deep-Dive**.

The four documentation requirements, restated with enhanced clarity, are:

- **Memory Consumption Profiling** — Document precisely how memory consumption grows as the scrollback history buffer accumulates hundreds of thousands of lines of rapidly-generated terminal output. The user explicitly requires *actual memory measurements*, not theoretical explanations alone. The document must present concrete numbers derived from inspecting the C-level data structures in `kitty/history.c` and `kitty/data-types.h`.
- **Scroll Responsiveness Under Concurrent Output** — Investigate and document whether the terminal remains responsive when the user scrolls back through a very large history while new output is still being generated. Characterize any observable latency or lag between scroll input and display update, and identify any signs of the system prioritizing one operation (I/O, parsing, rendering) over another.
- **Buffer Growth Boundaries and Allocation Transitions** — Identify and document the specific thresholds at which the buffer's behavior changes as it grows. This includes the segmented allocation model (segment boundaries at `SEGMENT_SIZE = 2048` lines), ring buffer expansion in the pager history subsystem, and the transition from in-buffer storage to pager-history serialization when the main circular buffer is full.
- **Observation Methodology** — Provide working temporary scripts and commands for memory monitoring and measurement. The user permits temporary observation scripts but mandates that the repository itself must remain unchanged.

### 0.1.2 Special Instructions and Constraints

- **Repository immutability**: The repository must remain unchanged. All measurement and observation activity must use temporary, external scripts that do not modify any tracked file.
- **Implementation rule — SWE-AtlasQnA-Repo**: The project rule requires creating a new markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory. The document must provide thinking and rationale behind all answers, base conclusions on the code as truth, and must not modify any existing files in the source repository.
- **Evidence-based answers**: Answers must be derived from the actual codebase (`kitty/history.c`, `kitty/data-types.h`, `kitty/screen.c`, `kitty/child-monitor.c`, etc.) rather than from assumptions.
- **Measurement emphasis**: The user explicitly asks for "actual memory measurements, not just understand the theory." The document must include both the theoretical memory model derived from the code and methodology for obtaining live measurements.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document memory consumption behavior**, we will create a section analyzing the segmented allocation model in `kitty/history.c` (lines 15–29), computing per-segment memory cost from `CPUCell` (12 bytes), `GPUCell` (20 bytes), and `LineAttrs` (1 byte) sizes defined in `kitty/data-types.h`, and describing the on-demand allocation pattern in `segment_for()` (line 36–42). The document will also cover the `PagerHistoryBuf` ring buffer growth path in `pagerhist_extend()` (lines 90–101).
- To **document scroll responsiveness**, we will analyze the three-thread architecture in `kitty/child-monitor.c` (I/O thread, main thread, talk thread), the `screen_history_scroll()` function in `kitty/screen.c` (line 4091), the `screen_update_cell_data()` render path (line 2740+), and the `repaint_delay`/`input_delay` timing parameters.
- To **document buffer boundaries**, we will trace the `add_segment()` allocation trigger in `kitty/history.c` (lines 17–29), the circular buffer overflow path in `historybuf_push()` (lines 276–284), and the pager-history spill logic in `pagerhist_push()` (lines 258–273).
- To **provide observation methodology**, we will design temporary measurement scripts using standard tools (`/proc/[pid]/status`, Python `resource` module, `time` commands) that can run alongside Kitty without modifying the repository.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are surfaced:

- **Configuration context**: The document should explain the `scrollback_lines` (default 2000), `scrollback_pager_history_size` (default 0, in MB), and `scrollback_fill_enlarged_window` configuration options from `kitty/options/definition.py` (lines 372–423), as these directly govern buffer sizing.
- **Per-column width scaling**: Memory consumption scales linearly with terminal column width (`xnum`). The document should present calculations for common terminal widths (80, 120, 200 columns).
- **Rewrap overhead during resize**: When the terminal is resized, `historybuf_rewrap()` in `kitty/history.c` (line 594) allocates segments in the destination buffer. This transient memory spike should be documented.
- **Rendering thread interaction**: The GPU rendering pipeline (documented in tech spec section 4.3.3) operates on a separate thread from input processing. The document should explain how `scrolled_by` state in `kitty/screen.h` (line 91) governs which lines are drawn from the history buffer versus the active line buffer during rendering.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation tree under `docs/` with configuration in `docs/conf.py`, using the Furo theme and multiple extensions. The project has an established documentation framework covering user manuals, protocol specifications, and performance guides.

- **Documentation framework**: Sphinx (version pinned via `docs/requirements.txt`: `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`)
- **Documentation generator configuration**: `docs/conf.py` — central Sphinx configuration with custom lexers, roles, and generated-document writers
- **Build system**: `docs/Makefile` — standard docs build driver with `help`, generic Sphinx targets, and `develop-docs` live-preview via `sphinx-autobuild`
- **Existing performance documentation**: `docs/performance.rst` — covers throughput benchmarks, CPU usage, keyboard-to-screen latency, and profiling instructions using gperftools, but does **not** address scrollback memory behavior or scroll responsiveness under concurrent output
- **Existing configuration documentation**: `docs/conf.rst` — documents user-facing configuration options including scrollback settings
- **Diagram tools**: Mermaid diagrams used extensively in the tech spec; the Sphinx docs themselves do not use Mermaid but rely on reStructuredText directives
- **API documentation tools**: No JSDoc/Sphinx autodoc for the C extension layer; the `fast_data_types` Python bindings are typed via `kitty/fast_data_types.pyi`

Critical finding: **No existing document covers the scrollback buffer's memory behavior, allocation model, or responsiveness under heavy concurrent load.** The `docs/performance.rst` file focuses on throughput (MB/s), CPU usage, and keyboard-to-screen latency but does not discuss scrollback memory consumption, buffer allocation boundaries, or scroll responsiveness with ongoing output.

### 0.2.2 Repository Code Analysis for Documentation

The following source files were examined to understand the scrollback buffer subsystem:

**Core scrollback implementation:**
- `kitty/history.c` — The authoritative implementation of `HistoryBuf`, segmented storage, pager-history ring buffer, ANSI serialization, and rewrap logic. Contains 625 lines of C code.
- `kitty/data-types.h` — Defines `CPUCell` (12 bytes), `GPUCell` (20 bytes), `LineAttrs` (1 byte), `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, and `LineBuf` structures.
- `kitty/line-buf.c` — Implements the `LineBuf` type (active screen buffer) with parallel CPU/GPU cell storage and logical-to-physical row mapping.
- `kitty/lineops.h` — Shared inline primitives for line manipulation across screen, scrollback, and text-extraction.
- `kitty/rewrap.h` — Generic rewrap logic that reflows content between buffers during resize.
- `3rdparty/ringbuf/ringbuf.h` — Public-domain byte-addressable ring buffer FIFO used for pager history.

**Screen and scroll integration:**
- `kitty/screen.c` — Contains `screen_history_scroll()` (line 4091), `screen_scroll()` (line 1590), `INDEX_UP` macro (line 1553), and `screen_update_cell_data()` (line 2740+). Manages `scrolled_by` state for mixed history/live buffer rendering.
- `kitty/screen.h` — Declares the `Screen` struct with `scrolled_by`, `scroll_changed`, `history_line_added_count`, and `historybuf` fields.

**I/O and rendering architecture:**
- `kitty/child-monitor.c` — Three-thread architecture (I/O, main, talk). Contains `parse_input()` (line 451), `render()` (line 874), `do_parse()` (line 438). Governs `repaint_delay` and `input_delay` timing.
- `kitty/state.h` — Global state including `scrollback_pager_history_size` (line 45), `scrollback_fill_enlarged_window` (line 46), `scrollback_indicator_opacity` (line 59).

**Configuration:**
- `kitty/options/definition.py` — Scrollback group (lines 369–423) defines `scrollback_lines` (default 2000), `scrollback_pager_history_size` (default 0), `scrollback_fill_enlarged_window` (default no), `scrollback_indicator_opacity` (default 1.0).
- `kitty/options/utils.py` — Parser functions: `scrollback_lines()` (line 557) treats negative values as `2^32 - 1`; `scrollback_pager_history_size()` (line 564) converts MB to bytes, capped at 4 GB.
- `kitty/options/types.py` — Typed options including `scrollback_lines: int = 2000` (line 572), `scrollback_pager_history_size: int = 0` (line 574).

**Tests:**
- `kitty_tests/datatypes.py` — Contains `test_historybuf()` (line 487) validating push, pop, index, rewrap, and large-buffer (3000-line) operations.
- `kitty_tests/__init__.py` — `filled_history_buf()` helper (line 184), `create_screen()` with configurable scrollback (line 237).

### 0.2.3 Web Search Research Conducted

No external web search is required for this documentation task. All answers are to be derived directly from the source code as the authoritative truth, per the project implementation rule. The codebase provides complete information about the buffer architecture, memory model, allocation strategy, threading model, and rendering pipeline.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules and structures must be analyzed and documented to answer the user's questions comprehensively:

- **Module: `kitty/history.c`**
  - Public APIs: `alloc_historybuf()`, `historybuf_add_line()`, `historybuf_pop_line()`, `historybuf_init_line()`, `historybuf_clear()`, `historybuf_rewrap()`, `historybuf_mark_line_clean()`, `historybuf_mark_line_dirty()`, `history_buf_endswith_wrap()`
  - Internal functions requiring analysis: `add_segment()`, `segment_for()`, `cpu_lineptr()`, `gpu_lineptr()`, `attrptr()`, `alloc_pagerhist()`, `pagerhist_extend()`, `pagerhist_push()`, `pagerhist_write_bytes()`, `pagerhist_rewrap_to()`
  - Current documentation: **Missing** — no architecture-level documentation for the segmented allocation model or pager-history overflow system
  - Documentation needed: Memory model with per-segment calculations, allocation trigger conditions, circular buffer overflow mechanics, pager-history growth dynamics

- **Module: `kitty/data-types.h`**
  - Key structures: `CPUCell` (12 bytes via `static_assert`), `GPUCell` (20 bytes via `static_assert`), `LineAttrs` (1 byte), `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `LineBuf`
  - Current documentation: **Missing** — sizes are asserted in code but not documented externally
  - Documentation needed: Byte-level structure layout with per-field breakdown, memory cost formulas

- **Module: `kitty/screen.c`**
  - Key functions: `screen_history_scroll()` (line 4091), `screen_scroll()` (line 1590), `screen_index()` (line 1571), `screen_update_cell_data()` (line 2740+), `INDEX_UP` macro (line 1553)
  - Current documentation: **Missing** — no documentation for the scroll-during-output interaction model
  - Documentation needed: How `scrolled_by` is adjusted during concurrent output, how the render path mixes history lines and live lines, dirty-line tracking

- **Module: `kitty/child-monitor.c`**
  - Key functions: `parse_input()` (line 451), `do_parse()` (line 438), `render()` (line 874), `render_os_window()` (line 833)
  - Current documentation: **Partial** — tech spec section 5.2.3 documents the three-thread architecture but not the specific timing interactions during scrollback
  - Documentation needed: How `input_delay` and `repaint_delay` affect scroll responsiveness, the render-scheduling loop during scrollback

- **Module: `kitty/options/definition.py` (scrollback group, lines 369–423)**
  - Configuration options: `scrollback_lines`, `scrollback_pager_history_size`, `scrollback_fill_enlarged_window`, `scrollback_indicator_opacity`, `wheel_scroll_multiplier`
  - Current documentation: **Exists** in `docs/conf.rst` for user-facing description but lacks technical depth on memory implications
  - Documentation needed: Technical relationship between config values and memory footprint

- **Module: `3rdparty/ringbuf/ringbuf.h`**
  - Key functions: `ringbuf_new()`, `ringbuf_capacity()`, `ringbuf_bytes_free()`, `ringbuf_bytes_used()`, `ringbuf_memcpy_into()`, `ringbuf_memmove_from()`
  - Current documentation: **Exists** — well-commented header with public-domain license and comprehensive function descriptions
  - Documentation needed: Integration context explaining how this ring buffer is used for pager history overflow

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented memory model**: No existing document describes the per-segment allocation sizes, the `SEGMENT_SIZE = 2048` constant, or the formula for total memory consumption as a function of `scrollback_lines` and terminal column width (`xnum`).
- **Undocumented buffer lifecycle**: The transition from segment allocation → circular buffer fill → pager-history overflow is not documented anywhere.
- **Undocumented scroll responsiveness model**: The interaction between `scrolled_by` state, `history_line_added_count`, and the `screen_update_cell_data()` render path during concurrent scrollback and output has no external documentation.
- **Undocumented observation methodology**: No measurement scripts or profiling guidance exists specifically for scrollback memory analysis (the existing `docs/performance.rst` covers throughput and CPU only).
- **Undocumented allocation thresholds**: The `SEGMENT_SIZE` boundary at which new segments are allocated, and the ring buffer doubling behavior in `pagerhist_extend()`, are only visible in source code.
- **Missing per-configuration memory estimates**: Users have no guide relating `scrollback_lines` values (e.g., 2000, 10000, 100000) to expected RAM usage.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will be a single comprehensive markdown file placed in the `blitzy/documentation/` directory per the SWE-AtlasQnA-Repo implementation rule. The internal structure of the document follows the user's four questions as natural sections:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction and Question Summary
        ├── Q1: Memory Consumption Under Heavy Load
        │   ├── Buffer Architecture Overview
        │   ├── Per-Segment Memory Calculations
        │   ├── Memory Growth Model (with table)
        │   ├── Pager-History Overflow and Ring Buffer Growth
        │   ├── Measurement Methodology (scripts + expected output)
        │   └── Rationale and Code Citations
        ├── Q2: Scroll Responsiveness During Concurrent Output
        │   ├── Three-Thread Architecture
        │   ├── scrolled_by State and Render Integration
        │   ├── Timing Parameters (input_delay, repaint_delay)
        │   ├── Observable Behavior and Prioritization
        │   └── Rationale and Code Citations
        ├── Q3: Buffer Growth Boundaries
        │   ├── Segment Allocation Trigger (SEGMENT_SIZE = 2048)
        │   ├── Circular Buffer Overflow to Pager History
        │   ├── Ring Buffer Growth Steps
        │   ├── Observable Allocation Events
        │   └── Rationale and Code Citations
        ├── Q4: Observation and Measurement Scripts
        │   ├── Memory Monitoring Script
        │   ├── Output Generation Script
        │   ├── Scroll Latency Observation Script
        │   └── Usage Instructions
        └── Configuration Reference
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract allocation sizes and structure layouts from `kitty/data-types.h` (CPUCell at line 223–228, GPUCell at line 216–221, LineAttrs at line 231–239)
- Extract segment allocation logic from `kitty/history.c` `add_segment()` (lines 17–29) and `segment_for()` (lines 36–42)
- Extract pager-history growth logic from `pagerhist_extend()` (lines 90–101) and `initial_pagerhist_ringbuf_sz()` (line 67)
- Extract scroll behavior from `screen_history_scroll()` in `kitty/screen.c` (line 4091) and `screen_update_cell_data()` (line 2740+)
- Extract timing parameters from `kitty/child-monitor.c` `render()` (line 874) and `do_parse()` (line 438)
- Derive examples by analyzing `kitty_tests/datatypes.py` `test_historybuf()` (line 487)

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Code examples using fenced code blocks with language identifiers
- Tables for memory calculations and configuration options
- Source citations as inline references: `Source: kitty/history.c:17`
- Mermaid diagrams for the buffer lifecycle and thread interaction model
- Consistent terminology: "history buffer" (not "scrollback buffer") for the `HistoryBuf` type, "pager history" for the `PagerHistoryBuf` ring buffer, "segment" for the `HistoryBufSegment` allocation unit

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Buffer Architecture Diagram** — Flowchart showing the relationship between `LineBuf` (active screen), `HistoryBuf` (circular segment buffer), and `PagerHistoryBuf` (ring buffer overflow)
- **Memory Growth Timeline** — Graph-like representation showing step-function memory increases at segment boundaries
- **Thread Interaction Diagram** — Sequence diagram showing I/O thread → main thread (parse_input) → render cycle interaction during concurrent scroll + output
- **Allocation Lifecycle Flowchart** — Decision flow from line output → history push → segment check → pager-history spill


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/history.c`, `kitty/data-types.h`, `kitty/screen.c`, `kitty/child-monitor.c`, `kitty/screen.h`, `kitty/rewrap.h`, `3rdparty/ringbuf/ringbuf.h`, `kitty/options/definition.py`, `kitty/options/utils.py`, `kitty/options/types.py`, `kitty/line-buf.c`, `kitty/lineops.h` | Comprehensive Q&A document answering all four user questions about scrollback buffer memory behavior, scroll responsiveness, allocation boundaries, and observation methodology. Contains Mermaid diagrams, memory calculation tables, code citations, temporary measurement scripts, and configuration reference. |

This is the sole file that will be created. Per the SWE-AtlasQnA-Repo implementation rule, no existing files in the repository will be modified.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation / Architecture Deep-Dive
Source Code:
    - kitty/history.c (primary: segmented allocation, pager history, push/pop/rewrap)
    - kitty/data-types.h (structure definitions: CPUCell, GPUCell, LineAttrs, HistoryBuf, PagerHistoryBuf)
    - kitty/screen.c (scroll behavior: screen_history_scroll, screen_update_cell_data, INDEX_UP)
    - kitty/screen.h (Screen struct: scrolled_by, history_line_added_count, historybuf)
    - kitty/child-monitor.c (thread model: parse_input, render, do_parse, timing parameters)
    - kitty/line-buf.c (LineBuf allocation and mutation)
    - kitty/rewrap.h (rewrap logic during resize)
    - 3rdparty/ringbuf/ringbuf.h (ring buffer API for pager history)
    - kitty/options/definition.py (scrollback configuration: scrollback_lines, scrollback_pager_history_size)
    - kitty/options/utils.py (scrollback_lines(), scrollback_pager_history_size() parsers)
    - kitty/options/types.py (typed defaults: scrollback_lines=2000, scrollback_pager_history_size=0)
Sections:
    - Introduction and Question Summary
    - Q1: Memory Consumption Under Heavy Load
        - Buffer Architecture (HistoryBuf → segmented HistoryBufSegment → PagerHistoryBuf overflow)
        - Per-Segment Memory Cost (formula + calculation table for 80/120/200 columns)
        - Memory Growth Model (step-function at SEGMENT_SIZE=2048 boundaries)
        - Pager-History Ring Buffer Growth (initial 1MB, doubling up to maximum_size)
        - Measurement Methodology (temporary scripts using /proc/[pid]/status)
    - Q2: Scroll Responsiveness During Concurrent Output
        - Three-Thread Architecture (I/O, Main, Talk from child-monitor.c)
        - scrolled_by adjustment path (screen_history_scroll → dirty_scroll)
        - Render cycle mixing (history lines + live lines in screen_update_cell_data)
        - Timing: input_delay and repaint_delay influence on responsiveness
        - Observable prioritization behavior
    - Q3: Buffer Growth Boundaries and Allocation Transitions
        - SEGMENT_SIZE=2048 boundary (add_segment trigger in segment_for)
        - Circular buffer full → pager-history spill (historybuf_push → pagerhist_push)
        - Ring buffer capacity growth (pagerhist_extend, MIN(maximum_size, current + MAX(1MB, needed)))
        - Observable allocation events via memory monitoring
    - Q4: Observation and Measurement Scripts
        - Memory monitoring script (polling /proc/[pid]/status VmRSS)
        - Output generation script (rapid line printing with configurable count)
        - Scroll latency observation approach
        - Usage instructions and expected output format
    - Configuration Reference Table
Diagrams:
    - Buffer architecture Mermaid flowchart
    - Thread interaction Mermaid sequence diagram
    - Allocation lifecycle Mermaid flowchart
Key Citations:
    - kitty/history.c:15 (SEGMENT_SIZE), :17-29 (add_segment), :36-42 (segment_for), :90-101 (pagerhist_extend), :258-273 (pagerhist_push), :276-284 (historybuf_push)
    - kitty/data-types.h:216-221 (GPUCell), :223-228 (CPUCell), :231-239 (LineAttrs), :282-290 (HistoryBuf)
    - kitty/screen.c:1553-1566 (INDEX_UP), :1590-1607 (screen_scroll), :4091-4120 (screen_history_scroll), :2740-2790 (screen_update_cell_data)
    - kitty/child-monitor.c:438-448 (do_parse), :874-877 (render timing)
    - kitty/options/definition.py:372-423 (scrollback config group)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need updating. The new file is placed in `blitzy/documentation/` which is a standalone output directory per the implementation rule, not part of the Sphinx documentation build. No changes are required to `docs/conf.py`, `docs/Makefile`, or `docs/requirements.txt`.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content dependencies**: The output document is self-contained.
- **Internal cross-references**: The document will reference `docs/performance.rst` for throughput benchmarks and `docs/conf.rst` for user-facing configuration documentation, using relative path citations rather than hyperlinks.
- **No navigation or TOC updates required**: The output directory `blitzy/documentation/` is independent of the Sphinx docs tree.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tools or packages need to be installed for this task. The output is a single standalone Markdown file that does not require a documentation generator, build process, or theme framework.

For reference, the project's existing documentation infrastructure uses the following packages (from `docs/requirements.txt`):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | (unpinned) | Existing project docs generator (not needed for this task) |
| pip | furo | (unpinned) | Existing Sphinx theme (not needed for this task) |
| pip | sphinx-copybutton | (unpinned) | Existing docs extension (not needed for this task) |
| pip | sphinxext-opengraph | (unpinned) | Existing docs extension (not needed for this task) |
| pip | sphinx-inline-tabs | (unpinned) | Existing docs extension (not needed for this task) |
| pip | sphinx-autobuild | (unpinned) | Existing live-preview tool (not needed for this task) |

The output document (`kitty_815df1e210e0.md`) uses only standard Markdown with Mermaid diagram syntax. Any Markdown renderer with Mermaid support (GitHub, GitLab, VS Code, etc.) can display the document correctly.

### 0.6.2 Project Runtime Dependencies (Reference Context)

The following runtime dependencies are relevant as context for understanding the scrollback buffer behavior documented in the output file:

| Registry | Package Name | Version Constraint | Purpose |
|----------|--------------|-------------------|---------|
| system | Python | >= 3.8 (per `pyproject.toml`) | Runtime for Kitty's Python layer including Screen, HistoryBuf Python bindings |
| system | GCC/Clang | C11 support | Compiles `kitty/history.c`, `kitty/screen.c`, `kitty/child-monitor.c` C extensions |
| system | Go | 1.22 (per `go.mod`) | Compiles `tools/` Go layer (not directly relevant to scrollback) |
| vendored | ringbuf | Public domain (in-tree) | `3rdparty/ringbuf/` — byte-addressable ring buffer FIFO for pager history |

### 0.6.3 Documentation Reference Updates

No link updates are required. The new document does not replace or redirect any existing documentation. It is an additive artifact in the `blitzy/documentation/` directory, fully independent of the project's existing `docs/` Sphinx tree.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (pre-task):**
- Scrollback memory model documented: 0/4 questions (0%)
- Buffer allocation architecture documented: 0% (only in-code comments exist)
- Scroll responsiveness during concurrent output documented: 0% (not addressed in `docs/performance.rst`)
- Observation methodology for scrollback: 0% (profiling docs cover throughput only)

**Target coverage: 100%** — All four user questions must be fully answered with code-backed evidence.

| Coverage Area | Current | Target | Gap |
|---|---|---|---|
| Memory consumption model | 0% | 100% | Full memory model with per-segment formulas, growth table, pager-history sizing |
| Scroll responsiveness analysis | 0% | 100% | Thread interaction model, timing parameter effects, observable lag characteristics |
| Allocation boundary documentation | 0% | 100% | SEGMENT_SIZE triggers, circular overflow, ring buffer growth steps |
| Measurement scripts and methodology | 0% | 100% | Working temporary scripts for memory monitoring, output generation, and latency observation |
| Configuration relationship mapping | ~30% (user-facing text in `docs/conf.rst`) | 100% | Technical depth linking config values to memory footprint and behavior |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every claim about buffer behavior must cite a specific source file and line number
- Memory calculations must show the full derivation from struct sizes through segment cost to total footprint
- All four user questions must be answered with both theoretical explanation and practical observation methodology
- Configuration options affecting scrollback must be listed with their exact defaults, ranges, and memory implications
- Mermaid diagrams must accurately reflect the actual code structure (not idealized)

**Accuracy validation:**
- All struct sizes cross-referenced against `static_assert` statements in `kitty/data-types.h` (CPUCell at line 228, GPUCell at line 221)
- `SEGMENT_SIZE = 2048` verified directly from `kitty/history.c` line 15
- Default configuration values verified against `kitty/options/types.py` (scrollback_lines=2000 at line 572)
- Thread model verified against `kitty/child-monitor.c` function signatures and scheduling logic
- Allocation logic traced through `add_segment()` → `segment_for()` → `historybuf_push()` call chain

**Clarity standards:**
- Technical accuracy with accessible language for developers who may not be familiar with Kitty internals
- Progressive disclosure: start with high-level architecture, then drill into byte-level calculations
- Consistent terminology: "history buffer" for `HistoryBuf`, "pager history" for `PagerHistoryBuf`, "segment" for `HistoryBufSegment`, "active buffer" for `LineBuf`
- Explicit rationale sections explaining *why* the code behaves as it does, not just *what* it does

**Maintainability:**
- Every technical claim includes a source citation with file path and line number
- Formulas are presented symbolically first, then with concrete values
- The document structure mirrors the user's questions for easy navigation

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per topic area**: At least one concrete numerical example per question (e.g., "For a 80-column terminal with scrollback_lines=10000, memory consumption is X bytes")
- **Diagram types required**: 3 Mermaid diagrams (buffer architecture flowchart, thread interaction sequence, allocation lifecycle)
- **Measurement script examples**: Complete, runnable bash/python scripts for each observation method
- **Code snippet examples**: Short C code excerpts from the source with line number citations illustrating key allocation and scroll logic


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable, a comprehensive Q&A document

**Source code analysis targets (read-only, for documentation content):**
- `kitty/history.c` — Complete analysis of segmented allocation, pager-history ring buffer, push/pop lifecycle, rewrap
- `kitty/data-types.h` — Structure size analysis for CPUCell, GPUCell, LineAttrs, HistoryBuf, HistoryBufSegment, PagerHistoryBuf
- `kitty/screen.c` — Analysis of `screen_history_scroll()`, `screen_scroll()`, `screen_update_cell_data()`, `INDEX_UP` macro, `scrolled_by` management
- `kitty/screen.h` — Analysis of `Screen` struct fields relevant to scrollback (scrolled_by, history_line_added_count, historybuf)
- `kitty/child-monitor.c` — Analysis of three-thread architecture, `parse_input()`, `render()`, `do_parse()`, timing parameters
- `kitty/line-buf.c` — Analysis of `LineBuf` allocation and buffer mutation patterns
- `kitty/lineops.h` — Analysis of shared line-manipulation primitives
- `kitty/rewrap.h` — Analysis of rewrap logic for resize scenarios
- `3rdparty/ringbuf/ringbuf.h` — Analysis of ring buffer API used by pager history
- `kitty/options/definition.py` — Scrollback configuration option definitions (lines 369–423)
- `kitty/options/utils.py` — Scrollback configuration parsers (lines 557–568)
- `kitty/options/types.py` — Typed option defaults (lines 570–574)
- `kitty/state.h` — Global state fields for scrollback configuration (lines 45–46, 59)
- `kitty_tests/datatypes.py` — Test patterns for HistoryBuf validation (lines 487–540)

**Documentation content areas:**
- Memory consumption modeling with concrete byte-level calculations
- Scroll responsiveness characterization under concurrent I/O and rendering
- Buffer allocation boundary identification and documentation
- Temporary observation/measurement script design
- Configuration option–to–memory-footprint mapping

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No changes to any file in the Kitty repository. The implementation rule explicitly forbids modifying existing files.
- **Test file modifications**: No changes to `kitty_tests/` or any other test infrastructure.
- **Feature additions or code refactoring**: This is a pure documentation task; no behavioral changes.
- **Deployment configuration changes**: No changes to build scripts, CI, or packaging.
- **Existing documentation updates**: No modifications to `docs/performance.rst`, `docs/conf.rst`, or any other existing `.rst` file. The output is isolated in `blitzy/documentation/`.
- **Sphinx documentation build**: The output is standalone Markdown, not part of the Sphinx docs tree.
- **Graphics protocol, keyboard protocol, or other subsystems**: Only the scrollback/history buffer subsystem is in scope.
- **Alternate screen buffer behavior**: The alternate screen buffer (`alt_linebuf`) does not have a history buffer and is out of scope.
- **GPU rendering shader internals**: Only the render scheduling and `screen_update_cell_data()` entry point are relevant; the GLSL shader pipeline is out of scope.
- **Remote control, kittens, shell integration**: These subsystems do not interact with the scrollback buffer in ways relevant to the user's questions.


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a single standalone Markdown file that does not require a build step.
- **Documentation preview command**: Any Markdown renderer with Mermaid support (e.g., `grip kitty_815df1e210e0.md`, VS Code Markdown Preview, GitHub web preview).
- **Diagram generation command**: Not applicable — Mermaid diagrams are embedded inline in the Markdown and rendered by the viewer.
- **Documentation deployment command**: Not applicable — the file is placed in `blitzy/documentation/` and is consumed directly.
- **Default format**: Markdown with Mermaid diagram blocks.
- **Citation requirement**: Every technical claim must reference the source file path and line number.
- **Style guide**: Answers must provide thinking and rationale behind conclusions. All facts must be derived from the code as the single source of truth. No assumptions.
- **Documentation validation**: Manual review that all four user questions are answered with code-backed evidence.

### 0.9.2 Output File Placement

The output file must be created at the exact path:

```
blitzy/documentation/kitty_815df1e210e0.md
```

This path is derived from:
- Directory: `blitzy/documentation/` (per the SWE-AtlasQnA-Repo rule)
- Filename: `kitty_815df1e210e0.md` (per the rule: `<source_branch_name>.md`, where the branch name is `kitty_815df1e210e0`)

### 0.9.3 Temporary Script Guidelines

The user permits temporary scripts for observation and measurement. These scripts:
- Must not be committed to the repository
- Must not modify any existing files
- Should use standard system tools (`/proc` filesystem, `ps`, `time`, Python `resource` module)
- Should be documented within the output markdown file as copyable code blocks
- Should include clear usage instructions and expected output format


## 0.10 Rules for Documentation

The following rules govern the creation of this documentation, derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The repository must remain unchanged. All output is confined to the new file at `blitzy/documentation/kitty_815df1e210e0.md`.
- **Base all answers on the code as the single source of truth.** Do not make assumptions. Every claim about buffer behavior, memory consumption, timing, or allocation must be traceable to a specific location in the source code.
- **Provide thinking and rationale behind the answers.** The document must not merely state conclusions but must walk the reader through the reasoning process, showing how each answer is derived from the code.
- **Include actual memory measurements, not just theory.** The user explicitly requires practical observation methodology alongside the theoretical memory model. Provide runnable scripts and expected measurement patterns.
- **Temporary scripts may be used for observation and measurement** but must not be committed to the repository. Document these scripts within the output markdown as copyable code blocks.
- **The document filename must be `kitty_815df1e210e0.md`** and must be placed in the `blitzy/documentation` directory, per the SWE-AtlasQnA-Repo implementation rule.
- **Cite all source code references with file paths and line numbers.** Use the format `Source: kitty/history.c:17` for inline citations.
- **Include Mermaid diagrams for complex architectural relationships.** The buffer lifecycle, thread interaction model, and allocation flow should each have a visual representation.
- **Present memory calculations for multiple terminal widths** (at minimum 80 and 120 columns) to help users understand the scaling behavior.
- **Document the configuration options that affect scrollback behavior** with their defaults, valid ranges, and memory implications.


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were inspected to derive conclusions for this Agent Action Plan:

**Core scrollback buffer implementation:**

| File Path | Purpose of Inspection |
|---|---|
| `kitty/history.c` | Primary source: segmented allocation model, SEGMENT_SIZE=2048, `add_segment()`, `segment_for()`, `historybuf_push()`, `pagerhist_push()`, `pagerhist_extend()`, `historybuf_rewrap()`, pager-history ring buffer integration, ANSI serialization |
| `kitty/data-types.h` | Structure definitions: `CPUCell` (12 bytes, line 228), `GPUCell` (20 bytes, line 221), `LineAttrs` (1 byte, line 239), `HistoryBuf` (lines 282–290), `HistoryBufSegment` (lines 263–266), `PagerHistoryBuf` (lines 268–272), `LineBuf` (lines 252–260), `ensure_space_for` macro (lines 352–359) |
| `kitty/line-buf.c` | `LineBuf` allocation, row addressing, buffer mutation, copy/serialization paths, rewrap integration |
| `kitty/lineops.h` | Shared inline primitives for line operations across screen, scrollback, and text extraction |
| `kitty/rewrap.h` | Generic rewrap logic: `rewrap_inner()`, copy_range, TrackCursor, `next_dest_line` macro |
| `3rdparty/ringbuf/ringbuf.h` | Ring buffer FIFO API: `ringbuf_new()`, `ringbuf_capacity()`, `ringbuf_bytes_free()`, `ringbuf_bytes_used()`, `ringbuf_memcpy_into()`, `ringbuf_memmove_from()`, `ringbuf_copy()` |

**Screen and scroll integration:**

| File Path | Purpose of Inspection |
|---|---|
| `kitty/screen.c` | `screen_history_scroll()` (line 4091), `screen_scroll()` (line 1590), `screen_index()` (line 1571), `INDEX_UP` macro (line 1553), `screen_update_cell_data()` (line 2740+), `dirty_scroll()`, `scrolled_by` management, history/live line mixing during render |
| `kitty/screen.h` | `Screen` struct: `scrolled_by` (line 91), `scroll_changed` (line 101), `history_line_added_count` (line 108), `historybuf` (line 107), `repaint_delay`/`input_delay` in render scheduling |

**I/O and rendering architecture:**

| File Path | Purpose of Inspection |
|---|---|
| `kitty/child-monitor.c` | Three-thread architecture (I/O, Main, Talk), `parse_input()` (line 451), `do_parse()` (line 438), `render()` (line 874), `render_os_window()` (line 833), timing parameters `OPT(input_delay)` and `OPT(repaint_delay)` |
| `kitty/state.h` | Global state fields: `scrollback_pager_history_size` (line 45), `scrollback_fill_enlarged_window` (line 46), `scrollback_indicator_opacity` (line 59) |

**Configuration:**

| File Path | Purpose of Inspection |
|---|---|
| `kitty/options/definition.py` | Scrollback configuration group (lines 369–423): `scrollback_lines` (default 2000), `scrollback_pager_history_size` (default 0), `scrollback_fill_enlarged_window` (default no), `scrollback_indicator_opacity` (default 1.0), `wheel_scroll_multiplier` |
| `kitty/options/utils.py` | Parser functions: `scrollback_lines()` (line 557, negative → 2^32-1), `scrollback_pager_history_size()` (line 564, MB → bytes, max 4GB) |
| `kitty/options/types.py` | Typed defaults: `scrollback_lines: int = 2000` (line 572), `scrollback_pager_history_size: int = 0` (line 574) |

**Tests:**

| File Path | Purpose of Inspection |
|---|---|
| `kitty_tests/datatypes.py` | `test_historybuf()` (line 487): push, pop, indexing, large buffer (3000 lines), rewrap operations |
| `kitty_tests/__init__.py` | `filled_history_buf()` helper (line 184), `create_screen()` with configurable scrollback (line 237) |

**Existing documentation:**

| File Path | Purpose of Inspection |
|---|---|
| `docs/performance.rst` | Existing performance documentation: throughput benchmarks, CPU usage, latency measurements, gperftools profiling — confirmed no coverage of scrollback memory or scroll responsiveness |
| `docs/conf.py` | Sphinx configuration: project metadata, theme, extensions — confirmed documentation infrastructure |
| `docs/requirements.txt` | Documentation dependencies: sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild |
| `docs/` (folder) | Full documentation tree structure assessment |

**Project configuration:**

| File Path | Purpose of Inspection |
|---|---|
| `pyproject.toml` | Python version constraint: `requires-python = ">=3.8"` |
| Root folder (`""`) | Repository structure assessment: identified `kitty/`, `docs/`, `kitty_tests/`, `3rdparty/`, `tools/` |

### 0.11.2 Tech Spec Sections Referenced

| Section | Purpose |
|---|---|
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | VT parser dispatch, GPU rendering pipeline, threaded rendering architecture, performance optimization parameters |
| 5.2 COMPONENT DETAILS | Child Monitor three-thread architecture (5.2.3), VT Parser and Screen Model (5.2.4), GPU Rendering Pipeline (5.2.5), Configuration System (5.2.8) |

### 0.11.3 Attachments

No attachments were provided by the user. No Figma URLs or external design references are applicable to this task.


