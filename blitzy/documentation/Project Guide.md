# Blitzy Project Guide
### kitty Graphics Flow-Control & Backpressure — Runtime-Grounded Answer Document

> **Task type:** Read-only Q&A / documentation investigation (governing rule "SWE-AtlasQnA-Repo")
> **Repository:** `kovidgoyal/kitty` @ base commit `815df1e21` · Branch `blitzy-ddea1b78-170f-467d-8b0e-3c89914d0bbf` · HEAD `97d2a396`
> **Sole deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` (1,108 lines)
> **Color legend:** 🟪 Completed / AI Work = **Dark Blue `#5B39F3`** · ⬜ Remaining = **White `#FFFFFF`** · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project answers a behavioral question about the **kitty terminal emulator**: how does it regulate the flow of terminal-graphics data when that data arrives faster than the terminal can comfortably process and respond to it? The deliverable is one runtime-grounded Markdown document that pinpoints — with byte-accurate `file:line` references and captured, unedited runtime output — where kitty decides to buffer, pause, or throttle on the input side; how it applies backpressure when writing responses under a stalled output path; and which adaptations are silent versus client-visible. The audience is engineers reasoning about kitty's I/O event loop, VT parser, and graphics subsystem. This is explicitly a **read-only investigation**: no source code is changed; the only new artifact is the answer document.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Completion — 89.2% (Hours)
    "Completed Work" : 33
    "Remaining Work" : 4
