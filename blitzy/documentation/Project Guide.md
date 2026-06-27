# Blitzy Project Guide
## kitty Terminal Emulator — Startup Investigation (Documentation Deliverable)

> **Commit/version pin:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` · kitty **v0.35.2**
> **Branch:** `blitzy-f95fb25b-8558-44e2-b163-a5fe70ef6834` · **HEAD:** `676daef2e`
> **Deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` (554 lines)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, code-grounded investigation document that explains **everything that happens when the kitty terminal emulator starts up** — from process launch until the terminal is ready to communicate with and render output from a shell — pinned to commit `815df1e2` (kitty v0.35.2). It targets kitty maintainers, contributors, and terminal-internals learners. Every claim is backed by a `[path:locator]` source citation and, where applicable, by **real runtime output** captured from a headless launch (Xvfb + Mesa llvmpipe software OpenGL). The scope is intentionally isolated: exactly one markdown file is added and **no** existing repository file is modified, so the change carries zero build, runtime, or behavioral risk to kitty itself.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextSize':'14px'}}}%%
pie showData
    title Completion — 89.8% Complete (44h of 49h)
    "Completed Work (AI)" : 44
    "Remaining Work" : 5
```

| Metric | Hours |
|---|---|
| **Total Hours** | **49** |
| Completed Hours — AI (autonomous) | 44 |
| Completed Hours — Manual | 0 |
| **Completed Hours — Total** | **44** |
| **Remaining Hours** | **5** |
| **Percent Complete** | **89.8%** |

> Completion is computed per the AAP-scoped methodology: `Completed ÷ (Completed + Remaining) = 44 ÷ 49 = 89.8%`. The work universe is (a) the AAP-specified investigation/authoring work and (b) the path-to-production for a documentation artifact (human review/merge). There is no application deployment, CI, or integration to perform — the document is standalone markdown that is not imported, built, linked, or tested by any other file.

### 1.3 Key Accomplishments

- ✅ Authored the complete 554-line investigation answering all four question clusters (Q1 startup systems, Q2 initial configuration, Q3 terminal↔shell readiness, Q4 display evidence), each with explicit rationale.
- ✅ Built kitty headlessly (`python3 setup.py --ignore-compiler-warnings` → exit 0; 122 C translation units; `fast_data_types.so` + native launcher produced; `--version` = "kitty 0.35.2").
- ✅ Stood up a reproducible headless harness (Xvfb + Mesa **llvmpipe**, `LIBGL_ALWAYS_SOFTWARE=1`) and captured first-hand evidence from `--debug-rendering`, `--debug-font-fallback`, and `--dump-bytes`, all exiting 0.
- ✅ Documented **6 code-as-truth corrections** that overturn common kitty lore (platform-specific OpenGL floor Linux 3.1 / macOS 3.3; no explicit core-profile request on Linux; kitty does not print `GL_VENDOR`/`GL_RENDERER` at startup; **13** `*.glsl` files; conditional tempfile fallback returns `None`; **no `--debug-config` CLI flag**).
- ✅ Verified **179** unique `[path:locator]` citations across **51** source files with **zero** inaccuracies, plus a self-applied validation checklist and a verified citation map.
- ✅ Honored both hard constraints: **read-only repository** (zero source/build/config changes branch-wide) and **cleanup** (clean working tree; no Xvfb, locks, or temp logs left behind).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None._ Automated validation found zero unresolved defects: 179/179 citations valid, build exit 0, runtime exit 0, read-only and cleanup contracts verified. | No release-blocking issues. The only gate is human SME review (see §1.6 / §2.2). | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| — | — | **No access issues identified.** The repository, build toolchain, and headless stack (Xvfb, Mesa llvmpipe, fonts) were all fully accessible in the authoritative container; the build and headless launches completed with exit 0. | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Conduct an SME technical review of `blitzy/documentation/kitty_815df1e210e0.md` — validate the Q1–Q4 explanations, the 6 corrections, and the reasoning; sanity-check a sample of the 179 citations against source at commit `815df1e2`.
2. **[Medium]** Perform an editorial proofread and confirm the markdown renders correctly (Mermaid flowchart, tables, intra-doc anchors); optionally re-run a citation spot-check.
3. **[Medium]** Approve the pull request and merge the single-file addition to the target branch.
4. **[Low]** (Optional) If publishing from a different environment, refresh the cosmetic evidence version strings (Mesa/LLVM) — already disclosed as non-material in the document's Limitations section.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement. All work was performed autonomously (AI). Total = **44 hours**.

| Component | Hours | Description |
|---|---|---|
| A. Headless software-OpenGL research | 2 | Confirmed the no-GPU approach: Xvfb + Mesa **llvmpipe** + `LIBGL_ALWAYS_SOFTWARE=1` to obtain the OpenGL context kitty requires (AAP §0.2.2). |
| B. Build + headless harness | 6 | Built kitty (122 C TUs, exit 0), resolved the gcc-15 `-Werror=switch` Wayland quirk via the project-native `--ignore-compiler-warnings` flag (touches no source), brought up Xvfb/llvmpipe, confirmed renderer with `glxinfo -B`, and captured evidence across diagnostic flags. |
| C. Q1 — Startup systems | 5 | Traced the 9-stage cold-start chain in source order, authored the flowchart, captured the 4-line `--debug-rendering` evidence, and explained the stdout-buffering ordering subtlety. |
| D. Q2 — Initial configuration | 4 | Documented `_get_config_dir()` precedence, built-in defaults, the unknown-key vs. bad-line two-path distinction, and the applied-config evidence + guardrail. |
| E. Q3 — Terminal↔shell readiness | 5 | Documented PTY allocation, `TERM`/`COLORTERM`, terminfo provisioning, shell-integration (OSC 133/7), the VT-parser→screen pipeline, and the `--dump-bytes` proof incl. the ONLCR `\r\r\n` reconciliation via `od -c`. |
| F. Q4 — Display evidence | 4 | Documented font discovery/rasterization, glyph cache → GPU atlas, the GLSL compile/link pipeline, scrollback, and the `--debug-font-fallback` evidence. |
| G. Code-as-truth corrections (6) | 3 | Verified and documented six corrections that overturn naive assumptions against the source at this commit. |
| H. Document structure & scaffolding | 2 | Title, preamble, table of contents, citation legend, and the Limitations & Honesty section. |
| I. Self-validation + verified citation map | 4 | Cross-checked 179 citations across 51 files with line-content verification; authored the self-applied validation checklist. |
| J. Code-review remediation (F1–F8) | 4 | Addressed eight review findings (commit `e8cacf3a6`, +186/−22): config-warning attribution, tempfile-fallback semantics, shader compile/link citations, the duplicate-CR reconciliation, the checklist/citation map, bare-path→citation conversions, build evidence, and the buffering explanation. |
| K. Final autonomous validation (5 gates) | 5 | Re-verified citations (0 inaccuracies), rebuilt (exit 0), re-ran the runtime harness (all flags exit 0), audited dependencies, aligned 3 environment-drift values to the authoritative image (re-verified live), and cleaned up. |
| **Total** | **44** | |

### 2.2 Remaining Work Detail

Each remaining item is path-to-production for a documentation artifact (human-gated). No compilation, configuration, integration, or deployment tasks apply to this AAP. Total = **5 hours**.

| Category | Hours | Priority |
|---|---|---|
| SME technical review of the document | 3 | High |
| Editorial proofread + citation/markdown spot-check | 1 | Medium |
| PR approval + merge | 1 | Medium |
| **Total** | **5** | — |

### 2.3 Reconciliation

- Section 2.1 total (**44**) = Completed Hours in §1.2 ✔
- Section 2.2 total (**5**) = Remaining Hours in §1.2 = Section 7 pie "Remaining Work" ✔
- Section 2.1 + Section 2.2 = **44 + 5 = 49** = Total Project Hours in §1.2 ✔
- Completion = 44 ÷ 49 = **89.8%** (used consistently in §1.2, §7, §8) ✔

---

## 3. Test Results

For this documentation deliverable, "tests" are the **autonomous validation checks** executed by Blitzy's validation systems (the Final Validator's five gates) and independently corroborated during this assessment. All entries originate from Blitzy's autonomous validation logs for this project; the AAP adds no test code to the repository (out of scope). "Coverage %" denotes pass completeness for each validation category.

| Test Category | Framework / Tool | Total | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Citation accuracy | grep + source cross-reference | 179 | 179 | 0 | 100% | 179 unique `(path,locator)` pairs across 51 files (274 raw occurrences); every cited line supports its claim. |
| Code-as-truth corrections | Manual source verification | 6 | 6 | 0 | 100% | C1–C6 confirmed against source at `815df1e2`. |
| Build — compile + link | `python3 setup.py --ignore-compiler-warnings` | 1 | 1 | 0 | 100% | Exit 0; 122 C translation units; 5 link steps; `fast_data_types.so` + launcher produced. |
| Build — negative control | `python3 setup.py` (plain) | 1 | 1 | 0 | 100% | Exit 1 **as expected** (gcc-15 `-Werror=switch`, Wayland backend only; irrelevant to X11/Xvfb). |
| Runtime harness — diagnostic flags | Xvfb + Mesa llvmpipe | 5 | 5 | 0 | 100% | `--debug-rendering`, `--debug-font-fallback`, `--dump-bytes`, unknown-config-key warning, and `--debug-config` rejection — all exit as designed. |
| Q3 child environment | `infocmp` + env capture | 1 | 1 | 0 | 100% | `TERM=xterm-kitty`, `COLORTERM=truecolor`, `TERMINFO`→bundled, `/dev/pts/0` confirmed. |
| Markdown integrity | `iconv` + `grep` | 1 | 1 | 0 | 100% | 16 balanced code blocks, valid UTF-8, intra-doc anchors resolve. |
| Read-only / scope | `git diff` | 1 | 1 | 0 | 100% | Exactly one file added; zero source/build/config changes branch-wide. |
| Dependency presence | `pkg-config` | 1 | 1 | 0 | 100% | All native `-dev` libs present (harfbuzz 10.2.0 ≥ 1.5, freetype2, fontconfig, lcms2, libpng, x11, xkbcommon, gl). |
| **Totals** | — | **196** | **196** | **0** | **100%** | All checks sourced from Blitzy autonomous validation logs. |

---

## 4. Runtime Validation & UI Verification

The deliverable is a static markdown document, so there is **no graphical UI** to verify; "runtime validation" refers to the headless launch of kitty that grounds the document's evidence. Outcomes (from the Xvfb + llvmpipe harness):

- ✅ **Operational** — Build: `python3 setup.py --ignore-compiler-warnings` exits 0; artifacts `kitty/launcher/kitty` and `kitty/fast_data_types.so` produced.
- ✅ **Operational** — Software OpenGL context: `glxinfo -B` reports Mesa / **llvmpipe** / OpenGL **4.5** core (comfortably above kitty's Linux floor of 3.1).
- ✅ **Operational** — GLFW / OS window: `--debug-rendering` prints "OS Window created".
- ✅ **Operational** — OpenGL version gate: GL version line printed and accepted (no fatal).
- ✅ **Operational** — PTY child spawn: `--debug-rendering` prints "Child launched".
- ✅ **Operational** — Font subsystem: `--debug-font-fallback` prints "Text fonts:" with four DejaVuSansMono faces plus an emoji fallback (Noto Color Emoji).
- ✅ **Operational** — VT parser → screen: `--dump-bytes` shows `draw …`, `select_graphic_rendition`, `screen_carriage_return`, and `screen_linefeed` dispatched correctly.
- ✅ **Operational** — Config parsing: an unknown config key produces the expected "Ignoring unknown config key" warning; valid keys are accepted silently.
- ⚠ **Partial / benign** — systemd user bus: "Failed to open systemd user bus … Connection refused" is expected in a container without a systemd user bus and is **not** a startup failure (documented in the deliverable's Limitations).
- **UI Verification:** **N/A** — no application UI is in scope; the document itself was integrity-checked (Mermaid flowchart, tables, anchors, UTF-8) instead.

---

## 5. Compliance & Quality Review

Cross-mapping AAP deliverables and constraints to Blitzy quality/compliance benchmarks. Fixes applied during autonomous validation are noted.

| AAP Requirement / Constraint | Benchmark | Status | Notes |
|---|---|---|---|
| Single file, correct name & location | Deliverable contract | ✅ Pass | `blitzy/documentation/kitty_815df1e210e0.md` (= `<source_branch_name>.md`). |
| Read-only repository | No existing source modified | ✅ Pass | `git diff` vs base = one add; zero `.py`/`.c`/`.h`/`.glsl`/config/build changes. |
| No extra code added | Documentation-only | ✅ Pass | Only the markdown file; no scripts/fixtures/tests committed. |
| Build-and-run performed | Evidence-grounded | ✅ Pass | First-hand build output (exit 0) + runtime logs quoted. |
| Code-as-truth, no assumptions | 100% citation accuracy | ✅ Pass | 179/179 citations valid across 51 files. |
| Rationale provided | Each Q1–Q4 includes reasoning | ✅ Pass | Explicit "Rationale" subsections throughout. |
| Cleanup of temporary artifacts | Clean working tree | ✅ Pass | No Xvfb, locks, temp logs, or throwaway config left behind. |
| Pinned to commit & version | Determinism | ✅ Pass | All citations/behavior pinned to `815df1e2` / v0.35.2. |
| Q1–Q4 answered in full | Content completeness | ✅ Pass | All four clusters with code + captured evidence. |
| 6 code-as-truth corrections | Technical accuracy | ✅ Pass | C1–C6 verified against source. |
| Headless harness (Xvfb + llvmpipe) | Reproducibility | ✅ Pass | Documented, ordered, reproducible command sequence. |

**Fixes applied during autonomous validation:** 8 code-review findings resolved (F1–F8, commit `e8cacf3a6`); 3 citation locators corrected (commit `518ed0281`); 3 environment-drift evidence values aligned to the authoritative image and re-verified live (commit `676daef2e`).

**Outstanding compliance items:** none beyond the human SME review gate (§2.2).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Captured evidence cosmetics (Mesa/LLVM version strings, font paths, timestamps) drift if re-run on a different image | Technical | Low | Medium | Document explicitly scopes these as cosmetic and pins all source citations/behavior to the commit; values already aligned to the authoritative env. | Mitigated |
| Citation line numbers would drift if the underlying repo advances past `815df1e2` | Technical | Low | Low | Intentional commit-pinned snapshot; every citation references the fixed commit. | Accepted (by design) |
| All GL evidence is software (llvmpipe), not hardware-accelerated | Technical | Low | N/A | Limitations section scopes claims to **correctness**, not performance. | Disclosed / Accepted |
| Security exposure from the change | Security | None | N/A | Static markdown; no code execution, auth, data, or network surface; zero dependencies added; zero source modified. | N/A |
| Document not wired into the Sphinx docs build (won't appear in rendered docs) | Operational | Informational | N/A | By design — AAP mandates a standalone file under `blitzy/documentation/`. | By design |
| Integration/coupling regressions | Integration | None | N/A | Document is not imported/built/linked/tested by any file; cannot affect kitty's build or behavior. | N/A |
| Read-only constraint violation (would be release-blocking) | Process/Compliance | Critical (potential) | None (verified) | Verified zero source/build/config changes branch-wide. | Verified clean |
| Cleanup contract violation (stray temp artifacts) | Process/Compliance | Low (potential) | None (verified) | Verified clean working tree; no Xvfb/locks/temp. | Verified clean |
| Human review not yet performed (technical-interpretation soundness) | Process | Low | Low | Schedule SME review (remaining task, §2.2); automated validation already found 0 issues. | Open |

**Overall risk profile: LOW.** No High/Critical risks are open; the dominant residual items are cosmetic environment-drift (mitigated) and the pending human review (the 5h remaining work).

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextSize':'14px'}}}%%
pie showData
    title Project Hours — 44 Completed / 5 Remaining
    "Completed Work" : 44
    "Remaining Work" : 5
```

