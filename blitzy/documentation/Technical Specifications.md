# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively traces and explains the complete lifecycle of a child process in the Kitty terminal emulator — from the moment a simple program runs and prints output through to its successful termination with exit status zero, including the exact mechanisms by which Kitty detects, reports, and displays information about that termination.

### 0.1.1 Core Documentation Objective

- **Category:** Create new documentation
- **Documentation type:** Technical Q&A / Architecture explanation document
- **Target audience:** A developer investigating Kitty's internal child-process lifecycle and shell integration behavior
- **Output file:** `blitzy/documentation/kitty_815df1e210e0.md` (per project implementation rules)

The user's questions span multiple layers of the Kitty architecture and require documentation that traces data and control flow across C native code, Python orchestration, Go tooling, and shell integration scripts. The specific documentation requirements, restated with technical precision, are:

- **Requirement 1 — Complete child-process exit flow:** Document the end-to-end sequence from a child program running inside a Kitty terminal window through to its successful exit with status zero, covering what happens to the Kitty window and the Kitty process itself.
- **Requirement 2 — Kitty's own exit code:** Explain the exact exit code that the Kitty process reports to its parent when a child exits cleanly (status 0) and all windows close.
- **Requirement 3 — User-visible completion message:** Identify the full message displayed to the user about a program completing, and which function assembles that message.
- **Requirement 4 — Child-process tracking subsystem:** Identify which runtime component is responsible for tracking the child process (the Child Monitor in `kitty/child-monitor.c`).
- **Requirement 5 — Message-generating function:** Identify the single function that turns the child's exit status into the user-facing notification message (`handle_cmd_end` in `kitty/window.py`).
- **Requirement 6 — OS-level signal for child termination:** Document that Kitty listens for `SIGCHLD` in its I/O thread's signal handler (`handle_signal` in `kitty/child-monitor.c`).
- **Requirement 7 — System call for exit status retrieval:** Document that Kitty calls `waitpid(-1, &status, WNOHANG)` inside the `reap_children` function in `kitty/child-monitor.c`.
- **Requirement 8 — Exit-status transport from shell to Kitty:** Explain the OSC 133 escape sequence protocol used by shell integration (specifically `ESC]133;D;$?\a` embedded in the shell prompt), parsed by `shell_prompt_marking()` in `kitty/screen.c`.
- **Requirement 9 — Location of child program's printed output:** Explain that stdout flows through the PTY → VT parser → screen model → GPU rendering pipeline and appears in the Kitty terminal window.

### 0.1.2 Special Instructions and Constraints

- **Read-only investigation:** The user explicitly states: "Do not add or modify any source files in the repository while you investigate." The only file to be created is the documentation output in `blitzy/documentation/`.
- **Cleanup requirement:** "Clean up any temporary files or scripts you create when you're done." No temporary artifacts should remain.
- **Implementation rules:** Per the project's SWE-AtlasQnA-Repo rules, the output must be a new markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory, providing thinking/rationale behind answers, basing all answers on code as truth, and not modifying any existing files.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the child-process exit flow, we will create a narrative tracing the path from `SIGCHLD` reception in `kitty/child-monitor.c` through `reap_children()`, `mark_child_for_removal()`, `parse_input()`, `death_notify` callback, to `Boss.on_child_death()` in `kitty/boss.py`, and ultimately window destruction.
- To document Kitty's exit code, we will trace the `main()` function in `kitty/main.py` which exits cleanly (code 0) when no exception occurs, and the event loop termination path in `process_global_state()` within `child-monitor.c`.
- To document the user-visible message, we will explain two distinct pathways: (a) shell integration's OSC 133;D notification processed by `handle_cmd_end()` in `kitty/window.py` which generates the message `"Command {cmdline} finished with status: {exit_status}.\nClick to focus."`, and (b) the window closure path when the child process itself terminates.
- To document the shell-to-Kitty escape sequence mechanism, we will trace from `shell-integration/bash/kitty.bash` line 239 (`\e]133;D;$?\a`) through the VT parser in `kitty/vt-parser.c`, to `shell_prompt_marking()` in `kitty/screen.c`, to `Window.cmd_output_marking()` and ultimately `handle_cmd_end()` in `kitty/window.py`.
- To document where the child's stdout appears, we will trace from the PTY file descriptor through `read_bytes()` in `kitty/child-monitor.c`, into the VT parser, to the screen model (`kitty/screen.c`, `kitty/line.c`, `kitty/line-buf.c`), and finally to the GPU rendering pipeline.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Two distinct exit-notification pathways:** The codebase reveals two separate mechanisms that must be clearly distinguished: (a) the PTY-EOF/SIGCHLD path that removes the window when the child process itself dies, and (b) the OSC 133 shell integration path that reports individual command exit statuses while the shell remains alive. The documentation must clearly differentiate these.
- **Configuration dependency:** The `close_on_child_death` option (default: `no`) and `notify_on_cmd_finish` option (default: `never`) materially affect what the user observes. These must be documented.
- **Hold mode behavior:** The `hold` flag in `kitty/child.py` triggers `cmdline_for_hold()`, wrapping the command with the `kitten run-shell --env=KITTY_HOLD=1` mechanism. When hold is active, `tools/tui/hold.go` displays `"Press Enter or Esc to exit"` after the child program finishes. This alternate path must be covered.
- **Thread architecture context:** The three-thread architecture of the Child Monitor (main thread, I/O thread, talk thread) is essential context for understanding how signals flow to the reaping logic.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation tree in `docs/` with comprehensive coverage of user-facing features, protocol specifications, and configuration, but limited internal-architecture documentation relevant to the child-process lifecycle questions.