```

<p align="center"><b>■ 89.2% Complete</b> (🟪 Completed <code>#5B39F3</code> · ⬜ Remaining <code>#FFFFFF</code>)</p>

| Metric | Hours |
|--------|-------|
| **Total Hours** | **37** |
| **Completed Hours** (AI: 33 · Manual: 0) | **33** |
| **Remaining Hours** | **4** |
| **Percent Complete** | **89.2%** |

> Completion is computed on AAP-scoped + path-to-production work only: `33 / (33 + 4) = 89.2%`.

### 1.3 Key Accomplishments

- ✅ Built kitty from source in the **default, canonical configuration** (`python3 setup.py build` → `-O3 -DNDEBUG`, `-Werror` clean, `kitty 0.35.2`).
- ✅ Answered **all five objectives** (OBJ-1 input pause/throttle, OBJ-2 write-side backpressure, OBJ-3 code-location map, OBJ-4 runtime signals, OBJ-5 quiet-vs-visible) with value + `file:line` + observed evidence.
- ✅ Exercised the **canonical PTY entry point** through two independent observation paths (in-process `kitty_tests` harness; headless `xvfb-run` PTY + `strace`), each confirmed stable across **≥2 runs**.
- ✅ Captured **actual, unedited runtime output** for every claim: graphics-protocol responses (`;OK`/`;ENODATA`/`;EINVAL`/`;ENOSPC`), the 320 MiB storage-quota eviction (`1280 → 2` images), and the verbatim 100 MiB write-cap drop log.
- ✅ Verified every `file:line` citation **byte-accurate** against source commit `815df1e21`.
- ✅ Preserved the **read-only constraint**: repository byte-for-byte unchanged except the doc; all temporary observation scripts and build artifacts cleaned up.
- ✅ Produced an honest **observed-vs-inferred ledger** with exact reproduction commands.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None_ | The single deliverable is validated production-ready: build clean, citations byte-accurate, runtime observations reproduce, repository unchanged except the doc. | — | — |

> **No critical unresolved issues.** No compilation errors, no failing checks, and no blockers remain.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _None_ | — | — | — | — |

> **No access issues identified.** Repository git operations, the C/Python/Go toolchain, and the container build/run environment were all confirmed working (git diff/log/status, `python3 setup.py build` EXIT=0, launcher `--version` all succeeded).

### 1.6 Recommended Next Steps

1. **[Medium]** Have a subject-matter expert read the answer document and confirm it answers the original question across all five objectives, then accept/sign off (≈2h).
2. **[Low]** Optionally re-run one or two of the documented probes (Path A and/or Path B) and re-verify a sample of `file:line` citations against commit `815df1e21` for independent confidence (≈2h).
3. **[Low]** If the document will be reused against a newer kitty, re-anchor the citations to the target commit (line numbers are pinned to `815df1e21`).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Build & runtime foundation | 2.0 | Canonical kitty build (`-O3 -DNDEBUG`), isolated `/tmp` workspace outside the checkout, headless (`xvfb`/`LIBGL_ALWAYS_SOFTWARE`) environment. |
| OBJ-1 — input read-pause investigation | 4.0 | Path B PTY flood > 1 MiB parser buffer; `strace` of `poll`/`read` on the `KittyChildMon` I/O thread; observed `POLLIN → 0` at 99.70%/99.98% `BUF_SZ` occupancy across 2 runs. |
| OBJ-2 — write-side backpressure investigation | 5.0 | Four modes (retention/control/eagain/cap); observed exactly 12 `EAGAIN` events, peak `write_buf_used` ≈631–634 KB, retention `50000/50000`, and the 100 MiB cap drop log via a bounded, stop-after-first-signal, PID-safe driver. |
| Graphics storage-pressure investigation | 4.0 | Path A in-process harness: real 320 MiB eviction (`1280 → 2` images at exactly `335544320` B), frame-cache `;ENOSPC` on the 9th frame, quiet `q=0/1/2` boundary, two-layer rejection. |
| OBJ-3 — code-location map | 3.0 | Traced the connected pipeline across 6 files; assembled the full `file:line` table + Mermaid diagram, each verified byte-accurate vs `815df1e21`. |
| OBJ-4 — runtime-signal capture & assembly | 2.0 | Assembled captured, unedited output next to each claim with the exact producing command (response bytes, `strace` lines, drop log). |
| OBJ-5 — quiet-vs-visible catalog | 1.5 | Classified each adaptation as silent / client-visible / operator-visible, grounding every row in a captured artifact. |
| Answer-document authoring | 5.0 | Wrote the 1,108-line document (six sections a–f), Mermaid diagrams, and GitHub-flavored Markdown tables (with escaped pipes). |
| Methodology compliance ledger | 2.0 | Observed-vs-inferred ledger, ≥2-run stability confirmation, exact build/invocation commands, and non-canonical labeling. |
| Read-only hygiene & cleanup | 1.5 | Isolated workspace, `timeout` wrappers, PID-safe reaping (never `pkill`), decoy-kitty safety proof, byte-for-byte-unchanged verification. |
| Iterative QA refinement & citation re-verification | 3.0 | Six-commit review/hardening history: code-review rewrite, probe hardening, stdout-scoping, and 100 MiB-cap arithmetic correction. |
| **Total Completed** | **33.0** | Matches Completed Hours in §1.2. |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review & acceptance sign-off (validate all 5 objectives answered; sanity-check reasoning & conclusions; confirm it answers the original question) | 2.0 | Medium |
| Independent reproduction spot-check & citation re-verification (re-run 1–2 Path A/Path B probes; re-verify a sample of the ~34 `file:line` citations vs `815df1e21`) | 2.0 | Low |
| **Total Remaining** | **4.0** | Matches Remaining Hours in §1.2 and §7. |

> **Cross-check:** §2.1 (33.0) + §2.2 (4.0) = **37.0** = Total Hours in §1.2. ✔

---

## 3. Test Results

> For this read-only Q&A / documentation task, the deliverable's "tests" are **validation checks** — citation accuracy, runtime reproduction, build verification, and read-only integrity. **Every row below originates from Blitzy's autonomous validation logs for this project** (plus independent PM re-verification of a sample). There is no deliverable application code to unit-test beyond these checks.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|:-----------:|:------:|:------:|:----------:|-------|
| Citation Accuracy | Source verification vs commit `815df1e21` (`sed`/`grep` + arithmetic) | 34 | 34 | 0 | 100% | Every `file:line` in the §(c) map (`BUF_SZ=1048576`, cap `=104857600`, quota `=335544320`, frame cache `×5=1677721600`). PM re-verified 12 independently. |
| Runtime Reproduction — Path A | In-process canonical `kitty_tests` PTY harness | 10 | 10 | 0 | 100% | Byte-identical across 2 runs: `;OK`/`;ENODATA`/`;EINVAL`/`;ENOSPC`, quiet `q=0/1/2` boundary, 320 MiB eviction `1280 → 2`, LRU oldest-first, two-layer rejection. |
| Runtime Reproduction — Path B | Headless PTY (`xvfb-run`) + `strace` on canonical launcher | 8 | 8 | 0 | 100% | Within stated stability ×2 runs: `POLLIN → 0`, 12 `EAGAIN`, `first_accept=3584 B`, retention `50000/50000`, control `50000/50000`, verbatim 100 MiB cap log. |
| Build / Compile Verification | `setup.py` (`-O3 -DNDEBUG`, `-Werror`) | 2 | 2 | 0 | 100% | C extensions + Go kitten; `EXIT=0`, 100% clean. Re-verified incremental build `EXIT=0` during assessment. |
| Read-only Integrity | `git diff` / `git status` | 2 | 2 | 0 | 100% | `git diff 815df1e21..HEAD` = only the doc added; working tree clean. |
| **TOTAL** | — | **56** | **56** | **0** | **100%** | All checks passed. |

---

## 4. Runtime Validation & UI Verification

**Runtime health (build & execution):**
- ✅ **Operational** — Canonical build `python3 setup.py build` → `EXIT=0`, `-Werror` clean (`-O3 -DNDEBUG`); launcher runs `kitty 0.35.2 created by Kovid Goyal`.
- ✅ **Operational** — Canonical entry path exercised in-process: `vt_parser` → `screen_handle_graphics_command` → `graphics.c` (Path A).
- ✅ **Operational** — Headless kitty launched under `xvfb-run` driving real PTY children with `strace` on the I/O-loop thread (Path B).

**API / protocol integration outcomes (kitty graphics protocol):**
- ✅ **Operational** — `;OK` success responses (with quiet-flag suppression at `q=1`/`q=2`).
- ✅ **Operational** — Error responses `;ENODATA`, `;EINVAL`, `;ENOSPC` reproduced verbatim.
- ✅ **Operational** — 320 MiB storage-quota eviction observed (`image_count 1280 → 2` at exactly `335544320` B; oldest-first LRU).
- ✅ **Operational** — 100 MiB write-buffer cap drop log reproduced verbatim (`"Too much data being sent to child with id: 1, ignoring it"`).
- ✅ **Operational** — Input read-pause (`POLLIN → 0`) and `EAGAIN`-aware write retention observed.

**UI verification:**
- ⚠ **Not applicable** — This is a headless, read-only investigation with **no UI deliverable**. kitty's GPU window was not the subject; graphics data was driven through the PTY/parser/screen/graphics pipeline headlessly. No visual screens to verify.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP deliverable and governing-rule mandates ("SWE-AtlasQnA-Repo") to Blitzy quality benchmarks.

| Benchmark / AAP Mandate | Status | Progress | Evidence |
|-------------------------|:------:|:--------:|----------|
| Deliverable at correct path `blitzy/documentation/kitty_815df1e210e0.md` | ✅ Pass | 100% | Present, 1,108 lines; committed. |
| All 5 objectives answered (OBJ-1…OBJ-5) with value + `file:line` + evidence | ✅ Pass | 100% | Doc §§(a)–(e); coverage recap in §(f). |
| Investigate-by-running (real captured output, not code-reading only) | ✅ Pass | 100% | Path A + Path B captured output throughout. |
| Canonical PTY entry point (no remote-control / debug-hook bypass) | ✅ Pass | 100% | In-process `kitty_tests` + `strace` on real launcher; §(f) non-canonical ledger. |
| Default / canonical build & config | ✅ Pass | 100% | `-O3 -DNDEBUG` build; §(f) build disclosure. |
| Observe at scale + stability across ≥2 runs | ✅ Pass | 100% | Two 20 s floods; `50000/50000` ×2; eviction/EAGAIN ×2. |
| Exercise edge/error branches (`ENOSPC`/`EINVAL`, eviction, 100 MiB cap) | ✅ Pass | 100% | Doc §(b)/§(d). |
| Before/during/after state for stateful mechanisms | ✅ Pass | 100% | Parser occupancy fill/pause; `write_buf` rise/drain. |
| Observed-vs-inferred distinction (honest labeling) | ✅ Pass | 100% | §(f) ledger; 2 items labeled INFERRED. |
| Read-only source repository (byte-for-byte unchanged) | ✅ Pass | 100% | `git diff 815df1e21..HEAD` = only the doc. |
| Temporary artifacts cleaned up | ✅ Pass | 100% | Workspace + probes + logs removed; tree clean. |
| Zero placeholders / complete content | ✅ Pass | 100% | No TODO/stub; full analytical answer. |

**Fixes applied during autonomous validation:**
- Replaced the inherited instrumented `--debug --extra-logging=event-loop` build with the canonical `-O3 -DNDEBUG` build the methodology requires (build artifacts are gitignored → read-only preserved).
- Corrected the 100 MiB-cap arithmetic and aligned the cap-onset ledger.
- Scoped the "byte-identical" claim to stdout and annotated the stderr `[PARSE ERROR]` timestamp.
- Hardened the embedded observation probes (including a `strace`-payload regex fix so the `EAGAIN` count matched).

**Outstanding compliance items:** None.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|:--------:|:-----------:|------------|--------|
| T1 — Citation line-numbers drift if checked against a kitty version other than `815df1e21` | Technical | Low | Medium | Doc pins the source commit prominently; re-anchor before reuse on a newer version. | Mitigated (documented) |
| T2 — Two secondary values are INFERRED, not observed (exact internal `read.sz` scalar; `repaint_delay` render coalescing) | Technical | Low | Low | Both honestly labeled INFERRED; a human can instrument to observe if a hard number is required. | Accepted / Documented |
| T3 — Timing-dependent magnitudes vary by host (instantaneous first-cycle `write_buf_used` ≈383 KB; 100 MiB cap onset ≈30 s) | Technical | Low | Low | Doc labels these instantaneous/timing-dependent and separates them from stable compile-time magnitudes (1/100/320 MiB). | Mitigated |
| S1 — No product security surface (documentation only; no code shipped, no auth/data/network, no dependency changes) | Security | N/A | — | Nothing to harden. | N/A |
| S2 — Observation ran kitty headless as root with `strace` + targeted `SIGKILL` | Security (environmental) | Low | Low | Temporary only; PID-safe reaping (never `pkill`); decoy-kitty safety proof; all artifacts removed. | Resolved |
| O1 — Reproducibility depends on toolchain/container (Python 3.13.7, gcc 15.2.0, go 1.22.12, `xvfb`) | Operational | Low | Medium | Doc records exact build + invocation commands and environment; stable magnitudes are compile-time constants (environment-independent). | Mitigated |
| O2 — Documentation staleness (kitty is actively developed) | Operational | Low–Medium | Medium | Commit-pinned; treat as a point-in-time reference. | Accepted |
| I1 — No external integrations, API keys, network config, or service dependencies | Integration | N/A | — | Doc lives outside all source trees; zero build coupling. | N/A |
| P1 — Read-only constraint must remain intact during human review (no accidental source edits) | Process/Scope | Low | Low | Awareness; re-verify `git diff` after review. | Mitigated |

> **Overall:** No High/Critical risks and no release-blocking issues. The profile is dominated by analytical accuracy / reproducibility of a point-in-time documentation artifact rather than code-in-production concerns.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Project Hours — 33 Completed / 4 Remaining
    "Completed Work" : 33
    "Remaining Work" : 4
```

