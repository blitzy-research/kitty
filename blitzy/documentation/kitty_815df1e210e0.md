# kitty — Startup Text‑Shaping / Layout Engine & GPU Glyph‑Atlas Configuration (Runtime‑Observed)

**Repository / branch:** `kovidgoyal/kitty` @ `kitty_815df1e210e0` · **kitty 0.35.2** (VCS `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)

This document answers four questions about how kitty configures its text‑shaping/layout
engine and its GPU glyph atlas **during startup**. Every behavioral claim below is grounded in
**real output captured at runtime** from a canonical build of kitty, paired with the exact command
that produced it and a `file:line` citation into the source. It is a **read‑only investigation**:
no source file was modified; all observation scripts/fixtures were temporary and removed on
completion (see §7).

The four questions:

- **Q1** — How the shaping/layout engine configures support for complex Unicode — *"ligatures, bidi, combining diacritics"* — and font fallback at startup.
- **Q2** — The exact font families and fallback chains that appear in verbose startup diagnostics for *"mixed Arabic (RTL) and English (LTR) text"*, plus the runtime configuration values confirming these selections before any text rendering begins.
- **Q3** — The default cell metrics, baseline positioning, and decoration alignment — *"(overline/underline)"* — shown in debug output.
- **Q4** — The initial GPU texture‑atlas page layout, sizing, and capacity for shaped glyphs across primary and fallback fonts, plus the startup logs that verify the atlas is ready.

---

## 0. Methodology, evidence labels, and canonicality

This investigation followed an **observe‑first** methodology: build kitty, run the relevant code
paths through the real entry point, capture the real output, and only then write. Every value was
confirmed **stable across at least two identical runs**.

**Evidence labels used throughout:**

- **[OBSERVED — canonical]** — captured from the real, user‑facing path: the built
  `kitty/launcher/kitty` binary with CLI flags, or text fed through the real PTY/input path, or a
  live OpenGL call intercepted non‑invasively at runtime.
- **[OBSERVED — test‑API]** — captured by exercising kitty's in‑repo `test_*` / `kitty.fonts.render`
  introspection API. This runs the **real** shaping / font‑selection / metric / sprite‑tracker C
  code, but it **bypasses the PTY/startup path** and **mocks the GPU upload**
  (`set_send_sprite_to_gpu`). Such values are real computations but are **non‑canonical relative to a
  live run** and are labeled as such.
- **[INFERRED — code‑derived]** — a statement derived from reading the source, not from a runtime
  observation (used only where no runtime surface exposes the value, and always labeled).

**Environment (canonical).** All build/run/observation was performed inside the user‑specified
Docker container (the host lacks `pkg-config`/`go`/dev‑headers and a matching Python, so it cannot
build or load kitty's compiled extension):

- Image: `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
- From: `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
- OS **Ubuntu 24.04**, Python **3.12.3**, gcc **13.3.0**, Go **1.23.4**.
- Headless GPU: `LIBGL_ALWAYS_SOFTWARE=1` + `xvfb-run` → Mesa **llvmpipe**, OpenGL **4.5** (≥ the 3.3 kitty requires).
- Required locale for runs: `LANG=C.UTF-8 LC_ALL=C.UTF-8`.

**Linux is the canonical exercised path.** Font discovery and the `identify_for_debug`
implementation are platform‑split: FreeType/FontConfig on Linux
[`kitty/freetype.c`:L738] vs CoreText on macOS [`kitty/core_text.m`:L966]. On this Linux
container the **FreeType/FontConfig** path is canonical; the **macOS CoreText** path
(`kitty/core_text.m`, `kitty/fonts/core_text.py`) is the documented equivalent and was **not run**.

**How font debug output is gated (verified).** The rich shaping/fallback trace only appears when
`--debug-font-fallback` is set. In C this is the macro
`#define debug_fonts(...) if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); }`
[`kitty/state.h`:L16], which `fonts.c` aliases via `#define debug debug_fonts`
[`kitty/fonts.c`:L18]. The flag is stored by `set_options` into `global_state.debug_font_fallback`
[`kitty/state.c`:L740]; the wiring is `set_options(...)` at [`kitty/main.py`:L249] and
[`kitty/boss.py`:L2649]. The startup `dump_font_debug()` call is guarded by
`args.debug_font_fallback` at [`kitty/main.py`:L228-L229].

---

## 1. Build & invocation commands used

**Canonical build command (verbatim):** `python3 setup.py` — the `Makefile` `all:` target runs
exactly this [`Makefile`:L12-L13]. It produces the launcher at **`kitty/launcher/kitty`** and the
compiled CPython extension **`kitty/fast_data_types.so`** (the checkout ships only C sources under
`kitty/launcher/` and a `.pyi` stub for `fast_data_types`, so the presence of these artifacts
confirms the build produced them).

Command and captured output (head), **[OBSERVED — canonical]**:

```
$ python3 setup.py --verbose
CC: ['gcc'] (13, 0)
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Detected: CompilerType.gcc
Updating Go generated files...
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten .../tools/cmd
```

The `VCSRevision` stamped into the build (`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`) matches the
branch `kitty_815df1e210e0`. Produced artifacts (`ls -l`):

```
kitty/fast_data_types.so   1213072 bytes
kitty/launcher/kitty         36224 bytes
```

**Version banner [OBSERVED — canonical]:**

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**Dependency versions confirmed in the container** (read‑only inventory; nothing added/updated/removed).
`pkg-config --modversion` **[OBSERVED — canonical]**:

| Package | Version | Requirement / cite |
|---|---|---|
| harfbuzz | **8.3.0** | `>= 1.5` required [`setup.py`:L609] ✓ |
| freetype2 | **26.1.20** (libtool iface; FreeType release 2.13.2) | [`setup.py`:L610] |
| fontconfig | **2.15.0** | Linux font discovery [`kitty/fontconfig.c`] |
| libpng | **1.6.43** | [`setup.py`:L610] |
| lcms2 | **2.14** | [`setup.py`:L611] |
| OpenGL (Mesa llvmpipe) | **4.5** (Compatibility/Core) | GL 3.3 runtime minimum |

**Canonical observation invocations used** (defined at [`kitty/cli.py`:L989-L993] for
`--debug-rendering`/`--debug-gl`, and [`kitty/cli.py`:L1002-L1005] for `--debug-font-fallback`):

```
./kitty/launcher/kitty --debug-font-fallback sh -c true
./kitty/launcher/kitty --debug-gl        -o close_on_child_death=yes sh -c '...'
./kitty/launcher/kitty --debug-rendering -o close_on_child_death=yes sh -c '...'
```

All GUI/rendering invocations were wrapped as:
`LANG=C.UTF-8 LC_ALL=C.UTF-8 LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s '-screen 0 1280x800x24' ...`.

**Default configuration only.** No custom `kitty.conf` was used. Because `--debug-config` echoes
**only non‑default options** (§2), the relevant defaults are asserted directly from source:
`font_family = monospace` [`kitty/options/definition.py`:L35], `font_size = 11.0`
[`kitty/options/definition.py`:L59], `force_ltr = no`
[`kitty/options/definition.py`:L64], `disable_ligatures = never`
[`kitty/options/definition.py`:L115].

---

## 2. Q1 — Complex‑Unicode shaping + font‑fallback configuration at startup

> The question names four items explicitly: **ligatures, bidi, combining diacritics**, and **font
> fallback**. User example, verbatim: *"ligatures, bidi, combining diacritics"*.

At startup, kitty selects fonts and builds its shaping pipeline around **HarfBuzz**. The single
buffer‑construction function that configures complex‑Unicode support per shaping run is
`load_hb_buffer` [`kitty/fonts.c`:L672-L690]; grapheme/ligature grouping happens in
`shape`/`group_state` [`kitty/fonts.c`:L776,L786]; startup font selection is `set_font_family`
[`kitty/fonts/render.py`:L172-L193] → `get_font_files` → FontConfig/FreeType.

### 2.1 Font selection at startup

`set_font_family` [`kitty/fonts/render.py`:L172-L193] resolves the default `font_family=monospace`
to concrete faces. On this container the primary face is **DejaVu Sans Mono**
(`fc-match monospace` → `DejaVuSansMono.ttf: "DejaVu Sans Mono" "Book"`) **[OBSERVED — canonical]**.
The full resolved set is shown in §3.

### 2.2 Combining diacritics — `codepoint_for_mark` accumulation

`load_hb_buffer` adds each cell's base character and then **accumulates its combining marks** into
the same shaping run via `codepoint_for_mark(first_cpu_cell->cc_idx[i])`
[`kitty/fonts.c`:L682-L685], so a base + combining‑mark sequence is fed as **multiple codepoints**
into one shaping run.

**[OBSERVED — test‑API]** via `kitty.fonts.render.shape_string` / `test_shape`
[`kitty/fonts/render.py`:L454; `kitty/fonts.c`:L1226] (each group tuple is
`(num_cells, num_glyphs, first_glyph, (glyph_ids,))`), DejaVu Sans Mono @ 11.0pt, **stable ×2**:

```
# base 'e' + U+0301 COMBINING ACUTE  (composing)
shape_string("e\u0301")  -> [(1, 1, 171)]
    # HarfBuzz ccmp/GSUB composes base+acute into ONE precomposed glyph 171 -> 1 cell, 1 glyph

# base 'e' + U+0347 + U+0305         (non-composing marks)
shape_string("e\u0347\u0305") -> [(1, 3, 72, (72, 0, 653))]
    # ONE cell, THREE glyphs: base e=72, U+0347=0 (.notdef; not in DejaVu), U+0305=653

# kitty's own test string "He\u0347\u0305llo"
shape_string("He\u0347\u0305llo") -> [(1,1,43), (1,3,72,(72,0,653)), (1,1,79), (1,1,79), (1,1,82)]
    # matches kitty_tests/fonts.py test_shaping expectation of the (1,3) group for e+2 marks
```

The `(1, 3, …)` group is the direct observable signature of `codepoint_for_mark` feeding the base
plus two combining marks into a single cell's shaping run.

### 2.3 Ligatures — `calt` and `group_state` grouping

Ligature grouping is performed by `shape(...)` via `group_state` [`kitty/fonts.c`:L776,L786];
`disable_ligatures` defaults to `never` [`kitty/options/definition.py`:L115], i.e. the OpenType
`calt` feature is **enabled**. Grouping is font‑dependent (it only occurs if the face actually
carries the ligatures).

**[OBSERVED — test‑API]**, **stable ×2**:

```
# DejaVu Sans Mono (the default monospace face; NO programming-ligature calt):
shape_string("A===B!=C")
  -> [(1,1,36), (1,1,32), (1,1,32), (1,1,32), (1,1,37), (1,1,4), (1,1,32), (1,1,38)]
     # 8 separate single-cell groups -> no ligature grouping
shape_string("----") -> [(1,1,16), (1,1,16), (1,1,16), (1,1,16)]

# Fira Code (a calt-carrying face), same inputs:
shape_string("A===B!=C", family=<FiraCode-Medium.otf>)
  -> [(1,1,4,(4,)), (3,3,1289,(1289,1289,1682)), (1,1,16,(16,)), (2,2,1023,(1023,1114)), (1,1,17,(17,))]
     # '===' grouped into ONE 3-cell group; '!=' grouped into ONE 2-cell group
shape_string("----", family=<FiraCode-Medium.otf>)
  -> [(4,4,1142,(1142,1141,1141,1143))]
     # a single 4-cell ligature group
```

So the *mechanism* (multiple source cells collapsed into one `(N,N,…)` shaping group) is what
`disable_ligatures=never` + `calt` enables; whether it fires depends on the selected face.

### 2.4 bidi — HarfBuzz segment‑property inference + `force_ltr`

`load_hb_buffer` calls `hb_buffer_guess_segment_properties(harfbuzz_buffer)`
[`kitty/fonts.c`:L687] to **infer** direction/script/language from the buffer's Unicode contents,
then applies the override `if (OPT(force_ltr)) hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR);`
[`kitty/fonts.c`:L688].

- **BEFORE `guess_segment_properties`:** the buffer's segment properties are unset (direction
  `HB_DIRECTION_INVALID`) — **[INFERRED — code‑derived]** from the HarfBuzz contract (this internal
  buffer state is not exposed to Python).
