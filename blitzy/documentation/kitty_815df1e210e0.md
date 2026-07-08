# How kitty handles a ZWJ multi-codepoint emoji in a 1×1 cell — a runtime-grounded investigation

**Subject:** kitty terminal emulator, version **0.35.2** (`kitty/constants.py:25` → `version: Version = Version(0, 35, 2)`).
**Branch:** `kitty_815df1e210e0`.
**Nature of this document:** a *read-only* code-investigation Q&A. Every behavioral claim below was produced by **building the real C extension and driving bytes through kitty's canonical VT-parser entry point**, then capturing the actual output. Nothing here is inferred from reading alone unless explicitly labeled "(from source)". The source tree was **not modified**; the only file created by this investigation is this document.

---

## How to read this document

Each answer leads with a **direct answer**, then shows the **actual, unedited program output** that produced it (in fenced `text` blocks), then a **cause → effect trace** citing the exact `file:line` and the specific function/struct responsible. The investigation was performed *run-first*: the code paths were exercised and captured **before** this prose was written, and every block below was reproduced and confirmed stable across at least two identical runs.

**Canonical entry point used for every observation** (no bypasses):

```text
UTF-8 bytes  ->  parse_bytes (kitty_tests/__init__.py:30)
             ->  kitty/vt-parser.c  (UTF-8 decode + CSI dispatch)
             ->  screen_draw_text   (kitty/screen.c:866)
             ->  draw_text_loop     (kitty/screen.c:763)
             ->  draw_combining_char (kitty/screen.c:663)  [for zero-width marks]
             ->  line_add_combining_char (kitty/line.c:457) [cell storage]
   CSI ... n ->  report_device_status (kitty/screen.c:2179) [DSR/CPR]
```

Bytes the terminal writes back toward the child (DSR/CPR responses) are captured through the `kitty_tests` `Callbacks` object (`class Callbacks` at `kitty_tests/__init__.py:39`; `write()` appends to `wtcbuf` at `:50-51`; `wtcbuf` is reset by `clear()` at `:96`). The internal `Line.add_combining_char` Python binding (`kitty/line.c:479`) was **never** used — calling it would bypass the parser and would not be a valid observation.

---

## The question (verbatim)

> "I am trying to get a practical understanding of how kitty handles complex Unicode at runtime, especially in cases where the screen state is hard to reason about just by reading the code. When the terminal receives a stream of zero width joiners that form a multi codepoint emoji, how does the internal screen buffer decide what to keep when there is almost no space available, such as a one by one cell? What does the terminal think is actually present in that cell once everything settles? If the terminal is then asked to report part of its current state through a control sequence query, what response does it generate and how does that reflect the earlier grapheme handling? I want to understand how normalization, grapheme breaking, and state reporting interact when the terminal is under extreme constraints. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward."

The headline input throughout is the **family emoji** `👨‍👩‍👧‍👦` =
`U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466`
fed as UTF-8 into a `Screen(callbacks, 1, 1, ...)` (one row, one column).

The question decomposes into four sub-questions:

- **Q1 — Retention:** what does the 1×1 line buffer *keep* when there is almost no space?
- **Q2 — Settled contents:** what does the terminal believe is present in the cell once the whole sequence is consumed?
- **Q3 — State query:** what exact bytes does a control-sequence state query emit, and how does that reflect the earlier grapheme handling?
- **Q4 — Interaction:** how do normalization, grapheme-cluster breaking, and state reporting interact under extreme constraints?

---

## TL;DR — direct answers

