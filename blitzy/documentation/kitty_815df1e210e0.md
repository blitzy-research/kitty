# Kitty Terminal Emulator: Font Rendering and GPU Subsystem Initialization Analysis

**Investigative analysis based on the kitty source code at branch `kitty_815df1e210e0`**

## Executive Summary

This document presents a comprehensive, code-grounded investigation of the kitty terminal emulator's font rendering pipeline and GPU subsystem initialization. It answers four core technical questions about the startup-time behavior of kitty's text shaping engine (HarfBuzz), font fallback chain construction, cell metric computation, and GPU texture atlas allocation. Every claim in this document is traced to specific source files, functions, and line numbers within the kitty codebase. No source files were modified during this investigation.

**Key Architectural Findings:**

- HarfBuzz text shaping is initialized per-face via `hb_ft_font_create()` in `kitty/freetype.c`, with a single global `hb_buffer_t` reused across all shaping operations (`kitty/fonts.c`, line 41)
- Font fallback chains are **not pre-built at startup** — they are constructed **lazily** at glyph-rendering time when unmapped codepoints are first encountered (`kitty/fonts.c:load_fallback_font()`)
- Cell metrics (width, height, baseline, underline/strikethrough positions and thicknesses) are computed from FreeType font units in `kitty/freetype.c:cell_metrics()` and then adjusted via `modify_font` directives in `kitty/fonts.c:calc_cell_metrics()`
- The GPU sprite map uses a `GL_TEXTURE_2D_ARRAY` with `GL_SRGB8_ALPHA8` format, dynamically grown from a single layer as glyphs are cached (`kitty/shaders.c:realloc_sprite_texture()`)
- kitty does **not** implement full Unicode Bidirectional Algorithm (UAX #9); HarfBuzz provides per-run direction detection only

---

## Table of Contents

1. [Text Shaping and Unicode Configuration at Startup](#section-1--text-shaping-and-unicode-configuration-at-startup)
   - 1.1 [HarfBuzz Font Initialization](#11-harfbuzz-font-initialization)
   - 1.2 [Global HarfBuzz Buffer and Default Features](#12-global-harfbuzz-buffer-and-default-features)
   - 1.3 [Per-Font Feature Application](#13-per-font-feature-application)
   - 1.4 [HarfBuzz Buffer Loading](#14-harfbuzz-buffer-loading)
   - 1.5 [The Shaping Call](#15-the-shaping-call)
   - 1.6 [Bidirectional Text Handling and Limitations](#16-bidirectional-text-handling-and-limitations)
   - 1.7 [Combining Diacritical Marks Support](#17-combining-diacritical-marks-support)
2. [Font Families and Fallback Chains in Startup Diagnostics](#section-2--font-families-and-fallback-chains-in-startup-diagnostics)
   - 2.1 [The `--debug-font-fallback` CLI Flag](#21-the---debug-font-fallback-cli-flag)
   - 2.2 [`dump_font_debug()` Output Format](#22-dump_font_debug-output-format)
   - 2.3 [Font Resolution Path for the Default Family](#23-font-resolution-path-for-the-default-family)
   - 2.4 [Lazy Fallback Chain Architecture](#24-lazy-fallback-chain-architecture)
   - 2.5 [Platform-Specific Fallback Resolution](#25-platform-specific-fallback-resolution)
   - 2.6 [Mixed Arabic/English Text Behavior](#26-mixed-arabicenglish-text-behavior)
   - 2.7 [`debug_config()` Output](#27-debug_config-output)
3. [Cell Metrics, Baseline, and Decoration Alignment](#section-3--cell-metrics-baseline-and-decoration-alignment)
   - 3.1 [`cell_metrics()` Implementation](#31-cell_metrics-implementation)
   - 3.2 [`calc_cell_metrics()` Adjustments](#32-calc_cell_metrics-adjustments)
   - 3.3 [Pre-rendered Sprites](#33-pre-rendered-sprites)
   - 3.4 [Overline Note](#34-overline-note)
4. [GPU Texture Atlas Initialization](#section-4--gpu-texture-atlas-initialization)
   - 4.1 [`alloc_sprite_map()` Behavior](#41-alloc_sprite_map-behavior)
   - 4.2 [Sprite Tracker Limits and Layout](#42-sprite-tracker-limits-and-layout)
   - 4.3 [`realloc_sprite_texture()` — Texture Allocation](#43-realloc_sprite_texture--texture-allocation)
   - 4.4 [Initial Sprite Population](#44-initial-sprite-population)
   - 4.5 [Atlas Growth Mechanism](#45-atlas-growth-mechanism)
   - 4.6 [`send_sprite_to_gpu()` — Per-Glyph Upload](#46-send_sprite_to_gpu--per-glyph-upload)
5. [Debug Observation Methods](#section-5--debug-observation-methods)
6. [Startup Flow Summary](#section-6--startup-flow-summary)
7. [Dependency Versions](#section-7--dependency-versions)

---

## Section 1 — Text Shaping and Unicode Configuration at Startup

This section traces how kitty's text shaping engine (HarfBuzz) and font pipeline configure support for complex Unicode scripts — including OpenType ligatures, bidirectional text, and combining diacritical marks — during the initial startup sequence.

### 1.1 HarfBuzz Font Initialization

**Source:** `kitty/freetype.c:init_ft_face()`, lines 211–240

When a new font face is loaded, `init_ft_face()` performs the following HarfBuzz initialization:

1. **Copy font metrics from FreeType:** The function copies critical metrics from the FreeType `FT_Face` struct into kitty's `Face` struct, including `ascender`, `descender`, `height`, `max_advance_width`, `underline_position`, and `underline_thickness` (line 214).

2. **Detect face capabilities:** Flags `is_scalable` (via `FT_IS_SCALABLE`), `has_color` (via `FT_HAS_COLOR`), and `is_variable` (via `FT_HAS_MULTIPLE_MASTERS`) are set (lines 216–218).

3. **Set font size:** If a `FontGroup` handle (`fg`) is provided, `set_size_for_face()` is called to configure the FreeType face to the desired point size and DPI (line 225).

4. **Create HarfBuzz font:** `hb_ft_font_create(self->face, NULL)` creates a HarfBuzz font object backed by the FreeType face (line 226). This is the primary integration point between FreeType's glyph rasterization and HarfBuzz's text shaping.

5. **Align load flags:** `hb_ft_font_set_load_flags(self->harfbuzz_font, get_load_flags(self->hinting, self->hintstyle, FT_LOAD_DEFAULT))` synchronizes HarfBuzz's glyph loading flags with kitty's hinting configuration (line 228). The `get_load_flags()` function (lines 102–110) maps kitty's hinting settings to FreeType load flags:
   - `hinting` enabled + `hintstyle >= 3` → `FT_LOAD_TARGET_NORMAL`
   - `hinting` enabled + `0 < hintstyle < 3` → `FT_LOAD_TARGET_LIGHT`
   - `hinting` disabled → `FT_LOAD_NO_HINTING`

6. **Extract strikethrough metrics from OS/2 table:** The TrueType OS/2 table is queried via `FT_Get_Sfnt_Table(self->face, FT_SFNT_OS2)` (line 230). If present, `self->strikethrough_position = os2->yStrikeoutPosition` and `self->strikethrough_thickness = os2->yStrikeoutSize` are stored (lines 232–233).

7. **Cache space glyph ID:** `self->space_glyph_id = glyph_id_for_codepoint((PyObject*)self, ' ')` caches the glyph index for the space character for fast lookup during rendering (line 238).

### 1.2 Global HarfBuzz Buffer and Default Features

**Source:** `kitty/fonts.c`, lines 41–46

The shaping subsystem maintains the following module-level static state:

```
static hb_buffer_t *harfbuzz_buffer = NULL;           // line 41
static hb_feature_t hb_features[3] = {{0}};           // line 42
static char_type shape_buffer[4096] = {0};             // line 43
static size_t max_texture_size = 1024, max_array_len = 1024;  // line 44
typedef enum { LIGA_FEATURE, DLIG_FEATURE, CALT_FEATURE } HBFeature;  // line 45
```

**Key observations:**

- **Single global buffer:** A single `hb_buffer_t` is created once and reused for all shaping operations across all font groups and windows. This avoids per-shape allocation overhead.
- **Three default feature slots:** The `hb_features[3]` array holds three HarfBuzz OpenType feature tags:
  - Index 0 (`LIGA_FEATURE`): Standard ligatures (`liga`)
  - Index 1 (`DLIG_FEATURE`): Discretionary ligatures (`dlig`)
  - Index 2 (`CALT_FEATURE`): Contextual alternates (`calt`)
- **Shape buffer:** A 4096-element `char_type` (UTF-32) array serves as the staging area for codepoints before they are passed to `hb_buffer_add_utf32()`.

### 1.3 Per-Font Feature Application

**Source:** `kitty/fonts.c:init_font()`, lines 293–327

When a `Font` struct is initialized, the feature set is determined as follows:

**Case 1 — Custom `font_features` configuration exists for this font** (lines 299–316):

If the `font_feature_settings` Python dict (populated from the user's `font_features` config option) contains an entry matching the font's PostScript name, those features are loaded into a newly-allocated `hb_feature_t` array. The **CALT feature is always appended as the last element** (line 314: `memcpy(f->ffs_hb_features + len, &hb_features[CALT_FEATURE], sizeof(hb_feature_t))`). This means CALT (contextual alternates) is unconditionally enabled for every font.

**Case 2 — No custom features** (lines 318–326):

If no custom features exist for the font, a default feature set is constructed:
- **CALT is always enabled** (line 325: `memcpy(f->ffs_hb_features + f->num_ffs_hb_features++, &hb_features[CALT_FEATURE], sizeof(hb_feature_t))`)
- **LIGA and DLIG are additionally enabled only for NimbusMonoPS fonts** (lines 321–323): The condition `strstr(psname, "NimbusMonoPS-") == psname` checks if the PostScript name starts with `"NimbusMonoPS-"`. Only for this specific font family are standard and discretionary ligatures enabled by default.

**Rationale:** Most monospace fonts do not have ligature features, and enabling LIGA/DLIG globally could cause unexpected behavior. NimbusMonoPS is special-cased because it is a common system monospace font that benefits from ligature support.

### 1.4 HarfBuzz Buffer Loading

**Source:** `kitty/fonts.c:load_hb_buffer()`, lines 671–689

This function fills the global HarfBuzz buffer with cell content for shaping:

1. **Clear previous contents:** `hb_buffer_clear_contents(harfbuzz_buffer)` resets the buffer (line 674).

2. **Iterate cells and collect codepoints** (lines 675–684):
   - For each cell, the base character `first_cpu_cell->ch` is placed into `shape_buffer` (line 679).
   - Wide characters (width 2) cause the next cell to be skipped (line 678: `if (prev_width == 2) { prev_width = 0; continue; }`).
   - **Combining characters** from `first_cpu_cell->cc_idx[]` are resolved to full Unicode codepoints via `codepoint_for_mark()` and appended to the shape buffer immediately after the base character (line 682: `shape_buffer[num++] = codepoint_for_mark(first_cpu_cell->cc_idx[i])`).

3. **Add to HarfBuzz buffer:** `hb_buffer_add_utf32(harfbuzz_buffer, shape_buffer, num, 0, num)` transfers the collected UTF-32 codepoints to the HarfBuzz buffer (line 685).

4. **Auto-detect segment properties:** `hb_buffer_guess_segment_properties(harfbuzz_buffer)` (line 687) instructs HarfBuzz to automatically determine:
   - **Script** (e.g., `HB_SCRIPT_ARABIC`, `HB_SCRIPT_LATIN`, `HB_SCRIPT_DEVANAGARI`)
   - **Direction** (e.g., `HB_DIRECTION_RTL` for Arabic, `HB_DIRECTION_LTR` for Latin)
   - **Language** (from system locale)

5. **Force-LTR override:** If the `force_ltr` option is enabled, the auto-detected direction is overridden: `hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR)` (line 688: `if (OPT(force_ltr)) hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR)`).

### 1.5 The Shaping Call

**Source:** `kitty/fonts.c:shape()`, lines 785–820

The `shape()` function orchestrates the complete shaping pipeline:

1. **Load the HarfBuzz buffer:** `load_hb_buffer(first_cpu_cell, first_gpu_cell, num_cells)` fills the buffer as described above (line 809).

2. **Determine feature count:** `size_t num_features = fobj->num_ffs_hb_features` gets the per-font feature count (line 811).

3. **Ligature suppression:** When `disable_ligature` is true, `num_features` is decremented by 1 (line 812: `if (num_features && !disable_ligature) num_features--`). Since the **last feature is always CALT** (contextual alternates), this effectively disables CALT for that shaping run. Note the logic: when `disable_ligature` is false, `num_features` is decremented (the `-calt` feature, which disables CALT, is excluded), meaning CALT remains active. When `disable_ligature` is true, the full feature array including the `-calt` entry is used, actually disabling CALT.

4. **Execute shaping:** `hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features)` (line 813) performs the actual OpenType shaping, applying GSUB (glyph substitution) and GPOS (glyph positioning) tables.

5. **Extract results:** Glyph info and positions are retrieved via `hb_buffer_get_glyph_infos()` and `hb_buffer_get_glyph_positions()` (lines 816–817).

### 1.6 Bidirectional Text Handling and Limitations

**kitty does NOT implement the full Unicode Bidirectional Algorithm (UAX #9).** This is a critical architectural fact.

**What kitty does provide:**

- **Per-run direction detection** via `hb_buffer_guess_segment_properties()` (line 687 of `kitty/fonts.c`). When a buffer contains Arabic text, HarfBuzz detects `HB_SCRIPT_ARABIC` and sets `HB_DIRECTION_RTL`. For Latin text, it detects `HB_SCRIPT_LATIN` and sets `HB_DIRECTION_LTR`.
- **`force_ltr` override** (`kitty/state.h`, line 75: `bool force_ltr`): When enabled, all text is forced to `HB_DIRECTION_LTR` regardless of script detection.

**What kitty does NOT provide:**

- No full BiDi reordering of mixed RTL/LTR text within a single line
- No implementation of the Unicode Bidirectional Algorithm (UAX #9)
- No character-level reordering — words in RTL scripts display in reversed visual order at the word level only

**The `force_ltr` option documentation** (from `kitty/options/definition.py`) explicitly states that "kitty does not support BIDI (bidirectional text)." For full BiDi support, users are directed to use GNU FriBidi as an external filter in conjunction with `force_ltr=yes`.

**Configuration defaults:**
- `force_ltr` defaults to `false` (`kitty/state.h`, line 75)

### 1.7 Combining Diacritical Marks Support

**Storage mechanism:**

- Each `CPUCell` has a `cc_idx[]` array that stores combining marks as compressed indices (defined in `kitty/data-types.h`).
- These compressed indices are resolved to full Unicode codepoints via `codepoint_for_mark()` (declared in `kitty/unicode-data.h`).

**Integration with HarfBuzz:**

- In `load_hb_buffer()` (line 682), combining characters are appended to the shape buffer immediately after their base character: `shape_buffer[num++] = codepoint_for_mark(first_cpu_cell->cc_idx[i])`.
- HarfBuzz then applies GPOS mark-to-base and mark-to-mark attachment rules from the font's OpenType tables to correctly position the combining marks.

**Precomposed form detection:**

- In `has_cell_text()` (`kitty/fonts.c`, line 447), HarfBuzz's Unicode compose function is used: `hb_unicode_compose(hb_unicode_funcs_get_default(), cell->ch, combining_chars[0], &ch)`. This checks whether a base character + combining character pair has a precomposed equivalent in the font.

**Unicode Standard coverage:**

- `kitty/unicode-data.c` is built from **Unicode Standard 15.0.0** (line 1: `"Unicode data, built from the Unicode Standard 15.0.0"`).
- The `is_combining_char()` function covers **6424 codepoints** (line 12: `"Combining and default ignored characters (6424 codepoints)"`).
- Arabic diacritics coverage includes:
  - U+0610–U+061A (line 33: `case 0x610 ... 0x61a`)
  - U+064B–U+065F (line 37: `case 0x64b ... 0x65f`)
  - U+0670 (line 39: `case 0x670`)
  - U+06D6–U+06DD, U+06DF–U+06E4, U+06E7–U+06E8, U+06EA–U+06ED (lines 41–48)

---

## Section 2 — Font Families and Fallback Chains in Startup Diagnostics

This section documents what font families and fallback chains kitty selects at launch, how to observe them via debug flags, and the critical architectural distinction between eager and lazy font loading.

### 2.1 The `--debug-font-fallback` CLI Flag

**Source:** `kitty/cli.py`, lines 1002–1005

The `--debug-font-fallback` CLI flag is defined as a `bool-set` type, meaning it sets a boolean to true when present on the command line.

**Activation chain:**

1. The CLI flag is parsed and stored in `args.debug_font_fallback`.
2. In `kitty/main.py:run_app()` (line 249), `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` propagates the value to the C-level global state.
3. This sets `global_state.debug_font_fallback = true` in the C runtime.
4. The C macro `debug_fonts(...)` in `kitty/state.h` (line 16) is gated on this flag:
   ```c
   #define debug_fonts(...) if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); }
   ```
5. Additionally, in `kitty/main.py:_run_app()` (lines 228–229), after the boss is started, `dump_font_debug()` is called if `args.debug_font_fallback` is set:
   ```python
   if args.debug_font_fallback:
       dump_font_debug()
   ```

### 2.2 `dump_font_debug()` Output Format

**Source:** `kitty/fonts/render.py:dump_font_debug()`, lines 161–170

This function provides a snapshot of the currently loaded fonts at startup:

```python
def dump_font_debug() -> None:
    cf = current_fonts()
    log_error('Text fonts:')
    for key, text in {'medium': 'Normal', 'bold': 'Bold', 'italic': 'Italic', 'bi': 'Bold-Italic'}.items():
        log_error(f'  {text}:', cf[key].identify_for_debug())
    ss = cf['symbol']
    if ss:
        log_error('Symbol map fonts:')
        for s in ss:
            log_error('  ' + s.identify_for_debug())
```

**Output structure:**
- `"Text fonts:"` header, followed by:
  - `"  Normal:"` + face debug identifier
  - `"  Bold:"` + face debug identifier
  - `"  Italic:"` + face debug identifier
  - `"  Bold-Italic:"` + face debug identifier
- `"Symbol map fonts:"` header (if any symbol map fonts are configured), followed by each face's debug identifier

**`identify_for_debug()` format:**

**Source:** `kitty/freetype.c:identify_for_debug()`, lines 737–743

```c
static PyObject*
identify_for_debug(PyObject *s, PyObject *a UNUSED) {
    Face *self = (Face*)s;
    FaceIndex instance;
    instance.val = self->face->face_index;
    return PyUnicode_FromFormat("%s: %V:%d", FT_Get_Postscript_Name(self->face), self->path, "[path]", instance.val);
}
```

This produces output in the format: `"PostScriptName: /path/to/font.ttf:face_index"` (e.g., `"DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0"`).

**Representative example output** (derived from the code's format strings — actual values depend on the system's fonts):

```
Text fonts:
  Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

> **Note:** The above example is representative and derived from the code's format strings. Actual font names depend on the system's installed fonts and FontConfig/CoreText defaults.

### 2.3 Font Resolution Path for the Default Family

**Source:** `kitty/fonts/common.py:get_font_files()`, lines 281–296

The font resolution path for the default `monospace` family:

1. `get_font_files(opts)` is called from `set_font_family()` in `kitty/fonts/render.py` (line 177).
2. For the medium (normal) font: `get_font_from_spec(opts.font_family, ...)` is called.
3. When `spec.system == 'auto'` (the default), the family resolves to `'monospace'`.
4. `find_best_match('monospace', bold=False, italic=False)` is called via the platform-specific backend:
   - **Linux:** `kitty/fonts/fontconfig.py:find_best_match()` uses FontConfig to match the system's default monospace font.
   - **macOS:** `kitty/fonts/core_text.py:find_best_match()` uses CoreText for font matching.
5. Bold, italic, and bold-italic variants are resolved via `get_font_from_spec()` with appropriate bold/italic flags (lines 289–295), referencing the medium font for variable font variant discovery.

### 2.4 Lazy Fallback Chain Architecture

> **CRITICAL ARCHITECTURAL FACT: No static fallback chain is pre-built at startup.**

This is the most important architectural detail regarding font fallback in kitty. The fallback font chain is constructed **lazily** — fonts are loaded on-demand only when a cell contains a codepoint that the primary font cannot render.

**The lazy loading flow:**

**Source:** `kitty/fonts.c`

1. **`font_for_cell()`** (lines 563–605) determines which font handles each cell:
   - Check if the cell is blank or contains a box-drawing character → use box font
   - Check symbol maps → use symbol map font if matched
   - Select bold/italic variant based on cell attributes
   - Call `has_cell_text()` to check if the selected font has glyphs for the cell's text
   - On miss → call `fallback_font()`

2. **`fallback_font()`** (lines 519–544):
   - Constructs a cache key from the cell's text content and style flags (bold, italic, emoji_presentation)
   - Checks the `fallback_font_map` hash table (using `uthash` `HASH_FIND_STR`) for a cached result (lines 528–532)
   - On cache hit → returns the cached font index immediately
   - On cache miss → calls `load_fallback_font()`
   - Stores the result in the `fallback_font_map` hash table for future lookups (lines 535–541)

3. **`load_fallback_font()`** (lines 480–517):
   - Guards against excessive fallback fonts: `if (fg->fallback_fonts_count > 100)` returns `MISSING_FONT` (line 482)
   - Determines the base font to derive from (medium, bold, italic, or bold-italic) (lines 485–487)
   - Calls the platform-specific `create_fallback_face()` to find a font that can render the cell's codepoints (line 489)
   - If `global_state.debug_font_fallback` is true, calls `output_cell_fallback_data()` to print debug info to stderr (line 492)
   - If a previously loaded fallback face is returned (as a Python integer index), reuses it (line 493)
   - Otherwise: initializes a new `Font` struct, sets its size to match the cell height, verifies it can actually render the text, and adds it to the font group's fallback array (lines 494–516)

**Why this matters for debug output:** Because fallback fonts are loaded lazily, the `--debug-font-fallback` output appears **incrementally** as new codepoints are encountered during rendering, NOT as a complete dump at startup time. The only startup-time font dump is `dump_font_debug()` which shows the four primary faces (Normal, Bold, Italic, Bold-Italic) and any symbol map fonts.

### 2.5 Platform-Specific Fallback Resolution

#### Linux (FontConfig)

**Source:** `kitty/fontconfig.c:create_fallback_face()`, lines 462–487

1. **Pattern creation:** An `FcPattern` is created via `FcPatternCreate()` (line 466).
2. **Family specification:** The family is set to `"monospace"` for text or `"emoji"` for emoji presentation (line 468: `FcPatternAddString, FC_FAMILY, (const FcChar8*)(emoji_presentation ? "emoji" : "monospace")`).
3. **Style specification:** Bold weight and italic slant are added if requested (lines 469–470).
4. **Charset matching:** The cell's Unicode codepoints are added to an `FcCharSet` via `FcCharSetAddChar()` (line 473: `add_charset(pat, num)`), ensuring FontConfig only returns fonts that contain the needed characters.
5. **Font matching:** `_fc_match(pat)` calls `FcFontMatch()` (through dynamically-loaded `libfontconfig.so` symbols) to find the best match (line 474).
6. **Deduplication:** Before creating a new face, existing fallback faces are checked via `iter_fallback_faces()` and `face_equals_descriptor()` (lines 478–479). If a match is found, the existing index is returned to avoid duplicate face loading.
7. **Face creation:** If no existing face matches, `face_from_descriptor()` creates a new face from the matched descriptor (line 481).

**Note:** FontConfig is dynamically loaded at runtime via `dlopen("libfontconfig.so")` (`kitty/fontconfig.c`, lines 19–20), not statically linked. This means FontConfig symbols are loaded through function pointers stored in the `dynamically_loaded_fc_symbol` struct (lines 45+).

#### macOS (CoreText)

**Source:** `kitty/core_text.m:create_fallback_face()`

On macOS, the fallback resolution uses CoreText's `CTFontCreateForString` API, which provides native system-level fallback font selection. CoreText examines the input string and returns a font that can render the specified characters, taking into account the system's font configuration and user preferences.

### 2.6 Mixed Arabic/English Text Behavior

For mixed Arabic (RTL) and English (LTR) text, the following behavior occurs:

1. **Script auto-detection:** When `load_hb_buffer()` is called with a line containing both Arabic and English text, `hb_buffer_guess_segment_properties()` detects the dominant script. For a buffer containing primarily Arabic text, it would detect `HB_SCRIPT_ARABIC` and `HB_DIRECTION_RTL`; for primarily Latin text, `HB_SCRIPT_LATIN` and `HB_DIRECTION_LTR`.

2. **Fallback for Arabic codepoints:** If the primary monospace font (e.g., DejaVu Sans Mono) lacks Arabic glyphs, `font_for_cell()` will call `fallback_font()` for each cell containing Arabic characters. This invokes `create_fallback_face()` which queries FontConfig for a font matching the `"monospace"` family with the Arabic codepoint's charset. The OS's FontConfig configuration determines which Arabic-capable font is selected (e.g., Noto Sans Arabic Mono, DejaVu Sans Mono with Arabic coverage, or similar).

3. **Debug output for fallback:** When `global_state.debug_font_fallback` is true, `output_cell_fallback_data()` (`kitty/fonts.c`, lines 457–468) prints fallback information to stderr for each cell:
   ```
   U+<codepoint> [bold] [italic] [emoji_presentation] <face_object>
   ```
   Where `U+<codepoint>` is the hex codepoint (e.g., `U+0627` for Arabic letter Alef), optional style flags, and the selected font face printed via `PyObject_Print(face, stderr, 0)`.

4. **Important limitation:** Since kitty does not implement full BiDi, mixed Arabic/English text will not be properly reordered within a line. Arabic words will be shaped correctly (with proper glyph joining via HarfBuzz's Arabic shaper) but the visual ordering of mixed-direction text segments will not follow the Unicode Bidirectional Algorithm.

### 2.7 `debug_config()` Output

**Source:** `kitty/debug_config.py`

The `debug_config()` function (invoked via `kitty --debug-config` or `Ctrl+Shift+F6`) provides a comprehensive snapshot of the runtime configuration:

- **Fonts section:** Iterates `current_fonts()` and calls `identify_for_debug()` on each face, printing the PostScript name and path for Normal, Bold, Italic, and Bold-Italic faces.
- **OpenGL version:** Prints the OpenGL version string via `opengl_version_string()`.
- **Configuration diffs:** Uses `compare_opts()` to show configuration values that differ from defaults, including `font_family`, `font_size`, `force_ltr`, and other settings.

---

## Section 3 — Cell Metrics, Baseline, and Decoration Alignment

This section documents the default cell metrics, baseline positioning, and decoration alignment values computed during the screen grid and shaping subsystem initialization.

### 3.1 `cell_metrics()` Implementation

**Source:** `kitty/freetype.c:cell_metrics()`, lines 386–405

The `cell_metrics()` function computes seven metric values from the medium font's FreeType data:

#### cell_width

**Source:** `kitty/freetype.c:calc_cell_width()`, lines 373–383

```c
static unsigned int
calc_cell_width(Face *self) {
    unsigned int ans = 0;
    for (char_type i = 32; i < 128; i++) {
        int glyph_index = FT_Get_Char_Index(self->face, i);
        if (load_glyph(self, glyph_index, FT_LOAD_DEFAULT)) {
            ans = MAX(ans, (unsigned int)ceilf((float)self->face->glyph->metrics.horiAdvance / 64.f));
        }
    }
    return ans;
}
```

The cell width is the **maximum horizontal advance** across all printable ASCII characters (codepoints 32–127), computed as `ceil(horiAdvance / 64.0)` (converting from 26.6 fixed-point to integer pixels). This ensures all ASCII characters fit within the cell grid.

#### cell_height

**Source:** `kitty/freetype.c:calc_cell_height()`, lines 140–152

```c
static unsigned int
calc_cell_height(Face *self, bool for_metrics) {
    unsigned int ans = font_units_to_pixels_y(self, self->height);
    if (for_metrics) {
        unsigned int underscore_height = get_height_for_char(self, '_');
        if (underscore_height > ans) {
            // ... increase cell height
            return underscore_height;
        }
    }
    return ans;
}
```

The cell height is `font_units_to_pixels_y(self, self->height)`, which computes `ceil(FT_MulFix(height, y_scale) / 64.0)` (line 93 of `freetype.c`). An **underscore overflow correction** is applied: if the underscore character (`_`) renders below the nominal bounding box, the cell height is increased to accommodate it. When `global_state.debug_font_fallback` is true, a diagnostic message is printed indicating the pixel increase.

#### baseline

```c
*baseline = font_units_to_pixels_y(self, self->ascender);   // line 391
```

The baseline is computed as `ceil(FT_MulFix(ascender, y_scale) / 64.0)` — the ascender value in pixels, which represents the distance from the top of the cell to the baseline.

#### underline_position

```c
*underline_position = MIN(*cell_height - 1,
    (unsigned int)font_units_to_pixels_y(self, MAX(0, self->ascender - self->underline_position)));  // line 392
```

The underline position is computed as `ascender - underline_position` (in font units), converted to pixels, and clamped to `cell_height - 1` to ensure it stays within the cell. The `underline_position` value from the font's `post` table represents the distance below the baseline, so `ascender - underline_position` converts it to a distance from the top of the cell.

#### underline_thickness

```c
*underline_thickness = MAX(1, font_units_to_pixels_y(self, self->underline_thickness));  // line 393
```

The underline thickness is converted from font units to pixels with a minimum of 1 pixel, ensuring underlines are always visible.

#### strikethrough_position

```c
if (self->strikethrough_position != 0) {
    *strikethrough_position = MIN(*cell_height - 1,
        (unsigned int)font_units_to_pixels_y(self, MAX(0, self->ascender - self->strikethrough_position)));  // lines 395-396
} else {
    *strikethrough_position = (unsigned int)floor(*baseline * 0.65);  // line 398
}
```

If the OS/2 table provides `yStrikeoutPosition`, it is converted from font units to pixels (relative to the top of the cell). Otherwise, a **fallback heuristic** of `floor(baseline * 0.65)` is used, placing the strikethrough at approximately 65% of the way from the top of the cell to the baseline.

#### strikethrough_thickness

```c
if (self->strikethrough_thickness > 0) {
    *strikethrough_thickness = MAX(1, font_units_to_pixels_y(self, self->strikethrough_thickness));  // lines 400-401
} else {
    *strikethrough_thickness = *underline_thickness;  // line 403
}
```

If the OS/2 table provides `yStrikeoutSize`, it is converted to pixels (minimum 1). Otherwise, the underline thickness is reused as the strikethrough thickness.

### 3.2 `calc_cell_metrics()` Adjustments

**Source:** `kitty/fonts.c:calc_cell_metrics()`, lines 372–422

After `cell_metrics()` computes the raw values, `calc_cell_metrics()` applies user-configurable adjustments:

#### Step 1: Get raw metrics (line 375)

```c
cell_metrics(fg->fonts[fg->medium_font_idx].face, &cell_width, &cell_height, &baseline,
             &underline_position, &underline_thickness, &strikethrough_position, &strikethrough_thickness);
```

#### Step 2: Apply `modify_font` cell_width/cell_height adjustments (lines 378–380)

```c
unsigned int before_cell_height = cell_height;
unsigned int cw = cell_width, ch = cell_height;
adjust_metric(&cw, OPT(cell_width).val, OPT(cell_width).unit, fg->logical_dpi_x);
adjust_metric(&ch, OPT(cell_height).val, OPT(cell_height).unit, fg->logical_dpi_y);
```

The `adjust_metric()` function (lines 350–363) supports three unit types:
- **POINT:** `a = round(adj * (dpi / 72.0))` — adjustment in typographic points
- **PERCENT:** `*metric = round(|adj| * metric / 100)` — percentage of current value
- **PIXEL:** `a = round(adj)` — direct pixel adjustment

#### Step 3: Enforce bounds (lines 381–395)

```c
#define MAX_DIM 1000
#define MIN_WIDTH 2
#define MIN_HEIGHT 4
```

Cell dimensions are clamped: width must be in [2, 1000] and height in [4, 1000]. Invalid adjustments are logged and ignored.

#### Step 4: Compute line height adjustment (line 388)

```c
int line_height_adjustment = cell_height - before_cell_height;
```

This tracks how much the cell height changed due to `modify_font` adjustments, used later for baseline and underline repositioning.

#### Step 5: Apply `modify_font` adjustments for decorations (lines 397–400)

```c
unsigned int baseline_before = baseline;
#define A(which, dpi) adjust_metric(&which, OPT(which).val, OPT(which).unit, fg->logical_dpi_##dpi);
    A(underline_thickness, y); A(underline_position, y);
    A(strikethrough_thickness, y); A(strikethrough_position, y); A(baseline, y);
#undef A
```

Each metric can be individually adjusted via the corresponding `modify_font` configuration option.

#### Step 6: Baseline adjustment propagation (lines 402–407)

```c
if (baseline_before != baseline) {
    int adjustment = baseline - baseline_before;
    baseline = adjust_ypos(baseline_before, cell_height, adjustment);
    underline_position = adjust_ypos(underline_position, cell_height, adjustment);
    strikethrough_position = adjust_ypos(strikethrough_position, cell_height, adjustment);
}
```

If the baseline was adjusted by `modify_font`, the `adjust_ypos()` function (lines 365–370) shifts all y-position metrics proportionally while respecting cell bounds.

#### Step 7: Underline position clamping (line 409)

```c
underline_position = MIN(cell_height - 1, underline_position);
```

#### Step 8: Line height adjustment compensation (lines 414–417)

```c
if (line_height_adjustment > 1) {
    baseline += MIN(cell_height - 1, (unsigned)line_height_adjustment / 2);
    underline_position += MIN(cell_height - 1, (unsigned)line_height_adjustment / 2);
}
```

When the cell height has increased by more than 1 pixel (from `modify_font cell_height`), the baseline and underline position are shifted down by half the adjustment. This centers the text vertically within the enlarged cell.

#### Step 9: Store final values (lines 418–421)

```c
sprite_tracker_set_layout(&fg->sprite_tracker, cell_width, cell_height);
fg->cell_width = cell_width; fg->cell_height = cell_height;
fg->baseline = baseline; fg->underline_position = underline_position;
fg->underline_thickness = underline_thickness;
fg->strikethrough_position = strikethrough_position;
fg->strikethrough_thickness = strikethrough_thickness;
```

The final seven metric values are stored in the `FontGroup` struct and the sprite tracker layout is configured for the new cell dimensions.

### 3.3 Pre-rendered Sprites

**Source:** `kitty/fonts/render.py:prerender_function()`, lines 364–396

The `prerender_function()` receives all cell metrics (cell_width, cell_height, baseline, underline_position, underline_thickness, strikethrough_position, strikethrough_thickness, cursor_beam_thickness, cursor_underline_thickness, dpi_x, dpi_y) and pre-renders the following sprites:

1. **Underline styles** (lines 391): `NUM_UNDERLINE_STYLES` (5) styles via `render_special()`:
   - Style 1: Single line
   - Style 2: Double line
   - Style 3: Curly (undercurl)
   - Style 4: Dotted
   - Style 5: Dashed

2. **Strikethrough** (line 392): `f(0, strikethrough=True)` — a horizontal line at `strikethrough_position` with `strikethrough_thickness`.

3. **Missing glyph** (line 393): `f(missing=True)` — rendered by `render_missing_glyph()` from `kitty/fonts/box_drawing.py`.

4. **Cursor shapes** (line 394): Three cursor sprites:
   - `c(1)`: Beam cursor (vertical line on left edge)
   - `c(2)`: Underline cursor (horizontal line at bottom edge)
   - `c(3)`: Hollow cursor (rectangle outline)

These are called from `send_prerendered_sprites()` in `kitty/fonts.c` (line 1458).

### 3.4 Overline Note

There is **no explicit "overline" metric** in kitty's implementation. Overline is not a built-in decoration style. The decoration system supports only underline (5 styles) and strikethrough.

---

## Section 4 — GPU Texture Atlas Initialization

This section details the initial page layout, sizing, capacity allocations, and startup behavior of the sprite map / texture atlas used by the GPU rendering pipeline.

### 4.1 `alloc_sprite_map()` Behavior

**Source:** `kitty/shaders.c:alloc_sprite_map()`, lines 50–70

The `alloc_sprite_map()` function is called once per OS window when the font group's sprite map is first needed (`kitty/fonts.c:send_prerendered_sprites_for_window()`, lines 1520–1527).

#### GPU limit queries (lines 52–54):

On first invocation (when `max_texture_size == 0`):

```c
glGetIntegerv(GL_MAX_TEXTURE_SIZE, &(max_texture_size));
glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, &(max_array_texture_layers));
```

These OpenGL queries determine the GPU's maximum 2D texture dimension and maximum number of array texture layers.

**Typical values on modern GPUs:**
- `GL_MAX_TEXTURE_SIZE`: 16384 (most modern GPUs)
- `GL_MAX_ARRAY_TEXTURE_LAYERS`: 2048 (most modern GPUs)

#### macOS caps (lines 55–59):

```c
#ifdef __APPLE__
max_texture_size = MIN(8192, max_texture_size);
max_array_texture_layers = MIN(512, max_array_texture_layers);
#endif
```

On macOS, the values are capped to `8192` and `512` respectively, because Apple systems may have multiple GPUs with different capabilities (as noted in the code comment referencing Apple's OpenGL capabilities page).

#### Sprite tracker initialization (line 61):

```c
sprite_tracker_set_limits(max_texture_size, max_array_texture_layers);
```

#### SpriteMap struct allocation (lines 63–69):

```c
SpriteMap *ans = calloc(1, sizeof(SpriteMap));
*ans = NEW_SPRITE_MAP;
ans->max_texture_size = max_texture_size;
ans->max_array_texture_layers = max_array_texture_layers;
ans->cell_width = cell_width; ans->cell_height = cell_height;
```

The `NEW_SPRITE_MAP` initializer (line 31) sets: `.xnum = 1, .ynum = 1, .last_num_of_layers = 1, .last_ynum = -1`.

### 4.2 Sprite Tracker Limits and Layout

#### `sprite_tracker_set_limits()`

**Source:** `kitty/fonts.c`, lines 236–240

```c
void
sprite_tracker_set_limits(size_t max_texture_size_, size_t max_array_len_) {
    max_texture_size = max_texture_size_;
    max_array_len = MIN(0xfffu, max_array_len_);
}
```

The `max_array_len` is capped at `0xfff` (4095) regardless of GPU capability. This is because sprite z-coordinates use 16-bit storage and the upper bits are reserved for flags (e.g., the colored glyph flag at bit 14).

#### `sprite_tracker_set_layout()`

**Source:** `kitty/fonts.c`, lines 275–281

```c
static void
sprite_tracker_set_layout(GPUSpriteTracker *sprite_tracker, unsigned int cell_width, unsigned int cell_height) {
    sprite_tracker->xnum = MIN(MAX(1u, max_texture_size / cell_width), (size_t)UINT16_MAX);
    sprite_tracker->max_y = MIN(MAX(1u, max_texture_size / cell_height), (size_t)UINT16_MAX);
    sprite_tracker->ynum = 1;
    sprite_tracker->x = 0; sprite_tracker->y = 0; sprite_tracker->z = 0;
}
```

**Grid calculation:**

| Parameter | Formula | Description |
|-----------|---------|-------------|
| `xnum` | `MIN(MAX(1, max_texture_size / cell_width), UINT16_MAX)` | Number of sprite columns per layer |
| `max_y` | `MIN(MAX(1, max_texture_size / cell_height), UINT16_MAX)` | Maximum number of sprite rows per layer |
| `ynum` | `1` (initial) | Current number of rows in use (grows dynamically) |
| `x, y, z` | `0, 0, 0` (initial) | Current write position in the atlas |

**Example capacity calculation:**

For typical values of `cell_width=8`, `cell_height=16`, `max_texture_size=16384`:

| Metric | Calculation | Value |
|--------|-------------|-------|
| `xnum` | `16384 / 8` | 2048 columns |
| `max_y` | `16384 / 16` | 1024 rows |
| Glyphs per layer | `2048 × 1024` | 2,097,152 |
| Maximum layers | `4095` (capped) | — |
| **Theoretical maximum** | `2048 × 1024 × 4095` | **~8.5 billion glyphs** |

In practice, only a fraction of this capacity is ever used, since the atlas grows dynamically from a single row.

### 4.3 `realloc_sprite_texture()` — Texture Allocation

**Source:** `kitty/shaders.c:realloc_sprite_texture()`, lines 107–134

This function creates or grows the GPU texture that stores all glyph sprites:

#### Texture creation (lines 109–117):

```c
GLuint tex;
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_2D_ARRAY, tex);
glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D_ARRAY, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
```

**Key parameters:**
- **Texture type:** `GL_TEXTURE_2D_ARRAY` — a 3D array of 2D textures, where each layer stores a grid of glyph sprites
- **Filtering:** `GL_NEAREST` for both minification and magnification — this prevents inter-cell glyph bleeding from bilinear interpolation
- **Wrapping:** `GL_CLAMP_TO_EDGE` — prevents texture coordinate wrapping artifacts

#### Storage allocation (lines 118–123):

```c
unsigned int xnum, ynum, z, znum, width, height, src_ynum;
sprite_tracker_current_layout(fg, &xnum, &ynum, &z);
znum = z + 1;
SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
width = xnum * sprite_map->cell_width;
height = ynum * sprite_map->cell_height;
glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum);
```

**Texture dimensions:**
- **Width:** `xnum × cell_width` (e.g., 2048 × 8 = 16384 pixels)
- **Height:** `ynum × cell_height` (starts at 1 × 16 = 16 pixels, grows as needed)
- **Depth:** `z + 1` layers (starts at 1, grows as needed)
- **Internal format:** `GL_SRGB8_ALPHA8` — sRGB color space with 8-bit alpha channel. This ensures proper gamma-correct blending of text glyphs.
- **Mipmap levels:** 1 (no mipmapping, since `GL_NEAREST` filtering is used)

#### Texture growth (lines 124–133):

When the texture needs to grow (because `sprite_map->texture_id` is already set):

```c
if (sprite_map->texture_id) {
    src_ynum = MAX(1, sprite_map->last_ynum);
    copy_image_sub_data(sprite_map->texture_id, tex, width,
                        src_ynum * sprite_map->cell_height, sprite_map->last_num_of_layers);
    glDeleteTextures(1, &sprite_map->texture_id);
}
```

Existing glyph data is copied via `copy_image_sub_data()` (lines 84–103):
- **Primary path:** `glCopyImageSubData()` from the `GL_ARB_copy_image` extension — a GPU-side copy with no CPU round-trip
- **Fallback path:** If `GL_ARB_copy_image` is unavailable, a slow `glGetTexImage()` + `glTexSubImage3D()` round-trip through CPU memory is used (with a warning logged once)

### 4.4 Initial Sprite Population

**Source:** `kitty/fonts.c:send_prerendered_sprites()`, lines 1449–1473

After `alloc_sprite_map()` creates the sprite map, `send_prerendered_sprites()` populates it with the initial set of pre-rendered sprites:

| Sprite Index | Content | Source |
|:---:|---------|--------|
| 0 | **Blank cell** — all zeros (transparent) | Lines 1453–1456: `ensure_canvas_can_fit(fg, 1)` then `current_send_sprite_to_gpu()` with cleared canvas |
| 1 | **Underline style 1** — single line | `prerender_function()` in `render.py`, line 391 |
| 2 | **Underline style 2** — double line | `prerender_function()` in `render.py`, line 391 |
| 3 | **Underline style 3** — curly (undercurl) | `prerender_function()` in `render.py`, line 391 |
| 4 | **Underline style 4** — dotted | `prerender_function()` in `render.py`, line 391 |
| 5 | **Underline style 5** — dashed | `prerender_function()` in `render.py`, line 391 |
| 6 | **Strikethrough** | `prerender_function()` in `render.py`, line 392 |
| 7 | **Missing glyph placeholder** | `prerender_function()` in `render.py`, line 393; constant `MISSING_GLYPH = NUM_UNDERLINE_STYLES + 2` in `kitty/fonts.c`, line 15 |
| 8 | **Cursor: beam** (vertical line) | `prerender_function()` in `render.py`, line 394: `c(1)` |
| 9 | **Cursor: underline** (horizontal line) | `prerender_function()` in `render.py`, line 394: `c(2)` |
| 10 | **Cursor: hollow** (rectangle outline) | `prerender_function()` in `render.py`, line 394: `c(3)` |

**Total: 11 pre-rendered sprites** (1 blank + 5 underline styles + 1 strikethrough + 1 missing glyph + 3 cursor shapes).

The blank cell at index 0 is sent first (lines 1453–1456), then `prerender_function()` is called (line 1458) to generate the remaining 10 sprites. Each sprite is rendered as an alpha mask, converted to RGBA via `render_alpha_mask()` with color `0xffffff`, and uploaded via `current_send_sprite_to_gpu()`.

### 4.5 Atlas Growth Mechanism

**Source:** `kitty/fonts.c:do_increment()`, lines 242–253

The atlas starts with a single texture layer and **grows dynamically** as more glyphs are cached:

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

**Growth sequence:**
1. `x` increments across columns within a row
2. When `x` reaches `xnum` → `x` resets to 0, `y` increments (new row)
3. `ynum` is updated to track the maximum row used (for texture reallocation)
4. When `y` reaches `max_y` → `y` resets to 0, `z` increments (new layer)
5. When `z` reaches `MIN(UINT16_MAX, max_array_len)` → error 2 (out of texture space)

**There is NO pre-allocation of all font glyph pages at startup.** The texture is allocated with only enough layers and rows to hold the current sprites, and `realloc_sprite_texture()` is called automatically when `send_sprite_to_gpu()` detects that the current allocation is insufficient.

### 4.6 `send_sprite_to_gpu()` — Per-Glyph Upload

**Source:** `kitty/shaders.c:send_sprite_to_gpu()`, lines 146–156

```c
void
send_sprite_to_gpu(FONTS_DATA_HANDLE fg, unsigned int x, unsigned int y, unsigned int z, pixel *buf) {
    SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
    unsigned int xnum, ynum, znum;
    sprite_tracker_current_layout(fg, &xnum, &ynum, &znum);
    if ((int)znum >= sprite_map->last_num_of_layers || (znum == 0 && (int)ynum > sprite_map->last_ynum))
        realloc_sprite_texture(fg);
    glBindTexture(GL_TEXTURE_2D_ARRAY, sprite_map->texture_id);
    glPixelStorei(GL_UNPACK_ALIGNMENT, 4);
    x *= sprite_map->cell_width; y *= sprite_map->cell_height;
    glTexSubImage3D(GL_TEXTURE_2D_ARRAY, 0, x, y, z,
                    sprite_map->cell_width, sprite_map->cell_height, 1,
                    GL_RGBA, GL_UNSIGNED_INT_8_8_8_8, buf);
}
```

**Key details:**
- Before uploading, checks if the texture needs reallocation (more layers or rows than currently allocated)
- Upload call: `glTexSubImage3D()` with format `GL_RGBA` and type `GL_UNSIGNED_INT_8_8_8_8`
- The (x, y, z) grid coordinates are converted to pixel coordinates by multiplying by cell dimensions
- Each sprite occupies exactly one `cell_width × cell_height` region in the texture

---

## Section 5 — Debug Observation Methods

All debug flags are **transient CLI arguments** — no persistent files are modified when using them.

| Debug Flag | CLI Argument | C Macro / Python Check | Output Location |
|---|---|---|---|
| Font fallback tracing | `--debug-font-fallback` | `global_state.debug_font_fallback` / `debug_fonts(...)` macro in `kitty/state.h` line 16 | stderr via `timed_debug_print()` |
| Font debug dump at startup | `--debug-font-fallback` | `args.debug_font_fallback` check in `kitty/main.py` line 228 | stderr via `dump_font_debug()` in `kitty/fonts/render.py` |
| Rendering/GL debug | `--debug-rendering` / `--debug-gl` | `global_state.debug_rendering` / `debug_rendering(...)` macro in `kitty/state.h` line 14 | stderr via `timed_debug_print()` |
| Configuration debug | `kitty --debug-config` (or Ctrl+Shift+F6) | `debug_config()` in `kitty/debug_config.py` | stdout (formatted) |

**OpenGL version requirements** (from `kitty/data-types.h`, lines 20–25):

| Platform | Minimum Version | Constant |
|----------|----------------|----------|
| Linux | OpenGL 3.1 (GLSL 140) | `OPENGL_REQUIRED_VERSION_MAJOR=3`, `OPENGL_REQUIRED_VERSION_MINOR=1` (line 24) |
| macOS | OpenGL 3.3 (GLSL 140) | `OPENGL_REQUIRED_VERSION_MAJOR=3`, `OPENGL_REQUIRED_VERSION_MINOR=3` (line 22) |

**Debug macro definitions** (`kitty/state.h`, lines 13–16):

```c
#define OPT(name) global_state.opts.name
#define debug_rendering(...) if (global_state.debug_rendering) { timed_debug_print(__VA_ARGS__); }
#define debug_input(...) if (OPT(debug_keyboard)) { timed_debug_print(__VA_ARGS__); }
#define debug_fonts(...) if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); }
```

These macros are no-ops when the corresponding debug flag is not set, ensuring zero runtime overhead in normal operation.

---

## Section 6 — Startup Flow Summary

The complete startup sequence from application entry to GPU-ready font rendering:

### Step 1: Application Entry

**Source:** `kitty/main.py:_main()`

`_main()` → `create_opts()` → `set_locale()` → `init_glfw()` → `run_app()`

### Step 2: Font System Initialization

**Source:** `kitty/main.py:run_app()` (line 247–249)

```python
def __call__(self, opts, args, bad_lines=(), talk_fd=-1):
    set_scale(opts.box_drawing_scale)
    set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)
    set_font_family(opts)
    _run_app(opts, args, bad_lines, talk_fd)
```

`set_options()` propagates debug flags to the C global state. `set_font_family()` initializes the font pipeline.

### Step 3: Font File Resolution

**Source:** `kitty/fonts/render.py:set_font_family()`, lines 173–193

1. `get_font_files(opts)` resolves medium/bold/italic/bi font descriptors from the configuration
2. `current_faces` list is built with font object tuples
3. `create_symbol_map(opts)` processes `symbol_map` configuration
4. `set_font_data()` pushes all resolved font data to the C layer

### Step 4: C-Level Font Data Storage

**Source:** `kitty/fonts.c:set_font_data()`, lines 1434–1447

Stores Python callbacks (`box_drawing_function`, `prerender_function`, `descriptor_for_idx`), font feature settings, symbol maps, and the font size. Calls `free_font_groups()` to clear prior state.

### Step 5: Font Group Initialization

**Source:** `kitty/fonts.c:initialize_font_group()`, lines 1494–1517

1. Allocates the `Font` array with capacity for primary fonts plus symbol map fonts (line 1496–1498)
2. Initializes the medium font (line 1501), then bold, italic, and bold-italic fonts (line 1502)
3. Initializes symbol map fonts (lines 1506–1509)
4. Calls `calc_cell_metrics()` to compute cell dimensions and decoration positions (line 1511)
5. Rescales symbol map fonts to match the computed cell height (lines 1513–1516)

### Step 6: GPU Sprite Map and Pre-rendered Sprites

**Source:** `kitty/fonts.c:send_prerendered_sprites_for_window()`, lines 1520–1527

When the first OS window is created:
1. `alloc_sprite_map(fg->cell_width, fg->cell_height)` queries GPU limits and creates the sprite map
2. `send_prerendered_sprites(fg)` uploads the 11 initial sprites (blank, underlines, strikethrough, missing glyph, cursors)

### Step 7: Debug Font Dump

**Source:** `kitty/main.py:_run_app()`, lines 228–229

After the boss is started and the first window is created:
```python
if args.debug_font_fallback:
    dump_font_debug()
```

If `--debug-font-fallback` is set, the loaded font information is printed to stderr.

---

## Section 7 — Dependency Versions

| Dependency | Version Constraint | Source | Purpose |
|---|---|---|---|
| **HarfBuzz** | ≥ 1.5 | Enforced at build time by `setup.py`: `at_least_version('harfbuzz', 1, 5)` | OpenType text shaping: ligatures, mark attachment, script/direction detection |
| **FreeType** | System library | Linked via `setup.py` build system | Glyph rasterization, cell metric computation, font face loading |
| **FontConfig** | System library | Dynamically loaded via `dlopen("libfontconfig.so")` in `kitty/fontconfig.c` (line 20) | Linux font discovery, matching, and fallback face creation |
| **CoreText** | System framework | macOS system framework | macOS font discovery, matching, and fallback via `CTFontCreateForString` |
| **OpenGL** | ≥ 3.1 (Linux) / ≥ 3.3 (macOS) | `kitty/data-types.h` lines 20–24 | GPU rendering: `GL_TEXTURE_2D_ARRAY`, `GL_SRGB8_ALPHA8`, sprite atlas |
| **GLSL** | 140 | `kitty/data-types.h` line 26: `#define GLSL_VERSION 140` | Shader language version for cell/fragment shaders |
| **Unicode Standard** | 15.0.0 | `kitty/unicode-data.c` line 1: `"Unicode data, built from the Unicode Standard 15.0.0"` | Character property tables: combining marks (6424 codepoints), emoji, symbols |
| **Python** | ≥ 3.8 | `pyproject.toml`: `requires-python = ">=3.8"` | Runtime interpreter |
| **GLFW** | 3.4 (vendored fork) | Vendored in `glfw/` directory | Platform windowing, input handling, OpenGL context management |
| **lcms2** | System library | Required by `setup.py` | ICC color management for sRGB gamma correction |
| **libpng** | System library | Required by `setup.py` | PNG decoding for graphics protocol and window icons |
| **Go** | 1.22 | `go.mod` | Go-based tooling |

---

*This document was generated through static analysis of the kitty terminal emulator source code. No source files were modified during the investigation. All debug flags referenced are transient CLI arguments that do not persist any changes to disk.*
