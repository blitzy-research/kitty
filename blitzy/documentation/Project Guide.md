# Blitzy Project Guide
## kitty — Compiled C Extensions and the Test Suite: An Evidence-Grounded Q&A

> **Deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` · **Branch:** `blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825` · **HEAD:** `5503c3462`
> **Legend / Blitzy brand colors:** <span style="color:#5B39F3">■</span> Completed / AI Work = Dark Blue `#5B39F3` · <span style="color:#FFFFFF; background:#333">■</span> Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, run-first investigative Q&A** against the `kitty` terminal emulator codebase. The sole deliverable is one Markdown document that answers, from directly observed runtime evidence, how kitty's compiled C extensions relate to its test-suite execution — which extension modules actually load, how failures cascade when an extension is unavailable, which extensions are critical vs. optional, and the exact import chains established at runtime. The target audience is engineers and reviewers who need a build-and-run-grounded reference. Business impact: a reusable, evidence-backed analysis of kitty's build → test → C-extension dependency structure. Technical scope spans the custom `setup.py` build system, four C/Objective-C `.so` extensions, the C launcher, the Go `kitten` binary, and the entire Python `unittest` harness — all consulted strictly as read-only references.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieOpacity':'1'}}}%%
pie showData title Completion Status — 94.8% Complete
    "Completed Work (AI)" : 55
    "Remaining Work" : 3
