# kitty startup: text shaping, cell metrics & GPU texture atlas — an evidence-based walkthrough

**Revision:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`) · **kitty 0.35.2**
**Platform exercised:** Linux / FreeType / FontConfig / Mesa (software GL). macOS/CoreText referenced for contrast only.

This document answers, for an engineer onboarding to kitty, how the terminal's text‑shaping/layout engine, screen‑grid cell metrics, and GPU texture atlas configure themselves **at startup, before any glyph is rendered**. It is built **run‑first**: kitty was compiled, launched headless with verbose logging, and fed mixed Arabic (RTL) + English (LTR) + combining‑diacritic + CJK + emoji text. Every behavioral claim below is paired with **(a) the actual observed runtime output** and **(b) a `file:line` citation** anchored to the revision above. Conclusions drawn from reading code but not directly logged at runtime are explicitly labeled **`(inferred)`**. The investigation is strictly read‑only: no source file was modified, diagnostics were enabled only through transient CLI flags, and the source tree's `git status` is clean on completion (see §2).

The four questions answered, in order:

1. **Q1 — Startup shaping & fallback configuration** (ligatures, bidi, combining diacritics, font fallback). → §3
2. **Q2 — Verbose startup diagnostics** for mixed Arabic (RTL) + English (LTR): exact font families, fallback chains, and the runtime config that confirms selections before first render. → §4
3. **Q3 — Cell metrics, baseline & decoration** (overline/underline/strikethrough) for complex grapheme clusters. → §5
4. **Q4 — GPU texture atlas initialization**: page layout, sizing, capacity, and readiness. → §6

A runtime configuration snapshot (§7), a Linux‑vs‑macOS platform note (§8), and a final coverage checklist (§9) follow.

---

## 2. Methodology & environment

### 2.1 Legend

- **Observed** — a value or line reproduced **verbatim** from captured runtime output, shown in a fenced block immediately with the command that produced it.
- **`(inferred)`** — a statement derived from reading the source at this revision, not directly emitted as a runtime log. Anything not directly loggable (e.g. the silent atlas allocation) is labeled this way.

### 2.2 Environment

- **Container:** `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0…` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), providing the C compiler, Go, pkg‑config, and the HarfBuzz/FreeType/FontConfig/OpenGL libraries.
- **Binary under test:** `kitty/launcher/kitty`, produced by the canonical CI build command `python3 setup.py build --verbose` (which compiles the C core `fonts.c`/`freetype.c`/`shaders.c`/`gl.c` and produces `kitty/launcher/kitty`, `kitty/launcher/kitten`, and `kitty/fast_data_types.so`). The binary was verified functional:

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

- **Linked shaping/rasterization libraries** (from `ldd kitty/fast_data_types.so` / `pkg-config`): **HarfBuzz 10.2.0** (`libharfbuzz.so.0.61020.0`), **FreeType** (`freetype2` 26.2.20), **FontConfig 2.15.0**. This resolves the manifest discrepancy noted in the plan — `setup.py` enforces only `harfbuzz >= 1.5` while `docs/build.rst` lists `>= 2.2.0`; the binary actually links HarfBuzz **10.2.0**, satisfying both.

### 2.3 Headless OpenGL context (and why the renderer matters)

kitty requires an OpenGL **3.3+** context; the minimum is enforced at startup in `gl_init` [kitty/gl.c:L74]. The investigation host has no display, so a dedicated X virtual framebuffer was started and an offscreen GL context obtained through it:

```bash
# dedicated Xvfb on display :101 (own display number, software GL, no device contention)
nohup Xvfb :101 -screen 0 1920x1080x24 +extension GLX +render -noreset > /tmp/blitzy_xvfb_101.log 2>&1 &
export DISPLAY=:101
```

The renderer is **Mesa llvmpipe** — a **software** rasterizer. This is disclosed because a software renderer can report **different** `GL_MAX_TEXTURE_SIZE` / `GL_MAX_ARRAY_TEXTURE_LAYERS` than a hardware GPU, which directly changes the atlas sizing reported in §6. The renderer string was captured both from an independent EGL probe and from kitty's own GL startup log (§6):

```text
$ /tmp/blitzy_gl_probe            # EGL pbuffer + OpenGL core context, calls glGetIntegerv exactly as shaders.c does
PROBE GL_VENDOR    = Mesa
PROBE GL_RENDERER  = llvmpipe (LLVM 20.1.8, 256 bits)
PROBE GL_VERSION   = 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2
PROBE GL_MAX_TEXTURE_SIZE         = 16384
PROBE GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
```

### 2.4 Transient verbose diagnostics (CLI only — no config edits)

Three debug flags were enabled **only on the command line**; no config file was touched:

- `--debug-font-fallback` [kitty/cli.py:L1002] sets `args.debug_font_fallback`, which at startup triggers `dump_font_debug()` (the `Text fonts:` banner) [kitty/main.py:L228-L229] and enables per‑codepoint fallback logging gated on `global_state.debug_font_fallback` [kitty/fonts.c:L492].
- `--debug-rendering` / `--debug-gl` [kitty/cli.py:L989] set `global_state.debug_rendering`, which enables the `GL version string:` log in `gl_init` [kitty/gl.c:L72] and turns on GL error checking.

`--config NONE` was used so reported families, metrics, and atlas sizes reflect **canonical defaults** rather than any host `kitty.conf`.

### 2.5 Exact run invocations

**Canonical run** (default `monospace` family; mixed Arabic + English + combining + CJK + emoji). stdout and stderr were captured to separate files because — as verified in §4 and §6 — the GL line goes to **stdout** while the font diagnostics go to **stderr**:

```bash
export DISPLAY=:101
# mixed input written as real UTF-8 (avoids the dash `printf` \xHH pitfall):
python3 - <<'PY'
lines=[
 "\u0645\u0631\u062d\u0628\u0627 \u0628\u0627\u0644\u0639\u0627\u0644\u0645 Hello World",  # Arabic RTL + English LTR
 "e\u0301 a\u0300 \u0645\u064e\u0631 combining-diacritics",                                 # base + combining marks
 "CJK \u4e2d\u6587 \u65e5\u672c\u8a9e",                                                     # CJK
 "emoji \U0001F600 \U0001F642",                                                              # emoji
]
open("/tmp/blitzy_evidence/mixed_canonical.txt","w",encoding="utf-8").write("\n".join(lines)+"\n")
PY
setsid ./kitty/launcher/kitty --config NONE -o sync_to_monitor=no \
  --debug-font-fallback --debug-rendering --debug-gl \
  sh -c 'sleep 0.5; cat /tmp/blitzy_evidence/mixed_canonical.txt; sleep 30' \
  > /tmp/blitzy_evidence/canonical_stdout.txt 2> /tmp/blitzy_evidence/canonical_stderr.txt &
