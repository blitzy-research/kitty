# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a comprehensive technical investigation document** that traces the complete journey of file data through kitty's file transfer protocol over SSH connections. This document will serve as an onboarding guide for a developer seeking deep understanding of the protocol internals, validated through hands-on experimentation.

- **Category:** Create new documentation
- **Documentation Type:** Technical investigation / Architecture deep-dive / Protocol walkthrough guide
- **Output Artifact:** A new markdown file named `kitty_815df1e210e0.md` placed in the `blitzy/documentation/` directory

The documentation requirements, restated with enhanced clarity:

- **Build-from-source walkthrough:** Document the process of building kitty from source, establishing an SSH connection via the SSH kitten (`kittens/ssh/`), and initiating a file transfer via the transfer kitten (`kittens/transfer/`)
- **Protocol handshake analysis:** Trace how the transfer kitten initiates the protocol handshake, identifying the specific OSC 5113 escape sequences (`\x1b]5113;....\x1b\\`) that establish a transfer session over the terminal connection, as implemented in `kittens/transfer/send.go` and `kittens/transfer/receive.go`
- **Rsync-style delta transfer internals:** Explain how kitty implements rsync-style delta transfer, documenting the data structures (`BlockHash`, `rolling_checksum`, `Operation`, `Differ`, `Patcher`) in `tools/rsync/algorithm.go` and `tools/rsync/api.go` that track file signatures and differences
- **Data flow and encoding:** Trace the actual byte-level data flow — how file chunks are base64-encoded inside OSC 5113 escape sequences in the terminal stream, and how the receiving side uses the terminal event loop (`lp.OnEscapeCode`) to demultiplex transfer data from regular terminal output
- **Transfer resumption behavior:** Investigate whether the protocol supports true resumption after interruption, documenting what state (if any) allows partial resumption and where resumption metadata is stored
- **Delta transfer efficiency demonstration:** Create a practical demonstration by transferring a file, modifying a small portion, and retransferring with `--transmit-deltas`, showing that the second transfer sends substantially less data and identifying the signature/delta mechanism that detected unchanged portions

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No source file modifications.** The user explicitly stated: "Do not modify any source files." All investigation must be read-only with respect to the repository source
- **Temporary test files permitted.** The user allows creating temporary test files for the demonstration, but they must be cleaned up afterward
- **Implementation rule compliance:** Per the `SWE-AtlasQnA-Repo` rule, the output must be a new markdown document named `kitty_815df1e210e0.md` placed in the `blitzy/documentation/` directory, comprehensively answering the questions posed, with rationale behind answers, based on code as the truth, and without modifying any existing files
- **Style preferences:** Technical depth with source code citations, Mermaid diagrams for protocol flows, code-referenced explanations (file:line format), and thinking/rationale behind each answer
- **No design system applies:** This is a pure documentation/investigation task with no UI components

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **protocol handshake**, we will trace the code path from `kittens/transfer/main.go:main()` through `send_main()` / `receive_main()` into `SendManager.start_transfer()` and `manager.start_transfer()`, identifying the exact OSC escape sequence construction in `SendManager.initialize()` at `kittens/transfer/send.go:384` where the prefix `\x1b]5113;id=<request_id>;` and suffix `\x1b\\` are assembled
- To document the **rsync delta mechanism**, we will analyze `tools/rsync/algorithm.go` (signature generation via `signature_iterator`, delta computation via `diff.read_next()`, and patching via `rsync.ApplyDelta()`) alongside `tools/rsync/api.go` (`Patcher`, `Differ`, `NewPatcher`, `NewDiffer`, `CreateSignatureIterator`, `CreateDelta`)
- To document the **data flow encoding**, we will trace how `split_for_transfer()` in `kittens/transfer/ftc.go:326` chunks data into 4096-byte segments, how `FileTransmissionCommand.Serialize()` base64-encodes payload fields, and how the receiver's `lp.OnEscapeCode` callback in `kittens/transfer/send.go:1224` / `kittens/transfer/receive.go:1121` parses incoming OSC payloads back into `FileTransmissionCommand` structs
- To document **transfer resumption**, we will investigate the protocol specification in `docs/file-transfer-protocol.rst` and the source code for any persistent state tracking, noting that the protocol as currently specified does **not** include built-in resumption — interrupted transfers restart from the beginning
- To demonstrate **delta efficiency**, we will design a test scenario using `--transmit-deltas` and observe the rsync stats output from `print_rsync_stats()` in `kittens/transfer/utils.go:109`

### 0.1.4 Inferred Documentation Needs

Based on code analysis, several implicit documentation needs have been identified:

- **Module interaction diagram:** The relationship between `kittens/transfer/` (Go command-line kitten), `kitty/file_transmission.py` (Python terminal-emulator side), and `tools/rsync/` (Go rsync library) requires a consolidated architecture diagram showing how the sender and receiver interact across the SSH tunnel
- **Escape sequence lifecycle:** The encoding from `FileTransmissionCommand` → serialized key=value → OSC 5113 wrapper → base64-encoded payload → terminal stream → receiver parsing needs a complete worked example
- **BlockHash data structure documentation:** The `BlockHash` struct (20 bytes: uint64 index + uint32 weak_hash + uint64 strong_hash) and the signature header (12 bytes) need visual documentation showing byte layout
- **State machine documentation:** Both sender (`WAITING_FOR_START` → `WAITING_FOR_DATA` → `TRANSMITTING` → `FINISHED` → `ACKNOWLEDGED`) and receiver (`state_waiting_for_permission` → `state_waiting_for_file_metadata` → `state_transferring`) state machines require Mermaid state diagrams
- **Compression decision logic:** The `should_be_compressed()` function in `kittens/transfer/utils.go:88` contains heuristics (file extension and MIME type checks) that should be documented
- **Security model documentation:** The bypass authentication mechanism using `KITTY_PUBLIC_KEY` environment variable and `encode_bypass()` in `kittens/transfer/utils.go:37` warrants documentation for the SSH context

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx/reStructuredText-based documentation system** with comprehensive coverage of protocols and kittens, but lacking a consolidated developer-oriented walkthrough of the file transfer data path.