```

| Metric | Hours |
|---|---|
| **Total Project Hours** | **58** |
| Completed Hours (AI + Manual) | 55 |
| Remaining Hours | 3 |
| **Percent Complete** | **94.8%** |

> Completion is computed with the PA1 AAP-scoped methodology: `Completed / (Completed + Remaining) = 55 / (55 + 3) = 55/58 = 94.8%`. All 15 AAP requirements are delivered and validated; the remaining 3 hours are mandatory human acceptance review + merge (never 100% before human review, per policy).

### 1.3 Key Accomplishments

- ✅ **All eight sub-questions (Q1–Q8) answered by name**, each with observed output and `file:line` citations.
- ✅ **Canonical build reproduced both paths:** default `python3 setup.py` aborts (exit 1, Wayland `-Werror=switch` environment artifact); documented fallback `python3 setup.py --ignore-compiler-warnings` exits 0 and produces all six artifacts.
- ✅ **Canonical test suite executed and stable:** `./kitty/launcher/kitty +launch test.py` → *Ran 145 tests, FAILED (failures=3, skipped=4)*, Go tests pass, stable across two runs.
- ✅ **Runtime load set observed via the real runner path:** exactly two kitty-authored `.so` modules load — `kitty.fast_data_types` and `kittens.transfer.rsync`.
- ✅ **Four-tier failure cascade reproduced** in temporary copies of the built tree (source never touched), establishing critical vs. optional classification.
- ✅ **Read-only compliance proven:** `git diff --name-status 815df1e210e0..HEAD` = one path (the deliverable); working tree clean; all temporary artifacts removed.
- ✅ **Deliverable exceeds the AAP:** carries an 8-item corrected-fact ledger fixing the AAP's pre-run expectations against observed reality.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None._ No blocking issues. The deliverable required zero fixes during validation. | None | — | — |

> The three test-suite failures (`test_font_selection`, `test_transfer_send`, `test_transfer_receive`) are **explicitly out-of-scope environment artifacts** (missing "Source Code Pro" font; ext4 setgid-bit mismatch), not defects — all three executed to an assertion, proving the extensions loaded. They are documented, not repaired, as the read-only MainRule requires.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| — | — | No access issues identified. The repository, toolchain, and native libraries were all available; the branch is checked out locally and the working tree is clean. | Resolved / N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** Perform the technical review of the deliverable — verify the Q1–Q8 answers are accurate and evidence-grounded; spot-check a sample of `file:line` citations and embedded command+output blocks against the current source tree (~2h).
2. **[Medium]** Confirm the answer document satisfies the question intent, then **sign off and merge** the branch (~1h).
3. **[Low, optional]** Optionally re-run one reproduction on a pinned older toolchain to mirror the AAP's stale version table (the document is already accurate for the live environment, which is the canonical approach per Rule 1).
4. **[Low, optional]** Optionally add a short executive TL;DR atop the document for non-technical readers.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every component below traces to a specific AAP requirement (Q1–Q8 investigation, build/run foundation, deliverable authoring, methodology compliance, and QA).

| Component | Hours | Description |
|---|---|---|
| Environment provisioning & toolchain/native-lib verification | 4 | Verify Python 3.13.7, Go 1.22.12, gcc 15.2.0, pkg-config 1.8.1 and 13+ native libraries (freetype2, fontconfig, harfbuzz, libpng, lcms2, libxxhash, libcrypto, xkbcommon, wayland, gl, dbus, x11, xcb); fonts (511 faces / 267 families). |
| Q1 — Canonical build (default abort + `--ignore-compiler-warnings` fallback) + 6 artifacts | 5 | Reproduce the default `-Werror=switch` abort at `glfw/wl_window.c:668`; apply documented fallback to exit 0; produce `fast_data_types.so`, `rsync.so`, `glfw-x11.so`, `glfw-wayland.so`, `kitty`, `kitten`. |
| Q1 — Canonical test-suite baseline + 2-run stability | 3 | Run `./kitty/launcher/kitty +launch test.py`; confirm *145 tests, failures=3, skipped=4*, Go pass; stable across two runs. |
| Q2/Q8 — Extension-to-test relationship + import-chain tracing | 5 | Trace the transitive then direct import of `fast_data_types` via `BaseTest`; document three import chains with verbatim tracebacks. |
| Q3 — Runtime load-set instrumentation (real `find_all_tests()` probe) | 4 | Drive the real runner path, inspect `sys.modules` at three points, distinguish kitty's own `.so` from stdlib/third-party (`_json` is a builtin). |
| Q4/Q7 — Four-tier failure-cascade reproduction (sandboxed built tree) | 8 | Render each of four extensions unavailable in a temporary tree copy; capture exact error text, banner state, and tests-run counts per tier. |
| Q5 — Dependency-structure synthesis (3-level hierarchy) | 3 | Synthesize load set + chains + cascade into a coherent root/branch/leaf structure. |
| Q6 — 23-module category-to-extension mapping | 3 | Enumerate every import site across all 23 test modules; build the category→extension matrix (15 direct + 8 transitive). |
| Deliverable authoring (2,867-line evidence-grounded Markdown) | 12 | Author the full document: observed output next to every claim, `file:line` citations, tables, Mermaid diagrams, baseline-artifact separation. |
| Read-only compliance + temp-artifact cleanup + git verification | 3 | Anchor evidence on the stable branch-point; prove `git diff` = one path; remove `/tmp` sandboxes; verify clean tree. |
| QA/validation refinement (4 commits, 8 QA findings, corrected-fact ledger) | 5 | Iterative accuracy refinement to runtime reality; resolve 8 QA findings; build the 8-item corrected-fact ledger. |
| **Total Completed** | **55** | |

### 2.2 Remaining Work Detail

Each item traces to a path-to-production need (human acceptance). No AAP requirement is outstanding.

| Category | Hours | Priority |
|---|---|---|
| Human review & verification of deliverable answers (Q1–Q8 accuracy, evidence sufficiency, `file:line` spot-checks) | 2 | High |
| Stakeholder acceptance sign-off & PR merge | 1 | Medium |
| **Total Remaining** | **3** | |

> Cross-section check: **2.1 (55) + 2.2 (3) = 58 = Total Project Hours** in §1.2. **Remaining (3)** is identical in §1.2, §2.2, and the §7 pie chart.

### 2.3 Hours Methodology Notes

- **Denominator = AAP scope only.** Hours cover the eight sub-question investigations, the build/run foundation, deliverable authoring, methodology compliance, and QA — plus path-to-production human acceptance. No out-of-scope work is included.
- **Completed = 55h** reflects the depth of a multi-language build (141 MB repo), a 145-test suite run, runtime instrumentation, four cascade experiments, and a 2,867-line evidence-grounded document refined across four commits.
- **Remaining = 3h** is purely human acceptance; there is no deployment, CI/CD, or integration work for a documentation deliverable.
- **Confidence:** High. The scope is well-defined and fully delivered; validation reproduced every claim with zero fixes.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** — the canonical kitty test suite executed via `./kitty/launcher/kitty +launch test.py` during the Q1 investigation and re-confirmed during validation (two stable runs).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Python unit + integration | `unittest` (via canonical launcher) | 145 | 138 | 3 | N/A | 4 skipped. The 3 failures (`test_font_selection`, `test_transfer_send`, `test_transfer_receive`) are **out-of-scope environment artifacts** (missing "Source Code Pro" font; ext4 setgid mismatch); all three executed to an assertion, proving the extensions loaded. |
| Go | `go test` | All | All | 0 | N/A | Suite summary: *"All Go tests succeeded"* (exact count not enumerated in the summary line). Go toolchain 1.22.12 required by the canonical entry (`reduce_go_pkgs()`). |
| C extensions (smoke) | `check_build.py` via Python bindings | — | Pass | 0 | N/A | `test_loading_extensions` asserts `fast_data_types` and `rsync` import successfully; `test_glfw_modules` validates GLFW backends as files (x11-only under `CI=true`). |

**Aggregate:** Python unittest **145 run / 138 passed / 3 failed (out-of-scope) / 4 skipped**; **Go: all succeeded**; **exit code 1** (attributable solely to the three documented environment artifacts). Results were **stable across two runs** (only wall-clock timing differed: 15.6 s vs 21.5 s).

> **Note:** The deliverable itself is documentation and has no in-scope unit tests. The task-level "tests" were the reproduction of each documented behavioral claim; all reproductions passed 100% and were stable across two runs.

---

## 4. Runtime Validation & UI Verification

Runtime behavior was validated end-to-end through the canonical entry point. There is **no graphical UI** for this task (the deliverable is a Markdown document; the subject is a headless terminal-emulator test run), so UI verification is not applicable; runtime validation is reported instead.

**Build & launcher**
- ✅ **Operational** — `python3 setup.py --ignore-compiler-warnings` exits 0; all six artifacts produced with expected byte sizes.
- ✅ **Operational** — `./kitty/launcher/kitty --version` → `kitty 0.35.2`; `./kitty/launcher/kitten --version` → `kitten 0.35.2`.
- ⚠ **Partial (by design / environment)** — default `python3 setup.py` aborts (exit 1) at `glfw/wl_window.c:668`; documented environment-version artifact with a documented fallback. Out of scope to fix.

**Test-suite runtime**
- ✅ **Operational** — canonical `test.py` runs 145 Python tests + Go suite; Go reports "All Go tests succeeded".
- ✅ **Operational** — Q3 load-set probe (real `find_all_tests()` path) observed exactly two kitty-authored `.so` modules load: `kitty.fast_data_types`, `kittens.transfer.rsync`.
- ✅ **Operational** — four-tier cascade reproduced deterministically (Tier 1/2 fatal; Tier 3 localized; Tier 4 no-op under CI).
- ⚠ **Partial (out-of-scope artifacts)** — three baseline failures are environment artifacts, not extension-load failures; explicitly documented.

**Repository state**
- ✅ **Operational** — working tree clean (`git status --porcelain` empty); tracked source unchanged; deliverable committed at HEAD `5503c3462`.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the binding SWE-AtlasQnA rule set to observed compliance.

| Requirement / Benchmark | Status | Progress | Evidence |
|---|---|---|---|
| **MainRule** — deliverable `blitzy/documentation/<branch>.md` created | ✅ Pass | 100% | `blitzy/documentation/kitty_815df1e210e0.md` (2,867 lines) exists and is committed. |
| **MainRule** — read-only; no source file modified/added except the answer doc | ✅ Pass | 100% | `git diff --name-status 815df1e210e0..HEAD` = one path (the deliverable); 2,867 insertions, 0 deletions. |
| **MainRule** — temporary scripts removed; repo left unchanged | ✅ Pass | 100% | Working tree clean; `/tmp` sandboxes/probes removed; build artifacts git-ignored. |
| **Rule 1** — run-first; canonical entry point; 2-run stability; exact commands | ✅ Pass | 100% | Canonical `./kitty/launcher/kitty +launch test.py` used; results stable across two runs; exact build/run commands recorded. |
| **Rule 2** — exhaustive condition coverage (happy + error/edge paths); before/during/after | ✅ Pass | 100% | Happy path + four unavailability edge paths; banner-printed vs. not, tests-run vs. zero captured per tier. |
| **Rule 3** — observed output next to each claim; inferred statements labeled | ✅ Pass | 100% | Every behavioral claim carries its command + complete unedited output; `Inferred:` prefix used where applicable. |
| **Rule 4** — every sub-question answered by name; `file:line` refs; cause→effect | ✅ Pass | 100% | Final coverage-pass table answers Q1–Q8 by name with citations and causal reasoning. |
| **Accuracy vs. runtime reality** | ✅ Pass | 100% | Final Validator: 100% accurate, zero fixes; independently re-confirmed by this report (test run, load probe, build fallback). |
| **Accuracy vs. AAP pre-run expectations** | ✅ Exceeds | 100% | 8-item corrected-fact ledger fixes AAP assumptions (e.g., dynamic `kitten` link, two `rsync` consumers, `_json` builtin, `/tmp` ext4, 511/267 fonts, 4th cascade tier). |
| **Structural validity of deliverable** | ✅ Pass | 100% | Markdown code fences balanced; all Q sections present; git evidence anchored on the stable permanent branch-point. |

**Fixes applied during autonomous validation:** none required for the deliverable (zero-fix). Prior QA cycles (commits `f939f88c5`, `4b76c25c6`, `5503c3462`) refined a Q2 count/terminology inconsistency, re-anchored the read-only git evidence on the stable branch-point, and resolved 8 QA findings — all reflected in the final document.

**Outstanding compliance items:** none. Only human acceptance review remains.

---

## 6. Risk Assessment

Overall risk is **LOW** — a read-only, fully validated investigation on a clean working tree with no code, credentials, or integrations introduced.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Environment-version drift: observed toolchain (Python 3.13.7, Go 1.22.12, gcc 15.2.0, wayland-protocols 1.45) differs from the AAP's stale table (3.12.3 / 1.22.2 / 13.3.0) | Technical | Low | Medium | Document reports **observed** versions per Rule 1 and explicitly notes the drift | Documented |
| Default build aborts (`-Werror=switch` at `glfw/wl_window.c:668` from newer `wayland-protocols`) | Technical | Low | High (deterministic) | Documented `--ignore-compiler-warnings` fallback; identified as environment artifact, not a code defect | Documented / out-of-scope to fix |
| Reproducibility depends on provisioning 13+ native libs, Go, and fonts | Operational | Low | Medium | Document lists exact prerequisites and observed versions; §9 provides copy-pasteable setup | Documented |
| Three baseline test failures misread as product defects | Operational | Low | Medium | Document classifies them as environment artifacts distinct from extension-load failures; all three executed to an assertion | Documented |
| Deliverable answers must satisfy stakeholder intent/format | Human review | Low | Low | 2,867-line exhaustive document; every sub-question answered by name; refined across four QA cycles | Open (pending review) |
| Security exposure | Security | None | None | Read-only investigation; no code, credentials, auth, or data handling; Markdown deliverable has no attack surface | N/A |
| External integration failure | Integration | None | None | No external integrations, APIs, or services; self-contained documentation | N/A |

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieOpacity':'1'}}}%%
pie showData title Project Hours Breakdown (Total 58h)
    "Completed Work" : 55
    "Remaining Work" : 3
```

