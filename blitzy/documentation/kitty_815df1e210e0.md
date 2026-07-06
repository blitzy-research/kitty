# How kitty parses & handles OSC 133 (shell integration) command-boundary sequences

> Runtime-grounded Q&A — `kitty 0.35.2`, commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

## Scope

This document answers three questions about how kitty parses and handles the **OSC 133** shell-integration ("semantic prompt" / FinalTerm) command-boundary escape sequences — the `A`, `B`, `C`, and `D` markers and the exit code carried by the `D` marker. Every factual claim below is grounded in **runtime output captured from kitty's own compiled C VT parser** (and, for the malformed cases, the real `Window.handle_cmd_end` bytecode), **not** from reading the code alone. Code references are given as `file:line` and name the specific function/method performing each step.

The three questions:

- **Q1 — Baseline capture.** Run a program whose output contains `OSC 133;A`, `OSC 133;B`, `OSC 133;C` (with a `cmdline` parameter), some literal text, and `OSC 133;D;42`. What is captured? Do the OSC sequences remain present? What is the total byte length? At what byte offset does the `D;42` marker appear?
- **Q2 — Exit-code variation.** Repeat with exit codes `0`, `1`, and `127` (and `99` for full-path evidence). Report the byte lengths and offsets, and determine whether the position of the exit-code *number* shifts between single-digit and three-digit codes, and by how much.
- **Q3 — Malformed / edge exit codes.** Send `OSC 133;D;not_a_number` and `OSC 133;D;` (empty). What value is ultimately recorded for each?

**Direct answers, up front.**

- **Q1.** The OSC sequences **remain present** in the raw child byte stream but are **consumed and never drawn** in the rendered screen (only `hello` is drawn). Total length is **56 bytes**; the `133;D` marker begins at offset **46** and the number `42` at offset **52**; the recorded exit status is **42** and the decoded command line is **`foo`**.
- **Q2.** The marker offset (**46**) and the exit-code-number start offset (**52**) are **invariant**; only the total length grows, by **+1 byte per additional digit** (55 → 56 → 57). The number's start does **not** shift. The value `99` is observed as an `int` through the entire handling path.
- **Q3.** Both malformed payloads canonically record **`last_cmd_exit_status = 0`** (via an `int()`-in-`try`, `except → 0`). The test-harness proxy is **non-canonical**: it instead leaves the prior value unchanged. The reported answer is **`0`**.

---

## Method — canonical build, invocation, and entry point

**Canonical build & version.**

```
# Build the C sources into kitty/fast_data_types.so:
python3 setup.py build

# Confirm the artifact under observation:
./kitty/launcher/kitty --version
# -> kitty 0.35.2 created by Kovid Goyal
```

- Interpreter: **Python 3.12.3**.
- **Environment-specific note (labeled clearly as environment-specific — NOT part of the canonical answer).** In this environment the build additionally required the documented `--ignore-compiler-warnings` flag (`CC=gcc-13 python3 setup.py build --ignore-compiler-warnings`), because a newer `wayland-protocols` (1.45) triggers `-Werror=switch` in the **unrelated GLFW Wayland backend** (GUI code). That flag disables `-Werror` only; it does not affect code generation, so the compiled VT parser is **functionally identical**. The canonical/documented command remains plain `python3 setup.py build`. The observations were also reproduced first-hand inside the pinned reference image (Python 3.12.3, `kitty 0.35.2`) with identical deterministic results.

**Real entry point used for observation (and why).**

- The bytes are driven through `parse_bytes(screen, data)` [`kitty_tests/__init__.py:L30`] into a compiled `Screen` created by `create_screen` [`kitty_tests/__init__.py:L237`]. This is the same headless harness kitty's own `parser`, `screen`, and `shell_integration` test modules use. The full GUI application requires a GPU/display and is not runnable headlessly, so the compiled VT parser + `Screen` is the canonical way to exercise OSC 133 parsing without a display.
- The malformed Q3 payloads are additionally driven through the **real** `Window.handle_cmd_end` [`kitty/window.py:L1408`] method object (its actual bytecode), bound to a minimal state carrier that provides only the attributes the method reads (`last_cmd_output_start_time`, `last_cmd_exit_status`, `last_cmd_cmdline`, `watchers.on_cmd_startstop`, `call_watchers`). Options are initialized exactly as the test harness does.
- **Non-canonical paths explicitly NOT used as the source of truth:** the `--dump-commands` debug hook [`kitty/vt-parser.c:L539`, under `#ifdef DUMP_COMMANDS`] and the remote-control emitter `write_osc(133, payload)` [`kitty/client.py:L251`]. They are mentioned only for completeness.

