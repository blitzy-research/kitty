# How kitty parses and records OSC 133 shell‑integration command‑boundary markers

**Repository:** `kovidgoyal/kitty` · **Branch:** `kitty_815df1e210e0` · **Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` · **kitty version:** `0.35.2`

This document answers three questions about how kitty's terminal emulator parses and records the
OSC 133 (FinalTerm / FTCS) semantic‑prompt markers `A` / `B` / `C` / `D`. Every value below was
**observed at runtime** by building kitty in its default configuration and driving the **real
compiled VT parser** (and, for the recorded exit status, the **real `kitty/window.py` receiver**)
inside the supplied Docker container. Each claim is paired with the exact command that produced
it, the complete unedited output, and a `file:line` citation naming the function that performs the
work.

---

## 1. Summary (direct answers)

- **Q1 — Baseline capture.** Feeding the BEL‑framed stream `OSC 133;A`, `OSC 133;B`,
  `OSC 133;C;cmdline=ls -la`, the literal text `total 0\n`, then `OSC 133;D;42` through the real
  parser yields **two different "captured outputs"**:
  - the **raw byte stream** that is fed to the parser still contains the OSC 133 escape bytes
    **verbatim** — its total length is **58 bytes** and the `D;42` marker's `D` character sits at
    **byte offset 53** (the numeric field `42` begins at offset 55);
  - the **rendered screen cell text** produced after parsing contains **only the literal output**
    `total 0` — the OSC 133 escape sequences have been **consumed by the parser and are absent**
    from the visible cells.
- **Q2 — Exit‑code variation.** For exit codes `0, 1, 42, 99, 127`, because the `D` marker is the
  **last** thing in the user's scenario, the **offset where the `D;<code>` marker starts is
  constant at 53 for every code** — it does *not* move. What changes is the **total stream length**,
  which grows by exactly the code's **digit‑count delta**: `57, 57, 58, 58, 59` bytes for
  `0, 1, 42, 99, 127`. (In the bash‑faithful variant where `D;<code>` is immediately followed by the
  next prompt's `OSC 133;A`, that trailing `A` marker's offset *does* shift — `57, 57, 58, 58, 59` —
  by the same digit‑count delta.) For code **99**, end‑to‑end through the real `Window` receiver, the
  parsed integer `Window.last_cmd_exit_status` becomes **`99`** and the command‑finish watcher
  payload carries **`'exit_status': 99`**.
- **Q3 — Malformed exit codes.** Feeding `OSC 133;D;not_a_number` and `OSC 133;D;` (empty) through
  the **real application receiver** `Window.handle_cmd_end` records **`0`** in **both** cases
  (`int(exit_status)` raises, the `except` clause assigns `0`). The test‑harness `Callbacks` receiver
  diverges (it uses `suppress` and **retains** its prior value `sys.maxsize = 9223372036854775807`);
  that value is **non‑canonical** and is reported only for comparison.

The remainder of this document shows the commands, the complete captured output, and the code
citations behind each of these answers.

---

## 2. Environment & build

All building and running was performed **inside the supplied Docker container**
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (Docker Hub reference
`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). The
container has the full C/Go toolchain and system libraries; the host inspection sandbox lacks the Go
compiler and `pkg-config` and therefore **cannot** build kitty (building there would yield a
non‑canonical artifact), so every runtime measurement below comes from the container.

Toolchain in the container: Python 3.12.3, Go 1.23.4, gcc 13.3.0, pkg‑config 1.8.1.

### 2.1 Build command (default / canonical)

The default build is `python3 setup.py` — this is exactly what the `Makefile` `all:` target runs
(`Makefile:L12-13` → `python3 setup.py $(VVAL)`):

```
$ cd /app                      # repository root, bind-mounted into the container
$ python3 setup.py             # canonical build (Makefile all:)
...
kitty/tools/cmd/at
kitty/tools/cmd/tool
kitty/tools/cmd/completion
SETUP_EXIT=0
```

Resulting artifacts (never committed — they are gitignored):

```
$ ls -la kitty/fast_data_types*.so kitty/launcher/kitty
-rwxr-xr-x 1 root root 1213072 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   36224 kitty/launcher/kitty
```

### 2.2 This is the canonical (no `DUMP_COMMANDS`) build

The `DUMP_COMMANDS` variant is a **separate compile** that is only defined for the aliased source
`kitty/vt-parser-dump.c`; the default `kitty/fast_data_types.so` compiles `kitty/vt-parser.c`
**without** `DUMP_COMMANDS` (`setup.py:L722`):

```
$ sed -n '721,722p' setup.py
    if src == 'kitty/vt-parser-dump.c':
        return 'kitty/vt-parser.c', [], ['DUMP_COMMANDS']
```

Because the canonical `.so` is built without `DUMP_COMMANDS`, the `#ifdef DUMP_COMMANDS`
`REPORT_OSC2(...)` test‑dump branch in the OSC dispatcher is compiled out and control reaches the
**real** `shell_prompt_marking()` call (see §3). This is confirmed at runtime in §3.3, where the
real `cmd_output_marking` callbacks fire (a dump build would instead emit `REPORT_OSC2` records).

### 2.3 Invocation of the observation scripts

Temporary observation scripts were placed under `/tmp` inside the container and run with the
**canonical launcher** (per the project's own test convention, which uses the launcher rather than a
bare `python3`):

```
$ cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py
```

(The full scripts are reproduced in the Appendix; they were deleted after the investigation and the
repository was verified clean — see §9.)

---

## 3. How OSC 133 is parsed & recorded (the dispatch chain)

### 3.1 Raw bytes → the real compiled parser

Bytes enter the real compiled VT parser through the headless entry point
`parse_bytes(screen, data)` (`kitty_tests/__init__.py:L30-36`), which drives the compiled parser via
the `Screen` test C‑API (`test_create_write_buffer` / `test_commit_write_buffer` /
`test_parse_written_data`):

```
kitty_tests/__init__.py:30   def parse_bytes(screen, data, dump_callback=None):
kitty_tests/__init__.py:31       data = memoryview(data)
kitty_tests/__init__.py:32       while data:
kitty_tests/__init__.py:33           dest = screen.test_create_write_buffer()
kitty_tests/__init__.py:34           s = screen.test_commit_write_buffer(data, dest)
kitty_tests/__init__.py:35           data = data[s:]
kitty_tests/__init__.py:36           screen.test_parse_written_data(dump_callback)
```

### 3.2 OSC dispatch → `shell_prompt_marking`

`dispatch_osc()` handles `case 133:` at `kitty/vt-parser.c:L536`. In the canonical build the
`#ifdef DUMP_COMMANDS` branch (`L537-540`) is compiled out and control reaches the real call
`shell_prompt_marking(self->screen, (char*)buf + i)` at `kitty/vt-parser.c:L544`:

```
kitty/vt-parser.c:536           case 133:
kitty/vt-parser.c:537   #ifdef DUMP_COMMANDS
kitty/vt-parser.c:538               START_DISPATCH
kitty/vt-parser.c:539               REPORT_OSC2(shell_prompt_marking, code, mv);
kitty/vt-parser.c:540               END_DISPATCH_WITHOUT_BREAK
kitty/vt-parser.c:541   #endif
kitty/vt-parser.c:542               if (limit > i) {
kitty/vt-parser.c:543                   buf[limit] = 0; // safe to do as we have 8 extra bytes after PARSER_BUF_SZ
kitty/vt-parser.c:544                   shell_prompt_marking(self->screen, (char*)buf + i);
kitty/vt-parser.c:545               }
kitty/vt-parser.c:546               break;
```