- **Q1 (retention):** The four emoji faces 👨👩👧👦 are **width-2 base characters** (`wcwidth == 2`), *not* combining marks. Only the ZWJ `U+200D` is zero-width (`wcwidth == 0`) and is classified as combining. On a one-column screen each width-2 base **overwrites** the single cell, so the buffer keeps only the **last** base that arrived; every intervening ZWJ (and every earlier base) is discarded on each overwrite. kitty does **not** collapse the ZWJ sequence into a single grapheme.
- **Q2 (settled contents):** The settled cell holds exactly one width-2 base codepoint **`0x1F466` (👦)**; the cell width is **2**; and the cursor is logically at **x = 2** on the 1-column screen. A cell can physically hold a base plus at most **3** combining marks (`CPUCell.cc_idx[3]`, `kitty/data-types.h:226`); beyond that the last slot is repeatedly overwritten.
- **Q3 (state query):** The canonical state query is a Device Status Report. On the settled cell, `CSI 6 n` (DSR-6, Cursor Position Report) returns `ESC [ 1 ; 2 R` — 1-based row 1, **column 2**. Column 2 is the **cursor position** (a consequence of the width-2 advance), **not** the stored codepoint. kitty implements **no DECRQCRA** (the standard content-checksum query), so **no standard control sequence exposes the raw cell contents**; the state report reflects the earlier grapheme handling only **positionally**.
- **Q4 (interaction):** kitty performs **no** Unicode NFC/NFD normalization and does **not** merge grapheme clusters. It relies purely on (a) per-codepoint width (`wcwidth`/`wcswidth`) and (b) combining classification (`is_combining_char`, `kitty/unicode-data.c:11`, generated from the **Unicode 15.0.0** database). State reporting then reflects only the resulting cursor position. Under extreme constraints, grapheme "identity" is effectively lost (overwrite wins) and the only observable state is positional.
- **Edge finding (labeled NON-DEFAULT):** With autowrap **disabled** (DECAWM off, `CSI ? 7 l`), drawing *any* width-2 character on a 1-column screen **crashes with SIGSEGV** (exit 139), due to an unsigned underflow at `kitty/screen.c:826`. In the default configuration (DECAWM on) the same input survives. This is **reported only, not patched** (read-only scope).

---

## Q1 & Q2 — What the 1×1 cell keeps, and what is present once settled

### Direct answer

The emoji faces 👨👩👧👦 are **width-2 base characters**; only ZWJ `U+200D` is a zero-width combining mark. On a single-column screen each width-2 base **overwrites** the one cell (the cell is *zeroed* then rewritten), so only the **last** base (👦) survives, and each ZWJ that had attached to a previous base is discarded along with it. The settled cell therefore holds a single codepoint **`0x1F466` (👦)**, has **width 2**, and leaves the cursor at **x = 2**. kitty does **not** merge the ZWJ run into one grapheme.

### Observation 1 — per-codepoint widths (why the faces overwrite and the ZWJ attaches)

Command: `python3 /tmp/kobs/q12.py` (excerpt). Driver: canonical `parse_bytes`; widths from the module's exported `wcwidth`/`wcswidth`.

```text
=== wcwidth per codepoint ===
  0x1f468 wcwidth=2
  0x200d wcwidth=0
  0x1f469 wcwidth=2
  0x200d wcwidth=0
  0x1f467 wcwidth=2
  0x200d wcwidth=0
  0x1f466 wcwidth=2
wcswidth(family string) = 8
```

Each face is width 2; each ZWJ is width 0. `wcswidth` of the whole 7-codepoint string is `8` = four width-2 faces + three width-0 ZWJ — a **pure per-codepoint summation**, with no grapheme awareness (a single "family" grapheme would measure 2, not 8).

### Observation 2 — the headline: full family emoji into a 1×1 screen (Q1/Q2)

```text
=== Q1/Q2 HEADLINE: 1x1 screen, full family emoji ===
INPUT family emoji codepoints: ['0x1f468', '0x200d', '0x1f469', '0x200d', '0x1f467', '0x200d', '0x1f466']
cursor (x,y) after draw: 2 0
line0 text repr: '👦'
line0 kept codepoints: ['0x1f466']
cell0 width: 2
```

**Answer to Q1:** the buffer keeps only `0x1F466` (👦) — the last base drawn. **Answer to Q2:** once settled, the cell contains the single width-2 base `0x1F466`, `cell0 width` is `2`, and the cursor is at `(x=2, y=0)`.

### Observation 3 — transitional (before / during / after), one codepoint at a time

This is the key evidence: after each ZWJ the cell holds *base + ZWJ* and the cursor does **not** advance (the ZWJ is combining); after each subsequent width-2 base the cell is fully **overwritten** and the prior base + ZWJ are lost.

```text
=== TRANSITIONAL: feed one codepoint at a time into a fresh 1x1 screen ===
BEFORE any input: line0='' width0=0 cursor=(0,0)
after 0x1f468: line0='👨' cps=['0x1f468'] cursor=(2,0)
after 0x200d: line0='👨\u200d' cps=['0x1f468', '0x200d'] cursor=(2,0)
after 0x1f469: line0='👩' cps=['0x1f469'] cursor=(2,0)
after 0x200d: line0='👩\u200d' cps=['0x1f469', '0x200d'] cursor=(2,0)
after 0x1f467: line0='👧' cps=['0x1f467'] cursor=(2,0)
after 0x200d: line0='👧\u200d' cps=['0x1f467', '0x200d'] cursor=(2,0)
after 0x1f466: line0='👦' cps=['0x1f466'] cursor=(2,0)
```

