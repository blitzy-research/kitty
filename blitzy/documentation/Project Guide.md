# Blitzy Project Guide — kitty OSC 133 Shell-Integration Q&A Documentation

> **Artifact under assessment:** `blitzy/documentation/kitty_815df1e210e0.md` (452 lines)
> **Repository:** kitty terminal emulator · **Version:** 0.35.2 · **Branch:** `blitzy-82bb3e4d-6d7a-4163-acad-5323f9aa2d1e` · **Base:** `815df1e210e0` · **HEAD:** `7ab4a424`
> **Color legend:** 🟦 Completed / AI Work = Dark Blue `#5B39F3` · ⬜ Remaining / Not Completed = White `#FFFFFF`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-backed markdown document — `blitzy/documentation/kitty_815df1e210e0.md` — that explains how the **kitty** terminal emulator (v0.35.2) parses and handles **OSC 133** shell-integration ("semantic prompt" / FinalTerm) command-boundary escape sequences: the `A`/`B`/`C`/`D` markers and the exit code carried by the `D` marker. Every answer is grounded in **actual runtime output** captured from kitty's compiled C VT parser, not code-reading alone. The audience is kitty maintainers and terminal-protocol engineers who need an authoritative, reproducible reference for kitty's OSC 133 behavior. The scope is strictly **read-only**: no existing source is modified — only the answer document is created — making the business impact a zero-risk knowledge asset.

### 1.2 Completion Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeWidth": "2px", "pieSectionTextColor": "#111111", "pieTitleTextSize": "15px"}}}%%
pie showData title Completion Status — 94.1% Complete
    "Completed Work (AI)" : 32
    "Remaining Work" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | 34.0 |
| **Completed Hours (AI + Manual)** | 32.0 (AI = 32.0, Manual = 0.0) |
| **Remaining Hours** | 2.0 |
| **Percent Complete** | **94.1%** |

*Completion is computed per the AAP-scoped hours methodology: `Completed ÷ (Completed + Remaining) = 32.0 ÷ 34.0 = 94.1%`. All remaining hours are human path-to-production work (review + merge); no autonomous AAP deliverable is outstanding.*

### 1.3 Key Accomplishments

- ✅ Authored the **452-line** runtime-grounded answer document at the mandated path `blitzy/documentation/kitty_815df1e210e0.md`.
- ✅ Built kitty **0.35.2** in its canonical configuration (`python3 setup.py build`) and drove all probes through the **real compiled C VT parser** (`fast_data_types.so`).
- ✅ Answered **Q1** (baseline capture), **Q2** (exit-code sweep), and **Q3** (malformed/edge) with complete, unedited captured output and per-block producing commands.
- ✅ Exercised every named case: markers `A`/`B`/`C`/`D` (including the `B`-is-a-no-op finding), exit codes `0`/`1`/`42`/`99`/`127`, and the malformed `not_a_number` + empty payloads.
- ✅ Documented the decisive **canonical (`int()`→`0`) vs non-canonical proxy** divergence for Q3, correctly flagging the harness proxy as non-canonical.
- ✅ **100% of `file:line` citations verified**; **58/58** OSC 133 code-path tests pass (parser 16/16, screen 36/36, shell_integration 6/6).
- ✅ **Read-only compliance:** exactly one added file (452 insertions, 0 deletions); temporary scripts removed; `git status --porcelain` empty.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| *No critical, release-blocking issues identified* | None — all AAP deliverables complete and validated | — | — |
| 6 pre-existing environmental test failures in out-of-scope subsystems (font handling, file-transfer kitten) | **None / non-blocking** — unrelated to OSC 133, excluded by AAP §0.4.2, proven not a regression (doc-only diff) | Optional (org-wide, separate task) | N/A |
| Human technical review & sign-off pending | Non-blocking — standard path-to-production gate | Human reviewer | With merge (≤2.0h) |