**Remaining hours by category (from §2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#5B39F3','primaryTextColor':'#111111','lineColor':'#B23AF2'}}}%%
graph LR
    A["Human review & verification<br/>2h · High"]:::c
    B["Stakeholder acceptance & merge<br/>1h · Medium"]:::c
    classDef c fill:#5B39F3,stroke:#B23AF2,color:#FFFFFF;
```

> **Integrity:** "Remaining Work" = **3h**, identical to §1.2 metrics and the §2.2 "Hours" sum. "Completed Work" = **55h**, identical to §2.1. Completion = **94.8%**.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, evidence-grounded answer to all eight sub-questions about kitty's compiled C extensions and its test suite. It builds kitty from source (documenting both the default abort and the working fallback), runs the 145-test canonical suite (stable across two runs), observes the exact runtime load set (two kitty-authored `.so` modules), reproduces a four-tier failure cascade, and classifies each extension as critical or optional — all inside a single 2,867-line document with observed output and `file:line` citations for every claim.

**Remaining gaps.** None in AAP scope. The only remaining work is **human acceptance**: a technical review of the answers (~2h) and stakeholder sign-off + merge (~1h).

**Critical path to production.** Review → sign-off → merge. There is no build/deploy/CI path for a documentation deliverable; the branch is clean and ready.

**Success metrics.** All 15 AAP requirements complete; 145-test suite reproduced and stable; read-only compliance proven (one changed path); zero fixes required; deliverable exceeds the AAP via an 8-item corrected-fact ledger.

**Production readiness assessment.** The project is **94.8% complete** and production-ready pending human acceptance review. Risk is LOW; the source repository is byte-identical in tracked state to the pre-deliverable branch-point.

| Metric | Value |
|---|---|
| AAP-scoped completion | **94.8%** (55h of 58h) |
| AAP requirements complete | 15 / 15 |
| In-scope defects | 0 |
| Read-only violations | 0 |
| Confidence | High |

---

## 9. Development Guide

All commands below were tested in the project environment and are copy-pasteable. Run from the repository root.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu-family; validated on Ubuntu 25.10 container).
- **Python:** 3.13.7 (satisfies `pyproject.toml` `requires-python = ">=3.8"`).
- **Go:** 1.22.12 (satisfies `go.mod` `go 1.22`) — **required** by the canonical test entry point.
- **C toolchain:** gcc 15.2.0, `pkg-config` 1.8.1.
- **Fonts:** fontconfig with ≥1 family installed (the test env eagerly loads the font DB).

Verify the toolchain:

```bash
python3 --version          # Python 3.13.7
go version                 # go version go1.22.12 linux/amd64  (ensure Go is on PATH)
gcc --version | head -1    # gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
pkg-config --version       # 1.8.1
```

### 9.2 Environment Setup

Ensure the Go toolchain is on `PATH` (the canonical test runner calls `reduce_go_pkgs()` and exits if `go` is absent):

```bash
export PATH="$PATH:/usr/local/go/bin"
```

Verify native libraries resolve (all should print a version):

```bash
for lib in freetype2 fontconfig harfbuzz libpng lcms2 libxxhash libcrypto \
           xkbcommon wayland-client gl dbus-1 x11 xcb; do
  printf "%-16s " "$lib"; pkg-config --modversion "$lib" 2>/dev/null || echo "MISSING"
done
```

### 9.3 Dependency Installation (native libraries)

On a fresh Debian/Ubuntu host, install the build prerequisites (see `docs/build.rst §Dependencies`):

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  build-essential pkg-config python3-dev \
  libfreetype-dev libfontconfig-dev libharfbuzz-dev libpng-dev liblcms2-dev \
  libxxhash-dev libssl-dev libxkbcommon-x11-dev libwayland-dev wayland-protocols \
  libgl1-mesa-dev libdbus-1-dev libxcursor-dev libxrandr-dev libxi-dev \
  libxinerama-dev libx11-xcb-dev zlib1g-dev fontconfig fonts-dejavu-core
```

### 9.4 Build

The default build action is `build`; the canonical command is:

```bash
python3 setup.py
```

> **Known environment behavior:** in an environment with a newer `wayland-protocols` (here 1.45), the default build **aborts with exit 1** at `glfw/wl_window.c:668` because `-Werror=switch` promotes an unhandled-enum warning to an error. This is an environment-version artifact, not a code defect. Use the documented fallback, which drops `-Werror`:

```bash
export PATH="$PATH:/usr/local/go/bin"
python3 setup.py --ignore-compiler-warnings      # exits 0; produces all six artifacts
```

Expected artifacts after a successful build:

```bash
ls -1 kitty/fast_data_types.so kittens/transfer/rsync.so \
      kitty/glfw-x11.so kitty/glfw-wayland.so \
      kitty/launcher/kitty kitty/launcher/kitten
```

### 9.5 Run the Test Suite (canonical entry point)

```bash
CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH="$PATH:/usr/local/go/bin" \
  ./kitty/launcher/kitty +launch test.py
```

Expected terminal summary (stable across runs; only timing varies):

```text
Ran 145 tests in <N>s

FAILED (failures=3, skipped=4)
All Go tests succeeded, ran in <N> seconds
# exit code: 1
```

> The exit code 1 and three failures are the documented **environment artifacts** (`test_font_selection` — missing "Source Code Pro" font; `test_transfer_send` / `test_transfer_receive` — ext4 setgid-bit mismatch). They are **not** extension-load failures — every test executed to an assertion, proving the extensions loaded.

### 9.6 Verification

```bash
./kitty/launcher/kitty --version           # kitty 0.35.2 created by Kovid Goyal
git status --porcelain --untracked-files=all   # (empty) => working tree clean
git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
# => A  blitzy/documentation/kitty_815df1e210e0.md   (the ONLY change)
```

### 9.7 Example Usage — reproduce the Q3 load set

Create a temporary probe **outside** the repository (never inside the tree), run it via the canonical launcher, then delete it:

```bash
cat > /tmp/loadprobe.py <<'PY'
import importlib, sys
importlib.import_module('kitty_tests.main')     # exactly test.py:L8
sys.modules['kitty_tests.main'].find_all_tests()  # the real runner collection path
for n, m in sorted(sys.modules.items()):
    f = getattr(m, '__file__', '') or ''
    if f.endswith('.so') and (n.startswith('kitty.') or n.startswith('kittens.')):
        print(n, '->', f)
PY
CI=true PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch /tmp/loadprobe.py
rm -f /tmp/loadprobe.py
```

Expected output (exactly two kitty-authored `.so` modules; no GLFW module):

```text
kittens.transfer.rsync -> .../kittens/transfer/rsync.so
kitty.fast_data_types -> .../kitty/fast_data_types.so
```

### 9.8 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| Build aborts, exit 1, `-Werror=switch` at `glfw/wl_window.c:668` | Newer `wayland-protocols` introduces enum values not handled by this snapshot | Use `python3 setup.py --ignore-compiler-warnings` |
| `go executable not found, current path: ...` (SystemExit) | Go not on `PATH`; canonical runner requires it | `export PATH="$PATH:/usr/local/go/bin"` before running `test.py` |
| `test_font_selection` fails: "The family: Source Code Pro is not available" | Specific font family not installed (environment artifact) | Expected in this environment; install "Source Code Pro" to clear, or treat as known artifact |
| `test_transfer_send` / `test_transfer_receive` fail | `/tmp` filesystem setgid-bit behavior (ext4) differs from expectation (environment artifact) | Expected; not an extension-load failure |
| Suite banner never prints, 0 tests run | `fast_data_types.so` missing/unloadable (Tier-1 critical) | Rebuild the extension; it is the suite's single-point-of-failure |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Default build (may abort in this env) | `python3 setup.py` |
| Build fallback (exit 0) | `python3 setup.py --ignore-compiler-warnings` |
| Clean object cache (optional) | `rm -rf build` |
| Run test suite (canonical) | `CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch test.py` |
| Launcher version | `./kitty/launcher/kitty --version` |
| Prove read-only compliance | `git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD` |
| Confirm clean tree | `git status --porcelain --untracked-files=all` |
| `make` wrappers | `make all` (→ `python3 setup.py`), `make test` |

### B. Port Reference

Not applicable — this project runs no network services. The test suite is a headless `unittest` + `go test` run with no listening ports.

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only written file) |
| `test.py` | Canonical entry point (`#!./kitty/launcher/kitty +launch`; imports `kitty_tests.main` at L8) |
| `kitty_tests/main.py` | Runner: `find_all_tests()` (direct import at L64), `reduce_go_pkgs()`, env isolation |
| `kitty_tests/__init__.py` | `BaseTest`; Tier-1 import sites (transitive L21, direct L22) |
| `kitty/config.py` → `kitty/conf/utils.py` | Transitive Tier-1 chain to `fast_data_types` (`utils.py:L27`) |
| `kitty_tests/check_build.py` | Build-verification smoke tests (`test_loading_extensions`, `test_glfw_modules`) |
| `kitty_tests/file_transmission.py` | Tier-2 trigger: top-level `rsync` import at L13 |
| `kitty_tests/glfw.py` | Tier-3 on-demand `ctypes.CDLL` GLFW load at L50 |
| `setup.py` | Custom build orchestrator (C extensions, GLFW, launcher, Go `kitten`) |
| `kitty/fast_data_types.so` | Central extension (critical, eager-loaded) |
| `kittens/transfer/rsync.so` | Second critical extension (collection-time) |
| `kitty/glfw-x11.so`, `kitty/glfw-wayland.so` | Optional GLFW backends (on-demand) |

