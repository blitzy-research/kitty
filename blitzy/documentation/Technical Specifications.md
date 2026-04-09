# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that captures the results of an investigative analysis of kitty terminal's OSC 133 shell integration escape sequence handling. The investigation encompasses runtime behavior, byte-level analysis, exit code processing, and edge-case handling — all derived from the actual source code and verified through executable tests against the compiled codebase.

- **Category:** Create new documentation
- **Documentation type:** Technical investigation report / Architecture documentation
- **Target document:** `blitzy/documentation/kitty_815df1e210e0.md`

The documentation requirements, restated with enhanced clarity:

- **R1 – OSC 133 sequence consumption:** Determine whether OSC 133 escape sequences (`\033]133;A\007`, `\033]133;C;cmdline=...\007`, `\033]133;D;...\007`) appear in the terminal's visible screen output after parsing, or if they are consumed by the VT parser. Provide the total byte length of a representative input sequence and identify the byte offset at which the `D;42` marker appears within the raw input stream.
- **R2 – Exit code variation analysis:** Run the same test structure with exit codes `0`, `1`, and `127`. Report byte lengths and `D;{code}` marker positions for each. Analyze whether the position of the exit code number shifts and by how much.
- **R3 – Exit code 99 runtime proof:** Demonstrate runtime evidence that exit code `99` is processed through the entire code path — from VT parser dispatch through C-layer shell_prompt_marking() through to the Python callback layer — by showing the before/after state change.
- **R4 – Invalid exit codes:** Determine what values are recorded when `OSC 133;D;not_a_number` (non-numeric) and `OSC 133;D;` (empty string after semicolon) are sent, covering both the test Callbacks behavior and the production `Window.handle_cmd_end()` behavior.
- **R5 – No-modification constraint:** No existing repository files may be modified during the investigation. Any test scripts created must be cleaned up when done.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL:** "Please don't modify any repository files during the investigation, and clean up any test scripts when done." — This means all analysis is read-only with respect to the repository. Ephemeral test scripts were created in `/tmp/`, executed, and deleted.
- **Style:** Technical investigation format with code-traced evidence, byte-level detail, and runtime proof.
- **Approach:** Answers must be derived from code as truth, not from assumptions. Every claim must reference specific source file paths and line ranges.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document OSC 133 sequence consumption (R1), we will **create** a new investigation document tracing the code path from `kitty/vt-parser.c:dispatch_osc()` (case 133) through `kitty/screen.c:shell_prompt_marking()` to the Python callback layer in `kitty/window.py:cmd_output_marking()`, verifying with a compiled test harness that OSC sequences are consumed by the VT parser and never appear in screen buffer output.
- To document exit code variation (R2), we will **create** a comparative byte-length analysis showing that the `D;{code}` marker offset remains constant at byte 44 (inside the fixed-prefix OSC envelope) while total input length varies by the digit count of the exit code string.
- To document exit code 99 processing (R3), we will **create** a runtime evidence section showing the sentinel-to-99 state transition in `callbacks.last_cmd_exit_status`, tracing the full code path with specific file references.
- To document invalid exit codes (R4), we will **create** an edge-case analysis covering both the test `Callbacks` class (`suppress(Exception)` path in `kitty_tests/__init__.py:79`) and the production `Window.handle_cmd_end()` (`except: set to 0` path in `kitty/window.py:1413-1415`).

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `shell_prompt_marking()` function in `kitty/screen.c` handles markers `A`, `C`, and `D` — the `B` marker is silently ignored (no case in the switch statement), which warrants documentation as a notable implementation detail.
- Based on structure: The OSC 133 handling spans three language layers (C parser → C screen model → Python callbacks), which requires a cross-layer documentation approach.
- Based on dependencies: The `decode_cmdline()` function in `kitty/window.py` supports both `cmdline=` (shell `%q` quoting) and `cmdline_url=` (URL percent-encoding) formats, adding a format-awareness detail to the investigation.
- Based on user requirements: The implementation rule mandates the output document be named `kitty_815df1e210e0.md` and placed in the `blitzy/documentation/` directory.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a Sphinx-based documentation framework with comprehensive coverage of shell integration concepts, but no existing investigative document covering the specific byte-level and runtime behavior questions posed by the user.

