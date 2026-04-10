# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative document** that comprehensively answers the user's questions about how kitty's internal subsystems behave when terminal graphics data arrives at a rate that exceeds the system's comfortable processing capacity. The document must trace the exact code paths responsible for buffering, flow control, backpressure, write-buffer management, storage quotas, and render-timing decisions, and explain how these mechanisms manifest at runtime when the terminal is under sustained graphics throughput pressure.

- **Request Category:** Create new documentation
- **Documentation Type:** Technical deep-dive / Q&A analysis document
- **Target Output:** A single Markdown file named `kitty_815df1e210e0.md` placed in the `blitzy/documentation/` directory of the destination repository

The user's questions decompose into the following concrete documentation requirements:

- **R-01 — Graphics Ingestion Under Load:** Explain how kitty handles large volumes of terminal graphics data arriving faster than the system can process it. Identify the exact code modules that implement buffering, chunked loading, and payload size limits.
- **R-02 — Buffer, Pause, and Throttle Decisions:** Document how the terminal decides whether to buffer, pause, or slow down data processing. Cover the VT parser's 1 MB ring buffer, the `input_delay` threshold, and the condition under which the buffer-full signal forces an early parse flush.
- **R-03 — Write-Back Under Pressure:** Explain what happens when the terminal needs to write response data (e.g., graphics acknowledgments) back to the child process while the output path is already congested. Cover the screen write buffer (`write_buf`), its 100 MB cap, POLLOUT-driven draining, and how undeliverable writes are handled.
- **R-04 — Code Locations:** Identify where these decisions live in the source code with file paths, function names, and line-number-level precision.
- **R-05 — Runtime Observability:** Describe the visible and measurable signs at runtime when the terminal shifts into pressure-handling behavior — including render-frame throttling, `repaint_delay` enforcement, synchronized-update pausing, and storage-quota eviction.
- **R-06 — Adaptation vs. Visibility:** Document whether the system's adaptation is silent (internal) or produces observable side effects (frame drops, eviction log messages, delayed acknowledgments).

### 0.1.2 Special Instructions and Constraints

- **Repository Immutability:** The user explicitly states the repository itself should remain unchanged. Temporary scripts may be created for observation but must be cleaned up afterward.
- **Implementation Rule — SWE-AtlasQnA-Repo:** The project rule requires creation of a new markdown document named `<source_branch_name>.md` (i.e., `kitty_815df1e210e0.md`) that comprehensively answers the questions posed. The document must be placed in the `blitzy/documentation` directory. No existing files in the source repository may be modified.
- **Evidence-Based Answers:** The rule mandates that answers must be based on the code as the source of truth, not assumptions. Thinking and rationale behind answers must be provided.
- **Cleanup Obligation:** Any temporary observation scripts must be removed after use, leaving no artifacts.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document graphics ingestion under load** (R-01), we will create a section analyzing `kitty/graphics.c` (payload ingestion in `load_image_data()`, `MAX_DATA_SZ` of 400 MB, chunked `g->more` loading, `DEFAULT_STORAGE_LIMIT` of 320 MB) and `kitty/parse-graphics-command.h` (APC command parsing and base64 decoding).
- To **document buffer/pause/throttle decisions** (R-02), we will create a section analyzing `kitty/vt-parser.c` (`BUF_SZ` = 1 MB ring buffer, `vt_parser_has_space_for_input()`, `input_delay` threshold in `run_worker()`, early flush when `self->read.sz + 16 * 1024 > BUF_SZ`).
- To **document write-back under pressure** (R-03), we will create a section analyzing `kitty/child-monitor.c` (`write_to_child()`, POLLOUT gating, `schedule_write_to_child()` with 100 MB cap) and `kitty/screen.c` (`write_buf`, `write_escape_code_to_child()` for graphics responses).
- To **map code locations** (R-04), we will provide a comprehensive table of file paths, function names, and specific mechanisms.
- To **document runtime observability** (R-05), we will analyze render-frame throttling (`render()` in `child-monitor.c`), `repaint_delay` enforcement, `PENDING_MODE 2026` synchronized updates (`screen_pause_rendering()`), and storage-quota eviction logging.
- To **document adaptation vs. visibility** (R-06), we will synthesize findings from all modules to characterize which behaviors are silent and which produce observable effects.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs were identified:

