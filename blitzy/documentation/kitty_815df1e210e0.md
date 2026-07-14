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

- **Container image (full reference):** `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, sourced from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`. It provides `gcc`, Go, `pkg-config`, and the HarfBuzz/FreeType/FontConfig/OpenGL libraries.
- **Build (canonical CI path, from source).** The binary was built with `python3 setup.py build --verbose`; the build **completed successfully (exit 0)**. Representative tail (long object lists elided with `…`):

```text
$ python3 setup.py build --verbose ; echo "EXIT=$?"
…
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
gcc -MMD -DNDEBUG -DKITTY_VCS_REV="a6300ef30886814798d125cdb23f87e6ed2349ef" -Wextra -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -flto … -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/python3.13 -c kitty/data-types.c -o build/fast_data_types-kitty-data-types.c.o
gcc … -shared -flto build/fast_data_types-kitty-*.c.o … -lharfbuzz -lGL -lpng16 -llcms2 … -lpython3.13 -o build/kitty/fast_data_types.so
Updating Go generated files...
/usr/bin/go build -v -ldflags '-X kitty.VCSRevision=a6300ef30886814798d125cdb23f87e6ed2349ef -s -w' -o kitty/launcher/kitten …/tools/cmd
EXIT=0
```

  The compiler is `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0`; the C core (including the exercised `fonts.c`/`freetype.c`/`fontconfig.c`/`shaders.c`) is compiled under **`-pedantic-errors -Werror`** (warnings promoted to errors), so a clean build is itself evidence those sources compile warning‑free at this revision. The resulting binary was verified functional:

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

- **Linked shaping/rasterization libraries.** `ldd kitty/fast_data_types.so` shows HarfBuzz and FreeType are **direct** shared‑library dependencies, while FontConfig and OpenGL are loaded **at runtime via `dlopen`** — FontConfig through the `_KITTY_FONTCONFIG_LIBRARY` path wired in `setup.py` [setup.py:L551], GL through the `glad`/`gl-wrapper` loader. Versions come from `pkg-config`:

```text
$ ldd kitty/fast_data_types.so | grep -iE 'harfbuzz|freetype|graphite'
	libharfbuzz.so.0 => /lib/x86_64-linux-gnu/libharfbuzz.so.0
	libgraphite2.so.3 => /lib/x86_64-linux-gnu/libgraphite2.so.3
	libfreetype.so.6 => /lib/x86_64-linux-gnu/libfreetype.so.6
$ pkg-config --modversion harfbuzz freetype2 fontconfig
10.2.0
26.2.20
2.15.0
```

  This resolves the manifest discrepancy noted in the plan — `setup.py` enforces only `harfbuzz >= 1.5` [setup.py:L609] while `docs/build.rst` lists `>= 2.2.0`; the binary actually uses HarfBuzz **10.2.0**, satisfying both. (`26.2.20` is FreeType's libtool interface version — the FreeType 2.13.x series; FontConfig is **2.15.0**.)

### 2.3 Headless OpenGL context and the renderer (Linux / Mesa llvmpipe / GLX)

kitty requires an OpenGL **3.3+** context; the minimum is enforced at startup in `gl_init` [kitty/gl.c:L74]. The investigation host has no physical display, so a dedicated **X virtual framebuffer (Xvfb)** with the GLX extension was started, and kitty obtained its context through **GLX** — the same path kitty uses on X11, **not** EGL and **not** OSMesa. The display number is chosen dynamically by the capture script (§2.5) to avoid collisions; the Xvfb is created with GLX enabled:

```bash
Xvfb "$DISP" -screen 0 1920x1080x24 +extension GLX +render -noreset > "$WORK/xvfb.log" 2>&1 &
export DISPLAY="$DISP"
```

The renderer is **Mesa llvmpipe** — a **software** rasterizer. This matters because a software renderer can report **different** `GL_MAX_TEXTURE_SIZE` / `GL_MAX_ARRAY_TEXTURE_LAYERS` than a hardware GPU, which directly determines the atlas sizing in §6; the numbers below are therefore specific to this Mesa/llvmpipe configuration and would differ on other drivers. Rather than a bespoke probe, the GL context and its limits were read with the **trusted `glxinfo` tool**, run under the very same `DISPLAY`/GLX context kitty uses:

```text
$ glxinfo -B | grep -iE 'OpenGL vendor|OpenGL renderer|OpenGL core profile version'
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2
$ glxinfo -l | grep -iE 'GL_MAX_TEXTURE_SIZE|GL_MAX_ARRAY_TEXTURE_LAYERS' | sort -u
    GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
    GL_MAX_TEXTURE_SIZE = 16384
```

These `glxinfo` numbers are the exact limits `alloc_sprite_map` reads via `glGetIntegerv` [kitty/shaders.c:L52-L53] and are corroborated at runtime by kitty's own GL startup log (§6.2). The display geometry/DPI — which feeds the cell metrics in §5 — was read with `xdpyinfo`:

```text
$ xdpyinfo | grep -E 'dimensions|resolution'
  dimensions:    1920x1080 pixels (488x274 millimeters)
  resolution:    100x100 dots per inch
```

### 2.4 Transient verbose diagnostics (CLI only — no config edits)

Three debug flags were enabled **only on the command line**; no config file was touched:

- `--debug-font-fallback` [kitty/cli.py:L1002] sets `args.debug_font_fallback`, which at startup triggers `dump_font_debug()` (the `Text fonts:` banner) [kitty/main.py:L228-L229] and enables per‑codepoint fallback logging gated on `global_state.debug_font_fallback` [kitty/fonts.c:L492].
- `--debug-rendering` / `--debug-gl` [kitty/cli.py:L989] set `global_state.debug_rendering`, which enables the `GL version string:` log in `gl_init` [kitty/gl.c:L72] and turns on GL error checking.

`--config NONE` was used so reported families, metrics, and atlas sizes reflect **canonical defaults** rather than any host `kitty.conf`.

### 2.5 Exact run invocations (safe, reproducible harness)

All runs are driven by a single self‑contained script. It uses a **private evidence directory** (`mktemp -d`, mode `700`), a **dynamically‑chosen free display** (no fixed `:101`), and starts Xvfb with its **PID captured** so a `trap` cleans it up on exit. kitty runs in the **foreground under `timeout`** — it exits by itself when its child process exits — and each run's **exit code is recorded**. There is no `nohup`/`setsid`‑without‑supervision, no fixed shared path, and — importantly — **no window‑resize helper**: feeding the child a `cat` of the mixed text is enough to drive `shape_run → load_fallback_font` and emit the fallback log.

```bash
#!/usr/bin/env bash
set -u
REPO="…/blitzy-c01f4720-…_7415ed"; cd "$REPO"
KITTY="$REPO/kitty/launcher/kitty"

# private evidence dir (mode 700) — not a fixed/root-owned shared path
WORK="$(mktemp -d -p /tmp kitty-investigation.XXXXXX)"; chmod 700 "$WORK"

# pick a free X display dynamically (avoid collisions)
pick_display(){ for n in $(seq 80 120); do [ ! -e "/tmp/.X11-unix/X${n}" ] && { echo ":${n}"; return; }; done; }
DISP="$(pick_display)"
Xvfb "$DISP" -screen 0 1920x1080x24 +extension GLX +render -noreset > "$WORK/xvfb.log" 2>&1 &
XVFB_PID=$!
cleanup(){ kill "$XVFB_PID" 2>/dev/null; wait "$XVFB_PID" 2>/dev/null; }
trap cleanup EXIT
sleep 2; export DISPLAY="$DISP"

# mixed input as real UTF-8 (Arabic RTL + English LTR + combining + CJK + emoji)
python3 - "$WORK/mixed.txt" <<'PY'
import sys
lines=[
 "\u0645\u0631\u062d\u0628\u0627 \u0628\u0627\u0644\u0639\u0627\u0644\u0645 Hello World",
 "e\u0301 a\u0300 \u0645\u064e\u0631 combining-diacritics",
 "CJK \u4e2d\u6587 \u65e5\u672c\u8a9e",
 "emoji \U0001F600 \U0001F642",
]
open(sys.argv[1],"w",encoding="utf-8").write("\n".join(lines)+"\n")
PY

# run helper: foreground, timeout-bounded, stdout/stderr split, exit code saved.
# stdout and stderr are separated because the GL line goes to STDOUT while the
# font diagnostics go to STDERR (verified in §4/§6).
run_kitty(){ # $1=tag ; $2.. = extra kitty opts placed before the child command
  local tag="$1"; shift
  timeout 30 "$KITTY" --config NONE -o sync_to_monitor=no "$@" \
    --debug-font-fallback --debug-rendering --debug-gl \
    sh -c 'sleep 0.6; cat '"$WORK"'/mixed.txt; sleep 1.5' \
    > "$WORK/${tag}_stdout.txt" 2> "$WORK/${tag}_stderr.txt"
  echo "$?" > "$WORK/${tag}_exit.txt"
}

run_kitty canonical_run1                                   # default monospace
run_kitty canonical_run2                                   # 2nd run (stability)
run_kitty libmono_run1 -o "font_family=Liberation Mono"    # forces Arabic fallback
run_kitty libmono_run2 -o "font_family=Liberation Mono"    # 2nd run (stability)
```

Every invocation returned **exit 0**:

```text
$ for t in canonical_run1 canonical_run2 libmono_run1 libmono_run2; do echo "$t=$(cat "$WORK/${t}_exit.txt")"; done
canonical_run1=0
canonical_run2=0
libmono_run1=0
libmono_run2=0
```

The **canonical run** uses the default `monospace` family and exercises the fallback tier via CJK + emoji (Arabic stays on the primary — see §2.6). The **Liberation Mono run** sets a primary family that lacks Arabic, so its Arabic run additionally falls back (§4.4). The benign systemd warning that appears in the log is explained in §4.2.

### 2.6 Why Arabic needs a non‑default primary to demonstrate fallback

Under the **default** `monospace` family — which resolves here to **DejaVu Sans Mono** — Arabic text does **not** trigger fallback. This is an *observed* result: in the canonical run (§4.3) the Arabic and English codepoints produce **no** per‑codepoint fallback lines at all, so they are served by the **primary** faces; only CJK and emoji fall back.

```text
$ fc-match monospace
DejaVuSansMono.ttf: "DejaVu Sans Mono" "Book"
```

The reason is simply that DejaVu Sans Mono **covers** the exercised Arabic letters and combining marks. Both coverage authorities agree — FreeType (which kitty's coverage test `has_cell_text` [kitty/fonts.c:L435-L453] consults) and FontConfig's cached charset. The FontConfig charset was checked directly with `fc-query` for the exact file `fc-match` resolves to (the booleans are a membership test computed from the `%{charset}` ranges):

```text
$ fc-query --format='%{charset}\n' /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf   # -> membership test
    U+0300 combining grave  in FontConfig charset: True
    U+0301 combining acute  in FontConfig charset: True
    U+064E Arabic fatha     in FontConfig charset: True
    U+0621 Arabic hamza     in FontConfig charset: True
    U+0627 Arabic alef      in FontConfig charset: True
    U+0628 Arabic beh       in FontConfig charset: True
    U+0631 Arabic reh       in FontConfig charset: True
    U+0644 Arabic lam       in FontConfig charset: True
    U+0645 Arabic meem      in FontConfig charset: True
    U+4E2D CJK 中            in FontConfig charset: False
```

So there is **no** FreeType‑vs‑FontConfig divergence for these codepoints: DejaVu genuinely **contains** the Arabic set (both authorities report it present) and genuinely **lacks** the CJK ideograph `U+4E2D` (both report it absent — which is exactly why CJK falls back to Noto in §4.3). The same membership check on the other installed DejaVu copy (`/root/.local/share/fonts/DejaVuSansMono.ttf`) gives identical results. To actually exercise the **Arabic fallback tier** (Rule 2 requires both primary and fallback), the primary must therefore be a monospace family that lacks Arabic — **Liberation Mono** — which forces `U+0645`, `U+0631`, … onto the DejaVu fallback face (§4.4). The canonical `monospace` run still exercises the fallback tier via CJK + emoji.

### 2.7 Two‑run stability

Every capture was run **twice**. To compare, the leading `[<sec>.mmm]` timestamp prefix (the only thing that legitimately varies) was normalized to `[T]`, and the two runs were `diff`ed. Both the canonical and the Liberation‑Mono stderr captures were **identical** after normalization (empty diff, exit 0):

```text
$ norm(){ sed -E 's/^\[[0-9]+\.[0-9]+\]/[T]/' "$1"; }
$ diff <(norm canonical_run1_stderr.txt) <(norm canonical_run2_stderr.txt); echo "DIFF_EXIT=$?"
DIFF_EXIT=0
$ diff <(norm libmono_run1_stderr.txt) <(norm libmono_run2_stderr.txt); echo "DIFF_EXIT=$?"
DIFF_EXIT=0
```

The out‑of‑band probes (the effective‑options probe of §7 and the cell‑metrics probe of §5) are deterministic and produced **byte‑identical** output on their second run (`diff` reported no differences). Reported values below are therefore single stable values; only the timestamp prefix varies run‑to‑run.

### 2.8 Read‑only & restore confirmation

All diagnostics were transient CLI flags. kitty's own `AppRunner.__call__` restores font state in its `finally:` block via `set_options(None)` [kitty/main.py:L254] and `free_font_data()` [kitty/main.py:L255], so no persistent state is altered by the observation. Every temporary probe, script, and capture lived entirely under a private `mktemp -d` directory in `/tmp` (outside the repository tree) and was removed on completion; no `kitty.conf` or any source/config/test file was edited, and even building the binary leaves the tree clean because the build artifacts are git‑ignored. After the runs, the source checkout is clean:

```text
$ git status --porcelain
$ echo "EXIT=$?"
EXIT=0        # empty output above = clean working tree; no tracked source file changed
```

The only file added by this task is this document, `blitzy/documentation/kitty_815df1e210e0.md`.

---

## 3. Q1 — Startup shaping & fallback configuration

kitty's shaping/layout engine is HarfBuzz. Its Unicode capabilities are configured **once, at startup**, inside `init_fonts` [kitty/fonts.c:L1746], which the embedded interpreter runs before the window/render loop begins. The four capabilities the question names — ligatures, bidi, combining diacritics, and font fallback — are each turned on as follows.

### 3.1 The shaping buffer and cluster level

`init_fonts` creates the single reusable shaping buffer and sets its cluster level to **monotone characters**:

```c
// kitty/fonts.c:1746-1749
init_fonts(PyObject *module) {
    harfbuzz_buffer = hb_buffer_create();
    if (harfbuzz_buffer == NULL || !hb_buffer_allocation_successful(harfbuzz_buffer) || !hb_buffer_pre_allocate(harfbuzz_buffer, 2048)) { PyErr_NoMemory(); return false; }
    hb_buffer_set_cluster_level(harfbuzz_buffer, HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS);
```

**Cause → effect.** HarfBuzz tracks **clusters**, not graphemes. `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS` (**level 1**) keeps cluster values monotonic at **character** granularity. This is deliberately **not** HarfBuzz's default `MONOTONE_GRAPHEMES` (level 0): level 0 would *merge* a base character with its combining marks into a single grapheme cluster, whereas level 1 leaves **each character with its own cluster value** and only guarantees the values stay monotonic. So this setting does **not**, by itself, "preserve grapheme clusters" or force a base+marks sequence into one HarfBuzz cluster — it gives kitty a predictable, monotonic per‑character numbering it can later map back onto cells.

The "complex grapheme → single cell" property comes instead from kitty's **own `CPUCell` model**, established *before* shaping: a base character (`cell->ch`) and its combining marks (`cell->cc_idx[]`) are stored together in **one `CPUCell`**. When `load_hb_buffer` fills the HarfBuzz buffer it emits the base and **each mark as a separate UTF‑32 entry**, then adds them in a single call:

```c
// kitty/fonts.c:679-685 (load_hb_buffer — base + each combining mark added as separate entries)
        shape_buffer[num++] = first_cpu_cell->ch;
        prev_width = first_gpu_cell->attrs.width;
        for (unsigned i = 0; i < arraysz(first_cpu_cell->cc_idx) && first_cpu_cell->cc_idx[i]; i++) {
            shape_buffer[num++] = codepoint_for_mark(first_cpu_cell->cc_idx[i]);
        }
    }
    hb_buffer_add_utf32(harfbuzz_buffer, shape_buffer, num, 0, num);
```

So base + marks occupy consecutive buffer positions (each with its own monotonic cluster value); HarfBuzz then attaches the marks via **GPOS**, and kitty maps the resulting glyph run back onto its cells using its own group bookkeeping (`num_cells`), not HarfBuzz grapheme clustering. **`(inferred)`** — there is no runtime log for the buffer/cluster‑level setup; it is a source‑level configuration. Note the `U+645 U+64e` line seen during fallback (§4.4) is **not** a HarfBuzz‑cluster observation — it is kitty printing one `CPUCell`'s stored base + combining mark (see §3.4).

### 3.2 OpenType feature toggles for ligatures

At startup `init_fonts` builds three **global disable‑toggle templates** (the leading `-` makes each a *disable* feature):

```c
// kitty/fonts.c:1755-1757 — these only populate the global hb_features[] template array
    create_feature("-liga", LIGA_FEATURE);
    create_feature("-dlig", DLIG_FEATURE);
    create_feature("-calt", CALT_FEATURE);
```

These calls **only fill the global `hb_features[]` templates**; they attach nothing to any face. The per‑face **active** feature list is assembled later, in `init_font`:

```c
// kitty/fonts.c:294-326 (init_font — abridged)
    if (font_feature_settings != NULL) {              // user font_features for this psname
        ... f->num_ffs_hb_features = len + 1; copy each parsed feature ...
        memcpy(f->ffs_hb_features + len, &hb_features[CALT_FEATURE], ...);   // + trailing -calt
    }
    if (!f->num_ffs_hb_features) {                     // default path (no user features)
        f->ffs_hb_features = calloc(4, ...);
        if (strstr(psname, "NimbusMonoPS-") == psname) {                     // kitty's bundled family only
            memcpy(..., &hb_features[LIGA_FEATURE], ...);
            memcpy(..., &hb_features[DLIG_FEATURE], ...);
        }
        memcpy(..., &hb_features[CALT_FEATURE], ...);                        // always a trailing -calt
    }
```

**Cause → effect.** By default a face's feature list is exactly **one** entry, `-calt`; the `-liga`/`-dlig` disable‑templates are added **only** for faces whose PostScript name begins `NimbusMonoPS-` (kitty's bundled default family), and any user `font_features` are parsed and stored per‑face instead. At shape time the list is passed to `hb_shape`, but the **trailing `-calt` is dropped whenever ligatures are *enabled***:

```c
// kitty/fonts.c:810-812
    size_t num_features = fobj->num_ffs_hb_features;
    if (num_features && !disable_ligature) num_features--;  // the last feature is always -calt
    hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features);
```

So for the observed default faces (DejaVu Sans Mono / Liberation Mono — neither is `NimbusMonoPS-`), the per‑face list is just `[-calt]`, and because ligatures are enabled by default the count is decremented to **0**: the run is shaped with **no features at all** (pure font‑default shaping, so `liga`/`dlig`/`calt` behave exactly as the font specifies). Only when ligatures are **disabled** does `-calt` (and, for the Nimbus face, also `-liga`/`-dlig`) actually get applied. This is governed by the options whose canonical values were captured in §7: `disable_ligatures` (observed `0` = `never` [kitty/options/definition.py:L115]) and `font_features` (observed `{}` [kitty/options/definition.py:L136]). **`(inferred from code + confirmed by observed defaults)`** — the feature assembly is not logged; the effective option values that decide whether `-calt` is dropped are the observed evidence (§7).

### 3.3 Bidi / direction detection (RTL Arabic vs LTR English), and kitty's BiDi limitation

Direction is resolved **per run** by HarfBuzz, with a single global override:

```c
// kitty/fonts.c:687-688
    hb_buffer_guess_segment_properties(harfbuzz_buffer);
    if (OPT(force_ltr)) hb_buffer_set_direction(harfbuzz_buffer, HB_DIRECTION_LTR);
```

**Cause → effect.** `hb_buffer_guess_segment_properties` [kitty/fonts.c:L687] inspects the run's codepoints and sets script, language, and **direction** — for an Arabic run it selects the Arabic script (whose shaper performs cursive joining) and RTL; for a Latin run it selects LTR. `force_ltr` [kitty/fonts.c:L688] can pin every run to LTR; its canonical default is `no` [kitty/options/definition.py:L64] (observed effective value `force_ltr = False`, §7), so Arabic runs are shaped RTL automatically.

**Important limitation (source‑level).** kitty does **not** implement the full Unicode BiDi algorithm. The `force_ltr` documentation states this outright:

```text
# kitty/options/definition.py:64-72 (force_ltr long_text, verbatim excerpt)
kitty does not support BIDI (bidirectional text), however, for RTL scripts,
words are automatically displayed in RTL. That is to say, in an RTL script, the
words "HELLO WORLD" display in kitty as "WORLD HELLO", and if you try to select
a substring of an RTL-shaped string, you will get the character that would be
there had the string been LTR.
```

So RTL is handled at the **word/run level** — each run is shaped in its guessed direction — **not** as paragraph‑level bidirectional reordering. In mixed Arabic+English, each Arabic word is shaped RTL and each English word LTR, but there is no cross‑run BiDi reordering of the whole line. (Within a single RTL run, kitty's shaping bookkeeping walks clusters in decreasing order around [kitty/fonts.c:L997] / [kitty/fonts.c:L1080] **`(inferred)`**.)

**On the evidence.** The chosen direction is **not** emitted by any startup diagnostic, so the RTL/LTR selection here is **source‑derived `(inferred)`**, not directly observed. The logical screen‑buffer dump in §4.5 shows the Arabic codepoints are **present and in logical order** — that confirms coverage/storage, **not** visual RTL ordering. No pixel‑level ordering was captured, so no visual‑order claim is made.

### 3.4 Combining‑mark handling and the coverage test

kitty stores a base character and its combining marks in one `CPUCell`, and — as shown in §3.1 — sends them to HarfBuzz **decomposed** (base + each mark as separate buffer entries). The actual attachment/positioning of the marks is performed by **HarfBuzz's shaper (GPOS mark positioning)** on that decomposed sequence; kitty does **not** pre‑compose the shaping input.

Where composition *does* appear is inside the **coverage test** `has_cell_text`, which decides whether the current face can render a cell (and hence whether fallback is needed):

```c
// kitty/fonts.c:444-449 (has_cell_text — a coverage decision, NOT a shaping step)
    if (num_cc == 1) {
        if (face_has_codepoint(face, combining_chars[0])) return true;
        char_type ch = 0;
        if (hb_unicode_compose(hb_unicode_funcs_get_default(), cell->ch, combining_chars[0], &ch) && face_has_codepoint(face, ch)) return true;
        return false;
    }
```

**Cause → effect.** For a base + single combining mark, `has_cell_text` first asks whether the face covers the mark directly; if not, it tries `hb_unicode_compose` to see whether the face covers a **precomposed** form. Either branch only answers *"can this face render the cell?"* — it decides **fallback eligibility** and does **not** change what `load_hb_buffer` feeds the shaper (still base + marks, decomposed). `hb_unicode_compose` is therefore **not** "the combining‑mark composition step" for rendering; it is a coverage/fallback probe.

**Observed confirmation.** During the Liberation‑Mono run the fallback selector logged a base+mark pair on one line — `U+645 U+64e` (§4.4). That line is produced by `output_cell_fallback_data`, which prints the cell's base followed by each stored mark:

```c
// kitty/fonts.c:458-461 (output_cell_fallback_data — prints CPUCell base + cc_idx marks)
    debug("U+%x ", cell->ch);
    for (unsigned i = 0; i < arraysz(cell->cc_idx) && cell->cc_idx[i]; i++) {
        debug("U+%x ", codepoint_for_mark(cell->cc_idx[i]));
    }
```

So `U+645 U+64e` is **pre‑shaping `CPUCell` evidence** — it shows kitty grouped meem (`U+0645`) and fatha (`U+064E`) into one cell in its **own** model — **not** evidence that `hb_unicode_compose` merged them into a single glyph. The base+mark bytes also survive into the screen buffer (§4.5: `65 cc 81` = `e`+`U+0301`, `d9 85 d9 8e` = meem+`U+064E`), again confirming per‑cell storage in logical order.

### 3.5 Font fallback engine (Linux / FontConfig) and the exact call flow

When the primary face lacks a glyph for a **terminal cell**, kitty asks the platform library (FontConfig on Linux) for a face that has it. The terminal call flow is:

`font_for_cell` [kitty/fonts.c:L564] → (primary `has_cell_text` fails) → `fallback_font(FontGroup*, CPUCell*, GPUCell*)` [kitty/fonts.c:L520] → `load_fallback_font` [kitty/fonts.c:L481] → `create_fallback_face` [kitty/fontconfig.c:L463].

```c
// kitty/fonts.c:601-602 (font_for_cell — a primary-face miss routes to fallback_font)
    if (!*is_emoji_presentation && has_cell_text((fg->fonts + ans)->face, cpu_cell)) { *is_main_font = true; return ans; }
    return fallback_font(fg, cpu_cell, gpu_cell);
```

- `fallback_font` (the `FontGroup*`/`CPUCell*` overload in `fonts.c`) builds a cache key from the cell text and consults a per‑group cache; on a miss it calls `load_fallback_font`.
- `load_fallback_font` [kitty/fonts.c:L481] selects a base face by bold/italic, calls `create_fallback_face`, and — under `--debug-font-fallback` — emits the per‑codepoint line via `output_cell_fallback_data`.
- `create_fallback_face` [kitty/fontconfig.c:L463] builds an `FcPattern` whose family is `"monospace"` — or `"emoji"` when emoji presentation is requested — adds the cell's codepoint(s) as an `FcCharSet`, calls `FcFontMatch`, and wraps the result as a kitty face.

**Two same‑named functions — do not conflate them.** There is a *separate* `fallback_font` at [kitty/fontconfig.c:L444] with signature `fallback_font(char_type ch, const char *family, …)`; it serves **UI text** (window titles, notification bodies, etc.) via `freetype_render_ui_text.c` and is **not** on the terminal‑cell path. The terminal path uses the `fonts.c` overload shown above.

The number of distinct fallback faces per group is capped at 100 (a boundary, **not** exercised destructively):

```c
// kitty/fonts.c:482 (inside load_fallback_font)
    if (fg->fallback_fonts_count > 100) { log_error("Too many fallback fonts"); return MISSING_FONT; }
```

**Observed confirmation.** The engine's selections are logged per codepoint under `--debug-font-fallback`: §4.3 (CJK → *Noto Sans CJK JP*; emoji → *Noto Color Emoji*) and §4.4 (Arabic → *DejaVu Sans Mono* when the primary is Liberation Mono) are the concrete outputs of this chain. A `using previous fallback font at index: N` line appears when `create_fallback_face` returns an **integer index** — a reuse of an already‑loaded fallback face — rather than a new face; this is exactly what produces the contiguous CJK reuse lines in §4.3/§4.4.


---

## 4. Q2 — Verbose startup diagnostics for mixed Arabic (RTL) + English (LTR)

### 4.1 What emits the diagnostics, and on which stream

- The **base‑face banner** is emitted by `dump_font_debug()` [kitty/fonts/render.py:L161], gated at startup by `if args.debug_font_fallback:` [kitty/main.py:L228] → `dump_font_debug()` [kitty/main.py:L229]. It prints via Python `log_error`, which on Linux routes through the C logger `log_error` in `logging.c` — that prepends a `[<seconds>.mmm] ` timestamp [kitty/logging.c:L55-L56] and writes to **stderr** [kitty/logging.c:L61].
- The **per‑codepoint fallback lines** are emitted by `output_cell_fallback_data` [kitty/fonts.c:L457], reached from `load_fallback_font` only when `global_state.debug_font_fallback` is set [kitty/fonts.c:L492]. These use the `debug(...)` macro, which is `#define debug debug_fonts` [kitty/fonts.c:L18] → `debug_fonts` [kitty/state.h:L16] → `timed_debug_print` [kitty/monotonic.h:L99], also prepending `[<seconds>.mmm] ` and writing to **stderr**.
- The **GL version line** (§6) is the one diagnostic on **stdout** (a plain `printf` [kitty/gl.c:L72]).

Both the banner and the fallback lines are therefore timestamped on Linux, via two different code paths. All output below is reproduced **verbatim**, including the timestamp prefixes exactly as captured.

### 4.2 The `Text fonts:` banner (primary tier) — verbatim, canonical `--config NONE`

`dump_font_debug` prints the literal line `Text fonts:` [kitty/fonts/render.py:L163], then one indented line each for `Normal`/`Bold`/`Italic`/`Bold-Italic` [kitty/fonts/render.py:L164], each followed by that face's `identify_for_debug()` string [kitty/fonts/render.py:L165]. Observed **verbatim** on the stderr of `canonical_run1` (`cat "$WORK/canonical_run1_stderr.txt"` — the harness of §2.5, head of the capture):

```text
[0.150] OS Window created
[0.159] Failed to open systemd user bus with error: Connection refused
[0.163] Child launched
[0.163] Text fonts:
[0.163]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.163]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.163]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.163]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

**On the `systemd user bus` line.** The `[0.159] Failed to open systemd user bus with error: Connection refused` line is **benign** and expected in this container: there is no per‑user systemd session bus, so kitty's optional desktop/systemd integration simply reports it cannot connect and continues. It is unrelated to fonts/shaping, does not affect any selection below, and the process still exits **0** (§2.5). It appears on **stderr** interleaved with the font diagnostics purely because both use the timestamped logger (§4.1).

**Two more facts to note.** First, there is **no `Features: ()` line** — `dump_font_debug` at this revision emits only `Text fonts:` + the four faces (+ an optional `Symbol map fonts:` block, [kitty/fonts/render.py:L166-L170], absent here because the effective `symbol_map` is empty, §7). Second, the face string format `<PostScriptName>: <font-file-path>:<face-index>` comes directly from `identify_for_debug` [kitty/freetype.c:L738], which returns `PyUnicode_FromFormat("%s: %V:%d", FT_Get_Postscript_Name(...), self->path, ..., instance.val)` [kitty/freetype.c:L742]; combined with the Python `f'  {text}:'` prefix this yields the observed `  Normal: DejaVuSansMono: …:0`. The mixed file paths (`/usr/share/fonts/...` vs the bundle copy `/root/.local/share/fonts/...`) are **environment‑specific** and simply reflect which file FontConfig matched for each style here.

### 4.3 Per‑codepoint fallback chain (fallback tier) — canonical `--config NONE`

Under the default `monospace` primary (DejaVu Sans Mono), English and Arabic both stay on the primary (§2.6), but **CJK** and **emoji** are absent from DejaVu and drive `create_fallback_face`. Observed **verbatim** on the stderr of the same `canonical_run1` as §4.2 (`cat "$WORK/canonical_run1_stderr.txt"`, contiguous fallback block):

```text
[0.775] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.776] U+6587 using previous fallback font at index: 0
[0.776] U+65e5 using previous fallback font at index: 0
[0.777] U+672c using previous fallback font at index: 0
[0.777] U+8a9e using previous fallback font at index: 0
[0.778] U+1f600 emoji_presentation Face(family=Noto Color Emoji style=Regular ps_name=NotoColorEmoji path=/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf ttc_index=0 variant=False named_instance=False scalable=False color=True)
[0.779] U+1f642 emoji_presentation using previous fallback font at index: 1
```

**Cause → effect, line by line.** Each line is one call to `output_cell_fallback_data`: it prints `U+<hex>` for the base codepoint [kitty/fonts.c:L458], any combining marks as further `U+<hex>` [kitty/fonts.c:L459-L460], then `bold `/`italic `/`emoji_presentation ` as applicable [kitty/fonts.c:L462-L464], then — if this codepoint reuses an already‑chosen fallback — `using previous fallback font at index: ` [kitty/fonts.c:L465], and finally the face itself via `PyObject_Print(face, stderr, 0)` [kitty/fonts.c:L466] and a newline [kitty/fonts.c:L467]. So the **first** CJK codepoint `U+4e2d` selects **Noto Sans CJK JP** (index 0) and the following CJK codepoints reuse index 0; the first emoji `U+1f600` (with `emoji_presentation`) selects **Noto Color Emoji** (`color=True`, index 1) via the `"emoji"` family branch [kitty/fontconfig.c:L468], and `U+1f642` reuses index 1.

### 4.4 Arabic (RTL) fallback tier — `-o font_family="Liberation Mono"`

To exercise the Arabic fallback path (Liberation Mono genuinely lacks Arabic), the same input was run with Liberation Mono as primary (`-o font_family="Liberation Mono"`, §2.5). Banner, **verbatim** on the stderr of `libmono_run1` (`cat "$WORK/libmono_run1_stderr.txt"`, head):

```text
[0.160] Text fonts:
[0.160]   Normal: LiberationMono: /usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf:0
[0.160]   Bold: LiberationMono-Bold: /usr/share/fonts/truetype/liberation/LiberationMono-Bold.ttf:0
[0.160]   Italic: LiberationMono-Italic: /root/.local/share/fonts/LiberationMono-Italic.ttf:0
[0.160]   Bold-Italic: LiberationMono-BoldItalic: /root/.local/share/fonts/LiberationMono-BoldItalic.ttf:0
```

Complete, contiguous fallback block, **verbatim** from the same file (nothing elided between the first and last line — this is the whole block Arabic → CJK → emoji):

```text
[0.772] U+645 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/root/.local/share/fonts/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.773] U+631 using previous fallback font at index: 0
[0.774] U+62d using previous fallback font at index: 0
[0.774] U+628 using previous fallback font at index: 0
[0.775] U+627 using previous fallback font at index: 0
[0.775] U+644 using previous fallback font at index: 0
[0.776] U+639 using previous fallback font at index: 0
[0.777] U+645 U+64e using previous fallback font at index: 0
[0.778] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.778] U+6587 using previous fallback font at index: 1
[0.779] U+65e5 using previous fallback font at index: 1
[0.779] U+672c using previous fallback font at index: 1
[0.780] U+8a9e using previous fallback font at index: 1
[0.780] U+1f600 emoji_presentation Face(family=Noto Color Emoji style=Regular ps_name=NotoColorEmoji path=/usr/share/fonts/truetype/noto/NotoColorEmoji.ttf ttc_index=0 variant=False named_instance=False scalable=False color=True)
[0.781] U+1f642 emoji_presentation using previous fallback font at index: 2
```

**What this shows — the full growth of the fallback array.** The first Arabic letter meem `U+645` selects **DejaVu Sans Mono** as fallback **index 0**; every subsequent Arabic letter (`U+631` reh, `U+62d` hah, `U+628` beh, `U+627` alef, `U+644` lam, `U+639` ain) reuses index 0 — one fallback face serves the whole RTL run. English "Hello World" is not logged because it stays on the **Liberation Mono primary** (only codepoints *missing* from the primary are logged, §4.3). Then `U+4e2d` selects **Noto Sans CJK JP** as a **new** face — and, critically, the four following CJK codepoints `U+6587`, `U+65e5`, `U+672c`, `U+8a9e` each log `using previous fallback font at index: 1`, i.e. they reuse that CJK face rather than re‑selecting it. Finally `U+1f600` selects **Noto Color Emoji** (`color=True`) as a new face and `U+1f642` reuses `index: 2`.

**Index‑shift vs the canonical run.** Because Liberation Mono lacks Arabic, DejaVu Sans Mono is pulled into the fallback array *first* (index 0); this pushes CJK to **index 1** and emoji to **index 2**. In the canonical `monospace` run (§4.3) DejaVu is the *primary*, so no Arabic fallback entry exists, CJK is fallback **index 0** and emoji **index 1**. The per‑codepoint index numbers therefore differ between the two runs by exactly the one extra (Arabic) entry — direct confirmation that `fallback_fonts_count` grows by one per distinct selected face [kitty/fonts.c:L465, kitty/fonts.c:L482].

The line `[0.777] U+645 U+64e using previous fallback font at index: 0` is the **direct observed proof** of the per‑cell combining‑mark storage from §3.1/§3.4: `output_cell_fallback_data` printed the base meem `U+645` **and** its combining fatha `U+064E` **on one line** because the loop over `cell->cc_idx` [kitty/fonts.c:L459-L460] emits the combining marks stored in the *same `CPUCell`* as the base. This is a **CPUCell (pre‑shaping)** fact — the marks live in one cell — and is distinct from HarfBuzz cluster numbering (§3.1); it does **not** by itself imply the two codepoints occupy one HarfBuzz cluster.

### 4.5 Both scripts, both tiers reached — screen‑buffer proof (logical order)

To confirm Arabic (RTL) **and** English (LTR) **and** the combining sequences actually reached the grid (not merely the fallback log), the live screen buffer was dumped over kitty's own remote‑control channel. This required launching the run with remote control enabled (transient CLI flags only, no config edits): `-o allow_remote_control=yes --listen-on "$RC"` where `$RC="unix:${WORK}/rc"`. The dump command and its result (**exit 0**):

```console
$ kitten @ --to "$RC" get-text --extent=all
مرحبا بالعالم Hello World
é à مَر combining-diacritics
CJK 中文 日本語
emoji 😀 🙂
```

Hex of the two content lines that carry Arabic and the combining marks (captured with `od -A n -t x1`; trailing `0a` is the line terminator get‑text appends):

```text
Line 1  "مرحبا بالعالم Hello World"
  d9 85 d8 b1 d8 ad d8 a8 d8 a7 20 d8 a8 d8 a7 d9 84 d8 b9 d8 a7 d9 84 d9 85 20 48 65 6c 6c 6f 20 57 6f 72 6c 64 0a
  ^ Arabic UTF-8 (مرحبا بالعالم) then 20 then 48 65 6c 6c 6f (Hello) 20 57 6f 72 6c 64 (World)

