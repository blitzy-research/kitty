# Blitzy Project Guide

## How kitty Regulates Graphics-Protocol Data Flow Under Pressure — Runtime-Verified Q&A Documentation

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers an authoritative, runtime-verified technical Q&A document explaining how the **kitty terminal emulator (v0.35.2)** regulates inbound and outbound flow of graphics-protocol (`APC _G`) data when it arrives faster than the terminal can comfortably process and respond to it. It is a **strictly read-only, code-comprehension investigation** — no kitty behavior is altered. The sole deliverable, `blitzy/documentation/kitty_815df1e210e0.md`, answers five sub-questions (buffering/pausing/throttling, responses under output congestion, where the decisions live, runtime manifestation, and silent-vs-visible adaptation) using verbatim observed runtime evidence paired with exact `file:line` citations. The audience is kitty maintainers and terminal-protocol engineers who need a grounded, reproducible explanation of the flow-control subsystem.

### 1.2 Completion Status

The project is **91.9% complete** on an AAP-scoped hours basis. All autonomous investigation and authoring work is delivered, validated, and committed; the remaining work is human-in-the-loop review and merge.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 91.9% Complete (28.5h of 31h)
    "Completed Work (AI)" : 28.5
    "Remaining Work" : 2.5
```

> Color key — **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`**.

| Metric | Value |
| --- | --- |
| **Total Hours** | **31.0 h** |
| **Completed Hours (AI + Manual)** | **28.5 h** (28.5 h AI · 0 h manual) |
| **Remaining Hours** | **2.5 h** |
| **Percent Complete** | **91.9 %** |

*Calculation (PA1, AAP-scoped): `28.5 / (28.5 + 2.5) = 28.5 / 31 = 91.9%`.*

### 1.3 Key Accomplishments

- ✅ **Sole in-scope deliverable authored and committed** — `blitzy/documentation/kitty_815df1e210e0.md` (644 lines), the `blitzy/documentation/` directory created as required.
- ✅ **All five AAP sub-questions answered** with cause→effect explanations and verbatim runtime evidence.
- ✅ **Read-only scope perfectly respected** — `git diff` shows exactly one file added (`+644/-0`); zero existing files touched; no dependency changes.
- ✅ **97 exact `file:line` citations**; a 28-citation independent audit found 100% accurate.
- ✅ **35/35 subject-matter unit tests pass** — graphics 19/19 and parser 16/16 (independently reproduced: `OK`, exit 0).
- ✅ **Runtime baseline verified** — `kitty 0.35.2 created by Kovid Goyal`; build present and runnable.
- ✅ **20 verbatim evidence blocks** captured across ≥2 runs and cross-checked in the mandated Docker image.
- ✅ **Exhaustive sibling coverage** — every limit constant, quiet level (`q=0/1/2`), and pending-mode termination cause enumerated in a coverage-pass table.
- ✅ **Exemplary evidence discipline** — claims labeled OBSERVED / INFERRED-SOURCE-CONFIRMED / NON-CANONICAL(none); two discrepancies self-reported and root-caused.
- ✅ **Temporary observation scripts removed** — working tree clean, repository left unchanged apart from the deliverable.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
| --- | --- | --- | --- |
| *None — no blocking issues.* | The deliverable is complete, validated, and committed. The only remaining work is standard human review (see §1.6 / §2.2). | — | — |

### 1.5 Access Issues

**No access issues identified.** The repository is present locally on branch `blitzy-3f8a176c-db57-4f04-8fc8-ae8ffbfd18e1` with a clean working tree, all three deliverable commits are authored by `agent@blitzy.com`, and the build toolchain (Python 3.13.7, Go 1.22.12, gcc 15.2.0) plus all native libraries are available for reproduction.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
| --- | --- | --- | --- | --- |
| Repository (`blitzy-3f8a176c…`) | Git read/write | None — clean tree, committed | ✅ Resolved | — |
| Build toolchain & native libs | Local execution | None — all present and verified | ✅ Resolved | — |
| Mandated Docker image | Container pull | Not required for this assessment; native build reused | ✅ N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform an SME technical review of `blitzy/documentation/kitty_815df1e210e0.md` — validate the flow-control narrative and confirm the two self-reported discrepancies and their source root-cause analyses. *(1.5 h)*
2. **[Medium]** Run a citation sampling audit — spot-check a representative subset of the 97 `file:line` references against kitty source at HEAD `815df1e210e0`. *(0.5 h)*
3. **[Low]** Approve the PR and merge, confirming the single-file read-only scope is preserved. *(0.5 h)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific Agent Action Plan requirement (the environment baseline, the five sub-questions, exhaustive coverage, and evidence discipline).

