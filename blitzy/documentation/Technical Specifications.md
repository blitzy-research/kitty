# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a comprehensive, code-grounded technical analysis of the keyboard progressive enhancement protocol stack's behavior during alternate screen buffer switches in the Kitty terminal emulator.

**Documentation Type:** Technical deep-dive / Q&A investigation document

**Category:** Create new documentation — a single Markdown document titled `kitty_815df1e210e0.md` placed in the `blitzy/documentation/` directory, as specified by the project's implementation rules.

The user's requirements decompose into the following concrete documentation needs:

- **Stack Isolation Proof:** Definitively answer whether the main and alternate screen buffers maintain fully independent keyboard enhancement flag stacks. The answer must be grounded in code analysis of the `Screen` structure in `kitty/screen.h` (line 128), which declares two separate arrays `main_key_encoding_flags[8]` and `alt_key_encoding_flags[8]`, and in the `screen_toggle_screen_buffer()` function in `kitty/screen.c` (lines 1068–1095) that swaps the `key_encoding_flags` pointer.

- **Round-Trip State Preservation:** Document what happens when a user pushes keyboard flags on the main buffer, switches to alternate, pushes different flags, then switches back to main. The user needs to know whether the main buffer's stack survives the round-trip. This requires tracing the pointer-swap logic in `screen_toggle_screen_buffer()`.

- **Byte Sequence Evidence:** For the same key press (e.g., Ctrl+Shift+A), document the actual escape sequences transmitted under different flag/buffer combinations — specifically: no flags (legacy mode), disambiguate mode (flag `0b1`), report-all-keys mode (flag `0b1000`), and combinations thereof. This requires analyzing `encode_glfw_key_event()` in `kitty/key_encoding.c` (lines 413–440) and the test assertions in `kitty_tests/keys.py`.

- **Stack Exhaustion Behavior:** Document the exact behavior when the 8-entry stack limit is exceeded. The code in `screen_push_key_encoding_flags()` at `kitty/screen.c` (lines 1234–1244) uses `memmove()` to evict the oldest entry — this needs to be clearly documented with concrete examples.

- **Cross-Buffer Exhaustion Independence:** Confirm whether stack exhaustion on one buffer affects the other buffer's stack. This follows directly from the two arrays being separate.

- **Edge Cases and Mode Interactions:** Document any conditions under which the stack isolation might break down or behave unexpectedly, including `screen_reset()` behavior, and interaction with modes like DECCKM (cursor key mode).

- **Controlled Test Scenarios:** Leverage the existing test infrastructure (`kitty_tests/screen.py` and `kitty_tests/keys.py`) to demonstrate and validate each of the above behaviors through actual test execution where possible, or through code-level walkthrough where runtime testing is constrained.

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**

- **No source file modifications.** The user explicitly states: "Don't modify any source files in the repository. Use only what exists in the codebase." All analysis must be read-only.
- **Runtime behavior over theory.** The user states: "I want to understand the actual runtime behavior, not just what the code says should happen." Where possible, the document should include output from running existing test infrastructure and code-level walkthroughs tracing actual execution paths.
- **Use existing test infrastructure.** The user asks to leverage "any test infrastructure or utilities that might help demonstrate this behavior" — the existing `kitty_tests/screen.py::test_key_encoding_flags_stack` test and the `parse_bytes()` function are key utilities.
- **Create output document as `kitty_815df1e210e0.md`** in `blitzy/documentation/` per the SWE-AtlasQnA-Repo rule: "Create a new markdown document named `<source_branch_name>.md`."
- **Provide thinking/rationale** behind all answers per implementation rules.
- **Base answers on code as truth** — do not make assumptions.

**Style Preferences:**

- Include concrete byte-sequence examples in hexadecimal and human-readable notation
- Use Mermaid diagrams for data flow and state transitions
- Include code citations with exact file paths and line numbers
- Organize content progressively: concepts → code evidence → test validation → edge cases

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **prove stack isolation**, we will trace the `Screen` struct's `main_key_encoding_flags[8]` vs `alt_key_encoding_flags[8]` arrays and the `key_encoding_flags` pointer swap in `screen_toggle_screen_buffer()` at `kitty/screen.c:1068–1095`.

- To **document round-trip behavior**, we will walk through the sequence of operations: push on main → toggle to alt → push on alt → toggle to main, tracing exactly how the pointer assignment (`self->key_encoding_flags = self->main_key_encoding_flags` at line 1086) restores the main screen's independent stack state.

- To **produce byte-sequence evidence**, we will analyze `encode_glfw_key_event()` in `kitty/key_encoding.c:413–440` which maps `key_encoding_flags` bits to the `KeyEvent` struct fields (`disambiguate`, `report_all_event_types`, `report_alternate_key`, `report_text`, `embed_text`), then trace `encode_key()` to show the actual output for a Ctrl+Shift+A key event under each flag configuration.

- To **document stack exhaustion**, we will analyze `screen_push_key_encoding_flags()` at `kitty/screen.c:1234–1244`, specifically the `memmove()` eviction logic when `current_idx == sz - 1`.

- To **demonstrate these behaviors**, we will reference and explain the existing `test_key_encoding_flags_stack` test at `kitty_tests/screen.py:952–993` which exercises push, pop, set, overflow, and reset operations.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are identified:

- **The 0x80 sentinel bit mechanism** — The code uses the high bit (0x80) of each `uint8_t` slot as a "valid entry" marker. `screen_current_key_encoding_flags()` iterates backward through the array looking for entries with bit 0x80 set, and the actual flags value is stored in the lower 7 bits (masked by `& 0x7f`). This non-obvious implementation detail is essential for understanding all stack operations but is not documented anywhere.

