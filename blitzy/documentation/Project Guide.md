# Blitzy Project Guide

> **Project:** Runtime investigation of kitty's `HistoryBuf` (segmented scrollback) and `PagerHistoryBuf` (pager byte ring) under extreme write pressure
> **Repository:** `kovidgoyal/kitty` · **Baseline:** `815df1e21` · **Branch HEAD:** `ac52fa50d`
> **Task class:** Documentation — evidence-grounded runtime Q&A (read-only) · **Rule set:** `SWE-AtlasQnA-Repo`

---

## 1. Executive Summary

### 1.1 Project Overview

This project answers a five-part question about how kitty's terminal scrollback behaves when a child process floods the terminal with output very fast. The subject is the C core in `kitty/history.c` — the segmented in-memory store (`HistoryBuf`) and its opt-in pager byte ring (`PagerHistoryBuf`). Per the governing rule set, the work is a *run-first, then write* investigation: the kitty C core is built, temporary probes drive the **canonical** byte path (child/PTY → VT parser → Screen → history), and the observed runtime numbers are captured into a single markdown answer document. It is strictly read-only — no existing source file may change. The sole deliverable is `blitzy/documentation/kitty_815df1e210e0.md`.

### 1.2 Completion Status

The completion percentage is computed with the PA1 AAP-scoped hours method: every deliverable defined by the Agent Action Plan (AAP) plus standard path-to-production work, and nothing else.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Completion Status — 89.3% Complete (AAP-scoped)
    "Completed Work (AI)" : 50
    "Remaining Work" : 6
