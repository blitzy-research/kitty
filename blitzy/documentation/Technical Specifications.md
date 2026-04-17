# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the requirement is to **investigate and document kitty's C-level PTY communication pipeline** by building kitty from source, running it, and empirically observing the process spawning, system calls, buffer management, and parsing functions involved when kitty communicates with its child shell process. Specifically:

- **Build kitty from source** and launch it with a functional display server to observe its runtime behavior
- **Identify the spawned shell process**: determine the exact process that gets spawned (executable path, command-line arguments), its PID, and the PTY device path that connects the shell to kitty
- **Trace system calls for `echo test123`**: determine what syscall kitty's C code uses to read data from the PTY master, what buffer size is passed, and how many bytes are returned for a simple echo command
- **Trace high-volume output with `yes hello`**: observe how kitty's reading behavior changes under continuous high-throughput output, measuring read frequency and typical byte counts per read
- **Identify the PTY master file descriptor number** kitty uses on its side of the PTY pair
- **Identify the C functions**: determine which C function in kitty reads from the PTY file descriptor, and which C function parses the incoming byte stream to separate printable text from terminal escape sequences
- **No source modification**: all existing source files must remain unmodified; only temporary helper scripts or log files are acceptable and must be deleted afterward

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint**: The user explicitly states "please refrain from altering any source files." Only temporary logs and small helper scripts are acceptable, and they must be deleted afterward.
- **Implementation rule (SWE-AtlasQnA-Repo)**: Create a new markdown document named `kitty_815df1e210e0.md` in `blitzy/documentation/` that comprehensively answers all questions.
- **Evidence-based answers**: Answers must be grounded in the actual source code and empirical observation (strace, process inspection), not assumptions.
- **No new code in the source repository** besides the requested documentation markdown file.

### 0.1.3 Technical Interpretation

These requirements translate to a code analysis and runtime observation task that produces a documentation artifact. The work involves:

- To **build kitty**, we will execute `python3 setup.py build --debug` after installing all required C development libraries (libfreetype, libharfbuzz, libfontconfig, libgl, libx11, etc.) and Go 1.22 toolchain.
- To **launch kitty**, we will start an Xvfb virtual framebuffer on display `:99` and run the compiled binary `kitty/launcher/kitty` against it.
- To **observe process spawning**, we will use `strace -f` to trace `clone()`, `openat(/dev/ptmx)`, `ioctl(TIOCSCTTY)`, and `execve()` calls, plus `ps` and `/proc` inspection.
- To **measure read behavior**, we will use `strace -f -T -e trace=read` to capture all `read()` syscalls on the PTY master file descriptor with timing and byte counts.
- To **identify C functions**, we will perform static analysis of `kitty/child-monitor.c` (the I/O thread that calls `read()`) and `kitty/vt-parser.c` (the VT parser state machine that separates text from escape sequences).
- To **produce the deliverable**, we will create `blitzy/documentation/kitty_815df1e210e0.md` with all findings.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

Since this is a pure investigation/documentation task (no source modifications allowed), the relevant files are those analyzed to derive answers. The following files were inspected through code reading and runtime observation:

**Core PTY Communication Files (Primary Analysis Targets)**