- **DURING / AFTER `guess_segment_properties`:** direction/script are inferred **per run** — an
  Arabic run → RTL / `Arab`, an English run → LTR / `Latn`. The **effect** is directly observable in
  the shaped output: Arabic receives contextual positional forms (see below), which only occurs if
  HarfBuzz inferred the Arabic script and RTL direction. **[OBSERVED — test‑API]**:

```
# Arabic contextual shaping (proves Arab-script + RTL inference):
shape_string("\u0645")      # isolated MEEM  -> [(1,1,1142)]
shape_string("\u0631")      # isolated REH   -> [(1,1,1127)]
shape_string("\u0645\u0631\u062d\u0628\u0627")  # the word مرحبا
  -> glyphs (3145, 3149, 3166, 3177, 3230)   # init/medial/final positional forms via GSUB
```

- **The `force_ltr` override does NOT fire.** `force_ltr` defaults to `no`
  [`kitty/options/definition.py`:L64]; this was confirmed canonically by the **empty** "Config
  options different from defaults" section of the in‑app debug‑config dump (§3.3) **[OBSERVED —
  canonical]**. Therefore HarfBuzz keeps its inferred RTL direction for Arabic.

**bidi policy — no word reordering (observed).** kitty does **not** implement a full Unicode
Bidirectional Algorithm; it relies on per‑run HarfBuzz shaping and stores cells in **logical
(input) order**. Fed *"mixed Arabic (RTL) and English (LTR) text"* through the real grid
**[OBSERVED — test‑API, real `Screen`]**:

```
s.draw("English \u0645\u0631\u062d\u0628\u0627 world")   # "English مرحبا world"
line.as_ansi() -> "English \u0645\u0631\u062d\u0628\u0627 world"   # IDENTICAL to input
s.cursor.x     -> 19                 # 8 ("English ") + 5 (مرحبا) + 6 (" world")
```

The line content round‑trips in the original logical order — kitty does **not reorder words**.
(The narrower statement that kitty "reverses characters within a word for RTL but does not reorder
words" is kitty's documented policy, corroborated by [`kitty/fonts.c`:L687-L688]; the pieces
**observed** here are: (a) logical grid order preserved, (b) Arabic RTL contextual shaping applied,
(c) `force_ltr=no`.) — **[INFERRED — code‑derived]** for the policy statement, **[OBSERVED]** for
(a)–(c).

### 2.5 Font fallback at startup

Fallback is exercised when the primary face lacks a glyph. **Honest, environment‑specific finding:**
on this container the primary **DejaVu Sans Mono covers both English *and* Arabic**, so Arabic does
**not** trigger the fallback chain here. Proven by shaping against the primary (glyph `0` = `.notdef`
= uncovered) **[OBSERVED — test‑API]**:

```
English 'a'        -> glyph 68     (covered)
Arabic MEEM U+0645 -> glyph 1142   (covered)
Arabic REH  U+0631 -> glyph 1127   (covered)
CJK   U+4F60       -> glyph 0      (NOT covered)
emoji U+1F601      -> glyph 0      (NOT covered)
```

The fallback path **is** exercised canonically for characters the primary lacks — see §3.2 for the
real `--debug-font-fallback` per‑character output (emoji → DejaVu Sans; CJK → no font available).


---

## 3. Q2 — Exact font families / fallback chains + pre‑render configuration for mixed Arabic/English

> User example, verbatim: *"mixed Arabic (RTL) and English (LTR) text"*.

### 3.1 Startup font families (`--debug-font-fallback` → `dump_font_debug()`)

Running the launcher with `--debug-font-fallback` triggers `dump_font_debug()`
[`kitty/fonts/render.py`:L161-L171], which logs `Text fonts:` followed by
`Normal`/`Bold`/`Italic`/`Bold-Italic` lines (and a `Symbol map fonts:` section if present). On
Linux each line is produced by FreeType `identify_for_debug()` with the format `"%s: %V:%d"` =
`PostScriptName: <path>:<face_index>` [`kitty/freetype.c`:L738].

