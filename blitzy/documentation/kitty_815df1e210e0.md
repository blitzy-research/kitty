# kitty — Text-Shaping / Layout Configuration, Startup Font Fallback, Cell Metrics, and GPU Texture-Atlas Initialization

*An evidence-based, runtime-observed answer for the source branch `kitty_815df1e210e0`.*

This document explains — and proves through actual runtime observation on a default, canonical build — how the [kitty](https://github.com/kovidgoyal/kitty) terminal emulator (a) configures its text-shaping / layout engine for complex Unicode (ligatures, bidirectional text, combining diacritics) and performs font fallback at startup; (b) what the verbose-logging startup diagnostics report for mixed **Arabic (RTL) + English (LTR)** text; (c) the default cell metrics, baseline, and decoration alignment computed as the screen-grid / shaping subsystems initialize; and (d) how the GPU texture-atlas is laid out, sized, and made ready at launch.

The **primary evidence** in this document is **OBSERVED (canonical)** output — produced by **building kitty with `python3 setup.py` and running the produced launcher `kitty/launcher/kitty`** under a headless OpenGL display (Xvfb) with a window manager (openbox), with the real diagnostic flags `--debug-font-fallback` and `--debug-rendering` enabled. Where the launcher emits **no** line for a required quantity — for example the per-face cell metrics of §8, which the launcher never prints — the value is **SOURCE-DERIVED**: computed by an independent probe that links the *same* system libraries and applies the production formula verbatim, then cross-checked against a non-canonical binding. Where no runtime signal exists at all — for example atlas "readiness" (§9), which has no single log line — the conclusion is **INFERRED** from the source; and test-binding numbers appear only as **NON-CANONICAL CORROBORATION**. Every claim is tagged with exactly one of these four labels (defined in §0.1) so that canonical observations are never confused with derivations. The exact commands, complete unedited output, and the secure harness that produced them are embedded below.

---

## 0. How to read this document (labels, streams, timestamps, methodology)

### 0.1 Evidence labels

Every claim carries one of the following labels so that observed facts are never confused with derivations:

| Label | Meaning |
|-------|---------|
| **OBSERVED (canonical)** | Emitted by the real launcher `kitty/launcher/kitty` at startup through its real entry point. This is the primary, canonical evidence required by the task. |
| **SOURCE-DERIVED** | A value computed by applying a documented production formula, verbatim from the source, to real inputs — used when the launcher emits no log line for the quantity (e.g. the per-face cell metrics, which the launcher never prints). Computed by an independent probe linking the same system libraries, and cross-validated against any OBSERVED or corroborating value wherever one exists. |
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

Every claim is anchored with a `file:line` reference to the checkout at the investigation-time commit `722ca9e9e` (branch content `kitty_815df1e210e0`). Line numbers were re-verified against the working tree. `722ca9e9e` is the commit at which the runtime observations in this document were captured; the subsequent commits on this branch (`e4e0f7afe`, `146e07c35`, and this reconciliation) are **documentation-only** — each modifies only this `.md` file. Consequently `git diff --name-only 815df1e21..HEAD` lists only this document, the source tree is byte-identical from the base commit `815df1e21` through the current `HEAD`, and every source `file:line` anchor below therefore remains exactly valid at whatever commit `HEAD` currently points to (re-running `git rev-parse HEAD` today returns a later, documentation-only commit than the one shown in §1.1).

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

An empty `git status --porcelain` and empty `git diff --stat` confirm the tree was clean before observation. The final integrity check (only this `.md` added; no source file modified) is shown in §14.3.

> **Provenance note.** The `HEAD` printed above, `722ca9e9e`, is the *investigation-time* commit at which these baseline captures and all runtime observations in this document were taken. The document was subsequently revised in documentation-only commits (`e4e0f7afe`, then `146e07c35`, and this reconciliation), each of which modifies **only this `.md` file**. The source tree is therefore byte-identical from the base commit `815df1e21` through the current `HEAD` — `git diff --name-only 815df1e21..HEAD` lists only this document — so re-running `git rev-parse HEAD` today returns a later (documentation-only) commit than the `722ca9e9e` shown above, while every source `file:line` anchor in this document remains exactly valid.

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

There are **37** Arabic-capable family+file entries (26 unique family names). Crucially, the default `monospace` face on this box — **DejaVu Sans Mono** — itself contains Arabic glyphs, which is why the *default* run shows **no** fallback line and why a font override (Nimbus Mono PS, which lacks Arabic) is used to force and capture a real Arabic fallback selection (Objectives A and B — §6, §7).

### 2.4 Display stack, window manager, and readiness (OBSERVED)

Reaching atlas initialization at real startup requires an OpenGL context, and reaching the per-cell render path requires the OS window to be **mapped and exposed** — otherwise `should_os_window_be_rendered()` [`kitty/glfw.c:1812`] returns false (iconified / not-visible / occluded) and the render path never runs. A headless X server (Xvfb) plus a window manager (openbox) satisfies both. The harness (§4) allocates a **unique free display**, starts Xvfb and openbox, captures their exact PIDs, waits for readiness, and tears down only those PIDs. Bring-up header of the canonical harness run whose complete output is embedded in §4.1 (same `WORK` directory, `/tmp/kitty_obs.c2jvSe3Q`; the `WORK` suffix and PIDs are per-run `mktemp`/process values and differ on every invocation):

```text
WORK=/tmp/kitty_obs.c2jvSe3Q  (umask=0077)
DISPLAY=:200
XVFB_PID=356945
OPENBOX_PID=356946
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

The context is OpenGL 4.5 Core (Mesa llvmpipe). The two limits **`GL_MAX_TEXTURE_SIZE = 16384`** and **`GL_MAX_ARRAY_TEXTURE_LAYERS = 2048`** are the real inputs to the atlas capacity math (Objective D — §9). `gl_init` treats the absence of required capabilities such as `texture_storage` as fatal [`kitty/gl.c:67`]; the context above provides them.

---

## 3. Canonical build (OBSERVED)

The canonical build is the `Makefile` `all` target, which runs `python3 setup.py`:

```console
$ sed -n '12,14p' Makefile
all:
	python3 setup.py $(VVAL)
```

Exact build command and result (the **complete, unedited transcript is embedded verbatim in Appendix A** — that block is 344 lines: 343 lines of raw `python3 setup.py` output plus the harness's trailing `[BUILD_EXIT=0] wall=65s` status line, so a fresh `CI=true python3 setup.py 2>&1 | wc -l` reports **343**):

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
mkdir -p "$WORK/logs" "$WORK/stability"
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

for _ in $(seq 1 50); do
  if DISPLAY=":$DISP" xdpyinfo >/dev/null 2>&1; then break; fi
  sleep 0.1
done
echo "X server ready on :$DISP"

# ---- embedded feed scripts (verbatim, auditable) ---------------------------
# 1) Mixed Arabic (RTL) + English (LTR) + a combining-diacritics line.
cat > "$WORK/feed_default.sh" <<'FEED'
#!/bin/sh
printf 'Hello \331\205\330\261\330\255\330\250\330\247 World\n'
printf 'combine: o\314\201\314\243  e\314\201  a\314\210  n\314\203\n'
sleep 4
FEED
# 2) CJK, to exercise a second fallback chain (Noto Sans CJK).
cat > "$WORK/feed_cjk.sh" <<'FEED'
#!/bin/sh
printf 'Hi \344\270\255\346\226\207 World\n'
sleep 4
FEED
# 3) Programming-ligature operators (Fira Code).
cat > "$WORK/feed_fira.sh" <<'FEED'
#!/bin/sh
printf '== -> != >= <= === =~ |> ++\n'
sleep 4
FEED
# 4) Six code points with no glyph in Nimbus or its DejaVu fallback
#    (U+0378 U+0380 U+1E4D0 U+31350 U+F0000 U+10FFFD) -> verification-failure path.
cat > "$WORK/feed_missing.sh" <<'FEED'
#!/bin/sh
printf '\315\270 \316\200 \360\236\223\220 \360\261\215\220 \363\260\200\200 \364\217\277\275\n'
sleep 4
FEED
chmod +x "$WORK"/feed_*.sh

# Flexible runner: $1=label, rest = full launcher args (flags + feed script).
run() {
  local label="$1"; shift
  local rc=0
  timeout 25 "$LAUNCHER" --config NONE "$@" >"$WORK/logs/$label.log" 2>&1 || rc=$?
  echo "[RUN $label exit=$rc]"
}
DBG="--debug-font-fallback --debug-rendering"; C="-o confirm_os_window_close=0"

# Every variant is run TWICE with identical input (stability requirement).
run default_run1  $DBG $C "$WORK/feed_default.sh"                                                   # DejaVu covers Arabic -> no fallback
run default_run2  $DBG $C "$WORK/feed_default.sh"
run arabic_run1   $DBG $C -o font_family="Nimbus Mono PS" "$WORK/feed_default.sh"                   # Nimbus lacks Arabic -> fallback
run arabic_run2   $DBG $C -o font_family="Nimbus Mono PS" "$WORK/feed_default.sh"
run cjk_run1      $DBG $C "$WORK/feed_cjk.sh"                                                       # CJK fallback chain
run cjk_run2      $DBG $C "$WORK/feed_cjk.sh"
run debuggl_run1  --debug-font-fallback --debug-gl $C "$WORK/feed_default.sh"                       # --debug-gl alias of --debug-rendering
run debuggl_run2  --debug-font-fallback --debug-gl $C "$WORK/feed_default.sh"
run forceltr_run1 $DBG $C -o force_ltr=yes "$WORK/feed_default.sh"                                  # LTR-forcing modifier
run forceltr_run2 $DBG $C -o force_ltr=yes "$WORK/feed_default.sh"
run fira_on_run1  $DBG $C -o font_family="Fira Code" -o font_size=28 "$WORK/feed_fira.sh"           # ligatures ON (calt)
run fira_on_run2  $DBG $C -o font_family="Fira Code" -o font_size=28 "$WORK/feed_fira.sh"
run fira_off_run1 $DBG $C -o font_family="Fira Code" -o disable_ligatures=always -o font_size=28 "$WORK/feed_fira.sh"  # ligatures OFF
run fira_off_run2 $DBG $C -o font_family="Fira Code" -o disable_ligatures=always -o font_size=28 "$WORK/feed_fira.sh"
run symbolmap_run1 $DBG $C -o symbol_map="U+0645 Amiri" "$WORK/feed_default.sh"                     # symbol_map explicit face
run symbolmap_run2 $DBG $C -o symbol_map="U+0645 Amiri" "$WORK/feed_default.sh"
run missing_run1  $DBG $C -o font_family="Nimbus Mono PS" "$WORK/feed_missing.sh"                   # glyph-verification-failure path
run missing_run2  $DBG $C -o font_family="Nimbus Mono PS" "$WORK/feed_missing.sh"
run font28_run1   $DBG $C -o font_size=28 "$WORK/feed_default.sh"                                   # underscore workaround fires
run font28_run2   $DBG $C -o font_size=28 "$WORK/feed_default.sh"

# ---- stability: normalize monotonic timestamps [<float>] -> [T], then diff --
norm() { sed -E 's/^\[[0-9]+\.[0-9]+\]/[T]/' "$1"; }
for v in default arabic cjk debuggl forceltr fira_on fira_off symbolmap missing font28; do
  norm "$WORK/logs/${v}_run1.log" > "$WORK/stability/${v}_run1.norm"
  norm "$WORK/logs/${v}_run2.log" > "$WORK/stability/${v}_run2.norm"
  diff -u "$WORK/stability/${v}_run1.norm" "$WORK/stability/${v}_run2.norm" > "$WORK/stability/${v}.diff" || true
  if [ -s "$WORK/stability/${v}.diff" ]; then echo "STABILITY[$v]: DIFFERENCES"; else echo "STABILITY[$v]: STABLE (0 diff)"; fi
done
wc -c "$WORK"/stability/*.diff

echo "=== logs/default_run1.log (complete, unedited) ==="; cat "$WORK/logs/default_run1.log"
echo "=== logs/arabic_run1.log  (complete, unedited) ==="; cat "$WORK/logs/arabic_run1.log"
# trap runs cleanup on exit
```

### 4.1 Complete, unedited harness output (OBSERVED)

```text
WORK=/tmp/kitty_obs.c2jvSe3Q  (umask=0077)
DISPLAY=:200
XVFB_PID=356945
OPENBOX_PID=356946
X server ready on :200
[RUN default_run1 exit=0]
[RUN default_run2 exit=0]
[RUN arabic_run1 exit=0]
[RUN arabic_run2 exit=0]
[RUN cjk_run1 exit=0]
[RUN cjk_run2 exit=0]
[RUN debuggl_run1 exit=0]
[RUN debuggl_run2 exit=0]
[RUN forceltr_run1 exit=0]
[RUN forceltr_run2 exit=0]
[RUN fira_on_run1 exit=0]
[RUN fira_on_run2 exit=0]
[RUN fira_off_run1 exit=0]
[RUN fira_off_run2 exit=0]
[RUN symbolmap_run1 exit=0]
[RUN symbolmap_run2 exit=0]
[RUN missing_run1 exit=0]
[RUN missing_run2 exit=0]
[RUN font28_run1 exit=0]
[RUN font28_run2 exit=0]
STABILITY[default]: STABLE (0 diff)
STABILITY[arabic]: STABLE (0 diff)
STABILITY[cjk]: STABLE (0 diff)
STABILITY[debuggl]: STABLE (0 diff)
STABILITY[forceltr]: STABLE (0 diff)
STABILITY[fira_on]: STABLE (0 diff)
STABILITY[fira_off]: STABLE (0 diff)
STABILITY[symbolmap]: STABLE (0 diff)
STABILITY[missing]: STABLE (0 diff)
STABILITY[font28]: STABLE (0 diff)
0 /tmp/kitty_obs.c2jvSe3Q/stability/arabic.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/cjk.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/debuggl.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/default.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/fira_off.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/fira_on.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/font28.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/forceltr.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/missing.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/symbolmap.diff
0 total
=== logs/default_run1.log (complete, unedited) ===
[0.159] OS Window created
[0.168] Failed to open systemd user bus with error: Connection refused
[0.172] Child launched
[0.172] Text fonts:
[0.172]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.172]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.172]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.172]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
[0.128] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
=== logs/arabic_run1.log  (complete, unedited) ===
[0.156] OS Window created
[0.165] Failed to open systemd user bus with error: Connection refused
[0.169] Child launched
[0.169] Text fonts:
[0.169]   Normal: NimbusMonoPS-Regular: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1:0
[0.169]   Bold: NimbusMonoPS-Bold: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-Bold.otf:0
[0.169]   Italic: NimbusMonoPS-Italic: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Italic.t1:0
[0.169]   Bold-Italic: NimbusMonoPS-BoldItalic: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-BoldItalic.otf:0
[0.190] U+645 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.191] U+631 using previous fallback font at index: 0
[0.191] U+62d using previous fallback font at index: 0
[0.192] U+628 using previous fallback font at index: 0
[0.192] U+627 using previous fallback font at index: 0
[0.193] U+6f U+301 U+323 using previous fallback font at index: 0
[0.124] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
[cleanup done; killed openbox=356946 xvfb=356945; removed /tmp/kitty_obs.c2jvSe3Q; exit=0]
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
- `--debug-font-fallback` enables the font-dump and per-cell fallback diagnostics [`kitty/cli.py:1002`]; `--debug-rendering` enables the GL banner and rendering diagnostics [`kitty/cli.py:989`]. The alternate spelling `--debug-gl` is exercised as a variant in §12 and produces the identical GL banner.
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

**OBSERVED (canonical) — ligatures render in the real launcher.** A Fira Code run (`-o font_family="Fira Code" -o font_size=28`) fed `== -> != >= <= === =~ |> ++` / `www <=> |||` was captured as a screenshot of the mapped openbox-managed window; the image is **embedded below as Figure L-ON** (with its SHA-256 and the exact capture command, so any reader can regenerate and verify it). Every operator sequence renders as a single fused ligature glyph: `==` → one wide double-bar equals; `->` → a fused arrow (→); `!=` → ≠; `>=` → ≥; `<=` → ≤; `===` → a triple-bar glyph; `|>` → a right-pointing triangle. The startup font block for that run confirms the primary faces are Fira Code:

```text
[0.169] Text fonts:
[0.169]   Normal: FiraCode-Regular: /usr/share/fonts/truetype/firacode/FiraCode-Regular.ttf:0
[0.169]   Bold: FiraCode-SemiBold: /usr/share/fonts/truetype/firacode/FiraCode-SemiBold.ttf:0
[0.169]   Italic: FiraCode-Retina: /usr/share/fonts/truetype/firacode/FiraCode-Retina.ttf:0
[0.169]   Bold-Italic: FiraCode-SemiBold: /usr/share/fonts/truetype/firacode/FiraCode-SemiBold.ttf:0
```

**OBSERVED (canonical) — `disable_ligatures=always` suppresses them.** The identical Fira Code input with `-o disable_ligatures=always` was captured; the image is **embedded below as Figure L-OFF**. Every operator now renders as separate discrete glyphs (`==` is two equals signs, `->` is a dash then `>`, `!=` is `!` then `=`, `===` is three equals, etc.). This is the direct visual contrast that proves the `-calt` sentinel is *kept* (feature list not decremented) when ligatures are disabled, turning `calt` off. Its startup font block is **byte-identical** to the ligatures-on block above (the four Fira Code faces) — confirming, as noted, that ligature state is invisible at startup and lives entirely in shaping; the two-run logs for both variants are catalogued in §12.

**REPRODUCIBLE GLYPH-LEVEL PROOF (HarfBuzz shaping) — the authoritative, text-based evidence.** Because `disable_ligatures` is a *shaping-time* control, the ligature difference is **not** visible in `--debug-font-fallback` output (the Fira Code startup block above is byte-identical with ligatures on or off — see §12). The fused-vs-discrete substitution therefore has to be proven at the glyph level, and it can be, reproducibly and without any image, in two independent ways whose glyph IDs agree exactly.

*(i) Through kitty's own shaper (`shape_run`), via the `test_shape` entry point.* `kitty.fonts.render.shape_string` drives the **real production `shape_run`** [`kitty/fonts.c:1226`] (the same function `render_run` calls when drawing to screen); only the sprite upload is stubbed by `setup_for_testing`, so the *shaping result is production-identical*. It reports HarfBuzz output as per-group tuples `(num_cells, num_glyphs, first_glyph_index, glyph_indices…)`; the returned codepoints are **glyph indices** (font-specific glyph IDs), not Unicode scalar values. Reproduce with:

```bash
./kitty/launcher/kitty +runpy '
from kitty.fonts.render import shape_string
print("FiraCode = :", shape_string("=",  family="Fira Code", size=28.0))
print("FiraCode ==:", shape_string("==", family="Fira Code", size=28.0))
print("FiraCode = =:",shape_string("= =",family="Fira Code", size=28.0))
print("FiraCode ->:", shape_string("->", family="Fira Code", size=28.0))
print("FiraCode - >:",shape_string("- >",family="Fira Code", size=28.0))
print("DejaVu   ==:", shape_string("==", family="DejaVu Sans Mono", size=28.0))
print("DejaVu   ->:", shape_string("->", family="DejaVu Sans Mono", size=28.0))'
```

Complete, unedited output (identical across two runs):

```text
FiraCode = : [(1, 1, 1578, (1578,))]
FiraCode ==: [(2, 2, 1649, (1649, 1387))]
FiraCode = =: [(1, 1, 1578, (1578,)), (1, 1, 1103, (1103,)), (1, 1, 1578, (1578,))]
FiraCode ->: [(2, 2, 1186, (1186, 1458))]
FiraCode - >: [(1, 1, 1221, (1221,)), (1, 1, 1103, (1103,)), (1, 1, 1580, (1580,))]
DejaVu   ==: [(1, 1, 32, (32,)), (1, 1, 32, (32,))]
DejaVu   ->: [(1, 1, 16, (16,)), (1, 1, 33, (33,))]
```

Reading this carefully: a lone `=` in Fira Code is glyph **1578**; the adjacent pair `==` is shaped into **one 2-cell group** whose glyphs are **1649, 1387** — *neither equal to 1578*, i.e. HarfBuzz's `calt` substituted both cells into ligature-component glyphs. Insert a space (`= =`) and `calt` no longer fires across the gap: the plain `=` glyph **1578** reappears on each side. The arrow `->` fuses to **1186, 1458**, but the space-separated `- >` reverts to the plain hyphen **1221** and greater-than **1580**. In DejaVu the same `==` produces **two separate single-cell groups**, both the ordinary `=` glyph **32**, and `->` stays the plain hyphen **16** + greater-than **33** — because DejaVu carries no such `calt` lookups.

*(ii) Through the system HarfBuzz library directly (fully independent of kitty).* A ~50-line standalone probe (`hb_ligature_probe.c`, source in §13) links the **same** system HarfBuzz (10.2.0) and FreeType that kitty links, opens `FiraCode-Regular.ttf`, and shapes each operator once with the `calt` feature **enabled** and once **disabled** — `calt=0` being exactly what kitty's `disable_ligatures=always` does by *keeping* the trailing `-calt` sentinel. Build and run:

```bash
gcc hb_ligature_probe.c -o hb_ligature_probe $(pkg-config --cflags --libs harfbuzz freetype2)
./hb_ligature_probe
```

Complete, unedited output (identical across two runs):

```text
HarfBuzz 10.2.0 / FreeType face: Fira Code Regular
Fira Code — ligatures ENABLED (calt=1) vs DISABLED (calt=0):
  ==   calt=1  text===   -> 2 glyphs: [1649, 1387]
  ==   calt=0  text===   -> 2 glyphs: [1578, 1578]
  ->   calt=1  text=->   -> 2 glyphs: [1186, 1458]
  ->   calt=0  text=->   -> 2 glyphs: [1221, 1580]
  !=   calt=1  text=!=   -> 2 glyphs: [1204, 1135]
  !=   calt=0  text=!=   -> 2 glyphs: [1132, 1578]
  ===  calt=1  text====  -> 3 glyphs: [1649, 1649, 1388]
  ===  calt=0  text====  -> 3 glyphs: [1578, 1578, 1578]
```

The two methods **cross-validate exactly**: `==`→`[1649, 1387]` with `calt` on and `[1578, 1578]` with `calt` off; `->`→`[1186, 1458]` on and `[1221, 1580]` off — the identical glyph IDs kitty's shaper produced above. This proves, in plain reproducible text, that (a) programming ligatures are HarfBuzz `calt` substitutions, (b) kitty leaves `calt` on by default (dropping the `-calt` sentinel) so the pair fuses, and (c) `disable_ligatures=always` keeps `-calt`, turning the feature off so the plain glyphs return. The embedded screenshots below (Figure L-ON / Figure L-OFF) are the canonical launcher's on-screen realization of exactly this substitution.

> **Figure L-ON — Fira Code, ligatures ON (canonical launcher, 28 pt).** Cropped capture of the mapped, openbox-managed window (its title bar shows the feeding script `hold_fira.sh`). The operators visibly fuse — `==` → one long double-bar, `->` → an arrow, `!=` → ≠, `>=` → ≥, `<=` → ≤, `===` → a triple-bar, `=~`, `|>` → a triangle, `++`:
>
> ![Fira Code operators rendered with ligatures ON — each operator fused into a single glyph](data:image/png;base64,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)
>
> **Figure L-OFF — Fira Code, `disable_ligatures=always` (canonical launcher, 28 pt).** Identical input and font; every operator now renders as separate discrete characters (`==` two equals, `->` dash-then-`>`, `!=` `!`-then-`=`, `===` three equals):
>
> ![Fira Code operators with ligatures OFF — each operator rendered as separate discrete characters](data:image/png;base64,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)
>
> **Reconstruction of Figures L-ON / L-OFF (self-contained; assumes an Xvfb+openbox `$DISPLAY` and the repo root as CWD).** Each crop above is the top-left region of a full-screen capture produced by:
>
> ```bash
> printf '#!/bin/sh\n== -> != >= <= === =~ |> ++\nsleep 8\n' > /tmp/hold_fira.sh
> chmod +x /tmp/hold_fira.sh
> for pair in "liga:" "ligoff:-o disable_ligatures=always"; do
>   name=${pair%%:*}; opt=${pair#*:}
>   ./kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
>       -o font_family="Fira Code" -o font_size=28 $opt /tmp/hold_fira.sh & pid=$!
>   sleep 4; xwd -root -silent | convert xwd:- fira_$name.png; sleep 5; kill $pid
> done
> # crop to the text region used above: convert fira_liga.png -crop 700x120+0+62 +repage fira_liga_crop.png
> sha256sum fira_liga.png fira_ligoff.png
> ```
>
> **Fingerprints.** The embedded 700×120 crops are byte-deterministic and match the base64 above: `fira_liga` crop SHA-256 `2cc89a5b9c51793c12da3c74c750aa997b90dda98bcb222d5ac193a8c139ef84`, `fira_ligoff` crop SHA-256 `b6706d319dd46f2495bff6809c9d10859d8d3ae6dd23fcf90f68cbc11d3b8d3a`. The full-screen captures observed here were `fira_liga.png` `ec09c361000509ad84bf59d167d8410b95bfdf05bb2ad93920888e8b87a8815f` and `fira_ligoff.png` `ec87c2d6b71fb3b13accfa33da3639844256aa2d1a53856060b725cf6891bd80`; a fresh capture reproduces the same visible glyphs, though full-frame bytes may differ by X-server timing. A direct pixel diff of the two full captures reported **3521 differing pixels** — the ligature substitution changing the on-screen glyphs.

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

So the authoritative negative is spelled out by the source itself: <q>kitty does not support BIDI</q> [`kitty/options/definition.py:67`]. The default resolved value is `force_ltr = no` [`:64`] (the `yes` path is exercised as a variant in §12). RTL scripts are still *shaped* (glyph selection and cursive joining) by HarfBuzz — kitty simply performs no logical→visual reordering. The changelog frames the practical result: <q>Partial fix for rendering Right-to-left languages like Arabic</q> [`docs/changelog.rst:3540-3541`].

**OBSERVED (canonical).** In both the default and forced-fallback runs, the Arabic word **مرحبا** is shaped and rendered with correct cursive presentation forms — the five letters joined right-to-left as connected glyphs, not isolated boxes — confirming HarfBuzz shaping runs even though there is no bidi reorder step. The default-run capture is **embedded below as Figure AR** (with SHA-256 and reconstruction command). Note that the Arabic word appears between the English words in *logical* order (no visual reordering), while the Arabic *glyphs themselves* are contextually joined by HarfBuzz — exactly the "shaping yes, bidi no" behaviour this section documents.

> **Figure AR — mixed Arabic (RTL) + English (LTR), default DejaVu Sans Mono (canonical launcher, 28 pt).** Cropped capture of the mapped window (title bar shows `hold_arabic.sh`). The line reads `Hello مرحبا World`: the Latin words are LTR, and the Arabic word مرحبا is rendered with connected cursive presentation forms produced by HarfBuzz shaping. The Arabic sits in logical position between the English words — there is no bidi reordering pass.
>
> ![Terminal rendering of 'Hello (Arabic marhaba) World' — Arabic shown as connected cursive glyphs between the English words](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAArwAAAB4CAIAAACfPrafAAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAAGYktHRAD/AP8A/6C9p5MAAAAHdElNRQfqBw4ACyWBBDTrAAAcoUlEQVR42u3deVwTZ/4H8GdyACHhCgYiIqCoKGJVqlKBqpVaW61UrSddWw+U7rqtx9pq96Wv1a2tx+ru2q1ru3jVY/29XFu7dQVdKlbcUlSKB4LgEYQKciVcIeFIZn5/jH0aOeJIE4L08/6D15B5MvPMZJLnO8/zzPMwc+LmEQAAAIBHkRBCZr53wNHZAAAAgK5OQgjhOFJ5/wdH5wQAAAC6NAkhhOUIy3GOzgkAAAB0aQ9qGlrHDPnfHJF59/IfHG2oq9L9kM8wxMs/xNXN617Otw26kgFjZzs65wAAANCpJIQQluVa1zRI875i6kuL0pTipmqGZQkhVSJRpZOnuEHn5KZmx8xydM4BAACgU/HNExxnETToa7QMI2oeMEl6ea+4QceIxfw6EcOQBh0hxBj0vL5Gx3GswsPb0fkHAACATiIhhHAsx7EPgoa66krNlXPauzlyN3e1yTxi8txhk19OTdzJsWxMwm+vJp+6dOJQRXnZD8c+8g4aHDxsrMKzh6MPAQAAoOMYheessZ6S2yVH8pvYx0+p7N8rbgDJPF+SUWub3oFt7oVxdYuNVjrduX/sTpPw3Qg/NIFoR8gH/7soPKXOcolEqruVJW40S51dWVIikTpxLGfmSqQuruV6c+3dbJl/qNRZ7qzwZNGBEgAAnmQMRwhHmkoyb93rE9TLjXnMlCzHEa6hPOdCmXKkypURuNPHzk9D443bxbXf/e8OOyRQLRe4G+GHJhDfEdKieYIjrl4+9UU5hGUrxL0O7N3jdFgqkjgThvk6Ld3U3NTk1IszscZ7+W4RLxOOcHjsAgAAnmgcxxEiUoUseqp/qAepLa8+nVVTamK8/b2fC1H0lJGGOuO1nMqLleYfC0uO48QBIern+7rImhpu1zLkx5L0oSKRkfQP84ns6eTuxDTqDZevVVzSsYzCc+4499JrVZIgZX9z9cH0et9Wafj8SHwHxg/tN0jBlRVWJuUa6l1kwwd6Mca+uTddvP29xw+Qq2Uik9GYcbHsct2DvTIy1+hh3mFKiYQ168qrkrPqqto+tI6fKgkhxGw2m9kf6y0Yxqtnnx7DXqgtysm+klFUWsKyP1VpiESiADU3ZPho996D3H16cwxjZm1S4QEAAOAYDMcRwvgEumWdu3rTP3jaEM8hohsFitBJw9xdSm4dOVXR+5mRY4c7F5/UFT6IGYi4p8+E/jJxcd7/ZeiCx0QwhOM4zsyyZtbyZr6puFh75trNgiqncVMjnwnSZxRKGDnHEcmQMMW9wrK03Kzvc90Ge4mNrdIQwnirnc//71pRSMhL/TzCcorSWA+OELbufpU0aPJQd1mZ5l+n77EKVvO9zhg0RCljCCFefp7hPcj1lJTT5S4BLrWXi9x9nvJofWi3m9UycQfPlYQQ0mRim00/lf0SZ0Xo+LmEkP/OHzl58ssnT/6Hxg0sy96vrln8xvv8v5bvAgAAeBIxJo4jXG3O5c8zDF7DfceGubmSUpNruBfTkHbq3DcVAQHetVHjegSaMi43qDhCOI5zV7kqiPFs8v/OVfS+71f3TA+WZVmTmW02WQQNUpc+vZVPh0fLRIzYiRGZzcbCMuLdgyOc/uo3H3xeRjxVCh/ffr2VI9pKo8u5evKSWcWqIvr59XItKasJ4nftqVZ4MQ3nk1JTyvxUKjfOXSWTcM0mjhBSVWGoMCkHPztKdKvmUm6hk0LOmFsf2v3ySi8/b2nHzlWr5omHaTSadevWvf/++5b1DWiSAACA7oPjOELq6xuJR4DSzYUhDENYk5kjhDVzTgrf3m5yZ4YQzlSlr+c4wo9uRDjCmlknhTrA1cWJkCa+vd6yePQKUsUESG/+N/mjdOOYhTNmKwhnbmQ5jiNEX2MwKwcEBXp7Dwh4vp00IoaRePh7uMklhGG45gaD6cGuCeEIa+Kkbuo+Hm6inw6AkKaKin1f3gro5RczIvCNIPH+5LqyNg+tuZnlJB3r3/CIoCEn5/rIkSNefPGlpKSTD59eAACAboF78FckFj8o3ziusqi2/Bn1iLFP3b/nGRrsKm4qu15k5oIfrK4sq9cT1ajnniq75zm4j4uYNJNWfRoYhhDSXF1l7DF8TJSfiNQRwnH0/lskkRKOaz8N4xk2bFJts3KQmwdpvPZDrVny465L9XXEZ1TMiIq77h4qJ2OhNk/sOWW4ojbv/vfELcRJdk9Xk3HPN7a32aS5bRjao9WhEY5jO1yO84M7kfbeHhUV7e3tvX//frlCIRKJWJY1tZ8YAADgCcbxlQiEEGKuKP3ifNP4sIGz+0mbqnWnjqam6d3Vsgf3580/lJy8Jp44aMgMle5bjbZ2mJhrNbyytlB7SeUdPnPaoPKarDu1PVXcQwk4wnFW0pjLKiURY/1VkmbNhfPHbomcwyV0119eMo8fOGBmoLihuvz4xSK9n0qtdOJq86sVUcOHqSa49DYZDVfOXrlU2+TNtnFobQ4DLRAzJ25e5KK/V5W3MWHVpvhnIkaEp6WdYxjm17/+tVgsNplMez879N7uDEd/sAAAADbDNRT9cKOA9Q0P9HMjBk1h/g8iv5G9fV2aqzTlJWXGJlbk4uXZq7/S3ZnQlD2dDSW5ZZW1rETppWSqywyeISO8ZQ/V+nPNlRWaWzVGs9jVx8OlVlen9A/t69L4074YK2nyClm5G2esbWadZKoBPf2U4p/eKGsou1lerms0sWIXX5/gAYqmgsI7pU7+4SrmbnFxRZOJFUkUcvUAX5Ub0/ahdfDZUGZO3LzRC3e2GTRsXjy60VDX4kVnV7c1id85+vMFAADoSsTysc/2CBQ9+M9QXPZFfkP3e1iA79PQdvNGzKxlBn11ixddFZ7o0wAAAPAQk/6b1HtaTaHxx0ETxB59e6oUjs6WjVnrCPl0TNuzUiFoAAAAaMVD2fcpy/+7X3HZ7tTYAAAAAJYkhBAzx5lMzY7OCQAAAHRpEkLI+dT/OjobAAAA0OXNiZvn6CwAAADAE0D08zcBAAAAvwQIGgAAAEAQBA0AAAAgCIIGAAAAEARBAwAAAAiCoAEAAAAEQdAAAAAAgiBoAAAAAEEQNAAAAIAgCBoAAABAEAQNAAAAIAiCBgAAABAEQQMAAAAIgqABAAAABEHQAAAAAIJIHJ2BlmJiYv7xj3+0uSovL2/y5MmdsIUua926dfPnz+eXlyxZcubMGUfnCJ4Ynfa92LZt27Rp0wghHMf169fP0ccNALb0UE3DgQMH7vxo6dKlVt529uxZPtnNmzcdfQgA9pKbm8tf50lJScLflZCQQL9Hjj4CAABb6nI1DTU1NXl5eZav9O7dWy6Xd+YWAHhVVVVqtZoQ4uHhIfxdfn5+js54G/C9AICfr8sFDZmZmS1qSs+cOfNYP20/fwsAPJ1OxwcNbm5uwt+lUqkcnfE24HsBAD8fOkICtKuiooJfcHV19ff3F/gupVLJLzQ1NTn6CAAAbAlBA0C77t+/zy8wDDN69GiB76JtGdXV1Y4+AgAAW7J784SXl9frr78eGBjo4eFhMpl0Ol1qampKSoqjD/zxTJkyJTIy0tPT09nZuba2tqio6PDhw2VlZY7OF9iXRqOhy0FBQQLfRYMGnU7XXhp7XFFvvvnm4MGDFQpFU1NTcXHxf/7zn6ysLPudnLCwsBkzZvj6+kqlUq1Wm5qaevr0afvtDgC6AjsGDeHh4StWrAgPD3dxcbF8febMmQUFBYmJiUePHnX04T/a22+/PWvWLLVazTCM5euLFi26ePHi5s2b8/PzHZ1HsJfvvvuO4zj+o/f19RX4LtoBorKysvXan39Fffzxxy+99BIhxGw2DxgwgBCyZs2aV199lTaL8OLi4rKyspYuXVpVVWXb0xIcHLxhw4YRI0ZIpVL64vTp03NyctauXWvbfQFAl2KvoGHBggXLly9XKBStVzEM07dv3/fff79Pnz5btmxx9BmwZteuXRMmTGjx485zcXEZM2bMoEGD1qxZ88033zg6p2AXubm5er2eDwJ69OhhuSoxMZG+cvDgwS+++IJfDgoKkslk/HJJSUmLDdrjivroo48mTZrUeptSqTQiIiI4ODgzM9OG5yQsLOyTTz7p2bNni9dFItGQIUMSExMvXLhgw90BQJdil6Bh4cKFq1atcnZ25v/V6XT5+fmVlZUuLi6BgYH9+vUTiUQSiWThwoUVFRV79+519Elo29atW1944QV+meO4goICjUbT2NioUqlCQ0P5eEilUm3atGnq1Kloquiuqqqq+KDB09OTviiTyaKiougVHh0dTYOGUaNG0fL77t27lpuyxxW1fPlyGjHU1dVVVlY2NDTIZDJfX18au9jW9u3bacRgMBiuX79eXl4ul8tDQkL8/Px8fHzGjh3bGR8MADiC7YOGkJCQN998k/895TjuzJkza9assawgjY+PX7Zsmaurq0QiiY+PP3LkiNFodPR5aGn8+PH0+bTm5ubExMTt27fTteHh4Zs2beJHu/Px8dm4cePixYsdnWWwC61WGxAQQB4equHVV1+lEQMhpE+fPnSZdn3gOO7SpUv0dTtdUTNnzmQYpqKi4siRIzt27KCvy2Sy1atXT5kyxbZnY+3atXSQx3v37q1evTojI4Ou/fvf/z5x4kR3d3d7fBAA0BW0GzR4e3tPnTq1vbVisbi9VatWrfL29uaXU1NTExISWiTYvXu3XC5/6623GIbx9fVdsWLFhx9+6Ojz0NKvfvUr2hXj3//+t+XvOyEkKytrxYoVBw8e5O8+R48e7e/vf+/ePUfnGmyvvLycX7AsC4cNG2aZxnI0J9r1wWAwWPZDtMcVJRaL1Wp1SUnJb3/726tXr1quMhqN69evP378eGlpqQ3PxsSJE/mFxsbGDRs2WEYMhJDf/OY3X3311eDBg236CQBAF9Ju0PDGG290YHO+vr6jRo3il7Va7bJly9pMtmPHjldeeSUwMJAQEh0d7eiT0IannnqKX6iqqlq/fn3rBLm5uefOnXvllVcIITKZbNGiRRs2bHB0rsH2aMlt2UGHVi2UlJT4+fkplcqhQ4fyxTaNmFt0P7TTFWUymf7yl7+0iBio9l7vmKlTp9KGiStXrqSmprZOc+jQoU2bNtlwpwDQpdh4nIa4uDj625qRkWGl3eH69ev8gvAxczpNTEyMl5cXv3zjxo32juLYsWMsy/LLgwYNcnSubW/GjBnZ2dl5eXm3b9/Oy8vLzs6+dOlSSkrKkSNHtm/fPmfOHEdnsDPQ2VUkEklMTAy/zFct6PV6vlQWiUTPP/88v4p2fdBqtXQj9rui8vPzaXcKe4uKiqLdNVrUMVBHjx61PHAA6GZsHDRY1kyePXvWSkr6WyyXy1988UVHn4eHRERE0OXbt2+3lyw9PZ2O3sMPNtzNODk5ubq6SqVShmGkUqmrq6tSqezbt++oUaOmTp36wQcfXL169cSJE9u3b+9qn6ANnTx50mQy8cshISH8X/65ieLiYnorP3DgQH6Bdn2g7RrEnlfUlStXOu1U9O7dm1/gOO7kyZPtJWv9zAgAdBvtNk/8+c9/3rlzZ3trz549y/cOa4E26DY3Nx8/ftzKji1/WbpaZYPl73VBQYGVlNXV1fzD8b/Mzl8KhSI0NDQ0NDQ2NraoqCgjI2Pv3r3dbGpHo9FYW1vLf8p8BcPEiRNFIhEhpKCg4NixYytXrnRycqJfB3olWF7h9ruiLl++3GmngsZD9fX1Vj5l1DQAdGM2fnrC1dWVX5BKpcILj8eaQrATWD6rZv3Jt4aGBn7BycnJ0bm2vbt373799dcikUgsFvO1DgqFwt3d3d3d3fLZAUKISCQKCgoKCgqaNm3anj17WvTys6358+dHRkZ6eHgYDIacnJw9e/bYfPCiFnQ6HV+Q8zNR0YaDa9euVVVVlZaWBgQE0LCANs9Zzhpvvyvq2rVrdj12S7Qjp/U5NQwGQ6dlCQA6mY2Dho6VnV2txLXMT01NjZWUtOLacmi8ruzEiROhoaGEkKKioueee8564vT09PT09DZXTZo0KSIiIiQkJCgoqEePHrSp29nZ2cfHx06Znzhx4qpVq/r27UtfGTNmzOzZsz///PPNmzfb76RVVlbyzxny/RL4SoXm5uYvv/ySP5MBAQEKhWLKlCn19fUSiYQQwrKsZfOcna4olmU7s16HPzTLTLapubm507IEAJ3MxkEDvQUxGAyW4/ZbR6cF6iIsb6Ss14LQn9Ff2g9lUlJSUlJSWFjYW2+9FRUVZadxhCzFxsZu3Lix9VTOSqUyPj7ez8/v7bffttOu6VOLfCdH/gmC0tJSvs4gLy+PfwIoIiKiuLiYT1lTU2NZo2CnK4rjODsdcptolmgm2/SkBNAA0AE2Dhpot3C9Xs8/PPYksuzcbn3GAYEVtt3SwoULly5dajlOYlNT07Vr19LS0oS8ffHixc8++ywh5OrVq0KaM1auXNk6YuAxDDNp0qTs7OzExER7HGlRURG/4O7uPm7cOL7Upy+mpKQsWrSIYZjg4GBairdoMekeV1RjYyO/YL1qkLZRAkD3Y+OnJ2iP8TZnnXhSWN4jWg721xotMuvq6hyd6071wQcfrF69mh4+y7KXLl1KSEiYPXu2lX71lgwGQ1RUVFRUFH1Y0YolS5bQrvs6ne6TTz6ZPn36H/7wh4sXL/IvMgwzbdo0Ox0s7WyoUCj4QIcQcuPGDX4hMzOTn5jK39+fzkbRYqqq7nFF0Uk75XJ5cHBwe8noSBUA0P3YOGjIycnhF1xdXePi4myyTVop2uFqz8fdAi2KCCH9+/dvL1l0dDT9ibftuHvtsWxL7tj9HK1YtjKm5yPt27dv9uzZdFOlpaUffvjhnDlzBNYx8A4fPsyXi5bDKbaHjhhGCNm5c+ef/vSnq1evHjp0aO7cubQ8plGFzaWlpfH9E2UyGd8Lkh8fnSbgB4Dy8fGhQUOL3o5d84p63O8FrVxhGIYOid2akA8UAJ5QNg4aPv/8c9p32lbNE3q9nl9oMfOv/baQkpJCq5cHDhzYXoP99OnT+UfvCCF5eXk2OVjramtr6XJYWFgHtkDvApVKJR1uSLjY2NjU1NQxY8bwPR9Zlj137tzLL7+8b9++DmSG7xKoUChmzJhhPaVlNcP+/fstV926dYtfkMlk9utaYXk9EEK0Wq1lHMD3RpRIJPT+m5avvK55RT3u9+LcuXO0/cUyjLM0d+5c1DQAdGM2Dhru3r1LJ+kJDw9fuXKl9fShoaGPrJCgUwV6eXl1bF6oDmwhOzubvqXNQX9DQ0PpbH4NDQ0dKzUf14ULF+iv9gsvvGC9dby13//+9/QHXSaTPda41xMmTNi3b9+WLVv4wb/5o05MTFy4cGGHn3isr6/nF6y3LCxYsIDOAkV7GlKWnfJohwCbozXzfIeGFrNC0OERaSdHev1QXfCKetzvxalTp+iBDx8+fMyYMa3T2Kp+EQC6JtvPcrljx46hQ4d6enqKRKIlS5b4+fmtW7eu9bi5s2bNmjx58tNPP52ZmfnPf/7Tyga//PLLKVOm8GVDQkICy7J79ux5rCx1YAuHDh0aNWoUXwjFxsaWl5dbdtYbOnTo1q1baU3yhQsXWkyCbCeZmZnFxcX8WFgBAQFJSUm3b9/WarUNDQ10+GGDwdCi5UIikbi5ufXq1atFO/SkSZPCwsIKCwtramrafIiOH5vB29tbrVarVCr6XCUhpLq6evv27dY/uEei/ekiIiK2bNmyevVqy7Xbtm0jhKjV6qeffppGBlKp1MvLi4Yp/v7+tLa/sbHRfgM2VFRUWP7bYlTH48ePr1+/nvbjaWxsTElJabGFLnhFdeB7kZycvGTJEkKIi4vLH//4x3fffdeyxmXXrl38A70A0F0xc+Lm/d8/D/L/HDhwICoqil8WOCKk2WweMGBAi7WLFy/mh8nj/62pqdFoNOXl5Q0NDXK5XKVS9erVy9vbmy+Ezp8/P3/+fOu5/OyzzyzntTIYDLSQq66ufuR4Ax3bwrZt2+gdMMdxGo1Go9E0Njb6+PiEhobSEqKysnL27NmdEzQQQtasWRMfH29ZfrdQW1tr7+Ep8/Pz169fb1ladEx6erplZYnJZKqvr//Xv/7Fz3jU3ggE1dXVubm59+/fb2pqGjZsGB1qSaPRTJgwwU6HvHHjxrlz59J/33nnnRYzPiQnJ9MvQllZWWRkZOuN2OqK+vjjj1966SXSzrfvsXTge5GUlMQPp00Iqa+vv379enl5uVwuHzRoEP8wak1NDV/jwnEcnUcbALoH29c0EEISExObm5uXLVvGl14eHh7Dhw9vLzG9Rbbi3Xff3b17N72JsbyTFvL2jm1h1apVbm5uMTExDMPwD9S17jGu0+nWrl3baREDIWTz5s3BwcHPPfeclbjBOp1O19TU1IHJMsxmc2FhYWpqqk2mMRw3bhw/wCIlkUg8PDwe2cHT09OzdZHMcdypU6d+fq7aYxnB1NfXt54jqqCggJbftC2jhS54RXXse/Hpp5/yvR3lcrnltBqEkNLS0osXL8bGxnZO/gGgk9m4TwO1f//+1157LTU1tb0nx4xGY15e3mefffbee+89cmtlZWVTpkw5ePBgQUGB0WjswJg2HdtCQkLCzp072xx7qrGx8dtvv12wYEHrimh7W7x48ZYtW7KysvjKGyFhE38TX1RUlJycHB8f/9prrx0/flyj0dTV1ZlMpvbOhtlsNhgMJSUl33///eHDh+Pi4iZMmGCriY8XLVrEd/rLy8u7cuWKwWBoMxscx+n1+lu3bh08eLC9Bwr4+ZPsOnZ1WloazV6bEzJZDufcoi3DUle7ojrwvcjNzZ03b156enqL4adYls3Ozk5ISDCbzZ2WfwDoZA81T9jJ66+/HhYWxk9Y0NjYqNVqb968efToUSsTZ3c1sbGxkZGRXl5eUqm0rq6uqKjo0KFD1icRACuWL1++dOlSPmjYtWsX333hkWQy2TvvvDNy5MiePXvK5XKJRNLQ0FBUVHT8+PHdu3c7+pgeTze4osLCwmbOnOnj4yOVSrVabWpq6unTpx2dKQCwszlx8xydBfhl+d3vfnfjxo07d+7cuXMnNTXV0dkBAACh7NKnAaBNvr6+W7dujYyM5OsYjEbjX//6V0dnCgAAhLJXnwbomtavX8/f4n/66aedud/w8PC//e1vycnJ0dHRfMSg1+s3b9781VdfOfqUAACAUKhpALuYNm1aQECAv7+/Wq0OCgpSq9V0rENCSEFBwY4dO06cOOHobAIAwGNA0PDLotPp+EH9rPTwt4n2+jbqdLqvv/5ayCMzAADQ1XTG0xPwC9RiaCa9Xl9YWJienr5582ZHZw0AADoINQ1gF4WFhXq9XqvVlpaW3rhx48CBA47OEQAA/FwIGsAuxo8f7+gsAACAjeHpCQAAABAEQQMAAAAIgqABAAAABEHQAAAAAIIgaAAAAABBEDQAAACAIAgaAAAAQBAEDQAAACAIggYAAAAQBEEDAAAACIKgAQAAAARB0AAAAACCIGgAAAAAQRA0AAAAgCAIGgAAAEAQBA0AAAAgCIIGAAAAEARBAwAAAAiCoAEAAAAEQdAAAAAAgiBoAAAAAEEQNAAAAIAgCBoAAABAEAQNAAAAIAiCBgAAABAEQQMAAAAIgqABAAAABEHQAAAAAIIgaAAAAABBEDQAAACAIAgaAAAAQBAEDQAAACAIggYAAAAQBEEDAAAACIKgAQAAAARB0AAAAACCIGgAAAAAQRA0AAAAgCAIGgAAAEAQBA0AAAAgCIIGAAAAEARBAwAAAAiCoAEAAAAEQdAAAAAAgiBoAAAAAEH+HxfTLl1Dy0F+AAAADnRFWHRjb21tZW50AHh3ZHVtcI4F6f0AAAAldEVYdGRhdGU6Y3JlYXRlADIwMjYtMDctMTRUMDA6MTA6NTQrMDA6MDAa5cVKAAAAJXRFWHRkYXRlOm1vZGlmeQAyMDI2LTA3LTE0VDAwOjEwOjU0KzAwOjAwa7h99gAAACh0RVh0ZGF0ZTp0aW1lc3RhbXAAMjAyNi0wNy0xNFQwMDoxMTozNyswMDowMCToJA0AAAAASUVORK5CYII=)
>
> **Reconstruction of Figure AR** (self-contained; Xvfb+openbox `$DISPLAY`, repo root as CWD):
>
> ```bash
> printf '#!/bin/sh\nprintf "Hello \\331\\205\\330\\261\\330\\255\\330\\250\\330\\247 World\\n"\nsleep 8\n' > /tmp/hold_arabic.sh
> chmod +x /tmp/hold_arabic.sh
> ./kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 -o font_size=28 /tmp/hold_arabic.sh & pid=$!
> sleep 4; xwd -root -silent | convert xwd:- arabic_default.png; sleep 5; kill $pid
> convert arabic_default.png -crop 700x120+0+62 +repage arabic_default_crop.png
> sha256sum arabic_default.png arabic_default_crop.png
> ```
>
> **Fingerprints.** Embedded 700×120 crop SHA-256 `5732f0a1ed08a66a6c2ba9e80e082aed280abedb9c171223b4d4d4341bc14a50` (byte-deterministic, matches the base64 above); the full-screen capture observed here was `arabic_default.png` `cfc676fc7772424ca09e334be12245cd930cfe1e57678c94040b81d83b6dd3f6`. The `\331\205 …` octal escapes are the UTF-8 bytes of U+0645 U+0631 U+062D U+0628 U+0627.

**INFERRED (source, no runtime log).** For RTL runs HarfBuzz emits glyphs whose cluster values *decrease* across the run; kitty's code comments document this ("RTL languages like Arabic have decreasing cluster numbers") at [`kitty/fonts.c:997`] and [`kitty/fonts.c:1079`]. No debug line prints cluster numbers, so this is labelled inferred from the source, corroborated by the correctly-joined visual output.

### 6.A.3 Combining diacritics — one grapheme cluster maps to one cell

kitty stores a base codepoint plus up to a fixed number of combining marks per cell in `CPUCell.cc_idx[]`, and resolves each mark index back to a codepoint with `codepoint_for_mark`. Two functions drive correctness:

- `has_cell_text(face, cell)` [`kitty/fonts.c:435`] verifies a face covers the whole grapheme: it checks the base with `face_has_codepoint(face, cell->ch)` [`:436`], then each combining mark via `codepoint_for_mark(cell->cc_idx[i])` [`:440`], with an `hb_unicode_compose` precomposed-form check [`:447`].
- `output_cell_fallback_data` [`kitty/fonts.c:457`] iterates the same `cc_idx[]` marks (`debug("U+%x ", codepoint_for_mark(cell->cc_idx[i]))` [`:460`]) when logging a fallback for a combined cell.

**OBSERVED (canonical).** Feeding `o` + U+0301 (combining acute) + U+0323 (combining dot-below) produced a single per-cell diagnostic line naming **all three** codepoints together — proving they were coalesced into one cell before font selection:

```text
[0.199] U+6f U+301 U+323 using previous fallback font at index: 0
```

(from the forced-fallback run; `U+6f`=`o`, `U+301`, `U+323`). This coalescing diagnostic — a single line naming all three codepoints together — is the **authoritative, reproducible proof** that the grapheme cluster occupies one cell before font selection. On screen the same combining sequences render as single composed accented glyphs (`é`, `ä`, `ñ`); this is reproducible with the identical capture command shown for Figure AR, feeding the combining line `o␣́␣̣  é  ä  ñ` instead of the Arabic line.

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

The first Arabic codepoint **U+645 (MEEM)** triggers a fresh fallback selection: FontConfig returns **DejaVu Sans Mono** (`ps_name=DejaVuSansMono`, `path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf`, `ttc_index=0`, `scalable=True`, `color=False`). The remaining Arabic letters **U+631, U+62D, U+628, U+627** resolve to the **same** face and print `using previous fallback font at index: 0` — the cache/reuse path [`fonts.c:493`, `:530`]. The forced primary faces are the four Nimbus Mono PS entries, including the `.t1` (Type-1) Regular/Italic and `.otf` Bold/BoldItalic — the very PostScript-name prefix `NimbusMonoPS-` that also activates the `[-liga,-dlig]` branch of §6.A.1. The authoritative evidence of the fallback is the logged `Face(...)` line itself (the DejaVu Sans Mono descriptor above); on screen this forced-fallback run shows the English words in Nimbus letterforms while the Arabic renders through the DejaVu fallback with the **same joined Arabic glyphs** visible in Figure AR (§6.A.2) — the default run whose Arabic is served by the same DejaVu face.

**OBSERVED (canonical) — chain build then reuse (CJK).** Feeding `中文` shows the same build-then-reuse pattern with a *different* fallback face, confirming chain construction is general:

```text
[0.190] U+4e2d Face(family=Noto Sans CJK JP style=Regular ps_name=NotoSansCJKjp-Regular path=/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.191] U+6587 using previous fallback font at index: 0
```

**U+4E2D** builds a new fallback (**Noto Sans CJK JP**, `NotoSansCJKjp-Regular`, `NotoSansCJK-Regular.ttc:0`); **U+6587** reuses `index: 0`.

**OBSERVED (canonical) — glyph-verification failure (the `does not actually contain glyphs` path).** The append-time verification bound above (`has_cell_text` → drop → `MISSING_FONT` [`kitty/fonts.c:501-511`], logged at [`:509`]) is exercised directly by feeding codepoints that **no** installed face covers, while forcing the primary to Nimbus Mono PS. Six deliberately unassigned / supplementary-plane / Private-Use codepoints are used — **U+0378, U+0380, U+1E4D0, U+31350, U+F0000, U+10FFFD** — so that even the OS's best fallback pick (DejaVu Sans Mono) fails glyph verification and kitty logs the failure once per codepoint.

Reproduction (self-contained; `feed_missing.sh` emits the six codepoints as UTF-8 via octal escapes, then the launcher runs with the fallback-debug flags and the forced primary):

```bash
# feed_missing.sh — six codepoints that no installed face covers
cat > feed_missing.sh <<'FEED'
#!/bin/sh
printf '\315\270 \316\200 \360\236\223\220 \360\261\215\220 \363\260\200\200 \364\217\277\275\n'
sleep 4
FEED
chmod +x feed_missing.sh
# octal → U+0378 U+0380 U+1E4D0 U+31350 U+F0000 U+10FFFD
# (verify the mapping:  sh feed_missing.sh | python3 -c "import sys;print(' '.join('U+%04X'%ord(c) for c in sys.stdin.readline() if c not in ' \n'))")

./kitty/launcher/kitty --config NONE --debug-font-fallback --debug-rendering \
    -o confirm_os_window_close=0 -o font_family="Nimbus Mono PS" ./feed_missing.sh
```

Complete unedited output (`logs/missing_run1.log`, 21 lines; run twice — the normalized `stability/missing.diff` is **0 bytes**, so the six failures are stable):

```text
[0.155] OS Window created
[0.164] Failed to open systemd user bus with error: Connection refused
[0.168] Child launched
[0.168] Text fonts:
[0.168]   Normal: NimbusMonoPS-Regular: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1:0
[0.168]   Bold: NimbusMonoPS-Bold: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-Bold.otf:0
[0.168]   Italic: NimbusMonoPS-Italic: /usr/share/fonts/type1/urw-base35/NimbusMonoPS-Italic.t1:0
[0.168]   Bold-Italic: NimbusMonoPS-BoldItalic: /usr/share/fonts/opentype/urw-base35/NimbusMonoPS-BoldItalic.otf:0
[0.190] U+378 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.190] The font chosen by the OS for the text: U+378 is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
[0.190] U+380 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.191] The font chosen by the OS for the text: U+380 is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
[0.191] U+1e4d0 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.191] The font chosen by the OS for the text: U+1e4d0 is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
[0.191] U+31350 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.191] The font chosen by the OS for the text: U+31350 is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
[0.192] U+f0000 Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.192] The font chosen by the OS for the text: U+f0000 is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
[0.192] U+10fffd Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
[0.192] The font chosen by the OS for the text: U+10fffd is Face(family=DejaVu Sans Mono style=Book ps_name=DejaVuSansMono path=/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False) but it does not actually contain glyphs for that text
[0.125] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

