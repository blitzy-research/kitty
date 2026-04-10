# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, comprehensive technical Q&A document** that traces the end-to-end behavior of the Kitty terminal emulator's SSH kitten subsystem. The user is onboarding into the Kitty repository and needs a code-grounded walkthrough that explains how the SSH kitten orchestrates a secure remote session — from initial invocation through bootstrap script execution on the remote host.

- **Request Category:** Create new documentation
- **Documentation Type:** Technical deep-dive / Q&A document — placed in `blitzy/documentation/kitty_815df1e210e0.md`
- **Primary Questions to Answer:**
  - How does the SSH kitten set up a secure session and share connections?
  - How does shared memory pass credentials securely between processes?
  - How are bootstrap scripts generated and what encoding tricks do they use for different shells?
  - How is the archive (tarball) with shell integration files built and sent over the SSH TTY?
  - How does the kitten track everything it needs for a connection (the `connection_data` struct)?
  - How does connection reuse work — what determines a fresh connection vs. piggybacking on an existing ControlMaster?
  - What is the end-to-end flow from `kitten ssh hostname` to the bootstrap executing on the remote machine?
  - How does the terminal (kitty process) communicate back and forth with the remote shell during setup?

### 0.1.2 Special Instructions and Constraints

- **Repository Immutability:** The user explicitly stated: "the repository itself should remain unchanged, and anything temporary should be cleaned up afterward." No source files in the repository may be modified.
- **Implementation Rule — SWE-AtlasQnA-Repo:** The implementation rules mandate:
  - Create a new markdown document named `kitty_815df1e210e0.md` (the source branch name)
  - Provide thinking and rationale behind the answers
  - Base all answers on the code as the single source of truth — no assumptions
  - Do not modify any existing files in the source repository
  - Place the generated document in the `blitzy/documentation` directory
- **Temporary Scripts:** Temporary scripts for observation are permissible during analysis, but must be cleaned up before completion.
- **Documentation Tone:** Technical deep-dive aimed at a developer who is onboarding and wants to "piece together how the whole thing actually works end-to-end." The document should be explanatory yet precise, grounded in specific function names, file paths, and line-level code citations.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **explain secure session setup and shared memory**, we will create a section in `blitzy/documentation/kitty_815df1e210e0.md` that traces through `kittens/ssh/main.go:bootstrap_script()`, the `shm.CreateTemp()` call, the JSON payload written to shared memory, and the password/request-ID verification handshake in `kittens/ssh/utils.py:get_ssh_data()` and `kitty/window.py:handle_remote_ssh()`.
- To **explain connection reuse**, we will document the `connection_sharing_args()` function in `kittens/ssh/main.go`, the ControlMaster/ControlPath/ControlPersist SSH options, the `master_is_functional()` check using `ssh -O check`, and how `run_control_master()` starts a background ControlMaster when needed.
- To **explain bootstrap script generation**, we will trace `get_remote_command()` → `bootstrap_script()` → `wrap_bootstrap_script()` in `kittens/ssh/main.go`, explaining the POSIX shell (`bootstrap.sh`) and Python (`bootstrap.py`) paths, the character substitution encoding (`' → \v`, `\ → \f`, `\n → \r`, `! → \b`), and the base64 encoding for Python interpreters.
- To **explain the archive (tarball) construction**, we will document `make_tarfile()` in `kittens/ssh/main.go`, which builds a gzip-compressed tar containing `data.sh` (serialized environment), `bootstrap-utils.sh`, shell integration scripts, terminfo entries, and optionally the kitten binary.
- To **explain the end-to-end TTY data exchange**, we will document how the bootstrap script sends a DCS escape (`\033P@kitty-ssh|...`) to the TTY, how `kitty/window.py:handle_remote_ssh()` processes this request, validates the password against shared memory via `kittens/ssh/utils.py:read_data_from_shared_memory()`, and streams the base64-encoded tarball back line-by-line over the TTY to the remote bootstrap script.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, additional implicit documentation needs include:

- **Askpass Mechanism:** The SSH kitten's `askpass.go` implements a novel askpass flow using shared memory IPC (`shm.CreateTemp` for askpass prompts, polling on byte 0 for completion, `kitty/window.py:handle_remote_askpass()` for the Kitty-side UI). This is integral to the secure session story and should be documented.
- **Configuration Loading:** The `ssh.conf` host-matching system in `config.go:load_config()` and `config_for_hostname()` determines per-host behavior (interpreter, shell integration, connection sharing, etc.) and directly influences the bootstrap flow.
- **Environment Serialization:** The `serialize_env()` function's dual-mode output (POSIX shell export statements vs. Python JSON arrays) is a key detail for understanding how the remote environment is reconstructed.
- **Login Shell Discovery:** The multi-strategy login shell detection in `bootstrap-utils.sh` (getent → id -P → python → perl → /etc/passwd → $SHELL) is a non-obvious aspect of the bootstrap's portability.
- **Cleanup and Error Handling:** The SHM unlink-on-read pattern, `cleanup_on_bootstrap_exit` trap in `bootstrap.sh`, and the `drain_potential_tty_garbage()` post-session cleanup in `main.go` are important for understanding the security and robustness story.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx-based documentation system with comprehensive coverage of user-facing features, but the SSH kitten's internal implementation is documented only at a user-guide level — there is no existing technical deep-dive tracing the end-to-end code paths.