```

**Arabic‑fallback run** (forces the fallback tier for Arabic by choosing a primary family that lacks Arabic — see the FreeType/FontConfig note in §2.6):

```bash
setsid ./kitty/launcher/kitty --config NONE -o font_family="Liberation Mono" -o sync_to_monitor=no \
  --debug-font-fallback --debug-rendering \
  sh -c 'sleep 0.5; cat /tmp/blitzy_evidence/mixed_canonical.txt; sleep 30' \
  > /tmp/blitzy_evidence/libmono_stdout.txt 2> /tmp/blitzy_evidence/libmono_stderr.txt &
```

To force a frame (X11 non‑Wayland does not use render‑frames, so a redraw must be provoked), the window was resized once via a tiny Xlib helper after launch; this reliably triggered `render_line → shape_run → load_fallback_font` and thus the fallback logging.

### 2.6 Why Arabic needs a non‑default primary to demonstrate fallback

An important, initially surprising finding: under the **default** `monospace` family (which resolves to **DejaVu Sans Mono** here), Arabic text does **not** trigger fallback. A direct FreeType `FT_Get_Char_Index` probe showed DejaVu Sans Mono actually **contains** the Arabic glyphs used (e.g. meem `U+0645`→gid 1142, reh `U+0631`→1127) as well as the combining marks (`U+0301`→649, `U+0300`→648, `U+064E`→1151). kitty's coverage test `has_cell_text` uses FreeType directly, so `font_for_cell` keeps Arabic on the **primary** face. (FontConfig's cached charset for the same file omits these codepoints — a FreeType‑vs‑FontConfig divergence — but FreeType is authoritative for kitty's decision.) To exercise the **fallback** tier for Arabic (Rule 2 requires both tiers), the primary was set to **Liberation Mono**, which genuinely lacks Arabic; the canonical `monospace` run still exercises the fallback tier via CJK + emoji.

### 2.7 Two‑run stability

The full capture was run **at least twice**. The banner faces, the per‑codepoint fallback selections, and the cell metrics were **identical** across runs (a `diff` of the two fallback captures showed only the leading `[<sec>.mmm]` timestamps differing). Reported values below are therefore single stable values; only the timestamp prefix varies run‑to‑run.

### 2.8 Read‑only & restore confirmation

All diagnostics were transient CLI flags. kitty's own `AppRunner.__call__` restores font state in its `finally:` block via `set_options(None)` [kitty/main.py:L254] and `free_font_data()` [kitty/main.py:L255], so no persistent state is altered by the observation. Temporary probes and captures live entirely under `/tmp` (outside the repository tree). After the runs, the source checkout is clean:

```text
$ git status --porcelain
$ echo "EXIT=$?"
EXIT=0        # empty output above = clean working tree; no tracked source file changed
```

The only persistent artifact of this task is this document, `blitzy/documentation/kitty_815df1e210e0.md`, which lives outside the tracked kitty source tree.

---

## 3. Q1 — Startup shaping & fallback configuration

kitty's shaping/layout engine is HarfBuzz. Its Unicode capabilities are configured **once, at startup**, inside `init_fonts` [kitty/fonts.c:L1746], which the embedded interpreter runs before the window/render loop begins. The four capabilities the question names — ligatures, bidi, combining diacritics, and font fallback — are each turned on as follows.

### 3.1 The shaping buffer and cluster level (grapheme‑cluster preservation)

`init_fonts` creates the single reusable shaping buffer and sets its cluster level to **monotone characters**:

```c
// kitty/fonts.c:1746-1749
init_fonts(PyObject *module) {
    harfbuzz_buffer = hb_buffer_create();
    if (harfbuzz_buffer == NULL || !hb_buffer_allocation_successful(harfbuzz_buffer) || !hb_buffer_pre_allocate(harfbuzz_buffer, 2048)) { PyErr_NoMemory(); return false; }
    hb_buffer_set_cluster_level(harfbuzz_buffer, HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS);
```

**Cause → effect.** HarfBuzz tracks **clusters**, not graphemes. `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS` [kitty/fonts.c:L1749] assigns each input character a cluster value and keeps those values monotonic through shaping, so a base character plus its combining marks (and the components of a ligature) stay grouped in one cluster that maps to one terminal cell. This is the mechanism that lets a "complex grapheme cluster" (§5) occupy a single fixed cell box. **`(inferred)`** — there is no dedicated runtime log for the buffer/cluster‑level setup; it is confirmed indirectly by the observed correct rendering and clustering of combining sequences (the `U+645 U+64e` combined fallback line in §4.4, and the base+combining bytes preserved in the screen buffer in §4.5).

### 3.2 OpenType feature toggles for ligatures

Three ligature‑related OpenType features are registered at startup:

```c
// kitty/fonts.c:1755-1757
    create_feature("-liga", LIGA_FEATURE);
    create_feature("-dlig", DLIG_FEATURE);
    create_feature("-calt", CALT_FEATURE);
```

**Cause → effect.** The leading `-` makes each a **disable** toggle: these pre‑built feature records are applied to a run only when ligatures must be **suppressed**. When ligatures are enabled (the default), the run is shaped with the font's default feature set and these toggles are not applied; when disabled, the appropriate subset is appended — and the last feature applied is always `-calt` [kitty/fonts.c:L812] so contextual alternates are turned off together with `liga`/`dlig`. This wiring is governed by two options whose canonical defaults were captured in §7: `disable_ligatures` (default `never` [kitty/options/definition.py:L115]) and `font_features` (default `none` [kitty/options/definition.py:L135-L136]). With the observed defaults, no feature toggle is applied, so ligatures shape normally. **`(inferred from code + confirmed by observed defaults)`** — the feature *registration* is not logged; the effective option values that decide whether the toggles fire are the observed evidence (§7).

### 3.3 Bidi / direction detection (RTL Arabic vs LTR English)

Direction is resolved per run automatically, with a single global override:

```c
// kitty/fonts.c:687-688
    hb_buffer_guess_segment_properties(harfbuzz_buffer);
    if (OPT(force_ltr)) hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR);
