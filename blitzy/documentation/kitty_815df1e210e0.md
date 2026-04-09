# Kitty OSC 133 Shell Integration: Escape Sequence Handling Investigation

| Field | Value |
|-------|-------|
| **Terminal** | kitty |
| **Version** | 0.35.2 (`kitty/constants.py:25` → `version: Version(0, 35, 2)`) |
| **Commit** | `815df1e21` |
| **Date** | 2025 |
| **Scope** | OSC 133 escape sequence consumption, byte-level analysis, exit code processing, edge-case handling |

---

## Overview

This document reports the findings of a technical investigation into how the kitty terminal emulator (v0.35.2) handles OSC 133 shell integration escape sequences. The investigation addresses five specific requirements:

| Req | Description |
|-----|-------------|
| **R1** | Whether OSC 133 sequences appear in visible screen output after parsing, the total byte length of a representative input sequence, and the byte offset at which `D;42` appears within the raw stream |
| **R2** | Comparative byte-length and `D;{code}` marker position analysis across exit codes `0`, `1`, and `127` |
| **R3** | Runtime proof that exit code `99` is processed through the entire code path — from VT parser dispatch through C-layer `shell_prompt_marking()` through to the Python callback layer |
| **R4** | Behavior when invalid exit codes (`not_a_number`, empty string) are sent, covering both the test `Callbacks` class and the production `Window.handle_cmd_end()` behavior |
| **R5** | No-modification constraint: no existing repository files were modified during the investigation |

### Methodology

- **Code-as-truth approach:** Every claim in this document cites specific source file paths and line numbers verified against the repository at commit `815df1e21`.
- **No assumptions:** All answers are derived from reading the actual source code and, where possible, verified through compiled test execution.
- **Build and test:** The investigation was performed by building kitty from source (`python3 setup.py build --debug`) and running ephemeral test scripts against the compiled `fast_data_types.so` C extension.
- **No-modification compliance (R5):** All test scripts were created in `/tmp/`, executed against the compiled modules, and deleted afterwards. No existing repository files were modified during this investigation.
- **Version confirmation:** `kitty/constants.py:25` confirms `version: Version(0, 35, 2)`.

---

## OSC 133 Code Path Architecture

The kitty terminal emulator processes OSC 133 shell integration escape sequences through a three-layer architecture spanning two programming languages (C and Python):

1. **Layer 1 — C VT Parser** (`kitty/vt-parser.c`): Parses raw byte input, identifies OSC escape sequences, extracts the OSC code (133), and dispatches to the screen model.
2. **Layer 2 — C Screen Model** (`kitty/screen.c`): Interprets the OSC 133 marker type (A, C, or D), updates internal screen state (line attributes, prompt kind), and invokes Python callbacks via the `CALLBACK` macro.
3. **Layer 3 — Python Callback Layer** (`kitty/window.py` in production; `kitty_tests/__init__.py` in tests): Receives the parsed marker data and updates application-level state (exit status, command line, timing).

### Sequence Diagram: OSC 133 Processing Pipeline

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

### Layer 1: VT Parser Dispatch (`kitty/vt-parser.c`)

The `dispatch_osc()` function (Source: `kitty/vt-parser.c:457`) is the entry point for all OSC (Operating System Command) escape sequences. It:

1. **Parses the OSC code** from the initial digits of the buffer (lines 466–475). For OSC 133, the first three bytes of the buffer payload are `1`, `3`, `3`, yielding code `133`. The variable `i` advances past the digits and the `;` separator, pointing to the payload after `133;`.
2. **Routes to case 133** (lines 536–546):
   ```c
   case 133:
       // #ifdef DUMP_COMMANDS block (lines 537-540) is debug-only
       if (limit > i) {
           buf[limit] = 0; // safe: 8 extra bytes after PARSER_BUF_SZ
           shell_prompt_marking(self->screen, (char*)buf + i);
       }
       break;
   ```