- **The set-flags modes (how=1,2,3)** — `screen_set_key_encoding_flags()` supports three `how` modes: replace (1), bitwise-OR (2), and bitwise-AND-NOT (3). The user didn't explicitly ask about this, but it's integral to understanding the protocol's flag manipulation semantics and belongs in the document.

- **VT parser routing** — The escape sequences `CSI > u` (push), `CSI < u` (pop), `CSI = u` (set), and `CSI ? u` (query) are dispatched in `kitty/vt-parser.c:1217–1241`. Documenting the full input-to-output pipeline helps the user understand how their escape code writes actually reach the stack functions.

- **Reset behavior** — `screen_reset()` at `kitty/screen.c:161–174` clears both `main_key_encoding_flags` and `alt_key_encoding_flags` with `memset(..., 0, ...)`. This is relevant because it demonstrates that a full terminal reset (CSI c or RIS) zeroes out both buffers' stacks simultaneously — one of the few operations that crosses the buffer isolation boundary.

- **DECCKM interaction** — The `on_key_input()` function in `kitty/keys.c:251` passes both `screen->modes.mDECCKM` and `screen_current_key_encoding_flags(screen)` to `encode_glfw_key_event()`. When DECCKM is set and no keyboard flags are pushed, arrow keys produce application-mode sequences (`ESC O A` instead of `ESC [ A`). This mode interaction is relevant to the user's question about "conditions where the stack isolation breaks down."


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a mature Sphinx/reStructuredText documentation infrastructure with comprehensive coverage of the keyboard protocol, but a gap in the specific area of screen-buffer interaction with the keyboard flag stack.

**Documentation framework:** Sphinx (no pinned version in `docs/requirements.txt`; dependency is listed as `sphinx`)
**Documentation generator configuration:** `docs/conf.py` — central Sphinx configuration with theme, extensions, roles, lexers
**Build driver:** `docs/Makefile` — standard Sphinx build targets plus `develop-docs` live preview via `sphinx-autobuild`
**Documentation theme:** `furo` (specified in `docs/requirements.txt`)
**Additional doc tooling:** `sphinx-copybutton`, `sphinxext-opengraph`, `sphinx-inline-tabs`, `sphinx-autobuild`
**API documentation tools:** None detected for auto-generation from source (no JSDoc, Sphinx autodoc, or similar configured)
**Diagram tools:** Mermaid references in the tech spec; the docs themselves use reStructuredText directives

**Key Existing Documentation Files Reviewed:**

| File | Relevance | Summary |
|------|-----------|---------|
| `docs/keyboard-protocol.rst` | **Primary** | The canonical specification of the Kitty progressive keyboard enhancement protocol. Documents the escape sequences (`CSI = u`, `CSI > u`, `CSI < u`, `CSI ? u`), the five flag bits, the push/pop stack semantics, and explicitly states "Terminals must maintain separate stacks for the main and alternate screens." Also documents the eviction behavior: "If a push request is received and the stack is full, the oldest entry from the stack must be evicted." |
| `docs/conf.rst` | Peripheral | Configuration file reference; may reference keyboard-related config options |
| `docs/actions.rst` | Peripheral | Action definitions including `push_keyboard_mode` and `pop_keyboard_mode` |
| `docs/faq.rst` | Low | FAQ entries that may touch on keyboard behavior |
| `docs/basic.rst` | Low | Basic usage documentation |

**Gap Identified:** While `docs/keyboard-protocol.rst` specifies the protocol at a high level, it does not provide:
- Concrete byte-sequence examples showing output under each flag combination
- Detailed walkthrough of the C implementation and data structures
- Empirical demonstration of stack isolation across buffer switches
- Documentation of the 0x80 sentinel bit mechanism or the exact stack depth (8 entries)
- Edge-case behavior documentation for stack exhaustion with specific eviction semantics

### 0.2.2 Repository Code Analysis for Documentation

**Critical source files examined for documentation content:**

| File | Purpose | Lines | Key Constructs |
|------|---------|-------|----------------|
| `kitty/screen.h` | Screen struct definition | 290 | `main_key_encoding_flags[8]`, `alt_key_encoding_flags[8]`, `*key_encoding_flags` pointer (line 128) |
| `kitty/screen.c` | Screen operations | 4932 | `screen_toggle_screen_buffer()` (L1068), `screen_push_key_encoding_flags()` (L1234), `screen_pop_key_encoding_flags()` (L1248), `screen_set_key_encoding_flags()` (L1220), `screen_current_key_encoding_flags()` (L1204), `screen_report_key_encoding_flags()` (L1212), `screen_reset()` (L161) |
| `kitty/keys.c` | Key event processing dispatch | 543 | `on_key_input()` (L166), `encode_glfw_key_event()` call with flags (L251) |
| `kitty/keys.h` | Key processing header | 22 | `encode_glfw_key_event()` signature, `KEY_BUFFER_SIZE` (128) |
| `kitty/key_encoding.c` | Key-to-escape-sequence encoder | 440 | `encode_glfw_key_event()` (L413), `encode_key()` (L367), `encode_function_key()` (L148), `serialize()` (L64), flag-to-field mapping (L419–423) |
| `kitty/vt-parser.c` | VT escape sequence parser/dispatcher | ~1250+ | CSI `u` handler routing push/pop/set/query (L1217–1241) |
| `kitty/modes.h` | Terminal mode constants | 90 | `TOGGLE_ALT_SCREEN_1` (47<<5), `TOGGLE_ALT_SCREEN_2` (1047<<5), `ALTERNATE_SCREEN` (1049<<5) |
| `kitty/data-types.h` | Utility macros | — | `arraysz()` macro (L42) used for stack size calculation |
| `kitty_tests/screen.py` | Screen test suite | ~1050 | `test_key_encoding_flags_stack()` (L952) — tests push, pop, set, overflow, reset |
| `kitty_tests/keys.py` | Key encoding test suite | ~653 | `test_encode_key_event()` — tests legacy and progressive encoding, all flag combinations |
| `kitty_tests/__init__.py` | Test infrastructure | ~280 | `parse_bytes()` helper, `Callbacks` class with `wtcbuf`, `create_screen()`, `PTY` class |
| `kitty/key_encoding.py` | Python key encoding utilities | ~400+ | `functional_key_number_to_name_map`, `KeyEvent` NamedTuple, `encode_key_event()`, `decode_key_event()` |