- **Documentation framework:** Sphinx (configured in `docs/conf.py`)
- **Documentation format:** reStructuredText (`.rst`) files under `docs/`
- **Existing shell integration docs:** `docs/shell-integration.rst` — covers the OSC 133 protocol specification, shell-specific setup (Bash, Zsh, Fish), and integration features at a protocol level
- **Changelog documentation:** `docs/changelog.rst` — references shell integration improvements across multiple releases
- **No existing Markdown investigation documents** in `blitzy/documentation/` — directory does not yet exist

Key documentation files discovered:

| File | Relevance |
|------|-----------|
| `docs/shell-integration.rst` | Protocol specification for OSC 133;A, OSC 133;C, OSC 133;D, including cmdline encoding formats |
| `docs/conf.py` | Sphinx documentation build configuration |
| `CONTRIBUTING.md` | Contributor guidelines |
| `README.asciidoc` | Project overview in AsciiDoc format |
| `INSTALL.md` | Build and installation instructions |

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for OSC 133-related code:

- **VT Parser dispatch:** `kitty/vt-parser.c` — `dispatch_osc()` function at line 457, case 133 at line 536
- **Screen-layer handling:** `kitty/screen.c` — `shell_prompt_marking()` at line 2328, `parse_prompt_mark()` at line 2316
- **Screen header declarations:** `kitty/screen.h` — function prototype at line 231
- **Data type definitions:** `kitty/data-types.h` — `PromptKind` enum at line 230 (`UNKNOWN_PROMPT_KIND`, `PROMPT_START`, `SECONDARY_PROMPT`, `OUTPUT_START`)
- **Python window callbacks:** `kitty/window.py` — `handle_cmd_end()` at line 1408, `cmd_output_marking()` at line 1453, `decode_cmdline()` at line 225
- **Test callbacks:** `kitty_tests/__init__.py` — `Callbacks.cmd_output_marking()` at line 71, `parse_bytes()` helper at line 32
- **Screen tests:** `kitty_tests/screen.py` — `test_prompt_marking()` at line 1056
- **Shell integration tests:** `kitty_tests/shell_integration.py` — integration tests for Bash, Zsh, Fish with real PTY subprocess interactions
- **Shell integration scripts:** `shell-integration/bash/kitty.bash`, `shell-integration/zsh/kitty-integration`, `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`
- **Go shell integration:** `tools/tui/shell_integration/api.go`, `tools/tui/shell_integration/api_test.go`

### 0.2.3 Web Search Research Conducted

No external web search was required. All answers were derived from the actual source code and verified through compiled test execution against the kitty 0.35.2 codebase. The investigation is grounded entirely in code-as-truth, as required by the implementation rules.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The investigation spans three language layers of the kitty terminal emulator. Every module involved in the OSC 133 code path is documented below.

- **Module: `kitty/vt-parser.c` — VT Parser / OSC Dispatcher**
  - Public APIs: `dispatch_osc()` (static, line 457)
  - Current documentation: Protocol-level documentation exists in `docs/shell-integration.rst`; no byte-level behavior documentation
  - Documentation needed: How the parser extracts the OSC code `133`, locates the payload after `133;`, null-terminates the buffer, and calls `shell_prompt_marking()`

- **Module: `kitty/screen.c` — Screen Model / Prompt Marking**
  - Public APIs: `shell_prompt_marking()` (line 2328), `parse_prompt_mark()` (static, line 2316)
  - Current documentation: No existing documentation of the marker-by-marker behavior (A/C/D cases)
  - Documentation needed: Switch-case logic for markers A, C, D; the extraction of `exit_status` from the `D;{code}` format; the `CALLBACK` macro invocation patterns; the fact that marker B is silently ignored

- **Module: `kitty/screen.h` / `kitty/data-types.h` — Type Definitions**
  - Public APIs: `PromptKind` enum (`UNKNOWN_PROMPT_KIND=0`, `PROMPT_START=1`, `SECONDARY_PROMPT=2`, `OUTPUT_START=3`)
  - Current documentation: None beyond inline code
  - Documentation needed: How line attributes track prompt state transitions

- **Module: `kitty/window.py` — Python Window Callbacks**
  - Public APIs: `handle_cmd_end()` (line 1408), `cmd_output_marking()` (line 1453), `decode_cmdline()` (line 225)
  - Current documentation: Inline docstrings only
  - Documentation needed: The `try: int(exit_status) / except: 0` error handling for invalid exit codes; the `decode_cmdline()` support for `cmdline=` vs `cmdline_url=` formats

