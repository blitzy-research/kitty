# Blitzy Project Guide — kitty OSC 133 Shell-Integration Investigation

> Runtime-grounded Q&A documentation deliverable for the kitty terminal emulator.
> Brand color key: **Completed / AI Work = Dark Blue `#5B39F3`**, **Remaining / Not Completed = White `#FFFFFF`**, Headings/Accents = Violet-Black `#B23AF2`, Highlight = Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

kitty is a fast, GPU-based terminal emulator. This project delivers one authoritative, **runtime-grounded** Q&A document explaining how kitty processes **OSC 133 shell-integration** ("FinalTerm semantic prompt") escape sequences — the command-boundary markers `A`, `B`, `C`, and `D[;<exit-code>]`. The target audience is developers investigating kitty's command-tracking internals. The document answers eight sub-questions covering what command output is captured, whether the OSC markers survive capture, the exact byte geometry per exit code, how the exit code is recorded, and how malformed codes are handled — every claim grounded in built-and-run evidence with exact `file:line` citations. Scope is strictly read-only: one markdown file is added; zero existing source is touched.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieStrokeWidth':'2px'}}}%%
pie showData title Completion Status: 92.9% Complete
    "Completed (AI)" : 26
    "Remaining" : 2
```

| Metric | Hours |
|---|---|
| **Total Hours** | **28** |
| **Completed Hours** (AI 26 + Manual 0) | **26** |
| **Remaining Hours** | **2** |
| **Percent Complete** | **92.9%** (26 ÷ 28) |

> All completed work is autonomous (AI). No manual hours were logged. Remaining hours are purely path-to-production human sign-off for a documentation artifact.

### 1.3 Key Accomplishments

- ✅ Built kitty's native extension (`kitty/fast_data_types.so`) and launcher; confirmed the OSC 133 code path executes end-to-end.
- ✅ Traced the OSC 133 byte path across **two language layers** — C (`vt-parser.c`, `screen.c`) and Python (`window.py`).
- ✅ Authored and ran **3 runtime probe drivers** (test-harness, production-method, wrapper-vs-lower-level) capturing verbatim output for **8 exit-code variations**.
- ✅ Delivered a **437-line, 10-section** answer document; **all 8 sub-questions** answered plus a final coverage pass.
- ✅ **71 `file:line` citations** in the document; ~60 independently re-verified exact with **zero drift**.
- ✅ Full autonomous suite green: **145 Python tests + all Go tests**; canonical `test_prompt_marking` passes (re-run this session).
- ✅ **Strict read-only compliance**: net diff = 1 added file, clean working tree; 3 ephemeral `/tmp` probes deleted.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None identified_ | No blocking issues. Deliverable is complete, committed, and independently verified. | — | — |

### 1.5 Access Issues

**No access issues identified.** The investigation required only local repository access and a build toolchain (both available). No repository permissions, service credentials, or third-party API access were needed — the task introduces no runtime service, network dependency, or external integration.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _None_ | — | No access issues encountered | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Have a kitty/terminal SME review `blitzy/documentation/kitty_815df1e210e0.md` for correctness and completeness against the original 3-part question (Q1/Q2/Q3).
2. **[Medium]** Independently reproduce the 3 probe drivers (`PYTHONPATH="$PWD"`, terminal width `cols=80`) to confirm the verbatim byte-geometry, capture strings, and recorded exit statuses.
3. **[Medium]** Merge/publish the deliverable to surface the answer document to the requester.
4. **[Low]** Optionally re-run the full test suite in the target environment as a final health check before merge.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| Environment setup & native build | 4.0 | Installed apt build deps (`.github/workflows/ci.py:84-88` + `libssl-dev`/`zlib1g-dev`, resolving an OpenSSL/zlib link failure); ran `python3 setup.py build` to produce `fast_data_types.so` + launcher; verified native import. |
| Static source trace + citation map | 5.0 | Traced the OSC 133 dispatch through C (`vt-parser.c` `case 133` → `screen.c` `shell_prompt_marking`) and Python (`window.py` `cmd_output_marking`/`handle_cmd_end`/`cmd_output`); built and verified the 60+ `file:line` citation map. |
| Probe-driver authoring & runtime experiments | 4.5 | Authored 3 throwaway drivers in `/tmp`; drove the exact user marker stream through a real `Screen` across 8 exit-code variations; captured plain + ANSI output and inspected cell text. |
| Answer-document authoring | 7.0 | Wrote the 437-line, 10-section document: environment, marker stream, pipeline, verbatim output, all 8 Q&A answers, harness-vs-production distinction, citation map, web corroboration, coverage pass. |
| Q1b correction + reproducibility refinements | 1.5 | Corrected the Q1b Python `cmd_output` wrapper ANSI-capture claim (commit `4e3332f1b`); added terminal-width + `PYTHONPATH` reproducibility notes (commit `fdc16338c`). |
| Autonomous validation & independent re-verification | 4.0 | Rebuilt; re-ran all 3 probes to identical output; verified all citations; ran full suite (145 Python + Go); clean `-Werror` compile; audited read-only + cleanup compliance. |
| **Total Completed** | **26.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| Human SME review & acceptance of the answer document (confirm it fully answers the original 3-part question) | 1.0 | High |
| Independent reproduction of probe commands + merge/publish the deliverable | 1.0 | Medium |
| **Total Remaining** | **2.0** | |

> **Hours reconciliation (integrity check):** Completed 26.0 + Remaining 2.0 = **28.0 Total** — matches Section 1.2. Remaining 2.0 matches the Section 7 pie "Remaining" value. All remaining items are path-to-production (human sign-off); no AAP deliverable is incomplete.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project (full-suite run by the Final Validator; canonical OSC 133 test independently re-run by this reporting pass).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Python unit/integration suite | `unittest` (`kitty_tests`) | 149 | 145 | 0 | Not instrumented | 4 environmental **skips**: fish-not-installed ×2, macOS-only Last-Resort font, frozen-build CA certs. Runner exit 0. |
| OSC 133 path (canonical) | `unittest` | 1 | 1 | 0 | — | `kitty_tests.screen.TestScreen.test_prompt_marking` — the reference test that exercises the exact OSC 133 capture path described in the document. Re-run this session → `OK` (Ran 1 test, exit 0). *(Subset of the Python suite; shown separately as the OSC-133-specific test.)* |
| Go test suite | `go test` | All (count not enumerated in logs) | All | 0 | Not instrumented | "All Go tests succeeded." |

- **Pass rate (non-skipped):** 100% (145/145 Python + all Go). **Failures/errors:** 0.
- **Coverage %:** the kitty suite is pass/fail and was not run under a coverage instrument, so a numeric percentage is not available; the OSC 133 code path's functional coverage is evidenced by `test_prompt_marking` plus the 3 runtime probe drivers (Section 4).
- **Integrity:** no synthetic or fabricated tests are listed; every entry traces to the autonomous run.

---

## 4. Runtime Validation & UI Verification

**Runtime health** (build → load → execute the OSC 133 path):

- ✅ **Operational** — Native extension builds and loads: `from kitty.fast_data_types import Screen, set_options` succeeds (`Screen: True | set_options: True`).
- ✅ **Operational** — OSC 133 parser dispatch executes: `vt-parser.c:536 case 133:` → `screen.c:2328 shell_prompt_marking`.
- ✅ **Operational** — All 3 probe drivers ran end-to-end; the test-harness (§4.1) and wrapper (§4.3) blocks reproduced **strictly identically** to the document; the production driver (§4.2) reproduced identical values.
- ✅ **Operational** — Exit-code recording verified: `99` recorded as `int` 99 in both drivers + `on_cmd_startstop` watcher; malformed → `0` (production) / sentinel unchanged (harness).
- ✅ **Operational** — Command-output capture verified: plain mode = `some text`; ANSI lower-level join = `\x1b[m\x1b]133;C\x1b\\some text`; Python wrapper strips the boundary → `\x1b[msome text`.

**Measured byte-geometry evidence** (observed; String Terminator `ST = ESC \`, two bytes):

| `D` argument | Total bytes | `D` marker OSC offset | exit-code digit offset | Recorded (production) | Recorded (test harness) |
|---|---|---|---|---|---|
| `0` | 62 | 51 | 59 | `0` | `0` |
| `1` | 62 | 51 | 59 | `1` | `1` |
| `42` | 63 | 51 | 59 | `42` | `42` |
| `99` | 63 | 51 | 59 | `99` | `99` |
| `127` | 64 | 51 | 59 | `127` | `127` |
| `not_a_number` | 73 | 51 | 59 | `0` | `9223372036854775807` (sentinel unchanged) |
| `` (empty `D;`) | 61 | 51 | (none) | `0` | `9223372036854775807` (sentinel unchanged) |
| bare `D` | 60 | 51 | (none) | `0` | `9223372036854775807` (sentinel unchanged) |

**UI verification:** ⚠ **Not applicable.** The deliverable is a markdown document describing terminal-internals byte processing; there is no user-facing UI to verify. Runtime verification is therefore code-path/measurement based, as above.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables/rules to quality benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Rule | Status | Progress | Evidence / Notes |
|---|---|---|---|
| Deliverable location & branch-name (`blitzy/documentation/kitty_815df1e210e0.md`) | ✅ Pass | 100% | File exists, 437 lines, committed at HEAD `fdc16338c`. |
| Run-first methodology (build & run before writing) | ✅ Pass | 100% | Native extension built; 3 probe drivers executed; values quoted from runs. |
| Verbatim observed output quoted with producing code | ✅ Pass | 100% | §4.1–4.3 verbatim blocks; §5 per-question answers reference them. |
| All 8 sub-questions answered + final coverage pass | ✅ Pass | 100% | §5 answers Q1a–Q3; §9 coverage pass marks each `[x]`. |
| Exact `file:line` citations; every claim grounded | ✅ Pass | 100% | §7 map (71 tokens); ~60 re-verified exact; 4 independent spot-checks exact; zero drift. |
| Read-only (no existing file modified/deleted) | ✅ Pass | 100% | Net diff vs base = `A` one file; `git status --porcelain` empty. |
| Cleanup of temporary scripts | ✅ Pass | 100% | 3 ephemeral `/tmp` probes deleted after use. |
| Test-harness vs production divergence documented | ✅ Pass | 100% | §6 explains `suppress(Exception)`/sentinel vs production `try/except → 0`. |
| Zero-placeholder policy (no TODO/FIXME/stub) | ✅ Pass | 100% | 0 placeholder markers in the 437-line document. |
| Build & suite health | ✅ Pass | 100% | Clean `-Werror` compile; 145 Python + all Go tests pass. |
| **Fix applied — Q1b accuracy** | ✅ Resolved | 100% | Commit `4e3332f1b` corrected the Python `cmd_output` wrapper ANSI-capture claim; re-verified identical. |
| **Fix applied — reproducibility** | ✅ Resolved | 100% | Commit `fdc16338c` added terminal-width (`cols=80`) + `PYTHONPATH` notes. |
| Human SME acceptance | ⬜ Outstanding | 0% | Path-to-production sign-off (Section 2.2, Section 6 risk #6). |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation/source drift if the repo is later rebased onto a newer kitty | Technical | Low | Low | Citations pinned to the stated revision (HEAD `815df1e21`); source is strictly read-only and unchanged | Mitigated |
| Answer accuracy (a measured value or behavior claim is wrong) | Technical | Low | Low | Independently re-verified — 3 probes re-run to identical output; ~60 citations exact; `test_prompt_marking` passes | Mitigated |
| Reproducibility friction (probes need `PYTHONPATH` + non-wrapping width `cols=80`) | Technical | Low | Low | Commit `fdc16338c` documents both; commands tested this session | Resolved |
| Byte totals assume `ST = ESC \` (BEL would shift by 1 byte/marker) | Technical | Low | Low | Assumption explicitly stated in §9; does not change behavior or recorded statuses | Documented |
| Build-dependency availability when reproducing in a fresh environment | Integration | Low | Low | Exact apt list cited (`ci.py:84-88` + `libssl-dev`/`zlib1g-dev`); validator reproduced the build | Mitigated |
| Deliverable not merged/surfaced to the requester | Operational | Low | Low | PR ready; merge/publish tracked as Section 2.2 remaining item | Open (human) |
| Security exposure | Security | None | N/A | Read-only investigation: zero dependency changes, zero source/config edits, no runtime service or credentials; ephemeral probes ran in `/tmp` and were deleted | Not applicable |

**Overall risk profile: LOW.** No High/Critical risks — expected for a read-only documentation task that adds no product code, no dependencies, and no attack surface.

---

## 7. Visual Project Status

**Project hours breakdown** (identical data to Section 1.2; Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieStrokeWidth':'2px'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 26
    "Remaining Work" : 2
```

**Remaining hours by category** (from Section 2.2 — sums to the 2.0 "Remaining Work" value above):

| Category | Hours | Priority | Bar |
|---|---|---|---|
| Human SME review & acceptance | 1.0 | High | ██████████ |
| Independent reproduction + merge/publish | 1.0 | Medium | ██████████ |
| **Total** | **2.0** | | |

**Priority distribution of remaining work:** High = 1.0h (50%), Medium = 1.0h (50%), Low = 0h.

> **Integrity:** the pie "Remaining Work" (2) equals Section 1.2 Remaining Hours (2) and the Section 2.2 Hours sum (2.0).

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.9% complete** (26 of 28 hours). The single AAP deliverable — the branch-named OSC 133 Q&A document — is fully authored, committed, and independently verified. It answers all eight sub-questions with verbatim runtime evidence and exact `file:line` citations, and it was produced under a strict run-first, read-only methodology (net diff = one added file; clean tree).

**Remaining gaps.** The outstanding 2 hours are entirely **human path-to-production sign-off**: (1) an SME review/acceptance of the document, and (2) an optional independent reproduction of the probe measurements followed by merge/publish. No AAP requirement is unfinished; no code, test, or configuration remains to be written.

**Critical path to production.** SME review → (optional) reproduce probes → merge the deliverable. Because the artifact is a self-contained document validated against a green build and test suite, this path carries **low risk** and short duration.

**Success metrics.** 18/18 AAP requirements Completed; 8/8 sub-questions answered; ~60/60 citations verified exact; 145 Python + all Go tests passing; 0 repository files modified; 0 placeholder markers.

**Production readiness assessment.** The deliverable is **production-ready** pending human acceptance. The validator reported all five production-readiness gates passed with no remaining issues; the 92.9% figure reflects the standard reservation of the final human review/merge step (per policy, autonomous completion is capped below 100%).

---

## 9. Development Guide

This guide reproduces the environment used to build kitty, verify the runtime, and (optionally) re-run the OSC 133 measurements. Core commands were executed and confirmed during this reporting pass.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu-family; validated on Ubuntu 25.10 container).
- **Python:** `>=3.8` (`pyproject.toml:2`); validated with **3.13.7**.
- **Go:** `1.22` (`go.mod:3`); validated with **1.22.12** (only needed for the Go test suite / kittens).
- **C toolchain:** `build-essential` (gcc/make).
- **Build libraries** (authoritative apt list at `.github/workflows/ci.py:84-88`), plus `libssl-dev` and `zlib1g-dev` for a clean link.