**[OBSERVED — canonical], stable ×2:**

```
$ LANG=C.UTF-8 LC_ALL=C.UTF-8 LIBGL_ALWAYS_SOFTWARE=1 \
    xvfb-run -a -s '-screen 0 1280x800x24' ./kitty/launcher/kitty --debug-font-fallback sh -c true
Text fonts:
  Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

So the **primary (`monospace`) family is DejaVu Sans Mono**, with its Bold/Oblique/BoldOblique
siblings for the bold/italic/bold‑italic slots.

### 3.2 Per‑character fallback chain for mixed Arabic/English (real PTY)

A **mixed Arabic (RTL) + English (LTR)** fixture that also contains a combining diacritic and a
ligature candidate was fed through the **real PTY/input path** of the built launcher (not remote
control). Because DejaVu Sans Mono covers English *and* Arabic (§2.5), those characters resolve to
the **primary** and emit no fallback line; characters the primary lacks drive the fallback path via
`output_cell_fallback_data` [`kitty/fonts.c`:L456-L468] and the "does not actually contain glyphs"
notice [`kitty/fonts.c`:L503-L509].

**[OBSERVED — canonical], stable ×2** (fixture: `English مرحبا emoji 😁 cjk 你好 done`):

```
$ ./kitty/launcher/kitty --debug-font-fallback sh -c "cat <fixture>; sleep 6"
...
U+1f601 emoji_presentation Face(family=DejaVu Sans ... ps_name=DejaVuSans path=.../DejaVuSans.ttf ...)
U+4f60 Face(family=DejaVu Sans Mono ... ps_name=DejaVuSansMono ...)
The font chosen by the OS for the text: U+4f60 is Face(...DejaVuSansMono...) but it does not actually contain glyphs for that text
U+597d ... (same "does not actually contain glyphs")
```

Interpretation, mapped to the named items:

- **English (LTR)** → the **primary** `monospace` = **DejaVu Sans Mono** (no fallback line emitted).
- **Arabic (RTL)** → also served by the **primary DejaVu Sans Mono** in this container (it covers
  Arabic), with real contextual shaping (§2.4). No separate Arabic fallback face was needed here
  because no Arabic‑specific font is installed and the primary already covers the range.
- **emoji** → falls back to **DejaVu Sans** (`ps_name=DejaVuSans`) — a genuine fallback selection.
- **CJK** → **no** installed font covers it; the fallback resolver returns the primary and reports
  it "does not actually contain glyphs" (notdef).

**[OBSERVED — test‑API] supplements** (`kitty.fast_data_types`): after rendering emoji,
`current_fonts()['fallback']` = `(DejaVuSans,)` (chain length 1); `current_fonts()['medium']` =
DejaVuSansMono (primary). `get_fallback_font(text, False, False)` [`kitty/fast_data_types.pyi`:L1071]
(single‑grapheme contract): MEEM → DejaVuSansMono (the primary itself), emoji U+1F601 → DejaVuSans,
CJK U+4F60 → `ValueError: No fallback font found`.

**Platform note.** These selections come from the **Linux FreeType/FontConfig** path
[`kitty/freetype.c`:L738] — the canonical exercised path on this container. The **macOS CoreText**
equivalent [`kitty/core_text.m`:L966] was **not run**.

### 3.3 Pre‑render configuration ("before any text rendering begins")

kitty's effective configuration is surfaced by the in‑app **debug‑config** action
`@ac('debug', 'Show the effective configuration kitty is running with')` [`kitty/boss.py`], which
calls `debug_config(get_options())` and prints the `OpenGL:` line via `opengl_version_string()`
[`kitty/debug_config.py`:L258] and the `Fonts:` block iterating `current_fonts()` via
`identify_for_debug()` [`kitty/debug_config.py`:L260-L263].

> **Honest finding:** in kitty **0.35.2**, `--debug-config` is **not** a startup CLI flag
> (`./kitty/launcher/kitty --debug-config` → `Unknown option: --debug-config`, exit 1). The
> canonical surface is the in‑app action, bound by default to `kitty_mod+f6`
> (`kitty_mod` = `ctrl+shift`) [`kitty/options/definition.py`:L4256].

To capture it **through the real path** (not remote control, not a mock), a real key event was
delivered to a live kitty window under Xvfb (window focused with `xdotool`, then
`xdotool key --clearmodifiers ctrl+shift+F6`; the action copies its output to the clipboard).
**[OBSERVED — canonical], stable ×3** (key lines):

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux 4e08812d9fb8 6.6.122+ ... x86_64
Ubuntu 24.04.2 LTS ...
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths: kitty: .../kitty/launcher/kitty ; ...
Config options different from defaults:

```

Two decisive pre‑render facts:

1. **`OpenGL: '4.5 (Core Profile) Mesa 25.2.8'`** confirms a **live** GL context is created — this is
   the context the atlas (§5) is allocated in.
2. **"Config options different from defaults:" is EMPTY** — canonical proof that the run is in
   **default configuration**, so the asserted defaults hold *before any text rendering begins*:
   `font_family=monospace` [`kitty/options/definition.py`:L35], `font_size=11.0`
   [`kitty/options/definition.py`:L59], `force_ltr=no` [`kitty/options/definition.py`:L64],
   `disable_ligatures=never` [`kitty/options/definition.py`:L115]. (These do not appear in the dump
   *because* they are defaults; only non‑default options are echoed.)