**Remaining hours by category (from §2.2):**

| Category | Hours | Priority | Bar |
|----------|:-----:|:--------:|-----|
| SME technical review & acceptance | 2.0 | Medium | 🟪🟪 |
| Independent reproduction spot-check | 2.0 | Low | 🟪🟪 |
| **Total Remaining** | **4.0** | — | |

> **Integrity check:** "Remaining Work" = **4** here = Remaining Hours in §1.2 = sum of §2.2 Hours. ✔ · "Completed Work" = **33** = Completed Hours in §1.2. ✔

---

## 8. Summary & Recommendations

**Achievements.** The investigation delivers a rigorous, runtime-grounded answer to how kitty regulates terminal-graphics data under pressure. It shows that on the **input side** kitty *pauses* rather than buffers without bound — de-arming `POLLIN` (`child-monitor.c:L1501`) once the fixed 1 MiB parser buffer (`vt-parser.c:L18`) fills — and coalesces input via a 3 ms `input_delay`. On the **output side**, responses drain non-blockingly and are retained on `EAGAIN` (`write_to_child`, `child-monitor.c:L1443-L1479`), with a 100 MiB hard cap that drops data and logs an error. The graphics subsystem enforces a 320 MiB storage quota with oldest-first eviction and returns protocol errors (`;ENOSPC`/`;EINVAL`). Every mechanism is cited by byte-accurate `file:line` and demonstrated with captured, unedited output across two independent observation paths and ≥2 runs.

