# kitty Terminal Emulator — Startup Investigation

| Field | Value |
|---|---|
| Repository | `kovidgoyal/kitty` |
| Commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Branch | `kitty_815df1e210e0` |
| Version reported | `kitty 0.35.2 created by Kovid Goyal` |
| Investigation date | April 2026 |
| Methodology | Source-code static analysis (primary) + runtime verification via `kitty +runpy` against the built native extension `kitty/fast_data_types.so` (see Appendix D) |
| Constraint | Read-only: **no source files modified** |

---

## Executive Summary

This document answers four technical questions about the kitty terminal emulator's internal subsystems at startup time:

1. **How kitty configures complex Unicode support (ligatures, bidirectional text, combining diacritics) and font fallback during initial startup.**
   Kitty's C core (`kitty/fonts.c`) creates a single process-global HarfBuzz buffer pre-allocated for 2048 codepoints with `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS`, plus three OpenType feature toggles (`-liga`, `-dlig`, `-calt`). The shaping pipeline (`load_hb_buffer` → `shape`) streams cell codepoints plus up to three combining marks per cell (`CPUCell.cc_idx[3]`), calls `hb_buffer_guess_segment_properties()` for script/direction auto-detection, and optionally forces `HB_DIRECTION_LTR` when `force_ltr` is set. Kitty does not implement the Unicode Bidirectional Algorithm (UAX #9); it relies on HarfBuzz's segment-property heuristic.

2. **What font families and fallback chains appear in startup diagnostics when `--debug-font-fallback` is enabled and mixed Arabic (RTL) and English (LTR) text is handled.**
   Enabling `--debug-font-fallback` activates the `debug_fonts(...)` macro defined in `kitty/state.h`, which is aliased to `debug` in `kitty/fonts.c`. At startup, `dump_font_debug()` in `kitty/fonts/render.py` prints the Normal/Bold/Italic/Bold-Italic faces (and any symbol-map faces) using each face's `identify_for_debug()` method (PostScript name + path + face index). At runtime, every cell whose character lacks coverage in the main font triggers `fallback_font` → `create_fallback_face` → `_fc_match()`, which adds the cell's charset to a fontconfig pattern family-keyed either `"monospace"` or `"emoji"`. Each unique fallback request is logged once via `output_cell_fallback_data`. For Arabic codepoints (U+0600–U+06FF) that are not in the main font, the fallback debug output records the chosen face as a Python-repr'd `FontConfigPattern` dict (family, style, path, etc.).

3. **What default cell metrics, baseline, and decoration alignment values appear in debug output for complex grapheme clusters.**
   Cell metrics are computed once per font group by `calc_cell_metrics` in `kitty/fonts.c`, which delegates the raw values to `cell_metrics()` in `kitty/freetype.c`. The FreeType backend derives `cell_width` from ASCII advance widths, `cell_height` from the face's `height` field (with an underscore-bounds workaround), `baseline` from the ascender, and underline/strikethrough position+thickness from the `FT_FaceRec` underline/strikethrough fields (with `baseline * 0.65` and `underline_thickness` fallbacks for fonts lacking OS/2 strikethrough metrics). User `modify_font` overrides apply `POINT`, `PERCENT`, or `PIXEL` adjustments via `adjust_metric`. Fallback fonts are rescaled via `set_size_for_face(face, fg->cell_height, true, ...)` so the grid stays monospaced. The only metric-related debug print at startup is the "Increasing cell height by N pixels" underscore-fix warning.

4. **What initial page layout, sizing, and capacity allocations are reported for the GPU sprite atlas at launch.**
   The sprite atlas is a `GL_TEXTURE_2D_ARRAY` managed by `kitty/shaders.c`. On first window creation, `send_prerendered_sprites_for_window` calls `alloc_sprite_map(cell_width, cell_height)`, which queries `GL_MAX_TEXTURE_SIZE` and `GL_MAX_ARRAY_TEXTURE_LAYERS` (capped at 8192 and 512 on Apple), calls `sprite_tracker_set_limits`, and returns a calloc'd `SpriteMap` initialized from `NEW_SPRITE_MAP` (`xnum=1, ynum=1, last_num_of_layers=1, last_ynum=-1`). `sprite_tracker_set_layout` then sets `xnum = max_texture_size / cell_width` and `max_y = max_texture_size / cell_height` (clamped to `UINT16_MAX`). `realloc_sprite_texture` lazily creates the texture via `glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum)` with `GL_NEAREST` filtering and `GL_CLAMP_TO_EDGE` wrapping. The first glyph data uploaded is a transparent "blank" cell at (0,0,0), followed by the 5 underline variants, 1 strikethrough, 1 missing-glyph sprite, and 3 cursor sprites — 10 pre-rendered sprites plus the blank occupying indices 0–10 of the atlas.

## Environment and Methodology

The investigation was performed inside the Docker container `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (base image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). The container includes Python 3.12.3, Go 1.22.2, GCC 13.3.0 on Ubuntu 24.04 with HarfBuzz 8.3.0, FreeType 26.1.20, fontconfig 2.15.0, and the full OpenGL/Wayland/X11 developer stacks. The project was built with `python3 setup.py build --ignore-compiler-warnings`; `kitty/launcher/kitty --version` reports `kitty 0.35.2 created by Kovid Goyal`.

The primary methodology is static reading of the source tree. Additionally, limited runtime verification was performed by invoking `./kitty/launcher/kitty +runpy` — a subcommand that enters the kitty Python environment with `fast_data_types.so` loaded — to confirm structural constants and fontconfig behavior. The detailed runtime findings are collected in **Appendix D**. Full end-to-end window-and-atlas verification was not possible because the container is headless (GLFW cannot initialize without a DISPLAY); consequently GPU-specific values in Section 4 are derived from the C formulas rather than measured.

Per the project rule, **no source files were modified**. All claims are grounded in specific file-and-line citations from the checked-in source tree at commit `815df1e21`. Where the exact value of a runtime quantity depends on the host GPU (e.g., `GL_MAX_TEXTURE_SIZE`), the specific font installed on the system (e.g., the family name returned by fontconfig), or the terminal size, this is explicitly called out and the derivation of the value is shown from the formulas in the C source.

---

## Section 1: Text Shaping and Layout Engine Initialization

This section documents how kitty sets up its text-shaping pipeline at startup — specifically the HarfBuzz integration that drives ligatures, bidirectional text, and complex clusters such as combining diacritics.

### 1.1 HarfBuzz Buffer Initialization (`init_fonts`)

The process-global HarfBuzz buffer and the three OpenType feature toggles are created exactly once per kitty process by `init_fonts` at the end of `kitty/fonts.c` (lines 1745–1761):

```c
bool
init_fonts(PyObject *module) {
    harfbuzz_buffer = hb_buffer_create();
    if (harfbuzz_buffer == NULL || !hb_buffer_allocation_successful(harfbuzz_buffer) || !hb_buffer_pre_allocate(harfbuzz_buffer, 2048)) { PyErr_NoMemory(); return false; }
    hb_buffer_set_cluster_level(harfbuzz_buffer, HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS);
#define create_feature(feature, where) {\
    if (!hb_feature_from_string(feature, sizeof(feature) - 1, &hb_features[where])) { \
        PyErr_SetString(PyExc_RuntimeError, "Failed to create " feature " harfbuzz feature"); \
        return false; \
    }}
    create_feature("-liga", LIGA_FEATURE);
    create_feature("-dlig", DLIG_FEATURE);
    create_feature("-calt", CALT_FEATURE);