### 3.3 Marker interpretation in `shell_prompt_marking` — and the missing `case 'B'`

`shell_prompt_marking()` (`kitty/screen.c:L2327-2356`) switches on the first character `buf[0]`
(`L2331`). There are **only** `case 'A'`, `case 'C'`, and `case 'D'` — **there is no `case 'B'`**:

```
kitty/screen.c:2331           switch (ch) {
kitty/screen.c:2332               case 'A': {
kitty/screen.c:2333                   PromptKind pk = PROMPT_START;
kitty/screen.c:2334                   self->prompt_settings.redraws_prompts_at_all = 1;
kitty/screen.c:2335                   self->prompt_settings.uses_special_keys_for_cursor_movement = 0;
kitty/screen.c:2336                   parse_prompt_mark(self, buf+1, &pk);
kitty/screen.c:2337                   self->linebuf->line_attrs[self->cursor->y].prompt_kind = pk;
kitty/screen.c:2338                   if (pk == PROMPT_START) CALLBACK("cmd_output_marking", "O", Py_False);
kitty/screen.c:2339               } break;
kitty/screen.c:2340               case 'C': {
kitty/screen.c:2341                   self->linebuf->line_attrs[self->cursor->y].prompt_kind = OUTPUT_START;
kitty/screen.c:2342                   const char *cmdline = "";
kitty/screen.c:2343                   if (strstr(buf + 1, ";cmdline") == buf + 1) {
kitty/screen.c:2344                       cmdline = buf + 2;
kitty/screen.c:2345                   }
kitty/screen.c:2346                   RAII_PyObject(c, PyUnicode_DecodeUTF8(cmdline, strlen(cmdline), "replace"));
kitty/screen.c:2347                   if (c) { CALLBACK("cmd_output_marking", "OO", Py_True, c); }
kitty/screen.c:2348                   else PyErr_Print();
kitty/screen.c:2349               } break;
kitty/screen.c:2350               case 'D': {
kitty/screen.c:2351                   const char *exit_status = buf[1] == ';' ? buf + 2 : "";
kitty/screen.c:2352                   CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);
kitty/screen.c:2353               } break;
kitty/screen.c:2354           }
```

Key facts from this switch:
- **`A`** → fires `cmd_output_marking` with `Py_False` (`L2338`), i.e. Python receives `is_start=False`.
- **`C`** → fires `cmd_output_marking` with `Py_True` and the cmdline string; note `cmdline = buf + 2`
  (`L2344`), so Python receives the string **including** the literal `cmdline=` prefix (e.g.
  `cmdline=ls -la`), passed via the `"OO"` format (`L2347`).
- **`D`** → the exit status is `buf[1] == ';' ? buf + 2 : ""` (`L2351`) and is passed to Python as a
  **string** via the `"Os"` format (`L2352`). Conversion to an integer happens later, in Python.
- **`B`** → there is no `case 'B'`; the byte sequence `OSC 133;B` still reaches (and is consumed by)
  the parser, but it matches no case and is a **no‑op** that records no state and produces no callback.

`parse_prompt_mark()` (`kitty/screen.c:L2316-2325`) tokenizes the parameter tail on `;`, recognizing
`k=s`, `redraw=0`, and `special_key=1`.

#### Runtime proof of the three callback forms and the no‑op `B`

**Command** (PROBE C — records every `cmd_output_marking(is_start, data)` the real parser fires for
each marker fed individually; full script in Appendix §A.3):

```
$ cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_c.py
```

**Complete output:**

```
OSC 133;A                    -> cmd_output_marking calls = [('False', "''")] ; screen cells = ''
OSC 133;B                    -> cmd_output_marking calls = [] ; screen cells = ''
OSC 133;C;cmdline=ls -la     -> cmd_output_marking calls = [('True', "'cmdline=ls -la'")] ; screen cells = ''
OSC 133;D;42                 -> cmd_output_marking calls = [('None', "'42'")] ; screen cells = ''

Full scenario A,B,C,text,D;42 ordered cmd_output_marking calls:
  call 0: is_start=False data=''
  call 1: is_start=True data='cmdline=ls -la'
  call 2: is_start=None data='42'
screen cells after full scenario = 'total 0'
```

This directly confirms the C source: `A` → one callback with `is_start=False` (`screen.c:L2338`,
`Py_False`); **`B` → zero callbacks** (no `case 'B'`; the sequence is consumed — `screen cells = ''`
— but records nothing); `C` → one callback with `is_start=True` and `data='cmdline=ls -la'` (the
`cmdline=` prefix is included, per `screen.c:L2344`); `D` → one callback with `is_start=None` and the
exit status as the **string** `'42'` (`screen.c:L2352`). Corroborating the design intent, kitty's own
zsh integration leaves its `B` emitter **commented out** (`shell-integration/zsh/kitty-integration:L226`).

### 3.4 The real Python receiver — `kitty/window.py`

The shipped application binds the parser callbacks to `Window`. `cmd_output_marking()`
(`kitty/window.py:L1453-1461`) routes a truthy `is_start` (the `C` marker) to command‑start recording
and everything else — including the `D` marker, which arrives with `is_start=None` — to
`handle_cmd_end()`:

```
kitty/window.py:1453       def cmd_output_marking(self, is_start: Optional[bool], cmdline: str = '') -> None:
kitty/window.py:1454           if is_start:
kitty/window.py:1455               start_time = monotonic()
kitty/window.py:1456               self.last_cmd_output_start_time = start_time
kitty/window.py:1457               cmdline = decode_cmdline(cmdline) if cmdline else ''
kitty/window.py:1458               self.last_cmd_cmdline = cmdline
kitty/window.py:1459               self.call_watchers(self.watchers.on_cmd_startstop, {"is_start": True, "time": start_time, 'cmdline': cmdline, 'exit_status': 0})
kitty/window.py:1460           else:
kitty/window.py:1461               self.handle_cmd_end(cmdline)
```

`handle_cmd_end()` (`kitty/window.py:L1408-1432`) is where the exit status is recorded:

```
kitty/window.py:1408       def handle_cmd_end(self, exit_status: str = '') -> None:
kitty/window.py:1409           if self.last_cmd_output_start_time == 0.:
kitty/window.py:1410               return
kitty/window.py:1411           self.last_cmd_output_start_time = 0.
kitty/window.py:1412           try:
kitty/window.py:1413               self.last_cmd_exit_status = int(exit_status)
kitty/window.py:1414           except Exception:
kitty/window.py:1415               self.last_cmd_exit_status = 0
kitty/window.py:1416           end_time = monotonic()
kitty/window.py:1417           last_cmd_output_duration = end_time - self.last_cmd_output_start_time
kitty/window.py:1418
kitty/window.py:1419           self.call_watchers(self.watchers.on_cmd_startstop, {
kitty/window.py:1420               "is_start": False, "time": end_time, 'cmdline': self.last_cmd_cmdline, 'exit_status': self.last_cmd_exit_status})
kitty/window.py:1421
kitty/window.py:1422           opts = get_options()
kitty/window.py:1423           when, duration, action, notify_cmdline = opts.notify_on_cmd_finish
kitty/window.py:1424
kitty/window.py:1425           if last_cmd_output_duration >= duration and when != 'never':
kitty/window.py:1426               cmd = NotificationCommand()
kitty/window.py:1427               cmd.title = 'kitty'
kitty/window.py:1428               s = self.last_cmd_cmdline.replace('\\\n', ' ')
kitty/window.py:1429               cmd.body = f'Command {s} finished with status: {exit_status}.\nClick to focus.'
```