---

## 4. Q3 — Cell metrics, baseline positioning, and decoration alignment

> User example, verbatim: *"(overline/underline)"*.

Cell metrics are computed by `calc_cell_metrics` [`kitty/fonts.c`:L373-L421], which calls
`cell_metrics(...)` on the selected medium face [`kitty/freetype.c`:L387] to produce seven values,
then stores them into the `FontGroup` [`kitty/fonts.c`:L420; struct fields `kitty/state.h`:L101].

### 4.1 The seven metric values (default `monospace`)

The full seven‑value set is **not** exposed to Python by name and is **not** auto‑printed by any
default debug flag (the FreeType `Face` object exposes only `postscript_name`/`identify_for_debug`/
`path`/`has_color`, etc. — no `baseline`/`ascender` attribute). The canonical surface used here is
the **real C→Python `prerender_function` boundary**: during genuine font setup, kitty passes all
seven C‑computed metrics into the Python prerender callback
`prerender_function(cell_width, cell_height, baseline, underline_position, underline_thickness,
strikethrough_position, strikethrough_thickness, ...)`. Intercepting that callback captures the
real `calc_cell_metrics` output.

**[OBSERVED — test‑API]** (DejaVu Sans Mono @ 11.0pt / 96 dpi), **stable ×2**; `cell_width`/`cell_height`
independently corroborated by `create_test_font_group()`'s return value:

```
selected medium face : DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
font_sz_in_pts       : 11.0   dpi: 96.0 96.0
create_test_font_group() -> (cell_width, cell_height) = (9, 18)
--- intercepted at prerender_function (the real calc_cell_metrics output) ---
  cell_width               = 9
  cell_height              = 18
  baseline                 = 14
  underline_position       = 15
  underline_thickness      = 1
  strikethrough_position   = 10
  strikethrough_thickness  = 1
```

| Metric | Value | Notes |
|---|---|---|
| `cell_width` | **9** px | also observable in the window‑size banner `"N x M cells" = width/cell_width` [`kitty/child-monitor.c`:L778] |
| `cell_height` | **18** px | likewise `height/cell_height` |
| `baseline` | **14** px | distance from cell top to the text baseline |
| `underline_position` | **15** px | below the baseline (larger y = lower); clamped `MIN(cell_height-1, …)` |
| `underline_thickness` | **1** px | |
| `strikethrough_position` | **10** px | |
| `strikethrough_thickness` | **1** px | |

**Canonicality note:** these are the **real** `calc_cell_metrics` numbers (the C computation runs
unchanged); the surface is labeled test‑API because it is read at the prerender callback rather than
through the PTY. `cell_width=9`/`cell_height=18` agree across two independent surfaces
(`create_test_font_group` return and the prerender callback), and all seven values reproduced
identically across runs.

### 4.2 Clamps and baseline‑driven re‑adjustment (boundary behavior)

**[INFERRED — code‑derived]** from `calc_cell_metrics` [`kitty/fonts.c`:L381-L416]; with default
config these paths are inert (no `modify_font`), so the §4.1 values are the raw FreeType‑derived
metrics:

- Dimension clamps `MAX_DIM=1000`, `MIN_WIDTH=2`, `MIN_HEIGHT=4` [`kitty/fonts.c`:L381-L383];
  `cell_width`/`cell_height` are validated after any `modify_font` adjustment (`fatal()` if out of
  range).
- If `modify_font` changes the baseline, `underline_position` and `strikethrough_position` are
  re‑adjusted by the same delta via `adjust_ypos()` [`kitty/fonts.c`:L401-L407].
- `underline_position = MIN(cell_height - 1, underline_position)` [`kitty/fonts.c`:L409].
- With default config all `adjust_metric()` calls are no‑ops (`OPT(cell_width).val` etc. are `0`) and
  `baseline_before == baseline`, so no re‑adjustment fires.

### 4.3 Decoration alignment — *"(overline/underline)"* disambiguation (required)

This is the crux of the verbatim example. The three line decorations must **not** be conflated:

- **underline** — a **cell‑level attribute** `CellAttrs.decoration` (a 3‑bit field)
  [`kitty/data-types.h`:L199], set by the SGR parser: SGR 4 = single (or a substyle `MIN(5, param)`),
  SGR 21 = double (`decoration=2`), SGR 24 = off (`decoration=0`) [`kitty/cursor.c`:L92-L111];
  `NUM_UNDERLINE_STYLES=5`, `DECORATION_MASK=7` [`kitty/data-types.h`:L212-L213]. It is **rendered**
  using the **font‑level metrics** `underline_position`/`underline_thickness` from
  `calc_cell_metrics`, sampled in the fragment shader as
  `underline_alpha = texture(sprites, underline_pos).a` [`kitty/cell_fragment.glsl`:L15,L130].
- **strikethrough** — a **cell‑level attribute** `CellAttrs.strike` (1 bit)
  [`kitty/data-types.h`:L202], set by SGR 9 / cleared by SGR 29 [`kitty/cursor.c`:L98,L114], and
  **rendered** using the font‑level `strikethrough_position`/`strikethrough_thickness`, sampled as
  `strike_alpha = texture(sprites, strike_pos).a` [`kitty/cell_fragment.glsl`:L17,L131].
