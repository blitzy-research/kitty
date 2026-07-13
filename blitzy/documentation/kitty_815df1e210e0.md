# kitty — Text-Shaping / Layout Configuration, Startup Font Fallback, Cell Metrics, and GPU Texture-Atlas Initialization

> **Repository:** `kovidgoyal/kitty` · **Source branch:** `kitty_815df1e210e0` · **HEAD:** `815df1e21` *("Wire up applying of font config")*
> **kitty version built & run:** `kitty 0.35.2 created by Kovid Goyal`
> **Investigation type:** strictly read-only, **runtime-observed** (RUN first, then write from what was observed).

This document answers four questions about how kitty configures its text-shaping/layout engine for complex Unicode, performs startup font fallback, computes cell metrics, and initializes its GPU texture atlas — **grounded in real runtime output captured from a canonical build**, with every behavioral claim placed next to the exact command, the complete unedited output, a `file:line` reference, and an **observed** / **INFERRED** label.

---

## 0. How to read this document (labels, streams, methodology)

* **`OBSERVED`** — a value/behavior captured from a real run (output shown verbatim, including the `[<seconds>] ` monotonic prefix emitted by `timed_debug_print` [`kitty/monotonic.h:99`]).
* **`INFERRED`** — a code-derived explanation used only where a value is not surfaced by a default log line; always grounded in `file:line`.
* **Output-stream split (critical).** kitty's font/render debug goes to **STDERR** (`timed_debug_print` → `fprintf(stderr, "[%.3f] ", …)` [`kitty/monotonic.h:99`]; the `debug_fonts` [`kitty/state.h:16`] and `debug_rendering` [`kitty/state.h:14`] macros route through it; `dump_font_debug` uses Python `log_error` → STDERR). **BUT** the GL version banner uses `printf(…)` → **STDOUT** [`kitty/gl.c:72`]. Every run below therefore captures **both** streams via `> log 2>&1`.
* **Debug gating.** With the flags OFF, the `debug_fonts`/`debug_rendering` macros are compile-time-guarded no-ops [`kitty/state.h:14,16`]; the flags **must** be passed to see any output. `--debug-font-fallback` additionally gates the per-cell fallback dump [`kitty/fonts.c:492`].
* **Canonical entry point.** All "launcher" runs use the real produced binary `./kitty/launcher/kitty`. Where a value is not surfaced by the launcher's default logging (see §1.4 for why), it is captured through a **thin Python binding over the same production C function** (e.g. `get_fallback_font` → `fallback_font` [`kitty/fonts.c:1678,520`]; `test_render_line` → `render_line` [`kitty/fonts.c:1591,1326`]; `test_shape` → `shape_run` [`kitty/fonts.c:1226`]; `create_test_font_group` → `send_prerendered_sprites` [`kitty/fonts.c:1450`]). The **non-canonical** `setup_for_testing` harness [`kitty/fonts/render.py:408`] is **never** used as primary evidence (it replaces the GPU upload with a Python callback and sets artificial sprite limits `sprite_map_set_limits(100000, 100)` [`kitty/fonts/render.py:421`]); see the honest negative in §6.3.

---

## 1. Canonical environment, build, and run commands

### 1.1 Toolchain (OBSERVED)

```console
$ python3 --version ; go version ; gcc --version | head -1
$ pkg-config --modversion harfbuzz fontconfig freetype2 gl
Python 3.13.7
go version go1.24.4 linux/amd64
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
10.2.0      # harfbuzz  (>= 1.5 required, setup.py:609)
2.15.0      # fontconfig
26.2.20     # freetype2
1.2         # gl
```

Container: `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (Ubuntu 25.10). Shaping = HarfBuzz `>= 1.5` [`setup.py:609`]; discovery/fallback = FontConfig; rasterization/metrics = FreeType; GPU = OpenGL. Python `>= 3.8` [`pyproject.toml:2`]; Go `1.22` [`go.mod:3`]. **No dependency was added, upgraded, or removed.**

### 1.2 Canonical build (OBSERVED)

The `Makefile` `all` target is `python3 setup.py` [`Makefile:12-13`]; the debug alternative is `python3 setup.py build --debug` [`Makefile:22-23`].

```console
$ python3 setup.py clean
$ CI=true python3 setup.py
...
Disabling building of wayland backend        # expected: headless build uses the X11 backend
...
[4/4] Linking launcher ...
# real 1m6.232s
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 40384 ... kitty/launcher/kitty
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The launcher binary is produced by `build_launcher` [`setup.py:1230`] and linked to `kitty/launcher/kitty` [`setup.py:1295`]. *(The "Disabling building of wayland backend" line is expected — the container has no `wayland-protocols`; the canonical Linux run uses the X11/GLFW backend under Xvfb.)*

### 1.3 Runtime prerequisites (OBSERVED)

* **Arabic-capable fonts** present so the RTL script can be shaped (37 faces incl. Amiri, Noto Naskh/Sans/Kufi Arabic, KacstOne). Proven with `fc-list :lang=ar family file | sort -u`.
* **OpenGL context** — real startup reaches atlas init only with a GL context; `gl_init` treats missing GL capabilities (e.g. `texture_storage`) as **fatal** [`kitty/gl.c`]. Headless **Xvfb** provides it:
  ```console
  $ Xvfb :99 -screen 0 1280x1024x24 -ac +extension GLX +render &
  $ export DISPLAY=:99
  $ glxinfo | grep -i "OpenGL core profile version"
  OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2   # llvmpipe
  ```

### 1.4 Canonical launcher invocation (OBSERVED)

```console
$ DISPLAY=:99 ./kitty/launcher/kitty --config NONE \
      --debug-font-fallback --debug-rendering \
      -o confirm_os_window_close=0 /tmp/kitty_obs/feed.sh > /tmp/kitty_obs/run1.log 2>&1
```