*There are no unresolved issues that block release or validation of the deliverable.*

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| — | — | **No access issues identified** | N/A | — |

*Repository access, build toolchain, and the headless test harness were all available. The canonical build succeeded and all in-scope tests executed. No repository permissions, service credentials, or third-party API access are required for this documentation deliverable.*

### 1.6 Recommended Next Steps

1. **[High]** Perform technical review & sign-off of `blitzy/documentation/kitty_815df1e210e0.md` — read the Q1/Q2/Q3 answers, optionally re-run the two verbatim Appendix scripts against a built tree to confirm the reported numbers, and confirm the canonical-vs-proxy caveat is unambiguous. *(1.5h)*
2. **[Medium]** Review the PR for read-only compliance (`git diff --name-status` = one added file), approve, and merge to `main`. *(0.5h)*
3. **[Low]** *(Optional, out of scope)* If an org-wide green full test suite is desired, address the 6 environmental failures (font-package naming, `/tmp` setgid) as a separate task — they are unrelated to this deliverable.
4. **[Low]** *(Informational)* If the document is later read against a different kitty revision, re-verify the line-number citations (pinned to commit `815df1e210e0` / kitty 0.35.2).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical build & environment setup | 3.0 | Install build prerequisites (harfbuzz, freetype, fontconfig, libpng, lcms2, libxxhash, gcc-13, go, pkg-config); `python3 setup.py build` → `fast_data_types.so` + launcher; verify `kitty 0.35.2`. |
| OSC 133 code-path investigation | 5.0 | Trace the full lifecycle: dispatch `vt-parser.c:L536` → `shell_prompt_marking` `screen.c:L2328` (A/C/D cases, exit-status slice, absent B) → `window.py` `cmd_output_marking`/`handle_cmd_end`/watcher; emit-side bash/zsh/fish; adjacent `history.c`/`client.py`; harness. |
| Web research — OSC 133 / FinalTerm semantics | 1.5 | Validate `A`/`B`/`C`/`D` field meanings, the `D;<code>` exit-code field, and the `ESC ] 133 ; … ST` wire format against authoritative external references (iTerm2, Contour, terminfo.dev). |
| Observation harness (2 runtime scripts) | 4.0 | Author & debug `osc133_parsepath.py` (drives probes through the real `parse_bytes` → compiled `Screen`) and `osc133_realwindow.py` (binds the real `Window.handle_cmd_end` bytecode + captures the `on_cmd_startstop` watcher payload). |
| Q1 — baseline capture analysis | 2.0 | Two notions of output (raw stream vs rendered screen); OSC present in raw / consumed on screen; total length 56; `133;D` offset 46, code-num offset 52; cmdline decode → 'foo'. |
| Q2 — exit-code sweep analysis | 2.5 | Sweep 0/1/42/99/127; lengths 55/55/56/56/57; offsets 46/52 invariant; digit-width shift analysis (+1 byte/digit); code-99 full-path runtime evidence. |
| Q3 — malformed/edge analysis | 2.0 | Real `Window.handle_cmd_end` with `not_a_number` and empty → canonical `last_cmd_exit_status = 0` (`int()`-except-`0`); record & label the non-canonical harness-proxy contrast. |
| Answer document authoring | 5.0 | Write the 452-line document: Q1/Q2/Q3 with captured output blocks + tables, supporting details, 19-item coverage self-check, reference index, and verbatim Appendix scripts. |
| Rigor — coverage, citations, stability, cleanup | 2.0 | 19-item coverage pass; `file:line` citation grounding; confirm run-to-run stability (≥2 runs, byte-identical); remove temp scripts; verify `git status --porcelain` empty. |
| Autonomous validation (Final Validator) | 5.0 | 11-phase validation: clean rebuild (C sources exit 0, deterministic `.so`), re-run both scripts ≥2×, verify 100% citations, run parser/screen/shell_integration tests (58/58), confirm read-only compliance. |
| **Total Completed** | **32.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & sign-off of the answer document (Q1/Q2/Q3 correctness, number/citation spot-check, canonical-vs-proxy caveat clarity) | 1.5 | High |
| PR review, read-only-compliance confirmation, approval & merge to `main` | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