```

**Cause → effect.** `hb_buffer_guess_segment_properties` [kitty/fonts.c:L687] inspects the run's codepoints and sets script, language, and **direction** — for an Arabic run it selects the Arabic script (whose shaper performs cursive joining) and RTL direction; for a Latin run it selects LTR. The `force_ltr` option [kitty/fonts.c:L688] can pin every run to LTR, but its canonical default is `no` [kitty/options/definition.py:L64] (observed effective value `force_ltr = False`, §7), so Arabic runs are detected RTL automatically. **Observed confirmation:** the Arabic run rendered and shaped correctly and its codepoints were logged during fallback selection (§4.3–§4.4), and the screen‑buffer dump (§4.5) shows the Arabic bytes present; RTL handling within a run uses decreasing cluster numbers in the shaping bookkeeping around [kitty/fonts.c:L997] and [kitty/fonts.c:L1080] **`(inferred)`**.

### 3.4 Combining‑mark composition

When a base character is followed by combining marks, kitty attempts a single composed codepoint before falling back to separate marks:

```c
// kitty/fonts.c:447
        if (hb_unicode_compose(hb_unicode_funcs_get_default(), cell->ch, combining_chars[0], &ch) && face_has_codepoint(face, ch)) return true;
```

**Cause → effect.** `hb_unicode_compose` [kitty/fonts.c:L447] asks HarfBuzz's default Unicode functions to compose base + first combining mark into one codepoint; if the face has a glyph for the composed form, that single glyph is used. Otherwise the base and marks remain in one cluster (per §3.1) and are positioned by the shaper. **Observed confirmation:** the combining sequences survived into the cell buffer as base+mark pairs — the get‑text dump in §4.5 shows `65 cc 81` (`e` + `U+0301`) and `d9 85 d9 8e` (meem + `U+064E`) — and the fallback selector even logged a base+mark pair on one line (`U+645 U+64e`, §4.4), demonstrating the marks stay clustered with their base.

### 3.5 Font fallback engine (Linux / FontConfig)

When the primary face lacks a glyph for a cell, kitty asks the platform library (FontConfig on Linux) for a face that has it:

- `fallback_font` [kitty/fontconfig.c:L444] is the entry point that builds a query for the missing codepoint(s).
- `create_fallback_face` [kitty/fontconfig.c:L463] builds an `FcPattern` whose family is `"monospace"` — or `"emoji"` when emoji presentation is requested [kitty/fontconfig.c:L468] — adds the cell's codepoint(s) as an `FcCharSet`, calls `FcFontMatch`, and wraps the result as a kitty face.
- The number of distinct fallback faces is capped: exceeding 100 aborts the lookup with an error (a boundary, **not** exercised destructively):

```c
// kitty/fonts.c:482
    if (fg->fallback_fonts_count > 100) { log_error("Too many fallback fonts"); return MISSING_FONT; }
```

**Observed confirmation.** The fallback engine's selections are logged per codepoint under `--debug-font-fallback`; §4.3 (CJK → *Noto Sans CJK JP*, emoji → *Noto Color Emoji*) and §4.4 (Arabic → *DejaVu Sans Mono* when the primary is Liberation Mono) are the concrete, observed outputs of `create_fallback_face`. The `"monospace"`/`"emoji"` family selection is visible in the fact that the emoji codepoints resolved to a dedicated color‑emoji face while text codepoints resolved to monospace faces.


---

## 4. Q2 — Verbose startup diagnostics for mixed Arabic (RTL) + English (LTR)

### 4.1 What emits the diagnostics, and on which stream

- The **base‑face banner** is emitted by `dump_font_debug()` [kitty/fonts/render.py:L161], gated at startup by `if args.debug_font_fallback:` [kitty/main.py:L228] → `dump_font_debug()` [kitty/main.py:L229]. It prints via Python `log_error`, which on Linux routes through the C logger `log_error` in `logging.c` — that prepends a `[<seconds>.mmm] ` timestamp [kitty/logging.c:L55-L56] and writes to **stderr** [kitty/logging.c:L61].
- The **per‑codepoint fallback lines** are emitted by `output_cell_fallback_data` [kitty/fonts.c:L457], reached from `load_fallback_font` only when `global_state.debug_font_fallback` is set [kitty/fonts.c:L492]. These use the `debug(...)` macro, which is `#define debug debug_fonts` [kitty/fonts.c:L18] → `debug_fonts` [kitty/state.h:L16] → `timed_debug_print` [kitty/monotonic.h:L99], also prepending `[<seconds>.mmm] ` and writing to **stderr**.
- The **GL version line** (§6) is the one diagnostic on **stdout** (a plain `printf` [kitty/gl.c:L72]).

Both the banner and the fallback lines are therefore timestamped on Linux, via two different code paths. All output below is reproduced **verbatim**, including the timestamp prefixes exactly as captured.

### 4.2 The `Text fonts:` banner (primary tier) — verbatim, canonical `--config NONE`

