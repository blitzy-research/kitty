# kitty Startup Font, Shaping, Cell-Metric & GPU Atlas Initialization — Investigation

> A read-only, code-grounded investigation answering four questions about how the
> [kitty](https://sw.kovidgoyal.net/kitty/) terminal emulator initializes its text-shaping,
> font-fallback, cell-metric, and GPU texture-atlas subsystems at startup. Every answer below
> pairs **reproduced debug output** with the **exact emitting source locator** (`file:line` +
> function) and a **rationale** explaining *why* each value appears.

---

## Methodology / Environment

| Item | Value |
|------|-------|
| Repository | `kitty` (terminal emulator, Kovid Goyal) |
| HEAD commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Document filename | `kitty_815df1e210e0.md` — derived from the source branch name `kitty_815df1e210e0` (the `<source_branch_name>.md` rule) |
| Reference image (build/run env) | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`) — the authoritative toolchain + native-library environment |
| Build command (run + result) | `python3 setup.py --ignore-compiler-warnings` — **executed in this environment and succeeded (exit code 0)**, finishing with `Linking kitty/fast_data_types ... done`. Builds a C extension + the Go `kitten` binary (`pyproject.toml` `requires-python = ">=3.8"`, `go.mod` `go 1.22`). The `--ignore-compiler-warnings` flag is **required** on this host because wayland-protocols 1.45 adds `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that trip `-Werror=switch` in `glfw/wl_window.c` (system-library drift, not a code defect). All build artifacts are git-ignored, so the source tree stays clean. |
| Toolchain present | gcc 15.2.0, Python 3.13.7, Go 1.23.4, pkg-config; native libs HarfBuzz, FreeType, FontConfig, libpng, OpenGL/Mesa, GLFW |
| GPU / GL backend | Mesa **llvmpipe** software rasterizer (`LLVM 20.1.8`), OpenGL **4.5 (Core Profile)** |
| Headless mechanism (actual) | `xvfb-run` virtual X server (`DISPLAY=:99`, no physical display) → kitty selects the **GLFW X11 backend** (`glfw-x11.so`) because `WAYLAND_DISPLAY` is unset and this is not macOS (`init_glfw`, `kitty/main.py:96`; `detect_if_wayland_ok`, `kitty/constants.py:196-204`). Confirmed live by `debug_config`'s `Running under: X11` line (see Q2). |
| Headless backend vs AAP plan | The AAP anticipated the **GLFW Null / OSMesa** backend for headless runs. `libOSMesa.so` *is* installed on this host, but it was **not** the path taken: `init_glfw` only ever selects `cocoa`/`wayland`/`x11`, and under `xvfb` the **x11** backend is chosen, with **llvmpipe** supplying software GL 4.5. GLFW-Null/OSMesa is a valid alternative headless route but is not what produced the captures below. |
| Resolved default font | `font_family monospace` → **DejaVu Sans Mono** on this host |

### How the diagnostics were produced

kitty was built from source in this environment with `python3 setup.py --ignore-compiler-warnings`
(exit code 0 — see the Methodology table), producing the git-ignored artifacts
`kitty/launcher/kitty`, `kitty/fast_data_types.so`, and `kitty/glfw-x11.so`/`kitty/glfw-wayland.so`.
All verbose output was then enabled through **runtime CLI flags only**; no source default was
changed and **no `kitty.conf` was written**. Runs were isolated to throw-away directories so
nothing persisted:

```sh
export KITTY_CONFIG_DIRECTORY=/tmp/blitzy_kitty_conf   # left EMPTY → built-in defaults only
export KITTY_CACHE_DIRECTORY=/tmp/blitzy_kitty_cache

# Q4 — GL/atlas readiness log:
xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --debug-rendering --debug-gl \
    sh -c 'printf "ready\n"; sleep 0.8; exit 0'

# Q1/Q2 — startup font selection diagnostics on the mixed Arabic/English sample:
xvfb-run -a -s "-screen 0 1280x800x24" ./kitty/launcher/kitty --debug-font-fallback \
    sh -c 'printf "Hello \u0645\u0631\u062d\u0628\u0627 World \u0639\u0631\u0628\u0649 123\n"; sleep 1.5; exit 0'
```

The operative example for Q2 is the ephemeral mixed sample **`Hello مرحبا World عربى 123`**
(Latin LTR + Arabic RTL + digits). It is never persisted.

**Observation note (important for honesty).** A windowed terminal only *shapes* and *rasterizes*
text when it actually draws, and under a headless `xvfb` server kitty elides drawing for a window
it treats as not visible. The per-glyph fallback log, the exact cell metrics, the shaped-run
structure, and the atlas sprite positions are produced on that draw path. To exercise those code
paths deterministically — **without modifying any source file** — this investigation drove kitty's
*own* public test API through `kitty +runpy` (a built-in subcommand that runs Python inside the
kitty interpreter and persists nothing):

- `kitty.fonts.render.setup_for_testing` / `set_font_family` / `dump_font_debug`
- `kitty.fast_data_types`: `test_shape`, `test_render_line`, `get_fallback_font`,
  `test_sprite_position_for`, `sprite_map_set_limits`, `sprite_map_set_layout`, `set_options`

Verbose font-fallback logging was switched on inside that harness exactly as the application does
it — by calling `set_options(get_options(), is_wayland=False, debug_rendering=False,
debug_font_fallback=True)` ([`kitty/state.c:726`](../../kitty/state.c), the `set_options`
`PYWRAP`). This is the same global flag the `--debug-font-fallback` CLI flag sets, so the C print
sites emit identical output. Every captured line below is real program output, not hand-written.

### Platform scope

kitty's font layer is platform-split. This investigation ran on **Linux**, where font discovery,
matching, and fallback go through **FontConfig** ([`kitty/fontconfig.c`](../../kitty/fontconfig.c))
and rasterization/metrics through **FreeType** ([`kitty/freetype.c`](../../kitty/freetype.c)). On
**macOS** the same responsibilities are served by **CoreText**
([`kitty/core_text.m`](../../kitty/core_text.m)); the one place this changes an *answer* is the
exact format of the debug font string (noted in Q2).

### Restoration

All instrumentation was transient: CLI flags, `+runpy` scripts kept under `/tmp`, and ephemeral
text samples persist nothing. No `kitty.conf` was created, no source default was edited, and the
compiled artifacts are git-ignored. The delivered state of the source tree is a **clean working
tree** — `git status --porcelain` is **empty** — with this document committed in the delivery
commit; `git diff <baseline>..HEAD --name-status` shows exactly one change,
`A  blitzy/documentation/kitty_815df1e210e0.md`.

---

## Q1 — Shaping & fallback configuration: "How does the text shaping and layout engine configure support for complex Unicode — ligatures, bidi (bidirectional), combining diacritics — and font fallback during initial startup?"

**Question (verbatim):** *"How does the text shaping and layout engine configure support for
complex Unicode — ligatures, bidi (bidirectional), combining diacritics — and font fallback during
initial startup?"*

kitty's text pipeline is **Python-orchestrated, C-executed**. At startup, Python selects the four
text faces and hands them — together with the user's `font_features` — to the C extension, which
configures HarfBuzz, computes metrics, and lays out the GPU atlas. The handoff is
[`set_font_family`](../../kitty/fonts/render.py) →
[`set_font_data`](../../kitty/fonts/render.py):

```python
# kitty/fonts/render.py
173  def set_font_family(opts: Optional[Options] = None, override_font_size: Optional[float] = None) -> None:
...
189      set_font_data(
190          render_box_drawing, prerender_function, descriptor_for_idx,
             ... opts.font_features.copy() ...
         )
```

Everything else in this answer happens on the C side, in
[`kitty/fonts.c`](../../kitty/fonts.c).

### (a) Reproduced output — shaped runs

Driving kitty's own `test_shape` over four representative inputs (`shape_string` →
`fast_data_types.test_shape`) returns one tuple **`(num_cells, num_glyphs, first_glyph,
glyph_ids)`** per shaped group:

