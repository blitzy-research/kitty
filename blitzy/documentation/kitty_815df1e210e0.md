# kitty and the ZWJ Family Emoji in a 1×1 Cell

**How the screen buffer decides what to keep, what the cell ends up containing, and what a state query reports — grounded in source and verbatim runtime output.**

- **Project:** kitty terminal emulator
- **Version pin:** kitty **0.35.2**, source HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`)
- **Scenario:** the family emoji `👨‍👩‍👧‍👦` = `U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466` (7 codepoints) drawn into a **1‑row × 1‑column** grid.
- **Methodology (binding):** every answer below is written from **observed behavior** — the `kitty.fast_data_types` C extension was built and driven headlessly, and the quoted output is the *actual* captured output, shown next to the command/code that produced it. Every factual claim also carries an exact `file:line` citation.

---

## Executive summary (the headline finding)

When a multi‑codepoint ZWJ emoji is forced into a 1×1 grid, kitty at this revision **does not** treat the sequence as one Unicode grapheme cluster. Instead:

- Each codepoint is classified **independently** by a per‑codepoint combining‑character heuristic (`is_combining_char`, `kitty/unicode-data.c:11`) — there is **no** UAX #29 GB11 emoji‑ZWJ clustering layer at this HEAD (`kitty/text-cache.*` does not exist).
- The four human‑figure **base** codepoints are *not* combining and are each **width 2** (`wcwidth_std`, `kitty/wcwidth-std.h:10`), so in a 1‑column grid each new base triggers autowrap (`kitty/screen.c:821`), and because DECAWM defaults on (`kitty/screen.c:33`) the previous base is pushed into scrollback (`kitty/history.c:287`).
- The ZWJ `U+200D` **is** combining (`kitty/unicode-data.c:323`), so it is appended to the base that precedes it (`kitty/line.c:457`), stored in the fixed three‑slot `CPUCell.cc_idx[3]` (`kitty/data-types.h:226`).

**Net effect:** the emoji is *fully preserved but fragmented* into four independent width‑2 base cells spread across the visible row plus three wrapped scrollback lines (`historybuf.count = 4`). The single visible 1×1 cell contains only the **last** base, the boy `U+1F466`, at width 2. A subsequent Cursor Position Report (`ESC[6n`) replies **`ESC[1;2R`** (bytes `b'\x1b[1;2R'`) — a *cell‑grid* coordinate that has **no awareness** of the emoji's intended single‑grapheme identity.

### Quick‑answer table

| Question | Answer (verbatim) | Primary citation |
|---|---|---|
| **Q1** What does the buffer keep? | Each width‑2 base becomes its own cell; the 1‑col grid autowraps, so the sequence fragments across `historybuf.count = 4` lines (3 content + 1 empty). ZWJ is stored as a combining mark on the preceding base. | `kitty/screen.c:763`, `:821`, `:33`, `kitty/history.c:287` |
| **Q2** What is in the cell? | The last base only: `'👦'` = `U+1F466`, codepoint count 1, cell width 2, cursor at `(2, 0)`. | `kitty/screen.c:836`, `kitty/data-types.h:224,226,198` |
| **Q3** What does a state query reply? | CPR `ESC[6n` → **`ESC[1;2R`** (`b'\x1b[1;2R'`); it reports the clamped cursor **cell** coordinate, not the grapheme. | `kitty/screen.c:2179`, `:2192`, `:2196` |
| **Q4** How do the mechanisms interact? | No normalization; grapheme "breaking" is a per‑codepoint `is_combining_char` heuristic (not UAX #29); width‑driven autowrap fragments the sequence; the cursor‑clamped CPR then reports a cell coordinate. | `kitty/unicode-data.c:11`, absent `kitty/text-cache.*` |

---

## Scenario and method (run‑first)

The answers were produced by **building and running** kitty, not by reading alone. Concretely:

1. The C extension was built headlessly (artifacts are gitignored, so the working tree stays clean):

   ```text
   CI=true python3 setup.py build --debug --skip-building-kitten --ignore-compiler-warnings
   # -> kitty/fast_data_types.so = 6142824 bytes (~6.1 MB)
   ```

   The build entry point is `def build(...)` (`setup.py:1084`) and the extension name is `'kitty/fast_data_types'` (`setup.py:1091`).

2. A GPU‑less `Screen` was constructed through the repository's headless test harness `kitty_tests/__init__.py`, which provides `parse_bytes` (`kitty_tests/__init__.py:30`), the `Callbacks` reply capture (`Callbacks.write` → `self.wtcbuf += bytes(data)`, `kitty_tests/__init__.py:51`), and the `Screen` constructor call shape `Screen(c, lines, cols, scrollback, cell_width, cell_height, 0, c)` (`kitty_tests/__init__.py:240`).

3. The family emoji (and simpler ZWJ pairs / width edge cases) were drawn with `Screen.draw(...)` — which drives `screen_draw_text` (`kitty/screen.c:866`, dispatched by the VT parser at `kitty/vt-parser.c:226` and `:236`) — and control‑sequence queries were fed through `parse_bytes`, with the exact reply bytes read back from `Callbacks.wtcbuf`.

The full build command, harness script, complete verbatim output, and environment details are in the **Observation‑method appendix** at the end. The relevant slices of that output are embedded inline beside each answer below.

---

## Q1 — What the screen buffer keeps under the 1×1 constraint

### The per‑codepoint draw loop

kitty draws text one codepoint at a time in `draw_text_loop` (`kitty/screen.c:763`). For each incoming codepoint `ch`, the loop first asks whether it is a combining character: `if (UNLIKELY(is_combining_char(ch)))` (`kitty/screen.c:806`). This is the crux of retention — **classification is per codepoint**, decided by `is_combining_char` (`kitty/unicode-data.c:11`), whose table is generated "from the Unicode Standard 15.0.0" (`kitty/unicode-data.c:1`) with a fast path `if (LIKELY(code < 173)) return false;` (`kitty/unicode-data.c:13`).

- **The ZWJ `U+200D` is combining.** It is matched by `case 0x200b ... 0x200f:` → `return true;` (`kitty/unicode-data.c:323`–`:324`). So each ZWJ is routed to `draw_combining_char(self, s, ch)` (`kitty/screen.c:810`) followed by `continue;` (`kitty/screen.c:811`) — it is **appended to the preceding base**, never occupying a cell of its own.
- **The emoji bases are not combining.** `U+1F468`, `U+1F469`, `U+1F467`, `U+1F466` all fall through the combining test. Their width is looked up via `char_width = wcwidth_std(ch)` (`kitty/screen.c:814`; `wcwidth_std` at `kitty/wcwidth-std.h:10`), and each emoji base is **width 2**. Each therefore **starts a new cell**.

### How a combining mark is stored (and the three‑slot cap)

`draw_combining_char` (`kitty/screen.c:663`) appends the mark to the current base via `line_add_combining_char(...)` (`kitty/screen.c:678`; function at `kitty/line.c:457`). Storage is the fixed three‑element array `combining_type cc_idx[3];` (`kitty/data-types.h:226`; `combining_type` is `uint16_t`, `kitty/data-types.h:62`) inside a 12‑byte `CPUCell` (`static_assert(sizeof(CPUCell) == 12)`, `kitty/data-types.h:228`).

The append logic fills slots 0–2 in order (`cell->cc_idx[i] = mark_for_codepoint(ch); return;`, the fill assignment at `kitty/line.c:464`), and on overflow it **overwrites the last slot**:

```c
// kitty/line.c:466  (LAST-SLOT OVERWRITE on overflow)
cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);
```

> Citation note: the *overwrite* statement is `kitty/line.c:466`; `kitty/line.c:464` is the in‑loop fill assignment. A combining char is also **not** attached to a null cell unless the previous cell is a width‑2 base (`kitty/line.c:459`–`:461`), which is exactly the emoji‑base case here.

### Why the 1‑column grid fragments the sequence

After computing a base's width, the loop performs the wrap check:

```c
// kitty/screen.c:821
if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width)) {
    if (self->modes.mDECAWM)          // kitty/screen.c:822
        continue_to_next_line(self);  // kitty/screen.c:823
}
```

With `columns == 1` and a width‑2 base, the wrap check `1 < cursor->x + 2` (`kitty/screen.c:821`) is satisfied for **every** base in the sequence — **including the very first one at `x = 0`**, where `1 < 0 + 2` already holds, because a width‑2 glyph can never fit in a single column. Autowrap (DECAWM, mode 7) is **on by default** — the initial mode block sets it true: `static const ScreenModes empty_modes = {0, .mDECAWM=true, .mDECTCEM=true, .mDECARM=true};` (`kitty/screen.c:33`) — so `continue_to_next_line` (`kitty/screen.c:823`, defined at `kitty/screen.c:524`) runs on **each** wrap, pushing the current line into the scrollback ring via `historybuf_add_line` (`kitty/history.c:287`, invoked at `kitty/history.c:342`). Because the first base wraps *before* any cell has been written to the row, the line pushed first is the still‑empty starting row — this is precisely the origin of the empty trailing history entry `hist[3]` seen below; each of the subsequent bases then wraps in turn, pushing the prior, **content‑bearing** base into scrollback. After each wrap the base is written with `s->cp[self->cursor->x].ch = ch;` (`kitty/screen.c:836`); the cursor advances (`kitty/screen.c:837`) and, for a width‑2 base, advances again while marking the GPU width (`kitty/screen.c:838`–`:843`).

Drawing a **single** base into a 1×1 grid confirms this first‑base wrap directly — the row pushed to scrollback is empty and `historybuf.count` is already `1` after just one base:

```text
=== First-base wrap probe (does the FIRST width-2 base wrap at x=0?) ===
draw 1 base U+1F468 into 1x1 -> historybuf.count = 1 ; hist[0] = '' (pre-write wrap at x=0 pushed an EMPTY line)
```

### Observed: fragmentation across scrollback

```text
=== Fragmentation across scrollback (family in 1x1) ===
historybuf.count    : 4
historybuf str      : '👧\u200d\n👩\u200d\n👨\u200d\n'
hist[0] = '👧\u200d' | U+1F467 U+200D
hist[1] = '👩\u200d' | U+1F469 U+200D
hist[2] = '👨\u200d' | U+1F468 U+200D
hist[3] = ''         (empty trailing history line; count still reports 4)
```

The scrollback is ordered newest‑first: the girl (`U+1F467`) was the last base to wrap and sits at `hist[0]`, the woman (`U+1F469`) at `hist[1]`, and the man (`U+1F468`) — the first content‑bearing base to wrap — at `hist[2]`. Each retains its **trailing ZWJ** as a combining mark, exactly as the storage model predicts. `historybuf.count` reports **4**, of which three carry content and one (`hist[3]`) is an empty trailing line. The boy (`U+1F466`) never wrapped and remains on the visible row (see Q2).

**Rationale.** With only one column, a width‑2 base cannot coexist with the next base, so autowrap fragments the sequence. Retention is therefore **per base cell** (each base plus up to three combining marks), *not* per grapheme cluster: the man/woman/girl bases — each still carrying the ZWJ that was meant to *join* them to the next figure — land on separate `continued` history lines, which is direct proof that kitty stored them as four independent cells rather than one cluster.

---

## Q2 — What the visible cell actually contains

Once the stream settles, the single visible cell contains **only the last base written**, the boy `U+1F466`, at width 2 — nothing else.

### Observed: the 1×1 cell after drawing the family

```text
=== CASE A: 1x1 screen, draw FAMILY ===
line(0) repr        : '👦'
line(0) codepoints  : U+1F466 | count = 1
cell[0] width       : 2
cursor (x,y)        : 2 0
CPR  ESC[6n  reply  : b'\x1b[1;2R'
DSR5 ESC[5n  reply  : b'\x1b[0n'
SIZE ESC[14t reply  : b'\x1b[4;20;10t'
```

- **Base codepoint:** `str(s.line(0))` is `'👦'`, i.e. exactly one codepoint `U+1F466`, `count = 1`. This is the value stored in `CPUCell.ch` (`char_type ch;`, `kitty/data-types.h:224`) by the base‑cell write `s->cp[self->cursor->x].ch = ch;` (`kitty/screen.c:836`).
- **Combining marks:** none. The cell's `cc_idx[3]` (`kitty/data-types.h:226`) is empty — the boy was the final codepoint in the stream, so no ZWJ or mark followed it to be appended (unlike the man/woman/girl bases now in scrollback, each of which carries a trailing `U+200D`).
- **Width:** `cell[0] width == 2`. The width came from `wcwidth_std` (`kitty/wcwidth-std.h:10`) and is stored in the GPU cell's 2‑bit width bitfield `uint16_t width : 2;` inside `CellAttrs` (`kitty/data-types.h:198`; `WIDTH_MASK (3u)` at `kitty/data-types.h:211`, `GPUCell` at `kitty/data-types.h:216`–`:220`). Because the base is width 2 in a 1‑column grid, writing it advances the logical cursor to `x = 2` (the double advance at `kitty/screen.c:838`–`:843`), which is why `cursor (x,y) : 2 0` — a coordinate that sits **past** the single column. That over‑the‑edge cursor is what the state query in Q3 must clamp.

**Rationale.** kitty's notion of "what is in the cell" is strictly the `CPUCell` model: a single base `ch` plus at most three combining marks in `cc_idx[3]`. Under the 1×1 constraint the sequence's earlier bases have already wrapped away into scrollback (Q1), so the only thing the terminal "considers present" in the visible cell is the last surviving base — the boy at `U+1F466`, occupying a width‑2 cell. There is no composite "family" object anywhere in the cell model; the cell is a base‑plus‑marks record, and here it holds just the base.

---

## Q3 — What a control‑sequence state query replies, and how it reflects the grapheme handling

### Cursor Position Report (the direct answer)

Querying `ESC[6n` (DSR‑CPR) yields **`ESC[1;2R`** (bytes `b'\x1b[1;2R'`). The handler is `report_device_status` (`kitty/screen.c:2179`); the CPR branch (`case 6`) does:

```c
// kitty/screen.c:2189   x = self->cursor->x;  y = self->cursor->y;   (here x=2, y=0)
// kitty/screen.c:2190   if (x >= self->columns) {                    (2 >= 1  -> true)
// kitty/screen.c:2191       if (y < self->lines - 1) { x = 0; y++; }  (0 < 0  -> false)
// kitty/screen.c:2192       else x--;                                 (last row -> x becomes 1)
// kitty/screen.c:2196   -> format "%s%u;%uR" with 1-based y+1, x+1   -> "\x1b[1;2R"
```

So the width‑2 base left the cursor at logical `x = 2`; since `x >= columns` (2 ≥ 1) and this is the last row, the else‑branch clamps with `x--` to `x = 1`, and the reply is emitted 1‑based as row `1`, column `2` → `ESC[1;2R`.

### The other state queries on the same 1×1 screen

```text
=== State queries on the 1x1 constrained screen ===
DA1  ESC[c   -> b'\x1b[?62;c'
DA2  ESC[>c  -> b'\x1b[>1;4000;35c'
XTVER ESC[>q -> b'\x1bP>|kitty(0.35.2)\x1b\\'
DECRQM ESC[?7$p -> b'\x1b[?7;1$y'
```

| Query | Reply (verbatim) | Handler / rationale |
|---|---|---|
| **DSR** `ESC[5n` | `ESC[0n` (`b'\x1b[0n'`) | `case 5` writes the literal `"0n"` ("terminal OK"), `kitty/screen.c:2186`. |
| **CPR** `ESC[6n` | `ESC[1;2R` (`b'\x1b[1;2R'`) | Clamped cursor cell coordinate, `kitty/screen.c:2189`–`:2196` (see above). |
| **Text‑area size** `ESC[14t` | `ESC[4;20;10t` (`b'\x1b[4;20;10t'`) | `screen_report_size` `case 14`: code `4`, height = `cell_height × lines = 20 × 1`, width = `cell_width × columns = 10 × 1`, format `"%u;%u;%ut"` (`kitty/screen.c:2147`–`:2164`). The harness uses `cell_width=10, cell_height=20` (`kitty_tests/__init__.py:240`), hence `4;20;10`. |
| **DA1** `ESC[c` | `ESC[?62;c` (`b'\x1b[?62;c'`) | `report_device_attributes` writes `"?62;c"` (`kitty/screen.c:2125`). |
| **DA2** `ESC[>c` | `ESC[>1;4000;35c` (`b'\x1b[>1;4000;35c'`) | `">1;" PRIMARY_VERSION ";" SECONDARY_VERSION "c"` (`kitty/screen.c:2128`); `PRIMARY_VERSION = 4000` and `SECONDARY_VERSION = 35` from `setup.py:605`–`:606`, matching `Version(0, 35, 2)` (`kitty/constants.py:25`). |
| **XTVERSION** `ESC[>q` | `ESC P>|kitty(0.35.2) ESC \` (`b'\x1bP>|kitty(0.35.2)\x1b\\'`) | `screen_xtversion` writes DCS `">|kitty(" XT_VERSION ")"` with `XT_VERSION = 0.35.2` (`kitty/screen.c:2137`, `setup.py:607`). |
| **DECRQM** `ESC[?7$p` | `ESC[?7;1$y` (`b'\x1b[?7;1$y'`) | `report_mode_status` for DECAWM: `ans = mDECAWM ? 1 : 2` → `1` (set), format `"%s%u;%u$y"` (`kitty/screen.c:2203`, value at `:2210`, framing at `:2240`). This confirms autowrap was on — the very mode that caused the Q1 fragmentation. |

### Reply framing and capture

All replies are framed by `get_prefix_and_suffix_for_escape_code` (`kitty/screen.c:955`): CSI replies use the prefix `"\033["` with an empty suffix (`kitty/screen.c:962`), while the DCS XTVERSION reply is wrapped in `"\033P"` … `"\033\\"` (prefix at `kitty/screen.c:959`, suffix at `kitty/screen.c:956`) — which is exactly the `ESC P … ESC \` you see around `>|kitty(0.35.2)`. The framed bytes are handed to `write_escape_code_to_child` (`kitty/screen.c:978`) and, in the headless harness, captured by `Callbacks.write` into `wtcbuf` (`kitty_tests/__init__.py:51`).

**Rationale (the crux).** The CPR reply encodes the **cell‑grid coordinate** of the cursor *after* a width‑2 base was written — i.e., it reflects cells and columns, then clamps the over‑the‑edge cursor with `x--`. It has **no awareness** of the emoji's intended single‑grapheme identity: nothing in `report_device_status` consults grapheme structure, combining marks, or the ZWJ sequence. The state report describes the *geometry of the buffer* (a width‑2 cell in a 1‑column row), not the *semantics of the text*. That is the direct link back to Q1/Q2: because grapheme handling is a per‑cell/per‑codepoint affair, the state query can only ever see cells.


---

## Q4 — Synthesis: how normalization, grapheme breaking, and state reporting interact

Three mechanisms combine to produce the observed behavior. The overarching point: **kitty at this revision has no grapheme‑cluster layer and no normalization step**, so a ZWJ emoji is handled as a stream of independent codepoints whose only cohesion is the combining‑mark attachment of the ZWJ itself — and geometry (width + autowrap) then dictates what survives where, with state reporting seeing only the resulting cells.

### 1. No Unicode normalization

kitty stores codepoints as received; it performs no NFC/NFD folding.

```text
No normalization: 'e'+U+0301 -> 'é' | U+0065 U+0301 ; NFC U+00E9 present? False
```

Drawing `e` + `U+0301` (combining acute) leaves the cell holding the **decomposed** pair `U+0065 U+0301`, not the precomposed NFC form `U+00E9`. The base is `U+0065` in `CPUCell.ch` (`kitty/data-types.h:224`) and the acute is a combining mark in `cc_idx` (`kitty/line.c:457`). There is no code path that would rewrite these to `U+00E9` — confirmed by `NFC U+00E9 present? False`.

### 2. Grapheme "breaking" is a per‑codepoint heuristic, not UAX #29 GB11

kitty's decision to keep a codepoint in the same cell is made *solely* by `is_combining_char` (`kitty/unicode-data.c:11`) per codepoint. There is **no** grapheme/text‑cache module at this HEAD:

```text
$ ls kitty/text-cache*
ls: cannot access 'kitty/text-cache*': No such file or directory
```

The Unicode standard's text‑segmentation rules intend that an emoji ZWJ sequence such as `👨‍👩‍👧‍👦` be treated as a **single** extended grapheme cluster — a conformance baseline this document compares kitty against.

> **External standard (context — not verified from this repository).** The preceding sentence is a claim about the *Unicode standard*, not about kitty's code. Unicode Standard Annex #29, *Unicode Text Segmentation* (UAX #29), §3.1.1 *Grapheme Cluster Boundary Rules*, rule **GB11** ("Do not break within emoji modifier sequences or emoji zwj sequences") gives the boundary formula `\p{Extended_Pictographic} Extend* ZWJ × \p{Extended_Pictographic}`, where `×` means "do not break here"; the `Extended_Pictographic` property values come from UTS #51. Under GB11 the family sequence `👨‍👩‍👧‍👦` is therefore intended to be a **single** extended grapheme cluster. Reference: UAX #29, <https://www.unicode.org/reports/tr29/#Grapheme_Cluster_Boundary_Rules> (an external specification consulted for context — it is **not** a `file:line` fact from this repository, and no code in this repository was found to implement it).

kitty 0.35.2 does **not** implement that clustering here: because the ZWJ is "combining" (`kitty/unicode-data.c:323`) but the emoji bases are **not** (they fall through to `wcwidth_std`, `kitty/wcwidth-std.h:10`, as width‑2 characters), the sequence is split at every base boundary. The `is_combining` probe makes the boundary rule explicit:

```text
is_combining probe: A+U+1F468 cursor 1->3 (NEW CELL); A+U+200D / A+U+0301 / A+U+FE0F / A+U+FE0E each 1->1 (appended)
```

Starting from `A` (cursor at `x=1`), appending the emoji **base** `U+1F468` advances the cursor `1 → 3` (a new width‑2 cell), whereas appending the ZWJ `U+200D`, a combining acute `U+0301`, or the variation selectors `U+FE0F`/`U+FE0E` each leaves the cursor at `1` (appended to `A`'s cell). Note the variation selectors are recognized by `draw_combining_char` and mapped to internal **mark indices** `VS15 = 1364, VS16 = 1365` (`kitty/unicode-data.h:5`) — these are kitty's internal `combining_type` indices, **not** the codepoints, which are `U+FE0E` (VS15) and `U+FE0F` (VS16). VS16 can promote a cell's width to 2 (`kitty/screen.c:679`–`:689`) and VS15 can demote it to 1 (`kitty/screen.c:690`–`:700`); appended to a plain `A` (not an emoji‑presentation base) neither changes `A`'s width, so the cursor stays at `1`.

### 3. The three‑slot combining cap (a second retention limit)

Even within a single cell, retention is bounded: `cc_idx` has exactly three slots (`kitty/data-types.h:226`), and overflow overwrites the last slot (`kitty/line.c:466`).

```text
Combining cap: 'a'+U+0301..U+0305 -> 'á̂̅' | U+0061 U+0301 U+0302 U+0305 (slot 2 = LAST mark)
```

Drawing `a` plus five distinct marks `U+0301 U+0302 U+0303 U+0304 U+0305` stores only `U+0061 U+0301 U+0302 U+0305`: slots 0 and 1 take the first two marks, and slot 2 is repeatedly overwritten, ending with the **last** mark `U+0305` (`U+0303` and `U+0304` are lost). This is the `kitty/line.c:466` overwrite in action.

### 4. How they interact under the 1×1 constraint

Putting it together for `👨‍👩‍👧‍👦` in a 1‑column grid:

- **Heuristic classification** makes each ZWJ a combining mark on the preceding base, and each human‑figure base an independent width‑2 cell (`kitty/unicode-data.c:11`, `kitty/wcwidth-std.h:10`).
- **Width‑driven autowrap** then fragments those independent bases: `1 < x + 2` fires the wrap check (`kitty/screen.c:821`), DECAWM is on (`kitty/screen.c:33`), so `continue_to_next_line` (`kitty/screen.c:524`) pushes each completed base into scrollback (`kitty/history.c:287`) — yielding `historybuf.count = 4` with the boy left visible.
- **State reporting** finally observes only the resulting geometry: the CPR handler reads the cursor at `x=2`, clamps it with `x--` (`kitty/screen.c:2192`), and replies `ESC[1;2R` — a cell coordinate, oblivious to the grapheme.

The column budget determines exactly how many codepoints survive on the visible row, which is further proof that retention is width/geometry‑driven rather than grapheme‑driven:

```text
ZWJ pair U+1F468 U+200D U+1F469 in 1x1 -> visible '👩' | U+1F469 (only woman survives)
cols=1  -> '👦' (1 cp: U+1F466)
cols=2  -> '👦' (1 cp: U+1F466)
cols=5  -> '👧\u200d👦' (3 cp: U+1F467 U+200D U+1F466)
cols=10 -> '👨\u200d👩\u200d👧\u200d👦' (7 cp: U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466)
```

A simpler ZWJ pair `U+1F468 U+200D U+1F469` in 1×1 keeps only the woman `U+1F469` (the last base). As the column count grows, more of the sequence fits before wrapping: 1 codepoint at 1–2 columns, 3 codepoints at 5 columns, and the **full 7 codepoints** at 10 columns (two width‑2 bases per two columns, plus the zero‑width ZWJs). If kitty were doing GB11 clustering, the survivor set would not scale linearly with raw column width like this — it would keep or drop the family as a unit.

**Rationale.** Normalization (absent), grapheme breaking (a per‑codepoint combining heuristic), and state reporting (cell‑geometry based) are three *independent* layers here. Their interaction under the 1×1 constraint is what fragments the emoji and produces a cursor‑clamped `ESC[1;2R`: no single layer "understands" the family emoji as one glyph, so the sequence is preserved only as scattered base cells, and the state query can only report the geometry those cells produced.


---

## Observation‑method appendix (full reproduction)

Everything above is reproducible. The investigation ran inside the project's build/run container at repository HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

### Build command

```text
CI=true python3 setup.py build --debug --skip-building-kitten --ignore-compiler-warnings
# -> produces kitty/fast_data_types.so
# observed size: 6142824 bytes (~6.1 MB)
```

`--skip-building-kitten` omits only the Go kitten binary (irrelevant to the screen/cell logic under study); `--ignore-compiler-warnings` tolerates unrelated windowing warnings. Neither affects the C screen model. Build entry `def build(...)` (`setup.py:1084`); extension `'kitty/fast_data_types'` (`setup.py:1091`).

The built artifact's size was measured directly with `stat`; the raw output is quoted verbatim below so the figure is grounded rather than asserted:

```text
$ stat -c '%n %s bytes' kitty/fast_data_types.so
kitty/fast_data_types.so 6142824 bytes
$ stat kitty/fast_data_types.so
  File: kitty/fast_data_types.so
  Size: 6142824   	Blocks: 12000      IO Block: 4096   regular file
Access: (0755/-rwxr-xr-x)  Uid: (    0/    root)   Gid: ( 1001/ UNKNOWN)
```

The measured size is exactly **`6142824` bytes** for this `--debug` build. (The size is build‑configuration dependent: the container ships a pre‑built *release* `kitty/fast_data_types.so` of `1221264` bytes, whereas the `--debug` rebuild documented here is `6142824` bytes.)

### Harness script (temporary; lived only under `/tmp`, removed afterward)

```python
import os, sys
sys.path.insert(0, os.getcwd())
from kitty_tests import Callbacks, parse_bytes
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty.fast_data_types import Screen, set_options
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults

# set_options() MUST precede Screen() (otherwise the C extension segfaults)
_o = Options(merge_result_dicts(defaults._asdict(),
             {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}))
finalize_keys(_o, {}); finalize_mouse_mappings(_o, {}); set_options(_o)

def mk(lines, cols):                       # cell_width=10, cell_height=20, scrollback=100
    c = Callbacks(); return Screen(c, lines, cols, 100, 10, 20, 0, c), c
def q(s, c, data):                         # feed a control sequence, capture the reply
    c.wtcbuf = b''; parse_bytes(s, data); return c.wtcbuf
def cps(text):
    return " ".join("U+%04X" % ord(ch) for ch in text)

FAMILY = '\U0001f468\u200d\U0001f469\u200d\U0001f467\u200d\U0001f466'

# ---- CASE A: family into a 1x1 grid ----
print("=== CASE A: 1x1 screen, draw FAMILY ===")
s, c = mk(1, 1); s.draw(FAMILY)
vis = str(s.line(0))
print("%-20s: %r" % ("line(0) repr", vis))
print("%-20s: %s | count = %d" % ("line(0) codepoints", cps(vis), len(vis)))
print("%-20s: %d" % ("cell[0] width", s.line(0).width(0)))
print("%-20s: %d %d" % ("cursor (x,y)", s.cursor.x, s.cursor.y))
print("%-20s: %r" % ("CPR  ESC[6n  reply", q(s, c, b'\x1b[6n')))
print("%-20s: %r" % ("DSR5 ESC[5n  reply", q(s, c, b'\x1b[5n')))
print("%-20s: %r" % ("SIZE ESC[14t reply", q(s, c, b'\x1b[14t')))
print()

# ---- Fragmentation across scrollback ----
print("=== Fragmentation across scrollback (family in 1x1) ===")
n = s.historybuf.count
print("%-20s: %d" % ("historybuf.count", n))
hs = "\n".join(str(s.historybuf.line(i)) for i in range(n))
print("%-20s: %r" % ("historybuf str", hs))
for i in range(n):
    ln = str(s.historybuf.line(i))
    if ln:
        print("hist[%d] = %r | %s" % (i, ln, cps(ln)))
    else:
        print("hist[%d] = %r         (empty trailing history line; count still reports 4)" % (i, ln))
print()

# ---- First-base wrap probe (proves the FIRST width-2 base also wraps at x=0) ----
print("=== First-base wrap probe (does the FIRST width-2 base wrap at x=0?) ===")
s1, _ = mk(1, 1); s1.draw('\U0001f468')   # draw ONE base only
h0 = str(s1.historybuf.line(0)) if s1.historybuf.count else '<none>'
print("draw 1 base U+1F468 into 1x1 -> historybuf.count = %d ; hist[0] = %r (pre-write wrap at x=0 pushed an EMPTY line)" % (s1.historybuf.count, h0))
print()

# ---- State queries ----
print("=== State queries on the 1x1 constrained screen ===")
print("DA1  ESC[c   -> %r" % q(s, c, b'\x1b[c'))
print("DA2  ESC[>c  -> %r" % q(s, c, b'\x1b[>c'))
print("XTVER ESC[>q -> %r" % q(s, c, b'\x1b[>q'))
print("DECRQM ESC[?7$p -> %r" % q(s, c, b'\x1b[?7$p'))
print()

# ---- Edge cases ----
print("=== Edge cases ===")
sp, _ = mk(1, 1); sp.draw('\U0001f468\u200d\U0001f469')
vp = str(sp.line(0))
print("ZWJ pair U+1F468 U+200D U+1F469 in 1x1 -> visible %r | %s (only woman survives)" % (vp, cps(vp)))
for cols in (1, 2, 5, 10):
    sc, _ = mk(1, cols); sc.draw(FAMILY)
    vc = str(sc.line(0))
    print("cols=%-2d -> %r (%d cp: %s)" % (cols, vc, len(vc), cps(vc)))
scap, _ = mk(1, 10); scap.draw('a\u0301\u0302\u0303\u0304\u0305')
vcap = str(scap.line(0))
print("Combining cap: 'a'+U+0301..U+0305 -> %r | %s (slot 2 = LAST mark)" % (vcap, cps(vcap)))
sn, _ = mk(1, 10); sn.draw('e\u0301')
vn = str(sn.line(0))
print("No normalization: 'e'+U+0301 -> %r | %s ; NFC U+00E9 present? %s" % (vn, cps(vn), '\u00e9' in vn))

def probe(cp):
    p, _ = mk(1, 10); p.draw('A'); before = p.cursor.x; p.draw(cp); return before, p.cursor.x
b0, a0 = probe('\U0001f468')
r200d = probe('\u200d'); r0301 = probe('\u0301'); rfe0f = probe('\ufe0f'); rfe0e = probe('\ufe0e')
assert r200d == r0301 == rfe0f == rfe0e, (r200d, r0301, rfe0f, rfe0e)
print("is_combining probe: A+U+1F468 cursor %d->%d (NEW CELL); A+U+200D / A+U+0301 / A+U+FE0F / A+U+FE0E each %d->%d (appended)" % (b0, a0, r200d[0], r200d[1]))
```

Notes: `s.draw(text)` drives `screen_draw_text` (`kitty/screen.c:866`); `parse_bytes(s, b'...')` feeds control sequences through the VT parser (`kitty_tests/__init__.py:30`); replies are captured in `Callbacks.wtcbuf` (`kitty_tests/__init__.py:51`), reset to `b''` before each query. `Screen(...)` uses `cell_width=10, cell_height=20`, which is why `ESC[14t` → `ESC[4;20;10t`. The history string is built by joining `str(historybuf.line(i))` (do **not** call `as_text_for_history_buf()` with no arguments — it requires at least one).

### Invocation

```text
LC_ALL=C.UTF-8 PYTHONPATH=/app python3 -u <script>
```

`PYTHONPATH=/app` is required so the freshly built `kitty.fast_data_types` is importable; `LC_ALL=C.UTF-8` provides a UTF‑8 locale. The script is run **unbuffered** (`python3 -u`) so that all output is flushed as it is produced.

### Complete verbatim output

```text
=== CASE A: 1x1 screen, draw FAMILY ===
line(0) repr        : '👦'
line(0) codepoints  : U+1F466 | count = 1
cell[0] width       : 2
cursor (x,y)        : 2 0
CPR  ESC[6n  reply  : b'\x1b[1;2R'
DSR5 ESC[5n  reply  : b'\x1b[0n'
SIZE ESC[14t reply  : b'\x1b[4;20;10t'

=== Fragmentation across scrollback (family in 1x1) ===
historybuf.count    : 4
historybuf str      : '👧\u200d\n👩\u200d\n👨\u200d\n'
hist[0] = '👧\u200d' | U+1F467 U+200D
hist[1] = '👩\u200d' | U+1F469 U+200D
hist[2] = '👨\u200d' | U+1F468 U+200D
hist[3] = ''         (empty trailing history line; count still reports 4)

=== First-base wrap probe (does the FIRST width-2 base wrap at x=0?) ===
draw 1 base U+1F468 into 1x1 -> historybuf.count = 1 ; hist[0] = '' (pre-write wrap at x=0 pushed an EMPTY line)

=== State queries on the 1x1 constrained screen ===
DA1  ESC[c   -> b'\x1b[?62;c'
DA2  ESC[>c  -> b'\x1b[>1;4000;35c'
XTVER ESC[>q -> b'\x1bP>|kitty(0.35.2)\x1b\\'
DECRQM ESC[?7$p -> b'\x1b[?7;1$y'

=== Edge cases ===
ZWJ pair U+1F468 U+200D U+1F469 in 1x1 -> visible '👩' | U+1F469 (only woman survives)
cols=1  -> '👦' (1 cp: U+1F466)
cols=2  -> '👦' (1 cp: U+1F466)
cols=5  -> '👧\u200d👦' (3 cp: U+1F467 U+200D U+1F466)
cols=10 -> '👨\u200d👩\u200d👧\u200d👦' (7 cp: U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466)
Combining cap: 'a'+U+0301..U+0305 -> 'á̂̅' | U+0061 U+0301 U+0302 U+0305 (slot 2 = LAST mark)
No normalization: 'e'+U+0301 -> 'é' | U+0065 U+0301 ; NFC U+00E9 present? False
is_combining probe: A+U+1F468 cursor 1->3 (NEW CELL); A+U+200D / A+U+0301 / A+U+FE0F / A+U+FE0E each 1->1 (appended)
```

### Environment

- Source repository at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (kitty **0.35.2**, `kitty/constants.py:25` → `Version(0, 35, 2)`; XTVERSION reply `b'\x1bP>|kitty(0.35.2)\x1b\\'`).
- Python **3.12.3**. kitty supports `requires-python = ">=3.8"` (`pyproject.toml:2`); CI exercises 3.8/3.9/3.10 (`.github/workflows/ci.yml:26`, `:34`, `:30`). The screen/cell behavior under study is implemented in C and is independent of the Python minor version.
- Built artifact `kitty/fast_data_types.so` = **6,142,824 bytes** (observed for this `--debug` build).

### Read‑only compliance

- **No existing repository file was modified.** The only file added is this document, `blitzy/documentation/kitty_815df1e210e0.md`.
- Temporary observation scripts lived only under `/tmp` and were **removed** after use.
- Build artifacts do not pollute the tree: `*.so` and `/build/` are gitignored (`.gitignore:1`, `.gitignore:14`).
- **Exact read‑only evidence.** After the deliverable was committed (and the temporary `/tmp` observation scripts removed), `git status --porcelain` produced **no output** — the working tree is clean and differs from the pre‑task baseline only by the single added, now‑committed document. Before that commit, the sole intended tracked change was the addition `A blitzy/documentation/kitty_815df1e210e0.md`. The empty status output is shown verbatim below:

```text
$ git status --porcelain
$
```

---

## Coverage pass

A final confirmation that every sub‑question was answered explicitly, each with an exact `file:line` citation **and** verbatim observed output.

| Sub‑question | Answered in | Key verbatim evidence | Anchor citation |
|---|---|---|---|
| **Q1** — what the buffer keeps under 1×1 | § Q1 | `historybuf.count = 4`; man/woman/girl+ZWJ on wrapped lines | `kitty/screen.c:763`, `:821`, `:33`; `kitty/history.c:287`; `kitty/line.c:466` |
| **Q2** — what the cell contains | § Q2 | `'👦'` = `U+1F466`, count 1, width 2, cursor `(2,0)` | `kitty/screen.c:836`; `kitty/data-types.h:224,226,198` |
| **Q3** — the state‑query reply | § Q3 | CPR `ESC[6n` → `b'\x1b[1;2R'`; DSR `b'\x1b[0n'`; size `b'\x1b[4;20;10t'`; DA1 `b'\x1b[?62;c'`; DA2 `b'\x1b[>1;4000;35c'`; XTVERSION `b'\x1bP>|kitty(0.35.2)\x1b\\'`; DECRQM `b'\x1b[?7;1$y'` | `kitty/screen.c:2179`, `:2192`, `:2196`, `:2186`, `:2147`–`:2164`, `:2125`, `:2128`, `:2137`, `:2203`, `:955` |
| **Q4** — synthesis | § Q4 | no‑norm `U+0065 U+0301` (NFC `U+00E9` absent); no `kitty/text-cache.*`; combining cap `U+0061 U+0301 U+0302 U+0305`; survivor scaling by columns | `kitty/unicode-data.c:11`, `:323`; `kitty/unicode-data.h:5`; `kitty/wcwidth-std.h:10`; `kitty/line.c:466` |

**All four sub‑questions are addressed.** Every quoted value — codepoints, counts, cursor coordinates, and control‑sequence reply bytes — is reproduced exactly as captured by building and running kitty headlessly, and every technical claim carries a `file:line` reference to source at HEAD `815df1e210e0`. The investigation was strictly **read‑only**: no existing repository file was changed, temporary scripts under `/tmp` were removed, and after the deliverable commit `git status --porcelain` is empty (the tree differs from its pre‑task baseline only by this newly created, now‑committed answer document).