`dump_font_debug` prints the literal line `Text fonts:` [kitty/fonts/render.py:L163], then one indented line each for `Normal`/`Bold`/`Italic`/`Bold-Italic` [kitty/fonts/render.py:L164], each followed by that face's `identify_for_debug()` string [kitty/fonts/render.py:L165]. Observed (stderr of the canonical run):

```text
[0.152] OS Window created
[0.162] Failed to open systemd user bus with error: Connection refused
[0.166] Child launched
[0.166] Text fonts:
[0.166]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.166]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.166]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.166]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

**Two facts to note.** First, there is **no `Features: ()` line** — `dump_font_debug` at this revision emits only `Text fonts:` + the four faces (+ an optional `Symbol map fonts:` block, [kitty/fonts/render.py:L166-L170], which is absent here because the effective `symbol_map` is empty, §7). Second, the face string format `<PostScriptName>: <font-file-path>:<face-index>` comes directly from `identify_for_debug` [kitty/freetype.c:L738], which returns `PyUnicode_FromFormat("%s: %V:%d", FT_Get_Postscript_Name(...), self->path, ..., instance.val)` [kitty/freetype.c:L742]; combined with the Python `f'  {text}:'` prefix this yields the observed `  Normal: DejaVuSansMono: …:0`. The mixed file paths (`/usr/share/fonts/...` vs the bundle copy `/root/.local/share/fonts/...`) are **environment‑specific** and simply reflect which file FontConfig matched for each style here.

### 4.3 Per‑codepoint fallback chain (fallback tier) — canonical `--config NONE`

Under the default `monospace` primary (DejaVu Sans Mono), English and Arabic both stay on the primary (§2.6), but **CJK** and **emoji** are absent from DejaVu and drive `create_fallback_face`. Observed (stderr, canonical run):

```text
[0.679] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.680] U+6587 using previous fallback font at index: 0
[0.681] U+65e5 using previous fallback font at index: 0
[0.681] U+672c using previous fallback font at index: 0
[0.681] U+8a9e using previous fallback font at index: 0
[0.682] U+1f600 emoji_presentation Face(family=Noto Color Emoji style=Regular ps_name=NotoColorEmoji path=/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf ttc_index=0 variant=False named_instance=False scalable=False color=True)
[0.683] U+1f642 emoji_presentation using previous fallback font at index: 1
```

**Cause → effect, line by line.** Each line is one call to `output_cell_fallback_data`: it prints `U+<hex>` for the base codepoint [kitty/fonts.c:L458], any combining marks as further `U+<hex>` [kitty/fonts.c:L459-L460], then `bold `/`italic `/`emoji_presentation ` as applicable [kitty/fonts.c:L462-L464], then — if this codepoint reuses an already‑chosen fallback — `using previous fallback font at index: ` [kitty/fonts.c:L465], and finally the face itself via `PyObject_Print(face, stderr, 0)` [kitty/fonts.c:L466] and a newline [kitty/fonts.c:L467]. So the **first** CJK codepoint `U+4e2d` selects **Noto Sans CJK JP** (index 0) and the following CJK codepoints reuse index 0; the first emoji `U+1f600` (with `emoji_presentation`) selects **Noto Color Emoji** (`color=True`, index 1) via the `"emoji"` family branch [kitty/fontconfig.c:L468], and `U+1f642` reuses index 1.

### 4.4 Arabic (RTL) fallback tier — `-o font_family="Liberation Mono"`

To exercise the Arabic fallback path (Liberation Mono genuinely lacks Arabic), the same input was run with Liberation Mono as primary. Banner (stderr):

```text
[0.175] Text fonts:
[0.175]   Normal: LiberationMono: /usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf:0
[0.175]   Bold: LiberationMono-Bold: /usr/share/fonts/truetype/liberation/LiberationMono-Bold.ttf:0
[0.175]   Italic: LiberationMono-Italic: /root/.local/share/fonts/LiberationMono-Italic.ttf:0
[0.175]   Bold-Italic: LiberationMono-BoldItalic: /root/.local/share/fonts/LiberationMono-BoldItalic.ttf:0
```

Arabic fallback chain (stderr):

```text
[0.686] U+645 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/root/.local/share/fonts/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.687] U+631 using previous fallback font at index: 0
[0.687] U+62d using previous fallback font at index: 0
[0.688] U+628 using previous fallback font at index: 0
[0.688] U+627 using previous fallback font at index: 0
[0.689] U+644 using previous fallback font at index: 0
[0.689] U+639 using previous fallback font at index: 0
[0.690] U+645 U+64e using previous fallback font at index: 0
[0.691] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.694] U+1f600 emoji_presentation Face(family=Noto Color Emoji style=Regular ps_name=NotoColorEmoji path=/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf ttc_index=0 variant=False named_instance=False scalable=False color=True)
[0.695] U+1f642 emoji_presentation using previous fallback font at index: 2
```

**What this shows.** The first Arabic letter meem `U+645` selects **DejaVu Sans Mono** as fallback **index 0**; every subsequent Arabic letter (`U+631` reh, `U+62d` hah, `U+628` beh, `U+627` alef, `U+644` lam, `U+639` ain) reuses index 0 — one fallback face serves the whole RTL run. English "Hello World" is not logged because it stays on the **Liberation Mono primary** (only codepoints *missing* from the primary are logged). CJK then takes index 1 and emoji index 2, i.e. the fallback array grows per distinct face.

The line `[0.690] U+645 U+64e using previous fallback font at index: 0` is the **direct observed proof** of combining‑mark clustering from §3.1/§3.4: `output_cell_fallback_data` printed the base meem `U+645` **and** its combining fatha `U+064E` **on one line** because the loop over `cell->cc_idx` [kitty/fonts.c:L459-L460] emits the combining marks that belong to the *same cell/cluster* as the base — the mark did not become its own cell.

### 4.5 Both scripts, both tiers reached — screen‑buffer proof

To confirm Arabic (RTL) **and** English (LTR) **and** the combining sequences actually reached the grid (not merely the log), the live screen buffer was dumped over remote control (`kitten @ get-text`) and hex‑encoded:

```text
Line 1  "مرحبا بالعالم Hello World"
  d9 85 d8 b1 d8 ad d8 a8 d8 a7 20 d8 a8 d8 a7 d9 84 d8 b9 d8 a7 d9 84 d9 85 20 48 65 6c 6c 6f 20 57 6f 72 6c 64
  ^ Arabic UTF-8 (مرحبا بالعالم) ....................................... ^20^ 48 65 6c 6c 6f (Hello) 20 57 6f 72 6c 64 (World)