- **Documentation framework:** Sphinx (version unspecified, uses `furo` theme)
- **Documentation generator configuration:** `docs/conf.py` (central Sphinx config), `docs/Makefile` (build driver)
- **Documentation build dependencies:** `docs/requirements.txt` — `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
- **Diagram tools detected:** Mermaid (used in tech spec), reStructuredText native diagrams, screenshots (`docs/kittens/transfer.rst` references `../screenshots/transfer.png`)
- **API documentation tools in use:** Auto-generated CLI references via `.. include:: /generated/cli-kitten-transfer.rst` directive in kitten docs; no standalone API doc generator (JSDoc, Godoc, etc.) configured for the Go code

**Existing file-transfer documentation found:**

| Document | Path | Purpose | Coverage Status |
|----------|------|---------|-----------------|
| File Transfer Protocol Spec | `docs/file-transfer-protocol.rst` | Authoritative protocol specification | Complete — covers session model, escape encoding, metadata, links, rsync deltas, compression, bypass auth |
| Transfer Kitten User Guide | `docs/kittens/transfer.rst` | End-user guide for the transfer kitten | Basic — covers usage, bypass, delta mode; lacks internals |
| SSH Kitten User Guide | `docs/kittens/ssh.rst` | SSH kitten documentation | Comprehensive for SSH setup; does not cover file transfer integration |
| Remote File Kitten Docs | `docs/kittens/remote_file.rst` | Remote file editing/downloading | Tangential — different workflow from bulk transfer |

**Gap identified:** No existing document traces the complete data path from user command through protocol handshake, rsync delta computation, base64 encoding, terminal stream multiplexing, and file reconstruction. The protocol spec (`docs/file-transfer-protocol.rst`) defines the wire format but not the implementation walkthrough. The user guide (`docs/kittens/transfer.rst`) covers invocation but not internals.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code to document:

- **Transfer kitten implementation:** `kittens/transfer/*.go`, `kittens/transfer/*.py`, `kittens/transfer/*.c`
- **Rsync library:** `tools/rsync/algorithm.go`, `tools/rsync/api.go`, `tools/rsync/api_test.go`
- **Python terminal-side handler:** `kitty/file_transmission.py`
- **SSH kitten (connection setup context):** `kittens/ssh/main.go`, `kittens/ssh/config.go`
- **Test coverage:** `kittens/transfer/ftc_test.go`, `kittens/transfer/send_test.go`, `kitty_tests/file_transmission.py`
- **Build system:** `setup.py`, `go.mod`, `pyproject.toml`, `Makefile`
- **Generated code:** `gen/go_code.py` (contains `FileTransferCode` constant generation)

Key directories examined:

| Directory | Contents | Relevance |
|-----------|----------|-----------|
| `kittens/transfer/` | 12 files (Go, Python, C, type stubs, tests) | Primary — sender/receiver, protocol model, rsync C extension |
| `tools/rsync/` | 3 files (Go algorithm, API, tests) | Primary — rsync delta engine |
| `kittens/ssh/` | 10 files (Go, Python, tests) | Context — SSH connection setup for transfer kitten availability |
| `kitty/` | `file_transmission.py` among many | Context — terminal emulator side of file transfer |
| `docs/` | Protocol spec, kitten docs, Sphinx config | Context — existing documentation to reference |

### 0.2.3 Web Search Research Conducted

No external web search is required for this task — the codebase contains the authoritative protocol specification (`docs/file-transfer-protocol.rst`), the complete implementation, and sufficient test coverage to answer all questions from source. The rsync algorithm reference cited in the code (`https://rsync.samba.org/tech_report/tech_report.html`) and the xxHash specification (`https://github.com/Cyan4973/xxHash/blob/dev/doc/xxhash_spec.md`) are external references that should be documented.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules requiring documentation:**

- **Module:** `kittens/transfer/ftc.go`
  - Public types: `Action`, `Compression`, `FileType`, `TransmissionType`, `QuietLevel`, `FileTransmissionCommand`
  - Key functions: `Serialize()`, `NewFileTransmissionCommand()`, `split_for_transfer()`, `safe_string()`
  - Current documentation: Protocol spec covers wire format; no implementation-level walkthrough exists
  - Documentation needed: Worked example of serialization/deserialization cycle, byte-level encoding walkthrough

- **Module:** `kittens/transfer/send.go`
  - Public types: `FileState`, `SendState`, `File`, `SendManager`, `SendHandler`, `ProgressTracker`
  - Key functions: `files_for_send()`, `send_main()`, `send_loop()`, `start_transfer()`, `transmit_next_chunk()`, `next_chunk()`, `on_file_transfer_response()`, `on_signature_data_received()`, `start_delta_calculation()`
  - Current documentation: None at implementation level
  - Documentation needed: Sender state machine, file discovery logic, delta negotiation flow, chunk transmission cycle

- **Module:** `kittens/transfer/receive.go`
  - Public types: `remote_file`, `manager`, `handler`, `patch_file`, `filesystem_file`, `receive_progress_tracker`
  - Key functions: `receive_main()`, `receive_loop()`, `request_files()`, `on_file_transfer_response()`, `finalize_transfer()`, `write_data()`, `collect_files()`
  - Current documentation: None at implementation level
  - Documentation needed: Receiver state machine, file reconstruction logic, rsync patching path, metadata finalization

- **Module:** `tools/rsync/algorithm.go`
  - Public types: `OpType`, `Operation`, `BlockHash`, `OperationWriter`
  - Key internal types: `rsync`, `rolling_checksum`, `signature_iterator`, `diff`
  - Key functions: `CreateSignatureIterator()`, `CreateDiff()`, `CreateDelta()`, `ApplyDelta()`
  - Current documentation: Protocol spec covers signature/delta format; no implementation guide
  - Documentation needed: Algorithm walkthrough, data structure diagrams, hash lookup mechanism

- **Module:** `tools/rsync/api.go`
  - Public types: `Api`, `Differ`, `Patcher`, `StrongHashType`, `WeakHashType`, `ChecksumType`
  - Key functions: `NewPatcher()`, `NewDiffer()`, `CreateSignatureIterator()`, `CreateDelta()`, `AddSignatureData()`, `StartDelta()`, `UpdateDelta()`, `FinishDelta()`
  - Current documentation: Code comments at top of file document workflow
  - Documentation needed: API usage flow, streaming signature/delta processing

- **Module:** `kittens/transfer/main.go`
  - Key functions: `main()`, `EntryPoint()`, `read_bypass()`
  - Current documentation: Minimal
  - Documentation needed: Entry point dispatch, direction-based routing to send/receive

- **Module:** `kittens/transfer/utils.go`
  - Key functions: `encode_bypass()`, `should_be_compressed()`, `print_rsync_stats()`, `random_id()`, `expand_home()`, `abspath()`
  - Current documentation: None
  - Documentation needed: Bypass encryption flow, compression heuristics

- **Module:** `kitty/file_transmission.py`
  - Key classes: `FileTransmissionCommand`, `PatchFile`, `DestFile`, `SourceFile`, `ActiveReceive`, `ActiveSend`, `FileTransmission`
  - Current documentation: None (implementation only)
  - Documentation needed: Terminal-emulator-side session management, how the emulator handles incoming transfer commands and dispatches to state machines

- **Module:** `kittens/ssh/main.go`
  - Relevant functions: SSH connection establishment, bootstrap generation, kitten binary deployment
  - Current documentation: `docs/kittens/ssh.rst` covers user-facing usage
  - Documentation needed: How SSH kitten makes transfer kitten available on remote host

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No end-to-end protocol walkthrough:** The protocol spec defines the format but no document traces a real transfer from command invocation to file completion
- **No rsync implementation guide:** The rsync algorithm in `tools/rsync/` has no developer documentation explaining the Go implementation of signature generation, rolling checksum, delta computation, or patch application
- **No data structure documentation:** The `BlockHash` (20 bytes), signature header (12 bytes), and operation types lack visual byte-layout diagrams
- **No state machine documentation:** Both sender and receiver state machines (`FileState`, `SendState`, `state` enum) are undocumented outside source comments
- **No escape sequence lifecycle documentation:** The journey from `FileTransmissionCommand` through `Serialize()` to OSC wrapping to terminal parsing has no consolidated explanation
- **No resumption behavior analysis:** No existing document addresses whether interrupted transfers can resume
- **No delta efficiency benchmarking guide:** No document shows how to measure and interpret rsync transfer efficiency using `print_rsync_stats()`
- **No cross-language interaction documentation:** The boundary between Go (kitten binary sending OSC commands) and Python (kitty terminal emulator processing them via `kitty/file_transmission.py`) is not documented

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive markdown document at `blitzy/documentation/kitty_815df1e210e0.md`:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction and Context
        ├── Building Kitty from Source
        ├── Establishing an SSH Connection via the SSH Kitten
        ├── Initiating a File Transfer
        ├── Protocol Handshake and Escape Sequences
        │   ├── OSC 5113 Escape Sequence Format
        │   ├── Session Initiation Flow
        │   ├── Permission Negotiation
        │   └── Worked Example: Serialization Cycle
        ├── Rsync-Style Delta Transfer
        │   ├── Data Structures (BlockHash, Operation, rolling_checksum)
        │   ├── Signature Generation
        │   ├── Delta Computation
        │   └── Patch Application
        ├── Data Flow: Encoding and Reassembly
        │   ├── Chunk Splitting and Base64 Encoding
        │   ├── Terminal Stream Multiplexing
        │   └── Receiver-Side Demultiplexing
        ├── Transfer Resumption Behavior
        ├── Delta Transfer Efficiency Demonstration
        │   ├── Test Setup
        │   ├── First Transfer (Full)
        │   ├── Second Transfer (Delta)
        │   └── Analysis of Efficiency
        └── Summary and Key Findings
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract protocol handshake flow by tracing `SendManager.initialize()` in `kittens/transfer/send.go:367-391` and `manager.start_transfer()` in `kittens/transfer/receive.go:464-470`
- Extract escape sequence encoding from `FileTransmissionCommand.Serialize()` in `kittens/transfer/ftc.go:163-222` and the OSC wrapping in `send_loop()` at `kittens/transfer/send.go:1223-1237`
- Generate rsync data structure documentation from `tools/rsync/algorithm.go:177-200` (`BlockHash`), `tools/rsync/algorithm.go:336-360` (`rolling_checksum`), and `tools/rsync/algorithm.go:72-92` (`Operation`)
- Create state machine diagrams from `FileState` enum at `kittens/transfer/send.go:37-43` and `state` enum at `kittens/transfer/receive.go:36-41`
- Document delta efficiency from `print_rsync_stats()` at `kittens/transfer/utils.go:109-114`
- Derive resumption analysis from absence of persistent state in the protocol spec (`docs/file-transfer-protocol.rst`) and implementation

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for protocol flows, state machines, and architecture
- Code snippets in fenced blocks with language tags for syntax highlighting
- Source citations as inline references: `Source: path/to/file.go:LineNumber`
- Tables for data structure layouts and enum definitions
- Consistent use of technical terminology aligned with codebase naming conventions

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create:

- **Architecture overview:** Shows interaction between SSH kitten, transfer kitten (Go binary on remote), terminal emulator (Python `kitty/file_transmission.py`), and rsync library
- **Protocol handshake sequence diagram:** Traces the send session from `action=send` through permission grant to `action=file` metadata
- **Sender state machine:** `WAITING_FOR_START` → `WAITING_FOR_DATA` → `TRANSMITTING` → `FINISHED` → `ACKNOWLEDGED`
- **Receiver state machine:** `state_waiting_for_permission` → `state_waiting_for_file_metadata` → `state_transferring`
- **Rsync delta workflow flowchart:** Signature generation → signature transmission → delta computation → delta transmission → patch application
- **Escape sequence encoding flowchart:** `FileTransmissionCommand` → `Serialize()` → `split_for_transfer()` → OSC wrapping → terminal write
- **Byte layout diagrams (tables):** Signature header (12 bytes), BlockHash (20 bytes), Operation types (variable)

### 0.4.4 Hands-On Demonstration Strategy

The delta transfer efficiency demonstration will follow this approach:

- Create a temporary test file of sufficient size (> 4096 bytes, as `rsync_capable` requires `file_size > 4096` per `kittens/transfer/send.go:131`)
- Transfer the file using the transfer kitten with `--transmit-deltas`
- Modify a small portion of the file
- Retransfer with `--transmit-deltas` and capture the rsync stats output
- Document that the second transfer transmits `delta_bytes + signature_bytes` which is substantially less than `total_bytes`
- Explain the mechanism: `rolling_checksum` identifies matching blocks via weak hash, `xxh3_64` strong hash confirms matches, and only non-matching data blocks are sent as `OpData` operations
- Note: Actual SSH-based demonstration may not be possible in the build environment; the document will describe the expected procedure and cite the code paths that produce the efficiency metrics

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kittens/transfer/ftc.go`, `kittens/transfer/send.go`, `kittens/transfer/receive.go`, `kittens/transfer/main.go`, `kittens/transfer/utils.go`, `kittens/transfer/algorithm.c`, `kittens/transfer/rsync.pyi`, `tools/rsync/algorithm.go`, `tools/rsync/api.go`, `kitty/file_transmission.py`, `kittens/ssh/main.go`, `docs/file-transfer-protocol.rst`, `docs/kittens/transfer.rst`, `docs/kittens/ssh.rst`, `docs/build.rst`, `setup.py`, `go.mod`, `pyproject.toml` | Complete technical investigation document answering all user questions about the file transfer protocol data path, rsync delta internals, escape sequence encoding, resumption behavior, and delta efficiency — with Mermaid diagrams, byte-layout tables, state machine documentation, code citations, and demonstration methodology |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation / Protocol Deep-Dive
Source Code:
  - kittens/transfer/ftc.go (protocol model: FileTransmissionCommand, enums, Serialize, parse)
  - kittens/transfer/send.go (sender state machine, file discovery, delta negotiation, chunk transmission)
  - kittens/transfer/receive.go (receiver state machine, file reconstruction, rsync patching)
  - kittens/transfer/main.go (entry point, direction dispatch)
  - kittens/transfer/utils.go (bypass encoding, compression heuristics, rsync stats)
  - kittens/transfer/algorithm.c (C rsync extension for Python side)
  - kittens/transfer/rsync.pyi (type stubs for C extension)
  - tools/rsync/algorithm.go (rsync engine: BlockHash, rolling_checksum, diff, Operation, ApplyDelta)
  - tools/rsync/api.go (public API: Patcher, Differ, signature/delta streaming)
  - kitty/file_transmission.py (terminal-emulator-side protocol handler)
  - kittens/ssh/main.go (SSH kitten connection establishment)
  - docs/file-transfer-protocol.rst (protocol specification)
  - docs/kittens/transfer.rst (user guide)
  - docs/kittens/ssh.rst (SSH kitten docs)
Sections:
  - Introduction and Context (purpose, scope, repository orientation)
  - Building Kitty from Source (build prerequisites, commands, Go toolchain)
  - Establishing an SSH Connection (kitten ssh workflow, bootstrap, kitten binary availability)
  - Initiating a File Transfer (kitten transfer invocation, direction/mode options)
  - Protocol Handshake and Escape Sequences
    - OSC 5113 format and key abbreviations
    - Session initiation: action=send → status=OK → action=file metadata
    - Permission negotiation and bypass authentication
    - Worked example: serialized escape code construction
  - Rsync-Style Delta Transfer
    - Data structures: BlockHash (20 bytes), signature header (12 bytes), Operation types
    - Signature generation: rolling_checksum, xxh3_64 strong hash, block iteration
    - Delta computation: hash_lookup map, window sliding, OpBlock/OpData/OpBlockRange/OpHash
    - Patch application: ApplyDelta with block seeking and checksum verification
  - Data Flow: Encoding and Reassembly
    - split_for_transfer chunking at 4096 bytes
    - Base64 encoding of data fields in Serialize()
    - Terminal stream multiplexing via OSC escape code wrapping
    - Receiver-side demultiplexing via lp.OnEscapeCode callback
  - Transfer Resumption Behavior (analysis showing no built-in resumption)
  - Delta Transfer Efficiency Demonstration
    - Test methodology
    - Expected rsync stats output analysis
    - Mechanism: unchanged block detection via rolling checksum + strong hash
  - Summary and Key Findings
Diagrams:
  - Architecture overview (SSH kitten → transfer kitten → terminal emulator)
  - Protocol handshake sequence diagram
  - Sender and receiver state machine diagrams
  - Rsync delta workflow flowchart
  - Escape sequence encoding pipeline
  - BlockHash and signature header byte layouts (tables)
Key Citations:
  - kittens/transfer/ftc.go:120-138 (FileTransmissionCommand struct)
  - kittens/transfer/ftc.go:163-222 (Serialize method)
  - kittens/transfer/ftc.go:326-338 (split_for_transfer)
  - kittens/transfer/send.go:37-43 (FileState enum)
  - kittens/transfer/send.go:83-108 (File struct with rsync fields)
  - kittens/transfer/send.go:363-391 (SendManager initialization, prefix/suffix construction)
  - kittens/transfer/send.go:759-791 (signature data received, delta calculation started)
  - kittens/transfer/send.go:915-981 (next_chunk with delta_loader)
  - kittens/transfer/send.go:1198-1262 (send_loop event loop with OnEscapeCode)
  - kittens/transfer/receive.go:69-123 (patch_file rsync patching)
  - kittens/transfer/receive.go:388-439 (request_files with signature generation)
  - kittens/transfer/receive.go:549-621 (on_file_transfer_response data handling)
  - kittens/transfer/receive.go:1071-1170 (receive_loop event loop)
  - tools/rsync/algorithm.go:30-38 (OpType enum)
  - tools/rsync/algorithm.go:72-92 (Operation struct)
  - tools/rsync/algorithm.go:177-200 (BlockHash struct and serialization)
  - tools/rsync/algorithm.go:336-360 (rolling_checksum)
  - tools/rsync/algorithm.go:362-569 (diff engine)
  - tools/rsync/api.go:47-68 (Api, Differ, Patcher structs)
  - tools/rsync/api.go:195-227 (CreateSignatureIterator)
  - tools/rsync/api.go:230-240 (CreateDelta)
  - tools/rsync/api.go:270-287 (NewPatcher with block size calculation)
  - kittens/transfer/utils.go:37-51 (encode_bypass)
  - kittens/transfer/utils.go:88-107 (should_be_compressed)
  - kittens/transfer/utils.go:109-114 (print_rsync_stats)
  - docs/file-transfer-protocol.rst:1-615 (protocol specification)
```

### 0.5.3 Documentation Configuration Updates

No documentation build configuration updates are required. The new file is a standalone markdown document placed in the `blitzy/documentation/` directory per the `SWE-AtlasQnA-Repo` implementation rule. It does not integrate into the Sphinx documentation build.

### 0.5.4 Cross-Documentation Dependencies

- The new document will reference `docs/file-transfer-protocol.rst` as the authoritative protocol specification
- The new document will reference `docs/kittens/transfer.rst` for user-facing usage context
- The new document will reference `docs/kittens/ssh.rst` for SSH connection setup context
- No navigation or table-of-contents updates are needed since the document lives outside the Sphinx tree

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to this documentation exercise:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| Go module | `kitty` (self) | go 1.22 | Main Go module for kitten binary containing transfer and rsync packages |
| Go module | `github.com/zeebo/xxh3` | v1.0.2 | XXH3 hash family used by rsync signature/delta for strong hashing and integrity checksums |
| Go module | `golang.org/x/exp` | v0.0.0-20230801115018 | Provides `constraints` package used in generic helper functions in send.go |
| Go module | `golang.org/x/sys` | v0.21.0 | Unix syscalls used by receiver for metadata application (UtimesNanoAt, Fchmodat) |
| Go module | `github.com/google/go-cmp` | v0.6.0 | Used in transfer and rsync test suites for structural comparison |
| pip | sphinx | (unversioned) | Documentation site generator for the existing Sphinx docs tree |
| pip | furo | (unversioned) | Material-style theme for Sphinx documentation |
| pip | sphinx-copybutton | (unversioned) | Copy button extension for code blocks |
| pip | sphinxext-opengraph | (unversioned) | OpenGraph metadata extension |
| pip | sphinx-inline-tabs | (unversioned) | Tabbed content extension |
| pip | sphinx-autobuild | (unversioned) | Live-preview development workflow |
| C library | xxhash | (vendored/3rdparty) | XXH3 hashing used in the C rsync extension (`kittens/transfer/algorithm.c`) |
| Python | Python ≥ 3.8 | ≥ 3.8 | Runtime for kitty's Python layer including `kitty/file_transmission.py` |
| system | zlib | system | RFC 1950 ZLIB compression used for file transfer data compression |

### 0.6.2 Build Dependencies for Investigation

For the documentation task (building kitty from source, running transfer experiments), the following build-time dependencies are relevant:

| Category | Requirement | Source |
|----------|-------------|--------|
| C Compiler | GCC or Clang with C11 support | `setup.py` — enforces `-std=c11` |
| Go Toolchain | Go 1.22+ | `go.mod` line 3: `go 1.22` |
| Python | Python ≥ 3.8 | `pyproject.toml` line 2: `requires-python = ">=3.8"` |
| OpenGL | Development headers for OpenGL/Mesa | Required for GPU rendering (not directly relevant to transfer) |
| pkg-config libraries | Various system libraries for font rendering, window management | Documented in `docs/build.rst` and `setup.py` |

### 0.6.3 Documentation Reference Updates

Not applicable — the new document is standalone and does not require updating any existing documentation links.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (for the user's questions):**

| User Question | Existing Coverage | Target Coverage | Gap |
|---------------|-------------------|-----------------|-----|
| Protocol handshake and escape sequences | `docs/file-transfer-protocol.rst` covers wire format (100%) | Implementation walkthrough with code citations | Implementation-level trace needed |
| Rsync delta transfer mechanism | Protocol spec covers signature/delta format (60%) | Full algorithm walkthrough with data structures | Implementation details, hash lookup, rolling checksum walkthrough |
| Data flow encoding/reassembly | Protocol spec covers OSC 5113 encoding (70%) | End-to-end byte flow with split_for_transfer and OnEscapeCode | Chunk splitting, multiplexing, demultiplexing |
| Transfer resumption behavior | No existing coverage (0%) | Analysis with code evidence | Entirely new analysis required |
| Delta efficiency demonstration | No existing coverage (0%) | Hands-on test methodology with expected results | Entirely new demonstration methodology |
| SSH connection setup for transfers | `docs/kittens/ssh.rst` covers user workflow (80%) | How SSH kitten makes transfer kitten available | Bootstrap/binary deployment details |
| Build from source | `docs/build.rst` exists (90%) | Summary with transfer-relevant focus | Minimal gap |

**Target coverage:** 100% of user questions answered with code-referenced evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- All seven user questions must receive thorough answers with thinking/rationale
- Every technical claim must cite specific source files and line numbers
- All data structures must have byte-layout documentation
- All state machines must have Mermaid diagrams
- The delta efficiency demonstration must include expected output format and interpretation guide

**Accuracy validation:**
- All code citations must reference actual file paths and line numbers verified during context gathering
- Protocol flow descriptions must match the serialization logic in `ftc.go` and the event loop handling in `send.go`/`receive.go`
- Rsync algorithm description must match the implementation in `tools/rsync/algorithm.go`, not just the protocol spec
- The resumption behavior conclusion must be supported by absence of persistent state in both the protocol spec and the implementation

**Clarity standards:**
- Technical accuracy with accessible language for a developer onboarding to the codebase
- Progressive disclosure: overview first, then detailed walkthrough per topic
- Consistent terminology matching codebase naming (e.g., "FileTransmissionCommand" not "transfer command object")
- Mermaid diagrams for all complex flows

**Maintainability:**
- Source citations use `path/to/file:LineNumber` format for traceability
- Document is self-contained — no external dependencies except codebase references
- Section structure allows selective reading for specific topics

### 0.7.3 Example and Diagram Requirements

- **Minimum code snippets:** At least one representative code excerpt per major topic (handshake, rsync, encoding, demultiplexing) — kept to 2-3 lines per snippet
- **Diagram types required:** Sequence diagram (protocol handshake), state diagram (sender/receiver), flowchart (rsync workflow, encoding pipeline), architecture diagram (system overview)
- **Worked examples:** At least one complete worked example showing an escape sequence being constructed from a `FileTransmissionCommand`
- **Table layouts:** Byte-level layout tables for `BlockHash` (20 bytes), signature header (12 bytes), and Operation types

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope (with trailing patterns)

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — Primary deliverable: comprehensive Q&A document tracing the file transfer protocol

**Source files analyzed (read-only, for citation only):**
- `kittens/transfer/ftc.go` — FileTransmissionCommand struct, serialization, parsing, escape encoding
- `kittens/transfer/send.go` — Send-side state machine, rsync signature handling, delta streaming, event loop
- `kittens/transfer/receive.go` — Receive-side state machine, signature generation, delta patching, file assembly
- `kittens/transfer/main.go` — Entry point and direction dispatch
- `kittens/transfer/utils.go` — Bypass encoding, compression heuristics, rsync stats
- `kittens/transfer/algorithm.c` — C rsync extension for Python-side xxh3 hashing
- `kittens/transfer/rsync.pyi` — Type stubs for Python rsync bindings
- `tools/rsync/algorithm.go` — Core rsync algorithm: rolling checksum, diff engine, delta operations, hash verification
- `tools/rsync/api.go` — Public API: Patcher, Differ, signature iteration, delta creation
- `kitty/file_transmission.py` — Python-side protocol handler: ActiveSend, ActiveReceive, FileTransmission
- `kittens/ssh/main.go` — SSH kitten entry point and connection bootstrap
- `docs/file-transfer-protocol.rst` — Authoritative protocol specification
- `docs/kittens/transfer.rst` — User-facing transfer kitten guide
- `docs/kittens/ssh.rst` — SSH kitten documentation
- `go.mod` — Go module definition and dependencies (xxh3 v1.0.2)
- `pyproject.toml` — Python project configuration (Python >=3.8)

**Documentation assets to produce within the deliverable:**
- Mermaid sequence diagrams (protocol handshake, delta transfer workflow)
- Mermaid state diagrams (sender file state, receiver state)
- Mermaid flowcharts (rsync algorithm, encoding pipeline)
- Byte-layout tables (signature header, BlockHash, Operation types)
- Escape sequence examples with worked encoding/decoding

**Temporary test artifacts (created and cleaned up during demonstration):**
- Temporary files for demonstrating delta transfer efficiency
- SSH connections for live protocol observation (if buildable)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — User explicitly states: "Do not modify any source files"
- **Test file modifications** — No existing test files will be altered
- **Feature additions or code refactoring** — Pure documentation task
- **Deployment configuration changes** — No infrastructure modifications
- **Documentation unrelated to file transfer protocol** — Other kittens (e.g., diff, icat, unicode_input) are not documented
- **Performance benchmarking** — The delta efficiency demonstration shows relative savings, not absolute performance metrics
- **Other transfer mechanisms** — Only the file transfer protocol over SSH is in scope; clipboard, pipe, or other I/O are excluded
- **Python-side implementation deep dive** — `kitty/file_transmission.py` is referenced for completeness but the focus is the Go-side implementation per user emphasis on the transfer kitten and rsync library
- **Third-party library internals** — The xxh3 hash library (`github.com/zeebo/xxh3`) is cited but its internal implementation is not traced
- **Windows/macOS platform-specific behavior** — Protocol analysis is platform-agnostic

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Output file:** `blitzy/documentation/kitty_815df1e210e0.md`
- **Output format:** Markdown with Mermaid diagrams, code snippets, and byte-layout tables
- **Documentation build command:** Not applicable — output is a standalone Markdown file, no documentation generator required
- **Documentation preview command:** Any Markdown renderer (e.g., grip, GitHub preview, VS Code Markdown preview)
- **Diagram generation:** Mermaid diagrams embedded inline using fenced mermaid code blocks; renderable by any Mermaid-compatible viewer
- **Citation requirement:** Every technical claim must reference a specific source file and line number using the format `Source: path/to/file.go:LineNumber`
- **Style guide:** Follow the repository's existing documentation style observed in `docs/file-transfer-protocol.rst` and `docs/kittens/transfer.rst` — clear prose with progressive disclosure, tables for structured data, code blocks for examples

### 0.9.2 Build and Test Workflow for Demonstration

The user requests building kitty from source and demonstrating delta transfer efficiency. The execution sequence is:

- **Build kitty:** Follow `docs/build.rst` — requires C compiler, Go 1.22, Python >=3.8, and system dependencies (libdbus, libgl, libx11, wayland, etc.)
- **Establish SSH connection:** Use `kitty +kitten ssh <host>` to bootstrap a transfer-capable session
- **First file transfer:** `kitty +kitten transfer <local_file> <remote_host>:<remote_path>`
- **Modify file locally:** Change a small portion of the test file (e.g., edit a few bytes in the middle)
- **Second file transfer with deltas:** `kitty +kitten transfer --transmit-deltas <local_file> <remote_host>:<remote_path>`
- **Observe rsync stats:** The transfer kitten outputs delta size vs. total size at completion via `print_rsync_stats()` in `kittens/transfer/utils.go`
- **Cleanup:** Remove all temporary test files from both local and remote

### 0.9.3 Documentation Validation

- **Link checking:** Not applicable — no internal or external hyperlinks requiring validation
- **Code citation accuracy:** All file paths and line numbers must be verified against the current state of the repository at branch `kitty_815df1e210e0`
- **Diagram correctness:** Mermaid diagrams must render without syntax errors; validate with mmdc (mermaid-cli) or any online Mermaid live editor
- **Content completeness:** Every user question must be answered with reasoning and code evidence — no question left partially addressed

## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are derived directly from the user's instructions and the project's implementation rules:

- **Do not modify any source files.** The generated documentation must be purely additive — no changes to any existing file in the kitty repository. Only the new markdown document and temporary test files are permitted.
- **Temporary test files are fine but clean them up afterwards.** Any files created for the delta efficiency demonstration must be removed after the demonstration is complete. Both local and remote sides must be cleaned.
- **Create a new markdown document named `kitty_815df1e210e0.md`.** The output file name matches the source branch name as required by the SWE-AtlasQnA-Repo rule.
- **Place the generated document in the `blitzy/documentation` directory in the destination repo.** The final path is `blitzy/documentation/kitty_815df1e210e0.md`.
- **Provide thinking / rationale behind the answers.** Every answer must include the reasoning process, not just the conclusion. Explain *why* the protocol works the way it does, not just *what* it does.
- **Do not make assumptions, base your answers on the code as the truth.** All claims must be substantiated by specific source file references. Where the code differs from the protocol spec or general rsync documentation, the code governs.

### 0.10.2 Documentation Style Rules

- **Evidence-first writing:** Lead with code citations, then explain. Format: "In `path/to/file.go:LineNumber`, the implementation does X because Y."
- **No speculative content:** If a behavior cannot be confirmed from the source code, explicitly state that it is not implemented rather than inferring or guessing.
- **Progressive disclosure:** Start each topic with a high-level summary, then drill into implementation details. A reader should be able to stop at any depth and have a correct (if incomplete) understanding.
- **Consistent terminology:** Use the exact names from the codebase: `FileTransmissionCommand` (not "transfer command"), `BlockHash` (not "block signature"), `OpBlock`/`OpData`/`OpHash`/`OpBlockRange` (not generic "operation types"), `rolling_checksum` (not "weak hash function").
- **Diagram conventions:** All Mermaid diagrams use consistent color scheme and follow left-to-right or top-to-bottom flow. State diagrams use descriptive transition labels matching enum names from the source code.
- **Code snippet brevity:** Code excerpts limited to 2-3 lines showing the key logic. Full function bodies are not reproduced — instead, cite the file and line range.

### 0.10.3 Content Integrity Rules

- **No external source dependency:** The document must be self-contained and comprehensible without needing to open any other file. All necessary context is included inline.
- **Bidirectional traceability:** Every section heading maps to a specific user question, and every user question maps to at least one section. A traceability note at the beginning of the document maps questions to sections.
- **Version pinning:** All file references are pinned to the `kitty_815df1e210e0` branch state. Line numbers reflect the repository as analyzed during this session.
- **Honest gap reporting:** Where the codebase does not implement a feature (e.g., transfer resumption), the document explicitly states this with evidence of absence (specific files searched, patterns not found).

## 0.11 References

### 0.11.1 Source Code Files Examined

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `kittens/transfer/ftc.go` | FileTransmissionCommand definition, serialization, OSC 5113 encoding | Struct with 16+ fields, reflection-based Serialize(), NewFileTransmissionCommand() parser, split_for_transfer() at 4096 bytes |
| `kittens/transfer/send.go` | Send-side state machine and event loop | FileState (5 states), SendState (4 states), rsync signature reception, delta streaming, prefix/suffix construction for OSC 5113 |
| `kittens/transfer/receive.go` | Receive-side state machine and file assembly | patch_file struct wrapping rsync Patcher, signature generation via CreateSignatureIterator, delta patching, temp-file-then-rename strategy |
| `kittens/transfer/main.go` | Transfer kitten entry point | Direction dispatch to send_main() or receive_main() |
| `kittens/transfer/utils.go` | Transfer utilities | Encrypted bypass encoding (X25519 + AES-GCM), compression heuristics by extension/MIME, rsync stats printing |
| `kittens/transfer/algorithm.c` | C rsync extension for Python | xxh64/xxh128 hasher implementations using xxhash library |
| `kittens/transfer/rsync.pyi` | Python type stubs | Exposes Hasher, Patcher, Differ, parse_ftc to Python |
| `tools/rsync/algorithm.go` | Core rsync algorithm | rolling_checksum, diff engine with sliding window, BlockHash (20 bytes), Operation types (Block/Data/Hash/BlockRange), ApplyDelta with XXH3-128 verification |
| `tools/rsync/api.go` | Rsync public API | Patcher.CreateSignatureIterator (12-byte header + BlockHash entries), Differ.AddSignatureData, Differ.CreateDelta, block_size = sqrt(file_size) capped at 1MB |
| `kitty/file_transmission.py` | Python-side protocol handler | ActiveSend, ActiveReceive, FileTransmission, PatchFile, DestFile, SourceFile classes |
| `kittens/ssh/main.go` | SSH kitten entry point | Connection bootstrapping, kitten binary availability on remote |
| `go.mod` | Go module definition | Module: kitty, Go 1.22, xxh3 v1.0.2 dependency |
| `pyproject.toml` | Python project config | Python >=3.8, strict mypy enforcement |

### 0.11.2 Documentation Files Examined

| File Path | Purpose | Key Findings |
|-----------|---------|--------------|
| `docs/file-transfer-protocol.rst` | Authoritative protocol specification (615 lines) | OSC 5113 wire format, key abbreviation table, value types, session model, rsync signature/delta binary formats, compression, bypass authentication, quiet modes |
| `docs/kittens/transfer.rst` | User-facing transfer kitten guide (88 lines) | Basic usage, bypass passwords, delta transfers with --transmit-deltas flag |
| `docs/kittens/ssh.rst` | SSH kitten documentation | Automatic shell integration, connection reuse, kitten binary availability on remote |

### 0.11.3 Folders Explored

| Folder Path | Purpose |
|-------------|---------|
| (root) | Repository root — revealed kittens/, tools/, docs/, kitty/, kitty_tests/, 3rdparty/, glfw/, gen/, shell-integration/ |
| `kittens/` | 20 kitten subpackages including transfer/ and ssh/ |
| `kittens/transfer/` | 12 files: Go sources, C extension, Python stubs, tests |
| `kittens/ssh/` | 10 files: Go sources, Python files, tests |
| `tools/` | Go support packages: rsync/, crypto/, tui/, cli/, config/ |
| `tools/rsync/` | 3 files: algorithm.go, api.go, api_test.go |
| `docs/` | Sphinx documentation root |
| `docs/kittens/` | 14 kitten documentation pages |

### 0.11.4 Tech Spec Sections Retrieved

| Section | Key Information Extracted |
|---------|-------------------------|
| 1.1 Executive Summary | Kitty is a GPU-accelerated terminal emulator with 3-language architecture (C/Python/Go) |
| 3.1 Programming Languages | Detailed language architecture: C for core engine, Python for extensibility/config, Go for CLI tooling/kittens |
| 4.9 Protocol Workflows | File Transfer Protocol flow with state diagram, features table covering compression, delta transfer, confirmation, backpressure, retry, wire format |

### 0.11.5 Attachments and External Resources

- **Attachments provided by user:** None
- **Figma screens provided:** None
- **External URLs referenced:** None
- **Branch analyzed:** `kitty_815df1e210e0`
- **Repository type:** Local clone of the kitty terminal emulator project (kovidgoyal/kitty)