- **Documentation framework:** Sphinx (version pinned in `docs/requirements.txt`)
- **Theme:** Furo (`docs/requirements.txt`)
- **Documentation generator configuration:** `docs/conf.py`
- **Documentation build driver:** `docs/Makefile` (supports `help`, generic Sphinx targets, `develop-docs` live preview via `sphinx-autobuild`)
- **API documentation tools in use:** No JSDoc/Sphinx autodoc for Python internals — documentation is hand-authored reStructuredText
- **Diagram tools detected:** Mermaid is used extensively in the technical specification; the existing docs use images and screenshots rather than inline diagrams
- **Documentation hosting:** The existing docs are published to the Kitty website (referenced in `README.asciidoc` and `publish.py`)

Key existing documentation assets examined:

| Documentation File | Relevance to User's Questions |
|---|---|
| `docs/shell-integration.rst` | Documents OSC 133 prompt marking and shell integration features — directly relevant to exit-status reporting |
| `docs/conf.rst` | Documents `close_on_child_death` and `notify_on_cmd_finish` options — relevant to observable behavior |
| `docs/faq.rst` | Troubleshooting and common questions — contextual but not directly covering child exit flow |
| `docs/overview.rst` | High-level architecture description — provides context |
| `docs/performance.rst` | Explains threaded rendering architecture — relevant to understanding I/O thread |
| `CONTRIBUTING.md` | Contributor workflow guidelines |
| `README.asciidoc` | Project introduction with links |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code modules relevant to documenting child-process lifecycle:

- **Child process management:** `kitty/child.py` (Python `Child` class — PTY allocation, fork/spawn, environment preparation), `kitty/child.c` (native child process helpers), `kitty/child-monitor.c` (native I/O thread loop, signal handling, `waitpid`, death notification)
- **Window lifecycle:** `kitty/window.py` (`Window` class — `handle_cmd_end()`, `cmd_output_marking()`, exit status tracking), `kitty/boss.py` (`Boss.on_child_death()` — window destruction and cleanup)
- **Shell integration:** `shell-integration/bash/kitty.bash` (OSC 133;D exit status emission in PS1), `kitty/screen.c` (`shell_prompt_marking()` — OSC 133 parsing), `kitty/shell_integration.py` (environment modification for shell integration injection)
- **VT parsing pipeline:** `kitty/vt-parser.c` (byte stream classification and routing), `kitty/screen.c` (screen model operations)
- **Hold mechanism:** `kitty/utils.py` (`cmdline_for_hold()` — wraps command for hold behavior), `tools/tui/hold.go` (`HoldTillEnter()`, `ExecAndHoldTillEnter()` — Go-side hold behavior with "Press Enter or Esc to exit" message), `tools/cmd/run_shell/main.go` (run-shell kitten entry point)
- **Main application lifecycle:** `kitty/main.py` (`main()`, `_run_app()` — startup and exit code determination), `kitty/entry_points.py` (entry point routing)
- **Configuration:** `kitty/options/definition.py` (`close_on_child_death` default `no`, `notify_on_cmd_finish` default `never`), `kitty/options/types.py` (typed `Options` class)

Key directories examined:

| Directory | Contents Relevant to Investigation |
|---|---|
| `kitty/` | Core application sources — child.py, child-monitor.c, window.py, boss.py, screen.c, main.py, utils.py, shell_integration.py, vt-parser.c |
| `shell-integration/bash/` | Bash shell integration script emitting OSC 133 sequences |
| `tools/tui/` | Go TUI library — hold.go (hold mechanism), run.go (RunShell, RunCommandRestoringTerminalToSaneStateAfter) |
| `tools/cmd/run_shell/` | Go run-shell kitten entry point |
| `kitty/options/` | Configuration definition and parsing — close_on_child_death, notify_on_cmd_finish |
| `docs/` | Existing Sphinx documentation tree |