where `feed.sh` prints a **mixed Arabic (RTL) + English (LTR)** line (`Hello مرحبا World`, Arabic `U+0645 U+0631 U+062D U+0628 U+0627`) and a CJK line, then exits. The full unedited `run1.log` is in **Appendix A**; it was reproduced **3×** (`run1/run2/run3`), byte-identical modulo the `[<seconds>]` prefix (see §5).

> **Note on flags.** In this version `--debug-config` is **not** a launcher CLI flag (it is an in-app action); the launcher rejects it with `Unknown option: --debug-config`. The valid, documented startup diagnostics are `--debug-font-fallback` [`kitty/cli.py:1002`] and `--debug-rendering` / `--debug-gl` [`kitty/cli.py:989`]. (`--debug-input` / `--debug-keyboard` also exist [`kitty/cli.py:996`] but are out of scope here.)

> **Why some details come from production-function bindings, not the launcher (OBSERVED + INFERRED).** Under headless Xvfb with **no compositor**, the OS window is reported not-visible/occluded, so `should_os_window_be_rendered` returns false and `render_os_window` returns early — the per-cell **shape/fallback/glyph-upload** path does not execute in the launcher. (Atlas *allocation* + prerendered upload still happen at window creation — proven by the GL banner + absence of any fatal.) To capture the shaping/fallback/metric/atlas-capacity **details** deterministically, each is exercised through the **same production C function** via its thin binding (named per §0), run ≥2× for stability. This is explicitly **not** `setup_for_testing`.

---

## 2. Objective A — Shaping / layout configuration for complex Unicode + startup font fallback

kitty shapes text with **HarfBuzz** and lays it out on a **fixed character-cell grid**. It has **no bidi reordering engine**. This section addresses each named mechanism — **ligatures**, **bidirectional (bidi) text**, **combining diacritics**, and **font fallback** — by name.

### 2.A.1 Ligatures — the HarfBuzz OpenType-feature model

**Mechanism (INFERRED, grounded).** kitty keeps a 3-entry feature table `static hb_feature_t hb_features[3] = {{0}};` [`kitty/fonts.c:42`] indexed by `typedef enum { LIGA_FEATURE, DLIG_FEATURE, CALT_FEATURE } HBFeature;` [`kitty/fonts.c:45`]. In `init_font` (defined at [`kitty/fonts.c:294`] — **note:** the AAP cited L295, which is the first *body* line; the function signature is at **L294**), the per-face feature set is assembled:

* **`CALT` (contextual alternates) is always enabled**, for every face: `memcpy(f->ffs_hb_features + f->num_ffs_hb_features++, &hb_features[CALT_FEATURE], sizeof(hb_feature_t));` [`kitty/fonts.c:325`].
* **`LIGA` + `DLIG` are enabled only for faces whose PostScript name begins `NimbusMonoPS-`**: `if (strstr(psname, "NimbusMonoPS-") == psname) { … LIGA_FEATURE … DLIG_FEATURE … }` [`kitty/fonts.c:321-323`].
* User `font_features` [`kitty/options/definition.py`] are merged per face (the branch that also `memcpy`s `CALT` at [`kitty/fonts.c:314`]).

Because `CALT` is on for **all** fonts, a ligature-capable programming font produces ligatures out of the box; `disable_ligatures` [`kitty/options/definition.py:115`] and `font_features` control/override this. *(External corroboration: the `kitty.conf` documentation states programming ligatures are implemented via the OpenType `calt` feature — consistent with the code above.)*

**How a ligature manifests in kitty (OBSERVED).** kitty's shaper groups the cells that form a ligature into **one shaped group spanning multiple cells** (`num_cells > 1`), rather than reducing the glyph count. The following exercises the **real** `shape_run` via `test_shape` [`kitty/fonts.c:1226`] and contrasts the default monospace font (CALT-only, no ligature lookups) with **Fira Code** (CALT ligatures present in the font). Stable across 2 runs.

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_liga3.py
=== DejaVu Sans Mono (default monospace, CALT-only): medium = DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0 ===
  '==': 2 group(s) -> no ligature (all single-cell groups)
      group: num_cells=1 num_glyphs=1 glyph_ids=(32,)
      group: num_cells=1 num_glyphs=1 glyph_ids=(32,)
  '->': 2 group(s) -> no ligature (all single-cell groups)
      group: num_cells=1 num_glyphs=1 glyph_ids=(16,)
      group: num_cells=1 num_glyphs=1 glyph_ids=(33,)
  '!=': 2 group(s) -> no ligature (all single-cell groups)
      group: num_cells=1 num_glyphs=1 glyph_ids=(4,)
      group: num_cells=1 num_glyphs=1 glyph_ids=(32,)

=== Fira Code (CALT ligatures): medium = FiraCode-Regular: /usr/share/fonts/truetype/firacode/FiraCode-Regular.ttf:0 ===
  '==': 1 group(s) -> LIGATURE: 1 multi-cell group(s)
      group: num_cells=2 num_glyphs=2 glyph_ids=(1649, 1387)
  '->': 1 group(s) -> LIGATURE: 1 multi-cell group(s)
      group: num_cells=2 num_glyphs=2 glyph_ids=(1186, 1458)
  '!=': 1 group(s) -> LIGATURE: 1 multi-cell group(s)
      group: num_cells=2 num_glyphs=2 glyph_ids=(1204, 1135)
```

**Reading:** with **DejaVu Sans Mono** (the default `monospace`), `==` shapes into **two independent single-cell groups** → no ligature. With **Fira Code**, `==`, `->`, `!=` each shape into **one group spanning 2 cells** — that grouping *is* the ligature, produced by the always-on `CALT` feature [`kitty/fonts.c:325`]. **OBSERVED.**

**The `NimbusMonoPS-` special case (OBSERVED, honest nuance).** kitty specifically enables `LIGA`+`DLIG` for Nimbus Mono PS [`kitty/fonts.c:321-323`]. The installed face's PostScript name is `NimbusMonoPS-Regular` — it matches the `strstr(psname, "NimbusMonoPS-") == psname` prefix test. However, setting it as `font_family` and shaping `ffi`/`==`/`->` produced **no** substitution:

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_liga.py
=== Nimbus Mono PS: medium = NimbusMonoPS-Regular: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1:0 ===
  'ffi': 3 chars -> 3 groups, 3 total glyphs; no ligature substitution
  '==':  2 chars -> 2 groups, 2 total glyphs; no ligature substitution
  '->':  2 chars -> 2 groups, 2 total glyphs; no ligature substitution
```

