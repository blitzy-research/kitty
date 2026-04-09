# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, comprehensive Q&A-style markdown document** that provides a deep, code-grounded exploration of Kitty's `HistoryBuf` internals under extreme stress conditions. The user seeks intuitive understanding — not a theory lecture — of how the scrollback memory structures actually behave when pushed beyond normal usage.

- **Category:** Create new documentation
- **Documentation type:** Technical deep-dive / Architecture exploration document
- **Target audience:** A developer or advanced user wanting to understand Kitty's history buffer internals at runtime, with evidence from actual code paths rather than abstract descriptions

The documentation requirements, restated with enhanced clarity, are:

- **Segment lifecycle under flood:** Document what actually unfolds inside `HistoryBuf` (defined in `kitty/data-types.h:282–290`, implemented in `kitty/history.c`) when an enormous volume of text is written in a short time — how segments are allocated via `add_segment()`, how the circular buffer fills via `historybuf_push()`, and what happens when `count` reaches `ynum`
- **Segmented scrollback and pager ring buffer interaction:** Explain the quiet handoff between the main segmented scrollback (`HistoryBufSegment` array) and the `PagerHistoryBuf` ring buffer (`3rdparty/ringbuf/ringbuf.c`), specifically the `pagerhist_push()` serialization path that triggers when the main buffer is full
- **Transition smoothness at segment boundaries:** Analyze whether the system hesitates, stalls, or shows observable latency at segment boundaries — specifically the `segment_for()` lazy allocation path (line 37–41 of `kitty/history.c`) and the `realloc()` on `self->segments`
- **Concurrent scrolling and data arrival:** Explain what changes when a user is actively scrolling (`scrolled_by` in `kitty/screen.h:91`) while new data continues to arrive, including the `screen_update_cell_data()` adjustment logic (line 2761 of `kitty/screen.c`)
- **Allocation, wrapping, and retention behavior:** Observe and document how the ring buffer overflow semantics in `ringbuf_memcpy_into()` silently advance the tail pointer, how `pagerhist_extend()` grows the ring buffer on demand up to `maximum_size`, and how UTF-8 boundary repair operates in `pagerhist_ensure_start_is_valid_utf8()`
- **Temporary scripts for observation, no repository modifications:** The user explicitly permits temporary scripts for observation but requires the repository to remain unchanged, with all temporaries cleaned up afterward

### 0.1.2 Special Instructions and Constraints

- **No repository modifications:** The user explicitly states "the repository itself should remain unchanged, and anything temporary should be cleaned up afterward." This means no source code edits, no test file changes, no build modifications
- **Code-as-truth doctrine:** Per the project implementation rules, answers must be "based on the code as the truth" — no assumptions, only evidence from actual source
- **Output file location and naming:** Per the `SWE-AtlasQnA-Repo` rule, the generated document must be named `kitty_815df1e210e0.md` (matching the source branch name) and placed in the `blitzy/documentation` directory
- **Thinking / rationale required:** The document must provide thinking and rationale behind each answer, not just bare facts
- **Observation-friendly:** The user wants to "see hints of how the underlying memory structures evolve as pressure builds" — the document should include concrete examples, code references with line numbers, and suggest temporary observation scripts where helpful

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **segment lifecycle**, we will create a new section in `blitzy/documentation/kitty_815df1e210e0.md` that traces the `historybuf_push()` → `pagerhist_push()` → `add_segment()` path with exact code references from `kitty/history.c`
- To document the **scrollback ↔ pager interaction**, we will analyze the data flow from `HistoryBufSegment` overflow through `pagerhist_write_bytes()` and `pagerhist_write_ucs4()` into the `ringbuf_t` FIFO, citing `3rdparty/ringbuf/ringbuf.c`
- To document **transition behavior at limits**, we will examine `segment_for()` (lazy `realloc` path), `pagerhist_extend()` (ring buffer growth), and `ringbuf_memcpy_into()` (silent overflow) for evidence of blocking or stalling
- To document **concurrent scroll + write**, we will trace the `screen_update_cell_data()` function which adjusts `scrolled_by` using `history_line_added_count` at render time, showing the render-time reconciliation approach
- To support **runtime observation**, we will describe temporary Python scripts using `kitty.fast_data_types.HistoryBuf` and `Screen` that can probe buffer states without modifying the repository

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs were identified:

- **Configuration impact documentation:** The user's questions are deeply affected by `scrollback_lines` (default 2000, defined at `kitty/options/definition.py:372`), `scrollback_pager_history_size` (default 0, at line 406), and `SEGMENT_SIZE` (hardcoded 2048, at `kitty/history.c:15`). The document must explain how these parameters shape the behavior under stress
- **Memory model explanation:** The dual-buffer architecture (segmented `HistoryBufSegment` array for structured line storage + `PagerHistoryBuf` ring buffer for serialized ANSI text) is not documented anywhere in the repository's existing docs and must be explained
- **Rewrap under load:** The user asks about wrapping behavior — the `pagerhist_rewrap_to()` function and its interaction with the ring buffer during width changes is an implicit need
- **Error/edge cases:** The `fatal()` calls in `add_segment()` and `segment_for()` (out-of-memory scenarios), plus the ring buffer's silent overwrite semantics, represent important edge cases to document

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx/reStructuredText documentation tree** in the `docs/` directory with comprehensive user-facing documentation, but **no existing deep-dive documentation on HistoryBuf internals or scrollback memory architecture**.

- **Current documentation framework:** Sphinx, configured via `docs/conf.py`
- **Documentation generator configuration:** `docs/Makefile` provides standard Sphinx build targets; `docs/requirements.txt` pins `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx_inline_tabs`, `sphinx-autobuild`
- **API documentation tools:** No formal API reference generator (no JSDoc, Doxygen, or Sphinx autodoc for C code). The C code in `kitty/history.c` has inline `#define` doc strings for Python-exposed methods but no standalone API reference
- **Diagram tools detected:** Mermaid is used in the technical specification (as seen in section 4.3). The existing docs use reStructuredText with no Mermaid integration; diagrams would be novel additions in any new markdown document
- **Documentation hosting/deployment:** The docs are published to the kitty website, built via Sphinx. The new document goes into `blitzy/documentation/`, a separate path from the Sphinx tree

Existing documentation files examined for scrollback-related content:

| File | Relevance | Finding |
|------|-----------|---------|
| `docs/conf.rst` | Configuration reference | Documents `scrollback_lines`, `scrollback_pager`, `scrollback_pager_history_size`, and `scrollback_fill_enlarged_window` options at a user-facing level only |
| `docs/performance.rst` | Performance tuning | Contains general performance advice but no HistoryBuf internals |
| `docs/faq.rst` | FAQ | No scrollback memory architecture discussion |
| `docs/overview.rst` | Project overview | High-level features only |
| `docs/unscroll.rst` | Unscroll feature | Documents the VT-420 SD extension, tangentially related to scrollback but not HistoryBuf internals |

**Conclusion:** No existing documentation covers the internal mechanics of `HistoryBuf`, `PagerHistoryBuf`, segment allocation, ring buffer overflow, or concurrent scroll/write behavior. The requested document fills a genuine documentation gap.

### 0.2.2 Repository Code Analysis for Documentation

The following code files were deeply analyzed to extract documentation material:

| Source File | Examination Focus | Key Findings |
|-------------|-------------------|--------------|
| `kitty/history.c` (625 lines) | Full HistoryBuf implementation | Segmented circular buffer, `SEGMENT_SIZE=2048`, lazy segment allocation, pager history serialization, rewrap logic |
| `kitty/data-types.h` (lines 262–290) | Struct definitions | `HistoryBufSegment`, `PagerHistoryBuf`, `HistoryBuf` type layouts |
| `3rdparty/ringbuf/ringbuf.c` (395 lines) | Ring buffer FIFO | Overflow semantics (tail advances on overwrite), one-byte sentinel for full/empty distinction, wrap-aware `memcpy_into` |
| `3rdparty/ringbuf/ringbuf.h` (252 lines) | Ring buffer API | Public interface: `ringbuf_new`, `ringbuf_capacity`, `ringbuf_bytes_free`, `ringbuf_memcpy_into`, `ringbuf_findchr` |
| `kitty/screen.c` (lines 1558, 2716, 4091) | Screen ↔ HistoryBuf interaction | `INDEX_UP` macro pushes to history, `screen_history_scroll` adjusts `scrolled_by`, `screen_update_cell_data` reconciles scroll position during render |
| `kitty/screen.h` (lines 80–175) | Screen struct | `scrolled_by` field, `historybuf` pointer, `history_line_added_count` |
| `kitty/rewrap.h` (97 lines) | Rewrap engine | `rewrap_inner()` function reflowing content across buffers with cursor tracking |
| `kitty/line-buf.c` | LineBuf implementation | Line buffer operations that feed into HistoryBuf via `historybuf_add_line()` |
| `kitty/child-monitor.c` (lines 1475–1575) | I/O loop | Poll-based data reading, `input_delay`-throttled main loop wakeup |
| `kitty/options/definition.py` (lines 370–435) | Configuration options | `scrollback_lines` (default 2000), `scrollback_pager_history_size` (default 0), `scrollback_pager`, `scrollback_fill_enlarged_window` |
| `kitty_tests/datatypes.py` (lines 486–540) | Unit tests | Tests for `HistoryBuf.push()`, `HistoryBuf.rewrap()`, large buffer (3000 lines) |
| `kitty_tests/screen.py` (lines 275–350) | Screen tests | Resize with scrollback, scrollback fill after resize |
| `kitty/window.py` (lines 1735–1790) | Scrollback pager launch | `show_scrollback()` calling `as_text(add_history=True)` to extract pager content |

### 0.2.3 Web Search Research Conducted

No external web search was needed for this task. The documentation objective is entirely code-internal — analyzing Kitty's own C source for behavioral characteristics. All answers are derived from the repository's source code as the authoritative truth, per the project's implementation rules.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The documentation must synthesize deep answers from the following code modules:

- **Module: `kitty/history.c`**
  - Public APIs: `historybuf_push()`, `historybuf_add_line()`, `historybuf_pop_line()`, `historybuf_clear()`, `historybuf_rewrap()`, `historybuf_init_line()`, `historybuf_cpu_cells()`, `historybuf_mark_line_dirty/clean()`, `history_buf_endswith_wrap()`
  - Internal functions: `add_segment()`, `free_segment()`, `segment_for()`, `index_of()`, `init_line()`, `pagerhist_push()`, `pagerhist_write_bytes()`, `pagerhist_write_ucs4()`, `pagerhist_extend()`, `pagerhist_clear()`, `pagerhist_ensure_start_is_valid_utf8()`, `pagerhist_rewrap_to()`
  - Current documentation: No standalone documentation exists — only inline `#define` doc strings for Python-exposed methods
  - Documentation needed: Complete behavioral analysis under stress conditions with code-path tracing

- **Module: `3rdparty/ringbuf/ringbuf.c`**
  - Public APIs: `ringbuf_new()`, `ringbuf_free()`, `ringbuf_reset()`, `ringbuf_capacity()`, `ringbuf_bytes_free()`, `ringbuf_bytes_used()`, `ringbuf_memcpy_into()`, `ringbuf_memcpy_from()`, `ringbuf_memmove_from()`, `ringbuf_move_char()`, `ringbuf_findchr()`, `ringbuf_copy()`
  - Current documentation: Header comments in `ringbuf.h` describe the API contract, but no behavioral documentation under overflow conditions
  - Documentation needed: Overflow semantics analysis — how `ringbuf_memcpy_into()` silently advances the tail on overflow, the one-byte sentinel invariant

- **Module: `kitty/screen.c` (scrollback interaction paths)**
  - Key functions: `screen_index()`, `screen_scroll()`, `screen_history_scroll()`, `screen_update_cell_data()`, `screen_reverse_scroll_and_fill_from_scrollback()`, `screen_clear_scrollback()`
  - Current documentation: No internal documentation on the `scrolled_by` reconciliation mechanism
  - Documentation needed: Concurrent read/write behavior analysis — how `scrolled_by` is adjusted during render when `history_line_added_count > 0`

