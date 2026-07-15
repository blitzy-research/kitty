# Blitzy Project Guide — kitty OSC 133 Shell-Integration Investigation

> **Document type:** Investigation / Documentation Q&A deliverable assessment
> **Repository:** `kitty` (kovidgoyal/kitty) · **Source branch:** `kitty_815df1e210e0` · **Base HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> **Working branch:** `blitzy-c95b69bb-fe12-46f0-ba32-022ad94c0d73` · **Deliverable HEAD:** `a959d9874`
> **Brand color legend:** <span style="color:#5B39F3">■</span> **Completed / AI Work** `#5B39F3` · <span style="color:#B23AF2">■</span> **Headings/Accents** `#B23AF2` · <span style="color:#A8FDD9">■</span> **Highlight** `#A8FDD9` · ⬜ **Remaining / Not Completed** `#FFFFFF`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **runtime-grounded investigation** answering how the *kitty* terminal emulator processes an **OSC 133** shell-integration marker stream (the command-boundary markers `A`, `B`, `C`, `D`). Its target users are engineers reasoning about kitty's semantic-prompt pipeline. The single deliverable is one Markdown document, `blitzy/documentation/kitty_815df1e210e0.md`, that answers six empirical questions (Q1–Q6) about output capture, escape-sequence preservation, byte-level geometry, exit-code parsing (including malformed inputs), and end-to-end proof that a value traverses kitty's C→Python pipeline. Per the governing rules, every answer was produced by **building and running the real code paths** — not by reading source — while the source tree remained strictly read-only.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextColor':'#333333'}}}%%
pie showData title Project Completion — 94.4% Complete
    "Completed Work (AI)" : 34
    "Remaining Work" : 2