Read it as pairs: `0x1f468` writes 👨 (cursor → 2); `0x200d` attaches to the same cell (`'👨\u200d'`, cursor stays 2); `0x1f469` **overwrites** the cell (now just 👩, the previous 👨+ZWJ gone); and so on until only `0x1f466` (👦) remains.

### Observation 4 — the combining-slot ceiling `cc_idx[3]` (retention boundary)

A cell can hold a base plus at most **3** combining marks: `combining_type cc_idx[3];` (`kitty/data-types.h:226`), with `static_assert(sizeof(CPUCell) == 12, ...)` at `:228` pinning the layout. Feeding base `a` plus five combining accents on a 5-column screen (so there is no overwrite/wrap to confuse the picture) settles at exactly **four** stored codepoints, the last slot repeatedly overwritten:

```text
=== cc_idx[3] COMBINING-SLOT OVERFLOW: base 'a' + five combining accents on a 5-col screen ===
after 'a'(0x61): cell0=['0x61'] (count=1)
after 0x301 (wcwidth=0): cell0=['0x61', '0x301'] (count=2)
after 0x302 (wcwidth=0): cell0=['0x61', '0x301', '0x302'] (count=3)
after 0x303 (wcwidth=0): cell0=['0x61', '0x301', '0x302', '0x303'] (count=4)
after 0x304 (wcwidth=0): cell0=['0x61', '0x301', '0x302', '0x304'] (count=4)
after 0x305 (wcwidth=0): cell0=['0x61', '0x301', '0x302', '0x305'] (count=4)
```

At the 4th codepoint (base + `cc_idx[0..2]`) the cell is full. The 5th and 6th marks **replace the last slot** (`0x303` → `0x304` → `0x305`); the count never grows past 4. This is exactly `cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);` at `kitty/line.c:466`. (Note also the guard at `kitty/line.c:459-462`: `line_add_combining_char` refuses to attach to a truly empty cell unless the previous cell is a wide base.)

### Observation 5 — presentation-selector variants (VS16 / VS15)

The variation selectors are themselves zero-width combining marks stored in the cell's `cc_idx`, but they can *change the cell's width*, handled inside `draw_combining_char`.

```text
=== VS16 (U+FE0F) emoji presentation: narrow base heart + VS16 on 1x5 ===
after 0x2764 heart (wcwidth=1, emoji_pres_base=True): cps=['0x2764'] width0=1 cursor=(1,0)
after 0xfe0f VS16 (wcwidth=0): cps=['0x2764', '0xfe0f'] width0=2 cursor=(2,0)

=== VS15 (U+FE0E) text presentation: base heart + VS15 on 1x5 ===
after 0x2764+0xfe0e VS15 (wcwidth(0xfe0e)=0): cps=['0x2764', '0xfe0e'] width0=1 cursor=(1,0)
```

VS16 (`U+FE0F`) turns the default-text-presentation heart `U+2764` (`wcwidth == 1`, and an emoji-presentation base per the exported `is_emoji_presentation_base`) into a **width-2** cell and advances the cursor to 2 — the widen path at `kitty/screen.c:679-689`. VS15 (`U+FE0E`) here is a no-op on width because the heart is already width 1; the narrowing path only fires when the cell width is already 2 (`gpu_cell->attrs.width == 2` at `kitty/screen.c:696`).

### Observation 6 — 1×5 and 2×2 contrasts (making the overwrite mechanism legible)

```text
=== 1x5 CONTRAST: family emoji on a 1-row, 5-col screen ===
line0='👧\u200d👦' cps=['0x1f467', '0x200d', '0x1f466'] cursor=(4,0)

=== 2x2 CONTRAST: family emoji on a 2-row, 2-col screen ===
line0='👧\u200d' cps0=['0x1f467', '0x200d'] | line1='👦' cps1=['0x1f466'] | cursor=(2,1)
```

With **5 columns**, two width-2 faces fit before the overwrite/wrap, leaving `👧‍<ZWJ>👦` on the line. With a **2×2** grid the wrapping scrolls, so the last two survivors land on separate rows (`👧‍<ZWJ>` on row 0, `👦` on row 1). Both confirm there is **no grapheme merging** — the survivors are simply whatever the width / overwrite / scroll arithmetic leaves behind.

### Cause → effect trace (Q1/Q2), grounded in `kitty/screen.c`

