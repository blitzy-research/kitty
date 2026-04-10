# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new investigative architecture documentation** that comprehensively answers how the kitty terminal emulator moves data between its native C core and Python kittens under concurrent, high-load runtime conditions.

- **Category**: Create new documentation
- **Documentation type**: Architecture deep-dive / Technical Q&A document

The user seeks a new markdown document that traces the precise runtime mechanics of:

- **Data transfer pathways** — How clipboard data (of arbitrary size) crosses from internal C `Screen` structures (`kitty/screen.c`, `kitty/screen.h`) through the VT parser (`kitty/vt-parser.c`) and Python clipboard manager (`kitty/clipboard.py`) into Python objects accessible by kittens
- **Concurrency behavior** — How the three-thread architecture (Main thread, I/O thread in `kitty/child-monitor.c`, Talk thread) interacts with the VT parser's internal `pthread_mutex_t lock` and the Screen's `write_buf_lock` when multiple operations compete for attention
- **Event delivery under load** — Whether expensive main-thread operations (such as scrollback scanning via `screen_history_scroll`, `as_text_for_history_buf` in `kitty/screen.c`) affect the delivery cadence of events to kittens running as overlays
- **Object ownership and memory management** — Where Python reference counting (`Py_INCREF`/`Py_DECREF` on `Screen` objects in `child-monitor.c`) and the `Tempfile` rollover strategy (`kitty/clipboard.py`) create ownership boundaries, and where those boundaries become hazardous under real-world timing
- **Subtle race windows** — Where the VT parser's deliberate lock-unlock-relock pattern inside `run_worker()` (lines 1420–1445 of `kitty/vt-parser.c`), the `input_delay`-gated wakeup coalescing in the I/O loop, and the GIL-mediated callback dispatch from C to Python create windows for races or dropped events

### 0.1.2 Special Instructions and Constraints

- **No repository modifications**: The user explicitly states "the repository itself should remain unchanged." The implementation rule also requires: "Do not modify any existing files in the source repository."
- **Temporary scripts allowed**: Observation scripts are permitted but must be cleaned up afterward; however, since the output is a documentation artifact, no temporary scripts are needed for the final deliverable
- **Output placement**: Per the `SWE-AtlasQnA-Repo` rule, the generated document must be named `kitty_815df1e210e0.md` and placed in the `blitzy/documentation` directory
- **Evidence-based answers**: "Do not make assumptions, base your answers on the code as the truth"
- **Provide rationale**: "Provide thinking / rationale behind the answers"

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **data transfer pathway** from core to kittens, we will trace the full call chain from `read_bytes()` in `kitty/child-monitor.c` through `vt_parser_commit_write()` / `run_worker()` in `kitty/vt-parser.c`, through `dispatch_osc()` → `clipboard_control()` in `kitty/screen.c`, up to the Python `ClipboardRequestManager` in `kitty/clipboard.py`, and across to the kitten overlay via `kittens/runner.py` and the TUI loop in `kittens/tui/loop.py`
- To document **concurrency mechanics**, we will analyze the three distinct mutex domains: the `children_lock` (global child array protection), the `Screen.write_buf_lock` (per-screen write buffer synchronization), and the VT parser's internal `pthread_mutex_t lock` (per-parser read/write buffer serialization), and explain how the I/O thread, main thread, and talk thread coordinate through these
- To document **event delivery under load**, we will explain how `input_delay` gating in the I/O loop (lines 1506–1570 of `child-monitor.c`) defers main-loop wakeups, how `parse_input()` processes all children sequentially under a snapshot-and-release locking pattern, and how expensive Python callbacks (such as `display_scrollback` in `boss.py`) block the main thread's event processing
- To document **race windows**, we will analyze the VT parser's `run_worker()` pattern where the lock is released during `consume_input()` (line 1431–1433 of `vt-parser.c`) allowing the I/O thread to write concurrently, and how the `Clipboard.set_mime()` / `get_clipboard_mime()` calls cross the Python/C boundary under GIL protection but without additional synchronization

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs surface:

- The `Tempfile` class in `kitty/clipboard.py` implements a rollover strategy from `BytesIO` to a filesystem `TemporaryFile` when clipboard data exceeds 16 MB; this memory management boundary is architecturally significant and must be documented
- The `SharedMemory` class in `kitty/shm.py` wraps POSIX shared memory for cross-process data sharing with `mmap`, but is not directly used in the clipboard pathway — this distinction must be documented to avoid confusion
- The kitten result serialization protocol (JSON → Base85 → DCS escape sequence) in `kittens/runner.py` (lines 96–106) represents a separate data channel from clipboard transport and must be distinguished
- The `WriteRequest.commit()` method in `kitty/clipboard.py` (lines 258–269) creates chunker closures that hold references to the `Tempfile` object, creating a deferred-read ownership pattern that persists beyond the original write request lifecycle
- The `screen.send_escape_code_to_child()` calls during clipboard fulfillment (e.g., lines 476–499 of `kitty/clipboard.py`) write data into the Screen's `write_buf` which is consumed by the I/O thread — this cross-thread data flow is a key concurrency boundary

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx/reStructuredText documentation system** with comprehensive user-facing protocol documentation and no existing architecture-internals documentation of the kind requested.

- **Documentation framework**: Sphinx (version unpinned in `docs/requirements.txt`, installed via `pip install sphinx`)
- **Theme**: Furo (`furo` in `docs/requirements.txt`)
- **Documentation generator configuration**: `docs/conf.py`
- **Build driver**: `docs/Makefile` (supports `make html`, `make develop-docs` for live preview)
- **Dependencies** (from `docs/requirements.txt`): `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
- **Diagram tools detected**: No Mermaid integration in Sphinx setup; diagrams in the tech spec use Mermaid but the existing docs tree uses inline ASCII protocol notation
- **API documentation tools**: None detected; no JSDoc, Sphinx autodoc, or Godoc configurations found for source-level API docs

**Key existing documentation relevant to this task:**

| Document | Path | Relevance |
|----------|------|-----------|
| Clipboard protocol spec | `docs/clipboard.rst` | Defines the OSC 52 / OSC 5522 wire protocol for clipboard transport |
| Kittens introduction | `docs/kittens_intro.rst` | Describes kitten concept and usage but not internal data flow |
| Performance guide | `docs/performance.rst` | Discusses rendering performance but not concurrency internals |
| Clipboard kitten docs | `docs/kittens/clipboard.rst` (inferred from `kittens/clipboard/main.py`) | User-facing clipboard kitten usage |
| Protocol extensions | `docs/protocol-extensions.rst` | Covers terminal protocol extensions at the wire level |

**Gap identified**: No existing documentation covers the internal runtime data-transfer mechanics between the C core and Python kittens, the concurrency model's mutex hierarchy, or the timing implications of clipboard operations under load.

### 0.2.2 Repository Code Analysis for Documentation

The following code areas were analyzed to support documentation of the user's questions:

**Core data transfer and concurrency files examined:**

| File | Purpose in Context | Lines |
|------|-------------------|-------|
| `kitty/child-monitor.c` | Three-thread event loop: I/O thread polling, main-thread parsing, talk-thread peer management | 2016 |
| `kitty/vt-parser.c` | VT parser state machine with internal `pthread_mutex_t` lock for thread-safe read/write buffer management | ~1500 |
| `kitty/vt-parser.h` | ParseData struct defining `input_read`, `write_space_created`, `time_since_new_input` | 39 |
| `kitty/screen.c` | Screen model with `clipboard_control()` dispatch, `send_escape_code_to_child()`, `as_text` family | 4932 |
| `kitty/screen.h` | Screen struct with `write_buf`, `write_buf_lock`, `paused_rendering`, `historybuf` | 289 |
| `kitty/clipboard.py` | Python clipboard manager: `Clipboard`, `WriteRequest`, `ReadRequest`, `ClipboardRequestManager` | 542 |
| `kitty/data-types.h` | Foundational C type definitions: `CPUCell`, `GPUCell`, `Line`, `LineBuf`, `HistoryBuf` | 438 |
| `kitty/state.h` | `GlobalState`, `OSWindow`, `Window`, `Tab` structs and `call_boss` macro | 401 |
| `kitty/threading.h` | Thread naming helper (`set_thread_name`) for `KittyChildMon`, `KittyWriteStdin` | 37 |
| `kitty/loop-utils.h` | `LoopData` struct for signal/wakeup file descriptors, `drain_fd()`, `self_pipe()` | 88 |
| `kitty/shm.py` | POSIX shared memory wrapper (not on clipboard path but architecturally adjacent) | 186 |
| `kitty/kittens.c` | C extension: `parse_input_from_terminal()` and `read_command_response()` for kitten I/O | 209 |
| `kittens/runner.py` | Kitten launcher: module resolution, execution, result serialization via DCS escape | 202 |
| `kittens/tui/loop.py` | TUI event loop: `TermManager`, `Loop`, raw TTY management, `selectors`-based I/O | ~500 |
| `kittens/tui/handler.py` | Base `Handler` class for kitten lifecycle events | ~200 |
| `kittens/clipboard/` | Clipboard kitten: Go implementation with `main.go`, `read.go`, `write.go`, `legacy.go` | Mixed |
| `kitty/boss.py` | Boss controller: `run_kitten_with_metadata()`, `display_scrollback()`, clipboard accessors | 3094 |
| `kitty/window.py` | Window: `clipboard_control()`, `ClipboardRequestManager` instantiation, `as_text()` | ~1800 |

### 0.2.3 Web Search Research Conducted

No web searches were required for this task — the user's questions are answerable entirely from code analysis. The existing tech spec sections (4.3, 4.8, 5.2) provide additional architectural context that was retrieved and cross-referenced.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation analysis to answer the user's questions. Each maps directly to one or more aspects of the data transfer, concurrency, and memory management topics.

**Module: `kitty/child-monitor.c` — Three-Thread Event Loop**
- Public APIs: `io_loop()`, `parse_input()`, `read_bytes()`, `write_to_child()`, `schedule_write_to_child()`
- Current documentation: No internal architecture docs exist
- Documentation needed: Thread model explanation, mutex hierarchy (`children_lock`, `talk_lock`, `screen.write_buf_lock`), I/O polling loop with `input_delay` gating, wakeup coalescing semantics

**Module: `kitty/vt-parser.c` — VT Parser State Machine**
- Public APIs: `run_worker()`, `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `vt_parser_has_space_for_input()`
- Current documentation: Protocol-level docs in `docs/clipboard.rst` but no internal concurrency docs
- Documentation needed: The lock-release-relock pattern in `run_worker()`, how I/O thread writes interleave with main-thread parsing, buffer capacity back-pressure (`BUF_SZ = 1MB`)