```

| Metric | Hours |
|--------|:-----:|
| **Total Hours** | **36** |
| Completed Hours (AI + Manual) | 34 |
| &nbsp;&nbsp;• Completed by Blitzy AI | 34 |
| &nbsp;&nbsp;• Completed manually | 0 |
| **Remaining Hours** | **2** |
| **Percent Complete** | **94.4%** |

> **Completion math (PA1, AAP-scoped):** `34 ÷ (34 + 2) = 34 ÷ 36 = 94.4%`. The percentage measures only AAP-scoped work plus the single path-to-production activity applicable to a documentation deliverable (human peer-review/acceptance).

### 1.3 Key Accomplishments

- ✅ **All six investigation questions (Q1–Q6) answered** with observed runtime output, `file:line` citations, and cause→effect reasoning.
- ✅ **kitty C extension (`kitty/fast_data_types.so`) built from source** via the repository's own `setup.py` primitives — 62 compilation units + 1 link, exit 0, zero warnings — **without modifying any source file**.
- ✅ **Real canonical entry points exercised** (not a debug/remote-control bypass): the `PTY` harness (forks a real child, captures raw bytes verbatim) and the real `kitty.window.Window` C→Python callback path.
- ✅ **Byte-geometry established across every exit code** (`0`, `1`, `42`, `99`, `127`) plus malformed (`not_a_number`, empty) and the no-semicolon edge, with an arithmetic derivation.
- ✅ **Exit-code string→int boundary proven** for `99` (sentinel overwrite + real `on_cmd_startstop` watcher) and malformed inputs shown to record `0` in production.
- ✅ **Test-double divergence documented** (`kitty_tests.Callbacks` leaves `sys.maxsize`, production `window.py` yields `0`) — documented, not "fixed", honoring the read-only mandate.
- ✅ **Read-only mandate honored** — exactly one file added; the git-ignored `.so` is the only other artifact; all temporary scripts removed.
- ✅ **Deliverable is a valid 924-line Markdown** with 9 internal sections, an exhaustive citations index, and a rule-compliance checklist.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All AAP-scoped work is complete and independently verified; no blocking issues remain. | — | — | — |

> There are **no critical unresolved issues**. Every question is answered, every runtime value reproduces two-run-stable, the extension compiles cleanly, and the working tree is clean.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _None identified_ | — | No repository-permission, credential, or third-party access was required for this investigation. | N/A | — |

> **No access issues identified.** The task required no external services, credentials, or network access; all behavior is determinable from the local source and its runtime output.

### 1.6 Recommended Next Steps

1. **[High]** Peer-review and accept the investigation answers in `blitzy/documentation/kitty_815df1e210e0.md` (verify Q1–Q6 completeness, cause→effect reasoning, and citations).
2. **[Medium]** Optionally reproduce 1–2 documented values on the target environment (build the `.so`, run the §5 byte-geometry driver and the §4-Q5 real-`Window` driver).
3. **[Low]** Approve and merge the documentation PR into the target branch.
4. **[Low]** *(Optional, out of scope)* Consider filing an upstream informational note about the `kitty_tests.Callbacks` vs `window.py` Q6 divergence — documented here, intentionally not changed under the read-only mandate.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each completed component traces to a specific AAP requirement. All source files are **read-only reference**; the only produced artifact is the documentation file (plus the git-ignored build `.so`).

| Component | Hours | Description |
|-----------|:-----:|-------------|
| Environment setup & C-extension build | 4 | Compile `kitty/fast_data_types.so` via the repo's own `setup.compile_c_extension` + `CompilationDatabase.build_all` (fast-data-types only), working around the out-of-scope GUI `-Werror=switch` failure; no source change. *(AAP §0.3.1, §0.6.2)* |
| Q1/Q2 — capture & preservation investigation | 5 | PTY-harness raw capture vs. parsed `cmd_output`; trace marker stripping through `line.c`/`history.c`/`window.py`; establish the `search_in_pager_hist=False` distinction. *(AAP Q1, Q2)* |
| Q3/Q4 — byte-geometry measurement | 3 | Measure total length and `D`-marker offsets across exit codes `0`/`1`/`42`/`127`; derive the length formula and the fixed offsets (57 / 65). *(AAP Q3, Q4)* |
| Q5 — runtime proof for exit code `99` | 5 | Drive the real `Window.cmd_output_marking` → `handle_cmd_end`; sentinel overwrite; real `Watchers` container + `on_cmd_startstop` watcher; prove the string→int boundary. *(AAP Q5)* |
| Q6 — malformed exit codes + divergence | 3 | Exercise `not_a_number` and empty through production `window.py` (→ `0`); document the `kitty_tests.Callbacks` divergence (`sys.maxsize`). *(AAP Q6)* |
| Edge-case coverage | 2 | `B` no-op, `D`-without-`C` guard early-return, and no-semicolon `D`. *(AAP Rule 2)* |
| Deliverable authoring | 8 | 924-line Markdown: 9 sections, byte-geometry tables, mermaid code-path diagram, cause→effect reasoning, exhaustive citations index, rule-compliance checklist. *(AAP Main Rule)* |
| QA refinement | 3 | F1–F7 QA findings addressed; stale internal cross-references corrected (commits `060bf4fde`, `a959d9874`). |
| Read-only compliance & cleanup | 1 | Git-integrity proofs; confirm `.so`/`build/` git-ignored; remove all temporary observation scripts. *(AAP Main Rule / Rule 3)* |
| **Total Completed** | **34** | |

### 2.2 Remaining Work Detail

Each remaining category traces to a path-to-production need. For a read-only documentation deliverable, the only applicable path-to-production activity is human peer-review/acceptance — there is **no** deployment, CI/CD, environment-configuration, or integration surface (nothing ships to a running system and no source changes were made).

| Category | Hours | Priority |
|----------|:-----:|----------|
| Technical peer-review of the investigation document (verify Q1–Q6, reasoning, citations) | 1.0 | High |
| Independent reproduction of key runtime values on the reviewer's environment (optional confidence check) | 0.5 | Medium |
| PR approval & merge | 0.5 | Low |
| **Total Remaining** | **2.0** | |

### 2.3 Hours Reconciliation

| Quantity | Hours | Source |
|----------|:-----:|--------|
| Completed (Section 2.1 total) | 34 | Sum of all completed components |
| Remaining (Section 2.2 total) | 2 | Sum of all remaining categories |
| **Total Project Hours** | **36** | Completed + Remaining |
| **Percent Complete** | **94.4%** | `34 ÷ 36` |

> **Cross-section check:** `2.1 (34) + 2.2 (2) = 36` = Total Hours in Section 1.2. Remaining `= 2` is identical in Sections 1.2, 2.2, and 7. ✔

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project and were **independently re-executed during this assessment** against a from-scratch rebuild of `kitty/fast_data_types.so`. Every result reproduced exactly.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|:-----------:|:------:|:------:|:----------:|-------|
| Native Unit — Screen / OSC 133 | Python `unittest` | 36 | 36 | 0 | N/A¹ | Includes kitty's own `test_prompt_marking` OSC 133 test |
| Native Unit — VT Parser | Python `unittest` | 16 | 16 | 0 | N/A¹ | OSC dispatch / `133` detection path |
| Native Unit — Data Types | Python `unittest` | 18 | 18 | 0 | N/A¹ | `line`/`history`/color primitives underpinning the path |
| **In-scope native subtotal** | Python `unittest` | **70** | **70** | **0** | N/A¹ | 100% pass, two-run stable |
| Runtime Reproduction — Q1/Q2 (capture vs parsed) | `PTY` harness + real `Screen` | 1 × 2 runs | ✅ | 0 | — | raw retains markers; `cmd_output` = `'some text'` / `'\x1b[msome text'` |
| Runtime Reproduction — Q3/Q4/§5 (byte geometry) | `PTY` harness (raw capture) | 8 × 2 runs | ✅ | 0 | — | lengths 66–79; `ESC]133;D`@57; exit-num@65 |
| Runtime Reproduction — Q5/Q6/§6 (exit-code path) | Real `kitty.window.Window` C→Python | 8 × 2 runs | ✅ | 0 | — | `99`→int 99; malformed→0; test-double `sys.maxsize` |

¹ *Coverage percentage is not emitted by kitty's native test harness; correctness is established by 100% pass plus the exhaustive runtime reproductions rather than a line-coverage metric.*

> **Out-of-scope note (transparency):** The `kitty_tests.shell_integration` module (which exercises the shell-*emitter*-through-*launcher* path, **not** the in-scope OSC 133 parsing/callback code path) reports 1 ERROR (requires the Go launcher binary — GUI/launcher is explicitly out of scope per AAP §0.5.2/§0.7.3) and 4 SKIP (fish/zsh not installed, no network). Its bash non-kitten test **passes** (a real end-to-end OSC 133 exercise). These do not affect the in-scope result.

---

## 4. Runtime Validation & UI Verification

Runtime health of the OSC 133 code path (all exercised via **real** canonical entry points):

- ✅ **Operational** — `kitty/fast_data_types.so` builds from scratch (62 units + 1 link, exit 0, zero warnings) and imports cleanly (`Screen`, `monotonic` present).
- ✅ **Operational** — `PTY` harness: forks a real child whose stdout is a pseudo-terminal, captures `received_bytes` verbatim while a real `Screen` parses the same bytes.
- ✅ **Operational** — Real `kitty.window.Window` C→Python callback path: `cmd_output_marking` → `handle_cmd_end`; `handle_cmd_end` tail `err = None` throughout (full method, including the post-watcher `get_options()` tail, runs cleanly).
- ✅ **Operational** — Real `Watchers` container with a real `on_cmd_startstop` watcher; dispatch is the inherited production `Window.call_watchers` (not monkeypatched) — confirmed `dispatch_is_production=True`.
- ✅ **Operational** — All Q1–Q6 runtime values reproduced **two-run stable** and match the documented values exactly.

UI / API verification:

- ⬜ **N/A — UI verification:** This is a headless terminal-parsing investigation; the GUI is explicitly out of scope (and the GUI toolchain does not compile in-container). No UI surface exists to verify.
- ⬜ **N/A — External API integration:** The task uses no external services, credentials, or network calls; there is no API integration to validate.

---

## 5. Compliance & Quality Review

AAP deliverables and governing rules cross-mapped to quality benchmarks. Fixes applied during autonomous work are noted.

| Benchmark / Rule | Requirement | Status | Evidence / Fixes Applied |
|------------------|-------------|:------:|--------------------------|
| Main Rule — Deliverable | Create `blitzy/documentation/<branch>.md` | ✅ Pass | `kitty_815df1e210e0.md` (924 lines) committed at HEAD `a959d9874` |
| Main Rule — Read-only source | No existing source file modified | ✅ Pass | `git diff --name-status base..HEAD` = single `A` (add); tree clean |
| Main Rule — No extra code | No code added beyond the doc | ✅ Pass | Only artifact besides the doc is the git-ignored `.so` |
| Main Rule — Script cleanup | Remove temporary observation scripts | ✅ Pass | Temp scripts lived under `/tmp` (outside repo); confirmed removed |
| Rule 1 — Run-first | Build & run real canonical paths | ✅ Pass | `.so` built; `PTY` harness + real `Window` path exercised (not a bypass) |
| Rule 2 — Exhaustive coverage | Every named condition + edges | ✅ Pass | Exit codes `0`/`1`/`42`/`99`/`127`, malformed, empty, no-semicolon, `B` no-op all covered |
| Rule 3 — Observed-output discipline | Show output by each claim; label inferred | ✅ Pass | Full unedited transcripts; `Inferred:` prefixes; observed env reported verbatim |
| Rule 4 — Grounded answering | Every sub-question, `file:line`, cause→effect | ✅ Pass | Q1–Q6 each answered; exhaustive §8 citations index; cause→effect throughout |
| Quality — Two-run stability | Values stable across ≥2 runs | ✅ Pass | Every condition run twice; `two_run_stable=True` |
| Quality — Citation accuracy | Citations match source | ✅ Pass | 100% of audited `file:line` references accurate (independently spot-checked) |
| Quality — QA findings | Address review findings | ✅ Pass | F1–F7 addressed (`a959d9874`); stale cross-refs fixed (`060bf4fde`) |
| Quality — Clean compile | Faithful `-Werror`/`-pedantic-errors` | ✅ Pass | 62 units + 1 link, zero warnings, exit 0 |
| Quality — Zero placeholders | No stubs/TODOs in deliverable | ✅ Pass | Complete, self-contained answers; no deferred work |

**Overall compliance: 13 / 13 benchmarks Pass (100%).**

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|:--------:|:-----------:|------------|--------|
| Build prerequisites (dev headers) absent on a reviewer's fresh machine | Technical | Low | Medium | Dev guide (§9) lists exact packages; build uses the repo's own `setup.py` primitives | Mitigated |
| Full GUI toolchain fails to compile (`-Werror=switch` in `glfw/wl_window.c`) | Technical | Low | Low | Out of scope; only `fast_data_types.so` is built (not `setup.build()`); documented | Accepted (worked around) |
| Environment drift — verified under Python 3.13.7 / gcc 15.2.0 vs AAP-nominal 3.12.3 / gcc 13.3.0 | Technical | Low | Low | Byte-geometry & exit-code logic are environment-independent; observed versions reported per Rule 3; re-verified on live env | Mitigated |
| A documented runtime value could be inaccurate | Technical (accuracy) | Medium | Very Low | Every Q1–Q6 value reproduced twice by the author **and** independently re-reproduced during this assessment — exact match | Resolved |
| Reproduction commands depend on kitty **internal** (non-public) APIs pinned to HEAD `815df1e2` | Integration | Low | Low | Findings explicitly pinned to the exact HEAD; commands embedded in the doc (no committed code → no maintenance burden) | Accepted (informational) |
| Reader misreads the Q6 test-double divergence as a production bug | Operational (clarity) | Low | Low | §6.2 explicitly labels the test double non-canonical; production value `0` stated as authoritative | Resolved |
| Read-only mandate violation / stray build artifacts | Operational | High | Very Low | `git status` clean; only 1 file added; `.so`/`build/` git-ignored (verified); temp scripts removed | Resolved |
| Security exposure introduced by the change | Security | None | None | Documentation-only addition: no source/dependency/config/network/auth surface; **zero attack surface** | N/A — none |

**Overall risk posture: LOW.** All material risks are Resolved or Mitigated; the two Accepted items are explicitly out-of-scope per the AAP and do not affect the deliverable's correctness. No security risks are introduced.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = `#5B39F3`, Remaining = `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextColor':'#333333'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 34
    "Remaining Work" : 2