*Optional, out-of-scope awareness items (0.0h, not part of the remaining total): (a) 6 environmental full-suite test failures in font/file-transfer subsystems; (b) re-verify pinned line numbers if reading against a different kitty version.*

### 2.3 Hours Reconciliation & Methodology

- **Total Project Hours** = Completed (2.1) + Remaining (2.2) = **32.0 + 2.0 = 34.0**.
- **Completion %** = Completed ÷ Total = **32.0 ÷ 34.0 = 94.1%**.
- **Cross-section integrity:** Section 2.1 total (32.0) = Section 1.2 Completed Hours; Section 2.2 total (2.0) = Section 1.2 Remaining Hours = Section 7 pie "Remaining Work"; 2.1 + 2.2 (34.0) = Section 1.2 Total Hours. ✅ All consistent.
- **Scope basis (PA1):** Every completed hour traces to an AAP requirement or a path-to-production activity; every remaining hour is human path-to-production. No item outside AAP scope is counted. All 21 AAP-specified requirements are classified **Completed**; the only **Not Started** items are the 2 human path-to-production tasks above.

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project; the **VT Parser** row was additionally re-executed live during this assessment (`Ran 16 tests … OK`).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Unit — VT Parser | kitty `test.py` (Python `unittest`) | 16 | 16 | 0 | 100%* | Exercises OSC dispatch incl. `case 133` (`vt-parser.c:L536`). Re-verified live this session. |
| Unit — Screen / `shell_prompt_marking` | kitty `test.py` (Python `unittest`) | 36 | 36 | 0 | 100%* | A/C/D marker handling and exit-status slice in `screen.c:L2328`. |
| Integration — Shell Integration | kitty `test.py` (Python `unittest`) | 6 | 6 | 0 | 100%* | bash/zsh/fish emit-side; validator installed zsh+fish to un-skip and exercise all cited emit lines. |
| **Total (in-scope, OSC 133 code path)** | | **58** | **58** | **0** | **100%*** | 100% pass. |

*\*Coverage column reflects **pass rate**; kitty's test harness reports pass/fail rather than line-coverage percentages. The 100% figure is the in-scope pass rate.*