- **overline** — **NOT IMPLEMENTED in kitty 0.35.2 [OBSERVED].** The literal `overline`/`OVERLINE`
  appears **nowhere** in `kitty/*.c`, `kitty/*.h`, or `kitty/*.glsl` (only a substring match inside
  the compiled Go `kitten` binary). The `CellAttrs` bitfield [`kitty/data-types.h`:L196-L208] has
  `decoration`/`strike`/`bold`/`italic`/`reverse`/`dim`/`mark`/`width` but **no overline bit**. The
  SGR parser [`kitty/cursor.c`:L84-L135] handles cases `0,1,2,3,4,7,9,21,22,23,24,27,29,221,222` plus
  colors and `DECORATION_FG_CODE(58)` — there is **no case 53 (SGR overline)** and **no case 55
  (overline off)**. `calc_cell_metrics` computes **no** overline metric.

**Conclusion for "(overline/underline)":** `underline` (and `strikethrough`) are **cell‑level
attributes rendered from font‑level metrics** computed by `calc_cell_metrics` — the observed values
in §4.1 are exactly those metrics. `overline` is **neither** a font‑level metric **nor** a cell
decoration attribute in this version — it is simply **absent**, so it cannot be conflated with the
underline/strikethrough metrics. This is the honest, grounded disambiguation.

---

## 5. Q4 — GPU texture‑atlas page layout, sizing, capacity, and the "atlas ready" signal

The atlas is a `GL_TEXTURE_2D_ARRAY` of `GL_SRGB8_ALPHA8` cells. Its page layout is computed by the
**GL‑independent** sprite tracker `sprite_tracker_set_layout` [`kitty/fonts.c`:L277-L282] (so it is
observable headlessly), while the **live** allocation is `realloc_sprite_texture` →
`glTexStorage3D` [`kitty/shaders.c`:L108-L133] and requires an active GL context.

### 5.1 Two distinct `max_texture_size` variables (critical nuance)

There are **two** separate `max_texture_size` variables, which explains the startup numbers:

- **`fonts.c` static:** `static size_t max_texture_size = 1024, max_array_len = 1024;`
  [`kitty/fonts.c`:L44] — used by `sprite_tracker_set_layout` for the layout arithmetic and by
  `do_increment` for the layer‑overflow check.
- **`shaders.c` static:** `static GLint max_texture_size = 0, max_array_texture_layers = 0;`
  [`kitty/shaders.c`:L32] — the guard in `alloc_sprite_map`.

Order of events at startup **[INFERRED — code‑derived]**, then **confirmed by the live capture in
§5.4**: font‑group creation runs `calc_cell_metrics` → `sprite_tracker_set_layout`
[`kitty/fonts.c`:L418] while `fonts.c`'s `max_texture_size` is still the **default 1024**. Later,
when the window's GL context is ready, `alloc_sprite_map` [`kitty/shaders.c`:L51-L61] (guarded by
`shaders.c`'s `max_texture_size==0`) queries the live GL limits and calls
`sprite_tracker_set_limits(...)`, updating `fonts.c`'s `max_texture_size` — **but it does not
re‑run `sprite_tracker_set_layout`**, so the initial atlas keeps the 1024‑based layout.

### 5.2 Initial page layout & capacity (the actual computed numbers)

`sprite_tracker_set_layout` [`kitty/fonts.c`:L277-L282]:
`xnum = MIN(MAX(1u, max_texture_size / cell_width), UINT16_MAX)`,
`max_y = MIN(MAX(1u, max_texture_size / cell_height), UINT16_MAX)`, `ynum = 1; x=y=z=0`.

With the observed default `max_texture_size=1024` and cell `9×18`, **[OBSERVED — test‑API], stable ×2**
(probed empirically by allocating distinct glyph keys and finding the wrap points):

```
sprite_map_set_limits(1024, 1024); sprite_map_set_layout(9, 18)
first allocation  (BEFORE any do_increment) = (0, 0, 0)
second allocation (AFTER  first do_increment) = (1, 0, 0)
observed xnum  = 113     # x wraps to a new row here      (1024 // 9  = 113)
observed max_y = 56      # y wraps to a new layer here     (1024 // 18 = 56)
observed per-layer capacity = xnum * max_y = 113 * 56 = 6328 cells/layer
# z increments at allocation #6328: (112, 55, 0) -> (0, 0, 1)
```

So the **initial page** is **113 columns × 56 rows** of cells per layer = **6328 glyph slots per
layer**, starting at `ynum=1` (one row physically allocated, grown on demand up to `max_y`).

### 5.3 Limits and the page‑increment sequence

- Default limits before GL: `max_texture_size = 1024, max_array_len = 1024`
  [`kitty/fonts.c`:L44].
- `sprite_tracker_set_limits` caps layers: `max_array_len = MIN(0xfffu, max_array_len_)` = **4095**
  ceiling [`kitty/fonts.c`:L237-L239].
- Live GL limits (from the same Mesa llvmpipe renderer kitty uses, `glxinfo -l` under Xvfb)
  **[OBSERVED — canonical]**: `GL_MAX_TEXTURE_SIZE = 16384`, `GL_MAX_ARRAY_TEXTURE_LAYERS = 2048`
  → after `sprite_tracker_set_limits`, `max_array_len = MIN(4095, 2048) = 2048`.
- **Apple‑only clamps do NOT apply on Linux:** the `max_texture_size = MIN(8192, …)` and
  `max_array_texture_layers = MIN(512, …)` clamps are inside `#ifdef __APPLE__`
  [`kitty/shaders.c`:L55-L60]; on this Linux container they are compiled out (observed: Linux).

