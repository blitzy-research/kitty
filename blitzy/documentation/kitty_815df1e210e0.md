# kitty — Text-Shaping / Layout Configuration, Startup Font Fallback, Cell Metrics, and GPU Texture-Atlas Initialization

*An evidence-based, runtime-observed answer for the source branch `kitty_815df1e210e0`.*

This document explains — and proves through actual runtime observation on a default, canonical build — how the [kitty](https://github.com/kovidgoyal/kitty) terminal emulator (a) configures its text-shaping / layout engine for complex Unicode (ligatures, bidirectional text, combining diacritics) and performs font fallback at startup; (b) what the verbose-logging startup diagnostics report for mixed **Arabic (RTL) + English (LTR)** text; (c) the default cell metrics, baseline, and decoration alignment computed as the screen-grid / shaping subsystems initialize; and (d) how the GPU texture-atlas is laid out, sized, and made ready at launch.

Every value in this document was produced by **building kitty with `python3 setup.py` and running the produced launcher `kitty/launcher/kitty`** under a headless OpenGL display (Xvfb) with a window manager (openbox), with the real diagnostic flags `--debug-font-fallback` and `--debug-rendering` enabled. The exact commands, complete unedited output, and the secure harness that produced them are embedded below.

---

## 0. How to read this document (labels, streams, timestamps, methodology)

### 0.1 Evidence labels

Every claim carries one of the following labels so that observed facts are never confused with derivations:

| Label | Meaning |
|-------|---------|
| **OBSERVED (canonical)** | Emitted by the real launcher `kitty/launcher/kitty` at startup through its real entry point. This is the primary, canonical evidence required by the task. |
| **OBSERVED (production binding)** | A real value returned by a production computation function invoked through a Python binding, where the binding does **not** alter the computation being reported (used only for `cell_width`/`cell_height`, whose math is independent of the substituted GPU-upload callback). |
| **SOURCE-DERIVED** | A value computed by applying a documented production formula, verbatim from the source, to real inputs — used when the launcher emits no log line for the quantity. Cross-validated against an OBSERVED value wherever one exists. |
| **INFERRED** | A conclusion reasoned from the source when no runtime signal exists (e.g., "atlas ready" has no single log line). |
| **NON-CANONICAL CORROBORATION** | A value from a test/bypass binding (`setup_for_testing`, `test_shape`, `create_test_font_group`, `get_fallback_font`) that replaces the real GPU upload with a Python callback and sets artificial sprite limits [`kitty/fonts/render.py:408-422`]. **Never** used as primary evidence — only to corroborate an OBSERVED or SOURCE-DERIVED fact, and always labelled as such. |

**Canonical-vs-corroboration policy.** The four objectives are answered from **OBSERVED (canonical)** launcher output and, where the launcher emits nothing (cell-metric fields, atlas capacity internals), from **SOURCE-DERIVED** values validated against the source. The in-repo test harness `setup_for_testing` is explicitly **non-canonical** [`kitty/fonts/render.py:408`] because it calls `set_send_sprite_to_gpu(send_to_gpu)` [`:422`], replacing the OpenGL upload with a Python dict callback, and `sprite_map_set_limits(100000, 100)` [`:421`], setting artificial atlas limits. It appears here only as clearly-labelled corroboration.

### 0.2 Output streams

kitty writes diagnostics to **both** stdout and stderr; the harness captures both by redirecting `> log 2>&1`.
- The `--debug-rendering` **GL version banner** is written to **STDOUT** by a dedicated `printf` in `gl.c` [`kitty/gl.c:72`].
- The **font-dump** (`Text fonts:` / `Symbol map fonts:`) and the per-cell **fallback** lines are written to **STDERR** via `log_error(...)`.

### 0.3 Timestamp provenance (correcting a common misconception)

Debug lines are prefixed with a bracketed number, but they do **not** all come from the same function:
- Per-cell **fallback / render** lines (`U+... using previous fallback font ...`) are printed by `timed_debug_print`, whose format string emits the monotonic `[%.3f] ` prefix [`kitty/monotonic.h:99`].
- The **font-dump** block (`Text fonts:` …) is printed by Python `log_error(...)` from `dump_font_debug()` [`kitty/fonts/render.py:161`], routed through kitty's logging layer (`logging.c`), **not** `timed_debug_print`.
- The **GL banner** is printed by its own `printf` in `gl.c` [`kitty/gl.c:72`], timestamped with `monotonic()` directly.

Because these three mechanisms interleave stdout and stderr, the GL-banner line frequently appears *after* the font block in a merged capture even though its monotonic timestamp is *smaller* — an ordering artifact of two streams, not a causal ordering. This is called out where it matters.

### 0.4 Debug-macro gating (correcting "compile-time no-op")

`debug_rendering` and `debug_fonts` are **runtime** `if`-style macros that test global-state flags — they are **not** compile-time no-ops:

```c
// kitty/state.h
#define debug_rendering(...) if (global_state.debug_rendering) { timed_debug_print(__VA_ARGS__); }   // :14
#define debug_fonts(...)     if (global_state.debug_font_fallback) { timed_debug_print(__VA_ARGS__); } // :16
```

The flags are populated at startup from the CLI (`--debug-rendering`, `--debug-font-fallback`); the code is always compiled in and gated at runtime. `fonts.c` additionally aliases `#define debug debug_fonts` [`kitty/fonts.c:18`].

### 0.5 Source-anchor convention

Every claim is anchored with a `file:line` reference to the checkout at HEAD `722ca9e9e` (branch content `kitty_815df1e210e0`). Line numbers were re-verified against the working tree.

---

## 1. Provenance and chronology — run first, then write

This investigation was performed by **running before writing**. The ordered procedure was:

1. **Baseline** the repository (clean tree, record HEAD).
2. **Build** the default configuration with `python3 setup.py`.
3. **Bring up** a headless GL display (Xvfb) + window manager (openbox).
4. **Run** the produced launcher with the debug flags, feeding literal mixed Arabic + English + combining-mark input; repeat each observation ≥ 2×.
5. **Capture** complete, unedited stdout+stderr for every run.
6. **Restore / clean up**: kill only the spawned PIDs, remove only the private scratch dir; the source tree is never modified.
7. **Verify integrity**: re-check `git status` — only this document is added.

### 1.1 Baseline (OBSERVED)

```console
$ git rev-parse --abbrev-ref HEAD
blitzy-a0ab8c39-b9ff-4c46-9691-2514989317d0
$ git rev-parse HEAD
722ca9e9e5ed9d9b7f457e1049516299ff862652
$ git status --porcelain
(exit=0)
$ git diff --stat
(exit=0)
$ git log --author=agent@blitzy.com --oneline -1
722ca9e9e docs: add runtime-observed investigation of kitty shaping/fallback, cell metrics, and GPU atlas init
```

An empty `git status --porcelain` and empty `git diff --stat` confirm the tree was clean before observation. The final integrity check (only this `.md` added; no source file modified) is shown in §8.4.

---

## 2. Canonical environment and infrastructure (OBSERVED)

### 2.1 Container identity

```console
$ cat /etc/os-release        # first 2 lines
PRETTY_NAME="Ubuntu 25.10"
NAME="Ubuntu"
$ uname -a
Linux reverse-code-generator-ed2699d3-g28x9 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux
```

The canonical build/run container is the image named in the setup instructions, `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas`), a Linux/FontConfig/FreeType environment (the canonical platform for this investigation; macOS CoreText is documented for completeness only).

### 2.2 Toolchain

```console
$ python3 --version
Python 3.13.7
$ go version
go version go1.24.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ pkg-config --modversion harfbuzz fontconfig freetype2 gl
10.2.0
2.15.0
26.2.20
1.2
```

These satisfy the repository's declared dependency floors: `harfbuzz >= 1.5` [`setup.py:609`], `requires-python = ">=3.8"` [`pyproject.toml:2`], `go 1.22` [`go.mod:3`]; shaping is HarfBuzz 10.2.0, discovery/fallback is FontConfig 2.15.0, rasterization/metrics is FreeType (module 26.2.20 = runtime FreeType 2.13.3), and the GL stack is present.

### 2.3 Arabic-capable fonts (so fallback can trigger)

```console
$ fc-list :lang=ar family file | sort -u | wc -l
37
$ fc-list :lang=ar family | sort -u
Amiri
Amiri Quran
Amiri Quran Colored
DejaVu Sans
DejaVu Sans Mono
DejaVu Sans,DejaVu Sans Condensed
KacstArt
KacstBook
KacstDecorative
KacstDigital
KacstFarsi
KacstLetter
KacstNaskh
KacstOffice
KacstOne
KacstPen
KacstPoster
KacstQurn
KacstScreen
KacstTitle
KacstTitleL
Noto Kufi Arabic
Noto Naskh Arabic
Noto Nastaliq Urdu
Noto Sans Arabic
mry_KacstQurn
```

There are **37** Arabic-capable family+file entries (26 unique family names). Crucially, the default `monospace` face on this box — **DejaVu Sans Mono** — itself contains Arabic glyphs, which is why the *default* run shows **no** fallback line and why a font override (Nimbus Mono PS, which lacks Arabic) is used to force and capture a real Arabic fallback selection (§ Objective A / B).

### 2.4 Display stack, window manager, and readiness (OBSERVED)

Reaching atlas initialization at real startup requires an OpenGL context, and reaching the per-cell render path requires the OS window to be **mapped and exposed** — otherwise `should_os_window_be_rendered()` [`kitty/glfw.c:1812`] returns false (iconified / not-visible / occluded) and the render path never runs. A headless X server (Xvfb) plus a window manager (openbox) satisfies both. The harness (§4) allocates a **unique free display**, starts Xvfb and openbox, captures their exact PIDs, waits for readiness, and tears down only those PIDs. Representative bring-up (from a harness run):

```text
WORK=/tmp/kitty_obs.EoeD969T  (umask=0077)
DISPLAY=:200
XVFB_PID=135862
OPENBOX_PID=135863
X server ready on :200
```

### 2.5 OpenGL context and GL limits (OBSERVED — raw)

```console
$ glxinfo -B | grep -iE 'OpenGL (version|core profile version|renderer)'
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.25.10.2

$ glxinfo -l | grep -iE 'GL_MAX_TEXTURE_SIZE|GL_MAX_ARRAY_TEXTURE_LAYERS'
    GL_MAX_TEXTURE_SIZE = 16384
    GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
    GL_MAX_TEXTURE_SIZE = 16384
    GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
```

The context is OpenGL 4.5 Core (Mesa llvmpipe). The two limits **`GL_MAX_TEXTURE_SIZE = 16384`** and **`GL_MAX_ARRAY_TEXTURE_LAYERS = 2048`** are the real inputs to the atlas capacity math (§ Objective D). `gl_init` treats the absence of required capabilities such as `texture_storage` as fatal [`kitty/gl.c:67`]; the context above provides them.

---

## 3. Canonical build (OBSERVED)

The canonical build is the `Makefile` `all` target, which runs `python3 setup.py`:

```console
$ sed -n '12,14p' Makefile
all:
	python3 setup.py $(VVAL)
```

Exact build command and result (the **complete, unedited 344-line transcript is embedded verbatim in Appendix A**):

```console
$ python3 setup.py clean
[clean exit=0]
$ CI=true python3 setup.py 2>&1   # timing captured
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
[5/85] Compiling kitty/graphics.c ...
[6/85] Compiling kitty/child-monitor.c ...
[7/85] Compiling kitty/fonts.c ...
[8/85] Compiling kitty/shaders.c ...
# … 74 further compile/link lines (full transcript in Appendix A) …
[BUILD_EXIT=0] wall=65s
```

The transcript begins with `Disabling building of wayland backend` (expected — the X11 backend is used on this box), compiles all 85 C translation units (including `[7/85] kitty/fonts.c`, `[8/85] kitty/shaders.c`, `[36/85] kitty/launcher/main.c`, `[85/85] kitty/gl-wrapper.c`), links `fast_data_types`, the GLFW X11 backend, the rsync helper and the launcher (`[4/4] Linking launcher`), then builds the Go `kitten`. It exits 0. The produced launcher:

```console
$ ls -l kitty/launcher/kitty
-rwxr-xr-x 1 root root 40384 Jul 13 18:15 kitty/launcher/kitty
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4a693e4304285476522c8ac6a4eef4babf9072b7, for GNU/Linux 3.2.0, not stripped
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The build modifies **no tracked source file** (`git diff --stat` remained empty afterward; the launcher and compiled extensions are git-ignored build artifacts).

---

## 4. The secure, self-contained observation harness

Per the reproducibility and safety requirements, the harness below is **fully embedded** (nothing opaque or deleted), is **secure** (`umask 077` + a unique `mktemp -d` 0700 scratch dir; no predictable shared `/tmp` paths — mitigating CWE-377 / CWE-59), is **collision-safe** (scans for a free X display; captures and kills only the exact PIDs it spawns), and uses **`set -euo pipefail`** so no failure is masked and no primary evidence is filtered. It is run from the repository root.

```bash
#!/usr/bin/env bash
# ============================================================================
# Canonical, self-contained, secure observation harness for kitty startup
# shaping / fallback / metrics / GPU-atlas investigation.
#
# Security & safety (addresses CWE-377/CWE-59 and shared-workspace hazards):
#   * umask 077                       -> all created files are private (0600/0700)
#   * WORK=$(mktemp -d ...)           -> unique, unpredictable 0700 scratch dir
#   * unique free DISPLAY (scanned)   -> no collision with parallel agents
#   * Xvfb + openbox PIDs captured    -> narrow teardown kills ONLY those PIDs
#   * set -euo pipefail               -> no masked failures; no evidence filtering
#   * embedded feed scripts (heredoc) -> fully auditable; nothing opaque/deleted
# Everything lives inside $WORK and is removed at the end; the source tree is
# never touched. Run it from the repository root.
# ============================================================================
set -euo pipefail
umask 077

REPO="$(pwd)"
LAUNCHER="$REPO/kitty/launcher/kitty"
[ -x "$LAUNCHER" ] || { echo "FATAL: launcher not built at $LAUNCHER (run: CI=true python3 setup.py)"; exit 1; }

WORK="$(mktemp -d /tmp/kitty_obs.XXXXXXXX)"
echo "WORK=$WORK  (umask=$(umask))"

# ---- pick a free X display -------------------------------------------------
DISP=200
while [ -e "/tmp/.X11-unix/X$DISP" ]; do DISP=$((DISP+1)); done
echo "DISPLAY=:$DISP"

# ---- start Xvfb (headless GL via llvmpipe) + capture PID --------------------
Xvfb ":$DISP" -screen 0 1280x1024x24 -ac +extension GLX +render >"$WORK/xvfb.log" 2>&1 &
XVFB_PID=$!
echo "XVFB_PID=$XVFB_PID"
export DISPLAY=":$DISP"

# ---- start a window manager so the OS window maps & is exposed --------------
# (without a WM the window is 'occluded' and should_os_window_be_rendered()
#  returns false, so the real render path never runs)
openbox >"$WORK/openbox.log" 2>&1 &
OPENBOX_PID=$!
echo "OPENBOX_PID=$OPENBOX_PID"

# ---- narrow teardown: kill ONLY the PIDs we spawned, remove ONLY $WORK ------
cleanup() {
  st=$?
  kill "$OPENBOX_PID" 2>/dev/null || true
  kill "$XVFB_PID"    2>/dev/null || true
  rm -rf "$WORK"
  echo "[cleanup done; killed openbox=$OPENBOX_PID xvfb=$XVFB_PID; removed $WORK; exit=$st]"
}
trap cleanup EXIT

# wait for the X server to accept connections
for _ in $(seq 1 50); do
  if DISPLAY=":$DISP" xdpyinfo >/dev/null 2>&1; then break; fi
  sleep 0.1
done
echo "X server ready on :$DISP"

# ---- embedded feed scripts (verbatim, auditable) ---------------------------
# Mixed Arabic (RTL) + English (LTR) + combining diacritics.
cat > "$WORK/feed_default.sh" <<'FEED'
#!/bin/sh
# 'Hello <ARABIC: mrhba> World' + a combining-mark line (base + U+0301 + U+0323)
printf 'Hello \331\205\330\261\330\255\330\250\330\247 World\n'
printf 'combine: o\314\201\314\243  e\314\201  a\314\210  n\314\203\n'
sleep 6
FEED
chmod +x "$WORK/feed_default.sh"

# Same input, but primary font forced to a face lacking Arabic (Nimbus Mono PS)
# so a real Arabic fallback selection is triggered and logged.

run_launcher() {  # $1=label  $2=logfile  (rest)=extra -o overrides + feed
  local label="$1"; shift
  local log="$1"; shift
  echo "=== RUN [$label] ==="
  timeout 25 "$LAUNCHER" --config NONE --debug-font-fallback --debug-rendering \
      -o confirm_os_window_close=0 "$@" >"$log" 2>&1 &
  local kpid=$!
  wait "$kpid"; echo "[$label exit=$?]"
}

# ---- default run (DejaVu covers Arabic -> NO fallback) x2 ------------------
run_launcher "default_run1" "$WORK/default_run1.log" "$WORK/feed_default.sh"
run_launcher "default_run2" "$WORK/default_run2.log" "$WORK/feed_default.sh"

# ---- forced Arabic-fallback run (Nimbus lacks Arabic) x2 -------------------
run_launcher "arabic_run1" "$WORK/arabic_run1.log" -o font_family="Nimbus Mono PS" "$WORK/feed_default.sh"
run_launcher "arabic_run2" "$WORK/arabic_run2.log" -o font_family="Nimbus Mono PS" "$WORK/feed_default.sh"

# ---- stability: normalize monotonic timestamps [<float>] -> [T], then diff --
norm() { sed -E 's/^\[[0-9]+\.[0-9]+\]/[T]/' "$1"; }
for pair in default arabic; do
  norm "$WORK/${pair}_run1.log" > "$WORK/${pair}_run1.norm"
  norm "$WORK/${pair}_run2.log" > "$WORK/${pair}_run2.norm"
  if diff -u "$WORK/${pair}_run1.norm" "$WORK/${pair}_run2.norm" >/dev/null; then
    echo "STABILITY[$pair]: run1==run2 after timestamp normalization (0 diff) -> STABLE"
  else
    echo "STABILITY[$pair]: DIFFERENCES:"; diff -u "$WORK/${pair}_run1.norm" "$WORK/${pair}_run2.norm"
  fi
done

echo "=== default_run1.log (complete, unedited) ==="; cat "$WORK/default_run1.log"
echo "=== arabic_run1.log (complete, unedited) ===";  cat "$WORK/arabic_run1.log"
# trap runs cleanup on exit
```

### 4.1 Complete, unedited harness output (OBSERVED)

```text
WORK=/tmp/kitty_obs.EoeD969T  (umask=0077)
DISPLAY=:200
XVFB_PID=135862
OPENBOX_PID=135863
X server ready on :200
=== RUN [default_run1] ===
[default_run1 exit=0]
=== RUN [default_run2] ===
[default_run2 exit=0]
=== RUN [arabic_run1] ===
[arabic_run1 exit=0]
=== RUN [arabic_run2] ===
[arabic_run2 exit=0]
STABILITY[default]: run1==run2 after timestamp normalization (0 diff) -> STABLE
STABILITY[arabic]: run1==run2 after timestamp normalization (0 diff) -> STABLE
=== default_run1.log (complete, unedited) ===
[0.154] OS Window created
[0.163] Failed to open systemd user bus with error: Connection refused
[0.167] Child launched
[0.167] Text fonts:
[0.167]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.167]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.167]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.167]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.126] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
=== arabic_run1.log (complete, unedited) ===
[0.160] OS Window created
[0.169] Failed to open systemd user bus with error: Connection refused
[0.172] Child launched
[0.173] Text fonts:
[0.173]   Normal: NimbusMonoPS-Regular: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1:0
[0.173]   Bold: NimbusMonoPS-Bold: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-Bold.otf:0
[0.173]   Italic: NimbusMonoPS-Italic: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Italic.t1:0
[0.173]   Bold-Italic: NimbusMonoPS-BoldItalic: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-BoldItalic.otf:0
[0.196] U+645 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.197] U+631 using previous fallback font at index: 0
[0.197] U+62d using previous fallback font at index: 0
[0.198] U+628 using previous fallback font at index: 0
[0.198] U+627 using previous fallback font at index: 0
[0.199] U+6f U+301 U+323 using previous fallback font at index: 0
[0.129] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[cleanup done; killed openbox=135863 xvfb=135862; removed /tmp/kitty_obs.EoeD969T; exit=0]
```

The trap-driven `cleanup` line at the end is the **narrow teardown proof**: only the two spawned PIDs are killed and only the private `$WORK` directory is removed.

The literal input fed to the terminal is `Hello مرحبا World` (English + Arabic **مرحبا** = U+0645 U+0631 U+062D U+0628 U+0627, "marhaba") plus a combining-mark line (`o` + U+0301 + U+0323, etc.). Byte-level confirmation of the Arabic input:

```console
$ printf 'Hello \331\205\330\261\330\255\330\250\330\247 World\n' | od -An -tx1
 48 65 6c 6c 6f 20 d9 85 d8 b1 d8 ad d8 a8 d8 a7
 20 57 6f 72 6c 64 0a
# d9 85=U+0645 MEEM, d8 b1=U+0631 REH, d8 ad=U+062D HAH, d8 a8=U+0628 BEH, d8 a7=U+0627 ALEF
```

---

## 5. Canonical launcher invocation

The single canonical invocation used throughout is:

```console
$ ./kitty/launcher/kitty --config NONE --debug-font-fallback --debug-rendering \
      -o confirm_os_window_close=0 [ -o <override> ] <feed-script>
```

- `--config NONE` guarantees a **default** configuration (no user `kitty.conf` is read), so reported defaults are the real product defaults.
- `--debug-font-fallback` enables the font-dump and per-cell fallback diagnostics [`kitty/cli.py:1002`]; `--debug-rendering` enables the GL banner and rendering diagnostics [`kitty/cli.py:989`]. The alternate spelling `--debug-gl` is exercised in §Objective D and produces the identical GL banner.
- `-o <key>=<value>` applies a **disclosed** override (used only to force the Arabic fallback path and to exercise modifier variants); every override is stated next to its output.
- Both stdout and stderr are captured (`> log 2>&1`).

The remainder of this document walks the four objectives, each backed by the OBSERVED output above (and additional runs embedded in-line), with SOURCE-DERIVED and NON-CANONICAL-CORROBORATION material clearly labelled.

---

## 6. Objective A — Shaping / layout configuration for complex Unicode + startup font fallback

kitty shapes each *run* of same-font cells with HarfBuzz. The entry chain is `render_line` → `render_run` [`kitty/fonts.c:1266`] → `shape_run` [`:1152`] → `shape` [`:786`], and `shape` ultimately calls `hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features)` [`kitty/fonts.c:813`]. The three complex-Unicode concerns — **ligatures**, **bidi**, and **combining diacritics** — are handled as follows.

### 6.A.1 Ligatures — the HarfBuzz **negative-feature sentinel** model

**This is the crux of ligature configuration, and it is the opposite of "CALT is always on."** kitty does not *enable* OpenType features to get ligatures; it relies on HarfBuzz's **defaults** (which already turn `liga` and `calt` on) and uses a small table of **negative** (disabling) features as overrides and as a drop-on-shape *sentinel*.

**Step 1 — the feature table is three *negative* features.** At font-subsystem init, `init_fonts` builds the global `hb_features[3]` table [`kitty/fonts.c:42`] (indexed by `enum { LIGA_FEATURE, DLIG_FEATURE, CALT_FEATURE }` [`:45`]) using a macro that parses **disabling** strings:

```c
// kitty/fonts.c:1750-1758
#define create_feature(feature, where) {\
    if (!hb_feature_from_string(feature, sizeof(feature) - 1, &hb_features[where])) { ... }}
    create_feature("-liga", LIGA_FEATURE);   // :1755  -> DISABLE standard ligatures
    create_feature("-dlig", DLIG_FEATURE);   // :1756  -> DISABLE discretionary ligatures
    create_feature("-calt", CALT_FEATURE);   // :1757  -> DISABLE contextual alternates