```text
##### Q1/Q3: test_shape() groups = (num_cells, num_glyphs, first_glyph, glyph_ids) #####
  [Latin 'Hello']                         -> [(1, 1, 43, (43,)), (1, 1, 72, (72,)), (1, 1, 79, (79,)), (1, 1, 79, (79,)), (1, 1, 82, (82,))]
  [combining 'e'+U+0301+U+0304]           -> [(1, 2, 171, (171, 652))]
  [Arabic U+0645 U+0631 U+062D U+0628 U+0627] -> [(1, 1, 3145, (3145,)), (1, 1, 3149, (3149,)), (1, 1, 3166, (3166,)), (1, 1, 3177, (3177,)), (1, 1, 3230, (3230,))]
  [ligature-candidate '=> != ==']         -> [(1, 1, 32, (32,)), (1, 1, 33, (33,)), (1, 1, 3, (3,)), (1, 1, 4, (4,)), (1, 1, 32, (32,)), (1, 1, 3, (3,)), (1, 1, 32, (32,)), (1, 1, 32, (32,))]
```

Three behaviors are visible and each is explained from code below:

1. **Combining diacritics** — `e` + `U+0301` (combining acute) + `U+0304` (combining macron)
   collapses to **one cell carrying two glyphs** `(1, 2, 171, (171, 652))`: the base and its mark(s)
   are shaped together in the *same* HarfBuzz run.
2. **Arabic (RTL run)** — the five Arabic code points shape to five contextual glyph forms, one
   cell each, in a right-to-left run.
3. **Ligatures** — `=> != ==` stays as eight separate single-glyph cells here because DejaVu Sans
   Mono carries no `calt` ligature substitutions; the `calt` feature *is* enabled (below), the font
   simply has nothing to substitute.

### (b) Emitting source locators

| Concern | Locator | What it does |
|---------|---------|--------------|
| Feature templates | `kitty/fonts.c:42` (`hb_feature_t hb_features[3]`), `:45` (`enum { LIGA_FEATURE, DLIG_FEATURE, CALT_FEATURE }`) | the three OpenType feature slots |
| Feature creation | `kitty/fonts.c:1755-1757` in `init_fonts` | `create_feature("-liga", …)`, `create_feature("-dlig", …)`, `create_feature("-calt", …)` |
| Per-face feature assignment | `kitty/fonts.c:294` `init_font` (`calt` appended at `:314` and always at `:325`; `liga`/`dlig` for `NimbusMonoPS-*` at `:321-323`) | builds each face's `ffs_hb_features` |
| Combining marks into buffer | `kitty/fonts.c:672` `load_hb_buffer` (mark loop `:682` via `codepoint_for_mark`) | base + combining marks enter the same run |
| Script/direction detection | `kitty/fonts.c:687` `hb_buffer_guess_segment_properties` | per-run script + direction (RTL for Arabic) |
| `force_ltr` override | `kitty/fonts.c:688` `if (OPT(force_ltr)) hb_buffer_set_direction(…, HB_DIRECTION_LTR)` | forces LTR when configured |
| Shaping + feature masking | `kitty/fonts.c:786` `shape` → `:812` (the `-calt` mask) → `:813` `hb_shape` | runs HarfBuzz with the per-font features |
| Fallback entry | `kitty/fonts.c:481` `load_fallback_font` (guard `:482`, debug emit `:492`) | loads a face for a missing glyph |
| Linux discovery | `kitty/fontconfig.c:444` `fallback_font`, `:276/:312` `FcFontMatch`, `:463` `create_fallback_face`, `:362-364` monospace fallback | FontConfig matching |
| macOS discovery | `kitty/core_text.m` (CoreText descriptors; `CTFontCopyPostScriptName` `:96`) | CoreText matching |

### (c) Rationale

**Ligatures are driven by the OpenType `calt` feature, and are ON by default — via a
double-negative.** This is the subtle part, and the code (not intuition) is the truth. At startup
`init_fonts` creates all three feature objects as *disable* strings:

```c
// kitty/fonts.c  (init_fonts)
1755  if (!create_feature("-liga", LIGA_FEATURE)) return false;
1756  if (!create_feature("-dlig", DLIG_FEATURE)) return false;
1757  if (!create_feature("-calt", CALT_FEATURE)) return false;
```

