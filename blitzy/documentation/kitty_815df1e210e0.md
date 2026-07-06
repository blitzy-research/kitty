# How kitty 0.35.2 Handles a ZWJ Multi-Codepoint Emoji in a 1×1 Cell (and What a State Query Then Reports)

> I am trying to get a practical understanding of how kitty handles complex Unicode at runtime, especially in cases where the screen state is hard to reason about just by reading the code. When the terminal receives a stream of zero width joiners that form a multi codepoint emoji, how does the internal screen buffer decide what to keep when there is almost no space available, such as a one by one cell? What does the terminal think is actually present in that cell once everything settles? If the terminal is then asked to report part of its current state through a control sequence query, what response does it generate and how does that reflect the earlier grapheme handling? I want to understand how normalization, grapheme breaking, and state reporting interact when the terminal is under extreme constraints. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.

This document answers four sub-questions, labelled throughout:

- **Q1** — By what mechanism does the screen buffer decide what to *keep* when a ZWJ multi-codepoint emoji is drawn into a 1×1 cell?
- **Q2** — Once the stream settles, what does the terminal consider actually present in that single cell?
- **Q3** — When asked to report its state via a control sequence, which exact response bytes does it emit, and how does that reflect the earlier grapheme handling?
- **Q4** — How do normalization, grapheme breaking/segmentation, and state reporting interact under extreme space constraint?

---

## How this was investigated

Every behavioral claim below was produced by **building kitty's native extension from source and driving its real C screen model through the real VT parser** — not by reading code alone. UTF-8 bytes (the family emoji, and the `ESC [ 6 n` / `ESC [ 5 n` queries) were fed through `parse_bytes(screen, data)` — which chains kitty's `test_create_write_buffer()` → `test_commit_write_buffer(...)` → `test_parse_written_data(...)`, i.e. the **full VT parser in `kitty/vt-parser.c`** — so the exact production code path is exercised end-to-end. The investigation is **read-only**: the sole file written is this document. The two probe scripts lived under `/tmp` (outside the repository) and were removed afterward, leaving `git status` clean.

Two conventions are kept strictly distinct in this document:

- **Observed** — a value captured verbatim from a probe run (embedded unedited, with the command that produced it).
- **Mechanism / inference from source** — the explanation of *why* the code produces that value, grounded in a `file:line` citation naming the responsible function or struct.

