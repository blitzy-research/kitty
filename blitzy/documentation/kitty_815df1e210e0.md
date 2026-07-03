# How kitty Processes OSC 133 Shell-Integration ("Semantic Prompt") Markers

A **run-first, evidence-based** answer document for the **kitty** terminal emulator at
branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

Every behavioral claim below is followed **immediately** by a pasted line of **real
output** captured from actually building and running kitty's own code path in this
environment. Statements that are read from source but not directly executed are labeled
`(inferred)`. All reported values were confirmed **identical across 3 runs** (≥2 required).

---

## Environment & Build

All values in this document were produced in the canonical Docker toolchain image
`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0…`. The versions below are
pasted **verbatim** from this environment (they may differ from any other machine and must
be read as reported here):

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ python3 --version
Python 3.13.7

$ python3 -c "import sys; print(sys.version)"
3.13.7 (main, Mar  3 2026, 12:19:54) [GCC 15.2.0]

$ cc --version | head -1
cc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ go version
go version go1.24.4 linux/amd64
```

**Exact build command (canonical, default configuration — no custom `kitty.conf`):**

```
$ python3 setup.py build --ignore-compiler-warnings
BUILD_EXIT_CODE=0
```

This compiles the C extension `kitty.fast_data_types`, which houses the VT parser
(`kitty/vt-parser.c`), the screen model (`kitty/screen.c`), and the scrollback
(`kitty/history.c`) [`setup.py:1084`]. The `--ignore-compiler-warnings` flag is required
in this image only because the system `wayland-protocols` adds enum members that trip
`-Werror=switch` in glfw; it is kitty's own flag and changes **no source**. In this image
the extension was already compiled, so the incremental build produced an empty log and
exited `0`. Import is verified:

```
$ python3 -c "import kitty.fast_data_types as f; print(f.__file__)"
/tmp/blitzy/kitty/blitzy-1fc99dd7-112c-411d-aef5-44ca74d806d7_c661da/kitty/fast_data_types.so

$ python3 -c "from kitty.window import decode_cmdline, cmd_output, Window, Watchers, CommandOutput; print('WINDOW IMPORTS OK')"
WINDOW IMPORTS OK
```

**String Terminator used by the test vector: `BEL` (`\x07`, one byte).** OSC 133 accepts
either `BEL` (`\a`/`\x07`) or `ESC \` (`\x1b\x5c`, two bytes) as the terminator; kitty's own
shell-integration scripts emit `BEL` (e.g. fish `\e]133;D\a`
[`shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:83`]). The terminator
choice changes the absolute byte counts, so it is stated explicitly and its effect is
measured in **Q3**.

---

## Methodology (canonical entry points)

- **Byte measurement / parser semantics** are taken through `parse_bytes(screen, data)`
  [`kitty_tests/__init__.py:30`], which feeds bytes through the **same C VT parser** used
  for live child-PTY output (via `test_create_write_buffer` → `test_commit_write_buffer` →
  `test_parse_written_data`). This is the canonical headless entry point.
- **Exit-status recording** is taken through the **production**
  `kitty.window.Window.handle_cmd_end` [`kitty/window.py:1408`], the real method invoked
  when a command finishes.
- **Non-canonical** sources — the `kitty_tests` `Callbacks` test-double
  [`kitty_tests/__init__.py:71-79`], the `#ifdef DUMP_COMMANDS` debug path
  [`kitty/vt-parser.c:537-541`], and remote control (`kitty @ get-text`) — are **labeled**
  where mentioned and never substituted for a production observation.

A single temporary probe script lived **outside** the repository at `/tmp/osc133_probe.py`,
was run **3 times** with byte-for-byte identical output (the `diff -u` / `wc -c -l` /
`sha256sum` verification proving this is pasted verbatim in the **Read-only integrity**
appendix), and was deleted afterward. The reference test vector (BEL terminator) is:

```python
b'\x1b]133;A\x07\x1b]133;B\x07\x1b]133;C;cmdline=ls -la\x07hello output\n\x1b]133;D;42\x07'
```

Constructed as: `A` marker + `B` marker + `C;cmdline=ls -la` marker + the text
`"hello output\n"` + `D;42` marker, each OSC sequence terminated by `BEL`.