3. **Key details:**
   - The `buf[limit] = 0` null-termination is safe because the parser buffer has 8 extra bytes allocated beyond `PARSER_BUF_SZ` (Source: `kitty/vt-parser.c:543`).
   - The `if (limit > i)` guard ensures empty payloads (where the OSC contained only `133;` with nothing after the semicolon) are not forwarded to `shell_prompt_marking()`.
   - The `#ifdef DUMP_COMMANDS` block (lines 537–540) is active only in debug builds and has no effect on production behavior.
   - **The OSC bytes are never written to the screen's character cells.** They are consumed entirely within the parser dispatch layer.

### Layer 2: Screen Model Handling (`kitty/screen.c`)

The `shell_prompt_marking()` function (Source: `kitty/screen.c:2328`) receives a `char *buf` pointing to the payload after `133;` (e.g., `"D;42"` for an exit code marker). Its function prototype is declared at `kitty/screen.h:231`.

**Guard condition:** `if (self->cursor->y < self->lines)` ensures the cursor is within the visible screen area before processing any marker (Source: `kitty/screen.c:2329`).

**Switch on `buf[0]`** (the marker character):

#### Case 'A' — Prompt Start (lines 2332–2339)

```c
case 'A': {
    PromptKind pk = PROMPT_START;
    self->prompt_settings.redraws_prompts_at_all = 1;
    self->prompt_settings.uses_special_keys_for_cursor_movement = 0;
    parse_prompt_mark(self, buf+1, &pk);
    self->linebuf->line_attrs[self->cursor->y].prompt_kind = pk;
    if (pk == PROMPT_START) CALLBACK("cmd_output_marking", "O", Py_False);
} break;
```

- Initializes `pk` to `PROMPT_START` (value `1` from the `PromptKind` enum).
- Calls `parse_prompt_mark()` (Source: `kitty/screen.c:2316`) to tokenize additional semicolon-separated parameters: `k=s` (sets `pk` to `SECONDARY_PROMPT`), `redraw=0`, `special_key=1`.
- Sets the current line's `prompt_kind` attribute in the line buffer.
- Invokes the Python callback with `is_start=False` (i.e., `Py_False`) only when `pk == PROMPT_START` (not for secondary prompts).

#### Case 'C' — Command Output Start (lines 2340–2348)

```c
case 'C': {
    self->linebuf->line_attrs[self->cursor->y].prompt_kind = OUTPUT_START;
    const char *cmdline = "";
    if (strstr(buf + 1, ";cmdline") == buf + 1) {
        cmdline = buf + 2;
    }
    RAII_PyObject(c, PyUnicode_DecodeUTF8(cmdline, strlen(cmdline), "replace"));
    if (c) { CALLBACK("cmd_output_marking", "OO", Py_True, c); }
    else PyErr_Print();
} break;
```

- Sets the current line's `prompt_kind` to `OUTPUT_START` (value `3`).
- Extracts the `cmdline` parameter: if the buffer after `C` starts with `;cmdline`, the cmdline string is `buf + 2` (e.g., `;cmdline=ls` → cmdline becomes `cmdline=ls`). Otherwise, cmdline is an empty string.
- Decodes the cmdline as UTF-8 and invokes the Python callback with `is_start=True` (i.e., `Py_True`).

#### Case 'D' — Command End / Exit Status (lines 2350–2353)

```c
case 'D': {
    const char *exit_status = buf[1] == ';' ? buf + 2 : "";
    CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);
} break;
```

- If `buf[1]` is `;` (as in `"D;42"`), extracts `exit_status = buf + 2` → `"42"`.
- If `buf[1]` is not `;` (as in bare `"D"`), uses an empty string `""` as the exit status.
- Invokes the Python callback with `is_start=None` (i.e., `Py_None`) and the exit status string.
- **No validation** of the exit status string occurs at the C layer — the string is passed through as-is.

#### No Case 'B' — Silent Ignore