| Component | Hours | Description |
| --- | --- | --- |
| Environment, Build & Reproducibility baseline (AAP §0.1.3/§0.8.1 → doc §1) | 2.5 | Built/ran kitty in default canonical config; captured exact build commands (`make` → `python3 setup.py`) verbatim from `Makefile`; recorded version banner `kitty 0.35.2`; documented canonical code path, default option values, and the scale/repeatability table. |
| Sub-Q1 — Buffer / Pause / Throttle (AAP §0.1.1·1 → doc §2) | 4.0 | Investigated and authored the 1 MiB `BUF_SZ` ceiling (OBSERVED fill sequence to `0`), the three-disjunct coalescing/force-flush predicate, and `POLLIN` de-registration producing kernel PTY backpressure (OBSERVED child `write()` blocked at 12288 B). |
| Sub-Q2 — Responses Under Output Pressure (AAP §0.1.1·2 → doc §3) | 3.5 | `write_buf` queue + lock, `POLLOUT`-only-when-pending, `EAGAIN`/`EWOULDBLOCK` retry, and the 100 MB output drop cap with its verbatim `log_error` line, driven through the genuine `ChildMonitor`/PTY/`Screen` path across 2 runs + Docker. |
| Sub-Q3 — Code-Location Mapping (AAP §0.1.1·3 → doc §4) | 2.0 | Named every decision site with `file:line` precision across all 8 subsystems (parser, child-monitor, graphics, disk-cache, screen, loop-utils, graphics-command parser, options). |
| Sub-Q4 — Runtime Manifestation (AAP §0.1.1·4 → doc §5) | 6.0 | Drove each limit past its boundary with verbatim protocol responses: `MAX_DATA_SZ` 400 MB → `EFBIG`; PNG size → `EINVAL`; `MAX_IMAGE_DIMENSION` 10000 px → `EINVAL` (with honest state-dependent discrepancy and full source root-cause); 320 MiB storage-quota LRU eviction; disk-cache offload; real `icat` chunk framing. |
| Sub-Q5 — Silent vs. Visible + Pending Mode (AAP §0.1.1·5 → doc §6) | 4.0 | Silent-mechanism and visible-sign tables; DEC private mode 2026 (synchronized output) with all termination causes exhaustively (including the honest finding that "excess data" is not a separate counter); quiet levels `q=0/1/2`. |
| Exhaustive Sibling Coverage & Coverage Pass (AAP §0.7/§0.4.4 → doc §7.1) | 2.0 | Coverage-pass table enumerating every named mechanism, limit constant, quiet level, and termination cause mapped to the section that answers it. |
| Evidence Discipline & 97-Citation Audit + 2 QA Rounds (AAP §0.7/§0.8.1 → doc §7.3) | 4.0 | One-claim-one-evidence verbatim capture; OBSERVED/INFERRED/NON-CANONICAL labeling; full citation audit; two QA iteration rounds (6 findings addressed + a citation-precision fix). |
| Read-Only Scope, Cleanup & Deliverable Convention (AAP §0.3/§0.5 → doc §7.4) | 0.5 | Enforced read-only scope, removed temp scripts, created `blitzy/documentation/`, named the file `kitty_815df1e210e0.md`, and committed. |
| **Total Completed** | **28.5** | Matches Completed Hours in §1.2. |

### 2.2 Remaining Work Detail

Each remaining category is human-in-the-loop path-to-production work; Blitzy does not self-approve its own documentation as production-final.

| Category | Hours | Priority |
| --- | --- | --- |
| SME technical review of deliverable accuracy & citations (validate flow-control narrative; confirm the 2 self-reported discrepancies and their root-cause analyses) | 1.5 | High |
| Citation sampling audit — spot-check a representative subset of the 97 `file:line` refs vs. kitty source @HEAD `815df1e210e0` | 0.5 | Medium |
| Merge / PR approval & close-out (confirm single-file read-only scope preserved) | 0.5 | Low |
| **Total Remaining** | **2.5** | Matches Remaining Hours in §1.2 and the pie chart in §7. |