### D. Technology Versions (observed)

| Component | Version | Note |
|---|---|---|
| Python | 3.13.7 (`/usr/bin/python3`) | satisfies `requires-python >= 3.8` |
| Go | go1.22.12 linux/amd64 (`/usr/local/bin/go`) | satisfies `go.mod` `go 1.22` |
| gcc | 15.2.0 (Ubuntu 15.2.0-4ubuntu4) | C/Objective-C compiler |
| pkg-config | 1.8.1 | native-library discovery |
| freetype2 / fontconfig | 26.2.20 / 2.15.0 | font stack |
| harfbuzz / libpng / lcms2 | 10.2.0 / 1.6.50 / 2.16 | shaping / PNG / color |
| libxxhash / libcrypto (openssl) | 0.8.3 / 3.5.3 | rsync hashing / crypto |
| xkbcommon / wayland-client | 1.7.0 / 1.24.0 | X11 / Wayland backends |
| **wayland-protocols** | **1.45** | newer than snapshot targets → default-build `-Werror=switch` abort |
| gl / dbus-1 / x11 / xcb | 1.2 / 1.16.2 / 1.8.12 / 1.17.0 | rendering / desktop / X11 |
| kitty / kitten (built) | 0.35.2 / 0.35.2 | launcher + Go binary |
| Pillow / pygments | 12.3.0 / 2.20.0 | optional Python test deps |