**Out-of-scope (not part of the deliverable, reported for completeness):** the full 145-test suite shows **6 environmental failures** — 4 in `kitty_tests.fonts` (installed font packages report PostScript names `FiraCode-*`/`UbuntuMono-*` vs the test's hardcoded `*Roman-*` on Ubuntu 25.10) and 2 in `kitty_tests.file_transmission` (container `/tmp` reports mode `0o40755` vs expected `0o42755` setgid). Both files contain **zero** `133` references and the branch diff is doc-only, so behavior is identical to the base commit. These are explicitly excluded by AAP §0.4.2 (font handling, file-transfer kitten) and are **not regressions**.

---

## 4. Runtime Validation & UI Verification

**Runtime health (compiled C VT parser — the canonical headless OSC 133 runtime):**

- ✅ **Canonical build** — `./kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal` (exit 0, verified this session).
- ✅ **Compiled extension operational** — `PYTHONPATH="$PWD" python3 -c "import kitty.fast_data_types; import kitty_tests; from kitty.window import Window"` resolves to the built tree (exit 0).
- ✅ **Q1 baseline reproduced** — total length **56**, `133;D` offset **46**, code-num offset **52**, rendered screen `['hello','','','','']`, cmdline `'foo'`, exit `42` (independently reproduced this session, exact match).
- ✅ **Q2 sweep reproduced** — lengths **55/55/56/56/57** for codes 0/1/42/99/127; offsets 46/52 invariant; +1 byte per digit.
- ✅ **Q2 code-99 full-path evidence** — `99` observed as `int` in both `last_cmd_exit_status` and the `on_cmd_startstop` watcher payload.
- ✅ **Q3 canonical** — real `Window.handle_cmd_end('not_a_number')` and `('')` both record `last_cmd_exit_status = 0`.
- ✅ **Run-to-run stability** — byte-identical output across ≥2 runs (only the documented monotonic `time` float varies).

**API integration:**

- ✅ **Real entry point** — probes flow through `parse_bytes(screen, data)` → compiled `Screen`; malformed cases additionally through the real `Window.handle_cmd_end` bytecode.
- ✅ **Non-canonical paths correctly avoided as source-of-truth** — `--dump-commands` hook and remote-control emitter mentioned but not used; harness proxy flagged non-canonical.

**UI verification:**

- ⚠ **GUI UI verification — Not Applicable.** kitty's full GUI requires a GPU/display and is not runnable headlessly (a documented constraint); the compiled VT parser + `Screen` is the canonical way to exercise OSC 133 without a display. This project has **no web/frontend UI** — the deliverable is a markdown document and the subject is a terminal control-sequence parser — so browser-based UI verification does not apply.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP mandates and rules to Blitzy's quality benchmarks:

| Requirement / Mandate (AAP) | Benchmark | Status | Evidence / Progress |
|------------------------------|-----------|--------|---------------------|
| Deliverable at mandated path `blitzy/documentation/kitty_815df1e210e0.md` | Correctness | ✅ Pass | `git diff` shows `A blitzy/documentation/kitty_815df1e210e0.md` (452 lines). |
| Read-only scope (no source/test/config/build modified) | Compliance | ✅ Pass | Diff = exactly 1 added file, 0 deletions; `git status` clean; temp scripts under `/tmp` removed. |
| Run-first-then-write (runtime-grounded) | Methodology | ✅ Pass | Build artifacts present; every claim backed by captured output blocks. |
| Real entry point; no bypass as source of truth | Methodology | ✅ Pass | `parse_bytes` + real `Window.handle_cmd_end`; `--dump-commands`/remote/proxy flagged non-canonical. |
| Canonical build + exact commands stated | Reproducibility | ✅ Pass | `python3 setup.py build`; `kitty 0.35.2` stated in §Method. |
| Exercise every condition (A/B/C/D; 0/1/42/99/127; malformed) | Completeness | ✅ Pass | 19-item coverage self-check, all ✅; `B`-no-op documented. |
| State before / during / after reported | Rigor | ✅ Pass | Default `last_cmd_exit_status = 0` before; recorded value after. |
| Complete, unedited output + producing command per claim | Evidence | ✅ Pass | 9 captured code blocks; per-block "Produced by …" notes. |
| Run-to-run stability (≥2 runs) | Reliability | ✅ Pass | Byte-identical across runs; confirmed by validator. |
| `file:line` citations naming the function/method | Traceability | ✅ Pass | Reference index; **100%** independently verified. |
| Web-search validation of OSC 133 / FinalTerm semantics | Correctness | ✅ Pass | §OSC 133/FinalTerm background (iTerm2/Contour/terminfo.dev). |
| Zero placeholders/TODOs | Quality | ✅ Pass | Full-text scan: no TODO/FIXME/placeholder in the deliverable. |

**Fixes applied during autonomous validation:** none required — the Final Validator found the deliverable complete, accurate, and correctly cited, and made no edits (validation-only; no new commit needed).
**Outstanding compliance items:** human technical review & sign-off (path-to-production).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Line-number citations pinned to kitty 0.35.2 / commit `815df1e210e0` could drift if read against a different revision | Technical | Low | Medium | Reference index is explicitly headed "verified at commit `815df1e210e0…`, kitty 0.35.2"; every citation scoped to that revision | Mitigated (disclosed) |
| Env-specific build flag `--ignore-compiler-warnings` needed here (newer `wayland-protocols` triggers `-Werror=switch` in the **unrelated** GLFW Wayland backend) | Technical | Low | Low-Medium | Doc labels it environment-specific & non-canonical; flag disables `-Werror` only; VT parser functionally identical; canonical command remains plain `setup.py build` | Mitigated (disclosed) |
| 6 out-of-scope environmental test failures (fonts, file-transfer) | Technical / QA | Low | N/A (pre-existing) | Proven not caused by deliverable (zero `133` refs; doc-only diff = base-identical behavior); excluded by AAP §0.4.2; not fixable without modifying out-of-scope files (forbidden) | Accepted / Documented |
| Human review is the sole quality gate before merge | Operational | Low | Low | Flagged as a High-priority remaining task (§1.6, §2.2) | Open (assigned to human) |
| Security surface | Security | None | — | Markdown-only deliverable; zero executable code, zero dependencies added, no auth/crypto/injection/XSS surface | N/A — none introduced |
| Production integration | Integration | None | — | Zero imports/dependencies/interface changes; strictly read-only | N/A — none |
| Appendix-script reproduction requires a locally built kitty tree + `PYTHONPATH` | Integration | Low | Low | §Method + Appendix give exact build & invocation commands; scripts reproduced verbatim | Mitigated (documented) |

**Overall:** minimal risk profile — **no High or Medium severity risks**, zero security and zero production-integration risk. Nothing blocks release; the only meaningful action is human review before merge.

---

## 7. Visual Project Status

**Project hours — Completed vs Remaining** (🟦 `#5B39F3` = Completed · ⬜ `#FFFFFF` = Remaining):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeWidth": "2px", "pieSectionTextColor": "#111111", "pieTitleTextSize": "15px"}}}%%
pie showData title Project Hours Breakdown (Total 34.0h)
    "Completed Work" : 32
    "Remaining Work" : 2
```

**Remaining work by category / priority** (sums to 2.0h — matches §1.2 Remaining and §2.2 total):

| Category | Hours | Priority |
|----------|-------|----------|
| Technical review & sign-off | 1.5 | High |
| PR review & merge | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

*Integrity check: pie "Remaining Work" = 2 = §1.2 Remaining Hours = §2.2 total = sum of the table above. ✅*

---

## 8. Summary & Recommendations

**Achievements.** The project is **94.1% complete** (32.0 of 34.0 hours). All **21 AAP-specified requirements are delivered and validated**: the mandated 452-line document exists at `blitzy/documentation/kitty_815df1e210e0.md` and answers Q1/Q2/Q3 with complete, unedited runtime output grounded in kitty's compiled C VT parser. Marker `A`/`B`/`C`/`D` handling (including the `B`-is-a-no-op finding), exit codes `0`/`1`/`42`/`99`/`127`, and the malformed `not_a_number`/empty payloads are all exercised. The decisive canonical (`int()`→`0`) vs non-canonical proxy divergence for Q3 is documented and correctly labeled. All `file:line` citations are verified (100%) and all 58 in-scope OSC 133 tests pass.

**Remaining gaps.** The outstanding **2.0 hours** is entirely **human path-to-production**: a technical review & sign-off of the document (1.5h) and PR review + merge (0.5h). There are no outstanding autonomous deliverables, no compilation blockers, and no failing in-scope tests.

**Critical path to production.** (1) Human reviewer reads the document and optionally re-runs the two verbatim Appendix scripts to confirm the reported numbers → (2) approve the read-only PR (one added file) → (3) merge. Estimated wall-clock: well within a single review session.

**Success metrics (all met for the in-scope deliverable):** 100% AAP requirement coverage · 100% in-scope test pass (58/58) · 100% citation accuracy · read-only compliance (1 added file, clean tree) · runtime reproducibility (byte-identical ≥2 runs).

**Production readiness assessment:** **READY pending human review.** The deliverable is complete, accurate, fully runtime-grounded, correctly cited, and read-only-compliant. Per Blitzy policy the project is held below 100% until human sign-off; the only work between here and merge is that review.

| Dimension | Status |
|-----------|--------|
| AAP requirements delivered | 21 / 21 (100%) |
| In-scope tests passing | 58 / 58 (100%) |
| Citation accuracy | 100% |
| Read-only compliance | ✅ (1 added file, clean tree) |
| Blocking issues | 0 |
| Completion | **94.1%** |

---

## 9. Development Guide

A step-by-step guide to build kitty, reproduce the runtime evidence behind the document, and verify read-only compliance. All commands were executed during this assessment and are copy-pasteable from the repository root.

### 9.1 System Prerequisites

- **OS:** Linux (validated on Ubuntu 25.10). macOS also supported by kitty upstream.
- **Python:** 3.12+ (3.12.3 used; `pyproject.toml` requires `>=3.8`).
- **Compiler & tooling:** `gcc-13` (or clang), Go `1.22`, `pkg-config`.
- **Build libraries** (per `docs/build.rst`): `harfbuzz` (≥ 2.2.0), `freetype`, `fontconfig`, `libpng`, `lcms2`, `libxxhash`; plus X11/OpenGL/xkbcommon/dbus headers for a full GUI build (not needed for the headless OSC 133 investigation).
- **No Python/pip dependencies** are required for the deliverable itself (it is pure markdown).

### 9.2 Environment Setup

```bash
# From the repository root (the directory containing setup.py):
cd /path/to/kitty
git rev-parse --abbrev-ref HEAD    # -> blitzy-82bb3e4d-6d7a-4163-acad-5323f9aa2d1e
```

No virtualenv is required to read or validate the document. To reproduce the runtime evidence you only need the built C extension (below) and `PYTHONPATH` pointing at the repo root.

### 9.3 Build (Canonical)

```bash
# Canonical build — compiles the C sources into kitty/fast_data_types.so and the launchers:
python3 setup.py build

# Environment-specific note (NOT part of the canonical answer):
# If a newer wayland-protocols triggers -Werror=switch in the unrelated GLFW Wayland backend,
# disable -Werror only (does not affect the VT parser):
#   CC=gcc-13 python3 setup.py build --ignore-compiler-warnings
```

### 9.4 Verification

```bash
# 1) Confirm the artifact under observation:
./kitty/launcher/kitty --version
# Expected: kitty 0.35.2 created by Kovid Goyal

# 2) Confirm the headless imports resolve to the freshly built tree:
PYTHONPATH="$PWD" python3 -c "import kitty.fast_data_types; import kitty_tests; from kitty.window import Window; print('imports OK')"
# Expected: imports OK
```

### 9.5 Reproduce the Runtime Evidence

The document's Appendix contains two throwaway observation scripts verbatim. Recreate them under `/tmp` (outside the repo, to preserve read-only cleanliness) and run:

```bash
# Q1 baseline, Q2 sweep, and the NON-CANONICAL harness-proxy block:
PYTHONPATH="$PWD" python3 /tmp/osc133_parsepath.py

# code-99 full-path evidence + the REAL Window.handle_cmd_end canonical table:
PYTHONPATH="$PWD" python3 /tmp/osc133_realwindow.py
```

A minimal one-liner reproduction of the Q1 result (uses the same real harness API — note `create_screen` is a *class method*, so instantiate `Callbacks` + `Screen` directly):

```bash
PYTHONPATH="$PWD" python3 - <<'PY'
from kitty_tests import Callbacks, parse_bytes
from kitty.fast_data_types import Screen
raw = (b'\x1b]133;A\x1b\\' + b'\x1b]133;B\x1b\\'
       + b'\x1b]133;C;cmdline=foo\x1b\\' + b'hello' + b'\x1b]133;D;42\x1b\\')
c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c)
parse_bytes(s, raw)
off = raw.index(b'133;D')
print('total raw length :', len(raw))          # 56
print('offset 133;D     :', off)               # 46
print('offset code-num  :', off + len(b'133;D;'))  # 52
print('rendered screen  :', [str(s.line(i)) for i in range(s.lines)])  # ['hello','','','','']
print('decoded cmdline  :', repr(c.last_cmd_cmdline))  # 'foo'
PY
```

### 9.6 Run the In-Scope Tests

```bash
export LANG=C.UTF-8
./kitty/launcher/kitty +launch test.py --module parser            # 16/16 OK
./kitty/launcher/kitty +launch test.py --module screen            # 36/36 OK
./kitty/launcher/kitty +launch test.py --module shell_integration # 6/6 OK (needs zsh + fish installed)
```

### 9.7 Read the Deliverable & Confirm Read-Only Compliance

```bash
# Read the answer document:
less blitzy/documentation/kitty_815df1e210e0.md