`do_increment` [`kitty/fonts.c`:L243-L256] advances `x → y → z` and, on layer overflow, sets
`*error=2` (`if (z >= MIN(UINT16_MAX, max_array_len)) *error=2;` [`kitty/fonts.c`:L250]). The
canonical increment mechanic was reproduced **exactly** against kitty's own `test_sprite_map`
pattern [`kitty_tests/fonts.py`:L119-L131] **[OBSERVED — test‑API], stable ×2**:

```
sprite_map_set_limits(10, 2); sprite_map_set_layout(5, 5)   # xnum=2, max_y=2, 2 layers
test_sprite_position_for(0) = (0, 0, 0)
test_sprite_position_for(1) = (1, 0, 0)
test_sprite_position_for(2) = (0, 1, 0)
test_sprite_position_for(3) = (1, 1, 0)
test_sprite_position_for(4) = (0, 0, 1)
test_sprite_position_for(5) = (1, 0, 1)
test_sprite_position_for(6) = (0, 1, 1)
test_sprite_position_for(7) = (1, 1, 1)
test_sprite_position_for(0, 1) = (0, 0, 2)   # ligature keys advance into the next layer
test_sprite_position_for(0, 2) = (1, 0, 2)
```

**atlas‑full (`error=2`) boundary [INFERRED — code‑derived], with an observed nuance:** the test API
does **not** surface `error=2` (allocations kept advancing past the layer limit) because
`sprite_position_for` assigns the position **before** `do_increment`, and the binding raises **only**
when `pos==NULL` (that is `error=1`) [`kitty/fonts.c`:L1576]. So `error=2` is a *soft* signal to the
render path, confirmed **not** observable via `test_sprite_position_for`.

### 5.4 Live GL allocation (`glTexStorage3D`) — the atlas actually created at startup

`realloc_sprite_texture` [`kitty/shaders.c`:L108-L133] reads the current layout, then calls
`glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum)`
[`kitty/shaders.c`:L123] with `width = xnum*cell_width`, `height = ynum*cell_height`, `znum = z+1`,
using `GL_NEAREST` filters and `GL_CLAMP_TO_EDGE` wrap; `NEW_SPRITE_MAP` initializes
`{ .xnum=1, .ynum=1, .last_num_of_layers=1, .last_ynum=-1 }` [`kitty/shaders.c`:L31].

The live path was **exercised** (per the persistence requirement). A plain `LD_PRELOAD` of
`glXGetProcAddressARB`/`eglGetProcAddress` did **not** intercept the call, because `glfw-x11.so`
`dlopen`s libGL/libEGL and `dlsym`s the loaders directly (its only undefined loader symbols are
`dlopen`/`dlsym`) — and kitty dispatches GL through GLAD function pointers
(`gladLoadGL(glfwGetProcAddress)` [`kitty/gl.c`:L55], `glad_glTexStorage3D`), not the dynamic
linker. The approach was varied: `dlsym` itself was interposed (non‑invasively, via
`dlvsym(RTLD_NEXT, "dlsym", "GLIBC_2.34")`) to wrap the getprocaddress functions and thereby wrap
`glTexStorage3D`. **No source was modified.**

**[OBSERVED — canonical, live GL], stable ×3:**

```
$ LANG=C.UTF-8 LC_ALL=C.UTF-8 LIBGL_ALWAYS_SOFTWARE=1 LD_PRELOAD=/tmp/glshim2.so \
    xvfb-run -a -s '-screen 0 1280x800x24' ./kitty/launcher/kitty --debug-gl \
    -o close_on_child_death=yes sh -c 'printf hi; sleep 2; true'
[GLSHIM] glTexStorage3D(target=0x8c1a=GL_TEXTURE_2D_ARRAY, levels=1, internalformat=0x8c43=GL_SRGB8_ALPHA8, width=1017, height=18, depth=1)
[0.143] OS Window created
[0.118] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

Decoding the live call against the source formula:

- `target = GL_TEXTURE_2D_ARRAY` (0x8C1A), `levels = 1`, `internalformat = GL_SRGB8_ALPHA8`
  (0x8C43) — exactly `glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, …)`
  [`kitty/shaders.c`:L123].
- `width = 1017 = xnum × cell_width = 113 × 9` → **live‑confirms `xnum = 113`**, i.e. the atlas uses
  the **default‑1024 layout**, not the live GL max of 16384 (which would give `width=16380`). This
  directly validates the two‑variable nuance in §5.1.
- `height = 18 = ynum × cell_height = 1 × 18` → `ynum = 1` at readiness.
- `depth = 1 = znum = z + 1` → `z = 0` (first layer).

Richer renders (52 distinct ASCII + Arabic + emoji × 20 lines) still produced a **single**
`glTexStorage3D` — the distinct‑glyph count stays under the 113 slots of the first row, so `ynum`
never grows and no reallocation occurs in a short session.

### 5.5 Primary and fallback fonts share one atlas

Sprite positions are tracked **per `Font`** (each `Font` has its own `sprite_position_hash_table`),
but the `(x,y,z)` coordinates are all drawn from the **shared** `fg->sprite_tracker`:
`sprite_position_for(fg, font, …)` assigns `s->x/y/z` from `fg->sprite_tracker`
[`kitty/fonts.c`:L257-L266]. Consequently glyphs from the **primary** (English → DejaVu Sans Mono)
and from a **fallback** (emoji → DejaVu Sans) allocate into the **same** texture array — the live
render in §5.4 included both and produced the one `1017×18×1` atlas texture. The glyph→sprite‑position
cache entry is `SpritePosItem` [`kitty/glyph-cache.c`:L12-L13]; upload is `send_sprite_to_gpu`
[`kitty/shaders.c`:L147-L155].

### 5.6 The "atlas ready" signal

**[OBSERVED + INFERRED].** There is **no explicit "atlas ready" log string** in the source (grep of
`kitty/shaders.c` shows only the raw `glTexStorage3D` at L123 with no adjacent debug print). Atlas
readiness **is** the **first successful `glTexStorage3D`** under a live GL context, invoked lazily
via `ensure_sprite_map` [`kitty/shaders.c`:L138-L139] on the first render. The observable startup
signals that verify this are, from `--debug-gl` / `--debug-rendering` (identical output):

```
[t] OS Window created
[t] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

