# How kitty handles a ZWJ multi-codepoint emoji in a 1×1 cell — and what a state-report query then says

**Subject repository:** [kovidgoyal/kitty](https://github.com/kovidgoyal/kitty) @ commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Investigation type:** read-only, *build-and-run-first*. The terminal engine was built and driven through its **real VT parser**; every behavioral claim below is backed by **byte-exact captured output** and a **`file:line` citation** naming the responsible function or struct.
**Deliverable:** this single Markdown document. No existing repository file was modified, created, or deleted; the temporary observation script lived outside the repository tree and was removed afterward.

---

## Section 1 — The question, decomposed, with the direct answer up front

### 1.1 The question (verbatim)

> I am trying to get a practical understanding of how kitty handles complex Unicode at runtime, especially in cases where the screen state is hard to reason about just by reading the code. When the terminal receives a stream of zero width joiners that form a multi codepoint emoji, how does the internal screen buffer decide what to keep when there is almost no space available, such as a one by one cell? What does the terminal think is actually present in that cell once everything settles? If the terminal is then asked to report part of its current state through a control sequence query, what response does it generate and how does that reflect the earlier grapheme handling? I want to understand how normalization, grapheme breaking, and state reporting interact when the terminal is under extreme constraints. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.

### 1.2 Four decomposed objectives

- **OBJ-1 — Retention under constraint:** how the internal screen buffer decides *what to keep* when a ZWJ emoji stream arrives into a 1×1 cell.
- **OBJ-2 — Settled cell content:** what the terminal believes is *actually present* in the cell once the stream is fully processed.
- **OBJ-3 — State reporting:** what response a control-sequence state query generates, and *how that response reflects the earlier grapheme handling*.
- **OBJ-4 — Interaction:** how *normalization*, *grapheme breaking*, and *state reporting* interact under extreme constraint.

### 1.3 Direct answer (TL;DR)

In a **1×1** grid the buffer keeps only the **last width-2 base emoji** of the ZWJ stream. Each successive base emoji is written into the single cell and **overwrites** whatever was there before (clearing that cell's combining slots), while each ZWJ (`U+200D`) is classified as a *combining* character and **appended** as a combining mark to whichever base is currently in the cell. Because the canonical family sequence `👨‍👩‍👧‍👦` ends on a **base** codepoint (Boy, `U+1F466`) with **no trailing ZWJ**, the settled cell contains exactly **one** codepoint — `U+1F466` (`👦`). The cursor sits at **`x=2`** (the width-2 base advanced it one column past the single column), and a Cursor-Position-Report query `CSI 6 n` replies with the exact bytes **`b'\x1b[1;2R'`** because `report_device_status` clamps the out-of-range column back to the last real column. kitty performs **no Unicode normalization** and runs **no grapheme-segmentation (UAX #29) state machine** on this path; it decides everything by **width classification** plus **combining-character membership**, using generated **Unicode 15.0.0** tables. The state report therefore does not "know" it saw a family emoji — it only echoes the cursor column that the width-2 base produced under the one-column clamp.

Everything above is **OBSERVED** (see Sections 4–6 for the captured output) and grounded in code (see the citations throughout). The remainder of this document shows the exact commands, the complete unedited output, and the mechanism behind each result.

---

## Section 2 — Environment and build

### 2.1 Versions actually observed in this environment

| Item | Value (observed here) | How obtained |
|------|-----------------------|--------------|
| CPython | **3.13.7** | `python3 --version` inside the venv `/tmp/kitty-venv` |
| C compiler | **gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0** | `gcc --version` |
| Unicode data version | **15.0.0** | first line of `kitty/unicode-data.c` (see below) |
| Repository commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` | `git rev-parse HEAD` |
| Intended reproduction image | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas`) | provided environment |

> **Note on versions (observed vs. planning reference):** the task's planning notes referenced CPython 3.12.3 / gcc 13.3.0. The actual host used for this investigation runs **CPython 3.13.7 / gcc 15.2.0**. Per the rule to *build and run in the default configuration and report the value that produces*, the versions above are the ones actually observed. The behavior under investigation is determined by the C source logic and the **source-baked Unicode 15.0.0 tables**, not by the Python or gcc version; the captured behavioral output is therefore version-independent, and this is confirmed empirically by the stability check in Section 2.5.

The Unicode provenance is stated in the generated data file itself — `kitty/unicode-data.c:L1`:

```c
// Unicode data, built from the Unicode Standard 15.0.0
// Code generated by gen-wcwidth.py, DO NOT EDIT.
```

### 2.2 Build command — canonical, and what actually happened here

The documented, **canonical** build command for kitty's C extension is:

```bash
python3 setup.py build
```

**Observed result in this environment: the canonical command SUCCEEDED with exit code `0`.** No warning-suppression workaround was necessary here. The reason is that `wayland-protocols` is **not installed** on this host, so kitty's build system disables the Wayland (GLFW) backend and never compiles the GUI/windowing code that is the source of the warning discussed below. The head of the captured build log:

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
```

A clean rebuild (the gitignored `build/` directory and `kitty/fast_data_types.so` were removed first, then `python3 setup.py build` was re-run) compiled **85 objects** and linked **4 targets**, ending in `done` with exit code `0`. Head and tail of that log:

```text
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
...
[84/85] Compiling kitty/simd-string-256.c ...
[85/85] Compiling kitty/gl-wrapper.c ...
 done
[1/4] Linking kitty/fast_data_types ...
[2/4] Linking [x11] kitty/glfw-x11 ...
[3/4] Linking kittens/transfer/rsync ...
[4/4] Linking launcher ...
 done
```

Note that only the **X11** GLFW backend is linked (`kitty/glfw-x11`); there is **no `glfw-wayland`** link and the object count is **85** rather than the 122 seen on a host that also builds the Wayland backend — both differences follow directly from Wayland being absent here.

### 2.3 The sandbox-only warning workaround (disclosed for reproducibility)

On a host **where `wayland-protocols` *is* present** (as in the original planning sandbox), the full default build additionally compiles the Wayland GUI code under `-pedantic-errors -Werror`, and that compile can fail on an unrelated `-Werror=switch` warning coming from a newer system copy of `wayland-protocols`:

```text
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_TOP' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM' not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
```

On such a host the documented workaround is:

```bash
export CI=true
python3 setup.py build --debug --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag flips exactly one line in the build system — `setup.py:L491`:

```python
werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'
```

This flag controls **only whether warnings are fatal**; it does **not** change the terminal engine's logic. The affected file (`glfw/wl_window.c`) is GUI/windowing code — the terminal engine proper (`kitty/screen.c`, `kitty/line.c`, `kitty/unicode-data.c`) compiles cleanly with or without it. **Therefore the observed Unicode/state-report behavior in this document is independent of the warning flag**, whether the build succeeds outright (as here) or requires the workaround (as on a Wayland-equipped host).

### 2.4 Import check (the engine we exercise)

The terminal engine is the CPython extension `kitty.fast_data_types`, which provides the `Screen`, `Line`, and `Cursor` types that this investigation drives:

```bash
PYTHONPATH=. python3 -c "import kitty.fast_data_types as f; from kitty.fast_data_types import Screen, Line, Cursor; print('import OK ->', f.__file__)"
# import OK -> /.../kitty/fast_data_types.so
```

### 2.5 Reproducibility and repository integrity

The observation script (Section 10) was executed **twice** from the repository root; the two runs produced **byte-identical** output (identical MD5), confirming stability. (The planning notes report 4/4 identical runs including one after the canonical-build attempt; this investigation independently reproduced 2/2 identical runs.) The build artifacts `kitty/fast_data_types.so` and `kitty/launcher/kitty` are gitignored (`.gitignore` contains `*.so`, `/build/`, and `/kitty/launcher/kitt*`), and `git status --porcelain` was empty after the rebuild — the repository working tree was left unchanged.

---

## Section 3 — Methodology: the canonical entry point

**Direct statement:** every input in this investigation flows into the terminal engine as **UTF-8 bytes through the real VT parser** — never through a convenience/direct-draw API, remote control, or a debug hook. This satisfies the requirement that the observed values be produced by the exact code path a real program would exercise.

### 3.1 The byte path

The driver is `kitty_tests.parse_bytes` — `kitty_tests/__init__.py:L30-L36`:

```python
def parse_bytes(screen, data, dump_callback=None):
    data = memoryview(data)
    while data:
        dest = screen.test_create_write_buffer()
        s = screen.test_commit_write_buffer(data, dest)
        data = data[s:]
        screen.test_parse_written_data(dump_callback)
```

The three `screen.test_*` methods are thin C bindings that push bytes into the genuine parser rather than around it:

- `test_create_write_buffer` — `kitty/screen.c:L4755`
- `test_commit_write_buffer` — `kitty/screen.c:L4762`
- `test_parse_written_data` — `kitty/screen.c:L4772`

The bytes are decoded and dispatched by the real byte-stream state machine in `kitty/vt-parser.c`; drawn text is routed to `screen_draw_text`, which contains the per-codepoint draw loop cited throughout Section 4.

### 3.2 Capturing control-sequence replies

Replies that the engine emits "to the child" (e.g., a Cursor Position Report) are captured by the test `Callbacks` object, whose `write` method accumulates them — `kitty_tests/__init__.py:L50-L51`:

```python
    def write(self, data) -> None:
        self.wtcbuf += bytes(data)
```

A query such as `CSI 6 n` is itself fed **through `parse_bytes`** (i.e., through the real parser), and the reply is read back byte-for-byte from `callbacks.wtcbuf`. This is why the reports in Section 6 are shown as literal `bytes` objects.

### 3.3 Building a 1×1 (and wider) `Screen`

The `Screen` constructor argument order is mirrored from `kitty_tests.create_screen` — `kitty_tests/__init__.py:L237-L240`:

```python
    def create_screen(self, cols=5, lines=5, scrollback=5, cell_width=10, cell_height=20, options=None):
        self.set_options(options)
        c = Callbacks()
        s = Screen(c, lines, cols, scrollback, cell_width, cell_height, 0, c)
        return s
```

So the order is `Screen(callbacks, LINES, COLS, scrollback, cell_width, cell_height, 0, callbacks)`. A **1×1** screen is therefore `Screen(cb, 1, 1, 0, 10, 20, 0, cb)` (1 line, 1 column, cell size 10×20 px). The observation script uses `cell_width=10, cell_height=20`, which is why the pixel-size reports in Section 6 come out as multiples of 10 and 20.

### 3.4 A note on the direct-draw API (non-canonical)

kitty's own existing tests in `kitty_tests/screen.py` frequently call `s.draw(<str>)` for convenience — for example `test_emoji_skin_tone_modifiers`, `test_regional_indicators`, and `test_variation_selectors`. `s.draw(<str>)` is the **direct draw API**: it takes an already-decoded Python `str` and bypasses the UTF-8 byte parser. Any value obtained that way would be **non-canonical** for this question. This document deliberately uses UTF-8 **bytes** through `parse_bytes` instead, so that the observed values are exactly what the byte-stream parser produces.

---

## Section 4 — Part 1: What the buffer keeps (retention under constraint) → OBJ-1

**Direct answer:** each cell has a **fixed capacity** — exactly one base codepoint plus **three** combining-mark slots — and a ZWJ stream into a 1×1 grid collapses to the **final base** because every new width-2 base emoji **overwrites** the single cell (clearing its combining slots), while every ZWJ in between is merely **appended** as a combining mark to the base that is currently present.

### 4.1 The fixed-capacity cell model

The CPU-side per-cell structure stores one base `ch` plus a **3-element** combining array — `kitty/data-types.h:L223-L228`:

```c
typedef struct {
    char_type ch;
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];
} CPUCell;
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
```

This is the entire basis of "what can be kept" in a cell: **one** base codepoint and **at most three** combining marks. There is no per-cell growable list.

### 4.2 The retention / overflow decision

The routine that decides how a combining mark is stored is `line_add_combining_char` — `kitty/line.c:L456-L467`:

```c
void
line_add_combining_char(CPUCell *cpu_cells, GPUCell *gpu_cells, uint32_t ch, unsigned int x) {
    CPUCell *cell = cpu_cells + x;
    if (!cell->ch) {
        if (x > 0 && (gpu_cells[x-1].attrs.width) == 2 && cpu_cells[x-1].ch) cell = cpu_cells + x - 1;
        else return; // don't allow adding combining chars to a null cell
    }
    for (unsigned i = 0; i < arraysz(cell->cc_idx); i++) {
        if (!cell->cc_idx[i]) { cell->cc_idx[i] = mark_for_codepoint(ch); return; }
    }
    cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);
}
```

Three behaviors matter for OBJ-1:

1. It targets the cell at `cpu_cells + x`.
2. It **refuses to attach a combining mark to a null (empty) cell** — *unless* the previous cell is a **width-2 base** (`x > 0 && width == 2 && cpu_cells[x-1].ch`), in which case it retargets the mark onto that wide base. This is exactly how the trailing half of a wide emoji still "belongs" to the base to its left.
3. It **fills the first empty `cc_idx` slot**; and once **all three** are full, the fourth (and any later) mark **overwrites the last slot** `cc_idx[arraysz(cc_idx)-1]` (i.e. `cc_idx[2]`). This is the overflow rule exercised in Section 4.5.

### 4.3 Where a ZWJ is routed (not given its own cell)

In the per-codepoint draw loop (inside `screen_draw_text`), a combining codepoint never occupies a fresh cell — `kitty/screen.c:L805-L812`:

```c
            if (is_ignored_char(ch)) continue;
            if (UNLIKELY(is_combining_char(ch))) {
                if (UNLIKELY(is_flag_codepoint(ch))) {
                    if (draw_second_flag_codepoint(self, ch)) continue;
                } else {
                    draw_combining_char(self, s, ch);
                    continue;
                }
```

- `is_ignored_char(ch)` (`kitty/unicode-data.c:L671`) is the gate that would *drop* a codepoint entirely; ZWJ is **not** in it (see Section 7).
- `is_combining_char(ch)` (`kitty/unicode-data.c:L11`) is **true** for ZWJ `U+200D` (see Section 7), so ZWJ takes the combining branch and is handed to `draw_combining_char` — it is **appended**, never allocated a new cell.

`draw_combining_char` locates the previous cell and calls `line_add_combining_char` — `kitty/screen.c:L663-L710` (head shown; the VS16/VS15 width-flip branches are analysed in Section 7):

```c
draw_combining_char(Screen *self, text_loop_state *s, char_type ch) {
    bool has_prev_char = false;
    index_type xpos = 0, ypos = 0;
    if (self->cursor->x > 0) {
        ypos = self->cursor->y;
        xpos = self->cursor->x - 1;
        has_prev_char = true;
    } else if (self->cursor->y > 0) {
        ypos = self->cursor->y - 1;
        xpos = self->columns - 1;
        has_prev_char = true;
    }
    if (has_prev_char) {
        CPUCell *cp; GPUCell *gp;
        linebuf_init_cells(self->linebuf, ypos, &cp, &gp);
        line_add_combining_char(cp, gp, ch, xpos);
        ...
```

### 4.4 Why a width-2 base overwrites the single cell

Each **base** emoji in the family sequence (`👨`, `👩`, `👧`, `👦`) is a width-2 codepoint. Writing it into the 1×1 grid places it in the single cell (column 0) and advances the cursor to `x=2` — one past the only column. The **next** base emoji is a fresh `screen_draw_text` cell write into that same single cell, which **clears the previous `ch` and its `cc_idx`** before storing the new base. Consequently, of the whole stream, only the **last** base and any combining marks that trail it survive. In the family sequence there is **no** trailing ZWJ after the final base (Boy), so only `U+1F466` remains.

*(Width classification for base codepoints is done by `wcwidth_std` — `kitty/wcswidth.c:L62`, table `kitty/wcwidth-std.h` — see Section 7.)*

### 4.5 OBSERVED: the incremental before / during / after trace (1×1)

This is the single most important piece of evidence for OBJ-1. The family stream was fed **one codepoint at a time** into a fresh 1×1 `Screen`, printing the cell after each step. Command (the full script is in Section 10):

```text
FAMILY = U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466
each chunk fed via: parse_bytes(s, chunk.encode('utf-8'))   # real VT parser
```

Complete, unedited output:

```text
   [before (empty screen)]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [after Man U+1F468 (width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa8']
      str(line0)   = '👨'
      codepoints    = ['U+1F468']  (len=1)
      as_ansi()     = '👨'
      cursor        = (x=2, y=0)
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👨\u200d'
      codepoints    = ['U+1F468', 'U+200D']  (len=2)
      as_ansi()     = '👨\u200d'
      cursor        = (x=2, y=0)
   [after Woman U+1F469 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa9']
      str(line0)   = '👩'
      codepoints    = ['U+1F469']  (len=1)
      as_ansi()     = '👩'
      cursor        = (x=2, y=0)
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👩\u200d'
      codepoints    = ['U+1F469', 'U+200D']  (len=2)
      as_ansi()     = '👩\u200d'
      cursor        = (x=2, y=0)
   [after Girl U+1F467 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa7']
      str(line0)   = '👧'
      codepoints    = ['U+1F467']  (len=1)
      as_ansi()     = '👧'
      cursor        = (x=2, y=0)
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👧\u200d'
      codepoints    = ['U+1F467', 'U+200D']  (len=2)
      as_ansi()     = '👧\u200d'
      cursor        = (x=2, y=0)
   [after Boy U+1F466 (final width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa6']
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
```

**Reading the trace (this is the "what to keep" decision made visible):**

- **before** — the cell is empty (`len=0`, cursor `x=0`).
- **base placed** — after Man, the cell holds one codepoint `U+1F468` and the cursor is at `x=2` (width-2 base advanced past the single column).
- **ZWJ appended** — after the ZWJ, the cell holds **two** codepoints `['U+1F468', 'U+200D']`: the ZWJ was appended into the base's first combining slot (`line_add_combining_char`, §4.2), not given its own cell; the cursor stays at `x=2`.
- **next base overwrites** — after Woman, the cell is back to a **single** codepoint `U+1F469`: the new width-2 base write cleared the old base *and its combining slot* (§4.4). The Man+ZWJ is gone.
- this base→ZWJ→overwrite cycle repeats for Girl, then Boy.
- **settled** — after the final base (Boy, `U+1F466`) with **no** trailing ZWJ, the cell keeps exactly `['U+1F466']`.

So the buffer's retention rule under the 1×1 constraint is simply: *keep the last width-2 base written into the cell, plus any combining marks (up to three) that trail it before the next base overwrites the cell.*


---

## Section 5 — Part 2: Settled cell content → OBJ-2

**Direct answer:** once the family ZWJ stream settles into the 1×1 cell, the terminal believes exactly **one** codepoint is present: **`U+1F466`** (`👦`, Boy). Length is 1.

### 5.1 How the settled content is read (canonical text accessors)

The cell was read through the same text accessors any consumer uses:

- `str(line)` → `__str__` → `line_as_unicode` — `kitty/line.c:L443-L444`.
- `line.as_ansi()` → `as_ansi` → `line_as_ansi` — `kitty/line.c:L416-L420`.

Both ultimately serialize each cell with `cell_as_unicode` — `kitty/line.c:L199-L207` — which emits the base `ch` followed by up to three combining marks:

```c
size_t
cell_as_unicode(CPUCell *cell, bool include_cc, Py_UCS4 *buf, char_type zero_char) {
    size_t n = 1;
    buf[0] = cell->ch ? cell->ch : zero_char;
    if (include_cc) {
        for (unsigned i = 0; i < arraysz(cell->cc_idx) && cell->cc_idx[i]; i++) buf[n++] = codepoint_for_mark(cell->cc_idx[i]);
    }
    return n;
}
```

(The UTF-8 equivalent is `cell_as_utf8` — `kitty/line.c:L221-L232`.) The loop `for (... i < arraysz(cell->cc_idx) && cell->cc_idx[i]; ...)` is the exact reason the serialized form is "base then whatever combining marks are non-empty" — it stops at the first empty slot.

### 5.2 OBSERVED: the settled cell

Command:

```text
s, c = new_screen(1, 1)                       # Screen(cb, 1, 1, 0, 10, 20, 0, cb)
parse_bytes(s, FAMILY.encode('utf-8'))        # whole stream at once, real VT parser
```

Complete, unedited output:

```text
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [after whole family stream (settled)]
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
```

The settled cell is `'👦'`, its codepoint list is `['U+1F466']`, `len == 1`, and `as_ansi()` agrees. This is identical to the end state of the incremental trace in §4.5, confirming that feeding the whole stream at once and feeding it codepoint-by-codepoint produce the same settled content — the retention rule is order-independent for this sequence.


---

## Section 6 — Part 3: State reporting → OBJ-3

**Direct answer:** on the settled 1×1 family cell, a Cursor-Position-Report request `CSI 6 n` replies with the exact bytes **`b'\x1b[1;2R'`** — row 1, column 2. The reported column is **2**, not 1, because the width-2 base advanced the cursor to `x=2`, and `report_device_status` clamps that out-of-range column back so the report reflects the last real column the grapheme consumed.

### 6.1 The DSR/CPR code and the clamp

`CSI 6 n` (Device Status Report — cursor position) is handled by `report_device_status` — `kitty/screen.c:L2179-L2199`:

```c
report_device_status(Screen *self, unsigned int which, bool private) {
    // We don't implement the private device status codes, since I haven't come
    // across any programs that use them
    unsigned int x, y;
    static char buf[64];
    switch(which) {
        case 5:  // device status
            write_escape_code_to_child(self, ESC_CSI, "0n");
            break;
        case 6:  // cursor position
            x = self->cursor->x; y = self->cursor->y;
            if (x >= self->columns) {
                if (y < self->lines - 1) { x = 0; y++; }
                else x--;
            }
            if (self->modes.mDECOM) y -= MAX(y, self->margin_top);
            // 1-based indexing
            int sz = snprintf(buf, sizeof(buf) - 1, "%s%u;%uR", (private ? "?": ""), y + 1, x + 1);
            if (sz > 0) write_escape_code_to_child(self, ESC_CSI, buf);
            break;
    }
```

**Applying it to the settled 1×1 family cell** (`x=2, y=0, columns=1, lines=1`):

- `x >= self->columns` → `2 >= 1` → **true**, so the clamp runs.
- `y < self->lines - 1` → `0 < 0` → **false**, so the `else` branch runs: `x--` → `x = 1`.
- `mDECOM` is off, so `y` is unchanged.
- 1-based output: `y + 1 = 1`, `x + 1 = 2` → the formatted body is `"1;2R"`.

`write_escape_code_to_child` prepends the CSI introducer `ESC [`, giving the full reply **`b'\x1b[1;2R'`**.

**How this reflects the earlier grapheme handling:** the report carries no notion of "family emoji" or "grapheme cluster." All it encodes is the **cursor column**, which is a consequence of the width classification from Part 1 — a single width-2 base was placed and moved the cursor to column 2, then the one-column geometry clamped it back to the last real column. The report is thus a faithful echo of *how many cells the surviving grapheme consumed under the constraint*, not of the codepoints that streamed through.

### 6.2 OBSERVED: `CSI 6 n` on the settled 1×1 cell

```text
   [after whole family stream (settled)]
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      CSI 6 n  cmd=parse_bytes(s, b'\x1b[6n')  -> reply = b'\x1b[1;2R'
```

### 6.3 OBSERVED: the wide-grid contrast (20×1)

Fed into a **20×1** grid, the very same byte stream is **not** collapsed: there is room, so the whole cluster survives (all 7 codepoints), the cursor lands at `x=8` (four width-2 bases = 8 columns), and `CSI 6 n` reports column 9:

```text
   [20x1 after whole family stream]
      str(line0)   = '👨\u200d👩\u200d👧\u200d👦'
      codepoints    = ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466']  (len=7)
      as_ansi()     = '👨\u200d👩\u200d👧\u200d👦'
      cursor        = (x=8, y=0)
      CSI 6 n reply = b'\x1b[1;9R'
```

Here `x=8 < columns=20`, so the clamp does **not** fire; the report is `y+1=1`, `x+1=9` → **`b'\x1b[1;9R'`**. Contrast is the point: the identical input yields **`b'\x1b[1;2R'`** in 1×1 (collapsed) versus **`b'\x1b[1;9R'`** in 20×1 (whole cluster retained). The difference in the state report is a direct readout of the difference in retention forced by geometry.

### 6.4 OBSERVED: related state reports on the settled 1×1 cell

Each of the following was issued through the real parser against the settled 1×1 family cell, and the reply captured from `wtcbuf`. Complete, unedited output:

```text
   settled cell = '👦'  cursor=(x=2,y=0)
   DSR-5  CSI 5 n     parse_bytes(s,b'\x1b[5n')  -> b'\x1b[0n'
   DA     CSI c       parse_bytes(s,b'\x1b[c')   -> b'\x1b[?62;c'
   DA>    CSI > c     parse_bytes(s,b'\x1b[>c')  -> b'\x1b[>1;4000;35c'
   size14 CSI 14 t    -> b'\x1b[4;20;10t'
   size16 CSI 16 t    -> b'\x1b[6;20;10t'
   size18 CSI 18 t    -> b'\x1b[8;1;1t'
   DECRPM CSI ?25 $p  -> b'\x1b[?25;1$y'
   DECRQSS DECSCUSR   parse_bytes(s, ESC P $ q SP q ESC \) -> b'\x1bP1$r1 q\x1b\\'
```

With mechanism and citation for each:

- **DSR-5** `CSI 5 n` → **`b'\x1b[0n'`** — the `case 5` branch writes the literal `"0n"` ("device OK") at `kitty/screen.c:L2185-L2186`. It is independent of the cell content.
- **Primary Device Attributes** `CSI c` → **`b'\x1b[?62;c'`** — `report_device_attributes` writes `"?62;c"` for the no-modifier case at `kitty/screen.c:L2121-L2126` (VT220-class identification).
- **Secondary DA** `CSI > c` → **`b'\x1b[>1;4000;35c'`** — same function, `'>'` case, format `">1;" PRIMARY_VERSION ";" SECONDARY_VERSION "c"`. The version numbers are compile-time defines (`setup.py:L730`; this build's `compile_commands.json` shows `PRIMARY_VERSION=4000`, `SECONDARY_VERSION=35`), so **these two numbers are version-dependent** — observed here as `4000;35`.
- **Size reports** `screen_report_size` — `kitty/screen.c:L2142-L2168` — format `"%u;%u;%ut"` = `code;height;width`:
  - `CSI 14 t` → **`b'\x1b[4;20;10t'`** — text area in pixels: `code=4`, `height = cell_height × lines = 20 × 1 = 20`, `width = cell_width × cols = 10 × 1 = 10`.
  - `CSI 16 t` → **`b'\x1b[6;20;10t'`** — single cell in pixels: `code=6`, `20 × 10`.
  - `CSI 18 t` → **`b'\x1b[8;1;1t'`** — text area in characters: `code=8`, `height = lines = 1`, `width = cols = 1`.
- **Mode status / DECRPM** `CSI ?25 $p` → **`b'\x1b[?25;1$y'`** — `report_mode_status` at `kitty/screen.c:L2203` reports mode 25 (cursor visibility) with value `1` (visible).
- **DECRQSS** (DCS `$ q` … ST) querying DECSCUSR cursor shape → **`b'\x1bP1$r1 q\x1b\\'`** — the `'$'` case at `kitty/screen.c:L2453-L2454` recognizes the `" q"` query and returns the current cursor shape. *(In the captured line the descriptive label prints a single backslash for the ST terminator; the reply payload itself ends in `ESC \` = `\x1b\\`, shown literally.)*

None of DSR-5, DA, DA>, the size reports, DECRPM, or DECRQSS depend on the cell's Unicode content — only `CSI 6 n` (§6.1–6.3) reflects the grapheme handling, via the cursor column.


---

## Section 7 — Part 4: How normalization, grapheme breaking, and state reporting interact → OBJ-4

**Direct answer:** they interact only **indirectly**, because kitty implements the first two as a **single width + combining-membership classification** rather than as separate passes, and the third (state reporting) merely reads out the cursor position that classification produced. Specifically: **normalization is absent**, there is **no grapheme-segmentation state machine**, and the `CSI 6 n` report reflects the outcome only through the cursor column.

### 7.1 Normalization: ABSENT

There is **no Unicode normalization pass** anywhere on this input path. No NFC/NFD/NFKC/NFKD transformation is applied to the incoming codepoints before they are placed or appended; the bytes decoded by the VT parser are classified and stored as-is. This is a deliberate finding, corroborated by the retention observations: had any normalization run, the combining marks in the overflow case (§7.6) would have been reordered/precomposed, but the observed codepoint lists preserve the raw input order and identity.

### 7.2 Grapheme breaking: NO UAX #29 state machine

kitty does **not** run a Unicode Text-Segmentation (UAX #29) extended-grapheme-cluster segmenter on this path. Instead each codepoint is classified independently by two properties:

- **width** — `wcwidth_std` (`kitty/wcswidth.c:L62`), backed by the table in `kitty/wcwidth-std.h`; and
- **combining membership** — `is_combining_char` (`kitty/unicode-data.c:L11`).

A non-combining codepoint with width > 0 **occupies (and thus overwrites) a cell**; a combining codepoint is **appended** to the current base (§4.2–4.3). "Grapheme handling" in kitty is therefore an *emergent* consequence of these per-codepoint decisions, not the output of a cluster-boundary algorithm. This is exactly why a ZWJ family sequence does **not** stay together in a 1×1 cell: nothing is tracking the cluster; each base simply overwrites the last.

### 7.3 Provenance: the classification tables are generated (Unicode 15.0.0)

The classification tables are **generated**, not hand-written. The generator is `gen_ucd()` — `gen/wcwidth.py:L403` — which emits `kitty/unicode-data.c` (via `create_header('kitty/unicode-data.c')` at `gen/wcwidth.py:L405`). The emitted file's first line records the standard version — `kitty/unicode-data.c:L1`: `// Unicode data, built from the Unicode Standard 15.0.0`. The same version is cross-confirmed in the Go width table (`tools/wcswidth/std.go`, `UnicodeDatabaseVersion = {15, 0, 0}`). There is no runtime normalization layer to generate — only membership/width tables.

### 7.4 Subtle classification facts that make the ZWJ behavior work

These three facts (each grounded in code) are what route a ZWJ into a combining slot rather than dropping it or giving it a cell:

- **`is_combining_char(0x200D)` is TRUE.** ZWJ falls inside `case 0x200b ... 0x200f:` at `kitty/unicode-data.c:L323`, which is a case within `is_combining_char` (`L11`). Hence ZWJ takes the combining branch of the draw loop (§4.3) and is appended, not allocated a new cell. *(Observed indirectly: after a ZWJ the cell length grows by one combining mark on the same base rather than producing a second cell — §4.5.)*
- **`is_ignored_char(0x200D)` is FALSE.** `is_ignored_char` (`kitty/unicode-data.c:L671`, "Control characters and non-characters") does **not** contain the `0x200b..0x200f` range, so the draw-loop gate `if (is_ignored_char(ch)) continue;` (`screen.c:L805`) does **not** drop ZWJ. The identical-looking `case 0x200b ... 0x200f:` at `kitty/unicode-data.c:L753` belongs to a **different** function, `is_non_rendered_char` (`L723`), which is **not** the draw-loop gate. The distinction matters: it is why ZWJ genuinely reaches `draw_combining_char`. *(Observed indirectly via §4.5: the ZWJ demonstrably survives as a combining mark on the base — it is neither dropped nor given its own cell.)*
- **VS15/VS16 are combining.** The variation selectors fall inside `case 0xfe00 ... 0xfe0f:` at `kitty/unicode-data.c:L407` (within `is_combining_char`), so they too are appended to the preceding base — which is what enables the width-flip logic in §7.5.

### 7.5 Variation-selector width flips (and how they change the state report)

Because the variation selectors are combining, they reach `draw_combining_char` (`kitty/screen.c:L663-L710`), which contains two special branches that **mutate the base cell's effective width** — and therefore the cursor column that `CSI 6 n` later echoes:

- **VS16 `U+FE0F` (emoji presentation) widens** a default text-presentation emoji to width 2 (branch `if (ch == 0xfe0f)`, `kitty/screen.c:L679-L689`): it sets `gpu_cell->attrs.width = 2`; if there is a spare column (`xpos + 1 < self->columns`) it zeroes the next cell and does `cursor->x++`, **otherwise** it calls `move_widened_char` (defined at `kitty/screen.c:L575`) and the cursor is **not** incremented.
- **VS15 `U+FE0E` (text presentation) narrows** to width 1 (branch `else if (ch == 0xfe0e)`, `kitty/screen.c:L690-L700`): it sets `attrs.width = 1` and does `cursor->x--`.

OBSERVED — base `U+2764` (❤, a default text-presentation emoji) followed by VS16, into 1×1 and 5×1:

```text
   [1x1 after VS16 (bytes b'\xe2\x9d\xa4\xef\xb8\x8f')]
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 after VS16 (bytes b'\xe2\x9d\xa4\xef\xb8\x8f')]
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'
```

In **1×1** there is no spare column, so `move_widened_char` runs and the cursor stays at `x=1` → `CSI 6 n` = **`b'\x1b[1;1R'`** (no clamp needed since `1 < 1` is false but `x >= columns` is `1 >= 1` → true, and `else x--` gives `x=0` → column 1). In **5×1** the widen finds room and does `cursor->x++`, landing at `x=2` → **`b'\x1b[1;3R'`**. The VS16 width flip literally changes the reported column.

OBSERVED — base `U+2764` followed by VS15, into 1×1 and 5×1:

```text
   [1x1 after VS15 (bytes b'\xe2\x9d\xa4\xef\xb8\x8e')]
      str(line0)   = '❤︎'
      codepoints    = ['U+2764', 'U+FE0E']  (len=2)
      as_ansi()     = '❤︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 after VS15 (bytes b'\xe2\x9d\xa4\xef\xb8\x8e')]
      str(line0)   = '❤︎'
      codepoints    = ['U+2764', 'U+FE0E']  (len=2)
      as_ansi()     = '❤︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
```

VS15 keeps/forces width 1: in 5×1 the cursor is at `x=1` → **`b'\x1b[1;2R'`**; in 1×1 the cursor is `x=1` and the clamp gives column 1 → **`b'\x1b[1;1R'`**.

### 7.6 The `cc_idx` overflow rule (more than three combining marks)

To exercise the overflow branch of `line_add_combining_char` (§4.2), a base `A` (`U+0041`) was followed by **four** combining marks `U+0300 U+0301 U+0302 U+0303` in a 5×1 grid. OBSERVED (rendering the codepoint list as authoritative, since stacked combining marks may not display cleanly):

```text
   [base A]
      codepoints    = ['U+0041']  (len=1)
   [+U+0300 (slot0)]
      codepoints    = ['U+0041', 'U+0300']  (len=2)
   [+U+0301 (slot1)]
      codepoints    = ['U+0041', 'U+0300', 'U+0301']  (len=3)
   [+U+0302 (slot2)]
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0302']  (len=4)
   [+U+0303 (OVERFLOW->last slot)]
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0303']  (len=4)
```

The first three marks fill `cc_idx[0..2]`. The **fourth** mark `U+0303` does not extend the list — it **overwrites the last slot**, so `U+0302` (which had been in `cc_idx[2]`) is replaced while `U+0300` and `U+0301` remain. The retained set is `['U+0041','U+0300','U+0301','U+0303']`, exactly as `cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);` (`kitty/line.c:L466`) dictates. This is the same fixed-capacity retention rule as Part 1, now visible on a single base. *(The cursor stays at `x=1` throughout — combining marks never advance it.)*

### 7.7 Contrast conditions: flag pairs and skin-tone modifiers are retained even in 1×1

Not every "multi-codepoint" sequence collapses the way the ZWJ family does. Two important contrasts, both OBSERVED, show that when the second codepoint is *combining/appendable*, **both** codepoints survive even in a 1×1 cell:

**Regional-indicator flag pair (US) `U+1F1FA U+1F1F8`.** The second regional indicator is appended to the first via `draw_second_flag_codepoint` (`kitty/screen.c:L633-L661`, which calls `line_add_combining_char`); `is_flag_codepoint` is the range `0x1F1E6..0x1F1FF` at `kitty/unicode-data.h:L82`. OBSERVED:

```text
   [1x1 after flag pair (bytes b'\xf0\x9f\x87\xba\xf0\x9f\x87\xb8')]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 after flag pair (bytes b'\xf0\x9f\x87\xba\xf0\x9f\x87\xb8')]
      ...
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'
```

**Skin-tone modifier `U+1F44B U+1F3FF`** (waving hand + dark skin tone). The modifier is combining and is appended to the wave base. OBSERVED:

```text
   [1x1 after wave+dark (bytes b'\xf0\x9f\x91\x8b\xf0\x9f\x8f\xbf')]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 after wave+dark (bytes b'\xf0\x9f\x91\x8b\xf0\x9f\x8f\xbf')]
      ...
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'
```

In both cases the 1×1 cell keeps **two** codepoints (`len=2`) — unlike the ZWJ family, where the cell keeps **one** (`len=1`). The distinction is entirely explained by classification: for the flag pair and skin-tone cases the second codepoint is *appended* to the first base (so both survive), whereas in the family sequence each interior element after a ZWJ is a **new width-2 base** that *overwrites* the single cell. The `CSI 6 n` column (2 in 1×1, 3 in 5×1) again just echoes the width-2 base's cursor advance under the geometry.

### 7.8 Putting the three mechanisms together

- **Normalization** never runs, so nothing recomposes or reorders the stream — the raw codepoints are what get classified.
- **Grapheme breaking** is not a segmentation pass but the emergent result of per-codepoint width + combining decisions; under a 1×1 constraint this means "last base wins," with combining marks (ZWJ included) merely riding along on whatever base is current until the next base overwrites the cell.
- **State reporting** does not observe graphemes at all; `CSI 6 n` reports the **cursor column**, which is the only place the earlier handling leaves a visible trace — and only because width classification moved the cursor. The clamp in `report_device_status` then maps the out-of-range cursor back onto the last real column.


---

## Section 8 — Observed vs. inferred

To keep the evidentiary boundary explicit:

**OBSERVED (real captured output from the built engine, byte-identical across 2 runs):**

- Every `str(line)`, codepoint list, `as_ansi()`, and cursor value in Sections 4–7.
- The settled 1×1 family cell = `'👦'` / `['U+1F466']` / `len=1`.
- All control-sequence replies shown as literal `bytes`: `CSI 6 n` = `b'\x1b[1;2R'` (1×1) and `b'\x1b[1;9R'` (20×1); DSR-5 `b'\x1b[0n'`; DA `b'\x1b[?62;c'`; DA> `b'\x1b[>1;4000;35c'`; size `b'\x1b[4;20;10t'`, `b'\x1b[6;20;10t'`, `b'\x1b[8;1;1t'`; DECRPM `b'\x1b[?25;1$y'`; DECRQSS `b'\x1bP1$r1 q\x1b\\'`.
- The VS16/VS15 cursor differences between 1×1 and 5×1; the flag-pair and skin-tone 1×1 retention (`len=2`); the `cc_idx` overflow codepoint lists.
- The build result in this environment (canonical `python3 setup.py build` → exit 0, Wayland disabled, 85 objects, 4 links) and the successful import.

**OBSERVED but version-dependent:**

- The Secondary-DA numbers `4000;35` in `b'\x1b[>1;4000;35c'` are compile-time version constants (`PRIMARY_VERSION`/`SECONDARY_VERSION`, `setup.py:L730`); a different kitty version would report different numbers. The *shape* of the reply is stable.

**INFERRED from reading the code (not directly visible as a distinct line in the output, but entailed by it + the source):**

- That the VS16 widen in the **1×1** case takes the `move_widened_char` path rather than `cursor->x++` is inferred from `kitty/screen.c:L679-L689` together with the observed fact that the cursor stayed at `x=1` (had the `cursor->x++` branch run, `x` would have become 2).
- That a new width-2 base write "clears the previous `ch` and `cc_idx`" is inferred from the cell-write semantics of `screen_draw_text` combined with the observed collapse from `len=2` (base+ZWJ) back to `len=1` (new base only) in §4.5.
- The exact internal slot that each mark occupies (`cc_idx[0]`, `[1]`, `[2]`) is inferred from `line_add_combining_char` (`kitty/line.c:L456-L467`) plus the observed codepoint-list growth and the overflow overwrite; the public accessors expose the resulting *list*, not the raw slot indices.

**NON-CANONICAL (explicitly avoided in the answer):** any value from `s.draw(<str>)` (direct draw API), remote control, or a debug hook. All observed values above came through `parse_bytes` → the real VT parser. If one wished to check `is_combining_char`/`is_ignored_char` by calling the classifier bindings directly, that would be a non-canonical direct-API check; this document instead demonstrates their effect **indirectly** through the observed cell contents.

---

## Section 9 — Coverage-pass checklist

Every named item the question implies, addressed by name:

| Item | Where addressed | Key observed evidence |
|------|-----------------|-----------------------|
| **Normalization** (interaction) | §7.1, §7.8 | ABSENT — raw codepoints preserved (overflow list unreordered) |
| **Grapheme breaking** (interaction) | §7.2, §7.8 | NO UAX #29 state machine; width + combining membership |
| **State reporting** (interaction) | §6, §7.8 | `CSI 6 n` → `b'\x1b[1;2R'` (1×1) |
| **1×1 constraint** | §4, §5, §6.1–6.2 | settled cell `len=1`; cursor `x=2`; clamp fires |
| **Primary family ZWJ emoji** | §4.5, §5.2 | incremental trace; settled `'👦'` `['U+1F466']` |
| **VS15 (narrow)** | §7.5 | 1×1 `b'\x1b[1;1R'`; 5×1 `b'\x1b[1;2R'` |
| **VS16 (widen)** | §7.5 | 1×1 `b'\x1b[1;1R'` (move_widened_char); 5×1 `b'\x1b[1;3R'` |
| **Regional-indicator flag pair** | §7.7 | 1×1 `'🇺🇸'` `len=2`, `b'\x1b[1;2R'` |
| **Skin-tone modifier** | §7.7 | 1×1 `'👋🏿'` `len=2`, `b'\x1b[1;2R'` |
| **>3-mark `cc_idx` overflow** | §7.6 | 4th mark overwrites last slot → `['U+0041','U+0300','U+0301','U+0303']` |
| **Wider-grid contrast** | §6.3, §7.5, §7.7 | 20×1 whole cluster survives, `b'\x1b[1;9R']`; 5×1 columns |
| **DSR/CPR `CSI 6 n`** | §6.1–6.3 | `b'\x1b[1;2R'` / `b'\x1b[1;9R'` |
| **DSR-5 `CSI 5 n`** | §6.4 | `b'\x1b[0n'` |
| **Device Attributes** | §6.4 | DA `b'\x1b[?62;c'`; DA> `b'\x1b[>1;4000;35c'` |
| **Size reports** | §6.4 | `b'\x1b[4;20;10t'`, `b'\x1b[6;20;10t'`, `b'\x1b[8;1;1t'` |
| **DECRPM (mode status)** | §6.4 | `b'\x1b[?25;1$y'` |
| **DECRQSS** | §6.4 | `b'\x1bP1$r1 q\x1b\\'` |
| **Canonical entry point** | §3 | all input via `parse_bytes` → real VT parser |
| **Reproducibility / stability** | §2.5 | 2/2 byte-identical runs (identical MD5) |
| **Repository integrity / cleanup** | §2.5, §10 | build artifacts gitignored; temp script outside repo, removed; `git status` clean |


---

## Section 10 — Appendix: the temporary observation script and its complete output

### 10.1 The observation script (verbatim)

This script lived **outside** the repository tree at `/tmp/kitty_obs/observe.py` and was **removed** after the observations were captured. It drives the **real VT parser** (canonical entry point) via `kitty_tests.parse_bytes`, and captures control-sequence replies from `Callbacks.write` → `self.wtcbuf`. It was executed from the repository root as `PYTHONPATH=. python3 /tmp/kitty_obs/observe.py`.

```python
# TEMPORARY observation script (lives OUTSIDE the kitty repo tree at /tmp/kitty_obs).
# Drives the REAL VT parser (canonical entry point) via kitty_tests.parse_bytes:
#   screen.test_create_write_buffer() -> test_commit_write_buffer(data,dest) -> test_parse_written_data()
# Captures control-sequence replies from Callbacks.write -> self.wtcbuf.
import sys
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes  # canonical byte-parser template

def new_screen(lines, cols):
    c = Callbacks()
    # Mirror kitty_tests.create_screen: Screen(callbacks, LINES, COLS, scrollback, cell_w, cell_h, 0, callbacks)
    s = Screen(c, lines, cols, 0, 10, 20, 0, c)
    return s, c

def cps(text):
    return [f'U+{ord(ch):04X}' for ch in text]

def show(s, label, y=0):
    line = s.line(y)
    text = str(line)
    print(f'   [{label}]')
    print(f'      str(line{y})   = {text!r}')
    print(f'      codepoints    = {cps(text)}  (len={len(text)})')
    print(f'      as_ansi()     = {line.as_ansi()!r}')
    print(f'      cursor        = (x={s.cursor.x}, y={s.cursor.y})')

def query(s, c, q):
    c.wtcbuf = b''                 # reset reply capture
    parse_bytes(s, q)              # drive query through REAL parser
    return bytes(c.wtcbuf)

def hdr(t):
    print('\n' + '='*78 + f'\n{t}\n' + '='*78)

# ---------------------------------------------------------------------------
FAMILY = '\U0001F468\u200D\U0001F469\u200D\U0001F467\u200D\U0001F466'
print(f'FAMILY emoji codepoints = {cps(FAMILY)} (Python len={len(FAMILY)})')
print(f'FAMILY UTF-8 bytes      = {FAMILY.encode("utf-8")!r}')

hdr('PRIMARY: family ZWJ emoji -> 1x1 Screen (INCREMENTAL: before/during/after)')
s, c = new_screen(1, 1)
show(s, 'before (empty screen)')
chunks = ['\U0001F468', '\u200D', '\U0001F469', '\u200D', '\U0001F467', '\u200D', '\U0001F466']
labels = ['after Man U+1F468 (width2 base)', 'after ZWJ U+200D', 'after Woman U+1F469 (new width2 base)',
          'after ZWJ U+200D', 'after Girl U+1F467 (new width2 base)', 'after ZWJ U+200D',
          'after Boy U+1F466 (final width2 base)']
for chunk, label in zip(chunks, labels):
    parse_bytes(s, chunk.encode('utf-8'))
    show(s, label + f'  <-- fed bytes {chunk.encode("utf-8")!r}')

hdr('PRIMARY (canonical single-stream): whole FAMILY at once -> 1x1, then CSI 6n')
s, c = new_screen(1, 1)
show(s, 'before')
parse_bytes(s, FAMILY.encode('utf-8'))
show(s, 'after whole family stream (settled)')
r6 = query(s, c, b'\x1b[6n')
print(f'      CSI 6 n  cmd=parse_bytes(s, {b"\x1b[6n"!r})  -> reply = {r6!r}')

hdr('SECONDARY: VS16 widen  U+2764 U+FE0F  into 1x1 and 5x1')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    b = '\u2764\uFE0F'.encode('utf-8')
    show(s, f'{cols}x1 before')
    parse_bytes(s, b)
    show(s, f'{cols}x1 after VS16 (bytes {b!r})')
    print(f'      CSI 6 n reply = {query(s,c,chr(0x1b).encode()+b"[6n")!r}')

hdr('SECONDARY: VS15 narrow  U+2764 U+FE0E  into 1x1 and 5x1')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    b = '\u2764\uFE0E'.encode('utf-8')
    parse_bytes(s, b)
    show(s, f'{cols}x1 after VS15 (bytes {b!r})')
    print(f'      CSI 6 n reply = {query(s,c,b"\x1b[6n")!r}')

hdr('SECONDARY: regional-indicator flag pair (US)  U+1F1FA U+1F1F8  into 1x1 and 5x1')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    b = '\U0001F1FA\U0001F1F8'.encode('utf-8')
    parse_bytes(s, b)
    show(s, f'{cols}x1 after flag pair (bytes {b!r})')
    print(f'      CSI 6 n reply = {query(s,c,b"\x1b[6n")!r}')

hdr('SECONDARY: skin-tone modifier  U+1F44B U+1F3FF  into 1x1 and 5x1')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    b = '\U0001F44B\U0001F3FF'.encode('utf-8')
    parse_bytes(s, b)
    show(s, f'{cols}x1 after wave+dark (bytes {b!r})')
    print(f'      CSI 6 n reply = {query(s,c,b"\x1b[6n")!r}')

hdr('SECONDARY: cc_idx OVERFLOW  A + U+0300 U+0301 U+0302 U+0303  (5x1, >3 marks)')
s, c = new_screen(1, 5)
show(s, 'before')
seq = ['A', '\u0300', '\u0301', '\u0302', '\u0303']
lbl = ['base A', '+U+0300 (slot0)', '+U+0301 (slot1)', '+U+0302 (slot2)', '+U+0303 (OVERFLOW->last slot)']
for ch, l in zip(seq, lbl):
    parse_bytes(s, ch.encode('utf-8'))
    show(s, l)

hdr('CONTRAST: whole FAMILY into WIDE grid 20x1 (compare to 1x1 collapse), then CSI 6n')
s, c = new_screen(1, 20)
parse_bytes(s, FAMILY.encode('utf-8'))
show(s, '20x1 after whole family stream')
print(f'      CSI 6 n reply = {query(s,c,b"\x1b[6n")!r}')

hdr('RELATED REPORTS on the settled 1x1 family cell')
s, c = new_screen(1, 1)
parse_bytes(s, FAMILY.encode('utf-8'))
print(f'   settled cell = {str(s.line(0))!r}  cursor=(x={s.cursor.x},y={s.cursor.y})')
print(f'   DSR-5  CSI 5 n     parse_bytes(s,{b"\x1b[5n"!r})  -> {query(s,c,b"\x1b[5n")!r}')
print(f'   DA     CSI c       parse_bytes(s,{b"\x1b[c"!r})   -> {query(s,c,b"\x1b[c")!r}')
print(f'   DA>    CSI > c     parse_bytes(s,{b"\x1b[>c"!r})  -> {query(s,c,b"\x1b[>c")!r}')
print(f'   size14 CSI 14 t    -> {query(s,c,b"\x1b[14t")!r}')
print(f'   size16 CSI 16 t    -> {query(s,c,b"\x1b[16t")!r}')
print(f'   size18 CSI 18 t    -> {query(s,c,b"\x1b[18t")!r}')
print(f'   DECRPM CSI ?25 $p  -> {query(s,c,b"\x1b[?25$p")!r}')
print(f'   DECRQSS DECSCUSR   parse_bytes(s, ESC P $ q SP q ESC \\) -> {query(s,c,b"\x1bP$q q\x1b\\")!r}')

print('\n[OBSERVE-DONE]')
```

### 10.2 Complete, unedited captured output

The following is the full output of one run; the second identical run produced byte-identical output (same MD5), confirming stability (§2.5).

```text
FAMILY emoji codepoints = ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466'] (Python len=7)
FAMILY UTF-8 bytes      = b'\xf0\x9f\x91\xa8\xe2\x80\x8d\xf0\x9f\x91\xa9\xe2\x80\x8d\xf0\x9f\x91\xa7\xe2\x80\x8d\xf0\x9f\x91\xa6'

==============================================================================
PRIMARY: family ZWJ emoji -> 1x1 Screen (INCREMENTAL: before/during/after)
==============================================================================
   [before (empty screen)]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [after Man U+1F468 (width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa8']
      str(line0)   = '👨'
      codepoints    = ['U+1F468']  (len=1)
      as_ansi()     = '👨'
      cursor        = (x=2, y=0)
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👨\u200d'
      codepoints    = ['U+1F468', 'U+200D']  (len=2)
      as_ansi()     = '👨\u200d'
      cursor        = (x=2, y=0)
   [after Woman U+1F469 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa9']
      str(line0)   = '👩'
      codepoints    = ['U+1F469']  (len=1)
      as_ansi()     = '👩'
      cursor        = (x=2, y=0)
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👩\u200d'
      codepoints    = ['U+1F469', 'U+200D']  (len=2)
      as_ansi()     = '👩\u200d'
      cursor        = (x=2, y=0)
   [after Girl U+1F467 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa7']
      str(line0)   = '👧'
      codepoints    = ['U+1F467']  (len=1)
      as_ansi()     = '👧'
      cursor        = (x=2, y=0)
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👧\u200d'
      codepoints    = ['U+1F467', 'U+200D']  (len=2)
      as_ansi()     = '👧\u200d'
      cursor        = (x=2, y=0)
   [after Boy U+1F466 (final width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa6']
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)

==============================================================================
PRIMARY (canonical single-stream): whole FAMILY at once -> 1x1, then CSI 6n
==============================================================================
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [after whole family stream (settled)]
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      CSI 6 n  cmd=parse_bytes(s, b'\x1b[6n')  -> reply = b'\x1b[1;2R'

==============================================================================
SECONDARY: VS16 widen  U+2764 U+FE0F  into 1x1 and 5x1
==============================================================================
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [1x1 after VS16 (bytes b'\xe2\x9d\xa4\xef\xb8\x8f')]
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [5x1 after VS16 (bytes b'\xe2\x9d\xa4\xef\xb8\x8f')]
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'

==============================================================================
SECONDARY: VS15 narrow  U+2764 U+FE0E  into 1x1 and 5x1
==============================================================================
   [1x1 after VS15 (bytes b'\xe2\x9d\xa4\xef\xb8\x8e')]
      str(line0)   = '❤︎'
      codepoints    = ['U+2764', 'U+FE0E']  (len=2)
      as_ansi()     = '❤︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 after VS15 (bytes b'\xe2\x9d\xa4\xef\xb8\x8e')]
      str(line0)   = '❤︎'
      codepoints    = ['U+2764', 'U+FE0E']  (len=2)
      as_ansi()     = '❤︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;2R'

==============================================================================
SECONDARY: regional-indicator flag pair (US)  U+1F1FA U+1F1F8  into 1x1 and 5x1
==============================================================================
   [1x1 after flag pair (bytes b'\xf0\x9f\x87\xba\xf0\x9f\x87\xb8')]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 after flag pair (bytes b'\xf0\x9f\x87\xba\xf0\x9f\x87\xb8')]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'

==============================================================================
SECONDARY: skin-tone modifier  U+1F44B U+1F3FF  into 1x1 and 5x1
==============================================================================
   [1x1 after wave+dark (bytes b'\xf0\x9f\x91\x8b\xf0\x9f\x8f\xbf')]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 after wave+dark (bytes b'\xf0\x9f\x91\x8b\xf0\x9f\x8f\xbf')]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'

==============================================================================
SECONDARY: cc_idx OVERFLOW  A + U+0300 U+0301 U+0302 U+0303  (5x1, >3 marks)
==============================================================================
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [base A]
      str(line0)   = 'A'
      codepoints    = ['U+0041']  (len=1)
      as_ansi()     = 'A'
      cursor        = (x=1, y=0)
   [+U+0300 (slot0)]
      str(line0)   = 'À'
      codepoints    = ['U+0041', 'U+0300']  (len=2)
      as_ansi()     = 'À'
      cursor        = (x=1, y=0)
   [+U+0301 (slot1)]
      str(line0)   = 'À́'
      codepoints    = ['U+0041', 'U+0300', 'U+0301']  (len=3)
      as_ansi()     = 'À́'
      cursor        = (x=1, y=0)
   [+U+0302 (slot2)]
      str(line0)   = 'À́̂'
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0302']  (len=4)
      as_ansi()     = 'À́̂'
      cursor        = (x=1, y=0)
   [+U+0303 (OVERFLOW->last slot)]
      str(line0)   = 'À́̃'
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0303']  (len=4)
      as_ansi()     = 'À́̃'
      cursor        = (x=1, y=0)

==============================================================================
CONTRAST: whole FAMILY into WIDE grid 20x1 (compare to 1x1 collapse), then CSI 6n
==============================================================================
   [20x1 after whole family stream]
      str(line0)   = '👨\u200d👩\u200d👧\u200d👦'
      codepoints    = ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466']  (len=7)
      as_ansi()     = '👨\u200d👩\u200d👧\u200d👦'
      cursor        = (x=8, y=0)
      CSI 6 n reply = b'\x1b[1;9R'

==============================================================================
RELATED REPORTS on the settled 1x1 family cell
==============================================================================
   settled cell = '👦'  cursor=(x=2,y=0)
   DSR-5  CSI 5 n     parse_bytes(s,b'\x1b[5n')  -> b'\x1b[0n'
   DA     CSI c       parse_bytes(s,b'\x1b[c')   -> b'\x1b[?62;c'
   DA>    CSI > c     parse_bytes(s,b'\x1b[>c')  -> b'\x1b[>1;4000;35c'
   size14 CSI 14 t    -> b'\x1b[4;20;10t'
   size16 CSI 16 t    -> b'\x1b[6;20;10t'
   size18 CSI 18 t    -> b'\x1b[8;1;1t'
   DECRPM CSI ?25 $p  -> b'\x1b[?25;1$y'
   DECRQSS DECSCUSR   parse_bytes(s, ESC P $ q SP q ESC \) -> b'\x1bP1$r1 q\x1b\\'

[OBSERVE-DONE]
```

> **Rendering note on the overflow block (§7.6):** in the `cc_idx` overflow section the combining marks stack visually on the base `A`, and different fonts/terminals may render the stacked glyphs differently. The authoritative representation is the **codepoint list**, which is stable and shown above: after the fourth mark the retained codepoints are `['U+0041','U+0300','U+0301','U+0303']` — the last slot's `U+0302` was overwritten by `U+0303` while `U+0300` and `U+0301` remain, exactly as `kitty/line.c:L466` dictates.

---

*End of document. All observed values above were produced by the built `kitty.fast_data_types` engine driven through the real VT parser at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`; the repository working tree was left unchanged and the temporary observation script was deleted.*