**Module: `kitty/clipboard.py` — Python Clipboard Manager**
- Public APIs: `Clipboard.set_text()`, `Clipboard.set_mime()`, `Clipboard.get_mime()`, `ClipboardRequestManager.parse_osc_5522()`, `ClipboardRequestManager.parse_osc_52()`, `WriteRequest.commit()`, `Tempfile` rollover
- Current documentation: Protocol-level in `docs/clipboard.rst`; no internal-architecture docs
- Documentation needed: Memory management (BytesIO → TemporaryFile rollover at 16 MB), chunked transfer via 4096-byte segments, deferred-read chunker closures, `max_size` enforcement

**Module: `kitty/screen.c` / `kitty/screen.h` — Screen Model**
- Public APIs: `clipboard_control()`, `send_escape_code_to_child()`, `as_text()`, `as_text_for_history_buf()`, `screen_history_scroll()`
- Current documentation: No internal architecture docs
- Documentation needed: How `clipboard_control()` dispatches to Python via `CALLBACK` macro, how `send_escape_code_to_child()` writes to `write_buf` (protected by `write_buf_lock`), impact of `as_text` operations on main-thread blocking

**Module: `kittens/runner.py` + `kittens/tui/loop.py` — Kitten Execution**
- Public APIs: `launch()`, `run_kitten()`, `create_kitten_handler()`, `Loop`, `TermManager`, `Handler`
- Current documentation: User-facing kitten docs exist; no internal data-flow docs
- Documentation needed: How kittens receive data (via child PTY overlay, not shared memory), result serialization protocol (JSON → Base85 → DCS escape), TTY raw-mode event loop isolation

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented concurrency architecture**: The three-thread model in `child-monitor.c` with its mutex hierarchy, wakeup coalescing, and snapshot-and-release locking pattern has zero internal documentation
- **Undocumented data-transfer pathway**: The full chain from child PTY → VT parser buffer → OSC dispatch → Python `clipboard_control()` → `ClipboardRequestManager` → `Screen.write_buf` → I/O thread → child PTY is not documented anywhere
- **Undocumented memory management boundaries**: The `Tempfile` rollover strategy, the chunker closure ownership pattern, and the `write_buf` growth/shrink logic are implementation details with no docs
- **Undocumented race windows**: The VT parser's deliberate lock-unlock-relock in `run_worker()`, the GIL-mediated C→Python callback dispatch, and the `input_delay` wakeup coalescing create subtle timing windows that are not documented
- **Missing kitten data isolation documentation**: How kittens run as separate child processes with their own PTY (overlay windows), receiving data through escape sequences rather than shared memory, is not explained in any architecture doc

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document placed at:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
```

The document will follow a logical Q&A investigation structure with embedded architecture diagrams:

```
kitty_815df1e210e0.md
├── Introduction and Context
├── Q1: How does clipboard data transfer from core to Python kittens?
│   ├── The full data pathway (VT parser → OSC dispatch → clipboard.py)
│   ├── Clipboard data size handling (BytesIO → TemporaryFile rollover)
│   └── Chunked response delivery (4096-byte segments via write_buf)
├── Q2: How does the three-thread concurrency model work?
│   ├── Thread roles (Main, I/O, Talk)
│   ├── Mutex hierarchy (children_lock, write_buf_lock, parser lock)
│   └── Wakeup coalescing and input_delay gating
├── Q3: Does expensive scrollback scanning affect event delivery to kittens?
│   ├── Main-thread blocking during as_text / history operations
│   ├── Impact on parse_input() scheduling
│   └── Kitten overlay isolation via separate child process
├── Q4: Where do timing, concurrency, and object ownership matter?
│   ├── VT parser lock-release-relock pattern in run_worker()
│   ├── Python reference counting across threads
│   ├── WriteRequest chunker closure lifecycle
│   └── GIL interaction with C mutex acquisition
├── Q5: How might subtle races emerge under real runtime conditions?
│   ├── I/O thread write during parser lock release window
│   ├── Clipboard self-offer race on concurrent read/write
│   ├── input_delay wakeup coalescing and event batching
│   └── Paused rendering vs. clipboard fulfillment timing
└── Conclusion and Key Takeaways
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract mutex patterns from `kitty/child-monitor.c` by analyzing `children_mutex(lock/unlock)` and `screen_mutex(lock/unlock, write)` usage
- Extract the VT parser lock protocol from `kitty/vt-parser.c` `run_worker()` (lines 1416–1446)
- Extract clipboard data flow from `kitty/clipboard.py` `ClipboardRequestManager.parse_osc_5522()` and `fulfill_read_request()`
- Extract kitten execution model from `kittens/runner.py` `launch()` and `kitty/boss.py` `run_kitten_with_metadata()`
- Extract memory management strategy from `kitty/clipboard.py` `Tempfile` class and `WriteRequest.commit()`

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for thread interaction and data-flow sequences
- Code references using inline citations: `Source: kitty/child-monitor.c:1480`
- Tables for structured comparisons (mutex hierarchy, thread responsibilities)
- All technical claims grounded in specific file paths and line numbers

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created:

- **Sequence diagram**: Full clipboard data transfer from child PTY through VT parser → OSC dispatch → Python `clipboard_control()` → `ClipboardRequestManager` → `write_buf` → I/O thread → child PTY
- **Flowchart**: Three-thread architecture showing Main thread, I/O thread, and Talk thread with their mutex domains
- **Sequence diagram**: The VT parser's `run_worker()` lock-release-relock protocol showing interleaving with I/O thread writes
- **Flowchart**: Memory management decision tree for clipboard data (BytesIO vs TemporaryFile, chunker closures)
- **Sequence diagram**: How kitten overlays receive events and how main-thread blocking affects delivery

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/vt-parser.h`, `kitty/clipboard.py`, `kitty/screen.c`, `kitty/screen.h`, `kitty/data-types.h`, `kitty/state.h`, `kitty/threading.h`, `kitty/loop-utils.h`, `kitty/shm.py`, `kitty/kittens.c`, `kitty/boss.py`, `kitty/window.py`, `kittens/runner.py`, `kittens/tui/loop.py`, `kittens/tui/handler.py`, `kittens/clipboard/main.go`, `kittens/clipboard/read.go`, `kittens/clipboard/write.go`, `kittens/clipboard/legacy.go`, `docs/clipboard.rst` | Comprehensive investigative Q&A document answering how kitty transfers data between core and kittens, clipboard handling under concurrency, memory management, event delivery timing, and subtle race conditions. Includes Mermaid diagrams, source citations, and rationale. |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Architecture Investigation / Technical Q&A
Source Code: kitty/child-monitor.c, kitty/vt-parser.c, kitty/clipboard.py,
             kitty/screen.c, kitty/screen.h, kitty/boss.py, kitty/window.py,
             kittens/runner.py, kittens/tui/loop.py, kittens/tui/handler.py
Sections:
    - Introduction and Context (purpose, scope, methodology)
    - Data Transfer Pathway (VT parser → OSC dispatch → clipboard.py → write_buf → PTY)
    - Three-Thread Concurrency Model (Main/I/O/Talk threads, mutex hierarchy)
    - Event Delivery Under Load (input_delay gating, main-thread blocking, kitten isolation)
    - Object Ownership and Memory Management (Tempfile rollover, chunker closures, refcounts)
    - Subtle Race Conditions (parser lock-release, GIL interaction, self-offer races)
    - Conclusion and Key Takeaways
Diagrams:
    - Sequence diagram: Clipboard data transfer end-to-end
    - Flowchart: Three-thread architecture with mutex domains
    - Sequence diagram: VT parser run_worker() lock protocol
    - Flowchart: Clipboard memory management decision tree
Key Citations:
    kitty/child-monitor.c (lines 1480-1578: io_loop, lines 438-538: parse_input)
    kitty/vt-parser.c (lines 1416-1446: run_worker, lines 1450-1484: write buffer API)
    kitty/clipboard.py (lines 26-66: Tempfile, lines 82-152: Clipboard, lines 233-329: WriteRequest, lines 332-542: ClipboardRequestManager)
    kitty/screen.c (line 2305-2307: clipboard_control dispatch)
    kitty/screen.h (lines 114-116: write_buf and write_buf_lock)
    kitty/boss.py (lines 1848-1879: display_scrollback, lines 1889-1976: run_kitten_with_metadata)
    kittens/runner.py (lines 87-106: launch and result serialization)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output document is a standalone Markdown file placed in `blitzy/documentation/`, which does not interact with the Sphinx documentation build system in `docs/`.

### 0.5.4 Cross-Documentation Dependencies

- The new document references the OSC 5522 protocol specification documented in `docs/clipboard.rst` for wire-format context
- No navigation links, table-of-contents updates, or index modifications are needed since the output is in a separate `blitzy/documentation/` directory
- No shared content/includes are affected

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This task produces a standalone Markdown file with embedded Mermaid diagram syntax. No documentation framework installation is required because the output is not processed by the repository's Sphinx build system. The file is consumed directly by Markdown renderers that support fenced code blocks (e.g., GitHub, GitLab, VS Code).

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | 7.2.x (repo-bundled) | Existing documentation build system in `docs/` — not used for this task |
| N/A | Mermaid (fenced blocks) | N/A | Diagrams are authored as `mermaid` fenced code blocks; rendering is handled by the consuming Markdown viewer |
| pip | Python (runtime) | 3.12.3 | Repository runtime used during code analysis; not a build dependency for the output document |

No additional packages need to be installed to produce or validate the output document. All diagrams use standard Mermaid syntax renderable by GitHub-flavored Markdown, GitLab Markdown, or any compatible Markdown viewer.

### 0.6.2 Documentation Reference Updates

Not applicable. The new document is placed in `blitzy/documentation/`, which is outside the existing documentation tree (`docs/`). No link updates, cross-references, or navigation changes to existing files are required.

The implementation rule explicitly states: "Do not modify any existing files in the source repository." Therefore, no existing documentation links will be altered.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's prompt poses five interconnected questions. Coverage is measured by how thoroughly each question is answered with evidence from source code.

- **Question coverage target: 5/5 (100%)**

| Question Domain | Source Modules | Current Documentation | Target Coverage |
|----------------|----------------|----------------------|-----------------|
| Data transfer between core and kittens | `kitty/child-monitor.c`, `kittens/runner.py`, `kittens/tui/loop.py` | None — existing docs cover user-facing behavior only | 100% — full pathway trace from C structs to Python objects |
| Clipboard data crossing internal structures into Python | `kitty/clipboard.py`, `kitty/screen.c`, `kitty/window.py`, `kitty/vt-parser.c` | `docs/clipboard.rst` covers protocol syntax; no internals documentation | 100% — end-to-end OSC flow with memory management details |
| Concurrency impact on event delivery | `kitty/child-monitor.c` (three threads), `kitty/vt-parser.c` (lock protocol) | None | 100% — mutex hierarchy, lock-release pattern, GIL interaction |
| Effect of expensive scrollback operations on kittens | `kitty/boss.py` (`display_scrollback`), `kitty/screen.h` (`historybuf`) | None | 100% — main-thread blocking analysis and kitten isolation model |
| Timing, object ownership, and subtle races | All concurrency-relevant files above | None | 100% — race window enumeration with trigger conditions |

- **Coverage gaps to address:**
  - Concurrency internals: Currently 0% documented externally, target 100% for the identified question domains
  - Clipboard data flow: Currently only protocol syntax documented, target 100% for internal pathway
  - Kitten isolation model: Currently 0% documented at architecture level, target 100% for process-boundary analysis

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every claim must cite a specific source file and line number or line range
  - Each of the five question domains must be addressed with at least one dedicated section
  - All public C functions and Python methods in the data transfer pathway must be named and their roles explained
  - All mutex instances (`children_lock`, `write_buf_lock`, VT parser internal lock) must be cataloged with acquisition order

- **Accuracy validation:**
  - Every function name, struct field, and constant value referenced in the document must match the current codebase (verified through `read_file` during analysis)
  - Buffer sizes (e.g., `BUF_SZ = 1024*1024` in `vt-parser.c`) must be stated with exact values, not approximations
  - Thread identities (Main, I/O `KittyChildMon`, Talk) must match the names set via `set_thread_name()` in `child-monitor.c`

- **Clarity standards:**
  - Progressive disclosure: begin each section with a high-level summary, then drill into implementation details
  - Use Mermaid diagrams to illustrate all multi-step flows (minimum 4 diagrams)
  - Provide explicit "thinking / rationale" blocks explaining why each answer follows from the code evidence, per the implementation rule
  - Define all non-obvious terms on first use (e.g., "GIL," "OSC 52," "OSC 5522," "DCS")

- **Maintainability:**
  - Source citations include file path and line numbers for traceability
  - Sections are self-contained to allow selective updates if the codebase evolves

### 0.7.3 Diagram Requirements

| Diagram Type | Subject | Source Files | Purpose |
|-------------|---------|--------------|---------|
| Sequence diagram | Clipboard write: child → PTY → VT parser → Python → write_buf → child | `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/clipboard.py` | Trace the complete round-trip of a clipboard write operation |
| Flowchart | Three-thread architecture with mutex domains | `kitty/child-monitor.c`, `kitty/screen.h`, `kitty/vt-parser.c` | Show thread responsibilities and shared-state boundaries |
| Sequence diagram | VT parser `run_worker()` lock-release protocol | `kitty/vt-parser.c` (lines 1416-1446) | Illustrate the deliberate interleaving window between I/O and main threads |
| Flowchart | Clipboard memory management: BytesIO → Tempfile rollover → chunker closure | `kitty/clipboard.py` (lines 26-66, 233-329) | Show object ownership transitions and deferred-read lifecycle |
| Sequence diagram | Kitten launch and result return | `kitty/boss.py`, `kittens/runner.py`, `kittens/tui/loop.py` | Show process-boundary data transfer and result serialization |

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — the sole deliverable

- **Source files analyzed for content extraction (read-only):**
  - `kitty/child-monitor.c` — three-thread event loop, mutex usage, I/O loop, parse_input
  - `kitty/vt-parser.c` — parser lock protocol, run_worker, write buffer API
  - `kitty/vt-parser.h` — parser struct declarations
  - `kitty/clipboard.py` — Clipboard class, Tempfile rollover, WriteRequest, ReadRequest, ClipboardRequestManager
  - `kitty/screen.c` — clipboard_control dispatch via CALLBACK macro, send_escape_code_to_child
  - `kitty/screen.h` — Screen struct with write_buf, write_buf_lock, historybuf, linebuf
  - `kitty/data-types.h` — CPUCell, GPUCell, Line, LineBuf, HistoryBuf struct definitions
  - `kitty/state.h` — GlobalState, OSWindow, Window structs, Options fields
  - `kitty/threading.h` — set_thread_name helper
  - `kitty/loop-utils.h` — LoopData, wakeup/signal fd helpers
  - `kitty/shm.py` — POSIX shared memory wrapper (architectural context)
  - `kitty/kittens.c` — read_command_response, parse_input_from_terminal
  - `kitty/boss.py` — run_kitten_with_metadata, display_scrollback, clipboard instances
  - `kitty/window.py` — clipboard_control dispatch, clipboard_request_manager instantiation
  - `kittens/runner.py` — launch, result serialization, kitten import mechanism
  - `kittens/tui/loop.py` — TermManager, selectors-based I/O, raw TTY mode
  - `kittens/tui/handler.py` — Handler base class
  - `kittens/clipboard/main.go` — Go-based clipboard kitten entry point
  - `kittens/clipboard/read.go` — clipboard read operations
  - `kittens/clipboard/write.go` — clipboard write operations
  - `kittens/clipboard/legacy.go` — legacy OSC 52 support

- **Existing documentation referenced (read-only):**
  - `docs/clipboard.rst` — OSC 5522 protocol specification
  - `docs/kittens_intro.rst` — kitten framework introduction
  - `docs/performance.rst` — performance characteristics

- **Temporary scripts** (if created for observation, must be cleaned up afterward)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — the implementation rule states: "Do not modify any existing files in the source repository"
- **Test file modifications** — no test files will be created or altered
- **Existing documentation updates** — files in `docs/` are reference-only; no edits to Sphinx sources
- **Feature additions or code refactoring** — this is a pure documentation exercise
- **Deployment configuration changes** — no CI/CD, Makefile, or build system changes
- **Documentation outside the specified question domains** — GPU rendering internals, font subsystem, GLFW windowing, configuration parsing, remote control authorization, and shell integration are not covered unless directly relevant to the five question domains
- **Shared memory (`kitty/shm.py`) deep analysis** — mentioned for architectural context only; it is not on the clipboard data path
- **Go tooling internals** — the Go clipboard kitten binary (`kittens/clipboard/`) handles user-facing clipboard I/O over the kitty protocol; its internal implementation is out of scope unless it illuminates the C-to-Python transfer question

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file not processed by any build system
- **Documentation preview command:** Any GitHub-flavored Markdown viewer or `grip blitzy/documentation/kitty_815df1e210e0.md` for local browser preview
- **Diagram generation command:** Mermaid diagrams are embedded as fenced code blocks (`mermaid`); rendering is delegated to the consuming viewer. No offline generation step is required.
- **Documentation deployment command:** Not applicable — file is committed directly to `blitzy/documentation/`
- **Default format:** Markdown with Mermaid diagram blocks
- **Citation requirement:** Every technical claim must reference a source file path and line number or range (e.g., `Source: kitty/child-monitor.c:1480-1495`)
- **Style guide:** Q&A investigative format with thinking/rationale blocks, per the implementation rule: "Provide thinking / rationale behind the answers"
- **Documentation validation:** Manual review — verify that all referenced function names, struct fields, constants, and line numbers match the current codebase

### 0.9.2 Output Directory Setup

The target directory `blitzy/documentation/` does not yet exist in the repository. It must be created before writing the output file:

```bash
mkdir -p blitzy/documentation
```

### 0.9.3 Cleanup Requirements

Per the user's explicit instruction: "Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward."

- Any temporary observation scripts created during analysis must be removed before task completion
- The only persistent artifact is `blitzy/documentation/kitty_815df1e210e0.md`
- No modifications to any pre-existing file in the repository are permitted

## 0.10 Rules for Documentation

The following rules govern this documentation task. They are drawn from the user's explicit instructions and from the project-level implementation rules.

- **Do not modify any existing files in the source repository.** The repository must remain in its original state. The sole new file permitted is `blitzy/documentation/kitty_815df1e210e0.md`.
- **Provide thinking and rationale behind the answers.** Every conclusion must include an explanation of why it follows from the code evidence. Do not state conclusions without supporting reasoning.
- **Do not make assumptions; base answers on the code as the truth.** All claims about behavior, timing, memory management, and concurrency must be traceable to specific source lines. Speculation must be explicitly labeled as such.
- **Place the generated document in the `blitzy/documentation` directory.** The file must be named `kitty_815df1e210e0.md` (matching the source branch name).
- **Create a new markdown document that comprehensively answers the question(s) posed in the prompt.** The document must address all five question domains: data transfer pathways, clipboard handling, concurrency impact on event delivery, effect of expensive operations on kittens, and subtle race conditions.
- **Temporary scripts may be used for observation, but anything temporary should be cleaned up afterward.** If any helper scripts are created during analysis, they must be deleted before the task concludes.
- **Include source citations for all technical details.** Every referenced function, struct, constant, or behavioral claim must cite the source file and line number.
- **Use Mermaid diagrams for all multi-step workflows.** Complex data flows, thread interactions, and state transitions must be illustrated with embedded Mermaid diagram blocks.
- **Maintain consistent terminology from the codebase.** Use the exact names found in source code (e.g., `children_lock`, `write_buf_lock`, `run_worker`, `consume_input`, `clipboard_control`, `Tempfile`, `ClipboardRequestManager`) rather than paraphrases.

## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were systematically explored to derive the conclusions and analysis in this Agent Action Plan.

**Core C Source Files:**

| File Path | Lines | Purpose in Analysis |
|-----------|-------|---------------------|
| `kitty/child-monitor.c` | 2016 | Three-thread event loop (`io_loop`, `parse_input`, `process_global_state`), mutex usage (`children_lock`, `write_buf_lock`), `schedule_write_to_child`, `write_to_child`, `read_bytes`, snapshot-and-release pattern |
| `kitty/vt-parser.c` | ~1500 | VT parser lock protocol (`run_worker` lock-release-relock), `consume_input` dispatch, `vt_parser_create_write_buffer`, `vt_parser_commit_write`, 1 MB circular buffer (`BUF_SZ`) |
| `kitty/screen.c` | 4932 | `clipboard_control` dispatch via `CALLBACK` macro (line 2305), `send_escape_code_to_child` (line 4464) |
| `kitty/screen.h` | 289 | `Screen` struct definition with `write_buf`, `write_buf_lock`, `historybuf`, `linebuf`, `vt_parser`, `callbacks` fields |
| `kitty/data-types.h` | 438 | `CPUCell`, `GPUCell`, `Line`, `LineBuf`, `HistoryBuf`, `Cursor`, `ColorProfile` struct definitions; `MAX_CHILDREN = 512` |
| `kitty/state.h` | 401 | `GlobalState`, `OSWindow`, `Window`, `Options` structs; `repaint_delay`, `input_delay`, `scrollback_pager_history_size` |
| `kitty/threading.h` | 37 | `set_thread_name` helper for platform-portable pthread naming |
| `kitty/loop-utils.h` | 88 | `LoopData` struct with wakeup/signal fds; `drain_fd`, `self_pipe` |
| `kitty/kittens.c` | 209 | `read_command_response` (DCS `@kitty-cmd` parsing with timeout), `parse_input_from_terminal` (escape sequence dispatch) |

**Python Source Files:**

| File Path | Lines | Purpose in Analysis |
|-----------|-------|---------------------|
| `kitty/clipboard.py` | 542 | `Clipboard` class (set_text, set_mime, get_mime, is_self_offer RuntimeError), `Tempfile` (BytesIO → TemporaryFile rollover), `WriteRequest` (base64 decoding, commit creating chunker closures), `ReadRequest` (4096-byte chunked responses), `ClipboardRequestManager` (OSC 52/5522 dispatch, in_flight_write_request pattern) |
| `kitty/boss.py` | 3094 | `run_kitten_with_metadata` (line 1889 — kitten overlay creation, `w.as_text()` for stdin), `display_scrollback` (line 1848 — scrollback piped to pager as overlay), `clipboard`/`primary_selection` as `Clipboard()` instances |
| `kitty/window.py` | ~2000 | `clipboard_control` (line 1391 — OSC 5522/52 dispatch), `clipboard_request_manager` instantiation (line 588) |
| `kitty/shm.py` | 186 | POSIX shared memory via `shm_open`/`shm_unlink`/`mmap`; `/kitty-` prefix naming; architectural context only |

**Kitten Framework Files:**

| File Path | Lines | Purpose in Analysis |
|-----------|-------|---------------------|
| `kittens/runner.py` | 202 | `launch()` (kitten `main()` execution), result serialization (JSON → Base85 via `\x1bP@kitty-kitten-result|...\x1b\\`), `create_kitten_handler`, `import_kitten_main_module` |
| `kittens/tui/loop.py` | ~800 | `TermManager` (raw TTY, `selectors`-based I/O), `init_state()` escape sequences, kitten event loop |
| `kittens/tui/handler.py` | ~300 | `Handler` base class (`screen_size`, `term_manager`, `schedule_write`, `tui_loop`) |

**Go Clipboard Kitten Files (contextual):**

| File Path | Purpose |
|-----------|---------|
| `kittens/clipboard/main.go` | Go-based clipboard kitten entry point |
| `kittens/clipboard/read.go` | Clipboard read operations |
| `kittens/clipboard/write.go` | Clipboard write operations |
| `kittens/clipboard/legacy.go` | Legacy OSC 52 support |

**Existing Documentation Files Referenced:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `docs/clipboard.rst` | OSC 5522 protocol specification — wire format context |
| `docs/kittens_intro.rst` | Kitten framework user introduction |
| `docs/performance.rst` | Performance characteristics documentation |

**Folders Explored:**

| Folder Path | Purpose |
|-------------|---------|
| `` (repository root) | Initial structure discovery |
| `kitty/` | Core C and Python source files |
| `kittens/` | Kitten extensions directory |
| `kittens/tui/` | TUI framework for kitten applications |
| `kittens/clipboard/` | Go clipboard kitten |
| `docs/` | Existing Sphinx documentation tree |
| `blitzy/` | Target output directory (does not yet exist) |

### 0.11.2 Tech Spec Sections Retrieved

| Section Heading | Purpose |
|----------------|---------|
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | VT parser dispatch architecture, OSC routing, GPU rendering pipeline |
| 4.6 REMOTE CONTROL SYSTEM FLOW | Command execution sequence, transport selection (contextual) |
| 4.8 KITTENS FRAMEWORK EXECUTION FLOW | Kitten resolution, execution lifecycle, result serialization, error isolation |
| 5.2 COMPONENT DETAILS | Child Monitor architecture, VT Parser, Boss Controller, GPU rendering, kittens framework |

### 0.11.3 User Attachments

No attachments were provided by the user. No Figma URLs were referenced.