**Remaining gaps & critical path to production.** The deliverable is essentially complete. The critical path is **human review and acceptance** (2h) followed by an optional **independent reproduction spot-check** (2h) — the only remaining 4 hours. There is no application code to deploy, no CI/CD, and no environment configuration in scope, so "production" for this documentation deliverable means SME acceptance and publication.

**Success metrics.** All five objectives answered with value + `file:line` + observed evidence; 56/56 validation checks passed; repository byte-for-byte unchanged except the doc; honest observed-vs-inferred ledger present.

**Production-readiness assessment.** The project is **89.2% complete** (33h / 37h). The single deliverable is validated production-ready — comprehensive, byte-accurate, runtime-grounded, honest, committed, and read-only-compliant. It is recommended to proceed directly to SME acceptance; no rework is anticipated.

| Metric | Value |
|--------|-------|
| Completion | 89.2% (33h / 37h) |
| Validation checks passed | 56 / 56 (100%) |
| Objectives fully answered | 5 / 5 |
| Source files modified | 0 (read-only honored) |
| Critical unresolved issues | 0 |

---

## 9. Development Guide

> Every command below was executed and verified on the assessment host. Run from the repository root unless noted. Building kitty does **not** modify the tracked repository — all build outputs (`build/`, `kitty/launcher/kitt*`) are gitignored.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu-family container; verified on Ubuntu 25.10).
- **Toolchain (verified present):**
  - Python **3.13.7**
  - gcc **15.2.0**
  - Go **1.22.12**