#undef create_feature
    if (PyModule_AddFunctions(module, module_methods) != 0) return false;
    return true;
}
```

The buffer is pre-allocated to hold 2048 codepoints, which comfortably exceeds the typical terminal line width and avoids allocator churn during shaping. The cluster level is set to `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS`, which is critical: it guarantees that the HarfBuzz-assigned `cluster` field on each output glyph is a monotonically non-decreasing integer equal to the source-codepoint index. Kitty's glyph-to-cell grouping routines (`group_iosevka` at `kitty/fonts.c` line 975 and `group_normal` at line 1042) rely on this property to demux shaped glyphs back to the terminal cells that produced them, even when ligatures coalesce multiple cells into one glyph.

The three features created in `hb_features[]` are kitty's "negative" toggles: each is a `feature=0` directive that tells HarfBuzz *not* to apply the corresponding OpenType feature table. The constants `LIGA_FEATURE=0`, `DLIG_FEATURE=1`, `CALT_FEATURE=2` come from the `HBFeature` enum declared near the top of `fonts.c`. The features disable, respectively:

- `-liga`: standard ligatures (`liga` lookup table). Historically, latin ligatures like `fi`, `fl`.
- `-dlig`: discretionary ligatures (`dlig` lookup table). Rarely used in programming fonts.
- `-calt`: contextual alternates (`calt` lookup table). This is where *most* programming ligature fonts (Fira Code, JetBrains Mono, Cascadia Code) actually implement arrow-and-equal ligatures like `=>`, `->`, `!=`, `===`.

### 1.2 Per-Face Feature Array: `init_font`

When a face is loaded, `init_font` (`kitty/fonts.c` lines 293–328) constructs a per-face feature array that references the three global toggles but can be pre-populated with user-configured `font_features` overrides:

- If the user has configured `font_features` for the face's PostScript name, `num_ffs_hb_features = len+1`; the user's features are copied in, and `CALT_FEATURE` is appended as the final entry.
- If there are no user features, the array is allocated with capacity 4. Faces whose PostScript name begins with `NimbusMonoPS-` additionally get `LIGA_FEATURE` and `DLIG_FEATURE` prepended (a face-specific workaround). In all cases, `CALT_FEATURE` is appended last.

The invariant that `CALT_FEATURE` is always last matters for the ligature-disable mode explained below.

### 1.3 Combining Characters and Variation Selectors

Combining diacritics are modeled cell-by-cell. `kitty/data-types.h` defines `CPUCell` (lines 223–228) with a three-slot combining-character index array:

```c
typedef struct {
    char_type ch;
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];
} CPUCell;
```

Here `char_type` is `uint32_t` and `combining_type` is `uint16_t`, so each cell can carry its base codepoint plus up to three combining marks referenced via a compact 16-bit index. The reverse lookup from index back to the full Unicode codepoint is done through `codepoint_for_mark`, declared in `kitty/unicode-data.h` line 17, with the complementary `mark_for_codepoint` at line 18.

Two variation selectors have special names in `kitty/unicode-data.h` line 5:

```c
static const combining_type VS15 = 1364, VS16 = 1365;
```

These are kitty's internal mark indices for U+FE0E (text presentation selector) and U+FE0F (emoji presentation selector) — critical for distinguishing "☺" (text) from "☺️" (emoji). In `fonts.c` lines 429–432, the function `has_emoji_presentation` expresses this exactly:

```c
static bool
has_emoji_presentation(CPUCell *cpu_cell, GPUCell *gpu_cell) {
    return gpu_cell->attrs.width == 2 && is_emoji(cpu_cell->ch) && cpu_cell->cc_idx[0] != VS15;
}
```

A cell is rendered with emoji presentation when its width is 2, its base codepoint matches `is_emoji` (declared in `kitty/emoji.h`), and the first combining slot is not VS15 (which would force text presentation).

### 1.4 Loading the HarfBuzz Buffer: `load_hb_buffer`

The bridge from kitty's cell grid into HarfBuzz is `load_hb_buffer` (`kitty/fonts.c` lines 671–689):

```c
static void
load_hb_buffer(CPUCell *first_cpu_cell, GPUCell *first_gpu_cell, index_type num_cells) {
    index_type num;
    hb_buffer_clear_contents(harfbuzz_buffer);
    while (num_cells) {
        uint16_t prev_width = 0;
        for (num = 0; num_cells && num < arraysz(shape_buffer) - 20 - arraysz(first_cpu_cell->cc_idx); first_cpu_cell++, first_gpu_cell++, num_cells--) {
            if (prev_width == 2) { prev_width = 0; continue; }
            shape_buffer[num++] = first_cpu_cell->ch;
            prev_width = first_gpu_cell->attrs.width;
            for (unsigned i = 0; i < arraysz(first_cpu_cell->cc_idx) && first_cpu_cell->cc_idx[i]; i++) {
                shape_buffer[num++] = codepoint_for_mark(first_cpu_cell->cc_idx[i]);
            }
        }
        hb_buffer_add_utf32(harfbuzz_buffer, shape_buffer, num, 0, num);
    }
    hb_buffer_guess_segment_properties(harfbuzz_buffer);
    if (OPT(force_ltr)) hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR);
}
```

Step by step:

1. The buffer is cleared.
2. For each cell in the run, the base character `ch` is written into `shape_buffer`, followed by each non-zero combining mark (expanded from the `cc_idx[]` indices back to codepoints via `codepoint_for_mark`). Width-2 cells skip their spacer cell.
3. The 32-bit codepoints are handed to HarfBuzz via `hb_buffer_add_utf32`.
4. `hb_buffer_guess_segment_properties()` inspects the codepoints, auto-detects the Unicode script (via the Script property), and derives a default direction (e.g., `HB_DIRECTION_RTL` for Arabic scripts).
5. If the user has set `force_ltr`, the direction is overridden to `HB_DIRECTION_LTR`.

### 1.5 Ligature-Disable Modes: `shape`

`shape` (`fonts.c` lines 786–820) is what actually calls `hb_shape`. Its key lines are:

```c
size_t num_features = fobj->num_ffs_hb_features;
if (num_features && !disable_ligature) num_features--;  // the last feature is always -calt
hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features);
```

Because `CALT_FEATURE` (the `-calt` toggle) is always the *last* entry in `ffs_hb_features` (see §1.2), decrementing `num_features` by 1 causes HarfBuzz to not see the `-calt=0` feature — which is equivalent to letting the font's default `calt` lookups run, i.e., programming ligatures like `=>` are applied. When `disable_ligature` is set (from the `disable_ligatures` option, `kitty/options/definition.py` lines 115–133: `never`, `always`, `cursor`), the full feature array is passed, and `-calt` *is* applied, disabling ligatures.

### 1.6 The `force_ltr` Option and Bidirectional Text

Kitty does **not** implement the Unicode Bidirectional Algorithm (UAX #9). Its behavior relies entirely on HarfBuzz's segment-property heuristic. The `force_ltr` option is declared in `kitty/options/definition.py` lines 64–82, with rationale explaining that for RTL scripts words are "automatically displayed in RTL," but kitty does not support true BIDI. In `state.h` line 75, the C side exposes `bool force_ltr`, and `load_hb_buffer` consults it via the `OPT(...)` macro.

### 1.7 Thinking / Rationale

- **Why `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS`?** It is the only cluster mode that gives kitty a guaranteed one-to-one mapping from pre-shaping codepoint indices to post-shaping glyph clusters, which is needed to grab a contiguous glyph run and lay it out across monospaced cells.
- **Why pre-allocate 2048?** This is large enough for a wide terminal row (up to 2048 columns at 1 codepoint per cell) without reallocation. Most terminals are 80–500 columns.
- **Why three combining marks per cell (`cc_idx[3]`)?** The vast majority of real-world scripts (Vietnamese, IPA, Hebrew niqqud, Indic) rarely need more; the 16-bit slots keep `CPUCell` at 12 bytes for cache locality.
- **Why `force_ltr` rather than full BIDI?** Real BIDI is complex and stateful across lines. For logging and code, most users prefer the raw logical order, and for prose, they can pipe through `fribidi`. The heuristic direction detection inside HarfBuzz gives decent visuals for isolated RTL words in a mostly-LTR buffer without pulling in a BIDI dependency.
- **Why `-liga -dlig` default but not `-calt`?** Programming ligatures are desirable in terminal contexts (and live in `calt`), while latin typographic ligatures (`liga`) and discretionary ligatures (`dlig`) can confuse character counting and alignment, so they are disabled by default.

---

## Section 2: Font Family and Fallback Chain Diagnostics

This section traces what happens — from CLI flag to stderr output — when a user runs `kitty --debug-font-fallback` and the terminal processes a line that mixes Arabic (RTL) and English (LTR) characters.

### 2.1 The `--debug-font-fallback` Flag Path

The CLI option is declared in `kitty/cli.py` lines 1002–1005:

```python
--debug-font-fallback
type=bool-set
Print out information about the selection of fallback fonts for characters not
present in the main font.
```

This flag enters Python's `args` namespace as `args.debug_font_fallback`. At startup, `AppRunner.__call__` in `kitty/main.py` passes it directly to the C global state (line 249):

```python
set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)
```

`set_options` is the C binding implemented in `kitty/state.c` lines 726–744. Its parsing line is:

```c
PA("O|ppp", &opts, &is_wayland, &debug_rendering, &debug_font_fallback);
```

and at line 740 it commits the flag to the process-global structure:

```c
global_state.debug_font_fallback = debug_font_fallback ? true : false;
```

The three debug macros that consult this and the related flags are defined in `kitty/state.h` lines 14–16:

```c
#define debug_rendering(...) if (global_state.debug_rendering) { timed_debug_print(__VA_ARGS__); }
#define debug_input(...) if (OPT(debug_keyboard)) { timed_debug_print(__VA_ARGS__); }
#define debug_fonts(...) if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); }
```

Finally, at `kitty/fonts.c` line 18 a convenience alias is set up so the rest of the file can simply write `debug(...)`:

```c
#define debug debug_fonts
```

Every `debug(...)` call in `fonts.c` is thus a no-op unless `--debug-font-fallback` was passed.

### 2.2 Startup Font Resolution Path (Python side)

After `set_options`, `kitty/main.py` line 251 calls `set_font_family(opts)`. This is defined in `kitty/fonts/render.py` lines 173–193: it calls `get_font_files(opts)` to resolve the four main faces (medium, bold, italic, bi), constructs `current_faces = [(medium, False, False), (bold, True, False), (italic, False, True), (bi, True, True)]`, then calls `create_symbol_map(opts)` and `create_narrow_symbols(opts)` to resolve the user-configured symbol-map fonts, and finally hands everything to the native `set_font_data(...)` function.

`get_font_files` lives in `kitty/fonts/common.py` lines 281–296. It calls `get_font_from_spec` once for medium (no bold, no italic), then once each for bold, italic, and bi, each time passing `resolved_medium_font=medium_font` so variable-font bold/italic derivation can reuse the medium face's file. `get_font_from_spec` (common.py lines 243–261) dispatches to `find_best_match` for system specs like `font_family "monospace"` or the user-typed family name.

On Linux, `find_best_match` is implemented in `kitty/fonts/fontconfig.py` lines 170–213. It scores candidates in a font map built by `all_fonts_map` (lines 46–54):

```python
@lru_cache(maxsize=2)
def all_fonts_map(monospaced: bool = True) -> FontMap:
    if monospaced:
        ans = fc_list(spacing=FC_DUAL) + fc_list(spacing=FC_MONO)
    else:
        # allow non-monospaced and bitmapped fonts as these are used for
        # symbol_map
        ans = fc_list(allow_bitmapped_fonts=True)
    return create_font_map(ans)