### 9.2 Environment Setup

```bash
# From the repository root on branch blitzy-bd823d75-c877-45ee-b01f-ea9987c28460
export CI=true
export LANG=C.UTF-8 LC_ALL=C.UTF-8
export PATH="$PATH:/usr/local/go/bin"   # so `go` is discoverable for the test suite
```

### 9.3 Dependency Installation

```bash
DEBIAN_FRONTEND=noninteractive sudo apt-get update
DEBIAN_FRONTEND=noninteractive sudo apt-get install -y \
  build-essential libgl1-mesa-dev libxi-dev libxrandr-dev libxinerama-dev \
  libxcursor-dev libxcb-xkb-dev libdbus-1-dev libxkbcommon-dev libharfbuzz-dev \
  libx11-xcb-dev libpng-dev liblcms2-dev libfontconfig-dev libxkbcommon-x11-dev \
  libcanberra-dev libxxhash-dev uuid-dev libsimde-dev libsystemd-dev \
  libssl-dev zlib1g-dev
```

### 9.4 Build

```bash
PATH=$PATH:/usr/local/go/bin CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  python3 setup.py build
# Produces: kitty/fast_data_types.so  and  kitty/launcher/kitty
```

### 9.5 Verification

```bash
# 1) Native extension present
stat -c '%s bytes  %n' kitty/fast_data_types.so        # e.g. 1253792 bytes (size is build/flag dependent)

# 2) Native extension loads (expect: True True)
PYTHONPATH="$PWD" python3 -c \
  "import kitty.fast_data_types as f; print(hasattr(f,'Screen'), hasattr(f,'set_options'))"

# 3) Canonical OSC 133 test (expect: 'test_prompt_marking ... ok' / 'OK', exit 0)
PATH=$PATH:/usr/local/go/bin CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  ./kitty/launcher/kitty +launch test.py --module screen prompt_marking

# 4) (Optional) Full autonomous suite (145 Python tests + Go)
PATH=$PATH:/usr/local/go/bin CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  ./kitty/launcher/kitty +launch test.py
```