- **Build helpers:** Pillow **12.3.0**, pygments **2.20.0**.
- **Observation tooling:** `xvfb-run` (`/usr/bin/xvfb-run`), `strace` (`/usr/bin/strace`).
- **System libraries** (provided by the container): `libgl1-mesa-dev`, `libxkbcommon-dev`, `libharfbuzz-dev`, `libpng-dev`, `liblcms2-dev`, `libfontconfig-dev`, `libxxhash-dev`, and related X/Wayland libs.

```bash
# Verify the toolchain
python3 --version          # -> Python 3.13.7
gcc --version | head -1    # -> gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
go version                 # -> go version go1.22.12 linux/amd64
python3 -c "import PIL, pygments; print('Pillow', PIL.__version__, '| pygments', pygments.__version__)"
```

### 9.2 Environment Setup

```bash
export PATH="$PATH:/usr/local/go/bin"
export GOPATH="$HOME/go"
export LANG=C.UTF-8
export LC_ALL=C.UTF-8
# For headless (Path B) GUI-less runs only:
export LIBGL_ALWAYS_SOFTWARE=1
```

### 9.3 Dependency Installation

No dependency changes are required by this task — the toolchain and libraries are investigation prerequisites already present in the container. The canonical build resolves all implicit build dependencies. (No `pip install`, no `go get`, no manifest edits.)

### 9.4 Build (Canonical)

```bash
# Canonical, default build a normal user would run (-O3 -DNDEBUG)
python3 setup.py build            # expected: exit status 0
# Produces the launcher at kitty/launcher/kitty
```