**The dispatch chain being exercised** (read-only citations):
the parser's OSC handler `case 133:` null-terminates the payload (`buf[limit]=0`) and calls
`shell_prompt_marking(self->screen, (char*)buf + i)` [`kitty/vt-parser.c:536-544`]; the OSC
code-parse loop has already consumed `133;`, so the payload handed to the handler begins at
the marker letter `A`/`B`/`C`/`D` [`kitty/vt-parser.c:466-477`] `(inferred)`.
`shell_prompt_marking` [`kitty/screen.c:2328`] switches on that letter and, via the
`CALLBACK` macro `PyObject_CallMethod(self->callbacks, …)` [`kitty/screen.c:87-91`], invokes
the Python method `cmd_output_marking` [`kitty/window.py:1453`], which for the command-end
case routes to `handle_cmd_end` [`kitty/window.py:1408`].

---

## Q1 — OSC 133 semantics: what kitty does with `A`, `B`, `C`, `D`

kitty's handler `shell_prompt_marking` [`kitty/screen.c:2328-2356`] contains cases for
`'A'`, `'C'`, and `'D'` only — **there is no `case 'B'`**. Each case below is paired with the
exact callback the **real C parser** fired when the marker was pushed through `parse_bytes`
(B1 probe). The probe subclassed `Callbacks` to log every
`cmd_output_marking(is_start, data)` call:

```
===== B1: Q1 semantics (real C parser via parse_bytes) =====
A -> [(False, '')]
B -> []
C;cmdline=ls -la -> [(True, 'cmdline=ls -la')]
D;42 -> [(None, '42')]
A;k=s (secondary) -> []
```

### `A` — prompt start (literal `A`)
**Cause → effect:** `case 'A'` sets `self->linebuf`'s current line attribute
`prompt_kind = PROMPT_START` and fires `CALLBACK("cmd_output_marking", "O", Py_False)`
**only** when the parsed prompt kind is `PROMPT_START` [`kitty/screen.c:2332-2339`]. The
prompt-mark tokens are parsed by `parse_prompt_mark` [`kitty/screen.c:2316`], which
recognises `k=s` → `SECONDARY_PROMPT`, `redraw=0`, and `special_key=1`
[`kitty/screen.c:2321-2323`] `(inferred from source; the observable consequence for `k=s` is
measured below)`.

- Evidence — plain `A` fires the prompt-start callback with `is_start=False`:
  ```
  A -> [(False, '')]
  ```
- Evidence — a **secondary** prompt (`A;k=s`) fires **no** callback, because the callback is
  gated on `PROMPT_START` and `k=s` selects `SECONDARY_PROMPT`:
  ```
  A;k=s (secondary) -> []
  ```

### `B` — end of prompt / start of typed command (literal `B`): **no-op**
**Cause → effect:** there is **no `case 'B'`** in `shell_prompt_marking`
[`kitty/screen.c:2332-2353`], so the marker is consumed by the parser and produces **no**
handler action and **no** callback.

- Evidence — pushing `B` through the real parser yields an empty callback log:
  ```
  B -> []
  ```
- Corroboration (provenance): kitty's zsh integration has its `B` emit **commented out**
  with the note that kitty does not use `B` prompt marking
  [`shell-integration/zsh/kitty-integration:222-226`] `(inferred — read from the script)`.

### `C` — command output start, optional `cmdline` (literal `C`)
**Cause → effect:** `case 'C'` sets `prompt_kind = OUTPUT_START` and, **only** when the
payload after the letter begins with `;cmdline`, decodes the command line by passing
`buf+2` — so the string handed to Python **includes** the `cmdline=` prefix — then fires
`CALLBACK("cmd_output_marking", "OO", Py_True, c)` [`kitty/screen.c:2340-2349`].

- Evidence — `C;cmdline=ls -la` fires the output-start callback with `is_start=True` and the
  decoded payload **including** the `cmdline=` prefix:
  ```
  C;cmdline=ls -la -> [(True, 'cmdline=ls -la')]
  ```
- Evidence — the Python side decodes that payload to the program name via `decode_cmdline`
  [`kitty/window.py:225`]:
  ```
  decode_cmdline('cmdline=ls -la') = 'ls'
  ```

