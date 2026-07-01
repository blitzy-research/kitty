# ZWJ Emoji in a 1×1 Cell: Buffer Retention, Settled Contents, and State Reporting in kitty `815df1e210e0`

## Summary

This document answers — with **runtime-verified evidence** — how the **kitty** terminal emulator at commit `815df1e210e0` (the 0.35.2 line) handles a stream of Zero-Width-Joiner (ZWJ) codepoints forming a multi-codepoint emoji when it is written into an extremely space-constrained screen, specifically the user's example of **a one-by-one (1×1) cell**.

**Headline answer (all values observed at runtime, shown verbatim below):** when the four-person family emoji `👨‍👩‍👧‍👦` = `U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466` (7 codepoints: four base emoji joined by three ZWJs) is written into a 1×1 screen, only the **last base emoji `U+1F466` (👦)** survives in the single cell, with **cell width 2**; and when the terminal is subsequently queried with the Cursor Position Report control sequence `ESC [ 6 n`, it writes back exactly `b'\x1b[1;2R'`. Its sibling Device Status Report query `ESC [ 5 n` returns exactly `b'\x1b[0n'`.

**Why this happens (one-line rationale):** the cell is a fixed 12-byte record holding one base codepoint plus three combining slots (`kitty/data-types.h:223-228`); this commit uses a **classic per-codepoint width + combining-character** model with **no Unicode normalization** and **no grapheme-cluster segmentation**, so under the 1×1 constraint each successive base emoji simply overwrites the single cell, leaving only the last base, and the reported cursor column is a direct function of that surviving cell's width.

This answer was written **after building and running** the real code paths (run-before-write); every behavioral claim below is placed next to the specific verbatim output line that demonstrates it, and every literal is cited with a `file:line` reference resolved against commit `815df1e210e0`.

---

## How this was investigated (methodology)

The investigation drove kitty's **production** C code paths headlessly, with no display server or GPU, using the harness kitty ships under `kitty_tests/`.

**The extension was already built and importable.** The terminal core (parser, screen model, cell storage, Unicode width, state reporting) is compiled from `kitty/*.c` into the Python extension `kitty/fast_data_types.so`. It imported successfully, so no rebuild was needed. Had a rebuild been required, the command would be:

```console
CFLAGS="-Wno-error" python3 setup.py build --skip-building-kitten
```

`-Wno-error` bypasses only an **environment-only** `wayland-protocols` enum mismatch at `glfw/wl_window.c:668` (newer `XDG_TOPLEVEL_STATE_CONSTRAINED_*` values trip `-Werror=switch`); it is a build flag, **not** a source edit.

**Runtime environment actually used:** Python **3.13.7** (`/opt/kitty-venv/bin/python`), with `PYTHONPATH` set to the repository root. (This differs from an earlier draft that referenced Python 3.12.3; per the report-exactly-what-is-observed rule, the true interpreter for the observations below is 3.13.7. The behavior is identical.)

**How a screen is built and read.** A `Screen` is constructed directly; its positional constructor is `Screen(callbacks, LINES, COLUMNS, scrollback, cell_width, cell_height, 0, callbacks)` — note the order is **(callbacks, LINES, COLUMNS, …)**. The helper `create_screen` in the harness confirms this order:

```python
# kitty_tests/__init__.py:237-240
def create_screen(self, cols=5, lines=5, scrollback=5, cell_width=10, cell_height=20, options=None):
    self.set_options(options)
    c = Callbacks()
    s = Screen(c, lines, cols, scrollback, cell_width, cell_height, 0, c)
```

So a **1×1** screen is `Screen(c, 1, 1, 0, 10, 20, 0, c)` (1 line, 1 column).

**How bytes are fed.** Input bytes are pushed through the production `vt-parser.c` / `screen.c` path via `parse_bytes` (`kitty_tests/__init__.py:30`):

```python
# kitty_tests/__init__.py:30
def parse_bytes(screen, data, dump_callback=None):
```

**How state is read.** The settled line text is read via `str(s.line(0))`, the per-cell width via `s.line(0).width(x)`, and the cursor via `s.cursor.x` / `s.cursor.y` — all idioms drawn from the existing tests in `kitty_tests/screen.py` and `kitty_tests/datatypes.py`.

**How control-sequence replies are captured.** Anything the terminal writes *back to the child* (its DSR/CPR replies) is captured by the `Callbacks.write` method into the `wtcbuf` buffer (`kitty_tests/__init__.py:50-51`), and reset by `Callbacks.clear` (`kitty_tests/__init__.py:95-96`):

```python
# kitty_tests/__init__.py:50-51
def write(self, data) -> None:
    self.wtcbuf += bytes(data)
```

