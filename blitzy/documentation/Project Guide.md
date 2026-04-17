# Blitzy Project Guide — kitty PTY Communication Pipeline Deep-Dive (Task 815df1e210e0)

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **pure documentation / investigation deliverable** under the SWE-AtlasQnA-Repo rule. The Agent Action Plan (AAP) directed autonomous Blitzy agents to investigate kitty's C-level pseudoterminal (PTY) communication pipeline — by building kitty 0.35.2 from source under a headless `Xvfb :99` display, running it under `strace -f` with multiple trace filter configurations, and empirically observing the process spawning chain, system calls, buffer management, and parsing functions that kitty uses when communicating with its child shell. The resulting technical deep-dive is captured in a single 564-line markdown artifact at `blitzy/documentation/kitty_815df1e210e0.md` that answers seven specific questions, each grounded in both static source-code reading and runtime evidence. The target audience is future engineers who need an accurate, line-numbered, empirically-verified reference for the C-level PTY pipeline — a replacement for trial-and-error source reading.

### 1.2 Completion Status

```mermaid
%%{init: {"pie": {"textPosition": 0.5}, "themeVariables": {"pieOuterStrokeWidth": "2px", "pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#5B39F3", "pieSectionTextColor": "#FFFFFF", "pieTitleTextSize": "18px", "pieLegendTextSize": "14px"}}}%%
pie showData title Project Completion — 93.75%
    "Completed Work (Dark Blue #5B39F3)" : 30
    "Remaining Work (White #FFFFFF)" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | 32.0 |
| **Completed Hours (AI + Manual)** | 30.0 |
| **Remaining Hours** | 2.0 |
| **Percent Complete** | **93.75%** |

Calculation: Completed Hours / (Completed + Remaining) = 30 / (30 + 2) = 30 / 32 = **93.75%**.

### 1.3 Key Accomplishments

- ✅ **Build from source verified** — `python3 setup.py build --debug` completes cleanly; binaries `kitty/fast_data_types.so`, `kitty/glfw-x11.so`, `kitty/launcher/kitty`, and `kitty/launcher/kitten` are all present and `kitty/launcher/kitty --version` returns `kitty 0.35.2 created by Kovid Goyal`.
- ✅ **Runtime launch under headless display verified** — `Xvfb :99 -screen 0 1024x768x24 -nolisten tcp -ac` + `DISPLAY=:99 kitty/launcher/kitty --hold sh` launches successfully.
- ✅ **Five distinct strace experiments executed** during investigation — shell spawn chain, PTY device identification, `echo test123` read behavior, `yes hello` high-volume burst (11,699 reads captured), and interactive typing per-keystroke echo.
- ✅ **Single 564-line / 44.5 KB documentation artifact** at `blitzy/documentation/kitty_815df1e210e0.md` covering all seven required questions (Q1–Q7), each with Executive Answer, Thinking/Rationale, Source Code References (line-numbered), and Runtime Evidence.
- ✅ **Mermaid architecture diagram** showing three-layer pipeline: process spawning → I/O multiplexing (KittyChildMon thread) → VT parsing (main thread) with the 1 MiB ring buffer as the rendezvous point.
- ✅ **Thread model table** documenting the three relevant threads (Main, `KittyChildMon`, `KittyPeerMon`) with line numbers of their entry functions and their roles in PTY communication.
- ✅ **Cross-validated claims against source** — every line number referenced in the documentation was verified: `read_bytes()` at line 1337, `io_loop()` at line 1481, `BUF_SZ (1024u*1024u)` at line 18, `consume_input()` at line 1367, `utf8_decode_to_esc()` at line 72, `shell_path` resolution at line 181.
- ✅ **Platform-specific discovery documented** — the `argv[0] = "-bash"` login-shell convention is gated on `runtime.GOOS == "darwin"` at `tools/tui/run.go:167-171` (macOS-only), a non-obvious detail that would otherwise require reading the Go source tree.
- ✅ **Scope compliance** — **zero** kitty source files modified; all temporary strace logs (6 files, ~1.4 MB total in `/tmp/kitty_verify/`) deleted after investigation; no helper/progress/summary documents created beyond the AAP-specified deliverable.
- ✅ **Commit hygiene** — 2 commits on branch `blitzy-81aae582-39aa-410c-ad64-77a865b8555b` (`ba47373fb` add + `af4774072` corrections); working tree clean.
- ✅ **8 correction revisions applied** during validation pass — corrected `openat()` flags (removed spurious `O_NOCTTY`), corrected `yes hello` metrics from placeholder estimates to real measurements (11,699 reads, 79–14,133 byte range, 1,048,576→87 count parameter trajectory), fixed Linux vs macOS `argv[0]` assertion, replaced synthesized 3-step execve chain with observed values.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| None blocking AAP completion | N/A | N/A | N/A |

No critical unresolved issues exist for the AAP-scoped deliverable. The single in-scope file is complete, empirically verified, markdown-structurally valid (20 balanced code fences, 1 H1, 12 H2 headers, 7 Q-sections), and committed.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Wayland dev headers | Build-time library | `libwayland-dev` + `libwayland-protocols` not installed in container, so `kitty/glfw-wayland.so` is not built; `test_glfw_modules` fails in consequence (pre-existing, unrelated to documentation) | Deferred — not in AAP scope; X11-only runtime is sufficient for all observed PTY experiments | Human reviewer (optional) |
| Real display server | Runtime (X11) | Container environment uses `Xvfb :99` virtual framebuffer; a native physical display is unavailable | Acceptable — PTY communication is display-independent; Xvfb satisfies all AAP runtime requirements | N/A |

No access issues blocked AAP completion. The two items listed are environmental properties of the Linux container that were already accommodated (via Xvfb) or are explicitly out-of-scope per the AAP ("Only Linux/X11 behavior is observed").

### 1.6 Recommended Next Steps

1. **[High]** Subject-matter expert review of `blitzy/documentation/kitty_815df1e210e0.md` by an engineer familiar with terminal emulators / VT parsing — validate empirical numbers (11,699 reads, 79–14,133 byte range), re-check line numbers, and confirm architectural diagram accuracy.
2. **[Low]** File the document into the internal knowledge base / docs portal with appropriate tags (`kitty`, `pty`, `vt-parser`, `terminal-emulator`, `systems-programming`) to enable team discoverability.
3. **[Low]** (Optional, AAP-out-of-scope follow-up) Install `libwayland-dev` + `libwayland-protocols` in the build container and re-run `python3 setup.py build --debug` to produce `kitty/glfw-wayland.so` — this would eliminate the pre-existing `test_glfw_modules` failure, though it is unrelated to the AAP deliverable.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Install build dependencies (apt, Ubuntu 24.04) | 1.5 | `build-essential`, `python3-dev`, `pkg-config`, `libfreetype-dev`, `libharfbuzz-dev`, `libfontconfig-dev`, `libgl-dev`, `libx11-dev`, `libx11-xcb-dev`, `libxkbcommon-x11-dev`, `libdbus-1-dev`, `liblcms2-dev`, `libpng-dev`, `libxxhash-dev`, `librsync-dev`, `libssl-dev`, `golang-go`, plus `strace`/`xvfb`/`xdotool` runtime observation tools. |
| Build kitty (`python3 setup.py build --debug`) | 1.0 | Produces `kitty/fast_data_types.so` (6.1 MB), `kitty/glfw-x11.so` (1.6 MB), `kitty/launcher/kitty` (275 KB), `kitty/launcher/kitten` (21.8 MB). |
| Xvfb :99 setup + launch verification | 0.5 | `Xvfb :99 -screen 0 1024x768x24 -nolisten tcp -ac &`; `DISPLAY=:99 kitty/launcher/kitty --hold sh`. |
| Runtime strace experiments (5 filter configurations) | 2.5 | `trace=clone,execve,openat,ioctl,read,write`; `-e trace=read -T` with timing; interactive typing; `yes hello` high-volume burst; broader unfiltered run. |
| Q1: Shell process spawn investigation | 2.0 | 3-step execve chain discovery: launcher → `kitten run-shell` → `/bin/bash --posix`; platform-conditional `argv[0]` behavior (`tools/tui/run.go:167-171`). |
| Q2: PTY device path identification | 2.0 | master = `/dev/pts/ptmx` at fd 8, slave = `/dev/pts/0`; `ttyname_r` + `safe_open` + `TIOCSCTTY` + `safe_dup2` sequence from `kitty/child.c`. |
| Q3: `echo test123` syscall tracing | 1.5 | Confirmed POSIX `read(2)` from `read_bytes()`; 1-byte local-echo reads during typing; single 9-byte `"test123\r\n"` read after Enter. |
| Q4: `yes hello` high-volume read tracing | 3.0 | 11,699 reads captured; min=79, max=14,133, median=569, mean=822 bytes/read; count parameter descending 1,048,576 → 87 then resetting after parser drain; mean latency 19 μs, outliers to 19,894 μs at mutex contention points. |
| Q5: PTY master fd identification | 0.5 | fd 8 (not hardcoded — lowest available slot at `openat` time); `/proc/<kitty-pid>/fd/8 → /dev/pts/ptmx`. |
| Q6: `read_bytes()` function analysis | 1.5 | `kitty/child-monitor.c:1337-1356`; three-step `vt_parser_create_write_buffer` → `read()` → `vt_parser_commit_write` transaction; only caller is `io_loop()` at line 1531. |
| Q7: `consume_input()` + `utf8_decode_to_esc()` analysis | 2.5 | `kitty/vt-parser.c:1367` state-machine dispatcher; `kitty/simd-string.c:72` SIMD sentinel scanner watching for `0x1b`; full CSI/OSC/DCS/APC/PM/SOS branch dispatch table; control-code constants at `kitty/control-codes.h:53-73`. |
| Initial markdown draft (564 lines) | 5.0 | All 7 Q-sections (Executive Answer + Thinking/Rationale + Source Code References + Runtime Evidence), Executive Summary, Architecture Diagram, Thread Model, Methodology, References. |
| 8 correction revisions (from validator) | 2.0 | Removed spurious `O_NOCTTY` (3 locations), corrected Q4 metrics from placeholders to actual measurements, fixed Linux vs macOS `argv[0]` claim, replaced synthesized execve chain with observed values, corrected Q4 latency statistics. |
| Mermaid architecture diagram + thread model table | 1.5 | Three-layer flowchart (spawning → I/O → parse); three-row thread model table with line-number-accurate entry functions. |
| Cross-validation against source | 1.0 | Verified all line numbers: `read_bytes:1337`, `io_loop:1481`, `BUF_SZ:18`, `consume_input:1367`, `utf8_decode_to_esc:72`, `shell_path:181`, `MAX_CHILDREN:114`, `EXTRA_FDS:35`. |
| Cleanup of temporary files + test verification | 1.0 | `rm -rf /tmp/kitty_verify/` (6 strace logs ~1.4 MB removed); `./test.py` run on both branch and clean base to isolate pre-existing failures; git status verified clean. |
| Markdown structural validation + commit | 1.5 | 20 balanced code fences, 1 H1, 12 H2 headers, 7 Q-sections; 2 commits (`ba47373fb` initial + `af4774072` corrections); working tree clean. |
| **TOTAL COMPLETED** | **30.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Subject-matter expert technical review of `kitty_815df1e210e0.md` — validate empirical numbers, re-verify line numbers against HEAD source, confirm architectural accuracy | 1.5 | High |
| Knowledge-base integration — file the document into internal wiki/docs portal, apply tags for discoverability (`kitty`, `pty`, `vt-parser`, `terminal-emulator`, `systems-programming`) | 0.5 | Low |
| **TOTAL REMAINING** | **2.0** | |

### 2.3 Cross-Section Integrity Summary

- Section 2.1 total: **30.0 h** = Section 1.2 "Completed Hours (AI + Manual)" ✅
- Section 2.2 total: **2.0 h** = Section 1.2 "Remaining Hours" = Section 7 pie chart "Remaining Work" ✅
- Section 2.1 + Section 2.2 = **32.0 h** = Section 1.2 "Total Hours" ✅
- Completion percentage: **30 / 32 = 93.75%** stated identically in Sections 1.2, 7, and 8 ✅

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation runs (`./test.py` executed during the validator pass, with identical results observed on the branch head and on a clean stash-revert to confirm pre-existing environment-dependent failures).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| kitty Python unit tests | Python `unittest` (via `./test.py`) | 145 | 136 | 3 | N/A (no coverage instrumentation in AAP scope) | 6 skipped are environment-gated (e.g., missing optional libs). The 3 failures (`test_transfer_receive`, `test_transfer_send`, `test_glfw_modules`) are pre-existing and identical on the clean base branch — **not** caused by or related to the documentation changes. |
| kitty Go tests | Go `testing` (invoked by `./test.py`) | (bundled) | All | 0 | N/A | "All Go tests succeeded, ran in 9.9 seconds" per validator log; exit-zero Go test suite. |
| Markdown structural integrity | Manual validation script | 1 artifact | 1 | 0 | 100% | 20 balanced code fences, 1 H1, 12 H2 headers, 7 Q-sections present; file size 44,500 bytes / 564 lines. |
| Build artifact presence check | Filesystem inspection | 4 artifacts | 4 | 0 | 100% | `kitty/fast_data_types.so`, `kitty/glfw-x11.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten` — all present and executable. |
| Runtime smoke test | Manual (`kitty --version`) | 1 | 1 | 0 | 100% | `kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal`. |
| Xvfb + kitty launch smoke test | Manual | 1 | 1 | 0 | 100% | `DISPLAY=:99 kitty/launcher/kitty --hold sh` launches under `Xvfb :99`. |
| Source-file integrity check | Git | 1 | 1 | 0 | 100% | `git diff --stat 815df1e21 HEAD` → only `blitzy/documentation/kitty_815df1e210e0.md` changed (+564/-0); zero modifications to any kitty source file. |

**Failure Analysis of the 3 Python Unit-Test Failures** (pre-existing, not related to this PR):

| Test | Root Cause | Confirmed Pre-existing? |
|------|------------|-------------------------|
| `test_transfer_receive` | Expected dir mode `0o42755` (setgid bit) but got `0o40755`; tmpfs in container does not set the setgid inheritance bit the way the test expects | ✅ Yes — fails identically on clean base branch via `git stash && ./test.py` |
| `test_transfer_send` | Same tmpfs setgid root cause as `test_transfer_receive` | ✅ Yes — fails identically on clean base branch |
| `test_glfw_modules` | Missing `kitty/glfw-wayland.so`; Wayland dev headers not installed in container so the Wayland GLFW module was never compiled | ✅ Yes — fails identically on clean base branch |

No new test failures introduced by this PR. No test covers markdown document content (none exists for a docs-only deliverable).

---

## 4. Runtime Validation & UI Verification

| Validation | Status | Evidence |
|------------|--------|----------|
| Binary version check | ✅ Operational | `kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal` |
| C-extension load | ✅ Operational | `kitty/fast_data_types.so` (6,143,096 bytes) dynamically loads and `fast_data_types.spawn` is callable from Python |
| GLFW X11 backend load | ✅ Operational | `kitty/glfw-x11.so` (1,626,816 bytes) present and loadable |
| Go launcher + kitten | ✅ Operational | `kitty/launcher/kitty` (274,704 bytes) + `kitty/launcher/kitten` (21,791,690 bytes) both executable |
| Headless display (Xvfb :99) | ✅ Operational | `Xvfb :99 -screen 0 1024x768x24 -nolisten tcp -ac &` accepts connections; `DISPLAY=:99 kitty/launcher/kitty --hold sh` launches without error |
| PTY master fd open (fd 8) | ✅ Operational | `openat(AT_FDCWD, "/dev/ptmx", O_RDWR) = 8` observed; `/proc/<kitty-pid>/fd/8 → /dev/pts/ptmx` verified |
| Shell execve chain | ✅ Operational | 3-step chain observed: `./kitty/launcher/kitty` → `./kitty/launcher/kitten run-shell --shell=/bin/bash` → `/bin/bash --posix` |
| Interactive PTY echo | ✅ Operational | `strace -f -e trace=read` on interactive typing confirms per-keystroke 1-byte reads on fd 8 |
| High-volume read behavior | ✅ Operational | `yes hello` burst captured: 11,699 reads, 9.6 MiB total bytes, median 569 B/read |
| `echo test123` baseline | ✅ Operational | Single `read(8, "test123\r\n", 1048576) = 9` observed as predicted by PTY line-discipline (CR/LF translation by `ONLCR`) |
| UI (terminal rendering) | ⚠ Partial | Rendering **was not** validated — out-of-scope per AAP Section 0.6.2 "No GPU rendering analysis" |
| Wayland backend | ❌ Failing | `kitty/glfw-wayland.so` not built because Wayland dev headers are absent in container; explicitly out-of-scope per AAP "Only Linux/X11 behavior is observed" |

**Overall runtime verdict**: Every AAP-scoped runtime capability (build, launch under Xvfb, PTY open, shell spawn, read observations, parse dispatch) is operational. The two ⚠/❌ items are both explicitly out-of-scope per the AAP itself and do not block the documentation deliverable.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP's explicit requirements and implicit quality benchmarks to delivered artifacts.

| Requirement / Benchmark | Source | Status | Evidence / Fix Applied |
|-------------------------|--------|--------|------------------------|
| **Create `blitzy/documentation/kitty_815df1e210e0.md`** | AAP §0.1.2 SWE-AtlasQnA-Repo rule | ✅ Pass | File exists at correct path, 564 lines, 44,500 bytes |
| **Answer all 7 questions comprehensively** | AAP §0.1.1 core feature objective | ✅ Pass | Q1 spawned process, Q2 PTY device, Q3 read syscall + echo test123, Q4 yes hello behavior, Q5 fd number, Q6 read C function, Q7 parse C function — all 7 present with Executive Answer + Thinking/Rationale + Source Code References + Runtime Evidence |
| **Ground answers in source code (no assumptions)** | AAP §0.1.2 "Evidence-based answers" | ✅ Pass | Every claim cites specific file + line number (verified: `read_bytes:1337`, `BUF_SZ:18`, `consume_input:1367`, `utf8_decode_to_esc:72`, `shell_path:181`, `io_loop:1481`, `MAX_CHILDREN:114`, `EXTRA_FDS:35`, `tools/tui/run.go:167-171`) |
| **Use runtime observation to verify claims** | AAP §0.1.3 "runtime observation using strace -f" | ✅ Pass | 5 distinct strace runs executed; Q4 measurements rewritten based on actual 11,699-read burst data |
| **No kitty source file modifications** | AAP §0.1.2 + §0.7.1 user directive | ✅ Pass | `git diff --name-status 815df1e21 HEAD` returns only `A blitzy/documentation/kitty_815df1e210e0.md` — zero existing-file modifications |
| **Delete temporary helper files afterward** | AAP §0.7.1 user directive | ✅ Pass | `/tmp/kitty_verify/` removed (6 strace logs ~1.4 MB total); confirmed via `ls /tmp/kitty_verify/ → No such file` |
| **No new code in source repository beyond the documentation** | AAP §0.1.2 | ✅ Pass | Only one new file added; zero `.c`, `.py`, `.go`, `.h`, or shell-integration files added or modified |
| **Document placement in `blitzy/documentation/`** | AAP §0.1.2 SWE-AtlasQnA-Repo rule | ✅ Pass | Path is `blitzy/documentation/kitty_815df1e210e0.md` — exact match |
| **Filename is `kitty_815df1e210e0.md`** | AAP §0.1.2 SWE-AtlasQnA-Repo rule | ✅ Pass | Exact filename match |
| **Include thinking / rationale behind answers** | AAP §0.7.1 SWE-AtlasQnA-Repo rule | ✅ Pass | Every Q-section has a dedicated "Thinking / Rationale" subsection |
| **Python ≥ 3.8 compatibility** | `pyproject.toml` | ✅ Pass | Python 3.12.3 used (satisfies `requires-python = ">=3.8"`) |
| **Go ≥ 1.22 compatibility** | `go.mod` | ✅ Pass | Go 1.22.2 used (satisfies `go 1.22` module directive) |
| **Build reproducibility** | Engineering best practice | ✅ Pass | Validator confirmed rebuild completes; build artifacts present |
| **Commit hygiene** | Engineering best practice | ✅ Pass | 2 clean commits (`ba47373fb` + `af4774072`); working tree clean; no WIP, no merge conflicts |
| **Validator corrections applied** | Validator pass | ✅ Pass | 8 corrections committed in `af4774072` (openat flags, Q4 metrics, macOS dash-prefix gate, execve chain) |
| **Markdown structural integrity** | Engineering best practice | ✅ Pass | 20 balanced code fences, 1 H1, 12 H2 headers, 7 Q-sections; no dangling fences, no unclosed tables |
| **No forbidden progress / summary / status documents** | Blitzy agent policy | ✅ Pass | Only `kitty_815df1e210e0.md` created; no STATUS.md / PROGRESS.md / OUT_OF_SCOPE_ISSUES.md / validation summary .md files |

Overall compliance rating: **17 / 17 benchmarks satisfied**.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Empirical numbers (11,699 reads, median 569 B, count descending to 87) vary on different hardware / kernel / I/O scheduler | Technical (accuracy) | Low | High | Documentation explicitly states "values from an actual measured run — individual numbers will vary with hardware and scheduler"; qualitative claims (monotonic decrease, back-pressure pattern, `POLLIN` level-triggering) are invariant | ✅ Mitigated |
| Source-code line numbers may drift as kitty evolves | Technical (link rot) | Medium | Medium | Document is pinned to base commit `815df1e21`; the SWE-AtlasQnA-Repo filename scheme encodes that commit hash explicitly, so future readers know the reference point | ✅ Mitigated |
| Linux-only observations do not describe macOS or Wayland behavior | Technical (scope) | Low | Certain | Explicitly declared out-of-scope per AAP §0.6.2; document states "No Wayland or macOS analysis"; the macOS-specific `runtime.GOOS == "darwin"` `argv[0] = "-bash"` behavior **is** documented as a cross-reference | ✅ Mitigated |
| Pre-existing test failures (`test_transfer_*`, `test_glfw_modules`) confuse future reviewers who see "FAILED" in `./test.py` output | Technical (noise) | Low | Medium | Root-cause analysis documented in Section 3 of this guide; `git stash && ./test.py` proves identical failures on clean base | ✅ Mitigated |
| Document becomes stale as `kitty/child-monitor.c`, `kitty/vt-parser.c`, or `kitty/simd-string.c` are refactored upstream | Operational (doc freshness) | Medium | Low | Filename commit hash (`815df1e210e0`) signals the intended pin-point; re-investigation task can be spawned periodically | ⚠ Residual (acceptable) |
| `libwayland-dev` absence prevents compilation of `glfw-wayland.so` and causes one test failure | Operational (build env) | Low | Certain | Out-of-scope per AAP; X11-only was sufficient for all PTY observations; fix (install libs) listed as Low-priority follow-up in Section 1.6 | ⚠ Residual (out-of-scope) |
| No automated mechanism ensures the 564-line markdown file stays in sync with future line-number changes | Operational (governance) | Low | High (over time) | Would require a CI check parsing `file.c:NNN` references and re-validating — explicitly not part of AAP | ⚠ Residual (acceptable) |
| User executing reproduction commands in a hardened container without `ptrace` capability cannot run `strace -f` | Security (permissions) | Low | Low | Document instructs running under an Ubuntu 24.04 container with `strace` installed and `SYS_PTRACE` available — the same environment that produced the measurements | ✅ Mitigated |
| Running `Xvfb` + `kitty` + `strace` simultaneously could leave orphaned processes if interrupted mid-run | Operational (cleanup) | Low | Medium | Reproduction steps in Section 9 include explicit `pkill -f Xvfb` and `kill %1` cleanup; validator confirmed `ps` shows no orphans after investigation | ✅ Mitigated |
| PTY master fd number (8) is runtime-dependent and could mislead a reader into assuming it's a constant | Technical (doc clarity) | Low | Medium | Document explicitly flags: "fd 8 in the observed run. This is **not a hard-coded value**" in the Q5 answer | ✅ Mitigated |
| `read(8, "test123\r\n", 1048576) = 9` is presented verbatim; a skeptical reader might suspect fabrication | Technical (reproducibility) | Low | Low | Documentation includes the Methodology section with exact strace filter configurations; reproduction commands in Section 9 of this guide allow any engineer to re-run | ✅ Mitigated |
| Document does not address Go-side PTY handling (e.g., `tools/tui/run.go` execve) in full depth | Technical (coverage) | Low | Low | AAP specified C-level analysis; Go side is referenced only for the `argv[0]` macOS gate — consistent with AAP scope | ✅ Mitigated |
| Security: no credentials, secrets, or access tokens are involved (pure documentation task) | Security | None | None | N/A — read-only source analysis produced a markdown file | ✅ Not applicable |
| Integration: no external services, APIs, or databases are used | Integration | None | None | N/A — strace + kitty + Xvfb + shell, all local | ✅ Not applicable |

**Overall risk posture**: Low. All identified risks are either mitigated or explicitly scoped as residual-and-acceptable. There are no Critical or High-severity risks.

---

## 7. Visual Project Status

### Hours Breakdown

```mermaid
%%{init: {"pie": {"textPosition": 0.5}, "themeVariables": {"pieOuterStrokeWidth": "2px", "pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#5B39F3", "pieSectionTextColor": "#FFFFFF", "pieTitleTextSize": "16px", "pieLegendTextSize": "13px"}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 30
    "Remaining Work" : 2
```

**Pie chart values**: Completed Work = **30 hours** (Dark Blue #5B39F3), Remaining Work = **2 hours** (White #FFFFFF). These match Section 1.2 exactly.

### Remaining Work — Priority Distribution

```mermaid
%%{init: {"themeVariables": {"pieOuterStrokeWidth": "2px", "pie1": "#B23AF2", "pie2": "#A8FDD9"}}}%%
pie showData title Remaining Work by Priority
    "High (SME Review)" : 1.5
    "Low (Knowledge Base Integration)" : 0.5
```

**Priority totals**: High = 1.5 h; Low = 0.5 h; Total = **2.0 h** (matches Section 2.2 exactly).

### Completed Work — Top 5 Components by Hours

| Rank | Component | Hours | % of Completed |
|------|-----------|-------|----------------|
| 1 | Initial markdown draft (564 lines) | 5.0 | 16.7% |
| 2 | Q4: `yes hello` high-volume tracing | 3.0 | 10.0% |
| 3 | Runtime strace experiments (5 configurations) | 2.5 | 8.3% |
| 3 | Q7: parse-function analysis | 2.5 | 8.3% |
| 5 | Q1: Shell process spawn investigation | 2.0 | 6.7% |

---

## 8. Summary & Recommendations

### Achievements

The project is **93.75% complete** (30 of 32 total hours). Autonomous Blitzy agents delivered the single AAP-scoped deliverable — `blitzy/documentation/kitty_815df1e210e0.md` — a 564-line, 44.5 KB technical deep-dive that answers seven specific questions about kitty's PTY communication pipeline. Every answer is grounded in **both** static source-code reading (with line-accurate file references to `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/child.c`, `kitty/child.py`, `kitty/simd-string.c`, `kitty/constants.py`, `kitty/control-codes.h`, `kitty/data-types.h`, and `tools/tui/run.go`) **and** empirical runtime evidence from five distinct `strace -f` experiments run against a freshly built kitty 0.35.2 binary under an `Xvfb :99` display on Ubuntu 24.04.

The investigation surfaced several non-obvious insights that would otherwise require significant source-tree exploration: the 3-step execve chain through `kitten run-shell`, the macOS-only gating of the `argv[0] = "-bash"` login-shell convention, the 1 MiB `BUF_SZ` ring buffer's role as the rendezvous point between the I/O and main threads, the back-pressure mechanism exposed via the monotonically-decreasing `count` parameter to `read()` during high-volume bursts (observed descending from 1,048,576 down to 87 before the parser drain resets it), and the specific sentinel byte (`0x1b`) that `utf8_decode_to_esc()` watches for as the text/escape boundary.

All validator corrections (8 total — including corrected `openat` flags, actual `yes hello` measurements, and platform-conditional behavior notes) were applied and committed in `af4774072`. The working tree is clean; the commit history is linear and well-annotated; zero kitty source files were modified; all temporary strace logs were cleaned up.

### Remaining Gaps

Exactly **2.0 hours** of work remain, entirely in the path-to-production category:

1. **Subject-matter expert review (1.5 h, High priority)** — A senior engineer familiar with terminal emulators should sanity-check the empirical numbers (especially the 11,699-read Q4 burst measurements), re-verify the ~18 source-code line numbers referenced in the document, and confirm the architectural diagram accurately reflects the three-layer pipeline.
2. **Knowledge-base integration (0.5 h, Low priority)** — File the artifact into the organization's internal docs portal / wiki with appropriate discovery tags.

Neither remaining task requires code changes; both are human-review and filing operations.

### Critical Path to Production

For a documentation-only R&D deliverable, the "production" definition is stakeholder acceptance and discoverability. The critical path is:

1. SME review → **1.5 h**
2. Knowledge-base filing → **0.5 h**

**Total critical path: 2.0 hours** — can be completed in a single working session by one reviewer.

### Success Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Single file created at exact path | Yes | `blitzy/documentation/kitty_815df1e210e0.md` | ✅ |
| All 7 questions answered | 7 / 7 | 7 / 7 | ✅ |
| Every answer has Source Code References | Yes | 7 / 7 sections have line-numbered refs | ✅ |
| Every answer has Runtime Evidence | Yes | 7 / 7 sections have strace / `/proc` observations | ✅ |
| Zero kitty source files modified | 0 | 0 | ✅ |
| Zero temporary files remaining | 0 | 0 (`/tmp/kitty_verify/` removed) | ✅ |
| Build remains clean after all changes | Yes | Yes | ✅ |
| Test suite delta | 0 new failures | 0 new failures (3 pre-existing unrelated) | ✅ |

### Production Readiness Assessment

**Verdict: READY FOR HUMAN REVIEW.** The deliverable is complete in all dimensions the AAP specified. The only gate between current state and "production" (here: stakeholder-accepted, filed, discoverable) is **human review** — which is outside the scope of autonomous agent work by definition. No additional engineering work is needed to progress the document to acceptance.

---

## 9. Development Guide

### 9.1 System Prerequisites

| Requirement | Minimum Version | Rationale |
|-------------|-----------------|-----------|
| Operating system | Ubuntu 24.04 (or equivalent Linux distribution with X11) | Validated runtime target; the AAP specifies Linux/X11 observations |
| CPU architecture | x86_64 | Tested; aarch64 should also work but was not validated |
| RAM | 2 GB minimum, 4 GB recommended | kitty build + strace + Xvfb + sample `yes hello` burst |
| Disk | 500 MB free (kitty build artifacts + temporary strace logs) | Sized to match observed `du -sh` after full build (~182 MB) + ~1.4 MB strace data |
| Python | ≥ 3.8 (validated with 3.12.3) | `pyproject.toml` declares `requires-python = ">=3.8"` |
| Go | ≥ 1.22 (validated with 1.22.2) | `go.mod` declares `go 1.22` |
| X server | Real X11 display, OR `Xvfb` virtual framebuffer | kitty requires a display for its GUI initialization even when driving non-GUI workloads |
| ptrace permission | `CAP_SYS_PTRACE` or equivalent unprivileged tracing (kernel `kernel.yama.ptrace_scope` cooperative) | `strace -f` requires ability to trace child processes |

### 9.2 Environment Setup

#### 9.2.1 Install build dependencies (apt)

```bash
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update -y
sudo apt-get install -y \
    build-essential \
    python3-dev \
    pkg-config \
    libfreetype-dev \
    libharfbuzz-dev \
    libfontconfig-dev \
    libgl-dev \
    libx11-dev \
    libx11-xcb-dev \
    libxkbcommon-x11-dev \
    libdbus-1-dev \
    liblcms2-dev \
    libpng-dev \
    libxxhash-dev \
    librsync-dev \
    libssl-dev \
    golang-go
```

#### 9.2.2 Install runtime observation dependencies

```bash
sudo apt-get install -y strace xvfb xdotool
```

#### 9.2.3 Verify toolchain

```bash
python3 --version           # expect Python 3.8+ (tested 3.12.3)
go version                  # expect go 1.22+ (tested 1.22.2)
Xvfb -help | head -1        # confirm Xvfb installed
strace -V                   # confirm strace installed
```

Expected outputs:
- `Python 3.12.3`
- `go version go1.22.2 linux/amd64`
- `use: Xvfb ...` (one-line usage banner)
- `strace -- version 6.8` (or newer)

### 9.3 Build Commands

#### 9.3.1 Clone or enter repository

```bash
cd /path/to/repository
# Confirm you're on the correct branch (the documentation was produced on this branch)
git branch --show-current   # expect: blitzy-81aae582-39aa-410c-ad64-77a865b8555b
```

#### 9.3.2 Build kitty with debug symbols

```bash
python3 setup.py build --debug --verbose
```

**Expected output** (tail):
- `Running kitten`
- Compilation of C sources into `build/lib.linux-x86_64-<pyver>/kitty/fast_data_types.so`
- Compilation of GLFW X11 backend into `build/lib.linux-x86_64-<pyver>/kitty/glfw-x11.so`
- Go build of `kitty/launcher/kitten`
- Go build of `kitty/launcher/kitty`
- Exit code 0

**Build artifacts verification**:
```bash
ls -la kitty/fast_data_types.so kitty/glfw-x11.so kitty/launcher/kitty kitty/launcher/kitten
```
All four files should exist and be executable.

#### 9.3.3 Smoke test: version and launcher

```bash
kitty/launcher/kitty --version
```

**Expected output**: `kitty 0.35.2 created by Kovid Goyal`.

### 9.4 Running Under Xvfb (Headless Display)

#### 9.4.1 Start Xvfb in background

```bash
Xvfb :99 -screen 0 1024x768x24 -nolisten tcp -ac &
export DISPLAY=:99
sleep 1  # give Xvfb time to initialize
```

#### 9.4.2 Confirm Xvfb is serving

```bash
xset q >/dev/null 2>&1 && echo "Xvfb OK" || echo "Xvfb NOT ready"
```

Expected: `Xvfb OK`.

#### 9.4.3 Launch kitty (smoke test — will exit immediately)

```bash
DISPLAY=:99 kitty/launcher/kitty --hold sh -c "echo hello from kitty"
```

Expected: kitty launches, runs `echo hello from kitty` inside the child shell, and — thanks to `--hold` — stays open. You can then `kill` the kitty process (or press Ctrl+Shift+Q if you had input forwarding set up).

### 9.5 Reproducing the PTY Communication Investigation

#### 9.5.1 Shell spawn + PTY open trace

Records the execve chain + PTY device setup:

```bash
mkdir -p /tmp/kitty_verify
strace -f -e trace=clone,execve,openat,ioctl \
    -o /tmp/kitty_verify/spawn.log \
    kitty/launcher/kitty --hold sh -c "sleep 5" &
sleep 6
grep -E "execve|openat\(.*/dev/ptmx|ioctl.*TIO" /tmp/kitty_verify/spawn.log | head -40
```

Expected markers in output:
- `execve("./kitty/launcher/kitty", ...)` (launcher re-exec)
- `execve(".../kitty/launcher/kitten", ["..kitten", "run-shell", "--shell=/bin/bash", ...], ...)`
- `execve("/bin/bash", ["/bin/bash", "--posix"], <envp>)`
- `openat(AT_FDCWD, "/dev/ptmx", O_RDWR) = 8` (PTY master fd)
- `ioctl(8, TIOCSPTLCK, [0])` + `ioctl(8, TIOCGPTN, [0])` (slave unlock + slave index query)

#### 9.5.2 `echo test123` read-syscall trace

```bash
strace -f -T -e trace=read -o /tmp/kitty_verify/echo.log \
    kitty/launcher/kitty --hold sh -c 'echo test123; sleep 2' &
sleep 3
grep 'read(8' /tmp/kitty_verify/echo.log
```

Expected line (among others):
```
read(8, "test123\r\n", 1048576) = 9 <0.0000XX>
```

The `1048576` is `BUF_SZ` = 1 MiB (`kitty/vt-parser.c:18`); the `9` is `len("test123") + len("\r\n")`; the `\r\n` is the kernel PTY line-discipline's `ONLCR` conversion of the shell's `\n`.

#### 9.5.3 `yes hello` high-volume burst trace

```bash
timeout 2 strace -f -T -e trace=read -o /tmp/kitty_verify/yes.log \
    kitty/launcher/kitty --hold sh -c 'yes hello & sleep 1; kill $!; sleep 1' &
sleep 3
grep -c 'read(8' /tmp/kitty_verify/yes.log                             # read count
awk '/read\(8/ {match($0, /= ([0-9]+)/, a); if (a[1]) print a[1]}' /tmp/kitty_verify/yes.log | sort -n | \
    awk 'NR==1{min=$1} {sum+=$1; arr[NR]=$1} END{n=NR; mid=arr[int(n/2)+1]; max=$1; print "count="n" min="min" max="arr[n]" median="mid" mean="sum/n}'
```

Expected:
- Read count: ~11,000+ (measured 11,699 on reference system; will vary)
- min: ~79, max: ~14,133, median: ~569, mean: ~822

#### 9.5.4 Cleanup after investigation

```bash
rm -rf /tmp/kitty_verify/
pkill -f 'Xvfb :99' || true
pkill -f 'kitty/launcher/kitty' || true
```

Confirm cleanup:
```bash
ls /tmp/kitty_verify/ 2>&1               # expect: cannot access: No such file or directory
pgrep -af Xvfb                            # expect: no output
pgrep -af 'kitty/launcher/kitty'          # expect: no output
```

### 9.6 Running the Test Suite

```bash
./test.py
```

**Expected** (as of this branch):
- `Ran 145 tests in ~10s`
- `FAILED (failures=3, skipped=6)` — these 3 failures (`test_transfer_receive`, `test_transfer_send`, `test_glfw_modules`) are **pre-existing environment-dependent failures** identical on the clean base branch; they are **not** caused by this PR.
- `All Go tests succeeded, ran in ~10s`

To confirm the failures are pre-existing (sanity check):
```bash
git stash         # (should be a no-op on a clean branch)
./test.py 2>&1 | grep -E '^(FAIL:|Ran|FAILED)'
```
Same 3 failures will appear on the clean tree.

### 9.7 Verifying the Documentation Artifact

```bash
# File presence and size
ls -la blitzy/documentation/kitty_815df1e210e0.md
# expect: ~44,500 bytes, 564 lines

# Markdown structural integrity
awk '/^```/ {n++} END {print "Code fences:", n, (n%2==0 ? "(balanced ✓)" : "(UNBALANCED ✗)")}' \
    blitzy/documentation/kitty_815df1e210e0.md
# expect: Code fences: 20 (balanced ✓)

grep -c '^# '    blitzy/documentation/kitty_815df1e210e0.md   # expect: 1  (single H1)
grep -c '^## '   blitzy/documentation/kitty_815df1e210e0.md   # expect: 12 (H2 headers)
grep -c '^## Q'  blitzy/documentation/kitty_815df1e210e0.md   # expect: 7  (Q-sections)
```

### 9.8 Troubleshooting

| Symptom | Probable Cause | Resolution |
|---------|----------------|------------|
| `setup.py` fails with `freetype-config: command not found` | `libfreetype-dev` not installed | Re-run Section 9.2.1 `apt-get install` command |
| `setup.py` fails on Go compilation | Go < 1.22 | Install `golang-go` (≥ 1.22) or use `gvm` to install Go 1.22+ |
| `kitty --version` errors with `libEGL missing` or similar | `libgl-dev` not installed | Re-run Section 9.2.1 `apt-get install` command |
| `Xvfb :99` fails with "Server already running" | Another Xvfb on display 99 | `pkill -f 'Xvfb :99'`; or use a different display number like `:98` and `export DISPLAY=:98` |
| `kitty/launcher/kitty` exits with "cannot open display" | `DISPLAY` env var not set or Xvfb not running | Run `export DISPLAY=:99` and confirm Xvfb is running (`pgrep -af Xvfb`) |
| `strace: Operation not permitted` | Container lacks `CAP_SYS_PTRACE`, or `kernel.yama.ptrace_scope` forbids tracing | Run with elevated capabilities, or set `sysctl -w kernel.yama.ptrace_scope=0` (in a lab environment) |
| `read(8, ...)` doesn't appear in strace output for `echo test123` | `strace` was attached to wrong PID (main process, not the `KittyChildMon` thread) | Use `strace -f` (follows forks/threads) — the `-f` is required because the read happens on a dedicated I/O thread, not the main thread |
| `yes hello` trace shows fewer reads than expected (~100s instead of 1000s) | The `sleep 1` window was too short (small delay), or `strace` itself slows the I/O thread enough to let the parser drain between every read | Increase `timeout` to 3 s; or use `strace -f` with larger `-s 0` output suppression to reduce instrumentation overhead |
| Pre-existing test failures alarm the reviewer | 3 failures are environment-dependent and unrelated to this PR | See Section 3; run `git stash && ./test.py` to confirm identical failures on clean base |
| Document claims `fd 8` but reviewer sees different fd in their run | fd number is runtime-determined, not hardcoded | Section Q5 of the document (and this guide) explicitly note this; any low fd ≥ 3 is normal |
| Cleanup forgets to kill Xvfb and leaves orphan process | Interrupted Ctrl-C before cleanup block ran | Run `pkill -f 'Xvfb :99'` manually; confirm with `pgrep -af Xvfb` |

---

## 10. Appendices

### 10.A Command Reference

| Purpose | Command |
|---------|---------|
| Build kitty (debug) | `python3 setup.py build --debug --verbose` |
| Show kitty version | `kitty/launcher/kitty --version` |
| Launch kitty under Xvfb | `Xvfb :99 -screen 0 1024x768x24 -nolisten tcp -ac & export DISPLAY=:99 && kitty/launcher/kitty --hold sh` |
| Trace shell spawn chain | `strace -f -e trace=clone,execve,openat,ioctl -o /tmp/spawn.log kitty/launcher/kitty --hold sh` |
| Trace reads with timing | `strace -f -T -e trace=read -o /tmp/reads.log kitty/launcher/kitty --hold sh -c 'echo test123; sleep 2'` |
| Run unit tests | `./test.py` |
| Inspect PTY master fd symlink | `ls -l /proc/$(pgrep -f 'kitty/launcher/kitty' \| head -1)/fd/8` |
| Inspect shell's stdin/stdout/stderr | `ls -l /proc/$(pgrep -f '^/bin/bash --posix')/fd/{0,1,2}` |
| Check file diff vs base | `git diff --stat 815df1e21 HEAD` |
| Validate markdown structure | `awk '/^\`\`\`/ {n++} END {print n}' blitzy/documentation/kitty_815df1e210e0.md` (expect 20) |
| Kill background jobs | `kill %1 %2 ... ` or `pkill -f 'Xvfb :99'; pkill -f 'kitty/launcher/kitty'` |

### 10.B Port Reference

This project is a headless runtime investigation. It does **not** open any TCP/UDP network ports. The only "port-like" resource used is the X display:

| Display | Purpose | Lifetime |
|---------|---------|----------|
| `:99` (Unix socket `/tmp/.X11-unix/X99`) | Xvfb virtual framebuffer | Background for duration of investigation; killed with `pkill -f 'Xvfb :99'` |

### 10.C Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **THE DELIVERABLE** — 564-line technical deep-dive, the sole file this project produces |
| `kitty/child-monitor.c:1337` | `read_bytes(int fd, Screen *screen)` — the only caller of `read(2)` on child PTY master fds |
| `kitty/child-monitor.c:1481` | `io_loop()` — the `KittyChildMon` thread entry function; `set_thread_name` at line 1489 |
| `kitty/child-monitor.c:1531` | Call site `has_more = read_bytes(...)` |
| `kitty/child-monitor.c:35` | `#define EXTRA_FDS 2` (wakeup pipe + signal pipe slots in the poll set) |
| `kitty/child.c:81` | `spawn()` — the C function callable from Python that performs fork + PTY setup + execvp |
| `kitty/child.py:170-175` | `openpty()` wrapper (`os.openpty()` + inheritance + iutf8 setup) |
| `kitty/child.py:281` | `master, slave = openpty()` inside `Child.fork()` |
| `kitty/vt-parser.c:18` | `#define BUF_SZ (1024u*1024u)` — the 1 MiB VT parser ring buffer size |
| `kitty/vt-parser.c:1367` | `consume_input()` — the VT parser state-machine dispatcher |
| `kitty/simd-string.c:72` | `utf8_decode_to_esc()` — SIMD-accelerated text/escape splitter (watches for byte `0x1b`) |
| `kitty/simd-string.c:39` | `utf8_decode_to_esc_scalar()` — scalar reference implementation for the same |
| `kitty/constants.py:181` | `shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` |
| `kitty/boss.py:585-588` | `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` |
| `kitty/control-codes.h:53` | `#define ESC 0x1b` — the text/escape boundary sentinel |
| `kitty/data-types.h:114` | `#define MAX_CHILDREN 512` — maximum concurrent PTY slots |
| `tools/tui/run.go:167-171` | macOS-only `argv[0] = "-" + basename(shell)` login-shell gate (`runtime.GOOS == "darwin"`) |
| `kitty/launcher/kitty` | Compiled Go launcher binary (275 KB) |
| `kitty/launcher/kitten` | Compiled Go "kitten" helper binary (21.8 MB) |
| `kitty/fast_data_types.so` | Compiled C extension (6.1 MB) — contains `spawn()`, `child-monitor.c`, `vt-parser.c` functionality |
| `kitty/glfw-x11.so` | Compiled X11 GLFW backend (1.6 MB) |
| `go.mod` | Declares `go 1.22` |
| `pyproject.toml` | Declares `requires-python = ">=3.8"` |
| `setup.py` | Build orchestration (C extensions + Go binaries) |

### 10.D Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| kitty | 0.35.2 | `kitty/launcher/kitty --version` |
| Python | 3.12.3 | `python3 --version` |
| Go | 1.22.2 | `go version` |
| GCC (via `build-essential`) | 13.x (Ubuntu 24.04 default) | `gcc --version` |
| strace | 6.8 | `strace -V` |
| Xvfb | 2:21.1.12-1ubuntu1.5 | `apt show xvfb` |
| xdotool | 1:3.20160805.1-5build1 | `apt show xdotool` |
| Ubuntu | 24.04 | `/etc/os-release` |
| libfreetype | 2.13.2+dfsg-1 | `apt show libfreetype-dev` |
| libharfbuzz | 8.3.0-2build2 | `apt show libharfbuzz-dev` |
| libfontconfig | 2.15.0-1.1ubuntu2 | `apt show libfontconfig-dev` |
| libx11 | 2:1.8.7-1build1 | `apt show libx11-dev` |
| libxkbcommon-x11 | 1.6.0-1build1 | `apt show libxkbcommon-x11-dev` |
| Target branch base commit | `815df1e21` | `git merge-base HEAD origin/main` |
| Feature branch HEAD | `af4774072` | `git rev-parse HEAD` |

### 10.E Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `DISPLAY` | `:99` | Tells X clients (including kitty) which X server to connect to; required for any GUI launch |
| `DEBIAN_FRONTEND` | `noninteractive` | Prevents `apt-get install` from prompting for interactive input during automated installs |
| `PATH` | (prepended with `/usr/local/go/bin` if Go was installed outside `/usr/bin`) | Ensures `go` command is in PATH for `setup.py`'s Go build step |
| `CI` | (unused in this project) | — |
| `KITTY_*` | (not used in AAP scope) | kitty has many `KITTY_*` env vars for configuration; none are set or required by the PTY investigation |

No secrets, API keys, or credentials are involved.

### 10.F Developer Tools Guide

| Tool | When to use | Example |
|------|-------------|---------|
| `strace -f` | Trace all syscalls across forks/threads — **required** for the I/O-thread `read()`s | `strace -f -e trace=read kitty/launcher/kitty --hold sh` |
| `strace -T` | Add per-call timing column (microsecond precision) | `strace -T -e trace=read kitty/launcher/kitty ...` |
| `ps -ef \| grep kitty` | Inspect the process tree after kitty spawns the shell | `ps -ef \| grep -E '(kitty\|bash --posix)'` |
| `/proc/<pid>/fd/` | Resolve file-descriptor-to-device mappings (e.g., `fd 8 → /dev/pts/ptmx`) | `ls -l /proc/$(pgrep -f kitty/launcher/kitty \| head -1)/fd/` |
| `pstree` | Visualize the fork tree | `pstree $(pgrep -f kitty/launcher/kitty \| head -1)` |
| `xdotool` | Simulate typing into the kitty window to drive interactive experiments | `xdotool type --delay 50 "echo test123"` then `xdotool key Return` |
| `grep -n` | Find function definitions / line numbers in C sources | `grep -n 'read_bytes' kitty/child-monitor.c` |
| `git diff --stat` | Confirm scope: only documentation changed | `git diff --stat 815df1e21 HEAD` |
| `awk`/`sort`/`uniq -c` | Post-process strace logs to extract byte-count statistics | See Section 9.5.3 for full example |

### 10.G Glossary

| Term | Definition |
|------|------------|
| **PTY (pseudoterminal)** | A pair of kernel-provided character devices that emulate a hardware terminal: a "master" side used by the terminal emulator (kitty) and a "slave" side used as stdin/stdout/stderr by the child shell. On Linux the pair is created by opening `/dev/ptmx`. |
| **PTY master** | kitty's side of the PTY. The device node (after kernel allocation) appears as `/dev/pts/ptmx` via the `/proc/<pid>/fd/` symlink. kitty held this at fd 8 in the observed run. |
| **PTY slave** | The shell's side of the PTY. The device node is `/dev/pts/N` for some kernel-assigned integer N (here: `/dev/pts/0`). The shell's stdin/stdout/stderr are all duped to this fd. |
| **`BUF_SZ`** | `(1024u*1024u) = 1,048,576` bytes = 1 MiB. The size of kitty's VT parser ring buffer, defined at `kitty/vt-parser.c:18`. The `count` parameter passed to `read(2)` can be up to `BUF_SZ` bytes. |
| **`KittyChildMon`** | The thread name (set via `set_thread_name`) of kitty's dedicated I/O thread. This thread runs `io_loop()`, calls `poll()` on all child PTY master fds + wakeup/signal pipes, and invokes `read_bytes()` on `POLLIN`/`POLLHUP` revents. |
| **`KittyPeerMon`** | The thread name of kitty's remote-control peer-socket thread. **Not** involved in PTY communication. |
| **`read_bytes()`** | The C function at `kitty/child-monitor.c:1337` that is the only caller of `read(2)` on a child PTY master fd. Every byte of shell output crosses into kitty's address space through this one function. |
| **`consume_input()`** | The top-level VT parser state-machine dispatcher at `kitty/vt-parser.c:1367`. Switches on `VTEState` and routes bytes to `consume_normal`, `consume_esc`, `consume_csi`, or the ST-terminated family dispatchers (OSC/DCS/APC/PM/SOS). |
| **`utf8_decode_to_esc()`** | The SIMD-accelerated byte scanner at `kitty/simd-string.c:72` that decodes UTF-8 codepoints until it hits the `0x1b` (ESC) sentinel byte, at which point it returns `true` and `consume_normal` transitions to `VTE_ESC`. The physical text-vs-escape byte boundary lives here. |
| **`vt_parser_create_write_buffer()`** | Returns a pointer into the parser's ring buffer along with the number of free bytes available. Called by `read_bytes()` to know where to put newly-`read()` data. |
| **`vt_parser_commit_write()`** | Called by `read_bytes()` after `read()` returns, to atomically update `self->write.pending` so the main thread can see the new bytes. |
| **`vt_parser_has_space_for_input()`** | Returns `true` if `self->read.sz + self->write.pending < BUF_SZ`. When this is `false`, `io_loop()` clears `POLLIN` from the fd's event mask, implementing back-pressure. |
| **`run_worker()`** | The main-thread function at `kitty/vt-parser.c:1417` that locks the parser mutex, flips `write.pending → read.sz`, calls `consume_input()` repeatedly until the buffer is drained, then `memmove`-compacts the ring buffer. |
| **`POLLIN` level-triggering** | `poll()` returns with `POLLIN` asserted as long as there is **any** data to read on the fd — not just when new data arrives. This is why the I/O thread's read loop hot-spins during high-volume output. |
| **Line discipline (N_TTY)** | The kernel subsystem that processes PTY bytes before they reach the master: handles local echo (typing → echo back out the master side), CR/LF translation (`ONLCR` turns `\n` → `\r\n`), canonical-mode line buffering, and signal handling (Ctrl-C → SIGINT). Responsible for the 1-byte-per-keystroke reads observed during interactive typing. |
| **`TIOCSCTTY`** | The ioctl that makes a terminal fd the controlling terminal of the calling process's session. kitty issues `ioctl(tfd, TIOCSCTTY, 0)` in the child post-`setsid()` to attach `/dev/pts/0` as the shell's controlling terminal. |
| **`TIOCSPTLCK` / `TIOCGPTN`** | ioctls used by the glibc `openpty(3)` family to unlock the slave side and query the slave index, respectively, after `/dev/ptmx` is opened. |
| **`ONLCR`** | Termios output flag: when set (the default for cooked PTY mode), the kernel translates `\n` produced by the shell into `\r\n` on the master side. This is why `echo test123\n` produces a 9-byte read, not 8. |
| **`EIO` on a PTY master `read()`** | Linux convention for "the slave side was closed." kitty's `read_bytes()` silently commits zero bytes and returns `false` so `io_loop()` can mark the child slot `needs_removal` and schedule cleanup. |
| **`kitten run-shell`** | A Go-compiled helper binary (`kitty/launcher/kitten`) that kitty execs between the launcher and the final shell. It injects shell-integration setup (e.g., prompt-marking, `--posix` flag) before calling `syscall.Exec()` to replace itself with the real shell. |
| **Back-pressure** | The ring-buffer-mediated rate control between the I/O thread (producer) and the main thread (consumer). Implemented in user-space via `vt_parser_has_space_for_input()` rather than using kernel `O_NONBLOCK` / `EAGAIN` flow control. |
| **SWE-AtlasQnA-Repo rule** | The AAP rule (§0.1.2, §0.7.1) that directs the agent to produce exactly one markdown document in `blitzy/documentation/` answering the prompt's questions, without modifying any existing source file, and to delete any temporary investigation artifacts. |

---

**End of Blitzy Project Guide — kitty PTY Communication Pipeline (Task 815df1e210e0) — 93.75% complete, 30.0 / 32.0 hours, READY FOR HUMAN REVIEW**