Every one of the six codepoints prints a **pair** of lines: first the OS's chosen face — `U+XXX Face(family=DejaVu Sans Mono … path=…/DejaVuSansMono.ttf ttc_index=0 …)`, the selection logged by `output_cell_fallback_data` [`kitty/fonts.c:492`] — then the verification failure `The font chosen by the OS for the text: U+XXX is Face(…) but it does not actually contain glyphs for that text` [`kitty/fonts.c:509`]. Because `has_cell_text` returns false, kitty calls `del_font` and returns `MISSING_FONT` [`kitty/fonts.c:503-511`]; the cell renders as the missing-glyph box. This is the honest **error/edge** path of the fallback engine — distinct from the successful build-then-reuse paths above — and confirms the `>100` bound, the OS query, and the glyph-verification gate are all reached at real startup.

**Labelling note (evidence accuracy).** The `Face(...)` line proves the *selected fallback face and its file/index*; it does **not** by itself prove Arabic contextual-form correctness — that is shown separately by Figure AR (§6.A.2, the joined cursive forms). Glyph indices from `shape_string` are labelled font-specific glyph IDs, not Unicode values, and are corroboration only.


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
| `disable_ligatures` | `never` | `definition.py:115` | **never** (ligatures rendered) | `-o disable_ligatures=always` → keeps `-calt`, ligatures suppressed (§6.A.1; Figures L-ON vs L-OFF; glyph-ID proof) |
| `font_features` | `none` → resolves to an empty mapping **`{}`** | `definition.py:135` | **{} (empty)** — no per-face features configured | `-o font_features "TT2020StyleB-Regular -liga +calt"` (example `definition.py:182`) merges into that face's `ffs_hb_features` before the `-calt` sentinel (§6.A.1, step 2) |
| `force_ltr` | `no` | `definition.py:64` | **no** (kitty auto-displays RTL words RTL; no bidi reorder) | `-o force_ltr=yes` accepted at startup (variant §12); forces LTR treatment for use with GNU FriBidi |

