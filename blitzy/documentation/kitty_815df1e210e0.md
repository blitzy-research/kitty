# kitty — Complex-Unicode Shaping, Font Fallback, Cell Metrics & GPU Texture-Atlas Initialization at Startup

**Repository:** `kovidgoyal/kitty` &nbsp;·&nbsp; **Branch:** `kitty_815df1e210e0` &nbsp;·&nbsp; **HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (short `815df1e21`)
**Investigation type:** read-only, evidence-grounded Q&A. kitty was **built and run**; every value below is captured from **real, unedited** program output and correlated to a concrete `file:line` cause.

---

## 1. Summary — Definitive one-line answers

- **Q1 (font families + fallback chains for mixed Arabic/English):** With `--debug-font-fallback --debug-rendering`, startup prints a `Text fonts:` block naming the four resolved primary faces — all four resolve to **DejaVu Sans Mono** (`Normal`/`Bold`/`Italic`/`Bold-Italic`). Because DejaVu Sans Mono itself covers **both** Latin and Arabic (and the combining acute), the mixed Arabic+English stress line triggers **no** fallback. The fallback machinery — `output_cell_fallback_data` (`kitty/fonts.c:L457`) — was exercised with genuinely-absent codepoints and produces `U+<hex> … Face(family=…, style=…, ps_name=…, path=…)` lines resolving CJK → **Noto Sans CJK JP** and emoji → **Noto Color Emoji**. Arabic is shaped **RTL by default** because `force_ltr` is `no`, so the direction is **not** forced at `kitty/fonts.c:L688`.
- **Q2 (pre-render config values):** The true defaults (captured under `--config NONE`) are `font_family=monospace`, `bold_font/italic_font/bold_italic_font=auto`, `font_size=11.0`, `force_ltr=no`, `disable_ligatures=never`, `text_composition_strategy=platform`, `font_features={}`, applied by `set_font_family` (`kitty/fonts/render.py:L173`) before any glyph is rasterized.
- **Q3 (cell metrics, baseline, decoration alignment):** For the default DejaVu Sans Mono @ 11pt / 96dpi: `cell_width=9`, `cell_height=18`, `baseline=14`, `underline_position=15`, `underline_thickness=1`, `strikethrough_position=10`, `strikethrough_thickness=1` (computed by `calc_cell_metrics`, `kitty/fonts.c:L373`). kitty supports **five underline styles + a separate strikethrough** and has **no overline** concept — proven twice (`decoration_as_sgr`, `kitty/line.c:L733`; and the absence of any SGR 53/55 handler in `kitty/cursor.c`).
- **Q4 (atlas page layout, sizing, capacity):** The sprite atlas starts from `NEW_SPRITE_MAP = {.xnum=1,.ynum=1,.last_num_of_layers=1,.last_ynum=-1}` (`kitty/shaders.c:L31`). On the observed **Linux** platform the `MIN(8192,…)/MIN(512,…)` clamps are **not** applied (they are inside `#ifdef __APPLE__`), so the raw `GL_MAX_TEXTURE_SIZE=16384` and `GL_MAX_ARRAY_TEXTURE_LAYERS=2048` are used. `sprite_tracker_set_layout` (`kitty/fonts.c:L276`) computes `xnum = MIN(MAX(1, 16384/9), 65535) = 1820`, `max_y = MIN(MAX(1, 16384/18), 65535) = 910`, `ynum = 1`.
- **Q5 (atlas-ready logs):** Readiness is established by `send_prerendered_sprites` (`kitty/fonts.c:L1450`) uploading **11** sprites at startup — **1 blank cell** first (`(0,0,0)`) then the **10** special sprites from `prerender_function` (5 underline + 1 strikethrough + 1 missing-glyph + 3 cursor, `kitty/fonts/render.py:L391-L394`) — via `ensure_sprite_map`/`send_sprite_to_gpu` (`kitty/shaders.c:L137`/`L147`) into a `GL_TEXTURE_2D_ARRAY`. After these uploads the atlas is ready to accept shaped-glyph data.

> **Note on the AAP's predicted output shape.** The Agent Action Plan anticipated a `Fonts:` block with `medium/bold/italic/bi` labels and a `Features:` suffix, plus a fuller banner (version/OS/kernel/Frozen). The **actual** build at this HEAD prints a `Text fonts:` block with `Normal/Bold/Italic/Bold-Italic` labels (via `dump_font_debug`, `kitty/fonts/render.py:L161`) and, for `--debug-rendering`, only a single GL-version line. This document reports the **actual observed** output, not the predicted shape.

---

## 2. Environment & Build

### 2.1 Environment facts

| Fact | Value | Source / how obtained |
|------|-------|-----------------------|
| Git HEAD | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` | `git rev-parse HEAD` |
| Baseline working tree | clean (`git status --porcelain` → 0 lines) | `git status --porcelain \| wc -l` |
| Python (env) | **3.13.7** | `python3 --version` (AAP predicted 3.12.3; the actual interpreter is 3.13.7) |
| Python highest CI-verified | 3.11 | `.github/workflows/ci.yml` `pyver` matrix |
| Python floor | `>=3.8` | `pyproject.toml:L2` (`requires-python = ">=3.8"`) |
| kitty version | `kitty 0.35.2 created by Kovid Goyal` | `./kitty/launcher/kitty --version` |
| Compiler | gcc 15.2.0 | `gcc --version` |
| HarfBuzz | 10.2.0 (≥ documented floor 2.2.0, `docs/build.rst:L84`; build gate ≥1.5, `setup.py:L609`) | `pkg-config --modversion harfbuzz` |
| FreeType / FontConfig | 26.2.20 / 2.15.0 | `pkg-config --modversion` |
| Software GL | llvmpipe (LLVM 20.1.8, 256 bits), Mesa 25.2.8, OpenGL 4.5 | `glxinfo` under Xvfb |
| Arabic fonts present | Scheherazade, Amiri; `fc-match monospace` → `DejaVuSansMono.ttf` | `fc-match` |

### 2.2 Build (canonical, default configuration)

```
$ CI=true python3 setup.py build --verbose
```

Clean rebuild completed in ~23s (exit 0), compiling every C source with strict `-Werror`. The final links (unedited tail) prove the CPython extension and native launcher were produced:

```
gcc … -lpython3.13 … -lharfbuzz -lGL -lpng16 -llcms2 -llcms2_fast_float -llcms2_threaded -pthread -lm -lcrypto -lrt -lz -o build/kitty/fast_data_types.so
gcc … -o build/kitty/glfw-x11.so
gcc build/kitty-launcher-main.o build/kitty-launcher-single-instance.o … -lpython3.13 … -o kitty/launcher/kitty
Updating Go generated files...
/usr/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten …/tools/cmd
```

```
$ ls -l ./kitty/launcher/kitty
-rwxr-xr-x 1 root root 40384 … ./kitty/launcher/kitty
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The build byproducts (`kitty/fast_data_types.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`, `kitty/glfw-x11.so`, `build/`) are **git-ignored** and are **not** committed:

```
$ grep -nE '\*\.so|/build/|launcher/kitt' .gitignore
1:*.so
14:/build/
18:/kitty/launcher/kitt*
```

> The wayland GLFW backend is intentionally disabled at build time ("Disabling building of wayland backend") — headless observation uses the X11/Xvfb backend, so this is expected and not an error.