**Search patterns used:**
- Keyboard flag stack functions: `grep -n "key_encoding_flags"` across `kitty/screen.c`, `kitty/screen.h`
- Screen buffer toggle: `grep -n "toggle_screen_buffer"` across the codebase
- VT parser routing: `grep -n "case.*'u'"` in `kitty/vt-parser.c`
- Test coverage: `grep -n "test_key_encoding\|toggle_screen"` in `kitty_tests/`
- Mode constants: full read of `kitty/modes.h`

### 0.2.3 Web Search Research Conducted

No external web search was required for this task. The kitty codebase itself contains the authoritative keyboard protocol specification (`docs/keyboard-protocol.rst`) and comprehensive test infrastructure. The user explicitly requested analysis grounded in the actual codebase rather than external references.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require analysis and documentation to answer the user's questions:

**Module: `kitty/screen.c` + `kitty/screen.h` — Screen Buffer and Keyboard Stack Management**
- Public APIs to document:
  - `screen_push_key_encoding_flags(Screen *self, uint32_t val)` — pushes a new flags entry onto the current buffer's stack
  - `screen_pop_key_encoding_flags(Screen *self, uint32_t num)` — pops `num` entries from the current buffer's stack
  - `screen_set_key_encoding_flags(Screen *self, uint32_t val, uint32_t how)` — modifies the top-of-stack entry using mode 1 (replace), 2 (OR), or 3 (AND-NOT)
  - `screen_current_key_encoding_flags(Screen *self)` — retrieves the topmost valid flags value from the current buffer's stack
  - `screen_report_key_encoding_flags(Screen *self)` — writes `CSI ? {flags} u` back to the child process
  - `screen_toggle_screen_buffer(Screen *self, bool save_cursor, bool clear_alt_screen)` — switches between main and alternate buffers, swapping the `key_encoding_flags` pointer
  - `screen_reset(Screen *self)` — full terminal reset that clears both stacks
- Current documentation: Protocol-level behavior described in `docs/keyboard-protocol.rst`; C-level implementation undocumented
- Documentation needed: Detailed walkthrough of each function's logic, the 0x80 sentinel mechanism, pointer-swap semantics, and stack exhaustion behavior

**Module: `kitty/key_encoding.c` — Key Event Encoding Engine**
- Public APIs to document:
  - `encode_glfw_key_event()` — converts a GLFW key event plus flags into the byte sequence sent to the child process
- Current documentation: Protocol output format documented in `docs/keyboard-protocol.rst`; encoding engine internals undocumented
- Documentation needed: Flag-bit-to-behavior mapping, concrete byte output for Ctrl+Shift+A under each flag combination

**Module: `kitty/keys.c` — Key Event Dispatch Pipeline**
- Key function: `on_key_input()` at line 166 — the entry point connecting platform key events to screen state and encoding
- Current documentation: Undocumented at implementation level
- Documentation needed: How `screen_current_key_encoding_flags(screen)` is consulted at line 251 and fed into encoding

**Module: `kitty/vt-parser.c` — Escape Sequence Routing**
- Key section: CSI `u` handler at lines 1217–1241, dispatching to push/pop/set/query based on start modifier character
- Current documentation: General VT parser behavior described in tech spec section 4.3.2
- Documentation needed: Specific routing of `CSI > u`, `CSI < u`, `CSI = u`, `CSI ? u` sequences

**Module: `kitty_tests/screen.py` — Test Infrastructure**
- Key test: `test_key_encoding_flags_stack()` at line 952 — validates push, pop, set with OR/AND-NOT, reset, and stack overflow
- Current documentation: Test code is self-documenting but lacks external documentation
- Documentation needed: Explanation of test scenarios and their validation of the protocol spec

**Module: `kitty_tests/keys.py` — Key Encoding Tests**
- Key tests: Disambiguate mode (L417–433), event type reporting (L435–443), alternate key reporting (L445–451), report-all-keys (L453–462), embed-text (L464–468)
- Documentation needed: Mapping test assertions to concrete byte-sequence evidence for the user's scenarios

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented internal data structures:** The `main_key_encoding_flags[8]` / `alt_key_encoding_flags[8]` dual-array design with `uint8_t` slots using bit 0x80 as a validity sentinel is entirely undocumented anywhere in the project.

- **Missing implementation-level walkthrough:** `docs/keyboard-protocol.rst` describes the *what* (protocol semantics) but not the *how* (C implementation mechanics). No document explains the pointer-swap mechanism or the memmove eviction strategy.

- **No cross-buffer interaction documentation:** The spec states buffers "must maintain separate stacks" but provides no worked examples showing what happens during buffer switches, stack state preservation, or cross-buffer exhaustion independence.

