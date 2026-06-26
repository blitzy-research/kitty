# How kitty's Screen Buffer Handles a ZWJ Emoji Stream Under Severe Space Constraint

**Repository:** `kovidgoyal/kitty` &nbsp;·&nbsp; **Branch:** `kitty_815df1e210e0` &nbsp;·&nbsp; **Commit pin:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

This document answers four questions about how kitty's terminal screen buffer behaves when it receives a stream of Zero-Width-Joiner (ZWJ) emoji and the available display space is severely constrained (a 1×1 cell grid):

- **Q1 — Constrained-space retention:** when the terminal receives a ZWJ-joined multi-codepoint emoji and there is almost no space, how does the internal screen buffer decide what to keep and what to discard?
- **Q2 — Settled cell contents:** once everything settles, what does the terminal consider to be actually present in the affected cell — which codepoints are retained, which are dropped?
- **Q3 — State-query reflection:** if the terminal is asked to report part of its current state via a control sequence, what response does it generate, and how does that response reflect the earlier grapheme handling?
- **Q4 — Interaction under extreme constraints:** how do normalization, grapheme breaking, and state reporting interact when space is extremely limited?

The **representative input** used throughout is the family emoji:

```
'\U0001f468\u200d\U0001f469\u200d\U0001f467\u200d\U0001f466'
=  U+1F468 ZWJ U+1F469 ZWJ U+1F467 ZWJ U+1F466
=  👨  ‍  👩  ‍  👧  ‍  👦   (four width-2 emoji bases joined by three ZWJ, U+200D)
```

This is exactly the sequence exercised by kitty's own `test_zwj` (`kitty_tests/screen.py:123-131`).

> **How to read this document.** Every factual claim about kitty carries a `path:line` citation into the source at commit `815df1e2`. Every behavioral conclusion was **confirmed by building the `fast_data_types` C extension and running a headless `Screen`** — not by static reading alone. Where the running screen diverged from the static reading, the divergence is reported and reconciled (it is **not** omitted). The captured harness output is reproduced verbatim in the "Observation harness & captured results" section.

> **Scope pin.** This analysis describes the behavior **at commit `815df1e2` only**, where kitty uses a **per-codepoint** model. It must **not** be conflated with kitty's later grapheme-cluster-segmentation work (issue #8533 / DEC mode 2027), which **postdates** this commit. This is a behavioral analysis; it is **not** a fix and makes **no** design recommendations.

---

## TL;DR — Direct answers

- **Q1 (what is kept).** A cell has **fixed capacity**: one primary codepoint (`char_type ch`) plus exactly **three** combining slots (`combining_type cc_idx[3]`) — `kitty/data-types.h:223-228`. Combining marks (which include ZWJ) are appended to the first free slot; on overflow, the **last** slot is overwritten, so intermediate overflow marks are discarded — `kitty/line.c:456-467`. But the family emoji's **bases are not combining** — each width-2 base starts a *new* cell, and in a 1-column grid each base trips the wrap branch and **scrolls the previous content into the scrollback history**. So in a 1×1 grid nothing is silently destroyed: the visible cell keeps the **last** base (👦), and the earlier base+ZWJ pairs are pushed one-per-line into scrollback (empirically confirmed).
- **Q2 (settled contents).** A settled cell is reconstructed by `cell_text` as exactly `ch` followed by its surviving `cc_idx` marks (up to three) — `kitty/line.c:40-50`. In the 1×1 grid the visible cell (row 0) contains **only `U+1F466` (👦)** with **no** combining marks (no ZWJ follows the last base). In the 20-column baseline each base cell holds the base plus its trailing ZWJ as one combining mark, e.g. cell 0 = `'👨\u200d'`.
- **Q3 (state query).** The Cursor Position Report (DSR 6 / CPR) handler `report_device_status` reports a **1-based** `ESC[row;colR` and **clamps the pending-wrap state** before reporting — `kitty/screen.c:2179-2201`. On the settled 1×1 grid it emits **`ESC[1;2R`**: `cursor.x == 2` is `>= columns (1)`, so the clamp on the last line decrements to `x = 1`, reported 1-based as column 2. The reported column is the **in-bounds projection of the per-codepoint cursor advance**, so the query mirrors the per-codepoint (not per-grapheme) width model.
- **Q4 (interaction).** kitty performs **no** NFC/NFD/`unicodedata` normalization in the screen path. "Grapheme breaking" here is purely **per-codepoint** combining classification (`is_combining_char`, `kitty/unicode-data.c:11`) plus per-codepoint width (`wcwidth_std`, via `kitty/wcswidth.c`). The VT parser decodes UTF-8 and draws **codepoint runs** with **no** grapheme-cluster segmentation — `kitty/vt-parser.c:230-236`. Therefore storage, cursor advance, and the CPR all reflect one consistent **per-codepoint** model: the family emoji becomes multiple width-2 cells, not one fused grapheme.

---

## Background (external framing — not kitty source)

This short section frames kitty's behavior within established terminal/Unicode practice. It is **background only**; the code and the empirical run below are the source of truth for kitty itself.