### 0.2.3 Web Search Research Conducted

No web search was required for this analysis. All answers are derived directly from the source code as the authoritative truth, consistent with the project's implementation rules ("Do not make assumptions, base your answers on the code as the truth").

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage to comprehensively answer the user's questions about child-process exit behavior:

**Module: `kitty/child-monitor.c` (Child Monitor — native C)**
- Public APIs / Functions:
  - `handle_signal()` — Signal handler that sets `child_died = true` on `SIGCHLD` (line 1362)
  - `reap_children()` — Calls `waitpid(-1, &status, WNOHANG)` to collect exit statuses (line 1413)
  - `mark_child_for_removal()` — Marks a child entry for removal by PID (line 1386)
  - `read_bytes()` — Reads from child PTY fd; returns `false` on EOF signaling child death (line 1337)
  - `io_loop()` — I/O thread main loop; polls child fds, reads signals, calls `reap_children` (line 1480)
  - `parse_input()` — Main thread function; processes remove queue and calls `death_notify` callback (line 451)
  - `report_reaped_pids()` — Reports reaped background-monitored PIDs to Boss (line 950)
- Current documentation: Internal C code, no external documentation
- Documentation needed: Architectural explanation of signal flow, waitpid usage, death notification callback chain

**Module: `kitty/boss.py` (Boss Controller — Python)**
- Public APIs / Functions:
  - `on_child_death(window_id)` — Callback invoked by `death_notify`; destroys window and cleans up tab/OS window hierarchy (line 881)
  - `on_monitored_pid_death(pid, exit_status)` — Handles background-monitored process deaths (line 2725)
- Current documentation: Exists in tech spec but not in a standalone Q&A format
- Documentation needed: Explanation of what happens after child death notification reaches Python

**Module: `kitty/window.py` (Window — Python)**
- Public APIs / Functions:
  - `handle_cmd_end(exit_status)` — Processes OSC 133;D exit status; generates notification message (line 1408)
  - `cmd_output_marking(is_start, cmdline)` — Dispatch function for OSC 133 A/C/D markers (line 1453)
  - `last_cmd_exit_status` — Instance attribute storing last command's exit status (line 244)
- Current documentation: Not documented externally
- Documentation needed: The notification message format and conditions under which it fires

**Module: `kitty/screen.c` (Screen Model — native C)**
- Public APIs / Functions:
  - `shell_prompt_marking(buf)` — Parses OSC 133 A/C/D markers and dispatches to Python callbacks (line 2328)
- Current documentation: Referenced in tech spec but requires detailed explanation
- Documentation needed: How OSC 133;D is parsed and translated to the `cmd_output_marking` callback

**Module: `shell-integration/bash/kitty.bash` (Bash Shell Integration)**
- Key mechanism: PS1 injection of `\e]133;D;$?\a\e]133;A\a` (line 239), emitting the last command's exit status via OSC 133;D escape sequence as part of the prompt
- Current documentation: `docs/shell-integration.rst` covers features at a high level
- Documentation needed: Precise explanation of how `$?` gets embedded in the OSC 133;D sequence

**Module: `kitty/main.py` (Application Entry — Python)**
- Public APIs / Functions:
  - `main()` — Top-level entry; catches exceptions and exits with code 1 on error, otherwise exits cleanly with code 0 (line 524)
  - `_run_app()` — Creates Boss, starts event loop via `boss.child_monitor.main_loop()` (line 202)
- Documentation needed: Explanation of Kitty's own exit code determination

**Module: `tools/tui/hold.go` (Hold Mechanism — Go)**
- Public APIs / Functions:
  - `HoldTillEnter()` — Displays "Press Enter or Esc to exit" and waits for keypress (line 16)
  - `ExecAndHoldTillEnter()` — Runs a command, then holds; exits with the command's exit code (line 44)
- Documentation needed: Explanation of the hold path as an alternate behavior

**Module: `kitty/options/definition.py` (Configuration — Python)**
- Key options: `close_on_child_death` (default `no`, line 2920), `notify_on_cmd_finish` (default `never`, line 3190)
- Documentation needed: How these options affect observable behavior

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps must be addressed by the new Q&A document:

- **Undocumented internal flow:** The complete SIGCHLD → waitpid → mark_child_for_removal → parse_input → death_notify → on_child_death chain is not documented in any existing user-facing or developer-facing document.
- **Two-path distinction:** No existing document clearly distinguishes between the "child process dies → window closes" path and the "shell reports command exit via OSC 133" path.
- **Precise message format:** The exact notification message (`"Command {s} finished with status: {exit_status}.\nClick to focus."`) and its generating function (`handle_cmd_end`) are not documented externally.
- **Kitty's own exit code:** No documentation explicitly states that Kitty exits with code 0 when all windows close cleanly.
- **Hold behavior:** The "Press Enter or Esc to exit" message from `tools/tui/hold.go` and its relationship to the `hold` flag in `kitty/child.py` is undocumented as a distinct flow.
- **Output display path:** The PTY → VT parser → screen model → GPU renderer pipeline for stdout display is documented in the tech spec but not in a concise Q&A format.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The documentation output is a single comprehensive markdown file placed per the project's implementation rules:

```text
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
```

The internal structure of the document should follow a logical Q&A narrative organized by the user's questions:

```text
blitzy/documentation/kitty_815df1e210e0.md
├── Introduction (context and scenario setup)
├── Section 1: Complete Child-Process Exit Flow
│   ├── The Two Pathways (shell integration vs. direct child death)
│   ├── SIGCHLD → waitpid → window closure chain
│   └── Mermaid diagram of the full flow
├── Section 2: Kitty's Own Exit Code
│   ├── What exit code Kitty reports (0 on clean shutdown)
│   └── Code path through main.py
├── Section 3: User-Visible Completion Message
│   ├── The handle_cmd_end notification message
│   ├── The hold-mode "Press Enter or Esc to exit" message
│   └── Configuration dependencies (notify_on_cmd_finish)
├── Section 4: Child-Process Tracking Subsystem
│   ├── Child Monitor (child-monitor.c) architecture
│   └── Three-thread model (I/O thread, main thread, talk thread)
├── Section 5: The Message-Generating Function
│   ├── handle_cmd_end() in kitty/window.py
│   └── Exact message format with code citation
├── Section 6: OS-Level Signal for Child Termination
│   ├── SIGCHLD handling in handle_signal()
│   └── Signal delivery to I/O thread
├── Section 7: System Call for Exit Status Retrieval
│   ├── waitpid(-1, &status, WNOHANG) in reap_children()
│   └── Status storage in ReapedPID struct
├── Section 8: Exit Status Transport (Shell → Kitty)
│   ├── OSC 133;D;$? escape sequence in bash PS1
│   ├── VT parser → shell_prompt_marking() → cmd_output_marking()
│   └── Mermaid sequence diagram
├── Section 9: Where Child Output Appears
│   ├── PTY → read_bytes → VT parser → screen model → GPU pipeline
│   └── Display in the Kitty terminal window
└── Summary and Rationale
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract signal handling flow from `kitty/child-monitor.c` lines 1359–1426 (handle_signal, reap_children)
- Extract death notification chain from `kitty/child-monitor.c` lines 451–526 (parse_input, death_notify callback)
- Extract window cleanup from `kitty/boss.py` lines 881–918 (on_child_death)
- Extract OSC 133 handling from `kitty/screen.c` lines 2327–2356 (shell_prompt_marking)
- Extract notification message from `kitty/window.py` lines 1408–1451 (handle_cmd_end)
- Extract shell integration mechanism from `shell-integration/bash/kitty.bash` line 239 (PS1 OSC 133;D injection)
- Extract hold behavior from `tools/tui/hold.go` lines 16–72 (HoldTillEnter, ExecAndHoldTillEnter)
- Extract Kitty exit code from `kitty/main.py` lines 524–531 (main function)

**Documentation Standards:**
- Markdown formatting with proper headers (# ## ###)
- Mermaid diagram integration for the child exit flow and OSC 133 transport
- Code citations as inline references: `Source: /path/to/file.py:LineNumber`
- Tables for configuration option descriptions
- Consistent terminology: "child process", "Child Monitor", "death_notify callback", "OSC 133;D"

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the documentation file:

- **Child Process Exit Flow Diagram** — A flowchart showing the complete path from SIGCHLD reception through reap_children, mark_child_for_removal, parse_input, death_notify callback, to on_child_death and window destruction
- **OSC 133 Exit Status Transport Diagram** — A sequence diagram showing the flow from shell command completion through bash PS1 evaluation, OSC 133;D;$? emission, VT parser, shell_prompt_marking, cmd_output_marking, and finally handle_cmd_end
- **Output Display Pipeline Diagram** — A flowchart showing child stdout traversing PTY, read_bytes, VT parser, screen model, and GPU rendering to the display

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/child-monitor.c`, `kitty/boss.py`, `kitty/window.py`, `kitty/screen.c`, `kitty/child.py`, `kitty/main.py`, `shell-integration/bash/kitty.bash`, `tools/tui/hold.go`, `tools/cmd/run_shell/main.go`, `kitty/options/definition.py` | Comprehensive Q&A document answering all user questions about child-process exit lifecycle, exit codes, notification messages, SIGCHLD handling, waitpid usage, OSC 133 escape sequence transport, and stdout display path. Includes Mermaid diagrams, code citations, and rationale. |