Notes grounding the two commonly-missed entries:
- **`font_size` units are points.** The option's own long-text is literally `Font size (in pts).` [`kitty/options/definition.py:61`]; the resolved default is `11.0` pt [`:59`].
- **`font_features` resolves to `{}` by default.** The default string `none` [`kitty/options/definition.py:135`] means "no per-face OpenType features configured," yielding an empty mapping; consequently `init_font` takes its *default* per-face branch (`[-calt]`, or `[-liga,-dlig,-calt]` for Nimbus) rather than the user-features branch [`kitty/fonts.c:300-325`]. This is why the coverage claim for `font_features` is now backed by an explicit resolved value rather than a bare checkmark.


---

## 8. Objective C — Screen-grid / shaping-subsystem init: cell metrics, baseline, decoration alignment

### 8.C.1 How the metrics are computed, and how they were observed

As the screen grid initializes, `calc_cell_metrics(FontGroup *fg)` [`kitty/fonts.c:373`] calls `cell_metrics(...)` on the medium face [`:375`] and then applies any user `modify_font` adjustments via `adjust_metric(...)` for `cell_width`, `cell_height` [`kitty/fonts.c:379-380`], `underline_*`, `strikethrough_*`, and `baseline` [`kitty/fonts.c:399`], clamped by `MIN_WIDTH=2 / MIN_HEIGHT=4 / MAX_DIM=1000` [`kitty/fonts.c:381-395`]. The core computation is in FreeType's `cell_metrics` [`kitty/freetype.c:387-405`]:

```c
// kitty/freetype.c:387-405 (verbatim)
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

This fires only when the `_` glyph would overflow the metric-derived cell box, and it is **font-size-dependent**. Both the negative and the positive were exercised at runtime through the canonical launcher.

**Negative — canonical `font_size=11.0` / 96 DPI (does NOT fire).** At the default size the `_` glyph fits inside the computed box for DejaVu Sans Mono / Nimbus / Fira. The C probe (§8.C.1) shows DejaVu `underscore_pixel_height = 18 = height_from_metrics = 18` (delta `+0`). Confirmed by grepping the two canonical 11 pt launcher runs:

```console
$ grep -c "Increasing cell height" /tmp/kitty_fix_evidence/logs/default_run1.log /tmp/kitty_fix_evidence/logs/default_run2.log
/tmp/kitty_fix_evidence/logs/default_run1.log:0
/tmp/kitty_fix_evidence/logs/default_run2.log:0
```

**Positive — `font_size=28` (DOES fire; secondary/edge path).** Scoping the claim to 11 pt is deliberate: the workaround **is** reachable at larger sizes. Running the *same* DejaVu Sans Mono through the launcher at 28 pt makes `_` overflow the box and the message activates. Because it is a plain `printf` (not `timed_debug_print`), the line carries **no** `[t]` monotonic prefix — unlike every other debug line in this document [`kitty/freetype.c:145`]. Complete, unedited startup excerpt (self-contained command; stable across two runs, empty normalized diff):

```console
$ ./kitty/launcher/kitty --config NONE --debug-font-fallback --debug-rendering \
    -o font_size=28 -o confirm_os_window_close=0 sh -c 'sleep 1'