- **No concrete byte-sequence reference table:** While the test files contain encoding assertions, there is no consolidated table showing "for key X with flags Y, the output is Z bytes."

- **Missing edge-case catalog:** No documentation covers: what happens when popping from an empty stack, the interaction between DECCKM cursor key mode and keyboard flags, or the behavior of `screen_reset()` on both stacks simultaneously.

- **No test scenario documentation:** The existing `test_key_encoding_flags_stack` test at `kitty_tests/screen.py:952` covers critical behaviors but is not accompanied by explanatory documentation for users trying to understand the protocol.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/kitty_815df1e210e0.md` will follow this structure:

```
blitzy/documentation/
└── kitty_815df1e210e0.md
    ├── Introduction and Summary
    ├── Data Structure Analysis
    │   ├── Screen Struct: Dual Key Encoding Flag Arrays
    │   ├── The 0x80 Sentinel Bit Mechanism
    │   └── Stack Depth and Array Layout
    ├── Stack Operations Deep Dive
    │   ├── Push Operation (CSI > flags u)
    │   ├── Pop Operation (CSI < num u)
    │   ├── Set Operation (CSI = flags ; mode u)
    │   ├── Query Operation (CSI ? u)
    │   └── Stack Exhaustion: Eviction via memmove
    ├── Buffer Switch Mechanics
    │   ├── screen_toggle_screen_buffer() Pointer Swap
    │   ├── Mode Constants: ALTERNATE_SCREEN (1049)
    │   └── VT Parser Routing for CSI u Sequences
    ├── Stack Isolation Proof
    │   ├── Code-Level Evidence
    │   ├── Round-Trip Scenario Walkthrough
    │   └── Cross-Buffer Exhaustion Independence
    ├── Byte Sequence Evidence
    │   ├── Encoding Pipeline: on_key_input → encode_glfw_key_event → encode_key
    │   ├── Flag Bit Mapping Table
    │   ├── Ctrl+Shift+A Under Each Flag Combination
    │   └── Consolidated Byte Sequence Reference
    ├── Test Validation
    │   ├── test_key_encoding_flags_stack Analysis
    │   ├── test_encode_key_event Flag Scenarios
    │   └── Test Execution Results
    ├── Edge Cases and Mode Interactions
    │   ├── screen_reset() Cross-Buffer Clear
    │   ├── DECCKM Cursor Key Mode Interaction
    │   ├── Pop from Empty Stack
    │   ├── Rapid Buffer Switching Under Manipulation
    │   └── Conditions Where Isolation Could Break
    └── Conclusion
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the `Screen` struct definition from `kitty/screen.h:88–170` to document the dual-array layout and pointer mechanism
- Trace `screen_push_key_encoding_flags()` at `kitty/screen.c:1234–1244` line-by-line to document the push logic including the sentinel bit, the backward scan, and the memmove eviction
- Trace `screen_toggle_screen_buffer()` at `kitty/screen.c:1068–1095` to document the pointer swap that implements buffer isolation
- Extract flag-to-behavior mapping from `encode_glfw_key_event()` at `kitty/key_encoding.c:413–424`:
  - Bit 0 (`& 1`) → `disambiguate`
  - Bit 1 (`& 2`) → `report_all_event_types`
  - Bit 2 (`& 4`) → `report_alternate_key`
  - Bit 3 (`& 8`) → `report_text`
  - Bit 4 (`& 16`) → `embed_text`
- Generate examples by analyzing test assertions in `kitty_tests/keys.py:417–468` for each flag combination
- Create Mermaid diagrams by mapping the data flow from VT parser → screen stack operations → key encoding → child output

**Documentation Standards:**

- Markdown formatting with proper heading hierarchy (`#`, `##`, `###`)
- Mermaid diagrams for: data structure layout, buffer-switch flow, encoding pipeline
- Code citations as inline references: `Source: kitty/screen.c:1234`
- Byte sequences in dual format: hex notation and readable escape notation
- Tables for parameter descriptions, flag mappings, and byte-sequence comparisons

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the output document:

- **Data Structure Diagram:** Shows the `Screen` struct with its two 8-element `uint8_t` arrays and the `key_encoding_flags` pointer pointing to one of them
- **Buffer Switch Sequence Diagram:** Step-by-step trace of push-on-main → toggle-to-alt → push-on-alt → toggle-to-main showing the pointer swap at each stage
- **Encoding Pipeline Flowchart:** From `on_key_input()` through `screen_current_key_encoding_flags()` to `encode_glfw_key_event()` to child PTY output
- **Stack Exhaustion Diagram:** Shows the memmove eviction when the 8th slot is occupied and a 9th push arrives
- **VT Parser Routing Diagram:** Shows how `CSI > u`, `CSI < u`, `CSI = u`, `CSI ? u` are dispatched from the VT parser to their respective handler functions


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kitty/screen.h`, `kitty/screen.c`, `kitty/keys.c`, `kitty/key_encoding.c`, `kitty/vt-parser.c`, `kitty/modes.h`, `kitty/data-types.h`, `kitty_tests/screen.py`, `kitty_tests/keys.py`, `kitty_tests/__init__.py`, `docs/keyboard-protocol.rst` | Comprehensive technical Q&A document covering keyboard protocol stack behavior during alternate screen buffer switches, stack isolation proof, byte-sequence evidence, stack exhaustion semantics, edge cases, and test validation. Full content with Mermaid diagrams, code citations, and byte-sequence tables. |

This is the sole documentation file to be created. No existing documentation files require modification, as the project rules explicitly state: "Do not modify any existing files in the source repository."

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical Deep-Dive / Q&A Investigation Document
Source Code References:
    - kitty/screen.h:88-170 (Screen struct with dual key_encoding_flags arrays)
    - kitty/screen.c:150 (Initialization: pointer set to main array)
    - kitty/screen.c:161-174 (screen_reset: memset both arrays to 0)
    - kitty/screen.c:1068-1095 (screen_toggle_screen_buffer: pointer swap)
    - kitty/screen.c:1204-1208 (screen_current_key_encoding_flags: backward scan for 0x80)
    - kitty/screen.c:1212-1217 (screen_report_key_encoding_flags: CSI ?{flags}u response)
    - kitty/screen.c:1220-1231 (screen_set_key_encoding_flags: replace/OR/AND-NOT modes)
    - kitty/screen.c:1234-1244 (screen_push_key_encoding_flags: push with memmove eviction)
    - kitty/screen.c:1248-1253 (screen_pop_key_encoding_flags: backward pop)
    - kitty/keys.c:166-273 (on_key_input: dispatch pipeline)
    - kitty/keys.c:251 (encode call with screen_current_key_encoding_flags)
    - kitty/key_encoding.c:413-440 (encode_glfw_key_event: flag-to-field mapping)
    - kitty/key_encoding.c:367-397 (encode_key: encoding decision logic)
    - kitty/key_encoding.c:64-98 (serialize: CSI sequence builder)
    - kitty/key_encoding.c:148-233 (encode_function_key: function key encoding)
    - kitty/vt-parser.c:1217-1241 (CSI u handler routing)
    - kitty/modes.h:75-77 (TOGGLE_ALT_SCREEN_1/2, ALTERNATE_SCREEN constants)
    - kitty/data-types.h:42 (arraysz macro)
    - kitty_tests/screen.py:952-993 (test_key_encoding_flags_stack)
    - kitty_tests/keys.py:417-468 (progressive enhancement flag tests)
    - kitty_tests/__init__.py:33-37 (parse_bytes helper)
    - kitty_tests/__init__.py:237-241 (create_screen helper)
    - docs/keyboard-protocol.rst:293-309 (push/pop stack spec with buffer isolation note)
Sections:
    - Introduction and Executive Summary
    - Data Structure Analysis (Screen struct, sentinel bit, stack depth)
    - Stack Operations Deep Dive (push, pop, set, query, exhaustion)
    - Buffer Switch Mechanics (pointer swap, mode constants, VT routing)
    - Stack Isolation Proof (code evidence, round-trip walkthrough, cross-buffer independence)
    - Byte Sequence Evidence (encoding pipeline, flag mapping, Ctrl+Shift+A scenarios)
    - Test Validation (existing test analysis, execution results)
    - Edge Cases and Mode Interactions (reset, DECCKM, empty pop, rapid switching)
    - Conclusion
Diagrams:
    - Screen struct data structure layout (Mermaid)
    - Buffer switch pointer-swap sequence (Mermaid)
    - Key encoding pipeline flowchart (Mermaid)
    - Stack exhaustion eviction diagram (Mermaid)
    - VT parser CSI u routing diagram (Mermaid)
Key Citations: kitty/screen.c, kitty/screen.h, kitty/key_encoding.c, kitty/keys.c,
    kitty/vt-parser.c, kitty/modes.h, kitty_tests/screen.py, kitty_tests/keys.py,
    docs/keyboard-protocol.rst
```

### 0.5.3 Documentation Files to Update Detail

No existing documentation files are to be updated. The project implementation rules require:
- "Do not modify any existing files in the source repository."
- All output is confined to the new `blitzy/documentation/kitty_815df1e210e0.md` file.

### 0.5.4 Documentation Configuration Updates

No documentation configuration updates are required. The new file is placed in `blitzy/documentation/` which is an independent output directory not integrated into the Sphinx documentation build system. The existing `docs/conf.py`, `docs/Makefile`, and `docs/requirements.txt` remain unmodified.

### 0.5.5 Cross-Documentation Dependencies

- **Shared reference:** The output document will cite and reference `docs/keyboard-protocol.rst` as the canonical protocol specification but will not link to it via navigation or ToC changes.
- **No navigation updates:** Since the output document is in `blitzy/documentation/` (a separate directory from `docs/`), no Sphinx navigation or index updates are needed.
- **No glossary updates:** The document will define terms inline as needed (e.g., "sentinel bit", "progressive enhancement flags").


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following tools and packages are relevant to this documentation exercise. Since the output is a standalone Markdown document (not built through the Sphinx pipeline), the critical dependencies are the project's build and test infrastructure needed to validate behaviors described in the document.

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| PyPI | sphinx | (unpinned) | Project's documentation site generator — used for `docs/` build; not directly needed for this task |
| PyPI | furo | (unpinned) | Sphinx theme for the project's documentation site |
| PyPI | sphinx-copybutton | (unpinned) | Copy button extension for code blocks in docs |
| PyPI | sphinxext-opengraph | (unpinned) | Open Graph metadata for docs pages |
| PyPI | sphinx-inline-tabs | (unpinned) | Inline tab extension for docs |
| PyPI | sphinx-autobuild | (unpinned) | Live-reload documentation preview |
| System | Python | >=3.8 | Runtime for project — required to run test infrastructure (`kitty_tests/`) |
| System | C compiler (gcc/clang) | — | Required to build native extensions (`kitty/screen.c`, `kitty/keys.c`, `kitty/key_encoding.c`) |
| System | OpenGL / GLFW | — | Required for full kitty build but not for Screen-object-level testing |
| Go module | `github.com/kovidgoyal/kitty` | — | Go-based tooling and CLI utilities |