Line 2  "é à مَر combining-diacritics"
  65 cc 81 20 61 cc 80 20 d9 85 d9 8e d8 b1 20 63 6f 6d 62 ...
  ^ 65(e)+cc81(U+0301)  61(a)+cc80(U+0300)  d985(meem)+d98e(U+064E)  d8b1(reh) ...
```

This shows both directions coexisting on one line (Arabic run immediately followed by the LTR `Hello World`) and the combining diacritics preserved as **base + combining** byte pairs in their cells — the runtime confirmation for §3.3 (bidi) and §3.4 (combining composition/clustering).

### 4.6 Runtime configuration that confirms selections *before* first render

The option values that feed shaping are fixed at `set_options` [kitty/main.py:L249] and `set_font_family` [kitty/main.py:L251], both of which run **before** `_run_app` [kitty/main.py:L252] starts the render loop — i.e. before any glyph is drawn. Their observed effective values under `--config NONE` are tabulated in §7; the ones that directly explain the diagnostics above are: `font_family = monospace` (→ DejaVu Sans Mono banner faces), `force_ltr = False` (→ Arabic detected RTL, §3.3), `disable_ligatures = 0`/never and `font_features = {}` (→ ligature toggles not applied, §3.2), and `symbol_map = {}` (→ no `Symbol map fonts:` block in the banner, §4.2).


---

## 5. Q3 — Cell metrics, baseline & decoration for complex grapheme clusters

### 5.1 Driver and observed values

Cell metrics are computed by `calc_cell_metrics` [kitty/fonts.c:L373], which calls `cell_metrics(...)` on the **medium (Normal) face** [kitty/fonts.c:L375] and then applies any `modify_font` adjustments. The values were captured with a transient probe that drives kitty's **real** `calc_cell_metrics`/`cell_metrics` path (via `kitty.fonts.render`'s test harness `setup_for_testing`/`prerender_function`, no GL required), for the default `monospace` face at two DPIs. Values were stable across two runs.

Observed at **DPI 96** (the harness default):

```text
cell_width=9  cell_height=18  baseline=14
underline_position=15  underline_thickness=1
strikethrough_position=10  strikethrough_thickness=1
cursor_beam_thickness=1.5  cursor_underline_thickness=2.0
medium face = DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
```

Observed at **DPI 100** (the actual Xvfb DPI — `xdpyinfo` reports 100×100 for the 1920×1080 `:101` screen, so this matches the live run, whose window measured 100 columns × 31 lines):

```text
cell_width=9  cell_height=19  baseline=15
underline_position=15  underline_thickness=1
strikethrough_position=11  strikethrough_thickness=1
```

Both sets are reported so the DPI dependence is explicit: the live headless run renders at **9×19** (DPI 100); the DPI‑96 numbers are the harness default. `cell_width` is 9 at both DPIs; `cell_height`, `baseline`, and `strikethrough_position` grow by one pixel from DPI 96 → 100.

### 5.2 Exact formulas (cause → effect)

Every value above is produced by `cell_metrics` [kitty/freetype.c:L387] from the medium face's FreeType metrics:

```c
// kitty/freetype.c:389-403 (abridged to the metric assignments)
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
```

- **`cell_width` / `cell_height`** come from `calc_cell_width` [kitty/freetype.c:L389] and `calc_cell_height` [kitty/freetype.c:L390].
- **`baseline = font_units_to_pixels_y(ascender)`** [kitty/freetype.c:L391] — the pixel distance from the top of the cell to the text baseline. Observed 14 px (DPI 96) / 15 px (DPI 100).
- **`underline_position = MIN(cell_height-1, px(MAX(0, ascender - underline_position)))`** [kitty/freetype.c:L392] — the font's underline position converted to a top‑origin pixel row, clamped to at most `cell_height-1`. Observed 15 px.
- **`underline_thickness = MAX(1, px(underline_thickness))`** [kitty/freetype.c:L393] — at least one pixel. Observed 1 px.
- **`strikethrough_position`** — if the font provides one, `MIN(cell_height-1, px(MAX(0, ascender - strikethrough_position)))` [kitty/freetype.c:L396]; otherwise the fallback `floor(baseline * 0.65)` [kitty/freetype.c:L398]. Observed 10 px (DPI 96) / 11 px (DPI 100). (Note `floor(14*0.65)=9` and `floor(15*0.65)=9`, so the observed 10/11 indicate DejaVu **does** provide a strikethrough position and the font‑provided branch is taken — the fallback would have produced 9.)
- **`strikethrough_thickness`** — if provided, `MAX(1, px(strikethrough_thickness))` [kitty/freetype.c:L401]; otherwise it inherits `underline_thickness` [kitty/freetype.c:L403]. Observed 1 px.

### 5.3 Bounds & error handling (boundaries, not exercised destructively)

`calc_cell_metrics` enforces `MIN_WIDTH 2`, `MIN_HEIGHT 4`, `MAX_DIM 1000` [kitty/fonts.c:L381-L383]; a zero cell width aborts immediately with `fatal("Failed to calculate cell width…")` [kitty/fonts.c:L376], and out‑of‑range heights/widths after `modify_font` adjustment abort via `fatal(...)` [kitty/fonts.c:L389-L392]. The underline position is additionally clamped to `MIN(cell_height-1, …)` [kitty/fonts.c:L409] **`(inferred)`** so a decoration never falls outside the cell. The observed metrics (9×18 / 9×19) sit comfortably inside these bounds, so none of the guards fired.

### 5.4 Overline — explicitly addressed (grep‑evidenced absence, not inference)

The question asks about **overline** and **underline** alignment. A repository‑wide search for the term returns **no matches** in kitty's C/H/Python/GLSL sources at this revision:

```text
$ grep -rIn "overline" kitty/ --include=*.c --include=*.h --include=*.py --include=*.glsl
(no output — ZERO overline matches in kitty source)
```

Therefore, at `815df1e210e0`, kitty has **no font‑derived overline metric and no overline decoration**. `cell_metrics` computes only **baseline**, **underline**, and **strikethrough** [kitty/freetype.c:L387-L403]; the cell fragment shader's decoration comment enumerates exactly `"decorations (cursor, underline, strikethrough)"` [kitty/cell_fragment.glsl:L129] — no overline branch. The two font‑derived decorations are selected in the vertex shader by their sprite positions: `underline_pos` from the `DECORATION_MASK` bits [kitty/cell_vertex.glsl:L184] and `strike_pos` from the strike bit [kitty/cell_vertex.glsl:L185]. This is a factual absence established by the grep above, **not** an `(inferred)` claim.

### 5.5 Link to complex grapheme clusters

Connecting §3.1 to the metrics above: the monotone‑character cluster level [kitty/fonts.c:L1749] keeps a base character plus its combining marks (and the pieces of a ligature) inside **one** HarfBuzz cluster, and the fixed cell box defined by the metrics in §5.1–§5.2 is the region that one cluster is laid into. So a "complex grapheme cluster" such as meem + fatha (observed clustered on one fallback line in §4.4, `U+645 U+64e`) is positioned by HarfBuzz within a single `cell_width × cell_height` box, with the baseline/underline/strikethrough rows above applying uniformly to whatever cluster occupies the cell.


---

## 6. Q4 — GPU texture atlas initialization

### 6.1 Important: the atlas allocation path is silent

Before the numbers, a critical disclosure that shapes how this section is evidenced. The GPU atlas code in `kitty/shaders.c` contains **no** `debug`/`debug_rendering` logging: `alloc_sprite_map` [kitty/shaders.c:L51], `realloc_sprite_texture` [kitty/shaders.c:L108], `ensure_sprite_map` [kitty/shaders.c:L137], and `send_sprite_to_gpu` [kitty/shaders.c:L147] print nothing, even with `--debug-rendering`/`--debug-gl`. (The only `log_error` anywhere in the file is a conditional `glCopyImageSubData` fallback warning [kitty/shaders.c:L90], which did **not** fire in our runs; the only other match is an unrelated window‑title `snprintf` [kitty/shaders.c:L688].)

Consequently:
- The **only directly‑observed** GL/atlas startup log is the `GL version string:` line from `gl_init` (§6.2). That, plus the **absence of GL errors** (which `--debug-gl` enables checking for), is what "verifies the atlas subsystem is ready" at this revision — there is no atlas‑specific log line, and none is invented here.
- The **raw GL limits** are reported as **observed via an independent GL probe** (§2.3) that calls the same `glGetIntegerv` queries kitty uses.
- The **derived page layout** (`xnum`/`max_y`/`znum`, layer cap) is reported as **computed from the observed limits + observed cell metrics**, with the exact source formulas cited — clearly labeled as derived, not as a kitty log.

### 6.2 Observed readiness proof — the GL version line (stdout)

With `--debug-rendering`, `gl_init` prints the GL version once the context is current [kitty/gl.c:L72] (a plain `printf`, hence **stdout**), immediately before enforcing the 3.3 minimum [kitty/gl.c:L74]. Observed (stdout of the canonical run):

```text
[0.125] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