- **Documentation Framework:** Sphinx, using the `furo` theme
- **Documentation Generator Configuration:** `docs/conf.py` — Sphinx configuration with custom lexers, roles, and auto-generated configuration reference includes
- **Documentation Build Driver:** `docs/Makefile` — standard Sphinx build with `develop-docs` live-preview via `sphinx-autobuild`
- **Documentation Dependencies:** `docs/requirements.txt` — `sphinx`, `furo`, `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
- **API Documentation Tools:** No dedicated API doc generators (no JSDoc, Godoc, or similar); documentation is hand-authored reStructuredText with auto-generated config/CLI reference blocks via `docs/conf.py` generator hooks
- **Diagram Tools:** Mermaid diagrams are used in the technical specification; the existing Sphinx docs do not include Mermaid but use inline code blocks and textual diagrams
- **Documentation Hosting:** Published via the project website; `publish.py` handles docs publishing and `docs/Makefile` has a `website` target

**Existing SSH Kitten Documentation Found:**

| File | Type | Content Summary |
|------|------|-----------------|
| `docs/kittens/ssh.rst` | User guide (reStructuredText) | User-facing SSH kitten documentation: feature overview, `ssh.conf` config format, real-world examples, "How it works" summary, copy command reference, manual terminfo copy instructions |
| `kittens/ssh/main.py` | Config schema | Defines the `ssh.conf` configuration options with long-text descriptions (hostname, interpreter, remote_dir, copy, shell_integration, login_shell, env, cwd, color_scheme, remote_kitty, share_connections, askpass, delegate, forward_remote_control) |
| `docs/shell-integration.rst` | User guide | Documents shell integration features broadly; references the SSH kitten's role in remote deployment |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to locate code relevant to the SSH kitten's end-to-end flow:

- **SSH Kitten Core:** `kittens/ssh/*.go`, `kittens/ssh/*.py` — 10 files comprising the SSH kitten's Go runtime, Python config layer, and tests
- **Bootstrap Scripts:** `shell-integration/ssh/bootstrap.sh`, `shell-integration/ssh/bootstrap.py`, `shell-integration/ssh/bootstrap-utils.sh` — the remote-side bootstrap system
- **Shared Memory (Go):** `tools/utils/shm/shm.go`, `tools/utils/shm/shm_syscall.go`, `tools/utils/shm/shm_fs.go` — POSIX shared memory MMap abstraction
- **Shared Memory (Python):** `kitty/shm.py` — Python SharedMemory class wrapping `shm_open`/`shm_unlink`
- **Kitty-side SSH Handler:** `kitty/window.py:handle_remote_ssh()` (line 1289), `kitty/window.py:handle_remote_askpass()` (line 1351) — the Kitty terminal's response to DCS escape sequences from the SSH kitten
- **SSH Data Provider:** `kittens/ssh/utils.py:get_ssh_data()` (line 115) — reads the SHM payload, validates password, and streams the base64-encoded tarball
- **Secrets/Tokens:** `tools/utils/secrets/tokens.go` — cryptographic random token generation for passwords and request IDs
- **Connection Cleanup:** `kitty/utils.py:cleanup_ssh_control_masters()` (line 1038), `kitty/constants.py:ssh_control_master_template` (line 188)
- **Tests:** `kittens/ssh/main_test.go`, `kittens/ssh/config_test.go`, `kittens/ssh/utils_test.go`, `kitty_tests/ssh.py`, `kitty_tests/shm.py`

**Key Directories Examined:**

| Directory | Files Examined | Relevance |
|-----------|---------------|-----------|
| `kittens/ssh/` | `main.go`, `askpass.go`, `config.go`, `utils.go`, `main.py`, `utils.py`, `main_test.go`, `config_test.go`, `utils_test.go` | Core SSH kitten implementation |
| `shell-integration/ssh/` | `bootstrap.sh`, `bootstrap.py`, `bootstrap-utils.sh` | Remote-side bootstrap scripts |
| `tools/utils/shm/` | `shm.go`, `shm_syscall.go`, `shm_fs.go`, `shm_test.go` | POSIX shared memory abstraction |
| `tools/utils/secrets/` | `tokens.go` | Cryptographic token generation |
| `kitty/` | `shm.py`, `window.py`, `utils.py`, `constants.py`, `boss.py` | Kitty-side SSH/askpass handlers |
| `docs/kittens/` | `ssh.rst` | Existing user-facing documentation |
| `docs/` | `conf.py`, `requirements.txt`, `Makefile` | Documentation build infrastructure |

### 0.2.3 Web Search Research Conducted

No web search is needed for this task. The user explicitly requested code-grounded analysis ("base your answers on the code as the truth"), and the repository provides all necessary source material. The SSH kitten implementation is entirely self-contained within the codebase, and the documentation task is to trace and explain existing code paths — not to research external best practices.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage in the new `kitty_815df1e210e0.md` document. Each module's public APIs and key functions are listed alongside their documentation status.

**Module: `kittens/ssh/main.go` — SSH Kitten Orchestrator**
- Key Functions: `run_ssh()`, `get_remote_command()`, `bootstrap_script()`, `wrap_bootstrap_script()`, `make_tarfile()`, `connection_sharing_args()`, `set_askpass()`, `serialize_env()`, `prepare_home_command()`, `prepare_exec_cmd()`, `read_data_from_shared_memory()`, `add_cloned_env()`, `parse_kitten_args()`, `drain_potential_tty_garbage()`, `change_colors()`
- Key Types: `connection_data` struct (lines 171–189)
- Current Documentation: User-guide level only in `docs/kittens/ssh.rst`; no internal technical walkthrough
- Documentation Needed: End-to-end flow narrative, function-by-function tracing, data flow diagrams

**Module: `kittens/ssh/askpass.go` — Askpass Shared Memory Handshake**
- Key Functions: `RunSSHAskpass()`, `trigger_ask()`
- Current Documentation: Briefly mentioned in `docs/kittens/ssh.rst` (askpass option)
- Documentation Needed: Shared memory polling protocol, DCS escape trigger, Kitty-side handler interaction

**Module: `kittens/ssh/config.go` — Configuration and Tar Packaging**
- Key Types: `EnvInstruction`, `CopyInstruction`, `ConfigSet`
- Key Functions: `load_config()`, `config_for_hostname()`, `ParseEnvInstruction()`, `ParseCopyInstruction()`, `get_file_data()`, `final_env_instructions()`, `quote_for_sh()`
- Current Documentation: Config options documented in `kittens/ssh/main.py` definitions
- Documentation Needed: Config loading flow, hostname matching logic, tar entry creation

**Module: `kittens/ssh/utils.go` — SSH Binary Probing and Argument Parsing**
- Key Functions: `SSHExe()`, `SSHOptions()`, `GetSSHCLI()`, `ParseSSHArgs()`, `GetSSHVersion()`, `RelevantKittyOpts()`
- Key Types: `SSHVersion`, `KittyOpts`, `ErrInvalidSSHArgs`
- Current Documentation: None at technical level
- Documentation Needed: SSH option discovery, argument partitioning, version detection

**Module: `kittens/ssh/utils.py` — Python-side SSH Helpers**
- Key Functions: `get_ssh_data()`, `read_data_from_shared_memory()`, `create_shared_memory()`, `get_connection_data()`, `patch_cmdline()`
- Current Documentation: None
- Documentation Needed: Data request/response protocol, SHM security validation

**Module: `shell-integration/ssh/bootstrap.sh` — POSIX Shell Bootstrap**
- Key Functions: `get_data()`, `untar_and_read_env()`, `read_base64_from_tty()`, `base64_encode()`/`base64_decode()`, `dcs_to_kitty()`, `cleanup_on_bootstrap_exit()`
- Current Documentation: Conceptual mention in `docs/kittens/ssh.rst` "How it works" section
- Documentation Needed: Step-by-step bootstrap flow, base64 detection strategy, TTY protocol

**Module: `shell-integration/ssh/bootstrap.py` — Python Bootstrap**
- Key Functions: `main()`, `get_data()`, `iter_base64_data()`, `apply_env_vars()`, `compile_terminfo()`, `exec_with_shell_integration()`, `install_kitty_bootstrap()`
- Current Documentation: None at technical level
- Documentation Needed: Python bootstrap alternative, env var application, shell-specific integration dispatch

**Module: `shell-integration/ssh/bootstrap-utils.sh` — Shared Bootstrap Utilities**
- Key Functions: `mv_files_and_dirs()`, `compile_terminfo()`, `prepare_for_exec()`, `exec_login_shell()`, login shell discovery chain (`using_getent`, `using_id`, `using_python`, `using_perl`, `using_passwd`, `using_shell_env`)
- Current Documentation: None
- Documentation Needed: File staging, terminfo compilation, login shell discovery chain

**Module: `tools/utils/shm/shm.go` — Shared Memory Abstraction**
- Key Interface: `MMap` (methods: `Close`, `Unlink`, `Slice`, `Name`, `Flush`, `Read`, `Write`, `Stat`)
- Key Functions: `CreateTemp()`, `WriteWithSize()`, `ReadWithSize()`, `ReadWithSizeAndUnlink()`
- Current Documentation: None
- Documentation Needed: SHM lifecycle, size-prefixed read/write protocol, security validation callback

**Module: `kitty/shm.py` — Python SharedMemory**
- Key Class: `SharedMemory` — `shm_open`/`shm_unlink` wrapper with `write_data_with_size()` / `read_data_with_size()`
- Current Documentation: None
- Documentation Needed: Python SHM creation, size-prefix protocol, permission mode (0o600)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No end-to-end technical trace exists** for the SSH kitten flow — the existing `docs/kittens/ssh.rst` provides a 12-line summary in the "How it works" section but does not trace code paths or cite specific functions
- **Shared memory security model is undocumented** — the UID/GID ownership check, 0o600 permission enforcement, and unlink-on-read pattern are present in code but not explained anywhere
- **Bootstrap script encoding is undocumented** — the character substitution scheme (`' → \v`, `\ → \f`, `\n → \r`, `! → \b`) in `wrap_bootstrap_script()` and the rationale for tcsh compatibility are not documented
- **Connection reuse decision logic is undocumented** — the interplay between `share_connections` config, `master_is_functional()` check, `need_to_request_data` flag, and the askpass version gate is not explained
- **Askpass shared memory IPC protocol is undocumented** — the polling loop, byte-0 completion signal, and DCS `@kitty-ask` escape are implementation details not covered in any existing docs
- **The `connection_data` struct is undocumented** — this central data structure that accumulates all connection state during the SSH flow has no documentation
- **Tarball structure is undocumented** — what goes into the archive (`data.sh`, `bootstrap-utils.sh`, shell integration files, terminfo, kitty binaries) and why is not documented

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive Markdown document placed at `blitzy/documentation/kitty_815df1e210e0.md`. The document structure is designed to answer each of the user's questions in a logical, progressive order — starting from the high-level flow and drilling into each subsystem.

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Introduction and Scope
        ├── End-to-End SSH Session Flow
        │   ├── Phase 1: Invocation and Argument Parsing
        │   ├── Phase 2: Configuration Loading and Host Matching
        │   ├── Phase 3: Connection Sharing (ControlMaster) Decision
        │   ├── Phase 4: Askpass Setup
        │   ├── Phase 5: Bootstrap Script Generation
        │   ├── Phase 6: Tarball Construction
        │   ├── Phase 7: Shared Memory Credential Storage
        │   ├── Phase 8: SSH Subprocess Launch and TTY Handshake
        │   └── Phase 9: Cleanup and TTY Garbage Drain
        ├── Shared Memory Security Model
        │   ├── SHM Creation and Permission Enforcement
        │   ├── Data Layout and Size-Prefixed Protocol
        │   ├── Ownership and Permission Validation on Read
        │   └── Unlink-on-Read Pattern
        ├── Bootstrap Script Deep Dive
        │   ├── POSIX Shell Bootstrap (bootstrap.sh)
        │   ├── Python Bootstrap (bootstrap.py)
        │   ├── Bootstrap Utilities (bootstrap-utils.sh)
        │   ├── Script Encoding and Wrapping
        │   └── Base64 Detection and Fallback Chain
        ├── Connection Reuse Logic
        │   ├── ControlMaster Configuration
        │   ├── Master Liveness Check
        │   ├── Decision Matrix
        │   └── Cleanup on Kitty Exit
        ├── Askpass Mechanism
        │   ├── SHM-Based IPC Protocol
        │   ├── DCS Escape Trigger
        │   └── Kitty-Side Handler
        ├── Tarball Archive Structure
        │   ├── Environment Script (data.sh)
        │   ├── Shell Integration Files
        │   ├── Terminfo Entries
        │   └── Remote Kitty Binaries
        ├── TTY Data Exchange Protocol
        │   ├── DCS Request from Bootstrap
        │   ├── Kitty-Side Validation and Response
        │   └── Base64 Streaming and Framing
        └── Key Data Structures
            ├── connection_data struct
            ├── Config / ConfigSet
            └── EnvInstruction / CopyInstruction
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract function signatures and control flow from `kittens/ssh/main.go` (primary orchestrator)
- Extract the bootstrap protocol from `shell-integration/ssh/bootstrap.sh` and `shell-integration/ssh/bootstrap.py`
- Extract the SHM security model from `tools/utils/shm/shm.go` and `kitty/shm.py`
- Generate the decision matrix for connection reuse by analyzing the conditional logic in `run_ssh()` (lines 597–798 of `kittens/ssh/main.go`)
- Create diagrams by mapping the interactions between `kittens/ssh/main.go`, `kitty/window.py`, `kittens/ssh/utils.py`, and `shell-integration/ssh/bootstrap.sh`

**Documentation Standards:**
- Markdown with Mermaid diagrams for flow visualizations
- Source citations as inline references: `Source: kittens/ssh/main.go:422` style
- Code examples kept short (2–3 lines) to illustrate key patterns
- Tables for structured data (config options, tarball contents, decision matrices)
- Thinking/rationale blocks explaining *why* each design choice was made, as required by the SWE-AtlasQnA-Repo rule

### 0.4.3 Diagram and Visual Strategy

The document will include the following Mermaid diagrams:

- **End-to-End Sequence Diagram:** Traces the full SSH session from `kitten ssh host` through argument parsing, config loading, ControlMaster check, bootstrap generation, SHM write, SSH subprocess launch, TTY DCS request, Kitty-side validation, tarball streaming, remote extraction, and login shell exec
- **Shared Memory Lifecycle Flowchart:** Shows SHM creation (with 0o600 permissions), JSON payload write, SHM name embedded in bootstrap script, remote-side DCS request, Kitty-side read + validate + unlink
- **Connection Reuse Decision Flowchart:** Visualizes the `share_connections` → `master_is_functional()` → `need_to_request_data` → `run_control_master()` decision tree
- **Bootstrap Encoding Flowchart:** Shows the two paths (Python: base64 encode → `eval(compile(base64.standard_b64decode(...)))` vs. Shell: character substitution → `tr` unwrap)
- **Tarball Structure Diagram:** Tree diagram of the archive contents under `home/`, `root/`, `data.sh`, `bootstrap-utils.sh`

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kittens/ssh/main.go`, `kittens/ssh/askpass.go`, `kittens/ssh/config.go`, `kittens/ssh/utils.go`, `kittens/ssh/utils.py`, `shell-integration/ssh/bootstrap.sh`, `shell-integration/ssh/bootstrap.py`, `shell-integration/ssh/bootstrap-utils.sh`, `tools/utils/shm/shm.go`, `kitty/shm.py`, `kitty/window.py`, `kitty/utils.py`, `kitty/constants.py` | Comprehensive Q&A document tracing the SSH kitten end-to-end flow: secure session setup, shared memory credential passing, bootstrap script generation and encoding, tarball construction, connection reuse logic, TTY data exchange protocol, askpass mechanism, and key data structures. Includes Mermaid diagrams, code citations, and rationale. |
| `docs/kittens/ssh.rst` | REFERENCE | N/A | Used as a reference for existing user-facing documentation style and content — not modified |
| `kittens/ssh/main.py` | REFERENCE | N/A | Used as a reference for configuration option definitions and descriptions — not modified |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Q&A / Deep-Dive Document
Source Code:
    - kittens/ssh/main.go (primary orchestrator, ~900 lines)
    - kittens/ssh/askpass.go (askpass SHM IPC, ~109 lines)
    - kittens/ssh/config.go (config parsing, tar packaging, ~411 lines)
    - kittens/ssh/utils.go (SSH binary probing, arg parsing, ~247 lines)
    - kittens/ssh/utils.py (Python SSH helpers, get_ssh_data, ~339 lines)
    - shell-integration/ssh/bootstrap.sh (POSIX shell bootstrap, ~164 lines)
    - shell-integration/ssh/bootstrap.py (Python bootstrap, ~318 lines)
    - shell-integration/ssh/bootstrap-utils.sh (shared utilities, ~251 lines)
    - tools/utils/shm/shm.go (SHM abstraction, ~253 lines)
    - tools/utils/shm/shm_syscall.go (syscall-based SHM impl, ~193 lines)
    - kitty/shm.py (Python SharedMemory, ~187 lines)
    - kitty/window.py:handle_remote_ssh (lines 1289-1292)
    - kitty/window.py:handle_remote_askpass (lines 1351-1379)
    - kitty/utils.py:cleanup_ssh_control_masters (lines 1038-1053)
    - kitty/constants.py:ssh_control_master_template (line 188)
    - tools/utils/secrets/tokens.go (token generation, ~43 lines)
Sections:
    - Introduction and Scope
    - End-to-End SSH Session Flow (9 phases)
    - Shared Memory Security Model (4 subsections)
    - Bootstrap Script Deep Dive (5 subsections)
    - Connection Reuse Logic (4 subsections)
    - Askpass Mechanism (3 subsections)
    - Tarball Archive Structure (4 subsections)
    - TTY Data Exchange Protocol (3 subsections)
    - Key Data Structures (3 subsections)
Diagrams:
    - End-to-end sequence diagram (SSH session lifecycle)
    - Shared memory lifecycle flowchart
    - Connection reuse decision flowchart
    - Bootstrap encoding flowchart (Python vs. Shell paths)
    - Tarball structure tree diagram
Key Citations:
    - kittens/ssh/main.go:run_ssh() — top-level SSH execution flow
    - kittens/ssh/main.go:bootstrap_script() — SHM creation, tarball, password gen
    - kittens/ssh/main.go:wrap_bootstrap_script() — encoding/wrapping logic
    - kittens/ssh/main.go:connection_sharing_args() — ControlMaster args
    - kittens/ssh/main.go:make_tarfile() — archive construction
    - kittens/ssh/main.go:set_askpass() — askpass environment setup
    - kittens/ssh/askpass.go:RunSSHAskpass() — askpass SHM handshake
    - kittens/ssh/utils.py:get_ssh_data() — Kitty-side data provider
    - shell-integration/ssh/bootstrap.sh:get_data() — remote TTY read loop
    - kitty/window.py:handle_remote_ssh() — DCS handler
    - kitty/window.py:handle_remote_askpass() — askpass DCS handler
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files require updates. The new document is a standalone Markdown file placed in `blitzy/documentation/` as specified by the implementation rules. It is not part of the Sphinx documentation tree and does not require changes to `docs/conf.py`, `docs/Makefile`, or any navigation configuration.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes** are needed — the document is self-contained
- **No navigation links** to other documents are required
- **No table of contents updates** are required
- **Internal cross-references** within the document will use Markdown heading anchors for section linking

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task produces a standalone Markdown file and does not require building or running the Kitty project. No documentation generation tools, build systems, or package installations are needed. The document is authored directly from source code analysis.

For reference, the project's documentation tooling (used by the Sphinx-based `docs/` tree, not by this task) is:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip | sphinx | (unpinned in `docs/requirements.txt`) | Documentation site generator for the `docs/` tree |
| pip | furo | (unpinned) | Sphinx theme |
| pip | sphinx-copybutton | (unpinned) | Code block copy buttons |
| pip | sphinxext-opengraph | (unpinned) | OpenGraph metadata for social sharing |
| pip | sphinx-inline-tabs | (unpinned) | Inline tab containers |
| pip | sphinx-autobuild | (unpinned) | Live-reload documentation preview |

### 0.6.2 Project Runtime Dependencies Relevant to SSH Kitten Analysis

The SSH kitten implementation spans two languages with the following relevant dependency context:

| Ecosystem | Dependency | Version | Relevance to SSH Kitten |
|-----------|-----------|---------|-------------------------|
| Go | `golang.org/x/sys/unix` | v0.21.0 | POSIX SHM syscalls (`shm_open`, `shm_unlink`), terminal control (`TCSANOW`), file operations |
| Go | `github.com/bmatcuk/doublestar/v4` | v4.6.1 | Glob pattern matching for `copy --glob` file specs in `config.go` |
| Go | `github.com/google/go-cmp` | v0.6.0 | Test comparisons in `main_test.go` |
| Python | `kitty.fast_data_types` | (built-in C extension) | `shm_open`, `shm_unlink`, `SHM_NAME_MAX` constants |
| Python | `kitty.shm` | (built-in) | `SharedMemory` class for askpass and clone-env SHM operations |
| Python | `kitty.utils` | (built-in) | `SSHConnectionData`, `cleanup_ssh_control_masters()` |

### 0.6.3 Documentation Reference Updates

No documentation reference updates are required. The new document at `blitzy/documentation/kitty_815df1e210e0.md` is self-contained and does not modify or link to any existing documentation files.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (SSH kitten internal documentation):**

| Coverage Area | Documented | Total | Percentage |
|---------------|-----------|-------|------------|
| End-to-end flow phases | 0 | 9 | 0% |
| Key Go functions in `kittens/ssh/` | 0 | 22 | 0% |
| Key Python functions in SSH utils | 0 | 6 | 0% |
| Bootstrap script functions | 0 | 15 | 0% |
| Shared memory lifecycle stages | 0 | 5 | 0% |
| Connection reuse decision paths | 0 | 4 | 0% |
| Key data structures | 0 | 5 | 0% |
| Mermaid diagrams (flow visualizations) | 0 | 5 | 0% |

**Target coverage after document creation: 100%** — Every question posed by the user and every implicit technical gap identified in Section 0.3 must be addressed in the final document.

**Coverage gaps to address:**
- `kittens/ssh/main.go`: Currently 0% internal documentation — target 100% of key functions traced and explained
- `kittens/ssh/askpass.go`: Currently 0% — target 100% of the SHM IPC protocol documented
- `shell-integration/ssh/`: Currently 0% at technical level — target 100% of bootstrap phases explained
- `tools/utils/shm/`: Currently 0% — target 100% of security model documented
- `kitty/window.py` SSH handlers: Currently 0% — target 100% of DCS handler logic explained

### 0.7.2 Documentation Quality Criteria

**Completeness Requirements:**
- Every user question from the prompt must have a dedicated, clearly headed section with a complete answer
- All code paths in the SSH session lifecycle must be traced with specific function names and file:line citations
- The shared memory security model must include permission values (0o600), ownership checks (UID/GID), and the unlink-on-read pattern
- The bootstrap encoding must explain both the Python (base64) and POSIX shell (character substitution) paths with the exact substitution table
- The connection reuse section must include the decision matrix showing all possible branches

**Accuracy Validation:**
- All function signatures cited must match the current codebase exactly
- All file paths must be verified against the repository structure
- Code examples must be extracted from actual source — no fabricated snippets
- The encoding substitution table must match the `strings.NewReplacer` call in `wrap_bootstrap_script()`

**Clarity Standards:**
- Technical accuracy with accessible language for a developer onboarding into the codebase
- Progressive disclosure: start with the high-level end-to-end flow, then drill into each subsystem
- Each section opens with a "Rationale" or "Thinking" paragraph explaining *why* the design choice was made (per SWE-AtlasQnA-Repo rules)
- Consistent terminology throughout: "bootstrap script" (not "startup script"), "shared memory" / "SHM" (not "IPC memory"), "ControlMaster" (not "connection multiplexer")

**Maintainability:**
- Source citations for every technical claim, formatted as `Source: path/to/file.go:line`
- Mermaid diagrams for visual flow understanding
- Self-contained document with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum code citations per major section:** 3–5 specific file:line references
- **Diagram types required:** Sequence diagram (end-to-end flow), flowcharts (decision logic, SHM lifecycle, bootstrap encoding), tree diagram (tarball structure)
- **Code example testing:** All examples are direct quotations from source files — verified against the repository
- **Visual content freshness:** Diagrams reflect the current codebase as analyzed during this task

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New Documentation Files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — the single deliverable document

**Source Code Analyzed for Documentation (read-only):**
- `kittens/ssh/**/*.go` — SSH kitten Go source files (main.go, askpass.go, config.go, utils.go)
- `kittens/ssh/**/*.py` — SSH kitten Python source files (main.py, utils.py)
- `kittens/ssh/**/*_test.go` — SSH kitten test files (for behavioral verification)
- `shell-integration/ssh/**` — Remote bootstrap scripts (bootstrap.sh, bootstrap.py, bootstrap-utils.sh)
- `tools/utils/shm/**/*.go` — Shared memory abstraction (shm.go, shm_syscall.go, shm_fs.go)
- `tools/utils/secrets/**/*.go` — Token generation (tokens.go)
- `kitty/shm.py` — Python SharedMemory class
- `kitty/window.py` — SSH and askpass DCS handlers (handle_remote_ssh, handle_remote_askpass)
- `kitty/utils.py` — SSH control master cleanup
- `kitty/constants.py` — SSH control master template
- `kitty/boss.py` — close_shared_ssh_connections action
- `docs/kittens/ssh.rst` — Existing SSH documentation (reference only)
- `docs/conf.py` — Documentation build configuration (reference only)
- `docs/requirements.txt` — Documentation dependencies (reference only)
- `kittens/ssh/main.py` — SSH config schema definitions (reference only)
- `kitty_tests/ssh.py` — Python-side SSH tests (for behavioral verification)
- `kitty_tests/shm.py` — SHM integration tests (for behavioral verification)

**Documentation Topics Covered:**
- End-to-end SSH session lifecycle (9 phases)
- Shared memory security model (creation, permissions, validation, unlink)
- Bootstrap script generation and encoding (Python and POSIX shell paths)
- Tarball archive construction and contents
- Connection reuse decision logic (ControlMaster)
- Askpass SHM IPC protocol
- TTY data exchange protocol (DCS escapes, base64 streaming)
- Key data structures (connection_data, Config, EnvInstruction, CopyInstruction)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No existing repository files are modified (per user instruction and SWE-AtlasQnA-Repo rules)
- **Test file modifications** — No test files are modified
- **Feature additions or code refactoring** — This is a documentation-only task
- **Sphinx documentation tree changes** — No changes to `docs/**`, `docs/conf.py`, or `docs/Makefile`
- **Documentation for non-SSH kittens** — Only the SSH kitten subsystem is documented
- **Remote control system internals** — Beyond what the SSH kitten uses for forwarding, the full RC system is out of scope
- **Shell integration internals for Bash/Zsh/Fish** — Only the SSH-specific bootstrap entry points are covered; the full shell integration system in `kitty/shell_integration.py` is out of scope except where it intersects with SSH
- **Build system and CI/CD documentation** — Not relevant to this task
- **GPU rendering, font pipeline, or other core engine documentation** — Not relevant
- **Deployment configuration changes** — Not applicable
- **Any files or paths matching `.blitzyignore` patterns** — None found, but the rule is honored

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Output file:** `blitzy/documentation/kitty_815df1e210e0.md`
- **Output format:** Markdown with Mermaid diagrams (`.md` extension)
- **Directory creation command:** `mkdir -p blitzy/documentation`
- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file, not part of the Sphinx documentation tree
- **Documentation preview command:** Any Markdown renderer (e.g., `grip kitty_815df1e210e0.md` or a GitHub/VS Code preview)
- **Diagram generation:** Mermaid syntax embedded inline within the Markdown file; no separate CLI generation step required
- **Documentation deployment command:** Not applicable — file is committed to the repository
- **Validation:** Visual review of Mermaid rendering, link checking for internal anchors, verification that all source citations resolve to actual repository paths

### 0.9.2 Content Standards

- **Citation requirement:** Every technical claim traces back to a specific source file and approximate line range (e.g., `Source: kittens/ssh/main.go:210-250`)
- **Style:** Technical reference prose with explanatory narrative; progressive disclosure from high-level flow to implementation detail
- **Terminology:** Consistent use of terms from the codebase — "kitten" (not "module"), "ControlMaster" (not "multiplexer"), "SHM" (not "shared file"), "bootstrap" (not "setup script"), "DCS" (not "escape code")
- **Diagram standard:** Mermaid `sequenceDiagram` for protocol flows, `flowchart TD` for decision logic, `classDiagram` for data structures
- **Code snippets:** Kept to 2–3 lines maximum for illustrative purposes; never full function bodies
- **Heading structure:** `#` for document title, `##` for major phases, `###` for sub-topics within phases, `####` for fine-grained details

### 0.9.3 Temporary Artifacts

- Any temporary scripts created for observation or tracing during analysis must be cleaned up after use
- The repository itself remains unchanged (per user instruction)
- The only persistent artifact produced is the output file at `blitzy/documentation/kitty_815df1e210e0.md`

## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

- **No repository modifications:** "The repository itself should remain unchanged" — the only new file is `blitzy/documentation/kitty_815df1e210e0.md`, which lives outside the Kitty source tree
- **Cleanup of temporary artifacts:** "Anything temporary should be cleaned up afterward" — any scratch scripts or temp files created during analysis must be removed before the task is marked complete
- **SWE-AtlasQnA-Repo rule:** The output file must be named `<source_branch_name>.md` (resolved to `kitty_815df1e210e0.md`) and placed in `blitzy/documentation/`
- **Code-grounded answers:** "Do not make assumptions, base your answers on the code as the truth" — every claim in the document must cite specific source files, functions, and approximate line ranges
- **Thinking / rationale:** "Provide thinking / rationale behind the answers" — the documentation must explain not just what the code does, but why architectural choices were made where the code makes that clear
- **No modifications to existing files:** "Do not modify any existing files in the source repository" — strictly enforced

### 0.10.2 Inferred Documentation Standards

- **Trace the full end-to-end flow:** The user explicitly requested understanding "from when a user initiates an SSH session all the way to when the bootstrap actually executes on the remote side" — the document must cover every phase of this journey without gaps
- **Explain the connection reuse decision logic:** The user called this "confusing" — the documentation must present a clear flowchart or decision tree showing when a fresh connection is opened versus when an existing ControlMaster is reused
- **Demystify bootstrap encoding:** The user described the character substitutions as "wild" — the documentation must walk through the encoding step by step with examples
- **Cover SHM security thoroughly:** The user specifically asked "how the shared memory piece keeps things secure" — the documentation must detail permissions, ownership validation, and the unlink-on-read pattern
- **Explain the tarball construction:** The user asked "how does that archive with all the shell integration stuff get built and sent over" — the documentation must enumerate tarball contents and the streaming protocol
- **Describe terminal-to-remote communication:** The user wants to understand "how the terminal communicates back and forth with the remote shell during setup" — the documentation must cover the DCS escape protocol in both directions

## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and directories were inspected during the analysis phase. Each entry was read in full using `read_file` or explored via `get_source_folder_contents`.

**SSH Kitten Go Source:**

| File | Lines | Purpose |
|------|-------|---------|
| `kittens/ssh/main.go` | 898 | Core SSH kitten orchestrator — session lifecycle, bootstrap generation, tarball construction, connection reuse, env serialization |
| `kittens/ssh/askpass.go` | 109 | SHM-based askpass IPC protocol — create SHM, write prompt, DCS trigger, poll for response |
| `kittens/ssh/config.go` | 411 | SSH configuration loading — hostname matching, env/copy instruction parsing, per-host option resolution |
| `kittens/ssh/utils.go` | 247 | SSH argument parsing, executable discovery, SSH version detection, option passthrough |
| `kittens/ssh/main_test.go` | 157 | Unit tests for env cloning, bootstrap script limits, tarball generation |

**SSH Kitten Python Source:**

| File | Lines | Purpose |
|------|-------|---------|
| `kittens/ssh/main.py` | 236 | Configuration schema definitions (hostname, interpreter, remote_dir, share_connections, etc.) |
| `kittens/ssh/utils.py` | 339 | Kitty-side data serving — SHM read/create, connection data assembly, password validation, base64 streaming |

**Remote Bootstrap Scripts:**

| File | Lines | Purpose |
|------|-------|---------|
| `shell-integration/ssh/bootstrap.sh` | 164 | POSIX shell bootstrap — base64 detection, TTY data reading, DCS-to-kitty requests, env setup |
| `shell-integration/ssh/bootstrap.py` | 318 | Python bootstrap — env application, terminfo compilation, shell-specific exec (zsh/fish/bash) |
| `shell-integration/ssh/bootstrap-utils.sh` | 251 | Shell utilities — login shell discovery chain, terminfo compilation, file installation, shell launchers |

**Shared Memory Layer:**

| File | Lines | Purpose |
|------|-------|---------|
| `tools/utils/shm/shm.go` | 253 | MMap interface, CreateTemp, size-prefixed read/write protocol (4-byte length header) |
| `tools/utils/shm/shm_syscall.go` | 193 | Darwin/FreeBSD syscall-based SHM implementation (shm_open, shm_unlink, mmap) |
| `tools/utils/shm/shm_fs.go` | — | Filesystem-based SHM fallback for Linux |
| `kitty/shm.py` | 187 | Python SharedMemory class — shm_open/shm_unlink wrappers, 0o600 permissions, UID/GID validation |

**Kitty Core Handlers:**

| File | Lines Read | Purpose |
|------|------------|---------|
| `kitty/window.py` | 1289–1292, 1351–1379 | `handle_remote_ssh()` — data serving to child; `handle_remote_askpass()` — UI prompt dispatch |
| `kitty/utils.py` | 1038–1053 | `cleanup_ssh_control_masters()` — glob ControlMaster sockets, send `ssh -O exit` |
| `kitty/constants.py` | 188 | `ssh_control_master_template` definition: `kssh-{kitty_pid}-{ssh_placeholder}` |
| `kitty/boss.py` | 1–50 (summary) | `close_shared_ssh_connections` action reference |

**Security and Token Generation:**

| File | Lines | Purpose |
|------|-------|---------|
| `tools/utils/secrets/tokens.go` | 43 | `TokenHex()` — cryptographically random token generation using `crypto/rand` |

**Tests (for behavioral verification):**

| File | Lines | Purpose |
|------|-------|---------|
| `kittens/ssh/main_test.go` | 157 | Go SSH tests — env cloning, script limits, tarball |
| `kitty_tests/ssh.py` | — | Python SSH integration tests |
| `kitty_tests/shm.py` | — | SHM integration tests |

**Documentation References (existing docs inspected for style):**

| File | Purpose |
|------|---------|
| `docs/kittens/ssh.rst` | Existing user-facing SSH kitten documentation |
| `docs/conf.py` | Sphinx documentation configuration |
| `docs/requirements.txt` | Documentation build dependencies |
| `docs/changelog.rst` | Project changelog format reference |

**Directories Explored:**

| Directory | Purpose |
|-----------|---------|
| Root (`""`) | Project root structure |
| `kittens/` | All kittens directory listing |
| `kittens/ssh/` | SSH kitten source directory |
| `tools/` | Shared tools directory |
| `tools/utils/shm/` | Shared memory implementation directory |
| `shell-integration/` | Shell integration root |
| `shell-integration/ssh/` | SSH-specific bootstrap scripts |
| `docs/` | Documentation root |
| `docs/kittens/` | Kitten-specific documentation |

### 0.11.2 Tech Spec Sections Retrieved

| Section | Key Information Extracted |
|---------|--------------------------|
| 4.7 SHELL INTEGRATION FLOW | SSH bootstrap sequence overview, shell-specific setup details, data exchange protocol description |
| 4.6 REMOTE CONTROL SYSTEM FLOW | Command execution model, authorization chain, transport mechanisms relevant to SSH forwarding |
| 4.8 KITTENS FRAMEWORK EXECUTION FLOW | Kitten resolution, module loading, entry points, result serialization |
| 6.4 Security Architecture | SSH bootstrap isolation model, SHM security architecture, 7-layer RC authorization chain |
| 2.1 Feature Catalog | Feature registry including F-005 (Kittens Framework) and F-008 (Shell Integration) |

### 0.11.3 User Attachments and External Resources

- **Attachments provided:** None (0 environments attached, no files in `/tmp/environments_files/`)
- **Figma URLs provided:** None
- **External URLs referenced:** None
- **Environment variables provided:** None
- **Secrets provided:** None