### `D` — command finished, optional exit status (literal `D`)
**Cause → effect:** `case 'D'` computes `exit_status = buf[1]==';' ? buf+2 : ""` and fires
`CALLBACK("cmd_output_marking", "Os", Py_None, exit_status)`, passing the **raw** exit-status
string [`kitty/screen.c:2350-2353`]. `is_start` is therefore `None` for `D`.

- Evidence — `D;42` fires the command-finished callback with `is_start=None` and the **raw**
  string `'42'`:
  ```
  D;42 -> [(None, '42')]
  ```

**Summary of `is_start` encoding (observed):** `False` ⇒ prompt start (`A`); `True` ⇒ output
start (`C`); `None` ⇒ command finished (`D`); `B` produces nothing. On the Python side,
`cmd_output_marking` treats a truthy `is_start` (`C`) as "output started" and both `False`
(`A`) and `None` (`D`) fall through to `handle_cmd_end` [`kitty/window.py:1453-1461`]
`(inferred from source; the exit-status consequence of the `D`/`None` path is measured in
Q4/Q5)`.

---

## Q2 — Capturing the marker stream

The exact reference vector (BEL terminator) was pushed through the real parser and measured
with Python `len()` / `find()`, and the three capture surfaces were read (B2 probe):

```
===== B2: Q2 capture + measurement (real C parser) =====
Q2c total len(v) = 63
Q2d D-marker offset v.find(b'\x1b]133;D;42\x07') = 52
    exit-code-digit offset = 60
screen_contents (parsed) = 'hello output'
OSC present in raw input bytes?  b']133;' in v -> True
OSC present in parsed screen text? ']133;' in screen -> False
Screen.cmd_output C-method chunks = ['\x1b[m', '\x1b]133;C\x1b\\abcd', '\n', '\x1b[m', '12']
Screen.cmd_output(last_run, as_ansi=True) [C-method joined] = '\x1b[m\x1b]133;C\x1b\\abcd\n\x1b[m12'
module kitty.window.cmd_output(as_ansi=True) = '\x1b[mabcd\n\x1b[m12'
decode_cmdline('cmdline=ls -la') = 'ls'
```

### Q2(a) — What is actually captured
What you get depends on **which surface** you read; there are three, and they differ:

1. **Raw child stream** (what a PTY exposes as `received_bytes`, initialised at
   [`kitty_tests/__init__.py:320`] and accumulated at [`kitty_tests/__init__.py:365-366`],
   where each child read is appended to `received_bytes` and then handed to the same
   `parse_bytes`) — the literal bytes fed in, **including** the OSC sequences. This is the
   input vector itself.
2. **Parsed screen text** (`PTY.screen_contents()`, `kitty_tests/__init__.py:404-410`) — only
   the visible cell contents; the parser has **consumed** every control sequence. For this
   vector the only visible text is `hello output`:
   ```
   screen_contents (parsed) = 'hello output'
   ```
   (The `\n` moves the cursor to the next line, which is blank and filtered out; the `A`/`B`/
   `C`/`D` OSC sequences leave no printable cells.)
3. **Last-command-output dump** — two different APIs that **differ** (see Q2(b)).

### Q2(b) — Are the OSC sequences still present?
**It depends on the surface, and this is measured both ways:**

- In the **raw input** the OSC bytes remain (the literal `]133;` is present):
  ```
  OSC present in raw input bytes?  b']133;' in v -> True
  ```
- In the **parsed screen text** the OSC bytes are gone (the parser consumed them):
  ```
  OSC present in parsed screen text? ']133;' in screen -> False
  ```