- The codepoint **U+200D (Zero-Width Joiner)** has a standards-defined display width of **zero**, and it asks text systems to treat the codepoints around it as joined. Terminals have **historically fed each codepoint to `wcwidth` individually**, so the cursor advances by the **sum of the per-codepoint base widths** rather than by a single grapheme width. For the family emoji that means four width-2 bases ⇒ a cursor advance of **8** cells. (Mitchell Hashimoto, *Grapheme Clusters and Terminal Emulators*; WezTerm issue #4223.)
- Multiple sources explicitly place **kitty among the terminals that use traditional per-codepoint width** at this period, and describe kitty's approach as keeping the cell-width definition unchanged while leaving multi-codepoint emoji to be merged at the font/shaping level, "with trailing empty space." (WezTerm issue #4223.) This corroborates the per-codepoint model observed in the code at `815df1e2`.
- The **canonical probe** for discovering how many cells a terminal allocated for a sequence is exactly the Q3 technique: **query the cursor position (CPR), print the sequence, then query again**, and compare the delta to the `wcwidth`-expected width. (jquast/ucs-detect.)
- **DEC mode 2027 / grapheme clustering** is a *later* proposal for opt-in grapheme support in terminals. kitty's own grapheme-segmentation work (issue #8533) postdates the commit studied here, so this document deliberately describes the **earlier per-codepoint model** and disclaims the later behavior.

Sources (for framing only): `mitchellh.com/writing/grapheme-clusters-in-terminals`, `github.com/wezterm/wezterm/issues/4223`, `github.com/jquast/ucs-detect`.

---

## Q1 — Constrained-space retention: what is kept vs. discarded

**Answer.** Two distinct mechanisms decide "what is kept" — and which one applies depends on whether an incoming codepoint is a **combining** codepoint or a **base** codepoint:

1. **For combining codepoints (this is where ZWJ lives):** the cell has a **fixed capacity** of one primary codepoint plus **three** combining slots. When more than three combining marks arrive on the same cell, the **last** slot is overwritten and intermediate overflow marks are discarded.
2. **For base codepoints (the four emoji of the family sequence):** each width-2 base is **not** combining, so it starts a **new** cell. In a 1-column grid a width-2 base cannot fit, so it triggers wrapping; with the default DECAWM-on, `continue_to_next_line` linefeeds the single-row screen, **scrolling the prior cell into scrollback**. Nothing is silently overwritten in place — the earlier bases are *relocated to history*, and the **last** base remains in the visible cell.

### The fixed-capacity cell model (the mechanism behind "what is kept")

Each screen cell is a `CPUCell`, which stores exactly one primary codepoint and three combining slots — `kitty/data-types.h:223-228`:

```c
typedef struct {
    char_type ch;                 // primary codepoint   (char_type = uint32_t,  data-types.h:57)
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];     // 3 combining slots   (combining_type = uint16_t, data-types.h:62)
} CPUCell;
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");  // data-types.h:228
```

The three-element array `cc_idx[3]` (`kitty/data-types.h:226`) is **precisely** the structure that decides "what to keep when there is almost no space": a cell can retain its base codepoint plus **at most three** combining codepoints. The `static_assert` that `sizeof(CPUCell) == 12` (`kitty/data-types.h:228`) confirms this is a fixed-size record (4 bytes `ch` + 2 bytes `hyperlink_id` + 3×2 bytes `cc_idx` = 12), so the capacity is a hard structural limit, not a dynamic list.

### The append / overflow rule

Combining codepoints are attached to a cell by `line_add_combining_char` — `kitty/line.c:456-467`:

```c
void
line_add_combining_char(CPUCell *cpu_cells, GPUCell *gpu_cells, uint32_t ch, unsigned int x) {
    CPUCell *cell = cpu_cells + x;
    if (!cell->ch) {                                                        // null-cell guard (L459-462)
        if (x > 0 && (gpu_cells[x-1].attrs.width) == 2 && cpu_cells[x-1].ch) cell = cpu_cells + x - 1;
        else return; // don't allow adding combining chars to a null cell
    }
    for (unsigned i = 0; i < arraysz(cell->cc_idx); i++) {                  // fill first empty slot (L463-465)
        if (!cell->cc_idx[i]) { cell->cc_idx[i] = mark_for_codepoint(ch); return; }
    }
    cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);       // OVERFLOW: overwrite cc_idx[2] (L466)
}
```

**Rationale.** Three behaviors fall directly out of this code:

- **Stable retention of the first three marks.** The loop (`kitty/line.c:463-465`) fills `cc_idx[0]`, then `cc_idx[1]`, then `cc_idx[2]`, returning as soon as it finds an empty slot. So the primary codepoint and the **first two** combining codepoints are stored stably and never disturbed by later arrivals.
- **Overflow overwrites the last slot.** Once all three slots are full, the final statement (`kitty/line.c:466`) overwrites `cc_idx[2]` with the **most-recent** combining codepoint. Therefore the third slot holds the *latest* mark, and any **intermediate** overflow marks (the 4th, 5th, … that arrived before the last) are **discarded**.
- **Null-cell guard / wide-base attachment.** A combining mark cannot attach to an empty cell (`!cell->ch`). The guard (`kitty/line.c:459-462`) makes one exception: if the previous cell is a width-2 base with a non-zero `ch`, the mark is redirected onto that base cell instead of being dropped. This matters at the 1-column boundary, because a combining mark that arrives "after" a wide base lands on the base rather than on the empty trailing/next cell.

**Empirical confirmation of the overflow rule** (20-column screen so wrapping does not interfere): drawing `U+1F468` followed by **five** ZWJ produced a cell holding the base plus exactly **three** ZWJ marks — `line0 = '👨\u200d\u200d\u200d'`, cell-0 codepoints `[0x1f468, 0x200d, 0x200d, 0x200d]`, `cursor.x = 2`. The 4th and 5th ZWJ overflowed and overwrote `cc_idx[2]` per `kitty/line.c:466`; because every overflow mark is the identical ZWJ, the visible result is base + three ZWJ. This is exactly the fixed-capacity-plus-overflow behavior predicted by the code.

### Why bases behave differently — width, wrapping, and the 1×1 edge

The per-codepoint width comes from `wcwidth_std(ch)`, and the placement/wrap logic lives in `draw_text_loop` — `kitty/screen.c:762-845`. The relevant excerpt:

```c
char_width = wcwidth_std(ch);                                                 // screen.c:814  (emoji base => 2)
if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width)) {   // screen.c:821  WRAP TEST
    if (self->modes.mDECAWM) {
        continue_to_next_line(self);                                          // screen.c:823  DECAWM on
        init_text_loop_line(self, s);
    } else {
        self->cursor->x = self->columns - char_width;                         // screen.c:826  DECAWM off (clamp)
        if (cursor_on_wide_char_trailer(self, s)) move_cursor_off_wide_char_trailer(self, s);
    }
}
// ... write the cell, then for a wide char also write a width-0 trailer:
zero_cells(s, s->cp + self->cursor->x, s->gp + self->cursor->x);              // screen.c:835
s->cp[self->cursor->x].ch = ch;                                              // screen.c:836
self->cursor->x++;                                                           // screen.c:837
if (char_width == 2) {                                                       // screen.c:838
    s->gp[self->cursor->x-1].attrs.width = 2;                               // screen.c:839
    zero_cells(s, s->cp + self->cursor->x, s->gp + self->cursor->x);         // screen.c:840
    s->gp[self->cursor->x].attrs.width = 0;                                 // screen.c:841
    self->cursor->x++;                                                       // screen.c:842
}
```

For a width-2 base in a **1-column** grid the wrap test `self->columns < self->cursor->x + char_width` is `1 < 0 + 2`, which is **always true** (`kitty/screen.c:821`). With DECAWM **on** (the default), `continue_to_next_line` (`kitty/screen.c:524`) is taken:

```c
static void
continue_to_next_line(Screen *self) {
    linebuf_set_last_char_as_continuation(self->linebuf, self->cursor->y, true);
    self->cursor->x = 0;
    screen_linefeed(self);
}
```

It marks the current line as continued, resets `cursor.x = 0`, and performs a linefeed. On a 1-line screen the linefeed **scrolls** the existing row into the scrollback history buffer. So, for the family emoji on a 1×1 grid, each successive base displaces the previous one into history rather than overwriting it in place.

**The 1×1 grid is genuinely one cell.** The `Screen` constructor sets `columns` directly (`kitty/screen.c:112`); `screen_resize` only clamps with `MAX(1u, columns)` (`kitty/screen.c:348`); and each line allocates exactly `xnum*ynum` cells with no padding (`kitty/line-buf.c:94-95`). A 1×1 screen therefore has exactly **one** addressable cell — the worst-case constrained space.

**Empirical confirmation (1×1 grid, default DECAWM-on), captured verbatim from the harness:**

```
1x1 (draw):  line0='👦'  codepoints=['0x1f466']  cursor.x=2 cursor.y=0  text_at(0)='👦'  CPR=b'\x1b[1;2R'
             scrollback history (0 = most recent): [(0,'👧\u200d'), (1,'👩\u200d'), (2,'👨\u200d'), (3,'')]
1x1 (bytes): line0='👦'  cursor.x=2 cursor.y=0  CPR=b'\x1b[1;2R'   (identical to the draw() path)
```

**What this shows for Q1.** Under the tightest possible constraint, kitty **keeps the last base in the visible cell** and **preserves the earlier base+ZWJ pairs one-per-line in the scrollback history** (👨‍, 👩‍, 👧‍ each appear on their own history line, each carrying the ZWJ that trailed it as a combining mark). The ZWJ that *followed* each earlier base attached to that base (per the null-cell guard / wide-base attachment), and the final base 👦 has no trailing ZWJ, so it sits alone. Nothing is silently destroyed in the default configuration; the content is **distributed across history lines** by the wrap-and-scroll mechanism, with the `cc_idx[3]` capacity bounding how many marks any single cell may carry.

---

## Q2 — Settled cell contents: what is actually present once input settles

**Answer.** A cell's settled contents are reconstructed by `cell_text` as **the primary codepoint `ch` followed by the surviving `cc_idx` marks (up to three)** — `kitty/line.c:40-50`:

```c
PyObject*
cell_text(CPUCell *cell) {
    PyObject *ans;
    unsigned num = 1;
    static Py_UCS4 buf[arraysz(cell->cc_idx) + 1];
    buf[0] = cell->ch;                                                            // primary codepoint
    for (unsigned i = 0; i < arraysz(cell->cc_idx) && cell->cc_idx[i]; i++)
        buf[num++] = codepoint_for_mark(cell->cc_idx[i]);                         // up to 3 surviving marks
    ans = PyUnicode_FromKindAndData(PyUnicode_4BYTE_KIND, buf, num);
    return ans;
}
```

This cell-level view is exposed to Python through `Line.text_at`, which calls `cell_text(self->cpu_cells + xval)` (`kitty/line.c:196`); the same function is registered as the sequence-item accessor (`.sq_item`, `kitty/line.c:942`), which is why a single cell is read in Python as `line[i]`. The whole-line string `str(screen.line(0))` is the concatenation of the cells' reconstructed text.

**Mark indirection (worth documenting).** `cc_idx[i]` is **not** a raw codepoint — it is a 16-bit *index* produced by `mark_for_codepoint` (`kitty/unicode-data.c:2755`) into a fixed table of known combining/format codepoints; `codepoint_for_mark` (`kitty/unicode-data.c:2749`) maps the index back to the codepoint. ZWJ (`U+200D` = 8205) is present in that table. So a settled cell physically stores `ch` (32-bit) plus up to three **16-bit mark indices** — the combining payload is compact and capacity-limited by design, dovetailing with the `cc_idx[3]` capacity from Q1.

### Settled contents for the representative input

**1×1 grid (the constrained case), captured verbatim:**

```
1x1 (draw):  line0='👦'   codepoints=['0x1f466']   text_at(0)='👦'
```

The single visible cell contains **only `U+1F466` (👦)** and **no** combining marks. Rationale: the four bases each wrapped-and-scrolled in turn (Q1), so the *last* base to be drawn occupies the cell; because no ZWJ follows the final base in the family sequence, the cell's `cc_idx` slots are all empty and `cell_text` returns just `ch`. The three earlier base+ZWJ pairs settled into scrollback lines (`[(0,'👧\u200d'),(1,'👩\u200d'),(2,'👨\u200d'),(3,'')]`), each of which `cell_text` would reconstruct as the base plus its single trailing ZWJ mark.

**20-column baseline (for contrast), captured verbatim:**

```
BASELINE cols=20: line0='\U0001f468\u200d\U0001f469\u200d\U0001f467\u200d\U0001f466'
                  cursor.x=8  cursor.y=0  roundtrip_ok=True
per-cell text_at(0..7) = ['👨\u200d', '\x00', '👩\u200d', '\x00', '👧\u200d', '\x00', '👦', '\x00']
```

Here the cells settle as: cell 0 = `'👨\u200d'` (base `U+1F468` + ZWJ mark), cell 1 = `'\x00'` (the width-0 trailer written at `kitty/screen.c:840-841`), cell 2 = `'👩\u200d'`, cell 3 = `'\x00'`, cell 4 = `'👧\u200d'`, cell 5 = `'\x00'`, cell 6 = `'👦'` (last base, no trailing ZWJ), cell 7 = `'\x00'`. The full-line reconstruction round-trips to the exact 7-codepoint family string (`roundtrip_ok=True`), because `str(line)` concatenates the base cells and the null trailer cells contribute nothing — this reproduces `test_zwj` exactly (`kitty_tests/screen.py:127-128`).

**Net for Q2.** "What is actually present" is precisely what `cell_text` reconstructs: `ch` + surviving `cc_idx` marks. In the constrained 1×1 case the settled visible cell holds **just the last emoji base, no marks**; the joined-together family is *not* fused into one cell, and the earlier bases survive in history rather than being destroyed.

---

## Q3 — State-query reflection: what a control sequence reports

**Answer.** The relevant query is the **Device Status Report — Report Cursor Position (DSR 6 / CPR)**. kitty handles it in `report_device_status`, case `6` — `kitty/screen.c:2179-2201`:

```c
case 6:  // cursor position
    x = self->cursor->x; y = self->cursor->y;                                  // screen.c:2189
    if (x >= self->columns) {                                                  // screen.c:2190  pending-wrap clamp
        if (y < self->lines - 1) { x = 0; y++; }                               // screen.c:2191
        else x--;                                                              // screen.c:2192
    }
    if (self->modes.mDECOM) y -= MAX(y, self->margin_top);                     // screen.c:2194  origin-mode adj.
    int sz = snprintf(buf, sizeof(buf) - 1, "%s%u;%uR",
                      (private ? "?": ""), y + 1, x + 1);                      // screen.c:2196  1-based ESC[row;colR
    if (sz > 0) write_escape_code_to_child(self, ESC_CSI, buf);               // screen.c:2197
    break;
```

Two features of this handler are central:

- **It reports a 1-based position.** The emitted bytes are `ESC[<row>;<col>R` with `y + 1` and `x + 1` (`kitty/screen.c:2196`). At the home position this yields `ESC[1;1R`, which is the repo's own known-good expectation (`kitty_tests/parser.py:421-422`).
- **It clamps the pending-wrap state before reporting.** kitty leaves the cursor in a "pending wrap" / past-the-end position (`cursor.x == columns`, or beyond when a wide char was placed at the right edge) after writing the last cell, rather than wrapping eagerly. When `x >= self->columns` (`kitty/screen.c:2190`), the report projects the cursor back **in bounds**: if not on the last line it reports column 0 of the next line (`kitty/screen.c:2191`); on the last line it decrements `x` (`kitty/screen.c:2192`). The clamp is applied **only for the report** — it does not move the actual cursor.

### What the query reports for the settled 1×1 grid

**Captured verbatim** (settled 1×1 grid after drawing the family emoji, default DECAWM-on):

```
cursor.x=2 cursor.y=0   visible-cell GPU width attr = 2
DSR 6  (CPR)      ->  b'\x1b[1;2R'
DSR 5  (status)   ->  b'\x1b[0n'
DA    (\x1b[c)    ->  b'\x1b[?62;c'
DECRPM mode 7     ->  b'\x1b[?7;1$y'      (1 = set => DECAWM on by default)
```

**Step-by-step rationale for the `ESC[1;2R` result:**

1. After the draw settles, `cursor.x == 2` and `cursor.y == 0` — the cursor advanced by the **width of the last base** (`char_width == 2`, `kitty/screen.c:837,842`) and is now past the single column.
2. In case 6, `x = 2`, `y = 0`. The test `x >= self->columns` is `2 >= 1` → true (`kitty/screen.c:2190`).
3. On a 1-line screen, `y < self->lines - 1` is `0 < 0` → false, so the `else` branch runs: `x--` → `x = 1` (`kitty/screen.c:2192`).
4. DECOM is off, so no origin adjustment. The report is built 1-based: `y + 1 = 1`, `x + 1 = 2` → **`ESC[1;2R`** (`kitty/screen.c:2196`).

**Why this answers Q3.** The reported column is the **in-bounds projection of the cursor advance that the per-codepoint width handling produced**. The cursor moved because each base is width-2 (per-codepoint width, Q4), not because a grapheme cluster was measured as a unit. So the CPR is a faithful mirror of the **per-codepoint** model: had kitty done grapheme-cluster segmentation, the family emoji would have advanced the cursor differently; instead the query reflects the codepoint-by-codepoint advance, clamped to the grid. This is exactly the ucs-detect probe technique (query position → print → query position) reading out kitty's per-codepoint width.

### Neighboring state queries (scope delineation)

Other "state report" control sequences exist, but the CPR is the one that reflects grapheme handling. For completeness and to bound the topic:

- **DA — Device Attributes** (`ESC[c`): `report_device_attributes` writes `ESC[?62;c` — `kitty/screen.c:2121` (observed `b'\x1b[?62;c'`). It reports terminal capabilities, independent of cell content.
- **DECRPM — Report Mode** (`ESC[?<n>$p`): `report_mode_status` writes `ESC[?<n>;<v>$y` — `kitty/screen.c:2203` (observed `b'\x1b[?7;1$y'` for mode 7, confirming DECAWM is on by default). It reports a mode's set/reset state, not cursor geometry.
- **DECRQSS — Request Status String** (`DCS $q ... ST`): handled via the status-report branch — `kitty/screen.c:2454`. It reports the value of a setting (e.g., SGR, margins), not the grapheme placement.

None of these reflect the ZWJ handling; **CPR (DSR 6)** is the query whose response encodes the per-codepoint cursor advance, which is why it is the answer to Q3.

---

## Q4 — Interaction under extreme constraints: normalization × grapheme breaking × reporting

**Answer.** Under extreme space pressure the three concerns interact through a single, consistent **per-codepoint** pipeline. There is **no normalization stage**, "grapheme breaking" is reduced to **per-codepoint combining classification**, width is **per-codepoint**, and the state report mirrors the resulting per-codepoint cursor advance. Each link in the chain is code-grounded below.

### 1. No normalization in the screen/parser path

There is **no** NFC/NFD/`unicodedata` normalization anywhere in the screen or parser path. A repository-wide search across `kitty/` and `gen/` for `normalize` / `unicodedata` / `NFC` / `NFD` returns **no** matches for `unicodedata`, `NFC`, or `NFD` at all; the only matches are for the substring `normalize`, and every one of them is unrelated to text handling and sits outside the drawing path — the `normalized` parameter of the generated OpenGL/GLAD wrapper (`kitty/gl-wrapper.h:2409`), keyboard-shortcut normalization (`normalize_shortcut`/`normalize_shortcuts`, `kitty/conf/generate.py:510,521`), window-layout bias normalization (`normalize_biases`, `kitty/layout/base.py:197`, `kitty/layout/tall.py:111`), and tar-metadata normalization in a build-time code generator (`gen/go_code.py:850`). None of these is Unicode canonical normalization, and none lives in the VT parser or the screen drawing path. The parser hands the **raw decoded codepoint run** straight to the screen (see §3 below). Consequently the bytes a program sends are stored as-is (subject only to combining classification and width), with no canonical re-composition or decomposition that could merge or reorder the family emoji's codepoints.

### 2. "Grapheme breaking" = per-codepoint combining classification

What plays the role of "grapheme breaking" here is the **per-codepoint** predicate `is_combining_char` — `kitty/unicode-data.c:11` (with a fast path `code < 173 → false` at `kitty/unicode-data.c:13`). Two ranges matter for emoji:

- **ZWJ is combining:** `U+200D` falls in `case 0x200b ... 0x200f: return true;` — `kitty/unicode-data.c:323`. So a ZWJ is treated as a combining mark on the **preceding** cell, appended via `line_add_combining_char` (Q1).
- **Skin-tone modifiers are combining:** `case 0x1f3fb ... 0x1f3ff: return true;` — `kitty/unicode-data.c:661`. So a Fitzpatrick modifier attaches to the preceding emoji base in the **same** cell.

Crucially, **emoji bases (e.g., `U+1F468`) are not combining**, so each base **starts a new cell**. This is the entire reason the family emoji becomes *multiple* cells: bases open cells, ZWJ/modifiers attach to whatever precedes them. There is **no** Unicode Annex-#29 grapheme-cluster segmentation — the decision is made one codepoint at a time.

**Empirical corroboration of per-codepoint combining:**

```
skin-tone  U+1F469 U+1F3FD : line0='👩🏽'  codepoints=[0x1f469,0x1f3fd]  cursor.x=2   (modifier joins base's cell)
X<ZWJ>Y    'X'\u200d'Y'     : line0='X\u200dY' roundtrip_ok=True  cursor.x=2  text_at(0)='X\u200d'  (ZWJ joins X; Y is a new cell)
```

The skin-tone case keeps both codepoints in one cell because the **last** codepoint is combining; the family case ends on a **base**, so the last codepoint opens its own cell — exactly the contrast that explains the 1×1 result. (Mirrors `test_emoji_skin_tone_modifiers`, `kitty_tests/screen.py:105`, and the `X<ZWJ>Y` sub-case of `test_zwj`, `kitty_tests/screen.py:129-134`, where `cursor.x == 2`.)

### 3. Per-codepoint width, and variation-selector adjustments

Width is computed per codepoint by `wcwidth_std` (the generated table defined in `kitty/wcwidth-std.h:10`, consumed in `kitty/wcswidth.c:62`; the Python-exposed wrapper is `wcswidth_std`, `kitty/wcswidth.c:131`), with emoji-presentation handling via `is_emoji_presentation_base` (`kitty/wcswidth.c:47,54`).

**Empirical per-codepoint widths** of the family sequence (kitty's own `wcswidth`):

```
U+1F468 = 2   U+200D = 0   U+1F469 = 2   U+200D = 0   U+1F467 = 2   U+200D = 0   U+1F466 = 2
wcswidth(whole family string) = 8
```

Bases are width-2; every ZWJ is width-0; the total is the **sum of the base widths (8)** — the textbook per-codepoint result. Variation selectors are the one place a *neighboring* codepoint changes a cell's width: in `draw_combining_char` (`kitty/screen.c:662-702`), **VS16 (`U+FE0F`)** promotes an emoji-presentation base to width 2 (and may call `move_widened_char`, `kitty/screen.c:575`, at the column boundary), while **VS15 (`U+FE0E`)** demotes to width 1. **ZWJ is neither** a variation selector nor a width-changer, so it is simply appended via `line_add_combining_char` and contributes width 0. (See also `draw_second_flag_codepoint`, `kitty/screen.c:638`, for the regional-indicator/flag pairing case.)

### 4. The parser draws codepoint runs, with no grapheme segmentation

The VT parser decodes UTF-8 with a `UTF8Decoder` field (`kitty/vt-parser.c:195`) and draws the decoded output as **codepoint runs**. In `consume_normal` the whole decoded run is handed to `screen_draw_text(..., utf8_decoder.output.storage, output.pos)` (`kitty/vt-parser.c:236`), and the single-codepoint fast path calls `screen_draw_text(&ch, 1)` (`kitty/vt-parser.c:226`) — in **neither** case is the run segmented into grapheme clusters. The `Screen.draw(str)` method used by tests/harness (`kitty/screen.c:3769`) likewise calls `draw_text` directly on the UCS-4 run; both paths converge on the same per-codepoint `draw_text_loop` (`kitty/screen.c:762`). This is confirmed empirically: the 1×1 result is **identical** whether driven by `Screen.draw()` or by feeding UTF-8 bytes through the real parser (`b'\x1b[1;2R'` in both, see Q1).

### 5. Charset translation is ruled out for emoji

kitty's single-byte charset translation (`charset_translations[4][256]`, `kitty/charsets.c:13`, and `translation_table`, `kitty/charsets.c:157`) applies only to bytes `< 256`. It therefore cannot touch emoji codepoints above `U+00FF` and plays no role in ZWJ-emoji handling.

### 6. Generator evidence for the classification tables

The combining classification is generated from Unicode category data by `gen/wcwidth.py`: codepoints in category `M*` (marks) are treated as combining (`gen/wcwidth.py:132`), and category `Cf` (format characters — which includes ZWJ and other zero-width chars) is **also** added to the combining set, with the in-code comment noting it "contains things like tags and zero width chars" (`gen/wcwidth.py:136-139`). This is the upstream reason ZWJ classifies as combining in `is_combining_char`.

### Net interaction conclusion

Under extreme constraint the three concerns line up into one model:

- **Normalization:** absent — codepoints are stored as decoded.
- **Grapheme breaking:** reduced to per-codepoint combining classification — bases open cells, ZWJ/modifiers attach to the preceding cell, with cell capacity bounded at `cc_idx[3]`.
- **Reporting:** the CPR reports the in-bounds projection of the **per-codepoint** cursor advance.

So at commit `815df1e2` the family emoji is handled as **multiple width-2 cells** (cursor advance = sum of base widths = 8 on a wide screen), **not** as one fused grapheme — and the state report faithfully reflects that. (The later grapheme-segmentation work, issue #8533 / mode 2027, postdates this commit and is explicitly out of scope here.)

---

## How the screen buffer stores combining characters

This section consolidates the storage mechanism that underpins Q1 and Q2.

**Fixed-capacity record.** The unit of storage is `CPUCell` — `kitty/data-types.h:223-228`. It holds:

- `char_type ch` — the **primary** codepoint, a `uint32_t` (`kitty/data-types.h:57`).
- `combining_type cc_idx[3]` — **three** combining slots, each a `uint16_t` *index* (`kitty/data-types.h:62,226`).

The `static_assert(sizeof(CPUCell) == 12, ...)` (`kitty/data-types.h:228`) nails the layout: 4 (`ch`) + 2 (`hyperlink_id`) + 6 (`cc_idx[3]`) = 12 bytes. The combining capacity is therefore a **hard, fixed limit of three** marks per cell — there is no growth, no spill list. This is the structural fact that answers "what can a cell keep."

**Indices, not codepoints.** Each `cc_idx[i]` is an index into a generated table of known combining/format codepoints. Conversion runs both ways: `mark_for_codepoint` (`kitty/unicode-data.c:2755`) maps a codepoint → index when storing; `codepoint_for_mark` (`kitty/unicode-data.c:2749`) maps index → codepoint when reading. ZWJ (`U+200D`) is in that table, so it round-trips through a mark index like any other combining codepoint.

**Append + overflow + guard.** `line_add_combining_char` (`kitty/line.c:456-467`) implements three rules already quoted in Q1:

- **Fill** the first empty slot `cc_idx[0..2]` (`kitty/line.c:463-465`).
- **Overflow:** when all three are full, overwrite `cc_idx[2]` with the newest mark (`kitty/line.c:466`); intermediate overflow marks are dropped. (Empirically: base + 5×ZWJ ⇒ `'👨\u200d\u200d\u200d'`, base + exactly 3 marks.)
- **Null-cell guard:** refuse to attach to an empty cell unless the previous cell is a width-2 base, in which case attach to that base instead (`kitty/line.c:459-462`). This routes a ZWJ that arrives after a wide base onto the base, not onto the empty trailing/next cell.

**Why this is the answer to "what is kept under constraint."** Because the capacity is fixed at three marks and bases are not combining, the buffer's retention policy is fully determined: a cell keeps its base plus its first three combining marks (with the third slot tracking the most-recent mark), and each new **base** either opens a new cell or, in a too-narrow grid, forces a wrap/scroll that relocates prior cells into history rather than fusing them.

---

## What the cell actually contains

This section consolidates the read-back mechanism that underpins Q2.

**Reconstruction.** `cell_text` (`kitty/line.c:40-50`) builds a Python string from a cell by writing `buf[0] = cell->ch` and then appending `codepoint_for_mark(cell->cc_idx[i])` for each non-zero slot (up to three). The result is a UCS-4 string of length 1–4: the base plus its surviving marks. The loop stops at the first zero slot, so trailing empty slots contribute nothing.

**Exposure to Python.** `Line.text_at` calls `cell_text(self->cpu_cells + xval)` (`kitty/line.c:196`) and is also registered as the sequence-item slot `.sq_item` (`kitty/line.c:942`), so a single cell reads as `line[i]` in Python. The whole-line `str(screen.line(0))` concatenates the per-cell reconstructions; width-0 trailer cells (whose `ch` is 0) contribute nothing to the visible string.

**Concrete settled contents (captured).**

- **1×1 grid:** the one visible cell = `'👦'` (`U+1F466`), **no** marks — `text_at(0)='👦'`. The earlier base+ZWJ pairs live in scrollback (`'👧\u200d'`, `'👩\u200d'`, `'👨\u200d'`).
- **20-col baseline:** base cells reconstruct as `'👨\u200d'`, `'👩\u200d'`, `'👧\u200d'`, `'👦'` (cells 0/2/4/6) interleaved with width-0 trailer cells `'\x00'` (cells 1/3/5/7); the full line round-trips to the original 7-codepoint family string.

So "what the cell actually contains" is, precisely and only, `ch` + its surviving `cc_idx` marks — confirmed against the running screen.

---

## Control-sequence state query (DSR / CPR walkthrough)

This section consolidates the reporting mechanism that underpins Q3.

**Handler.** `report_device_status` case 6 (`kitty/screen.c:2179-2201`) reads `x = cursor->x`, `y = cursor->y`, applies the **pending-wrap clamp**, applies origin-mode adjustment if DECOM is set, and emits a **1-based** `ESC[row;colR` via `snprintf("%s%u;%uR", private?"?":"", y+1, x+1)` (`kitty/screen.c:2196`).

**Pending-wrap clamp (the subtle part).** kitty does **not** wrap the cursor eagerly after writing the last cell; it leaves the cursor "past the end" (`cursor.x == columns`, or beyond for a wide char at the edge). The report fixes this up *for the report only* (`kitty/screen.c:2190-2193`):

```c
if (x >= self->columns) {
    if (y < self->lines - 1) { x = 0; y++; }   // not last line: roll to next line, column 0
    else x--;                                   // last line: step back one column
}
```

**Worked example (settled 1×1 grid):** `cursor.x = 2`, `cursor.y = 0`, `columns = 1`. `2 >= 1` → clamp; `0 < 0` is false → `x-- → 1`; report 1-based → `row 1, col 2` → **`ESC[1;2R`** (captured `b'\x1b[1;2R'`).

**Home-position sanity anchor.** At the home position the same handler yields `ESC[1;1R`, matching the repo's known-good test (`kitty_tests/parser.py:421-422`); the harness reproduced `b'\x1b[1;1R'` for CPR and `b'\x1b[0n'` for DSR 5, confirming the wiring.

**Interpretation.** The reported column is the **in-bounds projection of the per-codepoint cursor advance** — it tells a querying program where kitty thinks the cursor is after the codepoint-by-codepoint draw. That is exactly why the canonical "query → print → query" probe reads kitty's per-codepoint width out of the CPR.

---

## Normalization × grapheme-breaking × reporting interaction (synthesis)

This section consolidates Q4 into a single causal chain, end to end:

```
incoming bytes
   │  (no NFC/NFD/unicodedata anywhere in this path)
   ▼
vt-parser.c: UTF8Decoder decodes to a codepoint run            (vt-parser.c:195)
   │  consume_normal -> screen_draw_text(run)                  (vt-parser.c:236)
   │  (single-cp fast path: screen_draw_text(&ch,1))           (vt-parser.c:226)
   ▼  (no grapheme-cluster segmentation at the parser)
screen.c: draw_text_loop, per codepoint                        (screen.c:762-845)
   ├─ is_combining_char(ch)?                                   (unicode-data.c:11)
   │     ZWJ 0x200d -> true (range 0x200b..0x200f)             (unicode-data.c:323)
   │     skin-tone 0x1f3fb..0x1f3ff -> true                    (unicode-data.c:661)
   │     => combining: line_add_combining_char -> cc_idx[0..2] (line.c:456-467)
   │
   └─ else base: char_width = wcwidth_std(ch) (emoji => 2)     (screen.c:814; wcswidth.c:62)
         place in a NEW cell; advance cursor by width          (screen.c:835-843)
         if cursor.x + width > columns: wrap (DECAWM on)       (screen.c:821-823)
                                          or clamp (DECAWM off)(screen.c:826)
   ▼
cell settles: ch + up to 3 surviving marks                     (line.c:40-50)
   ▼
DSR 6 / CPR: clamp pending-wrap, emit 1-based ESC[row;colR     (screen.c:2179-2201)
   = in-bounds projection of the per-codepoint cursor advance
```

Every stage is **per codepoint**: no normalization merges or reorders codepoints; "grapheme breaking" is a per-codepoint combining test; width is per-codepoint; and the report mirrors the resulting advance. The model is internally consistent — storage (Q1/Q2), advance, and reporting (Q3) all agree — and it is a **per-codepoint** model, not a grapheme-cluster model.

---

## Observation harness & captured results

Per the project rule "build and run the source to analyse the repository behavior," all conclusions above were verified empirically on a **headless** `Screen`. The `fast_data_types` C extension (which contains the `Screen` object) was built from this commit, and observation scripts were run with the launcher (`./kitty/launcher/kitty +launch <script>`). The scripts lived **outside** the repository (under `/tmp`) and were deleted after results were captured; **no** script, fixture, test, or build artifact was added to the repository.

### Harness design

The harness follows the repository's own test pattern `BaseTest.create_screen` (`kitty_tests/__init__.py:237-240`), which constructs `Screen(callbacks, lines, cols, scrollback, cell_width, cell_height, 0, callbacks)` after finalizing `Options`. Child-bound writes (such as a CPR response) accumulate in the test `Callbacks.wtcbuf` (`kitty_tests/__init__.py:50-51`), cleared via `clear()` (`kitty_tests/__init__.py:95-96`); `parse_bytes` (`kitty_tests/__init__.py:30-37`) feeds raw bytes through the **real** VT parser. The CPR probe sends `ESC[6n`, which routes to `report_device_status(6)`, and reads back `wtcbuf`.

Note on a method-name detail discovered while running: a single cell's text is read in Python by **indexing** the line (`line[i]`), which is the `.sq_item` slot bound to `text_at` (`kitty/line.c:942`); there is no `.text_at(i)` Python method on `Line`. The harness used `line[i]` accordingly.

### Captured results (verbatim)

**Per-codepoint width (kitty's own `wcswidth`):**

```
U+1F468=2  U+200D=0  U+1F469=2  U+200D=0  U+1F467=2  U+200D=0  U+1F466=2
wcswidth('\U0001f468\u200d\U0001f469\u200d\U0001f467\u200d\U0001f466') = 8
```

**(1) 20-column baseline** — reproduces `test_zwj` exactly:

```
BASELINE cols=20: line0='\U0001f468\u200d\U0001f469\u200d\U0001f467\u200d\U0001f466'
                  cursor.x=8  cursor.y=0  roundtrip_ok=True   CPR=b'\x1b[1;9R'
per-cell text_at(0..7) = ['👨\u200d','\x00','👩\u200d','\x00','👧\u200d','\x00','👦','\x00']
```

`cursor.x == 8` and the full 7-codepoint round-trip match `kitty_tests/screen.py:127-128`. CPR `ESC[1;9R` = column 8 reported 1-based as 9 (here `x=8 < columns=20`, so no clamp; `x+1 = 9`).

**(2) 1×1 grid via `Screen.draw()`** — the constrained-space case:

```
1x1 (draw): line0='👦'  codepoints=['0x1f466']  cursor.x=2 cursor.y=0  text_at(0)='👦'  CPR=b'\x1b[1;2R'
            scrollback history (0 = most recent): [(0,'👧\u200d'),(1,'👩\u200d'),(2,'👨\u200d'),(3,'')]
```

**(3) 1×1 grid via the byte path** (`parse_bytes` → `UTF8Decoder` → `screen_draw_text`) — identical to (2):

```
1x1 (bytes): line0='👦'  cursor.x=2 cursor.y=0  CPR=b'\x1b[1;2R'   (same scrollback history)
```

The draw() and byte paths agree, confirming both converge on the same per-codepoint `draw_text_loop` (`kitty/screen.c:762`) with no grapheme segmentation.

**State queries on the settled 1×1 grid (default DECAWM-on):**

```
DSR 6 (CPR)    -> b'\x1b[1;2R'
DSR 5 (status) -> b'\x1b[0n'
DA   (ESC[c)   -> b'\x1b[?62;c'
DECRPM mode 7  -> b'\x1b[?7;1$y'     (1 => DECAWM set/on by default)
home CPR sanity-> b'\x1b[1;1R'       (matches kitty_tests/parser.py:421-422)
```

**Corroborating per-codepoint combining:**

```
skin-tone U+1F469 U+1F3FD : line0='👩🏽' codepoints=[0x1f469,0x1f3fd] cursor.x=2  (modifier joins base cell)
X<ZWJ>Y 'X\u200dY'        : line0='X\u200dY' roundtrip_ok=True cursor.x=2 text_at(0)='X\u200d'
overflow U+1F468 + 5×ZWJ  : line0='👨\u200d\u200d\u200d' cell0=[0x1f468,0x200d,0x200d,0x200d] cursor.x=2
```

### Reconciliation: static reading vs. running screen

The **static hypothesis** for the 1×1 grid (each width-2 base trips the wrap branch, `continue_to_next_line` linefeeds/scrolls, the last base lands in the single cell with no trailing ZWJ, and the CPR is the clamped projection of `cursor.x`) was **confirmed** by the run: visible cell = `'👦'`, `cursor.x = 2`, `cursor.y = 0`, CPR = `ESC[1;2R`. One refinement the run made explicit (and which static reading alone could understate): the earlier bases are **not lost** — they are pushed **one-per-line into the scrollback history**, each carrying the ZWJ that trailed it as a combining mark. The settled *visible* cell holds only the last base; the joined family is distributed across history lines, bounded per cell by the `cc_idx[3]` capacity.

### Divergence found and reported (not omitted): DECAWM-off + width-2 + 1-column ⇒ crash

Exercising the DECAWM **off** state on a 1×1 grid revealed a reproducible divergence that must be recorded as an **observed runtime property** of this commit (reported factually; this is **not** a design recommendation and the analysis suggests no change):

```
DECAWM ON,  1×1, width-2 emoji      -> SURVIVED  (line0='👨', cursor.x=2)   [takes continue_to_next_line, screen.c:823]
DECAWM off, 1×1, width-1 'X'        -> SURVIVED  (line0='X',  cursor.x=1)   [1-1=0, no underflow]
DECAWM off, 2-col, width-2 emoji    -> SURVIVED  (line0='👨', cursor.x=2)   [2-2=0, no underflow]
DECAWM off, 2-col, 2× width-2 emoji -> SURVIVED  (2nd wraps to 2-2=0)
DECAWM off, 1×1, width-2 emoji      -> SIGSEGV (signal 11, exit 139), 100% reproducible
```

**Root cause (code-grounded).** In the DECAWM-off branch of `draw_text_loop` the cursor is clamped with `self->cursor->x = self->columns - char_width;` (`kitty/screen.c:826`). `columns` is an unsigned index type; with `columns == 1` and `char_width == 2`, `1u - 2` **underflows** to `4294967295`. The subsequent unconditional writes `zero_cells(s->cp + self->cursor->x, ...)` and `s->cp[self->cursor->x].ch = ch` (`kitty/screen.c:835-836`, and the wide-char trailer at `:840`) then index the cell arrays far out of bounds, causing the segfault. The exact trigger condition is **DECAWM off AND `columns < char_width`**. With the **default DECAWM-on**, the wrap branch (`continue_to_next_line`, `kitty/screen.c:823`) is taken instead, so the default behavior is safe — which is why every other answer in this document (all at default DECAWM-on) is stable and reproducible.

**Reconciliation outcome.** The crash does not contradict any Q1–Q4 answer; all of those describe the **default DECAWM-on** path, which the run confirmed is safe. The DECAWM-off crash is an *additional* empirical finding at the extreme 1-column boundary, surfaced precisely because the build-and-run mandate exercised a path that static reading would have flagged only as an unsigned subtraction. It is documented here for completeness and faithfulness to the running extension, per the rule that a divergent empirical result must be reconciled and explained, never omitted.

---

## Rationale and closing note

**Rationale, recapped.** Each answer is grounded in a specific mechanism and confirmed against the running screen:

- *Q1 (what is kept)* follows from the **fixed `cc_idx[3]` capacity** (`kitty/data-types.h:226`) and the **append/overflow rule** (`kitty/line.c:463-466`) for combining marks, combined with the fact that **bases are not combining** and therefore wrap-and-scroll in a too-narrow grid (`kitty/screen.c:821-823`, `:524`). Confirmed: 1×1 keeps the last base in view and relocates earlier bases to history.
- *Q2 (settled contents)* follows from **`cell_text`** reconstructing `ch` + surviving marks (`kitty/line.c:40-50`). Confirmed: visible cell = `'👦'` with no marks.
- *Q3 (state query)* follows from **`report_device_status` case 6** with its **pending-wrap clamp** (`kitty/screen.c:2189-2196`). Confirmed: `ESC[1;2R`, the in-bounds projection of the per-codepoint advance.
- *Q4 (interaction)* follows from **no normalization**, **per-codepoint combining classification** (`kitty/unicode-data.c:11,323,661`), **per-codepoint width** (`kitty/wcswidth.c:62,131`), and **no grapheme segmentation in the parser** (`kitty/vt-parser.c:226,236`). Confirmed: identical results via the draw() and byte paths; width sum = 8.

**Closing pin.** At commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, kitty implements a **per-codepoint** model for ZWJ emoji: a ZWJ-joined sequence becomes **multiple width-2 cells** (with ZWJ stored as a zero-width combining mark on the preceding base), the cursor advance equals the **sum of the base widths**, and the CPR reports the in-bounds projection of that advance. kitty's **later** grapheme-cluster-segmentation work — issue #8533 and DEC private mode 2027 — **postdates** this commit and changes how width is assigned to ZWJ-based emoji; it is intentionally **out of scope** here. This document describes observed behavior at `815df1e2` only and recommends no changes.

---

## Appendix — Evidence / citation map

All entries are **reference-only** (read as evidence; never modified for this task).

| Claim | Source location |
|---|---|
| `char_type` = `uint32_t` | `kitty/data-types.h:57` |
| `combining_type` = `uint16_t` | `kitty/data-types.h:62` |
| `CPUCell` struct (`ch`, `hyperlink_id`, `cc_idx[3]`) | `kitty/data-types.h:223-227` |
| `static_assert(sizeof(CPUCell) == 12)` | `kitty/data-types.h:228` |
| `cell_text` reconstructs `ch` + ≤3 marks | `kitty/line.c:40-50` |
| `Line.text_at` → `cell_text`; `.sq_item` binding | `kitty/line.c:196`, `:942` |
| `line_add_combining_char` (guard / fill / overflow) | `kitty/line.c:456-467` (guard `:459-462`, fill `:463-465`, overflow `:466`) |
| `continue_to_next_line` (continuation + linefeed) | `kitty/screen.c:524` |
| `move_widened_char` | `kitty/screen.c:575` |
| `draw_second_flag_codepoint` | `kitty/screen.c:638` |
| `draw_combining_char` (VS16 promote / VS15 demote) | `kitty/screen.c:662-702` |
| `draw_text_loop` (width, wrap, wide-char placement) | `kitty/screen.c:762-845` (width `:814`, wrap test `:821`, DECAWM-on `:823`, DECAWM-off clamp `:826`, placement `:835-843`) |
| `Screen.draw` Python method → `draw_text` | `kitty/screen.c:3769` |
| `report_device_status` DSR/CPR (case 6, clamp, 1-based) | `kitty/screen.c:2179-2201` (clamp `:2190-2193`, emit `:2196`) |
| `report_device_attributes` (DA) | `kitty/screen.c:2121` |
| `report_mode_status` (DECRPM) | `kitty/screen.c:2203` |
| DECRQSS status-report branch | `kitty/screen.c:2454` |
| `is_combining_char` (fast path) | `kitty/unicode-data.c:11` (`code<173` `:13`) |
| ZWJ range `0x200b..0x200f → true` | `kitty/unicode-data.c:323` |
| skin-tone range `0x1f3fb..0x1f3ff → true` | `kitty/unicode-data.c:661` |
| `codepoint_for_mark` / `mark_for_codepoint` | `kitty/unicode-data.c:2749`, `:2755` |
| `wcwidth_std` (generated table) / used | `kitty/wcwidth-std.h:10`; `kitty/wcswidth.c:62` |
| `wcswidth_std` (Python-exposed) | `kitty/wcswidth.c:131` |
| `is_emoji_presentation_base` | `kitty/wcswidth.c:47,54` |
| `UTF8Decoder` field | `kitty/vt-parser.c:195` |
| single-codepoint draw path | `kitty/vt-parser.c:226` |
| `consume_normal` → `screen_draw_text(run)` | `kitty/vt-parser.c:236` |
| `charset_translations[4][256]` / `translation_table` (single-byte only) | `kitty/charsets.c:13`, `:157` |
| `M*` marks / `Cf` format → combining (generator) | `gen/wcwidth.py:132`, `:136-139` |
| line allocation `xnum*ynum`, no padding | `kitty/line-buf.c:94-95` |
| `Screen` sets `columns` directly; resize clamps `MAX(1u,...)` | `kitty/screen.c:112`, `:348` |
| `test_zwj` (family emoji, `cursor.x==8`, round-trip; `X<ZWJ>Y`) | `kitty_tests/screen.py:123-134` |
| `test_emoji_skin_tone_modifiers` | `kitty_tests/screen.py:105` |
| `test_regional_indicators` | `kitty_tests/screen.py:112` |
| `BaseTest.create_screen` (headless `Screen`) | `kitty_tests/__init__.py:237-240` |
| `Callbacks` / `wtcbuf` / `clear` / `parse_bytes` | `kitty_tests/__init__.py:50-51`, `:95-96`, `:30-37` |
| DSR/CPR known-good (`ESC[0n`, `ESC[1;1R`) | `kitty_tests/parser.py:418-422` |

**Build/version context (reference-only):** `setup.py` builds the `fast_data_types` extension; `pyproject.toml:2` declares `requires-python = ">=3.8"`; `.github/workflows/ci.yml` tests Python 3.8/3.9/3.10 with `gcc` and `clang`; `go.mod:3` pins `go 1.22` (Go is not needed to answer the Unicode question).