- **Disk Cache Pressure:** `kitty/disk-cache.c` implements a background writer thread with encrypted storage and defragmentation logic. Under graphics load, the disk cache plays a critical role in offloading frame data, and its behavior under pressure (hole tracking, write throttling, file truncation) should be documented.
- **Animation Frame Pressure:** `kitty/graphics.c` contains `scan_active_animations()` which drives animation frame advancement. Under heavy animated-image load, this creates render pressure that interacts with `repaint_delay` and render-frame scheduling.
- **Poll-Level Backpressure:** The I/O loop in `kitty/child-monitor.c` dynamically controls `POLLIN` based on `vt_parser_has_space_for_input()`. When the parser buffer is full, the system stops reading from the PTY entirely, creating kernel-level backpressure on the child process. This implicit flow-control mechanism is not documented anywhere and is critical to the user's question.
- **Graphics Response Congestion:** When `grman_handle_command()` returns a response string, it is written back via `write_escape_code_to_child()` into `screen->write_buf`. If the child process is not reading from its PTY, this buffer grows until the 100 MB cap, at which point further responses are silently dropped with a log warning.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation tree in `docs/` with comprehensive user-facing documentation, protocol specifications, and build infrastructure. However, no internal deep-dive documentation exists that addresses the specific questions about graphics data flow control, buffering, and backpressure under load.

- **Documentation framework:** Sphinx (version not pinned in `docs/requirements.txt`, theme: `furo`)
- **Documentation generator configuration:** `docs/conf.py` — Sphinx configuration with custom lexers, roles, and man-page hooks
- **Existing documentation files examined for relevance:**
  - `docs/graphics-protocol.rst` — Protocol specification for the Kitty Graphics Protocol. Covers wire format, transmission modes, and command structure but does **not** document internal buffering, flow control, or backpressure behavior.
  - `docs/performance.rst` — Covers `repaint_delay`, `input_delay`, `sync_to_monitor` options and benchmarking methodology. Provides useful context for render timing but does **not** explain internal buffer management or graphics-specific flow control.
  - `docs/conf.rst` — Configuration file format documentation.
  - `docs/faq.rst` — FAQ covering common user questions.
- **API documentation tools:** None in active use for C internals. The project relies on inline comments and Sphinx RST docs for user-facing documentation.
- **Diagram tools detected:** Mermaid diagrams are used in the tech spec; the existing docs use Sphinx image directives.
- **Documentation hosting:** The documentation is published via Sphinx to `sw.kovidgoyal.net/kitty/`.
- **Build driver:** `docs/Makefile` with `sphinx-build` and `sphinx-autobuild` for live preview.
- **Dependencies:** `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild` (from `docs/requirements.txt`).

### 0.2.2 Repository Code Analysis for Documentation

The following source code areas were searched and examined to establish the documentation foundation:

**Graphics Pipeline (Primary Focus):**
- `kitty/graphics.c` — Core graphics subsystem: payload ingestion, image storage, LRU eviction, animation, disk cache integration, response generation
- `kitty/graphics.h` — Data structures: `GraphicsCommand`, `Image`, `ImageRef`, `Frame`, `LoadData`, `GraphicsManager`
- `kitty/parse-graphics-command.h` — Generated APC parser: byte-oriented state machine for graphics commands with base64 payload decoding

**I/O and Buffering (Primary Focus):**
- `kitty/child-monitor.c` — Three-thread architecture (Main, I/O, Talk): PTY polling, `read_bytes()`, `write_to_child()`, `io_loop()`, `parse_input()`, render scheduling
- `kitty/vt-parser.c` — VT parser: 1 MB ring buffer (`BUF_SZ`), `run_worker()` with `input_delay` thresholding, `vt_parser_create_write_buffer()`, `vt_parser_has_space_for_input()`
- `kitty/vt-parser.h` — Parser API: `ParseData` structure with `input_read`, `write_space_created`, `has_pending_input`, `time_since_new_input`
- `kitty/screen.c` — Screen model: `write_buf` for responses, `screen_handle_graphics_command()`, `screen_pause_rendering()`, `write_escape_code_to_child()`
- `kitty/screen.h` — Screen structure: `write_buf`, `write_buf_sz`, `write_buf_used`, `write_buf_lock`, `vt_parser`

**Storage and Caching:**
- `kitty/disk-cache.c` — Background writer thread, encrypted on-disk cache, hole-tracking/defragmentation, lazy initialization
- `kitty/disk-cache.h` — DiskCache type API

**Configuration and Timing:**
- `kitty/state.h` — Global state singleton, `Options` with `repaint_delay`, `input_delay`, `sync_to_monitor`
- `kitty/options/definition.py` — Configuration schema: `repaint_delay` (default 10 ms), `input_delay` (default 3 ms), `sync_to_monitor` (default yes)
- `kitty/control-codes.h` — `PENDING_MODE 2026` for synchronized updates

**Rendering:**
- `kitty/child-monitor.c` — `render()`, `render_os_window()`, render-frame request/wait logic, animation scan scheduling