- **Module: `kitty_tests/__init__.py` — Test Callbacks**
  - Public APIs: `Callbacks.cmd_output_marking()` (line 71), `parse_bytes()` (line 32)
  - Current documentation: None
  - Documentation needed: The `suppress(Exception)` path that differs from production behavior

- **Module: Shell integration scripts (Bash/Zsh/Fish)**
  - Files: `shell-integration/bash/kitty.bash`, `shell-integration/zsh/kitty-integration`, `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`
  - Documentation needed: How each shell emits OSC 133 markers with exit codes (referenced for context)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No runtime investigation document exists** — The existing `docs/shell-integration.rst` describes the protocol specification but not what actually happens at the byte level when sequences are processed
- **No byte-offset analysis** — No documentation covers the relationship between exit code digit count and total input byte length
- **No edge-case documentation** — The behavior for invalid exit codes (`not_a_number`, empty string) is not documented anywhere
- **No cross-layer trace** — The three-language code path (C parser → C screen model → Python callbacks) has never been documented as an end-to-end trace
- **The `B` marker gap** — The fact that kitty's `shell_prompt_marking()` does not handle the `B` marker (used by iTerm2's full protocol) is undocumented

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/kitty_815df1e210e0.md` will follow this structure:

```
blitzy/
└── documentation/
    └── kitty_815df1e210e0.md
        ├── Overview (question summary and approach)
        ├── OSC 133 Code Path Architecture
        │   ├── VT Parser Dispatch (vt-parser.c)
        │   ├── Screen Model Handling (screen.c)
        │   └── Python Callback Layer (window.py)
        ├── Investigation 1: Sequence Consumption & Byte Analysis
        │   ├── Test Setup
        │   ├── Results (visible output, byte length, D;42 offset)
        │   └── Rationale
        ├── Investigation 2: Exit Code Variation (0, 1, 127)
        │   ├── Comparative Results Table
        │   ├── Position Shift Analysis
        │   └── Rationale
        ├── Investigation 3: Exit Code 99 Runtime Evidence
        │   ├── Before/After State
        │   ├── Full Code Path Trace
        │   └── Rationale
        ├── Investigation 4: Invalid Exit Codes
        │   ├── "not_a_number" behavior
        │   ├── Empty string behavior
        │   ├── Production vs Test difference
        │   └── Rationale
        ├── Additional Finding: B Marker Handling
        └── Source References
```

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach:**
  - Extract the `shell_prompt_marking()` switch logic from `kitty/screen.c:2328-2355`
  - Extract the `handle_cmd_end()` error handling from `kitty/window.py:1408-1415`
  - Extract the `dispatch_osc()` case 133 routing from `kitty/vt-parser.c:536-545`
  - Generate runtime evidence by running the compiled kitty C extension via `kitty_tests` infrastructure
  - Derive byte-offset calculations from the actual raw byte sequences used in tests

- **Documentation Standards:**
  - Markdown formatting with proper heading hierarchy
  - Code examples using fenced code blocks with language identifiers
  - Source citations as inline references: `Source: kitty/screen.c:2328`
  - Tables for comparative exit code analysis
  - Every claim backed by specific file path and line number

### 0.4.3 Diagram and Visual Strategy

The document will include a Mermaid sequence diagram showing the three-layer OSC 133 processing pipeline:

```mermaid
sequenceDiagram
    participant Input as Byte Stream
    participant VTP as vt-parser.c<br/>dispatch_osc()
    participant SCR as screen.c<br/>shell_prompt_marking()
    participant PY as window.py<br/>cmd_output_marking()
    
    Input->>VTP: ESC ] 133;D;42 BEL
    VTP->>VTP: Parse OSC code = 133
    VTP->>SCR: shell_prompt_marking(buf="D;42")
    SCR->>SCR: case 'D': exit_status = "42"
    SCR->>PY: CALLBACK("cmd_output_marking", Py_None, "42")
    PY->>PY: int("42") → 42
    PY->>PY: last_cmd_exit_status = 42