> **Test-runner syntax note:** `--module <name>` selects a test module; the positional argument is the **method** name and is auto-prefixed with `test_` (so `prompt_marking` → `test_prompt_marking`). A `Class.method` form such as `screen.TestScreen.test_prompt_marking` is **not** accepted ("No test named ... found").

### 9.6 View the Deliverable

```bash
wc -l blitzy/documentation/kitty_815df1e210e0.md   # 437
less  blitzy/documentation/kitty_815df1e210e0.md
```

### 9.7 Reproduce the OSC 133 Evidence (optional)

The document's measurements come from throwaway drivers that build a real `Screen`, drive the user's exact marker stream through `parse_bytes`, then read back the capture and recorded exit status. To reproduce, re-create the drivers described in the document's §4 **outside** the repository (e.g. `/tmp`) and run from the repo root:

```bash
PYTHONPATH="$PWD" python3 -B /tmp/osc133_probe.py
```

**Reproduction essentials** (both documented in the deliverable):
- `PYTHONPATH="$PWD"` — the drivers import `kitty_tests.parse_bytes` and `kitty.window`.
- Build the `Screen` with a **non-wrapping width** (`cols=80`). The `BaseTest.create_screen` default is `cols=5`, which would wrap `some text` and change the capture/cell-text observations. Byte-geometry itself is width-independent.