Inside `draw_text_loop` (`kitty/screen.c:763`), for each incoming codepoint:

1. **Combining marks are routed away from the cursor.** `if (UNLIKELY(is_combining_char(ch)))` at `:806` is true for the ZWJ; it calls `draw_combining_char(self, s, ch);` at `:810` and then `continue;` at `:811`. Because of the `continue`, the cursor is **not** advanced and no new cell is occupied — which is exactly why every `after 0x200d` line above shows the cursor unchanged. `draw_combining_char` (`:663`) attaches the mark to the *previous* cell (`cursor->x - 1`) via `line_add_combining_char` (`kitty/line.c:457`) at `screen.c:678`.
2. **Base characters take their width and can overwrite.** For a face, `char_width = wcwidth_std(ch);` at `:814` yields 2. The overwrite/wrap test `if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width))` at `:821` is true on a 1-column screen. In the **default** configuration (DECAWM on) the branch `if (self->modes.mDECAWM)` at `:822` takes `continue_to_next_line(self);` at `:823`, which wraps the cursor back to `x = 0` (and, on a 1-row screen, scrolls, keeping `y = 0`).
3. **The cell is zeroed, then rewritten.** `zero_cells(s, s->cp + self->cursor->x, s->gp + self->cursor->x);` at `:835` **clears** the cell, `s->cp[self->cursor->x].ch = ch;` at `:836` writes the new base, `self->cursor->x++;` at `:837`, and for a width-2 base the trailer handling at `:838-843` sets the wide-cell width and advances the cursor to 2. That `zero_cells` at `:835` is precisely why the previous face **and its ZWJ** are discarded on every overwrite.

So retention under extreme space is a direct consequence of *width-driven overwrite*: combining marks ride along with their base cell, and when a new width-bearing base lands on the only cell, the cell is zeroed and the prior contents (base + marks) are gone.

---

## Q3 — What a control-sequence state query reports

### Direct answer

The canonical "report part of your current state" query is a **Device Status Report**. On the settled 1×1 cell (cursor logically at `x = 2` on a 1-column screen), `CSI 6 n` (DSR-6, Cursor Position Report) returns `ESC [ 1 ; 2 R` — 1-based **row 1, column 2**. The reported **column 2 is the cursor position** (a direct consequence of the width-2 advance decisions from Q1/Q2), **not** the stored codepoint. kitty implements **no DECRQCRA** (the VT420 content-checksum query), so **no standard control sequence returns the raw cell contents**. The state report therefore reflects the earlier grapheme/width handling **only positionally**.

### Observation 7 — the full DSR family on the settled cell

The screen is first driven to the settled state (family emoji into 1×1), then each query is issued through `parse_bytes` with `callbacks.clear()` in between, and the child-directed response is read from `callbacks.wtcbuf`.

Command: `python3 -u /tmp/kobs/q3.py 2>&1`

```text
settled cell: cursor=(2,0) line0='👦'

=== Q3 DSR / CPR queries on the settled 1x1 cell (cursor x=2 on 1-col screen) ===
DSR 6  (CSI 6 n)  CPR                b'\x1b[6n'                         -> b'\x1b[1;2R'
DSR 5  (CSI 5 n)  status             b'\x1b[5n'                         -> b'\x1b[0n'
DSR ?6 (CSI ?6 n) DECXCPR            b'\x1b[?6n'                        -> b'\x1b[?1;2R'
DSR ?5 (CSI ?5 n) priv status        b'\x1b[?5n'                        -> b'\x1b[0n'
DSR 99 (CSI 99 n) unknown            b'\x1b[99n'                        -> b''
[0.055] [PARSE ERROR] Unknown CSI code: 'y' with start_modifier: '␀' and end_modifier: '*' and parameters: '1, 0, 0, 0, 0, 0'
DECRQCRA (CSI 1;0;0;0;0;0 * y)       b'\x1b[1;0;0;0;0;0*y'              -> b''
```

> Note on the `[0.055]` prefix: that leading number is a per-run monotonic timestamp emitted by kitty's `log_error`; it is the *only* value that varies between runs. Across repeated runs it was observed as `[0.055]`, `[0.057]`, `[0.059]`, … while every response-byte value stayed identical. All the `b'...'` responses are stable and deterministic.

**Query → response mapping:**