This is the only documentation file to be created. No existing files are modified, updated, or deleted — per the user's explicit constraint and the project's implementation rules.

### 0.5.2 New Documentation File Detail

**File:** `blitzy/documentation/kitty_815df1e210e0.md`

- **Type:** Technical Q&A / Architecture explanation
- **Source Code:**
  - `kitty/child-monitor.c` (lines 1337–1426, 451–526, 950–962, 1480–1549) — Signal handling, waitpid, read_bytes, I/O loop, death notification
  - `kitty/boss.py` (lines 881–918, 2725–2729) — on_child_death, on_monitored_pid_death
  - `kitty/window.py` (lines 1408–1461) — handle_cmd_end, cmd_output_marking, notification message
  - `kitty/screen.c` (lines 2327–2356) — shell_prompt_marking, OSC 133 A/C/D handling
  - `kitty/child.py` (lines 276–354) — Child.fork(), PTY allocation, spawn
  - `kitty/main.py` (lines 202–236, 524–531) — _run_app, main, exit code
  - `shell-integration/bash/kitty.bash` (line 239) — OSC 133;D;$? in PS1
  - `tools/tui/hold.go` (lines 16–72) — HoldTillEnter, ExecAndHoldTillEnter
  - `tools/cmd/run_shell/main.go` (lines 26–69) — run-shell kitten main
  - `kitty/options/definition.py` (lines 2920–2931, 3190–3239) — close_on_child_death, notify_on_cmd_finish
- **Sections:**
  - Introduction — Scenario context and overview
  - Complete Child-Process Exit Flow — End-to-end narrative with two pathways distinguished
  - Kitty's Own Exit Code — What exit code the Kitty process reports (0)
  - User-Visible Completion Message — The notification message format and hold-mode message
  - Child-Process Tracking Subsystem — Child Monitor three-thread architecture
  - The Message-Generating Function — `handle_cmd_end()` identification and analysis
  - OS-Level Signal — SIGCHLD in handle_signal()
  - System Call — waitpid in reap_children()
  - Exit Status Transport — OSC 133;D protocol from shell to Kitty
  - Where Output Appears — PTY → VT parser → GPU renderer pipeline
  - Summary — Consolidated answers with rationale
- **Diagrams:**
  - Child exit flow (Mermaid flowchart)
  - OSC 133 transport (Mermaid sequence diagram)
  - Output display pipeline (Mermaid flowchart)
- **Key Citations:**
  - `kitty/child-monitor.c:1370` (SIGCHLD case in handle_signal)
  - `kitty/child-monitor.c:1418` (waitpid call)
  - `kitty/child-monitor.c:522` (death_notify callback invocation)
  - `kitty/boss.py:881` (on_child_death)
  - `kitty/window.py:1429` (notification message format)
  - `kitty/screen.c:2350` (OSC 133;D parsing)
  - `shell-integration/bash/kitty.bash:239` (PS1 OSC 133;D injection)
  - `kitty/main.py:524` (main function exit behavior)
  - `tools/tui/hold.go:26` (hold message)

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The output file is a standalone markdown document in the `blitzy/documentation/` directory and is not integrated into the Sphinx documentation build system.

### 0.5.4 Cross-Documentation Dependencies

- The new document references source code files but does not link to or depend on any existing documentation files.
- No navigation, table of contents, index, or glossary updates are needed.
- No shared content or includes are affected.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tooling or dependencies are required for this task. The output is a plain Markdown file that does not need a build system, generator, or rendering tool. All diagrams are embedded as Mermaid code blocks, which are renderable by GitHub, GitLab, and standard Markdown viewers with Mermaid support.

For reference, the existing project documentation dependencies (from `docs/requirements.txt`) are:

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| pip | sphinx | (unpinned) | Sphinx documentation generator for existing docs |
| pip | furo | (unpinned) | Material-like theme for Sphinx docs |
| pip | sphinx-copybutton | (unpinned) | Copy button for code blocks |
| pip | sphinxext-opengraph | (unpinned) | OpenGraph metadata for docs |
| pip | sphinx-inline-tabs | (unpinned) | Inline tab support in docs |
| pip | sphinx-autobuild | (unpinned) | Live preview for docs development |

These are **not required** for the current task — they are listed only for completeness.