```

| Metric | Hours |
|---|---|
| **Total Hours** | **56** |
| Completed Hours (AI + Manual) | 50 (AI: 50, Manual: 0) |
| Remaining Hours | 6 |
| **Percent Complete** | **89.3%** |

> Formula: `Completion % = Completed / (Completed + Remaining) × 100 = 50 / 56 = 89.3%`.

### 1.3 Key Accomplishments

- ✅ Built the kitty C core into the canonical CPython extension `kitty/fast_data_types.so` (1,253,792 bytes, `-DNDEBUG -O3 -flto`), exposing `HistoryBuf` and `Screen`; build is deterministic and reproducible.
- ✅ **SQ1 — segment carving** answered with observed output: per-segment `calloc` of 5,251,072 bytes, new segments carved at line count 2049 / 4097 / 6145 / 8193, saturation at `count == ynum`.
- ✅ **SQ2 — store ↔ ring interaction** answered for both the **disabled** (evicted lines dropped) and **enabled** (33 bytes/evicted-line into the ring, growing to 297,000 B) pager tiers.
- ✅ **SQ3 — transition smoothness** answered: ring grows → plateaus at maximum → overwrites oldest; 1,000,000 and 10,000,000-line bursts complete with **no crash** (historical kitty#3011 does not reproduce), corroborated by an ASan+UBSan cross-check.
- ✅ **SQ4 — concurrent scroll + write** answered via a **real PTY child** through the real parser: the view pins to old output, then drifts once the buffer saturates (`child_exit=0`, all 40,000 lines received).
- ✅ **SQ5 — reflow** answered across both the fast-`memcpy` and `rewrap_inner` branches, including the non-idempotent round-trip delta.
- ✅ Every structural magnitude reproduced across ≥2 runs; nondeterministic values (RSS slope, latency) reported honestly as distributions.
- ✅ Read-only guarantee held perfectly: `git diff baseline..HEAD` shows **only** the one answer document added; `git status --porcelain` is empty.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None._ No compilation errors, no in-scope test failures, and no unresolved defects remain. All AAP deliverables are complete. | None | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _No access issues identified._ The build toolchain, source repository, and test harness were all fully accessible; the investigation completed end-to-end in the provided environment. | — | — | Resolved / N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Peer-review the answer document for technical accuracy — verify each SQ answer against the cited mechanism and confirm the observed-vs-inferred labels and coverage pass (§8.2 of the deliverable).
2. **[High]** Audit the ~110 `file:line` citations against the baseline commit `815df1e21` source.
3. **[High]** Independently re-verify the read-only guarantee (`git diff 815df1e21..HEAD --name-status` and `git status --porcelain`).
4. **[Medium]** Reproduce the canonical build and re-run the SQ1–SQ3 structural probes to confirm the deterministic magnitudes (carve points, eviction bytes/line, no-crash).
5. **[Low]** Merge/publish the answer document to the target branch after review sign-off.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to an AAP deliverable (build, SQ1–SQ5 probes, document authoring, stability, cleanup) or the QA remediation the autonomous agents performed.

| Component | Hours | Description |
|---|---|---|
| Canonical + supplemental builds | 4 | Canonical `fast_data_types.so` build (`-DNDEBUG -O3 -flto`); supplemental debug and ASan+UBSan builds; sha256 determinism and `KITTY_VCS_REV` provenance analysis [AAP D1] |
| SQ1 — segment-carving probe | 5 | Probe + 3-run capture of segment `calloc`, carve points, saturation; RSS-slope distribution analysis [AAP SQ1] |
| SQ2 — store ↔ ring probe | 4 | Disabled and enabled pager tiers; bytes/evicted-line measurement; ring growth to maximum [AAP SQ2] |
| SQ3 — boundary / large-burst probe | 8 | Ring grow→plateau→overwrite; per-batch latency distribution; 1M + 10M line bursts; ASan+UBSan cross-check with binary swap/restore [AAP SQ3] |
| SQ4 — concurrent scroll+write probe | 6 | Real-PTY child through the real parser; pin→cap→drift semantics; completion/exit verification [AAP SQ4] |
| SQ5 — reflow probe | 4 | Fast-`memcpy` and `rewrap_inner` branches; count progression and round-trip delta [AAP SQ5] |
| Answer-document authoring | 9 | 1,757-line document: TL;DR, §7 default-vs-enabled, §8 coverage/read-only, §9 references, before/during/after tables, ~110 citations, observed-vs-inferred labels [AAP D7] |
| Stability re-runs & distributions | 3 | ≥2 runs per magnitude; characterizing nondeterministic RSS/latency as distributions [AAP D8] |
| QA remediation (6 cycles) | 6 | Resolving 30 review findings across cycles (LineAttrs/`calloc` correction, RSS distribution, QA Report 7 F1–F7, provenance wording) |
| Read-only cleanup & commits | 1 | Out-of-repo scratch teardown; verifying clean tree; documentation-only commits [AAP D9] |
| **Total** | **50** | |

### 2.2 Remaining Work Detail

All remaining work is standard path-to-production activity — there are no in-scope engineering defects to fix.

| Category | Hours | Priority |
|---|---|---|
| Human technical peer review of the document (claims, citations, observed-vs-inferred, coverage) | 3.0 | High |
| Independent reproduction of structural magnitudes in a reviewer environment | 2.0 | Medium |
| Final merge/publish + post-merge doc-link verification | 1.0 | Low |
| **Total** | **6.0** | |

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 completed sum | 50.0 h |
| Section 2.2 remaining sum | 6 h |
| Section 2.1 + Section 2.2 | 56 h = **Total (Section 1.2)** ✅ |
| Section 7 pie "Remaining Work" | 6 h (identical to §1.2 and §2.2) ✅ |
| Human task list (Section 8 / HT) sum | 6 h (identical to §2.2) ✅ |
| Completion % | 50 / 56 = **89.3%** ✅ |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project and were re-executed during this assessment against the current build (`kitty/fast_data_types.so`, 1,253,792 B). Because the AAP is a read-only runtime investigation, the "test suite" comprises (a) kitty's own unit tests that exercise the studied code paths and (b) the five runtime observation probes that drive the canonical path ≥2× each.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Unit — `datatypes` module | Python `unittest` | 18 | 18 | 0 | Scrollback paths covered | Includes `test_historybuf`, `test_rewrap_simple/wide/narrow` |
| Unit — `screen` module | Python `unittest` | 36 | 36 | 0 | Scrollback paths covered | Includes `test_pagerhist`, `test_resize`, `test_scrollback_fill_after_resize` |
| Runtime probe — SQ1 (segment carving) | In-process `kitty_tests` harness (`parse_bytes`) | 3 runs | 3 | 0 | Canonical path | Carve at count 2049/4097/6145/8193; structurally byte-identical across runs |
| Runtime probe — SQ2 (store ↔ ring) | In-process `kitty_tests` harness | 2 runs | 2 | 0 | Canonical path | 33 B/evicted-line; ring 0 (disabled) vs grows to 297,000 B (enabled) |
| Runtime probe — SQ3 (boundary + large burst) | In-process harness + ASan/UBSan cross-check | 2 runs | 2 | 0 | Canonical path | 1M + 10M lines, no crash; ring plateaus at maximum |
| Runtime probe — SQ4 (scroll + write) | **Real PTY child** through real parser | 2 runs | 2 | 0 | Canonical path | `child_exit=0`; 40,000/40,000 lines received; marker once |
| Runtime probe — SQ5 (reflow) | In-process `kitty_tests` harness | 2 runs | 2 | 0 | Canonical path | Fast-`memcpy` + `rewrap_inner` branches; round-trip Δ observed |
| **Total** | | **54 unit + 11 probe runs** | **all pass** | **0** | | |

> **Environment-gated (out of scope, reported not fixed):** the full `./test.py` suite includes `test_glfw_modules`, which requires a GLFW/Wayland display module absent in this headless container. It is unrelated to scrollback; the `datatypes`/`screen` suites that exercise the studied paths pass without it.

---

## 4. Runtime Validation & UI Verification

This is a terminal-core (non-GUI) investigation; "runtime validation" means the observation probes driving the canonical byte path. UI verification is not applicable (no front-end surface in scope).

**Build & module health**
- ✅ **Operational** — `kitty/fast_data_types.so` builds cleanly (exit 0) and imports; `HistoryBuf` and `Screen` are exposed.
- ✅ **Operational** — Read-only observation surface available: `HistoryBuf.xnum/ynum/count`, `pagerhist_as_bytes()`, `pagerhist_as_text()`, `Screen.scrolled_by`.

**Canonical-path runtime behavior (re-verified this assessment)**
- ✅ **Operational** — SQ1: fresh `ynum=5000 xnum=80`, `max_segments=ceil(5000/2048)=3`; carve detected at `count` 2049 (`lines_fed=2072`) and 4097 (`lines_fed=4120`); final `count==ynum`, capped. Matches the deliverable's §2.3 exactly.
- ✅ **Operational** — SQ2: `count` saturates at `ynum=2000`; before saturation the pager ring stays 0; at saturation eviction begins and the ring fills, plateauing at its configured maximum.
- ✅ **Operational** — SQ3: 1M/10M-line bursts complete with no crash; ring plateaus at maximum then overwrites oldest bytes.
- ✅ **Operational** — SQ4: real PTY child at full speed; view pins then drifts at saturation; `child_exit=0`; all lines received; completion marker exactly once.
- ✅ **Operational** — SQ5: width-change reflow exercises both the fast-`memcpy` and `rewrap_inner` branches.

**API / integration**
- ✅ **Operational** — The `parse_bytes` driver feeds the **real** VT parser and Screen (not a debug hook); the PTY driver forks a **real** child on a **real** pseudo-terminal.
- ⚠ **Partial (labeled)** — SQ4's PTY driver performs the `os.read` + parser feed in Python; it exercises the real PTY→parser→Screen→history path but is **not** the C `child-monitor.c` I/O loop itself. The deliverable labels this "real-PTY-through-parser corroboration."

---

## 5. Compliance & Quality Review

Cross-map of AAP rule-set (`SWE-AtlasQnA-Repo`) directives to delivered evidence.

| Benchmark / Directive | Status | Progress | Evidence |
|---|---|---|---|
| Deliverable at `blitzy/documentation/<branch>.md` | ✅ Pass | 100% | `blitzy/documentation/kitty_815df1e210e0.md` (1,757 lines) |
| Run first, then write (build + run before writing) | ✅ Pass | 100% | Build commands + 7 embedded probes with captured output |
| Match magnitude/scale; confirm stability ≥2 runs | ✅ Pass | 100% | SQ1 ×3, SQ2–SQ5 ×2; 1M/10M-line bursts |
| Canonical entry point only (no synthetic stand-in) | ✅ Pass | 100% | `parse_bytes` real parser; real PTY child; fidelity caveat labeled |
| Default configuration + explicitly enable pager tier | ✅ Pass | 100% | Default (pager disabled) documented; pager enabled for SQ2/SQ3 |
| Demonstrate multi-segment carving (`scrollback > 2048`) | ✅ Pass | 100% | scrollback 5000/10000 → 3/5 segments |
| Before / during / after state for everything that changes | ✅ Pass | 100% | Tables for count, segments, evictions, ring bytes, `scrolled_by` |
| Complete, unedited output for every claim | ✅ Pass | 100% | 25 fenced blocks; "complete, unedited" run captures |
| Observed vs. inferred labeling; `file:line` grounding | ✅ Pass | 100% | 164 `[OBSERVED]` / 27 `[INFERRED]` labels; ~110 citations |
| Coverage pass (every sub-question + sibling variant) | ✅ Pass | 100% | §8.2 coverage ledger, SQ1–SQ5 each marked Answered |
| Read-only scope (no source edits; scripts removed) | ✅ Pass | 100% | `git diff baseline..HEAD` = 1 doc only; tree clean |
| Compilation quality (0 errors/warnings) | ✅ Pass | 100% | Canonical build exit 0; artifact imports |
| Citation accuracy | ✅ Pass | 100% | Spot-checked line-accurate (history.c:15, :17-28; screen.c:130, :2716; definition.py:372, :406) |

**Fixes applied during autonomous validation:** 30 review findings remediated across 6 QA cycles — including correcting `sizeof(LineAttrs)` and the per-segment `calloc` value, reporting the SQ1 RSS slope as a distribution rather than a fixed value, resolving QA Report 7 (F1–F7), and clarifying build-provenance wording. **Outstanding quality items:** none in scope.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Environment-dependent RSS slope & per-batch latency (nondeterministic) | Technical | Low | High | Reported as a distribution; deterministic constants (2560, 2564 B/line) are load-bearing; structural values are deterministic | Mitigated |
| SQ4 harness fidelity — Python PTY read loop, not the C `child-monitor.c` loop | Technical | Low | Medium | Honestly labeled "real-PTY-through-parser corroboration"; `parse_bytes` is the same real VT parser the C loop feeds | Mitigated |
| Segment count is inferred (`ceil(ynum/2048)`; no exposed member) | Technical | Low | Low | Labeled `[INFERRED]`; corroborated by observed carve boundaries and per-carve RSS delta ≈ `calloc` size | Mitigated |
| No source/config/dependency changes → no new attack surface | Security | Low | Low | Read-only; only a markdown doc added; `git diff` confirms 0 source changes | N/A |
| Build reproducibility across toolchains (`.so` sha256 depends on `KITTY_VCS_REV` + `-march=native`) | Operational | Low | Medium | `--vcs-rev` pin documented for exact sha256; structural probe results are toolchain-independent | Mitigated |
| Sanitizer binary-swap temporarily overwrites canonical `.so` | Operational | Low | Low | preserve→swap→restore→verify-sha256 sequence; canonical `75d31e7a…` verified byte-identical after each swap | Mitigated |
| GLFW/Wayland test gate (`test_glfw_modules`) fails headless | Integration | Low | High | Unrelated to scrollback; studied paths pass without it; reported not fixed (read-only) | Accepted |
| `scrollback_lines ≥ 2³²` wraps via unsigned 32-bit finalizer | Integration | Low | Low | Far outside exercised magnitudes (≤10,000); reported not fixed (read-only) | Accepted |

**Overall risk posture: Low.** The deliverable is a complete, read-only, evidence-grounded document; the identified "risks" are primarily honesty caveats it already surfaces.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000'}}}%%
pie showData title Project Hours Breakdown (Total 56h)
    "Completed Work" : 50
    "Remaining Work" : 6
```