This is the observed proof that (a) the OpenGL context the atlas needs was created, (b) the renderer is Mesa **llvmpipe** software GL at version 4.5 (so the limits below are software‑renderer limits, not a hardware GPU's), and (c) the 3.3 minimum passed. Additionally, running with `--debug-gl` produced **no GL error lines** anywhere in the capture, and an independent pixel sample of the window confirmed content was drawn (non‑background pixels present) — i.e. sprites were uploaded and the atlas was live.

### 6.3 GL limits query and capacity caps

On the first `alloc_sprite_map(cell_width, cell_height)` [kitty/shaders.c:L51], kitty queries the two GL limits and, on non‑Apple platforms, uses them **raw**:

```c
// kitty/shaders.c:53-61
glGetIntegerv(GL_MAX_TEXTURE_SIZE, &(max_texture_size));
glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, &(max_array_texture_layers));
#ifdef __APPLE__
    max_texture_size = MIN(8192, max_texture_size);
    max_array_texture_layers = MIN(512, max_array_texture_layers);
#endif
    sprite_tracker_set_limits(max_texture_size, max_array_texture_layers);
```

The `MIN(8192)/MIN(512)` caps are **macOS‑only** [kitty/shaders.c:L55-L59]; on this Linux/OSMesa host the **raw** limits are used. Observed via the GL probe (§2.3):

```text
GL_MAX_TEXTURE_SIZE         = 16384
GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
```

`sprite_tracker_set_limits` [kitty/fonts.c:L237] then stores the texture size and clamps the **layer cap** to `max_array_len = MIN(0xfffu, max_array_texture_layers)` [kitty/fonts.c:L239]. Computed from the observed limit: `MIN(0xfff, 2048) = MIN(4095, 2048) = ` **2048** layers.

### 6.4 Per‑layer page layout

`sprite_tracker_set_layout(sprite_tracker, cell_width, cell_height)` [kitty/fonts.c:L276] computes the page grid, with clamps that are more precise than a plain `max/cell`:

```c
// kitty/fonts.c:277-280
sprite_tracker->xnum = MIN(MAX(1u, max_texture_size / cell_width), (size_t)UINT16_MAX);
sprite_tracker->max_y = MIN(MAX(1u, max_texture_size / cell_height), (size_t)UINT16_MAX);
sprite_tracker->ynum = 1;
sprite_tracker->x = 0; sprite_tracker->y = 0; sprite_tracker->z = 0;
```

Computed from the observed limit (16384) and the observed cell metrics (§5.1):

| Quantity | Formula [kitty/fonts.c] | DPI 100 (cell 9×19, live run) | DPI 96 (cell 9×18) |
|---|---|---|---|
| `xnum` (sprites/row) | `MIN(MAX(1, 16384/cell_width), 65535)` [L277] | `16384/9` → **1820** | `16384/9` → **1820** |
| `max_y` (max rows/layer) | `MIN(MAX(1, 16384/cell_height), 65535)` [L278] | `16384/19` → **862** | `16384/18` → **910** |
| `ynum` (initial rows) | set to `1` [L279] | **1** | **1** |
| `x,y,z` (initial cursor) | set to `0` [L280] | **0,0,0** | **0,0,0** |
| layer cap | `MIN(0xfff, 2048)` [L239] | **2048** | **2048** |

So one atlas **layer** can hold up to `xnum × max_y` cells (≈ 1820 × 862 ≈ 1.57 million glyph cells per layer at the live DPI), the atlas starts with a single row (`ynum=1`) and grows, and up to **2048** array layers are available before the cap.

### 6.5 Immutable texture allocation — the "before" vs "during/after" states

The atlas is a `GL_TEXTURE_2D_ARRAY` allocated with **immutable** storage in `realloc_sprite_texture` [kitty/shaders.c:L108]:

```c
// kitty/shaders.c:118-123
sprite_tracker_current_layout(fg, &xnum, &ynum, &z);
znum = z + 1;
SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
width = xnum * sprite_map->cell_width; height = ynum * sprite_map->cell_height;
glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum);
```

- **Before any glyph upload:** the sprite map exists but has **no** GL texture — allocation is **lazy**. `ensure_sprite_map` calls `realloc_sprite_texture` only when `!sprite_map->texture_id` [kitty/shaders.c:L139]. So at the instant the atlas subsystem initializes there is no texture object yet. **`(inferred)`** from the lazy‑allocation code path — the empty state cannot be directly logged because the atlas path is silent (§6.1).
- **During/after first upload:** on the first cell that needs a sprite, `ensure_sprite_map` allocates immutable storage sized `width = xnum × cell_width`, `height = ynum × cell_height`, `znum = z+1` (initially 1 layer) with one mip level, internal format `GL_SRGB8_ALPHA8`, `GL_NEAREST` filtering and `GL_CLAMP_TO_EDGE` wrapping; then each shaped glyph's bitmap is uploaded via `send_sprite_to_gpu` [kitty/shaders.c:L147] → `glTexSubImage3D` [kitty/shaders.c:L155]. **Observed indirectly:** content was drawn in the live run (§6.2), which requires sprites to have been uploaded into this texture; there is no per‑upload log to quote.

At the live DPI (cell 9×19, initial `znum=1`), the first immutable allocation is therefore `width = 1820×9 = 16380`, `height = 1×19 = 19`, `1` layer, `GL_SRGB8_ALPHA8` — growing in layers as more distinct glyphs (primary + each fallback face's glyphs) are cached. **`(inferred)`** (computed from observed layout + metrics; not logged).

### 6.6 CPU‑side complement

The GPU atlas position for a glyph is paired with a CPU‑side hash cache in `kitty/glyph-cache.c`: a `SpritePosItem` [kitty/glyph-cache.c:L12-L16] records the `(x, y, z)` atlas coordinates keyed by glyph, populated through `find_or_create_sprite_position` [kitty/glyph-cache.c:L34]. This is the CPU counterpart that lets kitty reuse an already‑uploaded sprite rather than re‑rasterizing/re‑uploading — relevant because the fallback faces observed in §4 each contribute their own glyphs into the same shared atlas layers. **`(inferred)`** (structural; no runtime log).


---

## 7. Runtime configuration snapshot (values confirming the selections before first render)

These are the **effective** font/shaping option values under `--config NONE`, captured from `kitty.options.types.defaults` (i.e. the canonical defaults `set_options` [kitty/main.py:L249] installs before `set_font_family` [kitty/main.py:L251] and before the render loop starts [kitty/main.py:L252]). Each is the observed value paired with its definition and default source.

```text
font_family              = FontSpec(system='monospace')
bold_font                = FontSpec(system='auto')
italic_font              = FontSpec(system='auto')
bold_italic_font         = FontSpec(system='auto')
font_size                = 11.0
force_ltr                = False
disable_ligatures        = 0            # enum: 0 == 'never'
font_features            = {}           # 'none'
text_composition_strategy= 'platform'
symbol_map               = {}
narrow_symbols           = {}
modify_font              = {}
```

| Option | Observed value | Default def | Effect on the observed behavior |
|---|---|---|---|
| `font_family` | `monospace` | [kitty/options/definition.py:L35] | Resolves to the DejaVu Sans Mono banner faces (§4.2) |
| `bold_font` / `italic_font` / `bold_italic_font` | `auto` | [kitty/options/definition.py:L53], [L55], [L57] | Auto‑derives the Bold/Italic/Bold‑Italic faces shown in the banner |
| `font_size` | `11.0` | [kitty/options/definition.py:L59] | With the run DPI (100) yields the 9×19 cell metrics (§5.1) |
| `force_ltr` | `False` | [kitty/options/definition.py:L64] | Arabic runs auto‑detected RTL (§3.3); no LTR override applied |
| `disable_ligatures` | `0` (never) | [kitty/options/definition.py:L115] | The `-liga`/`-dlig`/`-calt` toggles (§3.2) are **not** applied |
| `font_features` | `{}` (none) | [kitty/options/definition.py:L135-L136] | No per‑font feature overrides at startup |
| `text_composition_strategy` | `platform` | [kitty/options/definition.py:L239] | Platform default text compositing |
| `symbol_map` | `{}` | [kitty/options/definition.py:L84-L85] | No symbol faces → banner has no `Symbol map fonts:` block (§4.2) |
| `narrow_symbols` | `{}` | [kitty/options/definition.py:L99-L100] | No narrow‑symbol overrides |
| `modify_font` | `{}` | [kitty/options/definition.py:L192-L193] | No cell/baseline/underline adjustment applied over the font metrics (§5.2) |

Because these are all fixed before `_run_app`, they are the runtime values that **confirm the font/shaping selections before any text is rendered**.

---

## 8. Platform contrast note (Linux vs macOS)

Everything observed above is the **Linux / FontConfig / FreeType / Mesa** path, which is canonical for this environment. For contrast (**not exercised here, so `(inferred)` from code**):

- **Discovery & fallback** on Linux use FontConfig (`create_fallback_face` [kitty/fontconfig.c:L463]); on macOS the equivalent lives in `kitty/core_text.m` using CoreText. The high‑level `dump_font_debug` banner and `output_cell_fallback_data` fallback‑line logic are platform‑independent (both in `render.py`/`fonts.c`), but the **face identity string** is produced per‑platform: `identify_for_debug` in FreeType [kitty/freetype.c:L738] vs its CoreText counterpart [kitty/core_text.m:L966], so the exact family/path/index formatting on macOS may differ from the `<PostScriptName>: <path>:<index>` form observed here.
- **Cell metrics** derive from FreeType face metrics on Linux [kitty/freetype.c:L387-L403]; macOS derives analogous metrics from CoreText, so concrete pixel values would differ with the platform's default monospace font and DPI.
- **Atlas sizing** would additionally differ on macOS because of the `#ifdef __APPLE__` caps `MIN(8192, …)` / `MIN(512, …)` [kitty/shaders.c:L55-L59] — macOS clamps `GL_MAX_TEXTURE_SIZE` to 8192 and array layers to 512, whereas the raw Linux limits here were 16384 / 2048 (§6.3).

---

## 9. Final coverage checklist

Every named item from the four‑part question, with its observed evidence and citation:

| Item | Addressed in | Evidence | Key citation(s) |
|---|---|---|---|
| ligatures | §3.2 | config `disable_ligatures=0`, `font_features={}` (§7) | [kitty/fonts.c:L1755-L1757], [kitty/fonts.c:L812], [kitty/options/definition.py:L115] |
| bidi | §3.3 | Arabic detected RTL; screen‑buffer dump (§4.5); `force_ltr=False` (§7) | [kitty/fonts.c:L687-L688], [kitty/options/definition.py:L64] |
| combining diacritics | §3.4, §4.4–§4.5 | `U+645 U+64e` clustered fallback line; `65 cc 81` / `d9 85 d9 8e` bytes | [kitty/fonts.c:L447], [kitty/fonts.c:L1749], [kitty/fonts.c:L459-L460] |
| font fallback | §3.5, §4.3–§4.4 | CJK→Noto CJK, emoji→Noto Emoji, Arabic→DejaVu | [kitty/fontconfig.c:L444], [kitty/fontconfig.c:L463], [kitty/fonts.c:L482] |
| Arabic (RTL) **and** English (LTR) | §4.4–§4.5 | Arabic fallback chain + `Hello World` on primary + hex dump | [kitty/fonts.c:L687], [kitty/fonts.c:L457-L467] |
| primary **and** fallback tiers | §4.2 + §4.3/§4.4 | `Text fonts:` banner **and** per‑codepoint fallback lines | [kitty/fonts/render.py:L161-L165], [kitty/fonts.c:L457-L467] |
| cell metrics | §5.1 | 9×18 (DPI 96) / 9×19 (DPI 100), stable ×2 | [kitty/fonts.c:L373-L375], [kitty/freetype.c:L387-L390] |
| baseline | §5.1–§5.2 | 14 px (DPI 96) / 15 px (DPI 100) | [kitty/freetype.c:L391] |
| underline **and** overline | §5.2, §5.4 | underline 15/1 px; overline = **grep‑evidenced absent** | [kitty/freetype.c:L392-L393], grep (0 matches), [kitty/cell_fragment.glsl:L129] |
| strikethrough | §5.2 | 10/1 px (DPI 96), 11/1 px (DPI 100); font‑provided branch | [kitty/freetype.c:L396-L403] |
| grapheme clusters | §3.1, §5.5 | monotone cluster keeps base+marks in one cell box | [kitty/fonts.c:L1749] |
| atlas page layout | §6.4 | xnum 1820, max_y 862 (DPI 100)/910 (DPI 96), ynum 1, x/y/z 0 | [kitty/fonts.c:L276-L280] |
| atlas sizing | §6.3, §6.5 | GL_MAX 16384 / 2048 (llvmpipe); `glTexStorage3D` SRGB8_ALPHA8 | [kitty/shaders.c:L53-L54], [kitty/shaders.c:L123] |
| atlas capacity | §6.3 | layer cap `MIN(0xfff, 2048)` = 2048 | [kitty/fonts.c:L239] |
| readiness logs | §6.1–§6.2 | `GL version string:` line (stdout) + no GL errors; atlas alloc **silent** | [kitty/gl.c:L72], [kitty/shaders.c:L108-L147] |
| runtime config snapshot | §7 | effective `--config NONE` option values | [kitty/options/definition.py] |

**Discipline confirmations:** every behavioral claim above pairs observed output with a `file:line` citation; code‑only conclusions (silent‑atlas empty state, RTL cluster‑number internals, CPU sprite cache, macOS contrast) are labeled `(inferred)`; two‑run stability is stated (§2.7); the software renderer (Mesa llvmpipe) is disclosed (§2.3, §6.2); and the read‑only/restore posture with a clean `git status` is confirmed (§2.8).