**Remaining hours by category** (from §2.2; "Remaining Work" total = **5h**, matching §1.2 and the pie above):

| Category | Hours | Bar |
|---|---|---|
| SME technical review | 3 | ███████████████ |
| Editorial proofread + spot-check | 1 | █████ |
| PR approval + merge | 1 | █████ |
| **Total** | **5** | |

> **Color key (Blitzy brand):** Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`; accents = Violet-Black `#B23AF2`.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a thorough, code-grounded, headless-verified investigation of kitty's startup path as a single standalone document. All four question clusters are answered with explicit rationale and real captured evidence; six code-as-truth corrections sharpen accuracy beyond common lore; and 179 citations across 51 files were verified with zero inaccuracies. The build (exit 0) and headless runtime (all diagnostic flags exit 0) confirm the document's "reproduced during this investigation" claims hold in the authoritative environment.

**Remaining gaps.** The work is **89.8% complete (44 of 49 hours)**. The remaining **5 hours** is entirely the human-gated path-to-production for a documentation artifact: an SME technical review (3h), an editorial/citation spot-check (1h), and PR approval + merge (1h). There is no application deployment, CI integration, or runtime service to stand up.

**Critical path to production.** SME review → minor edits if any → merge. Because the change is read-only with respect to all existing files and introduces no code coupling, the merge carries negligible technical risk.