| File | Role | Lines |
|------|------|-------|
| `kitty/child-monitor.c` | Multi-threaded I/O event loop; contains `read_bytes()` which calls POSIX `read()` on the PTY master fd; contains the `io_loop()` function running on the KittyChildMon thread | 2016 |
| `kitty/child.c` | PTY child process spawning; contains `spawn()` which calls `fork()`, sets up the slave PTY, and `execvp()` the shell | 225 |
| `kitty/child.py` | Python-level child process orchestration; calls `os.openpty()` to create the PTY pair, invokes `fast_data_types.spawn()` | 500 |
| `kitty/vt-parser.c` | VT terminal parser state machine; contains `consume_input()` which dispatches to `consume_normal()` for text and `consume_esc()`/`consume_csi()` for escape sequences | 1596 |
| `kitty/vt-parser.h` | Parser API header; declares `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `parse_worker()` | ~40 |
| `kitty/simd-string.c` | SIMD-accelerated UTF-8 decoder; contains `utf8_decode_to_esc()` which scans for the ESC byte (0x1b) to split text from escape sequences | ~250 |

**Supporting Infrastructure Files (Context Analysis)**

| File | Role |
|------|------|
| `kitty/data-types.h` | Defines `MAX_CHILDREN` (512), shared type definitions |
| `kitty/control-codes.h` | Defines ESC (0x1b), ESC_CSI ('['), ESC_OSC (']'), ESC_DCS ('P') constants |
| `kitty/screen.c` | Screen model; `screen_draw_text()` processes decoded printable text from the parser |
| `kitty/screen.h` | Screen interface declarations |
| `kitty/constants.py` | Defines `shell_path` using `pwd.getpwuid(os.geteuid()).pw_shell` |
| `kitty/launcher/main.c` | Native C launcher entry point |
| `setup.py` | Build system orchestrating C extension compilation and Go binary assembly |
| `pyproject.toml` | Declares `requires-python = ">=3.8"` |
| `go.mod` | Declares `go 1.22` module requirement |

### 0.2.2 Web Search Research Conducted

No external web research was required. All answers were derived from:
- Direct source code reading of the C and Python files
- Empirical runtime observation using `strace -f` with various trace filters
- Process tree inspection via `ps`, `pstree`, and `/proc` filesystem
- File descriptor analysis via `/proc/<pid>/fd/`

### 0.2.3 New File Requirements

**Single New Documentation File:**

| File | Purpose |
|------|---------|
| `blitzy/documentation/kitty_815df1e210e0.md` | Comprehensive answer document covering all PTY communication questions, per the SWE-AtlasQnA-Repo rule |

No other new files are created. All temporary strace logs and helper scripts used during investigation have been deleted.

## 0.3 Dependency Inventory

### 0.3.1 Build Dependencies Required

The following packages were installed to build kitty from source. These are the exact versions used from the Ubuntu 24.04 package repositories in the container:

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| apt | build-essential | 12.10ubuntu1 | C/C++ compiler toolchain (gcc) |
| apt | python3-dev | 3.12.3-0ubuntu2 | Python C API headers for extension compilation |
| apt | pkg-config | 1.8.1-2build1 | Build dependency resolution |
| apt | libfreetype-dev | 2.13.2+dfsg-1 | FreeType font rendering library |
| apt | libharfbuzz-dev | 8.3.0-2build2 | HarfBuzz text shaping engine |
| apt | libfontconfig-dev | 2.15.0-1.1ubuntu2 | Font configuration and discovery |
| apt | libgl-dev | 1.7.0-1build1 | OpenGL development headers |
| apt | libx11-dev | 2:1.8.7-1build1 | X11 client library |
| apt | libx11-xcb-dev | 2:1.8.7-1build1 | X11-XCB interop library |
| apt | libxkbcommon-x11-dev | 1.6.0-1build1 | XKB keyboard handling |
| apt | libdbus-1-dev | 1.14.10-4ubuntu4.1 | D-Bus IPC library |
| apt | liblcms2-dev | 2.14-2build1 | ICC color management |
| apt | libpng-dev | 1.6.43-5ubuntu0.5 | PNG image handling |
| apt | libxxhash-dev | 0.8.2-2build1 | xxHash fast hashing |
| apt | librsync-dev | 2.3.4-1.1ubuntu2 | rsync delta algorithm |
| apt | libssl-dev | 3.0.13-0ubuntu3 | OpenSSL cryptography |
| apt | golang-go | 2:1.22.2-2 | Go compiler for tools/ binary |

### 0.3.2 Runtime / Observation Dependencies

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| apt | strace | 6.8-0ubuntu2 | System call tracing for PTY read observation |
| apt | xvfb | 2:21.1.12-1ubuntu1.5 | Virtual X11 framebuffer (headless display) |
| apt | xdotool | 1:3.20160805.1-5build1 | Simulated keyboard input for interactive testing |

### 0.3.3 No Dependency Changes Required

This is a documentation-only task. No dependency manifests (`pyproject.toml`, `go.mod`, `setup.py`, etc.) are modified. All dependencies listed above were used solely for building and observing kitty at runtime.

## 0.4 Integration Analysis

### 0.4.1 PTY Communication Architecture Touchpoints

The investigation revealed a tightly integrated three-layer architecture connecting the shell process to kitty's display. The following integration points were identified through code analysis and runtime tracing:

**Layer 1: Process Spawning (Python → C → OS)**

- `kitty/child.py` → `Child.fork()` calls `os.openpty()` to create the PTY pair (master + slave), then calls `fast_data_types.spawn()` (a C extension function)
- `kitty/child.c` → `spawn()` performs `fork()` via the POSIX `clone()` syscall, sets up the child with `setsid()`, opens the PTY slave via `open("/dev/pts/N", O_RDWR|O_CLOEXEC)`, sets controlling terminal via `ioctl(fd, TIOCSCTTY, 0)`, redirects stdin/stdout/stderr to the slave, then calls `execvp()` with the shell
- `kitty/boss.py` → `Boss.add_child()` registers the child's PTY master fd with the ChildMonitor via `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)`

**Layer 2: I/O Multiplexing (C I/O Thread)**

- `kitty/child-monitor.c` → `io_loop()` runs on a dedicated thread named "KittyChildMon"
- The I/O loop uses `poll()` to multiplex: 2 extra fds (wakeup pipe + signal pipe) plus one fd per child (the PTY master)
- When `POLLIN` fires on a child fd, `read_bytes()` calls `read(fd, buf, available_buffer_space)` into the VT parser's internal buffer
- When `POLLOUT` fires, `write_to_child()` sends queued keystrokes from the screen's write buffer to the child via `write(fd, ...)`

**Layer 3: Parsing and Dispatch (C VT Parser)**

- `kitty/vt-parser.c` → `run_worker()` is called from the main thread's `parse_input()` → `do_parse()` cycle
- `consume_input()` dispatches to `consume_normal()` (plain text), `consume_esc()` (ESC sequences), `consume_csi()` (CSI sequences), and `dispatch_osc()`/`dispatch_dcs()` for structured escape codes
- `kitty/simd-string.c` → `utf8_decode_to_esc()` is the fast-path scanner that processes text bytes until it hits 0x1b (ESC), at which point the parser transitions to escape sequence handling
- `kitty/screen.c` → `screen_draw_text()` receives the decoded Unicode codepoints for display

### 0.4.2 Data Flow Diagram

```mermaid
flowchart TD
    Shell["/bin/bash --posix<br/>(child process)"] -->|writes to stdout| Slave["/dev/pts/0<br/>(PTY slave)"]
    Slave -->|kernel PTY layer| Master["/dev/pts/ptmx<br/>(PTY master, fd 8)"]
    Master -->|poll() + read()| IOThread["io_loop()<br/>KittyChildMon thread<br/>kitty/child-monitor.c"]
    IOThread -->|"vt_parser_create_write_buffer()<br/>+ vt_parser_commit_write()"| VTBuf["VT Parser Buffer<br/>1 MiB ring buffer<br/>kitty/vt-parser.c"]
    VTBuf -->|"run_worker() → consume_input()"| Parser["VT Parser State Machine<br/>kitty/vt-parser.c"]
    Parser -->|"consume_normal() → utf8_decode_to_esc()"| TextPath["screen_draw_text()<br/>kitty/screen.c"]
    Parser -->|"consume_esc() / consume_csi()"| EscPath["Escape Sequence Handlers<br/>screen.c, modes.h, etc."]
    
    Keyboard["Keyboard Input<br/>kitty/keys.c"] -->|"write_to_child()"| Master