### 9.8 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `error: externally-managed-environment` on `pip install` | Ubuntu system Python (PEP 668) | Use a venv (`python3 -m venv .venv && source .venv/bin/activate`) or pass `--break-system-packages`. |
| Build fails linking OpenSSL/zlib | Missing dev headers | `apt-get install -y libssl-dev zlib1g-dev`. |
| `ImportError: cannot import name 'create_screen' from 'kitty_tests'` | `create_screen` is a `BaseTest` **method**, not a module symbol | In probes, construct `Screen(...)` directly with `cols=80` (as the document's drivers do); `parse_bytes` **is** a module-level import. |
| `No test named ... found` | Wrong runner syntax | Use `--module screen prompt_marking` (bare method name). |
| Captured output shows wrapped/garbled `some text` | Screen width too small (`cols=5`) | Rebuild the `Screen` with `cols=80`. |
| `go` not found during tests | Go not on `PATH` | `export PATH="$PATH:/usr/local/go/bin"`. |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Build native extension + launcher | `CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH=$PATH:/usr/local/go/bin python3 setup.py build` |
| Verify native import | `PYTHONPATH="$PWD" python3 -c "import kitty.fast_data_types as f; print(hasattr(f,'Screen'), hasattr(f,'set_options'))"` |
| Canonical OSC 133 test | `./kitty/launcher/kitty +launch test.py --module screen prompt_marking` |
| Full test suite | `./kitty/launcher/kitty +launch test.py` |
| Deliverable size | `wc -l blitzy/documentation/kitty_815df1e210e0.md` |
| Confirm read-only tree | `git status --porcelain` (expect empty) |
| Net diff vs base | `git diff 815df1e21 --name-status` (expect `A blitzy/documentation/kitty_815df1e210e0.md`) |

### B. Port Reference

**Not applicable.** The deliverable is a document; no server, service, or network port is started or required.

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only file added) |
| `kitty/vt-parser.c` | OSC dispatch (`case 133` → `shell_prompt_marking`) |
| `kitty/screen.c` | `shell_prompt_marking` A/C/D handling; `cmd_output`/`find_cmd_output` capture |
| `kitty/window.py` | `cmd_output_marking`, `handle_cmd_end` (`int()` `try/except`), `cmd_output` wrapper, `decode_cmdline` |
| `kitty/history.c` | Scrollback detection of the `OSC 133;C` boundary (context) |
| `kitty_tests/__init__.py` | `parse_bytes` (module-level), `Callbacks` recorder (`sys.maxsize` sentinel), `create_screen` (`BaseTest` method) |
| `kitty_tests/screen.py` | `test_prompt_marking` reference test |
| `setup.py` | Build entry point (`fast_data_types.so`) |
| `.github/workflows/ci.py` | Authoritative apt build-dependency list (L84-88) |

