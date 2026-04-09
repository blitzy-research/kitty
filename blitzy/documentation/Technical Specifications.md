# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative technical document** that comprehensively answers a series of interrelated questions about how kitty's C code handles communication with the shell process it spawns, based on direct source-code analysis and live runtime observation.

- **Documentation Category**: Create new documentation
- **Documentation Type**: Technical investigation / Q&A reference document
- **Target Audience**: Developers seeking to understand kitty's PTY-to-shell communication internals

The user's requirements decompose into the following six precise questions, each requiring both code-level analysis and runtime evidence:

- **Q1 – Shell Process Identity**: When kitty starts a shell, what process gets spawned, what is its PID, what is the exact command line in the process list, and what PTY device path connects them?
- **Q2 – Read Syscall for Simple Output**: When `echo test123` is typed in kitty's terminal, what system calls does kitty make to read the result from the PTY, what buffer size is used, and how many bytes come back?
- **Q3 – Read Behavior Under High-Volume Output**: When `yes hello` generates continuous output, how does kitty's reading behavior change — what is the frequency of reads and the typical byte count per read?
- **Q4 – PTY Master File Descriptor**: What file descriptor number does kitty use to read from the PTY master side?
- **Q5 – C Function for PTY Reading**: In the C code, what function reads from the PTY file descriptor?
- **Q6 – C Function for Parsing**: What function parses the incoming data to separate printable text from escape sequences?

### 0.1.2 Special Instructions and Constraints

The user has specified the following critical constraints:

- **No Source File Modifications**: "Please refrain from altering any source files." The investigation must be entirely observational — reading code and tracing runtime behavior without modifying the kitty source tree.
- **Temporary Artifacts Allowed**: "Temporary logs or small helper scripts are acceptable, but delete them afterward." Any strace logs, helper scripts, or intermediary notes used during investigation must be cleaned up.
- **Output Placement Rule** (from implementation rules): The generated markdown document must be named `kitty_815df1e210e0.md` and placed in the `blitzy/documentation` directory.
- **No Assumptions**: "Do not make assumptions, base your answers on the code as the truth."
- **Thinking / Rationale**: "Provide thinking / rationale behind the answers."

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To answer Q1, we will **analyze `kitty/child.c` (the `spawn()` function) and `kitty/child.py` (the `Child.fork()` method)** to trace how `fork()` + `execvp()` create the shell, how `os.openpty()` creates the PTY pair, and what the slave device path is. We will **correlate with live strace of the spawn** to capture the actual PID, command-line, and `/dev/pts/N` path.
- To answer Q2, we will **strace the `KittyChildMon` I/O thread** while `echo test123` runs, capturing the exact `read()` syscall parameters and return values on the PTY master fd.
- To answer Q3, we will **strace the same thread during `yes hello`** output, collecting statistics on read frequency, buffer sizes, and byte counts under sustained high-volume output.
- To answer Q4, we will **inspect `/proc/<kitty_pid>/fd/`** to identify which fd number maps to `/dev/pts/ptmx`, and correlate with the strace data.
- To answer Q5, we will **document the `read_bytes()` function in `kitty/child-monitor.c`** which calls `read(fd, buf, available_buffer_space)` on the PTY master fd.
- To answer Q6, we will **document `consume_input()` and `consume_normal()` in `kitty/vt-parser.c`**, which use `utf8_decode_to_esc()` to separate printable UTF-8 text from ESC-initiated escape sequences.

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The relationship between `kitty/child.py`'s `openpty()` wrapper and the C `spawn()` function in `kitty/child.c` requires a clear narrative connecting the Python orchestration layer to the native fork/exec path.
- The three-thread architecture of the Child Monitor (`main thread`, `KittyChildMon` I/O thread, `talk thread`) is essential context for understanding which thread actually performs PTY reads.
- The 1 MiB ring buffer (`BUF_SZ = 1024*1024`) inside the VT parser, and how `vt_parser_create_write_buffer()` calculates available space, is critical for answering the buffer-size question accurately.
- The `input_delay` configuration parameter's effect on poll timeouts during high-volume output (transitioning from `-1` infinite wait to `0`–`2 ms` timeouts) explains the behavioral change between idle and busy states.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

The kitty repository contains a mature Sphinx/reStructuredText documentation system rooted in the `docs/` directory. Repository analysis reveals a comprehensive user-facing documentation tree with build tooling, but no existing document that specifically answers the PTY-to-shell communication questions posed by the user.