`init_font` then appends a trailing `-calt` to *every* face's feature list
(`kitty/fonts.c:325`), and for the bundled `NimbusMonoPS-*` family it also adds `-liga`/`-dlig`
(`kitty/fonts.c:321-323`). At shaping time, `shape` *masks the trailing `-calt` back off* unless
ligatures are being disabled:

```c
// kitty/fonts.c  (shape)
812  if (num_features && !disable_ligature) num_features--; // the last feature is always -calt
813  hb_shape(font, harfbuzz_buffer, group_state.features, num_features);
```

So in the default case the `-calt` entry is dropped before `hb_shape`, HarfBuzz applies its own
built-in `calt`/`liga` defaults, and ligatures render. When `disable_ligatures` is set the `-calt`
is *kept*, explicitly turning the contextual-alternates feature off. This is exactly why the
documented `disable_ligatures` option (`kitty/options/definition.py:115`, default `never`) is the
knob that controls programming ligatures, and why a user can keep `calt` while disabling `liga`
(e.g. the documented `font_features TT2020StyleB-Regular -liga +calt`). The `font_features` option
is indexed by **PostScript name** — confirmed by kitty's docs: the bracketed part of
`kitty +list-fonts --psnames` (e.g. `Fira Code Retina (FiraCode-Retina)`) is the key, and features
such as `+zero`, `+onum`, or stylistic sets `ss01`–`ss20` are applied per face in `init_font`.

**Combining diacritics ride with their base glyph.** `load_hb_buffer` fills the HarfBuzz buffer
from each cell's base code point followed by its stored combining marks
(`codepoint_for_mark(cell->cc_idx[i])`, `kitty/fonts.c:682`), so the base and marks are a single
run and HarfBuzz can compose or position them together — which is precisely the `(1, 2, 171,
(171, 652))` single-cell/two-glyph result observed above.

**RTL is handled per run, not by a full BIDI algorithm.** `hb_buffer_guess_segment_properties`
(`kitty/fonts.c:687`) auto-detects each run's script and direction, so an Arabic run shapes
right-to-left and a Latin run left-to-right. If `force_ltr` is set, `kitty/fonts.c:688` overrides
the direction to `HB_DIRECTION_LTR`.

> **Fidelity note (carried into Q2).** kitty does **not** implement a full Unicode BIDI
> algorithm. Per kitty's own documentation, *"kitty does not support BIDI (bidirectional text),
> however, for RTL scripts, words are automatically displayed in RTL"* — i.e. words/runs are
> reversed during shaping, but logical reordering across a mixed line is not performed, and a
> selection maps to logical (LTR) order. The `force_ltr` option (default `no`,
> `kitty/options/definition.py:64`) forces all text to LTR and is meant to be paired with the
> external **GNU FriBidi** tool to obtain real BIDI. Any answer about "bidi" must describe this
> run-level RTL behavior rather than implying full BIDI support.

**Font fallback is lazy and FontConfig-driven (on Linux).** When a glyph is missing from the
configured family, shaping calls `load_fallback_font` (`kitty/fonts.c:481`), which asks FontConfig
for a face that covers the code point (`fallback_font`/`create_fallback_face`,
`kitty/fontconfig.c:444/463`, via `FcFontMatch`). A guard caps runaway fallback at 100 faces
(`kitty/fonts.c:482`, *"Too many fallback fonts"*). The concrete faces selected at startup are the
subject of Q2.

---

## Q2 — Verbose startup diagnostics for mixed RTL/LTR: "With verbose logging enabled, what EXACT font families and fallback chains appear in startup diagnostics for handling mixed Arabic (RTL) and English (LTR) text, and what runtime configuration values confirm these selections BEFORE any text rendering begins?"

**Question (verbatim):** *"With verbose logging enabled, what EXACT font families and fallback
chains appear in startup diagnostics for handling mixed Arabic (RTL) and English (LTR) text, and
what runtime configuration values confirm these selections BEFORE any text rendering begins?"*

### (a) Reproduced output

**1 — Resolved text faces (`dump_font_debug`), emitted at startup with `--debug-font-fallback`:**