**Success metrics.** Citation accuracy 100% (179/179); build exit 0; runtime exit 0; read-only and cleanup contracts verified; markdown integrity intact.

**Production-readiness assessment.** The deliverable is **ready for human review**. Pending only SME sign-off and merge, it satisfies every AAP requirement and constraint. Completion is capped below 100% to reflect the irreducible human-review gate.

| Metric | Value |
|---|---|
| Completion | 89.8% (44h / 49h) |
| Open critical/high defects | 0 |
| Citation accuracy | 100% (179/179) |
| Source files modified | 0 |
| Files added | 1 (`blitzy/documentation/kitty_815df1e210e0.md`) |
| Recommended action | SME review → merge |

---

## 9. Development Guide

All commands below were tested in the authoritative container. Paths are relative to the repository root.

### 9.1 System Prerequisites

| Tool | Verified Version |
|---|---|
| Python | 3.13.7 |
| GCC | 15.2.0 (Ubuntu) |
| Git | 2.51.0 |
| Go | 1.24.4 |

Headless stack (all present): **Xvfb**, **glxinfo** (mesa-utils), **fc-match**, **infocmp**, **od**. Native `-dev` libraries (via `pkg-config`): harfbuzz 10.2.0 (≥ 1.5 required), freetype2, fontconfig, lcms2, libpng, x11, xkbcommon, gl.