- `CSI 6 n` → `b'\x1b[1;2R'` — Cursor Position Report, 1-based (row 1, column 2).
- `CSI 5 n` → `b'\x1b[0n'` — device status "OK".
- `CSI ?6 n` → `b'\x1b[?1;2R'` — the private DEC form (DECXCPR); the `?` prefix is added.
- `CSI ?5 n` → `b'\x1b[0n'`.
- `CSI 99 n` → `b''` — unknown DSR sub-code; no response.
- `DECRQCRA` (`CSI 1;0;0;0;0;0 * y`) → `b''` — **not implemented**; the parser logs `Unknown CSI code: 'y' ... end_modifier: '*'` and writes nothing back.

### Cause → effect trace (Q3), grounded in `kitty/screen.c` and `kitty/vt-parser.c`

- **Dispatch.** The VT parser routes a DSR to the screen at `kitty/vt-parser.c:1172-1173` (`case DSR:` → `CALL_CSI_HANDLER1P(report_device_status, 0, '?')`), landing in `report_device_status` (`kitty/screen.c:2179`).
- **DSR-5.** `case 5:` writes the literal `"0n"` at `:2185-2187` (→ `ESC [ 0 n`).
- **DSR-6 (CPR) — why column 2.** `case 6:` (`:2188-2198`) reads `x = self->cursor->x; y = self->cursor->y;` at `:2189`. On the settled cell `x = 2`, `columns = 1`, so the clamp at `:2190-2193` (`if (x >= self->columns) { if (y < self->lines - 1) { x = 0; y++; } else x--; }`) takes the `else x--;` branch on this last (and only) line, decrementing `x` from 2 to 1. The DECOM origin adjustment at `:2194` (`if (self->modes.mDECOM) y -= MAX(y, self->margin_top);`) is inert here (origin mode is off by default). Finally the 1-based report is formatted at `:2196` (`snprintf(buf, ..., "%s%u;%uR", (private ? "?": ""), y + 1, x + 1)`) → with `y+1 = 1`, `x+1 = 2` that is `"1;2R"`, written to the child at `:2197`. The private form (`?6`) sets `private = true`, prepending `?` → `"?1;2R"`.
- **DECRQCRA absence.** The sequence `CSI 1;0;0;0;0;0 * y` has an intermediate `*` and final byte `y` for which kitty has **no handler**; it falls through to the default `REPORT_ERROR("Unknown CSI code: '%c' ...")` at `kitty/vt-parser.c:1307`. `REPORT_ERROR` here resolves to `log_error(ERROR_PREFIX " " ...)` (`kitty/vt-parser.c:125`; `ERROR_PREFIX` = `"[PARSE ERROR]"` at `kitty/data-types.h:70`), which only *logs to stderr* — it writes **nothing** back to the child, so `wtcbuf` stays `b''`.

**Consequence for the question:** because kitty implements no DECRQCRA and DSR-6 reports only the cursor position, the earlier grapheme/width handling is observable through a control sequence **only as the cursor column** (here, 2, from the width-2 advance). There is no query that would answer "which codepoints are actually in the cell".

---

## Q4 — How normalization, grapheme breaking, and state reporting interact

### Direct answer (including the negative result)

kitty performs **no** Unicode NFC/NFD normalization on the input stream, and it does **not** merge grapheme clusters. It relies purely on:

1. **per-codepoint width** via `wcwidth`/`wcswidth`, and
2. **combining classification** via `is_combining_char` (`kitty/unicode-data.c:11`), generated from the **Unicode 15.0.0** database (banner at `kitty/unicode-data.c:1`).

State reporting (DSR-6 CPR, Q3) then reflects only the resulting **cursor position**. The net effect under extreme constraints: grapheme "identity" is effectively lost (overwrite wins, per Q1/Q2), and the only observable state is positional.

### Observation 8 — normalization negative test (the decisive negative result)

Feeding `e` (`U+0065`) followed by a combining acute accent (`U+0301`) keeps **both** codepoints **decomposed**; kitty does not fold them into the precomposed `é` (`U+00E9`). The contrast feeds precomposed `U+00E9` directly.

Command: `python3 /tmp/kobs/q4.py`

```text
=== Q4 NORMALIZATION negative test: 'e' (U+0065) + combining acute (U+0301) on 1x5 ===
after 0x65 'e': cps=['0x65'] cursor=(1,0)
after 0x301 combining-acute: cps=['0x65', '0x301'] cursor=(1,0)
NFC test -- kept codepoints: ['0x65', '0x301']   (precomposed 'e-acute' would be ['0xe9'])
str(line0)='é'

=== contrast: precomposed U+00E9 fed directly ===
after 0xe9: cps=['0xe9'] cursor=(1,0) str='é'
```