This is expected and honest: kitty **enables the features**, but the font must actually contain the `liga`/`dlig` GSUB lookups. Nimbus Mono PS (a Courier clone) has none, so nothing substitutes even with the features on. *(Also note `disable_ligatures` is applied in `render_line`'s `disable_ligature_strategy` post-processing, not inside `shape_run`; therefore `test_shape` output is unaffected by that option — an honest limitation of the shaping binding.)*

### 2.A.2 Bidirectional (bidi) text — **HONEST NEGATIVE: kitty has no bidi reordering engine**

**OBSERVED + grounded.** kitty explicitly does **not** implement BIDI. The `force_ltr` option's own documentation states it verbatim [`kitty/options/definition.py:64-82`]:

```python
opt('force_ltr', 'no',
    option_type='to_bool', ctype='bool',
    long_text='''
kitty does not support BIDI (bidirectional text), however, for RTL scripts,
words are automatically displayed in RTL. That is to say, in an RTL script, the
words "HELLO WORLD" display in kitty as "WORLD HELLO", … this option can be used
with the command line program :link:`GNU FriBidi …` to get BIDI support, because
it will force kitty to always treat the text as LTR, which FriBidi expects …
'''
    )
```

* **Resolved runtime value (OBSERVED):** `opts.force_ltr = False` (default `no` [`kitty/options/definition.py:64`]).
* **Arabic is still *shaped* by HarfBuzz, just not *reordered* by a bidi algorithm.** The RTL cluster handling is visible in the shaper as *decreasing* cluster numbers — the identical comment appears at **two** shaping sites: `// RTL languages like Arabic have decreasing cluster numbers` [`kitty/fonts.c:997`] and [`kitty/fonts.c:1079`]. **INFERRED** that cluster numbers decrease (the numbers are internal to `shape_run` and not exposed by `test_shape`); what *is* **OBSERVED** is that HarfBuzz shapes Arabic into contextual presentation-form glyphs (below).

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_shape.py     # REAL shape_run via test_shape, medium = DejaVu Sans Mono
[English]
  text='Hello': 5 groups
    cells=1 glyphs=1 first_glyph=43 ids=(43,)
    cells=1 glyphs=1 first_glyph=72 ids=(72,)
    cells=1 glyphs=1 first_glyph=79 ids=(79,)
    cells=1 glyphs=1 first_glyph=79 ids=(79,)
    cells=1 glyphs=1 first_glyph=82 ids=(82,)
[Arabic 'مرحبا' U+0645 U+0631 U+062D U+0628 U+0627]
  text='مرحبا': 5 groups
    cells=1 glyphs=1 first_glyph=3145 ids=(3145,)
    cells=1 glyphs=1 first_glyph=3149 ids=(3149,)
    cells=1 glyphs=1 first_glyph=3166 ids=(3166,)
    cells=1 glyphs=1 first_glyph=3177 ids=(3177,)
    cells=1 glyphs=1 first_glyph=3230 ids=(3230,)
```

The Arabic glyph IDs (3145, 3149, 3166, 3177, 3230) are **contextual presentation forms**, distinct from the raw codepoints — proof that HarfBuzz's Arabic shaper ran (in the main font). **OBSERVED.** RTL/Arabic rendering history is recorded in `docs/changelog.rst:3540-3541`.

### 2.A.3 Combining diacritics — grapheme clusters map to one cell

**Mechanism (grounded).** A base codepoint plus its combining marks are stored in a **single cell** (`ch` + `cc_idx[]`); coverage is tested by `has_cell_text` over the base and every mark [`kitty/fonts.c:435`], and each mark's codepoint is recovered by `codepoint_for_mark` (used in the fallback debug loop [`kitty/fonts.c:460,505-509`]).

**OBSERVED** (real `shape_run` via `test_shape`, stable across 2 runs):

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_combining.py
=== combining diacritics via REAL shape_run (medium=DejaVuSansMono: …/DejaVuSansMono.ttf:0) ===
  e + U+0301 (COMBINING ACUTE ACCENT): input 2 codepoints -> 1 shaped group(s)
      group: num_cells=1 num_glyphs=1 glyph_ids=(171,)
  a + U+0308 (COMBINING DIAERESIS): input 2 codepoints -> 1 shaped group(s)
      group: num_cells=1 num_glyphs=1 glyph_ids=(166,)
  n + U+0303 (COMBINING TILDE): input 2 codepoints -> 1 shaped group(s)
      group: num_cells=1 num_glyphs=1 glyph_ids=(179,)
  o + U+0301 + U+0323 (two marks): input 3 codepoints -> 1 shaped group(s)
      group: num_cells=1 num_glyphs=2 glyph_ids=(1548, 649)
```

**Reading:** every base+mark grapheme collapses to **one cell** (`num_cells=1`). Precomposable sequences shape to a single precomposed glyph (`e+́`→`é`=171, `a+̈`→`ä`=166, `n+̃`→`ñ`=179); a base with **two** marks (`o`+acute+dot-below) stays one cell but yields base + positioned-mark glyphs (`1548, 649`). This is kitty's grapheme-cluster handling: **one grapheme = one cell.** **OBSERVED.**

### 2.A.4 Font fallback during startup

**Algorithm (grounded).** `fallback_font` [`kitty/fonts.c:520`] first checks a per-cell hash cache; on a miss it calls `load_fallback_font` [`kitty/fonts.c:481`], which:
1. caps the chain at 100 entries — `if (fg->fallback_fonts_count > 100) { log_error("Too many fallback fonts"); … }` [`kitty/fonts.c:482`];
2. asks the platform (FontConfig on Linux) for a face;
3. under `--debug-font-fallback`, prints via `output_cell_fallback_data` [`kitty/fonts.c:457`] (gated at [`kitty/fonts.c:492`]);
4. verifies `has_cell_text` [`kitty/fonts.c:501`] and, if the OS-chosen font lacks the glyph, emits *"…is <face> but it does not actually contain glyphs for that text"* [`kitty/fonts.c:503-509`] and rejects it.

**Key discovery — the default font already covers basic Arabic (OBSERVED).** The default `monospace` resolves to **DejaVu Sans Mono**, whose charset includes the Arabic block:

```console
$ fc-query --format='%{charset}\n' /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf | tr ' ' '\n' | grep -E '^(621|628|62d|631|645)'
# ranges 621-63a and 640-655 cover U+0645, U+0631, U+062D, U+0628, U+0627
```

Consequently, for the **specific mixed Arabic + English line**, `font_for_cell` finds Arabic in the **main** font and **never calls `fallback_font`** — so **no** fallback line is emitted. This is correct behavior, demonstrated below through the **real** `render_line` path (`test_render_line` [`kitty/fonts.c:1591` → `1326`]):

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_render.py "Hello مرحبا World"
=== test_render_line REAL render_line->font_for_cell->fallback_font for: 'Hello مرحبا World' ===
=== render complete ===          # <-- no fallback line: main font covers Arabic
```

**The fallback chain — demonstrated with characters the main font lacks (OBSERVED).** Feeding a **CJK** line through the same real path triggers a genuine cross-font fallback and shows the chain being **constructed then reused**:

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_render.py "Hi 中文 World"
=== test_render_line REAL render_line->font_for_cell->fallback_font for: 'Hi 中文 World' ===
[0.050] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.050] U+6587 using previous fallback font at index: 0
=== render complete ===
```

* `U+4E2D` (中): a **new** fallback font is loaded — **Noto Sans CJK JP** — printed by `output_cell_fallback_data` (`debug("U+%x ")` [`kitty/fonts.c:458`] + `PyObject_Print(face, stderr, 0)` [`kitty/fonts.c:466`]).
* `U+6587` (文): the chain is **reused** — *"using previous fallback font at index: 0"* [`kitty/fonts.c:465`] (the cache/PyLong path).

Per-character fallback through the **production** `fallback_font` (via `get_fallback_font` [`kitty/fonts.c:1678` → `520`], one char per process), stable across 2 runs:

```console
$ for ch in 😀 中 가 م ; do KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_fb.py "$ch" | tail -1 ; done
[result] returned: DejaVuSans: /usr/share/fonts/truetype/dejavu/DejaVuSans.ttf:0                       # U+1F600 emoji -> DejaVu Sans
[result] returned: NotoSansCJKjp-Regular: /usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc:0     # U+4E2D 中 -> Noto Sans CJK JP
[result] returned: NotoSansCJKjp-Regular: /usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc:0     # U+AC00 가 -> Noto Sans CJK JP
[result] returned: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0               # U+0645 م -> stays in main font
```

**Reading:** fallback selects a genuinely different font for glyphs the main font lacks (emoji → **DejaVu Sans**; CJK/Hangul → **Noto Sans CJK JP**), and returns the **main** font for Arabic (which it already covers). The *"does not actually contain glyphs"* rejection branch [`kitty/fonts.c:503-509`] was **not** triggered in these runs (the OS returned covering fonts); its existence is **INFERRED** from the code path. **OBSERVED.**


---

## 3. Objective B — Verbose-logging startup diagnostics for mixed Arabic (RTL) + English (LTR)

With `--debug-font-fallback --debug-rendering` enabled, the startup diagnostics reveal (1) the exact **font families**, (2) the **fallback chains**, and (3) the runtime **configuration values** resolved **before any text rendering begins**.

### 3.B.1 Font families in the startup dump (OBSERVED)

`dump_font_debug()` is dispatched from startup only when the flag is set — `if args.debug_font_fallback: dump_font_debug()` [`kitty/main.py:228-229`] → [`kitty/fonts/render.py:161`]. Each per-face line is formatted by `identify_for_debug` as `<PostScript name>: <path>:<face index>` — the FreeType formatter returns `"%s: %V:%d"` [`kitty/freetype.c:738` (format string at `:742`)]. From the canonical launcher run (`run1.log`, verbatim, prefixes preserved):

```text
[0.163] Text fonts:
[0.163]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.163]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.163]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.163]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

* **Exact families / descriptors (OBSERVED):** Normal = `DejaVuSansMono`, Bold = `DejaVuSansMono-Bold`, Italic = `DejaVuSansMono-Oblique`, Bold-Italic = `DejaVuSansMono-BoldOblique` — each with its file path and face index `0`, in the `identify_for_debug` format.
* **No `Symbol map fonts:` block** appears because `symbol_map` is empty under `--config NONE`; `dump_font_debug` prints that block only when the symbol map is non-empty [`kitty/fonts/render.py:166-170`]. **OBSERVED** (honest: block absent by configuration, not by failure).

### 3.B.2 Fallback chains for mixed Arabic + English (OBSERVED)

As established in §2.A.4, with the **default** font the mixed Arabic+English line needs **no** fallback (DejaVu Sans Mono covers both scripts), so **no fallback chain** is constructed for it — the honest, observed result. A fallback **chain** is constructed for characters the main font lacks; the CJK example shows the chain built then reused:

```text
[0.050] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 …)
[0.050] U+6587 using previous fallback font at index: 0
```

Fallback face descriptors (via production `fallback_font`): emoji `U+1F600` → **DejaVu Sans** (`DejaVuSans.ttf:0`); `U+4E2D`/`U+AC00` → **Noto Sans CJK JP** (`NotoSansCJK-Regular.ttc:0`). These descriptor strings use the same `identify_for_debug` format as the `Text fonts:` block — cross-file consistency verified (§5).

### 3.B.3 Configuration values resolved BEFORE rendering (OBSERVED)

Startup ordering: `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` [`kitty/main.py:249`] → `set_font_family(opts)` [`kitty/main.py:251` → `kitty/fonts/render.py:173`] → (teardown later) `free_font_data()` [`kitty/main.py:255`]. The resolved values, captured from the running process **before** any glyph is rendered:

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe.py   # set_options(opts,False,True,True) then set_font_family(opts)
=== [observe] kitty real-code-path observation ===
[observe] opts.font_family = FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
[observe] opts.font_size   = 11.0
[observe] opts.bold_font   = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
[observe] opts.italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
[observe] opts.bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
[observe] opts.force_ltr   = False
[observe] opts.disable_ligatures = 0
[observe] set_options(opts, is_wayland=False, debug_rendering=True, debug_font_fallback=True)
[observe] set_font_family(opts) done (REAL, == main.py:251)
[observe] current_fonts() resolved faces (identify_for_debug):
[observe]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[observe]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[observe]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[observe]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[observe] symbol map fonts count: 0
```