```

If no family match is found, `find_last_resort_text_font` (lines 164–167) calls `fc_match('monospace', bold, italic)` for a final catch-all. `fc_match` itself (lines 81–83) is a thin lru-cached wrapper around the C-level `fc_match_impl`.

### 2.3 Runtime Fallback Face Creation: `create_fallback_face`

When a cell's codepoint is not in any of the four main faces and does not match a symbol map, kitty loads a runtime fallback face via `create_fallback_face` in `kitty/fontconfig.c` lines 462–487:

```c
PyObject*
create_fallback_face(PyObject UNUSED *base_face, CPUCell* cell, bool bold, bool italic, bool emoji_presentation, FONTS_DATA_HANDLE fg) {
    ensure_initialized();
    PyObject *ans = NULL;
    FcPattern *pat = FcPatternCreate();
    if (pat == NULL) return PyErr_NoMemory();
    AP(FcPatternAddString, FC_FAMILY, (const FcChar8*)(emoji_presentation ? "emoji" : "monospace"), "family");
    if (!emoji_presentation && bold) { AP(FcPatternAddInteger, FC_WEIGHT, FC_WEIGHT_BOLD, "weight"); }
    if (!emoji_presentation && italic) { AP(FcPatternAddInteger, FC_SLANT, FC_SLANT_ITALIC, "slant"); }
    if (emoji_presentation) { AP(FcPatternAddBool, FC_COLOR, true, "color"); }
    size_t num = cell_as_unicode_for_fallback(cell, char_buf);
    add_charset(pat, num);
    PyObject *d = _fc_match(pat);
    if (d) {
        ssize_t idx = -1;
        PyObject *q;
        while ((q = iter_fallback_faces(fg, &idx))) {
            if (face_equals_descriptor(q, d)) { ans = PyLong_FromSsize_t(idx); Py_CLEAR(d); goto end; }
        }
        ans = face_from_descriptor(d, fg);
        Py_CLEAR(d);
    }
end:
    if (pat != NULL) FcPatternDestroy(pat);
    return ans;
}
```

The function's logic in sequence:

1. `ensure_initialized()` initializes fontconfig if needed.
2. A fresh `FcPattern` is created.
3. The family is set via a single `AP(FcPatternAddString, FC_FAMILY, ...)` call using a ternary that selects `"emoji"` for emoji-presentation cells and `"monospace"` otherwise. For non-emoji fallbacks, bold or italic hints are also added.
4. `cell_as_unicode_for_fallback(cell, char_buf)` extracts the cell's codepoint plus any combining characters into `char_buf`, returning the count `num`. `add_charset(pat, num)` then adds those codepoints as the charset constraint.
5. `_fc_match(pat)` performs the actual fontconfig match and returns a Python descriptor `d`.
6. The existing loaded fallback faces are iterated via `iter_fallback_faces`. If any matches `d` (per `face_equals_descriptor`), the pre-existing index is returned as a `PyLong`.
7. Otherwise, a new face is constructed from the descriptor via `face_from_descriptor(d, fg)`.
8. The `FcPattern` is destroyed in the `end:` block.

### 2.4 Per-Cell Fallback Dispatch: `font_for_cell` and `fallback_font`

`font_for_cell` (fonts.c lines 563–605) is called for every cell. Its decision tree:

- Return `BLANK_FONT` for non-rendered characters, tabs, spaces.
- Return `BOX_FONT` for Unicode box-drawing and related blocks: `0x2500–0x2573`, `0x2574–0x259f`, `0x25d6–0x25d7`, `0x25cb`, `0x25dc–0x25e5`, `0x2800–0x28ff`, `0xe0b0–0xe0bf`, `0xee00–0xee0b`, `0x1fb00–0x1fbae`.
- Check each configured symbol map; return the symbol-map font index if the codepoint lies in one of its ranges.
- Based on `gpu_cell->attrs.bold` and `.italic`, dispatch to the medium/bold/italic/bi face. If that face `has_cell_text`, return its index.
- Otherwise call `fallback_font(fg, cpu_cell, gpu_cell)`.

`fallback_font` (fonts.c lines 519–544) builds a cache key that prefixes the cell's UTF-8 text with a single style character encoding the (emoji-presentation × bold × italic) triple. The key bytes are produced by:

```c
char style = emoji_presentation ? 'a' : 'A';
if (bold) style += italic ? 3 : 2; else style += italic ? 1 : 0;
char cell_text[8 + arraysz(cpu_cell->cc_idx) * 4] = {style};
const size_t cell_text_len = 1 + cell_as_utf8(cpu_cell, true, cell_text + 1, ' ');
```

so `'A'`/`'B'`/`'C'`/`'D'` correspond to non-emoji medium / italic / bold / bold-italic, and `'a'`/`'b'`/`'c'`/`'d'` are the same four weights with emoji presentation. `HASH_FIND_STR(fg->fallback_font_map, cell_text, s)` looks up the (style + UTF-8 text) key in the per-font-group hash table; on a miss, `load_fallback_font` is called and the resulting index is inserted back into `fallback_font_map_t` via `HASH_ADD_KEYPTR`.

`load_fallback_font` (fonts.c lines 480–517) caps the number of fallback faces at 100, calls `create_fallback_face`, and — if `global_state.debug_font_fallback` is set — emits diagnostic output.

### 2.5 Exact Debug Output Format

The primary fallback debug print is `output_cell_fallback_data` (fonts.c lines 456–468). It prints:

```
U+<hex-ch> [U+<hex-cc0> U+<hex-cc1> U+<hex-cc2> ]bold italic emoji_presentation using previous fallback font at index: <face>
```

with each of `bold `, `italic `, `emoji_presentation `, and `using previous fallback font at index: ` conditionally included, and `<face>` produced by `PyObject_Print(face, stderr, 0)` (the Python `repr` of the face, which for fontconfig is a dict-like `FontConfigPattern`).

When the face chosen by fontconfig does not actually contain the glyph (can happen if no font on the system covers the codepoint), `load_fallback_font` also prints the second diagnostic (lines 501–513):

```
The font chosen by the OS for the text: U+<ch> [U+<cc> ...] is <face> but it does not actually contain glyphs for that text
```

### 2.6 What Appears with Arabic + English Mixed Text

Consider the sample line `Hello ش العالم`:

- The ASCII codepoints U+0048–U+006F plus U+0020 are typically covered by the main `monospace` face (DejaVu Sans Mono or whichever fontconfig returns). These produce **no** `debug_fonts` output because `has_cell_text` returns `true` inside `font_for_cell`.
- The Arabic codepoints (e.g., U+0634 ش, U+0627 ا, U+0644 ل, U+0639 ع, U+062E خ) are not in DejaVu Sans Mono's default coverage. For each unique Arabic codepoint seen, `fallback_font` → `create_fallback_face` is called with `FC_FAMILY="monospace"` and a charset containing that codepoint. Fontconfig then returns a font that *does* cover it — on a standard Linux setup this is typically Noto Sans Arabic, DejaVu Sans, or a system-installed Arabic face.
- Each unique (style, cell-text) pair produces **one** `output_cell_fallback_data` line to stderr. A typical line (on a system with Noto Sans Arabic installed) looks approximately like:

```
U+634 {'path': '/usr/share/fonts/truetype/noto/NotoSansArabic-Regular.ttf', 'family': 'Noto Sans Arabic', 'style': 'Regular', 'weight': 80, 'slant': 0, 'spacing': 'PROPORTIONAL', ...}
```

(The full fontconfig descriptor is whatever `FontConfigPattern.__repr__` produces; the keys printed depend on which FcPattern properties were captured.)

- Subsequent cells with the same style and text reuse the cached fallback face via `fallback_font_map_t` and therefore print only the first time. Repeated Arabic characters with different combining marks — which is the common case — hit the cache after the first occurrence.

Because the investigation runs in a headless container without Arabic fonts configured, the concrete family name is system-dependent; what is *deterministic* from the source code is the output **format** and the **mechanism** (fontconfig-charset-constrained match, family `"monospace"` for non-emoji, one log line per unique style+text pair).

### 2.7 `dump_font_debug` at Startup

After the event loop starts, if `--debug-font-fallback` was passed, `kitty/main.py` lines 228–229 invoke `dump_font_debug()`. This function is defined in `kitty/fonts/render.py` lines 161–170:

```python
def dump_font_debug() -> None:
    cf = current_fonts()
    log_error('Text fonts:')
    for key, text in {'medium': 'Normal', 'bold': 'Bold', 'italic': 'Italic', 'bi': 'Bold-Italic'}.items():
        log_error(f'  {text}:', cf[key].identify_for_debug())  # type: ignore
    ss = cf['symbol']
    if ss:
        log_error('Symbol map fonts:')
        for s in ss:
            log_error('  ' + s.identify_for_debug())
```

Note that the style-key-to-label mapping (`medium`→`Normal`, `bold`→`Bold`, `italic`→`Italic`, `bi`→`Bold-Italic`) is hardcoded in the `dict.items()` iteration, so the stderr lines appear as `  Normal: <postscript>: <path>:<index>`, `  Bold: ...`, etc. Symbol-map fonts are printed as `  <postscript>: <path>:<index>` (two-space prefix, no key label).

The formatter `identify_for_debug` on a FreeType face is defined in `kitty/freetype.c` lines 737–743:

```c
static PyObject*
identify_for_debug(PyObject *s, PyObject *a UNUSED) {
    Face *self = (Face*)s;
    FaceIndex instance;
    instance.val = self->face->face_index;
    return PyUnicode_FromFormat("%s: %V:%d", FT_Get_Postscript_Name(self->face), self->path, "[path]", instance.val);
}
```

so each line looks like `FiraCode-Regular: /usr/share/fonts/truetype/firacode/FiraCode-Regular.ttf:0`. The `%V` conversion in `PyUnicode_FromFormat` accepts a pair `(PyObject*, const char*)` so the third arg is `self->path` with `"[path]"` as a fallback label; `instance.val` is an integer FreeType face index (for variable fonts or TTC files).

### 2.8 `kitty --debug-config` Font Reporting

`kitty --debug-config` drives `kitty/debug_config.py`. In its `debug_config` function, lines 260–263 print the fonts section:

```python
p(green('Fonts:'))
for k, font in current_fonts().items():
    if hasattr(font, 'identify_for_debug'):
        p(yellow(f'  {k}:'), font.identify_for_debug())