Expected: build completes with `EXIT=0`. A benign message — `Package 'wayland-protocols' ... not found` → `Disabling building of wayland backend` — is normal and does **not** affect the X11/headless observation path.

### 9.5 Run & Verify

```bash
# Launcher version (canonical run)
./kitty/launcher/kitty --version         # -> kitty 0.35.2 created by Kovid Goyal

# Read the deliverable
sed -n '1,60p' blitzy/documentation/kitty_815df1e210e0.md
wc -l blitzy/documentation/kitty_815df1e210e0.md    # -> 1108

# Verify the read-only constraint (must show ONLY the doc added)
git diff --name-status 815df1e21..HEAD   # -> A  blitzy/documentation/kitty_815df1e210e0.md
git status --porcelain                   # -> (empty = clean working tree)
```

### 9.6 Example Usage — Reproducing the Observations

**Path A — in-process canonical harness** (drives the real `vt_parser` → `screen_handle_graphics_command` → `graphics.c` chain; reproduces response bytes, quiet boundary, 320 MiB eviction, LRU):

```bash
cd "$WS/src"
PYTHONPATH="$WS/src" CI=true python3 "$WS/pathA_probe.py"     # + pathA_evict.py, pathA_lru.py
```

**Path B — headless PTY + strace** (reproduces read-pause `POLLIN → 0`, `EAGAIN` drain, and the 100 MiB cap):

```bash
# Launch a child on kitty's PTY, headless under Xvfb:
timeout 90 xvfb-run -a -s "-screen 0 1280x800x24" \
  ./kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 obj1_child.py "$LOG" 20

# Locate the I/O-loop thread and strace it:
TID=$(for t in /proc/$(pgrep -x kitty)/task/*; do \
        [ "$(cat "$t/comm")" = KittyChildMon ] && basename "$t"; done)
timeout 30 strace -tt -p "$TID" -e trace=poll,read -o "$STRACE"
```

### 9.7 Troubleshooting

- **`Package 'wayland-protocols' ... not found`** during build — benign; kitty disables the Wayland backend. The X11/`xvfb` observation path is unaffected; build still returns `EXIT=0`.
- **`xvfb-run: 200: 0: not found`** — the `-s "-screen 0 1280x800x24"` value contains spaces; pass the launch command via a **bash array**, not an unquoted scalar string (see doc §(b)).
- **Path B launcher always exits `0`** — the `kitty` launcher masks the child's exit code. Judge success from the child **LOG contents** (e.g. `RETENTION sent=50000 received_OK=50000`, `FLOOD done …`), not the launcher's exit code (see doc §(f)).
- **Killing a spawned kitty safely** — never `pkill kitty`. Resolve the specific PID as a descendant of your own `timeout`/`xvfb-run` wrapper (via a `pgrep -P` walk) and `SIGKILL` exactly that PID.
- **Citations don't match your source** — you are likely on a different commit; the doc's line numbers are pinned to `815df1e21`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Canonical build | `python3 setup.py build` |
| Launcher version | `./kitty/launcher/kitty --version` |
| Read-only diff check | `git diff --name-status 815df1e21..HEAD` |
| Working-tree cleanliness | `git status --porcelain` |
| Agent commit count | `git log --author="agent@blitzy.com" --oneline \| wc -l` |
| Verify a citation | `sed -n '18p' kitty/vt-parser.c` |
| Locate I/O-loop thread | `for t in /proc/$(pgrep -x kitty)/task/*; do [ "$(cat $t/comm)" = KittyChildMon ] && basename $t; done` |

### B. Port Reference