### 2.3 Hours Reconciliation

| Line | Hours |
| --- | --- |
| Section 2.1 — Completed Total | 28.5 |
| Section 2.2 — Remaining Total | 2.5 |
| **Total Project Hours (2.1 + 2.2)** | **31.0** |
| Percent Complete (`28.5 / 31`) | **91.9 %** |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation activity for this project (the kitty test harness executed via `./kitty/launcher/kitty +launch test.py --module <name>`), and were independently re-reproduced during this assessment. Because the deliverable is documentation, the relevant tests are the terminal's own subject-matter modules that exercise the documented flow-control mechanisms (graphics + VT parser). Line-coverage was not the validation metric for a read-only doc; the "Coverage" column reflects subject-matter coverage of the documented mechanisms.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| Unit — Graphics module | kitty test harness (`unittest`) | 19 | 19 | 0 | Subject-matter (graphics/quota/disk-cache/quiet) | `Ran 19 tests in 0.210s / OK`, exit 0. Includes `test_suppressing_gr_command_responses`, `test_graphics_quota_enforcement`, `test_disk_cache`, `test_load_png`. |
| Unit — VT Parser module | kitty test harness (`unittest`) | 16 | 16 | 0 | Subject-matter (APC `_G` parse, threading) | `Ran 16 tests in 0.058s / OK`, exit 0. Includes `test_graphics_command`, `test_parser_threading`. |
| Citation Audit (validation) | Source cross-reference | 97 refs (28 sampled) | 28 | 0 | 100% of sample accurate | Every sampled `file:line` matches source at HEAD `815df1e210e0`; corroborates the autonomous 100% citation audit. |
| Runtime Re-Verification (validation) | Observation harnesses (≥2 runs) | 20 evidence blocks | 20 | 0 | All headline claims | Every quoted value re-observed stable across ≥2 runs and cross-checked in the mandated Docker image. |
| **Total (unit)** | — | **35** | **35** | **0** | **100% pass** | Subject-matter suite for the documented subsystems. |

**Known non-blocking test note (out of scope, correctly no action):** `kitty_tests.fonts.Selection.test_font_selection` fails due to an "ubuntu mono" PostScript-name font-packaging artifact in the base image (it fails identically in the canonical Docker image). It is unrelated to the graphics flow-control subject and cannot be affected by the single in-scope documentation file.

---

## 4. Runtime Validation & UI Verification

kitty is a GPU-accelerated terminal emulator; it has **no web UI**. "Runtime validation" here means the compiled core builds, runs, and answers the graphics protocol on the real input path — which is exactly what the deliverable's evidence captures. "UI verification" is the terminal's on-the-wire protocol responses, quoted verbatim in the document.

**Runtime health:**

- ✅ **Operational** — `./kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal` (matches doc §1.2).
- ✅ **Operational** — build artifacts present and loadable: `kitty/launcher/kitty`, `kitty/launcher/kitten`, `kitty/fast_data_types.so`.
- ✅ **Operational** — `kitty.fast_data_types` imports (`Screen`, `ChildMonitor`); `kitty +launch <script>` executes in kitty's environment.

**Graphics protocol path (real `APC _G`, captured verbatim in the doc):**

- ✅ **Operational** — real `kitten icat` over a PTY: a real `Screen` answers the capability query; chunked `APC _G` framing observed (`m=1` non-final chunks, final chunk omits `m`).
- ✅ **Operational** — 100 MB output cap trips with the verbatim line `Too much data being sent to child with id: 1, ignoring it` (2 runs + Docker).
- ✅ **Operational** — graphics limits produce their protocol responses: `EFBIG:Too much data`, `EINVAL:PNG data size too large`, and (state-dependent) `EINVAL:Image too large`.
- ✅ **Operational** — 320 MiB storage quota performs **silent** LRU eviction (`non-OK responses during load: 0`); disk-cache offload moves bytes off-RAM (`size_on_disk … 4194304`).
- ✅ **Operational** — pending/synchronized mode (DEC 2026) status/toggle responses captured (`\x1b[?2026;1$y` / `;2$y`).

**Verification status indicators:**