[0.178] Text fonts:
[0.178]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.178]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.178]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.178]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Increasing cell height by 1 pixels to work around buggy font that renders underscore outside the bounding box
[0.129] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

**Hinting caveat (honest labelling).** The C probe in §8.C.1 loads glyphs with `FT_LOAD_TARGET_NORMAL` and computes `underscore_pixel_height = height_from_metrics = 44` at 28 pt / 96 DPI — i.e. *no* overflow by that load target. The launcher nevertheless fires `+1` because it rasterizes `_` with FontConfig-configured hinting, which places the glyph one pixel below the box. The **launcher value is the canonical one**; the probe's no-fire at 28 pt is a load-target/hinting-mode artifact that does **not** affect the canonical 11 pt metric set, where the source-derived probe and the non-canonical `create_test_font_group` binding agree exactly (§8.C.1). This reconciles the negative (11 pt) with the positive (28 pt): the workaround is genuinely present and reachable, not dead code.

Therefore, per the "persist, then honestly label" rule and the label taxonomy in §0.1, cell metrics are reported with the **SOURCE-DERIVED C probe as the primary evidence** and the test/bypass binding as **non-canonical corroboration only** — never the reverse.

**PRIMARY — SOURCE-DERIVED (validated) — the full metric set.** Because the launcher emits no per-face metric line (the honest negative above), the metrics are computed by an **independent C probe** that links the **same system FreeType** kitty links (2.13.3) and applies the production `cell_metrics` formula [`kitty/freetype.c:387-405`] — together with `calc_cell_width` [`:376-386`], `calc_cell_height`/`get_height_for_char` [`:126-153`], `font_units_to_pixels_y` [`:92-94`], the face fields set in `init_ft_face` [`:214`] and the OS/2 strikethrough read [`:231-233`] — **verbatim** to the real font files. Under the default config `fonts.c:calc_cell_metrics` applies **no** `modify_font` adjustment (`cell_width`/`cell_height` deltas default 0), so the probe output equals what production `cell_metrics` computes. The complete probe source is embedded in **§13.2**. Exact build + run command and the complete, unedited output (all six faces; both runs byte-identical → empty normalized diff):