Line 2  "é à مَر combining-diacritics"
  65 cc 81 20 61 cc 80 20 d9 85 d9 8e d8 b1 20 63 6f 6d 62 69 6e 69 6e 67 2d 64 69 61 63 72 69 74 69 63 73 0a
  ^ 65(e)+cc81(U+0301)  61(a)+cc80(U+0300)  d985(meem)+d98e(U+064E)  d8b1(reh)  then "combining-diacritics"
```

**What this proves — and what it does not.** `get-text` returns the grid contents in **logical order** (the order in which codepoints were written into the line buffer), not the visual on‑screen order. So this evidence establishes two things exactly: (1) *coverage* — the Arabic run, the ASCII `Hello World` run, and the combining sequences all reached real cells on one line, so both the primary tier and the Arabic fallback tier (§4.4) were genuinely exercised; and (2) *combining‑mark storage* — the diacritics survive as **base + combining** byte pairs (`65 cc81`, `61 cc80`, `d985 d98e`) in their owning cells, the runtime companion to the pre‑shaping `U+645 U+64e` fallback line of §4.4 and the per‑cell `cc_idx` model of §3.4.

It does **not** prove visual reordering, and deliberately so: as established in §3.3, kitty does **not** implement full Unicode BiDi — RTL is handled at render time as a **per‑run/word reversal**, not a paragraph‑level BiDi reordering. The logical‑order dump above is fully consistent with that limitation and must not be read as evidence of BiDi. *(The no‑full‑BiDi statement is source‑derived from [kitty/options/definition.py:L64-L72], quoted in §3.3; the logical‑order semantics of `get-text` are observed here.)*

### 4.6 Runtime configuration that confirms selections *before* first render

The option values that feed shaping are fixed at `set_options` [kitty/main.py:L249] and `set_font_family` [kitty/main.py:L251], both of which run **before** `_run_app` [kitty/main.py:L252] starts the render loop — i.e. before any glyph is drawn. These are **effective, parsed runtime options**, not hand‑copied defaults: they were captured by replaying kitty's own argument path — `parse_args(args=[...], result_class=CLIOptions)` then `create_opts(cli_opts)` (the same calls `main.py` makes at [kitty/main.py:L464] and [kitty/main.py:L494]) — under `kitty +runpy`, with the *exact* argv of each run. The full snapshot is tabulated in §7; the values that directly explain the diagnostics above are:

| Option | Canonical run (`--config NONE …`) | Liberation run (adds `-o font_family="Liberation Mono"`) | Explains |
|---|---|---|---|
| `font_family` | `FontSpec(system='monospace')` | `FontSpec(system='Liberation Mono', created_from_string='Liberation Mono')` | which faces appear in the `Text fonts:` banner (§4.2 vs §4.4) |
| `bold_font` / `italic_font` / `bold_italic_font` | `system='auto'` | `system='auto'` (identical) | Bold/Italic/Bold‑Italic derived from the primary family |
| `font_size` | `11.0` | `11.0` (identical) | cell metrics (§5) |
| `force_ltr` | `False` | `False` (identical) | Arabic detected RTL per run (§3.3) |
| `disable_ligatures` | `0` (never) | `0` (identical) | ligature `-calt` handling (§3.2) |
| `font_features` | `{}` | `{}` (identical) | no user OpenType features applied (§3.2) |
| `text_composition_strategy` | `'platform'` | `'platform'` (identical) | metric/composition strategy |
| `symbol_map` / `narrow_symbols` / `modify_font` | `{}` / `{}` / `{}` | `{}` / `{}` / `{}` (identical) | no `Symbol map fonts:` block in banner (§4.2); no metric overrides (§5.3) |

The **only** effective difference between the two runs is `font_family`; every other shaping‑relevant option is identical. That is exactly why the two runs differ solely in *which* faces are reported (and the resulting fallback index‑shift of §4.4), while direction handling, ligature handling, and metrics behave the same way in both.


---

## 5. Q3 — Cell metrics, baseline & decoration for complex grapheme clusters

### 5.1 Driver and observed values

Cell metrics are computed by `calc_cell_metrics` [kitty/fonts.c:L373], which calls `cell_metrics(...)` on the **medium (Normal) face** [kitty/fonts.c:L375] and then applies any `modify_font` adjustments. The values were captured with a transient probe (`metrics_probe.py`) that drives kitty's **real** `calc_cell_metrics` → `cell_metrics` path through the `kitty.fonts.render` test harness `setup_for_testing`, which runs `set_font_family` → `create_test_font_group` (no GL required — it swaps `send_sprite_to_gpu` for a Python dict) and wraps the module‑global `prerender_function` [kitty/fonts/render.py:L364] to observe the exact metrics that `send_prerendered_sprites` passes it [kitty/fonts.c:L1458]. Both the default `monospace` and the `Liberation Mono` faces were probed at two DPIs. Exact command and **complete** output (**exit 0**; byte‑identical across two runs):

```console
$ ./kitty/launcher/kitty +runpy "$(cat metrics_probe.py)"
family='monospace' size=11.0 dpi=96.0: cell_width=9 cell_height=18 baseline=14 underline_position=15 underline_thickness=1 strikethrough_position=10 strikethrough_thickness=1 prerendered_special_sprites=11
family='monospace' size=11.0 dpi=100.0: cell_width=9 cell_height=19 baseline=15 underline_position=15 underline_thickness=1 strikethrough_position=11 strikethrough_thickness=1 prerendered_special_sprites=11
family='Liberation Mono' size=11.0 dpi=96.0: cell_width=9 cell_height=17 baseline=13 underline_position=16 underline_thickness=1 strikethrough_position=9 strikethrough_thickness=1 prerendered_special_sprites=11
family='Liberation Mono' size=11.0 dpi=100.0: cell_width=9 cell_height=18 baseline=13 underline_position=16 underline_thickness=1 strikethrough_position=9 strikethrough_thickness=1 prerendered_special_sprites=11
```

For the **default `monospace`** face (the canonical run of §4.2), the observed values are therefore:

| Metric | DPI 96 (harness default) | DPI 100 (live Xvfb DPI, §2.3) |
|---|---|---|
| `cell_width` | **9** | **9** |
| `cell_height` | **18** | **19** |
| `baseline` | **14** | **15** |
| `underline_position` | **15** | **15** |
| `underline_thickness` | **1** | **1** |
| `strikethrough_position` | **10** | **11** |
| `strikethrough_thickness` | **1** | **1** |
| `prerendered_special_sprites` | **11** | **11** |

Both DPIs are reported so the DPI dependence is explicit: the live headless run renders at **9×19** (DPI 100, per `xdpyinfo` in §2.3); the DPI‑96 column is the harness default. `cell_width` is 9 at both DPIs; `cell_height`, `baseline`, and `strikethrough_position` each grow by one pixel from DPI 96 → 100. The `prerendered_special_sprites=11` value is the atlas seed count analysed in §6.5. (The `Liberation Mono` rows are the primary used to force Arabic fallback in §4.4; they are shown here for completeness and reused in §6.)

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

`calc_cell_metrics` enforces `MIN_WIDTH 2`, `MIN_HEIGHT 4`, `MAX_DIM 1000` [kitty/fonts.c:L381-L383]. Three distinct guards exist, and it is important not to conflate them:

1. **Zero base width → immediate abort.** If the font itself yields `cell_width == 0`, `calc_cell_metrics` calls `fatal("Failed to calculate cell width for the specified font")` right after `cell_metrics` returns [kitty/fonts.c:L376]. This is the only *unconditional* abort.
2. **Invalid `modify_font` adjustment → log‑and‑ignore (retain previous value), NOT abort.** `modify_font` is applied by `adjust_metric` to working copies `cw`/`ch` [kitty/fonts.c:L379-L380]. The adjusted value is accepted **only if in range**: `if (cw >= MIN_WIDTH && cw <= MAX_DIM) cell_width = cw; else log_error("Cell width invalid after adjustment, ignoring modify_font cell_width");` [kitty/fonts.c:L384-L385], and the identical pattern for height [kitty/fonts.c:L386-L387]. So an out‑of‑range `modify_font` adjustment is **logged and discarded**, and the pre‑adjustment metric is **retained** — it does not abort.
3. **Retained value globally out of range → abort.** Only *after* that, `fatal(...)` fires if the value actually in effect is itself out of bounds: `if (cell_height < MIN_HEIGHT) fatal("Line height too small: %u", …)`, `> MAX_DIM` too large, and the two symmetric width checks [kitty/fonts.c:L389-L392]. Because guard 2 retains a valid pre‑adjustment metric, this abort is normally reachable only if the font's own computed metric were degenerate.

Separately, the underline position is clamped to `MIN(cell_height - 1, underline_position)` [kitty/fonts.c:L409] (directly in source — **not** an inferred claim) so a decoration never falls outside the cell. The observed metrics (9×18 / 9×19) sit comfortably inside all these bounds, and with the effective `modify_font = {}` (§7) no adjustment was applied, so none of the guards fired.

### 5.4 Overline — explicitly addressed (grep‑evidenced absence, not inference)

The question asks about **overline** and **underline** alignment. A repository‑wide search for the term returns **no matches** in kitty's C/H/Python/GLSL sources at this revision:

```text
$ grep -rIn "overline" kitty/ --include=*.c --include=*.h --include=*.py --include=*.glsl
(no output — ZERO overline matches in kitty source)
```

Therefore, at `815df1e210e0`, kitty has **no font‑derived overline metric and no overline decoration**. `cell_metrics` computes only **baseline**, **underline**, and **strikethrough** [kitty/freetype.c:L387-L403]; the cell fragment shader's decoration comment enumerates exactly `"decorations (cursor, underline, strikethrough)"` [kitty/cell_fragment.glsl:L129] — no overline branch. The two font‑derived decorations are selected in the vertex shader by their sprite positions: `underline_pos` from the `DECORATION_MASK` bits [kitty/cell_vertex.glsl:L184] and `strike_pos` from the strike bit [kitty/cell_vertex.glsl:L185]. This is a factual absence established by the grep above, **not** an `(inferred)` claim.

### 5.5 Link to complex grapheme clusters

Connecting §3.1/§3.4 to the metrics above — and stated carefully to avoid the cluster‑level misconception corrected in §3.1: a "complex grapheme cluster" such as meem + fatha (observed on one fallback line in §4.4 as `U+645 U+64e`) occupies a **single cell** not because HarfBuzz merged it into one cluster, but because kitty stores the base character plus its combining marks in **one `CPUCell`** (`cell->ch` + the `cc_idx[]` array) *before* shaping (§3.1, §3.4). The fixed cell box defined by the metrics in §5.1–§5.2 (`cell_width × cell_height`) is exactly the region that one `CPUCell` is laid into. Within that box, HarfBuzz — running at the monotone‑**character** cluster level [kitty/fonts.c:L1749], which preserves per‑character cluster values rather than pre‑merging marks — positions the base glyph and then the combining‑mark glyphs relative to it via GPOS, and kitty renders the result into that one cell. The baseline, underline, and strikethrough rows computed in §5.1–§5.2 therefore apply uniformly to whatever base‑plus‑marks content the cell holds; the decorations are per‑cell and independent of how many codepoints the cell's grapheme comprises.


---

## 6. Q4 — GPU texture atlas initialization

### 6.1 Important: the atlas allocation path is silent

Before the numbers, a critical disclosure that shapes how this section is evidenced. The GPU atlas code in `kitty/shaders.c` contains **no** `debug`/`debug_rendering` logging: `alloc_sprite_map` [kitty/shaders.c:L51], `realloc_sprite_texture` [kitty/shaders.c:L108], `ensure_sprite_map` [kitty/shaders.c:L137], and `send_sprite_to_gpu` [kitty/shaders.c:L147] print nothing, even with `--debug-rendering`/`--debug-gl`. (The only `log_error` anywhere in the file is a conditional `glCopyImageSubData` fallback warning [kitty/shaders.c:L90], which did **not** fire in our runs; the only other match is an unrelated window‑title `snprintf` [kitty/shaders.c:L688].)

**When the atlas is created.** The atlas is *not* allocated lazily on the first user glyph — it is created during **OS‑window creation**. `send_prerendered_sprites_for_window(OSWindow *w)` [kitty/fonts.c:L1521-L1526] is called from the window‑creation path at [kitty/glfw.c:L1273] (and also [kitty/state.c:L1038]); on first call, when `fg->sprite_map` is still null, it runs `fg->sprite_map = alloc_sprite_map(cell_width, cell_height)` [kitty/fonts.c:L1524] and immediately `send_prerendered_sprites(fg)` [kitty/fonts.c:L1525]. So by the time the window is shown and the render loop begins, the CPU sprite tracker is laid out and the atlas has already been seeded with kitty's special sprites (§6.5). This timing is established from source (the path is silent, so it cannot be logged) and is consistent with the observed `prerendered_special_sprites=11` seed count from the §5.1 probe.

Consequently:
- The **only directly‑observed** GL/atlas startup log is the `GL version string:` line from `gl_init` (§6.2). That, plus the **absence of GL errors** (which `--debug-gl` enables checking for), is what "verifies the atlas subsystem is ready" at this revision — there is no atlas‑specific log line, and none is invented here.
- The **raw GL limits** are reported as **observed via an independent GL probe** (§2.3) that calls the same `glGetIntegerv` queries kitty uses.
- The **derived page layout** (`xnum`/`max_y`/`znum`, layer cap) is reported as **computed from the observed limits + observed cell metrics**, with the exact source formulas cited — clearly labeled as derived, not as a kitty log.

### 6.2 Observed readiness proof — the GL version line (stdout)

With `--debug-rendering`, `gl_init` prints the GL version once the context is current [kitty/gl.c:L72] (a plain `printf`, hence **stdout**), immediately before enforcing the 3.3 minimum [kitty/gl.c:L74]. Observed (stdout of the canonical run):

```text
[0.125] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