Mapping each named configuration value to its resolved runtime value and its default:

| Config key | Default (source) | Resolved value (OBSERVED) |
|---|---|---|
| `font_family` | `monospace` [`kitty/options/definition.py:35`] | `FontSpec(system='monospace')` → **DejaVu Sans Mono** |
| `bold_font` | `auto` | `FontSpec(system='auto')` → `DejaVuSansMono-Bold` |
| `italic_font` | `auto` | `FontSpec(system='auto')` → `DejaVuSansMono-Oblique` |
| `bold_italic_font` | `auto` | `FontSpec(system='auto')` → `DejaVuSansMono-BoldOblique` |
| `font_size` | `11.0` [`kitty/options/definition.py:59`] | `11.0` |
| `force_ltr` | `no` [`kitty/options/definition.py:64`] | `False` |
| `disable_ligatures` | `never (0)` [`kitty/options/definition.py:115`] | `0` |
| `symbol_map` | *(empty)* | 0 symbol-map fonts |

All values are resolved by `set_font_family` [`kitty/main.py:251`] **before** the render path runs — confirmed by the `Text fonts:` block appearing in the launcher log immediately after `Child launched` and before any glyph work. **OBSERVED.**


---

## 4. Objective C — Screen-grid / shaping-subsystem init: cell metrics, baseline, decoration alignment