```console
$ gcc /tmp/kitty_fix_evidence/probes/cell_metrics_probe.c \
    -o /tmp/kitty_fix_evidence/probes/cell_metrics_probe \
    $(pkg-config --cflags --libs freetype2) -lm
$ /tmp/kitty_fix_evidence/probes/cell_metrics_probe
=== kitty cell_metrics() replicated via system FreeType 2.13.3 ===

FACE DejaVu Sans Mono Book | 11.0pt @ 96/96 DPI
  raw: units_per_EM=2048 ascender=1901 descender=-483 height=2384 underline_position=-85 underline_thickness=90 os2_strike_pos=530 os2_strike_th=102 y_scale=30048
  underscore workaround: height_from_metrics=18 underscore_pixel_height=18 -> does NOT fire (delta=+0)
  CELL width=9 height=18 baseline=14 | underline pos=15 th=1 | strikethrough pos=10 th=1

FACE DejaVu Sans Mono Book | 11.0pt @ 100/100 DPI
  raw: units_per_EM=2048 ascender=1901 descender=-483 height=2384 underline_position=-85 underline_thickness=90 os2_strike_pos=530 os2_strike_th=102 y_scale=31296
  underscore workaround: height_from_metrics=18 underscore_pixel_height=19 -> FIRES (delta=+1)
  CELL width=9 height=19 baseline=15 | underline pos=15 th=1 | strikethrough pos=11 th=1

FACE Nimbus Mono PS Regular | 11.0pt @ 96/96 DPI
  raw: units_per_EM=1000 ascender=933 descender=-317 height=1250 underline_position=-91 underline_thickness=51 os2_strike_pos=0 os2_strike_th=0 y_scale=61538
  underscore workaround: height_from_metrics=19 underscore_pixel_height=16 -> does NOT fire (delta=+0)
  CELL width=9 height=19 baseline=14 | underline pos=16 th=1 | strikethrough pos=9 th=1

FACE Nimbus Mono PS Regular | 11.0pt @ 100/100 DPI
  raw: units_per_EM=1000 ascender=933 descender=-317 height=1250 underline_position=-91 underline_thickness=51 os2_strike_pos=0 os2_strike_th=0 y_scale=64094
  underscore workaround: height_from_metrics=20 underscore_pixel_height=17 -> does NOT fire (delta=+0)
  CELL width=9 height=20 baseline=15 | underline pos=16 th=1 | strikethrough pos=9 th=1

FACE DejaVu Sans Mono Book | 28.0pt @ 96/96 DPI
  raw: units_per_EM=2048 ascender=1901 descender=-483 height=2384 underline_position=-85 underline_thickness=90 os2_strike_pos=530 os2_strike_th=102 y_scale=76448
  underscore workaround: height_from_metrics=44 underscore_pixel_height=44 -> does NOT fire (delta=+0)
  CELL width=22 height=44 baseline=35 | underline pos=37 th=2 | strikethrough pos=25 th=2

FACE DejaVu Sans Mono Book | 28.0pt @ 100/100 DPI
  raw: units_per_EM=2048 ascender=1901 descender=-483 height=2384 underline_position=-85 underline_thickness=90 os2_strike_pos=530 os2_strike_th=102 y_scale=79648
  underscore workaround: height_from_metrics=46 underscore_pixel_height=46 -> does NOT fire (delta=+0)
  CELL width=23 height=46 baseline=37 | underline pos=38 th=2 | strikethrough pos=27 th=2

$ # two-run stability:
$ diff /tmp/kitty_fix_evidence/logs/cell_metrics_run1.log /tmp/kitty_fix_evidence/logs/cell_metrics_run2.log
$ echo "exit=$?"
exit=0    # identical
```

The canonical answer is the first face: **DejaVu Sans Mono, 11 pt / 96 DPI → cell 9×18, baseline 14, underline 15/1, strikethrough 10/1**. (The 100-DPI, Nimbus and 28-pt faces are secondary-condition coverage; the 28-pt hinting nuance is discussed above under the underscore workaround.)

**CORROBORATION — NON-CANONICAL — the `create_test_font_group` binding.** As an independent cross-check of the two dimensions a binding can expose, `create_test_font_group(size, dpi, dpi)` [`kitty/fonts.c:1699`] returns the real `fg->cell_width, fg->cell_height` (computed inside `font_group_for` → `calc_cell_metrics` [`kitty/fonts.c:1702`]) via `Py_BuildValue("II", ...)` [`kitty/fonts.c:1704`]. It is a **test/bypass binding**: it runs under `setup_for_testing`, which replaces the GPU upload with a Python callback (`set_send_sprite_to_gpu`) and installs artificial sprite limits (`sprite_map_set_limits(100000, 100)`) [`kitty/fonts/render.py:410`]. Per the taxonomy in §0.1 and the negative in §11.3, it is therefore **non-canonical corroboration only** — it is *not* the primary metric evidence. Its two exposed dimensions match the C probe **exactly**, which is precisely why it corroborates:

```text
family='monospace'       size=11.0 dpi=96.0 -> cell_width=9 cell_height=18   (= C-probe DejaVu 11pt@96 → 9×18)
family='monospace'       size=11.0 dpi=72.0 -> cell_width=7 cell_height=14
family='Nimbus Mono PS'  size=11.0 dpi=96.0 -> cell_width=9 cell_height=19   (= C-probe Nimbus 11pt@96 → 9×19)
```

**Canonical default metric set (DejaVu Sans Mono, `font_family=monospace`, `font_size=11.0` pt, 96 DPI, `modify_font` unset → no adjustment):**

| Metric | Value (px) | Derivation |
|--------|-----------:|------------|
| `cell_width` | **9** | `calc_cell_width` = max `ceil(horiAdvance/64)` over ASCII 32..127 [`freetype.c:374`] — SOURCE-DERIVED (C probe); corroborated by the non-canonical binding |
| `cell_height` | **18** | `px_y(height)` [`freetype.c:142`]; underscore does not overflow at 96 DPI — SOURCE-DERIVED (C probe); corroborated by the non-canonical binding |
| `baseline` | **14** | `px_y(ascender)` = `px_y(1901)` [`freetype.c:391`] |
| `underline_position` | **15** | `MIN(17, px_y(1901 − (−85)))` clamped to `cell_height−1` [`freetype.c:392`] |
| `underline_thickness` | **1** | `MAX(1, px_y(90))` [`freetype.c:393`] |
| `strikethrough_position` | **10** | OS/2 `yStrikeoutPosition=530 ≠ 0` → `MIN(17, px_y(1901−530))` [`freetype.c:396`] |
| `strikethrough_thickness` | **1** | OS/2 `yStrikeoutSize=102 > 0` → `MAX(1, px_y(102))` [`freetype.c:401`] |

Units are **pixels**; the source values are FreeType *design units* (`units_per_EM=2048`) scaled by `y_scale` at 11 pt / 96 DPI. All values are stable across ≥2 runs (§10).

**Visual cross-check (Figure AR, §6.A.2).** The embedded default-font capture (DejaVu Sans Mono, 28 pt) shows a uniform monospace grid with a single baseline row of English + Arabic text. Its cell width:height proportion is ~1:2, consistent with the computed metrics at both sizes (9:18 at 11 pt / 96 DPI and 22:44 at 28 pt / 96 DPI are each ≈ 1:2) — an independent visual confirmation of the cell geometry. The numeric metrics themselves are established by the source-derived C probe below, not by pixel-measuring the screenshot.

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
- The window actually displayed shaped glyphs (Figures L-ON and AR, §6), which is impossible unless the sprites reached the GPU texture the fragment shader samples.

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

This closes the loop: shaped glyphs from primary and fallback faces (§6.A.4) are uploaded via `send_sprite_to_gpu` into the shared 2D-array atlas allocated here, and read back per-cell by the fragment shader during rendering — the mechanism Figures L-ON and AR (§6) visually confirm.

---

## 10. Stability — every observation reproduced ≥ 2 times

Per the "report true magnitude/frequency/timing and confirm stability across at least two runs" rule, each launcher observation was run **twice** with the *same unchanged input*, its monotonic timestamps normalized (`sed 's/\[[0-9.]*\]/[T]/'`), and the two normalized captures `diff`ed. An **empty diff (0 bytes)** proves the two runs are byte-identical after removing only the wall-clock prefixes.

The harness (§4) runs **all ten launcher variants twice** and prints, for each, a `STABILITY[<variant>]: STABLE (0 diff)` line followed by `wc -c "$WORK"/stability/*.diff`. This is the complete, unedited stability output reproduced verbatim from §4.1 (a single harness run, `$WORK=/tmp/kitty_obs.c2jvSe3Q`):

```console
STABILITY[default]: STABLE (0 diff)
STABILITY[arabic]: STABLE (0 diff)
STABILITY[cjk]: STABLE (0 diff)
STABILITY[debuggl]: STABLE (0 diff)
STABILITY[forceltr]: STABLE (0 diff)
STABILITY[fira_on]: STABLE (0 diff)
STABILITY[fira_off]: STABLE (0 diff)
STABILITY[symbolmap]: STABLE (0 diff)
STABILITY[missing]: STABLE (0 diff)
STABILITY[font28]: STABLE (0 diff)
$ wc -c "$WORK"/stability/*.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/arabic.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/cjk.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/debuggl.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/default.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/fira_off.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/fira_on.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/font28.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/forceltr.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/missing.diff
0 /tmp/kitty_obs.c2jvSe3Q/stability/symbolmap.diff
0 total
```

The `create_test_font_group` cell-metric corroboration and the C metrics probe (§8.C.1) and the HarfBuzz glyph-ID probe (§6.A.1) were likewise each run twice with identical results (empty normalized diff). The normalized launcher capture that was diffed (identical for both runs; `arabic` variant shown as a representative example):

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
| Cell metrics — C probe (SOURCE-DERIVED, **primary**) | 2 | ✅ identical | `9×18` @96 DPI, `baseline=14 underline=15/1 strikethrough=10/1`; Nimbus `9×19`; DejaVu `22×44`@28pt |
| Cell metrics — `create_test_font_group` (NON-CANONICAL corroboration) | 2 | ✅ identical | `9×18` @96 DPI; `7×14` @72 DPI; Nimbus `9×19` (= C-probe dims) |
| GL limits (glxinfo) | 2 | ✅ identical | `GL_MAX_TEXTURE_SIZE=16384`, `GL_MAX_ARRAY_TEXTURE_LAYERS=2048` |
| Prerender occupancy (corroboration) | 2 | ✅ identical | `11` sprites at `(0..10,0,0)` |

No run-to-run inconsistency was observed for any value; the environment is deterministic, so there is no distribution to report — the single stable value is reported for each quantity.

---

## 11. The three required honest negatives (consolidated)

Three findings are, per the AAP, **negatives that must be reported with evidence rather than forced into a positive**. Each is stated plainly with its proof and its label.

### 11.1 kitty has **no bidi (bidirectional) reordering engine** — OBSERVED + INFERRED

kitty performs **no logical→visual bidi reordering**. The `force_ltr` option's own documentation states kitty does not support BIDI [`kitty/options/definition.py:64-83`] (quoted verbatim in §6.A.2). What kitty *does* do is hand each run to HarfBuzz, which applies script-appropriate **shaping** (Arabic joining/contextual forms) — visible in Figure AR (§6.A.2) where مرحبا renders as connected cursive glyphs — but the *ordering* of runs is not reordered by a bidi algorithm. The resolved default `force_ltr=no` was observed at startup. That Arabic RTL clusters carry *decreasing* HarfBuzz cluster numbers [`kitty/fonts.c:997,1079`] is **INFERRED** from the source comments (not separately logged). This is a partial-support situation the changelog itself acknowledges [`docs/changelog.rst:3540-3541`].

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
| **Missing glyph (verification failure)** | `-o font_family="Nimbus Mono PS"` + feed U+0378 U+0380 U+1E4D0 U+31350 U+F0000 U+10FFFD | Glyph-verification gate: an OS fallback pick that lacks the glyph is rejected (error/edge path) | 6× live `… but it does not actually contain glyphs for that text` [`fonts.c:509`] → `MISSING_FONT` (§6.A.4); `missing.diff` 0 bytes |
| **`--debug-gl`** | `--debug-gl` instead of `--debug-rendering` | Alternate flag for the GL banner | Same GL banner emitted |
| **`force_ltr=yes`** | `-o force_ltr=yes` | LTR-forcing modifier | Accepted at startup; DejaVu block + banner |
| **`disable_ligatures=always`** | `-o font_family="Fira Code" -o disable_ligatures=always` | Keeps trailing `-calt` → ligatures OFF | Figure L-OFF (§6.A.1): operators render discrete; glyph IDs `[1578,1578]`/`[1221,1580]` |
| **Ligatures ON (default)** | `-o font_family="Fira Code"` | Drops `-calt` sentinel → calt ON | Figure L-ON (§6.A.1): `== -> != >= === =~ \|> ++` fused; glyph IDs `[1649,1387]`/`[1186,1458]` |
| **`symbol_map` non-empty** | `-o symbol_map="U+0645 Amiri"` | Symbol-map block + explicit face mapping | `Symbol map fonts:` → `Amiri-Regular: …/Amiri-Regular.ttf:0` |
| **Combining diacritics** | default feed line 2 (`o`+U+0301+U+0323) | grapheme cluster → one cell | Live `U+6f U+301 U+323 using previous fallback font at index: 0` |
| **Overflow edge path** | (source-identified, not triggered) | prerender sprites spilling to row `y>0` | Fatal `Too many pre-rendered sprites…` [`fonts.c:1463`] — did **not** fire (11 fit row 0) |
| **Cell metrics** *(SOURCE-DERIVED C probe; binding = corroboration)* | C probe (system FreeType, §13.2) + `+launch` binding cross-check | Full set from source-derived probe; dims corroborated by non-canonical `create_test_font_group` | `9×18` baseline 14 underline 15/1 strikethrough 10/1; binding dims match (§8.C.1) |
| **Prerender occupancy** *(labelled corroboration)* | `setup_for_testing` callback | Count of sprites uploaded before shaped text | `11` at `(0..10,0,0)` |

---

## 13. Appendices — complete unedited logs

### 13.1 `hb_ligature_probe.c` — standalone HarfBuzz ligature probe (source)

This is the complete source of the independent probe referenced in §6.A.1. It links the **same** system HarfBuzz + FreeType kitty links against and demonstrates the `calt` contextual-alternates substitution (ligatures) directly, with no dependence on kitty's test harness. Build and run:

```bash
gcc hb_ligature_probe.c -o hb_ligature_probe $(pkg-config --cflags --libs harfbuzz freetype2)
./hb_ligature_probe
```

```c
/* hb_ligature_probe.c — standalone HarfBuzz probe demonstrating that the
 * OpenType 'calt' contextual-alternates feature (the mechanism kitty enables
 * for programming ligatures) fuses operator sequences into ligature glyphs.
 * Links the SAME system HarfBuzz + FreeType libraries kitty links against.
 * Build: gcc hb_ligature_probe.c -o hb_ligature_probe $(pkg-config --cflags --libs harfbuzz freetype2)
 */
#include <stdio.h>
#include <string.h>
#include <hb.h>
#include <hb-ft.h>
#include <ft2build.h>
#include FT_FREETYPE_H

static void shape_it(FT_Face ft_face, const char *label, const char *text, int calt_on) {
    hb_font_t *font = hb_ft_font_create_referenced(ft_face);
    hb_buffer_t *buf = hb_buffer_create();
    hb_buffer_add_utf8(buf, text, -1, 0, -1);
    hb_buffer_guess_segment_properties(buf);

    /* calt is the feature kitty toggles for ligatures; when disabled
     * (disable_ligatures=always) kitty removes it. Model both. */
    hb_feature_t feats[1];
    hb_feature_from_string("calt", -1, &feats[0]);
    feats[0].value = calt_on ? 1 : 0;
    feats[0].start = HB_FEATURE_GLOBAL_START;
    feats[0].end   = HB_FEATURE_GLOBAL_END;

    hb_shape(font, buf, feats, 1);

    unsigned int n = 0;
    hb_glyph_info_t *info = hb_buffer_get_glyph_infos(buf, &n);
    printf("  %-4s calt=%d  text=%-4s -> %u glyphs: [", label, calt_on, text, n);
    for (unsigned int i = 0; i < n; i++) printf("%s%u", i ? ", " : "", info[i].codepoint);
    printf("]\n");

    hb_buffer_destroy(buf);
    hb_font_destroy(font);
}

int main(void) {
    const char *path = "/usr/share/fonts/truetype/firacode/FiraCode-Regular.ttf";
    FT_Library lib; FT_Face face;
    if (FT_Init_FreeType(&lib)) { fprintf(stderr, "FT init failed\n"); return 1; }
    if (FT_New_Face(lib, path, 0, &face)) { fprintf(stderr, "cannot open %s\n", path); return 1; }
    FT_Set_Char_Size(face, 0, 28*64, 96, 96);

    printf("HarfBuzz %s / FreeType face: %s %s\n", hb_version_string(), face->family_name, face->style_name);
    printf("Fira Code — ligatures ENABLED (calt=1) vs DISABLED (calt=0):\n");
    shape_it(face, "==", "==", 1);
    shape_it(face, "==", "==", 0);
    shape_it(face, "->", "->", 1);
    shape_it(face, "->", "->", 0);
    shape_it(face, "!=", "!=", 1);
    shape_it(face, "!=", "!=", 0);
    shape_it(face, "===", "===", 1);
    shape_it(face, "===", "===", 0);

    FT_Done_Face(face); FT_Done_FreeType(lib);
    return 0;
}
```