```

### 0.4.3 Thread Architecture

| Thread | Name | Function | Role in PTY Communication |
|--------|------|----------|---------------------------|
| Main Thread | (unnamed) | `parse_input()` → `do_parse()` → `run_worker()` | Parses buffered input from VT parser, drives rendering |
| I/O Thread | KittyChildMon | `io_loop()` | Polls PTY fds, reads child output, writes keystrokes |
| Talk Thread | KittyPeerMon | `talk_loop()` | Handles remote control peer sockets (not PTY-related) |

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this task produces only a single documentation file with no source modifications, the execution plan is:

- **CREATE**: `blitzy/documentation/kitty_815df1e210e0.md` — Comprehensive answer document covering all PTY communication questions derived from source code analysis and runtime experimentation

No existing files are modified. All investigation was performed through read-only code analysis and non-invasive runtime tracing.

### 0.5.2 Implementation Approach

The documentation file will be produced by synthesizing findings from two complementary approaches:

**Static Code Analysis** — Reading the C source to identify exact function names, buffer sizes, data structures, and control flow:
- `read_bytes()` in `kitty/child-monitor.c` (line ~1337) is the function that issues the POSIX `read()` syscall on the PTY master fd
- `consume_input()` in `kitty/vt-parser.c` (line ~1367) is the entry point for parsing incoming data, dispatching to `consume_normal()` for printable text and `consume_esc()`/`consume_csi()` for escape sequences
- `BUF_SZ` is defined as `(1024u*1024u)` = 1,048,576 bytes in `kitty/vt-parser.c` (line 18)
- The available buffer space passed to `read()` is computed as `BUF_SZ - write.offset` by `vt_parser_create_write_buffer()`

**Runtime Experimentation** — Building kitty with `--debug`, launching it under `strace -f`, and observing actual system calls:
- Shell spawned: `/bin/bash --posix` via `clone()` + `execvp()`
- PTY master opened via `openat(AT_FDCWD, "/dev/ptmx", O_RDWR)` returning fd 8
- PTY slave assigned `/dev/pts/0` via `ioctl(TIOCGPTN)` + `ioctl(TIOCSPTLCK)`
- For `echo test123`: shell echo produces 1-byte reads for each character (PTY echo), then `"test123\r\n"` = 9 bytes for the command output
- For `yes hello`: reads range from 14 to 4095 bytes per call, with 95+ reads per burst; buffer size parameter decreases as the parser buffer fills
- The I/O thread (KittyChildMon) reads from fd 8, which maps to `/dev/pts/ptmx` in `/proc/<pid>/fd/`

### 0.5.3 Empirical Findings Summary

**Question 1: What process gets spawned?**
- Process: `/bin/bash --posix` (the user's login shell with `--posix` flag added by shell integration)
- The shell path is determined by `pwd.getpwuid(os.geteuid()).pw_shell` in `kitty/constants.py`

**Question 2: What PTY device path connects them?**
- PTY master: `/dev/pts/ptmx` (opened via `/dev/ptmx`), held as fd 8 in kitty
- PTY slave: `/dev/pts/0`, connected to the child shell's stdin/stdout/stderr

**Question 3: What system calls does kitty make to read from the PTY?**
- The POSIX `read()` syscall, issued from the `read_bytes()` function in `kitty/child-monitor.c`
- Buffer size: up to 1,048,576 bytes (BUF_SZ), reduced by already-buffered unprocessed data
- For `echo test123`: each echoed keystroke returns 1 byte; the final output `"test123\r\n"` returns 9 bytes

**Question 4: How does reading behavior change with `yes hello`?**
- Under high-volume output, reads happen in rapid succession without blocking (poll returns immediately with POLLIN)
- Typical byte count per read: 14–4095 bytes (median ~14 bytes for small bursts, up to ~4095 bytes when the kernel PTY buffer is full)
- The buffer size parameter passed to `read()` decreases across consecutive reads within a single fill cycle as the parser buffer accumulates data
- The parser buffer is drained when `run_worker()` calls `consume_input()`, at which point the full 1 MiB becomes available again

**Question 5: What fd number does kitty use?**
- File descriptor 8 for the PTY master side, confirmed both by strace and `/proc/<pid>/fd/8 -> /dev/pts/ptmx`

**Question 6: What C function reads from the PTY?**
- `read_bytes()` in `kitty/child-monitor.c` — calls `vt_parser_create_write_buffer()` to get a pointer into the parser's 1 MiB buffer, then calls `read(fd, buf, available_buffer_space)`, followed by `vt_parser_commit_write()`

**Question 7: What C function parses incoming data to separate text from escape sequences?**
- `consume_input()` in `kitty/vt-parser.c` is the top-level dispatcher
- For printable text: `consume_normal()` calls `utf8_decode_to_esc()` (in `kitty/simd-string.c`) which scans bytes until it finds 0x1b (ESC), decoding UTF-8 codepoints along the way, then passes them to `screen_draw_text()`
- For escape sequences: when ESC (0x1b) is found, the parser transitions to `VTE_ESC` state and dispatches to `consume_esc()`, `consume_csi()`, `dispatch_osc()`, `dispatch_dcs()`, etc.

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

- **Documentation output**: `blitzy/documentation/kitty_815df1e210e0.md`
- **Source files analyzed (read-only)**:
  - `kitty/child-monitor.c` — I/O thread, `read_bytes()`, `io_loop()`, `write_to_child()`
  - `kitty/child.c` — `spawn()` function, PTY slave setup, fork/exec
  - `kitty/child.py` — `Child.fork()`, `openpty()`, shell argument construction
  - `kitty/vt-parser.c` — `consume_input()`, `consume_normal()`, `consume_esc()`, `consume_csi()`, buffer management
  - `kitty/vt-parser.h` — Parser API declarations
  - `kitty/simd-string.c` — `utf8_decode_to_esc()`, SIMD-accelerated text/escape separator
  - `kitty/screen.c` — `screen_draw_text()` for processed text output
  - `kitty/control-codes.h` — ESC, CSI, OSC, DCS constant definitions
  - `kitty/data-types.h` — `MAX_CHILDREN` constant
  - `kitty/constants.py` — `shell_path` determination
  - `kitty/boss.py` — `add_child()` child monitor registration
  - `setup.py` — Build system configuration
  - `pyproject.toml` — Python version requirements
  - `go.mod` — Go module version

### 0.6.2 Explicitly Out of Scope

- **No source file modifications**: All kitty source files remain unmodified per user directive
- **No GPU rendering analysis**: The rendering pipeline (shaders, glyph cache, OpenGL) is not investigated
- **No remote control protocol analysis**: The talk thread and peer socket handling are not covered
- **No kittens framework analysis**: Built-in kittens and their execution lifecycle are not analyzed
- **No configuration system changes**: No kitty.conf modifications
- **No cross-platform analysis**: Only Linux/X11 behavior is observed (container environment)
- **No performance optimization**: Read frequency and buffer statistics are observational, not prescriptive
- **No Wayland or macOS analysis**: The empirical observations are specific to the X11 backend on Linux

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

- **SWE-AtlasQnA-Repo Rule**: Create a new markdown document named `kitty_815df1e210e0.md` that comprehensively answers the question(s) posed in the prompt. Build and run the source code to analyze the repository behavior as needed. Do not make assumptions — base answers on the code as the truth. Provide thinking and rationale behind the answers. Do not modify any existing files in the source repository. Do not add any other code in the source repository besides the requested document. Place the generated document in the `blitzy/documentation` directory.

- **User's Read-Only Directive**: "Please refrain from altering any source files. Temporary logs or small helper scripts are acceptable, but delete them afterward."

### 0.7.2 Implementation Conventions Followed

- All answers are grounded in specific source file references with line numbers and function names
- Empirical observations are captured via `strace -f` with multiple trace configurations to cross-validate findings
- Temporary files (strace logs) were created during investigation and deleted upon completion
- The output document uses structured markdown with clear section headings for each question
- All PTY master fd numbers, PIDs, and byte counts are from actual runtime observations, not from documentation or assumptions

## 0.8 References

### 0.8.1 Source Files Analyzed

| File Path | Purpose of Analysis |
|-----------|-------------------|
| `kitty/child-monitor.c` | I/O thread event loop, `read_bytes()` function, `io_loop()`, `write_to_child()`, `Child` struct, `EXTRA_FDS`, poll-based multiplexing |
| `kitty/child.c` | `spawn()` C function — fork, PTY slave setup, TIOCSCTTY, execvp |
| `kitty/child.py` | `Child.fork()` — openpty, shell argument construction, fd passing to ChildMonitor |
| `kitty/vt-parser.c` | `BUF_SZ` constant (1 MiB), `PS` struct (parser state with buffer), `consume_input()`, `consume_normal()`, `consume_esc()`, `consume_csi()`, `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `run_worker()` |
| `kitty/vt-parser.h` | Parser API: `vt_parser_create_write_buffer()`, `vt_parser_commit_write()`, `vt_parser_has_space_for_input()`, `parse_worker()` |
| `kitty/simd-string.c` | `utf8_decode_to_esc()` — SIMD-accelerated scanning for ESC (0x1b) byte to split text from escape sequences, `utf8_decode_to_esc_scalar()` reference implementation |
| `kitty/screen.c` | `screen_draw_text()` — receives decoded Unicode codepoints for rendering |
| `kitty/screen.h` | Screen interface declarations |
| `kitty/control-codes.h` | `ESC` (0x1b), `ESC_CSI` ('['), `ESC_OSC` (']'), `ESC_DCS` ('P') |
| `kitty/data-types.h` | `MAX_CHILDREN` (512) constant |
| `kitty/constants.py` | `shell_path` via `pwd.getpwuid()` |
| `kitty/boss.py` | `Boss.add_child()` registration of child fd with monitor |
| `setup.py` | Build system for C extensions and Go tools |
| `pyproject.toml` | `requires-python = ">=3.8"` |
| `go.mod` | `go 1.22` module requirement |
| `Makefile` | Developer build targets |

