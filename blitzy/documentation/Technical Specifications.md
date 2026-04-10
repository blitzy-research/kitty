# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new comprehensive Q&A-style reference document** that traces the end-to-end lifecycle of kitty's remote control system, answering specific technical questions about the architecture, transport mechanisms, protocol formats, and code paths involved when a `kitten @` command is issued.

**Category:** Create new documentation
**Documentation Type:** Technical deep-dive / architecture guide / Q&A reference document

The user's questions break down into the following concrete documentation requirements:

- **Transport Discovery Mechanism**: How does `kitten @ ls` locate a running kitty instance? Is it a socket, pipe, or something else? What is the actual socket path and how is it generated?
- **End-to-End Command Trace**: A full code-path trace from `kitten @ ls` invocation through to the JSON response, including the Go-side client, the transport layer, the Python-side server parsing, and the handler dispatch to `kitty/rc/ls.py`.
- **Shell Integration Connection**: How does shell integration enable remote control "without explicit configuration"? What environment variables bridge shell integration and remote control? Is it the same socket or a TTY-based mechanism?
- **Protocol Wire Format**: The actual DCS escape sequence framing, the JSON protocol structure, and a real example of `ls` command request/response payloads.
- **Logging and Debugging Behavior**: Where logging occurs during remote control processing, and how to use `--dump-commands`/`--dump-bytes` for diagnostics.
- **Command Handler Architecture**: How `kitty/rc/base.py` defines the `RemoteCommand` framework, and how new commands are registered and dispatched.

**Implicit documentation needs surfaced:**
- The role of `KITTY_PUBLIC_KEY` and the encryption handshake for password-protected remote control
- The `fd:`-based socket pair mechanism used for per-window remote control permissions
- The authorization chain (global allow, socket-only, password-based, custom auth scripts)
- The `child-monitor.c` peer management that bridges C-level socket accept with Python-level command dispatch

### 0.1.2 Special Instructions and Constraints