As the screen grid and shaping subsystems initialize, kitty computes the **cell metrics**, **baseline**, and **decoration** alignment for the font. The request names **overline AND underline** (plus strikethrough) — each is addressed explicitly below.

### 4.C.1 Cell metrics + baseline + decoration values (OBSERVED)

These are computed by `cell_metrics` [`kitty/freetype.c:387-406`] (Linux/FreeType), assembled by `calc_cell_metrics` [`kitty/fonts.c:373`] on the medium font's face (declaration `void cell_metrics(PyObject*, …7 out-params…)` [`kitty/fonts.h:27`]; macOS analog `kitty/core_text.m:523`, out of canonical scope). They flow to the GPU prerender step as the arguments to `prerender_function` in `send_prerendered_sprites` — `PyObject_CallFunction(prerender_function, "IIIIIIIffdd", fg->cell_width, fg->cell_height, fg->baseline, fg->underline_position, fg->underline_thickness, fg->strikethrough_position, fg->strikethrough_thickness, …)` [`kitty/fonts.c:1458`]. Captured by wrapping the **real** `prerender_function` [`kitty/fonts/render.py:364`] (not `setup_for_testing`), stable across 2 runs:

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_metrics.py
=== cell_metrics (REAL prerender_function via send_prerendered_sprites fonts.c:1458) ===
create_test_font_group returned cell_width=9 cell_height=18
  cell_width = 9
  cell_height = 18
  baseline = 14
  underline_position = 15
  underline_thickness = 1
  strikethrough_position = 10
  strikethrough_thickness = 1
  cursor_beam_thickness = 1.5
  cursor_underline_thickness = 2.0
  dpi_x = 96.0
  dpi_y = 96.0
```

| Metric | Value (OBSERVED) | Derivation [`kitty/freetype.c`] (INFERRED-from-code, confirmed) |
|---|---|---|
| `cell_width` | **9** | `calc_cell_width(self)` [`:390`] |
| `cell_height` | **18** | `calc_cell_height(self, true)` [`:391`] |
| `baseline` | **14** | `font_units_to_pixels_y(ascender)` [`:392`] |
| `underline_position` | **15** | `MIN(cell_height-1=17, px(MAX(0, ascender - underline_position)))` [`:393`] |
| `underline_thickness` | **1** | `MAX(1, px(underline_thickness))` [`:394`] |
| `strikethrough_position` | **10** | font-provided branch `MIN(17, px(ascender - strikethrough_position))` [`:397`]; the fallback `floor(baseline*0.65)`=`floor(9.1)`=**9** [`:399`] would apply only if the font omitted it — observed **10 ≠ 9** confirms DejaVu supplies its own value |
| `strikethrough_thickness` | **1** | `MAX(1, px(strikethrough_thickness))` [`:402`], else `= underline_thickness` [`:404`] |
| `dpi_x` / `dpi_y` | **96.0 / 96.0** | logical DPI passed to `create_test_font_group` |

The `create_test_font_group` return `(9, 18)` matches `cell_width`/`cell_height`, and these same dimensions feed the atlas capacity computation (§5 cross-consistency). **OBSERVED.**

### 4.C.2 Underline — real, measured (OBSERVED)

Underline is a first-class decoration. It has measured alignment (`underline_position = 15`, `underline_thickness = 1`, above), 5 underline **styles** are pre-rendered into the atlas (`NUM_UNDERLINE_STYLES = 5` [`kitty/data-types.h:213`]; range `1..NUM_UNDERLINE_STYLES+1` [`kitty/fonts/render.py:391`]), and the term appears **158×** across kitty/ text sources.

### 4.C.3 Strikethrough — real, measured (OBSERVED)

Strikethrough is likewise real: measured alignment (`strikethrough_position = 10`, `strikethrough_thickness = 1`), one strikethrough sprite is pre-rendered (`STRIKE_SPRITE_INDEX = NUM_UNDERLINE_STYLES + 1` [`kitty/shaders.py:162`]), and the term appears **63×** across kitty/ text sources.

### 4.C.4 Overline — **HONEST NEGATIVE: no overline decoration exists in the `kitty/` tree**

The request names **overline**; the honest, evidence-backed answer is that kitty implements **no** overline decoration. A recursive, case-insensitive, text-only search returns nothing:

```console
$ grep -rInI "overline" kitty/ ; echo "exit=$?"
exit=1                       # 1 = NO matches in any text source under kitty/