#undef create_feature
```

All three are `-liga`, `-dlig`, `-calt` (the leading `-` means *off*). There is no positive/enabling feature anywhere.

**Step 2 — each face gets a per-face feature list ending in the `-calt` sentinel.** `init_font` [`kitty/fonts.c:293-327`] assembles `f->ffs_hb_features` for the face:

- If the user set `font_features` for this PostScript name, the list is the parsed user features **plus a trailing `-calt`** [`:304-314`] (`memcpy(... &hb_features[CALT_FEATURE] ...)` [`:314`]).
- Otherwise (the default case) [`:318`]: if the PostScript name begins with `"NimbusMonoPS-"` [`:321`], the list is `[-liga, -dlig, -calt]` [`:322-325`]; for every other face it is just `[-calt]` [`:325`].

So in the **default** configuration every normal face ends up with exactly one feature, `-calt`, and Nimbus Mono PS with three, `[-liga, -dlig, -calt]`.

**Step 3 — normal shaping *drops* the trailing `-calt`; disabling ligatures *keeps* it.** In `shape`:

```c
// kitty/fonts.c:811-813
size_t num_features = fobj->num_ffs_hb_features;
if (num_features && !disable_ligature) num_features--;  // the last feature is always -calt
hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features);
```

`render_run` passes `disable_ligature = (disable_ligature_strategy == DISABLE_LIGATURES_ALWAYS)` [`kitty/fonts.c:1269`]. Therefore:

| Face / mode | `ffs_hb_features` | `num_features` passed to `hb_shape` | Net effect |
|-------------|-------------------|-------------------------------------|------------|
| Normal face, ligatures on (default) | `[-calt]` (len 1) | `1-1 = 0` → **no features** | HarfBuzz defaults apply → `liga`/`calt` **ON** → ligatures render |
| Nimbus Mono PS, ligatures on (default) | `[-liga, -dlig, -calt]` (len 3) | `3-1 = 2` → **`[-liga, -dlig]`** | std + discretionary ligatures **disabled**; `calt` left on |
| User `font_features "X +liga …"` | `[<user…>, -calt]` | `len-1` → user features only | exactly the user's request; the `-calt` sentinel is dropped |
| Any face, `disable_ligatures=always` | `[…, -calt]` | **not** decremented → trailing `-calt` **kept** | `calt` **OFF** → no ligatures |

In short: **ligatures are enabled by omission** (relying on HarfBuzz defaults), the negative table is an override mechanism, and the trailing `-calt` is a sentinel that is normally dropped and only survives when the user asks to disable ligatures. Nimbus Mono PS renders without `liga`/`dlig` **because kitty explicitly passes `-liga -dlig` for it**, not because the font lacks the lookups.

This is consistent with kitty's own option docs: `disable_ligatures` concerns "programming ligatures, typically implemented using the `calt` OpenType feature", and for other ligatures one uses `font_features` [`kitty/options/definition.py:115-131`].

**OBSERVED (canonical) — ligatures render in the real launcher.** A Fira Code run (`-o font_family="Fira Code" -o font_size=28`) fed `== -> != >= <= === =~ |> ++ ::` / `www <=> |||` was captured as a screenshot of the mapped openbox-managed window and inspected. Every operator sequence renders as a single fused ligature glyph: `==` → one wide double-bar equals; `->` → a fused arrow (→); `!=` → ≠; `>=` → ≥; `<=` → ≤; `===` → a triple-bar glyph; `|>` → a right-pointing triangle; `<=>` → a long double-headed arrow. The startup font block for that run confirms the primary faces are Fira Code:

```text
[0.169] Text fonts:
[0.169]   Normal: FiraCode-Regular: /usr/share/fonts/truetype/firacode/FiraCode-Regular.ttf:0
[0.169]   Bold: FiraCode-SemiBold: /usr/share/fonts/truetype/firacode/FiraCode-SemiBold.ttf:0
[0.169]   Italic: FiraCode-Retina: /usr/share/fonts/truetype/firacode/FiraCode-Retina.ttf:0
[0.169]   Bold-Italic: FiraCode-SemiBold: /usr/share/fonts/truetype/firacode/FiraCode-SemiBold.ttf:0
```

**OBSERVED (canonical) — `disable_ligatures=always` suppresses them.** The identical Fira Code input with `-o disable_ligatures=always` was captured and inspected: every operator now renders as separate discrete glyphs (`==` is two equals signs, `->` is a dash then `>`, `!=` is `!` then `=`, `===` is three equals, etc.). This is the direct visual contrast that proves the `-calt` sentinel is *kept* (feature list not decremented) when ligatures are disabled, turning `calt` off. Its startup font block is again the four Fira Code faces (complete log in Appendix A.4).

**NON-CANONICAL CORROBORATION (test binding `shape_string`).** To expose the mechanism at the shaping-group level, `kitty.fonts.render.shape_string` (which uses the `setup_for_testing` bypass — labelled non-canonical) reports HarfBuzz output as per-group tuples `(num_cells, num_glyphs, first_glyph_index, glyph_indices…)`; the codepoints returned are **glyph indices**, i.e. font-specific glyph IDs, not Unicode scalar values:

```text
single '='   family=Fira Code -> groups[1]: cells=1,glyphs=1,ids=(1578,)
pair   '=='  family=Fira Code -> groups[1]: cells=2,glyphs=2,ids=(1649, 1387)
pair   '=='  family=monospace -> groups[2]: cells=1,glyphs=1,ids=(32,) | cells=1,glyphs=1,ids=(32,)
triple '===' family=Fira Code -> groups[1]: cells=3,glyphs=3,ids=(1649, 1649, 1388)
arrow  '->'  family=Fira Code -> groups[1]: cells=2,glyphs=2,ids=(1186, 1458)
arrow  '->'  family=monospace -> groups[2]: cells=1,glyphs=1,ids=(16,) | cells=1,glyphs=1,ids=(33,)
```

Reading this carefully: a lone `=` in Fira Code is glyph **1578**; the pair `==` is shaped into **one 2-cell group** whose glyphs are **1649, 1387** — *neither equal to 1578*, i.e. HarfBuzz's `calt` substituted both cells into ligature-component glyphs. In DejaVu (monospace) the same `==` produces **two separate single-cell groups**, both the ordinary `=` glyph 32, un-substituted — because DejaVu has no such `calt` lookups. This corroborates, at the glyph level, that the fused rendering in the screenshots comes from HarfBuzz's default `calt` (which kitty leaves on by dropping the `-calt` sentinel) and is font-dependent.

### 6.A.2 Bidirectional (bidi) text — **HONEST NEGATIVE: kitty has no bidi reordering engine**

kitty does **not** implement the Unicode Bidirectional Algorithm. There is no reordering pass; the only related control is the boolean option `force_ltr`, declared with default `'no'` [`kitty/options/definition.py:64`], whose own long-text opens with this statement (reproduced verbatim through its first sentence; later sentences are abridged, with each `…` marking an elision):

```text
kitty/options/definition.py:66-82  (force_ltr long_text — opening sentence verbatim; … marks abridged elisions)
"kitty does not support BIDI (bidirectional text), however, for RTL scripts,
 words are automatically displayed in RTL. That is to say, in an RTL script, the
 words "HELLO WORLD" display in kitty as "WORLD HELLO" … assuming the Hebrew word
 ירושלים, selecting the character that on the screen appears to be ם actually
 writes into the selection buffer the character י … this option can be used with
 the command line program GNU FriBidi … to get BIDI support, because it will force
 kitty to always treat the text as LTR, which FriBidi expects for terminals."