```

**Remaining hours by category** (from Section 2.2; totals **2.0h**):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#FFFFFF','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#333333','pieTitleTextSize':'14px'}}}%%
pie showData title Remaining Work by Category (hours)
    "Peer-review [High]" : 1.0
    "Independent reproduction [Medium]" : 0.5
    "PR approval & merge [Low]" : 0.5
```

> **Integrity check:** Pie "Remaining Work" (`2`) = Section 1.2 Remaining Hours (`2`) = Section 2.2 sum (`1.0 + 0.5 + 0.5 = 2.0`). Pie "Completed Work" (`34`) = Section 2.1 total (`34`). ✔

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **complete and fully grounded in runtime evidence**. All six questions are answered: (Q1/Q2) the raw pseudo-terminal capture retains every OSC 133 sequence verbatim while kitty's parsed `cmd_output` strips them to `'some text'`; (Q3/Q4) across exit codes `0`/`1`/`42`/`99`/`127` the `ESC]133;D` marker sits at byte 57 and the exit-code number at byte 65 with **zero positional shift**, only the total length growing with digit count; (Q5) the value `99` is proven to cross the C→Python boundary as the string `'99'`, become `int 99`, and reach a real `on_cmd_startstop` watcher; (Q6) malformed and empty exit codes record `0` in production `window.py`, with the `kitty_tests.Callbacks` divergence (`sys.maxsize`) documented but intentionally not changed.