$ grep -rInI "overline" kitty/*.c kitty/*.h kitty/*.glsl kitty/*.py kitty/fonts/*.py ; echo "exit=$?"
exit=1                       # none in C core, GLSL, or font Python

$ echo "underline in kitty/: $(grep -rInI underline kitty/ | wc -l);  strikethrough: $(grep -rInI strikethrough kitty/ | wc -l)"
underline in kitty/: 158;  strikethrough: 63
```

Binary matches that `grep` reports without `-I` (`kitty/launcher/kitten`, `*.pyc`, `*.so`) are **not** kitty decoration code — they are vendored syntax-highlighter grammars inside the Go `kitten` (the CSS text-decoration value `overline` and a COBOL keyword `OVERLINE`), unrelated to terminal rendering. The **only** meaningful match anywhere in the repository is an unrelated **XKB keysym**:

```console
$ grep -rn "overline" glfw/
glfw/xkb-compat-shim.h:135:    { 0x047e, 0x203e }, /*                    overline ‾ OVERLINE */
```

**Conclusion (OBSERVED):** underline and strikethrough are real, measured decorations; **overline is a definitive negative** — it does not exist as a text decoration anywhere in the `kitty/` tree; the sole repo match is the XKB keysym at `glfw/xkb-compat-shim.h:135`.


---

## 5. Objective D — GPU texture-atlas initialization at launch

kitty caches each rendered glyph's alpha mask in a GPU **texture atlas** (a `GL_TEXTURE_2D_ARRAY`). This section documents the initial **page layout**, **sizing**, **capacity**, and the **readiness** signals — with before/after state.

### 5.D.1 Readiness signal #1 — GL version banner (OBSERVED, STDOUT)

Emitted by `printf("[%.3f] GL version string: %s\n", …)` gated by `--debug-rendering` [`kitty/gl.c:72`] (string built by `gl_version_string` [`kitty/gl.c:42`]). From the canonical launcher run (`run1.log`; note this line is on **STDOUT**, hence the both-stream capture):

```text
[0.126] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

`gl_init` treats a missing required capability (e.g. `texture_storage`) as **fatal** [`kitty/gl.c`]; the banner appearing with **no** fatal means the GL context is ready for atlas allocation. **OBSERVED.**

### 5.D.2 Initial page layout & structures (OBSERVED / grounded)

The GPU-side texture wrapper starts at **one cell / one layer**:

```c
// kitty/shaders.c:24-31
typedef struct {
    unsigned int cell_width, cell_height;
    int xnum, ynum, x, y, z, last_num_of_layers, last_ynum;
    GLuint texture_id;
    GLint max_texture_size, max_array_texture_layers;
} SpriteMap;
static const SpriteMap NEW_SPRITE_MAP = { .xnum = 1, .ynum = 1, .last_num_of_layers = 1, .last_ynum = -1 };
```

The CPU-side allocator is `GPUSpriteTracker { size_t max_y; unsigned int x, y, z, xnum, ynum; }` [`kitty/fonts.c:16-19`].

### 5.D.3 GL limits + capacity math (OBSERVED inputs, INFERRED formula)

`alloc_sprite_map` queries the hardware limits once — `glGetIntegerv(GL_MAX_TEXTURE_SIZE, …)` and `glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, …)` [`kitty/shaders.c:51-53`] — then `sprite_tracker_set_limits(…)`. On the canonical llvmpipe context these are **OBSERVED** as `GL_MAX_TEXTURE_SIZE = 16384`, `GL_MAX_ARRAY_TEXTURE_LAYERS = 2048`. The per-layer capacity is set by `sprite_tracker_set_layout` [`kitty/fonts.c:276-279`]:

```c
sprite_tracker->xnum  = MIN(MAX(1u, max_texture_size / cell_width),  (size_t)UINT16_MAX);
sprite_tracker->max_y = MIN(MAX(1u, max_texture_size / cell_height), (size_t)UINT16_MAX);
sprite_tracker->ynum  = 1;
sprite_tracker->x = 0; sprite_tracker->y = 0; sprite_tracker->z = 0;
```

Computed from the **observed** limits + observed cell `9×18`:

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_atlas.py
...
=== capacity (sprite_tracker_set_layout fonts.c:276; GL limits 16384/2048 observed, cell 9x18) ===
  xnum=1820 cols/layer, max_y=910 rows/layer, ynum0=1, z0=0; per-layer=1656200; layers<= 2048
  initial glTexStorage3D: width=xnum*cw=16380, height=ynum*ch=18, znum=z+1=1, GL_SRGB8_ALPHA8 (shaders.c:123)