The project's core runtime dependencies relevant to the documented child-process lifecycle are:

| Language | Component | Version | Relevance |
|---|---|---|---|
| Python | CPython | >= 3.8 (from `pyproject.toml`) | Hosts Boss controller, Window, Child classes |
| C | C11 standard | Enforced via `-std=c11` | Child Monitor, VT Parser, Screen Model |
| Go | Go modules | 1.22 (from `go.mod`) | Hold mechanism, run-shell kitten, TUI library |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates. The new file is self-contained and does not introduce links that need to be referenced from other documents.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis** for the user's specific questions:

| Question Area | Currently Documented | Status | Target |
|---|---|---|---|
| Complete child-process exit flow (SIGCHLD → window close) | Not documented externally | Gap | Full narrative with Mermaid diagram |
| Kitty's own exit code on clean child exit | Not documented | Gap | Precise answer with code citation |
| User-visible completion message and its generating function | Partially (shell-integration.rst mentions OSC 133) | Incomplete | Exact message format, function name, conditions |
| Child-process tracking subsystem identification | Referenced in tech spec only | Incomplete | Clear identification of Child Monitor with thread model |
| OS-level signal (SIGCHLD) | Not documented externally | Gap | Signal name, handler function, code location |
| System call (waitpid) | Not documented externally | Gap | System call, arguments, wrapper function |
| Exit status transport (OSC 133;D) | Partially (shell-integration.rst) | Incomplete | Complete chain from bash PS1 to handle_cmd_end |
| Where child output appears | Partially (performance.rst, tech spec) | Incomplete | Complete PTY → GPU pipeline trace |

**Target coverage:** 100% of the user's nine documented requirements must be answered with source code evidence and rationale.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question receives a direct, unambiguous answer
- Every answer includes a source code citation with file path and line number
- Both the shell-integration path (OSC 133;D for per-command exit status) and the direct child-death path (SIGCHLD/PTY EOF for window closure) are clearly distinguished
- Configuration options that affect behavior (`close_on_child_death`, `notify_on_cmd_finish`) are documented with default values
- The hold-mode alternate path is documented as a distinct case

**Accuracy validation:**
- All code citations must reference actual functions, line numbers, and variable names found in the repository
- The notification message format (`"Command {s} finished with status: {exit_status}.\nClick to focus."`) must be quoted exactly from `kitty/window.py:1429`
- The hold message (`"Press Enter or Esc to exit"`) must be quoted exactly from `tools/tui/hold.go:26`
- Signal name (SIGCHLD), system call (waitpid), and escape sequence (OSC 133;D) must be technically precise

**Clarity standards:**
- Technical accuracy with accessible language — assume the reader understands terminal emulators and Unix process management
- Progressive disclosure — start with the high-level flow, then drill into each mechanism
- Consistent terminology throughout: "child process", "Child Monitor", "I/O thread", "death_notify callback", "OSC 133;D"

**Maintainability:**
- Source citations with file paths and line numbers for traceability
- Self-contained document — no external dependencies

### 0.7.3 Example and Diagram Requirements

- Minimum 3 Mermaid diagrams (child exit flow, OSC 133 transport, output display pipeline)
- Short code snippets for key functions (2–3 lines each, showing function signatures and critical operations)
- Tables for configuration options and their defaults
- No screenshots required — all visuals are Mermaid diagrams

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable

**Source files analyzed for documentation content (read-only):**
- `kitty/child-monitor.c` — SIGCHLD handling, waitpid, I/O loop, death notification
- `kitty/boss.py` — on_child_death window cleanup
- `kitty/window.py` — handle_cmd_end notification message generation
- `kitty/screen.c` — OSC 133 shell_prompt_marking parsing
- `kitty/child.py` — Child class, fork/spawn, PTY allocation
- `kitty/main.py` — Application entry, exit code determination
- `kitty/vt-parser.c` — Byte stream classification (contextual)
- `kitty/utils.py` — cmdline_for_hold function
- `kitty/options/definition.py` — close_on_child_death, notify_on_cmd_finish definitions
- `kitty/options/types.py` — Options class with defaults
- `shell-integration/bash/kitty.bash` — OSC 133;D PS1 injection
- `tools/tui/hold.go` — HoldTillEnter and ExecAndHoldTillEnter
- `tools/tui/run.go` — RunShell and RunCommandRestoringTerminalToSaneStateAfter
- `tools/cmd/run_shell/main.go` — run-shell kitten entry point