> The AAP's dependency table listed slightly older versions (e.g., Python 3.12.3, Go 1.22.2, gcc 13.3.0). Per Rule 1, the values above are the versions **actually observed** in this environment; the deliverable reports observed values and notes the drift.

### E. Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `PATH` | append `:/usr/local/go/bin` | make the Go toolchain discoverable for the canonical test runner |
| `CI` | `true` | headless test mode; relaxes GLFW file-check to x11-only and extends PTY timeouts |
| `LANG` / `LC_ALL` | `C.UTF-8` | deterministic locale for the test run |

### F. Developer Tools Guide

| Tool | Use |
|---|---|
| `setup.py` | Multi-language build (C/Objective-C extensions, GLFW backends, C launcher, Go `kitten`); `--ignore-compiler-warnings` drops `-Werror` |
| `Makefile` | `make all` → `python3 setup.py`; `make test` |
| `git diff --name-status <branch-point> HEAD` | Prove read-only compliance (exactly one changed path) |
| `pkg-config --modversion <lib>` | Verify native library availability/versions |
| `./kitty/launcher/kitty +launch <script>` | Canonical launcher for running Python under the built kitty runtime |

### G. Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the governing task specification |
| **Canonical entry point** | `./kitty/launcher/kitty +launch test.py` — the real, non-bypassing way to run the suite |
| **`fast_data_types.so`** | Kitty's core C extension; the suite's single-point-of-failure (eager-loaded via `BaseTest`) |
| **`rsync.so`** | `kittens/transfer` C extension; second critical dependency (loaded at collection) |
| **GLFW backends** | `glfw-x11.so` / `glfw-wayland.so` — optional, loaded on demand, never as Python modules headless |
| **Failure tier** | A classification of how far the suite progresses when a given extension is unavailable (Tier 1/2 fatal; Tier 3 localized; Tier 4 no-op under CI) |
| **Corrected-fact ledger** | The 8 items where the deliverable corrects the AAP's pre-run expectations against observed reality |
| **Environment artifact** | A failure caused by the host environment (missing font, filesystem behavior), not a code defect |

---

*Generated by the Blitzy assessment agent. Completion (94.8%) reflects AAP-scoped work only, computed as 55 completed hours / 58 total hours. Colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.*