- **CRITICAL**: The user explicitly requires: "Don't modify any files in the repository. If you need to create temporary scripts for testing, that's fine, but don't change the actual codebase files. And delete all those temporary scripts/files after task completion."
- **Implementation Rule**: Per project rules, the output must be a new markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation` directory.
- **Style**: The document should provide thinking and rationale behind each answer, tracing through the actual code as the source of truth.
- **No assumptions** — every claim must be backed by evidence from the source code.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the transport discovery mechanism**, we will trace through `tools/cmd/at/main.go:setup_global_options()` → `os.Getenv("KITTY_LISTEN_ON")` → `utils.ParseSocketAddress()` → `do_socket_io()` vs `do_tty_io()`, and on the server side `kitty/main.py:expand_listen_on()` → `kitty/boss.py:listen_on()`.
- To **document the end-to-end command trace**, we will follow the path from `tools/cmd/at/main.go:send_rc_command()` through serialization, transport, to `kitty/boss.py:peer_message_received()` → `_handle_remote_command()` → `kitty/remote_control.py:handle_cmd()` → `kitty/rc/ls.py:LS.response_from_kitty()`.
- To **document the shell integration connection**, we will trace `kitty/child.py:get_final_env()` which sets `KITTY_LISTEN_ON` and `KITTY_PUBLIC_KEY`, combined with `kitty/shell_integration.py:modify_shell_environ()` and the per-window `fd:` socket pair mechanism in `kitty/boss.py`.
- To **document the protocol wire format**, we will cite the DCS framing in `tools/cmd/at/socket_io.go` (prefix `\x1bP@kitty-cmd`, suffix `\x1b\\`) and the JSON schema from `docs/rc_protocol.rst`.
- To **document logging behavior**, we will reference `kitty/boss.py:DumpCommands`, `kitty/remote_control.py:log_error()`, and the `--dump-commands`/`--dump-bytes` CLI options.

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `kitty/rc/` directory contains 41 command modules, each following a consistent `RemoteCommand` subclass pattern. Documenting this pattern is essential for the user's stated goal of "eventually adding a new command."
- Based on structure: The transport layer spans Go (`tools/cmd/at/`) and Python (`kitty/remote_control.py`), requiring consolidated cross-language documentation.
- Based on dependencies: The encryption subsystem (`tools/crypto/`, `kitty/remote_control.py:CommandEncrypter`) ties into remote control via `KITTY_PUBLIC_KEY` and needs documentation to explain the "without explicit configuration" shell integration pathway.
- Based on user journey: The user needs a setup guide (how to enable RC), a tracing guide (step-by-step code walkthrough), an example section (real JSON payloads), and a "how to add a new command" checklist.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx-based documentation system with comprehensive coverage of remote control concepts, but lacking a single unified code-trace document connecting all layers.

**Documentation framework:** Sphinx (version unspecified in `docs/requirements.txt`, but the file lists `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`)

**Documentation generator configuration:** `docs/conf.py` (Sphinx configuration) and `docs/Makefile` (build driver)

**Existing remote-control documentation found:**

| File | Lines | Coverage |
|------|-------|----------|
| `docs/remote-control.rst` | 352 | Tutorial, socket-based RC, shell usage, password auth, custom auth scripts, key mappings |
| `docs/rc_protocol.rst` | 107 | Wire protocol specification, JSON schema, encryption protocol, async/streaming |
| `docs/shell-integration.rst` | 470 | Shell integration features, prompt markers, completions, CWD tracking |

**API documentation tools in use:** Sphinx with reStructuredText; generated docs under `docs/generated/` (referenced via `.. include:: generated/rc.rst` and `.. include:: generated/cli-kitten-at.rst`)

**Diagram tools detected:** Mermaid diagrams are used in the technical specification sections (4.6, 4.7) but the Sphinx docs themselves use inline text descriptions rather than embedded diagrams.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code to document:

- **Remote control server-side:** `kitty/remote_control.py` (524 lines) — core protocol handling, encryption, transport classes, authorization
- **RC command framework:** `kitty/rc/base.py` (466 lines) — `RemoteCommand` base class, payload handling, matching infrastructure
- **RC command implementations:** `kitty/rc/*.py` — 41 command modules including `ls.py`, `send_text.py`, `launch.py`, `set_colors.py`, etc.
- **Boss controller:** `kitty/boss.py` (3094 lines) — `_handle_remote_command()`, `peer_message_received()`, `list_os_windows()`, authorization chain
- **Go client entry:** `tools/cmd/at/main.go` (411 lines) — `send_rc_command()`, `setup_global_options()`, serialization, `EntryPoint()`
- **Socket transport:** `tools/cmd/at/socket_io.go` (185 lines) — `do_socket_io()`, DCS framing, `response_reader`
- **TTY transport:** `tools/cmd/at/tty_io.go` (175 lines) — `do_tty_io()`, chunked terminal I/O via event loop
- **Socket address parsing:** `tools/utils/sockets.go` (50 lines) — `ParseSocketAddress()` supporting `unix:`, `tcp:`, `fd:` protocols
- **Constants/paths:** `kitty/constants.py` (305 lines) — `runtime_dir()`, version, `RC_ENCRYPTION_PROTOCOL_VERSION`
- **Child environment:** `kitty/child.py` (line 246) — `KITTY_LISTEN_ON` and `KITTY_PUBLIC_KEY` injection
- **Socket path expansion:** `kitty/main.py` (lines 325-343) — `expand_listen_on()` with PID suffixing and tempdir resolution
- **Shell integration:** `kitty/shell_integration.py` (233 lines) — environment setup for bash/zsh/fish
- **C-level peer management:** `kitty/child-monitor.c` — `accept_peer()`, `queue_peer_message()`, `inject_peer()`

Key directories examined:
- `kitty/rc/` — all 41 RC command modules
- `tools/cmd/at/` — Go-side client implementation (18 files)
- `tools/crypto/` — encryption subsystem
- `tools/utils/` — socket parsing utilities
- `shell-integration/` — shell startup scripts (bash, zsh, fish, ssh)
- `docs/` — existing Sphinx documentation (44+ rst files)

### 0.2.3 Related Documentation Found

| Document | Relevance |
|----------|-----------|
| `docs/remote-control.rst` | Primary user-facing remote control guide; covers tutorial, socket usage, passwords, mappings |
| `docs/rc_protocol.rst` | Wire protocol specification with JSON schema and encryption details |
| `docs/shell-integration.rst` | Shell integration features; indirectly enables RC via environment variables |
| `CONTRIBUTING.md` | Contributor workflows for bug reporting and code submissions |
| `README.asciidoc` | Project introduction with links to website, FAQ, CI status |
| `docs/conf.rst` | Configuration file documentation including `allow_remote_control` and `listen_on` options |

### 0.2.4 Web Search Research Conducted

No web search was required for this documentation task. The existing repository source code and Sphinx documentation provide comprehensive primary sources. The `docs/rc_protocol.rst` file contains the authoritative wire protocol specification, and all architecture details are traceable directly in the codebase.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage in the new reference document:

**Module: `kitty/remote_control.py`**
- Public APIs: `encode_send()`, `parse_cmd()`, `handle_cmd()`, `do_io()`, `create_basic_command()`, `send_response_to_client()`, `SocketIO`, `RCIO`, `CommandEncrypter`, `NoEncryption`, `PasswordAuthorizer`
- Current documentation: Partially covered in `docs/rc_protocol.rst` (wire format only); internal code flow undocumented
- Documentation needed: End-to-end trace, transport selection logic, encryption flow, authorization chain

**Module: `kitty/rc/base.py`**
- Public APIs: `RemoteCommand` base class, `command_for_name()`, `all_command_names()`, `PayloadGetter`, `parse_subcommand_cli()`, `NoResponse`, `AsyncResponder`, `StreamInFlight`
- Current documentation: Implicitly documented through individual command help strings
- Documentation needed: Framework architecture guide for adding new commands

**Module: `kitty/rc/ls.py`**
- Public APIs: `LS` class with `message_to_kitty()`, `response_from_kitty()`, `protocol_spec`, `options_spec`
- Current documentation: CLI help text exists; JSON output format not documented with examples
- Documentation needed: Concrete JSON response example, field descriptions

**Module: `kitty/boss.py` (remote control methods)**
- Public APIs: `_handle_remote_command()`, `peer_message_received()`, `handle_remote_cmd()`, `list_os_windows()`, `ask_if_remote_cmd_is_allowed()`, `listen_on()`
- Current documentation: Not documented at code-trace level
- Documentation needed: Server-side dispatch flow, peer management, authorization decision tree

**Module: `tools/cmd/at/main.go`**
- Public APIs: `send_rc_command()`, `setup_global_options()`, `EntryPoint()`, `create_serializer()`, `get_pubkey()`
- Current documentation: CLI help via `--help`; internal architecture undocumented
- Documentation needed: Client-side command construction, transport selection logic

**Module: `tools/cmd/at/socket_io.go` and `tty_io.go`**
- Public APIs: `do_socket_io()`, `do_tty_io()`, `simple_socket_io()`, `do_chunked_io()`
- Current documentation: Not documented
- Documentation needed: Transport implementation details, DCS framing

**Module: `kitty/main.py` (listen_on expansion)**
- Public APIs: `expand_listen_on()`
- Current documentation: Referenced in `docs/remote-control.rst` tutorial but implementation details not traced
- Documentation needed: Socket path generation rules, PID templating, tempdir resolution

**Module: `kitty/child.py` (environment propagation)**
- Public APIs: `get_final_env()` (lines 234-250)
- Current documentation: Not documented at code level
- Documentation needed: How `KITTY_LISTEN_ON` and `KITTY_PUBLIC_KEY` flow to child processes

**Module: `kitty/child-monitor.c` (peer management)**
- Public APIs: `accept_peer()`, `inject_peer()`, `queue_peer_message()`
- Current documentation: Not documented
- Documentation needed: C-level socket accept, peer lifecycle, message queueing to Python

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No unified code-trace document**: The existing docs cover usage (`remote-control.rst`) and protocol (`rc_protocol.rst`) separately, but no document traces a command end-to-end across Go client → transport → C peer manager → Python handler → RC module → response.
- **Socket path generation undocumented at code level**: The user is confused about where to find sockets in `/tmp` — the `expand_listen_on()` function that appends `-{kitty_pid}` and resolves relative paths to `tempfile.gettempdir()` is not documented.
- **Shell integration ↔ remote control bridge undocumented**: The mechanism by which `KITTY_LISTEN_ON` is set in child processes via `child.py:get_final_env()` is not explained in existing docs.
- **TTY-based transport (DCS escape sequence path) undocumented at code level**: While `rc_protocol.rst` shows the escape sequence format, the actual code flow through `RCIO`/`do_tty_io()` → kitty's VT parser → `handle_remote_cmd()` is not traced.
- **`fd:` socket pair mechanism undocumented**: The per-window remote control via `socket.socketpair()` + `inject_peer()` in `boss.py` is not documented anywhere.
- **Logging/debugging workflow undocumented**: The `DumpCommands` class and `--dump-commands`/`--dump-bytes` flags for debugging RC are not documented in the context of remote control troubleshooting.
- **Real JSON examples missing**: The protocol spec shows the schema but no complete real-world `ls` request/response pair.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/kitty_815df1e210e0.md` will be a single comprehensive markdown file structured as follows:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction and Rationale
        ├── 1. Transport Discovery: How kitten @ Finds Kitty
        │   ├── 1.1 The --to Flag and KITTY_LISTEN_ON
        │   ├── 1.2 Socket Path Generation (expand_listen_on)
        │   ├── 1.3 Socket vs TTY: Transport Selection Logic
        │   └── 1.4 The fd: Socket Pair Mechanism
        ├── 2. End-to-End Command Trace: kitten @ ls
        │   ├── 2.1 Go Client: Command Construction
        │   ├── 2.2 Serialization and DCS Framing
        │   ├── 2.3 Socket Transport Path
        │   ├── 2.4 TTY Transport Path
        │   ├── 2.5 C-Level Peer Management
        │   ├── 2.6 Python Server Dispatch
        │   ├── 2.7 The ls Handler
        │   └── 2.8 Response Serialization
        ├── 3. Shell Integration and Remote Control
        │   ├── 3.1 Environment Variable Injection
        │   ├── 3.2 KITTY_PUBLIC_KEY and Encryption
        │   └── 3.3 Why It Works "Without Configuration"
        ├── 4. Protocol Wire Format
        │   ├── 4.1 DCS Frame Structure
        │   ├── 4.2 JSON Command Schema
        │   ├── 4.3 Real ls Request/Response Example
        │   └── 4.4 Encrypted Command Format
        ├── 5. Authorization and Security
        │   ├── 5.1 Authorization Decision Tree
        │   └── 5.2 Password-Based and Custom Auth
        ├── 6. Logging and Debugging
        │   ├── 6.1 DumpCommands and --dump-bytes
        │   └── 6.2 Error Logging Points
        └── 7. Adding a New Remote Control Command
            ├── 7.1 The RemoteCommand Framework
            ├── 7.2 Step-by-Step Checklist
            └── 7.3 Code Generation Pipeline
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract transport selection logic from `tools/cmd/at/main.go:send_rc_command()` (line 280: `utils.IfElse(global_options.to_network == "", do_tty_io, do_socket_io)`)
- Extract socket path generation rules from `kitty/main.py:expand_listen_on()` (lines 325-343)
- Extract environment propagation from `kitty/child.py:get_final_env()` (lines 246-249)
- Extract DCS framing constants from `tools/cmd/at/socket_io.go:cmd_escape_code_prefix` and `cmd_escape_code_suffix`
- Extract JSON schema from `docs/rc_protocol.rst` and `kitty/remote_control.py:create_basic_command()`
- Generate response examples by analyzing `kitty/rc/ls.py:LS.response_from_kitty()` and `kitty/boss.py:list_os_windows()`
- Extract authorization flow from `kitty/boss.py:_handle_remote_command()` (lines 590-650)
- Extract the `RemoteCommand` pattern from `kitty/rc/base.py:RemoteCommand` class (lines 319-429)

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration using triple-backtick mermaid blocks for end-to-end flow visualization
- Code examples using triple-backtick language blocks with syntax highlighting (Python, Go, JSON, shell)
- Source citations as inline references: `Source: kitty/remote_control.py:52`
- Tables for comparison of transport modes, authorization layers, environment variables

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Transport Selection Flowchart**: Decision tree showing how `kitten @` chooses between socket I/O and TTY I/O based on `--to` flag, `KITTY_LISTEN_ON`, and `fd:` presence
- **End-to-End Sequence Diagram**: Full trace from `kitten @ ls` → Go serialization → DCS framing → C peer accept → Python parse → ls handler → JSON response → client output
- **Authorization Chain Diagram**: Flowchart of the multi-layer authorization in `_handle_remote_command()`
- **Environment Variable Propagation**: How kitty sets `KITTY_LISTEN_ON`, `KITTY_PUBLIC_KEY`, `KITTY_WINDOW_ID` in child processes
- **Socket Path Resolution**: Decision tree for `expand_listen_on()` showing PID suffixing, tempdir resolution, and abstract socket handling

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/remote_control.py`, `kitty/rc/base.py`, `kitty/rc/ls.py`, `kitty/boss.py`, `kitty/main.py`, `kitty/child.py`, `kitty/constants.py`, `kitty/shell_integration.py`, `kitty/child-monitor.c`, `tools/cmd/at/main.go`, `tools/cmd/at/socket_io.go`, `tools/cmd/at/tty_io.go`, `tools/utils/sockets.go`, `docs/remote-control.rst`, `docs/rc_protocol.rst` | Comprehensive Q&A document answering all user questions about kitty's remote control system with end-to-end code traces, protocol examples, and architectural diagrams |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Deep-Dive / Q&A Reference Document
Source Code: Multiple files across kitty/, tools/cmd/at/, tools/utils/, docs/
Sections:
    - Introduction (purpose, rationale, document scope)
    - Transport Discovery (socket path generation, KITTY_LISTEN_ON, socket vs TTY selection)
    - End-to-End Command Trace (Go client → serialization → transport → C peer → Python dispatch → ls handler → response)
    - Shell Integration Connection (KITTY_LISTEN_ON injection, KITTY_PUBLIC_KEY, fd: mechanism)
    - Protocol Wire Format (DCS framing, JSON schema, real ls example)
    - Authorization and Security (decision tree, password auth, custom scripts)
    - Logging and Debugging (DumpCommands, --dump-bytes, error logging)
    - Adding a New Command (RemoteCommand framework, step-by-step checklist)
Diagrams:
    - Transport selection flowchart (Mermaid)
    - End-to-end sequence diagram (Mermaid)
    - Authorization chain diagram (Mermaid)
    - Environment variable propagation diagram (Mermaid)
    - Socket path resolution diagram (Mermaid)
Key Citations:
    - kitty/remote_control.py (transport classes, encryption, authorization)
    - kitty/rc/base.py (RemoteCommand framework)
    - kitty/rc/ls.py (ls command implementation)
    - kitty/boss.py (server-side dispatch, peer management)
    - kitty/main.py (socket path expansion)
    - kitty/child.py (environment propagation)
    - kitty/constants.py (runtime_dir, version)
    - kitty/child-monitor.c (C-level peer management)
    - tools/cmd/at/main.go (Go client entry)
    - tools/cmd/at/socket_io.go (socket transport)
    - tools/cmd/at/tty_io.go (TTY transport)
    - tools/utils/sockets.go (address parsing)
    - docs/rc_protocol.rst (wire protocol spec)
    - docs/remote-control.rst (user-facing RC guide)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are needed. The output file is a standalone Markdown document placed in `blitzy/documentation/` per the project implementation rules. It does not integrate into the existing Sphinx documentation build system.

### 0.5.4 Cross-Documentation Dependencies

- The new document references content from `docs/remote-control.rst` and `docs/rc_protocol.rst` but does not modify them
- The new document complements the existing Sphinx docs by providing the internal code-trace perspective that the user-facing docs intentionally omit
- No navigation links, table of contents updates, or index/glossary updates are needed since the output is in a separate `blitzy/documentation/` directory

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task produces a standalone Markdown file and does not require any documentation generation tools. The document is hand-authored based on source code analysis.

**Project runtime dependencies relevant to the documentation scope** (for reference/accuracy validation only — not installed for this task):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| PyPI | python | >=3.8 | Runtime for kitty Python modules (per `pyproject.toml`) |
| Go modules | go | 1.22 | Runtime for Go-based kitten CLI (per `go.mod`) |
| PyPI | sphinx | (unspecified) | Existing docs build tool (per `docs/requirements.txt`) |
| PyPI | furo | (unspecified) | Sphinx theme for existing docs |
| Go modules | golang.org/x/sys | 0.21.0 | Unix system calls for TTY and socket operations |
| Go modules | github.com/seancfoley/ipaddress-go | 1.6.0 | IP address parsing for TCP socket addresses |

**Existing documentation tools in the repository** (for context, not modified):

| Tool | Config Location | Purpose |
|------|----------------|---------|
| Sphinx | `docs/conf.py` | reStructuredText documentation site generator |
| Furo theme | `docs/conf.py` | Material-style HTML theme for docs |
| sphinx-copybutton | `docs/requirements.txt` | Copy buttons on code blocks |
| sphinx-autobuild | `docs/requirements.txt` | Live-preview development workflow |
| Makefile | `docs/Makefile` | Build driver for `make html`, `make develop-docs` |

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are required. The new file is a standalone document in the `blitzy/documentation/` directory and does not need to integrate with or update links in existing documentation files.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis for the user's questions:**

| User Question | Existing Doc Coverage | Gap |
|---------------|----------------------|-----|
| How does `kitten @ ls` find the running kitty instance? | `docs/remote-control.rst` mentions `--to` and `KITTY_LISTEN_ON` at usage level | No code-level trace of transport selection logic |
| What is the actual socket path? | `docs/remote-control.rst` shows `unix:/tmp/mykitty` example | `expand_listen_on()` PID-suffixing and tempdir behavior undocumented |
| Is it a socket, pipe, or something else? | `docs/rc_protocol.rst` mentions DCS escape sequences | No explanation of dual-path (socket vs TTY) architecture |
| How does shell integration enable RC without configuration? | `docs/shell-integration.rst` covers features but not RC bridge | `KITTY_LISTEN_ON` injection via `child.py` undocumented |
| What do protocol messages look like on the wire? | `docs/rc_protocol.rst` shows JSON schema | No real `ls` request/response example |
| Where in Python code does incoming command get parsed? | Not documented | Entire server-side dispatch chain undocumented at code level |
| How is the command routed to the `ls` handler? | Not documented | `command_for_name()` → dynamic import undocumented |
| What JSON comes back from `ls`? | Not documented with examples | `list_os_windows()` output structure undocumented |

**Target coverage:** 100% of user questions answered with code-level evidence and working examples.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question receives a dedicated section with code citations
- All code paths are traced with file paths and line numbers
- At least one complete JSON example for `ls` request and response
- At least three Mermaid diagrams (transport selection, end-to-end flow, authorization chain)

**Accuracy validation:**
- Every code citation must reference actual file paths and line numbers verified during repository analysis
- JSON examples must match the schema defined in `docs/rc_protocol.rst` and the `create_basic_command()` function
- Socket path generation rules must match the logic in `expand_listen_on()`
- Authorization flow must match the conditional chain in `_handle_remote_command()`

**Clarity standards:**
- Technical accuracy with accessible language for a developer already reading kitty source code
- Progressive disclosure: overview first, then detailed code trace
- Consistent use of "client-side" (Go) vs "server-side" (Python/C) terminology
- Cross-references between sections where concepts overlap

**Maintainability:**
- All source citations include file paths for traceability
- The document is self-contained and does not depend on external resources

### 0.7.3 Example and Diagram Requirements

- Minimum 1 complete request/response JSON pair for `ls` command
- Minimum 1 DCS-framed wire format example
- 5 Mermaid diagrams: transport selection, end-to-end trace, authorization chain, environment propagation, socket path resolution
- Code snippets limited to 2-3 key lines per reference point for readability

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — the sole deliverable document

**Source code analyzed for documentation content (read-only):**
- `kitty/remote_control.py` — transport, encryption, authorization, protocol handling
- `kitty/rc/base.py` — RemoteCommand framework, command discovery, payload handling
- `kitty/rc/ls.py` — ls command implementation (exemplar for the user's trace)
- `kitty/rc/*.py` — all 41 command modules (for framework documentation)
- `kitty/boss.py` — server-side dispatch, peer management, authorization chain, `list_os_windows()`
- `kitty/main.py` — `expand_listen_on()` socket path generation
- `kitty/child.py` — `get_final_env()` environment propagation
- `kitty/constants.py` — `runtime_dir()`, version constants, `RC_ENCRYPTION_PROTOCOL_VERSION`
- `kitty/shell_integration.py` — shell environment modification
- `kitty/child-monitor.c` — C-level peer accept, message queueing, fd injection
- `tools/cmd/at/main.go` — Go client entry, `send_rc_command()`, global options
- `tools/cmd/at/socket_io.go` — socket transport, DCS framing
- `tools/cmd/at/tty_io.go` — TTY transport, chunked I/O
- `tools/utils/sockets.go` — `ParseSocketAddress()` protocol parsing
- `tools/crypto/` — encryption utilities (referenced but not deeply traced)
- `docs/remote-control.rst` — existing user-facing RC documentation
- `docs/rc_protocol.rst` — existing wire protocol specification
- `docs/shell-integration.rst` — existing shell integration documentation
- `shell-integration/bash/kitty.bash` — bash integration script (environment context)
- `shell-integration/zsh/kitty.zsh` — zsh integration script (environment context)
- `shell-integration/fish/` — fish integration scripts (environment context)

**Topics in scope:**
- Transport discovery and selection (socket vs TTY vs fd)
- Socket path generation and resolution
- End-to-end command trace for `kitten @ ls`
- Shell integration ↔ remote control bridge
- Protocol wire format with real examples
- Authorization and security chain
- Logging and debugging mechanisms
- RemoteCommand framework architecture (for adding new commands)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications**: No files in the repository will be modified (per user instructions)
- **Test file modifications**: No test files will be created or modified
- **Feature additions or code refactoring**: No functional changes to the codebase
- **Deployment configuration changes**: No changes to build, CI, or deployment configs
- **Existing documentation updates**: No modifications to `docs/remote-control.rst`, `docs/rc_protocol.rst`, or any other existing documentation
- **Non-RC topics**: Graphics protocol, keyboard protocol, file transfer protocol, and other kitty subsystems not related to remote control
- **Individual command documentation**: The document covers the `ls` command as an exemplar but does not document all 41 RC commands individually
- **Shell integration features unrelated to RC**: Prompt markers, cursor shape, CWD notifications, completions (only the RC-bridge aspect of shell integration is in scope)
- **SSH bootstrap system**: The `shell-integration/ssh/` bootstrap is out of scope except as it relates to `KITTY_PUBLIC_KEY` propagation

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command**: Not applicable — the output is a standalone Markdown file that does not require a build step
- **Documentation preview command**: Any Markdown renderer (e.g., `grip blitzy/documentation/kitty_815df1e210e0.md`, VS Code preview, or GitHub rendering)
- **Diagram generation command**: Mermaid diagrams are embedded inline in the Markdown and rendered by any Mermaid-compatible viewer
- **Documentation deployment command**: Not applicable — the file is committed to the repository
- **Default format**: Markdown with embedded Mermaid diagrams
- **Citation requirement**: Every technical claim must reference source file paths and line numbers
- **Style guide**: Follow the user's stated preference for "thinking/rationale behind the answers" with code-as-truth approach. Progressive disclosure from high-level overview to detailed code trace.
- **Documentation validation**: Manual review of file path citations against actual repository structure

### 0.9.2 Output File Specification

- **File path**: `blitzy/documentation/kitty_815df1e210e0.md`
- **Naming convention**: `<source_branch_name>.md` where source branch is `kitty_815df1e210e0`
- **Directory**: `blitzy/documentation/` (to be created)
- **Format**: GitHub-Flavored Markdown with Mermaid diagram support
- **Encoding**: UTF-8
- **No modifications to existing files**: Per user instruction, no repository files are modified
- **Temporary files**: Any temporary scripts created for testing must be deleted after task completion

## 0.10 Rules for Documentation

The following rules are explicitly specified or directly inferred from the user's instructions:

- **Do not modify any existing files in the source repository** — the document must be created as a new file only
- **If temporary scripts are created for testing, delete them after task completion** — clean up any transient artifacts
- **Base all answers on the code as the source of truth** — no assumptions; every claim must be verifiable in the codebase
- **Provide thinking and rationale behind the answers** — explain not just "what" but "why" and "how we know" for each finding
- **Create the output document as `kitty_815df1e210e0.md`** in the `blitzy/documentation` directory per the SWE-AtlasQnA-Repo implementation rule
- **Cover the full end-to-end trace** including socket path generation, transport selection, DCS framing, C-level peer management, Python dispatch, `ls` handler, and JSON response format
- **Include real protocol examples** showing the JSON wire format for both request and response
- **Explain the shell integration connection** to remote control, particularly the environment variable bridge (`KITTY_LISTEN_ON`, `KITTY_PUBLIC_KEY`)
- **Document logging behavior** including `DumpCommands`, `--dump-commands`, `--dump-bytes`, and error logging points
- **Address the user's specific confusion points**: socket path location in `/tmp`, the "without explicit configuration" shell integration mechanism, and the TTY vs socket transport distinction

## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

**Root-level exploration:**
- Root folder (`""`) — repository structure overview, build files, documentation
- `pyproject.toml` — Python version requirements (`>=3.8`)
- `go.mod` — Go module version (`go 1.22`) and dependency graph
- `setup.py` — build system, version validation

**Core remote control system files (read in full):**
- `kitty/remote_control.py` (524 lines) — transport classes, encryption, authorization, protocol handling
- `kitty/rc/base.py` (466 lines) — RemoteCommand framework, command discovery, payload handling, matching infrastructure
- `kitty/rc/ls.py` (79 lines) — ls command implementation, JSON response generation
- `kitty/rc/__init__.py` — package marker

**Server-side dispatch and infrastructure:**
- `kitty/boss.py` (3094 lines) — lines 160-200 (`listen_on()`), 340-380 (initialization), 432-500 (`list_os_windows()`), 590-680 (`_handle_remote_command()`, authorization chain), 776-870 (`peer_message_received()`, `handle_remote_cmd()`)
- `kitty/constants.py` (305 lines) — `runtime_dir()`, version constants, `RC_ENCRYPTION_PROTOCOL_VERSION`
- `kitty/main.py` — lines 325-343 (`expand_listen_on()`), 405-409 (listen_on setup)
- `kitty/child.py` — lines 200-260 (`Child.__init__()`, `get_final_env()` with `KITTY_LISTEN_ON` and `KITTY_PUBLIC_KEY` injection)
- `kitty/entry_points.py` (197 lines) — CLI entry points, `main()` dispatch
- `kitty/utils.py` — lines 502-525 (`parse_address_spec()` with `unix:`, `tcp:`, `tcp6:` parsing)
- `kitty/child-monitor.c` — lines 1620-1680 (`accept_peer()`, `queue_peer_message()`, `inject_peer()`)
- `kitty/shell_integration.py` (233 lines) — `modify_shell_environ()`, shell-specific env setup

**Go client-side implementation:**
- `tools/cmd/at/main.go` (411 lines) — `send_rc_command()`, `setup_global_options()`, `EntryPoint()`, serialization
- `tools/cmd/at/socket_io.go` (185 lines) — `do_socket_io()`, `simple_socket_io()`, DCS framing, `response_reader`
- `tools/cmd/at/tty_io.go` (175 lines) — `do_tty_io()`, `do_chunked_io()`, TTY event loop transport
- `tools/utils/sockets.go` (50 lines) — `ParseSocketAddress()` with `unix:`, `tcp:`, `fd:` handling
- `tools/cmd/main.go` — Go binary entry point

**Existing documentation:**
- `docs/remote-control.rst` (352 lines) — user-facing remote control guide
- `docs/rc_protocol.rst` (107 lines) — wire protocol specification
- `docs/shell-integration.rst` (470 lines) — shell integration features
- `docs/requirements.txt` — Sphinx documentation dependencies
- `docs/conf.py` — Sphinx configuration (referenced)

**Folder structure explored:**
- `kitty/` — core application tree (all children enumerated)
- `kitty/rc/` — all 41 RC command modules listed
- `tools/` — Go support library tree
- `tools/cmd/` — command assembly layer
- `tools/cmd/at/` — remote control `@` command (18 files)
- `kittens/` — kitten subsystem (21 subpackages)
- `shell-integration/` — shell integration scripts (bash, zsh, fish, ssh)
- `docs/` — Sphinx documentation tree (44+ files)

**Technical specification sections retrieved:**
- Section 4.6: REMOTE CONTROL SYSTEM FLOW — command execution sequence, authorization chain, transport selection
- Section 4.7: SHELL INTEGRATION FLOW — environment setup, SSH bootstrap, active integration features

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

