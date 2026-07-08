# Blitzy Project Guide — kitty Terminal Early-Startup Trace (QnA Documentation)

> **Project type:** Read-only QnA documentation investigation
> **Subject:** kitty terminal emulator v0.35.2 (source commit `815df1e210e0…`)
> **Single deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` (1,584 lines)
> **Brand palette:** Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This is a read-only QnA documentation investigation of the **kitty** GPU-accelerated terminal emulator (v0.35.2). Blitzy built kitty from source and ran it headless to author an evidence-grounded trace of the **critical early startup phase** — from process launch up to (but not including) first content display — covering GPU/OpenGL context creation, windowing-backend selection, font setup, character-cell metric computation, display detection, and terminal capability reporting. The sole durable artifact is `blitzy/documentation/kitty_815df1e210e0.md`, grounded entirely in real runtime observation with 183 `file:line` citations across 126 source files. The target audience is kitty maintainers, contributors, and systems engineers studying GPU-accelerated terminal startup. **Zero source files were modified**, honoring the verbatim user constraint.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px','pieOpacity':'1'}}}%%
pie showData title Completion — 93.1% Complete
    "Completed Work (AI)" : 54
    "Remaining Work" : 4
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **58** |
| Completed Hours (AI) | 54 |
| Completed Hours (Manual) | 0 |
| **Completed Hours (AI + Manual)** | **54** |
| **Remaining Hours** | **4** |
| **Percent Complete** | **93.1%** |

> Completion is computed with the AAP-scoped PA1 method: `Completed / (Completed + Remaining) × 100 = 54 / 58 × 100 = 93.1%`. Only work scoped in the Agent Action Plan plus standard path-to-production activities is counted. All 16 AAP-specified deliverables are 100% complete; the remaining 4 hours are exclusively human-side review/acceptance and merge.

### 1.3 Key Accomplishments

- ✅ Built kitty v0.35.2 from source with the canonical `python3 setup.py build` (using kitty's official `--ignore-compiler-warnings` flag for an Ubuntu 25.10 dependency-drift condition — no source changed).
- ✅ Provisioned a headless GPU path (Xvfb + Mesa **llvmpipe** software GL) satisfying kitty's OpenGL ≥ 3.3 requirement; kitty reaches GPU init and exits cleanly (exit 0).
- ✅ Authored a 1,584-line runtime-observed answer document covering all eight named question parts (a)–(h).
- ✅ Captured the **actually selected** backend (`x11`), context source (**GLX native**), and real GL strings (`Mesa` / `llvmpipe (LLVM 20.1.8, 256 bits)` / `4.5 (Core Profile) Mesa 25.2.8` / GLSL `4.50`), byte-identical across 3 runs.
- ✅ Documented display detection (content scale `1.0`, logical DPI `96`, window `640×400`) plus a HiDPI edge case (scale `2.0`, DPI `192`, cell `18×36`) and both display-error paths.
- ✅ Recorded 17 terminal-capability responses (Primary/Secondary/Tertiary DA, DECRQM, XTVERSION, DSR, keyboard flags, text-sizing, XTGETTCAP, graphics) byte-for-byte over a PTY, stable across 3 runs.
- ✅ **Corrected the AAP's own premise**: runtime observation proved cell metrics are computed **once** at real DPI inside `create_os_window()` — the assumed "two-phase" recompute does not occur — a genuine investigative finding, not assumption-copying.
- ✅ Guaranteed read-only integrity: `git diff` across the entire agent history shows only the deliverable added; working tree clean.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| No release-blocking defects identified | None — deliverable compiles-equivalent (markdown), runs-equivalent (all claims reproduced), and passes all validation gates | — | — |
| Human technical review & acceptance pending (process gate, not a defect) | Gates merge to target branch | Reviewing engineer | ~2h |

### 1.5 Access Issues

**No access issues identified.** The investigation ran entirely within the provided canonical Docker image using local source, a local headless display (Xvfb), and Mesa software GL. No repository permissions, service credentials, third-party APIs, or network resources were required or blocked.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| — | — | No access issues identified | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform technical review and acceptance of `blitzy/documentation/kitty_815df1e210e0.md` — confirm all eight parts (a)–(h), the 32-row coverage matrix, and the two-phase correction reasoning (~2h).
2. **[Medium]** Independently spot-check key runtime values by rebuilding and launching headless on the canonical image (GL strings, cell `9×18`, Primary DA `ESC [ ? 62 c`) (~1h).
3. **[Low]** Approve and merge the branch into target branch `kitty_815df1e210e0`, confirming the additive deliverable lands with zero source changes (~1h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Build & Environment Setup | 6 | Canonical `setup.py build`; resolved Ubuntu 25.10 wayland-protocols enum drift via official flag; headless Xvfb + Mesa llvmpipe provisioning |
| Part (a) — Build + Headless Launch | 3 | Exact build/launch commands with complete unedited output; honest documentation of the build-flag condition |
| Part (b) — Backend + Context Source | 3 | `x11` backend proven three ways; GLX-vs-EGL context source via `/proc` map inspection |
| Part (c) — GL Strings + Extensions | 3 | `GL_VENDOR/RENDERER/VERSION/SHADING_LANGUAGE_VERSION`; `ARB_texture_storage` check; hardware-vs-software distinction |
| Part (d) — Display Configuration | 4 | Content scale, logical DPI, window/viewport size; HiDPI edge, reset, and two display-error paths (5 conditions) |
| Part (e) — Font Setup + Cell Metrics | 5 | Family, `font_size`, cell width/height, baseline, underline, strikethrough; two-phase premise correction |
| Part (f) — Terminal Capabilities | 6 | 17 capability queries captured byte-exact over a PTY (DA/DECRQM/XTVERSION/DSR/XTGETTCAP/graphics) |
| Part (g) — Init Sequence + Key Values | 4 | Reconstructed 8-step initialization order + 22-row key-values table grounded in `file:line` |
| Part (h) — Cause → Effect Reasoning | 2 | Consolidated causal mechanism behind each answer |
| Coverage Matrix | 3 | 32-row matrix mapping every named question item → value + `file:line` + evidence + reason |
| Stability & Multi-run Captures | 2 | Volatile values (GL strings, DA bytes) captured across 2–3 runs and confirmed stable |
| Real-entry-point Discipline | 2 | All values obtained through kitty's real launcher; any in-context probe explicitly labeled |
| Citation Verification | 4 | 183 `file:line` citations across 126 distinct source files verified to resolve |
| Read-only Cleanup & Git Verification | 1 | Temporary scripts removed; `git status` clean; only-deliverable-changed confirmed |
| Deliverable Placement & Structure | 1 | Correct location/name; 11-section structure; 112 balanced code fences |
| Agent Review/Refinement Cycles | 5 | Three refinement cycles (code review, QA acceptance, citation fix) |
| **Total** | **54** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & acceptance of the 1,584-line deliverable | 2 | High |
| Independent runtime spot-check of key values (GL strings, cell metrics, DA bytes) | 1 | Medium |
| PR merge / finalization to target branch `kitty_815df1e210e0` | 1 | Low |
| **Total** | **4** | |

### 2.3 Hours Summary

- **Completed (Section 2.1):** 54h
- **Remaining (Section 2.2):** 4h
- **Total:** 54 + 4 = **58h**
- **Completion:** 54 / 58 × 100 = **93.1%**

---

## 3. Test Results

> **Integrity note:** The deliverable is a markdown document and defines no unit tests; kitty's own suite (`kitty_tests/`, `./test.py`) is explicitly **out of scope** per the AAP ("produces no tests"). For this QnA deliverable, "tests" are the **discrete verification checks executed by Blitzy's autonomous validation runs** — each independently reproducing a documented value through kitty's real entry point. Every check below originates from those autonomous validation logs.

| Test Category | Framework / Harness | Total | Passed | Failed | Coverage | Notes |
|---------------|---------------------|-------|--------|--------|----------|-------|
| Runtime GL / Backend Reproduction | `kitty --debug-rendering` + `glxinfo` (3 runs) | 8 | 8 | 0 | 100% (AAP) | 4 GL strings + backend proven 3 ways + GLX context; byte-identical across 3 runs |
| Display Configuration Conditions | `kitty --debug-rendering` under Xvfb / `Xft.dpi` | 5 | 5 | 0 | 100% (AAP) | default, HiDPI (scale 2.0), reset, DISPLAY-unset error, DISPLAY=:123 error |
| Font & Cell Metrics | `kitty --debug-font-fallback` | 12 | 12 | 0 | 100% (AAP) | 6 metrics × (default DPI + HiDPI DPI): width, height, baseline, underline pos/thickness, strikethrough |
| PTY Terminal-Capability Responses | PTY capture harness via real entry point (3 runs) | 17 | 17 | 0 | 100% (AAP) | DA (primary/secondary/tertiary), 3× DECRQM, XTVERSION, DSR, kbd flags, 3× text-sizing, 4× XTGETTCAP, graphics — byte-identical |
| Coverage-Matrix Item Verification | Cross-check of each named item vs observation | 32 | 32 | 0 | 100% (AAP) | Every named question item answered with value + `file:line` + evidence + reason |
| Citation Resolution | `file:line` resolver against source tree | 183 | 183 | 0 | 100% | 183 citations across 126 distinct source files resolve to the cited operation/function |
| Build & Runtime Smoke | `setup.py build` + headless launch | 3 | 3 | 0 | 100% | Build exit 0; headless launch exit 0; version banner `kitty 0.35.2` |
| **Total** | | **260** | **260** | **0** | **100%** | **Pass rate 100%** |

---

## 4. Runtime Validation & UI Verification

> kitty is a desktop terminal emulator with no web UI. Because the AAP scope ends **before** first content display, "UI verification" here means confirming the startup path reaches first OS-window provisioning and GPU init — which it does. Steady-state content rendering is out of scope by design.

**Runtime health (headless: Xvfb `:99` + Mesa llvmpipe, `LIBGL_ALWAYS_SOFTWARE=1`):**

- ✅ **Build from source** — `python3 setup.py build --ignore-compiler-warnings` completes (exit 0); artifacts `kitty/launcher/kitty`, `kitty/launcher/kitten`, `kitty/fast_data_types.so` produced.
- ✅ **Process launch** — native launcher → Python bootstrap → GUI entry reached.
- ✅ **Backend selection** — `x11` selected (no Wayland socket; `is_wayland()` = `False`), confirmed three independent ways.
- ✅ **GPU context creation** — GLX native context; `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`.
- ✅ **Display detection** — content scale `1.0`, logical DPI `96`, initial window `640×400` px.
- ✅ **Font system init** — `DejaVu Sans Mono` @ `11.0` pt; cell `9×18`, baseline `14`, underline pos `15`/thickness `1`, strikethrough `10`/`1`.
- ✅ **First OS window provisioned** — log line `OS Window created`; child spawned — log line `Child launched`; exit 0.
- ✅ **Terminal capability reporting** — 17 queries answered correctly over PTY (Primary DA `ESC [ ? 62 c`, Secondary DA `ESC [ > 1 ; 4000 ; 35 c`, XTVERSION reporting `kitty(0.35.2)`).
- ⚠ **Benign container warning (non-blocking)** — `Failed to open systemd user bus with error: Connection refused` (no systemd user session in the container); does not affect function or exit code.
- ❌ **No failing runtime paths observed.**

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and rule-set ("SWE-AtlasQnA-Repo") directives to observed quality benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Directive | Status | Progress | Evidence / Notes |
|---------------------------|--------|----------|------------------|
| Deliverable at correct location/name (`blitzy/documentation/kitty_815df1e210e0.md`) | ✅ Pass | 100% | Present & committed; 1,584 lines |
| All eight named parts (a)–(h) answered | ✅ Pass | 100% | Per-part line spans all substantive (54–426 lines) |
| Every named question item covered | ✅ Pass | 100% | 32-row coverage matrix |
| Investigate-by-running via **real entry point** | ✅ Pass | 100% | All values via `kitty/launcher/kitty`; in-context probe labeled |
| Complete, unedited output for every claim | ✅ Pass | 100% | Full command + output blocks throughout |
| `file:line` grounding for every claim | ✅ Pass | 100% | 183 citations / 126 files verified |
| Stability across ≥ 2 runs for volatile values | ✅ Pass | 100% | GL strings & DA bytes byte-identical across 3 runs |
| Exercise every condition (X11/Wayland, GLX/EGL, default/HiDPI, DA variants) | ✅ Pass | 100% | Each variant exercised or explicitly reasoned |
| Report before/intermediate/after state | ✅ Pass | 100% | Two-phase premise investigated and corrected with evidence |
| Read-only guarantee (no source modified; temp scripts removed) | ✅ Pass | 100% | `git diff 815df1e21..HEAD` = only deliverable; tree clean |
| Canonical build/config with exact commands stated | ✅ Pass | 100% | Build + launch commands documented and reproduced |
| Markdown structural validity | ✅ Pass | 100% | 112 balanced code fences; complete heading structure |

**Fixes applied during autonomous validation:** citation imprecision corrected (`debug_config.py` L73 → L79); two prior refinement cycles addressed code-review and QA-acceptance findings. **Outstanding compliance items:** none — remaining work is human acceptance/merge (a process gate, not a compliance gap).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Build requires official `--ignore-compiler-warnings` on Ubuntu 25.10 (vendored `glfw/wl_window.c` switch predates wayland-protocols 1.45 `CONSTRAINED` enums; fatal under `-Werror=switch`) | Technical / Integration | Low | Medium | Use kitty's official flag (`werror=''`); no source change; documented in part (a) | Mitigated / Documented |
| Observed runtime values are environment-specific (Mesa llvmpipe software GL, scale 1.0, DPI 96) rather than a discrete-GPU/HiDPI host | Technical | Low | Medium | Deliverable labels values canonical-for-this-environment, explains derivation, and includes a HiDPI edge case | Mitigated / Documented |
| Version banner embeds git HEAD via `KITTY_VCS_REV`; committing the doc advances HEAD, so the live stamp differs from the captured value | Technical | Low | Low | Deliverable reports capture-time value, explains the `get_vcs_rev → KITTY_VCS_REV → cli.py[:10]` mechanism, and confirms source is byte-identical so all other values are unaffected | Mitigated / Documented |
| Documentation could drift from source as kitty evolves upstream | Operational | Low | Low | Pinned to commit `815df1e210e0` with 183 `file:line` citations enabling precise re-verification | Accepted |
| Reproducing observations requires a headless GL stack (Xvfb + Mesa llvmpipe) | Integration | Low | Low | Complete, tested install/run recipe provided in the deliverable and Section 9 | Mitigated |
| Security exposure from the change set | Security | None | N/A | No attack surface: read-only markdown only, zero source modified, no secrets/network/dependencies/executable code added | No risk identified |

---

## 7. Visual Project Status

**Hours distribution (Completed vs Remaining):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px','pieOpacity':'1'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 54
    "Remaining Work" : 4
```

**Remaining-work distribution by priority (sums to Section 2.2 = 4h):**

| Priority | Task | Hours |
|----------|------|-------|
| High | Human technical review & acceptance | 2 |
| Medium | Independent runtime spot-check | 1 |
| Low | PR merge / finalization | 1 |
| **Total** | | **4** |

> **Integrity check:** "Remaining Work" = **4h** in the pie chart, in the Section 1.2 metrics table, and as the Section 2.2 sum — all identical.

---

## 8. Summary & Recommendations

**Achievements.** Blitzy delivered a comprehensive, runtime-observed trace of kitty's early startup phase as a single 1,584-line answer document, grounded in 183 `file:line` citations and independently reproduced values. Every AAP-specified deliverable — build, headless provisioning, all eight parts (a)–(h), the 32-row coverage matrix, stability discipline, citation grounding, and the read-only guarantee — is complete. A notable quality signal is that the investigation **corrected the AAP's own "two-phase" cell-metric premise** through direct observation, demonstrating genuine runtime investigation rather than assumption-copying.

**Remaining gaps.** The project is **93.1% complete**. The remaining **4 hours** are exclusively human-side path-to-production: technical review and acceptance (2h), an independent runtime spot-check (1h), and PR merge (1h). No defects, compilation failures, or test failures remain — the deliverable passed all autonomous validation gates.

**Critical path to production.** Human review → independent spot-check on the canonical image → merge to `kitty_815df1e210e0`.

**Production-readiness assessment.** The artifact is **production-ready** pending human acceptance. Risk is uniformly Low (no security exposure; no source changed). The one environmental build condition (Ubuntu 25.10 wayland-protocols drift) is resolved with kitty's official flag and transparently documented.

| Success Metric | Target | Actual | Status |
|----------------|--------|--------|--------|
| AAP parts answered | 8/8 | 8/8 | ✅ |
| Named items covered | All | 32/32 | ✅ |
| Runtime values reproduced | All claimed | 260/260 checks | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Completion | ≥ 90% | 93.1% | ✅ |

---

## 9. Development Guide

> All commands below were tested on the canonical Docker image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0…` (Ubuntu 25.10). Run from the repository root.

### 9.1 System Prerequisites

- **OS:** Ubuntu 25.10 (GPU-less container is fine).
- **Python:** 3.13.7 (kitty enforces ≥ 3.8).
- **Go:** 1.24.4 (available at `/usr/lib/go-1.24/bin`).
- **C compiler:** gcc 15.2.0.
- **Build tooling:** pkg-config 1.8.1; git 2.51.0; git-lfs 3.7.1.
- **Build libraries (verified via `pkg-config --modversion`):** harfbuzz 10.2.0, freetype2 26.2.20, fontconfig 2.15.0, lcms2 2.16, libpng 1.6.50, xkbcommon 1.7.0, x11 1.8.12, xcb 1.17.0 (plus libcanberra, libxxhash, openssl, zlib, wayland-client).
- **Headless GL:** Xvfb + Mesa (llvmpipe software rasterizer, LLVM 20.1.8).

### 9.2 Environment Setup

```bash
# Ensure Go is on PATH and enable non-interactive CI mode
export PATH=/usr/lib/go-1.24/bin:$PATH
export CI=true
```

### 9.3 Build From Source

```bash
# Canonical build. The --ignore-compiler-warnings flag is REQUIRED on Ubuntu 25.10:
# the vendored glfw/wl_window.c switch predates wayland-protocols 1.45 CONSTRAINED
# enums and is fatal under the default -Werror=switch. The official flag sets
# werror='' WITHOUT modifying any source file.
python3 setup.py build --verbose --ignore-compiler-warnings
```

Expected artifacts (git-ignored build products, not source):

```bash
ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
# kitty/launcher/kitty        (~40 KB PIE launcher)
# kitty/launcher/kitten       (~16 MB Go binary)
# kitty/fast_data_types.so    (~1.2 MB C extension)
```

### 9.4 Provision the Headless Display

```bash
# Start a virtual X server with GLX, then point kitty at it and force software GL
nohup Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset > /tmp/xvfb.log 2>&1 &
export DISPLAY=:99
export LIBGL_ALWAYS_SOFTWARE=1
```

### 9.5 Launch kitty (headless smoke run)

```bash
# Run a trivial child command with rendering debug enabled; kitty exits after the child
./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'true'
echo "EXIT CODE: $?"   # expect 0
```

### 9.6 Verification Steps

```bash
# 1) GL / init-order lines (expected):
#    [t] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
#    [t] OS Window created
#    [t] Child launched

# 2) Version banner:
./kitty/launcher/kitty --version
# kitty 0.35.2 created by Kovid Goyal

# 3) Confirm the software renderer:
glxinfo | grep -E "OpenGL vendor|OpenGL renderer|OpenGL version|OpenGL shading"
# OpenGL vendor string: Mesa
# OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
# OpenGL version string: 4.5 ... Mesa 25.2.8-0ubuntu0.25.10.2
# OpenGL shading language version string: 4.50

# 4) Read-only guarantee — only the deliverable changed since the base commit:
git diff --name-status 815df1e21..HEAD
# A   blitzy/documentation/kitty_815df1e210e0.md
git status --porcelain    # (empty output = clean tree)
```

### 9.7 Example Usage — kitty native debug facilities

```bash
# GPU / backend / GL context details
./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'true'
# Font resolution & fallback chain
./kitty/launcher/kitty --debug-font-fallback -o confirm_os_window_close=0 sh -c 'true'
# Effective configuration (defaults as observed)
./kitty/launcher/kitty --debug-config -o confirm_os_window_close=0 sh -c 'true'
```

### 9.8 Troubleshooting

- **Build stops at `wl_window.c` with `-Werror=switch`** → add `--ignore-compiler-warnings` (see 9.3). Environmental drift, not a kitty defect.
- **`GLFW initialization failed`** → `DISPLAY` is unset, Xvfb is not running, or `DISPLAY` points at a non-existent server (e.g. `:123`). Start Xvfb and `export DISPLAY=:99` (see 9.4).
- **`Failed to open systemd user bus with error: Connection refused`** → benign container artifact (no systemd user session); safe to ignore; does not affect exit code.
- **Suspect hardware GL / hang** → ensure `export LIBGL_ALWAYS_SOFTWARE=1` so Mesa selects the llvmpipe software rasterizer.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---------|---------|
| `export PATH=/usr/lib/go-1.24/bin:$PATH` | Put Go 1.24 on PATH for the build |
| `python3 setup.py build --verbose --ignore-compiler-warnings` | Canonical build (with Ubuntu 25.10 flag) |
| `Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset` | Start headless X server with GLX |
| `export DISPLAY=:99; export LIBGL_ALWAYS_SOFTWARE=1` | Target Xvfb; force Mesa software GL |
| `./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'true'` | Headless smoke run with GL debug |
| `./kitty/launcher/kitty --version` | Print version banner |
| `glxinfo \| grep OpenGL` | Confirm vendor/renderer/version/GLSL |
| `git diff --name-status 815df1e21..HEAD` | Verify only the deliverable changed |

### Appendix B — Port Reference

Not applicable — kitty is a desktop terminal emulator; the early-startup trace uses **no network (TCP/UDP) ports**. Single-instance coordination, when enabled, uses a **UNIX domain socket** (not a TCP port). The only "address" used here is the X display number **`:99`** (Xvfb), which is a local display, not a network port.

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only added file) |
| `kitty/launcher/main.c` | Native process entry; embeds CPython |
| `kitty/main.py` | Backend selection; `set_scale → set_font_family → _run_app` |
| `kitty/boss.py` | First OS-window creation |
| `kitty/glfw.c` | GPU bootstrap; DPI derivation; window hints |
| `kitty/gl.c` | GL load + version/extension gate (fatal if < 3.3) |
| `kitty/fonts.c` | `calc_cell_metrics()` — cell width/height/baseline |
| `kitty/screen.c` | `report_device_attributes()` — Primary/Secondary DA |
| `kitty/shaders.c` | Exposes `GL_VENDOR/RENDERER/VERSION/GLSL` strings |
| `glfw/x11_window.c` | GLX-vs-EGL context-source selection |
| `kitty/launcher/kitty`, `kitty/fast_data_types.so` | Build artifacts (git-ignored) |

### Appendix D — Technology Versions (observed)

| Component | Version |
|-----------|---------|
| kitty | 0.35.2 |
| Python | 3.13.7 |
| Go | 1.24.4 |
| gcc | 15.2.0 |
| pkg-config | 1.8.1 |
| git / git-lfs | 2.51.0 / 3.7.1 |
| Mesa (GL) | 25.2.8 (llvmpipe, LLVM 20.1.8, 256 bits) |
| OpenGL / GLSL | 4.5 Core Profile / 4.50 |
| harfbuzz / freetype2 / fontconfig | 10.2.0 / 26.2.20 / 2.15.0 |
| lcms2 / libpng / xkbcommon | 2.16 / 1.6.50 / 1.7.0 |
| x11 / xcb | 1.8.12 / 1.17.0 |

### Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `PATH` | prepend `/usr/lib/go-1.24/bin` | Make Go 1.24 available to the build |
| `CI` | `true` | Non-interactive build/tooling mode |
| `DISPLAY` | `:99` | Target the Xvfb virtual X server |
| `LIBGL_ALWAYS_SOFTWARE` | `1` | Force Mesa llvmpipe software GL |
| `KITTY_VCS_REV` | (build-time) | Git rev stamped into the version banner |

### Appendix F — Developer Tools Guide (kitty native debug flags)

| Flag | Reveals |
|------|---------|
| `--debug-rendering` / `--debug-gl` | GL strings, selected backend, context source, extension checks |
| `--debug-font-fallback` | Font family resolution and fallback chain |
| `--debug-config` | Effective (default) configuration values |
| `--debug-input` / `--debug-keyboard` | Input/keyboard and capability-reporting details |
| `--version` | Version banner (`kitty 0.35.2 …`), VCS-stamped |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the primary directive defining project scope |
| **llvmpipe** | Mesa's Gallium software (CPU) OpenGL rasterizer used when no GPU is present |
| **GLX / EGL** | Two mechanisms for creating an OpenGL context on X11; kitty uses GLX (native) here |
| **DPI** | Dots per inch; kitty derives logical DPI as `content_scale × 96` on Linux |
| **Cell metrics** | Per-character cell width/height/baseline derived from font metrics × logical DPI |
| **DA (Device Attributes)** | Terminal capability query/response (Primary `ESC [ c`, Secondary `ESC [ > c`) |
| **DECRQM** | DEC Request Mode — queries whether a given terminal mode is set |
| **XTVERSION / XTGETTCAP** | xterm extensions for version reporting and terminfo capability queries |
| **PTY** | Pseudo-terminal; the channel over which capability query/response bytes were captured |
| **Xvfb** | X virtual framebuffer — an in-memory X server enabling headless GUI/GL execution |