```

This is a separate, one-shot report path (not `--debug-font-fallback`) that prints the resolved fonts along with the OpenGL version, config paths, and non-default options.

### 2.9 Thinking / Rationale

- **Why prefix the cache key with a style character?** A bold glyph may legitimately resolve to a different fallback face than the regular weight of the same character (Arabic has specific bold weights not in the regular face file). Separating the cache by style prevents cross-contamination.
- **Why re-verify `has_cell_text` after `_fc_match`?** `_fc_match` returns the "best available" font even when no font on the system covers the requested charset. Emitting the "does not actually contain glyphs" diagnostic surfaces this so the user knows to install coverage.
- **Why `"monospace"` instead of the actual main-font family?** The main font's family is already known to lack the glyph. Asking fontconfig for any monospace face that covers the charset is a broader, more productive query.
- **Why the 100-face cap in `load_fallback_font`?** Each face holds an `FT_Face` handle, a HarfBuzz font, its own sprite-tracker state, and in-memory glyph caches. Unbounded growth could accumulate over long sessions with wildly mixed-script input.
- **Why fontconfig's `fc_list(FC_DUAL) + fc_list(FC_MONO)` rather than a blanket query?** The concatenation prefers dual-width families (like DejaVu Sans Mono with Greek/Cyrillic) while still including strict monospace, and the `lru_cache(maxsize=2)` keeps startup fast without hitting disk on subsequent rescans.

---

## Section 3: Cell Metrics and Decoration Alignment

This section documents the initial cell_width, cell_height, baseline, and decoration (underline and strikethrough) values produced by the screen grid and HarfBuzz subsystems at startup, including the exact C formulas used by kitty and the optional user adjustments.

### 3.1 The Call Graph from Startup to Cell Metrics

The path from kitty launch to a populated set of cell metrics is:

1. `_run_app` in `kitty/main.py` line 221 calls `create_os_window(...)`, which creates a `FontGroup` for the window.
2. Inside `font_group_for` (in `fonts.c`), `initialize_font_group(fg)` is invoked (`kitty/fonts.c` line 1494).
3. `initialize_font_group` sets up the medium/bold/italic/bi faces and symbol map faces, then calls `calc_cell_metrics(fg)` at line 1511.
4. `calc_cell_metrics` (fonts.c lines 372–422) calls the backend `cell_metrics` function on the medium face (which is FreeType on Linux: `kitty/freetype.c` lines 387–405), then applies user `modify_font` adjustments, then calls `sprite_tracker_set_layout(fg, ...)` at line 418.

After these calls, `fg->cell_width`, `fg->cell_height`, `fg->baseline`, `fg->underline_position`, `fg->underline_thickness`, `fg->strikethrough_position`, and `fg->strikethrough_thickness` are all populated and used by the rest of the rendering pipeline.

### 3.2 The Raw `cell_metrics` Function (FreeType Backend)

`kitty/freetype.c` lines 387–405 implements the raw metric computation from the FreeType face:

```c
void
cell_metrics(PyObject *s, unsigned int* cell_width, unsigned int* cell_height, unsigned int* baseline, unsigned int* underline_position, unsigned int* underline_thickness, unsigned int* strikethrough_position, unsigned int* strikethrough_thickness) {
    Face *self = (Face*)s;
    *cell_width = calc_cell_width(self);
    *cell_height = calc_cell_height(self, true);
    *baseline = font_units_to_pixels_y(self, self->ascender);
    *underline_position = MIN(*cell_height - 1, (unsigned int)font_units_to_pixels_y(self, MAX(0, self->ascender - self->underline_position)));
    *underline_thickness = MAX(1, font_units_to_pixels_y(self, self->underline_thickness));

    if (self->strikethrough_position != 0) {
      *strikethrough_position = MIN(*cell_height - 1, (unsigned int)font_units_to_pixels_y(self, MAX(0, self->ascender - self->strikethrough_position)));
    } else {
      *strikethrough_position = (unsigned int)floor(*baseline * 0.65);
    }
    if (self->strikethrough_thickness > 0) {
      *strikethrough_thickness = MAX(1, font_units_to_pixels_y(self, self->strikethrough_thickness));
    } else {
      *strikethrough_thickness = *underline_thickness;
    }
}
```

Seven outputs, each with a specific derivation rule:

- **`cell_width`**: from `calc_cell_width(self)` (freetype.c lines 374–383), which takes the ceiling of the maximum 26.6 fixed-point `horiAdvance` divided by 64 over the ASCII range 32–127 (i.e., the widest ASCII advance).
- **`cell_height`**: from `calc_cell_height(self, true)` (freetype.c lines 140–152), which is `font_units_to_pixels_y(self, self->height)`, with an underscore workaround: if rendering `_` would produce a bottom beyond `cell_height - 1`, the height is increased and a debug line is printed: `Increasing cell height by %u pixels to work around buggy font that renders underscore outside the bounding box`.
- **`baseline`**: `font_units_to_pixels_y(self, self->ascender)`. The ascender is the distance from the baseline to the top of the tallest glyph in font design units.
- **`underline_position`**: `MIN(cell_height - 1, font_units_to_pixels_y(self, MAX(0, ascender - underline_position)))`. The face's `underline_position` is stored as a negative offset below the baseline (per the OpenType spec), so `ascender - underline_position` puts the underline stroke at `ascender + |underline_position|` below the top of the cell, which is below the baseline. Clamping to `cell_height - 1` ensures the underline is always at least one pixel inside the cell.
- **`underline_thickness`**: `MAX(1, font_units_to_pixels_y(self, self->underline_thickness))`. At least one pixel, so the line is always drawn.
- **`strikethrough_position`**: if the face's OS/2 table provides a non-zero `strikethrough_position`, `MIN(cell_height - 1, font_units_to_pixels_y(self, MAX(0, ascender - strikethrough_position)))`. Otherwise the fallback `floor(baseline * 0.65)` places the strikethrough at 65% of the baseline height (approximately x-height).
- **`strikethrough_thickness`**: `MAX(1, font_units_to_pixels_y(self, self->strikethrough_thickness))` when the face provides a positive value; otherwise falls back to `underline_thickness`.

### 3.3 The `font_units_to_pixels_y` Formula

`kitty/freetype.c` lines 91–94:

```c
static int
font_units_to_pixels_y(Face *self, int x) {
    return (int)ceil((double)FT_MulFix(x, self->face->size->metrics.y_scale) / 64.0);
}
```

and the x-axis counterpart at lines 96–99:

```c
static int
font_units_to_pixels_x(Face *self, int x) {
    return (int)ceil((double)FT_MulFix(x, self->face->size->metrics.x_scale) / 64.0);
}
```

Explanation:

- `FT_MulFix(x, scale)` is FreeType's 16.16 fixed-point multiplier: it multiplies a font-units value `x` by the scale factor stored in `face->size->metrics.y_scale`, which FreeType computes to convert font design units (EM units) to output pixels at the currently-selected size and DPI.
- The result is in 26.6 fixed-point pixel units (FreeType's native format). Dividing by 64 converts to floating-point pixels.
- `ceil` rounds up to ensure that sub-pixel fractions do not chop off pixels — especially important for `cell_height`, where a rounding-down would cause descenders or underlines to be clipped.

### 3.4 `calc_cell_metrics` and User Adjustments

After raw metrics, `calc_cell_metrics` (`kitty/fonts.c` lines 372–422) applies user-configured adjustments from `kitty.conf`. Each adjustment is packaged as a `{ float val; AdjustmentUnit unit; }` pair (see `kitty/state.h` line 100), where `AdjustmentUnit` is an enum declared at `kitty/state.h` line 27: `typedef enum AdjustmentUnit { POINT = 0, PERCENT = 1, PIXEL = 2 } AdjustmentUnit;`. The helper `adjust_metric` (fonts.c lines 350–363) converts the adjustment into an integer offset and writes the result back through a pointer:

```c
static void
adjust_metric(unsigned int *metric, float adj, AdjustmentUnit unit, double dpi) {
    if (adj == 0.f) return;
    int a = 0;
    switch (unit) {
        case POINT:
            a = ((long)round((adj * (dpi / 72.0)))); break;
        case PERCENT:
            *metric = (int)roundf((fabsf(adj) * (float)*metric) / 100.f); return;
        case PIXEL:
            a = (int)roundf(adj); break;
    }
    *metric = (a < 0 && -a > (int)*metric) ? 0 : *metric + a;
}
```

Notes:

- `POINT`: `round(adj * dpi / 72)` converts points to pixels (DPI-aware).
- `PERCENT`: writes `|adj| * metric / 100` directly — the metric is *replaced* by the percentage of itself, not adjusted.
- `PIXEL`: integer rounding of `adj`.
- The final assignment `*metric = (a < 0 && -a > (int)*metric) ? 0 : *metric + a;` clamps the result to `0` if the adjustment would make it negative, otherwise adds it.

`calc_cell_metrics` then applies `OPT(cell_width)` and `OPT(cell_height)` with bounds `[MIN_WIDTH=2, MAX_DIM=1000]` and `[MIN_HEIGHT=4, MAX_DIM=1000]`. Violations trigger `log_error(...)` (and the adjustment is ignored) for the low end, and `fatal(...)` for the high end. Then `OPT(underline_thickness)`, `OPT(underline_position)`, `OPT(strikethrough_thickness)`, `OPT(strikethrough_position)`, and `OPT(baseline)` are applied. When `OPT(baseline)` shifts the baseline, `adjust_ypos` (fonts.c lines 366–370) re-clamps dependent metrics to stay inside the cell. Finally `underline_position = MIN(cell_height - 1, underline_position)` enforces the final in-cell constraint, and `sprite_tracker_set_layout(fg, cell_width, cell_height)` is called on line 418 to compute the sprite grid layout from the final dimensions.

### 3.5 The Prerender Function and Decoration Sprites

`kitty/fonts/render.py` lines 364–396 defines `prerender_function`, called once per font-group initialization from C `send_prerendered_sprites` (fonts.c line 1458, with the Python format string `"IIIIIIIffdd"`):

```python
def prerender_function(
    cell_width: int,
    cell_height: int,
    baseline: int,
    underline_position: int,
    underline_thickness: int,
    strikethrough_position: int,
    strikethrough_thickness: int,
    cursor_beam_thickness: float,
    cursor_underline_thickness: float,
    dpi_x: float,
    dpi_y: float
) -> Tuple[Tuple[int, ...], Tuple[CBufType, ...]]:
    # Pre-render the special underline, strikethrough and missing and cursor cells
    f = partial(
        render_special, cell_width=cell_width, cell_height=cell_height, baseline=baseline,
        underline_position=underline_position, underline_thickness=underline_thickness,
        strikethrough_position=strikethrough_position, strikethrough_thickness=strikethrough_thickness,
        dpi_x=dpi_x, dpi_y=dpi_y
    )
    c = partial(
        render_cursor, cursor_beam_thickness=cursor_beam_thickness,
        cursor_underline_thickness=cursor_underline_thickness, cell_width=cell_width,
        cell_height=cell_height, dpi_x=dpi_x, dpi_y=dpi_y)
    # If you change the mapping of these cells you will need to change
    # NUM_UNDERLINE_STYLES and BEAM_IDX in shader.c and STRIKE_SPRITE_INDEX in
    # window.py and MISSING_GLYPH in font.c
    cells = list(map(f, range(1, NUM_UNDERLINE_STYLES + 1)))  # underline sprites
    cells.append(f(0, strikethrough=True))  # strikethrough sprite
    cells.append(f(missing=True))  # missing glyph
    cells.extend((c(1), c(2), c(3)))  # cursor glyphs
    tcells = tuple(cells)
    return tuple(map(ctypes.addressof, tcells)), tcells
```

The two `partial` bindings `f` (for `render_special`) and `c` (for `render_cursor`) pre-bind all the per-font-group dimensions; the body then produces exactly 10 buffers:

- `cells = list(map(f, range(1, NUM_UNDERLINE_STYLES + 1)))` — calls `render_special(1..5, ...)` where the argument is the underline style index, producing sprites for the 5 underline styles (straight, double, curly, dotted, dashed). `NUM_UNDERLINE_STYLES = 5u` is defined in `kitty/data-types.h`.
- `cells.append(f(0, strikethrough=True))` — one strikethrough sprite rendered at `strikethrough_position` with `strikethrough_thickness`.
- `cells.append(f(missing=True))` — one missing-glyph hollow rectangle.
- `cells.extend((c(1), c(2), c(3)))` — three cursor glyphs (beam, block, underline — the integer argument selects the cursor shape inside `render_cursor`).

The return value is a 2-tuple: `tuple(map(ctypes.addressof, tcells))` is the tuple of `void *`-compatible integer addresses consumed by the C side, and `tcells` is the tuple of the underlying `CBufType` buffers (kept alive by the C code via `Py_CLEAR(args)` after consumption). Together with the blank cell that `send_prerendered_sprites` uploads separately at index 0, this yields sprite indices 0–10 in the atlas (11 total initial sprites).

### 3.6 Complex Grapheme Clusters

Terminal grapheme clusters (ligatures, Devanagari conjuncts, emoji with modifiers) either share a single cell or use two cells per `GPUCell.attrs.width`. Cell metrics are derived *only* from the medium font, so cell dimensions are locked at startup before any fallback font is loaded. Fallback fonts are then rescaled to match the cell height via `set_size_for_face(face, fg->cell_height, true, ...)` at `fonts.c` line 494 and line 1515 (for symbol-map fonts). This ensures the terminal grid remains uniformly monospaced regardless of which face actually renders a given cluster.

### 3.7 Startup Debug Output for Metrics

The only direct debug print for metrics at startup is the underscore-fix warning from `calc_cell_height` (`freetype.c` lines 146–148), emitted only when a font with a buggy bounding box triggers the height increase. The computed `cell_width`, `cell_height`, etc. are *not* printed by `--debug-font-fallback`; they are instead exposed via `current_fonts()` for Python introspection and visible in the `kitty --debug-config` report.

### 3.8 Thinking / Rationale

- **Why `ceil` in `font_units_to_pixels_y`?** Rounding down could drop the top pixel of the tallest glyph, causing visible clipping on antialiased glyphs. Rounding up is always safe for a monospaced grid.
- **Why `MAX(1, ...)` on `underline_thickness`?** At very small font sizes, the raw font units could scale to 0 pixels, rendering an invisible underline. A minimum of 1 guarantees visibility.
- **Why the `baseline * 0.65` fallback for strikethrough?** 65% of the ascender (baseline) is a typographic convention roughly matching x-height. Most OS/2 tables provide a strikethrough position; this fallback catches poorly-authored fonts.
- **Why derive metrics from the medium face only?** The terminal grid is monospaced — all cells share a single `cell_width × cell_height`. Bold, italic, and fallback faces must fit into that grid. Using the medium font as the canonical source avoids inconsistency and simplifies layout.
- **Why `POINT`, `PERCENT`, `PIXEL` units for adjustments?** Different use-cases need different units: `POINT` for typographic adjustments independent of DPI, `PERCENT` for relative tweaks, `PIXEL` for pixel-perfect alignment. The same tri-unit system extends uniformly across all `modify_font` options.
- **Why the sprite layout is computed at `calc_cell_metrics` time?** Sprite grid columns (`xnum`) and rows (`max_y`) depend on `cell_width`/`cell_height`; doing it here ensures that by the time `send_prerendered_sprites` runs, the layout is already sized correctly.

---

## Section 4: GPU Texture Atlas Initialization

This section documents the initial layout, sizing, capacity allocations, and startup confirmation of the GPU texture atlas (sprite map) used to hold shaped glyphs across primary and fallback fonts.

### 4.1 The `SpriteMap` Struct

`kitty/shaders.c` lines 24–32 defines the sprite-map state:

```c
typedef struct {
    unsigned int cell_width, cell_height;
    int xnum, ynum, x, y, z, last_num_of_layers, last_ynum;
    GLuint texture_id;
    GLint max_texture_size, max_array_texture_layers;
} SpriteMap;