- ⚠ **Partial (by design, honestly labeled)** — two behaviors are **INFERRED / SOURCE-CONFIRMED** rather than OBSERVED because they require kitty's live I/O `poll()` thread or GPU render loop (not present in a headless harness): `POLLIN`/`POLLOUT` interest gating and the 2000 ms pending-mode timeout expiry. Both are grounded in exact source literals.
- ❌ **Failing** — none within scope.

---

## 5. Compliance & Quality Review

Cross-mapping of the AAP mandates (SWE-AtlasQnA-Repo rules) to their compliance status, including fixes applied during autonomous validation.

| Benchmark / AAP Rule | Requirement | Status | Progress / Evidence |
| --- | --- | --- | --- |
| Deliverable location & form | One Markdown doc `<branch>.md` in `blitzy/documentation/` | ✅ Pass | `blitzy/documentation/kitty_815df1e210e0.md` created; directory created. |
| Investigate by running the code first | Build & run real paths; observe before writing | ✅ Pass | Build verified; 20 runtime evidence blocks captured. |
| Observe true magnitude/timing | Sufficient scale; stable across ≥2 runs | ✅ Pass | Scale/repeatability table in doc §1.5; all headline values stable ≥2 runs + Docker. |
| Exercise the exact code path | Real `icat` / raw `APC _G`, not bypass hooks | ✅ Pass | Genuine `Screen`/`vt_parser`/`GraphicsManager`; NON-CANONICAL: none. |
| Default, canonical configuration | Default `kitty.conf`; report version verbatim | ✅ Pass | No custom conf; `input_delay=3`, `repaint_delay=10`, `sync_to_monitor=yes` confirmed. |
| Quote observed output verbatim | Exact log lines, sizes, error strings | ✅ Pass | 20 verbatim `console` blocks; one claim, one evidence. |
| Answer every part & named item | Exhaustive over all siblings | ✅ Pass | Coverage-pass table (doc §7.1) enumerates every mechanism/limit/level/cause. |
| Be exact & grounded (`file:line`) | Cite exact literals with `file:line` | ✅ Pass | 97 citations; 28 independently audited → 100% accurate. |
| Provide reasoning | Rationale per answer | ✅ Pass | Doc §7.2 "why kitty is designed this way". |
| Read-only scope | No existing files modified; scripts removed | ✅ Pass | `git diff` = 1 file added; temp scripts removed; tree clean. |
| Evidence labeling | OBSERVED / INFERRED / NON-CANONICAL | ✅ Pass | Labels applied consistently; 2 discrepancies self-reported and root-caused. |

**Fixes applied during autonomous validation:** (1) six QA findings addressed on the flow-control deliverable (commit `3d4ae05dd`); (2) `docs/performance.rst` citation corrected from `L109-L110` to `L109-L111` so the quoted phrase is covered (commit `231f765a7`). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

The risk profile is uniformly **LOW** — the expected profile for a strictly read-only documentation deliverable that adds no code, dependencies, or runtime footprint.

| Risk | Category | Severity | Probability | Mitigation | Status |
| --- | --- | --- | --- | --- | --- |
| Documentation makes an inaccurate technical claim | Technical | Low | Low | 97 `file:line` citations (28 audited → 100% accurate); every headline claim OBSERVED across ≥2 runs + Docker; 35/35 subject-matter tests pass | Mitigated |
| INFERRED claims not runtime-observed (`POLLIN`/`POLLOUT` gating; 2000 ms pending timeout) | Technical | Low | Low | Explicitly labeled INFERRED / SOURCE-CONFIRMED; grounded in exact source literals; not headlessly observable without a GPU display | Accepted / Documented |
| Reviewer must confirm the 2 self-reported discrepancies | Technical | Low | Low | State-dependent `EINVAL` traced to `graphics.c` abort-ordering (L695/L717/L2177/L765); `icat` 131072-byte chunk vs. docs' conservative 4096 explained | Documented / Mitigated |
| Introduction of a security vulnerability | Security | None | None | Read-only Markdown adds no code, no dependencies, no attack surface; exposes no secrets | N/A |
| Impact on a running/deployed service | Operational | None | None | Static document; no deploy, runtime, or service footprint | N/A |
| Full runtime reproduction environment availability | Operational | Low | Low | Native build reused; mandated Docker image documented; only 2 INFERRED items need a live display | Documented |
| Broken external integration / credentials / network | Integration | None | None | No external services, APIs, credentials, or network config; standalone doc with no code links | N/A |