```

So the authoritative negative is spelled out by the source itself: <q>kitty does not support BIDI</q> [`kitty/options/definition.py:67`]. The default resolved value is `force_ltr = no` [`:64`] (the `yes` path is exercised as a variant in §Coverage). RTL scripts are still *shaped* (glyph selection and cursive joining) by HarfBuzz — kitty simply performs no logical→visual reordering. The changelog frames the practical result: <q>Partial fix for rendering Right-to-left languages like Arabic</q> [`docs/changelog.rst:3540-3541`].

**OBSERVED (canonical).** In both the default and forced-fallback runs, the Arabic word **مرحبا** is shaped and rendered with correct cursive presentation forms (the captured screenshots show the five letters joined right-to-left as connected glyphs, not isolated boxes), confirming HarfBuzz shaping runs even though there is no bidi reorder step.

**INFERRED (source, no runtime log).** For RTL runs HarfBuzz emits glyphs whose cluster values *decrease* across the run; kitty's code comments document this ("RTL languages like Arabic have decreasing cluster numbers") at [`kitty/fonts.c:997`] and [`kitty/fonts.c:1079`]. No debug line prints cluster numbers, so this is labelled inferred from the source, corroborated by the correctly-joined visual output.

### 6.A.3 Combining diacritics — one grapheme cluster maps to one cell

kitty stores a base codepoint plus up to a fixed number of combining marks per cell in `CPUCell.cc_idx[]`, and resolves each mark index back to a codepoint with `codepoint_for_mark`. Two functions drive correctness:

- `has_cell_text(face, cell)` [`kitty/fonts.c:435`] verifies a face covers the whole grapheme: it checks the base with `face_has_codepoint(face, cell->ch)` [`:436`], then each combining mark via `codepoint_for_mark(cell->cc_idx[i])` [`:440`], with an `hb_unicode_compose` precomposed-form check [`:447`].
- `output_cell_fallback_data` [`kitty/fonts.c:457`] iterates the same `cc_idx[]` marks (`debug("U+%x ", codepoint_for_mark(cell->cc_idx[i]))` [`:460`]) when logging a fallback for a combined cell.

**OBSERVED (canonical).** Feeding `o` + U+0301 (combining acute) + U+0323 (combining dot-below) produced a single per-cell diagnostic line naming **all three** codepoints together — proving they were coalesced into one cell before font selection:

```text
[0.199] U+6f U+301 U+323 using previous fallback font at index: 0
```

(from the forced-fallback run; `U+6f`=`o`, `U+301`, `U+323`). The captured default-run screenshot likewise shows `é`, `ä`, `ñ` rendered as single composed accented glyphs.

**NON-CANONICAL CORROBORATION (`shape_string`).** The same `o`+U+0301+U+0323 shaped as `num_cells=1, num_glyphs=2` (`ids=(1548, 649)`) — one grapheme cluster occupying one cell, rendered with a base glyph plus one combining glyph.

### 6.A.4 Font fallback during startup

When the primary face lacks a glyph for a cell, kitty selects a fallback face. The path is `font_for_cell` → `fallback_font` [`kitty/fonts.c:520`] → (on cache miss) `load_fallback_font` [`kitty/fonts.c:481`] → native `create_fallback_face` [`kitty/fontconfig.c:463`], which uses FontConfig's matcher (`fallback_font` in [`kitty/fontconfig.c:444`]).

**Cache first.** `fallback_font` builds a per-cell key (a style byte + the cell's UTF-8 text) and consults the `fallback_font_map` hash: `HASH_FIND_STR(fg->fallback_font_map, cell_text, s)` [`kitty/fonts.c:530`]; a hit returns the cached index immediately [`:532`]; a miss calls `load_fallback_font` and records the result [`:534-541`].

**Bound (exact threshold).** `load_fallback_font` begins with `if (fg->fallback_fonts_count > 100) { log_error("Too many fallback fonts"); return MISSING_FONT; }` [`kitty/fonts.c:482`]. This is a strict `>` — the count may **reach 101** fallback fonts (when count is 100, `100 > 100` is false and the 101st is appended); only once the count *exceeds* 100 is a further face rejected. (The Python-level test binding raises the analogous "Too many fallback fonts" at [`kitty/fonts.c:1694`].)

**Match → verify → append/reuse.** `create_fallback_face` [`:489`] asks the OS/FontConfig for a face. If the debug flag is on, `output_cell_fallback_data` logs the choice [`:492`]. If the returned object is a `PyLong` it is an **index into the existing chain** → reuse (this is the "using previous fallback font at index: N" line) [`:493`]. Otherwise the new face is appended and **glyph-verified** with `has_cell_text` [`:501`]; if it does not actually cover the cell, kitty logs `"… but it does not actually contain glyphs for that text"` [`:509`], drops it (`del_font`), and returns `MISSING_FONT` [`:503-511`]. On success it bumps `fallback_fonts_count` and `fonts_count` [`:513-514`].

**OBSERVED (canonical) — default run: NO fallback (alternate/primary path).** With the default `monospace` face (DejaVu Sans Mono, which *does* contain Arabic + the combining marks), the whole line is covered by the primary face and **no** fallback line is emitted. This is the honest default observation (complete `default_run1.log` in §4.1): only the four-face `Text fonts:` block and the GL banner appear.

**OBSERVED (canonical) — forced Arabic fallback (disclosed override).** Because the default face covers Arabic, a real Arabic fallback is forced with the disclosed override `-o font_family="Nimbus Mono PS"` (Nimbus Mono PS has no Arabic glyphs). The launcher then selects and logs the fallback face for the Arabic cells, giving the exact **family / PostScript name / path / face-index** required:

```text
[0.173] Text fonts:
[0.173]   Normal: NimbusMonoPS-Regular: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1:0
[0.173]   Bold: NimbusMonoPS-Bold: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-Bold.otf:0
[0.173]   Italic: NimbusMonoPS-Italic: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Italic.t1:0
[0.173]   Bold-Italic: NimbusMonoPS-BoldItalic: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-BoldItalic.otf:0
[0.196] U+645 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.197] U+631 using previous fallback font at index: 0
[0.197] U+62d using previous fallback font at index: 0
[0.198] U+628 using previous fallback font at index: 0
[0.198] U+627 using previous fallback font at index: 0
```

The first Arabic codepoint **U+645 (MEEM)** triggers a fresh fallback selection: FontConfig returns **DejaVu Sans Mono** (`ps_name=DejaVuSansMono`, `path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf`, `ttc_index=0`, `scalable=True`, `color=False`). The remaining Arabic letters **U+631, U+62D, U+628, U+627** resolve to the **same** face and print `using previous fallback font at index: 0` — the cache/reuse path [`fonts.c:493`, `:530`]. The forced primary faces are the four Nimbus Mono PS entries, including the `.t1` (Type-1) Regular/Italic and `.otf` Bold/BoldItalic — the very PostScript-name prefix `NimbusMonoPS-` that also activates the `[-liga,-dlig]` branch of §6.A.1. The captured screenshot of this run shows the English words in Nimbus letterforms while the Arabic renders through the DejaVu fallback (same joined Arabic glyphs as the default run) — the visual confirmation of the logged fallback.

**OBSERVED (canonical) — chain build then reuse (CJK).** Feeding `中文` shows the same build-then-reuse pattern with a *different* fallback face, confirming chain construction is general:

```text
[0.190] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.191] U+6587 using previous fallback font at index: 0
```

**U+4E2D** builds a new fallback (**Noto Sans CJK JP**, `NotoSansCJKjp-Regular`, `NotoSansCJK-Regular.ttc:0`); **U+6587** reuses `index: 0`.

**Labelling note (evidence accuracy).** The `Face(...)` line proves the *selected fallback face and its file/index*; it does **not** by itself prove Arabic contextual-form correctness — that is shown separately by the rendered screenshot (joined cursive forms). Glyph indices from `shape_string` are labelled font-specific glyph IDs, not Unicode values, and are corroboration only.


---

## 7. Objective B — Verbose-logging startup diagnostics for mixed Arabic (RTL) + English (LTR)

### 7.B.1 The complete startup chain and *when* each thing happens

The launcher reaches the font diagnostics through this exact chain (every link verified in source):

```text
kitty/launcher/kitty  (native launcher)
  └─ Py_RunMain() with run-data key "kitty_main"        [kitty/launcher/main.c:168 (kitty_main key) → :216 (Py_RunMain call)]
      └─ ./__main__.py :  from kitty.entry_points import main; main()   [/__main__.py:5-7]
          └─ kitty.entry_points.main()                  [kitty/entry_points.py:183]
              └─ (no '+' subcommand) from kitty.main import main as kitty_main; kitty_main()  [:194-195]
                  └─ kitty.main.main()                  [kitty/main.py:524]
                      └─ run_app(opts, cli_opts, …)     [kitty/main.py:518]  (run_app is an AppRunner)
                          └─ AppRunner.__call__(…)       [kitty/main.py:247]
                              1. set_options(opts, is_wayland(), args.debug_rendering, args.debug_font_fallback)   [:249]
                              2. set_font_family(opts)   [:251]   ← font DESCRIPTORS resolved, BEFORE any window
                              3. _run_app(opts, args, …) [:252]
                                   a. create_os_window(… load_all_shaders …)  [:221] ← GL init + atlas alloc + prerender + GL banner
                                   b. boss = Boss(…)      [:226]
                                   c. boss.start(…)       [:227]
                                   d. if args.debug_font_fallback: dump_font_debug()  [:228-229] ← font DUMP, AFTER window+prerender
                                   e. boss.child_monitor.main_loop()  [:234] ← per-cell shaping / fallback / glyph upload
                              4. finally: set_options(None) [:254]; free_font_data()  [:255]
```

Three distinct timing points must not be conflated (this is the subtlety the diagnostics expose):

1. **Descriptor resolution — *before* the window.** `set_font_family(opts)` [`kitty/main.py:251`, impl `kitty/fonts/render.py:173`] resolves the primary `font_family`/`bold`/`italic`/`bold-italic` faces from the options *before* `_run_app` creates any OS window. This is where the configured faces are chosen.
2. **GL init, atlas allocation, prerender, GL banner — during `create_os_window`.** [`kitty/main.py:221`] The GL context is created, the sprite atlas is allocated, the decoration/cursor sprites are prerendered, and the `--debug-rendering` GL banner is printed — all *before* any user text is shaped.
3. **The font *dump* — *after* window+prerender.** `dump_font_debug()` runs only if `--debug-font-fallback` was passed [`kitty/main.py:228-229`] and executes *after* the window and prerender, but still *before* the first user keystroke is shaped in `main_loop` [`:234`]. It is therefore a faithful report of the *resolved configuration* the terminal will use, emitted before any user text is rendered — but it is **not** evidence that "no glyph work has occurred," because prerender already uploaded the decoration sprites in step (a).

`dump_font_debug` itself is small and prints via `log_error` (stderr), not `timed_debug_print`:

```python
# kitty/fonts/render.py:161-171
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

Each face line is formatted by `identify_for_debug` as `<PostScript name>: <path>:<face index>` [`kitty/freetype.c:738`].

### 7.B.2 Font families in the startup dump (OBSERVED)

From the canonical **default** run (`default_run1.log`, §4.1), the complete `Text fonts:` block — four faces, each `PostScriptName: path:index`:

```text
[0.167] Text fonts:
[0.167]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.167]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.167]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.167]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

Because the default `font_family = monospace` resolves to **DejaVu Sans Mono** here, all four faces are DejaVu Sans Mono weights (Book/Bold/Oblique/BoldOblique), each at face index `:0`. The forced-fallback run instead resolves the four **Nimbus Mono PS** faces (see §6.A.4), and the Fira Code run resolves the four **Fira Code** faces (§6.A.1) — demonstrating the block faithfully reflects the resolved primary configuration.

**`Symbol map fonts:` block (OBSERVED — default absent, override present).** `dump_font_debug` prints a `Symbol map fonts:` block **only if** `cf['symbol']` is non-empty [`kitty/fonts/render.py:167-171`]. The `+symbol_map` option's documented value carries **`add_to_default=False`** [`kitty/options/definition.py:86`], so the *real* default map is **empty** — hence no such block appears in the default run (confirmed: absent in `default_run1/2.log`). Feeding a disclosed override `-o symbol_map="U+0645 Amiri"` makes the block appear:

```text
[0.177] Symbol map fonts:
[0.177]   Amiri-Regular: /usr/share/fonts/opentype/fonts-hosny-amiri/Amiri-Regular.ttf:0
```

i.e. U+0645 is mapped to **Amiri** (`Amiri-Regular`, `.../fonts-hosny-amiri/Amiri-Regular.ttf:0`).

### 7.B.3 The nine named configuration values, resolved before rendering (OBSERVED / SOURCE-DERIVED)

The task names nine keys. Each is reported with its **default**, **source anchor**, **resolved value under `--config NONE`**, and a **modifier observation** (what a disclosed override does at runtime). The primary-face resolved values are corroborated by the `Text fonts:` block above; `font_size` units and the boolean/enum defaults are grounded in the option definitions.

| Key | Default | Source anchor | Resolved (`--config NONE`) | Modifier observation |
|-----|---------|---------------|----------------------------|----------------------|
| `font_family` | `monospace` | `definition.py:35` | **DejaVu Sans Mono** (`Normal: DejaVuSansMono …ttf:0`) | `-o font_family="Nimbus Mono PS"` → four NimbusMonoPS faces + Arabic fallback (§6.A.4); `-o font_family="Fira Code"` → four Fira Code faces + ligatures (§6.A.1) |
| `bold_font` | `auto` | `definition.py:53` | **DejaVuSansMono-Bold** (derived) | resolves to the primary family's bold weight; overridable to any face |
| `italic_font` | `auto` | `definition.py:55` | **DejaVuSansMono-Oblique** (derived) | resolves to the primary family's italic/oblique weight |
| `bold_italic_font` | `auto` | `definition.py:57` | **DejaVuSansMono-BoldOblique** (derived) | resolves to the primary family's bold-italic weight |
| `font_size` | `11.0` | `definition.py:59`; units **points** per `long_text='Font size (in pts).'` `definition.py:61` | **11.0 pt** | drives cell metrics (§8) and atlas cell dimensions (§9); e.g. `-o font_size=28` used for the ligature/Arabic screenshots |
| `symbol_map` | **empty** (documented value has `add_to_default=False`) | `definition.py:84-86` | **{} (empty)** → no `Symbol map fonts:` block | `-o symbol_map="U+0645 Amiri"` → block appears mapping U+0645→Amiri (§7.B.2) |
| `disable_ligatures` | `never` | `definition.py:115` | **never** (ligatures rendered) | `-o disable_ligatures=always` → keeps `-calt`, ligatures suppressed (§6.A.1, live screenshot contrast) |
| `font_features` | `none` → resolves to an empty mapping **`{}`** | `definition.py:135` | **{} (empty)** — no per-face features configured | `-o font_features "TT2020StyleB-Regular -liga +calt"` (example `definition.py:182`) merges into that face's `ffs_hb_features` before the `-calt` sentinel (§6.A.1, step 2) |
| `force_ltr` | `no` | `definition.py:64` | **no** (kitty auto-displays RTL words RTL; no bidi reorder) | `-o force_ltr=yes` accepted at startup (variant §Coverage); forces LTR treatment for use with GNU FriBidi |

Notes grounding the two commonly-missed entries:
- **`font_size` units are points.** The option's own long-text is literally `Font size (in pts).` [`kitty/options/definition.py:61`]; the resolved default is `11.0` pt [`:59`].
- **`font_features` resolves to `{}` by default.** The default string `none` [`kitty/options/definition.py:135`] means "no per-face OpenType features configured," yielding an empty mapping; consequently `init_font` takes its *default* per-face branch (`[-calt]`, or `[-liga,-dlig,-calt]` for Nimbus) rather than the user-features branch [`kitty/fonts.c:300-325`]. This is why the coverage claim for `font_features` is now backed by an explicit resolved value rather than a bare checkmark.


---

## 8. Objective C — Screen-grid / shaping-subsystem init: cell metrics, baseline, decoration alignment

### 8.C.1 How the metrics are computed, and how they were observed

As the screen grid initializes, `calc_cell_metrics(FontGroup *fg)` [`kitty/fonts.c:373`] calls `cell_metrics(...)` on the medium face [`:375`] and then applies any user `modify_font` adjustments via `adjust_metric(...)` for `cell_width`, `cell_height` [`kitty/fonts.c:379-380`], `underline_*`, `strikethrough_*`, and `baseline` [`kitty/fonts.c:399`], clamped by `MIN_WIDTH=2 / MIN_HEIGHT=4 / MAX_DIM=1000` [`kitty/fonts.c:381-395`]. The core computation is in FreeType's `cell_metrics` [`kitty/freetype.c:387-405`]:

```c
// kitty/freetype.c:387-405 (verbatim)
cell_metrics(PyObject *s, unsigned int* cell_width, unsigned int* cell_height, unsigned int* baseline,
             unsigned int* underline_position, unsigned int* underline_thickness,
             unsigned int* strikethrough_position, unsigned int* strikethrough_thickness) {
    Face *self = (Face*)s;
    *cell_width = calc_cell_width(self);                                        // max ceil(horiAdvance/64), ASCII 32..127
    *cell_height = calc_cell_height(self, true);                               // px_y(height), unless '_' overflows
    *baseline = font_units_to_pixels_y(self, self->ascender);                  // ascender in px
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

where `font_units_to_pixels_y(x) = ceil(FT_MulFix(x, size->metrics.y_scale) / 64.0)` [`kitty/freetype.c:92-94`], `calc_cell_width` is the max `ceil(horiAdvance/64)` over ASCII 32..127 [`kitty/freetype.c:374-385`], and `calc_cell_height(for_metrics=true)` is `px_y(height)` unless the `_` glyph would overflow the box [`kitty/freetype.c:141-152`].

**Documented attempt to observe from the launcher (honest negative).** The launcher, even with `--debug-font-fallback --debug-rendering`, emits **no** cell-metric line. The *only* metric-adjacent debug print is the buggy-font underscore workaround inside `calc_cell_height`:

```c
// kitty/freetype.c:145-149
if (underscore_height > ans) {
    if (global_state.debug_font_fallback) printf(
        "Increasing cell height by %u pixels to work around buggy font that renders underscore outside the bounding box\n", underscore_height - ans);
    return underscore_height;
}
```

This fires only for a *buggy* font where `_` exceeds the computed cell height. For DejaVu Sans Mono / Nimbus / Fira at the canonical **96 DPI** it does **not** fire — confirmed by grepping every captured launcher log:

```console
$ grep -niE "Increasing cell height|underscore" /tmp/kitty_evidence/logs/*.log
(no output)  -> NOT PRESENT in any launcher log
```

Therefore, per the "persist, then honestly label" rule, cell metrics are reported as follows:

**OBSERVED (production binding) — `cell_width` / `cell_height`.** `create_test_font_group(size, dpi, dpi)` [`kitty/fonts.c:1699`] returns the real `fg->cell_width, fg->cell_height` (computed inside `font_group_for` → `calc_cell_metrics` [`kitty/fonts.c:1702`]) and hands them back via `Py_BuildValue("II", ...)` [`kitty/fonts.c:1704`]. This binding's substitutions (GPU-upload callback, sprite limits) are **not** on the metric-computation path, so the values are the genuine production metrics. Run through the launcher (`kitty +launch <probe>`), stable across two runs:

```text
family='monospace' size=11.0 dpi=96.0 -> cell_width=9 cell_height=18
family='monospace' size=11.0 dpi=72.0 -> cell_width=7 cell_height=14
family='Nimbus Mono PS' size=11.0 dpi=96.0 -> cell_width=9 cell_height=19
```

**SOURCE-DERIVED (validated) — the full metric set.** The remaining fields (`baseline`, `underline_position/thickness`, `strikethrough_position/thickness`) are not exposed by any binding, so they were computed by an independent C probe that links the **same system FreeType** and applies the `cell_metrics` formulas **verbatim** to the real `DejaVuSansMono.ttf` at 11 pt / 96 DPI. The probe is *validated* because its `cell_width`/`cell_height` reproduce the production-binding values **exactly** (9×18 at 96 DPI, 7×14 at 72 DPI), so its other outputs are trustworthy reproductions of what production `cell_metrics` computes:

```text
system FreeType 2.13.3
FONT /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf @ 11pt/96dpi
  cell_width=9 cell_height=18 (underscore_h=18, buggy=no)
  baseline=14
  underline_position=15 underline_thickness=1
  strikethrough_position=10 strikethrough_thickness=1 (os2_strike_pos=530 os2_strike_size=102)
  [raw design units] units_per_EM=2048 ascender=1901 descender=-483 height=2384 underline_position=-85 underline_thickness=90 y_scale=30048
```

**Canonical default metric set (DejaVu Sans Mono, `font_family=monospace`, `font_size=11.0` pt, 96 DPI, `modify_font` unset → no adjustment):**

| Metric | Value (px) | Derivation |
|--------|-----------:|------------|
| `cell_width` | **9** | `calc_cell_width` = max `ceil(horiAdvance/64)` over ASCII 32..127 [`freetype.c:374`] — OBSERVED via production binding |
| `cell_height` | **18** | `px_y(height)` [`freetype.c:142`]; underscore does not overflow at 96 DPI — OBSERVED via production binding |
| `baseline` | **14** | `px_y(ascender)` = `px_y(1901)` [`freetype.c:391`] |
| `underline_position` | **15** | `MIN(17, px_y(1901 − (−85)))` clamped to `cell_height−1` [`freetype.c:392`] |
| `underline_thickness` | **1** | `MAX(1, px_y(90))` [`freetype.c:393`] |
| `strikethrough_position` | **10** | OS/2 `yStrikeoutPosition=530 ≠ 0` → `MIN(17, px_y(1901−530))` [`freetype.c:396`] |
| `strikethrough_thickness` | **1** | OS/2 `yStrikeoutSize=102 > 0` → `MAX(1, px_y(102))` [`freetype.c:401`] |

Units are **pixels**; the source values are FreeType *design units* (`units_per_EM=2048`) scaled by `y_scale` at 11 pt / 96 DPI. All values are stable across ≥2 runs (§ Stability).

**Screenshot cross-check.** The captured 28-pt default render shows a uniform monospace grid whose cell width:height proportion (~1:2) matches the 9:18 metric, with a single baseline row of English text and correctly-seated diacritics — an independent visual confirmation of the cell geometry.

### 8.C.2 Underline — real, measured

Underline is a genuine, computed decoration. Its **position** (`underline_position = 15`) and **thickness** (`underline_thickness = 1`) come from `cell_metrics` [`freetype.c:392-393`] as above. kitty prerenders **five** distinct underline style sprites — `NUM_UNDERLINE_STYLES (5u)` [`kitty/data-types.h:213`] — via `prerender_function` (`range(1, NUM_UNDERLINE_STYLES + 1)` [`kitty/fonts/render.py:391`]), i.e. straight, double, curly, dotted, dashed. These are uploaded to the atlas at prerender (§9).

### 8.C.3 Strikethrough — real, measured

Strikethrough is likewise real. Its **position** (`strikethrough_position = 10`) derives from the OS/2 `yStrikeoutPosition` (530 design units → 10 px) [`freetype.c:396`], and its **thickness** (`strikethrough_thickness = 1`) from OS/2 `yStrikeoutSize` (102 → 1 px) [`freetype.c:401`]; when a font omits these, kitty falls back to `floor(baseline*0.65)` for the position [`:398`] and to the underline thickness [`:403`]. The strikethrough sprite is prerendered at index `STRIKE_SPRITE_INDEX = NUM_UNDERLINE_STYLES + 1` [`kitty/shaders.py:162`].

### 8.C.4 Overline — **HONEST NEGATIVE: no overline decoration exists**

There is **no overline text decoration** anywhere in the terminal core. Scoped search evidence:

```console
$ grep -rniE "overline" kitty/ | wc -l
0
$ grep -rniE "overline" glfw/
glfw/xkb-compat-shim.h:135:    { 0x047e, 0x203e }, /*                    overline ‾ OVERLINE */
```

The `kitty/` source tree contains **zero** occurrences of "overline". The single repository match is in the bundled windowing layer, `glfw/xkb-compat-shim.h:135`, and is an **XKB keyboard keysym** table entry (`0x047e → U+203E`, the *keyboard* keysym named OVERLINE) — completely unrelated to text-cell decoration. This is a definitive, evidence-backed negative: kitty draws blank, underline (5 styles), strikethrough, missing-glyph, and cursor sprites (§9), but **not** an overline. (The earlier document's environment-sensitive raw match *counts* are replaced here with the tightly-scoped `kitty/`-vs-`glfw/` evidence above, which is stable and meaningful.)


---

## 9. Objective D — GPU texture-atlas initialization at launch

### 9.D.1 The atlas data model

kitty's glyph atlas is a single `GL_TEXTURE_2D_ARRAY` texture per `FontGroup`, tracked by two cooperating structs:

- **`SpriteMap`** (GPU-side, in `kitty/shaders.c`) holds the texture id, the cell dimensions, the current write cursor `(x, y, z)`, the layout `(xnum, ynum)`, and the queried GL limits [`kitty/shaders.c:24-30`]:

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

- **`GPUSpriteTracker`** (CPU-side layout tracker, in `kitty/fonts.c`) [`kitty/fonts.c:35-38`] — note this is the correct location; it is **not** at lines 16-19 (a prior stale reference):

```c
// kitty/fonts.c:35-38
typedef struct {
    size_t max_y;
    unsigned int x, y, z, xnum, ynum;
} GPUSpriteTracker;
```

### 9.D.2 Capacity taxonomy — five distinct numbers, do not conflate them

This is the single most misreported area, so the five values are separated explicitly. All derive from the **real, observed** GL limits on the canonical llvmpipe context and the **observed** default cell size 9×18:

```console
$ glxinfo -B | grep -iE 'OpenGL (version|core profile version|renderer)'
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.25.10.2

$ glxinfo -l | grep -iE 'GL_MAX_TEXTURE_SIZE|GL_MAX_ARRAY_TEXTURE_LAYERS'
    GL_MAX_TEXTURE_SIZE = 16384
    GL_MAX_ARRAY_TEXTURE_LAYERS = 2048
```

| # | Quantity | Value | Where set / how derived |
|---|----------|------:|-------------------------|
| 1 | **Placeholder layout** (compile-time, before any GL query) | `xnum=1, ynum=1, 1 layer` (1 slot) | `NEW_SPRITE_MAP` [`kitty/shaders.c:31`] — a static initializer, *not* the real capacity |
| 2 | **Columns per layer** (`xnum`) | **1820** | `MIN(MAX(1, max_texture_size / cell_width), UINT16_MAX)` = `16384/9` = 1820 [`kitty/fonts.c:277`] |
| 3 | **Max rows per layer** (`max_y`) | **910** | `MIN(MAX(1, max_texture_size / cell_height), UINT16_MAX)` = `16384/18` = 910 [`kitty/fonts.c:278`] |
| 4 | **Slots per fully-grown layer** | **1,656,200** | `xnum × max_y` = `1820 × 910` |
| 5 | **Layer cap** & **total capacity** | **2048** layers → **3,391,897,600** slots | `max_array_len = MIN(0xfffu, layers)` = `MIN(4095, 2048)` = 2048 [`kitty/fonts.c:236-239`]; total = per-layer × 2048 |

Two more values worth separating from the above:

- **Initial *physical* allocation.** The first `glTexStorage3D` allocates `width = xnum × cell_width = 1820 × 9 = 16380` px wide, `height = ynum × cell_height = 1 × 18 = 18` px tall, `znum = z + 1 = 1` layer [`kitty/shaders.c:120-123`]. So the atlas is born as a **16380 × 18 × 1** immutable texture holding **1820 slots** (one row), and grows a row/layer at a time.
- **Occupancy immediately after prerender.** **11** slots are filled before any shaped text glyph is uploaded (§9.D.5).

`sprite_tracker_set_layout` resets the write cursor and computes `xnum`/`max_y`, with `ynum` starting at 1 [`kitty/fonts.c:276-281`]:

```c
// kitty/fonts.c:276-281
sprite_tracker_set_layout(GPUSpriteTracker *sprite_tracker, unsigned int cell_width, unsigned int cell_height) {
    sprite_tracker->xnum = MIN(MAX(1u, max_texture_size / cell_width), (size_t)UINT16_MAX);
    sprite_tracker->max_y = MIN(MAX(1u, max_texture_size / cell_height), (size_t)UINT16_MAX);
    sprite_tracker->ynum = 1;
    sprite_tracker->x = 0; sprite_tracker->y = 0; sprite_tracker->z = 0;
}
```

The write cursor advances via `do_increment` [`kitty/fonts.c:243`]: `x++`; on reaching `xnum` it wraps to the next row (`y++`, growing `ynum` up to `max_y`); on reaching `max_y` it wraps to the next layer (`z++`). This is the **copy-on-grow** model: each growth reallocates a larger immutable texture and copies the existing image forward.

### 9.D.3 GL limits are queried at real startup; storage is immutable sRGB

`alloc_sprite_map` queries the two limits **once** from the live GL context [`kitty/shaders.c:51-54`]:

```c
// kitty/shaders.c:51-54
alloc_sprite_map(unsigned int cell_width, unsigned int cell_height) {
    if (!max_texture_size) {
        glGetIntegerv(GL_MAX_TEXTURE_SIZE, &(max_texture_size));
        glGetIntegerv(GL_MAX_ARRAY_TEXTURE_LAYERS, &(max_array_texture_layers));
```

`sprite_tracker_set_limits` then clamps the usable layer count to `MIN(0xfffu, max_array_len_)` [`kitty/fonts.c:236-239`]. The texture itself is allocated **immutably** as `GL_SRGB8_ALPHA8` on a `GL_TEXTURE_2D_ARRAY` with `GL_NEAREST` filtering and `GL_CLAMP_TO_EDGE` wrapping, via `glTexStorage3D` [`kitty/shaders.c:110-123`]:

```c
// kitty/shaders.c:122-123
width = xnum * sprite_map->cell_width; height = ynum * sprite_map->cell_height;
glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, width, height, znum);
```

Because `glTexStorage3D` creates *immutable-format* storage, growth cannot resize in place — hence a fresh texture is generated (`glGenTextures`) and the previous image is blitted into it, matching the copy-on-grow behavior in §9.D.2.

### 9.D.4 Upload path — real, but SILENT (Finding #4 handled honestly)

Individual glyph uploads go through `send_sprite_to_gpu`, which reallocs if the cursor crossed a layer/row boundary and then uploads one cell with `glTexSubImage3D` [`kitty/shaders.c:147-155`]:

```c
// kitty/shaders.c:147-155
send_sprite_to_gpu(FONTS_DATA_HANDLE fg, unsigned int x, unsigned int y, unsigned int z, pixel *buf) {
    ...
    if ((int)znum >= sprite_map->last_num_of_layers || (znum == 0 && (int)ynum > sprite_map->last_ynum)) realloc_sprite_texture(fg);
    glBindTexture(GL_TEXTURE_2D_ARRAY, sprite_map->texture_id);
    ...
    glTexSubImage3D(GL_TEXTURE_2D_ARRAY, 0, x, y, z, sprite_map->cell_width, sprite_map->cell_height, 1, GL_RGBA, GL_UNSIGNED_INT_8_8_8_8, buf);
```

**Crucially, `send_sprite_to_gpu` emits no debug line** — there is no `--debug-rendering` log for each glyph upload. This is the honest resolution of Finding #4: the earlier document treated the Python test callback's "0 → 11 events" as GPU evidence, but that callback (`set_send_sprite_to_gpu` in the non-canonical `setup_for_testing` harness [`kitty/fonts/render.py:408-422`]) *replaces* the real `glTexSubImage3D` call and therefore proves nothing about the GPU. The correct evidence that the real upload path executed is **indirect but canonical**:

- The live launcher run in §6.A.4 emitted real `output_cell_fallback_data` lines for the Arabic run — which occur only *inside* the per-cell render/upload path (`render_run` → glyph rendering → `current_send_sprite_to_gpu`). The presence of those live fallback lines proves the real render+upload path ran end-to-end in the canonical process.
- The window actually displayed shaped glyphs (screenshots §6), which is impossible unless the sprites reached the GPU texture the fragment shader samples.

The **prerender count of 11** is therefore reported as **SOURCE-DERIVED** (from `prerender_function` composition, §9.D.5) and *corroborated* — clearly labelled as such — by the non-canonical test callback, which independently reported exactly 11 sprites at contiguous positions:

```text
[LABELED NON-CANONICAL CORROBORATION — test callback, not the GPU path]
cell_width=9 cell_height=18
prerendered sprite count = 11
positions (x,y,z) = [(0,0,0),(1,0,0),(2,0,0),(3,0,0),(4,0,0),(5,0,0),(6,0,0),(7,0,0),(8,0,0),(9,0,0),(10,0,0)]
distinct layers z = [0], distinct rows y = [0]
```

### 9.D.5 What fills the atlas *before* any shaped text — the 11 prerendered sprites

At window creation, `send_prerendered_sprites(fg)` uploads a fixed set of decoration/cursor sprites *before* the terminal shapes a single character [`kitty/fonts.c:1450-1471`]. It first uploads the **blank cell** at slot 0, then calls `prerender_function` with the exact cell metrics from Objective C — `baseline`, `underline_position/thickness`, `strikethrough_position/thickness` [`kitty/fonts.c:1458`] — directly linking §8's metrics to the atlas contents:

```c
// kitty/fonts.c:1455-1458 (blank cell, then prerender the decorations/cursors)
current_send_sprite_to_gpu((FONTS_DATA_HANDLE)fg, x, y, z, fg->canvas.buf);   // blank
...
PyObject *args = PyObject_CallFunction(prerender_function, "IIIIIIIffdd",
    fg->cell_width, fg->cell_height, fg->baseline, fg->underline_position, fg->underline_thickness,
    fg->strikethrough_position, fg->strikethrough_thickness, ...);
```

`prerender_function` [`kitty/fonts/render.py:364-395`] composes the sprite list: **5** underline styles (`range(1, NUM_UNDERLINE_STYLES + 1)` [`kitty/fonts/render.py:391`]) + **1** strikethrough + **1** missing-glyph + **3** cursor shapes (block/beam/underline) + the **1** blank uploaded first = **11**. That matches the observed occupancy exactly. These occupy slots `(0..10, 0, 0)` — all on row 0, layer 0 — consistent with the 1820-slot first row.

### 9.D.6 Readiness signal — INFERRED (Finding #83/#94 handled honestly)

There is **no single explicit "atlas ready" log line**. Readiness is *inferred* from two canonical, observed facts occurring in order:

1. The GL context reaches the required version — proven by the `--debug-rendering` banner, printed to **STDOUT** with a monotonic prefix by `gl.c` itself [`kitty/gl.c:72`], observed identically on every run:

```text
[0.123] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

   The required GL capability is enforced as fatal at init: `ARB_TEST(texture_storage)` aborts startup if the extension is missing [`kitty/gl.c:67`] — so reaching the banner and continuing implies immutable-storage support is present.

2. The 11 prerendered sprites upload successfully (§9.D.5) *before* the first shaped glyph. A successful prerender is the de-facto "atlas is allocated and accepting data" signal; if the prerender sprites overflowed a single row, startup would abort with a fatal error — the documented edge/error path [`kitty/fonts.c:1463`]:

```c
// kitty/fonts.c:1463
if (y > 0) { fatal("Too many pre-rendered sprites for your GPU or the font size is too large"); }
```

This fatal did **not** fire in any run (all 11 sprites fit on row 0), which is itself evidence the atlas was correctly sized and ready.

### 9.D.7 Shared atlas across primary + fallback fonts; the shader that samples it

There is **one** `sprite_map`/`GPUSpriteTracker` per `FontGroup`, shared by the primary font and every fallback font in that group — so an Arabic fallback glyph and a Latin primary glyph land in the *same* atlas texture. Per-font glyph→slot bookkeeping is kept in each `Font`'s own `sprite_position_hash_table` via `find_or_create_sprite_position` [`kitty/glyph-cache.c:34`], keyed on the glyph-index tuple. The cell fragment shader reads the atlas as a `sampler2DArray` [`kitty/cell_fragment.glsl:10`] and samples the glyph (and separately the underline/strikethrough/cursor sprites) with `texture(sprites, …)` [`kitty/cell_fragment.glsl:124,130-132`]:

```glsl
// kitty/cell_fragment.glsl:10,124
uniform sampler2DArray sprites;
...
vec4 text_fg = texture(sprites, sprite_pos);
```

This closes the loop: shaped glyphs from primary and fallback faces (§6.A.4) are uploaded via `send_sprite_to_gpu` into the shared 2D-array atlas allocated here, and read back per-cell by the fragment shader during rendering — the mechanism the live screenshots visually confirm.

---

## 10. Stability — every observation reproduced ≥ 2 times

Per the "report true magnitude/frequency/timing and confirm stability across at least two runs" rule, each launcher observation was run **twice** with the *same unchanged input*, its monotonic timestamps normalized (`sed 's/\[[0-9.]*\]/[T]/'`), and the two normalized captures `diff`ed. An **empty diff (0 bytes)** proves the two runs are byte-identical after removing only the wall-clock prefixes.

```console
$ # produced by the harness (§4): normalize [<float>] -> [T], then diff run1 vs run2
$ wc -c stability/*.diff
0 stability/default.diff
0 stability/arabic.diff
0 stability/cjk.diff
0 total
```

The normalized captures that were diffed (identical for both runs):

```text
# stability/arabic_run1.norm  ==  stability/arabic_run2.norm  (diff empty)
[T] OS Window created
[T] Failed to open systemd user bus with error: Connection refused
[T] Child launched
[T] Text fonts:
[T]   Normal: NimbusMonoPS-Regular: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1:0
[T]   Bold: NimbusMonoPS-Bold: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-Bold.otf:0
[T]   Italic: NimbusMonoPS-Italic: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Italic.t1:0
[T]   Bold-Italic: NimbusMonoPS-BoldItalic: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-BoldItalic.otf:0
[T] U+645 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[T] U+631 using previous fallback font at index: 0
[T] U+62d using previous fallback font at index: 0
[T] U+628 using previous fallback font at index: 0
[T] U+627 using previous fallback font at index: 0
[T] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

Stability summary (values confirmed identical on both runs):

| Observation | Runs | Stable? | Invariant values |
|-------------|-----:|:-------:|------------------|
| Default (DejaVu, no fallback) | 2 | ✅ empty diff | 4-face DejaVuSansMono block; GL banner `4.5 (Core Profile)`; no fallback line |
| Forced Arabic fallback (Nimbus) | 2 | ✅ empty diff | 4-face Nimbus block; `U+645 … DejaVuSansMono …`; `U+631/62d/628/627 using previous fallback font at index: 0` |
| CJK chain (`Hi 中文 World`) | 2 | ✅ empty diff | `U+4e2d … Noto Sans CJK JP …`; `U+6587 using previous fallback font at index: 0` |
| Cell metrics (production binding) | 2 | ✅ identical | `9×18` @96 DPI; `7×14` @72 DPI; Nimbus `9×19` |
| C-probe full metric set | 2 | ✅ identical | `baseline=14 underline_position=15 underline_thickness=1 strikethrough_position=10 strikethrough_thickness=1` |
| GL limits (glxinfo) | 2 | ✅ identical | `GL_MAX_TEXTURE_SIZE=16384`, `GL_MAX_ARRAY_TEXTURE_LAYERS=2048` |
| Prerender occupancy (corroboration) | 2 | ✅ identical | `11` sprites at `(0..10,0,0)` |

No run-to-run inconsistency was observed for any value; the environment is deterministic, so there is no distribution to report — the single stable value is reported for each quantity.

---

## 11. The three required honest negatives (consolidated)

Three findings are, per the AAP, **negatives that must be reported with evidence rather than forced into a positive**. Each is stated plainly with its proof and its label.

### 11.1 kitty has **no bidi (bidirectional) reordering engine** — OBSERVED + INFERRED

kitty performs **no logical→visual bidi reordering**. The `force_ltr` option's own documentation states kitty does not support BIDI [`kitty/options/definition.py:64-83`] (quoted verbatim in §6.A.2). What kitty *does* do is hand each run to HarfBuzz, which applies script-appropriate **shaping** (Arabic joining/contextual forms) — visible in `arabic_default.png` where مرحبا renders as connected cursive glyphs — but the *ordering* of runs is not reordered by a bidi algorithm. The resolved default `force_ltr=no` was observed at startup. That Arabic RTL clusters carry *decreasing* HarfBuzz cluster numbers [`kitty/fonts.c:997,1079`] is **INFERRED** from the source comments (not separately logged). This is a partial-support situation the changelog itself acknowledges [`docs/changelog.rst:3540-3541`].

### 11.2 There is **no overline text decoration** — OBSERVED (scoped negative)

Covered fully in §8.C.4: `grep -rniE "overline" kitty/` returns **0**; the only repository match is an unrelated XKB keyboard keysym at `glfw/xkb-compat-shim.h:135`. kitty prerenders blank, 5 underline styles, strikethrough, missing-glyph, and 3 cursor sprites — but never an overline.

### 11.3 The `setup_for_testing` harness is **non-canonical** — OBSERVED (used only as labelled corroboration)

`setup_for_testing` [`kitty/fonts/render.py:408-422`] calls `sprite_map_set_limits(100000, 100)` [`:421`] and `set_send_sprite_to_gpu(send_to_gpu)` [`:422`], which **replaces the real `glTexSubImage3D` upload** with a Python callback and sets artificial sprite limits. Every value obtained through it (the 11-sprite prerender count, the `test_shape` glyph-index tuples) is therefore labelled **NON-CANONICAL CORROBORATION** throughout this document and never used as primary evidence. The primary evidence is always the real launcher (`kitty/launcher/kitty`) run under Xvfb+openbox.

---

## 12. Variant / secondary-condition coverage

Per "exercise every condition, including secondary ones," the following variants were each run through the **real launcher** (except the two clearly-labelled binding probes). Every command is reproducible.

| Variant | Command (abridged; all via `./kitty/launcher/kitty --config NONE`) | What it exercised | Result |
|---------|--------------------------------------------------------------------|-------------------|--------|
| **Default** | `--debug-font-fallback --debug-rendering` + Arabic+English+combining feed | Primary shaping path, DejaVu covers Arabic | 4-face DejaVu block, GL banner, **no** fallback (honest default) |
| **Forced Arabic fallback** | `-o font_family="Nimbus Mono PS"` + same feed | Real fallback selection + `NimbusMonoPS-` code path | Live `U+645 … DejaVuSansMono` fallback lines |
| **CJK fallback (chain build+reuse)** | feed `Hi 中文 World` | New fallback + reuse of existing chain entry | `U+4e2d → Noto Sans CJK JP`; `U+6587 … index: 0` |
| **`--debug-gl`** | `--debug-gl` instead of `--debug-rendering` | Alternate flag for the GL banner | Same GL banner emitted |
| **`force_ltr=yes`** | `-o force_ltr=yes` | LTR-forcing modifier | Accepted at startup; DejaVu block + banner |
| **`disable_ligatures=always`** | `-o font_family="Fira Code" -o disable_ligatures=always` | Keeps trailing `-calt` → ligatures OFF | `fira_ligoff.png`: operators render discrete (contrast vs `fira_liga.png`) |
| **Ligatures ON (default)** | `-o font_family="Fira Code"` | Drops `-calt` sentinel → calt ON | `fira_liga.png`: `== -> != >= === <=>` visually fused |
| **`symbol_map` non-empty** | `-o symbol_map="U+0645 Amiri"` | Symbol-map block + explicit face mapping | `Symbol map fonts:` → `Amiri-Regular: …/Amiri-Regular.ttf:0` |
| **Combining diacritics** | default feed line 2 (`o`+U+0301+U+0323) | grapheme cluster → one cell | Live `U+6f U+301 U+323 using previous fallback font at index: 0` |
| **Overflow edge path** | (source-identified, not triggered) | prerender sprites spilling to row `y>0` | Fatal `Too many pre-rendered sprites…` [`fonts.c:1463`] — did **not** fire (11 fit row 0) |
| **Cell metrics** *(labelled binding)* | `+launch metrics_probe.py` | Real `calc_cell_metrics` via `create_test_font_group` | `9×18`; full set via validated C-probe |
| **Prerender occupancy** *(labelled corroboration)* | `setup_for_testing` callback | Count of sprites uploaded before shaped text | `11` at `(0..10,0,0)` |

---

## 13. Appendices — complete unedited logs

All raw launcher captures referenced above were written to `stability/*.norm` and per-run `logs/*.log` inside the harness's private `$WORK` directory and mirrored to the evidence directory during the investigation; the **relevant unedited excerpts are embedded inline** at the point of each claim (§4.1, §6, §7, §8, §9, §10). Each embedded block shows the command that produced it and preserves the `[<seconds>]` monotonic prefixes (or the `[T]` normalization explicitly marked as such for the stability diffs). No claim in this document relies on a value that is not shown next to it.

Because the harness removes `$WORK` on exit (§4, narrow teardown) and the investigation restores the tree to contain only this document (§14.3), the logs are not committed — reproducing them is a matter of re-running the embedded harness verbatim. The one full-length artifact reproduced verbatim in-document is the **canonical build transcript in Appendix A** (all 344 lines).

---

## 14. Final coverage pass

### 14.1 Objective-by-objective, named-item coverage

Every mechanism, function, flag, file, and "e.g./such as" item named in the four objectives is addressed below. Evidence type: **OBS** = observed live-launcher output; **SRC** = source-derived (with `file:line`); **COR** = labelled non-canonical corroboration; **VIS** = screenshot; **NEG** = evidence-backed negative.

| Objective / named item | Where | Evidence |
|------------------------|-------|----------|
| **A — ligatures** (`calt`/`liga`/`dlig`, `disable_ligatures`, `font_features`) | §6.A.1 | SRC [fonts.c:42,45,293-327,811-813,1755-1757] + OBS/VIS `fira_liga.png` vs `fira_ligoff.png` + COR |
| **A — bidi** (`force_ltr`, HarfBuzz RTL) | §6.A.2, §11.1 | SRC [definition.py:64-83; fonts.c:997,1079] + VIS + NEG |
| **A — combining diacritics** (`has_cell_text`, `codepoint_for_mark`) | §6.A.3 | OBS `U+6f U+301 U+323 …` + SRC [fonts.c:435-447,457-466] + VIS + COR |
| **A — font fallback at startup** (`fallback_font`, `load_fallback_font`, `create_fallback_face`, cap `>100`) | §6.A.4 | OBS Arabic+CJK fallback lines + SRC [fonts.c:481-520; fontconfig.c:463] |
| **B — startup diagnostics** (`--debug-font-fallback`, `dump_font_debug`, `identify_for_debug`) | §7.B.1-2 | OBS Text/Symbol font blocks + SRC [render.py:161-171; freetype.c:738] |
| **B — font families** (Normal/Bold/Italic/Bold-Italic) | §7.B.2 | OBS 4-face blocks (DejaVu / Nimbus / Fira / Noto) |
| **B — 9 config values** (`font_family`,`bold_font`,`italic_font`,`bold_italic_font`,`font_size`,`symbol_map`,`disable_ligatures`,`font_features`,`force_ltr`) | §7.B.3 | OBS + SRC [definition.py:35,53,55,57,59,64,86,115,135] |
| **C — cell metrics** (width/height/baseline) | §8.C.1 | OBS (dims) + SRC (full set, validated) [freetype.c:387-405] |
| **C — underline** (position/thickness, 5 styles) | §8.C.2 | SRC [freetype.c:392-393; data-types.h:213] |
| **C — strikethrough** (position/thickness) | §8.C.3 | SRC [freetype.c:396-403; shaders.py:162] |
| **C — overline** | §8.C.4, §11.2 | NEG (0 in `kitty/`; XKB keysym only) |
| **D — initial atlas layout / sizing** (`NEW_SPRITE_MAP`, `glTexStorage3D`) | §9.D.1-3 | SRC [shaders.c:24-31,51-54,108-123] + OBS GL limits |
| **D — capacity** (per-layer / layers / total) | §9.D.2 | SRC [fonts.c:236-239,276-281] + OBS GL limits arithmetic |
| **D — prerendered sprites** (blank/underline/strike/missing/cursor = 11) | §9.D.5 | SRC [fonts.c:1450-1471; render.py:364-395] + COR |
| **D — readiness** (GL banner, `texture_storage` fatal) | §9.D.6 | OBS banner [gl.c:72] + SRC [gl.c:67; fonts.c:1463] |
| **D — shared primary/fallback atlas + shader sampling** | §9.D.7 | SRC [glyph-cache.c:34; cell_fragment.glsl:10,124] |
| Methodology: streams, timestamps, macro gating | §0 | SRC [monotonic.h:99; logging.c; state.h:14,16; gl.c:72] |
| Build + run commands | §3, §5 | OBS build transcript + invocation |
| Stability (≥2 runs) | §10 | OBS empty diffs |
| Three honest negatives | §11 | NEG×3 |
| Variant coverage | §12 | OBS/VIS/COR per row |

### 14.2 Remaining checkpoint-named diagnostic flags (out-of-scope of the four objectives)

For an exhaustive named-item pass, three further diagnostic flags named in the checkpoint scope are accounted for here. None of them emits any of the Objective A–D startup signals investigated above.

- **`--debug-config` is not a startup CLI flag at all.** Passing it to the real launcher is rejected before startup begins:

```text
$ ./kitty/launcher/kitty --config NONE --debug-config -e true
Unknown option: --debug-config
$ echo $?
1
```

  It does not appear in `kitty --help`. `kitty/cli.py` defines only `--debug-rendering`/`--debug-gl` [`kitty/cli.py:989`], `--debug-input`/`--debug-keyboard` [`kitty/cli.py:996`] and `--debug-font-fallback` [`kitty/cli.py:1002`] — there is no `--debug-config` option. The real `debug_config` is instead a **runtime keyboard action**, bound by default to `kitty_mod+f6` [`kitty/options/definition.py:4255-4256`] and handled by `Boss.debug_config` [`kitty/boss.py:3060`]; triggering it is interactive input handling, which is out-of-scope for a *startup* investigation per AAP §0.5.2. Its objective-relevant substance is already covered canonically: the resolved fonts it prints via `font.identify_for_debug()` in `debug_config()` [`kitty/debug_config.py:262-263`] use the **same `identify_for_debug` formatter** (`"%s: %V:%d"` [`kitty/freetype.c:738`]) that `dump_font_debug` already emits at real startup in §7.B.2. So `--debug-config` is corroborative only and adds no startup evidence beyond §7.

- **`--debug-input` / `--debug-keyboard` are real CLI flags** [`kitty/cli.py:996`, `dest=debug_keyboard`] whose help text is *"Print out key and mouse events as they are received."* They debug **keyboard/mouse input events** — explicitly out-of-scope (AAP §0.5.2) and unrelated to Objectives A–D (shaping/fallback, startup diagnostics, cell metrics, GPU atlas). They emit none of the font/render/atlas signals this document investigates.

Thus every diagnostic flag named in the checkpoint scope is accounted for: the three startup flags `--debug-font-fallback`, `--debug-rendering` and `--debug-gl` are exercised canonically above (§6–§9), while `--debug-config`, `--debug-input` and `--debug-keyboard` are scoped out for the reasons just given (a non-CLI keyboard action, and out-of-scope input-event debugging), with the one objective-relevant piece of `--debug-config` — its `identify_for_debug` font dump — already shown canonically in §7.

### 14.3 Read-only integrity

The investigation modified **no** source file. The only repository change is the addition of this document, `blitzy/documentation/kitty_815df1e210e0.md`. All temporary observation scripts and the harness `$WORK` directory were removed, the headless display stack was torn down (narrow, PID-scoped), and `git status` confirms a clean tree apart from this single added file.

---

## Appendix A — Complete, unedited canonical build transcript

The full output of `CI=true python3 setup.py` (referenced in §3), reproduced verbatim (344 lines, `BUILD_EXIT=0`, wall 65 s). Preceded by `python3 setup.py clean` (exit 0). No line is elided.

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
[5/85] Compiling kitty/graphics.c ...
[6/85] Compiling kitty/child-monitor.c ...
[7/85] Compiling kitty/fonts.c ...
[8/85] Compiling kitty/shaders.c ...
[9/85] Compiling kitty/vt-parser.c ...
[10/85] Compiling kitty/vt-parser.c ...
[11/85] Compiling kitty/state.c ...
[12/85] Compiling [x11] glfw/input.c ...
[13/85] Compiling kitty/mouse.c ...
[14/85] Compiling [x11] glfw/xkb_glfw.c ...
[15/85] Compiling kitty/freetype.c ...
[16/85] Compiling [x11] glfw/window.c ...
[17/85] Compiling kitty/line.c ...
[18/85] Compiling kitty/glfw-wrapper.c ...
[19/85] Compiling kittens/transfer/algorithm.c ...
[20/85] Compiling [x11] glfw/x11_init.c ...
[21/85] Compiling kitty/freetype_render_ui_text.c ...
[22/85] Compiling [x11] glfw/egl_context.c ...
[23/85] Compiling kitty/disk-cache.c ...
[24/85] Compiling [x11] glfw/glx_context.c ...
[25/85] Compiling kitty/line-buf.c ...
[26/85] Compiling kitty/data-types.c ...
[27/85] Compiling kitty/colors.c ...
[28/85] Compiling kitty/history.c ...
[29/85] Compiling kitty/keys.c ...
[30/85] Compiling [x11] glfw/x11_monitor.c ...
[31/85] Compiling kitty/fontconfig.c ...
[32/85] Compiling [x11] glfw/context.c ...
[33/85] Compiling kitty/crypto.c ...
[34/85] Compiling [x11] glfw/ibus_glfw.c ...
[35/85] Compiling kitty/key_encoding.c ...
[36/85] Compiling kitty/launcher/main.c ...
[37/85] Compiling [x11] glfw/monitor.c ...
[38/85] Compiling kitty/font-names.c ...
[39/85] Compiling [x11] glfw/backend_utils.c ...
[40/85] Compiling kitty/charsets.c ...
[41/85] Compiling [x11] glfw/linux_joystick.c ...
[42/85] Compiling [x11] glfw/init.c ...
[43/85] Compiling [x11] glfw/dbus_glfw.c ...
[44/85] Compiling kitty/gl.c ...
[45/85] Compiling [x11] glfw/vulkan.c ...
[46/85] Compiling [x11] glfw/osmesa_context.c ...
[47/85] Compiling kitty/cursor.c ...
[48/85] Compiling kitty/launcher/single-instance.c ...
[49/85] Compiling kitty/desktop.c ...
[50/85] Compiling kitty/loop-utils.c ...
[51/85] Compiling 3rdparty/ringbuf/ringbuf.c ...
[52/85] Compiling kitty/simd-string.c ...
[53/85] Compiling kitty/systemd.c ...
[54/85] Compiling kitty/shlex.c ...
[55/85] Compiling kitty/child.c ...
[56/85] Compiling kitty/kittens.c ...
[57/85] Compiling 3rdparty/base64/lib/codec_choose.c ...
[58/85] Compiling kitty/png-reader.c ...
[59/85] Compiling [x11] glfw/linux_notify.c ...
[60/85] Compiling kitty/rowcolumn-diacritics.c ...
[61/85] Compiling kitty/hyperlink.c ...
[62/85] Compiling kitty/wcswidth.c ...
[63/85] Compiling kitty/fast-file-copy.c ...
[64/85] Compiling 3rdparty/base64/lib/lib.c ...
[65/85] Compiling [x11] glfw/posix_thread.c ...
[66/85] Compiling kitty/window_logo.c ...
[67/85] Compiling kitty/glyph-cache.c ...
[68/85] Compiling kitty/logging.c ...
[69/85] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[70/85] Compiling 3rdparty/base64/lib/tables/tables.c ...
[71/85] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[72/85] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[73/85] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[74/85] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[75/85] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[76/85] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[77/85] Compiling kitty/utmp.c ...
[78/85] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[79/85] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[80/85] Compiling kitty/cleanup.c ...
[81/85] Compiling [x11] glfw/monotonic.c ...
[82/85] Compiling kitty/monotonic.c ...
[83/85] Compiling kitty/simd-string-128.c ...
[84/85] Compiling kitty/simd-string-256.c ...
[85/85] Compiling kitty/gl-wrapper.c ...
 done
[1/4] Linking kitty/fast_data_types ...
[2/4] Linking [x11] kitty/glfw-x11 ...
[3/4] Linking kittens/transfer/rsync ...
[4/4] Linking launcher ...
 done
go: downloading golang.org/x/sys v0.21.0
go: downloading github.com/seancfoley/ipaddress-go v1.6.0
go: downloading github.com/edwvee/exiffix v0.0.0-20240229113213-0dbb146775be
go: downloading github.com/alecthomas/chroma/v2 v2.14.0
go: downloading github.com/dlclark/regexp2 v1.11.0
go: downloading golang.org/x/image v0.17.0
go: downloading github.com/bmatcuk/doublestar/v4 v4.6.1
go: downloading golang.org/x/exp v0.0.0-20230801115018-d63ba01acd4b
go: downloading howett.net/plist v1.0.1
go: downloading github.com/google/uuid v1.6.0
go: downloading github.com/ALTree/bigfloat v0.2.0
go: downloading github.com/kovidgoyal/imaging v1.6.3
go: downloading github.com/zeebo/xxh3 v1.0.2
go: downloading github.com/shirou/gopsutil/v3 v3.24.5
go: downloading github.com/rwcarlsen/goexif v0.0.0-20190401172101-9e8deecbddbd
go: downloading github.com/disintegration/imaging v1.6.2
go: downloading github.com/klauspost/cpuid/v2 v2.2.5
go: downloading github.com/seancfoley/bintree v1.3.1
go: downloading github.com/tklauser/go-sysconf v0.3.12
go: downloading github.com/tklauser/numcpus v0.6.1
image/color
crypto/internal/fips140deps/byteorder
crypto/internal/fips140deps/cpu
vendor/golang.org/x/crypto/internal/alias
golang.org/x/exp/constraints
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
crypto/internal/boring/sig
vendor/golang.org/x/crypto/cryptobyte/asn1
internal/nettrace
log/internal
container/list
encoding
kitty
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
crypto/internal/fips140/alias
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
unicode/utf16
github.com/shirou/gopsutil/v3/common
weak
hash
maps
internal/singleflight
crypto/internal/fips140deps/godebug
vendor/golang.org/x/net/dns/dnsmessage
math/rand/v2
crypto/internal/impl
net/http/internal/ascii
encoding/base32
vendor/golang.org/x/text/transform
regexp/syntax
bufio
encoding/binary
crypto/internal/fips140/subtle
context
embed
runtime/cgo
crypto
hash/adler32
hash/crc32
crypto/internal/sysrand
io/ioutil
encoding/hex
log
vendor/golang.org/x/sys/cpu
kitty/tools/utils/shlex
flag
github.com/bmatcuk/doublestar/v4
vendor/golang.org/x/net/http2/hpack
unique
github.com/ALTree/bigfloat
encoding/asn1
github.com/seancfoley/bintree/tree
net/url
image/color/palette
crypto/internal/randutil
crypto/subtle
crypto/internal/fips140
golang.org/x/image/riff
compress/bzip2
compress/flate
crypto/internal/entropy
golang.org/x/image/tiff/lzw
compress/lzw
crypto/internal/fips140/sha256
os/exec
net/http/internal
image
mime/quotedprintable
database/sql/driver
os/signal
encoding/xml
vendor/golang.org/x/text/unicode/bidi
crypto/tls/internal/fips140tls
crypto/internal/fips140/sha3
net/netip
crypto/internal/fips140/sha512
encoding/base64
github.com/klauspost/cpuid/v2
vendor/golang.org/x/crypto/cryptobyte
github.com/rwcarlsen/goexif/tiff
crypto/x509/pkix
vendor/golang.org/x/crypto/internal/poly1305
github.com/dlclark/regexp2/syntax
vendor/golang.org/x/text/unicode/norm
regexp
golang.org/x/sys/unix
encoding/pem
mime
encoding/json
github.com/shirou/gopsutil/v3/internal/common
compress/gzip
compress/zlib
archive/zip
crypto/internal/fips140/hmac
crypto/sha3
vendor/golang.org/x/text/secure/bidirule
golang.org/x/image/bmp
image/internal/imageutil
golang.org/x/image/ccitt
image/png
golang.org/x/image/vp8l
golang.org/x/image/vp8
crypto/internal/fips140/check
crypto/internal/fips140hash
image/draw
image/jpeg
crypto/internal/fips140/hkdf
crypto/internal/fips140/bigmod
crypto/internal/fips140/tls12
crypto/internal/fips140/edwards25519/field
crypto/internal/fips140/aes
crypto/internal/fips140/nistec/fiat
golang.org/x/image/tiff
crypto/internal/fips140/tls13
github.com/zeebo/xxh3
golang.org/x/image/webp
vendor/golang.org/x/net/idna
image/gif
crypto/internal/fips140/edwards25519
github.com/dlclark/regexp2
crypto/internal/fips140/drbg
crypto/internal/fips140only
crypto/internal/fips140/mlkem
crypto/internal/fips140/aes/gcm
github.com/kovidgoyal/imaging
github.com/disintegration/imaging
crypto/internal/fips140/rsa
howett.net/plist
crypto/internal/fips140/ed25519
crypto/rc4
crypto/md5
crypto/dsa
crypto/cipher
github.com/rwcarlsen/goexif/exif
crypto/internal/fips140/nistec
crypto/internal/boring
crypto/des
vendor/golang.org/x/crypto/chacha20
crypto/rand
crypto/aes
crypto/internal/boring/bbig
crypto/sha512
crypto/hmac
crypto/sha1
crypto/sha256
github.com/edwvee/exiffix
vendor/golang.org/x/crypto/chacha20poly1305
crypto/ed25519
kitty/tools/utils/secrets
crypto/rsa
github.com/alecthomas/chroma/v2
github.com/tklauser/numcpus
github.com/shirou/gopsutil/v3/mem
github.com/tklauser/go-sysconf
github.com/shirou/gopsutil/v3/cpu
github.com/alecthomas/chroma/v2/styles
github.com/alecthomas/chroma/v2/lexers
os/user
net
crypto/internal/fips140/ecdh
crypto/elliptic
crypto/internal/fips140/ecdsa
crypto/ecdh
crypto/internal/hpke
crypto/ecdsa
archive/tar
github.com/google/uuid
github.com/shirou/gopsutil/v3/net
net/textproto
vendor/golang.org/x/net/http/httpproxy
crypto/x509
github.com/seancfoley/ipaddress-go/ipaddr
vendor/golang.org/x/net/http/httpguts
mime/multipart
github.com/shirou/gopsutil/v3/process
crypto/tls
net/http/httptrace
net/http
kitty/tools/utils
kitty/tools/utils/base85
kitty/tools/tty
kitty/tools/utils/paths
kitty/tools/rsync
kitty/tools/wcswidth
kitty/tools/crypto
kitty/tools/tui/shell_integration
kitty/tools/utils/humanize
kitty/tools/utils/style
kitty/tools/cli/markup
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/cli
kitty/tools/tui/shortcuts
kitty/tools/cmd/mouse_demo
kitty/tools/config
kitty/tools/utils/shm
kitty/kittens/query_terminal
kitty/kittens/hyperlinked_grep
kitty/kittens/show_key
kitty/tools/tui
kitty/tools/tui/readline
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/kittens/ask
kitty/tools/cmd/run_shell
kitty/kittens/hints
kitty/tools/tui/graphics
kitty/tools/cmd/edit_in_kitty
kitty/tools/cmd/show_error
kitty/tools/themes
kitty/tools/cmd/update_self
kitty/tools/cmd/at
kitty/kittens/unicode_input
kitty/kittens/transfer
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/choose_fonts
kitty/kittens/themes
kitty/kittens/ssh
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
[BUILD_EXIT=0] wall=65s
```