```python
# kitty_tests/__init__.py:95-96
def clear(self) -> None:
    self.wtcbuf = b''
```

kitty's own tests use exactly this mechanism for DSR/CPR (`kitty_tests/parser.py:418-422`): `ESC [ 5 n` asserts `wtcbuf == b'\033[0n'` and `ESC [ 6 n` asserts `wtcbuf == b'\033[1;1R'` (note `\033` == `\x1b` == ESC).

**Read-only + cleanup.** The three temporary observation scripts (`/tmp/obs_zwj.py`, `/tmp/obs_cc.py`, `/tmp/obs_wide.py`) live in `/tmp`, **outside** the repository, and were removed after use. `kitty/fast_data_types.so` is a gitignored build artifact. The repository is left unchanged apart from this single answer document; `git status --porcelain` reports a clean working tree.

---

## R1 — What the internal screen buffer keeps (cell model + retention algorithm)

### The cell-storage model: one base codepoint + exactly three combining slots

The internal screen buffer stores each cell as a **fixed-size** record. There is **no variable-length grapheme buffer**, which is the structural reason a long ZWJ sequence cannot be retained whole in one cell.

```c
// kitty/data-types.h:57
typedef uint32_t char_type;
// kitty/data-types.h:62
typedef uint16_t combining_type;
```

```c
// kitty/data-types.h:223-227
typedef struct {
    char_type ch;
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];
} CPUCell;
// kitty/data-types.h:228
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
```

- The base codepoint is the single `char_type ch` field (`kitty/data-types.h:224`).
- Combining characters go into exactly **three** slots: `combining_type cc_idx[3]` (`kitty/data-types.h:226`).
- The whole record is a fixed **12 bytes** (`static_assert(sizeof(CPUCell) == 12, ...)` at `kitty/data-types.h:228`) — a hard structural cap: one base + at most three combining marks per cell.
- The stored width is a 2-bit field, `#define WIDTH_MASK (3u)` (`kitty/data-types.h:211`), so the maximum width a cell records is 3.

### The retention decision: overwrite the *last* combining slot when full

The concrete "what to keep" decision lives in `line_add_combining_char` (`kitty/line.c:457`). It fills the first empty combining slot; when all three are occupied, it **overwrites the last slot** rather than growing the cell:

```c
// kitty/line.c:457
line_add_combining_char(CPUCell *cpu_cells, GPUCell *gpu_cells, uint32_t ch, unsigned int x) {
    ...
    // kitty/line.c:463-465  (fill first empty slot)
    for (unsigned i = 0; i < arraysz(cell->cc_idx); i++) {
        if (!cell->cc_idx[i]) { cell->cc_idx[i] = mark_for_codepoint(ch); return; }
    }
    // kitty/line.c:466  (all slots full -> overwrite the LAST slot)
    cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);
}
```

This is the exact retention rule: the **last** combining slot is overwritten; the middle marks that overflowed are dropped. (This is demonstrated directly by the combining-overflow control in the Control Experiments section: input `e` + five marks settles to `U+0065 U+0301 U+0302 U+0305`, dropping `U+0303` and `U+0304`.)

### Why ZWJ is treated as a combining mark (width 0)

ZWJ (`U+200D`) is classified as a combining character, so it is routed to combining handling and attached to the preceding base cell rather than occupying its own cell. The classifying range is:

```c
// kitty/unicode-data.c:323
case 0x200b ... 0x200f:
```

`0x200D` (ZWJ) falls inside `0x200b ... 0x200f`, so `is_combining_char(0x200D)` is true. Its width is zero — confirmed at runtime:

```console
$ python3 /tmp/obs_wide.py
...
wcwidth(0x200D) = 0
```

### Variation selectors and emoji-presentation width (VS16 `0xfe0f`, VS15 `0xfe0e`)

Beyond ZWJ, two Unicode variation selectors can change a base emoji's **width**, and this commit handles them explicitly — this is the emoji-presentation-width half of the "grapheme/combining handling" the question asks about. Whether a base is *eligible* to be widened or narrowed is decided by `is_emoji_presentation_base` (`kitty/wcwidth-std.h:2942`):

```c
// kitty/wcwidth-std.h:2942
is_emoji_presentation_base(uint32_t code) {
```

The two selectors are named combining-mark constants (`kitty/unicode-data.h:5`):

```c
// kitty/unicode-data.h:5
static const combining_type VS15 = 1364, VS16 = 1365;
```

**VS16 (`0xfe0f`) widens a default-narrow emoji base to width 2; VS15 (`0xfe0e`) narrows it to width 1.** In the draw path this fixup is applied by `draw_combining_char` after the selector has been attached to the preceding cell (`kitty/screen.c:663-701`):

```c
// kitty/screen.c:679  — VS16 branch: widen the preceding base to width 2
if (ch == 0xfe0f) {
    // kitty/screen.c:682
    if (gpu_cell->attrs.width != 2 && cpu_cell->cc_idx[0] == VS16 && is_emoji_presentation_base(cpu_cell->ch)) {
        gpu_cell->attrs.width = 2;   // kitty/screen.c:683
    // ...
// kitty/screen.c:690  — VS15 branch: narrow the preceding base to width 1
} else if (ch == 0xfe0e) {
    // kitty/screen.c:696
    if (gpu_cell->attrs.width == 2 && cpu_cell->cc_idx[0] == VS15 && is_emoji_presentation_base(cpu_cell->ch)) {
        gpu_cell->attrs.width = 1;   // kitty/screen.c:697
```

The stateless width machine encodes the same rule (`kitty/wcswidth.c:47-61`): a trailing `0xfe0f` adds 1 to a width-1 emoji-presentation base, and a trailing `0xfe0e` subtracts 1 from a width-2 base — each guarded by `is_emoji_presentation_base`:

```c
// kitty/wcswidth.c:46-58  (VS16 / VS15 cases; the is_emoji_presentation_base-guarded bodies span kitty/wcswidth.c:47-61)
case 0xfe0f: {
    if (is_emoji_presentation_base(state->prev_ch) && state->prev_width == 1) {
        ans += 1;
        state->prev_width = 2;
    } else state->prev_width = 0;
} break;
case 0xfe0e: {
    if (is_emoji_presentation_base(state->prev_ch) && state->prev_width == 2) {
        ans -= 1;
        state->prev_width = 1;
    } else state->prev_width = 0;
} break;
```

**Claims, each next to its single supporting evidence line** (all observed from `/tmp/obs_wide.py`; the selectors are themselves zero-width and only adjust the *preceding* cell):

- **A person/family base emoji `U+1F468` is intrinsically width 2**, so it needs no VS16 widening → `wcwidth(0x1F468) = 2`.
- **VS16 `0xfe0f` contributes zero width of its own** (its effect is applied to the preceding base by the `is_emoji_presentation_base`-guarded branch above) → `wcwidth(0xFE0F) = 0`.
- **VS15 `0xfe0e` contributes zero width of its own** (likewise a modifier of the preceding base) → `wcwidth(0xFE0E) = 0`.

**Rationale:** `wcwidth()` reports each codepoint's *own* contribution, so both variation selectors report `0`; their widening/narrowing is applied to the *previous* cell's stored 2-bit width field (`WIDTH_MASK`, `kitty/data-types.h:211`) by the `is_emoji_presentation_base`-guarded code above. In the 1×1 ZWJ scenario the input contains no VS15/VS16, so these branches never fire and the surviving `U+1F466` simply keeps the width-2 it computed from `wcwidth_std` (R2). These selectors are documented here because they are the emoji-presentation-width mechanism the per-codepoint model relies on, and the question asks for the width handling to be covered by name.

### How the stream is placed under the 1×1 constraint

Autowrap (DECAWM) is on by default:

```c
// kitty/screen.c:33
static const ScreenModes empty_modes = {0, .mDECAWM=true, .mDECTCEM=true, .mDECARM=true};
```

```c
// kitty/modes.h:50
#define DECAWM (7 << 5)
```

In the text draw loop `draw_text_loop` (`kitty/screen.c:763`):

- Combining characters (including ZWJ) are routed away from occupying a cell:
  ```c
  // kitty/screen.c:806
  if (UNLIKELY(is_combining_char(ch))) {
      // kitty/screen.c:810
      draw_combining_char(self, s, ch);
      // kitty/screen.c:811
      continue;
  }
  ```
- Each non-combining codepoint's width is computed per-codepoint:
  ```c
  // kitty/screen.c:814
  char_width = wcwidth_std(ch);
  ```
- A wide (width-2) base that does not fit wraps under DECAWM:
  ```c
  // kitty/screen.c:821
  if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width)) {
      // kitty/screen.c:823
      continue_to_next_line(self);
  ```
- The base is written by first zeroing the cell, then setting `ch` — so each new base **overwrites** whatever was in the single 1×1 cell:
  ```c
  // kitty/screen.c:835
  zero_cells(s, s->cp + self->cursor->x, s->gp + self->cursor->x);
  // kitty/screen.c:836
  s->cp[self->cursor->x].ch = ch;
  ```
- Width-2 bookkeeping (leader/trailer) then advances the cursor by 2 (`kitty/screen.c:838-843`).

**The 1×1 consequence:** each of the four base emoji (`U+1F468`, `U+1F469`, `U+1F467`, `U+1F466`) is a width-2 non-combining codepoint, so each overwrites the single cell in turn; the three ZWJs are zero-width combining marks that attach to whichever base is current. When a base is overwritten, any ZWJ attached to the prior base is discarded with it. Therefore only the **last** base — `U+1F466` — remains. This is confirmed verbatim in R2.

---

## R2 — Settled cell contents (verbatim)

**Command / code that produced the evidence** (`/tmp/obs_zwj.py`, a 1×1 screen fed the family emoji):

```python
# /tmp/obs_zwj.py  (key lines)
c = Callbacks()
s = f.Screen(c, 1, 1, 0, 10, 20, 0, c)  # 1 line x 1 column
emoji = "\U0001F468\u200D\U0001F469\u200D\U0001F467\u200D\U0001F466"
parse_bytes(s, emoji.encode('utf-8'))
line0 = s.line(0)
print("settled line(0) repr:", repr(str(line0)))
print("settled codepoints: %s (count=%d)" % (" ".join("U+%04X" % ord(ch) for ch in str(line0)), len(str(line0))))
print("cell(0).width = %d" % line0.width(0))
print("cursor.x = %d  cursor.y = %d" % (s.cursor.x, s.cursor.y))
```

**Verbatim output:**

```text
settled line(0) repr: '👦'
settled codepoints: U+1F466 (count=1)
cell(0).width = 2
cursor.x = 2  cursor.y = 0
```

**Claims, each next to its single supporting evidence line:**

- **Only the last base emoji survives.** → `settled codepoints: U+1F466 (count=1)` and `settled line(0) repr: '👦'`. The cell holds exactly one codepoint, `U+1F466` (👦, the last of the four bases); the man/woman/girl bases and all three ZWJs are gone.
- **The settled cell width is 2.** → `cell(0).width = 2`. Even in a 1-column screen, the surviving emoji records the width-2 it computed via `wcwidth_std` (`kitty/screen.c:814`), stored in the 2-bit width field (`WIDTH_MASK`, `kitty/data-types.h:211`).
- **The cursor ends at column 2 (0-based x), row 0.** → `cursor.x = 2  cursor.y = 0`. The single width-2 base advanced the cursor by 2 via the width-2 bookkeeping at `kitty/screen.c:838-843`.

**Rationale:** the surviving content is `U+1F466` — not a "collapsed" family cluster — because the buffer never segments a grapheme cluster; it stores one base per cell (`kitty/data-types.h:224`) and overwrites the cell on each new base (`kitty/screen.c:835-836`). No normalization altered the stored codepoint (see R5).

---

## R3 — The state-query response (verbatim bytes)

The relevant control sequence is the **Cursor Position Report (CPR)**, i.e. **Device Status Report parameter 6**, `ESC [ 6 n`. Its sibling state query is **DSR 5**, `ESC [ 5 n`.

**Command / code that produced the evidence** (same 1×1 screen, after the emoji settled):

```python
# /tmp/obs_zwj.py  (continued)
c.clear(); parse_bytes(s, b'\x1b[6n')
print("ESC[6n (DSR6/CPR) -> %r" % c.wtcbuf)
c.clear(); parse_bytes(s, b'\x1b[5n')
print("ESC[5n (DSR5) -> %r" % c.wtcbuf)
```

**Verbatim output:**

```text
ESC[6n (DSR6/CPR) -> b'\x1b[1;2R'
ESC[5n (DSR5) -> b'\x1b[0n'
```

**Byte-level confirmation** (hex of the same buffers):

```text
CPR bytes hex: 1b5b313b3252 = b'\x1b[1;2R'   (ESC [ 1 ; 2 R)
DSR5 bytes hex: 1b5b306e     = b'\x1b[0n'     (ESC [ 0 n)
```

**Claims, each next to its single supporting evidence line:**

- **The CPR (`ESC [ 6 n`) response is exactly `b'\x1b[1;2R'`** (i.e. `ESC [ 1 ; 2 R`, meaning row 1, column 2, 1-based). → `ESC[6n (DSR6/CPR) -> b'\x1b[1;2R'`.
- **The DSR 5 (`ESC [ 5 n`) response is exactly `b'\x1b[0n'`** (i.e. `ESC [ 0 n`, "terminal OK"). → `ESC[5n (DSR5) -> b'\x1b[0n'`.

**Where these bytes come from in the source.** The parser dispatches the DSR control sequence to the screen op:

```c
// kitty/vt-parser.c:1172-1173
case DSR:
    CALL_CSI_HANDLER1P(report_device_status, 0, '?');
```

and `report_device_status` (`kitty/screen.c:2179`) generates the bytes:

```c
// kitty/screen.c:2186  (DSR 5 -> "0n")
write_escape_code_to_child(self, ESC_CSI, "0n");
// kitty/screen.c:2189  (DSR 6 / CPR: read cursor)
x = self->cursor->x; y = self->cursor->y;
// kitty/screen.c:2190-2192  (clamp when x is past the last column)
if (x >= self->columns) { if (y < self->lines - 1) { x = 0; y++; } else x--; }
// kitty/screen.c:2196  (format 1-based row;col)
int sz = snprintf(buf, sizeof(buf) - 1, "%s%u;%uR", (private ? "?": ""), y + 1, x + 1);
```

**Rationale for the exact `1;2R`** (state the clamp math explicitly): the settled cursor is raw `cursor.x = 2` (the width-2 base advanced x by 2). In `report_device_status`, `x (=2) >= columns (=1)` is true, so the clamp runs; since `y (=0) < lines-1 (=0)` is false, the `else x--;` branch executes, giving `x = 1`. The response is formatted 1-based as `y+1 = 1`, `x+1 = 2` → `CSI 1;2R` = `b'\x1b[1;2R'`. This matches the observed bytes exactly.

---

## R4 — How the response reflects the earlier grapheme/combining handling

The CPR response is the **observable proxy** for how many cells the settled content consumed, which in turn is determined entirely by the combining/width handling in the draw loop.

**The reported column is a direct function of the surviving cell's width.** In the 1×1 case the sole surviving base is `U+1F466`, a width-2 emoji. Its base write (`kitty/screen.c:836`) plus the width-2 advance (`kitty/screen.c:838-843`) drove `cursor.x` to 2, which the CPR clamp (`kitty/screen.c:2190-2192`) reduces to a reported column of 2:

```text
# from /tmp/obs_zwj.py
cursor.x = 2  cursor.y = 0
ESC[6n (DSR6/CPR) -> b'\x1b[1;2R'
```

**The zero-width ZWJs contributed nothing to the advance.** The three ZWJs were routed through `draw_combining_char` (`kitty/screen.c:810`) and did not occupy a cell or advance the cursor; only base-emoji widths did. This is why the reported column reflects a single width-2 cell, not seven codepoints. Runtime confirmation of ZWJ's zero width:

```text
# from /tmp/obs_wide.py
wcwidth(0x200D) = 0
```

**The contrast makes the causation observable.** With room to spare (the roomy control), two width-2 bases plus a zero-width ZWJ produced `cursor.x = 4`, and the CPR reflected that advance:

```text
# from /tmp/obs_wide.py  (20-column screen, farmer emoji)
cursor.x = 4
ESC[6n (DSR6/CPR) -> b'\x1b[1;5R'
```

So the state report is a faithful mirror of the combining/width handling: `b'\x1b[1;2R'` in the 1×1 case encodes "one width-2 cell survived"; `b'\x1b[1;5R'` in the roomy case encodes "two width-2 cells plus a zero-width joiner were laid down (2 + 0 + 2 = 4, reported 1-based as column 5)". The CPR does not report codepoint counts or cluster identity — it reports **cells consumed**, which is exactly what the per-codepoint combining model produced.

---

## R5 — Normalization + grapheme breaking + state reporting under extreme constraints

The user's question names three mechanisms by name — **normalization**, **grapheme breaking**, and **state reporting** — and asks how they interact when the terminal is under extreme constraints. Each is addressed explicitly below.

### Normalization — ABSENT

The text path performs **no Unicode normalization** (no NFC/NFD/NFKC); codepoints are stored **as received**. A search of the text-path C sources returns nothing:

```console
$ grep -niE "nfc|nfd|nfkc|normaliz" kitty/screen.c kitty/line.c kitty/line-buf.c kitty/unicode-data.c kitty/wcswidth.c kitty/vt-parser.c
$ echo "exit=$?"
exit=1
```

(empty output; exit status 1 = no matches).

The only `normaliz`-style matches anywhere in the C tree are **not** text normalization — they are font-rasterization gray-level normalization and OpenGL vertex-attribute normalization. Full verbatim output:

```console
$ grep -rniE "normaliz" kitty/*.c kitty/*.h
kitty/freetype.c:496:    // Normalize gray levels to the range [0..255]
kitty/gl-wrapper.h:797:#define GL_NORMALIZE 0x0BA1
kitty/gl-wrapper.h:1077:#define GL_SIGNED_NORMALIZED 0x8F9C
kitty/gl-wrapper.h:1340:#define GL_UNSIGNED_NORMALIZED 0x8C17
kitty/gl-wrapper.h:1365:#define GL_VERTEX_ATTRIB_ARRAY_NORMALIZED 0x886A
kitty/gl-wrapper.h:2409:typedef void (GLAD_API_PTR *PFNGLVERTEXATTRIBPOINTERPROC)(GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void * pointer);
kitty/gl-wrapper.h:9139:static void GLAD_API_PTR glad_debug_impl_glVertexAttribPointer(GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const void * pointer) {
kitty/gl-wrapper.h:9140:    _pre_call_gl_callback("glVertexAttribPointer", (GLADapiproc) glad_glVertexAttribPointer, 6, index, size, type, normalized, stride, pointer);
kitty/gl-wrapper.h:9141:    glad_glVertexAttribPointer(index, size, type, normalized, stride, pointer);
kitty/gl-wrapper.h:9142:    _post_call_gl_callback(NULL, "glVertexAttribPointer", (GLADapiproc) glad_glVertexAttribPointer, 6, index, size, type, normalized, stride, pointer);
```

Every match is either the font-rasterization comment at `kitty/freetype.c:496` or an OpenGL symbol / `glVertexAttribPointer` wrapper in `kitty/gl-wrapper.h`; none is Unicode text normalization.

**Accuracy note:** `kitty/fast-file-copy.c` contains **no** `normaliz` match; the apparent case-insensitive `nfd` hits there are the substring inside `infd`/`outfd` (I/O file descriptors), e.g. `kitty/fast-file-copy.c:19  copy_with_buffer(int infd, int outfd, ...)`. The genuine `normaliz` references are `kitty/freetype.c:496` and `kitty/gl-wrapper.h` (as listed above) — **not** `kitty/fast-file-copy.c`.

**Rationale:** because there is no normalization, the stored codepoints are exactly those received; the retention decision is governed purely by the cell model (`kitty/data-types.h:223-228`) plus the draw loop (`kitty/screen.c:763`, `:835-836`), not by any pre-combining or canonical-composition step.

### Grapheme breaking — per-codepoint, NOT cluster segmentation

This commit uses the **classic per-codepoint width + combining-char** model, **not** Unicode grapheme-cluster segmentation. A repository search for the grapheme-clustering / text-sizing machinery returns no matches:

```console
$ grep -rniE "grapheme_cluster|text_sizing|multicell" kitty/*.c kitty/*.h
$ echo "exit=$?"
exit=1
```

(empty output; exit status 1 = no matches).

There is likewise **no DEC private mode 2027** (the ecosystem's grapheme-clustering indicator) in the terminal state code; `2027` appears only in Unicode/width data tables, never as a mode in `screen.c`/`modes.h`:

```console
$ grep -nE "2027" kitty/screen.c kitty/modes.h
$ echo "exit=$?"
exit=1
$ grep -rln "2027" kitty/*.c kitty/*.h
kitty/unicode-data.c
kitty/wcwidth-std.h
```

The direct runtime consequence is that a ZWJ sequence is **not** held together as one unit; each codepoint's width is handled independently and joiners are zero-width combining marks. The roomy control makes this per-codepoint summation visible: the farmer `🧑‍🌾` advances the cursor by 2 + 0 + 2 = 4, not by a single cluster width:

```text
# from /tmp/obs_wide.py
settled codepoints: U+1F9D1 U+200D U+1F33E (count=3)
widths: (2, 0, 2, 0, 0)
cursor.x = 4
```

(A later kitty release added grapheme-cluster segmentation; commit `815df1e210e0` predates it — see the ecosystem framing section.)

### State reporting — reflects the settled cell/cursor

State reporting is performed by `report_device_status` (`kitty/screen.c:2179`), which for CPR reads the live cursor (`kitty/screen.c:2189`), clamps it (`kitty/screen.c:2190-2192`), and formats a 1-based `CSI row;col R` (`kitty/screen.c:2196`). Its output is therefore a function of whatever the settled cell/cursor became.

### How the three interact when space is exhausted

1. **Normalization** never runs, so the 7 input codepoints are stored exactly as received (nothing is pre-composed or decomposed).
2. **Grapheme breaking** is per-codepoint, so the ZWJ sequence is not treated as one indivisible cluster; ZWJ is a zero-width combining mark attached to the preceding base, and each base is an independent width-2 codepoint.
3. Under the **1×1 constraint**, the draw loop's overwrite behavior (`kitty/screen.c:835-836`) means each successive base emoji overwrites the single cell, and any ZWJ attached to a prior base is discarded with it — so only the last base (`U+1F466`) remains (R2).
4. **State reporting** (CPR) then reports a column derived from that single settled cell's width-2 advance, clamped to the 1-column screen → `b'\x1b[1;2R'` (R3/R4).

The net effect: the extreme constraint collapses a 7-codepoint ZWJ emoji down to **one** base codepoint, and the state query makes that collapse **externally observable** as the column value `2` in `b'\x1b[1;2R'`.

---

## Control experiments (why the 1×1 outcome is what it is)

Two controls isolate what is caused by the 1×1 limit versus what is intrinsic to the model.

### Control A — combining-overflow (proves the overwrite-last-slot rule)

**Command / code:** `/tmp/obs_cc.py` — a 10-column screen fed base `e` (`U+0065`) followed by five combining marks `U+0301 U+0302 U+0303 U+0304 U+0305`.

```python
# /tmp/obs_cc.py  (key lines)
s = f.Screen(c, 2, 10, 0, 10, 20, 0, c)
data = "e\u0301\u0302\u0303\u0304\u0305"
parse_bytes(s, data.encode('utf-8'))
line0 = s.line(0)
print("settled codepoints:", " ".join("U+%04X" % ord(ch) for ch in str(line0).rstrip(' ')))
print("cell(0).width = %d" % line0.width(0))
```

**Verbatim output:**

```text
settled codepoints: U+0065 U+0301 U+0302 U+0305 (count=4)
cell(0).width = 1
```

**Claim + evidence:** the cell keeps the base + the first two marks + the **last** mark, dropping the two middle overflow marks (`U+0303` and `U+0304` are dropped) → `settled codepoints: U+0065 U+0301 U+0302 U+0305 (count=4)`. This is the concrete demonstration of the "overwrite the last combining slot when full" rule at `kitty/line.c:466`: the three slots hold `U+0301`, `U+0302`, then the third (last) slot is overwritten from `U+0303`→`U+0304`→`U+0305`, so only `U+0305` remains in it. The cell width stays 1 → `cell(0).width = 1` (`e` is a width-1 base and combining marks are zero-width).

### Control B — roomy screen (proves the per-codepoint model, isolates the 1×1 cause)

**Command / code:** `/tmp/obs_wide.py` — a 20-column screen fed the farmer emoji `U+1F9D1 U+200D U+1F33E`.

```python
# /tmp/obs_wide.py  (key lines)
s = f.Screen(c, 2, 20, 0, 10, 20, 0, c)
data = "\U0001F9D1\u200D\U0001F33E"
parse_bytes(s, data.encode('utf-8'))
line0 = s.line(0)
print("settled line(0) repr:", repr(str(line0)))
print("widths:", tuple(line0.width(x) for x in range(5)))
print("cursor.x = %d" % s.cursor.x)
c.clear(); parse_bytes(s, b'\x1b[6n'); print("ESC[6n (DSR6/CPR) -> %r" % c.wtcbuf)
for cp in (0x200D, 0x1F468, 0xFE0F, 0xFE0E):
    print("wcwidth(0x%04X) = %d" % (cp, f.wcwidth(cp)))
```

**Verbatim output:**

```text
settled line(0) repr: '🧑\u200d🌾'
settled codepoints: U+1F9D1 U+200D U+1F33E (count=3)
widths: (2, 0, 2, 0, 0)
cursor.x = 4
ESC[6n (DSR6/CPR) -> b'\x1b[1;5R'
wcwidth(0x200D) = 0
wcwidth(0x1F468) = 2
wcwidth(0xFE0F) = 0
wcwidth(0xFE0E) = 0
```

**Claims + evidence:**

- **With room, both bases are kept and the ZWJ is attached to the first** → `settled codepoints: U+1F9D1 U+200D U+1F33E (count=3)` and `settled line(0) repr: '🧑\u200d🌾'`. Nothing was overwritten because the second base landed in a *different* cell.
- **Per-codepoint widths sum 2 + 0 + 2 = 4** → `widths: (2, 0, 2, 0, 0)` and `cursor.x = 4`.
- **The CPR reflects that advance** → `ESC[6n (DSR6/CPR) -> b'\x1b[1;5R'` (raw x = 4, not past the 20-column edge, so reported 1-based as column 5).
- **ZWJ `U+200D` is zero-width** → `wcwidth(0x200D) = 0`.
- **A person/family base emoji `U+1F468` is intrinsically width 2** → `wcwidth(0x1F468) = 2`.
- **The VS16 variation selector `0xFE0F` is itself zero-width** → `wcwidth(0xFE0F) = 0`.
- **The VS15 variation selector `0xFE0E` is itself zero-width** → `wcwidth(0xFE0E) = 0`.

These four probes confirm the **per-codepoint width model**: the joiner and both variation selectors each report width `0`, while each base carries its own width, so the family/farmer widths sum member-by-member (`2 + 0 + 2 = 4`). The variation selectors report `0` for their *own* contribution even though they can still adjust the *preceding* base's stored width — see the variation-selector subsection under R1 (`is_emoji_presentation_base` at `kitty/wcwidth-std.h:2942`; VS16 `0xfe0f` / VS15 `0xfe0e` fixups at `kitty/screen.c:663-701` and `kitty/wcswidth.c:47-61`).

### The key contrast (the observable proof)

```text
1×1   -> ESC[6n -> b'\x1b[1;2R'   (single base survived; column 2)
roomy -> ESC[6n -> b'\x1b[1;5R'   (both bases survived; column 5)
```

The retention outcome is caused by **overwrite-under-constraint in the draw loop** (`kitty/screen.c:835-836`), **not** by any grapheme-aware collapsing: given room, the model happily keeps both bases and the joiner (Control B); only the 1×1 limit forces the single-base outcome. The CPR column value (`2` vs `5`) is the observable proxy for how many cells the settled content consumed.

---

## Ecosystem framing (secondary, general context)

This section is secondary to the runtime evidence above and is provided only for context.

- **Classic per-codepoint model.** The behavior observed here — summing each codepoint's `wcwidth` so a ZWJ emoji advances the cursor by the sum of its members' widths (e.g. 2 + 0 + 2 = 4 for the farmer) — is the classic terminal model that predates grapheme-cluster awareness. kitty `815df1e210e0` exhibits exactly this model.
- **The extreme-constraint edge case is a known terminal problem.** When only one cell remains and a wide emoji arrives, it wraps; a later narrowing variation selector does not move it back — illustrating that a string's rendered width depends on the width of the screen it is drawn on. The 1×1 scenario is the sharpest form of this edge case.
- **Grapheme clustering is later work.** kitty subsequently added an explicit cell-splitting algorithm based on Unicode grapheme segmentation (and the associated text-sizing protocol), which fixes long-standing ZWJ-emoji issues. Commit `815df1e210e0` predates that change, which is why the per-codepoint model is what is observed (empty grep for `grapheme_cluster|text_sizing|multicell`, R5).
- **CPR is the standard state query for this purpose.** The Cursor Position Report (DSR 6) is the sequence Unicode-compliance tooling uses to ask "where is the cursor?" after printing test characters and to compare the reported column against the expected width — which is precisely how this document uses it.
- **DEC mode 2027 context.** The broader ecosystem uses DEC private mode 2027 as a binary indicator of grapheme-clustering support; its absence in this commit (R5) is consistent with the per-codepoint model.

---

## Coverage pass

Re-reading the question, every distinct thing it names is addressed below by name, with a pointer to where:

- [x] **ZWJ (`U+200D`)** — classified as a combining character at `kitty/unicode-data.c:323` (`case 0x200b ... 0x200f:`); zero width confirmed at runtime `wcwidth(0x200D) = 0` (R1, R4, R5).
- [x] **Multi-codepoint emoji** — the family `👨‍👩‍👧‍👦` = `U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466` (R2); the farmer `🧑‍🌾` = `U+1F9D1 U+200D U+1F33E` (Control B).
- [x] **Internal screen buffer** — the fixed 12-byte `CPUCell` (one base + three combining slots) at `kitty/data-types.h:223-228`; retention algorithm `line_add_combining_char` at `kitty/line.c:457-466` (R1).
- [x] **1×1 cell** — `Screen(c, 1, 1, 0, 10, 20, 0, c)`; each base overwrites the single cell so only the last base survives (methodology, R1, R2).
- [x] **Settled cell contents** — `U+1F466` (👦), width 2 (R2); combining-overflow control settles to `U+0065 U+0301 U+0302 U+0305` (Control A).
- [x] **Control-sequence state query + its response** — CPR `ESC [ 6 n` → `b'\x1b[1;2R'`; DSR 5 `ESC [ 5 n` → `b'\x1b[0n'` (R3); source at `kitty/vt-parser.c:1172-1173` and `kitty/screen.c:2179-2196`.
- [x] **Normalization** — ABSENT; genuine `normaliz` matches are `kitty/freetype.c:496` + `kitty/gl-wrapper.h` (font/OpenGL), **not** `kitty/fast-file-copy.c` (R5).
- [x] **Grapheme breaking** — per-codepoint, no cluster segmentation; empty grep for `grapheme_cluster|text_sizing|multicell`, and no DEC mode 2027 (R5).
- [x] **State reporting** — `report_device_status` at `kitty/screen.c:2179`; the reported column reflects the settled cell width and clamped cursor (R3, R4).

**Scope / read-only confirmation:** this document (`blitzy/documentation/kitty_815df1e210e0.md`) is the only change to the repository. No kitty source, header, test, configuration, or build file was modified. The temporary observation scripts `/tmp/obs_zwj.py`, `/tmp/obs_cc.py`, `/tmp/obs_wide.py` lived outside the repository and were removed after use; `git status --porcelain` reports a clean working tree.