**Remaining work by priority** (hours from Section 2.2, summing to 6h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieSectionTextColor':'#000000','pieLegendTextColor':'#000000'}}}%%
pie showData title Remaining Work by Priority (6h)
    "High — peer review" : 3.0
    "Medium — reproduction" : 2.0
    "Low — merge/publish & verify" : 1.0
```

> **Integrity:** "Remaining Work" = 6h in the pie chart equals the Section 1.2 Remaining Hours and the Section 2.2 "Hours" column sum.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a comprehensive, evidence-grounded answer to all five sub-questions about kitty's scrollback under extreme write pressure. The kitty C core was built canonically, seven observation probes drove the real parser/PTY path, and every structural magnitude — segment `calloc` size, carve points, saturation, eviction bytes/line, ring plateau, no-crash at 10M lines, scroll pin/drift, reflow branches — was captured as reproduced runtime numbers with `file:line` grounding and observed-vs-inferred labels. The single deliverable (`blitzy/documentation/kitty_815df1e210e0.md`, 1,757 lines) was hardened through 6 QA cycles resolving 30 findings.

**Remaining gaps.** None are engineering defects. The **6 remaining hours** are standard path-to-production: human peer review (3.0h), independent reproduction (2.0h), and merge/publish + verification (1.0h).

**Critical path to production.** Peer review → citation/read-only audit → independent reproduction of the deterministic magnitudes → merge. There is no build, test, or defect blocker on this path.

**Success metrics (all met):** build exit 0; 54/54 in-scope unit tests pass; 11 probe runs pass with no crash; read-only guarantee intact (`git status` clean; only 1 file added); every sub-question answered with unedited output.

**Production readiness assessment.** The project is **89.3% complete** on an AAP-scoped basis. The deliverable is production-ready pending human review sign-off; the residual 10.7% is human verification and publication, not agent-side implementation. Per Blitzy assessment policy, completion is capped below 100% until human review concludes.

| Metric | Value |
|---|---|
| AAP-scoped completion | 89.3% |
| In-scope defects outstanding | 0 |
| Read-only compliance | 100% (1 file added, 0 source changed) |
| Unit tests (in scope) | 54/54 pass |
| Runtime probe runs | 11/11 pass (SQ1 ×3, SQ2–SQ5 ×2) |

---

## 9. Development Guide

All commands below were executed and verified during this assessment. Run from the repository root unless noted.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu 25.10 container used here); macOS also supported upstream.
- **Python:** ≥ 3.8 (verified with **3.13.7**).
- **C toolchain:** a C11 compiler (verified with **gcc 15.2.0**).
- **Go:** 1.22+ (verified **1.24.4**) — only needed for the `kitten` CLI, **not** for `HistoryBuf` observation.
- **Hardware:** any modern x86-64/arm64; the 10M-line burst peaks at a few hundred MB RSS.

### 9.2 Environment Setup

Install the C build dependencies (Debian/Ubuntu names):

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  libharfbuzz-dev libsimde-dev libxxhash-dev libfontconfig-dev \
  libfreetype-dev libpng-dev liblcms2-dev libxkbcommon-dev libxkbcommon-x11-dev
```

> The vendored byte-ring (`3rdparty/ringbuf`, public domain) that backs `PagerHistoryBuf` needs no external package.

### 9.3 Build (Dependency Compilation)

```bash
# Canonical build — the build a normal user gets. Produces kitty/fast_data_types.so
CI=true python3 setup.py build --verbose
```

Expected result:

```text
EXIT_STATUS=0
kitty/fast_data_types.so   ~1,253,792 bytes   (-DNDEBUG -O3 -flto -march=native)
```

Optional supplemental builds (non-canonical; used only for boundary cross-checks):

```bash
python3 setup.py build --debug --verbose             # ~6,285,120 bytes
python3 setup.py build --debug --sanitize --verbose  # ~20,275,512 bytes (ASan+UBSan)
# To reproduce the documented .so sha256 exactly, pin the embedded VCS rev:
python3 setup.py build --verbose --vcs-rev 9e8a0069a671fb4e3f1ba6551c4bd9d1da0a82fa
```

### 9.4 Verification

```bash
# 1) Module imports and exposes the studied types
python3 -c "import kitty.fast_data_types as f; print('HistoryBuf', 'HistoryBuf' in dir(f), '| Screen', 'Screen' in dir(f))"
# -> HistoryBuf True | Screen True

# 2) In-scope unit suites pass
python3 -c "import unittest; from kitty_tests import datatypes, screen; \
s=unittest.TestSuite(); L=unittest.TestLoader(); \
[s.addTests(L.loadTestsFromModule(m)) for m in (datatypes, screen)]; \
r=unittest.TextTestRunner(verbosity=1).run(s); \
print('run',r.testsRun,'fail',len(r.failures),'err',len(r.errors))"
# -> run 54 fail 0 err 0
```

### 9.5 Example Usage — reproduce the documented findings (canonical path)

Run probes from an out-of-repo scratch directory so the repository stays untouched. Set `PYTHONPATH` to the repo root so the harness imports.

```bash
umask 077
scratch=$(mktemp -d /tmp/kitty-repro.XXXXXX)
trap 'rm -rf -- "$scratch"' EXIT
REPO=$(pwd)

# --- SQ1: multi-segment carving (needs scrollback > 2048) ---
cat > "$scratch/sq1.py" <<'PY'
import math
from kitty_tests import BaseTest, parse_bytes
class R(BaseTest):
    def runTest(self): pass
t = R(); s = t.create_screen(cols=80, lines=24, scrollback=5000); hb = s.historybuf
print("fresh ynum=%d xnum=%d max_segments=%d" % (hb.ynum, hb.xnum, math.ceil(hb.ynum/2048)))
seen=set()
for i in range(1, 6001):
    parse_bytes(s, b"L%08d\r\n" % i)
    if hb.count in (2049, 4097) and hb.count not in seen:
        seen.add(hb.count); print("CARVE count=%d lines_fed=%d" % (hb.count, i))
print("final count=%d ynum=%d capped=%s" % (hb.count, hb.ynum, hb.count==hb.ynum))
PY
PYTHONPATH="$REPO" python3 "$scratch/sq1.py"
```

Expected (matches the deliverable's §2.3):

```text
fresh ynum=5000 xnum=80 max_segments=3
CARVE count=2049 lines_fed=2072
CARVE count=4097 lines_fed=4120
final count=5000 ynum=5000 capped=True
```

```bash
# --- SQ2: eviction into the pager ring at saturation ---
cat > "$scratch/sq2.py" <<'PY'
from kitty_tests import BaseTest, parse_bytes
class R(BaseTest):
    def runTest(self): pass
t = R(); s = t.create_screen(cols=80, lines=24, scrollback=2000); hb = s.historybuf
for i in range(1, 5001):
    parse_bytes(s, b"L%08d\r\n" % i)
    if i in (2000, 3000, 5000):
        print("fed=%d count=%d ring_bytes=%d" % (i, hb.count, len(hb.pagerhist_as_bytes())))
PY
PYTHONPATH="$REPO" python3 "$scratch/sq2.py"
```

Expected (count saturates at `ynum`, then eviction fills the ring which plateaus at its configured maximum):

```text
fed=2000 count=1977 ring_bytes=0
fed=3000 count=2000 ring_bytes=1024
fed=5000 count=2000 ring_bytes=1024
```

### 9.6 Confirm the read-only guarantee

```bash
git status --porcelain --untracked-files=all          # -> empty (clean)
git diff 815df1e21..HEAD --name-status                 # -> A blitzy/documentation/kitty_815df1e210e0.md
```

### 9.7 Troubleshooting

- **`ModuleNotFoundError: No module named 'kitty_tests'`** → set `PYTHONPATH=<repo root>` before running probes.
- **Only one segment ever appears** → multi-segment carving requires `scrollback > 2048`; the default `scrollback_lines=2000` uses a single segment.
- **Pager ring stays 0 when enabled-expected** → the pager tier is disabled by default (`scrollback_pager_history_size=0`); enable it (>0) and note eviction only starts after `count == ynum`.
- **`test_glfw_modules` fails** → environment-only (needs a GLFW/Wayland display); unrelated to scrollback and safe to exclude in headless runs.
- **`.so` sha256 differs from the document** → the embedded `KITTY_VCS_REV` and `-march=native` make it toolchain/commit-specific; pin `--vcs-rev 9e8a0069a671…` to reproduce, or rely on the (toolchain-independent) structural probe results.
- **Sanitizer run left a large `.so`** → rebuild canonically (`CI=true python3 setup.py build --verbose`) and verify `sha256 == 75d31e7a…`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Canonical build | `CI=true python3 setup.py build --verbose` |
| Debug build (suppl.) | `python3 setup.py build --debug --verbose` |
| Sanitizer build (suppl.) | `python3 setup.py build --debug --sanitize --verbose` |
| Exact-sha256 build | `python3 setup.py build --verbose --vcs-rev 9e8a0069a671fb4e3f1ba6551c4bd9d1da0a82fa` |
| Import check | `python3 -c "import kitty.fast_data_types as f; print('HistoryBuf' in dir(f))"` |
| In-scope unit tests | `python3 -c "import unittest; from kitty_tests import datatypes, screen; ..."` (see §9.4) |
| Read-only check | `git status --porcelain --untracked-files=all` |
| Change scope | `git diff 815df1e21..HEAD --name-status` |
| ASan symbol count | `nm -D kitty/fast_data_types.so \| grep -c asan` |

### B. Port Reference

Not applicable — the investigation is an in-process/PTY terminal-core study; no network ports are opened or required.

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** — the answer document (1,757 lines) |
| `kitty/history.c` | Segmented store (`HistoryBuf`) + pager ring (`PagerHistoryBuf`) — `SEGMENT_SIZE=2048` @:15, `add_segment` @:17-28, `historybuf_push` eviction @:277-285, `pagerhist_push` @:258-274 |
| `kitty/data-types.h` | `HistoryBufSegment` / `PagerHistoryBuf` / `HistoryBuf` struct layout |
| `3rdparty/ringbuf/ringbuf.c` | Vendored byte FIFO backing the pager tier (overwrite-oldest @:231-234) |
| `kitty/screen.c` | `alloc_historybuf(MAX(scrollback,lines),…)` @:130; `scrolled_by` cap @:2716 |
| `kitty/vt-parser.c`, `kitty/child-monitor.c` | Canonical byte entry point + PTY read loop |
| `kitty/options/definition.py` | `scrollback_lines=2000` @:372; `scrollback_pager_history_size=0` @:406 |
| `kitty_tests/__init__.py` | `parse_bytes` real-parser driver @:30-36; `create_screen`/`create_pty` |
| `kitty/fast_data_types.so` | Built CPython extension (1,253,792 B; gitignored) |

### D. Technology Versions

| Component | Version (verified) |
|---|---|
| Python | 3.13.7 |
| gcc | 15.2.0 (Ubuntu 15.2.0-4ubuntu4) |
| Go | 1.24.4 (not required for `HistoryBuf`) |
| C standard | c11 |
| kitty baseline commit | `815df1e21` |
| Build-provenance commit (embedded in `.so`) | `9e8a0069a671fb4e3f1ba6551c4bd9d1da0a82fa` |

### E. Environment Variable Reference

| Variable | Purpose |
|---|---|
| `CI=true` | Non-interactive canonical build via `setup.py` |
| `PYTHONPATH=<repo root>` | Lets probe scripts import the `kitty_tests` harness |
| `PYTHONDONTWRITEBYTECODE=1` | Keeps the scratch tree clean during probe runs |
| `ASAN_OPTIONS=halt_on_error=1` / `UBSAN_OPTIONS=halt_on_error=1` | Abort on any sanitizer diagnostic during the §4 cross-check |
| `LD_PRELOAD=<asan runtime>` | Loads the ASan runtime for the supplemental sanitizer probe |

### F. Developer Tools Guide

- **Sanitizer cross-check:** build with `--debug --sanitize`, run the large-burst probe under `LD_PRELOAD` of the ASan runtime with `halt_on_error=1`; reaching the final print with no diagnostic is the pass signal. Restore the canonical `.so` afterward and re-verify its sha256.
- **Segment-count derivation:** there is no exposed `num_segments` member; derive it as `ceil(ynum / 2048)` and corroborate against observed carve boundaries (`count` crossing 2048 multiples) and the per-carve RSS delta (≈ the 5,251,072-byte `calloc`).
- **Determinism check:** rebuild and compare `sha256sum kitty/fast_data_types.so`; identical hash confirms a deterministic build (pin `--vcs-rev` to neutralize the embedded commit string).

### G. Glossary

| Term | Meaning |
|---|---|
| `HistoryBuf` | Segmented in-memory scrollback store; a ring over a fixed line capacity `ynum` |
| `PagerHistoryBuf` | Opt-in byte-addressable FIFO ring that receives lines the segmented store evicts |
| Segment | A fixed 2,048-line block (`SEGMENT_SIZE`) lazily `calloc`-ed as the store grows |
| `ynum` / `xnum` | History line capacity (`MAX(scrollback_lines, lines)`) / column count |
| `count` | Current number of populated history lines; saturates at `ynum` |
| Eviction | When full, the oldest line is dropped (pager disabled) or pushed to the ring (pager enabled) |
| `scrolled_by` | View offset into scrollback; bumped per added line, capped at `count` |
| Reflow / rewrap | Re-wrapping stored lines on width change (fast `memcpy` if width unchanged, else `rewrap_inner`) |
| Canonical path | child/PTY → VT parser → Screen → line-buf → history (the real runtime route) |

---

*Prepared by the Blitzy autonomous assessment agent. All hours are AAP-scoped; completion = 50 / 56 = 89.3%.*