> **Version context up front (details in [§ Version context](#version-context)):** this commit is `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, kitty **0.35.2** (June 2024). It **predates** kitty's later text-sizing / grapheme-segmentation protocol (GitHub #8226 and #8533, 2025). Therefore this build uses the **classic per-codepoint width model** — each codepoint's width comes from `wcwidth_std`, ZWJ (`U+200D`) is zero-width/combining, and there is **no grapheme clustering into a single cell**. Do not read these results against a newer kitty.

---

## Canonical build & run baseline

**Build command (default configuration):**

```
CI=true python3 setup.py build --debug --ignore-compiler-warnings
```

- The `build` action is defined at `setup.py:1084` (`def build(...)`); the `--ignore-compiler-warnings` command-line flag is defined at `setup.py:2003`.
- **Why `--ignore-compiler-warnings`:** it only drops `-Werror` / `-pedantic-errors`. The toggle is `setup.py:491`: `werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'`. This prevents a pre-existing, unrelated `switch` warning in bundled GLFW Wayland code from aborting the build under `-Werror`. It is a legitimate build flag, **not** a repository modification.
- **Observed build result (this environment):** a forced clean rebuild completed with **exit code 0**. It compiled 122 C translation units and linked 5 targets, including the native extension:

```
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
```

- **Native extension produced:** `kitty/fast_data_types.so`, observed size **6,285,000 bytes**. This is the only artifact needed to drive the C `Screen` model; a display server, GPU, or fonts are not required.
- **Reported version: 0.35.2** — confirmed at runtime:

```
$ PYTHONPATH=. python3 -c "import kitty.fast_data_types as f; from kitty.constants import str_version; print(f.Screen, str_version)"
<class 'fast_data_types.Screen'> 0.35.2
```

  The version lives in `kitty.constants.str_version` (it is **not** an attribute of `fast_data_types`). The changelog corroborates the release date at `docs/changelog.rst:59`: `0.35.2 [2024-06-22]`; `git show -s --format=%ci HEAD` reports the commit date `2024-06-24`.
- **Toolchain observed (this environment):** Ubuntu 25.10, Python 3.13.7, gcc 15.2.0, pkg-config 1.8.1. kitty's `pyproject.toml:2` requires `requires-python = ">=3.8"`.
- **Build artifacts are gitignored:** `git check-ignore kitty/fast_data_types.so build` prints both paths, so the build leaves no tracked changes.

> **Note on the build exit code.** kitty's build also links a Go launcher; if the `go` tool is absent the build exits non-zero *after* the C extension has already linked (the launcher is out of scope and not needed for this investigation). In this environment `go` (1.24.4) is present, so the build completed fully with **exit 0** and even `[5/5] Linking launcher` succeeded. Either way, `kitty/fast_data_types.so` is produced and imports cleanly — that is the prerequisite for every observation below.

### Observation harness (real entry point — no bypass)

The in-memory `Screen` is constructed exactly as kitty's own `BaseTest.create_screen(...)` does (`kitty_tests/__init__.py:237`): `set_options(Options())`, then `Screen(callbacks, lines, cols, scrollback, cell_width, cell_height, 0, callbacks)` (`kitty_tests/__init__.py:240`). A `Callbacks` object (`kitty_tests/__init__.py:39`) whose `write` appends child-directed bytes to `wtcbuf` (`kitty_tests/__init__.py:50-51`) is **retained** so the terminal's reply to a control sequence can be read back. Settled cell text is read with `str(screen.line(0))` (`kitty_tests/__init__.py:407`).

Bytes are fed through `parse_bytes(screen, data)` (`kitty_tests/__init__.py:30`), which loops `screen.test_create_write_buffer()` → `screen.test_commit_write_buffer(...)` → `screen.test_parse_written_data(...)` — the **real VT parser** in `kitty/vt-parser.c`.

> **Why not `screen.draw(...)`?** kitty's own unit tests (e.g. `test_zwj`, `kitty_tests/screen.py:123`) use the `Screen.draw()` Python binding (`kitty/screen.c:3769`), which calls `screen_draw_text` (`kitty/screen.c:866`) directly and **bypasses** the parser. This investigation deliberately uses `parse_bytes` instead, so that both the emoji bytes **and** the `ESC [ 6 n` query traverse the real routing in `dispatch_csi` (`kitty/vt-parser.c:1027`) and the DSR dispatch (`kitty/vt-parser.c:1172-1173`). This is the mandated real entry point.

**User example, carried through exactly.** The family ZWJ emoji `👨‍👩‍👧‍👦` = `U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466` (7 codepoints) is drawn into a **1×1 cell** (`cols=1, lines=1`) as the primary case, with a **20-column** unconstrained screen as the contrasting baseline.

---

## Q1 — The "keep" decision under constraint

**Observed** — the family emoji fed codepoint-by-codepoint into the 1×1 screen through the real parser (`python3 /tmp/kitty_probe.py`, unedited):

```
family codepoints: ['0x1f468', '0x200d', '0x1f469', '0x200d', '0x1f467', '0x200d', '0x1f466']

=== 1x1 progression ===
before: '' cursor=(0,0)
after 0x1f468 : '👨' cursor=(2,0)
after 0x200d  : '👨\u200d' cursor=(2,0)
after 0x1f469 : '👩' cursor=(2,0)
after 0x200d  : '👩\u200d' cursor=(2,0)
after 0x1f467 : '👧' cursor=(2,0)
after 0x200d  : '👧\u200d' cursor=(2,0)
after 0x1f466 : '👦' cursor=(2,0)
SETTLED line(0)= '👦' codepoints= ['0x1f466'] cursor=(x=2,y=0)
```

Reading the progression: each **base emoji** replaces the single cell's content and parks the cursor at `x=2`; each **ZWJ** (`U+200D`) attaches to the base currently in the cell (the cell momentarily reads e.g. `'👨\u200d'`) **without moving the cursor**; the next base then overwrites cell and ZWJ together. When the stream settles the cell holds only `👦` (`U+1F466`).

**Mechanism / inference from source.** The per-codepoint draw loop `draw_text_loop` (`kitty/screen.c:763`) processes each codepoint individually:

1. `int char_width = 1;` is the default (`kitty/screen.c:803`). For non-ASCII input (`if (ch > DEL)`, `kitty/screen.c:804`; `DEL` is `0x7f`, `kitty/control-codes.h:56`): default-ignorable codepoints are skipped by `if (is_ignored_char(ch)) continue;` (`kitty/screen.c:805`), and combining codepoints are detected by `if (UNLIKELY(is_combining_char(ch)))` (`kitty/screen.c:806`).
2. **ZWJ `U+200D` is classified combining** (and is zero-width), so — not being a flag codepoint — it routes to `draw_combining_char(self, s, ch); continue;` (`kitty/screen.c:810`). `draw_combining_char` (`kitty/screen.c:663`) locates the **preceding** cell and calls `line_add_combining_char` (`kitty/line.c:457`, declared `kitty/lineops.h:90`) to append the mark **without advancing the cursor**. This is exactly why every `after 0x200d` line shows the ZWJ glued onto the current base while the cursor stays at `x=2`.
3. For each width-2 **base emoji**, `char_width = wcwidth_std(ch);` (`kitty/screen.c:814`) returns 2. The autowrap/overflow test `if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width))` (`kitty/screen.c:821`) is **true** on a one-column screen (`1 < 0 + 2`). With DECAWM on by default (`kitty/screen.c:822`), the loop advances the line and re-initialises it — `continue_to_next_line(self); init_text_loop_line(self, s);` (`kitty/screen.c:823-824`) — then writes the cell and zeroes the wide-char trailer (`kitty/screen.c:835-843`: `zero_cells(...)`, `s->cp[self->cursor->x].ch = ch;`, set the base cell `width = 2`, zero the trailer cell to `width = 0`, and advance the cursor).

**The "keep" decision (net effect).** Because a single column cannot fit a width-2 glyph, **each successive base emoji wraps and overwrites the one cell**; the ZWJ that had just attached is discarded together with the base it was attached to. There is **no grapheme clustering** that would merge the seven codepoints into one cell (see [Q4](#q4--how-normalization-grapheme-breaking--state-reporting-interact) and [§ Version context](#version-context)). **Only the final base survives.**

---

## Q2 — Final cell content

**Observed** — on the 1×1 screen the settled cell retains **only the final base emoji `👦` (`U+1F466`)**:

```
SETTLED line(0)= '👦' codepoints= ['0x1f466'] cursor=(x=2,y=0)
```

The cell text is read via `str(screen.line(0))` (`kitty_tests/__init__.py:407`). The three earlier bases (`👨 👩 👧`) and all intermediate ZWJs are gone — they were overwritten by the wrap-and-rewrite described in Q1.

**Mechanism / inference from source — why only one base (plus marks) could ever fit.** The cell storage model is:

```c
typedef struct {
    char_type ch;                 // kitty/data-types.h:224  (the single base codepoint)
    hyperlink_id_type hyperlink_id; // kitty/data-types.h:225
    combining_type cc_idx[3];     // kitty/data-types.h:226  (up to three combining marks)
} CPUCell;                        // kitty/data-types.h:227
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell"); // kitty/data-types.h:228
```

A `CPUCell` (`kitty/data-types.h:224-226`) holds **exactly one base codepoint (`ch`) plus at most three combining marks (`cc_idx[3]`)** — with a `hyperlink_id` field sitting between them. It structurally **cannot** hold a fully-joined seven-codepoint sequence. On the 1×1 screen the surviving cell is the last base `👦` with empty combining slots; on a wider screen the same struct is why each family member occupies its own cell with the ZWJ folded into a neighbour (see [Q4](#q4--how-normalization-grapheme-breaking--state-reporting-interact)).

---

## Q3 — State-query response (byte-exact)

**Observed** — immediately after the family emoji settles on the 1×1 screen, driving the DSR queries through the real parser yields (unedited):

```
ESC[6n -> b'\x1b[1;2R' hex= 1b5b313b3252
ESC[5n -> b'\x1b[0n' hex= 1b5b306e
```

The cursor-position report query `ESC [ 6 n` (CPR) returns exactly **`ESC [ 1 ; 2 R`** — raw bytes `b'\x1b[1;2R'`, hex `1b5b313b3252`. The device-status query `ESC [ 5 n` returns **`b'\x1b[0n'`** (hex `1b5b306e`, "terminal OK").

**Mechanism / inference from source — how `1;2R` is derived.** The query is routed by `dispatch_csi` (`kitty/vt-parser.c:1027`); the DSR case (`kitty/vt-parser.c:1172`) invokes `CALL_CSI_HANDLER1P(report_device_status, 0, '?')` (`kitty/vt-parser.c:1173`), reaching `report_device_status` (`kitty/screen.c:2179`). Inside, the CPR branch is `case 6:` (`kitty/screen.c:2188`):

```c
case 5:  // device status
    write_escape_code_to_child(self, ESC_CSI, "0n");   // kitty/screen.c:2186
    break;
case 6:  // cursor position                            // kitty/screen.c:2188
    x = self->cursor->x; y = self->cursor->y;
    if (x >= self->columns) {                          // kitty/screen.c:2190  (clamp)
        if (y < self->lines - 1) { x = 0; y++; }
        else x--;
    }                                                  // kitty/screen.c:2193
    if (self->modes.mDECOM) y -= MAX(y, self->margin_top);
    // 1-based indexing
    int sz = snprintf(buf, sizeof(buf) - 1, "%s%u;%uR", (private ? "?": ""), y + 1, x + 1); // kitty/screen.c:2196
    if (sz > 0) write_escape_code_to_child(self, ESC_CSI, buf);
    break;
```

**Derivation confirmed by the observed cursor.** On the settled 1×1 screen the cursor is `x=2` and `y=0` (from Q1/Q2). Because `x = 2 ≥ columns = 1`, the clamp at `kitty/screen.c:2190-2193` fires; because `y = 0` is **not** `< lines - 1 = 0`, the `else` branch runs `x-- → x = 1`. The `snprintf` at `kitty/screen.c:2196` then formats `y+1 = 1` and `x+1 = 2`, producing **`"1;2R"`**. This is the direct downstream consequence of the earlier grapheme handling: the width-2 final base left the cursor parked one column past a one-wide screen, and the CPR clamp translates that overflow into a report of column 2.

**Byte-exactness.** These bytes are captured from the raw `Callbacks.write` buffer (`wtcbuf`, `kitty_tests/__init__.py:39-164`; the write is at `kitty_tests/__init__.py:50-51`) and quoted unedited — never re-serialised or "cleaned up".

---

## Q4 — How normalization, grapheme breaking & state reporting interact

Pulling the pieces together (mechanism / inference, with citations):

- **Width & combining classification (the "normalization" inputs).** kitty's Unicode classification is generated by `gen/wcwidth.py`: the width function `wcwidth_std` (emitted at `gen/wcwidth.py:509`), `is_emoji_presentation_base` (`gen/wcwidth.py:516`), `is_combining_char` (which folds in marks and default-ignorables, `gen/wcwidth.py:409`), and `is_ignored_char` (Unicode categories `Cc Cs`, `gen/wcwidth.py:416`). The generator emits the width table into `kitty/wcwidth-std.h` and the classification tables into `kitty/unicode-data.c`, both declared in `kitty/unicode-data.h:11-12`. In `draw_text_loop` these decide, per codepoint, whether it is skipped, merged as a combining mark, or written as a width-1/width-2 base.
- **There is no grapheme clustering into a single cell in this version.** Each codepoint is width-classified **individually** and either advances the cursor (width 1 or 2) or is merged into the previous cell (width 0 / combining). ZWJ does not fuse the surrounding emoji into one grapheme cell — it is simply a zero-width combining mark. This is the classic per-codepoint model (see [§ Version context](#version-context)).
- **Cell capacity bounds what can be kept.** A `CPUCell` (`kitty/data-types.h:224-226`) holds one base plus up to three combining marks, so a seven-codepoint joined emoji cannot live in a single 1×1 cell regardless of segmentation. Under autowrap the successive bases overwrite one another and only the last base — with now-empty combining slots — remains.
- **State reporting reads the cursor the draw loop left behind.** `report_device_status` (`kitty/screen.c:2179`) does not re-examine the cell contents; it reports the **cursor**. The width-2 final base left `x = 2` on a one-wide screen, which the CPR clamp (`kitty/screen.c:2190-2193`) turns into the reported column 2 (`ESC [ 1 ; 2 R`).
- **Net composition.** Width classification + combining-merge + autowrap + fixed cell capacity + cursor-state reporting compose to yield exactly two observable facts under the 1×1 constraint: the settled cell is `👦`, and the CPR reply is `ESC [ 1 ; 2 R`. Normalization/classification decides *what each codepoint is*, autowrap + cell capacity decide *what survives*, and state reporting merely echoes *where the cursor ended up* — it does not "see" the discarded graphemes.

---

## 20-column baseline (contrast)

**Observed** (unedited):

```
=== 20-col baseline ===
line(0)= '👨\u200d👩\u200d👧\u200d👦' roundtrip==input: True cursor.x= 8
ESC[6n -> b'\x1b[1;9R' hex= 1b5b313b3952
```

On a 20-column screen the full seven-codepoint sequence **round-trips** — `str(screen.line(0))` equals the input string — and the cursor advances to **`x = 8`**: four width-2 base emoji contribute 8 columns, while the three ZWJs are zero-width and fold into the preceding bases. This cross-checks kitty's own `test_zwj` (`kitty_tests/screen.py:123`), which asserts `s.cursor.x == 8` for the same sequence on a `cols=20` screen.

Here `ESC [ 6 n` returns **`ESC [ 1 ; 9 R`** (`b'\x1b[1;9R'`, hex `1b5b313b3952`): because `x = 8 < columns = 20`, the CPR clamp does **not** fire, so `y+1 = 1` and `x+1 = 9`. The contrast between this (`1;9R`, no clamp, full sequence retained) and the 1×1 result (`1;2R`, clamp fires, only the last base retained) is the crux of Q3 — the *same* code produces both, differing only in whether the width-2 write overflowed the screen.

---

## Determinism

The 1×1 case was run three times with identical input in one process, and the whole probe was additionally re-run as a separate process; all results were identical.

**Observed** (unedited):

```
=== determinism (3 runs) ===
run 1: '👦' x=2 y=0 b'\x1b[1;2R'
run 2: '👦' x=2 y=0 b'\x1b[1;2R'
run 3: '👦' x=2 y=0 b'\x1b[1;2R'
```

The tuple `('👦', x=2, y=0, b'\x1b[1;2R')` is **stable and deterministic** — there is no run-to-run variation.

---

## Sibling-condition coverage (same code paths)

To confirm the mechanism generalises, the same draw/classify/report paths were exercised on adjacent cases (`python3 /tmp/kitty_probe2.py`, unedited):

```
=== widths ===
1F468 2
1F469 2
1F467 2
1F466 2
200D 0
200B 0
200C 0
FE0F 0
FE0E 0
1F1EE 2
1F3FD 0
2764 1
231A 2

=== zero-width joiners X<zw>Y on 5x5 ===
ZWSP 'X\u200bY' cursor.x= 2
ZWNJ 'X\u200cY' cursor.x= 2
ZWJ 'X\u200dY' cursor.x= 2

=== variation selectors ===
HEART w1 + VS16 '❤️' cursor.x= 2
HEART w1 + VS15 '❤︎' cursor.x= 1
WATCH w2 + VS16 '⌚️' cursor.x= 2
WATCH w2 + VS15 '⌚︎' cursor.x= 1

=== regional indicator flag pair 1F1EE 1F1F3 ===
5x5: '🇮🇳' cursor.x= 2
1x1: '🇮🇳' cursor.x= 2 ESC[6n= b'\x1b[1;2R'

=== wide char + backspace ===
after emoji cursor.x= 2
after BS cursor.x= 1 line= '👨'
```

Interpretation (each with its citation):

- **Per-codepoint widths** (observed via `fast_data_types.wcwidth`, i.e. `wcwidth_std`, generated at `gen/wcwidth.py:509`): the four family bases and both regional indicators are width **2**; `U+200D` / `U+200B` / `U+200C` (the zero-width joiners/spaces), `U+FE0F` / `U+FE0E` (variation selectors) and `U+1F3FD` (a skin-tone modifier) are width **0**. (`wcwidth_std` / `is_combining_char` are not exposed as module-level Python; the exposed helpers are `wcwidth` / `wcswidth` / `is_emoji_presentation_base`.)
- **Zero-width joiners** `X<zw>Y` on a 5×5 screen: `U+200B`, `U+200C`, `U+200D` all round-trip and leave `cursor.x = 2` — the zero-width codepoint merges into the preceding `X` cell via `draw_combining_char` (`kitty/screen.c:663`) → `line_add_combining_char` (`kitty/line.c:457`). This is the same combining-merge that swallowed the ZWJ in the family sequence.
- **Variation selectors** (special-cased inside `draw_combining_char`, `kitty/screen.c:663` — a VS16 `0xfe0f` "widen" branch and a VS15 `0xfe0e` "narrow" branch): `U+2764` HEART (base width 1) **widens to 2** with VS16 (`❤️`, `cursor.x=2`) and stays 1 with VS15 (`❤︎`, `cursor.x=1`); `U+231A` WATCH (base width 2) **narrows to 1** with VS15 (`cursor.x=1`) and stays 2 with VS16 (`cursor.x=2`). Both codepoints are retained in the cell — a variation selector is a combining mark that also adjusts the base cell's width.
- **Regional-indicator flag pair** `U+1F1EE U+1F1F3` (handled via `is_flag_pair`, `kitty/screen.c:633`, and `move_widened_char`, `kitty/screen.c:575`): on 5×5 it round-trips to `'🇮🇳'` with `cursor.x = 2`; forced into a **1×1** screen it retains both regional indicators and `ESC [ 6 n` returns `ESC [ 1 ; 2 R` — the same clamped column-2 report as the family case, for the same reason (a width-2 result parks the cursor past a one-wide screen).
- **Wide char + backspace** (cf. `test_backspace_wide_characters`, `kitty_tests/screen.py:271`): after drawing `U+1F468` the cursor is at `x = 2`; a single `BS` (`0x08`) moves it to `x = 1` (onto the wide-char trailer) while the cell text is unchanged (`'👨'`).
- **`ESC [ 5 n`** on the 1×1 screen after the family emoji returns `b'\x1b[0n'` — the non-CPR device-status branch (`case 5:`, `kitty/screen.c:2186`), reported for completeness.
- These also corroborate kitty's `test_emoji_skin_tone_modifiers` (`kitty_tests/screen.py:105`) and `test_zwj` (`kitty_tests/screen.py:123`), which cover the same classification/merge behaviour on larger screens.

---

## Version context

This section is **external corroboration** (web search + kitty's own docs/changelog), kept distinct from the observed runtime behavior above. It establishes that the results belong to the **classic per-codepoint model** and must not be read against a newer kitty.

- **This commit is June 2024.** `git show -s --format=%ci HEAD` reports `2024-06-24`; the changelog dates the release at `docs/changelog.rst:59`: `0.35.2 [2024-06-22]`.
- **It predates kitty's text-sizing / grapheme-segmentation work.**
  - GitHub **#8226** — the text-sizing protocol ("display text in different sizes") — is 2025 work that introduced the ability for clients to declare explicit cell widths.
  - GitHub **#8533** — the RFC "Specifying how terminals process Unicode text" — is dated **April 2025** and follows on from #8226. It specifies full grapheme segmentation based on the Unicode standard's rules and explicitly warns that for ZWJ-based emoji "the width kitty assigns to these has changed" under the new algorithm. kitty's changelog records the behavior change: *"Now kitty does full grapheme segmentation following the Unicode 16 spec when splitting text into cells (#8533)."* (shipped in a later 0.4x release, 2025).
- **Therefore this build uses the classic per-codepoint width model:** each codepoint's width comes from `wcwidth_std`, `U+200D` is zero-width/combining, and text is split into cells **without** grapheme clustering. This is exactly what the runtime shows: 8-column cursor advance for the family sequence on a wide screen, and single-base survival in the 1×1 cell.
- **The newer model does not apply to this commit.** The later text-sizing protocol adds a rule that "if the multicell block … is larger than the screen size in either dimension, the terminal must discard the character," together with a refactored multicell cell model (a `ch_or_idx` / `is_multicell` text-cache design). Those are **post-0.35.2** refactors. For this commit the authoritative cell model is the classic `CPUCell` at `kitty/data-types.h:224-226`. If a newer kitty were used, the settled 1×1 cell and the CPR reply could differ; the results here are specific to 0.35.2.

---

## Repository integrity & cleanup proof

- The two temporary probes lived at `/tmp/kitty_probe.py` and `/tmp/kitty_probe2.py` — **outside** the repository — and were removed after use.
- `git status --porcelain` shows **no modified or deleted tracked files**; the only new path is the `blitzy/` tree containing this document.
- `git check-ignore kitty/fast_data_types.so build` confirms both build artifacts are **gitignored**, so building the native extension leaves no tracked changes.
- **This answer document under `blitzy/documentation/` is the only addition.** No existing source file was modified, added to, or deleted — consistent with the read-only mandate.

---

## Appendix — the probe scripts (for reproduction)

Both scripts were created under `/tmp` (outside the repository) and removed after use. Run from the repository root: `cd <repo> && python3 /tmp/kitty_probe.py`.

**`/tmp/kitty_probe.py`** — primary case (1×1 progression, settled cell, `ESC[6n`/`ESC[5n`, 20-column baseline, determinism):

```python
import sys
sys.path.insert(0, '.')
from kitty.fast_data_types import Screen, set_options
from kitty.options.types import Options

class Callbacks:
    def __init__(self):
        self.wtcbuf = b''
    def write(self, data):
        self.wtcbuf += bytes(data)
    def __getattr__(self, name):
        return lambda *a, **k: None

def parse_bytes(screen, data):
    data = memoryview(data)
    while data:
        dest = screen.test_create_write_buffer()
        s = screen.test_commit_write_buffer(data, dest)
        data = data[s:]
        screen.test_parse_written_data(None)

set_options(Options())

def make_screen(cols, lines):
    c = Callbacks()
    s = Screen(c, lines, cols, 0, 10, 20, 0, c)
    return s, c

family = '\U0001F468\u200d\U0001F469\u200d\U0001F467\u200d\U0001F466'
print('family codepoints:', [hex(ord(ch)) for ch in family])

print('\n=== 1x1 progression ===')
s, c = make_screen(1, 1)
print("before:", repr(str(s.line(0))), "cursor=(%d,%d)" % (s.cursor.x, s.cursor.y))
for ch in family:
    parse_bytes(s, ch.encode('utf-8'))
    print("after %-8s:" % hex(ord(ch)), repr(str(s.line(0))), "cursor=(%d,%d)" % (s.cursor.x, s.cursor.y))
settled = str(s.line(0))
print("SETTLED line(0)=", repr(settled), "codepoints=", [hex(ord(x)) for x in settled], "cursor=(x=%d,y=%d)" % (s.cursor.x, s.cursor.y))

c.wtcbuf = b''
parse_bytes(s, b'\x1b[6n')
print("ESC[6n ->", repr(c.wtcbuf), "hex=", c.wtcbuf.hex())
c.wtcbuf = b''
parse_bytes(s, b'\x1b[5n')
print("ESC[5n ->", repr(c.wtcbuf), "hex=", c.wtcbuf.hex())

print('\n=== 20-col baseline ===')
s2, c2 = make_screen(20, 1)
parse_bytes(s2, family.encode('utf-8'))
line0 = str(s2.line(0))
print("line(0)=", repr(line0), "roundtrip==input:", line0 == family, "cursor.x=", s2.cursor.x)
c2.wtcbuf = b''
parse_bytes(s2, b'\x1b[6n')
print("ESC[6n ->", repr(c2.wtcbuf), "hex=", c2.wtcbuf.hex())

print('\n=== determinism (3 runs) ===')
for i in range(3):
    s3, c3 = make_screen(1, 1)
    parse_bytes(s3, family.encode('utf-8'))
    c3.wtcbuf = b''
    parse_bytes(s3, b'\x1b[6n')
    print("run %d:" % (i+1), repr(str(s3.line(0))), "x=%d y=%d" % (s3.cursor.x, s3.cursor.y), repr(c3.wtcbuf))
```

**`/tmp/kitty_probe2.py`** — sibling conditions (widths, zero-width joiners, variation selectors, flag pair, wide+backspace):

```python
import sys
sys.path.insert(0, '.')
from kitty.fast_data_types import Screen, set_options, wcwidth
from kitty.options.types import Options
class Callbacks:
    def __init__(self): self.wtcbuf = b''
    def write(self, data): self.wtcbuf += bytes(data)
    def __getattr__(self, name): return lambda *a, **k: None
def parse_bytes(screen, data):
    data = memoryview(data)
    while data:
        dest = screen.test_create_write_buffer()
        s = screen.test_commit_write_buffer(data, dest)
        data = data[s:]
        screen.test_parse_written_data(None)
set_options(Options())
def mk(cols, lines):
    c = Callbacks(); return Screen(c, lines, cols, 0, 10, 20, 0, c), c

print("=== widths ===")
for name,cp in [('1F468',0x1F468),('1F469',0x1F469),('1F467',0x1F467),('1F466',0x1F466),('200D',0x200D),('200B',0x200B),('200C',0x200C),('FE0F',0xFE0F),('FE0E',0xFE0E),('1F1EE',0x1F1EE),('1F3FD',0x1F3FD),('2764',0x2764),('231A',0x231A)]:
    print(name, wcwidth(cp))

print("\n=== zero-width joiners X<zw>Y on 5x5 ===")
for zw,label in [(0x200B,'ZWSP'),(0x200C,'ZWNJ'),(0x200D,'ZWJ')]:
    s,c=mk(5,5)
    txt='X'+chr(zw)+'Y'
    parse_bytes(s, txt.encode('utf-8'))
    print(label, repr(str(s.line(0))), 'cursor.x=',s.cursor.x)

print("\n=== variation selectors ===")
for base,bn in [(0x2764,'HEART w1'),(0x231A,'WATCH w2')]:
    for vs,vn in [(0xFE0F,'VS16'),(0xFE0E,'VS15')]:
        s,c=mk(5,5)
        parse_bytes(s,(chr(base)+chr(vs)).encode('utf-8'))
        print(bn,'+',vn, repr(str(s.line(0))), 'cursor.x=',s.cursor.x)

print("\n=== regional indicator flag pair 1F1EE 1F1F3 ===")
s,c=mk(5,5); parse_bytes(s,'\U0001F1EE\U0001F1F3'.encode('utf-8'))
print('5x5:', repr(str(s.line(0))),'cursor.x=',s.cursor.x)
s,c=mk(1,1); parse_bytes(s,'\U0001F1EE\U0001F1F3'.encode('utf-8'))
c.wtcbuf=b''; parse_bytes(s,b'\x1b[6n')
print('1x1:', repr(str(s.line(0))),'cursor.x=',s.cursor.x,'ESC[6n=',repr(c.wtcbuf))

print("\n=== wide char + backspace ===")
s,c=mk(5,5); parse_bytes(s,'\U0001F468'.encode('utf-8')); print('after emoji cursor.x=',s.cursor.x)
parse_bytes(s,b'\x08'); print('after BS cursor.x=',s.cursor.x,'line=',repr(str(s.line(0))))
```

---

### Citation quick-reference (verified against the on-disk source at commit `815df1e2`)

- **Draw & wrap path (`kitty/screen.c`):** `draw_text_loop` L763; default `char_width = 1` L803; non-ASCII branch `if (ch > DEL)` L804; `is_ignored_char` skip L805; `is_combining_char` L806; combining route `draw_combining_char(...); continue;` L810; `wcwidth_std(ch)` L814; autowrap test L821; DECAWM guard L822; wrap calls `continue_to_next_line` / `init_text_loop_line` L823-824; width-2 write + trailer zero L835-843; `draw_combining_char` L663 (VS16 `0xfe0f` widen / VS15 `0xfe0e` narrow branches); `move_widened_char` L575; `is_flag_pair` L633; `screen_draw_text` L866; `draw()` Python binding L3769; `report_device_status` L2179; device-status `case 5` "0n" L2186; CPR `case 6` L2188; clamp L2190-2193; `snprintf "%s%u;%uR"` L2196.
- **Parser (`kitty/vt-parser.c`):** `dispatch_csi` L1027; DSR case L1172; `CALL_CSI_HANDLER1P(report_device_status, 0, '?')` L1173.
- **Cell model (`kitty/data-types.h`):** `char_type ch` L224; `hyperlink_id` L225; `combining_type cc_idx[3]` L226; `} CPUCell;` L227; `static_assert(sizeof(CPUCell) == 12)` L228.
- **Combining attach (`kitty/line.c` / `kitty/lineops.h`):** `line_add_combining_char` L457; declaration L90.
- **Classification (`kitty/unicode-data.h`; generated into `kitty/unicode-data.c` and `kitty/wcwidth-std.h`):** `is_combining_char` decl L11; `is_ignored_char` decl L12.
- **Generators (`gen/wcwidth.py`):** `is_combining_char` L409; `is_ignored_char` (`Cc Cs`) L416; `wcwidth_std` L509; `is_emoji_presentation_base` L516.
- **Harness (`kitty_tests/__init__.py`):** `parse_bytes` L30; `Callbacks` L39 (`write` L50-51, `clear` L95-96); `create_screen` L237 (`Screen(...)` L240); `str(screen.line(i))` L407.
- **Cross-check tests (`kitty_tests/screen.py`):** `test_emoji_skin_tone_modifiers` L105; `test_zwj` L123; `test_backspace_wide_characters` L271.
- **Build (`setup.py`):** `def build` L1084; `--ignore-compiler-warnings` L2003; `werror` toggle L491.