Relevant fields: `last_cmd_exit_status: int` is declared at `kitty/window.py:L244` and initialized to
`0` at `L572` (`last_cmd_output_start_time = 0.` at `L569`, `last_cmd_cmdline = ''` at `L571`). Screen
text capture is provided by the module‑level `cmd_output()` at `kitty/window.py:L457` (and the method
`Window.cmd_output` at `L1583`).

The default of `notify_on_cmd_finish` is `'never'` (`kitty/options/definition.py:L3190`
`opt('notify_on_cmd_finish', 'never', …)`), which is why the notification body at `L1429` is normally
suppressed (see §5.3 and §6).

### 3.5 The test‑harness receiver — `kitty_tests/__init__.py` (non‑canonical for exit status)

The headless harness drives the **real** C parser but with a **test** `Callbacks` receiver, whose
`cmd_output_marking` (`kitty_tests/__init__.py:L71-79`) records exit status differently — it uses
`with suppress(Exception)`:

```
kitty_tests/__init__.py:71       def cmd_output_marking(self, is_start: Optional[bool], data: str = '') -> None:
kitty_tests/__init__.py:72           if is_start:
kitty_tests/__init__.py:73               self.last_cmd_at = monotonic()
kitty_tests/__init__.py:74               self.last_cmd_cmdline = decode_cmdline(data) if data else data
kitty_tests/__init__.py:75           else:
kitty_tests/__init__.py:76               if self.last_cmd_at != 0:
kitty_tests/__init__.py:77                   self.last_cmd_at = 0
kitty_tests/__init__.py:78                   with suppress(Exception):
kitty_tests/__init__.py:79                       self.last_cmd_exit_status = int(data)
```

`Callbacks.last_cmd_exit_status` is initialized to `sys.maxsize` (`L48`, reset to `sys.maxsize` in
`clear()` at `L106`). Because `suppress` swallows the `ValueError` from `int()` on a malformed value,
the assignment is skipped and the **prior value is retained** — this diverges from the real `Window`
receiver and is treated as **non‑canonical** for Q3 (see §5.4 and §6).

---

## 4. Q1 — Baseline capture

**Scenario (BEL‑framed, exactly as kitty's own shell integration frames OSC 133):**

```python
ESC = b'\x1b'; BEL = b'\x07'
def osc(payload): return ESC + b']' + payload + BEL
data = osc(b'133;A') + osc(b'133;B') + osc(b'133;C;cmdline=ls -la') + b'total 0\n' + osc(b'133;D;42')
```

**Command** (PROBE A; full script in Appendix §A.1):

```
$ cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py
```

**Complete, unedited output — Q1 section:**

```
========================================================================
Q1 -- BASELINE CAPTURE (code=42), RAW byte stream measurements
========================================================================
marker byte lengths:
  len(A                 ) = 8   repr=b'\x1b]133;A\x07'
  len(B                 ) = 8   repr=b'\x1b]133;B\x07'
  len(C;cmdline=ls -la  ) = 23   repr=b'\x1b]133;C;cmdline=ls -la\x07'
  len(D;42              ) = 11   repr=b'\x1b]133;D;42\x07'
literal text repr = b'total 0\n'  len = 8
RAW stream repr   = b'\x1b]133;A\x07\x1b]133;B\x07\x1b]133;C;cmdline=ls -la\x07total 0\n\x1b]133;D;42\x07'
len(stream)                = 58
stream.find(b"D;42")       = 53
stream.find(b"\x1b]133;D") = 47
numeric field begins at    = 55 (after "D;")
OSC-133 bytes present in RAW stream?  True

--- Rendered SCREEN cell text after parse_bytes (OSC consumed) ---
screen_contents(screen) repr = 'total 0'
ESC (0x1b) byte present in screen text?  False
substring "]133;" present in screen text?  False
literal "total 0" present in screen text?   True
non-blank screen lines (y, text): [(0, 'total 0')]
kitty.window.cmd_output(screen, CommandOutput.last_run) repr = 'total 0\n'
```

### 4.1 There are two distinct "captured outputs"

The word *captured* is answerable two ways, and the answer to Q1(a)/(b) depends on which one:

- **RAW byte stream** — the exact bytes fed to the parser (and, in the `PTY` harness, accumulated in
  `received_bytes`, `kitty_tests/__init__.py:L320` / `L365`). Here the OSC 133 escape bytes are
  **present verbatim**, as the `repr` above shows:
  `b'\x1b]133;A\x07\x1b]133;B\x07\x1b]133;C;cmdline=ls -la\x07total 0\n\x1b]133;D;42\x07'`.
  `OSC-133 bytes present in RAW stream? True`. **The byte length and the `D;42` offset are measured on
  this stream.**
- **Rendered SCREEN cell text** — obtained after `parse_bytes` either by reading the screen lines
  (`str(screen.line(i))`, exactly as `PTY.screen_contents`, `kitty_tests/__init__.py:L404-410`) or via
  `kitty.window.cmd_output(screen, CommandOutput.last_run)` (`kitty/window.py:L457`). Here the parser
  has **consumed** the OSC 133 sequences, so they are **absent**: `ESC (0x1b) byte present in screen
  text? False`, `substring "]133;" present in screen text? False`, and only the literal command output
  survives — `screen_contents = 'total 0'`, the single non‑blank line `(0, 'total 0')`, and
  `cmd_output(...) = 'total 0\n'`.

### 4.2 The four sub‑answers

- **Q1(a) — What is actually captured?** Two things: the **raw stream** (all bytes, OSC included) and
  the **screen cells** (only the literal `total 0`). The `cmdline=ls -la` from the `C` marker is not a
  screen cell either — it is delivered to the receiver as a callback argument (see §3.3), not rendered.
- **Q1(b) — Are the OSC sequences still present?** **Yes in the raw stream** (`True`), **No in the
  rendered screen text** (`False` for both the `ESC` byte and the `]133;` substring). The parser
  consumes OSC 133 as control data; it never becomes visible cells.
- **Q1(c) — Total byte length of the captured stream?** **`len(stream) = 58` bytes.** This is the sum
  of the marker lengths and the literal text: `A(8) + B(8) + C;cmdline=ls -la(23) + total 0\n(8) +
  D;42(11) = 58`.
- **Q1(d) — Byte offset of the `D;42` marker?** **`stream.find(b"D;42") = 53`** — the `D` character is
  at offset 53. Equivalently the full `D` marker `\x1b]133;D` begins at offset 47
  (`stream.find(b"\x1b]133;D") = 47`), and the numeric field `42` begins at offset 55.

These numbers are **stable across two runs** (byte‑identical; see §7). The `find`/`len` are computed
directly on the exact emitted bytes, not on a re‑serialized copy.


---

## 5. Q2 — Exit‑code variation

### 5.1 Byte lengths and marker offsets per code (D‑last, the user's exact scenario)

**Command** (PROBE A; same run as §4):

```
$ cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py
```

**Complete, unedited output — Q2 sweep section:**

```
========================================================================
Q2 -- EXIT-CODE SWEEP 0,1,42,99,127 (D-last, user scenario)
========================================================================
code   digits   len(stream)    find(b"D;<code>")
0      1        57             53              
1      1        57             53              
42     2        58             53              
99     2        58             53              
127    3        59             53              
```

| exit code | digit count | `len(stream)` | `find(b"D;<code>")` (start offset of the `D` marker) |
|:---------:|:-----------:|:-------------:|:----------------------------------------------------:|
| `0`       | 1           | **57**        | **53** |
| `1`       | 1           | **57**        | **53** |
| `42`      | 2           | **58**        | **53** |
| `99`      | 2           | **58**        | **53** |
| `127`     | 3           | **59**        | **53** |

### 5.2 Q2(b) — Does the position of the exit‑code number shift, and by how much?

**Direct answer (the plain reading, observed):** In the user's exact scenario the `D` marker is the
**last** thing emitted, so **the offset at which the `D;<code>` marker (and hence the exit‑code number)
begins does NOT move — it is `53` for every code `0/1/42/99/127`.** Everything before the `D` marker is
byte‑identical across the codes, so its start offset is constant.

What *does* change is the **total stream length**, which grows by exactly the **digit‑count delta** of
the code: `57` bytes for the 1‑digit codes `0` and `1`, `58` bytes for the 2‑digit codes `42` and `99`,
and `59` bytes for the 3‑digit code `127`. In other words the trailing part of the stream (the numeric
field and the final BEL) lengthens by one byte per extra digit, while its **start** stays fixed.

Determinism/reasoning: the offset of the `D` marker is governed purely by the fixed‑length prefix
(`A(8) + B(8) + C;cmdline=ls -la(23) + total 0\n(8) = 47` bytes up to the `D` marker's `ESC`; the `D`
character then sits 6 bytes later at offset 53). Since that prefix never varies, the start offset is
constant; only the code's digit count changes the tail length.

**Bash‑faithful variant (where an offset *does* shift).** kitty's bash integration emits
`D;<code>` immediately followed by the next prompt's `OSC 133;A` (`shell-integration/bash/kitty.bash:L239`,
`\[\e]133;D;\$?\a\e]133;A\a\]`). When a trailing `OSC 133;A` follows the `D` marker, *that trailing
`A` marker's* offset shifts by the digit‑count delta, because it now sits after the variable‑length
`D;<code>` marker:

```
========================================================================
Q2 -- BASH-FAITHFUL variant (D;<code> immediately followed by OSC 133;A, per bash:L239)
========================================================================
code   digits   len(stream)    find(b"D;<code>")  find(trailing A)  
0      1        65             53                 57                
1      1        65             53                 57                
42     2        66             53                 58                
99     2        66             53                 58                
127    3        67             53                 59                
```

Here the `D` marker still starts at the constant offset `53`, but the trailing `A` marker moves:
`57` (1‑digit) → `58` (2‑digit) → `59` (3‑digit) — again exactly the digit‑count delta. This is the
sense in which "the marker after the exit code shifts": anything emitted *after* `D;<code>` moves by
the number of extra digits.

### 5.3 Q2 for code `99` — end‑to‑end evidence through the real `Window` receiver

For the recorded exit status, the canonical source is the **real `kitty/window.py` `Window` receiver**,
not the test‑harness `Callbacks`. A genuine `Window` requires a live `Boss` / OS‑window / child process
and cannot be constructed headlessly, so — per the sanctioned fallback — PROBE B executes the **real,
unmodified shipped bytecode** of `Window.cmd_output_marking` (`kitty/window.py:L1453-1461`) and
`Window.handle_cmd_end` (`kitty/window.py:L1408-1432`) bound to a scaffolded state object. **Only the
surrounding state is scaffolded** (and the terminal `notify_with_command` collaborator is captured to
avoid emitting a real desktop notification); the `int()`/`except` logic, the watcher payload, the option
gate, and the body string are all produced by the real bytecode. The recording is **armed by the real
`C` marker path** first (a prior `C` is required — see §6, nuance #3).

**Command** (PROBE B; full script in Appendix §A.2):

```
$ cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_b.py
```

**Complete, unedited output (RUN 1; RUN 2 is identical except for the monotonic `time` field — see §7):**

```
========================================================================
CANONICAL real-Window receiver: kitty.window.Window (executed bytecode, scaffolded state)
========================================================================
Genuine Window needs a live Boss/OS-window/child -> impractical headlessly;
executing the REAL Window.cmd_output_marking / Window.handle_cmd_end bytecode bound to a scaffold.
[env] kitty.fast_data_types.monotonic() sample = 0.034037024  (SMALL: seconds since monotonic-clock start)

########################## RUN 1 ##########################
[default] get_options().notify_on_cmd_finish = ('never', 5.0, 'notify', ())
[default] after C-arm: last_cmd_output_start_time != 0 -> True ; last_cmd_cmdline = 'ls'
[default] Q2 code=99: last_cmd_exit_status  before=0  after=99  type(after)=int
[default] Q2 code=99: watcher payload (on_cmd_startstop, is_start=False) = {'is_start': False, 'time': 0.035097171, 'cmdline': 'ls', 'exit_status': 99}
[default] Q2 code=99: notify_with_command call count = 0 -> notification body SUPPRESSED under default (gate L1425: when != "never" is False)
[default] Q3 malformed exit_status='not_a_number' : last_cmd_exit_status  before=999  after=0  (canonical Window)
           watcher payload exit_status = 0
[default] Q3 malformed exit_status='' : last_cmd_exit_status  before=999  after=0  (canonical Window)
           watcher payload exit_status = 0
[toggle 'always' ] notify_on_cmd_finish = ('always', 5.0, 'notify', ())
[toggle 'always' ] Q2 code=99: last_cmd_exit_status = 99 ; notify_with_command call count = 0
[toggle 'always 0'] notify_on_cmd_finish = ('always', 0.0, 'notify', ())
[toggle 'always 0'] Q2 code=99: last_cmd_exit_status = 99 ; notify_with_command call count = 1
[toggle 'always 0'] Q2 code=99: NotificationCommand.body = 'Command ls finished with status: 99.\nClick to focus.'
[state] Window init last_cmd_exit_status (before any marker) = 0
[state] after C (armed), before D: last_cmd_exit_status = 0
[state] after D;99                : last_cmd_exit_status = 99
```

**Default‑config canonical evidence for code `99`:**
- The parsed integer `Window.last_cmd_exit_status` becomes **`99`** (`kitty/window.py:L1413`), an `int`
  (`type(after)=int`), transitioning from its init `0`.
- The **unconditional** command‑finish watcher payload (`kitty/window.py:L1419-1420`) carries
  **`'exit_status': 99`**: `{'is_start': False, 'time': 0.035097171, 'cmdline': 'ls', 'exit_status': 99}`.
  (`cmdline` is `'ls'` because `decode_cmdline('cmdline=ls -la')` returns the first shlex token,
  `kitty/window.py:L225-232`; this does not affect the recorded exit status.)
- **Notification body under the default config:** `notify_with_command` is called **0 times** — the
  body is **not constructed**. The default `notify_on_cmd_finish` is `('never', 5.0, 'notify', ())`
  (`kitty/options/definition.py:L3190`) and the gate at `kitty/window.py:L1425`
  (`if last_cmd_output_duration >= duration and when != 'never'`) fails because `when == 'never'`.

**Non‑default toggle to exhibit the body text (explicitly labeled NON‑DEFAULT).** Setting
`notify_on_cmd_finish` to a non‑`'never'` value makes the body constructible. Note the honest subtlety:
with `'always'` (which keeps the default 5.0‑second duration threshold) the body is **still not built**
in this environment — `notify_with_command call count = 0` — because the duration gate is not satisfied
(see §6, nuance #2, and the discrepancy note below). Dropping the threshold to zero with `'always 0'`
does surface the body:

- `[toggle 'always 0'] NotificationCommand.body = 'Command ls finished with status: 99.\nClick to
  focus.'` — the body, built by the real `kitty/window.py:L1429`, contains **`99`** (from the **raw
  string** `exit_status`, not the parsed int).

> **Observed discrepancy with the reference reasoning (reported honestly).** A reference note
> assumed that because `handle_cmd_end` zeroes `last_cmd_output_start_time` at `L1411` *before*
> computing `last_cmd_output_duration = end_time - self.last_cmd_output_start_time` at `L1417`, the
> duration would be a *huge* monotonic value that always satisfies the threshold — leaving `when !=
> 'never'` as the only effective gate. **What I actually observed is that kitty's `monotonic()` returns
> a SMALL value** (`0.034037024` seconds at process start; the watcher `time` field is likewise
> `~0.035`). So `last_cmd_output_duration ≈ 0.035` seconds, and the gate `0.035 >= 5.0` is **False**.
> The ordering "bug" (subtracting against the just‑zeroed start time, so the computed "duration" is the
> absolute monotonic timestamp rather than the true command elapsed time) **is real**, but its
> magnitude is small, not huge — so `when != 'never'` is *not* the only effective gate: the duration
> threshold matters too. That is why `'always'` (threshold 5.0) does not surface the body here, while
> `'always 0'` (threshold 0.0) does. I report the observation over the reference assumption.


---

## 6. Q3 — Malformed exit codes

Feeding `OSC 133;D;not_a_number` and `OSC 133;D;` (empty exit status). The exit status arrives at the
receiver as a **string** (`kitty/screen.c:L2352`, format `"Os"`); the string→int conversion happens
only in the receiver. The behavior therefore diverges between the two receivers.

### 6.1 Canonical answer — the real `Window` receiver records `0` for both

Through the real `Window.handle_cmd_end` (`kitty/window.py:L1408-1415`), `int(exit_status)` raises and
the `except` clause **actively assigns `0`** (`L1415`). To prove the `except` clause *writes* `0`
(rather than merely leaving a pre‑existing `0`), PROBE B seeds a sentinel `999` before feeding the
malformed value.

**Command** (PROBE B; same run as §5.3):

```
$ cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_b.py
```

**Relevant output lines (canonical `Window` path):**

```
[default] Q3 malformed exit_status='not_a_number' : last_cmd_exit_status  before=999  after=0  (canonical Window)
           watcher payload exit_status = 0
[default] Q3 malformed exit_status='' : last_cmd_exit_status  before=999  after=0  (canonical Window)
           watcher payload exit_status = 0
```

- **`OSC 133;D;not_a_number`** → `Window.last_cmd_exit_status` transitions `999 → 0`. Recorded value:
  **`0`**.
- **`OSC 133;D;` (empty)** → `Window.last_cmd_exit_status` transitions `999 → 0`. Recorded value:
  **`0`**.
- The command‑finish watcher payload (`kitty/window.py:L1419-1420`) likewise carries
  **`'exit_status': 0`** in both cases.

So the canonical, default‑config answer to Q3 is **`0` for both malformed inputs**, because the
`except Exception: self.last_cmd_exit_status = 0` branch (`kitty/window.py:L1414-1415`) runs.

### 6.2 Non‑canonical comparison — the test‑harness `Callbacks` retains `sys.maxsize`

The test‑harness `Callbacks` receiver (`kitty_tests/__init__.py:L71-79`) uses `with
suppress(Exception)` around `self.last_cmd_exit_status = int(data)` (`L78-79`). On a malformed value
the `int()` raises, `suppress` swallows it, the assignment is **skipped**, and the **prior value is
retained**. This value is **non‑canonical** (it is the test harness, not the shipped application) and
is shown only for comparison.

**Relevant output lines (PROBE A, non‑canonical `Callbacks` path):**

```
========================================================================
NON-CANONICAL: Callbacks recorded exit status (test-harness receiver)
========================================================================
sys.maxsize = 9223372036854775807
code='42'           (valid    ) before=9223372036854775807  after=42                   (recorded)
code='99'           (valid    ) before=9223372036854775807  after=99                   (recorded)
code='not_a_number' (malformed) before=9223372036854775807  after=9223372036854775807  (retained sys.maxsize)
code=''             (empty    ) before=9223372036854775807  after=9223372036854775807  (retained sys.maxsize)
```

- Valid codes `42`/`99` → recorded as `42`/`99` (here the two receivers agree, since `int()` succeeds).
- Malformed `not_a_number` and empty `''` → **retains `sys.maxsize = 9223372036854775807`** (the init
  value, `kitty_tests/__init__.py:L48`). **This is NON‑CANONICAL.**

### 6.3 Why they diverge

Both receivers call `int(<string>)`, which raises `ValueError` on `'not_a_number'` and on `''`. The
difference is purely the error‑handling construct:
- **`Window`** wraps it in `try: … except Exception: self.last_cmd_exit_status = 0`
  (`kitty/window.py:L1412-1415`) → the field is **set to `0`**.
- **`Callbacks`** wraps it in `with suppress(Exception): self.last_cmd_exit_status = int(data)`
  (`kitty_tests/__init__.py:L78-79`) → the assignment is **skipped**, so the field **keeps its prior
  value**.

The canonical answer to "what value actually gets recorded" is therefore **`0`** (the real `Window`
application path).

---

## 7. State transitions & stability

### 7.1 State transitions of `last_cmd_exit_status` (before / during / after the `D` marker)

The recording is armed by a prior `C` marker (nuance #3 below); the `D` marker then records the status.
From PROBE B (real `Window`), the observed transition for the valid code `99`:

```
[state] Window init last_cmd_exit_status (before any marker) = 0
[state] after C (armed), before D: last_cmd_exit_status = 0
[state] after D;99                : last_cmd_exit_status = 99
```

- **Before any marker:** `0` (the `Window` init at `kitty/window.py:L572`).
- **After `C` (armed), before `D`:** still `0` — the `C` marker arms the recording
  (`last_cmd_output_start_time` becomes non‑zero) but does not touch the exit status.
- **After `D;99`:** `99` (recorded by `int('99')` at `kitty/window.py:L1413`).

For the malformed inputs (§6.1) the transition is `999 (seeded sentinel) → 0`, proving the `except`
branch writes `0`. For the non‑canonical `Callbacks` (§6.2) the transition on malformed input is
`sys.maxsize → sys.maxsize` (no change).

Nuance #3 in action — the `D` handler is **gated by a prior `C`**: `handle_cmd_end` returns
immediately unless `last_cmd_output_start_time != 0` (`kitty/window.py:L1409-1410`); the `Callbacks`
path is gated by `last_cmd_at != 0` (`kitty_tests/__init__.py:L76`). This is why the scenario ordering
`A → B → C → text → D` matters: the `C` marker arms the recording and the first `D` after it records
the status.

### 7.2 Stability across ≥2 runs

- **PROBE A** (Q1 measurements and the Q2 sweep) is **byte‑for‑byte identical across two runs**:

```
$ ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py > run1.txt
$ ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py > run2.txt
$ diff run1.txt run2.txt && echo IDENTICAL
IDENTICAL: PROBE A output is byte-for-byte stable across 2 runs
```

- **PROBE C** (dispatch‑chain evidence) is likewise **byte‑for‑byte identical across two runs**:

```
$ diff probe_c_run1.txt probe_c_run2.txt && echo IDENTICAL
IDENTICAL: PROBE C output is byte-for-byte stable across 2 runs
```

- **PROBE B** runs the canonical `Window` measurements twice internally (`RUN 1` / `RUN 2`). Every
  recorded value is identical across the two runs — `last_cmd_exit_status` (`99`; `0`; `0`), the
  watcher payload `exit_status` (`99`; `0`; `0`), the notify call counts (`0`; `0`; `1`), and the body
  string. The **only** field that differs between runs is the monotonic `time` value in the watcher
  payload (`0.035097171` in RUN 1 vs `0.041982918` in RUN 2), which is an inherent timestamp and not
  one of the reported quantities.

All reported byte lengths, offsets, and recorded exit statuses are therefore stable.

---

## 8. Reasoning & consolidated citations

**Why the raw stream keeps the OSC bytes but the screen does not (Q1).** The VT parser treats OSC 133
as control data: `dispatch_osc()` routes `case 133` (`kitty/vt-parser.c:L536-546`) to
`shell_prompt_marking` (`kitty/screen.c:L2327-2356`), which updates line attributes / fires callbacks
but **never writes cells**. The escape bytes therefore exist only in the input stream, not in the
rendered screen — exactly what the `repr` (OSC present) vs `screen_contents` (`'total 0'`, OSC absent)
outputs show.

**Why the `D;<code>` start offset is constant but the length grows (Q2).** The bytes before the `D`
marker are a fixed‑length prefix (47 bytes to the `D` marker's `ESC`, `D` char at 53), so the marker's
start offset cannot move; only the code's digit count changes the tail length, giving lengths
`57/57/58/58/59`. Anything emitted *after* `D;<code>` (e.g. the bash‑faithful trailing `A`) shifts by
the digit‑count delta.

**Why malformed exit codes record `0` in the real app (Q3).** The exit status is delivered as a string
(`kitty/screen.c:L2351-2352`) and converted with `int()` only in `handle_cmd_end`; the real `Window`
uses `try/except → 0` (`kitty/window.py:L1412-1415`), so a non‑numeric or empty string yields `0`. The
harness `Callbacks` uses `suppress`, retaining its prior value — a divergence that is non‑canonical.

**Consolidated `file:line` references:**

- `kitty_tests/__init__.py:L30-36` — `parse_bytes`, the real headless entry point into the compiled parser.
- `kitty_tests/__init__.py:L48` — `Callbacks.last_cmd_exit_status = sys.maxsize` (init; non‑canonical baseline).
- `kitty_tests/__init__.py:L71-79` — `Callbacks.cmd_output_marking`; malformed path uses `suppress` (`L78-79`).
- `kitty_tests/__init__.py:L320, L365, L404-410` — `PTY.received_bytes` accumulation; `screen_contents` line reading.
- `kitty/vt-parser.c:L536-546` — OSC `case 133`; `#ifdef DUMP_COMMANDS` dump branch (compiled out) vs the real `shell_prompt_marking` call at `L544`.
- `kitty/screen.c:L2316-2325` — `parse_prompt_mark` (param tokenizer).
- `kitty/screen.c:L2327-2356` — `shell_prompt_marking`; `case 'A'` (`L2338`, `Py_False`), `case 'C'` (`L2344`, `L2347`, `Py_True`), `case 'D'` (`L2351-2352`, `Py_None`, `"Os"` string); **no `case 'B'`**.
- `kitty/window.py:L244, L569, L571, L572` — `last_cmd_exit_status`/`last_cmd_output_start_time`/`last_cmd_cmdline` declaration and init.
- `kitty/window.py:L1453-1461` — `cmd_output_marking` (routes `C` to arm, `D` to `handle_cmd_end`).
- `kitty/window.py:L1408-1432` — `handle_cmd_end`: guard (`L1409`), `try int` (`L1413`), `except → 0` (`L1415`), watcher payload (`L1419-1420`), notify gate (`L1425`), body (`L1429`).
- `kitty/window.py:L225-232` — `decode_cmdline`. `kitty/window.py:L457` — module `cmd_output`.
- `kitty/options/definition.py:L3190` — `notify_on_cmd_finish` default `'never'`. `kitty/options/utils.py:L753-779` — `NotifyOnCmdFinish` parser.
- `shell-integration/bash/kitty.bash:L208, L239` — BEL‑framed `C;cmdline` and `D;$?` + trailing `A`.
- `shell-integration/zsh/kitty-integration:L145, L149, L218, L226` — BEL‑framed emitters; commented‑out `B` at `L226`.
- `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:L83, L85, L91, L96` — BEL‑framed emitters.
- `setup.py:L722` — `DUMP_COMMANDS` separate compile variant. `Makefile:L12-13` — `all:` → `python3 setup.py`.


---

## 9. Reproducibility appendix

### 9.1 Exact commands

All commands were executed inside the container (host wrapper shown once, then the in‑container form).
The long‑lived container `kitty_setup` bind‑mounts the repository at `/app`:

```
# Build (canonical) — inside the container, from the repo root:
cd /app && python3 setup.py

# Run each probe via the canonical launcher (scripts kept under /tmp, never in the repo):
cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py
cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_b.py
cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_c.py

# Host-side wrapper actually used (container name kitty_setup; required env):
docker exec -e LANG=C.UTF-8 -e LC_ALL=C.UTF-8 -e TMPDIR=/fasttmp kitty_setup \
  bash -lc 'cd /app && ./kitty/launcher/kitty +launch /tmp/osc133_probe_a.py'
```

### 9.2 Cleanup — repository left unchanged

The temporary probe scripts were removed after the investigation and the working tree verified clean.
The only retained artifact is this document.

```
$ rm -f /tmp/osc133_probe_*.py            # inside the container
$ find . -name 'osc133_probe*' | grep -v '/.git/'      # in the repo tree
none in repo (good)
$ git status --porcelain
?? blitzy/documentation/kitty_815df1e210e0.md
```

Build artifacts (`kitty/fast_data_types*.so`, `kitty/launcher/kitty`, `build/`) are gitignored and are
**not** committed.

### 9.3 PROBE A — `osc133_probe_a.py` (Q1 + Q2 lengths/offsets + non‑canonical `Callbacks`)

```python
#!/usr/bin/env python3
# PROBE A -- parser-level OSC 133 investigation via the REAL compiled VT parser.
# Drives kitty_tests.parse_bytes(screen, data) against a real Screen bound to the
# test Callbacks receiver. Measures Q1 (baseline capture) and Q2 (exit-code sweep)
# byte lengths / marker offsets on the EXACT emitted bytes, reads rendered screen
# cells to show the OSC bytes are consumed, and exhibits the (non-canonical)
# Callbacks recorded exit status.
import sys

# --- Set default (canonical) options, exactly as kitty_tests.BaseTest.set_options ---
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty.fast_data_types import Screen, set_options, get_options

def install_default_options():
    final_options = {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}
    options = Options(merge_result_dicts(defaults._asdict(), final_options))
    finalize_keys(options, {})
    finalize_mouse_mappings(options, {})
    set_options(options)
    return options

install_default_options()

from kitty_tests import Callbacks, parse_bytes
from kitty.window import cmd_output, CommandOutput

# --- BEL-framed OSC 133 emitter, matching kitty's own shell-integration scripts ---
ESC = b'\x1b'; BEL = b'\x07'
def osc(payload: bytes) -> bytes:
    return ESC + b']' + payload + BEL

def new_screen(cb, lines=24, cols=80, scrollback=100, cw=10, ch=20):
    # Screen ctor per kitty_tests create_screen (L237-241) / PTY (L319)
    return Screen(cb, lines, cols, scrollback, cw, ch, 0, cb)

CMDLINE = b'133;C;cmdline=ls -la'
TEXT = b'total 0\n'

def build_stream(code_bytes: bytes, trailing_A: bool = False) -> bytes:
    data = osc(b'133;A') + osc(b'133;B') + osc(CMDLINE) + TEXT + osc(b'133;D;' + code_bytes)
    if trailing_A:
        data += osc(b'133;A')  # bash-faithful: D;<code> immediately followed by next prompt A (bash L239)
    return data

def hr(t):
    print('\n' + '=' * 72 + '\n' + t + '\n' + '=' * 72)

# ======================================================================
hr('Q1 -- BASELINE CAPTURE (code=42), RAW byte stream measurements')
data = build_stream(b'42')
print('marker byte lengths:')
for name, b in [('A', osc(b'133;A')), ('B', osc(b'133;B')),
                ('C;cmdline=ls -la', osc(CMDLINE)), ('D;42', osc(b'133;D;42'))]:
    print('  len(%-18s) = %d   repr=%r' % (name, len(b), b))
print('literal text repr =', repr(TEXT), ' len =', len(TEXT))
print('RAW stream repr   =', repr(data))
print('len(stream)                =', len(data))
print('stream.find(b"D;42")       =', data.find(b'D;42'))
print('stream.find(b"\\x1b]133;D") =', data.find(b'\x1b]133;D'))
print('numeric field begins at    =', data.find(b'D;42') + 2, '(after "D;")')
print('OSC-133 bytes present in RAW stream? ', (b'\x1b]133;' in data))

# Feed through the REAL compiled parser
cb = Callbacks()
screen = new_screen(cb)
parse_bytes(screen, data)

print('\n--- Rendered SCREEN cell text after parse_bytes (OSC consumed) ---')
# Read visible cells the same way kitty_tests PTY.screen_contents does (L404-410): str(screen.line(i))
def screen_contents(scr):
    out = []
    for i in range(scr.lines):
        x = str(scr.line(i))
        if x:
            out.append(x)
    return '\n'.join(out)
visible = screen_contents(screen)
print('screen_contents(screen) repr =', repr(visible))
print('ESC (0x1b) byte present in screen text? ', ('\x1b' in visible))
print('substring "]133;" present in screen text? ', (']133;' in visible))
print('literal "total 0" present in screen text?  ', ('total 0' in visible))
nonblank = [ (y, str(screen.line(y))) for y in range(screen.lines) if str(screen.line(y)).strip() ]
print('non-blank screen lines (y, text):', nonblank)
co = cmd_output(screen, CommandOutput.last_run)
print('kitty.window.cmd_output(screen, CommandOutput.last_run) repr =', repr(co))

# ======================================================================
hr('Q2 -- EXIT-CODE SWEEP 0,1,42,99,127 (D-last, user scenario)')
print('%-6s %-8s %-14s %-16s' % ('code', 'digits', 'len(stream)', 'find(b"D;<code>")'))
for code in ['0', '1', '42', '99', '127']:
    d = build_stream(code.encode())
    print('%-6s %-8d %-14d %-16d' % (code, len(code), len(d), d.find(b'D;' + code.encode())))

hr('Q2 -- BASH-FAITHFUL variant (D;<code> immediately followed by OSC 133;A, per bash:L239)')
print('%-6s %-8s %-14s %-18s %-18s' % ('code', 'digits', 'len(stream)', 'find(b"D;<code>")', 'find(trailing A)'))
for code in ['0', '1', '42', '99', '127']:
    d = build_stream(code.encode(), trailing_A=True)
    trailing_A_off = d.rfind(osc(b'133;A'))
    print('%-6s %-8d %-14d %-18d %-18d' % (code, len(code), len(d), d.find(b'D;' + code.encode()), trailing_A_off))

# ======================================================================
hr('NON-CANONICAL: Callbacks recorded exit status (test-harness receiver)')
print('sys.maxsize =', sys.maxsize)
for code in ['42', '99', 'not_a_number', '']:
    cb = Callbacks()
    scr = new_screen(cb)
    before = cb.last_cmd_exit_status
    d = build_stream(code.encode())
    parse_bytes(scr, d)
    after = cb.last_cmd_exit_status
    label = 'valid' if code.isdigit() else ('empty' if code == '' else 'malformed')
    print('code=%-14r (%-9s) before=%-20d after=%-20d %s'
          % (code, label, before, after,
             '(retained sys.maxsize)' if after == sys.maxsize else '(recorded)'))
```

### 9.4 PROBE B — `osc133_probe_b.py` (canonical real‑`Window` path: Q2 code=99 + Q3 malformed)

```python
#!/usr/bin/env python3
# PROBE B -- CANONICAL recorded-exit-status investigation via the REAL shipped
# kitty.window.Window receiver.  We execute the REAL, UNMODIFIED bytecode of
# Window.cmd_output_marking (kitty/window.py:L1453-1461) and Window.handle_cmd_end
# (L1408-1451) bound to a scaffolded state object.  A genuine Window requires a
# live Boss / OS-window / child process and cannot be built headlessly, so per the
# sanctioned fallback we bind the real function objects to a types.SimpleNamespace
# that supplies exactly the attributes the real functions read/write.  ONLY the
# surrounding state (and the terminal notify_with_command collaborator, which we
# capture to avoid emitting a real desktop notification) is scaffolded -- the
# int()/except logic, the watcher payload, the option gate and the body string are
# all produced by the real shipped bytecode.
import sys, types
import kitty.window as W
from kitty.window import Window

from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty.fast_data_types import set_options, get_options
from kitty.options.utils import notify_on_cmd_finish as parse_notify_on_cmd_finish

def install_options(notify=None):
    final_options = {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}
    if notify is not None:
        # Parse the raw config string into the NotifyOnCmdFinish namedtuple exactly
        # as kitty's own option loader does (kitty/options/utils.py:L760-779).
        final_options['notify_on_cmd_finish'] = parse_notify_on_cmd_finish(notify)
    options = Options(merge_result_dicts(defaults._asdict(), final_options))
    finalize_keys(options, {})
    finalize_mouse_mappings(options, {})
    set_options(options)
    return options

# Capture (not send) desktop notifications: the body STRING is still built by the
# real handle_cmd_end bytecode at L1429; we only intercept the collaborator.
NOTIFY_CALLS = []
_real_notify = W.notify_with_command
def fake_notify_with_command(cmd, window_id, notify_implementation=None):
    NOTIFY_CALLS.append(cmd)
W.notify_with_command = fake_notify_with_command

def make_scaffold():
    s = types.SimpleNamespace()
    s.last_cmd_output_start_time = 0.0     # window.py L569 init; guard reads L1409
    s.last_cmd_exit_status = 0             # window.py L572 init
    s.last_cmd_cmdline = ''                # window.py L571 init
    s.id = 1
    s.watchers = types.SimpleNamespace(on_cmd_startstop=[])
    s.captured_payloads = []
    def call_watchers(which, data):        # capture real payload dict (L1419-1420 / L1459)
        s.captured_payloads.append(dict(data))
    s.call_watchers = call_watchers
    # Bind the REAL shipped bytecode as bound methods of the scaffold:
    s.cmd_output_marking = Window.cmd_output_marking.__get__(s)
    s.handle_cmd_end = Window.handle_cmd_end.__get__(s)
    return s

def hr(t):
    print('\n' + '=' * 72 + '\n' + t + '\n' + '=' * 72)

def measure(tag):
    print('\n########################## %s ##########################' % tag)

    # ---------- Q2 code=99, DEFAULT config (notify_on_cmd_finish='never') ----------
    install_options()  # default
    opts = get_options()
    print('[default] get_options().notify_on_cmd_finish =', tuple(opts.notify_on_cmd_finish))
    s = make_scaffold()
    NOTIFY_CALLS.clear()
    # Arm via the REAL 'C' path (cmd_output_marking with is_start truthy):
    s.cmd_output_marking(True, 'cmdline=ls -la')
    print('[default] after C-arm: last_cmd_output_start_time != 0 ->', s.last_cmd_output_start_time != 0.0,
          '; last_cmd_cmdline =', repr(s.last_cmd_cmdline))
    before = s.last_cmd_exit_status
    # 'D;99' arrives with is_start=None -> handle_cmd_end('99'):
    s.cmd_output_marking(None, '99')
    after = s.last_cmd_exit_status
    print('[default] Q2 code=99: last_cmd_exit_status  before=%r  after=%r  type(after)=%s'
          % (before, after, type(after).__name__))
    print('[default] Q2 code=99: watcher payload (on_cmd_startstop, is_start=False) =',
          s.captured_payloads[-1])
    print('[default] Q2 code=99: notify_with_command call count =', len(NOTIFY_CALLS),
          '-> notification body SUPPRESSED under default (gate L1425: when != "never" is False)')

    # ---------- Q3 malformed via REAL Window.handle_cmd_end ----------
    for bad in ['not_a_number', '']:
        install_options()  # default
        s = make_scaffold()
        s.cmd_output_marking(True, 'cmdline=ls -la')     # arm
        s.last_cmd_exit_status = 999                     # sentinel: prove except ACTIVELY writes 0
        before = s.last_cmd_exit_status
        s.cmd_output_marking(None, bad)                  # -> handle_cmd_end(bad)
        after = s.last_cmd_exit_status
        print('[default] Q3 malformed exit_status=%r : last_cmd_exit_status  before=%r  after=%r  (canonical Window)'
              % (bad, before, after))
        print('           watcher payload exit_status =', s.captured_payloads[-1]['exit_status'])

    # ---------- Q2 code=99 body text under NON-DEFAULT toggle ----------
    # 'always' keeps the DEFAULT 5.0s duration threshold. Because handle_cmd_end zeroes
    # last_cmd_output_start_time at L1411 BEFORE computing duration at L1417, the duration
    # becomes end_time - 0 = end_time = the SMALL monotonic timestamp (~0.04s, see watcher
    # 'time' field). So the gate `last_cmd_output_duration >= duration` (0.04 >= 5.0) is
    # FALSE and the body is NOT built even with when='always'. To surface the body we drop
    # the threshold to 0 with 'always 0'. Both are NON-DEFAULT toggles (default is 'never').
    for cfg in ['always', 'always 0']:
        install_options(cfg)  # NON-DEFAULT toggle
        opts = get_options()
        print('[toggle %-9r] notify_on_cmd_finish =' % cfg, tuple(opts.notify_on_cmd_finish))
        s = make_scaffold()
        NOTIFY_CALLS.clear()
        s.cmd_output_marking(True, 'cmdline=ls -la')         # arm
        s.cmd_output_marking(None, '99')                     # -> handle_cmd_end('99')
        print('[toggle %-9r] Q2 code=99: last_cmd_exit_status =' % cfg, s.last_cmd_exit_status,
              '; notify_with_command call count =', len(NOTIFY_CALLS))
        if NOTIFY_CALLS:
            print('[toggle %-9r] Q2 code=99: NotificationCommand.body =' % cfg, repr(NOTIFY_CALLS[-1].body))
    install_options()  # reset to default

    # ---------- state transition summary ----------
    install_options()
    s = make_scaffold()
    print('[state] Window init last_cmd_exit_status (before any marker) =', s.last_cmd_exit_status)
    s.cmd_output_marking(True, 'cmdline=ls -la')
    print('[state] after C (armed), before D: last_cmd_exit_status =', s.last_cmd_exit_status)
    s.cmd_output_marking(None, '99')
    print('[state] after D;99                : last_cmd_exit_status =', s.last_cmd_exit_status)

hr('CANONICAL real-Window receiver: kitty.window.Window (executed bytecode, scaffolded state)')
print('Genuine Window needs a live Boss/OS-window/child -> impractical headlessly;')
print('executing the REAL Window.cmd_output_marking / Window.handle_cmd_end bytecode bound to a scaffold.')
from kitty.fast_data_types import monotonic
print('[env] kitty.fast_data_types.monotonic() sample = %r  (SMALL: seconds since monotonic-clock start)' % monotonic())
measure('RUN 1')
measure('RUN 2')
```

### 9.5 PROBE C — `osc133_probe_c.py` (dispatch‑chain evidence: A/B/C/D callback forms)

```python
#!/usr/bin/env python3
# PROBE C -- dispatch-chain evidence. Records every cmd_output_marking(is_start, data)
# callback the REAL compiled parser fires for each OSC 133 marker fed individually,
# proving: A -> 1 call (is_start is Py_False, screen.c L2338); B -> 0 calls (NO case 'B',
# screen.c switch L2331-2354 has only A/C/D); C -> 1 call (is_start is Py_True, data
# carries 'cmdline=...' from screen.c L2344); D -> 1 call (is_start is Py_None, data is
# the exit-status STRING from screen.c L2351-2352). Also shows B is consumed (no cells).
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty.fast_data_types import Screen, set_options

o = Options(merge_result_dicts(defaults._asdict(), {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}))
finalize_keys(o, {}); finalize_mouse_mappings(o, {}); set_options(o)

from kitty_tests import Callbacks, parse_bytes

ESC = b'\x1b'; BEL = b'\x07'
def osc(p): return ESC + b']' + p + BEL

class RecordingCallbacks(Callbacks):
    def __init__(self):
        super().__init__()
        self.marking_calls = []
    def cmd_output_marking(self, is_start, data=''):
        # is_start is Py_False for 'A', Py_True for 'C', Py_None for 'D'
        self.marking_calls.append((repr(is_start), repr(data)))
        return super().cmd_output_marking(is_start, data)

def screen_contents(scr):
    return '\n'.join(str(scr.line(i)) for i in range(scr.lines) if str(scr.line(i)))

for label, payload in [("OSC 133;A", b'133;A'),
                       ("OSC 133;B", b'133;B'),
                       ("OSC 133;C;cmdline=ls -la", b'133;C;cmdline=ls -la'),
                       ("OSC 133;D;42", b'133;D;42')]:
    cb = RecordingCallbacks()
    scr = Screen(cb, 24, 80, 100, 10, 20, 0, cb)
    # 'D' requires a prior armed 'C' to record; for the D-only demo we still show the
    # callback FIRES (arming only affects whether the receiver records state).
    data = osc(payload)
    parse_bytes(scr, data)
    print('%-28s -> cmd_output_marking calls = %s ; screen cells = %r'
          % (label, cb.marking_calls, screen_contents(scr)))

# Full ordered scenario A,B,C,text,D;42 -- show the ordered call log
cb = RecordingCallbacks()
scr = Screen(cb, 24, 80, 100, 10, 20, 0, cb)
full = osc(b'133;A') + osc(b'133;B') + osc(b'133;C;cmdline=ls -la') + b'total 0\n' + osc(b'133;D;42')
parse_bytes(scr, full)
print('\nFull scenario A,B,C,text,D;42 ordered cmd_output_marking calls:')
for i, c in enumerate(cb.marking_calls):
    print('  call %d: is_start=%s data=%s' % (i, c[0], c[1]))
print('screen cells after full scenario =', repr(screen_contents(scr)))
```

---

*End of investigation. The only file added to the repository is this document
(`blitzy/documentation/kitty_815df1e210e0.md`); no existing source, test, shell‑integration,
configuration, or build file was modified, and no build artifacts were committed.*