together with the intercepted `glTexStorage3D(... width=1017, height=18, depth=1)` succeeding and
the process exiting cleanly (`--debug-gl` enables GL error checking, and no GL error was reported).
The interpretation *"ready = the first successful `glTexStorage3D` allocation"* is labeled
**[INFERRED]** because kitty emits no dedicated readiness log line.


---

## 6. Coverage matrix & stability

Every concrete item named across the four questions, addressed by name:

| Named item | Where answered | Label |
|---|---|---|
| **ligatures** | §2.3 (`group_state`/`shape`; `disable_ligatures=never`→`calt`; Fira Code `===`/`!=`/`----` grouped) | OBSERVED — test‑API |
| **bidi** | §2.4 (`hb_buffer_guess_segment_properties`; `force_ltr=no`; no word reordering) | OBSERVED + code‑derived |
| **combining diacritics** | §2.2 (`codepoint_for_mark`; `e´`→171; `e+U+0347+U+0305`→`(1,3,…)`) | OBSERVED — test‑API |
| **font fallback** | §2.5, §3.2 (emoji→DejaVu Sans; CJK→no font; Arabic covered by primary) | OBSERVED — canonical |
| **Arabic (RTL)** | §2.4, §3.2 (contextual GSUB positional forms; RTL inference) | OBSERVED |
| **English (LTR)** | §2, §3.2 (primary DejaVu Sans Mono; LTR inference) | OBSERVED |
| **cell metrics** | §4.1 (`cell_width=9`, `cell_height=18`) | OBSERVED — test‑API |
| **baseline** | §4.1 (`baseline=14`) | OBSERVED — test‑API |
| **overline** | §4.3 (NOT implemented: no SGR 53/55, no `CellAttrs` bit, no metric, no GLSL) | OBSERVED |
| **underline** | §4.1, §4.3 (`underline_position=15`/`thickness=1`; cell attr `decoration` rendered from font metric) | OBSERVED |
| **atlas page layout** | §5.2 (113 × 56 cells/layer; `ynum=1` initial) | OBSERVED — test‑API + live |
| **atlas sizing** | §5.4 (live `glTexStorage3D` `1017×18×1`, `GL_SRGB8_ALPHA8`) | OBSERVED — canonical (live GL) |
| **atlas capacity** | §5.2/§5.3 (6328 slots/layer; layer cap `MIN(4095,2048)=2048`) | OBSERVED + code‑derived |
| **primary font** | §5.5 (DejaVu Sans Mono glyphs → shared atlas) | OBSERVED |
| **fallback fonts** | §5.5 (DejaVu Sans emoji glyphs → same shared atlas) | OBSERVED |
| **"atlas ready" logs** | §5.6 (no explicit log; readiness = first `glTexStorage3D`; `OS Window created` + GL 4.5) | OBSERVED + INFERRED |

**User‑example phrases preserved verbatim:** *"ligatures, bidi, combining diacritics"* (§2),
*"mixed Arabic (RTL) and English (LTR) text"* (§3), *"(overline/underline)"* (§4).

**Stability.** Every reported value was reproduced across at least two identical runs
(`--debug-config` ×3, `--debug-font-fallback` ×2, shaping ×2, metrics ×2, sprite‑tracker layout ×2,
live `glTexStorage3D` ×3). No run‑to‑run variation was observed in any reported value.

---

## 7. Restore & final repository state

This was a **read‑only investigation**. All observation was done with per‑invocation CLI flags and
temporary scripts/fixtures placed in the **container‑local `/tmp`** (which is not part of the
bind‑mounted repository), so no repository file was ever touched by the observation harness. The
temporary artifacts used and removed on completion were:
`/tmp/obs_debug_config.sh`, `/tmp/fixture_ar_en.txt`, `/tmp/fixture_fallback.txt`,
`/tmp/obs_shape_fallback.py`, `/tmp/obs_q1.py`, `/tmp/obs_metrics.py`, `/tmp/obs_q4_atlas.py`,
`/tmp/obs_q4_atlas2.py`, `/tmp/glshim.c`, `/tmp/glshim.so`, `/tmp/glshim2.c`, `/tmp/glshim2.so`.

No persistent settings were changed (debug flags were only ever passed per‑invocation on the command
line). No custom `kitty.conf` was introduced. Build artifacts (`*.so`, `kitty/launcher/kitt*`) are
git‑ignored and do not affect repository status. The **only** change to the repository is the
creation of this document, `blitzy/documentation/kitty_815df1e210e0.md`.