```text
[t] Text fonts:
[t]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[t]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[t]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[t]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

(`[t]` is kitty's monotonic timestamp prefix, e.g. `[0.037]`.) There is **no** `Symbol map fonts:`
section because the default configuration defines no `symbol_map` (see rationale).

**2 — Per-glyph fallback chain (`output_cell_fallback_data`).** For the literal mixed sample
`Hello مرحبا World عربى 123`, **no fallback lines are produced at all** — on this host the
configured `monospace` (DejaVu Sans Mono) already covers Latin, the Arabic letters, *and* the
digits, so the primary face serves every cell. To exhibit a genuine **primary → fallback chain**,
rendering code points that DejaVu Sans Mono lacks (`A` + CJK `你` + emoji `😀`) yields:

```text
[t] U+4f60 Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[t] U+1f600 emoji_presentation Face(family=Noto Color Emoji style=Regular ps_name=NotoColorEmoji path=/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf ttc_index=0 variant=False named_instance=False scalable=False color=True)
```

`A` (`U+0041`) produces no line — it is in the primary face. A direct fallback query confirms the
Arabic case is served by the primary, while CJK/emoji map to distinct fallback faces:

```text
U+0645 (Arabic Meem) -> Face(family=DejaVu Sans Mono ... ps_name=DejaVuSansMono path=.../DejaVuSansMono.ttf ...)
U+4F60 (CJK)          -> Face(family=Noto Sans CJK JP ... ps_name=NotoSansCJKjp-Regular ...)
U+1F600 (emoji)       -> Face(family=Noto Color Emoji ... color=True)
```

### (b) Emitting source locators

| Output | Locator (function) | Notes |
|--------|--------------------|-------|
| `Text fonts:` / `Normal/Bold/Italic/Bold-Italic` / `Symbol map fonts:` | `kitty/fonts/render.py:161` `dump_font_debug` (`:163` header, `:164` face loop, `:167` `if ss:` guard, `:168-170` symbol loop) | prints each face's `identify_for_debug()` |
| String format `"<psname>: <path>:<index>"` | `kitty/freetype.c:738` `identify_for_debug` → `:742` `PyUnicode_FromFormat("%s: %V:%d", FT_Get_Postscript_Name(...), self->path, ..., instance.val)` | **FreeType/Linux** form |
| macOS variant `"<psname>: <path>"` | `kitty/core_text.m:966-967` `PyUnicode_FromFormat("%V: %V", postscript_name, path)` | **no** trailing `:index` |
| Per-glyph fallback line | `kitty/fonts.c:457` `output_cell_fallback_data` (`U+%x`, optional `bold`/`italic`/`emoji_presentation`, then `PyObject_Print(face, stderr, 0)`) | gated by the `debug` macro = `debug_fonts` (`kitty/fonts.c:18`) |
| Fallback selection | `kitty/fonts.c:481` `load_fallback_font`, debug emit at `:492` (`if (global_state.debug_font_fallback) …`) | |
| Startup invocation | `kitty/main.py:228-229` `if args.debug_font_fallback: dump_font_debug()` | between `boss.start` (`:227`) and `child_monitor.main_loop` (`:234`) |
| Flag plumbing | `kitty/cli.py:1002` (`--debug-font-fallback`) → `kitty/main.py:249` `set_options(...)` → `kitty/state.c:740` `global_state.debug_font_fallback = …` → `kitty/state.h:16` `debug_fonts` macro | |
| Config dump (resolved values) | `kitty/debug_config.py:231` `debug_config` (`:258` `OpenGL:`, `:260-263` `Fonts:` loop) — a Boss runtime action (`kitty/boss.py:3060`), **not** a CLI flag | |
| Option schema | `kitty/options/definition.py`: `font_family` `:35`, `force_ltr` `:64`, `symbol_map` `:84`, `narrow_symbols` `:99`, `disable_ligatures` `:115`, `font_features` `:136` | |

### (c) Rationale

**The EXACT families and paths come from the system, captured live.** The four `Text fonts:`
lines are produced by `dump_font_debug` (`kitty/fonts/render.py:161`), which calls each resolved
face's `identify_for_debug()`. On Linux that method
(`kitty/freetype.c:738-742`) formats the string as **`"<PostScript name>: <path>:<index>"`** —
exactly what we see: `DejaVuSansMono: /usr/share/fonts/.../DejaVuSansMono.ttf:0`. With the default
`font_family monospace`, FontConfig resolves the regular/bold/italic/bold-italic variants to the
DejaVu Sans Mono family shown. (On macOS the same line would read `DejaVuSansMono:
/path/to/font` with **no** trailing `:0`, per `kitty/core_text.m:966`.)

**There is no `Symbol map fonts:` block** because `dump_font_debug` only prints it `if ss:`
(`kitty/fonts/render.py:167`), and the default options define no `symbol_map`. This is honest
default behavior, not a missing capture.

**The fallback chain is per-glyph and lazy.** `output_cell_fallback_data` (`kitty/fonts.c:457`)
prints one line per code point that required a fallback face: `U+%x` for the base (and any
combining marks), an optional `emoji_presentation` flag, then the chosen `Face(...)`. The observed
`U+4f60 → Noto Sans CJK JP` and `U+1f600 emoji_presentation → Noto Color Emoji (color=True)` are
the real selections FontConfig returned for those code points. The crucial, code-true nuance for
the *mixed Arabic/English* scenario is that **the Arabic run did not trigger any fallback** —
DejaVu Sans Mono covers it — so the "fallback chain" for this particular sample is empty and the
primary family handles both the LTR and RTL runs. This is the faithful answer; a fabricated
"Arabic→SomeArabicFont" line would be wrong on this system.

**"BEFORE any text rendering begins" is a structural guarantee, not a timing coincidence.** The
font-selection dump is invoked at `kitty/main.py:228-229`, strictly *after* `boss.start(window_id,
startup_sessions)` (`:227`) and *before* `boss.child_monitor.main_loop()` (`:234`):

```python
# kitty/main.py
227  boss.start(window_id, startup_sessions)
228  if args.debug_font_fallback:
229      dump_font_debug()
...
234  boss.child_monitor.main_loop()
```

Font resolution happens during `set_font_family` inside `boss.start`; the diagnostic prints the
*already-resolved* faces; and only afterward does the render loop spin up. So the diagnostics
provably reflect the selections **before** any normal text rendering.

**Runtime configuration values that confirm the selections (reproduced).** With no `kitty.conf`,
the effective values are the schema defaults in `kitty/options/definition.py`. Driving kitty's own
interpreter (`kitty +runpy`) to print the resolved `Options` reproduces the required confirming
values exactly:

```text
##### Q2: RESOLVED DEFAULT CONFIG (kitty.options.types.defaults) #####
  font_family      = FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
  bold_font        = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
  italic_font      = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
  bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
  font_size        = 11.0
  font_features    = {}
  force_ltr        = False
  symbol_map       = {}
  narrow_symbols   = {}
  disable_ligatures = 0
```

These are precisely the values that confirm the selections *before* rendering: `font_family`
resolves to the system `monospace` alias (→ DejaVu Sans Mono, the family shown in the `Text fonts:`
block above); `bold_font`/`italic_font`/`bold_italic_font` are `auto` (derived from the family);
`force_ltr` is `False` (so per-run RTL applies, per Q1); `font_features`, `symbol_map`, and
`narrow_symbols` are all empty; and `disable_ligatures` is `0` (the `never` enum, so `calt` stays
on). Schema locators: `font_family` `:35`, `force_ltr` `:64`, `symbol_map` `:84`, `narrow_symbols`
`:99`, `disable_ligatures` `:115`, `font_features` `:136`.

The same resolved faces are independently confirmed by kitty's `debug_config` action
(`kitty/debug_config.py:231`), reproduced here against the default options (the version/uname/lsb
header lines it also prints are elided for brevity):

```text
Running under: X11
OpenGL:
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

