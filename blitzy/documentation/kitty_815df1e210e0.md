# How kitty handles a ZWJ-joined multi-codepoint emoji in its screen buffer under extreme space constraints — and how a state query reflects it

> **Subject:** kitty terminal emulator **v0.35.2** — `kitty/constants.py:L25` → `version: Version = Version(0, 35, 2)`
> **Branch / commit:** `kitty_815df1e210e0` @ `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> **Method:** every value below was observed at runtime by driving kitty's **real VT byte-parser** (`kitty_tests.parse_bytes`) against the compiled `kitty/fast_data_types.so` extension. Nothing here is guessed; the one place an environment-external artifact is reused (a `gdb` backtrace) is called out explicitly, and the crash it describes was itself reproduced live.

## TL;DR (the direct answers)

The single idea behind all four answers is that **kitty's screen buffer is width-driven, not grapheme-cluster-driven.** It runs **no Unicode normalization** and **no UAX #29 grapheme-cluster segmentation**; it decides cell boundaries purely from `wcwidth_std` (`kitty/wcswidth.c`), and it stores a Zero-Width Joiner (ZWJ), a variation selector, or a second regional indicator as a *combining mark* inside a fixed **three-slot** cell (`CPUCell.cc_idx[3]`, `kitty/data-types.h:L223-L227`).

- **Q1 — What is kept in a 1×1 cell?** Only the **last** base emoji survives visibly. A width-2 base can never fit in a 1-column screen, so with autowrap on (the default) each base scrolls the single line and is redrawn at column 0; after the family `👨‍👩‍👧‍👦` the visible cell holds just **`👦` / `U+1F466`**, cursor at `x=2`.
- **Q2 — What does the terminal think is in the cell?** In the 1×1 case, exactly one double-width `👦` (`U+1F466`) and nothing else; cursor `x=2, y=0`. On a wide screen the four bases stay in **four separate** width-2 cells (kitty never merges them into one cluster).
- **Q3 — What does a state query report?** A Cursor Position Report (`CSI 6 n`) answers `ESC[1;2R` in the 1×1 case and `ESC[1;9R` on a 20-column screen. The reported **column is a direct readout of how many columns the width-based bookkeeping consumed** — 9, not the 3 a cluster-collapsing terminal would report.
- **Q4 — How do normalization, grapheme breaking, and state reporting interact?** Normalization: **none** (decomposed and precomposed forms are stored differently, byte-for-byte). Grapheme breaking: **width-based**, not UAX #29. State reporting: a **passive mirror** of the resulting cell/cursor geometry, with no independent notion of graphemes.

The six concepts the question names are each addressed below **by name**: the **ZWJ**, the **multi-codepoint emoji**, the **screen-buffer cell**, **grapheme breaking**, **normalization**, and **control-sequence state reporting**.

---

## 1. Methodology — build & harness (reproducible)

All observations were produced from the compiled C core, driven through kitty's genuine VT byte-parser. No mock, no debug shortcut, and specifically **not** `Screen.draw()` (a higher-level Python helper that bypasses the byte parser) — the canonical entry point is `kitty_tests.parse_bytes`, which pushes raw UTF-8 bytes through `kitty/vt-parser.c`.

### 1a. OS build dependencies (apt; environment-only, never committed)

```
build-essential pkg-config libfreetype-dev libharfbuzz-dev libfontconfig1-dev libpng-dev \
liblcms2-dev libxxhash-dev libcanberra-dev libxkbcommon-dev libdbus-1-dev libssl-dev zlib1g-dev \
libgl1-mesa-dev libglvnd-dev libegl1-mesa-dev libwayland-dev wayland-protocols libx11-dev \
libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libxkbcommon-x11-dev libsimde-dev
```

### 1b. Building `kitty/fast_data_types.so`

The canonical full build is:

```
CI=true python3 setup.py build --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag is required only because a newer system `wayland-protocols` adds enum values that trip kitty's default `-Werror` in the **out-of-scope** GLFW Wayland windowing backend (`glfw/wl_window.c`); the kitty C core itself compiles clean and the flag does not change any runtime behavior. Only the `fast_data_types` C extension is needed for headless `Screen` testing — the GLFW backend and the Go "kittens" are irrelevant to the screen-buffer question. When a windowing/Go toolchain is not present, the extension alone can be produced by neutralizing `compile_glfw`/`compile_kittens` and executing the queued compile/link commands via `CompilationDatabase.build_all()` (the context manager's `__exit__` only writes the JSON compile DB; it does not compile), using a small driver kept **outside** the repository tree:

```python
# build_fdt.py — kept OUTSIDE the repo tree; run as: REPO=<repo_root> python3 build_fdt.py
import sys, os
REPO = os.environ['REPO']
sys.argv = ['setup.py', 'build', '--skip-building-kitten']
sys.path.insert(0, REPO)
os.chdir(REPO)
import setup
setup.compile_glfw = lambda *a, **k: None      # neutralize windowing (GLFW/Wayland/X11)
setup.compile_kittens = lambda *a, **k: None    # neutralize Go kitten builds
args = setup.option_parser().parse_args(namespace=setup.Options())
setup.verbose = args.verbose > 0
args.prefix = os.path.abspath(args.prefix)
os.chdir(setup.src_base)
os.makedirs(setup.build_dir, exist_ok=True)
with setup.CompilationDatabase(args.incremental) as cdb:
    args.compilation_database = cdb
    setup.build(args)
    cdb.build_all()   # <-- actually execute queued compile+link commands
print("BUILD_FDT_DONE")
```

The result is `kitty/fast_data_types.so` (~1.2 MB), which imports cleanly for headless `Screen` testing. Build outputs are covered by `.gitignore` (`*.so` at `.gitignore:L1`, `/build/` at `.gitignore:L14`), so they never alter tracked repository state.

### 1c. The real entry point — `kitty_tests.parse_bytes` (`kitty_tests/__init__.py:L30-L36`)

```python
def parse_bytes(screen, data, dump_callback=None):
    data = memoryview(data)
    while data:
        dest = screen.test_create_write_buffer()
        s = screen.test_commit_write_buffer(data, dest)
        data = data[s:]
        screen.test_parse_written_data(dump_callback)
```

### 1d. Harness (observation script, kept outside the repo tree)

`Options` are initialized once via `set_options` before a `Screen` is constructed. Control-sequence replies (DSR/CPR) are captured from the child-write buffer `Callbacks.wtcbuf` — the sink of `Callbacks.write` (`kitty_tests/__init__.py:L50-L51` → `self.wtcbuf += bytes(data)`; initialized `self.wtcbuf = b''` at `kitty_tests/__init__.py:L96`). Cell text is read back with `line[x]` (→ `text_at`/`cell_as_unicode`, `kitty/line.c:L193`/`L200`), cell width with `line.width(x)`, and the cursor via `screen.cursor.x`/`.y`.

```python
import sys, os
REPO = os.environ["REPO"]; sys.path.insert(0, REPO); os.chdir(REPO)
from kitty_tests import Callbacks, parse_bytes
from kitty.fast_data_types import Screen, set_options
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults

def set_opts():
    o = Options(merge_result_dicts(defaults._asdict(),
                {"scrollback_pager_history_size": 1024, "click_interval": 0.5}))
    finalize_keys(o, {}); finalize_mouse_mappings(o, {}); set_options(o)

def make_screen(cols, lines=1, scrollback=0):
    set_opts(); c = Callbacks()
    s = Screen(c, lines, cols, scrollback, 10, 20, 0, c)   # (callbacks, lines, columns, scrollback, cell_w, cell_h, 0, callbacks)
    return s, c

def feed(s, text): parse_bytes(s, text.encode("utf-8"))    # REAL byte parser

def cpr(s, c):                      # Cursor Position Report round-trip
    c.wtcbuf = b""                  # clear child-write buffer
    parse_bytes(s, b"\x1b[6n")      # CSI 6 n
    return c.wtcbuf                 # reply bytes captured from Callbacks.write

FAMILY = "\U0001f468\u200d\U0001f469\u200d\U0001f467\u200d\U0001f466"  # man ZWJ woman ZWJ girl ZWJ boy
```

### 1e. Invocation, stability, and cleanup

Every scenario was run as `REPO=<repo_root> python3 <script>.py`. Each scenario was executed **at least twice and confirmed byte-identical**; the crash in the appendix was confirmed **3/3**. All observation scripts lived **outside** the repository tree and were removed afterward, so the repository ends unchanged except for this single document (`git status --porcelain` shows only this file).

---

## 2. Q1 — Retention under no space: what the buffer keeps in a 1×1 cell

**Direct answer:** when the family emoji `👨‍👩‍👧‍👦` (`U+1F468 ZWJ U+1F469 ZWJ U+1F467 ZWJ U+1F466`) is streamed into a **1-column, 1-line** screen, the buffer ends up keeping **only the last base emoji, `👦` / `U+1F466`**, in the single visible cell, with the cursor at `x=2`. Each preceding *base + ZWJ* group is pushed off the visible line before the next base is drawn.

### 2.1 The mechanism, codepoint by codepoint

Feeding the family **one codepoint at a time** into a 1×1 screen shows every step:

Command: `make_screen(cols=1, lines=1)` then `feed(s, ch)` for each codepoint of `FAMILY`.

```
# family = U+1F468 ZWJ U+1F469 ZWJ U+1F467 ZWJ U+1F466 ; Screen(lines=1, cols=1)
  command: parse_bytes(s, b'\xf0\x9f\x91\xa8')   # man U+1F468
    -> cell0='👨' width(0)=2 [U+1F468]   cursor x=2 y=0
  command: parse_bytes(s, b'\xe2\x80\x8d')   # ZWJ U+200D
    -> cell0='👨\u200d' width(0)=2 [U+1F468 U+200D]   cursor x=2 y=0
  command: parse_bytes(s, b'\xf0\x9f\x91\xa9')   # woman U+1F469
    -> cell0='👩' width(0)=2 [U+1F469]   cursor x=2 y=0
  command: parse_bytes(s, b'\xe2\x80\x8d')   # ZWJ U+200D
    -> cell0='👩\u200d' width(0)=2 [U+1F469 U+200D]   cursor x=2 y=0
  command: parse_bytes(s, b'\xf0\x9f\x91\xa7')   # girl U+1F467
    -> cell0='👧' width(0)=2 [U+1F467]   cursor x=2 y=0
  command: parse_bytes(s, b'\xe2\x80\x8d')   # ZWJ U+200D
    -> cell0='👧\u200d' width(0)=2 [U+1F467 U+200D]   cursor x=2 y=0
  command: parse_bytes(s, b'\xf0\x9f\x91\xa6')   # boy U+1F466
    -> cell0='👦' width(0)=2 [U+1F466]   cursor x=2 y=0
```

**Cause → effect (naming the code that does the work):**

- Each **base emoji** is a width-2 codepoint (`wcwidth_std(U+1F468) == 2`, see §6.4). It is classified inside the per-codepoint loop of **`draw_text_loop`** (`kitty/screen.c:L803-L845`): it is not ignored (`is_ignored_char`, `L805`) and not combining (`is_combining_char`, `L806`), so its width is taken at `L814` (`char_width = wcwidth_std(ch);`), it is written into the cell at `L836` (`s->cp[self->cursor->x].ch = ch;`), and because `char_width == 2` the code at `L838-L843` marks the cell width 2, zeroes the trailer, and advances the cursor twice — hence `cursor x=2`.
- The **ZWJ** (`U+200D`, width 0) is classified as a combining character (`is_combining_char`, `kitty/unicode-data.c:L11`) at `draw_text_loop` `kitty/screen.c:L806`, and is routed to **`draw_combining_char`** (`kitty/screen.c:L810`, then `L663-L702`). That function attaches it to the *previous* cell via **`line_add_combining_char`** (`kitty/screen.c:L678` → `kitty/line.c:L457-L467`). This is exactly why each `cell0='👨\u200d'` line shows the base **plus** the ZWJ stored on it (`[U+1F468 U+200D]`), and why the cursor does **not** move for the ZWJ.
- The **next base** cannot fit. On a 1-column screen the fit check at **`kitty/screen.c:L821`** — `if (self->columns < self->cursor->x + (unsigned int)char_width)` — is true (`1 < 2`), and since autowrap (DECAWM) is on by default the `L822-L824` branch runs `continue_to_next_line(self); init_text_loop_line(self, s);`. The new base is then drawn at column 0, which is why cell 0 shows the *new* base and the previous *base + ZWJ* is gone from view.

### 2.2 Refinement (observed): the net "overwrite" is really **scroll-then-redraw**

On a 1-column screen a width-2 base can *never* fit — the fit check `self->columns < self->cursor->x + char_width` (`kitty/screen.c:L821`) is true even for the very first base (`1 < 0+2`). With autowrap on, each base triggers **`continue_to_next_line`** (`kitty/screen.c:L523-L528`; `L526` `self->cursor->x = 0;`, `L527` `screen_linefeed(self);`), which **scrolls** the single line up; the new base is drawn at column 0 of the now-blank line. So the visible 1×1 cell always ends as only the last base `👦`/`U+1F466` (cursor `x=2`), while the earlier *base + ZWJ* groups scroll into scrollback history.

Command: `make_screen(cols=1, scrollback=…)`, `feed(s, FAMILY)`, then read `s.line(0)` and `s.historybuf`.

```
scrollback=0:   visible='👦' [U+1F466] cursor x=2 y=0 ; historybuf.count=1
scrollback=100: visible='👦' [U+1F466] cursor x=2 y=0 ; historybuf.count=4
  history[0]='👧\u200d' [U+1F467 U+200D]
  history[1]='👩\u200d' [U+1F469 U+200D]
  history[2]='👨\u200d' [U+1F468 U+200D]
  history[3]=''         []
```

The history is newest-first: the three evicted *base + ZWJ* groups plus the initial blank line were scrolled off, and the visible cell retains only the newest base. (The blank oldest line reads back as the empty string `''` via `str(historybuf.line(3))`; read by cell index it is a NUL, `'\x00'`/`U+0000` — the same empty line, two readback representations.) In short: the terminal's screen buffer decides *what to keep* purely by width bookkeeping — a width-2 base overwrites the visible column and everything attached to the prior base leaves the viewport. The AAP's "overwrites cell 0" is the correct **net visible effect**; the precise observed mechanism is **scroll-then-redraw**.

---


## 3. Q2 — What the terminal thinks is actually present once everything settles

**Direct answer:** in the 1×1 case the single cell holds **exactly one double-width `👦` (`U+1F466`) and nothing else**, and the cursor rests at `x=2, y=0`. kitty does **not** merge the ZWJ sequence into one grapheme; the 1×1 result is a consequence of space, not of clustering — as the wide-screen contrast proves.

Command: `feed(s, FAMILY)` on each screen; read every cell of `line(0)`.

```
# 1x1: parse_bytes(s, FAMILY.encode('utf-8'))  bytes=b'\xf0\x9f\x91\xa8\xe2\x80\x8d\xf0\x9f\x91\xa9\xe2\x80\x8d\xf0\x9f\x91\xa7\xe2\x80\x8d\xf0\x9f\x91\xa6'
  1x1 final: cell0='👦' width=2 [U+1F466]  str(line0)='👦'  cursor x=2 y=0
# 20x1: parse_bytes(s, FAMILY.encode('utf-8'))
  20x1 final: x0='👨\u200d'(w2) x1='\x00'(w0) x2='👩\u200d'(w2) x3='\x00'(w0) x4='👧\u200d'(w2) x5='\x00'(w0) x6='👦'(w2) x7='\x00'(w0)
  20x1 str(line0)='👨\u200d👩\u200d👧\u200d👦'  cursor x=8 y=0
  20x1 str == input family? True
```

**Cause → effect:**

- The unit of storage is the **screen-buffer cell**, a `CPUCell` (`kitty/data-types.h:L223-L227`): one primary codepoint `char_type ch` plus a fixed array of three combining slots `combining_type cc_idx[3]`. A wide (width-2) glyph occupies **two** columns: the primary cell (whose `GPUCell` `attrs.width` bit-field, `kitty/data-types.h:L192-L216`, is set to 2) and a following trailer cell whose stored `ch` is NUL (`\x00`) with width 0 — set at `draw_text_loop` `kitty/screen.c:L838-L843`.
- On the **20-column** screen there is room for every base, so the four bases occupy the **separate** cells `x0, x2, x4, x6`, each with its ZWJ stored as a combining mark on the preceding base (`👨\u200d`, `👩\u200d`, `👧\u200d`, then `👦`). Cells `x1, x3, x5, x7` are the NUL wide-char trailers. The cursor advances a full **8 columns** — four width-2 glyphs — and `str(line0)` round-trips to the exact input (`str == input family? True`).
- On the **1×1** screen there is room for only one column, so only the final base remains visible; the settled content is the single `👦` (`U+1F466`).

This 20×1 result matches kitty's own regression test **`test_zwj`** (`kitty_tests/screen.py:L123-L128`), which asserts `str(s.line(0)) == q` and `s.cursor.x == 8` for the same family sequence — confirming kitty stores four discrete double-width cells rather than one merged cluster.

---

## 4. Q3 — The control-sequence state query and how it reflects the grapheme handling

**Direct answer:** asked for its cursor position with a **Cursor Position Report** (`CSI 6 n` — the control sequence `ESC [ 6 n`), kitty replies `ESC[1;2R` in the 1×1 case and `ESC[1;9R` on the 20-column screen. The reported **column** is a direct readout of how many columns the width-based grapheme handling consumed.

Command: `feed(s, FAMILY)` then `cpr(s, c)` (sends `b'\x1b[6n'`, reads the reply from `Callbacks.wtcbuf`).

```
  1x1 : cursor before query x=2 y=0 ; send ESC[6n -> reply b'\x1b[1;2R' (ESC[1;2R)
  20x1: cursor before query x=8 y=0 ; send ESC[6n -> reply b'\x1b[1;9R' (ESC[1;9R)
```

**Cause → effect (naming the code that does the work):**

- The query is handled by **`report_device_status`**, case 6 (`kitty/screen.c:L2179-L2201`, case body `L2188-L2198`). It reads the internal cursor (`L2189` `x = self->cursor->x; y = self->cursor->y;`), then emits a **1-based** reply `CSI <row>;<col> R` (`L2196` `snprintf(buf, ..., "%s%u;%uR", (private ? "?" : ""), y + 1, x + 1)`, written back to the child at `L2197`).
- In the **1×1** case the internal cursor is `x=2`. Because `x >= self->columns` (2 ≥ 1) **and** it is the last line, the clamp at `L2190-L2193` takes the `else x--;` branch (`L2192`), so `x` becomes 1 and the reported column is `x + 1 = 2` → **`ESC[1;2R`**.
- In the **20-column** case `x=8` is within bounds (`8 < 20`), so no clamp applies and the reported column is `8 + 1 = 9` → **`ESC[1;9R`**.
- **How this reflects the earlier grapheme handling:** the reported column is exactly the width the buffer consumed. A terminal that had collapsed the whole family into a single double-width grapheme would leave its cursor at column 2 and report **column 3**; kitty reports **column 9**, exposing that it stored **four separate double-width cells**, not one merged cluster. State reporting is thus a faithful mirror of the width-based cell/cursor geometry.

(The related geometry query `screen_report_size`, `kitty/screen.c:L2142-L2178`, similarly just reports stored dimensions; it is mentioned only for completeness and is not exercised here.)

---


## 5. Q4 — How normalization, grapheme breaking, and state reporting interact under extreme constraints

**Direct answer:** they barely "interact" at all, because two of the three subsystems the question names **do not exist as such** in kitty's C core. There is **no normalization step** and **no UAX #29 grapheme-cluster segmenter**; there is only **width-based segmentation** feeding a fixed-capacity cell, and **state reporting is a passive mirror** of the geometry that segmentation produced. Each named sub-part, explicitly:

### 5a. Normalization — **NONE**

kitty performs no Unicode normalization. A decomposed sequence and its precomposed equivalent are stored **differently**, byte-for-byte as received.

Command: `feed(s, "e\u0301")` vs `feed(s, "\u00e9")` on a 20×1 screen.

```
# decomposed:  parse_bytes(s, ('e'+U+0301).encode('utf-8'))  bytes=b'e\xcc\x81'
  decomposed 'e'+U+0301 -> cell0='é' [U+0065 U+0301]  width=1
# precomposed: parse_bytes(s, U+00E9.encode('utf-8'))          bytes=b'\xc3\xa9'
  precomposed  U+00E9   -> cell0='é' [U+00E9]  width=1
```

The decomposed form is stored as base `U+0065` plus combining `U+0301` (the acute accent attached via `line_add_combining_char`, `kitty/line.c:L457-L467`); the precomposed form is stored as the single codepoint `U+00E9`. They render alike but are **not** unified — there is no NFC/NFD pass and no reordering. This is corroborated statically in §6.5: the C core contains no normalization code at all.

### 5b. Grapheme breaking — **WIDTH-BASED, not UAX #29**

kitty's "grapheme breaking" is width-based segmentation performed inline in **`draw_text_loop`** (`kitty/screen.c:L803-L818`): a codepoint with `wcwidth_std(ch) >= 1` begins a **new cell**, while a width-0 / combining codepoint attaches to the **previous** cell. This applies uniformly to the ZWJ (`U+200D`), the zero-width space (`U+200B`), and the variation selectors (`U+FE0F`, `U+FE0E`).

Command: feed `X` + a zero-width codepoint + `Y` on 20×1.

```
  'X'+U+200B(ZWSP)+'Y' on 20x1: str='X\u200bY' cells x0='X\u200b' x1='Y' x2='\x00' cursor x=2
  'X'+U+200D(ZWJ)+'Y'  on 20x1: str='X\u200dY' cells x0='X\u200d'[U+0058 U+200D] x1='Y' cursor x=2
```

Both `X` and `Y` are width-1 and each begins its own cell; the zero-width joiner/space attaches to the preceding `X`. The cursor advances only 2 columns — the width of `X` and `Y` — with the zero-width codepoint contributing nothing. This matches kitty's own `test_zwj` zero-width cases (`kitty_tests/screen.py:L129-L134`: `X\u200bY`, `X\u200cY`, `X\u200dY` each satisfy `str == input` and `cursor.x == 2`).

The design is **deliberate**. kitty's changelog explains why ZWJ is preserved as a combining mark rather than used to collapse a cluster (`docs/changelog.rst:L3272-L3275`):

> Round-trip the zwj unicode character. Rendering of sequences containing zwj is still not implemented, since it can cause the collapse of an unbounded number of characters into a single cell. However, kitty at least preserves the zwj by storing it as a combining character.

The per-cell combining capacity is likewise an intentional constant — three marks — recorded in the changelog (`docs/changelog.rst:L962`): "Increase the max number of combining chars per cell from two to three, without increasing memory usage." That capacity is realized as `combining_type cc_idx[3]` in `CPUCell` (`kitty/data-types.h:L223-L227`).

### 5c. State reporting — **a passive mirror of cell/cursor geometry**

State reporting has **no independent notion of graphemes**. Whatever the width-based buffer decided — how many cells were consumed and where the cursor landed — is exactly what `report_device_status` reflects, as demonstrated in Q3 (`ESC[1;9R` exposes 8 consumed columns / four double-width cells). It reads `self->cursor->x`/`.y` and prints them; nothing more.

**Interaction under extreme constraints, summarized:** under a 1×1 constraint the width-2 base cannot fit, so autowrap scrolls each base off and only the last remains visible; normalization never runs, so nothing is merged or reordered; grapheme breaking is purely `wcwidth_std`-driven, so the ZWJ is stored — not consumed — as a combining mark; and CPR faithfully reports the column that this width-based bookkeeping produced. The three concerns compose into one width-driven pipeline with reporting bolted passively onto its tail.

---


## 6. Edge-case appendix (exhaustive condition coverage)

These conditions complete the "extreme constraints" picture the question asks about. Each was observed at runtime and confirmed stable across repeated runs.

### 6.1 Combining-mark overflow — the fixed 3-slot limit

A `CPUCell` holds a primary `ch` plus `cc_idx[3]` (three combining slots, `kitty/data-types.h:L223-L227`). Adding a base plus five combining marks keeps the base and the first two marks stable, while the third slot always holds the **last-written** mark.

Command: `make_screen(cols=5)`, `feed(s, 'e')`, then feed each of `U+0300 U+0301 U+0302 U+0303 U+0304` one at a time.

```
  after 'e'         -> cell0='e' [U+0065]
  after U+0300      -> cell0='è' [U+0065 U+0300]
  after U+0301      -> cell0='è́' [U+0065 U+0300 U+0301]
  after U+0302      -> cell0='è́̂' [U+0065 U+0300 U+0301 U+0302]
  after U+0303      -> cell0='è́̃' [U+0065 U+0300 U+0301 U+0303]
  after U+0304      -> cell0='è́̄' [U+0065 U+0300 U+0301 U+0304]
```

**Cause → effect:** after `U+0302` all three slots are full (`[U+0065 U+0300 U+0301 U+0302]`); the next mark `U+0303` **overwrites slot 3** (`U+0302` discarded) → `[U+0065 U+0300 U+0301 U+0303]`, and `U+0304` likewise → `[U+0065 U+0300 U+0301 U+0304]`. The base and first two marks are stable; slot 3 always holds the last-written mark. The mechanism is **`line_add_combining_char`** (`kitty/line.c:L457-L467`): the loop at `L463-L464` fills the first empty slot, and when all are full `L466` (`cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);`) overwrites the final slot. This matches kitty's own `test_line` (`kitty_tests/datatypes.py:L198-L210`).

### 6.2 Variation selectors — VS16 (`U+FE0F`) promotes, VS15 (`U+FE0E`) demotes

```
# VS16 promotes narrow emoji base U+2716 (width 1) to width 2:
  cols=1: after U+2716 cell0='✖' w=1 cursor.x=1 ; after +VS16 cell0='✖️' [U+2716 U+FE0F] w=2 cursor.x=1 ; ESC[6n->ESC[1;1R
  cols=5: after U+2716 cell0='✖' w=1 cursor.x=1 ; after +VS16 cell0='✖️' [U+2716 U+FE0F] w=2 cursor.x=2 ; ESC[6n->ESC[1;3R
# VS15 demotes wide emoji base U+1F600 (width 2) to width 1:
  cols=5: after U+1F600 cell0='😀' w=2 cursor.x=2 ; after +VS15 cell0='😀︎' [U+1F600 U+FE0E] w=1 cursor.x=1
```

**Cause → effect:** both selectors are combining codepoints handled by **`draw_combining_char`** (`kitty/screen.c:L662-L702`). VS16 is handled at `L679-L689` (sets the cell width to 2; advances the cursor at `L687` only if there is room, otherwise `move_widened_char` at `L688`); VS15 is handled at `L690-L699` (sets width 1; `self->cursor->x--` at `L698`). On the **1-column** screen the VS16 width promotion happens in place but the cursor cannot advance (stays `x=1` → `ESC[1;1R`); on the **5-column** screen it advances to `x=2` (`ESC[1;3R`). This is the same promotion tested by kitty's `test_emoji_presentation` (`kitty_tests/fonts.py:L202-L226`).

### 6.3 Regional-indicator flag pair (`U+1F1FA U+1F1F8` = 🇺🇸)

```
  after RI-U cell0='🇺' w=2 cursor.x=2 ; after RI-S cell0='🇺🇸' [U+1F1FA U+1F1F8] w=2 cursor.x=2 str(line0)='🇺🇸'
```

**Cause → effect:** the two regional indicators **merge into a single double-width cell** — the second RI is stored as a combining codepoint on the first — leaving the cursor at `x=2`. The mechanism is **`draw_second_flag_codepoint`** (`kitty/screen.c:L637-L651`): it locates the previous cell (`L642` `xpos = self->cursor->x - 2;`) and, when `is_flag_pair(cell->ch, ch)` holds and the first combining slot is empty (`L651`), attaches the second indicator via `line_add_combining_char`.

### 6.4 Supporting: runtime widths driving the segmentation

Widths via the extension's `wcswidth` (each a single codepoint):

```
man U+1F468=2, ZWJ U+200D=0, heavy-x U+2716=1, VS16 U+FE0F=0, VS15 U+FE0E=0,
comb-acute U+0301=0, RI-U U+1F1FA=2, ZWSP U+200B=0
```

These are the exact widths that `draw_text_loop` uses to decide cell boundaries: width-2 codepoints begin a two-column glyph, width-1 codepoints begin a one-column cell, and width-0 codepoints attach as combining marks. `wcwidth_std` is defined in `kitty/wcswidth.c` (used at `L62`; the emoji-presentation helper `is_emoji_presentation_base` is at `L47`/`L54`).

### 6.5 Static proof: no normalization, no UAX #29

A grep of the C core (`kitty/*.c kitty/*.h`) confirms the absence of any normalization or grapheme-cluster machinery:

- **Zero** matches for `grapheme`, `uax29`, or `grapheme_break`.
- The only `nfd` match is the substring inside the identifier `infd` (an input file descriptor) in `kitty/fast-file-copy.c`.
- `normaliz` appears only in FreeType gray-level code (`kitty/freetype.c:496`, "Normalize gray levels…") and in OpenGL `GL_*` constants (`kitty/gl-wrapper.h`).

This confirms kitty does **no** Unicode normalization and uses **no** UAX #29 grapheme-cluster segmenter — its "grapheme breaking" is purely width-based, exactly as the runtime observations show.

### 6.6 Boundary defect — reproducible crash (documented, **not fixed**)

Drawing a width-2 emoji into a **1-column** screen with **autowrap disabled** (`CSI ?7l`) crashes the process deterministically. This is reported as an observation with its root cause; **repairing it is out of scope and no patch is produced.**

Command: on an N-column screen, `feed(s, "\x1b[?7l")` (CSI ?7l → disable DECAWM/autowrap), then feed one width-2 emoji `U+1F600`. Run 3×.

```
########## 1-column screen, autowrap OFF, run x3 ##########
STEP: autowrap disabled on 1-col screen; about to draw U+1F600 (width 2)
run 1: exit code=139  => killed by signal 11 (SEGV)
STEP: autowrap disabled on 1-col screen; about to draw U+1F600 (width 2)
run 2: exit code=139  => killed by signal 11 (SEGV)
STEP: autowrap disabled on 1-col screen; about to draw U+1F600 (width 2)
run 3: exit code=139  => killed by signal 11 (SEGV)

########## 2-column screen, autowrap OFF (control) ##########
STEP: autowrap disabled on 2-col screen; about to draw U+1F600 (width 2)
SURVIVED: cols=2 cursor.x=2 cell0='😀' width=2
2-col: exit code=0
```

gdb backtrace (command: `gdb -q --batch -ex run -ex bt --args python3 crash.py 1`):

```
Program received signal SIGSEGV, Segmentation fault.
0x00007ffff70757f3 in draw_text_loop.lto_priv () from .../kitty/fast_data_types.so
#0  draw_text_loop.lto_priv ()        from .../kitty/fast_data_types.so
#1  draw_text.lto_priv ()             from .../kitty/fast_data_types.so
#2  run_worker.lto_priv ()            from .../kitty/fast_data_types.so
#3  test_parse_written_data.lto_priv () from .../kitty/fast_data_types.so
#4  0x0000000000550c3c in ?? ()
#5  PyObject_Vectorcall ()
#6  _PyEval_EvalFrameDefault ()
...
```

**Root cause (explanation only — not patched):** in the autowrap-**off** branch, `kitty/screen.c:L826` computes `self->cursor->x = self->columns - char_width;`. With `columns == 1` and `char_width == 2`, and because the cursor/columns types are **unsigned** (`index_type` = `unsigned int`, `kitty/data-types.h:L65`; `unsigned int columns`, `kitty/screen.h:L91`), `1 - 2` underflows to `4294967295`. Subsequent cell access — `cursor_on_wide_char_trailer` reading `s->gp[self->cursor->x - 1]` (`kitty/screen.c:L717-L719`) and `zero_cells` at `s->cp + self->cursor->x` (`kitty/screen.c:L835`) — then dereferences far out of bounds → SIGSEGV. The **2-column** screen does **not** crash (`2 - 2 = 0` is valid), which confirms the trigger is specifically `columns < char_width` in the no-wrap path.

> Note on the backtrace: the crash and its call chain (`draw_text_loop` → `draw_text` → `run_worker` → `test_parse_written_data`) were reproduced live (3/3 SIGSEGV, plus the surviving 2-column control). The exact hexadecimal addresses and the `.lto_priv` symbol suffix are build-specific and may differ between toolchains, but the faulting function (`draw_text_loop`) and the frame chain are stable.

---


## 7. Final coverage pass

Every question, every named concept, and every edge condition, with its primary code citation:

| Item asked | Where answered | Observed result | Primary citation(s) |
|---|---|---|---|
| **Q1** — retention under no space (1×1) | §2 | only last base `👦`/`U+1F466` kept; cursor `x=2` | `screen.c:L821` fit check, `L822-L824` autowrap, `L523-L528` scroll |
| **Q2** — settled cell contents | §3 | 1×1 = single `👦`; 20×1 = four width-2 cells, cursor `x=8` | `data-types.h:L223-L227` `CPUCell`; `screen.py:L123-L128` `test_zwj` |
| **Q3** — state-query response | §4 | `CSI 6 n` → `ESC[1;2R` (1×1) / `ESC[1;9R` (20×1) | `screen.c:L2188-L2198` `report_device_status` |
| **Q4a** — normalization | §5a | none — decomposed `[U+0065 U+0301]` ≠ precomposed `[U+00E9]` | `line.c:L457-L467`; §6.5 static proof |
| **Q4b** — grapheme breaking | §5b | width-based (not UAX #29); ZWJ stored as combining mark | `screen.c:L803-L818`; `changelog.rst:L3272-L3275`, `L962` |
| **Q4c** — state reporting | §5c | passive mirror of cell/cursor geometry | `screen.c:L2179-L2201` |
| **ZWJ** (`U+200D`) | §2, §5b | width 0; attached to preceding base, not consumed | `unicode-data.c:L11` `is_combining_char`; `line.c:L457-L467` |
| **Multi-codepoint emoji** (family) | §2, §3 | four separate width-2 bases, never merged | `screen.c:L838-L843`; `screen.py:L123-L128` |
| **Screen-buffer cell** | §3, §6.1 | primary `ch` + `cc_idx[3]`; width is a 2-bit field | `data-types.h:L192-L216`, `L223-L227` |
| **Grapheme breaking** | §5b, §6.4 | driven by `wcwidth_std` | `wcswidth.c:L62`; `screen.c:L803-L818` |
| **Normalization** | §5a, §6.5 | none (no NFC/NFD, no reordering) | §6.5 static proof |
| **Control-sequence state reporting** | §4, §5c | `report_device_status` CPR mirrors geometry | `screen.c:L2179-L2201` |
| Edge — combining 3-slot overflow | §6.1 | slot 3 overwritten by last mark | `line.c:L466`; `datatypes.py:L198-L210` |
| Edge — VS16 / VS15 | §6.2 | VS16 promotes to width 2; VS15 demotes to width 1 | `screen.c:L679-L689`, `L690-L699` |
| Edge — regional-indicator flag pair | §6.3 | merge into one double-width cell | `screen.c:L637-L651` |
| Edge — autowrap on/off | §2.2, §6.6 | on → scroll; off + `columns < char_width` → SIGSEGV | `screen.c:L822-L824`, `L826` |
| Defect — reproducible crash | §6.6 | SIGSEGV 3/3; 2-col control survives; documented, not fixed | `screen.c:L826`; `data-types.h:L65`; `screen.h:L91` |

**Through-line:** kitty's screen buffer is **width-driven, not cluster-driven**. It never normalizes and never runs UAX #29 segmentation; it decides cell boundaries purely by `wcwidth_std`, stores ZWJ / variation selectors / a second regional indicator as combining marks in a fixed three-slot cell, and its control-sequence state reporting is a passive mirror of the resulting cell/cursor geometry. Under a 1×1 constraint the width-2 base cannot fit, so autowrap scrolls each base off and only the last (`👦`/`U+1F466`) remains visible — and `CSI 6 n` faithfully reports the column that this width-based bookkeeping produced.

*All values above were observed by driving kitty v0.35.2's real VT byte-parser (`parse_bytes`) against the compiled `fast_data_types` extension; each scenario was confirmed stable across at least two runs and the crash across three. No inferred (non-observed) statements were required.*