**Topics covered:**
- SIGCHLD signal handling and delivery to I/O thread
- waitpid system call usage and exit status collection
- PTY EOF detection as child death indicator
- Child Monitor three-thread architecture
- Window destruction flow through Boss controller
- OSC 133;D escape sequence protocol for exit status reporting
- Shell integration bash PS1 injection mechanism
- handle_cmd_end notification message generation
- Kitty process exit code (0 on clean shutdown)
- Hold-mode behavior and message
- Child program stdout display pipeline (PTY → VT parser → screen → GPU)
- Configuration options affecting observable behavior

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing source files in the repository will be modified (per user's explicit instruction and project rules)
- **Test file modifications:** No test files will be created or modified
- **Feature additions or code refactoring:** No code changes of any kind
- **Existing documentation updates:** No updates to `docs/**/*.rst` or any other existing documentation files
- **Deployment configuration changes:** No changes to build, deploy, or CI configuration
- **Unrelated documentation:** Topics not related to the child-process exit lifecycle (e.g., graphics protocol, clipboard, font rendering, keyboard protocol) are not covered
- **Non-bash shell integration details:** While the OSC 133 mechanism is common across shells, detailed analysis is scoped to the bash integration script; Zsh and Fish integration scripts are not analyzed in detail
- **Remote control and SSH integration:** Not relevant to the user's questions
- **Temporary files:** Any temporary files created during investigation must be cleaned up (per user's explicit instruction)

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not part of the Sphinx build
- **Documentation preview command:** Any Markdown viewer or renderer (e.g., `grip`, VS Code Markdown preview, GitHub rendering)
- **Diagram generation command:** Mermaid diagrams are embedded inline and rendered by Mermaid-compatible viewers; no separate generation step needed
- **Documentation deployment command:** Not applicable — the file is committed directly to the `blitzy/documentation/` directory
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim must reference a specific source file and line number
- **Style guide:** Follow the project's SWE-AtlasQnA-Repo implementation rules: provide thinking/rationale behind answers, base all answers on code as truth, do not make assumptions
- **Documentation validation:** Manual review — verify that all nine user questions are answered with evidence from the codebase

### 0.9.2 Pre-Generation Verification Steps

Before generating the documentation file, verify:
- The `blitzy/documentation/` directory exists (create if not)
- No file named `kitty_815df1e210e0.md` already exists
- All source code citations have been verified against the actual codebase
- No temporary files or scripts remain from the investigation phase

### 0.9.3 Post-Generation Cleanup

Per the user's explicit instruction ("clean up any temporary files or scripts you create when you're done"):
- Remove any temporary scripts created during investigation
- Verify no investigation artifacts remain outside the `blitzy/documentation/` directory
- Confirm that no source files in the repository have been modified

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The only file to be created is `blitzy/documentation/kitty_815df1e210e0.md`. No source code files, test files, configuration files, or existing documentation files should be altered.
- **Do not make assumptions — base all answers on the code as the truth.** Every claim in the documentation must be traceable to a specific source file and line number in the repository. No speculative or inferred behavior should be presented as fact without evidence.
- **Provide thinking and rationale behind the answers.** The document should not merely state answers but explain why the answer is what it is, with citations to the code that demonstrate the reasoning.
- **Clean up any temporary files or scripts created during investigation.** No investigation artifacts should remain after the documentation is generated.
- **Place the generated document in the `blitzy/documentation` directory.** The output file must be named `kitty_815df1e210e0.md` (matching the source branch name) and placed in the `blitzy/documentation/` directory.
- **Comprehensively answer the question(s) posed in the prompt.** All nine user questions must be addressed with direct, unambiguous answers supported by code evidence.

## 0.11 References

### 0.11.1 Source Files and Folders Searched

The following files and folders were systematically examined to derive the conclusions documented in this Agent Action Plan:

**Core Child Process Lifecycle Files:**

| File Path | Lines Examined | Key Findings |
|---|---|---|
| `kitty/child-monitor.c` | 80–120, 451–538, 940–1010, 1220–1260, 1337–1440, 1480–1560 | SIGCHLD handling in `handle_signal()` (line 1370), `waitpid(-1, &status, WNOHANG)` in `reap_children()` (line 1418), `read_bytes()` EOF detection (line 1337), `death_notify` callback in `parse_input()` (line 522), I/O thread loop (line 1480), `mark_child_for_removal()` (line 1386), `report_reaped_pids()` (line 950), `ReapedPID` struct (line 91) |
| `kitty/boss.py` | 860–930, 2725–2729 | `on_child_death(window_id)` (line 881) — window destruction and tab cleanup; `on_monitored_pid_death(pid, exit_status)` (line 2725) — background process death handling |
| `kitty/window.py` | 244, 572, 1408–1461 | `handle_cmd_end(exit_status)` (line 1408) — notification message generation; `cmd_output_marking(is_start, cmdline)` (line 1453) — OSC 133 dispatch; `last_cmd_exit_status` attribute (line 244) |
| `kitty/screen.c` | 2327–2356 | `shell_prompt_marking(buf)` (line 2328) — OSC 133 A/C/D parsing; case 'D' (line 2350) extracts exit status and calls `cmd_output_marking` callback |
| `kitty/child.py` | 1–501 | `Child` class — PTY allocation via `openpty()` (line 170), `fork()` method with `fast_data_types.spawn()` (line 333), hold behavior via `cmdline_for_hold()` (line 330), environment preparation in `get_final_env()` (line 233) |
| `kitty/main.py` | 1–50, 200–260, 500–532 | `main()` (line 524) — top-level entry catching exceptions, exits with code 1 on error; `_run_app()` (line 202) — creates Boss, starts main loop; clean exit implies code 0 |
| `kitty/utils.py` | 1192–1203 | `cmdline_for_hold()` (line 1192) — wraps command with `kitten run-shell --env=KITTY_HOLD=1` |

**Shell Integration Files:**

| File Path | Lines Examined | Key Findings |
|---|---|---|
| `shell-integration/bash/kitty.bash` | 88–250, 288–350 | OSC 133;D PS1 injection at line 239: `_ksi_prompt[ps1]+="\[\e]133;D;\$?\a\e]133;A\a\]"` — embeds last command's exit status via `$?` in the OSC 133;D escape sequence |

**Go Tooling Files:**

| File Path | Lines Examined | Key Findings |
|---|---|---|
| `tools/tui/hold.go` | 1–72 | `HoldTillEnter()` (line 16) — displays `"\x1b[1;32mPress Enter or Esc to exit\x1b[m"` (line 26); `ExecAndHoldTillEnter()` (line 44) — runs command, then holds, exits with command's exit code |
| `tools/tui/run.go` | 148–210 | `RunShell()` (line 148) — shell execution with integration; `RunCommandRestoringTerminalToSaneStateAfter()` (line 191) — command execution with terminal state restoration |
| `tools/cmd/run_shell/main.go` | 1–110 | run-shell kitten entry point; calls `tui.RunCommandRestoringTerminalToSaneStateAfter(args)` for pre-shell commands, then `tui.RunShell()` |

**Configuration Files:**

| File Path | Lines Examined | Key Findings |
|---|---|---|
| `kitty/options/definition.py` | 2920–2931, 3190–3239 | `close_on_child_death` default `no` (line 2920); `notify_on_cmd_finish` default `never` (line 3190) |
| `kitty/options/types.py` | 70, 500 | `close_on_child_death: bool = False` (line 500) |

**Project Metadata Files:**

| File Path | Lines Examined | Key Findings |
|---|---|---|
| `pyproject.toml` | 1–43 | `requires-python = ">=3.8"` |
| `go.mod` | 1–10 | `go 1.22` |
| `docs/requirements.txt` | Full file | sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild |

**Folders Explored:**

| Folder Path | Depth | Purpose |
|---|---|---|
| (root) | Level 0 | Identify top-level project structure and documentation assets |
| `kitty/` | Level 1 | Core application sources — identified all relevant child process, window, boss, screen, and configuration files |
| `shell-integration/` | Level 1 | Shell integration scripts — identified bash integration with OSC 133 mechanism |
| `shell-integration/bash/` | Level 2 | Bash-specific integration — kitty.bash with PS1 OSC 133;D injection |
| `tools/tui/` | Level 2 | Go TUI library — hold.go, run.go |
| `tools/cmd/run_shell/` | Level 3 | run-shell kitten entry point |
| `kitty/options/` | Level 2 | Configuration definitions and types |
| `docs/` | Level 1 | Existing documentation tree — assessed infrastructure and existing coverage |

### 0.11.2 Tech Spec Sections Retrieved

| Section Heading | Relevance |
|---|---|
| 4.1 HIGH-LEVEL SYSTEM WORKFLOW | Application lifecycle overview, event loop architecture, system boundary architecture |
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | VT parser dispatch, screen model, GPU rendering pipeline |
| 4.5 WINDOW AND TAB LIFECYCLE | Child process launch, window state transitions (ChildExited → HoldMode/Closing) |
| 4.7 SHELL INTEGRATION FLOW | OSC 133 prompt markers, shell environment setup |
| 4.10 ERROR HANDLING AND RECOVERY FLOWS | Shell integration error isolation, startup error handling |
| 5.2 COMPONENT DETAILS | Child Monitor three-thread architecture, Boss controller, VT parser, Shell Integration |

### 0.11.3 Attachments

No attachments were provided for this project. No Figma screens or external design files were referenced.