### 0.8.2 Folders Explored

| Folder Path | Purpose of Exploration |
|-------------|----------------------|
| (root) | Repository structure discovery, build system files |
| `kitty/` | Core application source tree — C extensions, Python orchestration, shaders |
| `kitty/launcher/` | Native binary launcher (main.c), compiled `kitty` executable |

### 0.8.3 Runtime Experiments Conducted

| Experiment | Strace Filter | Key Observations |
|-----------|--------------|-----------------|
| Shell spawn tracing | `clone,execve,openat,ioctl,read,write` | PTY opened at fd 8 via `/dev/ptmx`; child PID forks, opens `/dev/pts/0`, sets TIOCSCTTY, execve `/bin/bash` |
| `echo test123` read behavior | `read` with timestamps | Each echoed character = 1-byte read; final output `"test123\r\n"` = 9 bytes; buffer parameter = 1,048,576 |
| `yes hello` high-volume reads | `read` with timestamps | 95+ reads per burst; byte counts 14–4095; buffer parameter decreases as data accumulates; reads complete in ~15–50 μs each |
| Interactive typing | `read,write,openat,clone,ioctl` | Confirmed fd 8 = PTY master; I/O thread writes keystrokes to fd 8, reads shell echo back from fd 8 |
| Process tree inspection | `ps`, `pstree`, `/proc/<pid>/fd/` | Shell process = `/bin/bash --posix`; fd 8 links to `/dev/pts/ptmx` |

### 0.8.4 Attachments

No external attachments were provided with this task. No Figma URLs were specified.