- **Documentation Framework**: Sphinx (version unspecified in `docs/requirements.txt`, but pinned as a dependency)
- **Documentation Generator Configuration**: `docs/conf.py` (central Sphinx configuration with custom lexers, roles, and man-page hooks)
- **Build Driver**: `docs/Makefile` (provides `make html`, `make man`, `make develop-docs` via `sphinx-autobuild`)
- **Dependencies**: `docs/requirements.txt` listing `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
- **API Documentation Tools**: None detected for C code (no Doxygen or Breathe config); Python typing stubs exist in `kitty/fast_data_types.pyi` and `kitty/typing.pyi`
- **Diagram Tools**: Mermaid is used in the technical specification; no diagram generation tooling is installed in the project itself
- **Documentation Hosting**: The project references an online docs site (linked from `README.asciidoc`)

The existing documentation covers user-facing topics (configuration, keyboard protocol, graphics protocol, shell integration, kittens) but does not include internal C-level architecture documentation on PTY communication mechanics.

### 0.2.2 Repository Code Analysis for Documentation

The following source files were examined to build the documentation content:

**Primary C Sources for PTY Communication**:
- `kitty/child.c` — Contains the `spawn()` function that forks the child process with PTY slave redirection, calls `fork()`, `setsid()`, `ioctl(TIOCSCTTY)`, and `execvp()`. 225 lines.
- `kitty/child-monitor.c` — Contains the `io_loop()` function (the `KittyChildMon` thread), the `read_bytes()` function that performs `read(fd, buf, available_buffer_space)`, and the `write_to_child()` function. 2016 lines.
- `kitty/vt-parser.c` — Contains `consume_input()`, `consume_normal()`, `consume_esc()`, `consume_csi()`, the `run_worker()` parse dispatch, and the `vt_parser_create_write_buffer()` / `vt_parser_commit_write()` buffer management API. 1596 lines.
- `kitty/simd-string.c` — Contains `utf8_decode_to_esc()` and its scalar/SIMD implementations that classify bytes as printable UTF-8 text versus ESC sentinels. Also contains `utf8_decode_to_esc_128()` (SSE) and `utf8_decode_to_esc_256()` (AVX2) fast paths.

**Primary Python Sources for PTY Setup**:
- `kitty/child.py` — Contains the `Child` class with `fork()` method that calls `os.openpty()`, sets up stdin/ready pipes, calls `fast_data_types.spawn()`, stores `self.child_fd = master`, and sets the fd non-blocking.

**Supporting Headers**:
- `kitty/vt-parser.h` — Declares `Parser`, `ParseData`, `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `parse_worker()`, and `parse_worker_dump()`.
- `kitty/simd-string.h` — Defines `UTF8Decoder` struct with `output.storage` (uint32_t array), `state` (UTF-8 decoder state), and `num_consumed` counter.
- `kitty/control-codes.h` — Defines control byte constants: `ESC` (0x1b), `BEL`, `BS`, `HT`, `LF`, `CR`, `SO`, `SI`, etc.
- `kitty/screen.h` — Defines the `Screen` struct with `write_buf`, `write_buf_sz`, `write_buf_used`, `write_buf_lock`, and `vt_parser` fields.

### 0.2.3 Runtime Experimentation Conducted

Kitty v0.35.2 was built from source on Ubuntu 24.04 with Python 3.12.3 and Go 1.22.2. An Xvfb virtual framebuffer (display :99) was used to provide a headless X11 environment. The following experiments were conducted with `strace`:

- **Spawn observation**: `strace -f -e trace=clone,fork,execve,openat` captured the fork/exec path showing `clone(SIGCHLD)` followed by `openat("/dev/pts/0", O_RDWR|O_CLOEXEC)` and `execve("/bin/bash", ["/bin/bash", "--posix"], ...)`.
- **Echo test**: `strace -p <KittyChildMon_TID> -e trace=read,poll` during `echo test123` captured `read(8, "test123\r\n", 1048576) = 9`.
- **High-volume test**: The same strace during `yes hello` captured a rapid burst of reads on fd 8, each returning between ~300 and ~2350 bytes, with poll timeouts shrinking from infinite to 0–2 ms.
- **FD inspection**: `ls -la /proc/<kitty_pid>/fd/` confirmed `fd 8 -> /dev/pts/ptmx` as the PTY master, and `readlink /proc/<child_pid>/fd/0` confirmed `/dev/pts/0` as the PTY slave.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage to answer the user's questions comprehensively:

- **Module: `kitty/child.py`**
  - Public APIs: `Child.__init__()`, `Child.fork()`, `openpty()`
  - Current documentation: No internal architecture doc exists
  - Documentation needed: PTY creation flow, fork orchestration, fd assignment

- **Module: `kitty/child.c`**
  - Public APIs: `spawn()` (C function exposed via Python C API)
  - Current documentation: Source comments only, no standalone doc
  - Documentation needed: Fork/exec sequence, PTY slave redirection, signal setup, controlling terminal establishment

- **Module: `kitty/child-monitor.c`**
  - Public APIs: `read_bytes()`, `io_loop()`, `write_to_child()`, `do_parse()`, `add_child()`
  - Current documentation: No standalone doc
  - Documentation needed: I/O thread architecture, poll loop mechanics, read buffer management, high-volume behavior

- **Module: `kitty/vt-parser.c`**
  - Public APIs: `consume_input()`, `consume_normal()`, `consume_esc()`, `consume_csi()`, `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `run_worker()`
  - Current documentation: No standalone doc
  - Documentation needed: State machine dispatch, buffer ring architecture, text vs. escape separation

- **Module: `kitty/simd-string.c` / `kitty/simd-string.h`**
  - Public APIs: `utf8_decode_to_esc()`, `utf8_decode_to_esc_scalar()`, `UTF8Decoder` struct
  - Current documentation: No standalone doc
  - Documentation needed: How the ESC sentinel scan works (byte 0x1b detection), SIMD acceleration paths

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the documentation gaps include:

- **No existing document** answers the specific PTY communication questions posed by the user
- **No internal architecture document** exists that traces the data flow from child process spawn through PTY read to VT parsing
- **No runtime behavior documentation** captures actual syscall traces, buffer sizes, or byte counts during normal and high-volume terminal output
- **No fd-level documentation** maps kitty's file descriptor layout to its functional purposes
- **The existing Sphinx docs** cover user-facing features but not the C-level I/O internals

The new document `kitty_815df1e210e0.md` will fill all of these gaps by providing a single, self-contained reference that answers each question with code citations and runtime evidence.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will be placed at:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
```

The document itself follows a Q&A structure aligned to the six questions, with each answer combining code analysis and runtime evidence:

```
blitzy/documentation/kitty_815df1e210e0.md
├── Introduction (scope, methodology, build environment)
├── Q1: Shell Process Spawning
│   ├── Python orchestration (child.py: openpty, fork)
│   ├── C fork/exec path (child.c: spawn)
│   ├── Runtime evidence (PID, cmdline, PTY device)
│   └── Mermaid diagram: spawn sequence
├── Q2: Reading `echo test123` from the PTY
│   ├── read_bytes() in child-monitor.c
│   ├── vt_parser_create_write_buffer() buffer size
│   ├── Strace evidence: read(8, "test123\r\n", 1048576) = 9
│   └── Data flow narrative
├── Q3: High-Volume Reading with `yes hello`
│   ├── Poll loop behavior change
│   ├── Strace evidence: read sizes, poll timeouts
│   ├── Buffer accumulation pattern
│   └── Comparison table: idle vs. busy
├── Q4: PTY Master File Descriptor Number
│   ├── fd layout from /proc/<pid>/fd/
│   ├── Code path: child.py stores master as child_fd
│   └── Mapping table: all kitty fds
├── Q5: C Function That Reads from the PTY
│   ├── read_bytes() analysis (child-monitor.c:1337)
│   ├── io_loop() polling and dispatch
│   └── Code snippet with citation
├── Q6: C Function That Parses PTY Data
│   ├── consume_input() dispatch (vt-parser.c:1367)
│   ├── consume_normal() + utf8_decode_to_esc() (vt-parser.c:230, simd-string.c:38)
│   ├── VTE state machine routing
│   └── Mermaid diagram: parse dispatch
└── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach**:
- Extract function signatures and logic from `kitty/child.c`, `kitty/child-monitor.c`, `kitty/vt-parser.c`, and `kitty/simd-string.c` via `read_file`
- Correlate code with runtime strace output captured during experiments on Xvfb :99
- Generate Mermaid diagrams by mapping the spawn sequence and parse dispatch flow from actual code paths

**Documentation Standards**:
- Markdown format with `##` / `###` headings
- Code snippets cite exact file and line number: `Source: kitty/child-monitor.c:1337`
- Mermaid diagrams in fenced blocks for spawn sequence and VT parser dispatch
- Tables for structured data (fd mapping, read statistics, idle-vs-busy comparison)
- All answers include a "Rationale" paragraph explaining the reasoning behind each conclusion