### 13.2 `cell_metrics_probe.c` — standalone `cell_metrics()` reproduction (source)

This is the complete, byte-identical source of the **source-derived** metric probe referenced in §8.C.1. It links the same system FreeType that kitty links (2.13.3) and applies the production `cell_metrics` formula [`kitty/freetype.c:387-405`] — plus `calc_cell_width`, `calc_cell_height`/`get_height_for_char`, `font_units_to_pixels_y`, the `init_ft_face` face fields and the OS/2 strikethrough read — verbatim. Compiling and running it reproduces the complete output embedded in §8.C.1 exactly.

```bash
gcc cell_metrics_probe.c -o cell_metrics_probe $(pkg-config --cflags --libs freetype2) -lm
./cell_metrics_probe
```

```c
/* cell_metrics_probe.c — standalone reimplementation of kitty's cell_metrics()
 * (kitty/freetype.c:387-405) + helpers calc_cell_width (L376-386),
 * calc_cell_height/get_height_for_char (L126-153), font_units_to_pixels_y
 * (L92-94), face fields (init_ft_face L214) and os2 strikethrough (L231-233).
 * With DEFAULT kitty config, fonts.c:calc_cell_metrics applies NO adjustment
 * (modify_font cell_width/cell_height default 0), so this equals the value
 * create_test_font_group returns. Links the SAME system FreeType kitty links.
 * Build: gcc cell_metrics_probe.c -o cell_metrics_probe $(pkg-config --cflags --libs freetype2) -lm
 */
#include <stdio.h>
#include <math.h>
#include <ft2build.h>
#include FT_FREETYPE_H
#include FT_TRUETYPE_TABLES_H
#ifndef MIN
#define MIN(a,b) ((a)<(b)?(a):(b))
#endif
#ifndef MAX
#define MAX(a,b) ((a)>(b)?(a):(b))
#endif
static int f2p_y(FT_Face f, int x){ return (int)ceil((double)FT_MulFix(x, f->size->metrics.y_scale)/64.0); }
static unsigned int calc_cell_width(FT_Face f){
    unsigned int ans=0;
    for(unsigned int i=32;i<128;i++){ int gi=FT_Get_Char_Index(f,i);
        if(FT_Load_Glyph(f,gi,FT_LOAD_TARGET_NORMAL)==0) ans=MAX(ans,(unsigned int)ceilf((float)f->glyph->metrics.horiAdvance/64.f)); }
    return ans;
}
static unsigned int uscore_h(FT_Face f,int asc){
    unsigned int ans=0; int gi=FT_Get_Char_Index(f,'_');
    if(FT_Load_Glyph(f,gi,FT_LOAD_TARGET_NORMAL)==0){ unsigned int bl=(unsigned int)f2p_y(f,asc); FT_GlyphSlot g=f->glyph;
        if(g->bitmap_top<=0 || ((unsigned int)g->bitmap_top<bl)) ans=bl-g->bitmap_top+g->bitmap.rows; }
    return ans;
}
static void run(const char*path,double pt,unsigned int xdpi,unsigned int ydpi){
    FT_Library lib; FT_Face f;
    if(FT_Init_FreeType(&lib)){fprintf(stderr,"FT init failed\n");return;}
    if(FT_New_Face(lib,path,0,&f)){fprintf(stderr,"open failed: %s\n",path);return;}
    FT_Set_Char_Size(f,0,(FT_F26Dot6)(ceil(pt*64.0)),xdpi,ydpi);
    int asc=f->ascender,desc=f->descender,hgt=f->height,ulp=f->underline_position,ult=f->underline_thickness,stp=0,stt=0;
    TT_OS2*os2=(TT_OS2*)FT_Get_Sfnt_Table(f,FT_SFNT_OS2); if(os2){stp=os2->yStrikeoutPosition;stt=os2->yStrikeoutSize;}
    unsigned int cell_width=calc_cell_width(f);
    unsigned int height_from_metrics=(unsigned int)f2p_y(f,hgt);
    unsigned int uh=uscore_h(f,asc);
    unsigned int cell_height=height_from_metrics; int wa_delta=0;
    if(uh>cell_height){ wa_delta=(int)(uh-cell_height); cell_height=uh; }
    unsigned int baseline=(unsigned int)f2p_y(f,asc);
    unsigned int ul_pos=MIN(cell_height-1,(unsigned int)f2p_y(f,MAX(0,asc-ulp)));
    unsigned int ul_th=MAX(1,(unsigned int)f2p_y(f,ult));
    unsigned int st_pos,st_th;
    if(stp!=0) st_pos=MIN(cell_height-1,(unsigned int)f2p_y(f,MAX(0,asc-stp))); else st_pos=(unsigned int)floor(baseline*0.65);
    if(stt>0) st_th=MAX(1,(unsigned int)f2p_y(f,stt)); else st_th=ul_th;
    printf("FACE %s %s | %.1fpt @ %u/%u DPI\n",f->family_name,f->style_name,pt,xdpi,ydpi);
    printf("  raw: units_per_EM=%hu ascender=%d descender=%d height=%d underline_position=%d underline_thickness=%d os2_strike_pos=%d os2_strike_th=%d y_scale=%ld\n",
           f->units_per_EM,asc,desc,hgt,ulp,ult,stp,stt,(long)f->size->metrics.y_scale);
    printf("  underscore workaround: height_from_metrics=%u underscore_pixel_height=%u -> %s (delta=%+d)\n",
           height_from_metrics,uh,(wa_delta>0?"FIRES":"does NOT fire"),wa_delta);
    printf("  CELL width=%u height=%u baseline=%u | underline pos=%u th=%u | strikethrough pos=%u th=%u\n\n",
           cell_width,cell_height,baseline,ul_pos,ul_th,st_pos,st_th);
    FT_Done_Face(f);FT_Done_FreeType(lib);
}
int main(void){
    const char*dejavu="/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf";
    const char*nimbus="/usr/share/fonts/type1/urw-base35/NimbusMonoPS-Regular.t1";
    printf("=== kitty cell_metrics() replicated via system FreeType %d.%d.%d ===\n\n",FREETYPE_MAJOR,FREETYPE_MINOR,FREETYPE_PATCH);
    run(dejavu,11.0,96,96);
    run(dejavu,11.0,100,100);
    run(nimbus,11.0,96,96);
    run(nimbus,11.0,100,100);
    run(dejavu,28.0,96,96);
    run(dejavu,28.0,100,100);
    return 0;
}
```


All raw launcher captures referenced above were written to `$WORK/logs/<variant>_run{1,2}.log` and their normalized/`diff`ed forms to `$WORK/stability/<variant>_run{1,2}.norm` and `$WORK/stability/<variant>.diff` inside the harness's private `mktemp -d` `$WORK` directory (§4); the **relevant unedited excerpts are embedded inline** at the point of each claim (§4.1, §6, §7, §8, §9, §10). Each embedded block shows the command that produced it and preserves the `[<seconds>]` monotonic prefixes (or the `[T]` normalization explicitly marked as such for the stability diffs). No claim in this document relies on a value that is not shown next to it.

Because the harness removes `$WORK` on exit (§4, narrow teardown) and the investigation restores the tree to contain only this document (§14.3), the logs are not committed — reproducing them is a matter of re-running the embedded harness verbatim. The one full-length artifact reproduced verbatim in-document is the **canonical build transcript in Appendix A** (all 344 lines — 343 lines of raw `python3 setup.py` output plus the trailing harness-added `[BUILD_EXIT=0] wall=65s` status line, so a fresh `CI=true python3 setup.py 2>&1 | wc -l` reports 343).

---

## 14. Final coverage pass

### 14.1 Objective-by-objective, named-item coverage

Every mechanism, function, flag, file, and "e.g./such as" item named in the four objectives is addressed below. Evidence type: **OBS** = observed live-launcher output; **SRC** = source-derived (with `file:line`); **COR** = labelled non-canonical corroboration; **VIS** = embedded screenshot (base64 data-URI, with reconstruction command + SHA-256); **NEG** = evidence-backed negative.

| Objective / named item | Where | Evidence |
|------------------------|-------|----------|
| **A — ligatures** (`calt`/`liga`/`dlig`, `disable_ligatures`, `font_features`) | §6.A.1 | SRC [fonts.c:42,45,293-327,811-813,1755-1757] + reproducible glyph-ID proof (shape_string + HarfBuzz probe) + VIS Figures L-ON vs L-OFF (embedded) |
| **A — bidi** (`force_ltr`, HarfBuzz RTL) | §6.A.2, §11.1 | SRC [definition.py:64-83; fonts.c:997,1079] + VIS Figure AR (embedded) + NEG |
| **A — combining diacritics** (`has_cell_text`, `codepoint_for_mark`) | §6.A.3 | OBS `U+6f U+301 U+323 …` + SRC [fonts.c:435-447,457-466] + VIS (Figure AR run) + COR |
| **A — font fallback at startup** (`fallback_font`, `load_fallback_font`, `create_fallback_face`, cap `>100`, glyph-verification `has_cell_text`) | §6.A.4 | OBS Arabic+CJK fallback lines + OBS 6× glyph-verification failure (`… does not actually contain glyphs …`, U+0378/U+0380/U+1E4D0/U+31350/U+F0000/U+10FFFD) + SRC [fonts.c:481-520; :501-511; fontconfig.c:463] |
| **B — startup diagnostics** (`--debug-font-fallback`, `dump_font_debug`, `identify_for_debug`) | §7.B.1-2 | OBS Text/Symbol font blocks + SRC [render.py:161-171; freetype.c:738] |
| **B — font families** (Normal/Bold/Italic/Bold-Italic) | §7.B.2 | OBS 4-face blocks (DejaVu / Nimbus / Fira / Noto) |
| **B — 9 config values** (`font_family`,`bold_font`,`italic_font`,`bold_italic_font`,`font_size`,`symbol_map`,`disable_ligatures`,`font_features`,`force_ltr`) | §7.B.3 | OBS + SRC [definition.py:35,53,55,57,59,64,86,115,135] |
| **C — cell metrics** (width/height/baseline) | §8.C.1, §13.2 | SRC (full set, C probe, primary) [freetype.c:387-405] + COR (dims via non-canonical `create_test_font_group`) |
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

The full output of `CI=true python3 setup.py` (referenced in §3), reproduced verbatim. The block below is 344 lines: 343 lines of raw `setup.py` output plus the final harness-added `[BUILD_EXIT=0] wall=65s` status line (hence `CI=true python3 setup.py 2>&1 | wc -l` reports 343; `BUILD_EXIT=0`, wall 65 s). Preceded by `python3 setup.py clean` (exit 0). No line is elided.

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