- In the **last-command-output dump** the answer splits by API. Using the classic
  prompt/output scenario from kitty's own unit test (`mark_prompt`→`\033]133;A\007`, draw
  `$ 0`, `mark_output`→`\033]133;C\007`, draw `abcd`/`12`, `mark_prompt`; matching
  `kitty_tests/screen.py:1059-1063` and the assertion at `kitty_tests/screen.py:1124`):
  - The **C method** `Screen.cmd_output(CommandOutput.last_run, callback, as_ansi=True)` fires
    its callback **once per line**, and one of those chunks **is** the literal
    `OSC 133;C` sequence — so the joined result **re-emits** `\x1b]133;C\x1b\\`:
    ```
    Screen.cmd_output C-method chunks = ['\x1b[m', '\x1b]133;C\x1b\\abcd', '\n', '\x1b[m', '12']
    Screen.cmd_output(last_run, as_ansi=True) [C-method joined] = '\x1b[m\x1b]133;C\x1b\\abcd\n\x1b[m12'
    ```
    This matches the repository's own assertion at `kitty_tests/screen.py:1124`
    (`lco(as_ansi=True) == '\x1b[m\x1b]133;C\x1b\\abcd\n\x1b[m12'`).
  - The **module wrapper** `kitty.window.cmd_output(screen, as_ansi=True)`
    (`PTY.last_cmd_output`, `kitty_tests/__init__.py:412-414`) post-processes those chunks:
    for each of the first lines that `startswith('\x1b]133;C')` it strips the leading marker
    via `x.partition('\\')[-1]` [`kitty/window.py:466-467`]. Because chunk `[1]` above
    (`'\x1b]133;C\x1b\\abcd'`) does start with that marker, it is stripped to `'abcd'`, so the
    OSC bytes are **removed**:
    ```
    module kitty.window.cmd_output(as_ansi=True) = '\x1b[mabcd\n\x1b[m12'
    ```
  There is **no contradiction** between these two lines — they are two different APIs (the raw
  C method re-emits; the Python wrapper strips). The scrollback retention path likewise
  searches for the **literal** marker `reverse_find(buf, sz, "\x1b]133;C\x1b\\")`
  [`kitty/history.c:475`] `(inferred — read from source)`.

### Q2(c) — Total byte length
The full stream is **63 bytes**:

```
Q2c total len(v) = 63
```

Composition (BEL terminator): `ESC]133;A\x07` (8) + `ESC]133;B\x07` (8) +
`ESC]133;C;cmdline=ls -la\x07` (23) + `hello output\n` (13) + `ESC]133;D;42\x07` (11) =
**63** bytes.

### Q2(d) — Byte offset of the `D;42` marker
The `D` marker begins at byte offset **52**, and the exit-code **digits** begin at offset
**60**:

```
Q2d D-marker offset v.find(b'\x1b]133;D;42\x07') = 52
    exit-code-digit offset = 60
```

Offset `52` equals the length of everything before the `D` marker (the `A`, `B`,
`C;cmdline=ls -la` markers plus `hello output\n`); offset `60` is `52 + len("ESC]133;D;")`
= `52 + 8`.


---

## Q3 — Exit-code variation (`0`, `1`, `127`, plus `42`/`99` for continuity)

The same vector was rebuilt with **only** the `D;<code>` field changed, and each was measured
(B3 probe). `total_len` = full stream length; `D_off` = offset where the `D` marker begins;
`digit_off` = offset where the exit-code digits begin:

```
===== B3: Q3 exit-code table (real parser, BEL terminator) =====
code           total_len  D_off  digit_off
0                    62    52        60
1                    62    52        60
42                   63    52        60
99                   63    52        60
127                  64    52        60
not_a_number         73    52        60
(empty)              61    52        60
ESC-backslash (ST) variant, code 42: total_len=67 D_off=55 digit_off=63
```

**Findings (measured, BEL terminator):**

- The **`D`-marker offset (`52`) and the exit-code-digit offset (`60`) never change.** The
  `D` marker is appended after a **fixed-length prefix** (the `A`, `B`, `C;cmdline=…` markers
  plus `hello output\n`), so where the code *starts* is constant regardless of the code's
  value or length. **The position where the exit-code number appears does not shift.**
- Only the **total length** grows, by exactly **`(digits − 1)`** relative to a one-digit code:
  - `0` and `1` are **identical** at **62** bytes (one digit each).
  - `42` is **+1** → **63** bytes (two digits).
  - `127` is **+2** → **64** bytes (three digits).
  - `99` is two digits → **63** bytes (same as `42`).
  - `not_a_number` (12 characters) → **73** bytes (`62 + 11`).
  - empty (`""`) → **61** bytes (one byte shorter than a one-digit code, the digit removed).