### 9.2 View / Verify the Deliverable

```bash
# Confirm the single deliverable exists (expect: 554 lines)
wc -l blitzy/documentation/kitty_815df1e210e0.md

# Verify the read-only constraint (expect exactly: A  blitzy/documentation/kitty_815df1e210e0.md)
git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD

# Markdown integrity: fence lines must be even (expect 32 → 16 balanced blocks); UTF-8 valid
grep -c '^```' blitzy/documentation/kitty_815df1e210e0.md
iconv -f UTF-8 -t UTF-8 blitzy/documentation/kitty_815df1e210e0.md >/dev/null && echo "UTF-8 OK"

# Working tree must be clean
git status --porcelain
```

### 9.3 Reproduce the Headless Harness (optional — regenerates the evidence)

```bash
# 1. Build (canonical for this toolchain; compiles fast_data_types, vendored GLFW, launcher)
python3 setup.py --ignore-compiler-warnings        # expect: exit 0

# 2. Virtual X11 display (Xvfb has no GPU)
Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &
export DISPLAY=:99

# 3. Force Mesa's llvmpipe software renderer
export LIBGL_ALWAYS_SOFTWARE=1

# 4. Confirm the active (software) renderer (expect: Mesa / llvmpipe / OpenGL 4.5 core)
glxinfo -B