The `Fonts:` block — built from the same `identify_for_debug()` strings as `dump_font_debug`
(`kitty/debug_config.py:260-263`) — shows `medium/bold/italic/bi` all resolved to the DejaVu Sans
Mono family, matching the `Text fonts:` capture earlier in this answer. (`OpenGL:` is blank in *this*
particular dump only because it was produced inside the font-test harness, which has no live GL
context; the live string `'4.5 (Core Profile) Mesa 25.2.8-…'` is the one captured in the
`--debug-gl` run in Q4. `Running under: X11` reflects the xvfb/X11 GLFW backend actually used — see
Methodology.) Note that `debug_config` is a **Boss runtime method** (`kitty/boss.py:3060`) reached
from inside a running instance, **not** a `--debug-config` CLI flag: no such flag exists in
`kitty/cli.py`, where the available debug flags are `--debug-rendering`/`--debug-gl` at `:989`,
`--debug-input`/`--debug-keyboard` at `:996`, and `--debug-font-fallback` at `:1002`. Because
`dump_font_debug()` is invoked at `kitty/main.py:228-229` *before* the render loop, the
authoritative "before rendering" evidence is that startup dump, and the resolved option values
above corroborate it.


---

## Q3 — Grid & shaping initialization metrics: "As the screen grid and shaping subsystems initialize, what default cell metrics, baseline positioning, and decoration alignment (overline/underline) values appear in debug output for complex grapheme clusters?"

**Question (verbatim):** *"As the screen grid and shaping subsystems initialize, what default cell
metrics, baseline positioning, and decoration alignment (overline/underline) values appear in
debug output for complex grapheme clusters?"*

### (a) Reproduced output

The exact metrics computed for the default `monospace` (DejaVu Sans Mono) at `font_size 11.0` and
96 DPI, captured at the point the values are handed to the decoration pre-render
(`prerender_function`, fed from `kitty/fonts.c:1458`):

```text
##### Q3: EXACT CELL METRICS (captured at prerender_function, fonts.c:1458) #####
  cell_width                = 9
  cell_height               = 18
  baseline                  = 14
  underline_position        = 15
  underline_thickness       = 1
  strikethrough_position    = 10
  strikethrough_thickness   = 1
  cursor_beam_thickness     = 1.5
  cursor_underline_thickness = 2.0
  dpi_x                     = 96.0
  dpi_y                     = 96.0
```

A "complex grapheme cluster" occupies the grid exactly like a simple character — one base cell
that also carries its combining marks. From the Q1 `test_shape` capture, the cluster
`e` + `U+0301` + `U+0304` shapes to a single cell holding two glyphs:

```text
  [combining 'e'+U+0301+U+0304] -> [(1, 2, 171, (171, 652))]
```

The same `cell_width`/`cell_height`/`baseline`/decoration metrics above apply uniformly to that
cluster cell.

| Metric | Value (px) | Locator | Rationale |
|--------|-----------:|---------|-----------|
| `cell_width` | 9 | `kitty/freetype.c:387` `cell_metrics`; stored in `kitty/fonts.c` `calc_cell_metrics` | advance width of the monospace face at this size |
| `cell_height` | 18 | `kitty/freetype.c:387` `cell_metrics` | line height (ascent+descent+line-gap) at this size |
| `baseline` | 14 | `kitty/freetype.c:391` `*baseline = font_units_to_pixels_y(self, self->ascender)` | baseline = ascender, in pixels |
| `underline_position` | 15 | `kitty/freetype.c:392` `MIN(cell_height-1, px(ascender - underline_position))` | just below the baseline, clamped to `cell_height-1` (=17) |
| `underline_thickness` | 1 | `kitty/freetype.c:393` `MAX(1, px(underline_thickness))` | floored at 1 px so it is never invisible |
| `strikethrough_position` | 10 | `kitty/freetype.c` `cell_metrics` (`:398` fallback `floor(baseline*0.65)`) | from the font's own strikeout metric here (the `0.65` fallback would give 9) |
| `strikethrough_thickness` | 1 | `kitty/freetype.c:403` (defaults to `underline_thickness` when unset) | |
| `cursor_beam_thickness` | 1.5 | option default (`kitty/options/definition.py`), pts → DPI-scaled | passed through `prerender_function` |
| `cursor_underline_thickness` | 2.0 | option default, pts → DPI-scaled | passed through `prerender_function` |

### (b) Emitting source locators

| Concern | Locator (function) |
|---------|--------------------|
| Font-unit → pixel conversion | `kitty/freetype.c:92` `font_units_to_pixels_y` = `(int)ceil(FT_MulFix(x, size->metrics.y_scale) / 64.0)` |
| Baseline / underline / strikethrough derivation | `kitty/freetype.c:387-403` `cell_metrics` (baseline `:391`, underline pos `:392`, underline thk `:393`, strikethrough fallback `:398`, strikethrough thk `:403`) |
| Metric finalization + clamps + `modify_font` | `kitty/fonts.c:373-419` `calc_cell_metrics` (`fatal()` bounds via `MIN_WIDTH 2`/`MIN_HEIGHT 4`/`MAX_DIM 1000`; `sprite_tracker_set_layout` at `:418`; stores `fg->cell_width/cell_height/baseline/underline_*/strikethrough_*`) |
| Metrics → decoration pre-render | `kitty/fonts.c:1458` `PyObject_CallFunction(prerender_function, "IIIIIIIffdd", cell_width, cell_height, baseline, underline_position, underline_thickness, strikethrough_position, strikethrough_thickness, cursor_beam_thickness, cursor_underline_thickness, dpi_x, dpi_y)` |
| Decoration sprite renderers | `kitty/fonts/render.py`: `render_special` `:284` (underline-position clamp `:298`, style dispatch `:317`, thickness clamp `:316`), `add_line` `:203`, `add_dline` `:211`, `add_curl` `:231`, `add_dots` `:267`, `add_dashes` `:276`, `prerender_function` `:364` (underline sprites `range(1, NUM_UNDERLINE_STYLES+1)` `:391`) |
| Grapheme-cluster cell storage | `kitty/line.c` combining marks in `cc_idx[]` emitted via `codepoint_for_mark` (`:46`, `:204`, `:214`); cell width via `wcwidth_std` (`kitty/line.c:12`, `:348`) |
| Combining/width handling | `kitty/screen.c:663` `draw_combining_char`; `wcwidth-std.h` include `:27` |