The 'B' marker (used in the full iTerm2 OSC 133 protocol for command region start) has **no case** in this switch statement. If `\033]133;B\007` is sent, the switch falls through without matching any case, and the function returns without any state change or callback invocation. See the [Additional Finding: B Marker Handling](#additional-finding-b-marker-handling) section for details.

### Layer 3: Python Callback Layer

#### Production Path (`kitty/window.py`)

**`cmd_output_marking()`** (Source: `kitty/window.py:1453–1461`):

```python
def cmd_output_marking(self, is_start: Optional[bool], cmdline: str = '') -> None:
    if is_start:
        start_time = monotonic()
        self.last_cmd_output_start_time = start_time
        cmdline = decode_cmdline(cmdline) if cmdline else ''
        self.last_cmd_cmdline = cmdline
        self.call_watchers(self.watchers.on_cmd_startstop, {...})
    else:
        self.handle_cmd_end(cmdline)
```

- When `is_start` is `True` (marker 'C'): Records start time, decodes the cmdline via `decode_cmdline()`, stores `last_cmd_cmdline`, and notifies watchers.
- When `is_start` is `None` (marker 'D'): Calls `handle_cmd_end(cmdline)` where the `cmdline` parameter actually holds the exit status string (passed from the C layer's `"Os"` format string).
- When `is_start` is `False` (marker 'A' with `PROMPT_START`): Calls `handle_cmd_end()` with an empty string.

**`handle_cmd_end()`** (Source: `kitty/window.py:1408–1415`):

```python
def handle_cmd_end(self, exit_status: str = '') -> None:
    if self.last_cmd_output_start_time == 0.:
        return
    self.last_cmd_output_start_time = 0.
    try:
        self.last_cmd_exit_status = int(exit_status)
    except Exception:
        self.last_cmd_exit_status = 0
```

- **CRITICAL:** On `int()` failure (invalid string), the production code falls back to `0`. This means every D marker always results in a valid integer exit status in production.

**`decode_cmdline()`** (Source: `kitty/window.py:225–232`):

```python
def decode_cmdline(x: str) -> str:
    ctype, sep, val = x.partition('=')
    if ctype == 'cmdline':
        return next(shlex_split(val, True))
    if ctype == 'cmdline_url':
        from urllib.parse import unquote
        return unquote(val)
    return ''
```

Supports two cmdline encoding formats:
- `cmdline=...` — shell `%q` quoting, decoded via `shlex_split` (used by Bash: `shell-integration/bash/kitty.bash:208`)
- `cmdline_url=...` — URL percent-encoding, decoded via `urllib.parse.unquote` (used by Fish: `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:91`)

#### Test Path (`kitty_tests/__init__.py`)

**`Callbacks.cmd_output_marking()`** (Source: `kitty_tests/__init__.py:71–79`):

```python
def cmd_output_marking(self, is_start: Optional[bool], data: str = '') -> None:
    if is_start:
        self.last_cmd_at = monotonic()
        self.last_cmd_cmdline = decode_cmdline(data) if data else data
    else:
        if self.last_cmd_at != 0:
            self.last_cmd_at = 0
            with suppress(Exception):
                self.last_cmd_exit_status = int(data)
```

- **CRITICAL DIFFERENCE from production:** The test class uses `contextlib.suppress(Exception)` (imported at `kitty_tests/__init__.py:15`), which silently eats any `ValueError` from `int()`. On failure, `last_cmd_exit_status` **remains at its previous value** (the sentinel `sys.maxsize = 9223372036854775807`, initialized at `kitty_tests/__init__.py:48` and reset in `clear()` at line 106) rather than falling to `0`.

### PromptKind Enum (`kitty/data-types.h:230`)

```c
typedef enum {
    UNKNOWN_PROMPT_KIND = 0,
    PROMPT_START        = 1,
    SECONDARY_PROMPT    = 2,
    OUTPUT_START        = 3
} PromptKind;
```

These values are stored in the `prompt_kind` field of `LineAttrs` (a 2-bit bitfield at `kitty/data-types.h:236`), allowing each line in the screen buffer to track whether it is part of a prompt, a secondary prompt, or command output.

### Shell Emission Context

Each supported shell emits OSC 133 markers in its integration script:

**Bash** (`shell-integration/bash/kitty.bash`):
- Line 208: `printf "\e]133;C;cmdline=%q\a" "$last_cmd"` — emits the C marker with the command line encoded using shell `%q` quoting.
- Line 239: PS1 includes `\e]133;D;\$?\a` — emits the D marker with the shell's exit status variable `$?`.

**Zsh** (`shell-integration/zsh/kitty-integration`):
- Line 145: `print -nu $_ksi_fd '\e]133;D;'$cmd_status'\a'` — emits the D marker with the command's exit status.
- Line 149: `print -nu $_ksi_fd '\e]133;D\a'` — emits a bare D marker (no semicolon, no exit status) when no command was tracked.

**Fish** (`shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`):
- Line 91: `printf '\e]133;C;cmdline_url=%s\a' (string escape --style=url -- "$argv")` — emits the C marker with URL-encoded cmdline.
- Line 96: `echo -en "\e]133;D;$status\a"` — emits the D marker with Fish's `$status` variable.

---

## Investigation 1: OSC 133 Sequence Consumption & Byte Analysis

### Question (R1)

Do OSC 133 escape sequences appear in the terminal's visible screen output after parsing? What is the total byte length of a representative input sequence, and at what byte offset does the `D;42` marker appear?

### Test Setup

A test script creates a `Screen` and `Callbacks` instance (using the `parse_bytes()` helper from `kitty_tests/__init__.py:30–36`), then feeds a composite byte sequence containing four distinct segments:

1. `\033]133;A\007` — prompt start marker (OSC 133;A)
2. `Hello World` — visible text content
3. `\033]133;C;cmdline=ls\007` — command output start marker with cmdline parameter (OSC 133;C)
4. `\033]133;D;42\007` — command end marker with exit code 42 (OSC 133;D)

The complete raw byte sequence:

```python
data = b'\033]133;A\007Hello World\033]133;C;cmdline=ls\007\033]133;D;42\007'
```

### Byte-Level Breakdown

| Byte Range | Length | Content | Description |
|------------|--------|---------|-------------|
| `[0..7]` | 8 bytes | `\033]133;A\007` | OSC 133;A prompt start marker |
| `[8..18]` | 11 bytes | `Hello World` | Visible text |
| `[19..37]` | 19 bytes | `\033]133;C;cmdline=ls\007` | OSC 133;C command output start with cmdline |
| `[38..48]` | 11 bytes | `\033]133;D;42\007` | OSC 133;D;42 command end marker |

### Results

- **Total input byte length:** **49 bytes** (verified via `len(data)`)
- **`D;42` byte offset:** **44** (verified via `data.find(b'D;42')`)
  - The `D` character is at byte position 44 within the raw input stream: 8 bytes (A marker) + 11 bytes (text) + 19 bytes (C marker) + 6 bytes (OSC prefix `\033]133;` of the D marker) = 44
- **Visible screen output:** Only **`Hello World`** appears in the screen buffer. No OSC escape bytes are present in the visible output.

### Rationale

OSC 133 sequences are **consumed** by the VT parser and never appear in the screen's character cell buffer. The processing flow:

1. The VT parser's `dispatch_osc()` function (Source: `kitty/vt-parser.c:457`) intercepts each OSC escape sequence (`\033]...BEL`) as the byte stream is read. The parser identifies the sequence boundaries (ESC `]` through BEL) and extracts the numeric code.
2. For code 133, the parser calls `shell_prompt_marking()` (Source: `kitty/vt-parser.c:544`), passing only the payload after `133;`. The raw OSC bytes are never forwarded to the character cell buffer.
3. `shell_prompt_marking()` (Source: `kitty/screen.c:2328`) updates internal state (line attributes, prompt kind) and invokes Python callbacks. It does not write any characters to the screen.
4. The non-OSC text `Hello World` passes through the normal character processing pipeline and is written to screen cells as visible characters.

Therefore: OSC 133 sequences modify internal terminal state (prompt marking, exit codes) but are **invisible** in the screen buffer output.

---

## Investigation 2: Exit Code Variation (0, 1, 127)

### Question (R2)

Run the same test structure with exit codes `0`, `1`, and `127`. Report byte lengths and `D;{code}` marker positions for each. Analyze whether the position of the exit code number shifts.

### Test Setup

The same composite byte sequence structure from Investigation 1, with the exit code in the `D;{code}` marker varied:

```python
# Template (exit_code substituted for each test):
data = b'\033]133;A\007Hello World\033]133;C;cmdline=ls\007\033]133;D;{exit_code}\007'
```

### Comparative Results

| Exit Code | Total Input Bytes | `D;{code}` Offset | Parsed `last_cmd_exit_status` |
|-----------|-------------------|--------------------|-------------------------------|
| `0` | **48** | **44** | `0` |
| `1` | **48** | **44** | `1` |
| `127` | **50** | **44** | `127` |

### Position Shift Analysis

The `D` marker byte offset remains **constant at 44** across all three exit codes. This is because:

1. **The D marker's position depends only on preceding content.** The bytes before the D marker — `\033]133;A\007` (8 bytes), `Hello World` (11 bytes), `\033]133;C;cmdline=ls\007` (19 bytes), and the OSC prefix of the D marker `\033]133;` (6 bytes) — are **identical** across all tests. The `D` character always starts at byte offset `8 + 11 + 19 + 6 = 44`.

2. **Only the total input length varies.** The variable portion is the exit code digit string between `D;` and `\007`:
   - 1-digit codes (`0`, `1`): total = 48 bytes
   - 3-digit codes (`127`): total = 50 bytes
   - Delta: `+len(str(exit_code)) - 1` bytes compared to a 1-digit code

3. **At the C layer**, `shell_prompt_marking()` in `kitty/screen.c:2350–2351` extracts the exit status as `buf + 2` (after the `D;` prefix within the OSC payload). The parsing start point is always at the same relative offset within the OSC payload, regardless of how many digits the exit code has.

### Rationale

The OSC 133;D envelope has a fixed-format prefix:
- `\033]133;D;` = 9 bytes (ESC, `]`, `1`, `3`, `3`, `;`, `D`, `;`) — fixed
- Exit code digits — variable (`len(str(exit_code))` bytes)
- `\007` = 1 byte (BEL terminator) — fixed

The `D` character's absolute position in the input stream is determined entirely by the total length of all preceding content, which does not change when only the exit code varies. The exit code digits are the **only** variable-length component, and they appear **after** the `D` character.

---

## Investigation 3: Exit Code 99 Runtime Evidence

### Question (R3)

Demonstrate runtime evidence that exit code `99` is processed through the entire code path — from VT parser dispatch through C-layer `shell_prompt_marking()` through to the Python callback layer — by showing the before/after state change.

### Test Methodology

1. Create a `Screen` and `Callbacks` instance using the test infrastructure (`kitty_tests/__init__.py`).
2. Record the initial sentinel value of `callbacks.last_cmd_exit_status`.
3. Feed the composite byte sequence including `\033]133;C\007` (to set `last_cmd_at`) followed by `\033]133;D;99\007` (the exit code under test).
4. Record the final value of `callbacks.last_cmd_exit_status`.

### Before/After State

| Property | Before | After |
|----------|--------|-------|
| `callbacks.last_cmd_exit_status` | **9223372036854775807** (`sys.maxsize`) | **99** |

- The sentinel value `9223372036854775807` is `sys.maxsize` on a 64-bit system, set during `Callbacks.__init__()` (Source: `kitty_tests/__init__.py:48`) and `clear()` (Source: `kitty_tests/__init__.py:106`).
- After processing the `D;99` marker, the value changes to exactly `99`.

### Full Code Path Trace

The following trace shows exit code `99` traversing all three layers:

**Step 1 — Byte Stream Input:**
Input bytes `\033]133;D;99\007` enter the VT parser's read buffer.

**Step 2 — VT Parser Dispatch** (Source: `kitty/vt-parser.c:457`):
`dispatch_osc()` parses the initial digits of the buffer, extracting OSC code = `133`. The variable `i` advances past `133;`, pointing to the remaining payload `D;99`.

**Step 3 — Case 133 Routing** (Source: `kitty/vt-parser.c:536–545`):
The `case 133:` handler null-terminates the buffer at `buf[limit] = 0` and calls:
```c
shell_prompt_marking(self->screen, (char*)buf + i);
// buf + i points to "D;99\0"
```

**Step 4 — Screen Model Entry** (Source: `kitty/screen.c:2328`):
`shell_prompt_marking()` reads `ch = buf[0]` → `'D'`.

**Step 5 — Case 'D' Processing** (Source: `kitty/screen.c:2350–2353`):
```c
case 'D': {
    const char *exit_status = buf[1] == ';' ? buf + 2 : "";
    // buf[1] == ';' → true, so exit_status = "99"
    CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);
} break;
```
The `CALLBACK` macro invokes the Python callback with `is_start = None` (from `Py_None`) and `exit_status = "99"` (from the `"Os"` format: `O` = object, `s` = C string).

**Step 6 — Python Callback (Test Path)** (Source: `kitty_tests/__init__.py:71–79`):
```python
def cmd_output_marking(self, is_start, data=''):
    if is_start:   # is_start is None (falsy) → skip
        ...
    else:
        if self.last_cmd_at != 0:  # True (set by prior C marker)
            self.last_cmd_at = 0
            with suppress(Exception):
                self.last_cmd_exit_status = int(data)
                # int("99") = 99, no exception
                # last_cmd_exit_status is now 99
```

**Step 7 — Production Path Equivalent** (Source: `kitty/window.py:1408–1415`):
In the production `Window` class, `handle_cmd_end("99")` would execute:
```python
try:
    self.last_cmd_exit_status = int("99")  # = 99
except Exception:
    self.last_cmd_exit_status = 0  # not reached
```
Same result: `last_cmd_exit_status = 99`.

### Conclusion

The sentinel-to-99 state transition (`9223372036854775807` → `99`) proves that exit code `99` is processed through the **entire** code path:

1. VT parser byte intake (`dispatch_osc()`)
2. C-layer screen model (`shell_prompt_marking()`, case 'D')
3. Python callback layer (`cmd_output_marking()` → `int("99")` → state storage)

Each layer is necessary: the VT parser extracts the OSC payload, the screen model parses the marker type and exit status string, and the Python callback converts the string to an integer and stores it.

---

## Investigation 4: Invalid Exit Codes

### Question (R4)

What values are recorded when `OSC 133;D;not_a_number` (non-numeric) and `OSC 133;D;` (empty string after semicolon) are sent? Cover both the test `Callbacks` behavior and the production `Window.handle_cmd_end()` behavior.

### Case 1: Non-Numeric Exit Code (`not_a_number`)

**Input:** `\033]133;D;not_a_number\007`

**C Layer** (Source: `kitty/screen.c:2350–2352`):
`shell_prompt_marking()` extracts `exit_status = "not_a_number"` (`buf[1] == ';'` is true, so `exit_status = buf + 2`). No validation occurs at the C layer — the string is passed through to the Python callback as-is.

**Test Callbacks behavior** (Source: `kitty_tests/__init__.py:78–79`):
```python
with suppress(Exception):
    self.last_cmd_exit_status = int("not_a_number")
```
- `int("not_a_number")` raises `ValueError`.
- `contextlib.suppress(Exception)` catches the exception silently.
- `last_cmd_exit_status` **remains at its previous value** — the sentinel `9223372036854775807` (`sys.maxsize`) if no prior valid D marker was processed.

**Production Window behavior** (Source: `kitty/window.py:1412–1415`):
```python
try:
    self.last_cmd_exit_status = int("not_a_number")
except Exception:
    self.last_cmd_exit_status = 0
```
- `int("not_a_number")` raises `ValueError`.
- The `except Exception` block catches it and explicitly sets `last_cmd_exit_status` to **`0`**.

**Key difference:** Test leaves the sentinel unchanged; production falls to `0`.

### Case 2: Empty Exit Code (Empty String After Semicolon)

**Input:** `\033]133;D;\007`

**C Layer** (Source: `kitty/screen.c:2351`):
`buf[1] == ';'` is true, so `exit_status = buf + 2` → `""` (points to the null terminator set by the VT parser at `kitty/vt-parser.c:543`).

**Test Callbacks behavior:**
- `int("")` raises `ValueError`.
- `suppress(Exception)` catches it silently.
- `last_cmd_exit_status` **remains at sentinel** (`sys.maxsize`).

**Production Window behavior:**
- `int("")` raises `ValueError`.
- `except Exception: self.last_cmd_exit_status = 0`
- `last_cmd_exit_status` is **set to `0`**.

### Case 3: No Semicolon at All (Bare `D` Marker)

**Input:** `\033]133;D\007`

**C Layer** (Source: `kitty/screen.c:2351`):
`buf[1] == ';'` evaluates to false — `buf[1]` is the null terminator (set at `kitty/vt-parser.c:543`), not `;`. Therefore `exit_status = ""` (the fallback empty string literal).

The behavior is identical to Case 2: `int("")` raises `ValueError`, and the test vs. production divergence applies.

> **Note:** This is the format emitted by Zsh's `shell-integration/zsh/kitty-integration:149` when no command status is available: `print -nu $_ksi_fd '\e]133;D\a'`.

### Summary Table

| Input | C Layer `exit_status` | Test Callbacks Result | Production Window Result |
|-------|----------------------|-----------------------|--------------------------|
| `D;not_a_number` | `"not_a_number"` | Stays at sentinel (`sys.maxsize` = `9223372036854775807`) | Falls to **`0`** |
| `D;` (empty after `;`) | `""` | Stays at sentinel (`sys.maxsize` = `9223372036854775807`) | Falls to **`0`** |
| `D` (no semicolon) | `""` | Stays at sentinel (`sys.maxsize` = `9223372036854775807`) | Falls to **`0`** |

### Rationale for the Difference

The divergence between test and production stems from two different Python error-handling patterns:

- **Test `Callbacks`** (Source: `kitty_tests/__init__.py:78`): Uses `contextlib.suppress(Exception)`, which **silently discards** any exception. The assignment `self.last_cmd_exit_status = int(data)` is never executed when `int()` raises, so the attribute retains its previous value. This is a simpler, test-oriented approach that preserves state for test assertions.

- **Production `Window.handle_cmd_end()`** (Source: `kitty/window.py:1412–1415`): Uses a `try/except Exception` block with an **explicit fallback assignment** to `0`. This ensures that after processing any D marker — valid or invalid — the production code always has a valid integer exit status. This design choice prevents undefined-state scenarios in the UI.

---

## Additional Finding: B Marker Handling

### Background

The full iTerm2 OSC 133 protocol includes four markers:
- **A** — Prompt start
- **B** — Command region start (after the user presses Enter, before the command's output begins)
- **C** — Command output start
- **D** — Command end with exit status

The protocol specification is referenced in `docs/shell-integration.rst:441–443`, which states that markers A, C, and D are "exactly what is needed for shell integration in kitty" and links to the full iTerm2 protocol for reference.

### Finding

Kitty's `shell_prompt_marking()` switch statement (Source: `kitty/screen.c:2331–2354`) only handles markers **A**, **C**, and **D**. There is **no case for B**.

If `\033]133;B\007` is sent to kitty:
1. The VT parser dispatches it to `shell_prompt_marking()` with `buf = "B"`.
2. The switch on `buf[0]` does not match any case (`'A'`, `'C'`, or `'D'`).
3. The function returns without any state change, callback invocation, or error.
4. The sequence is **silently ignored**.

### Analysis

This is an **intentional subset implementation**, not a bug. Kitty's shell integration uses a three-marker model (A → C → D) that maps directly to the prompt lifecycle:

| Marker | Kitty Usage | iTerm2 Full Protocol |
|--------|-------------|----------------------|
| **A** | Prompt start — sets line attribute | Same |
| **B** | *Not implemented* — silently ignored | Command region start |
| **C** | Output start — records cmdline, starts timing | Command output start |
| **D** | Command end — records exit status | Same |

The B marker would indicate the boundary between the prompt and the command text (after Enter is pressed). Kitty achieves the same functional separation using the A→C transition, where A marks the prompt and C marks the start of command output. The intermediate B state is unnecessary for kitty's feature set (prompt navigation, command output extraction, exit status tracking).

None of kitty's shell integration scripts (Bash, Zsh, or Fish) emit the B marker, confirming that this is by design.

---

## Source References

| File | Lines | Purpose |
|------|-------|---------|
| `kitty/vt-parser.c` | 457–545 | OSC dispatch logic; `dispatch_osc()` function; case 133 routing to `shell_prompt_marking()` |
| `kitty/screen.c` | 2316–2355 | `parse_prompt_mark()` tokenizer and `shell_prompt_marking()` — A/C/D marker handling |
| `kitty/screen.h` | 231 | Function prototype: `void shell_prompt_marking(Screen *self, char *buf)` |
| `kitty/data-types.h` | 230 | `PromptKind` enum: `UNKNOWN_PROMPT_KIND`, `PROMPT_START`, `SECONDARY_PROMPT`, `OUTPUT_START` |
| `kitty/window.py` | 225–232 | `decode_cmdline()` — cmdline format parsing (`cmdline=` and `cmdline_url=`) |
| `kitty/window.py` | 1408–1415 | `handle_cmd_end()` — exit status parsing with `try/except` fallback to `0` |
| `kitty/window.py` | 1453–1461 | `cmd_output_marking()` — routes A/C/D callbacks to appropriate handlers |
| `kitty_tests/__init__.py` | 30–36 | `parse_bytes()` helper — feeds raw bytes through VT parser for testing |
| `kitty_tests/__init__.py` | 39–108 | `Callbacks` class — test callback implementation with `suppress(Exception)` error handling |
| `kitty_tests/screen.py` | 1056–1130 | `test_prompt_marking()` — existing test coverage for OSC 133 prompt marking features |
| `shell-integration/bash/kitty.bash` | 208, 239 | Bash: `printf "\e]133;C;cmdline=%q\a"` and PS1 `\e]133;D;\$?\a` emission |
| `shell-integration/zsh/kitty-integration` | 145–149 | Zsh: `\e]133;D;$cmd_status\a` and bare `\e]133;D\a` emission |
| `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish` | 83–96 | Fish: `\e]133;C;cmdline_url=%s\a` and `\e]133;D;$status\a` emission |
| `docs/shell-integration.rst` | 415–463 | OSC 133 protocol specification, marker definitions, cmdline encoding formats |
| `kitty/constants.py` | 25 | Version confirmation: `version: Version(0, 35, 2)` |

---

> **Note:** All line numbers correspond to the kitty source repository at commit `815df1e21` (version 0.35.2). No existing repository files were modified during this investigation (R5 compliance). All ephemeral test scripts were created in `/tmp/`, executed against the compiled C extension, and cleaned up afterwards.