```

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/screen.h`, `kitty/data-types.h`, `kitty/window.py`, `kitty_tests/__init__.py`, `kitty_tests/screen.py`, `shell-integration/bash/kitty.bash`, `shell-integration/zsh/kitty-integration`, `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`, `docs/shell-integration.rst` | Complete investigation document answering all five requirements: OSC 133 sequence consumption, byte-level analysis, exit code variation, runtime proof for code 99, and invalid exit code handling |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Investigation Report
Source Code:
  - kitty/vt-parser.c (lines 457-545: dispatch_osc, case 133)
  - kitty/screen.c (lines 2316-2355: parse_prompt_mark, shell_prompt_marking)
  - kitty/screen.h (line 231: function prototype)
  - kitty/data-types.h (line 230: PromptKind enum)
  - kitty/window.py (lines 225-228: decode_cmdline; lines 1408-1415: handle_cmd_end; lines 1453-1461: cmd_output_marking)
  - kitty_tests/__init__.py (lines 32-35: parse_bytes; lines 71-79: Callbacks.cmd_output_marking)
  - shell-integration/bash/kitty.bash (line 208: OSC 133;C emission; line 239: OSC 133;D emission)
  - shell-integration/zsh/kitty-integration (lines 145-149: OSC 133;D emission)
  - shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish (lines 83-96: OSC 133 emission)
  - docs/shell-integration.rst (lines 426-463: protocol specification)
Sections:
  - Overview (question summary and methodology)
  - OSC 133 Code Path Architecture (three-layer trace with Mermaid diagram)
  - Investigation 1: Sequence Consumption & Byte Analysis (R1)
  - Investigation 2: Exit Code Variation 0/1/127 (R2)
  - Investigation 3: Exit Code 99 Runtime Evidence (R3)
  - Investigation 4: Invalid Exit Codes (R4)
  - Additional Finding: B Marker Handling
  - Source References
Diagrams:
  - Sequence diagram: OSC 133 processing pipeline across three language layers
Key Citations:
  - kitty/vt-parser.c, kitty/screen.c, kitty/window.py, kitty_tests/__init__.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration changes are needed. The output is a standalone Markdown file placed in `blitzy/documentation/`, which is independent of the Sphinx documentation build system used by the kitty project.

### 0.5.4 Cross-Documentation Dependencies

- The new document references the protocol specification in `docs/shell-integration.rst` for OSC 133 marker definitions
- No navigation, TOC, or index updates are required since the file resides in `blitzy/documentation/` (outside the Sphinx documentation tree)

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages were required to conduct the investigation (building the kitty C extension to run the test harness):

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| system | Python | 3.12.3 | Runtime for kitty's Python layer and test infrastructure (project requires >=3.8) |
| system | GCC | 13.3.0 | C compiler for building kitty's C extensions (`fast_data_types.so`) |
| system | Go | 1.22.2 | Go compiler for building kitty's Go-based tooling |
| apt | libharfbuzz-dev | (system) | HarfBuzz text shaping library (build dependency) |
| apt | libfontconfig-dev | (system) | Font configuration library (build dependency) |
| apt | libgl1-mesa-dev | (system) | OpenGL development headers (build dependency) |
| apt | libxkbcommon-x11-dev | (system) | XKB keyboard handling library (build dependency) |
| apt | libssl-dev | 3.0.13 | OpenSSL development headers (build dependency for Go tools) |
| apt | libsimde-dev | 0.7.2 | SIMD Everywhere headers for SIMD string operations (build dependency) |
| apt | libpython3-dev | 3.12 | Python C API headers (build dependency) |
| pip | Sphinx | (not installed) | Documentation build tool (used by project but not needed for this task) |

### 0.6.2 Documentation Reference Updates

No link updates are required. The output document is a new standalone file with no pre-existing inbound or outbound link dependencies.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

- **User questions addressed:** 5/5 (100%)
  - R1: OSC 133 sequence consumption and byte analysis — ✅ Answered with runtime proof
  - R2: Exit code variation (0, 1, 127) byte lengths and positions — ✅ Answered with comparative table
  - R3: Exit code 99 runtime evidence — ✅ Answered with sentinel-to-value state transition
  - R4: Invalid exit codes ("not_a_number", empty) — ✅ Answered with both test and production code paths
  - R5: No repository modifications — ✅ All tests run via `/tmp/` scripts, cleaned up
- **Source files traced:** 10 source files across 3 language layers
- **Code path coverage:** Complete end-to-end trace from VT parser byte intake through Python callback storage

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every investigation result includes the test input data, the observed output, and the rationale (why the code produces that result)
  - Exit code variation analysis includes a comparative table with all three codes side by side
  - Invalid exit code analysis covers both the test framework behavior and the production `Window.handle_cmd_end()` behavior, noting the difference