Not applicable — the investigation is headless and defines no network services or listening ports.

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** — the answer document (1,108 lines). |
| `kitty/child-monitor.c` | Native I/O event loop; `POLLIN`/`POLLOUT` gating (`L1501`/`L1503`); `write_to_child` `EAGAIN` drain (`L1443-L1479`); 100 MiB cap + drop log (`L341-L344`). |
| `kitty/vt-parser.c` | 1 MiB `BUF_SZ` input buffer (`L18`); `vt_parser_has_space_for_input` (`L1477-L1481`); `input_delay` batching (`L1425`). |
| `kitty/screen.c` | Graphics dispatch `screen_handle_graphics_command` (`L1047-L1051`); `write_escape_code_to_child`. |
| `kitty/graphics.c` | 320 MiB quota (`L25`); `apply_storage_quota` eviction (`L290-L299`); `finish_command_response` (`L759-L782`); frame cache ×5 (`L1570-L1573`). |
| `kitty/disk-cache.c` | On-disk backing store behind the quota (`disk_cache_total_size`). |
| `kitty/options/definition.py` | `repaint_delay` (`L866`), `input_delay` (`L878`) timing defaults. |
| `kitty_tests/` | PTY-based harness pattern imitated by Path A probes. |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| kitty (built) | 0.35.2 |
| Python | 3.13.7 |
| gcc | 15.2.0 (Ubuntu 15.2.0-4ubuntu4) |
| Go | 1.22.12 |
| Pillow | 12.3.0 |
| pygments | 2.20.0 |
| Base commit | `815df1e21` |
| HEAD commit | `97d2a396` |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `PATH` | `$PATH:/usr/local/go/bin` | Make the Go toolchain discoverable for the kitten build. |
| `GOPATH` | `$HOME/go` | Go module/workspace path. |
| `LANG` / `LC_ALL` | `C.UTF-8` | Deterministic locale for the build and PTY I/O. |
| `LIBGL_ALWAYS_SOFTWARE` | `1` | Software GL for headless (Path B) runs under `xvfb-run`. |
| `CI` | `true` | Non-interactive mode for the in-process (Path A) harness. |
| `KITTY_PRINT_BYTES_SENT_TO_CHILD` | _(unused)_ | Compile-time write-dump flag noted in the AAP; **not used** — `strace` was used against the canonical binary instead. |

### F. Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `git` | Verify read-only constraint (`diff`/`status`/`log`), inspect the 6-commit refinement history. |
| `setup.py` | Canonical kitty build (`-O3 -DNDEBUG`). |
| `strace` | Observe `poll`/`read`/`write` syscalls on the `KittyChildMon` I/O thread (Path B). |
| `xvfb-run` | Run kitty headless (no GPU display) to drive real PTY children. |
| `kitty_tests` harness | In-process canonical-path exercise of the parser/screen/graphics chain (Path A). |
| `sed`/`grep` | Byte-accurate `file:line` citation verification against `815df1e21`. |

### G. Glossary

| Term | Definition |
|------|------------|
| **APC** | Application Program Command — the escape-sequence introducer (`ESC _ … ESC \`) that carries kitty graphics-protocol commands. |
| **`BUF_SZ`** | The fixed **1 MiB** (`1048576`-byte) VT-parser input buffer (`vt-parser.c:L18`). |
| **Backpressure** | Slowing/withholding data flow when a downstream buffer is full — here, de-arming `POLLIN` (input) and retaining bytes on `EAGAIN` (output). |
| **`EAGAIN`** | The `errno` returned by a non-blocking `write()` when the pipe is full; triggers retain-and-retry in `write_to_child`. |
| **`POLLIN` / `POLLOUT`** | `poll(2)` interest flags the I/O loop arms/de-arms to gate reading from / writing to the child. |
| **Storage quota** | kitty's **320 MiB** per-buffer image-store limit (`graphics.c:L25`); exceeding it evicts the oldest images. |
| **Quiet flag (`q`)** | Graphics-protocol control: `q=1` suppresses `;OK`, `q=2` suppresses errors too. |
| **Path A / Path B** | The two canonical observation methods: in-process `kitty_tests` harness / headless `xvfb` PTY + `strace`. |
| **Observed vs Inferred** | Observed = captured at runtime; Inferred = derived from code/config after genuine attempts, explicitly labeled. |

---

<p align="center"><i>Generated by the Blitzy Platform · Completion computed on AAP-scoped + path-to-production work only · 🟪 Completed <code>#5B39F3</code> · ⬜ Remaining <code>#FFFFFF</code></i></p>