The decomposed input stores **two** codepoints `['0x65', '0x301']`; the precomposed input stores **one** codepoint `['0xe9']`. Both render visually as `é`, but the **stored codepoints differ** — proving kitty neither NFC-folds (`e`+`◌́` → `é`) nor NFD-expands (`é` → `e`+`◌́`). It preserves the stream as received.

### Observation 9 — combining/grapheme classification observed through behavior

`is_combining_char` is not exported to Python, so classification is demonstrated *behaviorally*: a width-0 codepoint that attaches to the previous cell without advancing the cursor is being treated as a combining mark.

```text
=== combining classification OBSERVED via behavior (ZWJ attaches, does not occupy) ===
after 'A'(0x41,wcwidth=1): cps=['0x41'] cursor=(1,0)
after ZWJ(0x200d,wcwidth=0): cps=['0x41', '0x200d'] cursor=(1,0)
```

`A` (width 1) advances the cursor to 1. The ZWJ (`U+200D`, width 0) does **not** advance the cursor and is appended to the same cell — behaviorally confirming the ZWJ is treated as combining. In source, `is_combining_char` (`kitty/unicode-data.c:11`) returns `true` for the ZWJ via the range `case 0x200b ... 0x200f: return true;` at `kitty/unicode-data.c:323` (`U+200D` lies inside `0x200b…0x200f`).

### Why this is *summation*, not grapheme measurement

From Observation 1, `wcswidth('👨‍👩‍👧‍👦') == 8` = four width-2 faces + three width-0 ZWJ. A grapheme-cluster–aware measurement of the family emoji would be **2** (one cluster, one wide glyph). The value 8 is therefore direct evidence that kitty sums **per codepoint**, with no cluster segmentation. The classification tables themselves are generated by `gen/wcwidth.py` (`unicode_version()` at `:61`; the banner is emitted at `:294`; the `#define UNICODE_MAJOR/MINOR/PATCH_VERSION` values are emitted at `:531-534`) and are marked "DO NOT EDIT"; the module reports its baked-in database as `(15, 0, 0)`.

### How the three interact under extreme constraints

- **Normalization:** absent — the stream is stored as-is (Observation 8).
- **Grapheme breaking:** absent as a *merging* step — codepoints are individually classified (combining vs. width-bearing) and either attach to a cell or occupy/overwrite a cell (Observations 3, 9). No cluster is ever formed.
- **State reporting:** purely positional — DSR-6 reports the cursor column produced by the width arithmetic (Q3), never the stored codepoints.

So on a 1×1 cell the three do not cooperate to preserve a "family emoji"; they interact only insofar as width classification drives overwrite, and the overwrite result is then summarized positionally by the state report.

---

## Edge finding — a DECAWM-off SIGSEGV (LABELED NON-DEFAULT)

> **This section is explicitly NON-DEFAULT.** The default-configuration answers above (DECAWM on) are unaffected. This edge path is exercised only to satisfy exhaustiveness, and the default-config control is presented alongside it. The defect is **reported only — it is NOT patched** (patching would violate the read-only scope of this task).

### What happens

With autowrap **disabled** (DECAWM off, set via `CSI ? 7 l` = `b'\x1b[?7l'`), drawing **any** width-2 character on a 1-column screen causes a **segmentation fault** (shell exit code **139**). It is reproduced here with the CJK ideograph `U+4E00` (`wcwidth == 2`) rather than an emoji, to show it is a **width-2 property, not emoji-specific**. The default (DECAWM on) control is shown first and survives.

Command: `bash /tmp/kobs/edge_driver.sh` (each Python invocation run under `python3 -u` so the pre-crash stdout line is not lost on segfault).

```text
=== DECAWM ON control (default) ===
DECAWM on (default): cursor.x=2 kept=['0x4e00']
control exit code: 0

=== DECAWM OFF crash, RUN 1 ===
DECAWM off set; drawing width-2 CJK U+4E00 on 1-col screen ...
run1 exit code: 139

=== DECAWM OFF crash, RUN 2 (stability) ===
DECAWM off set; drawing width-2 CJK U+4E00 on 1-col screen ...
run2 exit code: 139
```

For each crashed run the parent shell (bash) additionally prints, to its own stderr:

```text
Segmentation fault      (core dumped) python3 -u /tmp/kobs/decawm_off.py
```