- **Accuracy validation:**
  - All results were produced by actually building kitty 0.35.2 from source and running tests against the compiled `fast_data_types.so` C extension
  - Every code reference includes the exact file path and line number verified against the repository at commit `815df1e21`
  - The byte calculations are derived from the actual Python `repr()` of the test byte strings

- **Clarity standards:**
  - Technical accuracy with step-by-step code path traces
  - Progressive disclosure: overview first, then detailed per-investigation sections
  - Consistent terminology: "OSC 133;X" for escape sequences, "VT parser" for the C parser layer, "screen model" for the C screen layer, "callback layer" for Python

- **Maintainability:**
  - Source citations with file:line format for traceability
  - Self-contained document with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Code examples per investigation:** Minimum 1 (the actual byte sequence used and the observed result)
- **Diagram types:** 1 Mermaid sequence diagram showing the three-layer OSC 133 processing pipeline
- **Tables:** 1 comparative table for exit code byte analysis, 1 summary table for invalid exit code behavior

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation files:**
  - `blitzy/documentation/kitty_815df1e210e0.md` — The sole output artifact
- **Source code analysis (read-only):**
  - `kitty/vt-parser.c` — OSC dispatch logic
  - `kitty/screen.c` — shell_prompt_marking() implementation
  - `kitty/screen.h` — function declarations
  - `kitty/data-types.h` — PromptKind enum
  - `kitty/window.py` — handle_cmd_end(), cmd_output_marking(), decode_cmdline()
  - `kitty_tests/__init__.py` — parse_bytes(), Callbacks class
  - `kitty_tests/screen.py` — test_prompt_marking()
  - `kitty_tests/shell_integration.py` — integration test patterns
  - `shell-integration/bash/kitty.bash` — OSC 133 emission in Bash
  - `shell-integration/zsh/kitty-integration` — OSC 133 emission in Zsh
  - `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish` — OSC 133 emission in Fish
  - `docs/shell-integration.rst` — Protocol specification reference
- **Runtime investigation:**
  - Building kitty C extensions for test execution
  - Running ephemeral test scripts in `/tmp/` against the compiled modules
  - Byte-level analysis of OSC 133 input sequences

### 0.8.2 Explicitly Out of Scope

- Source code modifications to any repository file (per user instruction: "Please don't modify any repository files")
- Test file modifications
- Feature additions or code refactoring
- Changes to the Sphinx documentation build system
- Documentation of OSC 133 features beyond the scope of the user's questions (e.g., the `redraw=0` or `special_key=1` parameters of `OSC 133;A`)
- Documentation of unrelated shell integration features (CWD notifications, cursor shape management, clone-session)
- Deployment configuration changes
- Persistent test scripts (all cleaned up per user instruction)

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Build command for investigation harness:** `python3 setup.py build --debug` (builds `kitty/fast_data_types.so` C extension)
- **Test execution pattern:** Python scripts in `/tmp/` importing from `kitty_tests` and `kitty.fast_data_types`
- **Default format:** Markdown (`.md`) per the implementation rule specifying `blitzy/documentation/<source_branch_name>.md`
- **Citation requirement:** Every technical claim references source file path and line number
- **Style guide:** Technical investigation format — question, method, result, rationale
- **Cleanup requirement:** All ephemeral test scripts removed after execution

### 0.9.2 Key Runtime Findings Summary

The following findings were established through compiled test execution and will be documented:

| Investigation | Key Finding |
|---------------|-------------|
| OSC 133 consumption | OSC sequences are consumed by the VT parser; only `Hello World` appears on screen. No OSC bytes in visible output. |
| Byte analysis (D;42) | Total input: 49 bytes. `D;42` appears at byte offset 44 within the raw input stream. |
| Exit code 0 | Total: 48 bytes, D offset: 44, parsed status: 0 |
| Exit code 1 | Total: 48 bytes, D offset: 44, parsed status: 1 |
| Exit code 127 | Total: 50 bytes, D offset: 44, parsed status: 127 |
| Position shift | D marker offset is constant (44). Total length varies only by exit code digit count (+2 bytes for 3-digit vs 1-digit). |
| Exit code 99 proof | `last_cmd_exit_status` changes from sentinel (9223372036854775807) to exactly 99 |
| "not_a_number" | Test Callbacks: stays at sentinel (int fails, suppressed). Production Window: falls to 0 |
| Empty exit code | Test Callbacks: stays at sentinel. Production Window: falls to 0 |