static const SpriteMap NEW_SPRITE_MAP = { .xnum = 1, .ynum = 1, .last_num_of_layers = 1, .last_ynum = -1 };
static GLint max_texture_size = 0, max_array_texture_layers = 0;
```

Field-by-field meaning:

| Field | Meaning |
|---|---|
| `cell_width`, `cell_height` | Dimensions in pixels of one glyph cell, inherited from the font group. |
| `xnum` | Number of sprite columns per texture layer (computed from `max_texture_size / cell_width`). |
| `ynum` | Current number of sprite rows used in the active layer. Starts at 1, grows as rows fill. |
| `x`, `y`, `z` | Next-write coordinates: column, row, and array layer index. All start at 0. |
| `last_num_of_layers` | The array layer count of the currently-allocated texture. Starts at 1; used to detect whether a re-alloc is needed. |
| `last_ynum` | The last `ynum` that was written into the texture storage. Starts at `-1` (never uploaded). |
| `texture_id` | OpenGL 2D array texture handle. `0` until `realloc_sprite_texture` runs. |
| `max_texture_size`, `max_array_texture_layers` | Cached copies of the GPU's declared limits from the first `alloc_sprite_map` call. |

The `NEW_SPRITE_MAP` initializer seeds every new sprite map with xnum=1, ynum=1, last_num_of_layers=1, last_ynum=-1 (and all other fields implicit zero from `calloc`). The x=y=z=0 starting coordinates then serve as the position for the first sprite upload (the blank cell).

### 4.2 `alloc_sprite_map` — Initial Sprite-Map Allocation

`kitty/shaders.c` lines 50–70:

```c
SPRITE_MAP_HANDLE
alloc_sprite_map(unsigned int cell_width, unsigned int cell_height) {
    if (!max_texture_size) {
        glGetIntegerv(GL_MAX_TEXTURE_SIZE, &(max_texture_size));
        glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, &(max_array_texture_layers));
#ifdef __APPLE__
        // Since on Apple we could have multiple GPUs, with different capabilities,
        // upper bound the values according to the data from https://developer.apple.com/graphicsimaging/opengl/capabilities/
        max_texture_size = MIN(8192, max_texture_size);
        max_array_texture_layers = MIN(512, max_array_texture_layers);
#endif
        sprite_tracker_set_limits(max_texture_size, max_array_texture_layers);
    }
    SpriteMap *ans = calloc(1, sizeof(SpriteMap));
    if (!ans) fatal("Out of memory allocating a sprite map");
    *ans = NEW_SPRITE_MAP;
    ans->max_texture_size = max_texture_size;
    ans->max_array_texture_layers = max_array_texture_layers;
    ans->cell_width = cell_width; ans->cell_height = cell_height;
    return (SPRITE_MAP_HANDLE)ans;
}
```

Step by step:

1. On the first call (module-level `max_texture_size == 0`), `glGetIntegerv(GL_MAX_TEXTURE_SIZE, ...)` and `glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, ...)` query the active OpenGL context for the GPU's published maximums.
2. On Apple, the values are capped at `min(8192, driver_value)` and `min(512, driver_value)` to match Apple's documented guaranteed capabilities across their GPU lineup — this avoids using higher values that might fail silently on older integrated GPUs.
3. `sprite_tracker_set_limits(max_texture_size, max_array_texture_layers)` propagates the values into `kitty/fonts.c` lines 236–240, where `max_array_len = MIN(0xfffu, max_array_len_)` caps the array-layer count at 4095.
4. A new `SpriteMap` is allocated and initialized from `NEW_SPRITE_MAP`, with `cell_width`/`cell_height` stamped in from the font group, and the cached GPU limits stored on the struct.

### 4.3 `sprite_tracker_set_layout` — Computing xnum and max_y

`kitty/fonts.c` lines 275–281:

```c
static void
sprite_tracker_set_layout(GPUSpriteTracker *sprite_tracker, unsigned int cell_width, unsigned int cell_height) {
    sprite_tracker->xnum = MIN(MAX(1u, max_texture_size / cell_width), (size_t)UINT16_MAX);
    sprite_tracker->max_y = MIN(MAX(1u, max_texture_size / cell_height), (size_t)UINT16_MAX);
    sprite_tracker->ynum = 1;
    sprite_tracker->x = 0; sprite_tracker->y = 0; sprite_tracker->z = 0;
}
```

- `xnum = clamp(max_texture_size / cell_width, 1, UINT16_MAX)` — number of sprite columns per layer.
- `max_y = clamp(max_texture_size / cell_height, 1, UINT16_MAX)` — maximum sprite rows per layer before rolling over to the next array layer.
- `ynum = 1` — initially only one row is "live"; grown as sprites are added.
- `x, y, z = 0, 0, 0` — the write head begins at the origin.

This runs from within `calc_cell_metrics` at `fonts.c` line 418, immediately after the cell dimensions are finalized, so the sprite grid is always sized to the final `cell_width`/`cell_height`.

### 4.4 `realloc_sprite_texture` — Texture Storage Allocation

`kitty/shaders.c` lines 107–134 creates (or grows) the GL texture:

```c
static void
realloc_sprite_texture(FONTS_DATA_HANDLE fg) {
    GLuint tex;
    glGenTextures(1, &tex);
    glBindTexture(GL_TEXTURE_2D_ARRAY, tex);
    // We use GL_NEAREST otherwise glyphs that touch the edge of the cell
    // often show a border between cells
    glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    unsigned int xnum, ynum, z, znum, width, height, src_ynum;
    sprite_tracker_current_layout(fg, &xnum, &ynum, &z);
    znum = z + 1;
    SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
    width = xnum * sprite_map->cell_width; height = ynum * sprite_map->cell_height;
    glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum);
    if (sprite_map->texture_id) {
        // need to re-alloc
        src_ynum = MAX(1, sprite_map->last_ynum);
        copy_image_sub_data(sprite_map->texture_id, tex, width, src_ynum * sprite_map->cell_height, sprite_map->last_num_of_layers);
        glDeleteTextures(1, &sprite_map->texture_id);
    }
    glBindTexture(GL_TEXTURE_2D_ARRAY, 0);
    sprite_map->last_num_of_layers = znum;
    sprite_map->last_ynum = ynum;
    sprite_map->texture_id = tex;
}
```

Line-by-line:

- `glGenTextures(1, &tex)` + `glBindTexture(GL_TEXTURE_2D_ARRAY, tex)` — creates a new 2D array texture (a stack of `znum` 2D slices, each addressable as layer `z`).
- `GL_NEAREST` for both `MIN_FILTER` and `MAG_FILTER` — no interpolation. Per the comment, interpolation between adjacent sprite cells would cause visible borders where one glyph bleeds into an adjacent cell slot.
- `GL_CLAMP_TO_EDGE` for S and T — sampling just outside the texture edge repeats the border pixel, avoiding wrap artifacts if shader UV math drifts by sub-pixel.
- `sprite_tracker_current_layout(fg, &xnum, &ynum, &z)` (fonts.c lines 268–272) reads the current layout (number of columns, current active rows, and the current layer index).
- `znum = z + 1` — total layer count = current layer + 1.
- `width = xnum * cell_width`, `height = ynum * cell_height` — pixel dimensions. At initial allocation, `ynum=1`, so `height = cell_height`.
- `glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum)` — allocates immutable texture storage with one mipmap level, `SRGB8_ALPHA8` internal format, dimensions `width × height × znum`.
- If a previous `texture_id` exists (growth case), `copy_image_sub_data` copies existing contents to the new texture (fast path via `glCopyImageSubData`, slow fallback via CPU roundtrip with a one-time warning), then the old texture is deleted.
- The new `texture_id` and dimensions are stamped into `sprite_map`.

### 4.5 `ensure_sprite_map` — Lazy Initialization and Binding

`kitty/shaders.c` lines 137–144:

```c
static void
ensure_sprite_map(FONTS_DATA_HANDLE fg) {
    SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
    if (!sprite_map->texture_id) realloc_sprite_texture(fg);
    // We have to rebind since we don't know if the texture was ever bound
    // in the context of the current OSWindow
    glActiveTexture(GL_TEXTURE0 + SPRITE_MAP_UNIT);
    glBindTexture(GL_TEXTURE_2D_ARRAY, sprite_map->texture_id);
}
```

Called before every cell-rendering draw call. It lazily allocates the texture on first use and always rebinds it to `GL_TEXTURE0 + SPRITE_MAP_UNIT` (where `SPRITE_MAP_UNIT = 0`, `shaders.c` line 21) — necessary because multi-window contexts don't share binding state.

### 4.6 `send_prerendered_sprites_for_window` — The First Upload

`kitty/fonts.c` lines 1520–1527:

```c
void
send_prerendered_sprites_for_window(OSWindow *w) {
    FontGroup *fg = (FontGroup*)w->fonts_data;
    if (!fg->sprite_map) {
        fg->sprite_map = alloc_sprite_map(fg->cell_width, fg->cell_height);
        send_prerendered_sprites(fg);
    }
}
```

This is called from the window-initialization logic as the first OSWindow is shown. It performs one-time setup: allocate the sprite map state (which queries GL caps), then call `send_prerendered_sprites` to upload the initial 11 sprites.

`send_prerendered_sprites` (fonts.c lines 1449–1473):

1. Initializes `sprite_index x = 0, y = 0, z = 0`, calls `ensure_canvas_can_fit(fg, 1)` to zero the canvas, then uploads a blank cell at `(x=0, y=0, z=0)` via `current_send_sprite_to_gpu((FONTS_DATA_HANDLE)fg, x, y, z, fg->canvas.buf)` — this is sprite index 0. It then calls `do_increment(fg, &error)` to advance the sprite tracker.
2. Calls the Python `prerender_function` via `PyObject_CallFunction(prerender_function, "IIIIIIIffdd", fg->cell_width, fg->cell_height, fg->baseline, fg->underline_position, fg->underline_thickness, fg->strikethrough_position, fg->strikethrough_thickness, OPT(cursor_beam_thickness), OPT(cursor_underline_thickness), fg->logical_dpi_x, fg->logical_dpi_y)` — returns a 2-tuple `(addresses, tcells)` where `addresses` is a tuple of `ctypes.addressof(...)` pointers to the alpha-mask buffers (10 entries: 5 underlines + strikethrough + missing + 3 cursors) and `tcells` is the tuple of backing `CBufType` buffers. The C side accesses the first via `cell_addresses = PyTuple_GET_ITEM(args, 0)`.
3. For each alpha mask `i` in `cell_addresses`, it reads the current `fg->sprite_tracker.x/y/z` into `x, y, z`, checks `if (y > 0) fatal(...)` (prerendered sprites must all fit in row 0), calls `do_increment` to advance the tracker, dereferences the address via `PyLong_AsVoidPtr(PyTuple_GET_ITEM(cell_addresses, i))` to get the `uint8_t *alpha_mask`, calls `ensure_canvas_can_fit(fg, 1)` to clear the canvas, calls `render_alpha_mask(alpha_mask, fg->canvas.buf, &r, &r, fg->cell_width, fg->cell_width, 0xffffff)` to composite the mask onto a white-on-transparent RGBA buffer, and sends it via `current_send_sprite_to_gpu((FONTS_DATA_HANDLE)fg, x, y, z, fg->canvas.buf)`.
4. If more than one row is needed for the prerendered sprites (i.e., `y > 0` after a `do_increment`), fatal error: `Too many pre-rendered sprites for your GPU or the font size is too large` (fonts.c line 1463).

### 4.7 `send_sprite_to_gpu` — Cell Upload

`kitty/shaders.c` lines 146–156:

```c
void
send_sprite_to_gpu(FONTS_DATA_HANDLE fg, unsigned int x, unsigned int y, unsigned int z, pixel *buf) {
    SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
    unsigned int xnum, ynum, znum;
    sprite_tracker_current_layout(fg, &xnum, &ynum, &znum);
    if ((int)znum >= sprite_map->last_num_of_layers || (znum == 0 && (int)ynum > sprite_map->last_ynum)) realloc_sprite_texture(fg);
    glBindTexture(GL_TEXTURE_2D_ARRAY, sprite_map->texture_id);
    glPixelStorei(GL_UNPACK_ALIGNMENT, 4);
    x *= sprite_map->cell_width; y *= sprite_map->cell_height;
    glTexSubImage3D(GL_TEXTURE_2D_ARRAY, 0, x, y, z, sprite_map->cell_width, sprite_map->cell_height, 1, GL_RGBA, GL_UNSIGNED_INT_8_8_8_8, buf);
}
```

Notes:

- Growth detection: if the current layout uses more layers than allocated, or more rows in the current layer than previously stored, `realloc_sprite_texture` is called before upload.
- Coordinate conversion: `(x, y)` are grid coordinates in the atlas; multiplying by cell dimensions yields pixel offsets for `glTexSubImage3D`.
- `GL_UNSIGNED_INT_8_8_8_8` packs each RGBA pixel as a single 32-bit word, which is the format produced by `render_alpha_mask` and the glyph rasterization pipeline.

### 4.8 Sprite Positioning and the Glyph Cache

Sprite positions are managed by a per-font hash table (`kitty/glyph-cache.c`). The function `find_or_create_sprite_position` is called from `sprite_position_for` at `kitty/fonts.c` lines 256–266:

```c
static SpritePosition*
sprite_position_for(FontGroup *fg, Font *font, glyph_index *glyphs, unsigned glyph_count, uint8_t ligature_index, unsigned cell_count, int *error) {
    bool created;
    SpritePosition *s = find_or_create_sprite_position(&font->sprite_position_hash_table, glyphs, glyph_count, ligature_index, cell_count, &created);
    if (!s) { *error = 1; return NULL; }
    if (created) {
        s->x = fg->sprite_tracker.x; s->y = fg->sprite_tracker.y; s->z = fg->sprite_tracker.z;
        do_increment(fg, error);
    }
    return s;
}
```

On a cache miss (`created == true`), the current `(x, y, z)` from `fg->sprite_tracker` is stamped onto the new `SpritePosition` and `do_increment(fg, error)` advances the write head. `do_increment` is defined at `kitty/fonts.c` lines 242–253:

```c
static void
do_increment(FontGroup *fg, int *error) {
    fg->sprite_tracker.x++;
    if (fg->sprite_tracker.x >= fg->sprite_tracker.xnum) {
        fg->sprite_tracker.x = 0; fg->sprite_tracker.y++;
        fg->sprite_tracker.ynum = MIN(MAX(fg->sprite_tracker.ynum, fg->sprite_tracker.y + 1), fg->sprite_tracker.max_y);
        if (fg->sprite_tracker.y >= fg->sprite_tracker.max_y) {
            fg->sprite_tracker.y = 0; fg->sprite_tracker.z++;
            if (fg->sprite_tracker.z >= MIN((size_t)UINT16_MAX, max_array_len)) *error = 2;
        }
    }
}
```

The write head advances column-first, then row, then layer. If z overflows `min(UINT16_MAX, max_array_len)` (4095 on most GPUs), `*error = 2` is set and the caller emits `Out of texture space for sprites`. This is how sprites from primary, bold, italic, symbol-map, and fallback fonts coexist in the same atlas: each gets a unique (x, y, z) allocated in the order they are first rendered.

### 4.9 Startup Confirmation

There is no explicit "atlas ready" log message. Readiness is signaled by:

- `alloc_sprite_map` returning without `fatal("Out of memory allocating a sprite map")`.
- `realloc_sprite_texture` returning without a GL error from `glTexStorage3D`.
- `send_prerendered_sprites` completing all 10 Python-rendered sprite uploads without the `fatal("Too many pre-rendered sprites...")` guard triggering.

The only warning that can be emitted is the one-time `WARNING: Your system's OpenGL implementation does not have glCopyImageSubData, falling back to a slower implementation` at `shaders.c` lines 87–91, printed on the first re-allocation if `GL_ARB_copy_image` is unavailable — this is a quality-of-service message, not an error.