**Probe input (verbatim).** The string terminator `ST` is `ESC \` (`\x1b\\`). The order is `A`, `B`, `C`, text, `D`. A `C` output-start marker **must** precede `D`, or no exit status is recorded — see the guard at [`kitty/window.py:L1409-L1410`]:

```python
b'\x1b]133;A\x1b\\' + b'\x1b]133;B\x1b\\' + b'\x1b]133;C;cmdline=foo\x1b\\' + b'hello' + b'\x1b]133;D;<code>\x1b\\'
```

---

## Q1 — Baseline capture with `D;42`

**Direct answers.**

- **(a) What is captured.** There are two distinct notions of "output". kitty does **not** rewrite the child's stdout, so the OSC 133 sequences **remain present** in the **raw child byte stream** kitty receives on the PTY. In the **rendered screen buffer**, the OSC sequences are **consumed as control data and never drawn** — only the literal text `hello` appears. In addition, the handler decodes the `cmdline` parameter (`last_cmd_cmdline == 'foo'`) and records the exit status (`last_cmd_exit_status == 42`).
- **(b) Do the OSC sequences remain present in the captured output?** In the raw stream: **YES** — `'133;D;42'`, `'133;A'`, and `'133;C;cmdline'` are all found. In the rendered screen: **NO** — they are consumed.
- **(c) Total byte length:** **56** bytes.
- **(d) Byte offset of the `D;42` marker:** the `133;D` substring begins at offset **46**; the exit-code number `42` begins at offset **52**.

**Captured output (complete, unedited):**

```
======================================================================
Q1 - baseline D;42
======================================================================
RAW repr        : b'\x1b]133;A\x1b\\\x1b]133;B\x1b\\\x1b]133;C;cmdline=foo\x1b\\hello\x1b]133;D;42\x1b\\'
total raw length: 56
offset '133;D'  : 46
offset code-num : 52
bytes at codenum: b'42\x1b\\'
OSC present '133;D;42' : True
OSC present '133;A'    : True
OSC present '133;C;cmdline' : True
rendered screen buffer : ['hello', '', '', '', '']
decoded last_cmd_cmdline: 'foo'
proxy last_cmd_exit_status: 42
```

**Causal reasoning (naming the functions).**

- **Dispatch.** OSC `133` payloads reach `case 133:` [`kitty/vt-parser.c:L536`]. The canonical branch null-terminates the buffer (`buf[limit] = 0`, [`kitty/vt-parser.c:L543`]) and calls `shell_prompt_marking(self->screen, (char*)buf + i)` [`kitty/vt-parser.c:L544`].
- **Marker handling.** `shell_prompt_marking` [`kitty/screen.c:L2328`] reads `buf[0]` and switches on it [`kitty/screen.c:L2331`]:
  - `A` → prompt start; fires `CALLBACK("cmd_output_marking", "O", Py_False)` [`kitty/screen.c:L2338`].
  - `C` → output start; if the payload begins with `;cmdline` it slices `cmdline = buf + 2` [`kitty/screen.c:L2343-L2344`], UTF-8 decodes it [`kitty/screen.c:L2346`], and fires `CALLBACK("cmd_output_marking", "OO", Py_True, c)` [`kitty/screen.c:L2347`]. On the Python side, `decode_cmdline` [`kitty/window.py:L225`] turns `cmdline=foo` into `'foo'` — matching the observed `decoded last_cmd_cmdline: 'foo'`.
  - `D` → computes `exit_status = buf[1] == ';' ? buf + 2 : ""` [`kitty/screen.c:L2351`] and fires `CALLBACK("cmd_output_marking", "Os", Py_None, exit_status)` [`kitty/screen.c:L2352`].
- **Why only `hello` is drawn.** The OSC bytes are parsed as an Operating System Command control string and routed to the handler; they are never written into screen cells. Only the literal run `hello` between the `C` and `D` markers lands in the screen buffer — hence `rendered screen buffer : ['hello', '', '', '', '']`.
- **`OSC 133;B` is a no-op in this handler.** There is **no `case 'B'`** in `shell_prompt_marking` (the switch handles only `A`, `C`, `D`). In the FinalTerm protocol `B` denotes command/input start, but kitty's C handler does not act on it — it is silently ignored while still being consumed as an OSC control string (so it never appears on screen).

---

## Q2 — Exit-code sweep (`0`, `1`, `42`, `99`, `127`)

**Direct answer.** The position of the `133;D` marker (offset **46**) and the **start** offset of the exit-code number (offset **52**) are **invariant** across single-digit and three-digit codes. Only the **total byte length grows — by +1 byte per additional digit** (55 → 56 → 57). The number's start offset does **not** shift.

**Captured output (complete, unedited):**

```
======================================================================
Q2 - exit-code sweep
======================================================================
  code  raw_len  off 133;D  off code#  proxy_exit
     0       55         46         52           0
     1       55         46         52           1
    42       56         46         52          42
    99       56         46         52          99
   127       57         46         52         127