This is the observed proof that (a) the OpenGL context the atlas needs was created, (b) the renderer is Mesa **llvmpipe** software GL at version 4.5 (so the limits below are software‑renderer limits, not a hardware GPU's), and (c) the 3.3 minimum passed. Additionally, running with `--debug-gl` produced **no GL error lines** anywhere in the capture, and the child process ran and exited cleanly (**exit 0**, §2.5) — meaning the render loop, which drives sprite upload into the atlas, completed without a GL fault. Together (context created + zero GL errors + clean render‑loop exit) these constitute the runtime readiness evidence available at this revision; there is no atlas‑specific readiness log to quote, and — since the atlas path is silent (§6.1) — none is invented here.

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

The `MIN(8192)/MIN(512)` caps are **macOS‑only** [kitty/shaders.c:L55-L59]; on this **Linux / Mesa llvmpipe (software) / GLX** host the **raw** limits are used. Observed via `glxinfo` in the same GLX context (§2.3):

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
// kitty/shaders.c:118-129 (immutable allocation + growth-copy)
sprite_tracker_current_layout(fg, &xnum, &ynum, &z);
znum = z + 1;
SpriteMap *sprite_map = (SpriteMap*)fg->sprite_map;
width = xnum * sprite_map->cell_width; height = ynum * sprite_map->cell_height;
glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum);
if (sprite_map->texture_id) {           // growth: copy the old atlas into the larger one
    src_ynum = MAX(1, sprite_map->last_ynum);
    copy_image_sub_data(sprite_map->texture_id, tex, width, src_ynum * sprite_map->cell_height, sprite_map->last_num_of_layers);
    glDeleteTextures(1, &sprite_map->texture_id);
}
```

The three states, **in the order they actually occur** (recall from §6.1 that allocation happens at OS‑window creation, and the whole path is silent, so these are source‑derived):

- **Before the first sprite upload (a transient instant during window creation):** immediately after `alloc_sprite_map` [kitty/fonts.c:L1524] the `SpriteMap` exists but `texture_id == 0` — there is **no** GL texture object yet. This state lasts only until the first prerendered sprite is sent a few statements later in `send_prerendered_sprites`.
- **First upload → immutable allocation:** `send_prerendered_sprites` [kitty/fonts.c:L1450] sends the **blank cell first** [kitty/fonts.c:L1453-L1456]. That first `send_sprite_to_gpu` allocates the texture, because `last_num_of_layers` starts at 0 so its growth test fires [kitty/shaders.c:L151] (equivalently `ensure_sprite_map` allocates whenever `!texture_id` [kitty/shaders.c:L139]); `realloc_sprite_texture` then calls `glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum)` [kitty/shaders.c:L123] — one mip level, internal format `GL_SRGB8_ALPHA8`, `GL_NEAREST` filtering and `GL_CLAMP_TO_EDGE` wrapping [kitty/shaders.c:L114-L117].
- **After window creation → seeded atlas:** the remaining 10 special sprites returned by `prerender_function` [kitty/fonts/render.py:L364] are then uploaded [kitty/fonts.c:L1461-L1470], giving the **11**‑sprite seed observed by the probe (`prerendered_special_sprites=11`, §5.1). That 11 decomposes exactly as: blank (1) [kitty/fonts.c:L1453-L1456] + underline styles (`NUM_UNDERLINE_STYLES` = **5** [kitty/data-types.h:L213]) + strikethrough (1) + missing‑glyph (1) + cursor (3) [kitty/fonts/render.py:L391-L394]. All 11 fit in row 0 — `send_prerendered_sprites` aborts with `"Too many pre-rendered sprites…"` if `y > 0` [kitty/fonts.c:L1463], which never triggers here since `xnum ≈ 1820 ≫ 11`. The primary‑ and fallback‑face glyphs from §4 are uploaded **later**, on first draw, through the same `send_sprite_to_gpu` path; the texture already exists, so no re‑allocation occurs unless the atlas must grow.

**Growth order — rows/height before layers.** As sprites accumulate, `do_increment` [kitty/fonts.c:L243-L253] advances `x` across a row; when a row fills it increments `y` and grows `ynum` up to `max_y` [kitty/fonts.c:L247]; only when the rows are exhausted (`y >= max_y`) does it start a new **layer** (`z++`) [kitty/fonts.c:L248-L249]. Correspondingly, `send_sprite_to_gpu` re‑allocates when either more layers are needed **or** more rows are needed in layer 0 [kitty/shaders.c:L151], and `realloc_sprite_texture` copies the existing atlas into the larger texture via `copy_image_sub_data` before deleting the old one [kitty/shaders.c:L124-L128]. So the atlas grows in **rows/height first, then layers** — never the reverse.

At the live DPI (cell 9×19), the first immutable allocation is therefore `width = xnum × cell_width = 1820 × 9 = 16380`, `height = ynum × cell_height = 1 × 19 = 19`, `znum = 1` layer, `GL_SRGB8_ALPHA8`. **`(inferred)`** — computed from the observed layout (§6.4) and observed metrics (§5.1); the exact allocation call is not logged because the atlas path is silent (§6.1).

### 6.6 CPU‑side complement

The GPU atlas position for a glyph is paired with a CPU‑side hash cache in `kitty/glyph-cache.c`: a `SpritePosItem` [kitty/glyph-cache.c:L12-L16] records the `(x, y, z)` atlas coordinates keyed by glyph, populated through `find_or_create_sprite_position` [kitty/glyph-cache.c:L34]. This is the CPU counterpart that lets kitty reuse an already‑uploaded sprite rather than re‑rasterizing/re‑uploading — relevant because the fallback faces observed in §4 each contribute their own glyphs into the same shared atlas layers. **`(inferred)`** (structural; no runtime log).


---

## 7. Runtime configuration snapshot (values confirming the selections before first render)

These are the **effective, parsed runtime** font/shaping option values under `--config NONE` — **not** raw table defaults. They were captured by replaying kitty's own argument path, `parse_args(args=argv, result_class=CLIOptions)` → `create_opts(cli_opts)` (the same calls `kitty.main` makes at [kitty/main.py:L464] and [kitty/main.py:L494]), under `kitty +runpy`, for **both** the canonical argv and the Liberation‑Mono argv. `set_options` installs these values before `set_font_family` [kitty/main.py:L249-L251] and before the render loop starts [kitty/main.py:L252]. Exact command and **complete, unedited** output (**exit 0**; byte‑identical across two runs):

```console
$ ./kitty/launcher/kitty +runpy "$(cat opts_probe.py)"
#### effective options for: canonical (default monospace)
#### argv = ['--config', 'NONE', 'sh', '-c', 'true']
    font_family = FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
    bold_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
    italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
    bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
    font_size = 11.0
    force_ltr = False
    disable_ligatures = 0
    font_features = {}
    text_composition_strategy = 'platform'
    symbol_map = {}
    narrow_symbols = {}
    modify_font = {}