- **Module: `kitty/data-types.h` (struct definitions)**
  - Types: `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `CPUCell`, `GPUCell`, `LineAttrs`
  - Current documentation: Struct definitions with minimal comments
  - Documentation needed: Memory layout documentation showing per-segment allocation sizes and the dual-buffer architecture

- **Configuration options requiring documentation:**
  - `scrollback_lines` (default 2000) at `kitty/options/definition.py:372` — determines `ynum`
  - `scrollback_pager_history_size` (default 0 MB) at `kitty/options/definition.py:406` — determines `PagerHistoryBuf.maximum_size`
  - `SEGMENT_SIZE` (hardcoded 2048) at `kitty/history.c:15` — determines lines per segment
  - These are critical parameters that control every aspect of the user's questions

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented internal architecture:** The dual-buffer model (segmented `HistoryBufSegment` array + `PagerHistoryBuf` ring buffer) has no documentation anywhere in the repository
- **No stress-behavior documentation:** No existing document discusses what happens when the buffer is full, how segments are allocated incrementally, or how the pager ring buffer grows and eventually wraps
- **No concurrent-access documentation:** The `scrolled_by` reconciliation in `screen_update_cell_data()` — where the render thread adjusts the scroll position by the count of newly added history lines — is completely undocumented
- **No memory-model documentation:** Per-segment memory cost (`xnum * SEGMENT_SIZE * sizeof(CPUCell) + xnum * SEGMENT_SIZE * sizeof(GPUCell) + SEGMENT_SIZE * sizeof(LineAttrs)`) is not documented anywhere
- **No observation guidance:** No documentation exists on how to probe or observe buffer state at runtime using the Python-exposed `fast_data_types.HistoryBuf` API

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The single output file will follow a Q&A-style deep-dive structure:

```
blitzy/documentation/
└── kitty_815df1e210e0.md
    ├── Introduction and Context
    ├── Q1: What unfolds inside HistoryBuf under flood?
    │   ├── Segment allocation lifecycle
    │   ├── Circular index mechanics
    │   └── Memory cost analysis
    ├── Q2: How do segmented scrollback and pager ring buffer interact?
    │   ├── The pagerhist_push() serialization path
    │   ├── Ring buffer growth via pagerhist_extend()
    │   └── Ring buffer overflow (silent overwrite)
    ├── Q3: Are there hesitation points at segment boundaries?
    │   ├── Lazy segment allocation in segment_for()
    │   ├── realloc() on the segments array
    │   └── Ring buffer reallocation vs. overwrite
    ├── Q4: What happens when scrolling while new data arrives?
    │   ├── The scrolled_by reconciliation mechanism
    │   ├── Render-time adjustment via history_line_added_count
    │   └── Edge cases: scroll position exceeding history count
    ├── Q5: How do allocation, wrapping, and retention evolve under pressure?
    │   ├── UTF-8 boundary repair
    │   ├── Rewrap on width change under load
    │   └── Configuration levers and their effects
    ├── Observation Techniques (temporary scripts)
    └── Summary of Key Insights
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach:**
  - Extract function signatures and control flow from `kitty/history.c` using direct code reading
  - Trace the `historybuf_push()` → `pagerhist_push()` → `ringbuf_memcpy_into()` chain for the overflow path
  - Trace `screen_index()` → `INDEX_UP` → `historybuf_add_line()` for the screen-to-history push path
  - Analyze `screen_update_cell_data()` for the concurrent scroll+write reconciliation mechanism
  - Generate memory cost formulas from struct sizes in `kitty/data-types.h` (`sizeof(CPUCell)=12`, `sizeof(GPUCell)=20`, `sizeof(LineAttrs)=1`)