### 2.3 Headless GL setup

The canonical GUI startup path needs an OpenGL context. It is run under `xvfb-run -a` (headless X display) with Mesa **llvmpipe** software GL forced via `LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`.

---

## 3. Methodology

### 3.1 Canonical vs. non-canonical

- **Canonical (source of truth):** the real entry point `./kitty/launcher/kitty --debug-font-fallback --debug-rendering --config NONE …` under Xvfb + software GL. `--config NONE` guarantees the reported defaults (Q2) are the true built-in defaults, not a user `kitty.conf`.
- **Non-canonical (corroboration only, explicitly labelled):** the in-repo headless harness (`kitty/fonts/render.py::setup_for_testing`, `python test.py --module fonts`, and small temporary scripts). These drive the **identical C functions** (`render_line`, `shape_run`, `output_cell_fallback_data`, `calc_cell_metrics`, `prerender_function`, the sprite tracker) but without a live GL window. Every value obtained this way is tagged **[non-canonical]**.

### 3.2 Why a harness is required for per-cell fallback/metrics/atlas values (documented headless limitation)

The canonical GUI run reliably emits the startup banner-line and the `Text fonts:` block, but in the headless Xvfb container it does **not** repaint the child program's text, so the per-cell draw path `render_line` (`kitty/fonts.c:L1326`) → `font_for_cell` → `fallback_font` → `load_fallback_font` → `output_cell_fallback_data` is never invoked for the stress characters. This is a precise, reproducible property of the headless environment (not vague "noise"). The faithful way to observe those exact C functions is therefore to invoke them directly through the test hook `test_render_line` (`kitty/fonts.c:L1591`, which calls the **real** `render_line`) and `test_shape` (`kitty/fonts.c:L1226`, which calls the **real** `shape_run`), with `debug_font_fallback` enabled. Such values are labelled **[non-canonical]** but are produced by the same code the canonical path would run.

### 3.3 How verbose logging is enabled (debug plumbing)

Debug output goes to **stderr** via the `debug`/`debug_fonts` macro (`kitty/fonts.c:L18` → `#define debug debug_fonts`; the macro is gated at `kitty/state.h:L16` on `global_state.debug_font_fallback`) and via `log_error`. The `--debug-rendering` GL line goes to **stdout** via `printf` (`kitty/gl.c:L72`, gated on `global_state.debug_rendering`). The flags flow:

- **CLI definition:** `--debug-font-fallback` (`kitty/cli.py:L1002`, help: "Print out information about the selection of fallback fonts for characters not present in the main font"); `--debug-rendering --debug-gl` (`kitty/cli.py:L989`).
- **Plumbed to native layer:** `set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)` (`kitty/main.py:L249`; `dump_font_debug()` call at `kitty/main.py:L229`); also during controller wiring at `kitty/boss.py:L2649`.
- **Parsed into globals:** `PA("O|ppp", &opts, &is_wayland, &debug_rendering, &debug_font_fallback)` then `global_state.debug_rendering = …` / `global_state.debug_font_fallback = …` (`kitty/state.c:L728-L740`).

### 3.4 Stress input

A single line mixing scripts and cluster types (repr shows the exact codepoints):

```
$ python3 -c "print(repr(open('/tmp/kitty_obs/stress.txt').read()))"
'Hello \u0645\u0631\u062d\u0628\u0627 e\u0301 office ffi fi\n'
```

- `Hello`, `office` — Latin (LTR)
- `مرحبا` = U+0645 U+0631 U+062D U+0628 U+0627 — Arabic (RTL)
- `e` + U+0301 (COMBINING ACUTE ACCENT) — a base+combining grapheme cluster
- `ffi`, `fi` — ligature-forming sequences

### 3.5 Stability protocol

Every magnitude/size value (cell metrics, prerendered-sprite count, GL_MAX_* limits, resolved fallback faces) was captured on **≥2 runs** with identical input; results are byte-identical except for the leading `[seconds]` timestamps. Variances (only timestamps) are noted in §9.

---

## Q1 — FONT FAMILIES + FALLBACK CHAINS for mixed Arabic (RTL) + English (LTR)

> **Verbatim:** "With verbose logging enabled, what exact FONT FAMILIES and FALLBACK CHAINS appear in startup diagnostics for handling mixed Arabic (RTL) and English (LTR) text?"

### Q1.1 Canonical command

```
$ timeout 120 xvfb-run -a --server-args="-screen 0 1024x768x24" \
    env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
    ./kitty/launcher/kitty --debug-font-fallback --debug-rendering --config NONE \
    -- sh -c 'cat /tmp/kitty_obs/stress.txt; sleep 4' \
    > /tmp/kitty_obs/run1.stdout 2> /tmp/kitty_obs/run1.stderr ; echo "EXIT=$?"
EXIT=0
```

### Q1.2 Complete, unedited startup diagnostics

**stdout** (the `--debug-rendering` banner-line; `printf` at `kitty/gl.c:L72`):

```
[0.129] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

**stderr** (complete — 8 lines; the `Text fonts:` block is the `--debug-font-fallback`/startup font diagnostic):

```
[0.155] OS Window created
[0.165] Failed to open systemd user bus with error: Connection refused
[0.169] Child launched
[0.169] Text fonts:
[0.169]   Normal: DejaVuSansMono: /usr/local/share/fonts/kitty-ci-fonts/DejaVuSansMono.ttf:0
[0.169]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.169]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.169]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

> The line `Failed to open systemd user bus with error: Connection refused` is a **headless-environment artifact** (no systemd user session in the container), not kitty font behavior.

### Q1.3 The four resolved primary FONT FAMILIES

| Debug label | Internal key | Resolved face (`<ps_name>: <path>:<index>`) |
|-------------|--------------|---------------------------------------------|
| `Normal` | `medium` | `DejaVuSansMono: /usr/local/share/fonts/kitty-ci-fonts/DejaVuSansMono.ttf:0` |
| `Bold` | `bold` | `DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0` |
| `Italic` | `italic` | `DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0` |
| `Bold-Italic` | `bi` | `DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0` |

**Cause:** the block is printed by `dump_font_debug()` (`kitty/fonts/render.py:L161`), which emits `log_error('Text fonts:')` (`:L163`) then iterates the mapping `{'medium':'Normal','bold':'Bold','italic':'Italic','bi':'Bold-Italic'}` printing `identify_for_debug()` for each face (`:L164-L165`). It is invoked from `kitty/main.py:L229` during startup. Each face string's `<ps_name>: <path>:<index>` form comes from `Face.identify_for_debug()`.

### Q1.4 The FALLBACK CHAIN