**Remaining gaps & critical path to production.** The **project is 94.4% complete** (34 of 36 hours). The remaining **2 hours** are entirely the human review/acceptance gate — technical peer-review (1.0h), optional independent reproduction (0.5h), and PR approval/merge (0.5h). There is no deployment, CI/CD, or integration work, because the deliverable is a read-only documentation artifact that ships nothing to a running system.

**Success metrics.** 70/70 in-scope native tests pass (including kitty's own OSC 133 test); the C extension rebuilds cleanly (62 units + 1 link, exit 0, zero warnings); all Q1–Q6 runtime values reproduce two-run-stable and were independently re-verified; 100% of audited citations are accurate; the working tree is clean with exactly one file added.

**Production-readiness assessment.** **Ready for human review.** The deliverable meets every governing rule (run-first, exhaustive, observed-output, grounded, read-only), contains no placeholders, and has no unresolved issues. Recommendation: proceed to peer-review and merge.

| Metric | Value |
|--------|-------|
| Completion | 94.4% (34 / 36 h) |
| In-scope tests | 70 / 70 pass (100%) |
| Compliance benchmarks | 13 / 13 pass (100%) |
| Critical issues | 0 |
| Access issues | 0 |
| Security risks introduced | 0 |
| Files changed vs base | 1 (added) |

---

## 9. Development Guide

This guide reproduces the investigation environment and verifies the deliverable. Every command was tested during this assessment.

### 9.1 System Prerequisites

- **OS:** Linux (verified on Ubuntu 25.10). **Hardware:** any modern x86-64 host; the build is CPU-light (~1 min).
- **Python:** ≥ 3.8 (verified with **3.13.7**, invoked as `python3`).
- **C toolchain:** `gcc` (verified **15.2.0**), `pkg-config`, `build-essential`.
- **Dev headers:** FreeType, HarfBuzz, Fontconfig, LittleCMS (`lcms2`), `libpng`, `xkbcommon`. *(xxhash and SIMDe are vendored under `3rdparty/`; Wayland/GLFW headers are **not** needed for the in-scope extension.)*
- **VCS:** Git + Git LFS.

### 9.2 Environment Setup

```bash
# From the repository root:
cd /tmp/blitzy/kitty/blitzy-c95b69bb-fe12-46f0-ba32-022ad94c0d73_5515dc
export REPO="$(pwd)"
echo "REPO=$REPO"
```

### 9.3 Dependency Installation (fresh machine only)

No `pip`/`npm` project dependencies are added by this task. On a fresh machine, install the native build headers:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  build-essential pkg-config \
  libfreetype-dev libharfbuzz-dev libfontconfig-dev \
  liblcms2-dev libpng-dev libxkbcommon-dev

# Sanity-check a few libraries are discoverable:
for lib in freetype2 harfbuzz fontconfig lcms2 libpng xkbcommon; do
  pkg-config --exists "$lib" && echo "OK  $lib $(pkg-config --modversion $lib)" || echo "MISSING $lib"
done
```

### 9.4 Build the C Extension (fast-data-types only — no source change)

```bash
cd "$REPO"
rm -f kitty/fast_data_types.so kittens/transfer/rsync.so && rm -rf build   # git-ignored artifacts
python3 - <<'PYEOF'
import os, sys
REPO = os.environ["REPO"]; os.chdir(REPO); sys.path.insert(0, REPO)
import setup                                     # the repository's own build script
args = setup.Options()
setup.init_env_from_args(args, native_optimizations=True)
sources, headers = setup.find_c_files()
headers.append(setup.build_ref_map(args.skip_code_generation))
headers.append(setup.build_uniforms_header(args.skip_code_generation))
with setup.CompilationDatabase(args.incremental) as cdb:
    args.compilation_database = cdb
    # Queue ONLY the kitty/fast_data_types extension — no compile_glfw, no compile_kittens.
    setup.compile_c_extension(setup.kitty_env(args), "kitty/fast_data_types",
        cdb, sources, headers, build_dsym=args.build_dsym)
    cdb.build_all()
print("BUILD_SCRIPT_DONE")
PYEOF
echo "[[build exit $?]]"
```

**Expected:** a progress log ending `[62/62] …`, then `[1/1] Linking kitty/fast_data_types …`, `BUILD_SCRIPT_DONE`, and `[[build exit 0]]`. The artifact `kitty/fast_data_types.so` (~1,253,792 bytes) is produced; `kittens/transfer/rsync.so` remains **absent** (proving a fast-data-types-only build).

### 9.5 Verification

```bash
cd "$REPO"
# 1) Extension imports:
python3 -c "import sys; sys.path.insert(0,'$REPO'); import kitty.fast_data_types as f; print('Screen present:', hasattr(f,'Screen'))"
# 2) In-scope native tests (expect: Ran 70 tests ... OK):
python3 -m unittest kitty_tests.screen kitty_tests.parser kitty_tests.datatypes
# 3) kitty's own OSC 133 test (expect: ok):
python3 -m unittest kitty_tests.screen.TestScreen.test_prompt_marking
# 4) Read-only integrity (expect: single 'A blitzy/documentation/...'; empty porcelain):
git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD
git status --porcelain
```

### 9.6 Example Usage — Reproduce a Documented Value

**Byte geometry (Q3 — `D;42`):**

```bash
python3 - <<'PYEOF'
ST = b'\x1b\\'
s = (b'\x1b]133;A'+ST + b'\x1b]133;B'+ST +
     b'\x1b]133;C;cmdline=some_command'+ST + b'some text' + b'\x1b]133;D;42'+ST)
print("len =", len(s), "| ESC]133;D @", s.find(b'\x1b]133;D'),
      "| exit-num @", s.find(b'133;D;')+6, "  (expect 69 / 57 / 65)")
PYEOF
```

**Parsed output & exit-code path (Q1/Q2/Q5/Q6):** run the self-contained heredocs in the deliverable's §4 (Q1(b) `PTY` driver) and §4-Q5 (real-`Window` driver). Both print full, unedited transcripts and are two-run stable.

### 9.7 Troubleshooting

- **`RuntimeError: Must call set_options() before using get_options()`** when driving `handle_cmd_end` directly — initialize options exactly as `kitty_tests.BaseTest.set_options` does (the 3-line `Options(...)` / `finalize_keys` / `set_options` block shown in §4-Q5) before invoking the real-`Window` tail.
- **GUI/full build fails with `-Werror=switch` in `glfw/wl_window.c`** — expected and **out of scope**; build only `fast_data_types` via `compile_c_extension` (do **not** run `setup.build()`).
- **Missing dev headers** — run `pkg-config --exists <lib>` to identify the gap and install per §9.3.
- **`ImportError: kitty.fast_data_types`** — run the build (§9.4) from `$REPO`; ensure `$REPO` is the current directory / on `sys.path`.
- **Tests can't find the `.so`** — confirm `kitty/fast_data_types.so` exists and that the `python3` running the tests is the same interpreter that built it.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Set repo root | `export REPO="$(pwd)"` |
| Build extension only | `python3 - <<'PYEOF' … compile_c_extension('kitty/fast_data_types', …); cdb.build_all() … PYEOF` (see §9.4) |
| Import check | `python3 -c "import kitty.fast_data_types as f; print(hasattr(f,'Screen'))"` |
| Run in-scope tests | `python3 -m unittest kitty_tests.screen kitty_tests.parser kitty_tests.datatypes` |
| kitty's OSC 133 test | `python3 -m unittest kitty_tests.screen.TestScreen.test_prompt_marking` |
| Read-only diff | `git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD` |
| Clean-tree check | `git status --porcelain` |
| Confirm `.so` git-ignored | `git check-ignore kitty/fast_data_types.so` |

### B. Port Reference

| Port | Service |
|------|---------|
| _None_ | This investigation runs no network service and binds no ports (headless, offline). |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** — the Q1–Q6 answer document (924 lines) |
| `kitty/vt-parser.c` | OSC `133` detection, buffer null-termination, dispatch *(L536–546)* |
| `kitty/screen.c` | `shell_prompt_marking` — `A`/`C`/`D` switch, exit-status string extraction *(L2327–2356)* |
| `kitty/screen.h` | `shell_prompt_marking` declaration *(L231)* |
| `kitty/data-types.h` | `PromptKind` enum *(L230)* |
| `kitty/line.c` | ANSI marker **writer** (`write_mark`, `WRITE_MARK`) *(L328–360)* |
| `kitty/history.c` | Pager-history marker suffix selection *(L474–478)* |
| `kitty/window.py` | `cmd_output_marking`, `handle_cmd_end` (`int()` parsing), `cmd_output` stripping *(L225, L457–468, L1408–1422, L1453–1461)* |
| `kitty_tests/__init__.py` | `parse_bytes`, `PTY` harness, divergent `Callbacks` double *(L30–36, L39/48/71/78, L277–366)* |
| `kitty_tests/screen.py` | Existing `cmd_output` ANSI-reconstruction test *(L1123–1124)* |
| `setup.py` | Build primitives (`compile_c_extension`, `CompilationDatabase`) *(L81, L108, L856, L906)* |
| `kitty/fast_data_types.so` | Git-ignored build artifact (prerequisite) |

### D. Technology Versions

| Component | Version (observed) | Notes |
|-----------|--------------------|-------|
| OS | Ubuntu 25.10 (x86-64) | Runtime host |
| Python | 3.13.7 | Invoked as `python3`; satisfies kitty's ≥ 3.8 minimum |
| gcc | 15.2.0 | With `-Werror -pedantic-errors` (faithful) |
| FreeType / HarfBuzz / Fontconfig | 26.2 / 10.2 / 2.15 | Build deps (verified) |
| lcms2 / libpng / xkbcommon | 2.16 / 1.6.50 / 1.7 | Build deps (verified) |
| Git / Git LFS | system / 3.7.1 | LFS hooks non-blocking |

> *AAP-nominal versions were Python 3.12.3 / gcc 13.3.0; per Rule 3 the **observed** versions above are authoritative. Findings are environment-independent and were re-verified on the observed environment.*

### E. Environment Variable Reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `REPO` | Repository root; consumed by the build and reproduction commands | `/tmp/blitzy/kitty/blitzy-c95b69bb-fe12-46f0-ba32-022ad94c0d73_5515dc` |
| `DEBIAN_FRONTEND` | Non-interactive `apt-get` (fresh-machine install) | `noninteractive` |

> No application/runtime environment variables are required — the deliverable ships nothing to a running system.

### F. Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `python3 -m unittest` | Runs kitty's native test suites (screen/parser/datatypes) |
| `setup.py` primitives | `compile_c_extension` + `CompilationDatabase.build_all` build the extension without the GUI |
| `git diff` / `git status` / `git check-ignore` | Verify the read-only mandate and clean tree |
| `pkg-config` | Confirm native build headers are discoverable |
| `kitty_tests.PTY` | Real headless entry point: forks a child, captures raw pty bytes |

### G. Glossary

| Term | Definition |
|------|------------|
| **OSC 133** | The "Prompt Markers" (semantic-prompt) shell-integration protocol emitting prompt/command boundaries |
| **`A` / `B` / `C` / `D`** | Prompt-start / prompt-end (kitty **ignores** `B`) / command-output-start / command-end markers |
| **ST** | String Terminator = `ESC \` = bytes `\x1b\\` |
| **`received_bytes`** | Raw verbatim capture of the child's stdout on the pty (pre-parse) |
| **`cmd_output`** | kitty's parsed command output (post-parse; markers stripped) |
| **`handle_cmd_end`** | `window.py` method that converts the exit-status string via `int()` (malformed → `0`) |
| **Test double** | `kitty_tests.Callbacks` — diverges from production `window.py` for malformed exit codes (leaves `sys.maxsize`) |
| **`fast_data_types.so`** | Aggregated kitty C extension exposing `Screen`, `monotonic`, etc. |

---

*Generated by the Blitzy Platform. Completion percentage reflects AAP-scoped work plus applicable path-to-production activity (PA1 methodology). Completed = `#5B39F3`, Remaining = `#FFFFFF`.*