## 0.10 Rules for Documentation

The following rules are explicitly specified by the user and implementation guidelines:

- **Do not modify any existing files in the source repository.** All analysis is read-only. The output document is created in `blitzy/documentation/`, not in the existing `docs/` tree.
- **Clean up any test scripts when done.** All ephemeral scripts created in `/tmp/` for the investigation must be deleted after execution.
- **Do not make assumptions, base answers on the code as the truth.** Every claim in the output document must reference specific source file paths and be verified through actual code execution where possible.
- **Provide thinking / rationale behind the answers.** Each investigation section must include not just the result but the reasoning traced through the code.
- **Create the output as `kitty_815df1e210e0.md`** in the `blitzy/documentation/` directory, matching the source branch name `kitty_815df1e210e0`.
- **Comprehensive answers required.** The document must cover all five requirements (R1–R5) completely, with no placeholder content or deferred analysis.

## 0.11 References

### 0.11.1 Files and Folders Searched

The following source files were retrieved, analyzed, and used to derive the investigation conclusions:

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `kitty/vt-parser.c` (lines 457–545) | OSC dispatch logic; case 133 routing to `shell_prompt_marking()` |
| `kitty/screen.c` (lines 2316–2355) | `shell_prompt_marking()` implementation — A/C/D marker handling, exit status extraction |
| `kitty/screen.c` (lines 3527–3680) | `find_cmd_output()` and `cmd_output()` — command output retrieval using prompt markers |
| `kitty/screen.h` (line 231) | Function prototype for `shell_prompt_marking()` |
| `kitty/data-types.h` (line 230) | `PromptKind` enum definition: `UNKNOWN_PROMPT_KIND`, `PROMPT_START`, `SECONDARY_PROMPT`, `OUTPUT_START` |
| `kitty/window.py` (lines 225–228) | `decode_cmdline()` — cmdline format parsing (`cmdline=` and `cmdline_url=`) |
| `kitty/window.py` (lines 455–470) | `cmd_output()` — output retrieval with OSC 133;C stripping |
| `kitty/window.py` (lines 1408–1461) | `handle_cmd_end()` and `cmd_output_marking()` — exit status parsing with try/except fallback to 0 |
| `kitty_tests/__init__.py` (lines 32–35) | `parse_bytes()` helper — test infrastructure for feeding bytes to the VT parser |
| `kitty_tests/__init__.py` (lines 40–108) | `Callbacks` class — test callback implementation with `suppress(Exception)` for exit status parsing |
| `kitty_tests/screen.py` (lines 1056–1130) | `test_prompt_marking()` — existing test coverage for OSC 133 prompt marking |
| `kitty_tests/shell_integration.py` | Integration tests for Bash, Zsh, Fish shell integration with real PTY |
| `shell-integration/bash/kitty.bash` (lines 127, 208, 239) | Bash OSC 133 marker emission (`printf "\e]133;C;cmdline=%q\a"`, `\e]133;D;\$?\a`) |
| `shell-integration/zsh/kitty-integration` (lines 124–218) | Zsh OSC 133 marker emission and state tracking |
| `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish` (lines 72–96) | Fish OSC 133 marker emission |
| `docs/shell-integration.rst` (lines 415–463) | OSC 133 protocol specification documentation |
| `kitty/constants.py` (line 25) | Version identification: kitty 0.35.2 |
| `pyproject.toml` (line 2) | Python version requirement: `>=3.8` |
| `setup.py` | Build system for compiling C extensions |
| `tools/tui/shell_integration/api.go` | Go-side shell integration (context) |
| `tools/tui/shell_integration/api_test.go` | Go-side shell integration tests (context) |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

### 0.11.4 Tech Spec Sections Referenced

| Section | Content Used |
|---------|-------------|
| 4.7 SHELL INTEGRATION FLOW | OSC 133 prompt boundary markers protocol, shell-specific setup details, active integration features table |
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | VT Parser dispatch architecture, OSC routing to shell markers, sequence classification matrix |
| 1.1 Executive Summary | Project overview (kitty 0.35.2, three-language architecture: C/Python/Go, GPLv3) |