**No Critical or High risks identified.**

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 31h)
    "Completed Work" : 28.5
    "Remaining Work" : 2.5
```

> **Completed Work = Dark Blue `#5B39F3`** · **Remaining Work = White `#FFFFFF`** · Headings/accents `#B23AF2`.
> The "Remaining Work" value (**2.5 h**) equals Remaining Hours in §1.2 and the sum of the §2.2 Hours column.

**Remaining hours by priority (from §2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Priority (2.5h)
    "High — SME review" : 1.5
    "Medium — citation audit" : 0.5
    "Low — merge/close-out" : 0.5
```

| Category (from §2.2) | Hours | Priority |
| --- | --- | --- |
| SME technical review | 1.5 | High |
| Citation sampling audit | 0.5 | Medium |
| Merge / PR approval & close-out | 0.5 | Low |
| **Total** | **2.5** | — |

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, exhaustive, and rigorously evidenced technical Q&A document explaining kitty's graphics flow-control and backpressure behavior. All five AAP sub-questions are answered with cause→effect reasoning, 97 exact `file:line` citations, and 20 verbatim runtime-evidence blocks captured across at least two runs. The strictly read-only scope was honored perfectly (one file added, zero existing files touched), and the subject-matter test suites pass 35/35.

**Remaining gaps.** There are no technical gaps. The remaining **2.5 hours** are entirely human-in-the-loop: an SME technical review, a citation sampling audit, and merge approval. This is standard path-to-production for expert documentation — Blitzy does not self-certify its own documentation as production-final.

**Critical path to production.** SME technical review (1.5 h) → citation sampling audit (0.5 h) → merge/close-out (0.5 h).

**Success metrics.**

| Metric | Result |
| --- | --- |
| AAP sub-questions answered | 5 / 5 |
| Subject-matter unit tests | 35 / 35 pass |
| Citations audited accurate | 28 / 28 (of 97) |
| Existing files modified | 0 (read-only scope honored) |
| Evidence blocks (≥2 runs) | 20 |
| AAP-scoped completion | **91.9 %** |

**Production-readiness assessment.** The deliverable is **production-ready pending human sign-off**. At **91.9% complete**, everything within Blitzy's autonomous remit is finished and validated; the document is accurate, internally consistent, well-formed (54 balanced code fences, 7 resolvable TOC anchors), respects read-only scope, and is committed. The recommended action is a focused SME review followed by merge.

---

## 9. Development Guide

All commands below were executed and verified in the assessment environment. Run them from the repository root: `/tmp/blitzy/kitty/blitzy-3f8a176c-db57-4f04-8fc8-ae8ffbfd18e1_305011`.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu 25.10 in the container; the mandated image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` is authoritative).
- **Python:** ≥ 3.8 (verified `Python 3.13.7`) — runs kitty and its build.
- **Go toolchain:** 1.22 (verified `go1.22.12`) — builds the Go CLI/kittens including `icat`.
- **C compiler:** gcc/clang (verified `gcc 15.2.0`) — compiles the CPython C-extension.
- **Native libraries (all verified present via `pkg-config`):** fontconfig 2.15.0, freetype2 26.2.20, harfbuzz 10.2.0, libpng 1.6.50, lcms2 2.16, openssl/libcrypto 3.5.3, plus OpenGL + Wayland/X11 for GPU rendering at runtime.

### 9.2 Environment Setup

```bash
# UTF-8 locale (prevents encoding issues in the test harness / observation scripts)
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8

# Ensure world-writable /tmp for temporary observation scripts (they live OUTSIDE the repo)
chmod 1777 /tmp 2>/dev/null || true

# (Optional) project virtualenv, per the validation environment
# python3 -m venv /opt/kitty-venv && source /opt/kitty-venv/bin/activate
```

### 9.3 Build (default / canonical)

The canonical build is `make`, whose `all:` target runs `python3 setup.py` (verified verbatim in the root `Makefile`):