**Note:** The documentation versions listed as "(unpinned)" reflect the actual state of `docs/requirements.txt`, which does not pin specific versions. This is the canonical source for documentation dependencies.

### 0.6.2 Documentation Reference Updates

Not applicable. Since this task creates a single standalone Markdown file in `blitzy/documentation/` and does not modify any existing files, no link updates are required within the existing documentation tree. The new document is self-contained with all internal references resolved via file-path citations (e.g., `Source: kitty/screen.c:1234`).


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (for the specific topic of keyboard stack + buffer interaction):**

| Documentation Area | Current State | Target |
|--------------------|---------------|--------|
| Keyboard protocol stack semantics (push/pop/set/query) | Covered at protocol level in `docs/keyboard-protocol.rst` | 100% — extend with implementation detail |
| Stack isolation between main/alt buffers | One paragraph in `docs/keyboard-protocol.rst` (lines 300–309) | 100% — full code-level proof with worked examples |
| Stack data structure internals (0x80 sentinel, array layout) | 0% — completely undocumented | 100% — comprehensive walkthrough |
| Byte-sequence output under each flag combination | 0% — exists only in test assertions | 100% — consolidated reference table |
| Stack exhaustion/eviction behavior | One sentence in `docs/keyboard-protocol.rst` (line 302–303) | 100% — code walkthrough with examples |
| Cross-buffer exhaustion independence | 0% — not addressed | 100% — proof with code evidence |
| DECCKM interaction with keyboard flags | 0% — undocumented | 100% — documented interaction |
| screen_reset() cross-buffer clear behavior | 0% — undocumented | 100% — documented with code citation |
| VT parser routing for CSI u variants | 0% — undocumented at dispatch level | 100% — complete routing documentation |
| Test-based validation of behaviors | 0% — tests exist but are not documented externally | 100% — test walkthrough included |

**Overall target coverage:** 100% of all user questions and inferred needs addressed in the output document.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question (stack isolation, round-trip behavior, byte sequences, exhaustion, edge cases) has a dedicated section with a definitive answer
- All answers include rationale/thinking as required by the implementation rules
- All claims are backed by specific file paths and line numbers from the codebase
- The 0x80 sentinel bit mechanism is fully explained with bit-level examples
- The memmove eviction strategy is traced step-by-step

**Accuracy validation:**
- All byte-sequence examples are derived from the `encode_glfw_key_event()` implementation in `kitty/key_encoding.c` and validated against test assertions in `kitty_tests/keys.py`
- Stack operation traces are verified against the `test_key_encoding_flags_stack` test at `kitty_tests/screen.py:952`
- Buffer-switch pointer assignments are verified against the `screen_toggle_screen_buffer()` implementation
- No assumptions are made — all conclusions are grounded in code (per implementation rules)

**Clarity standards:**
- Technical accuracy with progressive disclosure: concept → code evidence → worked example → edge case
- Consistent use of terminology: "main buffer", "alternate buffer", "keyboard flags stack", "progressive enhancement flags", "sentinel bit"
- Byte sequences shown in both hex and readable escape notation
- All Mermaid diagrams are syntactically valid and render correctly