# 5. Launch kitty with a diagnostic flag against a self-terminating child
kitty/launcher/kitty --debug-rendering sh -c 'true'
#   swap in: --debug-font-fallback | --debug-keyboard | --dump-bytes <file> | -o opt=val
```

### 9.4 Verification Steps

- **Build:** expect `exit=0`, "Linking … fast_data_types", "Linking launcher", and `kitty/launcher/kitty --version` → "kitty 0.35.2".
- **Renderer:** `glxinfo -B` shows `llvmpipe` and `OpenGL core profile version string: 4.5`.
- **Startup log (`--debug-rendering`):** "OS Window created", "Child launched", and a "GL version string: …" line.
- **Fonts (`--debug-font-fallback`):** "Text fonts:" followed by the resolved faces.
- **Bytes (`--dump-bytes <file>`):** `draw …`, `select_graphic_rendition …`, `screen_carriage_return`, `screen_linefeed`.

### 9.5 Troubleshooting

- **Plain `python3 setup.py` fails with `-Werror=switch` in `glfw/wl_window.c`** → use `--ignore-compiler-warnings` (gcc-15 + newer wayland-protocols add `xdg-shell` enums; Wayland-only, irrelevant to the X11/Xvfb target; the flag modifies no source).
- **"Failed to open systemd user bus … Connection refused"** → benign in a container without a systemd user bus; not a startup failure.
- **Fatal "OpenGL version is X.Y, version >= … required"** → ensure `LIBGL_ALWAYS_SOFTWARE=1` is exported and `glxinfo -B` reports `llvmpipe`.
- **Cleanup after the harness** → stop Xvfb by its exact PID (`kill "$(pgrep -x Xvfb)"`; never use broad `pkill`), then `rm -f /tmp/.X99-lock` and remove any temporary logs/config. Build outputs (`*.so`, `build/`) are gitignored.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Build (canonical) | `python3 setup.py --ignore-compiler-warnings` |
| Build entry point | `Makefile` `all:` → `python3 setup.py $(VVAL)` |
| Start virtual display | `Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &` |
| Confirm renderer | `glxinfo -B` |
| Launch (render debug) | `kitty/launcher/kitty --debug-rendering sh -c 'true'` |
| Launch (font debug) | `kitty/launcher/kitty --debug-font-fallback sh -c 'true'` |
| Launch (byte dump) | `kitty/launcher/kitty --dump-bytes /tmp/bytes.dump sh -c '…'` |
| Verify scope | `git diff --name-status 815df1e2 HEAD` |
| Verify version | `kitty/launcher/kitty --version` |

### B. Port / Display Reference

| Resource | Value | Notes |
|---|---|---|
| Network ports | **None** | The investigation uses no network services. |
| Virtual X11 display | `:99` | Provided by Xvfb for the headless harness. |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only file added). |
| `kitty/launcher/main.c` | Native launcher (embeds CPython). |
| `kitty/main.py` | `main()` / `AppRunner` / `_run_app` orchestration. |
| `kitty/glfw.c`, `kitty/gl.c` | GLFW init + OpenGL version gate. |
| `kitty/child.py` | PTY allocation, env, terminfo, child spawn. |
| `kitty/vt-parser.c`, `kitty/screen.c` | VT parser → screen pipeline. |
| `kitty/fonts/`, `kitty/freetype.c`, `kitty/fontconfig.c` | Font discovery + rasterization. |
| `kitty/shaders.c`, `kitty/*.glsl` (13 files) | GLSL compile/link pipeline. |
| `setup.py`, `Makefile` | Build entry points. |

### D. Technology Versions

| Component | Version |
|---|---|
| kitty | 0.35.2 (commit `815df1e2`) |
| Python | 3.13.7 |
| GCC | 15.2.0 |
| Go | 1.24.4 |
| Mesa / llvmpipe | 25.2.x / LLVM 20.1.8 (OpenGL 4.5 core) |
| HarfBuzz | 10.2.0 |
| Vendored GLFW | 3.4 |

### E. Environment Variable Reference

| Variable | Purpose |
|---|---|
| `DISPLAY` | Targets the Xvfb virtual display (`:99`). |
| `LIBGL_ALWAYS_SOFTWARE` | Forces Mesa's llvmpipe software OpenGL (no GPU under Xvfb). |
| `KITTY_CONFIG_DIRECTORY` | Overrides the config-directory search (used to demonstrate config application). |
| `TERM` | Set by kitty for the child to `xterm-kitty`. |
| `COLORTERM` | Set by kitty for the child to `truecolor`. |
| `TERMINFO` | Points the child at kitty's bundled terminfo database. |
| `KITTY_SHELL_INTEGRATION` | Set by shell-integration env injection. |

### F. Developer Tools Guide (kitty diagnostic flags)

| Flag | Effect |
|---|---|
| `--debug-rendering` / `--debug-gl` | GL/render diagnostics; error-checks all OpenGL calls; prints the GL version line. |
| `--debug-font-fallback` | Dumps resolved font matches ("Text fonts:" + fallback faces). |
| `--debug-input` / `--debug-keyboard` | Prints inbound key/mouse events (not exercised in a non-interactive run). |
| `--dump-bytes <file>` | Writes raw bytes received from the child and logs parsed commands. |
| `-o key=value` | One-off config override on the command line. |
| _`--debug-config`_ | **Does not exist** as a CLI flag; `debug_config` is a keyboard action (`kitty_mod+f6`). |

### G. Glossary

| Term | Definition |
|---|---|
| Xvfb | X virtual framebuffer — a headless X11 server with no GPU. |
| llvmpipe | Mesa's LLVM-based CPU (software) OpenGL rasterizer. |
| PTY | Pseudo-terminal: master/slave pair connecting kitty to the shell. |
| VT parser | kitty's escape-sequence state machine (`kitty/vt-parser.c`). |
| OSC 133 / OSC 7 | Shell-integration escape codes for prompt marks / CWD reporting. |
| ONLCR | Terminal output flag mapping `\n` → `\r\n` (explains the observed `\r\r\n`). |
| SGR | Select Graphic Rendition — the `ESC[…m` styling sequence. |
| AAP | Agent Action Plan — the directive defining this project's scope. |

> **Cross-section integrity (validated before submission):** §1.2 Remaining (5) = §2.2 sum (5) = §7 pie "Remaining Work" (5); §2.1 (44) + §2.2 (5) = §1.2 Total (49); completion 44 ÷ 49 = 89.8% used consistently in §1.2/§7/§8; all Section 3 entries originate from Blitzy autonomous validation logs; Blitzy brand colors applied (Completed `#5B39F3`, Remaining `#FFFFFF`).