# Confirm the branch adds exactly one file and the tree is clean:
git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD   # -> A blitzy/documentation/kitty_815df1e210e0.md
git status --porcelain                                                  # -> (empty)
```

### 9.8 Troubleshooting

- **`ImportError: cannot import name 'create_screen' from 'kitty_tests'`** — `create_screen` is a *method* on the test class, not a top-level export. Instantiate directly: `c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c)`.
- **Locale / UnicodeDecodeError when running tests** — `export LANG=C.UTF-8` before invoking `test.py`.
- **Build fails with `-Werror=switch` in the Wayland/GLFW backend** — environment-specific; use `CC=gcc-13 python3 setup.py build --ignore-compiler-warnings` (disables `-Werror` only; the VT parser is unaffected).
- **`shell_integration` tests skip zsh/fish cases** — install `zsh` and `fish` so the emit-side tests are exercised rather than skipped.
- **`ModuleNotFoundError: kitty` when running scripts** — ensure `PYTHONPATH="$PWD"` is set and you are in the repository root where `setup.py build` was run.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Canonical build | `python3 setup.py build` |
| Build (env-specific warnings) | `CC=gcc-13 python3 setup.py build --ignore-compiler-warnings` |
| Version check | `./kitty/launcher/kitty --version` |
| Headless import check | `PYTHONPATH="$PWD" python3 -c "import kitty.fast_data_types; import kitty_tests; from kitty.window import Window"` |
| Reproduce Q1/Q2/proxy | `PYTHONPATH="$PWD" python3 /tmp/osc133_parsepath.py` |
| Reproduce code-99 + real-Window table | `PYTHONPATH="$PWD" python3 /tmp/osc133_realwindow.py` |
| Run parser tests | `LANG=C.UTF-8 ./kitty/launcher/kitty +launch test.py --module parser` |
| Run screen tests | `LANG=C.UTF-8 ./kitty/launcher/kitty +launch test.py --module screen` |
| Run shell-integration tests | `LANG=C.UTF-8 ./kitty/launcher/kitty +launch test.py --module shell_integration` |
| Read-only diff check | `git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD` |
| Clean-tree check | `git status --porcelain` |

### Appendix B — Port Reference

*Not applicable.* This project runs no network service and binds no ports — the subject is a terminal control-sequence parser exercised headlessly, and the deliverable is a static markdown document.

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (answer document, 452 lines) |
| `kitty/vt-parser.c` | OSC dispatch — `case 133:` at L536; canonical call at L544; `--dump-commands` hook at L539 |
| `kitty/screen.c` | `shell_prompt_marking` at L2328 — A/C/D cases, exit-status slice at L2351; no `case 'B'` |
| `kitty/window.py` | `decode_cmdline` L225; `handle_cmd_end` L1408 (`int()` L1413, `except→0` L1415, guard L1409-1410, watcher L1419-1420); `cmd_output_marking` L1453 |
| `kitty_tests/__init__.py` | Headless harness — `parse_bytes` L30, `Callbacks` proxy L71 (`suppress` L78-79), `create_screen` L237 |
| `kitty/history.c` | Scrollback prompt-jump `reverse_find` of `\x1b]133;C\x1b\\` at L475 |
| `kitty/client.py` | Remote-control `write_osc(133, payload)` at L251 (non-canonical emitter) |
| `shell-integration/{bash,zsh,fish}/…` | Emit-side scripts (where the real `$?`/`$status`/`$cmd_status` originates) |
| `kitty/fast_data_types.so` | Compiled C extension (VT parser + `Screen`) produced by the build |
| `kitty/launcher/kitty` | Built launcher used for `--version` and `+launch test.py` |

### Appendix D — Technology Versions

| Component | Version |
|-----------|---------|
| kitty | 0.35.2 |
| Python | 3.12.3 (also reproduced on 3.13.7) |
| gcc | 13 |
| Go | 1.22 |
| harfbuzz | ≥ 2.2.0 (10.2.0 in validation env) |
| Base commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Branch HEAD | `7ab4a424` |

### Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `PYTHONPATH` | `"$PWD"` (repo root) | Ensures `import kitty` / `import kitty_tests` resolve to the freshly built tree |
| `LANG` | `C.UTF-8` | Avoids locale/Unicode errors when running `test.py` |
| `CC` | `gcc-13` | (Env-specific) selects the compiler for the build |

*No secrets, API keys, or service credentials are required by this project.*

### Appendix F — Developer Tools Guide

- **Build system:** `setup.py` compiles the C sources into `kitty/fast_data_types.so` and builds Go tooling; see `docs/build.rst` for the full prerequisite list.
- **Test runner:** `test.py` invoked via `./kitty/launcher/kitty +launch test.py --module <name>`; matches tests by method name. Use `--module parser|screen|shell_integration` for the OSC 133 code path.
- **Headless harness:** `kitty_tests.parse_bytes(screen, data)` drives raw bytes through the compiled parser; `Callbacks` provides the callback surface (note: its malformed-input handling is non-canonical relative to the real `Window`).
- **Docs (upstream):** kitty's own docs are reStructuredText compiled by Sphinx under `docs/`; the Blitzy deliverable is standalone markdown under `blitzy/documentation/` and is not part of the Sphinx build.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **OSC 133** | Operating System Command 133 — the FinalTerm/iTerm2 "semantic prompt" shell-integration protocol for marking command boundaries. |
| **FinalTerm markers** | `A` = prompt start; `B` = command/input start; `C` = command output start (FTCS_COMMAND_EXECUTED); `D [;<code>]` = command finished, optional exit code. |
| **ST (String Terminator)** | Ends an OSC string; `ESC \` (`\x1b\\`) or `BEL` (`\a`). The probe uses `ESC \`. |
| **`shell_prompt_marking`** | kitty's C handler (`screen.c:L2328`) that switches on the marker byte; handles `A`/`C`/`D`, has no `B` case. |
| **`last_cmd_exit_status`** | The integer exit status recorded by `Window.handle_cmd_end`; default `0`; set via `int(exit_status)` with an `except→0` fallback. |
| **Canonical vs non-canonical** | Canonical = the real production `Window` code path (source of truth). Non-canonical = test-harness proxy / debug hooks / remote emitter (labeled, not used as source of truth). |
| **Raw stream vs rendered screen** | The raw child bytes (OSC sequences present) vs the screen cell buffer (OSC consumed as control data, never drawn). |
| **AAP** | Agent Action Plan — the primary directive defining project scope and requirements. |

---

*This Blitzy Project Guide was generated from the Agent Action Plan, the Final Validator's autonomous validation logs, and direct repository/runtime inspection. All hours are AAP-scoped (PA1); all figures are consistent across Sections 1.2, 2.1, 2.2, and 7.*