**Maintainability:**
- Source citations use format `Source: <filepath>:<line>` for traceability
- Each section is self-contained enough to be updated independently if the code changes
- Flag-bit-to-behavior mapping table can be updated if new bits are added

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per key scenario:** At least 4 byte-sequence examples for Ctrl+Shift+A (no flags, disambiguate, report-all-keys, report-all-keys + report-text)
- **Diagram types required:** Mermaid flowcharts (encoding pipeline, VT routing), Mermaid sequence diagrams (buffer-switch round-trip), conceptual diagrams (data structure layout, stack eviction)
- **Code example testing:** All byte sequences are cross-referenced against `kitty_tests/keys.py` assertions; stack operations against `kitty_tests/screen.py:952` assertions
- **Visual content freshness:** Diagrams are derived from the current codebase at commit `815df1e21`


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/kitty_815df1e210e0.md` — the sole output artifact

**Source code analyzed (read-only) for documentation content:**
- `kitty/screen.h` — Screen struct definition with `main_key_encoding_flags[8]`, `alt_key_encoding_flags[8]`, and `*key_encoding_flags`
- `kitty/screen.c` — All keyboard stack functions (`screen_push_key_encoding_flags`, `screen_pop_key_encoding_flags`, `screen_set_key_encoding_flags`, `screen_current_key_encoding_flags`, `screen_report_key_encoding_flags`), `screen_toggle_screen_buffer`, `screen_reset`
- `kitty/keys.c` — `on_key_input()` dispatch pipeline, `screen_current_key_encoding_flags()` usage
- `kitty/keys.h` — `encode_glfw_key_event()` signature, `KEY_BUFFER_SIZE`
- `kitty/key_encoding.c` — Complete encoding engine: `encode_glfw_key_event()`, `encode_key()`, `encode_function_key()`, `serialize()`, flag-to-field mapping
- `kitty/vt-parser.c` — CSI `u` handler routing at lines 1217–1241
- `kitty/modes.h` — `TOGGLE_ALT_SCREEN_1`, `TOGGLE_ALT_SCREEN_2`, `ALTERNATE_SCREEN`, `DECCKM` constants
- `kitty/data-types.h` — `arraysz()` macro
- `kitty/key_encoding.py` — Python key encoding utilities and functional key number maps

**Test files analyzed (read-only) for validation evidence:**
- `kitty_tests/screen.py` — `test_key_encoding_flags_stack()` at lines 952–993
- `kitty_tests/keys.py` — `test_encode_key_event()` with all flag-combination scenarios
- `kitty_tests/__init__.py` — `parse_bytes()`, `Callbacks`, `BaseTest.create_screen()`

**Existing documentation analyzed (read-only) for context:**
- `docs/keyboard-protocol.rst` — canonical protocol specification, push/pop/stack/buffer-isolation semantics
- `docs/requirements.txt` — documentation tooling dependencies

**Topics covered in the output document:**
- Keyboard progressive enhancement flag stack data structures
- Stack push, pop, set, and query operations with implementation detail
- Alternate screen buffer switching and its effect on the active keyboard stack
- Stack isolation proof between main and alternate buffers
- Round-trip state preservation across buffer switches
- Byte-sequence output under different flag combinations for specific key presses
- Stack exhaustion behavior (8-entry limit, memmove eviction)
- Cross-buffer exhaustion independence
- Edge cases: screen_reset, DECCKM interaction, empty-stack pop, rapid buffer switching
- VT parser escape sequence routing for CSI u variants
- Test infrastructure analysis and validation

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — The user explicitly prohibits modifying any source files. No changes to `kitty/`, `kitty_tests/`, `docs/`, or any other directory.
- **New test file creation** — No new test files will be written; analysis relies on existing tests and code walkthroughs.
- **Existing documentation updates** — No changes to `docs/keyboard-protocol.rst` or any other `.rst` file.
- **Sphinx documentation build configuration** — No changes to `docs/conf.py`, `docs/Makefile`, or `docs/requirements.txt`.
- **Feature additions or code refactoring** — No code changes of any kind.
- **Deployment or build configuration changes** — No changes to `Makefile`, `setup.py`, `pyproject.toml`, or CI configuration.
- **Non-keyboard terminal features** — Graphics protocol, clipboard, file transfer, shell integration, and all other terminal features are out of scope.
- **Mouse input handling** — Mouse tracking modes and protocols are not covered.
- **Font rendering, GPU pipeline, or window management** — Unrelated subsystems excluded.
- **Go tooling code** — The Go-based tools in `tools/` are not relevant to keyboard protocol internals.
- **Third-party dependencies** — No changes to `glfw/`, `3rdparty/`, `glad/`, or vendored code.


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, not integrated into the Sphinx build.
- **Documentation preview command:** Any Markdown renderer/viewer can preview `blitzy/documentation/kitty_815df1e210e0.md`. For example: `python -m markdown blitzy/documentation/kitty_815df1e210e0.md` or simply view in a Markdown-capable editor/viewer.
- **Diagram generation command:** Mermaid diagrams are embedded inline within the Markdown document using fenced code blocks (` ```mermaid ... ``` `). They render natively in GitHub, GitLab, and compatible Markdown viewers. No external generation step is required.
- **Default format:** Markdown with Mermaid diagrams.
- **Citation requirement:** Every section must reference source files using the format `Source: <filepath>:<line>` or `Source: <filepath>:<start_line>-<end_line>`.
- **Style guide:** Progressive disclosure structure — concepts first, then code evidence, then worked examples, then edge cases. All byte sequences shown in dual format (hex and readable escape notation). Tables for structured data. Mermaid diagrams for architectural and flow relationships.
- **Documentation validation:** Manual review of all code citations against the codebase. Cross-reference all byte-sequence claims against the `kitty_tests/keys.py` test assertions. Validate all stack-operation claims against `kitty_tests/screen.py:952-993`.

### 0.9.2 Output File Specification

- **File path:** `blitzy/documentation/kitty_815df1e210e0.md`
- **File name derivation:** Per the SWE-AtlasQnA-Repo implementation rule, the file is named `<source_branch_name>.md` where the source branch name is `kitty_815df1e210e0`
- **Directory:** `blitzy/documentation/` — created if it does not exist
- **Format:** GitHub Flavored Markdown (GFM) with Mermaid fenced code blocks
- **Encoding:** UTF-8


## 0.10 Rules for Documentation

The following rules and constraints are explicitly emphasized by the user and by the project's implementation rules, and must be strictly followed during documentation generation:

- **Do not modify any existing files in the source repository.** All analysis is read-only. The only file created is `blitzy/documentation/kitty_815df1e210e0.md`. No changes to `kitty/`, `kitty_tests/`, `docs/`, or any other existing directory or file.

- **Base all answers on the code as truth.** Do not make assumptions about behavior. Every claim must be traceable to a specific file and line number in the codebase. If the code contradicts the protocol documentation, the code is authoritative.

- **Provide thinking and rationale behind all answers.** Each conclusion must be accompanied by the reasoning chain: which code was examined, what it does, and why that leads to the stated conclusion.

- **Focus on actual runtime behavior, not just theoretical analysis.** Where existing test infrastructure supports validation (e.g., `test_key_encoding_flags_stack`, `test_encode_key_event`), reference and explain the test results. Where runtime testing is constrained by the environment (no GUI, no GLFW), provide detailed code-level walkthroughs that trace exact execution paths.

- **Use existing test infrastructure and utilities.** Reference `parse_bytes()`, `Callbacks`, `create_screen()`, and existing test methods to demonstrate behaviors. Do not create new test files.

- **Create the output document as `kitty_815df1e210e0.md`** in the `blitzy/documentation/` directory, following the naming convention `<source_branch_name>.md` specified by the SWE-AtlasQnA-Repo implementation rule.

- **Include concrete byte-sequence evidence.** Show the actual bytes that would be sent to the child process for specific key presses under specific flag combinations. Use both hex notation and human-readable escape notation.

- **Document edge cases comprehensively.** Cover stack exhaustion, empty-stack pop, screen_reset cross-buffer behavior, DECCKM interaction, and rapid buffer switching.

- **All source code citations must use exact file paths and line numbers** from the repository to enable traceability and verification.


## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were searched across the codebase to derive all conclusions in this Agent Action Plan:

**Root-level exploration:**
- Repository root (`""`) — full children listing to understand project structure

**Core source files (read and analyzed):**

| File Path | Lines Read | Purpose in Analysis |
|-----------|-----------|---------------------|
| `kitty/screen.h` | 1–290 (complete) | Screen struct definition revealing `main_key_encoding_flags[8]`, `alt_key_encoding_flags[8]`, `*key_encoding_flags` pointer at line 128; function declarations for all keyboard stack operations |
| `kitty/screen.c` | 140–180, 1060–1110, 1195–1260, 4445–4460 | Initialization (line 150), `screen_reset()` clearing both arrays (lines 173–174), `screen_toggle_screen_buffer()` pointer swap (lines 1068–1095), all keyboard stack functions (lines 1204–1253), `toggle_alt_screen` Python binding (line 4449) |
| `kitty/keys.c` | 1–543 (complete) | `on_key_input()` dispatch pipeline (lines 166–273), `screen_current_key_encoding_flags()` usage at line 251, `encode_key_for_tty` Python wrapper (lines 311–322) |
| `kitty/keys.h` | 1–22 (complete) | `encode_glfw_key_event()` signature, `KEY_BUFFER_SIZE` constant (128) |
| `kitty/key_encoding.c` | 1–440 (complete) | Full encoding engine: flag-to-field mapping (lines 419–423), `encode_key()` (lines 367–397), `serialize()` (lines 64–98), `encode_function_key()` (lines 148–233), legacy encoding paths |
| `kitty/modes.h` | 1–90 (complete) | Terminal mode constants: `TOGGLE_ALT_SCREEN_1` (47<<5), `TOGGLE_ALT_SCREEN_2` (1047<<5), `ALTERNATE_SCREEN` (1049<<5), `DECCKM` (1<<5) |
| `kitty/vt-parser.c` | 1210–1250 | CSI `u` handler routing: query (`?`), set (`=`), push (`>`), pop (`<`) dispatched to corresponding `screen_*` functions (lines 1217–1241) |
| `kitty/data-types.h` | Line 42 | `arraysz()` macro definition used for stack size calculation |
| `kitty/key_encoding.py` | 1–80 | Python-side key encoding: functional key number-to-name map, `KeyEvent` type |
| `kitty/keys.py` | 1–50 | Python key dispatch: `keyboard_mode_name()` function using `screen.current_key_encoding_flags()` |
| `kitty/fast_data_types.pyi` | Line 1146 | Type stub confirming `current_key_encoding_flags()` return type |

**Test files (read and analyzed):**

| File Path | Lines Read | Purpose in Analysis |
|-----------|-----------|---------------------|
| `kitty_tests/screen.py` | 952–1040 | `test_key_encoding_flags_stack()` — validates push, pop, set (with OR and AND-NOT modes), reset, and stack overflow (8+ pushes) |
| `kitty_tests/keys.py` | 1–653 (complete) | `test_encode_key_event()` — comprehensive tests for legacy encoding, disambiguate mode (flag 0b1), event type reporting (flag 0b10), alternate key reporting (flag 0b100), report-all-keys (flag 0b1000), embed-text (flag 0b10000), and round-trip validation |
| `kitty_tests/__init__.py` | 1–80, 237–257 | `parse_bytes()` helper function, `Callbacks` class with `wtcbuf` buffer, `BaseTest.create_screen()` factory |

**Documentation files (read and analyzed):**

| File Path | Lines Read | Purpose in Analysis |
|-----------|-----------|---------------------|
| `docs/keyboard-protocol.rst` | Lines 60–75, 260–320 (grep-targeted) | Protocol specification: push/pop escape sequences, stack size limit, buffer isolation requirement ("Terminals must maintain separate stacks for the main and alternate screens"), eviction rule |
| `docs/requirements.txt` | Complete | Documentation tooling dependencies: sphinx, furo, sphinx-copybutton, sphinxext-opengraph, sphinx-inline-tabs, sphinx-autobuild |
| `docs/` folder | Folder listing | Identified `keyboard-protocol.rst` as the primary relevant doc file; surveyed all `.rst` files for keyboard/protocol/key references |

**Configuration and metadata files:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `pyproject.toml` | Python version requirement: `>=3.8` |
| `setup.py` | Build system, version extraction, interpreter validation |
| `key_encoding.json` | Static key-name-to-code lookup table referenced by input stack |

**Folders explored:**
- `""` (root) — project structure overview
- `kitty/` — core application source tree
- `kitty_tests/` — test suite
- `docs/` — documentation tree

### 0.11.2 Tech Spec Sections Retrieved

| Section Heading | Purpose |
|----------------|---------|
| 4.3 TERMINAL INPUT/OUTPUT PIPELINE | Understanding of input processing flow, VT parser dispatch matrix, keyboard protocol dispatch path |
| 4.5 WINDOW AND TAB LIFECYCLE | Window/screen creation flow, child process environment, understanding of Screen object provisioning |

### 0.11.3 Attachments

No attachments were provided by the user for this project.

### 0.11.4 External URLs

No Figma screens or external URLs were provided by the user for this project.