**Testing:**
- `kitty_tests/graphics.py` — Graphics protocol test suite using `send_command()` / `parse_response()` helpers
- `kitty_tests/parser.py` — VT parser test suite

**Key directories examined:**
- `kitty/` — Core application tree (C extensions, Python orchestration, GLSL shaders)
- `docs/` — Sphinx documentation tree
- `kitty_tests/` — Test suite
- `tools/cmd/benchmark/` — Go-based benchmark kitten for throughput measurement

### 0.2.3 Web Search Research Conducted

No web search was conducted, as the user's questions are entirely code-centric and the implementation rule explicitly requires basing answers on the code as the source of truth. All findings are derived from direct repository inspection.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage to comprehensively answer the user's questions:

**Module: `kitty/vt-parser.c` (VT Parser — Input Buffer Management)**
- Public APIs: `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `vt_parser_has_space_for_input()`, `run_worker()` (static, via `parse_worker`)
- Current documentation: No internal documentation exists; only the user-facing `docs/performance.rst` references `input_delay`
- Documentation needed: Detailed explanation of the 1 MB ring buffer, backpressure signaling, `input_delay` thresholding, and early flush when buffer nears capacity

**Module: `kitty/child-monitor.c` (Child Monitor — I/O Loop and Render Scheduling)**
- Public APIs: `io_loop()`, `read_bytes()`, `write_to_child()`, `parse_input()`, `do_parse()`, `render()`, `render_os_window()`
- Current documentation: Partial — `docs/performance.rst` describes `repaint_delay`/`input_delay` at a user level
- Documentation needed: Deep explanation of poll-based backpressure (POLLIN gating), write draining (POLLOUT), main-loop wakeup throttling, and render-frame timing

**Module: `kitty/graphics.c` (Graphics Engine — Storage and Ingestion)**
- Public APIs: `grman_handle_command()`, `load_image_data()`, `apply_storage_quota()`, `scan_active_animations()`, `grman_pause_rendering()`
- Current documentation: `docs/graphics-protocol.rst` covers wire format but not internal pressure handling
- Documentation needed: Chunked loading (`g->more`), `MAX_DATA_SZ` (400 MB), `DEFAULT_STORAGE_LIMIT` (320 MB), LRU eviction strategy, disk-cache offloading, animation frame advancement under load

**Module: `kitty/screen.c` (Screen Model — Write Buffer and Responses)**
- Public APIs: `write_to_child()`, `write_escape_code_to_child()`, `screen_handle_graphics_command()`, `screen_pause_rendering()`
- Current documentation: None for write buffer management; synchronized updates partially referenced in protocol docs
- Documentation needed: Write buffer lifecycle, 100 MB hard cap, response path for graphics acknowledgments, `PENDING_MODE 2026` freeze semantics

**Module: `kitty/disk-cache.c` (Disk Cache — Persistent Storage Under Pressure)**
- Public APIs: `add_to_disk_cache()`, `read_from_disk_cache()`, `clear_disk_cache()`, `disk_cache_wait_for_write()`
- Current documentation: None
- Documentation needed: Background writer thread, hole tracking, defragmentation trigger, write pacing under heavy graphics animation

**Module: `kitty/options/definition.py` (Configuration — Timing Parameters)**
- Relevant options: `repaint_delay`, `input_delay`, `sync_to_monitor`
- Current documentation: `docs/performance.rst` covers these at a user level
- Documentation needed: How these options interact with the flow-control and rendering subsystems under pressure

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented Internal Flow Control:** No existing document explains the poll-level backpressure mechanism where `vt_parser_has_space_for_input()` controls whether `POLLIN` is set for a child PTY fd. This is the primary flow-control mechanism under graphics load.
- **Undocumented Write Buffer Lifecycle:** The screen's `write_buf` (used for graphics command responses) has a 100 MB hard cap with silent data discard on overflow, but this is not documented anywhere.
- **Undocumented Storage Quota Enforcement:** While the graphics protocol documentation mentions storage, it does not explain the LRU eviction algorithm, the `trim_predicate` logic that first removes unreferenced images, or the `oldest_img_first` sorting strategy.
- **Undocumented Render Throttling Under Load:** The interaction between `repaint_delay`, `input_delay`, render-frame readiness, and the `set_maximum_wait()` mechanism is entirely internal and undocumented.
- **Undocumented Disk Cache Pressure Behavior:** The background writer thread, defragmentation heuristics, and XOR encryption of cached data are implementation details not visible in any documentation.
- **Undocumented `PENDING_MODE 2026` Interaction:** The synchronized update mechanism (`screen_pause_rendering`) with its expiration timer and snapshot-based freeze is referenced in protocol specs but its internal pressure-relief role is not documented.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/kitty_815df1e210e0.md` will be structured as a self-contained investigative analysis document with the following hierarchy:

```
blitzy/documentation/
└── kitty_815df1e210e0.md
    ├── Introduction (question restatement and scope)
    ├── Architecture Overview (threads, buffers, data paths)
    ├── VT Parser Buffer and Input Backpressure
    │   ├── The 1 MB Ring Buffer
    │   ├── input_delay Threshold and Early Flush
    │   └── Poll-Level POLLIN Gating
    ├── Graphics Data Ingestion Under Pressure
    │   ├── Chunked Payload Loading
    │   ├── MAX_DATA_SZ and Size Limits
    │   └── Storage Quota and LRU Eviction
    ├── Write-Back Path Under Output Congestion
    │   ├── Screen Write Buffer Lifecycle
    │   ├── POLLOUT-Driven Draining
    │   └── The 100 MB Cap and Silent Discard
    ├── Render Timing and Frame Throttling
    │   ├── repaint_delay Enforcement
    │   ├── Render Frame Readiness
    │   └── Synchronized Updates (PENDING_MODE 2026)
    ├── Disk Cache Under Sustained Load
    │   ├── Background Writer Thread
    │   └── Defragmentation and Hole Reuse
    ├── Runtime Observability
    │   ├── Silent Adaptations
    │   └── Observable Side Effects
    ├── Code Location Reference Table
    └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract buffer size constants and flow-control conditions from `kitty/vt-parser.c` (lines 18–21, 1420–1490)
- Extract I/O loop polling logic from `kitty/child-monitor.c` (lines 1490–1580)
- Extract storage quota enforcement from `kitty/graphics.c` (lines 25, 78, 290–300, 521, 2157–2184)
- Extract write buffer management from `kitty/child-monitor.c` (lines 323–365, 1443–1475) and `kitty/screen.c` (lines 104–115, 947–987)
- Extract render timing from `kitty/child-monitor.c` (lines 820–895) and `kitty/options/definition.py` (lines 866–900)
- Extract synchronized update mechanics from `kitty/screen.c` (lines 2490–2545) and `kitty/control-codes.h` (line 235)
- Generate examples by referencing test patterns in `kitty_tests/graphics.py`

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for data flow and thread interaction
- Code references using format: `Source: kitty/file.c:LineNumber`
- Tables for code location summaries and constant catalogs
- Thinking/rationale provided for all conclusions (per implementation rule)

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the document:

- **Thread Architecture Diagram:** Flowchart showing the three-thread model (Main, I/O, Talk) and how data flows between them, with buffer boundaries marked
- **Backpressure Flow Diagram:** Sequence diagram showing the poll-level POLLIN gating when the VT parser buffer fills, and how this propagates kernel-level backpressure to the child process
- **Graphics Ingestion Pipeline:** Flowchart from APC escape sequence arrival through parsing, buffering, storage quota check, eviction, and GPU upload
- **Write-Back Congestion Diagram:** Sequence diagram showing graphics command response generation, write buffer accumulation, POLLOUT draining, and overflow behavior
- **Render Timing Diagram:** Flowchart showing the interaction between `repaint_delay`, `input_delay`, render-frame readiness, and `set_maximum_wait()`

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/vt-parser.c`, `kitty/child-monitor.c`, `kitty/graphics.c`, `kitty/graphics.h`, `kitty/screen.c`, `kitty/screen.h`, `kitty/disk-cache.c`, `kitty/parse-graphics-command.h`, `kitty/state.h`, `kitty/options/definition.py`, `kitty/control-codes.h`, `kitty/vt-parser.h`, `kitty/loop-utils.c` | Complete investigative document answering all user questions about graphics data pressure handling, buffering, flow control, write-back congestion, render throttling, and runtime observability with code-referenced evidence |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Deep-Dive / Q&A Analysis Document
Source Code:
    - kitty/vt-parser.c (lines 18-21, 194, 1405-1490)
    - kitty/child-monitor.c (lines 323-365, 437-446, 1335-1580, 820-895)
    - kitty/graphics.c (lines 25, 78, 290-300, 519-625, 1543-1573, 1760-1800, 2157-2184)
    - kitty/graphics.h (lines 1-128)
    - kitty/screen.c (lines 104-115, 484-489, 947-987, 1047-1060, 2485-2545)
    - kitty/screen.h (lines 114-158)
    - kitty/disk-cache.c (full file — background writer, defrag)
    - kitty/parse-graphics-command.h (full file — APC parser)
    - kitty/state.h (lines 37, 51, 252, 269)
    - kitty/options/definition.py (lines 866-900)
    - kitty/control-codes.h (line 235)
    - kitty/vt-parser.h (lines 13-38)
    - kitty/loop-utils.c (wakeup mechanism)