```

* **`xnum` = `16384 / 9` = 1820** columns/layer; **`max_y` = `16384 / 18` = 910** rows/layer; **`ynum`** starts at **1**, **`z`** at **0**.
* **Per-layer capacity = 1820 × 910 = 1,656,200 cells**; up to **2048** layers.
* Cells advance via `do_increment` [`kitty/fonts.c:243`]: `x++` until `x >= xnum` → `x=0, y++` (`ynum` grows toward `max_y`); `y >= max_y` → `y=0, z++` (new layer); `z >= max_array_len` → error.

### 5.D.4 Immutable texture allocation (grounded)

`realloc_sprite_texture` [`kitty/shaders.c:108-123`] allocates the array texture **immutably**: `GL_TEXTURE_2D_ARRAY`, `GL_NEAREST` filtering, `GL_CLAMP_TO_EDGE` wrap, then `glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum)` [`kitty/shaders.c:123`], where `width = xnum*cell_width`, `height = ynum*cell_height`, `znum = z+1`. Initial: **16380 × 18 × 1 layer**, `GL_SRGB8_ALPHA8`. Per-glyph upload is `send_sprite_to_gpu` [`kitty/shaders.c:147-155`]; growth copies the existing image into a larger allocation.

### 5.D.5 Readiness signal #2 — prerendered sprites (OBSERVED, before/after)

Before any **shaped text** glyph is added, `send_prerendered_sprites` [`kitty/fonts.c:1450`] uploads the blank/underline/strikethrough/missing/cursor sprites. Counted through the **real** `set_send_sprite_to_gpu` callback (signature `(x, y, z, buf)`), stable across 2 runs:

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_atlas.py
=== ATLAS before/after (prerendered sprites uploaded at font-group creation) ===
NUM_UNDERLINE_STYLES = 5
[BEFORE] sprites uploaded = 0 (atlas empty)
[AFTER]  sprites uploaded = 11 prerendered sprites (cell 9x18)
         breakdown = 1 blank + 5 underline + 1 strikethrough + 1 missing + 3 cursor = 11
         (x,y,z) positions = [(0, 0, 0), (1, 0, 0), (2, 0, 0), (3, 0, 0), (4, 0, 0), (5, 0, 0), (6, 0, 0), (7, 0, 0), (8, 0, 0), (9, 0, 0), (10, 0, 0)]
         all on layer z=0, row y=0: True
```

**Before/after (OBSERVED):** atlas **empty (0 sprites)** → **11 prerendered sprites** — `1 blank + 5 underline (NUM_UNDERLINE_STYLES=5) + 1 strikethrough + 1 missing-glyph + 3 cursor` — occupying columns `x = 0..10` on layer `z=0`, row `y=0` (sequential via `do_increment`). This happens at window/font-group creation, **before** any shaped text glyph is uploaded. `prerender_function` builds these [`kitty/fonts/render.py:391-393`]; `MISSING_GLYPH = NUM_UNDERLINE_STYLES + 2` [`kitty/fonts.c:15`].

### 5.D.6 "Atlas ready" — INFERRED (no single log line)

**INFERRED:** there is **no** single "atlas ready" log line. Readiness is *manifested* by (1) the GL version banner [`kitty/gl.c:72`] followed by (2) the successful upload of the 11 prerendered sprites with no fatal — after which the atlas is ready to receive shaped glyph data via `send_sprite_to_gpu` [`kitty/shaders.c:147`].

### 5.D.7 Edge/error path (grounded)

If the prerendered sprites overflow a single atlas row (i.e. `y` advances past 0 during prerender), startup **aborts**: `if (y > 0) { fatal("Too many pre-rendered sprites for your GPU or the font size is too large"); }` [`kitty/fonts.c:1463`]. *(Note: the AAP cited L1462; the actual line is **L1463**.)* At cell `9×18` all 11 sprites fit within row 0 (`x = 0..10 < xnum = 1820`), so the abort is not triggered.

---

## 6. Stability, cross-file consistency, and the three required negatives

### 6.1 Stability (OBSERVED)

Every observation was run **≥2×** and confirmed stable (values identical; only the `[<seconds>]` prefix varies):

* Canonical launcher `run1/run2/run3` — **3 runs**, byte-identical modulo timestamps (`diff` of timestamp-stripped logs = identical): families + GL banner stable.
* Cell metrics — 2 runs identical (`9×18`, baseline 14, underline 15/1, strike 10/1, dpi 96).
* Atlas — 2 runs identical (`0 → 11` sprites; `xnum 1820`, `max_y 910`).
* Fallback (CJK render) — 3 runs identical; Arabic no-fallback — 2 runs; per-char fallback — 2 runs.
* Shaping, ligatures, combining — 2 runs each, identical.

No run-to-run inconsistency was observed; the values are deterministic for the fixed font environment.

### 6.2 Cross-file evidence consistency (VERIFIED)

* The family/descriptor strings from `dump_font_debug` [`kitty/fonts/render.py:161`] match the `identify_for_debug` format `<PS name>: <path>:<index>` [`kitty/freetype.c:738`]. ✔
* The `cell_width`/`cell_height` (`9`/`18`) in the metric output [`kitty/freetype.c:387`] match the values used to compute atlas capacity in `sprite_tracker_set_layout` [`kitty/fonts.c:276`]. ✔
* All debug-line prefixes match `timed_debug_print`'s `"[%.3f] "` [`kitty/monotonic.h:99`]. ✔

### 6.3 The three required honest negatives (OBSERVED)

1. **No BIDI engine.** kitty does not implement BIDI; only `force_ltr` (default `no` → `False`) + HarfBuzz shaping (RTL clusters decrease). [`kitty/options/definition.py:64-82`; `kitty/fonts.c:997,1079`] — §2.A.2.
2. **No overline decoration** anywhere in `kitty/` (`grep -rInI overline kitty/` → exit 1); only the XKB keysym `glfw/xkb-compat-shim.h:135`. — §4.C.4.
3. **`setup_for_testing` is non-canonical** [`kitty/fonts/render.py:408`] — it replaces the GPU upload with a Python callback and sets `sprite_map_set_limits(100000, 100)` [`kitty/fonts/render.py:421`]. It was **never** used as primary evidence; all runtime detail was captured through production-function bindings (`get_fallback_font`→`fallback_font`, `test_render_line`→`render_line`, `test_shape`→`shape_run`, `create_test_font_group`→`send_prerendered_sprites`, wrapped `prerender_function`).