- **Terminator effect (stated explicitly):** the numbers above use the **`BEL`** terminator.
  With the two-byte `ESC \` (String Terminator) instead, each of the four OSC sequences gains
  one byte, so for code `42` the total grows by **+4** (to **67**) and the offsets become
  **`D_off=55` / `digit_off=63`** (each shifted `+3` by the three preceding OSC terminators):
  ```
  ESC-backslash (ST) variant, code 42: total_len=67 D_off=55 digit_off=63
  ```

So, to answer directly: across exit codes `0`, `1`, `127`, the **byte length** is
`62`, `62`, `64` respectively, and the **position** of the `D` marker (`52`) and of the
exit-code digits (`60`) is the **same for every code** — the exit-code position does **not**
shift; only the total length changes, by `(digits − 1)`.


---

## Q4 — Exit code `99`: runtime proof it propagated end-to-end

Exit-status parsing happens in the **Python callback layer**, not in C: the C `D` handler
passes the exit status through as a **raw string**, and the production
`Window.handle_cmd_end` executes `int(exit_status)` (with a fallback of `0` on any exception)
[`kitty/window.py:1413-1415`]. To observe the real path, the probe drove the production
`Window.handle_cmd_end` directly (B4 probe), with the window's `last_cmd_exit_status`
pre-seeded to an impossible **sentinel `-999999`** to prove the real method overwrote it, and
an `on_cmd_startstop` watcher attached to capture the payload kitty broadcasts:

```
===== B4: Q4/Q5 production Window.handle_cmd_end (DEFAULT config) =====
default notify_on_cmd_finish = NotifyOnCmdFinish(when='never', duration=5.0, action='notify', cmdline=())
handle_cmd_end('0') -> last_cmd_exit_status = 0    watcher exit_status = 0
handle_cmd_end('1') -> last_cmd_exit_status = 1    watcher exit_status = 1
handle_cmd_end('42') -> last_cmd_exit_status = 42    watcher exit_status = 42
handle_cmd_end('99') -> last_cmd_exit_status = 99    watcher exit_status = 99
handle_cmd_end('127') -> last_cmd_exit_status = 127    watcher exit_status = 127
handle_cmd_end('not_a_number') -> last_cmd_exit_status = 0    watcher exit_status = 0
handle_cmd_end('') -> last_cmd_exit_status = 0    watcher exit_status = 0
```

**The proof that `99` made it through the entire code path (default configuration):**

- The window's `last_cmd_exit_status` — pre-seeded to the sentinel `-999999` — was
  **overwritten to `99`** by the real `handle_cmd_end`, proving the parse executed and stored
  the value:
  ```
  handle_cmd_end('99') -> last_cmd_exit_status = 99    watcher exit_status = 99
  ```
- The **`on_cmd_startstop` watcher payload** kitty broadcasts to registered watchers
  [`kitty/window.py:1419-1420`] carried the **parsed integer `99`** (the `watcher exit_status
  = 99` above). In the **default** configuration this watcher payload is the primary Q4
  evidence, because `notify_on_cmd_finish` defaults to `when='never'` — its default value is
  declared at [`kitty/options/types.py:560`] as
  `NotifyOnCmdFinish(when='never', duration=5.0, action='notify', cmdline=())`, and the
  config-option default is set at [`kitty/options/definition.py:3190`] — so **no notification
  fires**:
  ```
  default notify_on_cmd_finish = NotifyOnCmdFinish(when='never', duration=5.0, action='notify', cmdline=())
  ```

### Q4 (non-default enhancement — labeled **NON-DEFAULT**)
To additionally show `99` inside a user-facing notification, the probe set
`notify_on_cmd_finish` to `when='always'` and intercepted `kitty.window.notify_with_command`
at runtime (monkeypatched **in the temporary script only** — no repo file changed) to read
the notification body [`kitty/window.py:1429`]:

```
===== B4 (NON-DEFAULT): notify_on_cmd_finish when='always' -> capture body =====
raw='99'           last_cmd_exit_status=99   body='Command ls -la finished with status: 99.\nClick to focus.'
raw='not_a_number' last_cmd_exit_status=0    body='Command ls -la finished with status: not_a_number.\nClick to focus.'
raw=''             last_cmd_exit_status=0    body='Command ls -la finished with status: .\nClick to focus.'
```

This demonstrates a **raw-vs-parsed** distinction: the notification **body** interpolates the
**raw** `exit_status` string (`kitty/window.py:1429`), while `last_cmd_exit_status` and the
watcher payload hold the **parsed int** (with the `except → 0` fallback). This `when='always'`
case is **non-default**; the default answer above (no notification) stands.

---

## Q5 — Invalid / empty exit status (`not_a_number` and empty `""`)

Through the **production** `Window.handle_cmd_end`, both an unparseable value and an empty
value are recorded as **`0`**, via the `except Exception: self.last_cmd_exit_status = 0`
fallback [`kitty/window.py:1413-1415`]:

- `not_a_number` → `0` (both the stored status and the watcher payload):
  ```
  handle_cmd_end('not_a_number') -> last_cmd_exit_status = 0    watcher exit_status = 0
  ```
- empty (`''`) → `0` (both the stored status and the watcher payload):
  ```
  handle_cmd_end('') -> last_cmd_exit_status = 0    watcher exit_status = 0
  ```

### NON-CANONICAL contrast — the `kitty_tests` `Callbacks` test-double
The `kitty_tests` `Callbacks` double is a **synthetic stand-in**, **not** the production path.
It initialises `last_cmd_exit_status = sys.maxsize` and its command-end branch is
`with suppress(Exception): self.last_cmd_exit_status = int(data)`
[`kitty_tests/__init__.py:71-79`], so an invalid or empty value leaves the `sys.maxsize`
sentinel **unchanged** — **diverging** from production's `0`:

```
===== B4 (NON-CANONICAL): kitty_tests Callbacks test-double =====
Callbacks double: start_sentinel=9223372036854775807  '99'           -> 99
Callbacks double: start_sentinel=9223372036854775807  'not_a_number' -> 9223372036854775807
Callbacks double: start_sentinel=9223372036854775807  ''             -> 9223372036854775807
```

This is **why Q4/Q5 must be answered from `Window.handle_cmd_end`**, not from the double: the
double records `9223372036854775807` (i.e. `sys.maxsize`) for `not_a_number` and empty,
whereas the real production path records **`0`**. The values `9223372036854775807` above are
**non-canonical** and do not answer the question — the canonical answer is `0`.


---

## Canonical vs. non-canonical sourcing

| Source | Status | Used for |
|--------|--------|----------|
| `parse_bytes(screen, data)` [`kitty_tests/__init__.py:30`] | **canonical** | Q1 marker semantics; Q2/Q3 byte length & offset measurements (drives the real C VT parser) |
| Production `Window.handle_cmd_end` [`kitty/window.py:1408`] | **canonical** | Q4/Q5 exit-status recording, watcher payload |
| `kitty_tests` `Callbacks` test-double [`kitty_tests/__init__.py:71-79`] | **non-canonical** | Contrast only; records `sys.maxsize` for invalid/empty, diverging from production `0` |
| `#ifdef DUMP_COMMANDS` debug path [`kitty/vt-parser.c:537-541`] | **non-canonical** | Not used; would only fire in a debug build |
| Remote control (`kitty @ get-text`) | **non-canonical** | Not used |