- **Documentation Standards:**
  - Markdown formatting with proper headers (# ## ###)
  - Mermaid diagrams for the dual-buffer architecture and data flow
  - Code citations as inline references: `Source: kitty/history.c:276`
  - Tables for configuration parameter summaries and memory calculations
  - Consistent terminology: "segmented scrollback" for `HistoryBufSegment` array, "pager ring buffer" for `PagerHistoryBuf`

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to include in the output document:

- **Dual-buffer architecture diagram:** Showing the `HistoryBuf` containing the segmented scrollback (array of `HistoryBufSegment`) alongside the `PagerHistoryBuf` ring buffer, with arrows showing the overflow serialization path
- **Segment allocation lifecycle:** Flowchart showing `segment_for()` → lazy `add_segment()` → `realloc()` on the segments pointer array
- **Overflow and ring buffer growth:** Sequence diagram showing `pagerhist_push()` → `pagerhist_write_bytes()` → `pagerhist_extend()` (if space insufficient) → `ringbuf_memcpy_into()` (with overflow tail advance)
- **Concurrent scroll+write reconciliation:** Flowchart showing `screen_update_cell_data()` adjusting `scrolled_by` by `history_line_added_count` at render time

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/history.c`, `kitty/data-types.h`, `3rdparty/ringbuf/ringbuf.c`, `kitty/screen.c`, `kitty/screen.h`, `kitty/rewrap.h`, `kitty/options/definition.py`, `kitty/line-buf.c`, `kitty/child-monitor.c`, `kitty_tests/datatypes.py` | Comprehensive Q&A deep-dive on HistoryBuf behavior under extreme load, covering segment allocation, pager ring buffer interaction, concurrent scroll/write, and runtime observation techniques |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical deep-dive / Architecture Q&A
Source Code:
  - kitty/history.c (primary — all HistoryBuf logic)
  - kitty/data-types.h (struct definitions: HistoryBuf, HistoryBufSegment, PagerHistoryBuf, CPUCell, GPUCell, LineAttrs)
  - 3rdparty/ringbuf/ringbuf.c (ring buffer FIFO implementation)
  - 3rdparty/ringbuf/ringbuf.h (ring buffer API contract)
  - kitty/screen.c (screen_index, screen_history_scroll, screen_update_cell_data)
  - kitty/screen.h (Screen struct with scrolled_by, historybuf, history_line_added_count)
  - kitty/rewrap.h (rewrap_inner reflow engine)
  - kitty/line-buf.c (LineBuf operations feeding HistoryBuf)
  - kitty/child-monitor.c (I/O loop data arrival path)
  - kitty/options/definition.py (scrollback configuration options)
  - kitty_tests/datatypes.py (HistoryBuf unit tests)
  - kitty/window.py (show_scrollback pager launch)
Sections:
  - Introduction (purpose, scope, configuration context)
  - Q1: HistoryBuf filling under flood (segment lifecycle, circular indexing, memory cost)
  - Q2: Scrollback ↔ pager ring buffer interaction (pagerhist_push, ring buffer growth, overflow)
  - Q3: Transition smoothness at segment boundaries (lazy allocation, realloc behavior)
  - Q4: Concurrent scrolling and data arrival (scrolled_by reconciliation, render-time adjustment)
  - Q5: Allocation, wrapping, and retention under pressure (UTF-8 repair, rewrap, config levers)
  - Observation Techniques (temporary Python scripts using fast_data_types API)
  - Summary of Key Insights
Diagrams:
  - Mermaid: Dual-buffer architecture (HistoryBufSegment array + PagerHistoryBuf ring buffer)
  - Mermaid: Segment allocation lifecycle flowchart
  - Mermaid: Ring buffer overflow and growth sequence
  - Mermaid: Concurrent scroll+write reconciliation at render time
Key Citations:
  - kitty/history.c:15 (SEGMENT_SIZE)
  - kitty/history.c:17-29 (add_segment)
  - kitty/history.c:36-42 (segment_for)
  - kitty/history.c:66-67 (initial_pagerhist_ringbuf_sz)
  - kitty/history.c:89-101 (pagerhist_extend)
  - kitty/history.c:218-226 (pagerhist_write_bytes)
  - kitty/history.c:258-273 (pagerhist_push)
  - kitty/history.c:275-284 (historybuf_push)
  - kitty/history.c:391-432 (pagerhist_rewrap_to)
  - kitty/history.c:594-614 (historybuf_rewrap)
  - 3rdparty/ringbuf/ringbuf.c:211-238 (ringbuf_memcpy_into overflow)
  - kitty/screen.c:1558-1564 (INDEX_UP macro)
  - kitty/screen.c:2761 (scrolled_by adjustment in screen_update_cell_data)
  - kitty/screen.c:4091-4119 (screen_history_scroll)
  - kitty/data-types.h:216-228 (CPUCell=12 bytes, GPUCell=20 bytes)
  - kitty/data-types.h:231-239 (LineAttrs=1 byte)
  - kitty/data-types.h:262-290 (HistoryBufSegment, PagerHistoryBuf, HistoryBuf)
  - kitty/options/definition.py:372-425 (scrollback config options)
```

### 0.5.3 Cross-Documentation Dependencies

- **No cross-doc navigation updates needed:** The new file resides in `blitzy/documentation/`, which is independent of the Sphinx documentation tree in `docs/`
- **No documentation configuration updates:** No `mkdocs.yml`, `docusaurus.config.js`, or `docs/conf.py` modifications are required
- **No index/glossary updates:** The new document is standalone
- **Internal source citations:** All code references within the document point to exact file paths and line numbers in the repository

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tooling or packages are required for this task. The output is a single standalone Markdown file placed in `blitzy/documentation/`. No documentation site generator, build tool, or diagram rendering pipeline is involved.

The documentation content itself depends on understanding the following project runtime dependencies (already present in the repository):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| C (vendored) | ringbuf | N/A (3rdparty/ringbuf/) | Ring buffer FIFO backing the `PagerHistoryBuf`; CC0 public domain by Drew Hess |
| Python (built-in) | fast_data_types | N/A (compiled C extension) | Python-exposed `HistoryBuf`, `LineBuf`, `Screen` types used for observation scripts |
| Python | Python | >=3.8 (per `pyproject.toml`) | Runtime for kitty's Python layer and observation scripts |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file `blitzy/documentation/kitty_815df1e210e0.md` is self-contained and does not participate in any existing documentation navigation or link structure.

No link transformation rules apply — this is a greenfield documentation artifact with no predecessor documents to redirect from.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **Current coverage analysis:**
  - HistoryBuf internal architecture documented: 0/5 user questions (0%)
  - Segment allocation lifecycle documented: Not at all in existing docs
  - Pager ring buffer overflow semantics documented: Not at all in existing docs
  - Concurrent scroll/write behavior documented: Not at all in existing docs
  - Configuration impact on stress behavior documented: Partially (user-facing option descriptions exist in `docs/conf.rst`, but no internal mechanics)

- **Target coverage:** 100% of the five core questions posed by the user, each answered with code evidence and rationale
- **Coverage gaps to address:**

| Topic Area | Current State | Target |
|------------|---------------|--------|
| Segment allocation under flood | Undocumented | Full trace: `historybuf_push()` → `segment_for()` → `add_segment()` |
| Pager ring buffer interaction | Undocumented | Full trace: `pagerhist_push()` → `pagerhist_write_bytes()` → `ringbuf_memcpy_into()` |
| Transition smoothness at limits | Undocumented | Analysis of `realloc` in `add_segment()`, `pagerhist_extend()`, ring buffer overflow |
| Concurrent scroll + data arrival | Undocumented | Full trace: `scrolled_by` adjustment in `screen_update_cell_data()` |
| Runtime observation techniques | Undocumented | Temporary Python scripts using `fast_data_types` API |

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every user question is answered with explicit code references (file:line)
  - Every claim includes rationale explaining *why* the code behaves that way
  - Configuration parameters are documented with their defaults and effects on stress behavior
  - Memory cost formulas are provided with concrete numeric examples

- **Accuracy validation:**
  - All code references verified against actual source files in the repository
  - Struct sizes validated via `static_assert` in `kitty/data-types.h` (CPUCell=12 bytes at line 228, GPUCell=20 bytes at line 221)
  - Configuration defaults verified against `kitty/options/definition.py`
  - Ring buffer overflow semantics verified against `3rdparty/ringbuf/ringbuf.c:211–238`

- **Clarity standards:**
  - Progressive disclosure: start with high-level architecture, then trace specific code paths
  - Mermaid diagrams for visual understanding of data flow
  - Concrete numeric examples (e.g., "at 80 columns, one segment costs 80 × 2048 × 12 + 80 × 2048 × 20 + 2048 × 1 = 5,245,952 bytes ≈ 5 MB")
  - Consistent terminology throughout: "segmented scrollback," "pager ring buffer," "circular index"

- **Maintainability:**
  - Source citations with file paths and line numbers for traceability
  - Clearly labeled sections matching the user's original questions
  - Self-contained document requiring no external cross-references

### 0.7.3 Example and Diagram Requirements

- **Minimum examples:** At least one concrete numeric example per major topic (segment memory cost, ring buffer capacity progression, scroll position arithmetic)
- **Diagram types required:** 4 Mermaid diagrams (architecture, segment lifecycle, ring buffer overflow, scroll reconciliation)
- **Observation script examples:** At least 2 temporary Python script examples showing how to probe `HistoryBuf` state at runtime
- **Code citation density:** Every behavioral claim references a specific source file and line range

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — the sole output artifact

- **Source code analyzed (read-only, for documentation content):**
  - `kitty/history.c` — HistoryBuf implementation (segments, pager history, rewrap)
  - `kitty/data-types.h` — Struct definitions (HistoryBuf, HistoryBufSegment, PagerHistoryBuf, CPUCell, GPUCell, LineAttrs)
  - `3rdparty/ringbuf/ringbuf.c` — Ring buffer FIFO implementation
  - `3rdparty/ringbuf/ringbuf.h` — Ring buffer API interface
  - `kitty/screen.c` — Screen ↔ HistoryBuf interaction (index, scroll, render)
  - `kitty/screen.h` — Screen struct with scrolled_by, historybuf fields
  - `kitty/rewrap.h` — Rewrap engine for terminal resize
  - `kitty/line-buf.c` — LineBuf operations feeding into HistoryBuf
  - `kitty/child-monitor.c` — I/O loop data arrival
  - `kitty/options/definition.py` — Scrollback configuration options
  - `kitty/window.py` — Scrollback pager launch mechanism
  - `kitty_tests/datatypes.py` — HistoryBuf unit tests
  - `kitty_tests/screen.py` — Screen resize/scrollback tests

- **Documentation content topics:**
  - HistoryBuf segment allocation lifecycle
  - PagerHistoryBuf ring buffer growth and overflow semantics
  - Interaction between segmented scrollback and pager ring buffer
  - Concurrent scrolling and data arrival behavior
  - Memory cost analysis and configuration impact
  - Runtime observation techniques via temporary scripts

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be modified, per the user's explicit instruction: "the repository itself should remain unchanged"
- **Test file modifications:** No changes to `kitty_tests/` files
- **Build system changes:** No changes to `setup.py`, `Makefile`, or any build configuration
- **Existing documentation updates:** No changes to `docs/**/*.rst` or `docs/conf.py`
- **Feature additions or code refactoring:** This is a documentation-only task
- **Deployment configuration changes:** No changes to CI/CD, packaging, or release infrastructure
- **GPU rendering pipeline documentation:** Not relevant to the HistoryBuf question
- **Font handling documentation:** Not relevant to the HistoryBuf question
- **Keyboard/mouse input pipeline documentation:** Not relevant to the HistoryBuf question
- **Graphics protocol documentation:** Not relevant to the HistoryBuf question
- **Permanent observation infrastructure:** The user allows temporary scripts but explicitly requires cleanup; no permanent observability tooling is in scope

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file that does not participate in any documentation build pipeline
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown and render in any Mermaid-compatible viewer (GitHub, VS Code, etc.)
- **Documentation deployment command:** Not applicable — the file is committed directly to the `blitzy/documentation` directory
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with exact paths and line numbers
- **Style guide:** Q&A deep-dive style with progressive disclosure — each question starts with an intuitive explanation, followed by a code-path trace, then concrete examples
- **Documentation validation:** Manual review; verify all cited line numbers match the repository source

### 0.9.2 File Placement and Naming

Per the `SWE-AtlasQnA-Repo` implementation rule:

- **File name:** `kitty_815df1e210e0.md` (matches the source branch name `kitty_815df1e210e0`)
- **Directory:** `blitzy/documentation/`
- **Full path:** `blitzy/documentation/kitty_815df1e210e0.md`
- **Directory creation:** The `blitzy/documentation/` directory must be created if it does not yet exist

### 0.9.3 Repository Integrity Requirements

- **No existing files modified:** Verified — the only file operation is CREATE of one new file
- **Temporary scripts:** If observation scripts are described in the document, they must include explicit cleanup instructions (e.g., "delete the script file after use")
- **Git state:** Only one new file should appear in `git status` after execution: `blitzy/documentation/kitty_815df1e210e0.md`

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **"Do not modify any existing files in the source repository."** — The `SWE-AtlasQnA-Repo` rule explicitly prohibits modifications to existing repository files. The only permissible operation is creating the new documentation file.
- **"Do not make assumptions, base your answers on the code as the truth."** — Every behavioral claim in the document must cite specific source code. No speculative statements about performance characteristics unless directly supported by code evidence.
- **"Provide thinking / rationale behind the answers."** — Each answer must include not just what happens, but why the code works that way — the design rationale inferred from the implementation patterns.
- **"Create a new markdown document named `<source_branch_name>.md`"** — The output file must be named `kitty_815df1e210e0.md` exactly.
- **"Place the generated document in the `blitzy/documentation` directory in the destination repo."** — The file goes into `blitzy/documentation/`, creating the directory if necessary.
- **"the repository itself should remain unchanged, and anything temporary should be cleaned up afterward"** — Any observation scripts described in the document must be clearly labeled as temporary with cleanup instructions. No permanent artifacts beyond the single markdown file.
- **"Temporary scripts may be used for observation"** — The document may include example Python scripts that use `kitty.fast_data_types` to probe buffer states, but must clearly indicate these are for temporary use only.

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were directly retrieved and analyzed to derive the conclusions in this Agent Action Plan:

| File Path | Lines Examined | Purpose |
|-----------|---------------|---------|
| `kitty/history.c` | 1–625 (full file) | Primary: HistoryBuf implementation — segments, pager history, push/pop, rewrap, serialization |
| `kitty/data-types.h` | 1–400 | Struct definitions: `HistoryBuf`, `HistoryBufSegment`, `PagerHistoryBuf`, `CPUCell`, `GPUCell`, `LineAttrs`, `Line`, `LineBuf` |
| `3rdparty/ringbuf/ringbuf.c` | 1–395 (full file) | Ring buffer FIFO: overflow semantics, `memcpy_into`, `memmove_from`, `findchr`, `copy` |
| `3rdparty/ringbuf/ringbuf.h` | 1–110 | Ring buffer API contract: `ringbuf_new`, `ringbuf_capacity`, head/tail semantics |
| `kitty/screen.c` | Lines 1540–1640, 2700–2800, 4085–4130 | Screen ↔ HistoryBuf interaction: `screen_index()`, `INDEX_UP` macro, `screen_history_scroll()`, `screen_update_cell_data()` |
| `kitty/screen.h` | Lines 80–175 | Screen struct: `scrolled_by`, `historybuf`, `history_line_added_count`, `paused_rendering` |
| `kitty/rewrap.h` | 1–97 (full file) | Rewrap engine: `rewrap_inner()`, cursor tracking, line-copy logic |
| `kitty/options/definition.py` | Lines 370–435 | Configuration: `scrollback_lines`, `scrollback_pager_history_size`, `scrollback_pager`, `scrollback_fill_enlarged_window` |
| `kitty/window.py` | Lines 1735–1790 | Pager launch: `show_scrollback()` extracting pager content via `as_text(add_history=True)` |
| `kitty/child-monitor.c` | Lines 1475–1575 | I/O loop: poll-based data reading, `input_delay`-throttled main loop wakeup |
| `kitty_tests/datatypes.py` | Lines 486–540 | Unit tests: `test_historybuf()` covering push, rewrap, large buffers (3000 lines) |
| `kitty_tests/screen.py` | Lines 275–350 | Screen tests: `test_resize()` with scrollback, `test_scrollback_fill_after_resize()` |
| `kitty/line-buf.c` | Summary only | LineBuf implementation: row addressing, clear, index, copy, rewrap |
| `pyproject.toml` | Full file | Python version requirement: `>=3.8` |
| `setup.py` | Lines 1–30 | Build system entry point |

The following folders were explored for structural context:

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| Root (`""`) | 0 | Repository root: identified all top-level directories and files |
| `kitty/` | 1 | Core application tree: located `history.c`, `screen.c`, `data-types.h`, `rewrap.h` |
| `docs/` | 1 | Documentation tree: verified no existing HistoryBuf internals documentation |
| `3rdparty/ringbuf/` | 2 | Vendored ring buffer: analyzed `ringbuf.c` and `ringbuf.h` |
| `kitty_tests/` | 1 | Test suite: found `datatypes.py` and `screen.py` with HistoryBuf tests |
| `kittens/pager/` | 2 | Pager kitten: identified `main.go` (entry point), `file_input.go` (input reader) |
| `kitty/options/` | 2 | Configuration: found `definition.py` with scrollback option definitions |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 External References

No external URLs, Figma screens, or third-party documentation were referenced. All analysis is based entirely on the repository source code.