---

## 7. Appendix A — exact commands and complete, unedited output

### A.1 Build

```console
$ python3 setup.py clean
$ CI=true python3 setup.py
# … "Disabling building of wayland backend" (expected, X11 backend used) …
# [4/4] Linking launcher ...
# real 1m6.232s
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 40384 kitty/launcher/kitty
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### A.2 Canonical launcher run — complete `run1.log` (STDOUT+STDERR, prefixes preserved)

```console
$ DISPLAY=:99 ./kitty/launcher/kitty --config NONE --debug-font-fallback --debug-rendering \
      -o confirm_os_window_close=0 /tmp/kitty_obs/feed.sh > /tmp/kitty_obs/run1.log 2>&1
```
```text
[0.150] OS Window created
[0.159] Failed to open systemd user bus with error: Connection refused
[0.162] Child launched
[0.163] Text fonts:
[0.163]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.163]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.163]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.163]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.126] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```
*(The `[0.126]` GL banner precedes the `[0.163]` font block in wall-clock order because the two streams — STDOUT vs STDERR — are interleaved on flush; both were captured via `2>&1`. `run2.log`/`run3.log` are identical modulo the `[<seconds>]` prefix.)*

### A.3 Fallback through the real render path

```console
$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_render.py "Hi 中文 World"
=== test_render_line REAL render_line->font_for_cell->fallback_font for: 'Hi 中文 World' ===
[0.050] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.050] U+6587 using previous fallback font at index: 0
=== render complete ===

$ KITTY_REPO=$PWD python3 /tmp/kitty_obs/observe_render.py "Hello مرحبا World"
=== test_render_line REAL render_line->font_for_cell->fallback_font for: 'Hello مرحبا World' ===
=== render complete ===
```

### A.4 Metrics + atlas (see §4, §5 for the full blocks)

Metrics `9×18 / baseline 14 / underline 15,1 / strikethrough 10,1 / dpi 96×96` (`observe_metrics.py`); atlas `0 → 11 prerendered sprites`, `xnum=1820`, `max_y=910`, initial `glTexStorage3D 16380×18×1 GL_SRGB8_ALPHA8` (`observe_atlas.py`).

---

## 8. Coverage pass — every named item addressed

**Objectives**

* **A — shaping/layout + startup fallback:** ligatures §2.A.1 (CALT always-on [`fonts.c:325`], LIGA/DLIG for `NimbusMonoPS-` [`fonts.c:321-323`], Fira Code multi-cell ligature OBSERVED); bidi §2.A.2 (**negative**); combining diacritics §2.A.3 (`has_cell_text` [`fonts.c:435`], one grapheme = one cell OBSERVED); font fallback §2.A.4 (`fallback_font` [`fonts.c:520`] → `load_fallback_font` [`fonts.c:481`], chain built + reused OBSERVED). ✔
* **B — startup diagnostics:** families §3.B.1; fallback chains §3.B.2; config values before rendering §3.B.3. ✔
* **C — cell metrics/baseline/decorations:** metrics/baseline §4.C.1; underline §4.C.2; strikethrough §4.C.3; **overline negative** §4.C.4. ✔
* **D — GPU atlas:** GL banner §5.D.1; layout §5.D.2; capacity §5.D.3; immutable allocation §5.D.4; prerendered before/after §5.D.5; readiness INFERRED §5.D.6; edge abort §5.D.7. ✔

**Flags:** `--debug-font-fallback` [`kitty/cli.py:1002`] ✔ · `--debug-rendering` / `--debug-gl` [`kitty/cli.py:989`] ✔ · `--debug-config` (not a launcher flag in 0.35.2 — see §1.4) ✔ · `--debug-input` / `--debug-keyboard` [`kitty/cli.py:996`] (named; out of scope) ✔

**Config keys:** `font_family` ✔ · `bold_font` ✔ · `italic_font` ✔ · `bold_italic_font` ✔ · `font_size` ✔ · `symbol_map` (empty, no block) ✔ · `disable_ligatures` ✔ · `font_features` ✔ · `force_ltr` ✔ (all in §3.B.3 / §2.A).

**Named functions/structs:** `dump_font_debug` [`fonts/render.py:161`], `set_font_family` [`fonts/render.py:173`], `set_options` [`main.py:249`], `identify_for_debug` [`freetype.c:738`], `init_font` [`fonts.c:294`], `hb_features`/`HBFeature` [`fonts.c:42,45`], `has_cell_text` [`fonts.c:435`], `output_cell_fallback_data` [`fonts.c:457`], `load_fallback_font` [`fonts.c:481`], `fallback_font` [`fonts.c:520`], `calc_cell_metrics` [`fonts.c:373`], `cell_metrics` [`freetype.c:387`, `fonts.h:27`], `SpriteMap`/`NEW_SPRITE_MAP` [`shaders.c:24,31`], `alloc_sprite_map` [`shaders.c:51`], `sprite_tracker_set_layout` [`fonts.c:276`], `realloc_sprite_texture` [`shaders.c:108`], `send_sprite_to_gpu` [`shaders.c:147`], `send_prerendered_sprites` [`fonts.c:1450`], `gl_version_string` [`gl.c:42`], `timed_debug_print` [`monotonic.h:99`], `do_increment` [`fonts.c:243`]. ✔

**Three required negatives:** no BIDI engine §6.3/§2.A.2 ✔ · no overline §6.3/§4.C.4 ✔ · `setup_for_testing` non-canonical §6.3 ✔

**AAP line-number deviations noted:** `init_font` defined at `fonts.c:294` (AAP L295) · "Too many pre-rendered sprites" fatal at `fonts.c:1463` (AAP L1462) · `setup_for_testing` at `fonts/render.py:408` (AAP L410).

**macOS note:** CoreText paths (`kitty/core_text.m:523,966`, `kitty/fonts/core_text.py`) documented for completeness only; the canonical run is the Linux/FontConfig/FreeType Docker container.