```

The same data as a Markdown table:

| code | raw_len | offset `133;D` | offset code-num | recorded exit_status |
|------|---------|----------------|-----------------|----------------------|
| 0    | 55      | 46             | 52              | 0   |
| 1    | 55      | 46             | 52              | 1   |
| 42   | 56      | 46             | 52              | 42  |
| 99   | 56      | 46             | 52              | 99  |
| 127  | 57      | 46             | 52              | 127 |

**Digit-width analysis.** Everything before the number — `\x1b]133;A\x1b\\` + `\x1b]133;B\x1b\\` + `\x1b]133;C;cmdline=foo\x1b\\` + `hello` + `\x1b]133;D;` — is fixed-length (46 bytes up to `133;D`, and 52 bytes up to the first digit). Therefore the number always starts at offset **52**; only the trailing digits (and the following 2-byte `ST`) extend the total. Hence total lengths **55** (1 digit), **56** (2 digits), and **57** (3 digits). The marker offset **46** is likewise fixed because the `A` + `B` + `C` + `hello` prefix is constant.

**Code-99 full-path evidence.** Q2 asks for concrete evidence that `99` traversed the *entire* handling path. This was produced by invoking the **real** `Window.handle_cmd_end('99')` [`kitty/window.py:L1408`] (actual bytecode, options initialized exactly as the test harness does):

```
=== code-99 full-path evidence (isolated) ===
last_cmd_exit_status : 99 ( int )
on_cmd_startstop keys: ['is_start', 'time', 'cmdline', 'exit_status']
payload['exit_status']: 99 ( int )
payload['cmdline']    : 'foo'
payload['is_start']   : False
```

This demonstrates the full chain: the C slice of the exit-status substring [`kitty/screen.c:L2350-L2352`] → the `int()` conversion in `handle_cmd_end` [`kitty/window.py:L1413`] → assignment to the `last_cmd_exit_status` state field → the `on_cmd_startstop` watcher fired with the integer `exit_status` [`kitty/window.py:L1419-L1420`]. The value `99` is observed as an `int` in **both** the recorded state (`last_cmd_exit_status`) and the watcher payload (`payload['exit_status']`).

---

## Q3 — Malformed / edge exit codes (`not_a_number` and empty)

**Direct answer (CANONICAL).** For **both** `OSC 133;D;not_a_number` and `OSC 133;D;` (empty payload), the value ultimately recorded is **`last_cmd_exit_status = 0`**. Cause: `handle_cmd_end` performs `self.last_cmd_exit_status = int(exit_status)` [`kitty/window.py:L1413`] inside a `try`; when `int()` raises (a non-numeric string, or the empty string), the `except Exception:` branch sets `self.last_cmd_exit_status = 0` [`kitty/window.py:L1415`].

**Captured output — from the REAL `Window.handle_cmd_end` bytecode (complete, unedited):**

```
=== REAL Window.handle_cmd_end (canonical) ===
handle_cmd_end(            '0') -> last_cmd_exit_status=0 (int); watcher={'is_start': False, 'time': 0.047, 'cmdline': 'foo', 'exit_status': 0}
handle_cmd_end(            '1') -> last_cmd_exit_status=1 (int); watcher={'is_start': False, 'time': 0.047, 'cmdline': 'foo', 'exit_status': 1}
handle_cmd_end(           '42') -> last_cmd_exit_status=42 (int); watcher={'is_start': False, 'time': 0.048, 'cmdline': 'foo', 'exit_status': 42}
handle_cmd_end(           '99') -> last_cmd_exit_status=99 (int); watcher={'is_start': False, 'time': 0.048, 'cmdline': 'foo', 'exit_status': 99}
handle_cmd_end(          '127') -> last_cmd_exit_status=127 (int); watcher={'is_start': False, 'time': 0.048, 'cmdline': 'foo', 'exit_status': 127}
handle_cmd_end( 'not_a_number') -> last_cmd_exit_status=0 (int); watcher={'is_start': False, 'time': 0.048, 'cmdline': 'foo', 'exit_status': 0}
handle_cmd_end(             '') -> last_cmd_exit_status=0 (int); watcher={'is_start': False, 'time': 0.048, 'cmdline': 'foo', 'exit_status': 0}
```

(The `time` values are monotonic floats and vary run-to-run; they are **not** reported values.)

**NON-CANONICAL contrast (explicitly labeled non-canonical).** The test-harness `Callbacks.cmd_output_marking` proxy [`kitty_tests/__init__.py:L71`] wraps the conversion in `with suppress(Exception)` [`kitty_tests/__init__.py:L78-L79`], so a failed `int()` **leaves the previous value unchanged**. Its `last_cmd_exit_status` starts at `sys.maxsize` (`9223372036854775807`, set in `Callbacks.__init__` [`kitty_tests/__init__.py:L48`]):

```
--- NON-CANONICAL test-harness Callbacks proxy (with suppress -> leaves prior value) ---
initial proxy last_cmd_exit_status: 9223372036854775807
after C+D;42                     : 42
after C+D;not_a_number (unchanged): 42
after C+D; (empty) (unchanged)   : 42
```

**Contrast, stated plainly.** The **canonical** real-`Window` result for both malformed payloads is **`0`**. The **non-canonical** harness proxy instead leaves the prior value (`42`) unchanged. The reported answer is **`0`**; the proxy `42` is presented only as a labeled caveat about the observation vehicle, never as the recorded value.

---

## Supporting details

### Two notions of "output"

- **(a) Raw PTY byte stream.** kitty does not rewrite the child's stdout, so the OSC 133 control strings are **present** in the bytes kitty receives on the PTY. **All offsets and lengths in this document are measured on this raw stream.**
- **(b) Rendered screen buffer.** The OSC 133 sequences are parsed as control strings and **consumed**; they are never written into screen cells. Only the literal text (`hello`) is drawn.

### State before / during / after (recorded exit status)

- **Before** any `D` marker: `last_cmd_output_start_time = 0.` [`kitty/window.py:L569`] and `last_cmd_exit_status = 0` [`kitty/window.py:L572`] (the `Window` defaults).
- **During**: the guard in `handle_cmd_end` returns early unless a `C` output-start preceded the `D` (`if self.last_cmd_output_start_time == 0.: return`) [`kitty/window.py:L1409-L1410`]. This is exactly why the probe sends a `C` marker before `D`.
- **After** a valid `D`: the value becomes the parsed integer (e.g. `42`, `99`). After a malformed `D`: it becomes `0`.

### Distinct representation: the finish notification

The optional command-finish notification embeds the **raw** payload string via `cmd.body = f'Command {s} finished with status: {exit_status}.\nClick to focus.'` [`kitty/window.py:L1429`]. This is the raw marker string — a distinct representation from the parsed integer stored in `last_cmd_exit_status`. (With the default `notify_on_cmd_finish = 'never'`, this notification path is skipped, but the `on_cmd_startstop` watcher still fires with the parsed integer.)

### Terminator nuance (`ST` vs `BEL`)

The investigation probe and the scrollback prompt-jump marker in `history.c` (`reverse_find` of `"\x1b]133;C\x1b\\"`, [`kitty/history.c:L475`]) use the **`ST`** terminator (`ESC \`). The real shell-integration emit scripts instead use the **`BEL`** terminator (`\a` = `0x07`). Both are valid OSC string terminators per the wire format `ESC ] 133 ; <Command> [; <Parameters>] ST`, where `ST` is `ESC \` or `BEL`. The choice of terminator does not change the parsed marker or the recorded exit code.

### Emit-side context (where the real exit code originates)

When a real program is run through kitty, the shell-integration scripts inject the markers; the `D` exit code is the shell's own `$?` / `$status` / `$cmd_status`:

- **bash** — `shell-integration/bash/kitty.bash`: emits `C;cmdline=%q` [L208] and `D;$?` together with `A` [L239] (`A;k=s` for PS2 at [L240]).
- **zsh** — `shell-integration/zsh/kitty-integration`: emits `D;$cmd_status` [L145], a bare `D` [L149], `A` [L153], `A;k=s` [L163], and `C;cmdline=%q` [L218].
- **fish** — `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish`: emits a bare `D` [L83], `A;special_key=1` [L85], `C;cmdline_url=%s` [L91], and `D;$status` [L96].

### Run-to-run stability

Two consecutive runs of the observation script produced **byte-identical** output except for the monotonic `time` float in the watcher payload (which is expected to vary and is not a reported value). This satisfies the stability requirement — every reported value (byte lengths, offsets, recorded exit statuses, decoded command line, watcher keys) is stable across runs, and was additionally reproduced identically in both Python 3.12.3 (pinned reference image) and Python 3.13.7.

### OSC 133 / FinalTerm background (validated against external references)

For context only — the runtime observations above are the primary evidence. In the FinalTerm "semantic prompt" protocol adopted by kitty:

- `A` = prompt start (`FTCS_PROMPT`).
- `B` = command/input start (`FTCS_COMMAND_START`).
- `C` = command output start (`FTCS_COMMAND_EXECUTED`).
- `D [;<code>]` = command finished (`FTCS_COMMAND_FINISHED`), with an optional exit code.

The `C;cmdline=…` / `C;cmdline_url=…` parameter is a **kitty extension** over the base FinalTerm protocol, decoded by `decode_cmdline` [`kitty/window.py:L225`].

---

## Reference `file:line` index (verified at commit `815df1e210e0…`, `kitty 0.35.2`)

- `kitty/vt-parser.c`: `case 133:` **L536**; `#ifdef DUMP_COMMANDS` **L537** + `REPORT_OSC2(...)` **L539** (non-canonical); canonical branch **L542-L546** (`buf[limit]=0` **L543**; `shell_prompt_marking(self->screen, (char*)buf + i)` **L544**).
- `kitty/screen.c`: `shell_prompt_marking` **L2328**; `switch (buf[0])` **L2331**; `case 'A'` **L2332** (callback **L2338**); `case 'C'` **L2340** (cmdline slice **L2343-L2344**, decode **L2346**, callback **L2347**); `case 'D'` **L2350** (`exit_status` slice **L2351**, callback **L2352**); **no `case 'B'`**.
- `kitty/window.py`: `decode_cmdline` **L225**; defaults `last_cmd_output_start_time = 0.` **L569**, `last_cmd_exit_status = 0` **L572**; `handle_cmd_end` **L1408** (guard **L1409-L1410**, `int()` **L1413**, `except → 0` **L1415**, watcher **L1419-L1420**, raw-string notification **L1429**); `cmd_output_marking` **L1453** (routes `D` → `handle_cmd_end` **L1460-L1461**).
- `kitty_tests/__init__.py`: `parse_bytes` **L30**; `Callbacks.__init__` `last_cmd_exit_status = sys.maxsize` **L48**; `Callbacks.cmd_output_marking` proxy **L71** (`with suppress(Exception)` **L78-L79**); `create_screen` **L237**; `create_pty` **L243**.
- `kitty/history.c`: scrollback prompt-jump `reverse_find` of `"\x1b]133;C\x1b\\"` **L475**.
- `kitty/client.py`: remote-control `write_osc(133, payload)` **L251** (non-canonical emitter).

---

*All captured output blocks above are reproduced verbatim from the observation runs. Deterministic values were confirmed identical across Python 3.12.3 (pinned reference image) and Python 3.13.7, and were stable across repeated runs. The investigation was strictly read-only: temporary observation scripts lived under `/tmp` and were removed, leaving the repository unchanged apart from this document.*