### D. Technology Versions

| Component | Constraint | Validated |
|---|---|---|
| Python | `>=3.8` (`pyproject.toml:2`) | 3.13.7 |
| Go | `1.22` (`go.mod:3`) | 1.22.12 |
| C compiler | — | gcc 15.2.0 (clean `-Werror`) |
| Build artifact | — | `kitty/fast_data_types.so` (1,253,792 bytes this env — size is build/flag-dependent and not a value any answer depends on) |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `CI` | `true` | Non-interactive test/build behavior |
| `LANG` / `LC_ALL` | `C.UTF-8` | Deterministic locale for build/tests |
| `PATH` | append `/usr/local/go/bin` | Make `go` discoverable for the Go suite |
| `PYTHONPATH` | `"$PWD"` (repo root) | Let probes import `kitty` / `kitty_tests` |

### F. Developer Tools Guide

- **Test runner:** `test.py` launches via `./kitty/launcher/kitty +launch` and imports `kitty_tests.main`. Filter by module with `--module <name>` and by test method with a bare positional name (auto-prefixed `test_`).
- **Parser driver:** `kitty_tests.parse_bytes(screen, data)` feeds raw bytes through the built `Screen` (wraps `data` in a `memoryview`).
- **Recorder:** the harness `Callbacks` object records `last_cmd_exit_status` (initialized to the `sys.maxsize` sentinel) via `with suppress(Exception): ... = int(data)`, which diverges from production's `try/except → 0`.
- **Capture:** `screen.cmd_output(which, callback, as_ansi)` returns command output; the Python `window.cmd_output` wrapper strips a leading synthesized `OSC 133;C` boundary.