The crash reproduces **deterministically**: beyond the two runs above, the DECAWM-off script was run 8 times total and exited **139 every time**. Exit `139` is `128 + SIGSEGV`, and `SIGSEGV == 11` (`128 + 11 == 139`).

### Root cause (grounded in `kitty/screen.c`)

In `draw_text_loop` (`kitty/screen.c:763`), the overwrite/wrap test at `:821` (`if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width))`) is true for a width-2 char on a 1-column screen. The two branches diverge:

- **DECAWM on (default) — survives.** `if (self->modes.mDECAWM)` at `:822` takes `continue_to_next_line(self);` at `:823`, wrapping/scrolling safely; the subsequent cell write lands in-bounds and the control observes `cursor.x = 2`, cell keeps `0x4e00`.
- **DECAWM off — crashes.** The `else` branch at `:825-828` executes `self->cursor->x = self->columns - char_width;` at `:826`. Here `self->columns` is unsigned `1` and `char_width` is `2`, so `1u - 2` **underflows to `UINT_MAX`**. `self->cursor->x` is now `4294967295`, and the immediately following cell write `s->cp[self->cursor->x]` at `:835-836` indexes catastrophically out of bounds → **SIGSEGV**.

The defect is thus an unsigned underflow at `kitty/screen.c:826` on the non-default DECAWM-off path. Again: **reported, not fixed.**

---

## Appendix — build and method (for reproducibility)

**Versions (observed in this container):**

- kitty **0.35.2** — `python3 -c "from kitty.constants import version, str_version; print(tuple(version), str_version)"` → `(0, 35, 2) 0.35.2` (`kitty/constants.py:25`).
- Python **3.13.7** — this satisfies the declared floor `requires-python = ">=3.8"` (`pyproject.toml:2`), which `setup.py` validates in `check_version_info()` (`setup.py:30`, invoked at `:47`; it is a minimum-only check).
- Unicode database **15.0.0** — `python3 -c "import kitty.fast_data_types as f; print(f.unicode_database_version())"` → `(15, 0, 0)`, matching the `kitty/unicode-data.c:1` banner "built from the Unicode Standard 15.0.0".

> **Environment notes (run-first is authoritative).** Two values observed here differ from an earlier reference environment and are reported as observed: (a) Python is **3.13.7** (not 3.12.3) — both satisfy the `>=3.8` floor; (b) the build **exits 0** here because the Go toolchain is present, so the Go-based `kitten` CLI also builds — in a Go-less environment `setup.py build` exits non-zero after the required `fast_data_types.so` has already linked successfully. The essential fact (that `fast_data_types.so` builds and drives every observation) holds either way.

**Build command (exact):**

```text
python3 setup.py build --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag is used because a modern GCC otherwise appends `-pedantic-errors -Werror` and a harmless `-Werror=switch` in the Wayland GLFW backend would fail the build. Observed link step:

```text
[1/1] Linking kitty/fast_data_types ...
 done
```

**Produced artifact:** `kitty/fast_data_types.so` — **1,253,792 bytes** (~1.2 MB), gitignored via the `*.so` rule (and `build/` via `/build/`), confirmed with `git check-ignore kitty/fast_data_types.so build`. Building therefore does not alter tracked source.

**Observation invocation pattern:** options are initialized from `kitty.options.types.defaults`, merged via `kitty.options.parse.merge_result_dicts`, finalized with `kitty.config.finalize_keys` / `finalize_mouse_mappings`, and installed with `set_options(...)`. Then:

```text
Screen(callbacks, rows, cols, 0, cell_width, cell_height, 0, callbacks)   # e.g. rows=1, cols=1
parse_bytes(screen, utf8_bytes)                                           # canonical parser path
inspect: str(screen.line(y)), screen.line(y)[x], screen.line(y).width(x),
         screen.cursor.x, screen.cursor.y, callbacks.wtcbuf