Every reported value in Q1–Q5 comes from a **canonical** source. The single non-canonical
result shown (the `Callbacks` double's `9223372036854775807`) is explicitly labeled and is
presented only to explain why the production path is required.

---

## Coverage pass

Re-reading the prompt, every named item is addressed with its literal and a cause→effect
reason, each backed by a pasted evidence line:

| Named item | Answer (literal) | Where |
|------------|------------------|-------|
| Marker `A` | prompt start; fires `cmd_output_marking(is_start=False)`; `A -> [(False, '')]` | Q1 → `A` |
| Marker `B` | **no-op** (no `case 'B'`); `B -> []` | Q1 → `B` |
| Marker `C` | output start; decodes optional `cmdline`; `C;cmdline=ls -la -> [(True, 'cmdline=ls -la')]` | Q1 → `C` |
| Marker `D` | command finished; passes raw status; `D;42 -> [(None, '42')]` | Q1 → `D` |
| `cmdline` parameter | decoded incl. `cmdline=` prefix; `decode_cmdline('cmdline=ls -la') = 'ls'` | Q1 → `C`, Q2 |
| Capture (a)/(b) | raw retains OSC (`True`); parsed screen drops OSC (`False`); wrapper strips, C method re-emits | Q2(a), Q2(b) |
| Total byte length | `63` | Q2(c) |
| `D;42` offset | `52` (digits at `60`) | Q2(d) |
| Exit code `0` | `total_len 62`, `D_off 52`, `digit_off 60`; recorded `0` | Q3, Q4 table |
| Exit code `1` | `total_len 62`, `D_off 52`, `digit_off 60`; recorded `1` | Q3, Q4 table |
| Exit code `42` | `total_len 63`, `D_off 52`, `digit_off 60`; recorded `42` | Q2, Q3, Q4 table |
| Exit code `99` | `total_len 63`; recorded `99`; sentinel `-999999` overwritten; watcher `99` | Q3, Q4 |
| Exit code `127` | `total_len 64`, `D_off 52`, `digit_off 60`; recorded `127` | Q3, Q4 table |
| `not_a_number` | recorded **`0`** (production); non-canonical double leaves `9223372036854775807` | Q5 |
| empty (`""`) | recorded **`0`** (production); non-canonical double leaves `9223372036854775807` | Q5 |
| Exit-code position shift | **no shift** — `D_off`/`digit_off` constant; only total length grows by `(digits − 1)` | Q3 |
| String Terminator | `BEL` (`\x07`); `ESC \` variant shifts offsets to `55`/`63`, `+4` total | Environment, Q3 |

---

## Appendix — Read-only integrity & reproducibility

- **Terminator:** `BEL` (`\x07`, 1 byte) for the primary vector; the `ESC \` variant is
  measured and labeled in Q3.
- **Stability (reproduced in this environment, 3 runs):** the temporary probe
  `/tmp/osc133_probe.py` (outside the repository) was run **3 times**; the output was
  **byte-for-byte identical** — `diff -u` reported no differences and all three runs share a
  single SHA-256 — at **`2744` bytes / `49` lines** each run. Exact commands and their real,
  verbatim output:
  ```
  $ export PYTHONPATH="$(pwd)"   # repo root, so kitty.fast_data_types is importable
  $ for i in 1 2 3; do python3 /tmp/osc133_probe.py > /tmp/osc133_run$i.txt; done
  $ wc -c -l /tmp/osc133_run1.txt /tmp/osc133_run2.txt /tmp/osc133_run3.txt
    49 2744 /tmp/osc133_run1.txt
    49 2744 /tmp/osc133_run2.txt
    49 2744 /tmp/osc133_run3.txt
   147 8232 total
  $ diff -u /tmp/osc133_run1.txt /tmp/osc133_run2.txt && echo "IDENTICAL: run1 == run2"
  IDENTICAL: run1 == run2
  $ diff -u /tmp/osc133_run2.txt /tmp/osc133_run3.txt && echo "IDENTICAL: run2 == run3"
  IDENTICAL: run2 == run3
  $ sha256sum /tmp/osc133_run1.txt /tmp/osc133_run2.txt /tmp/osc133_run3.txt
  424c8bbec05fff36739b1d249a9a45ee762185748eb70867f7f81751b101afcd  /tmp/osc133_run1.txt
  424c8bbec05fff36739b1d249a9a45ee762185748eb70867f7f81751b101afcd  /tmp/osc133_run2.txt
  424c8bbec05fff36739b1d249a9a45ee762185748eb70867f7f81751b101afcd  /tmp/osc133_run3.txt
  ```
- **Canonical build command:** `python3 setup.py build --ignore-compiler-warnings` (exit `0`),
  default configuration (no custom `kitty.conf`).
- **Read-only & cleanup:** no existing repository file was modified, added, or deleted. Every
  build artifact produced while observing was **physically removed** from the working tree
  afterward — **not merely `.gitignore`d** — namely `kitty/fast_data_types.so` (and the other
  `*.so`: `kitty/glfw-wayland.so`, `kitty/glfw-x11.so`, `kittens/transfer/rsync.so`), `build/`,
  the generated headers `kitty/uniforms_generated.h` and `kitty/docs_ref_map_generated.h`, all
  30 source-tree `__pycache__/` directories, `constants_generated.go`, the `kitty/launcher/`
  binaries (`kitty`, `kitten`), the `glfw/wayland-*-protocol.[ch]` files, and every
  `*_generated.{go,s,bin}` — as were the temporary probe (`/tmp/osc133_probe.py`) and its run
  captures. Only the `.venv/` environment directory (a Python virtualenv, not a kitty build
  artifact) is intentionally retained. Shell-only cleanup verification (no Python, so no
  `__pycache__/` is regenerated):
  ```
  $ find . -name "fast_data_types*.so" -not -path "./.venv/*"
  (no output — none remain)
  $ test -d build && echo present || echo absent
  absent
  $ find kitty -maxdepth 1 -name "*_generated.h"
  (no output — none remain)
  $ find . -type d -name __pycache__ -not -path "./.venv/*"
  (no output — none remain)
  $ find . \( -name "*_generated.go" -o -name "*_generated.s" -o -name "*_generated.bin" \) -not -path "./.venv/*" | wc -l
  0
  $ find . -name "*.so" -not -path "./.venv/*" | wc -l
  0
  ```
- **Integrity (final delivered state):** this document is committed, so the working tree is
  clean (empty `git status --porcelain`) and the only baseline→HEAD change is this single added
  file:
  ```
  $ git status --porcelain
  $ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
  A	blitzy/documentation/kitty_815df1e210e0.md
  ```

### Reproduction — the probe's canonical entry points
The probe imports the real parser and the production callback, then measures:

```python
from kitty.fast_data_types import Screen, set_options, get_options
from kitty.options.types import Options, defaults
from kitty.options.parse import merge_result_dicts
from kitty.window import decode_cmdline, cmd_output, Window, Watchers, CommandOutput
from kitty_tests import parse_bytes, Callbacks

set_options(Options(merge_result_dicts(defaults._asdict(), {})))  # default/canonical config

# Q1/Q2/Q3: drive the REAL C parser
parse_bytes(screen, b'\x1b]133;A\x07\x1b]133;B\x07\x1b]133;C;cmdline=ls -la\x07hello output\n\x1b]133;D;42\x07')

# Q4/Q5: production exit-status path (sentinel proves overwrite)
w = Window.__new__(Window)
w.id = 1; w.last_cmd_cmdline = 'ls -la'
w.last_cmd_exit_status = -999999
w.last_cmd_output_start_time = monotonic() - 1.0   # non-zero, else handle_cmd_end early-returns
w.watchers = Watchers(); w.watchers.on_cmd_startstop = [recorder]
w.handle_cmd_end('99')   # -> w.last_cmd_exit_status == 99, watcher payload exit_status == 99
```

### Source references (all read-only; cited, never modified)
- `kitty/vt-parser.c:536-544` — `case 133:` null-terminates payload, calls
  `shell_prompt_marking`; `#ifdef DUMP_COMMANDS` non-canonical debug path at `537-541`.
- `kitty/screen.c:87-91` — `CALLBACK` macro (C→Python bridge).
- `kitty/screen.c:2316-2325` — `parse_prompt_mark` (`k=s`→secondary, `redraw`, `special_key`).
- `kitty/screen.c:2328-2356` — `shell_prompt_marking`: `A` (2332), `C` (2340), `D` (2350); **no `B`**.
- `kitty/window.py:225` — `decode_cmdline`; `:275` — `class CommandOutput(IntEnum)`;
  `:457-468` — module `cmd_output` (strip at `466-467`).
- `kitty/window.py:1408-1429` — `handle_cmd_end`: early-return `1409`; `int(exit_status)`
  `1413`; `= 0` fallback `1415`; watcher payload `1419-1420`; notification body (raw) `1429`.
- `kitty/window.py:1453` — `cmd_output_marking` routing.
- `kitty/options/types.py:560` — default `notify_on_cmd_finish = NotifyOnCmdFinish(when='never',
  duration=5.0, action='notify', cmdline=())`; `kitty/options/definition.py:3190` — the
  `notify_on_cmd_finish` config-option default (`'never'`).
- `kitty/history.c:475` — scrollback search for literal `"\x1b]133;C\x1b\\"`.
- `kitty_tests/__init__.py:30` — `parse_bytes`; `:71-79` — `Callbacks` double (sentinel
  `sys.maxsize`); `:320` — `received_bytes` init, `:365-366` — raw-byte accumulation +
  `parse_bytes` dispatch; `:404-414` — `screen_contents` / `last_cmd_output`.
- `kitty_tests/screen.py:1059-1063`, `:1124` — classic prompt/output scenario & assertion.
- `shell-integration/**` — BEL-terminated OSC 133 emitters (fish `:83`; zsh `B` commented
  out `:222-226`; bash `:127`,`:208`).
- `setup.py:1084` — builds `kitty/fast_data_types`; `pyproject.toml:2` —
  `requires-python = ">=3.8"`; `go.mod:3` — `go 1.22`; `CONTRIBUTING.md` — tests via `./test.py`.