### G. Glossary

| Term | Meaning |
|---|---|
| **OSC 133** | Operating System Command 133 — FinalTerm "semantic prompt" shell-integration protocol. `A` = prompt start, `B` = command/input start, `C` = command-output start, `D[;<exit-code>]` = command finished. |
| **ST (String Terminator)** | Terminates an OSC sequence: `ESC \` (two bytes) or `BEL` `\x07` (one byte). All byte counts here assume `ESC \`. |
| **`fast_data_types`** | kitty's native C extension exposing `Screen`, `set_options`, etc. |
| **`shell_prompt_marking`** | The C handler (`screen.c`) that processes OSC 133 `A`/`C`/`D` markers (no `B` case → `B` is a dispatched no-op). |
| **Plain vs ANSI capture** | Plain capture returns visible text (`some text`); ANSI-preserving capture regenerates only a bare `OSC 133;C` boundary rather than echoing the original markers. |
| **Sentinel (`sys.maxsize`)** | `9223372036854775807` on a 64-bit build — the test harness's placeholder for "no valid exit code recorded"; **not** a real exit code. |
| **Cell text** | The screen's stored character grid. OSC 133 control bytes are consumed by the parser (set line attributes + fire a callback) and are never written into cell text. |

---

*Completion basis: PA1 AAP-scoped, hours-based — 26 completed ÷ 28 total = 92.9%. Colors: Completed `#5B39F3`, Remaining `#FFFFFF`. Cross-section integrity validated (Rules 1–5).*