### (c) Rationale

**`baseline = ascender`, expressed in pixels.** `cell_metrics` sets `*baseline =
font_units_to_pixels_y(self, self->ascender)` (`kitty/freetype.c:391`), and
`font_units_to_pixels_y` (`:92`) scales font design units by the face's `y_scale` and converts the
26.6 fixed-point result to integer pixels (`/64.0`, `ceil`). For DejaVu Sans Mono at this size the
ascender works out to **14 px**, so glyphs sit on a baseline 14 px down from the cell top, leaving
4 px of descender space within the 18 px cell.

**Underline sits just under the baseline and is clamped.** `underline_position` is
`MIN(cell_height-1, px(ascender - underline_position))` (`kitty/freetype.c:392`) — it follows the
font's underline metric but can never reach the very last row (`cell_height-1 = 17`), giving **15**
here. `underline_thickness` is `MAX(1, …)` (`:393`), so it can never collapse to 0 — hence **1 px**.

**Strikethrough uses the font's own metric when present.** The observed
`strikethrough_position = 10` is *not* the `floor(baseline * 0.65) = floor(9.1) = 9` fallback at
`kitty/freetype.c:398`; DejaVu Sans Mono supplies a real strikeout position, so kitty uses it.
`strikethrough_thickness` defaults to `underline_thickness` (= 1) when the font leaves it unset
(`:403`). This is a useful demonstration that the fallback path exists but is not always taken.

**`modify_font` would shift these consistently.** `calc_cell_metrics` (`kitty/fonts.c:373-419`)
applies any `modify_font` adjustments and, when the baseline moves, re-derives the underline and
strikethrough positions by the same amount — matching kitty's documentation that *modifying the
baseline automatically adjusts underline & strikethrough positions*. No `modify_font` was set
here, so the raw font-derived values stand.

**On "overline" — an honest mapping.** The question pairs "overline/underline". kitty's
pre-rendered decoration sprites are the **underline styles** (straight, double, curly, dotted,
dashed — `render_special` dispatch at `kitty/fonts/render.py:317`), the **strikethrough**, the
**missing-glyph** box, and three **cursor** glyphs (`prerender_function`, `:364-391`). There is
**no separate "overline" sprite** in this pre-render set, and — verified by a repository-wide
search at this revision — **no source-supported overline rendering path exists at all**: the only
`overline` token anywhere in the tree is an unrelated X11 keysym→Unicode mapping
(`glfw/xkb-compat-shim.h:135`), and the cell shaders sample only underline and strikethrough
sprites (`underline_pos`/`strike_pos`, `kitty/cell_vertex.glsl:183-185`; the
`// ... decorations (cursor, underline, strikethrough)` blend at `kitty/cell_fragment.glsl:129-136`).
So in startup *debug output* the overline simply has no distinct metric; the decoration metrics
that do appear are the underline/strikethrough position and thickness above.

**Grapheme clusters do not change the grid math.** Combining marks are stored on the base cell in
`cc_idx[]` (`kitty/line.c`) and re-emitted into the shaping run via `codepoint_for_mark`; a wide
glyph occupies a base cell plus a continuation cell (`wcwidth_std`, `kitty/line.c:348`). Either
way the per-cell metrics above apply unchanged — which is why the combining cluster `(1, 2, 171,
(171, 652))` is laid out with the very same `cell_width=9`, `cell_height=18`, `baseline=14`.


---

## Q4 — GPU texture atlas initialization: "For GPU texture atlas initialization at launch, what initial page layout, sizing, and capacity allocations get reported for shaped glyphs across primary and fallback fonts, and what startup logs verify the atlas is ready to receive shaped glyph data?"

**Question (verbatim):** *"For GPU texture atlas initialization at launch, what initial page
layout, sizing, and capacity allocations get reported for shaped glyphs across primary and fallback
fonts, and what startup logs verify the atlas is ready to receive shaped glyph data?"*

### (a) Reproduced output

**1 — GL / atlas readiness log (`--debug-rendering --debug-gl`).** This is the startup line that
verifies the OpenGL context — and therefore the sprite-atlas/shader machinery — is initialized:

```text
[0.133] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[0.158] OS Window created
[0.171] Child launched
```

No `glCopyImageSubData` slow-path warning was emitted, i.e. the `ARB_copy_image` fast path is
available on this Mesa/llvmpipe stack.

**2 — Device limits that size the atlas.** On first sprite-map allocation kitty queries the live
GL maxima (here via `glxinfo` under the same headless server):

```text
GL_MAX_TEXTURE_SIZE         = 16384
GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
```

**3 — Page layout & sprite-position allocation.** With the real device texture size (16384) and
the Q3 cell size (9×18), the computed page geometry and the first allocated sprite slots are:

```text
##### Q4: ATLAS PAGE LAYOUT (device limits 16384/2048, cell 9x18) #####
  set_layout(cell_w=9, cell_h=18) => xnum = 16384//9 = 1820 ; max_y = 16384//18 = 910 ; origin (x,y,z)=(0,0,0)
  sprite positions (x,y,z) assigned to successive distinct glyph keys:
    glyph_key=1 -> (0, 0, 0)
    glyph_key=2 -> (1, 0, 0)
    glyph_key=3 -> (2, 0, 0)
    glyph_key=4 -> (3, 0, 0)
    glyph_key=5 -> (4, 0, 0)
    glyph_key=6 -> (5, 0, 0)
    glyph_key=7 -> (6, 0, 0)
    glyph_key=8 -> (7, 0, 0)
```

To make the wrap order explicit, the same allocator over a deliberately tiny texture
(`max_texture_size=36` → `xnum=4`, `max_y=2`) walks **x → y → z** (column → row → page/layer):

```text
##### Q4 [illustrative x->y->z] max_texture_size=36, cell 9x18 => xnum=4, max_y=2 #####
    (0,0,0) (1,0,0) (2,0,0) (3,0,0)   <- row y=0 fills across x
    (0,1,0) (1,1,0) (2,1,0) (3,1,0)   <- row y=1
    (0,0,1) (1,0,1) (2,0,1) (3,0,1)   <- page/layer z=1
```