Sections:
    - Introduction (question restatement and document scope)
    - Architecture Overview (three-thread model, buffer locations)
    - VT Parser Buffer and Input Backpressure
        - The 1 MB Ring Buffer (BUF_SZ, read/write regions)
        - input_delay Threshold and Early Flush logic
        - Poll-Level POLLIN Gating
    - Graphics Data Ingestion Under Pressure
        - Chunked Payload Loading (g->more, direct/file/shm modes)
        - MAX_DATA_SZ and Size Limits (400 MB max, 10000px dimension cap)
        - Storage Quota and LRU Eviction (320 MB default, oldest-first)
    - Write-Back Path Under Output Congestion
        - Screen Write Buffer Lifecycle (BUFSIZ initial, dynamic growth)
        - POLLOUT-Driven Draining
        - The 100 MB Cap and Silent Discard
    - Render Timing and Frame Throttling
        - repaint_delay Enforcement (10 ms default)
        - Render Frame Readiness (sync_to_monitor)
        - Synchronized Updates (PENDING_MODE 2026, 2-second expiration)
    - Disk Cache Under Sustained Load
        - Background Writer Thread
        - Defragmentation and Hole Reuse
    - Runtime Observability
        - Silent Adaptations (backpressure, eviction, throttling)
        - Observable Side Effects (frame drops, delayed acknowledgments, log messages)
    - Code Location Reference Table (all mechanisms with file:line citations)
    - Summary (synthesis answering the user's six questions)
Diagrams:
    - Thread architecture with buffer boundaries (Mermaid flowchart)
    - Backpressure propagation sequence (Mermaid sequence diagram)
    - Graphics ingestion pipeline (Mermaid flowchart)
    - Write-back congestion flow (Mermaid sequence diagram)
    - Render timing decision tree (Mermaid flowchart)
Key Citations:
    kitty/vt-parser.c, kitty/child-monitor.c, kitty/graphics.c,
    kitty/screen.c, kitty/disk-cache.c, kitty/state.h,
    kitty/options/definition.py, kitty/control-codes.h
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be created or modified. The output document is a standalone Markdown file placed in a new `blitzy/documentation/` directory. It does not integrate with the existing Sphinx documentation system, as the implementation rule specifies placement in `blitzy/documentation/` within the destination repo.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes:** The output document is self-contained
- **No navigation links required:** The document stands alone and does not integrate into the existing docs navigation
- **No table of contents updates:** The document is not part of the Sphinx tree
- **Source code citations:** All technical claims must include `Source: <file>:<line>` references back to the kitty repository

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation task is a standalone Markdown file creation exercise. No documentation build tools are required for the output itself. However, the following tools and packages are relevant to the documentation context:

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| pip | sphinx | (unpinned) | Project's existing documentation generator (`docs/requirements.txt`) |
| pip | furo | (unpinned) | Sphinx theme used by the project |
| pip | sphinx-copybutton | (unpinned) | Code block copy button in existing docs |
| pip | sphinxext-opengraph | (unpinned) | OpenGraph metadata for existing docs |
| pip | sphinx-inline-tabs | (unpinned) | Tabbed content in existing docs |
| pip | sphinx-autobuild | (unpinned) | Live preview for existing docs development |
| N/A | Mermaid | N/A | Diagrams embedded in Markdown (rendered by consumers) |

Since the output is a standalone `.md` file with embedded Mermaid diagram blocks, no build tool installation is required for this documentation task. The Mermaid diagrams will render natively in any Markdown viewer that supports the `mermaid` code block syntax (GitHub, GitLab, VS Code, etc.).

### 0.6.2 Project Runtime Context

The following runtime and build dependencies are documented here for completeness, as they define the environment in which the code under analysis operates:

| Component | Version Constraint | Source |
|-----------|-------------------|--------|
| Python | >= 3.8 | `pyproject.toml` (`requires-python = ">=3.8"`) |
| Go | 1.22 | `go.mod` (`go 1.22`) |
| C Standard | C11 | `setup.py` (enforced via `-std=c11`) |
| OpenGL | 3.3+ | Required by GPU rendering pipeline |
| GLFW | 3.4 (vendored fork) | `glfw/` directory |
| FreeType | System library | Required by font pipeline |

### 0.6.3 Documentation Reference Updates

No link updates are required. The output document `blitzy/documentation/kitty_815df1e210e0.md` is a new, self-contained file that does not modify or reference any existing documentation links.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the user's questions:**
- Internal buffering/flow control documented: 0/6 questions (0%) — no existing document addresses these topics at the internal code level
- User-facing performance tuning documented: 2/6 partially — `docs/performance.rst` covers `repaint_delay`/`input_delay` at a user level, and `docs/graphics-protocol.rst` describes the wire format

**Target coverage:** 100% of all six user questions answered with code-referenced evidence

| Question Area | Current Coverage | Target Coverage | Key Source Files |
|--------------|-----------------|-----------------|------------------|
| Graphics ingestion under load | 0% | 100% | `kitty/graphics.c`, `kitty/parse-graphics-command.h` |
| Buffer/pause/throttle decisions | 0% | 100% | `kitty/vt-parser.c`, `kitty/child-monitor.c` |
| Write-back under pressure | 0% | 100% | `kitty/child-monitor.c`, `kitty/screen.c` |
| Code locations | 0% | 100% | All source files in scope |
| Runtime observability | 0% | 100% | `kitty/child-monitor.c`, `kitty/screen.c`, `kitty/graphics.c` |
| Adaptation vs. visibility | 0% | 100% | Synthesis of all modules |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question must have a dedicated section with a clear answer
- All answers must cite specific source code file paths and line numbers
- Every buffer, threshold, and limit must have its exact value documented with the source constant name
- All Mermaid diagrams must accurately represent the code paths they illustrate
- Thinking and rationale must be provided behind each answer (per implementation rule)

**Accuracy validation:**
- All constant values (`BUF_SZ`, `MAX_DATA_SZ`, `DEFAULT_STORAGE_LIMIT`, `MAX_IMAGE_DIMENSION`, `PENDING_MODE`, default `input_delay`, default `repaint_delay`) must be verified against the source code
- All function names and file paths must be verified to exist in the repository
- All data flow descriptions must match the actual call chain in the code

**Clarity standards:**
- Progressive disclosure: start with high-level architecture, then dive into each subsystem
- Consistent terminology: use "VT parser buffer" not "read buffer" or "input buffer" interchangeably without definition
- Each code mechanism described with the pattern: "What it does → Where it lives → When it triggers → What the effect is"

**Maintainability:**
- All source citations include file path and line number for traceability
- The document is self-contained and does not depend on external documentation

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 5 Mermaid diagrams (thread architecture, backpressure propagation, graphics pipeline, write-back congestion, render timing)
- **Code reference table:** One comprehensive table mapping all mechanisms to file:line locations
- **Constants catalog:** One table listing all relevant buffer sizes, limits, timeouts, and quotas with their values and source locations
- **Visual content freshness:** All diagrams reflect the current state of the codebase as inspected

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive Q&A document

**Source code modules analyzed for documentation content (read-only):**
- `kitty/vt-parser.c` — VT parser ring buffer, input_delay thresholding, backpressure signaling
- `kitty/vt-parser.h` — Parser API: `ParseData`, buffer management functions
- `kitty/child-monitor.c` — I/O loop, poll-based backpressure, write draining, render scheduling
- `kitty/graphics.c` — Graphics engine: ingestion, storage quota, LRU eviction, animation, disk cache
- `kitty/graphics.h` — Graphics data structures and manager API
- `kitty/parse-graphics-command.h` — APC command parser with base64 decoding
- `kitty/screen.c` — Screen model: write buffer, graphics command handling, paused rendering
- `kitty/screen.h` — Screen structure: write_buf fields, vt_parser reference
- `kitty/disk-cache.c` — Persistent disk cache: background writer, defragmentation
- `kitty/disk-cache.h` — DiskCache type API
- `kitty/state.h` — Global state: Options, OSWindow, render timing fields
- `kitty/state.c` — State initialization: render_frames setup
- `kitty/options/definition.py` — Configuration schema: repaint_delay, input_delay, sync_to_monitor
- `kitty/control-codes.h` — PENDING_MODE constant (2026)
- `kitty/loop-utils.c` — Event loop wakeup mechanism
- `kitty/loop-utils.h` — LoopData structure

**Existing documentation reviewed for context (read-only):**
- `docs/performance.rst` — Performance tuning guide
- `docs/graphics-protocol.rst` — Graphics protocol specification
- `docs/requirements.txt` — Documentation tool dependencies
- `docs/Makefile` — Documentation build driver
- `docs/conf.py` — Sphinx configuration

**Test files reviewed for context (read-only):**
- `kitty_tests/graphics.py` — Graphics protocol test suite
- `kitty_tests/parser.py` — VT parser test suite

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the kitty repository will be modified. The implementation rule explicitly prohibits modifying existing files.
- **Test file modifications:** No test files will be created or modified.
- **Feature additions or code refactoring:** This is a documentation-only task.
- **Deployment configuration changes:** No CI/CD, packaging, or build system changes.
- **Sphinx documentation system changes:** The output document does not integrate into the existing Sphinx tree. No `docs/` files are modified.
- **Existing documentation updates:** No existing `.rst` or `.md` files in the repository are modified.
- **GPU shader analysis:** While the graphics shaders (`graphics_vertex.glsl`, `graphics_fragment.glsl`) render images, their internal behavior is not relevant to the data flow pressure questions.
- **Font pipeline analysis:** Font rendering is not part of the graphics data pressure path.
- **Remote control system:** Not relevant to graphics throughput pressure.
- **Shell integration:** Not relevant to the questions asked.
- **macOS-specific or Wayland-specific code paths:** Only generic/cross-platform flow control mechanisms are documented unless platform-specific behavior directly affects pressure handling.
- **Temporary observation scripts:** While the user mentions temporary scripts may be used, the documentation task itself does not require creating them. If any were created, they would need cleanup per user instructions.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — output is a standalone Markdown file, not part of the Sphinx build
- **Documentation preview command:** Any Markdown renderer (e.g., `grip`, VS Code preview, GitHub rendering)
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown and rendered by the consuming viewer
- **Documentation deployment command:** Not applicable — file is placed in `blitzy/documentation/` directory
- **Default format:** Markdown with embedded Mermaid diagrams
- **Citation requirement:** Every technical claim must include `Source: <file_path>:<line_or_range>` references
- **Style guide:** Technical deep-dive style with progressive disclosure, code-referenced evidence, and clear section headers
- **Documentation validation:** Manual review of all source citations against the repository; verify all referenced file paths and line numbers exist

### 0.9.2 Output File Specifications

- **File path:** `blitzy/documentation/kitty_815df1e210e0.md`
- **File name derivation:** Source branch name `kitty_815df1e210e0` with `.md` extension (per implementation rule)
- **Directory:** `blitzy/documentation/` — must be created if it does not exist
- **Format:** UTF-8 encoded Markdown
- **Estimated size:** 3,000–5,000 lines covering all six questions with full code citations, diagrams, and tables

### 0.9.3 Key Constants Catalog

The following constants are central to the document's technical content and must be accurately cited:

| Constant | Value | File | Line | Purpose |
|----------|-------|------|------|---------|
| `BUF_SZ` | 1,048,576 (1 MB) | `kitty/vt-parser.c` | 18 | VT parser ring buffer size |
| `BUF_EXTRA` | 64 bytes | `kitty/vt-parser.c` | 20 | Extra alignment bytes after buffer |
| `MAX_ESCAPE_CODE_LENGTH` | 262,144 (BUF_SZ/4) | `kitty/vt-parser.c` | 21 | Maximum single escape code length |
| `MAX_DATA_SZ` | 400,000,000 (~400 MB) | `kitty/graphics.c` | 521 | Maximum graphics payload data size |
| `MAX_IMAGE_DIMENSION` | 10,000 px | `kitty/graphics.c` | 674 | Maximum image width or height |
| `DEFAULT_STORAGE_LIMIT` | 335,544,320 (320 MB) | `kitty/graphics.c` | 25 | Default graphics storage quota per buffer |
| `PENDING_MODE` | 2026 | `kitty/control-codes.h` | 235 | Private mode for synchronized updates |
| `repaint_delay` default | 10 ms | `kitty/options/definition.py` | 866 | Delay between screen updates |
| `input_delay` default | 3 ms | `kitty/options/definition.py` | 878 | Delay before processing program input |
| Write buffer hard cap | 100 MB | `kitty/child-monitor.c` | 341 | Maximum screen write buffer size |
| Buffer-full flush threshold | BUF_SZ - 16 KB | `kitty/vt-parser.c` | 1425 | Triggers early parse when buffer nears capacity |

## 0.10 Rules for Documentation

The following rules are mandated by the user's instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The kitty codebase must remain unchanged. Only a new file in `blitzy/documentation/` may be created.
- **Create a new markdown document named `kitty_815df1e210e0.md`** in the `blitzy/documentation` directory in the destination repo. This file name is derived from the source branch name.
- **Provide thinking and rationale behind the answers.** Every conclusion must include the reasoning that led to it, not just the conclusion itself.
- **Do not make assumptions — base answers on the code as the source of truth.** All claims must be verifiable by reading the referenced source files and line numbers.
- **Temporary scripts may be used for observation but must be cleaned up afterward.** Any transient artifacts created during analysis must be removed, leaving no trace in the repository.
- **Include source code citations for all technical details.** Every technical claim must reference the specific file path and, where meaningful, line numbers.
- **Use Mermaid diagrams for complex data flows and thread interactions.** Visual representations must accompany textual explanations for multi-component interactions.
- **Maintain progressive disclosure structure.** Start with high-level architecture, then progressively dive into each subsystem's pressure-handling behavior.
- **Catalog all relevant constants, thresholds, and limits.** Provide a reference table of buffer sizes, caps, timeouts, and quotas with their exact values and source locations.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were directly inspected during the analysis phase to derive conclusions for this Agent Action Plan:

**Core I/O and Buffering:**
| File | Purpose of Inspection |
|------|----------------------|
| `kitty/child-monitor.c` | I/O loop, poll-based backpressure, write draining, render scheduling, input_delay wakeup throttling |
| `kitty/vt-parser.c` | VT parser ring buffer (BUF_SZ), input_delay thresholding, early flush, vt_parser_has_space_for_input() |
| `kitty/vt-parser.h` | Parser API: ParseData structure, buffer management function signatures |
| `kitty/loop-utils.c` | Event loop wakeup mechanism (eventfd/self-pipe) |
| `kitty/loop-utils.h` | LoopData definition |

**Graphics Pipeline:**
| File | Purpose of Inspection |
|------|----------------------|
| `kitty/graphics.c` | Graphics engine: payload ingestion, chunked loading, MAX_DATA_SZ, storage quota, LRU eviction, animation scanning, disk cache integration |
| `kitty/graphics.h` | Data structures: GraphicsCommand, Image, ImageRef, Frame, LoadData, GraphicsManager, storage_limit field |
| `kitty/parse-graphics-command.h` | APC command parser: byte-oriented state machine, base64 payload decoding |
| `kitty/png-reader.c` (summary only) | PNG decoding for graphics data |

**Screen and Write Buffer:**
| File | Purpose of Inspection |
|------|----------------------|
| `kitty/screen.c` | Screen model: write_buf management, screen_handle_graphics_command(), write_escape_code_to_child(), screen_pause_rendering(), PENDING_MODE handling |
| `kitty/screen.h` | Screen structure: write_buf, write_buf_sz, write_buf_used, write_buf_lock, vt_parser reference |

**State and Configuration:**
| File | Purpose of Inspection |
|------|----------------------|
| `kitty/state.h` | Global state singleton, Options structure (repaint_delay, input_delay, sync_to_monitor), render timing fields |
| `kitty/state.c` | State initialization: render_frames setup |
| `kitty/options/definition.py` | Configuration schema: repaint_delay (10ms default), input_delay (3ms default), sync_to_monitor (default yes) |
| `kitty/control-codes.h` | PENDING_MODE constant (2026) |
| `kitty/data-types.h` (summary only) | Core data type definitions |

**Storage and Caching:**
| File | Purpose of Inspection |
|------|----------------------|
| `kitty/disk-cache.c` | Background writer thread, encrypted on-disk cache, hole tracking, defragmentation, lazy initialization |
| `kitty/disk-cache.h` (summary only) | DiskCache type API |

**Documentation and Testing:**
| File | Purpose of Inspection |
|------|----------------------|
| `docs/performance.rst` | Existing performance documentation: repaint_delay, input_delay, sync_to_monitor, benchmark methodology |
| `docs/graphics-protocol.rst` | Existing graphics protocol documentation: wire format, transmission modes |
| `docs/requirements.txt` | Documentation tool dependencies |
| `docs/Makefile` | Documentation build driver |
| `docs/conf.py` | Sphinx configuration |
| `kitty_tests/graphics.py` | Graphics protocol test suite: send_command/parse_response patterns |
| `kitty_tests/parser.py` (identified) | VT parser test suite |

**Project Configuration:**
| File | Purpose of Inspection |
|------|----------------------|
| `pyproject.toml` | Python version requirement (>=3.8), tooling configuration |
| `go.mod` | Go version (1.22) and module dependencies |
| `setup.py` | Build system, C11 enforcement |

**Folders Explored:**
| Folder | Depth | Purpose |
|--------|-------|---------|
| `` (root) | Level 0 | Repository structure overview |
| `kitty/` | Level 1 | Core application tree — all C extensions, Python modules, GLSL shaders |
| `docs/` | Level 1 | Documentation tree — RST files, Sphinx config, assets |
| `kitty_tests/` | Level 1 | Test suite (graphics.py, parser.py identified) |
| `kittens/` | Level 1 | Built-in kittens listing |
| `tools/cmd/benchmark/` | Level 2 | Benchmark kitten source |

**Tech Spec Sections Retrieved:**
| Section | Purpose |
|---------|---------|
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | VT parser dispatch, GPU rendering pipeline, performance architecture |
| 4.9 PROTOCOL WORKFLOWS | Graphics protocol flow, storage and caching details, file transfer backpressure |
| 5.2 COMPONENT DETAILS | Child Monitor three-thread architecture, VT Parser dispatch architecture, GPU rendering pipeline |
| 2.1 Feature Catalog | Feature registry for F-001, F-002, F-013 context |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens or external URLs were referenced in the user's prompt.