```

`parse_bytes` and `Callbacks` come from `kitty_tests/__init__.py` (`:30` and `:39`) — the sanctioned, non-bypassing observation harness. Per-cell codepoints were read via the `Line` sequence accessor `line[x]` (`text_at`, `kitty/line.c:193`), which returns the cell's base plus its combining marks.

**Read-only / cleanup discipline:** all temporary observation scripts lived **outside** the tracked tree, under `/tmp/kobs/`, and were removed after the investigation. No existing repository file was modified, added, or deleted. At completion, `git status --porcelain` shows only this new document as an added path; the gitignored `kitty/fast_data_types.so` and `build/` do not appear.

---

## Coverage pass

Each item below states **what was asked → the observed answer → the grounding citation**.

**Q1 — retention under extreme space.** What does a 1×1 cell keep? → It keeps only the **last** width-2 base (`0x1F466`, 👦); intervening ZWJ and earlier faces are discarded on each overwrite. → `draw_text_loop` overwrite: `zero_cells` at `kitty/screen.c:835` then cell write at `:836`; combining route at `:806-811`.

**Q2 — settled contents.** What is present once settled? → One codepoint `0x1F466`, `cell0 width = 2`, cursor `(2, 0)`; a cell holds base + at most 3 marks. → `CPUCell.cc_idx[3]` `kitty/data-types.h:226` (`static_assert` `:228`); overflow overwrite `kitty/line.c:466`.

**Q3 — state query.** What bytes come back and how do they reflect grapheme handling? → `CSI 6 n` → `b'\x1b[1;2R'` (position, not contents); DSR-5 → `b'\x1b[0n'`; `?6` → `b'\x1b[?1;2R'`; `?5` → `b'\x1b[0n'`; unknown `99` → `b''`; DECRQCRA → `b''`. Reflects grapheme handling **only positionally**. → `report_device_status` `kitty/screen.c:2179` (case-6 arithmetic `:2188-2198`); DSR dispatch `kitty/vt-parser.c:1172-1173`.

**Q4 — interaction.** How do normalization, grapheme breaking, and state reporting interact? → No NFC/NFD (decomposed stays 2 codepoints vs. precomposed 1); no grapheme merging; classification via Unicode-15.0.0 `is_combining_char`; state report purely positional. → `is_combining_char` `kitty/unicode-data.c:11`, ZWJ range `:323`, banner `:1`; generator `gen/wcwidth.py:61,:294,:531-534`.

**Named items:**

- **1×1 headline** → cursor `(2,0)`, `line0='👦'`, kept `['0x1f466']`, width 2 → `kitty/screen.c:835-843`.
- **Transitional before/during/after** → ZWJ attaches (cursor unchanged), next base overwrites → `kitty/screen.c:806-811, :835`.
- **`cc_idx[3]` overflow boundary** → settles at 4 codepoints; last slot overwritten `0x303→0x304→0x305` → `kitty/line.c:466`, `kitty/data-types.h:226`.
- **VS16 (`U+FE0F`)** → widens heart to width 2, cursor → 2 → `kitty/screen.c:679-689`.
- **VS15 (`U+FE0E`)** → no-op here (width stays 1; narrowing needs width 2) → `kitty/screen.c:690-700, :696`.
- **1×5 contrast** → `line0='👧\u200d👦'`, cursor `(4,0)` → width/overwrite arithmetic, `kitty/screen.c:821-843`.
- **2×2 contrast** → `line0='👧\u200d'`, `line1='👦'`, cursor `(2,1)` → wrap/scroll, `kitty/screen.c:823`.
- **DSR 5 / 6 / ?5 / ?6 / unknown 99** → `b'\x1b[0n'` / `b'\x1b[1;2R'` / `b'\x1b[0n'` / `b'\x1b[?1;2R'` / `b''` → `kitty/screen.c:2185-2198`.
- **DECRQCRA absence** → `b''` + `[PARSE ERROR] Unknown CSI code: 'y'` → `kitty/vt-parser.c:1307` (log-only, `:125`; `ERROR_PREFIX` `kitty/data-types.h:70`).
- **`report_device_status` case-6 trace** → `x=2 ≥ cols=1 → x-- → 1 → "1;2R"` → `kitty/screen.c:2189-2196`.
- **NFC/NFD negative test** → decomposed `['0x65','0x301']` vs precomposed `['0xe9']`, both render `é` → no normalization step in the draw path (`kitty/screen.c:763` loop stores codepoints as-received).
- **Combining classification** → ZWJ width 0 attaches without advancing → `kitty/unicode-data.c:11, :323`.
- **`wcswidth` summation** → `wcswidth('family') == 8` (4×2 + 3×0), not cluster measurement → exported `wcswidth`; tables `gen/wcwidth.py`.
- **Labeled non-default DECAWM-off SIGSEGV + default control** → off: exit 139 (deterministic); on: cursor.x=2, kept `['0x4e00']`, exit 0 → underflow `kitty/screen.c:826`; safe wrap `:823`. Reported only.

**Q1, Q2, Q3, Q4 and every named condition are answered above, each with its actual observed output and a specific `file:line` citation.**