### (b) Emitting source locators

| Concern | Locator (function) |
|---------|--------------------|
| Pre-GL-init defaults | `kitty/fonts.c:44` `static size_t max_texture_size = 1024, max_array_len = 1024;` |
| GL-limit query (first alloc) | `kitty/shaders.c:51` `alloc_sprite_map` → `:53` `glGetIntegerv(GL_MAX_TEXTURE_SIZE, …)`, `:54` `glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, …)`; Apple caps 8192/512 at `:58-59`; `sprite_tracker_set_limits(…)` at `:61` |
| shaders.c own statics | `kitty/shaders.c:32` `static GLint max_texture_size = 0, max_array_texture_layers = 0;` (distinct from the `fonts.c:44` statics; pushed across via `sprite_tracker_set_limits`) |
| Limit clamp | `kitty/fonts.c:237` `sprite_tracker_set_limits` → `:239` `max_array_len = MIN(0xfffu, max_array_len_)` (cap = 4095) |
| Page geometry | `kitty/fonts.c:276` `sprite_tracker_set_layout` → `:277` `xnum = MIN(MAX(1u, max_texture_size/cell_width), UINT16_MAX)`, `:278` `max_y = … / cell_height`, `:279` `ynum = 1`, `:280` `x = y = z = 0` |
| Position assignment | `kitty/fonts.c:257` `sprite_position_for` (assigns current `(x,y,z)`, then `do_increment`); `:268-269` `sprite_tracker_current_layout` returns `(xnum, ynum, z)` |
| Capacity advance / exhaustion | `kitty/fonts.c:242-253` `do_increment` (`x++`; row-full → `y++`, grow `ynum` up to `max_y`; col-full → `z++`; `*error = 2` when `z >= MIN(UINT16_MAX, max_array_len)`) |
| Glyph→sprite cache | `kitty/glyph-cache.h:13-19` `SpritePosition { bool rendered, colored; sprite_index x, y, z; }`; `:23-24` `find_or_create_sprite_position`; impl `kitty/glyph-cache.c:34` |
| Readiness log | `kitty/gl.c:72` `if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", …, gl_version_string())`; helper `gl_version_string` `:42-48` builds `"'<GL_VERSION>' Detected version: <maj>.<min>"` |
| Slow-path warning (not hit) | `kitty/shaders.c:90` `"…does not have glCopyImageSubData, falling back to a slower implementation"` (when `!GLAD_GL_ARB_copy_image`, `:86`) |

### (c) Rationale

**Sizing: defaults are placeholders replaced by device maxima at GL init.** Before a GL context
exists, `kitty/fonts.c:44` seeds `max_texture_size = max_array_len = 1024`. On the first sprite-map
allocation, `alloc_sprite_map` (`kitty/shaders.c:51`) queries the real `GL_MAX_TEXTURE_SIZE`
(**16384**) and `GL_MAX_ARRAY_TEXTURE_LAYERS` (**2048**) and pushes them through
`sprite_tracker_set_limits` (`kitty/fonts.c:237`). The layer count is clamped by
`max_array_len = MIN(0xfffu, …)` (`:239`), i.e. capped at **4095**; since the device reports 2048,
the effective `max_array_len` here is **2048**.

**Page layout: a page is a `xnum × max_y` grid of cell-sized slots, starting at the origin.**
`sprite_tracker_set_layout` (`kitty/fonts.c:276-280`) computes `xnum = max_texture_size /
cell_width = 16384 / 9 = 1820` and `max_y = max_texture_size / cell_height = 16384 / 18 = 910`,
sets `ynum = 1`, and resets the cursor to `(x, y, z) = (0, 0, 0)`. So at launch the atlas is one
texture-array layer presenting up to 1820 columns × 910 rows of 9×18 sprite slots, with the next
free slot at the origin — exactly the `origin (0,0,0)` and `glyph_key=1 -> (0,0,0)` observed.

**Capacity: allocation advances x → y → z, and signals exhaustion at the layer cap.** Each newly
shaped glyph key (whether from the primary or a fallback font) is assigned the current slot by
`sprite_position_for` (`kitty/fonts.c:257`) and then `do_increment` (`:242-253`) advances the
cursor: `x++` across a row; when `x` reaches `xnum` it wraps to `y++` (growing `ynum` toward
`max_y`); when `y` reaches `max_y` it wraps to `z++` — a new texture-array **layer / "page"** —
and sets `*error = 2` once `z >= MIN(UINT16_MAX, max_array_len)`. The tiny-texture capture above
reproduces this precisely: with `xnum=4, max_y=2`, slot assignment fills row 0, then row 1, then
jumps to layer `z=1`.

**Primary and fallback glyphs share one atlas — but the cache table is per-font.** The sharing is
at the *allocator* level, not the *lookup* level, and the distinction matters. Each `Font` owns its
**own** sprite-position cache — a per-`Font` member `SpritePosition *sprite_position_hash_table`
(`kitty/fonts.c:61`) — keyed by the **glyph-id(s) / ligature-index / cell-count** tuple
(`find_or_create_sprite_position(&font->sprite_position_hash_table, glyphs, glyph_count,
ligature_index, cell_count, …)`, `kitty/fonts.c:259`; key fields in `SpritePositionHead`,
`kitty/glyph-cache.h:13-24`). What *is* shared is the enclosing `FontGroup`'s **single** sprite
tracker (`GPUSpriteTracker sprite_tracker`, `kitty/fonts.c:88`): when a glyph key is first inserted
into a given font's table (`created == true`), `sprite_position_for` (`kitty/fonts.c:256-266`)
stamps the new entry with the tracker's *current* `(x, y, z)` and then calls `do_increment`
(`kitty/fonts.c:242-253`) to advance that one shared cursor. So glyphs from the configured family
and from fallback faces (e.g. the `Noto Sans CJK JP` and `Noto Color Emoji` faces seen in Q2) each
keep their own per-font lookup table, yet they all draw their slots from the **same** `(x, y, z)`
sequence in the **same** atlas, in the order they are first shaped.