```bash
# Default canonical build (produces kitty/launcher/kitty, kitten, and fast_data_types.so)
make

# Optional tracing builds (NOT required — and NOT used as the source of any documented value)
make debug              # python3 setup.py build --debug
make debug-event-loop   # python3 setup.py build --debug --extra-logging=event-loop

# Other Makefile targets
make test               # python3 setup.py test
make clean              # python3 setup.py clean
```

> In this assessment the existing native build was reused (a documentation-only task requires no recompile); artifacts were already present and runnable.

### 9.4 Run & Verify the Baseline

```bash
# Version banner — expected: kitty 0.35.2 created by Kovid Goyal
./kitty/launcher/kitty --version

# Confirm the compiled core imports
./kitty/launcher/kitty +launch python -c "import kitty.fast_data_types as f; print(f.Screen, f.ChildMonitor)"
```

Expected version output:

```text
kitty 0.35.2 created by Kovid Goyal
```

### 9.5 Verification Steps (subject-matter tests)

```bash
# Graphics module — expected: Ran 19 tests ... OK  (exit 0)
./kitty/launcher/kitty +launch test.py --module graphics

# VT parser module — expected: Ran 16 tests ... OK  (exit 0)
./kitty/launcher/kitty +launch test.py --module parser
```

Confirm the read-only scope and authorship:

```bash
# Expected: a single line — A  blitzy/documentation/kitty_815df1e210e0.md
git diff --name-status origin/kitty_815df1e210e0...HEAD

# Expected: 3 commits, all authored by agent@blitzy.com
git log --author="agent@blitzy.com" origin/kitty_815df1e210e0..HEAD --oneline
```

### 9.6 Example Usage — Read & Reproduce the Deliverable

```bash
# Read the deliverable
sed -n '1,60p' blitzy/documentation/kitty_815df1e210e0.md

# Verify markdown integrity (even number of code fences => balanced)
grep -c '```' blitzy/documentation/kitty_815df1e210e0.md      # -> 54 (even)

# Spot-check a citation against source (example: the 1 MiB parse buffer)
sed -n '18p' kitty/vt-parser.c                                 # -> #define BUF_SZ (1024u*1024u)

# Drive the REAL graphics path (as the doc does) — temp scripts must live OUTSIDE the repo and be removed after:
#   kitty +kitten icat <image>          # real icat over a PTY
#   or raw chunked APC _G sequences fed into the real Screen/vt_parser via the test helpers
```

### 9.7 Troubleshooting

- **"No test named [...] found":** the runner selects modules with `--module <name>` (e.g., `--module graphics`), not a bare positional module name.
- **Encoding errors in harness output:** ensure `LANG`/`LC_ALL` are set to `en_US.UTF-8` (see §9.2).
- **Two behaviors can't be observed headlessly:** `POLLIN`/`POLLOUT` interest gating and the 2000 ms pending-mode timeout require kitty's live I/O `poll()` thread / GPU render loop; the doc labels these **INFERRED / SOURCE-CONFIRMED** and grounds them in exact source literals.
- **`kitty_tests.fonts` font test fails:** this is a base-image font-packaging artifact ("ubuntu mono" PS name) that also fails in the canonical Docker image; it is unrelated to graphics flow-control and requires no action.
- **Keep the repo unchanged:** place any observation scripts under `/tmp` (outside the repo) and remove them afterward; verify with `git status` (expected: clean tree).

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
| --- | --- |
| Default build | `make` |
| Debug build | `make debug` |
| Event-loop tracing build | `make debug-event-loop` |
| Version banner | `./kitty/launcher/kitty --version` |
| Run graphics tests | `./kitty/launcher/kitty +launch test.py --module graphics` |
| Run parser tests | `./kitty/launcher/kitty +launch test.py --module parser` |
| Read-only scope check | `git diff --name-status origin/kitty_815df1e210e0...HEAD` |
| Authorship check | `git log --author="agent@blitzy.com" origin/kitty_815df1e210e0..HEAD --oneline` |
| Fence-balance check | `grep -c '```' blitzy/documentation/kitty_815df1e210e0.md` |

### Appendix B — Port Reference

**Not applicable.** kitty is a terminal emulator; the flow-control subject uses a local PTY and the graphics `APC _G` escape-sequence path — no network ports are opened or required.

### Appendix C — Key File Locations

| File | Role |
| --- | --- |
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (Q&A document) |
| `kitty/vt-parser.c` | 1 MiB `BUF_SZ`, coalescing predicate, `vt_parser_has_space_for_input`, DCS pending toggle |
| `kitty/child-monitor.c` | Threaded `poll()` loop, `POLLIN`/`POLLOUT` gating, `EAGAIN` retry, 100 MB output cap |
| `kitty/graphics.c` | 320 MiB storage quota + LRU, `MAX_DATA_SZ` (EFBIG), `MAX_IMAGE_DIMENSION` (EINVAL), quiet handling |
| `kitty/disk-cache.c` | Off-RAM image byte store (add/read/remove/defrag) |
| `kitty/screen.c` | `write_buf` + lock, `screen_pause_rendering` timeout, `PENDING_MODE` (2026) |
| `kitty/loop-utils.c` | `eventfd`/self-pipe cross-thread wakeup |
| `kitty/parse-graphics-command.h` | APC `_G` command parsing (`m=`, `q=` keys) |
| `kitty/options/definition.py` | Defaults: `input_delay=3`, `repaint_delay=10`, `sync_to_monitor=yes` |
| `docs/graphics-protocol.rst`, `docs/performance.rst` | Wire-format & synchronized-update references |

### Appendix D — Technology Versions

| Component | Version |
| --- | --- |
| kitty | 0.35.2 (HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`) |
| Python | 3.13.7 |
| Go | 1.22.12 |
| gcc | 15.2.0 |
| fontconfig / freetype2 / harfbuzz | 2.15.0 / 26.2.20 / 10.2.0 |
| libpng / lcms2 / openssl | 1.6.50 / 2.16 / 3.5.3 |

### Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
| --- | --- | --- |
| `LANG` | `en_US.UTF-8` | UTF-8 locale for harness/observation output |
| `LC_ALL` | `en_US.UTF-8` | Overrides locale categories to UTF-8 |
| `CI` | `true` (optional) | Non-interactive test runs |

> No custom `kitty.conf` is used — defaults are in force (canonical configuration), which is required by the AAP.

### Appendix F — Developer Tools Guide

| Tool | Usage |
| --- | --- |
| `kitty +launch <script.py>` | Run Python inside kitty's environment with the compiled `fast_data_types` module available |
| `test.py --module <name>` | Select a test module (e.g., `graphics`, `parser`) |
| `kitty +kitten icat <image>` | Drive the real graphics input path (`APC _G`) |
| `make debug-event-loop` | Optional event-loop tracing build (`--extra-logging=event-loop`) |
| `KITTY_PRINT_BYTES_SENT_TO_CHILD` | Compile-time instrumentation hook (any value it produces must be labeled INSTRUMENTATION; not used for documented values) |

### Appendix G — Glossary

| Term | Meaning |
| --- | --- |
| **APC `_G`** | Application Programming Command escape sequence carrying kitty graphics-protocol commands |
| **`BUF_SZ`** | The fixed 1 MiB (`1024×1024`) VT parse buffer per child |
| **Backpressure** | Slowing a fast producer by not draining its output — here, withholding `POLLIN` so the PTY fills and the child's `write()` blocks |
| **`POLLIN` / `POLLOUT`** | `poll()` interest flags for readable / writable file descriptors |
| **`EAGAIN` / `EWOULDBLOCK`** | Non-blocking I/O "try again later" errno on a would-block write |
| **`EFBIG` / `EINVAL` / `ENODATA`** | Graphics-protocol error responses (too much data / invalid / insufficient data) |
| **LRU eviction** | Least-Recently-Used removal of old images when the 320 MiB storage quota is exceeded |
| **DEC private mode 2026** | Synchronized Output (pending mode) — `CSI ?2026h`/`l` for atomic, tearing-free updates |
| **PTY** | Pseudo-terminal; the kernel buffer between kitty and the child process |
| **OBSERVED / INFERRED-SOURCE-CONFIRMED / NON-CANONICAL** | Evidence labels: runtime-captured / read-from-source-not-headlessly-observable / obtained via a bypassing path (none used) |

---

*Cross-section integrity verified: Remaining hours (2.5 h) are identical across §1.2, §2.2, and §7; §2.1 (28.5 h) + §2.2 (2.5 h) = 31 h Total; all test data originates from Blitzy's autonomous validation logs; brand colors applied (Completed `#5B39F3`, Remaining `#FFFFFF`).*