### 0.4.3 Diagram and Visual Strategy

Two Mermaid diagrams will be included:

- **Spawn Sequence Diagram**: Shows the flow from `Child.fork()` in Python through `os.openpty()`, pipe creation, `fast_data_types.spawn()`, and into the C `spawn()` function with `fork()` → child-side `setsid()` + `ioctl(TIOCSCTTY)` + `dup2()` + `execvp()` → parent-side `os.close(slave)` + `os.set_blocking(master, False)`.
- **VT Parser Dispatch Flowchart**: Shows `read_bytes()` → `vt_parser_commit_write()` → `run_worker()` → `consume_input()` → state-based dispatch to `consume_normal()` / `consume_esc()` / `consume_csi()` / `consume_osc()` etc., with `consume_normal()` calling `utf8_decode_to_esc()` to split text from escape sequences.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/child.c`, `kitty/child.py`, `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/vt-parser.h`, `kitty/simd-string.c`, `kitty/simd-string.h`, `kitty/control-codes.h`, `kitty/screen.h` | Complete Q&A document answering all six PTY communication questions with code citations and runtime evidence from strace experiments |

No other documentation files are in scope. The user's implementation rule specifies: "Do not modify any existing files in the source repository." The only artifact produced is a single new markdown document.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation / Q&A Reference
Source Code:
    - kitty/child.c (spawn function, fork/exec path)
    - kitty/child.py (Child class, openpty, fork orchestration)
    - kitty/child-monitor.c (io_loop, read_bytes, write_to_child)
    - kitty/vt-parser.c (consume_input, consume_normal, consume_esc, buffer management)
    - kitty/vt-parser.h (Parser struct, ParseData, API declarations)
    - kitty/simd-string.c (utf8_decode_to_esc, scalar/SIMD implementations)
    - kitty/simd-string.h (UTF8Decoder struct definition)
    - kitty/control-codes.h (ESC, BEL, BS, LF, CR constants)
    - kitty/screen.h (Screen struct with write_buf and vt_parser fields)
Sections:
    - Introduction (build environment, methodology)
    - Q1: Shell process identity (PID, cmdline, PTY device)
    - Q2: PTY read syscalls for echo test123 (buffer size, byte count)
    - Q3: High-volume reading behavior with yes hello (frequency, byte counts)
    - Q4: PTY master file descriptor number
    - Q5: C function that reads from the PTY (read_bytes)
    - Q6: C function that parses incoming data (consume_input, consume_normal, utf8_decode_to_esc)
    - Summary
Diagrams:
    - Mermaid sequence diagram: child process spawn flow
    - Mermaid flowchart: VT parser dispatch and text/escape separation
Key Citations:
    - kitty/child-monitor.c:1337 (read_bytes function)
    - kitty/child-monitor.c:1480 (io_loop function)
    - kitty/child.c:80 (spawn function)
    - kitty/child.py:170 (openpty function)
    - kitty/child.py:276 (Child.fork method)
    - kitty/vt-parser.c:18 (BUF_SZ = 1024*1024)
    - kitty/vt-parser.c:230 (consume_normal function)
    - kitty/vt-parser.c:1367 (consume_input function)
    - kitty/vt-parser.c:1417 (run_worker function)
    - kitty/vt-parser.c:1450 (vt_parser_create_write_buffer)
    - kitty/simd-string.c:38 (utf8_decode_to_esc_scalar)
    - kitty/simd-string.c:72 (utf8_decode_to_esc dispatch)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be updated. The new document is a standalone markdown file placed in `blitzy/documentation/`, which is not part of the kitty Sphinx documentation tree. No `mkdocs.yml`, `docusaurus.config.js`, or `docs/conf.py` changes are required.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes** are needed — the document is fully self-contained.
- **No navigation links** to other documents are required.
- **No table of contents updates** are needed in any existing file.
- The document references source files by relative path within the repository root (e.g., `kitty/child-monitor.c:1337`).

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to this documentation exercise:

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| apt | python3 | 3.12.3 | Python runtime required to build kitty |
| apt | golang-go | 1.22.2 | Go compiler for kitty's Go tools |
| apt | gcc | 13.2.0 | C compiler for kitty's native extensions |
| apt | pkg-config | (system) | Library discovery for build system |
| apt | libfreetype-dev | (system) | FreeType font rendering dependency |
| apt | libfontconfig-dev | (system) | Font configuration dependency |
| apt | libharfbuzz-dev | (system) | Text shaping dependency |
| apt | liblcms2-dev | (system) | Color management dependency |
| apt | libxxhash-dev | (system) | Fast hashing dependency |
| apt | libx11-dev | (system) | X11 windowing dependency |
| apt | libgl-dev | (system) | OpenGL rendering dependency |
| apt | libssl-dev | (system) | OpenSSL for crypto dependency |
| apt | libsimde-dev | 0.7.2 | SIMD portability headers for simd-string.c |
| apt | libx11-xcb-dev | (system) | X11-XCB bridge dependency |
| apt | libdbus-1-dev | (system) | D-Bus integration dependency |
| apt | xvfb | (system) | Virtual framebuffer for headless testing |
| apt | strace | (system) | Syscall tracing for runtime evidence |
| pip | sphinx | (unversioned) | Docs framework (existing project dependency) |
| pip | furo | (unversioned) | Sphinx theme (existing project dependency) |

Note: The `docs/requirements.txt` does not pin specific versions for Sphinx or its extensions. The build dependencies listed above were required to compile kitty v0.35.2 from source in order to conduct the live runtime experiments.

### 0.6.2 Documentation Reference Updates

No documentation reference updates are required. The new document `blitzy/documentation/kitty_815df1e210e0.md` is a standalone file that does not link to or from any existing documentation. No link transformation rules apply.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **User questions covered**: 6/6 (100%)
  - Q1 Shell process identity: Answered via `child.c:spawn()`, `child.py:fork()`, and live process inspection
  - Q2 Read syscall for simple output: Answered via `child-monitor.c:read_bytes()` and strace of `echo test123`
  - Q3 High-volume reading behavior: Answered via strace of `yes hello` and poll-loop analysis in `child-monitor.c:io_loop()`
  - Q4 PTY master fd number: Answered via `/proc/<pid>/fd/` inspection (fd 8)
  - Q5 C function that reads: Answered by identifying `read_bytes()` at `child-monitor.c:1337`
  - Q6 C function that parses: Answered by identifying `consume_input()` at `vt-parser.c:1367` and `consume_normal()` + `utf8_decode_to_esc()` at `vt-parser.c:230` / `simd-string.c:38`
- **Source files analyzed**: 9 primary C/Python/header files documented
- **Runtime experiments conducted**: 4 (spawn trace, echo test, yes hello test, fd inspection)

### 0.7.2 Documentation Quality Criteria

- **Completeness Requirements**:
  - Every question receives a direct, unambiguous answer at the start of its section
  - Every answer is supported by both source-code citation (file path and line number) and runtime evidence (strace output or /proc inspection)
  - Thinking and rationale are provided for each answer as required by the user's implementation rules

- **Accuracy Validation**:
  - Code citations reference exact line numbers verified via `read_file`
  - Runtime data was captured from an actual kitty v0.35.2 build running under Xvfb :99
  - Buffer sizes and byte counts are taken directly from strace output, not inferred
  - The `BUF_SZ = 1024u*1024u = 1,048,576` value is confirmed from `vt-parser.c:18`

- **Clarity Standards**:
  - Technical accuracy paired with narrative explanations
  - Mermaid diagrams for complex flows (spawn sequence, parse dispatch)
  - Tables for structured comparisons (idle vs. busy read behavior, fd mapping)
  - Progressive disclosure: answer first, then evidence, then rationale

- **Maintainability**:
  - All citations use `Source: <file>:<line>` format for easy cross-referencing
  - Document is self-contained with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Diagrams**: 2 Mermaid diagrams (spawn sequence, VT parser dispatch)
- **Code snippets**: Short excerpts (2–5 lines) from `read_bytes()`, `consume_normal()`, `utf8_decode_to_esc_scalar()`, and `spawn()`
- **Strace excerpts**: Verbatim lines from strace output for `echo test123` and `yes hello`
- **Tables**: fd mapping table, idle-vs-busy comparison table, file reference table

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file**:
  - `blitzy/documentation/kitty_815df1e210e0.md` — the sole output artifact
- **Source files analyzed for documentation content** (read-only):
  - `kitty/child.c` — fork/exec spawn path
  - `kitty/child.py` — Python PTY orchestration and Child class
  - `kitty/child-monitor.c` — I/O thread, read_bytes(), io_loop(), write_to_child()
  - `kitty/vt-parser.c` — VT parser state machine, consume_input(), consume_normal(), buffer management
  - `kitty/vt-parser.h` — Parser and ParseData struct declarations
  - `kitty/simd-string.c` — utf8_decode_to_esc() and SIMD variants
  - `kitty/simd-string.h` — UTF8Decoder struct definition
  - `kitty/control-codes.h` — ESC and control byte constants
  - `kitty/screen.h` — Screen struct with write_buf and vt_parser fields
  - `kitty/screen.c` — screen_draw_text() and draw_text() entry point
  - `kitty/constants.py` — shell_path determination logic
- **Runtime observations** (captured and reported, not persisted):
  - Strace of spawn, echo test123, yes hello
  - /proc/<pid>/fd/ inspection
  - Process tree and thread inspection via ps
- **Directory creation**:
  - `blitzy/documentation/` — created if not already present

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No `.c`, `.py`, `.h`, or any other source file may be altered (per user instruction: "Please refrain from altering any source files")
- **Test file modifications**: No test files are created or modified
- **Feature additions or code refactoring**: Not applicable
- **Existing documentation updates**: No changes to `docs/**/*.rst`, `README.asciidoc`, `CONTRIBUTING.md`, `INSTALL.md`, `SECURITY.md`, or any other existing file
- **Build system changes**: No changes to `Makefile`, `setup.py`, `pyproject.toml`, `go.mod`
- **Documentation deployment**: No docs build or hosting changes
- **GPU rendering pipeline**: Not relevant to PTY communication questions
- **Font subsystem**: Not relevant to PTY communication questions
- **Remote control system**: Not relevant to PTY communication questions
- **Kittens framework**: Not relevant to PTY communication questions
- **Shell integration scripts**: Not directly relevant (shell integration modifies shell environment, not the PTY read path)
- **Temporary artifacts**: Any strace logs or helper scripts used during investigation are cleaned up after use, as instructed

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a standalone Markdown file, not part of the Sphinx docs tree
- **Documentation preview command**: Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram generation command**: Mermaid diagrams are embedded as fenced code blocks within the Markdown and are rendered by any Mermaid-compatible viewer
- **Documentation deployment command**: Not applicable
- **Default format**: Markdown with Mermaid diagrams
- **Citation requirement**: Every technical claim must reference a source file and line number
- **Style guide**: Q&A format with "Rationale" sub-sections explaining reasoning, per the user's rule: "Provide thinking / rationale behind the answers"
- **Documentation validation**: Manual review — verify that all six questions are answered with both code evidence and runtime evidence

### 0.9.2 Build and Runtime Environment for Experiments

The following environment was used to build kitty and conduct the runtime experiments:

- **OS**: Ubuntu 24.04 (container)
- **Python**: 3.12.3
- **Go**: 1.22.2
- **GCC**: 13.2.0
- **Kitty version built**: 0.35.2
- **Display server**: Xvfb :99 (1024x768x24)
- **Tracing tool**: strace (attached to KittyChildMon thread TID)
- **Build command**: `python3 setup.py build --ignore-compiler-warnings`

These details are documented in the output file's Introduction section so that the experiments can be reproduced.

## 0.10 Rules for Documentation

The following rules are explicitly mandated by the user and must be followed precisely:

- **No source file modifications**: "Please refrain from altering any source files." — Zero changes to any `.c`, `.py`, `.h`, `.rst`, `.md`, or any other file in the kitty source tree.
- **Temporary artifacts must be cleaned up**: "Temporary logs or small helper scripts are acceptable, but delete them afterward." — Any strace output files, helper scripts, or intermediary notes created during investigation must be removed before completion.
- **Output file naming and placement**: The generated document must be named `kitty_815df1e210e0.md` (matching the source branch name) and placed in the `blitzy/documentation` directory.
- **No assumptions**: "Do not make assumptions, base your answers on the code as the truth." — Every answer must be grounded in specific source code references or runtime observations, never inferred from general knowledge about terminals or PTYs.
- **Rationale required**: "Provide thinking / rationale behind the answers." — Each answer section must include an explanation of the reasoning process, not just the conclusion.
- **No modification of existing repository files**: "Do not modify any existing files in the source repository." — This reinforces the no-modification constraint from the implementation rules.
- **Code as ground truth**: All conclusions must trace back to specific lines in the source code, corroborated by runtime strace evidence where applicable.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files were retrieved and analyzed during context gathering:

| File Path | Lines | Purpose in Investigation |
|---|---|---|
| `kitty/child.c` | 225 | C `spawn()` function: fork, PTY slave redirection, setsid, TIOCSCTTY, execvp |
| `kitty/child.py` | 500 | Python `Child` class: `openpty()`, `fork()`, PTY master fd storage, non-blocking setup |
| `kitty/child-monitor.c` | 2016 | `ChildMonitor` type: `io_loop()`, `read_bytes()`, `write_to_child()`, `do_parse()`, poll multiplexing |
| `kitty/vt-parser.c` | 1596 | VT parser state machine: `consume_input()`, `consume_normal()`, `consume_esc()`, `consume_csi()`, buffer management (`BUF_SZ`, `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`) |
| `kitty/vt-parser.h` | 38 | Parser and ParseData struct declarations, thread-safe API |
| `kitty/simd-string.c` | ~240 | `utf8_decode_to_esc()` dispatch and `utf8_decode_to_esc_scalar()` implementation |
| `kitty/simd-string.h` | 59 | `UTF8Decoder` struct, `utf8_decoder_ensure_capacity()`, SIMD variant declarations |
| `kitty/control-codes.h` | ~90 | ESC (0x1b), BEL, BS, HT, LF, CR, SO, SI, and escape sequence introducer constants |
| `kitty/screen.h` | ~170 | `Screen` struct with `write_buf`, `write_buf_sz`, `write_buf_used`, `vt_parser` field |
| `kitty/screen.c` | 4932 | `screen_draw_text()` entry point, write_buf initialization (`BUFSIZ`) |
| `kitty/constants.py` | ~190 | `shell_path` determination via `pwd.getpwuid().pw_shell` |
| `kitty/data-types.h` | (scanned) | Core type definitions and macros |
| `docs/requirements.txt` | 6 | Sphinx documentation dependencies |
| `pyproject.toml` | ~50 | Python version requirement (`>=3.8`), tooling config |
| `Makefile` | ~65 | Build targets: `all`, `test`, `clean`, `debug` |
| `setup.py` | (header scanned) | Build system: C extension compilation, version extraction |

**Folders explored**:

| Folder Path | Purpose |
|---|---|
| `` (root) | Repository root: identified all top-level files and 13 sub-folders |
| `kitty/` | Core application tree: identified all 130+ C, Python, GLSL, and header files |
| `docs/` | Documentation tree: assessed existing Sphinx/RST infrastructure |

### 0.11.2 Tech Spec Sections Referenced

| Section | Relevance |
|---|---|
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | VT parser dispatch architecture, keyboard/mouse input paths, GPU rendering pipeline |
| 5.2 COMPONENT DETAILS | Child Monitor three-thread architecture, VT Parser and Screen Model, GLFW platform layer |

### 0.11.3 Attachments

No attachments were provided by the user. No Figma URLs or external design references are applicable to this task.

### 0.11.4 Runtime Experiments Conducted

| Experiment | Command | Key Finding |
|---|---|---|
| Build kitty | `python3 setup.py build --ignore-compiler-warnings` | Successfully built kitty v0.35.2 |
| Spawn trace | `strace -f -e trace=clone,fork,execve,openat` | `clone(SIGCHLD)` → child opens `/dev/pts/0` → `execve("/bin/bash", ...)` |
| Interactive shell inspection | `ps --ppid`, `/proc/<pid>/fd/`, `ps -T` | Child: `/bin/bash --posix` on `pts/0`; Kitty fd 8 → `/dev/pts/ptmx`; thread `KittyChildMon` |
| Echo test strace | `strace -p <KittyChildMon_TID> -e trace=read,poll` during `echo test123` | `read(8, "test123\r\n", 1048576) = 9` |
| High-volume strace | Same strace during `yes hello` | Rapid burst reads: 300–2350 bytes/read, poll timeouts drop to 0–2 ms, buffer space shrinks progressively |