**Readiness: the GL version line is the "atlas is ready" signal.** `kitty/gl.c:72` prints the
GL version string under `--debug-rendering`/`--debug-gl` once the context is created and the
loader has detected the version; `gl_version_string` (`:42-48`) formats it as `'<GL_VERSION>'
Detected version: <maj>.<min>`. The captured `'4.5 (Core Profile) Mesa 25.2.8-…' Detected version:
4.5` confirms a live OpenGL 4.5 core context — the prerequisite for the sprite textures and shaders
that receive shaped glyph data. Under llvmpipe the version string names a **software** renderer;
that is the truthful readiness log for this headless run, and the absence of the
`kitty/shaders.c:90` warning confirms the fast `glCopyImageSubData` atlas-copy path is in use.


---

## Fidelity & Caveats

- **Run-level RTL, not full BIDI.** kitty does not implement the Unicode Bidirectional Algorithm.
  RTL scripts are shaped right-to-left per run (`hb_buffer_guess_segment_properties`,
  `kitty/fonts.c:687`), but logical reordering across a mixed line is not performed and selection
  maps to logical (LTR) order. `force_ltr` (`kitty/options/definition.py:64`, default `no`) forces
  LTR throughout and is intended for use with external **GNU FriBidi**. The Q1/Q2 answers describe
  this actual capability rather than the broader "bidi" implied by the question wording.

- **The mixed Arabic/English sample produced no Arabic fallback** on this host because the
  resolved `monospace` (DejaVu Sans Mono) covers Latin, Arabic, and digits. The genuine
  primary→fallback chain in Q2 was therefore demonstrated with code points DejaVu Sans Mono lacks
  (CJK, emoji). The exact families and paths are host-dependent — they are captured live, never
  assumed.

- **No dedicated "overline" decoration sprite** exists in the startup pre-render set
  (`prerender_function`, `kitty/fonts/render.py:364`). The pre-rendered decorations are the
  underline styles, strikethrough, missing-glyph box, and cursor glyphs; a repository-wide search
  at this revision found **no source-supported overline rendering path** (the cell shaders sample
  only underline and strikethrough sprites — `kitty/cell_vertex.glsl:183-185`,
  `kitty/cell_fragment.glsl:129-136`), so the overline has no distinct cell-metric in debug output
  (Q3).

- **Headless / software-renderer caveats.** Runs used `xvfb` + Mesa **llvmpipe**, so the GL
  readiness string (Q4) names a software renderer; the device texture maxima (16384 / 2048) and
  the resulting page geometry are real for this stack and will differ on other GPUs. Because a
  headless window is treated as not visible, the shaping/metric/atlas internals were exercised
  through kitty's own public test API via `kitty +runpy` (no source modified, nothing persisted);
  the emitted values are identical to those the draw path would produce.

- **Observation-harness fragility (not a kitty bug).** Repeated `get_fallback_font` calls driven
  outside kitty's normal application lifecycle could crash the interpreter *during teardown*, after
  all diagnostics had already been emitted; the captured data is unaffected, and capture scripts
  used `os._exit(0)` to obtain clean exit codes. This is an artifact of driving internals directly,
  not a defect in kitty's runtime.

- **Platform variance.** Linux uses FontConfig (`kitty/fontconfig.c`) + FreeType
  (`kitty/freetype.c`); macOS uses CoreText (`kitty/core_text.m`). The only answer this changes is
  the `identify_for_debug` string format: FreeType emits `"<psname>: <path>:<index>"`
  (`kitty/freetype.c:742`) whereas CoreText emits `"<psname>: <path>"` with no trailing index
  (`kitty/core_text.m:966`).

- **Restoration.** All instrumentation was via runtime CLI flags and ephemeral, out-of-tree
  `+runpy` scripts and text samples. No `kitty.conf` was written and no source default was changed;
  compiled artifacts are git-ignored. `git status --porcelain` is empty (clean working tree) and
  `git diff <baseline>..HEAD --name-status` reports only
  `A  blitzy/documentation/kitty_815df1e210e0.md`.

---

### Source locator index

For convenience, the principal locators cited above (verified against HEAD
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`):

| File | Lines | Role |
|------|-------|------|
| `kitty/fonts.c` | 18, 42, 44, 45, 61, 88, 237-280, 294-325, 457-468, 481-492, 672-688, 786-813, 373-419, 1458, 1755-1757 | shaping features, fallback, metrics finalization, atlas tracker, per-font glyph cache + shared sprite tracker |
| `kitty/fonts/render.py` | 161-170, 173-191, 203-317, 364-401 | Python orchestration, debug dump, decoration sprites |
| `kitty/freetype.c` | 92, 387-403, 738-742 | unit→pixel, cell metrics, debug font string (Linux) |
| `kitty/core_text.m` | 96, 966-967 | CoreText font string (macOS) |
| `kitty/fontconfig.c` | 276, 312, 362-364, 444, 463 | Linux font matching / fallback |
| `kitty/shaders.c` | 32, 51-69, 86-90 | GL-limit query, atlas allocation, copy-path warning |
| `kitty/cell_vertex.glsl` / `cell_fragment.glsl` | vtx:183-185; frag:129-136 | decoration sprite sampling — underline + strikethrough only (no overline path) |
| `kitty/gl.c` | 42-48, 72 | GL version readiness log |
| `kitty/glyph-cache.h` / `.c` | h:13-24, c:34-53 | glyph→sprite-position cache |
| `kitty/main.py` | 227-234, 249 | startup ordering, flag wiring |
| `kitty/cli.py` | 989, 996, 1002 | `--debug-*` flag definitions |
| `kitty/state.c` / `state.h` | c:726-740, h:13-16 | debug-flag plumbing, print macros |
| `kitty/debug_config.py` | 231, 258-263 | resolved config dump (Boss action) |
| `kitty/options/definition.py` | 35, 64, 84, 99, 115, 136 | option schema defaults |
| `kitty/line.c` / `kitty/screen.c` | line:12,46,204,214,348; screen:27,663 | grapheme-cluster cell storage / width |