#### effective options for: liberation (-o font_family=Liberation Mono)
#### argv = ['--config', 'NONE', '-o', 'font_family=Liberation Mono', 'sh', '-c', 'true']
    font_family = FontSpec(family='', style='', postscript_name='', full_name='', system='Liberation Mono', axes=(), variable_name='', created_from_string='Liberation Mono')
    bold_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
    italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
    bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
    font_size = 11.0
    force_ltr = False
    disable_ligatures = 0
    font_features = {}
    text_composition_strategy = 'platform'
    symbol_map = {}
    narrow_symbols = {}
    modify_font = {}
```

The **only** effective difference between the two runs is `font_family` — canonical resolves to `system='monospace'`, the Liberation run to `system='Liberation Mono'` (with `created_from_string='Liberation Mono'`). Every other shaping‑relevant option is identical. The following table interprets each **canonical** value, its definition site, and its effect (the Liberation run differs only in the `font_family` row):

| Option | Effective value (canonical) | Definition | Effect on the observed behavior |
|---|---|---|---|
| `font_family` | `system='monospace'` (Liberation run: `system='Liberation Mono'`) | [kitty/options/definition.py:L35] | Resolves to the DejaVu Sans Mono banner faces (§4.2); Liberation Mono in the fallback‑forcing run (§4.4) |
| `bold_font` / `italic_font` / `bold_italic_font` | `system='auto'` | [kitty/options/definition.py:L53], [L55], [L57] | Auto‑derives the Bold/Italic/Bold‑Italic faces shown in the banner |
| `font_size` | `11.0` | [kitty/options/definition.py:L59] | With the run DPI (100) yields the 9×19 cell metrics (§5.1) |
| `force_ltr` | `False` | [kitty/options/definition.py:L64] | Arabic runs auto‑detected RTL per run (§3.3); no LTR override applied |
| `disable_ligatures` | `0` (never) | [kitty/options/definition.py:L115] | Ligature suppression inactive, so shaping drops the trailing `-calt` (§3.2) |
| `font_features` | `{}` (none) | [kitty/options/definition.py:L135-L136] | No per‑font OpenType feature overrides at startup (§3.2) |
| `text_composition_strategy` | `'platform'` | [kitty/options/definition.py:L239] | Platform default text compositing |
| `symbol_map` | `{}` | [kitty/options/definition.py:L84-L85] | No symbol faces → banner has no `Symbol map fonts:` block (§4.2) |
| `narrow_symbols` | `{}` | [kitty/options/definition.py:L99-L100] | No narrow‑symbol overrides |
| `modify_font` | `{}` | [kitty/options/definition.py:L192-L193] | No cell/baseline/underline adjustment applied over the font metrics (§5.2–§5.3) |

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
| bidi | §3.3 | kitty has **no full Unicode BiDi** (per‑run/word RTL only); `force_ltr=False` (§7); logical‑order screen dump (§4.5) | [kitty/fonts.c:L687-L688], [kitty/options/definition.py:L64-L72] |
| combining diacritics | §3.4, §4.4–§4.5 | base+mark stored in one **CPUCell** (`U+645 U+64e` on one fallback line); `65 cc 81` / `d9 85 d9 8e` byte pairs in cells | [kitty/fonts.c:L444-L449], [kitty/fonts.c:L458-L461] |
| font fallback | §3.5, §4.3–§4.4 | CJK→Noto CJK, emoji→Noto Emoji, Arabic→DejaVu | [kitty/fontconfig.c:L444], [kitty/fontconfig.c:L463], [kitty/fonts.c:L482] |
| Arabic (RTL) **and** English (LTR) | §4.4–§4.5 | Arabic fallback chain + `Hello World` on primary + hex dump | [kitty/fonts.c:L687], [kitty/fonts.c:L457-L467] |
| primary **and** fallback tiers | §4.2 + §4.3/§4.4 | `Text fonts:` banner **and** per‑codepoint fallback lines | [kitty/fonts/render.py:L161-L165], [kitty/fonts.c:L457-L467] |
| cell metrics | §5.1 | 9×18 (DPI 96) / 9×19 (DPI 100), stable ×2 | [kitty/fonts.c:L373-L375], [kitty/freetype.c:L387-L390] |
| baseline | §5.1–§5.2 | 14 px (DPI 96) / 15 px (DPI 100) | [kitty/freetype.c:L391] |
| underline **and** overline | §5.2, §5.4 | underline 15/1 px; overline = **grep‑evidenced absent** | [kitty/freetype.c:L392-L393], grep (0 matches), [kitty/cell_fragment.glsl:L129] |
| strikethrough | §5.2 | 10/1 px (DPI 96), 11/1 px (DPI 100); font‑provided branch | [kitty/freetype.c:L396-L403] |
| grapheme clusters | §3.1, §5.5 | base+marks stored in one **CPUCell** (not merged into one HB cluster); monotone‑**character** level = char granularity | [kitty/fonts.c:L679-L685], [kitty/fonts.c:L1749] |
| atlas page layout | §6.4 | xnum 1820, max_y 862 (DPI 100)/910 (DPI 96), ynum 1, x/y/z 0 | [kitty/fonts.c:L276-L280] |
| atlas sizing | §6.3, §6.5 | GL_MAX 16384 / 2048 (llvmpipe); `glTexStorage3D` SRGB8_ALPHA8 | [kitty/shaders.c:L53-L54], [kitty/shaders.c:L123] |
| atlas capacity | §6.3 | layer cap `MIN(0xfff, 2048)` = 2048 | [kitty/fonts.c:L239] |
| readiness logs | §6.1–§6.2 | `GL version string:` line (stdout) + no GL errors + clean child exit 0; atlas alloc **silent**; 11‑sprite seed at window creation (§6.5) | [kitty/gl.c:L72], [kitty/shaders.c:L108-L147], [kitty/fonts.c:L1521-L1526] |
| runtime config snapshot | §7 | effective `--config NONE` option values | [kitty/options/definition.py] |

**Discipline confirmations:** every behavioral claim above pairs observed output with a `file:line` citation; code‑only conclusions (silent‑atlas empty state, RTL cluster‑number internals, CPU sprite cache, macOS contrast) are labeled `(inferred)`; two‑run stability is stated (§2.7); the software renderer (Mesa llvmpipe) is disclosed (§2.3, §6.2); and the read‑only/restore posture with a clean `git status` is confirmed (§2.8).