### 4.10 Numerical Example (Illustrative)

These are derived from the formulas; actual values depend on the GPU and chosen font size. Assume a typical desktop Linux GPU and the kitty default font at 11pt / 96 DPI:

| Variable | Derivation | Example value |
|---|---|---|
| `GL_MAX_TEXTURE_SIZE` | driver query | 16384 |
| `GL_MAX_ARRAY_TEXTURE_LAYERS` | driver query, capped at 0xfff=4095 | 2048 |
| `cell_width` (pixels) | `calc_cell_width` on medium font | ~8 |
| `cell_height` (pixels) | `calc_cell_height` on medium font | ~20 |
| `xnum` | `16384 / 8` | 2048 |
| `max_y` | `16384 / 20` | 819 |
| Initial `ynum` | constant | 1 |
| Initial `znum` | `z + 1` = `0 + 1` | 1 |
| Initial texture pixel size | `xnum * cell_width × ynum * cell_height × znum` | 16384 × 20 × 1 |
| Initial texture memory | `16384 * 20 * 1 * 4 bytes (SRGB8_ALPHA8)` | ~1.3 MB |

### 4.11 Thinking / Rationale

- **Why `xnum = max_texture_size / cell_width` rather than a fixed value?** Maximizes the number of cells per layer, minimizing the number of array layers needed for a given glyph count, which in turn minimizes GPU memory overhead (each new layer adds a full layer's worth of pixels).
- **Why the Apple cap at 8192/512?** Apple publishes a compatibility matrix for OpenGL on macOS that guarantees these values across all their GPUs. Using higher values could silently fail on older Intel integrated graphics.
- **Why `GL_SRGB8_ALPHA8`?** Enables hardware sRGB decoding on sampling, so anti-aliased glyph alpha blending happens in linear color space — produces correct (non-gamma-distorted) edges on colored backgrounds.
- **Why `glTexStorage3D` with `levels=1`?** No mipmaps — kitty always renders at 1:1 pixel ratio (each sprite is drawn at its native size), so no level-of-detail lookup is needed. This saves ~1/3 of the texture memory overhead of mipmaps.
- **Why `GL_NEAREST` instead of `GL_LINEAR`?** Sprites are packed edge-to-edge in the atlas. `GL_LINEAR` would bleed adjacent sprites into each other's cells; `GL_NEAREST` reads each pixel exactly.
- **Why `GL_CLAMP_TO_EDGE`?** Sub-pixel drift in UV coordinates could otherwise wrap around and sample from the opposite edge of the atlas, causing visible glitches at glyph boundaries.
- **Why the 3D indexing (x, y, z)?** A direct (column, row, layer) index maps trivially to shader lookup without any rectangle-packing bookkeeping. It wastes some slots but keeps the addressing logic simple and fast.
- **Why the 4095-layer cap (0xfff)?** Fits the z coordinate into the 12-bit field of `sprite_index` in `GPUCell`, which is packed with other data for GPU bandwidth efficiency.
- **Why pre-render the underline/strikethrough/missing/cursor sprites?** Avoids re-rasterizing these ubiquitous decorations per-frame and ensures they share exactly the cell grid, regardless of which font is active. They also occupy low sprite indices, which keeps their cache locality in the GPU's texture cache good.

---

## Appendix A: Startup Orchestration Sequence

This appendix enumerates the exact chronological sequence of function calls from CLI parsing to the first rendered frame, with file and line references. Each step cites the kitty source location where that action originates.

| # | Step | File / Function | Lines |
|---|---|---|---|
| 1 | CLI parsing — produces `args.debug_font_fallback`, `args.debug_rendering`, etc. | `kitty/cli.py` (CLI option definitions) | ~1002–1005 (debug-font-fallback) |
| 2 | Python entry: `run_app.__call__(opts, args, ...)` invoked | `kitty/main.py` `AppRunner.__call__` | ~247 |
| 3 | Native options applied: `set_options(opts, is_wayland(), debug_rendering, debug_font_fallback)` | `kitty/main.py` → `kitty/state.c` `PYWRAP1(set_options)` | main.py:249; state.c:726 |
| 4 | Global debug flag stamped: `global_state.debug_font_fallback = debug_font_fallback ? true : false` | `kitty/state.c` | state.c:740 |
| 5 | Font resolution: `set_font_family(opts)` called | `kitty/main.py` → `kitty/fonts/render.py` `set_font_family` | main.py:251; render.py:173 |
| 6 | Python font discovery: `get_font_files(opts)` (medium/bold/italic/bi) | `kitty/fonts/common.py` `get_font_files` / `get_font_from_spec` | common.py:281, 243 |
| 7 | Platform dispatch: Linux → `kitty/fonts/fontconfig.py` `find_best_match` / `all_fonts_map` / `fc_match` | `kitty/fonts/fontconfig.py` | 170, 47, 81 |
| 8 | Symbol maps constructed: `create_symbol_map(opts)` / `create_narrow_symbols(opts)` | `kitty/fonts/render.py` | inside `set_font_family` |
| 9 | Native handoff: `set_font_data(render_box_drawing, prerender_function, descriptor_for_idx, ...)` | `kitty/fonts/render.py` → `kitty/fonts.c` | render.py:189; fonts.c:1433–1447 |
| 10 | Window creation: `create_os_window(...)` | `kitty/main.py` `_run_app` | main.py:221 |
| 11 | Font group allocated for window, `initialize_font_group(fg)` | `kitty/fonts.c` | 1494–1517 |
| 12 | Faces initialized: `initialize_font(fg, 0, "medium")`, bold/italic/bi via `I()` macro, plus symbol-map faces | `kitty/fonts.c` `initialize_font` | 1476–1492 |
| 13 | Cell metrics computed: `calc_cell_metrics(fg)` → backend `cell_metrics` → apply `OPT(...)` adjustments → `sprite_tracker_set_layout(fg, ...)` | `kitty/fonts.c` → `kitty/freetype.c` | fonts.c:1511, 372–422; freetype.c:387 |
| 14 | Symbol fonts rescaled to `fg->cell_height` via `set_size_for_face(...)` | `kitty/fonts.c` | 1515 |
| 15 | Shaders loaded and compiled: `load_all_shaders()` | `kitty/main.py` → `kitty/shaders.c` | main.py (GLFW init); shaders.c (shader compile) |
| 16 | Boss/Child monitor instantiated | `kitty/main.py` (`_run_app`) | ~200–240 |
| 17 | First window displayed → `send_prerendered_sprites_for_window(w)` | `kitty/fonts.c` | 1520–1527 |
| 18 | Sprite map allocated: `alloc_sprite_map(cell_width, cell_height)` — first call queries `GL_MAX_TEXTURE_SIZE` / `GL_MAX_ARRAY_TEXTURE_LAYERS`, caps at 8192/512 on Apple, calls `sprite_tracker_set_limits` | `kitty/shaders.c` | 50–70 |
| 19 | Initial 11 sprites uploaded: `send_prerendered_sprites(fg)` — blank cell at (0,0,0), then `prerender_function(...)` returns 10 sprite alpha masks (5 underlines, 1 strikethrough, 1 missing glyph, 3 cursors), each composited with `render_alpha_mask` and uploaded via `current_send_sprite_to_gpu` | `kitty/fonts.c` → `kitty/fonts/render.py` → `kitty/shaders.c` `send_sprite_to_gpu` | fonts.c:1449–1473; render.py:364–396; shaders.c:146–156 |
| 20 | Lazy GL texture allocation: `ensure_sprite_map(fg)` → `realloc_sprite_texture(fg)` — creates `GL_TEXTURE_2D_ARRAY` with `glTexStorage3D(..., 1, GL_SRGB8_ALPHA8, width, height, znum)`, `GL_NEAREST`/`GL_CLAMP_TO_EDGE` | `kitty/shaders.c` | 107–144 |
| 21 | Optional post-startup: if `args.debug_font_fallback`, `dump_font_debug()` prints resolved fonts to stderr | `kitty/main.py` → `kitty/fonts/render.py` | main.py:228–229; render.py:161–170 |
| 22 | Main loop: `boss.child_monitor.main_loop()` — per-line shaping, fallback dispatch, sprite uploads happen on demand | `kitty/main.py` / `kitty/boss.py` / C main loop | — |

### Execution notes

- Steps 1–10 are strictly single-threaded Python.
- Step 11 (`initialize_font_group`) runs synchronously on the main thread inside the native `create_os_window` flow.
- Step 18 (`alloc_sprite_map`) is the first GPU query, so it requires an active GL context — that context exists by this point because `create_os_window` already called GLFW context creation.
- Steps 19 and 20 happen during the first `draw_cells` call after the window is shown; the atlas is lazy-initialized on first draw.
- Once step 22's main loop starts, font fallback and sprite allocation are entirely on-demand: `load_fallback_font` and `sprite_position_for` are called as new glyphs are encountered.

---

## Appendix B: Summary of Key Debug Output Formats

This table lists every debug/warning/fatal output produced by the startup-time font, metric, and texture-atlas paths, in the exact format produced by the C source.

| Condition | Source file : approximate line | Exact format |
|---|---|---|
| Fallback font selected for a cell | `kitty/fonts.c` 457–467 | `U+<hex-ch> [U+<hex-cc> ...] [bold ][italic ][emoji_presentation ][using previous fallback font at index: ]<face-repr>\n` |
| Fallback font returned but doesn't contain the requested glyph | `kitty/fonts.c` 501–509 | `The font chosen by the OS for the text: U+<hex-ch> [U+<hex-cc> ...] is <face-repr> but it does not actually contain glyphs for that text\n` |
| Cell height increased due to buggy underscore font | `kitty/freetype.c` 146–148 | `Increasing cell height by %u pixels to work around buggy font that renders underscore outside the bounding box\n` |
| `dump_font_debug` header for Normal/Bold/Italic/Bold-Italic faces | `kitty/fonts/render.py` 163–166 | `Text fonts:\n  <kind>: <postscript>:<path>:<index>\n` |
| `dump_font_debug` Symbol map section | `kitty/fonts/render.py` 167–170 | `Symbol map fonts:\n  <postscript>:<path>:<index>\n` |
| `debug_config` Fonts section | `kitty/debug_config.py` 260–263 | `Fonts:\n  <key>: <postscript>:<path>:<index>\n` |
| `debug_config` OpenGL section | `kitty/debug_config.py` 258 (format from `kitty/gl.c` 42–49) | `OpenGL: '<glGetString(GL_VERSION)>' Detected version: <major>.<minor>\n` |
| One-time `glCopyImageSubData` unavailable warning | `kitty/shaders.c` 87–91 | `WARNING: Your system's OpenGL implementation does not have glCopyImageSubData, falling back to a slower implementation\n` |
| Too many pre-rendered sprites (fatal) | `kitty/fonts.c` 1463 | `Too many pre-rendered sprites for your GPU or the font size is too large\n` |
| Sprite map out of texture space (via `sprite_map_set_error`, set as Python exception) | `kitty/fonts.c` 225–234 (string at line 230) | `Out of texture space for sprites` |
| Sprite map OOM (fatal) | `kitty/shaders.c` 64 | `Out of memory allocating a sprite map\n` |

### Notes on format strings

- Angle-bracketed placeholders (`<face-repr>`, `<hex-ch>`, etc.) are runtime-substituted values.
- Square-bracketed segments (`[bold ]`, `[italic ]`, etc.) are conditional — emitted only when the flag is set.
- `<face-repr>` is produced by `PyObject_Print(face, stderr, 0)`, which for FontConfig-backed faces prints the Python repr of the `FontConfigPattern` dict-like object (path, family, style, weight, slant, spacing).
- `<postscript>:<path>:<index>` is `identify_for_debug`'s format from `kitty/freetype.c` lines 737–743: `PyUnicode_FromFormat("%s: %V:%d", FT_Get_Postscript_Name(face), path, "[path]", instance.val)`.
- All `debug_fonts(...)` / `debug(...)` calls in `fonts.c` are gated on `global_state.debug_font_fallback`, via the `debug_fonts` macro at `kitty/state.h` line 16 and the alias `#define debug debug_fonts` at `kitty/fonts.c` line 18.
- The sprite-space error emitted by `sprite_map_set_error` is raised as a Python exception via `PyErr_SetString(PyExc_RuntimeError, ...)` (`kitty/fonts.c` line 230) and bubbled up, rather than printed directly to stderr.

---

## Appendix C: File & Function Reference Table

This appendix lists every source file cited in this document with the relevant functions or macros, their approximate line numbers, and the role each plays in the startup path.

| File | Function / Macro | Lines | Role |
|---|---|---|---|
| `kitty/cli.py` | `--debug-font-fallback` option | 1002–1005 | CLI flag definition (`type=bool-set`) |
| `kitty/main.py` | `AppRunner.__call__` | 247 | Startup orchestration after CLI parse |
| `kitty/main.py` | `_run_app` | ~200–240 | Core window-creation and Boss startup |
| `kitty/main.py` | `dump_font_debug` invocation | 228–229 | Conditional debug dump after Boss start |
| `kitty/state.c` | `PYWRAP1(set_options)` | 726–744 | Applies CLI flags to `global_state` |
| `kitty/state.h` | `OPT(name)` macro | 13 | Shortcut for `global_state.opts.name` |
| `kitty/state.h` | `debug_rendering` macro | 14 | Gated `timed_debug_print` for rendering path |
| `kitty/state.h` | `debug_fonts` macro | 16 | Gated `timed_debug_print` for font path |
| `kitty/state.h` | `AdjustmentUnit` enum | 27 | `POINT=0, PERCENT=1, PIXEL=2` |
| `kitty/state.h` | `force_ltr` field | 75 | Option value on `GlobalState.opts` |
| `kitty/state.h` | `debug_font_fallback` field | 270 | Global flag on `GlobalState` |
| `kitty/fonts/render.py` | `dump_font_debug` | 161–170 | Prints Text/Symbol fonts via `identify_for_debug` |
| `kitty/fonts/render.py` | `set_font_family` | 173–193 | Resolves faces, computes symbol maps, calls native `set_font_data` |
| `kitty/fonts/render.py` | `prerender_function` | 364–396 | Rasterizes decoration and cursor alpha masks |
| `kitty/fonts/render.py` | `render_special` | 284– | Per-glyph special rendering |
| `kitty/fonts/common.py` | `get_font_files` | 281–296 | Resolves medium/bold/italic/bi files |
| `kitty/fonts/common.py` | `get_font_from_spec` | 243–261 | Dispatches spec to `find_best_match` |
| `kitty/fonts/fontconfig.py` | `all_fonts_map` | 47–54 | Calls `fc_list(FC_DUAL) + fc_list(FC_MONO)` |
| `kitty/fonts/fontconfig.py` | `fc_match` | 81–83 | lru-cached fontconfig match |
| `kitty/fonts/fontconfig.py` | `find_best_match` | 170–213 | Exact-match or fontconfig fallback |
| `kitty/fonts/fontconfig.py` | `find_last_resort_text_font` | 164–167 | fc_match on "monospace" as ultimate fallback |
| `kitty/fonts.c` | `#define debug debug_fonts` | 18 | Alias for brevity in the fonts path |
| `kitty/fonts.c` | `sprite_tracker_set_limits` | 236–240 | Propagates GPU caps to fonts.c-local globals |
| `kitty/fonts.c` | `do_increment` | 242–253 | Advances sprite (x, y, z) write head |
| `kitty/fonts.c` | `sprite_position_for` | 256–266 | Cache lookup + allocation for glyph sprites |
| `kitty/fonts.c` | `sprite_tracker_current_layout` | 268–272 | Reads current xnum/ynum/z for realloc |
| `kitty/fonts.c` | `sprite_tracker_set_layout` | 275–281 | Computes xnum, max_y from cell dimensions |
| `kitty/fonts.c` | `init_font` | 293–328 | Per-face feature setup (liga/dlig/calt) |
| `kitty/fonts.c` | `adjust_metric` | 350–363 | POINT/PERCENT/PIXEL → integer offset |
| `kitty/fonts.c` | `adjust_ypos` | 366–370 | Clamps adjusted y-position within cell |
| `kitty/fonts.c` | `calc_cell_metrics` | 372–422 | Orchestrates metric computation and user adjustments |
| `kitty/fonts.c` | `has_emoji_presentation` | 430–432 | emoji + width==2 + not VS15 |
| `kitty/fonts.c` | `has_cell_text` | 434–454 | Face-contains-glyph check |
| `kitty/fonts.c` | `output_cell_fallback_data` | 456–468 | Debug-prints fallback face choice |
| `kitty/fonts.c` | `load_fallback_font` | 480–517 | Bounded (≤100) fallback face loading |
| `kitty/fonts.c` | `fallback_font` | 519–544 | Cache-first fallback dispatch |
| `kitty/fonts.c` | `font_for_cell` | 563–605 | Dispatch: BLANK/BOX/symbol/main/fallback |
| `kitty/fonts.c` | `load_hb_buffer` | 671–689 | Populates HB buffer, applies force_ltr |
| `kitty/fonts.c` | `shape` | 786–820 | Calls `hb_shape` with feature toggles |
| `kitty/fonts.c` | `shape_run` | ~1152 | Per-run shaping loop |
| `kitty/fonts.c` | `render_line` | ~1326 | Orchestrates per-line rendering |
| `kitty/fonts.c` | `send_prerendered_sprites` | 1449–1473 | Uploads blank + 10 pre-rendered sprites |
| `kitty/fonts.c` | `initialize_font` | 1476–1492 | Resolves and initializes a single face |
| `kitty/fonts.c` | `initialize_font_group` | 1494–1517 | Whole-group initialization + metric calc |
| `kitty/fonts.c` | `send_prerendered_sprites_for_window` | 1520–1527 | First-window sprite-map alloc + upload |
| `kitty/fonts.c` | `init_fonts` | 1745–1761 | Module init: creates HB buffer + -liga/-dlig/-calt features |
| `kitty/fonts.h` | `cell_metrics` prototype | ~27 | Backend-neutral cell metric API |
| `kitty/fonts.h` | `create_fallback_face` prototype | ~29 | Backend-neutral fallback creation |
| `kitty/freetype.c` | `font_units_to_pixels_y` | 91–94 | `ceil(FT_MulFix(x, y_scale) / 64)` |
| `kitty/freetype.c` | `font_units_to_pixels_x` | 96–99 | `ceil(FT_MulFix(x, x_scale) / 64)` |
| `kitty/freetype.c` | `calc_cell_height` | 141–152 | `font_units_to_pixels_y(height)` + underscore fix |
| `kitty/freetype.c` | `calc_cell_width` | 374–383 | Max ASCII horiAdvance / 64 |
| `kitty/freetype.c` | `cell_metrics` | 387–405 | Baseline + underline + strikethrough |
| `kitty/freetype.c` | `identify_for_debug` | 738–743 | `postscript: path:index` formatting |
| `kitty/fontconfig.c` | `create_fallback_face` | 462–487 | Runtime fontconfig-based fallback |
| `kitty/shaders.c` | `SpriteMap` struct | 24–29 | GPU atlas state |
| `kitty/shaders.c` | `NEW_SPRITE_MAP` | 31 | Struct initializer |
| `kitty/shaders.c` | `alloc_sprite_map` | 50–70 | Queries GL caps, allocates state |
| `kitty/shaders.c` | `copy_image_sub_data` | 84–104 | Fast/slow texture copy for resize |
| `kitty/shaders.c` | `realloc_sprite_texture` | 107–134 | `glTexStorage3D` allocation |
| `kitty/shaders.c` | `ensure_sprite_map` | 137–144 | Lazy init + bind for draw |
| `kitty/shaders.c` | `send_sprite_to_gpu` | 146–156 | `glTexSubImage3D` cell upload |
| `kitty/glyph-cache.c` | `find_or_create_sprite_position` | — | uthash lookup / insert keyed on glyph IDs |
| `kitty/glyph-cache.h` | `SpritePosition`, `GlyphProperties` | — | Hash entry structs |
| `kitty/unicode-data.h` | `VS15 = 1364, VS16 = 1365` | 5 | Variation selector mark indices |
| `kitty/unicode-data.h` | `is_combining_char` | ~11 | `codepoint → combining_type` |
| `kitty/unicode-data.h` | `codepoint_for_mark` | 17 | `combining_type → codepoint` |
| `kitty/unicode-data.h` | `mark_for_codepoint` | 18 | `codepoint → combining_type` (inverse) |
| `kitty/emoji.h` | `is_emoji` | ~10 | Emoji classification |
| `kitty/data-types.h` | `CPUCell.cc_idx[3]` | ~226 | Up to 3 combining chars per cell |
| `kitty/data-types.h` | `NUM_UNDERLINE_STYLES = 5u` | ~213 | Decoration style count |
| `kitty/data-types.h` | `FONTS_DATA_HEAD` macro | ~347 | Font-group header shared between C/backend |
| `kitty/options/definition.py` | `font_family` | ~35 | Font family option |
| `kitty/options/definition.py` | `force_ltr` | 64–82 | Override HarfBuzz auto-direction to LTR |
| `kitty/options/definition.py` | `symbol_map` | 84–97 | Codepoint-range → font mapping |
| `kitty/options/definition.py` | `narrow_symbols` | 99–112 | Narrow-width symbol map |
| `kitty/options/definition.py` | `disable_ligatures` | 115–133 | `never` / `always` / `cursor` |
| `kitty/options/definition.py` | `font_features` | ~135 | Per-face HarfBuzz feature overrides |
| `kitty/debug_config.py` | `debug_config` Fonts section | 260–263 | Prints `current_fonts()` via `identify_for_debug` |

---

## Appendix D: Runtime Verification

While the primary methodology of this investigation is source-code static analysis, the build environment permitted limited runtime introspection of the native `kitty/fast_data_types.so` extension via `kitty +runpy`. This appendix records the results of runtime verification performed inside the Docker container to corroborate specific claims in the main body. No source files were modified to produce these observations.

### D.1 Build and Version Verification

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

Binary at `kitty/launcher/kitty` (36224 bytes). Build command per the project setup: `python3 setup.py build --ignore-compiler-warnings`.

### D.2 Structural Constants Match the Source

Confirmed by running `./kitty/launcher/kitty +runpy` with a short introspection script:

| Constant (Python) | Runtime Value | Source Citation |
|---|---|---|
| `NUM_UNDERLINE_STYLES` | `5` | `kitty/data-types.h` line 213 |
| `CELL_PROGRAM` | `0` | shader program enum in `kitty/shaders.c` |
| `BORDERS_PROGRAM` | `4` | shader program enum in `kitty/shaders.c` |
| `GRAPHICS_PROGRAM` | `5` | shader program enum in `kitty/shaders.c` |

The `NUM_UNDERLINE_STYLES = 5` value is what drives the prerender count in §3.6: 5 underline sprites + 1 strikethrough + 1 missing + 3 cursors = 10 prerendered sprites plus the blank at index 0.

### D.3 Fontconfig-Based Font Discovery (Observed)

Invoking `kitty.fonts.fontconfig.all_fonts_map(True)` (monospaced=True) in this container returned the following:

```python
{
    'family_map': ['dejavu sans mono', 'liberation mono'],  # keys (lowercased)
    'ps_map':     ['dejavusansmono', 'dejavusansmono-bold',
                   'dejavusansmono-boldoblique', 'dejavusansmono-oblique',
                   'liberationmono', 'liberationmono-bold',
                   'liberationmono-bolditalic', 'liberationmono-italic'],
    'full_map':   ['dejavu sans mono', 'dejavu sans mono bold',
                   'dejavu sans mono bold oblique', 'dejavu sans mono oblique',
                   'liberation mono', 'liberation mono bold',
                   'liberation mono bold italic', 'liberation mono italic'],
    'variable_map': {},
}
```

This confirms the structure described in §2.2 and validates the following claims from `kitty/fonts/fontconfig.py` lines 46–54:

- `all_fonts_map` returns a dict with four keys: `family_map`, `ps_map`, `full_map`, `variable_map`.
- Keys in `family_map` and `full_map` are lowercased family / full names; `ps_map` uses the PostScript name as the key.
- Each value is a `Descriptor` list (full fontconfig dict per entry, with `path`, `style`, `spacing`, `postscript_name`, etc.).

### D.4 `fc_match()` Fallback Resolution (Observed)

The behavior described in §2.3 (`create_fallback_face` → `_fc_match`) was exercised directly by calling `kitty.fast_data_types.fc_match('monospace', False, FC_WEIGHT_REGULAR, FC_SLANT_ROMAN, codepoint, False)` for three test codepoints:

| Codepoint | Character | Chosen Descriptor (excerpt) |
|---|---|---|
| `U+0041` | `A` (English) | `family='DejaVu Sans Mono'`, `style='Oblique'`, `path='/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf'`, `spacing='MONO'` |
| `U+0627` | `ا` (Arabic ALIF) | `family='DejaVu Sans Mono'`, `style='Oblique'`, `path='/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf'`, `spacing='MONO'` |
| `U+05D0` | `א` (Hebrew ALEF) | `family='DejaVu Sans Mono'`, `style='Oblique'`, `path='/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf'`, `spacing='MONO'` |

In this specific container environment, DejaVu Sans Mono happens to provide Arabic and Hebrew coverage, so **no fallback face would actually be created at runtime for Arabic/Hebrew text** on a system configured with DejaVu — `has_cell_text(medium_face, cell)` would return true, and the path described in §2.4 (`load_fallback_font`) is short-circuited.

This concretely illustrates the "depends on the host's installed fonts" caveat in §2.6: on a system without DejaVu but with only a Latin-covering font (e.g., a minimal installation with only Liberation Mono), Arabic codepoints *would* trigger `fallback_font`, producing debug output of the form cited in §2.5. The selection of DejaVu by the Ubuntu 24.04 base `fontconfig` defaults is the reason both scripts resolved to the same face here.

Note the descriptor fields returned by `fc_match` — `weight=80` (FC_WEIGHT_REGULAR is 80 in fontconfig), `width=100` (FC_WIDTH_NORMAL), `slant=110` (FC_SLANT_OBLIQUE — the match happened to land on the Oblique style due to unrelated fontconfig substitution rules in the container) — these are the exact field names and integer values written to stderr by `PyObject_Print(face, stderr, 0)` when `output_cell_fallback_data` runs with `--debug-font-fallback`.

### D.5 Headless Window Creation Fails (Expected)

```text
$ ./kitty/launcher/kitty --debug-font-fallback
[0.059] [glfw error 65544]: X11: The DISPLAY environment variable is missing
GLFW initialization failed
```

Exit code 1. This is the expected behavior inside a headless Docker container: `init_glfw()` in `kitty/main.py` calls `glfwInit()` which fails on missing DISPLAY. Because the sprite atlas requires an OpenGL context (as described in §4.2 and §4.4), no actual `glTexStorage3D` allocation can be observed from inside this container. The per-window startup sequence in Appendix A step 6 onward is therefore only verifiable by code review, not by direct measurement, in this environment. All GPU-related values in §4 are therefore derived from the C source formulas — not measured — per the caveat in the Environment and Methodology section.

### D.6 Summary of Runtime Verification Outcome

- ✔ `NUM_UNDERLINE_STYLES = 5` confirmed at runtime — matches `kitty/data-types.h` line 213.
- ✔ Shader program IDs (`CELL_PROGRAM=0`, `BORDERS_PROGRAM=4`, `GRAPHICS_PROGRAM=5`) confirmed at runtime.
- ✔ `all_fonts_map()` structure (four-key dict with `family_map`/`ps_map`/`full_map`/`variable_map`) confirmed — matches `kitty/fonts/fontconfig.py` lines 46–54.
- ✔ `fc_match()` returns a descriptor dict with expected keys (`descriptor_type`, `path`, `family`, `style`, `postscript_name`, `spacing`, `weight`, `width`, `slant`, `color`, etc.) — matches the repr format that `output_cell_fallback_data` prints.
- ✖ Per-window startup sequence (sprite atlas allocation, prerendered sprite upload) cannot be exercised in a headless container — verified only through source code analysis.
- ✖ Actual Arabic-to-non-DejaVu fallback cannot be observed in this container because DejaVu Sans Mono covers U+0600–U+06FF; the debug output format is therefore quoted verbatim from `kitty/fonts.c` lines 457–467, not captured live.

These observations corroborate the static-analysis claims made in Sections 1–4 where runtime verification was feasible, and honestly disclose the limits of runtime verification where it was not.

---

## Document End

This investigation is based on static reading of the kitty source tree at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, supplemented by limited runtime verification of the built native extension `kitty/fast_data_types.so` (see Appendix D). Every formula, constant, and debug format in the document has a direct citation to a file and line in the kitty source tree. Runtime-dependent values (actual font chosen by fontconfig, actual `GL_MAX_TEXTURE_SIZE`, actual cell dimensions after `modify_font`) are noted as dependent on the specific runtime environment and are not guessed. No source files in the kitty repository were modified during this investigation.