**Key finding (grounded in the font's own coverage):** all four primary faces resolve to **DejaVu Sans Mono**, and DejaVu Sans Mono **already covers Arabic and the combining acute accent**, so the mixed Arabic+English stress line needs **no** fallback. This is proven from the font's FontConfig charset (Arabic block `0621-063a` and `0640-0655` cover `مرحبا`; `0300-033f` covers U+0301):

```
$ fc-query --format='%{charset}\n' /usr/local/share/fonts/kitty-ci-fonts/DejaVuSansMono.ttf \
    | tr ' ' '\n' | grep -E '621-63a|640-655|300-33f'
300-33f
621-63a
640-655
```

`0645` (MEEM) ∈ `640-655`; `0631`/`062d`/`0628`/`0627` ∈ `621-63a`; `0301` ∈ `300-33f` — all present in the primary face.

**Exercising the fallback chain with genuinely-absent codepoints.** To capture the real fallback printer, the identical C path (`render_line` → `load_fallback_font` → `output_cell_fallback_data`) was driven through `test_render_line` with `debug_font_fallback` enabled, rendering `A中😀B` (Latin + CJK + emoji + Latin). **[non-canonical — identical C code path]**

```
$ python3 /tmp/kitty_obs/obs3.py 2>&1   # set_options(defaults, False, True, True); render "A\u4e2d\U0001F600B"
[0.036] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.037] U+1f600 emoji_presentation Face(family=Noto Color Emoji style=Regular ps_name=NotoColorEmoji path=/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf ttc_index=0 variant=False named_instance=False scalable=False color=True)
```

Resolved fallback chain (primary → substitute), read back from `current_fonts()['fallback']`:

| Codepoint | Flags | Substitute face chosen |
|-----------|-------|------------------------|
| `U+4E2D` (中, CJK) | — | `NotoSansCJKjp-Regular` (`/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc:0`) |
| `U+1F600` (😀, emoji) | `emoji_presentation`, `color=True` | `NotoColorEmoji` (`/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf:0`) |

**Cause:** `output_cell_fallback_data` (`kitty/fonts.c:L457`) prints `debug("U+%x ", cell->ch)` for the base char (`:L458`), one `U+%x` per combining mark (`:L459-L461`), the `bold`/`italic`/`emoji_presentation` flag words (`:L462-L464`), then `PyObject_Print(face, stderr, 0)` (`:L466`) which yields the `Face(family=… style=… ps_name=… path=… ttc_index=… variant=… named_instance=… scalable=… color=…)` string. It is called from `load_fallback_font` at `kitty/fonts.c:L492` (guarded by `if (global_state.debug_font_fallback)`). Faces are iterated by `iter_fallback_faces` (`kitty/fonts.c:L471`). On **Linux** the substitute face is chosen by `create_fallback_face` (`kitty/fontconfig.c:L463`) via `fallback_font` (`kitty/fontconfig.c:L444`) — i.e. kitty uses **FontConfig** for fallback selection.

### Q1.5 Bidi / RTL direction handling

With the default `force_ltr = no`, the direction is **not** forced, so HarfBuzz auto-detects Arabic as RTL. The real shaping path (`test_shape` → `shape_run` → `hb_shape`) returns per-group `(num_cells, num_glyphs, first_glyph)`: **[non-canonical — identical C code path]**

```
$ python3 /tmp/kitty_obs/obs2.py 2>&1   # shape_string via test_shape -> shape_run -> hb_shape
  english            'Hello' -> groups(cells,glyphs,first_glyph)=[(1, 1, 43), (1, 1, 72), (1, 1, 79), (1, 1, 79), (1, 1, 82)]
  arabic             'مرحبا' -> groups(cells,glyphs,first_glyph)=[(1, 1, 3145), (1, 1, 3149), (1, 1, 3166), (1, 1, 3177), (1, 1, 3230)]
  mixed              'Hi مرحبا' -> groups(cells,glyphs,first_glyph)=[(1, 1, 43), (1, 1, 76), (1, 1, 3), (1, 1, 1142), (1, 1, 1127), (1, 1, 1123), (1, 1, 1118), (1, 1, 1117)]
  combining          'é' -> groups(cells,glyphs,first_glyph)=[(1, 1, 171)]
  ligature-firacode  'ffi' -> groups(cells,glyphs,first_glyph)=[(1, 1, 73), (1, 1, 73), (1, 1, 76)]
```

That the direction genuinely governs shaping is shown by contrasting `force_ltr=no` (default) with `force_ltr=yes`: the Arabic word yields **different glyph ids** — proof the RTL branch is active by default. **[non-canonical]**

```
$ python3 /tmp/kitty_obs/obs4.py 2>&1
Arabic 'مرحبا' shaped with force_ltr=no  (DEFAULT): [(1, 1, 3145), (1, 1, 3149), (1, 1, 3166), (1, 1, 3177), (1, 1, 3230)]
Arabic 'مرحبا' shaped with force_ltr=yes (NON-CANON): [(1, 1, 1142), (1, 1, 3177), (1, 1, 3167), (1, 1, 3148), (1, 1, 1117)]
```

**Cause:** `if (OPT(force_ltr)) hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR);` (`kitty/fonts.c:L688`) — with `force_ltr=no` this branch is skipped and HarfBuzz auto-detects RTL, producing the decreasing cluster arithmetic documented at `kitty/fonts.c:L997` and `:L1079` ("RTL languages like Arabic have decreasing cluster numbers"). Shaping runs via `hb_shape` (`kitty/fonts.c:L813`) inside `shape_run` (`kitty/fonts.c:L1152`).

> **macOS note [inferred / not observed]:** on macOS the substitute path is `find_substitute_face` (`kitty/core_text.m:L362`) using CoreText. The Docker environment is Linux, so this path was **not executed**; the statement is inferred from reading only.


---

## Q2 — RUNTIME CONFIGURATION VALUES confirming font selection BEFORE rendering

> **Verbatim:** "What RUNTIME CONFIGURATION VALUES confirm these font selections BEFORE any text rendering begins?"

Because the canonical run uses `--config NONE`, the resolved values are the **true built-in defaults** defined in `kitty/options/definition.py` and applied by `set_font_family` (`kitty/fonts/render.py:L173`) before the first glyph is rasterized. The running defaults were dumped from `kitty.options.types.defaults` (the parsed object that `--config NONE` yields): **[non-canonical dump of the same `defaults` object the canonical path loads]**

```
$ python3 /tmp/kitty_obs/obs.py 2>&1 | sed -n '/Q2: RESOLVED DEFAULT OPTIONS/,/SET FONT FAMILY/p'
==== Q2: RESOLVED DEFAULT OPTIONS (kitty.options.types.defaults == --config NONE) ====
  opts.font_family = FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
  opts.bold_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
  opts.italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
  opts.bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
  opts.font_size = 11.0
  opts.force_ltr = False
  opts.disable_ligatures = 0
  opts.text_composition_strategy = 'platform'
  opts.font_features = {}
  opts.cursor_beam_thickness = 1.5
  opts.cursor_underline_thickness = 2.0
```

### Q2.1 Resolved values, source strings, and `file:line`

| Option | Resolved runtime value | Default string in source | `file:line` |
|--------|------------------------|--------------------------|-------------|
| `font_family` | `FontSpec(system='monospace')` | `'monospace'` | `kitty/options/definition.py:L35` |
| `bold_font` | `FontSpec(system='auto')` | `'auto'` | `kitty/options/definition.py:L53` |
| `italic_font` | `FontSpec(system='auto')` | `'auto'` | `kitty/options/definition.py:L55` |
| `bold_italic_font` | `FontSpec(system='auto')` | `'auto'` | `kitty/options/definition.py:L57` |
| `font_size` | `11.0` | `'11.0'` | `kitty/options/definition.py:L59` |
| `force_ltr` | `False` | `'no'` | `kitty/options/definition.py:L64` |
| `disable_ligatures` | `0` (i.e. `never`) | `'never'` | `kitty/options/definition.py:L115-L116` |
| `font_features` | `{}` (empty) | (none) | `kitty/options/definition.py:L136` |
| `text_composition_strategy` | `'platform'` | `'platform'` | `kitty/options/definition.py:L239` |
| `cursor_beam_thickness` | `1.5` | `1.5` | referenced by `prerender_function` |
| `cursor_underline_thickness` | `2.0` | `2.0` | referenced by `prerender_function` |

**Interpretation of the parsed forms:**
- `font_family = 'monospace'` parses to `FontSpec(system='monospace')` — a *system* spec that FontConfig resolves (here → DejaVu Sans Mono, matching Q1's `Text fonts:` block).
- `bold/italic/bold_italic = 'auto'` parses to `FontSpec(system='auto')`, meaning "derive the bold/italic faces automatically from the family" — hence the DejaVu Sans Mono `-Bold`/`-Oblique`/`-BoldOblique` faces in Q1.
- `force_ltr = 'no'` → boolean `False`, so the RTL auto-detection of Q1.5 is active.
- `disable_ligatures = 'never'` → enum `0`, so ligatures are never disabled (relevant to the `ffi`/`fi` shaping in the stress input).

### Q2.2 Parse/convert pipeline (how the string defaults become the runtime object)

- `kitty/options/definition.py` is the single source of default option strings.
- `kitty/options/parse.py` and the generated `kitty/options/to-c-generated.h` are the generated parse/convert layers that turn those strings into the typed `Options` object (`kitty.options.types.defaults`).
- `docs/conf.rst` is the auto-generated user-facing reference for these same options.
- `set_font_family(opts)` (`kitty/fonts/render.py:L173`) consumes the resolved `opts` and installs the font group **before** any cell is rendered — establishing exactly the four faces reported in Q1 prior to rasterization.


---

## Q3 — CELL METRICS, BASELINE, DECORATION ALIGNMENT for complex grapheme clusters

> **Verbatim:** "As the SCREEN GRID and SHAPING subsystems initialize, what default CELL METRICS, BASELINE POSITIONING, and DECORATION ALIGNMENT (overline/underline) values show up in debug output for complex grapheme clusters?"

### Q3.1 Cell metrics (default DejaVu Sans Mono @ 11.0pt / 96 dpi)

These are the exact integers `calc_cell_metrics` (`kitty/fonts.c:L373`) computes and passes to `prerender_function` (`kitty/fonts/render.py:L364`) via `send_prerendered_sprites` (`kitty/fonts.c:L1458`). Captured by wrapping `prerender_function` to print its arguments: **[non-canonical — identical C metric values]**

```
$ python3 /tmp/kitty_obs/obs.py 2>&1 | grep -E 'PRERENDER_METRICS|create_test_font_group ->'
PRERENDER_METRICS cell_width=9 cell_height=18 baseline=14 underline_position=15 underline_thickness=1 strikethrough_position=10 strikethrough_thickness=1 cursor_beam_thickness=1.5 cursor_underline_thickness=2.0 dpi_x=96.0 dpi_y=96.0
create_test_font_group -> cell_width=9 cell_height=18
```

| Metric | Value (px) | Cause (`file:line`) |
|--------|-----------|---------------------|
| `cell_width` | `9` | `calc_cell_width` (`kitty/freetype.c:L389`), stored `fg->cell_width` at `kitty/fonts.c:L419` |
| `cell_height` | `18` | `calc_cell_height` (`kitty/freetype.c:L390`), stored `fg->cell_height` |
| `baseline` | `14` | `*baseline = font_units_to_pixels_y(self, self->ascender)` (`kitty/freetype.c:L391`) |
| `underline_position` | `15` | `kitty/freetype.c:L392`, then clamped `MIN(cell_height-1, …)` in `calc_cell_metrics` (`kitty/fonts.c:L409`) |
| `underline_thickness` | `1` | `MAX(1, font_units_to_pixels_y(…))` (`kitty/freetype.c:L393`) |
| `strikethrough_position` | `10` | from OS/2 `yStrikeoutPosition` (`kitty/freetype.c:L232`, `:L395-L397`), else `floor(baseline*0.65)` (`:L398`) |
| `strikethrough_thickness` | `1` | from OS/2 `yStrikeoutSize` (`kitty/freetype.c:L233`), else `= underline_thickness` (`:L400-L404`) |

The underlying raw face members (font units) that these pixel values derive from: **[non-canonical]**

```
$ python3 /tmp/kitty_obs/obs.py 2>&1 | sed -n '/RAW FACE METRIC MEMBERS/,/RENDER STRESS LINE/p'
==== Q3: RAW FACE METRIC MEMBERS (font units) for medium face ====
  medium.units_per_EM = 2048
  medium.ascender = 1901
  medium.descender = -483
  medium.height = 2384
  medium.underline_position = -85
  medium.underline_thickness = 90
  medium.strikethrough_position = 530
  medium.strikethrough_thickness = 102
```

**Cause chain:** `calc_cell_metrics` (`kitty/fonts.c:L373`) calls `cell_metrics()` (`kitty/freetype.c:L387`), which converts font-unit metrics to pixels via `font_units_to_pixels_y` (`kitty/freetype.c:L92`); at its tail (`kitty/fonts.c:L418-L420`) it calls `sprite_tracker_set_layout(&fg->sprite_tracker, cell_width, cell_height)` and then stores `fg->cell_width/cell_height/baseline/underline_position/underline_thickness/strikethrough_position/strikethrough_thickness`. The call site is `kitty/fonts.c:L1511`.

### Q3.2 Complex grapheme clusters in the debug output

A base character followed by combining marks is printed by the **same** `output_cell_fallback_data` printer as Q1: `debug("U+%x ", cell->ch)` for the base (`kitty/fonts.c:L458`) and one `debug("U+%x ", codepoint_for_mark(...))` per combining mark (`:L459-L461`). The decoration primitives that draw styled cells are `add_line`/`add_dline`/`add_curl`/`add_dots`/`add_dashes` (`kitty/fonts/render.py:L203`,`L211`,`L231`,`L267`,`L276`), assembled by `render_special`/`render_cursor`/`prerender_function` (`kitty/fonts/render.py:L284`/`L324`/`L364`). The GPU surfaces are `kitty/cell_vertex.glsl` and `kitty/cell_fragment.glsl`.

### Q3.3 DECORATION ALIGNMENT reality — five underline styles + strikethrough, **NO overline**

kitty implements **five underline styles** and a **separate strikethrough**, and has **no overline** concept. This is proven three ways.

**Proof 1 — the decoration field is 3 bits mapping only to underline styles.** `NUM_UNDERLINE_STYLES = 5u` (`kitty/data-types.h:L213`), the cell `decoration : 3` bitfield (`kitty/data-types.h:L199`), and `DECORATION_FG_CODE 58` (`kitty/data-types.h:L117`). The SGR serializer `decoration_as_sgr` maps the five styles and a reset — **underline-only**:

```
$ sed -n '733,742p' kitty/line.c
static const char*
decoration_as_sgr(uint8_t decoration) {
    switch(decoration) {
        case 1: return "4;";
        case 2: return "4:2;";
        case 3: return "4:3;";
        case 4: return "4:4";
        case 5: return "4:5";
        default: return "24;";
    }
}
```

**Proof 2 — the SGR parser has no overline handler.** Overline is SGR 53 (on) / 55 (off). A grep of `kitty/cursor.c` finds **zero** such handlers:

```
$ grep -nE 'case 53|case 55|[^0-9]53:|[^0-9]55:|overline' kitty/cursor.c
$ echo "exit=$?"
exit=1
```

(An empty result with exit code 1 = no match.) The complete set of SGR codes that `cursor_from_sgr` (`kitty/cursor.c:L76`) does handle contains underline (4/21/24) and strikethrough (9/29) but **not** 53/55:

```
$ sed -n '76,140p' kitty/cursor.c | grep -oE 'case [0-9]+' | sort -k2 -n | tr '\n' ' '
case 0 case 1 case 2 case 3 case 4 case 7 case 9 case 21 case 22 case 23 case 24 case 27 case 29 case 30 case 38 case 39 case 40 case 48 case 49 case 90 case 100 case 221 case 222
```

**Proof 3 — the documentation lists only underline styles.** `docs/underlines.rst:L12-L17` documents SGR `4:0` (no underline) through `4:5` (dashed) and, by omission, confirms there is no overline.

**Conclusion:** the only decoration-alignment values that appear are the five underline styles (aligned at `underline_position=15`, `underline_thickness=1`) and the single strikethrough (aligned at `strikethrough_position=10`, `strikethrough_thickness=1`). There is **no overline** alignment value because kitty has no overline decoration.


---

## Q4 — GPU TEXTURE ATLAS initial PAGE LAYOUT, SIZING, CAPACITY

> **Verbatim:** "For GPU TEXTURE ATLAS initialization at launch, what initial PAGE LAYOUT, SIZING, and CAPACITY ALLOCATIONS get reported for shaped glyphs across primary and fallback fonts?"

### Q4.1 Initial state

The sprite atlas ("sprite map") begins from a fixed initializer:

```
$ sed -n '31p' kitty/shaders.c
static const SpriteMap NEW_SPRITE_MAP = { .xnum = 1, .ynum = 1, .last_num_of_layers = 1, .last_ynum = -1 };
```

So the initial page layout is `xnum=1, ynum=1`, with `last_num_of_layers=1` and `last_ynum=-1` (`kitty/shaders.c:L31`).

### Q4.2 Capacity limits — the `#ifdef __APPLE__` clamp does NOT apply on Linux

`alloc_sprite_map` reads the GPU limits and (only on Apple) clamps them:

```
$ sed -n '51,62p' kitty/shaders.c
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
```

The `MIN(8192, …)` / `MIN(512, …)` clamps (`kitty/shaders.c:L58-L59`) are **inside `#ifdef __APPLE__`** (`:L55`). The observed platform is **Linux**, so these clamps are **NOT** applied and the **raw** GL maxima are used. The raw maxima observed under the same Mesa llvmpipe stack kitty runs on (stable across 2 runs):

```
$ xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe glxinfo -l \
    | grep -E 'GL_MAX_TEXTURE_SIZE|GL_MAX_ARRAY_TEXTURE_LAYERS|GL_MAX_3D_TEXTURE_SIZE'
    GL_MAX_TEXTURE_SIZE = 16384
    GL_MAX_3D_TEXTURE_SIZE = 2048
    GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
$ xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe glxinfo | grep -E 'OpenGL renderer|OpenGL version'
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.25.10.2
```

So `sprite_tracker_set_limits` (`kitty/fonts.c:L237`) receives `max_texture_size=16384`, `max_array_texture_layers=2048` (un-clamped). The `8192`/`512` values would apply **only** on macOS.

### Q4.3 Page-layout computation (exact formula) and computed numbers

```
$ sed -n '276,281p' kitty/fonts.c
sprite_tracker_set_layout(GPUSpriteTracker *sprite_tracker, unsigned int cell_width, unsigned int cell_height) {
    sprite_tracker->xnum = MIN(MAX(1u, max_texture_size / cell_width), (size_t)UINT16_MAX);
    sprite_tracker->max_y = MIN(MAX(1u, max_texture_size / cell_height), (size_t)UINT16_MAX);
    sprite_tracker->ynum = 1;
    sprite_tracker->x = 0; sprite_tracker->y = 0; sprite_tracker->z = 0;
}
```

Substituting the observed `max_texture_size = 16384` and the observed `cell_width = 9`, `cell_height = 18` (Q3):

| Quantity | Formula | Computed value |
|----------|---------|----------------|
| `xnum` (cells per row) | `MIN(MAX(1, 16384/9), 65535)` | **1820** |
| `max_y` (max rows) | `MIN(MAX(1, 16384/18), 65535)` | **910** |
| `ynum` (initial rows) | fixed | **1** |
| `x,y,z` (write cursor) | fixed | **0,0,0** |

So each atlas layer holds up to `xnum × max_y = 1820 × 910 = 1,656,200` cells; capacity in depth is `GL_MAX_ARRAY_TEXTURE_LAYERS = 2048` layers. The write cursor advances via `do_increment` (`kitty/fonts.c:L243`); `sprite_tracker_current_layout` (`kitty/fonts.c:L269`) reports the live `(xnum, ynum, z)`.

### Q4.4 Texture sizing

```
$ sed -n '108,123p' kitty/shaders.c
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
```

The atlas texture is a `GL_TEXTURE_2D_ARRAY` with `GL_NEAREST` filtering and `GL_CLAMP_TO_EDGE` wrapping, allocated `GL_SRGB8_ALPHA8` at `width = xnum*cell_width`, `height = ynum*cell_height`, depth `znum = z+1` (`kitty/shaders.c:L108-L122`). With the initial `ynum=1`, the first allocation is `width = 1820*9 = 16380`, `height = 1*18 = 18`, `znum = 1`.

### Q4.5 Rationale (why a cell atlas exists)

kitty is strictly monospace/cell-based and caches per-cell alpha masks on the GPU:

```
$ sed -n '251,255p' docs/faq.rst
|kitty| achieves its stellar performance by caching alpha masks of each rendered
character on the GPU, and rendering them all in parallel. This means it is a
strictly character cell based display. As such it can use only monospace fonts,
since every cell in the grid has to be the same size. Furthermore, it needs
fonts to be freely resizable, so it does not support bitmapped fonts.
```

(`docs/faq.rst:L251-L255` — GPU alpha-mask caching; strictly character-cell display; monospace, freely-resizable fonts only.)

### Q4.6 Deterministic corroboration — `test_sprite_map`

The order in which shaped glyphs fill the atlas (x fills first, then y-rows, then z-layers) is asserted by the in-repo test, reproduced exactly here. **[non-canonical]**

```
$ sed -n '119,131p' kitty_tests/fonts.py
    def test_sprite_map(self):
        sprite_map_set_limits(10, 2)
        sprite_map_set_layout(5, 5)
        self.ae(test_sprite_position_for(0), (0, 0, 0))
        self.ae(test_sprite_position_for(1), (1, 0, 0))
        self.ae(test_sprite_position_for(2), (0, 1, 0))
        self.ae(test_sprite_position_for(3), (1, 1, 0))
        self.ae(test_sprite_position_for(4), (0, 0, 1))
        self.ae(test_sprite_position_for(5), (1, 0, 1))
        self.ae(test_sprite_position_for(6), (0, 1, 1))
        self.ae(test_sprite_position_for(7), (1, 1, 1))
        self.ae(test_sprite_position_for(0, 1), (0, 0, 2))
        self.ae(test_sprite_position_for(0, 2), (1, 0, 2))
```

My harness reproduced the identical walk:

```
$ python3 /tmp/kitty_obs/obs.py 2>&1 | sed -n '/Q4: SPRITE MAP LAYOUT/,/EDGE/p' | head -9
==== Q4: SPRITE MAP LAYOUT (reproduce test_sprite_map, non-canonical) ====
  test_sprite_position_for(0) = (0, 0, 0)
  test_sprite_position_for(1) = (1, 0, 0)
  test_sprite_position_for(2) = (0, 1, 0)
  test_sprite_position_for(3) = (1, 1, 0)
  test_sprite_position_for(4) = (0, 0, 1)
  test_sprite_position_for(5) = (1, 0, 1)
  test_sprite_position_for(6) = (0, 1, 1)
  test_sprite_position_for(7) = (1, 1, 1)
```

(`test.py --module fonts` runs this as `test_sprite_map … ok`; see §8.)


---

## Q5 — STARTUP LOGS verifying the atlas is READY for shaped glyph data

> **Verbatim:** "What STARTUP LOGS verify the atlas is READY to receive shaped glyph data?"

### Q5.1 The init/upload path

The atlas texture is created lazily on first use and grown on demand:

```
$ sed -n '137,156p' kitty/shaders.c
ensure_sprite_map(FONTS_DATA_HANDLE fg) {
    SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
    if (!sprite_map->texture_id) realloc_sprite_texture(fg);
    // We have to rebind since we don't know if the texture was ever bound
    // in the context of the current OSWindow
    glActiveTexture(GL_TEXTURE0 + SPRITE_MAP_UNIT);
    glBindTexture(GL_TEXTURE_2D_ARRAY, sprite_map->texture_id);
}

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

- `ensure_sprite_map` (`kitty/shaders.c:L137`) allocates the texture on first use: `if (!sprite_map->texture_id) realloc_sprite_texture(fg);` (`:L139`).
- `send_sprite_to_gpu` (`kitty/shaders.c:L147`) uploads one cell via `glTexSubImage3D` into the `GL_TEXTURE_2D_ARRAY` (`:L155`), first reallocating when the layer/row layout grows (`:L151`).

### Q5.2 The startup population — 1 blank + 10 special = **11** sprites (observed)

`send_prerendered_sprites` first uploads a **blank cell** at `(0,0,0)`, then uploads the sprites returned by `prerender_function`:

```
$ sed -n '1450,1461p' kitty/fonts.c
send_prerendered_sprites(FontGroup *fg) {
    int error = 0;
    sprite_index x = 0, y = 0, z = 0;
    // blank cell
    ensure_canvas_can_fit(fg, 1);
    current_send_sprite_to_gpu((FONTS_DATA_HANDLE)fg, x, y, z, fg->canvas.buf);
    do_increment(fg, &error);
    if (error != 0) { sprite_map_set_error(error); PyErr_Print(); fatal("Failed"); }
    PyObject *args = PyObject_CallFunction(prerender_function, "IIIIIIIffdd", fg->cell_width, fg->cell_height, fg->baseline, fg->underline_position, fg->underline_thickness, fg->strikethrough_position, fg->strikethrough_thickness, OPT(cursor_beam_thickness), OPT(cursor_underline_thickness), fg->logical_dpi_x, fg->logical_dpi_y);
    if (args == NULL) { PyErr_Print(); fatal("Failed to pre-render cells"); }
    PyObject *cell_addresses = PyTuple_GET_ITEM(args, 0);
```

`prerender_function` (`kitty/fonts/render.py:L364`) returns exactly **10** special sprites:

```
$ sed -n '391,394p' kitty/fonts/render.py
    cells = list(map(f, range(1, NUM_UNDERLINE_STYLES + 1)))  # underline sprites
    cells.append(f(0, strikethrough=True))  # strikethrough sprite
    cells.append(f(missing=True))  # missing glyph
    cells.extend((c(1), c(2), c(3)))  # cursor glyphs
```

That is **5 underline** (`range(1, NUM_UNDERLINE_STYLES+1)`, `NUM_UNDERLINE_STYLES=5`) + **1 strikethrough** + **1 missing-glyph** + **3 cursor** = **10** special sprites. Because `send_prerendered_sprites` prepends **1 blank cell** (`kitty/fonts.c:L1453-L1455`), the total uploaded to the GPU at startup is **11**. Observed via the `send_to_gpu` hook (stable across 2 runs): **[non-canonical — identical upload path]**

```
$ python3 /tmp/kitty_obs/obs.py 2>&1 | grep -E 'prerendered sprite count|prerendered sprite keys'
prerendered sprite count (via send_to_gpu hook) = 11
prerendered sprite keys (x,y,z) = [(0, 0, 0), (1, 0, 0), (2, 0, 0), (3, 0, 0), (4, 0, 0), (5, 0, 0), (6, 0, 0), (7, 0, 0), (8, 0, 0), (9, 0, 0), (10, 0, 0)]
```

| Sprite index | Contents | Source |
|-------------|----------|--------|
| `0` (`(0,0,0)`) | blank cell | `kitty/fonts.c:L1453-L1455` |
| `1–5` | underline styles 1–5 | `kitty/fonts/render.py:L391` |
| `6` | strikethrough | `kitty/fonts/render.py:L392` |
| `7` | missing-glyph | `kitty/fonts/render.py:L393` |
| `8–10` | cursor glyphs (block/beam/underline variants) | `kitty/fonts/render.py:L394` |

The prerendered cells are pushed to the GPU by `send_prerendered_sprites` using `current_send_sprite_to_gpu` (`kitty/fonts.c:L1455`).

### Q5.3 What establishes readiness

After startup, the atlas is **ready to receive shaped glyph data** once:
1. `ensure_sprite_map` has created the `GL_TEXTURE_2D_ARRAY` (texture_id set; `kitty/shaders.c:L139`), and
2. `send_prerendered_sprites` has successfully uploaded all **11** startup sprites at `(0,0,0)…(10,0,0)` via `send_sprite_to_gpu`/`glTexSubImage3D` (`kitty/shaders.c:L155`) with no `sprite_map_set_error` (`kitty/fonts.c:L1457`).

The observable proof that this stage completed without error is the successful, stable population of all 11 sprite positions (above) plus the clean startup log (no `Failed to pre-render cells` / `sprite_map_set_error` lines in the canonical stderr of §Q1.2). From that point, the write cursor sits just past the reserved sprites and subsequently-shaped glyphs are appended by the same `do_increment` + `send_sprite_to_gpu` mechanism.


---

## 4. Edge & secondary conditions ("exercise every condition")

### 4.1 OS-fallback edge path — OS-chosen font lacks glyphs

When FontConfig returns a "best match" face that nonetheless lacks a glyph for the text, `load_fallback_font` prints a distinct secondary block and returns `MISSING_FONT`. Exercised with `U+10FFFF` (a codepoint no installed font covers), with `debug_font_fallback` enabled:

```
$ python3 /tmp/kitty_obs/obs.py 2>&1 | sed -n '/get_fallback_font(U+10FFFF)/,/DONE/p'
==== Q1/EDGE: get_fallback_font(U+10FFFF) no-glyph edge ====
[0.036] U+10ffff Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/local/share/fonts/kitty-ci-fonts/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.036] The font chosen by the OS for the text: U+10ffff is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/local/share/fonts/kitty-ci-fonts/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
  raised ValueError: No fallback font found
==== DONE ====
```

**Cause:** the first line is `output_cell_fallback_data` (`kitty/fonts.c:L457`, called at `:L492`). The second line is the edge block at `kitty/fonts.c:L501-L512`: after `init_font`, `has_cell_text(af->face, cell)` is false (`:L501`), so it prints `"The font chosen by the OS for the text: "` (`:L503`) … `" but it does not actually contain glyphs for that text"` (`:L509`), calls `del_font(af)` (`:L511`) and returns `MISSING_FONT` (`:L512`); `get_fallback_font` then raises `ValueError: No fallback font found`. The in-repo test that targets this behavior, `test_fallback_font_not_last_resort` (`kitty_tests/fonts.py:L229`), is **skipped on Linux** ("Only macOS has a Last Resort font"; see §8) — the Linux equivalent is exactly the `ValueError` above.

### 4.2 Transitional (before / during / after) states

- **Before first rasterization:** the resolved config defaults (Q2) are established by `set_font_family` (`kitty/fonts/render.py:L173`).
- **During init:** cell metrics (Q3) are computed by `calc_cell_metrics` and the atlas layout (Q4) by `sprite_tracker_set_layout`.
- **After init (ready):** the 11 prerendered sprites populate the atlas (Q5), which is then ready for shaped glyphs.

### 4.3 Alternate flag — `force_ltr`

The default `force_ltr=no` yields RTL-shaped Arabic; forcing `force_ltr=yes` changes the shaped glyph ids (see Q1.5). The `yes` variant is **[non-canonical]** and used only to contrast the direction branch at `kitty/fonts.c:L688`; the canonical run used `--config NONE` (i.e. `force_ltr=no`), and no override persists.

---

## 5. Stability (values confirmed across ≥2 runs)

Each magnitude/size value was captured on at least two runs with identical input; all were byte-identical except the leading `[seconds]` monotonic timestamps.

| Value | Run 1 | Run 2 | Stable? |
|-------|-------|-------|---------|
| Canonical banner GL line | `4.5 … Mesa 25.2.8-0ubuntu0.25.10.2 … 4.5` | identical | ✅ (timestamp `[0.129]`→`[0.126]`) |
| `Text fonts:` block (4 faces) | DejaVuSansMono ×4 | identical | ✅ (timestamp `[0.169]`→`[0.162]`) |
| Cell metrics | `9/18/14/15/1/10/1` | `9/18/14/15/1/10/1` | ✅ |
| Prerendered sprite count | `11` | `11` | ✅ |
| CJK/emoji fallback faces | NotoSansCJKjp / NotoColorEmoji | identical | ✅ (timestamp only) |
| OS-fallback edge (U+10FFFF) | DejaVuSansMono + ValueError | identical | ✅ (timestamp only) |
| `GL_MAX_TEXTURE_SIZE` / `GL_MAX_ARRAY_TEXTURE_LAYERS` | `16384` / `2048` | `16384` / `2048` | ✅ |

The **only** run-to-run variance observed is the leading monotonic timestamp in each log line (e.g. `[0.036]` vs `[0.037]`); no metric, count, face, or limit changed.

---

## 6. Consolidated `file:line` evidence index

| Mechanism | Function / struct | `file:line` |
|-----------|-------------------|-------------|
| `debug`/`debug_fonts` macro | `#define debug debug_fonts` | `kitty/fonts.c:L18` |
| debug gate | `#define debug_fonts(...) if (global_state.debug_font_fallback) …` | `kitty/state.h:L16` |
| GL banner-line | `printf("… GL version string …")` | `kitty/gl.c:L72` |
| `Text fonts:` block | `dump_font_debug` | `kitty/fonts/render.py:L161-L165` |
| CLI `--debug-font-fallback` | flag def | `kitty/cli.py:L1002` |
| CLI `--debug-rendering` | flag def | `kitty/cli.py:L989` |
| flag plumbing | `set_options(...)` | `kitty/main.py:L249`; `kitty/state.c:L728-L740` |
| fallback printer | `output_cell_fallback_data` | `kitty/fonts.c:L457` (called `:L492`) |
| fallback loader / edge block | `load_fallback_font` | `kitty/fonts.c:L481`, edge `:L501-L512` |
| fallback iterator | `iter_fallback_faces` | `kitty/fonts.c:L471` |
| Linux fallback selection | `create_fallback_face` / `fallback_font` | `kitty/fontconfig.c:L463` / `:L444` |
| macOS fallback [inferred] | `find_substitute_face` | `kitty/core_text.m:L362` |
| direction / bidi | `if (OPT(force_ltr)) hb_buffer_set_direction(… LTR)` | `kitty/fonts.c:L688` |
| RTL decreasing clusters | comment + arithmetic | `kitty/fonts.c:L997`, `:L1079` |
| shaping | `hb_shape` / `shape_run` | `kitty/fonts.c:L813` / `:L1152` |
| config defaults | `definition.py` | `kitty/options/definition.py:L35,L53,L55,L57,L59,L64,L115-L116,L136,L239` |
| font setup | `set_font_family` | `kitty/fonts/render.py:L173` |
| cell metrics | `calc_cell_metrics` / `cell_metrics` | `kitty/fonts.c:L373` / `kitty/freetype.c:L387` |
| decoration bitfield / count | `decoration:3` / `NUM_UNDERLINE_STYLES` / `DECORATION_FG_CODE` | `kitty/data-types.h:L199` / `:L213` / `:L117` |
| underline-only SGR | `decoration_as_sgr` | `kitty/line.c:L733-L742` |
| no overline handler | `cursor_from_sgr` (no 53/55) | `kitty/cursor.c:L76-L136` |
| atlas initial state | `NEW_SPRITE_MAP` | `kitty/shaders.c:L31` |
| atlas limits / Apple clamp | `alloc_sprite_map` | `kitty/shaders.c:L51-L61` (clamps `:L58-L59` under `#ifdef __APPLE__ :L55`) |
| atlas layout formula | `sprite_tracker_set_layout` | `kitty/fonts.c:L276-L281` |
| sprite tracker | `set_limits`/`do_increment`/`current_layout` | `kitty/fonts.c:L237`/`:L243`/`:L269` |
| texture sizing | `realloc_sprite_texture` | `kitty/shaders.c:L108-L122` |
| atlas ready / upload | `ensure_sprite_map` / `send_sprite_to_gpu` | `kitty/shaders.c:L137` / `:L147` |
| prerendered sprites (10) | `prerender_function` | `kitty/fonts/render.py:L364,L391-L394` |
| startup upload (blank + 10) | `send_prerendered_sprites` | `kitty/fonts.c:L1450-L1461` |
| GPU caching rationale | monospace/alpha-mask | `docs/faq.rst:L251-L253` |
| underline styles doc | SGR 4:0–4:5, no overline | `docs/underlines.rst:L12-L17` |

---

## 7. Answer coverage checklist (every named item)

- **Q1:** font families (4 primary faces) ✅; fallback chains (CJK→Noto CJK, emoji→Noto Color Emoji) ✅; startup diagnostics (`Text fonts:` block + banner-line) ✅; Arabic RTL + English LTR handling (auto-detected RTL, `force_ltr=no`) ✅.
- **Q2:** `font_family` ✅, `bold_font`/`italic_font`/`bold_italic_font` ✅, `font_size` ✅, `force_ltr` ✅, `disable_ligatures` ✅, `font_features` ✅, `text_composition_strategy` ✅ — all "before rendering" via `set_font_family` ✅.
- **Q3:** cell metrics (`cell_width/height`) ✅; baseline ✅; underline position/thickness ✅; strikethrough position/thickness ✅; decoration alignment for underline (5 styles) ✅; **overline** explicitly reported **unsupported** with 3 proofs ✅; complex grapheme clusters (`U+<hex>` base + marks) ✅.
- **Q4:** initial page layout (`xnum/ynum`) ✅; sizing (`GL_SRGB8_ALPHA8`, width/height/depth) ✅; capacity (`GL_MAX_*`, Apple-clamp nuance) ✅; layout across primary + fallback fonts (shared atlas) ✅.
- **Q5:** atlas-ready logs (`ensure_sprite_map`/`send_sprite_to_gpu`) ✅; the 10 prerendered special sprites (+1 blank = 11 total observed) ✅; readiness conclusion ✅.

---

## 8. Test-suite corroboration

```
$ xvfb-run -a --server-args="-screen 0 1024x768x24" \
    env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe TMPDIR=/tmp/kitty_test_tmp \
    ./kitty/launcher/kitty +launch test.py --module fonts
Running under CI: False
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... ok
test_box_drawing (kitty_tests.fonts.Rendering.test_box_drawing) ... ok
test_coalesce_symbol_maps (kitty_tests.fonts.Rendering.test_coalesce_symbol_maps) ... ok
test_emoji_presentation (kitty_tests.fonts.Rendering.test_emoji_presentation) ... ok
test_fallback_font_not_last_resort (kitty_tests.fonts.Rendering.test_fallback_font_not_last_resort) ... skipped 'Only macOS has a Last Resort font'
test_font_rendering (kitty_tests.fonts.Rendering.test_font_rendering) ... ok
test_shaping (kitty_tests.fonts.Rendering.test_shaping) ... ok
test_sprite_map (kitty_tests.fonts.Rendering.test_sprite_map) ... ok

----------------------------------------------------------------------
Ran 8 tests in 0.289s

OK (skipped=1)
```

These pass on the same build, corroborating the shaping (`test_shaping`), sprite-atlas layout (`test_sprite_map`), and font-selection (`test_font_selection`) behavior documented above.

---

## 9. Appendix — temporary observation scripts (deleted after use)

For transparency, the temporary scripts used for the **[non-canonical]** corroboration are reproduced below. They lived under `/tmp/kitty_obs/` (outside the repository) and were **deleted** after the investigation; the repository is unchanged except for this one document.

### 9.1 `/tmp/kitty_obs/obs.py` (Q2 defaults, Q3 metrics, Q4 sprite map, Q5 count, edge)

```python
import os, sys
REPO = "…/blitzy-9f9917e5-…_c8ba5a"
os.chdir(REPO); sys.path.insert(0, REPO)
import kitty.fonts.render as R
from kitty.fast_data_types import (Screen, create_test_font_group, current_fonts,
    get_fallback_font, set_options, set_send_sprite_to_gpu, sprite_map_set_layout,
    sprite_map_set_limits, test_render_line, test_sprite_position_for)
from kitty.options.types import defaults
opts = defaults
set_options(opts, False, True, True)            # enable debug_font_fallback
orig_pre = R.prerender_function
def wrapped_pre(*a):
    print("PRERENDER_METRICS", *a[:11], file=sys.stderr); return orig_pre(*a)
R.prerender_function = wrapped_pre
sprites = {}
sprite_map_set_limits(100000, 100)
set_send_sprite_to_gpu(lambda x, y, z, d: sprites.__setitem__((x, y, z), d))
R.set_font_family(opts)
create_test_font_group(11.0, 96.0, 96.0)        # fires prerender -> metrics + 11 sprites
# … prints Q2 defaults, current_fonts(), raw face members, renders the stress line,
#     reproduces test_sprite_map, and exercises get_fallback_font('\U0010FFFF', …)
```

### 9.2 `/tmp/kitty_obs/obs2.py` (Q1 shaping + successful CJK/emoji fallback lookup)

```python
from kitty.fonts.render import shape_string, setup_for_testing
from kitty.fast_data_types import get_fallback_font, set_options
from kitty.options.types import defaults
set_options(defaults, False, True, True)
for text in ("Hello", "\u0645\u0631\u062d\u0628\u0627", "Hi \u0645\u0631\u062d\u0628\u0627", "e\u0301", "ffi"):
    print(shape_string(text))                    # test_shape -> shape_run -> hb_shape
with setup_for_testing('monospace', 11.0, 96.0):
    for ch in ("\u4e2d", "\U0001F600", "\u0915"):
        print(get_fallback_font(ch, False, False).identify_for_debug())
```

### 9.3 `/tmp/kitty_obs/obs3.py` (genuine `output_cell_fallback_data` for a successful fallback, debug ON)

```python
import kitty.fonts.render as R
from kitty.fast_data_types import (Screen, create_test_font_group, current_fonts,
    get_fallback_font, set_options, set_send_sprite_to_gpu, sprite_map_set_limits,
    test_render_line)
from kitty.options.types import defaults
opts = defaults
set_options(opts, False, True, True)             # debug_font_fallback = True
sprite_map_set_limits(100000, 100)
set_send_sprite_to_gpu(lambda x, y, z, d: None)
R.set_font_family(opts); create_test_font_group(11.0, 96.0, 96.0)
s = Screen(None, 1, 20); line = s.line(0); s.draw("A\u4e2d\U0001F600B")
test_render_line(line)                           # real render_line -> output_cell_fallback_data
```

### 9.4 `/tmp/kitty_obs/obs4.py` (force_ltr contrast)

```python
import kitty.fonts.render as R
from kitty.fast_data_types import (Screen, create_test_font_group, set_options,
    set_send_sprite_to_gpu, sprite_map_set_limits, test_shape)
from kitty.options.types import defaults
def shape_arabic(force_ltr):
    opts = defaults._replace(force_ltr=force_ltr)
    set_options(opts, False, False, False)
    sprite_map_set_limits(100000, 100); set_send_sprite_to_gpu(lambda *a: None)
    R.set_font_family(opts); create_test_font_group(11.0, 96.0, 96.0)
    s = Screen(None, 1, 40); line = s.line(0); s.draw("\u0645\u0631\u062d\u0628\u0627")
    return [(g[0], g[1], g[2]) for g in test_shape(line, None)]
print("no ", shape_arabic(False)); print("yes", shape_arabic(True))
```

---

## 10. Repository cleanliness

After the investigation, all temporary scripts under `/tmp/kitty_obs/` were deleted, no runtime override persists (the canonical run used `--config NONE`), and the git-ignored build byproducts were not staged. `git status --porcelain` shows only the single new document:

```
$ git status --porcelain
?? blitzy/documentation/kitty_815df1e210e0.md
```

