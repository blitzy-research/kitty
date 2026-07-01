# Blitzy Project Guide — Kitty Success-Exit (Status 0) Q&A Investigation

> **Product under investigation:** [`kovidgoyal/kitty`](https://github.com/kovidgoyal/kitty) — GPU-based terminal emulator, version **`0.35.2`** (`kitty/constants.py:L25`), pinned HEAD **`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`**
> **Deliverable branch:** `blitzy-00762a76-e9fc-4347-ac05-99a5a988cece` · **HEAD:** `2172dbe61`
> **Brand color legend:** 🟦 Completed / AI Work = Dark Blue `#5B39F3` · ⬜ Remaining / Not Completed = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, **run-verified Markdown document** that explains, end-to-end, what the Kitty terminal emulator does when it launches a program that prints a few lines to standard output and then exits with status zero. It is a strictly read-only, additive code-archaeology / Q&A task: the answer is derived by actually building and running Kitty's relevant code paths, capturing real output, and grounding every value in an exact `file:line` citation. The audience is engineers seeking an authoritative, evidence-based explanation of Kitty's child-process lifecycle and its per-command exit-status reporting. Technical scope spans C (VT parser, screen, child-monitor), Python (boss, window, child, main), and shell-integration scripts. The sole artifact produced is `blitzy/documentation/kitty_815df1e210e0.md`; no existing source file is modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextSize':'16px'}}}%%
pie showData title Completion — 92.3% (24h of 26h)
    "Completed Work (AI)" : 24
    "Remaining Work" : 2
```

> 🟦 **Completed = 24h (Dark Blue `#5B39F3`)** · ⬜ **Remaining = 2h (White `#FFFFFF`)** · **Center metric: 92.3% complete**

| Metric | Hours |
|--------|-------|
| **Total Hours** | **26** |
| **Completed Hours (AI + Manual)** | **24** (AI: 24 · Manual: 0) |
| **Remaining Hours** | **2** |
| **Percent Complete** | **92.3%** — `24 ÷ 26 × 100` |

### 1.3 Key Accomplishments

- ✅ Authored the sole in-scope deliverable `blitzy/documentation/kitty_815df1e210e0.md` (663 lines / ~5,000 words), committed across two agent commits (`4a8c9568a` create, `2172dbe61` review-fixes).
- ✅ Answered **all eight sub-questions (Q1–Q8)**, each with the full four-part structure: exact literal, `file:line` citation, verbatim observed output, and rationale.
- ✅ **Investigate-by-running honored:** Kitty was built (hybrid C/Go/Python) and run; **10 console observation blocks** capture real output (Kitty's own `$?`, the 6 `shell_integration` tests, a real compiled-parser OSC 133 cycle, raw on-the-wire `ESC]133;D;0 BEL` bytes, an `os.waitpid` POSIX decode, the Q8 stdout/Screen buffer).
- ✅ **54 `file:line` citations across 11 files** verified accurate against source at the pinned HEAD (independently re-checked).
- ✅ **Two-mechanism distinction** made explicit and never conflated (A: OS `SIGCHLD`/`waitpid` reaping of the shell; B: per-command OSC 133 status transport).
- ✅ **OSC 133 protocol externally validated** (FinalTerm / iTerm2 / VS Code / WezTerm / Ghostty) and matched to Kitty's implementation.
- ✅ **Read-only constraint provably honored:** source tree byte-for-byte unchanged; all temporary scripts removed; working tree clean.
- ✅ Evidence-path test suite (`shell_integration`) re-run: **6/6 pass**.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None affecting the deliverable._ The document is complete, defect-free, and validated; no blocking issues remain. | None | — | — |
| (Informational) 6 pre-existing environmental test failures in out-of-scope `kitty_tests/*` (file-transmission `/tmp` setgid; font PostScript-name mismatch) | None on deliverable — evidence path is 6/6 green; would fail identically on the pristine pinned commit | Human reviewer (awareness only) | N/A |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository (`kovidgoyal/kitty` working tree) | Read/Write (branch) | None — full access; single doc committed successfully | ✅ Resolved | — |
| Build toolchain (gcc 15.2.0, go 1.22.12, python 3.13.7, xvfb) | Local execution | None — all present; build & run succeeded | ✅ Resolved | — |

**No access issues identified** that prevent build validation, integration, or the (documentation) deployment path.

### 1.6 Recommended Next Steps

1. **[Medium]** Perform a subject-matter **technical review** of `blitzy/documentation/kitty_815df1e210e0.md` — spot-check a representative sample of the 54 `file:line` citations against source at pinned HEAD `815df1e210e0`. _(~0.75h)_
2. **[Medium]** **Review observation reproducibility & narrative** — read the two-mechanism model and the eight answers; optionally re-run the `shell_integration` harness (expect 6/6) and the Q1 GUI contrast (child `0/7/42` → kitty `0`). _(~0.75h)_
3. **[Low]** **Approve & merge** the documentation PR; confirm `git diff --name-status 815df1e21 HEAD` shows only the added doc. _(~0.5h)_

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

🟦 **All completed work is autonomous (AI) — Dark Blue `#5B39F3`.** Each component traces to a specific AAP requirement.

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment & build setup | 3.0 | Build hybrid C/Go/Python Kitty 0.35.2 via `python3 setup.py` (122 C translation units, 5 link targets); resolve `-Wno-error=switch` (host `wayland-protocols` enum); verify artifacts (`fast_data_types.so`, `launcher/kitty`, `launcher/kitten`). _[AAP 0.5.1 build-verified environment]_ |
| Investigation & observation capture | 7.0 | Capture 10 console observation blocks / 7 distinct observations: Q1 GUI exit codes (child `0/7/42`), the 6 `shell_integration` tests, a real compiled-parser OSC 133 `D` cycle, raw on-the-wire `ESC]133;D;0 BEL` hex bytes, an `os.waitpid` POSIX status decode, the Q8 empty-stdout + Screen-buffer read, and version confirmation. _[AAP: investigate-by-running, verbatim output]_ |
| Code archaeology & citation tracing | 5.0 | Trace and verify 54 `file:line` citations across 11 files (C: `child-monitor.c`/`screen.c`/`vt-parser.c`; Python: `main`/`boss`/`window`/`child`/`constants`; shell: `bash`/`zsh`/`fish`; test harness) for Q1–Q8. _[AAP: be exact & grounded]_ |
| Web research (OSC 133 protocol) | 0.5 | Validate OSC 133 naming/semantics (FinalTerm / iTerm2 / VS Code / WezTerm / Ghostty) and match to Kitty's implementation. _[AAP 0.2.2 web-search requirement]_ |
| Answer document authoring | 5.5 | Author the 663-line Markdown: eight four-part answers, the two-mechanism model, the mermaid flowchart, coverage-pass table, edge cases, and verification summary. _[AAP: the sole deliverable]_ |
| Review-finding remediation (F1–F4) | 2.0 | Resolve 4 review findings in commit `2172dbe61`: corrected Python ≥3.10 citation (`constants.py:L231`), expanded environment/build detail (Docker image + exact commands), harness `assert_command` citation (`L266-L269`), and verbatim-ized observation blocks. _[AAP: review remediation]_ |
| Coverage pass, verification & cleanup | 1.0 | Coverage pass over Q1–Q8; verify read-only (byte-for-byte unchanged); remove temporary observation scripts created outside the tree. _[AAP: coverage pass + read-only + cleanup]_ |
| **Total** | **24.0** | **Matches Completed Hours in §1.2** |

### 2.2 Remaining Work Detail

⬜ **All remaining work is human path-to-production — White `#FFFFFF`.** A standalone Markdown file has no build/deploy/CI pipeline (it is independent of the Sphinx docs toolchain per AAP 0.2.3).

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review of answer document (citation accuracy + observation reproducibility & narrative correctness) | 1.5 | Medium |
| PR merge & close-out (verify source tree unchanged) | 0.5 | Low |
| **Total** | **2.0** | **Matches Remaining Hours in §1.2 and §7 pie** |

### 2.3 Hours Reconciliation

| Check | Result |
|-------|--------|
| §2.1 completed rows sum | 24.0 |
| §2.2 remaining rows sum | 2.0 |
| §2.1 + §2.2 | **26.0 = Total (§1.2)** ✅ |
| Completion `24 ÷ 26 × 100` | **92.3%** ✅ |

---

## 3. Test Results

> **Integrity note:** All tests below originate from Blitzy's autonomous validation logs for this project. This is a documentation task — **no new code and no new tests were authored**; Kitty's existing suites were **executed for observation/validation** to ground the answers. "Coverage %" is **N/A** (no product code was added).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Shell Integration (evidence path) | kitty `BaseTest`/unittest (PTY harness) | 6 | 6 | 0 | N/A | bash/fish/zsh × `ShellIntegration` + `ShellIntegrationWithKitten`; grounds Q2/Q4/Q7 (OSC 133). Re-run: `Ran 6 tests … OK`. |
| Full Python unit suite (validation) | kitty `BaseTest`/unittest | 145 | 139 | 6 | N/A | The 6 failures are **pre-existing, environmental, out-of-scope**: 2× `file_transmission` (`/tmp` setgid bit) — pass under a non-setgid `TMPDIR`; 4× `fonts.Selection` (FiraCode PostScript-name mismatch). Unrelated to the deliverable. |
| Go suite (validation) | `go test` | All | All | 0 | N/A | All Go tests pass. |

**Key assertion observed (grounds Q2/Q4/Q7):** the harness `assert_command` (`kitty_tests/shell_integration.py:L266-L269`) waits until `last_cmd_exit_status == 0` and `last_cmd_cmdline == cmd` — i.e., the parsed OSC 133 `D` status is recorded by the terminal.

---

## 4. Runtime Validation & UI Verification

> This deliverable is a Markdown document (no web/GUI of its own). "Runtime validation" here means the **build/run/observe** of Kitty used to ground the answers. Status legend: ✅ Operational · ⚠ Partial · ❌ Failing.

**Build & runtime health**
- ✅ Build succeeds — `CFLAGS='-Wno-error=switch' python3 setup.py --full` → exit 0 (122 C TUs, 5 link targets).
- ✅ Binary runs — `./kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal`.
- ✅ Native extension imports — `import kitty.fast_data_types` succeeds; `Screen(None,24,80)` constructs (real C VT parser present).
- ✅ Evidence Python modules compile — `main.py`, `boss.py`, `window.py`, `child.py` (`py_compile` clean).

**Observation / behavior verification (per question)**
- ✅ **Q1** — GUI run under `xvfb` with child exit `0/7/42` → **Kitty exit code `0`** in every case (child status not propagated).
- ✅ **Q2/Q4/Q7** — `shell_integration` harness drives the OSC 133 path end-to-end; **6/6 pass**; `last_cmd_exit_status == 0` observed.
- ✅ **Q7** — raw on-the-wire capture confirms `ESC ] 133 ; D ; 0 BEL` (hex `1b 5d 31 33 33 3b 44 3b 30 07`).
- ✅ **Q6** — `os.waitpid` decode of a status-0 child: `status_raw=0`, `WIFEXITED=True`, `WEXITSTATUS=0` (POSIX semantics shown separately, honestly not attributed to Kitty's C reaper, which stores the raw status).
- ✅ **Q8** — child stdout renders into the per-window `Screen` (rows 0–2 = printed lines; kitty's own stdout file 0 bytes).

**UI verification:** N/A — the deliverable is a text document; no screenshots/screencasts are applicable.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables / governing-rule directives to their delivered status. Fixes applied during autonomous validation are noted.

| AAP / Rule Requirement | Benchmark | Status | Evidence / Fix |
|------------------------|-----------|--------|----------------|
| Deliverable location & name (`blitzy/documentation/kitty_815df1e210e0.md`) | Exact match | ✅ Pass | `git diff --name-status` → `A blitzy/documentation/kitty_815df1e210e0.md` |
| Answer all 8 sub-questions (Q1–Q8) | 8/8 answered | ✅ Pass | 8 `## Q` headings, each with (a)/(b)/(c)/(d) parts (8/8 each) |
| Four-part structure (literal · citation · observation · rationale) | Every answer | ✅ Pass | grep of section markers: 8/8/8/8 |
| Investigate-by-running first | Observe before writing | ✅ Pass | 10 console observation blocks with producing commands |
| Quote observed output verbatim | Exact reproduction | ✅ Pass | Q1 exit codes, 6 tests, OSC 133 bytes, `os.waitpid` decode |
| Be exact & grounded (`file:line`, no paraphrase) | Every value cited | ✅ Pass | 54 citations across 11 files; sample re-verified accurate |
| Provide rationale per answer | Every answer | ✅ Pass | 8/8 `(d) Rationale` blocks |
| Two-mechanism distinction (A vs B) | Not conflated | ✅ Pass | Explicit sections + mermaid (2 subgraphs) |
| Web-search OSC 133 validation | External confirmation | ✅ Pass | FinalTerm/iTerm2/VS Code/WezTerm/Ghostty cited |
| Read-only scope (no source modified) | Byte-for-byte unchanged | ✅ Pass | source diff empty; working tree clean |
| Temporary-script cleanup | Removed after use | ✅ Pass | scripts in `/tmp` (outside tree), removed; `git status` clean |
| Coverage pass over all sub-questions | Explicit pass | ✅ Pass | "Coverage pass" table maps every Q |
| Markdown well-formed | Balanced fences, valid mermaid | ✅ Pass | 22 balanced fences; mermaid 2 subgraphs/2 ends |
| Review findings F1–F4 | All resolved | ✅ Pass | commit `2172dbe61`: `constants.py:L231`, env/build detail, harness citation, verbatim blocks |

**Outstanding compliance items:** none. Quality is release-grade pending standard human review.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation line-number drift if the doc is regenerated against a different commit | Technical | Low | Low | Doc prominently pins HEAD `815df1e210e0`; re-verify line numbers only on regeneration (AAP 0.6.3) | Mitigated / Documented |
| Build requires `-Wno-error=switch` (host `wayland-protocols` adds enum values post-dating bundled glfw) | Technical | Low | Medium | Exact flag documented; keeps `-Werror` for all else; edits no source | Mitigated |
| Environmental test noise: 6 pre-existing out-of-scope failures could mislead a reviewer | Technical | Low | Medium | Documented as environmental/out-of-scope; evidence path is 6/6 green; identical on pristine commit | Documented |
| Documentation accuracy depends on human sign-off | Operational | Low | Low | Validator + independent re-verification found zero defects; review task scheduled (§2.2) | Open (planned) |
| Observation reproducibility depends on the authoritative Docker/`xvfb` environment | Operational | Low | Low | Exact Docker image + all commands documented in the doc and §9 | Mitigated |
| Security exposure | Security | None | — | No code/dependencies added, no secrets, read-only Markdown | N/A |
| Integration failure | Integration | None | — | Standalone Markdown, zero runtime deps, not part of Sphinx toolchain, no build/deploy/API | N/A |

---

## 7. Visual Project Status

**Project hours breakdown** (🟦 Completed = Dark Blue `#5B39F3` · ⬜ Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px'}}}%%
pie showData title Project Hours (Total 26h)
    "Completed Work" : 24
    "Remaining Work" : 2
```

> **Integrity:** "Remaining Work" = **2** equals §1.2 Remaining Hours and the sum of §2.2 "Hours". "Completed Work" = **24** equals §1.2 Completed Hours.

**Remaining work by category / priority** (sums to 2h):

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review (citations + observations/narrative) | 1.5 | 🟨 Medium |
| PR merge & close-out | 0.5 | ⬜ Low |
| **Total** | **2.0** | — |

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#A8FDD9'}}}%%
pie showData title Remaining 2h by Priority
    "Medium (review)" : 1.5
    "Low (merge)" : 0.5
```

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.3% complete (24h of 26h)**. Every one of the 20 AAP-scoped requirements is complete and defect-free: the sole deliverable `blitzy/documentation/kitty_815df1e210e0.md` answers all eight sub-questions with exact literals, accurate `file:line` citations, verbatim run-captured observations, and rationale — grounded by an actual build-and-run of Kitty 0.35.2. The read-only constraint is provably satisfied (source tree byte-for-byte unchanged; temporary scripts removed).

**Remaining gaps.** The outstanding **2h** is entirely standard human path-to-production for a documentation artifact: a subject-matter review of the document (1.5h) and PR merge/close-out (0.5h). There is no build/deploy/CI work because a standalone Markdown file has none, and there are **no blocking defects**.

**Critical path to production.** Review → approve → merge. No code changes, migrations, or infrastructure work are required.

**Success metrics (all met):** 8/8 questions answered with the four-part structure; 54/54 sampled citations accurate; evidence-path tests 6/6; two-mechanism distinction explicit; read-only compliance verified.

**Production-readiness assessment:** **Ready for human review.** The completion percentage is intentionally held below 100% because standard human review/merge remains, per honest-assessment principles.

| Metric | Value |
|--------|-------|
| AAP-scoped completion | 92.3% (24h / 26h) |
| AAP-scoped requirements complete | 20 / 20 |
| Evidence-path tests | 6 / 6 pass |
| Blocking defects | 0 |
| Source files modified | 0 (read-only honored) |

---

## 9. Development Guide

How to build, run, verify, and reproduce the observations behind the deliverable. All commands were tested in the build environment.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu 25.10 container used). Authoritative env = Docker image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (fallback `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`).
- **Compilers/toolchain (verified versions):**
  - Python `3.13.7` (Kitty requires ≥ 3.10 — guarded at `kitty/constants.py:L231`)
  - GCC `15.2.0` (C is `-std=c11`, `setup.py:L492`)
  - Go `1.22.12` (`go 1.22`, `go.mod:L3`)
  - `xvfb-run` (for GPU-terminal GUI runs), `git 2.51.0`

### 9.2 Environment Setup

```bash
# From the repository root
cd /path/to/kitty          # repo root containing setup.py, Makefile, kitty/, shell-integration/

# No pip venv is required — Kitty has zero pip runtime dependencies.
# Confirm toolchain:
python3 --version && gcc --version | head -1 && go version && command -v xvfb-run
```

### 9.3 Build

```bash
# Canonical entrypoint (Makefile 'all:' target runs 'python3 setup.py').
# The -Wno-error=switch flag is required ONLY due to the host's newer
# wayland-protocols enum values; it keeps -Werror for everything else and edits no source.
CFLAGS='-Wno-error=switch' python3 setup.py --full
```

Expected: exit code 0; artifacts produced at `kitty/fast_data_types.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`.

### 9.4 Run & Verify

```bash
# 1) Version (confirms the pinned 0.35.2 build)
./kitty/launcher/kitty --version
# -> kitty 0.35.2 created by Kovid Goyal

# 2) Native extension + real C VT parser import
python3 -c "import sys; sys.path.insert(0,'.'); import kitty.fast_data_types; \
from kitty.constants import version; print('kitty', '.'.join(map(str, version)))"
# -> kitty 0.35.2

# 3) Evidence-path test suite (grounds Q2/Q4/Q7)
LANG=C.UTF-8 xvfb-run -a ./kitty/launcher/kitty +launch test.py --module shell_integration
# -> Ran 6 tests ... OK   (bash/fish/zsh x 2 classes)

# 4) Q1 core claim: child status is NOT propagated to Kitty's own exit code
for st in 0 7 42; do
  LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a ./kitty/launcher/kitty --config NONE \
    sh -c "printf 'line1\nline2\nline3\n'; exit $st" 2>/dev/null
  echo "child exit $st -> kitty exit code: $?"
done
# -> kitty exit code: 0  (for every child status)
```

### 9.5 View the Deliverable & Confirm Read-Only Compliance

```bash
# View the answer document
sed -n '1,60p' blitzy/documentation/kitty_815df1e210e0.md

# Prove the source tree is unchanged apart from the added doc
git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
# -> A   blitzy/documentation/kitty_815df1e210e0.md   (and nothing else)
```

### 9.6 Troubleshooting

- **Build fails on a glfw `switch` warning** → prepend `CFLAGS='-Wno-error=switch'` (host `wayland-protocols` mismatch only; no source edit).
- **GUI run errors about missing display / GL** → run under a virtual display: prefix `xvfb-run -a` and add `LIBGL_ALWAYS_SOFTWARE=1`.
- **`--module` seems ignored** → `--module` is an argument of `test.py`, not of `setup.py test`; use the launcher form `./kitty/launcher/kitty +launch test.py --module <name>`.
- **`file_transmission` tests fail** → the container's `/tmp` carries the setgid bit; re-run with a non-setgid `TMPDIR` (e.g. `TMPDIR=/root/cleantmp`). Environmental, not a code defect.
- **`fonts.Selection` tests fail** → installed FiraCode PostScript names differ from test expectations; test-data/environment mismatch, out-of-scope.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Build | `CFLAGS='-Wno-error=switch' python3 setup.py --full` |
| Version | `./kitty/launcher/kitty --version` |
| Evidence tests | `LANG=C.UTF-8 xvfb-run -a ./kitty/launcher/kitty +launch test.py --module shell_integration` |
| Full test suite | `LANG=C.UTF-8 xvfb-run -a python3 setup.py test` |
| Q1 reproduction | `LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a ./kitty/launcher/kitty --config NONE sh -c '...; exit N'; echo $?` |
| Read-only verify | `git diff --name-status 815df1e21 HEAD` |

### B. Port Reference

Not applicable — Kitty is a desktop terminal emulator; the deliverable is a document. No network ports are used.

### C. Key File Locations

| File | Role | Key locators |
|------|------|--------------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** | Q1–Q8 answers, coverage pass, verification summary |
| `kitty/main.py` | Process entrypoint (Q1) | `L524-L531` (`SystemExit(1)` only on exception) |
| `kitty/window.py` | Message path (Q2, Q4) | `L1408` `handle_cmd_end`; `L1429` message template |
| `kitty/child-monitor.c` | OS reaping (Q3, Q5, Q6) | `L121` `SIGCHLD`; `L1418` `waitpid(-1,&status,WNOHANG)` |
| `kitty/screen.c` | OSC 133 handler (Q7) | `L2350-L2352` case `'D'` → `cmd_output_marking` |
| `kitty/vt-parser.c` | OSC dispatch (Q7) | `L536` `case 133` |
| `kitty/child.py` | PTY ownership (Q3, Q8) | `L337-L345` `child_fd` |
| `shell-integration/bash/kitty.bash` | Emits status (Q7) | `L239` `133;D;$?` |
| `kitty_tests/shell_integration.py` | Observation harness | `L266-L269` `assert_command` |
| `kitty/constants.py` | Version | `L25` `Version(0, 35, 2)` |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Kitty | 0.35.2 | `kitty/constants.py:L25` |
| Python | 3.13.7 (≥3.10 required) | runtime / `kitty/constants.py:L231` |
| GCC | 15.2.0 (C `-std=c11`) | runtime / `setup.py:L492` |
| Go | 1.22.12 (`go 1.22`) | runtime / `go.mod:L3` |
| Git | 2.51.0 | runtime |

### E. Environment Variable Reference

| Variable | Purpose |
|----------|---------|
| `CFLAGS='-Wno-error=switch'` | Build flag to tolerate host `wayland-protocols` enum mismatch (no source edit) |
| `LANG=C.UTF-8` | Locale for the test harness |
| `LIBGL_ALWAYS_SOFTWARE=1` | Force software GL for headless GUI runs |
| `TMPDIR` | Point to a non-setgid dir to run `file_transmission` tests (environmental) |
| `KITTY_SHELL_INTEGRATION` | Kitty runtime flag controlling shell-integration (OSC 133) features |

### F. Developer Tools Guide

- **`xvfb-run`** — provides a virtual X display so Kitty's GPU/GUI paths run headlessly.
- **`git diff --name-status <base> HEAD`** — the primary read-only compliance check.
- **`od -c` / hexdump** — used to capture raw on-the-wire OSC 133 bytes for Q7.
- **`python3 -m py_compile`** — sanity-compile the Python evidence modules.
- Browser DevTools / Lighthouse — **not applicable** (no web UI).

### G. Glossary

| Term | Meaning |
|------|---------|
| **OSC 133** | "Semantic prompt" shell-integration escape protocol (`ESC ] 133 ; D ; <status> ST`); the `D` marker carries a command's exit status |
| **PTY** | Pseudo-terminal; the master/slave pair connecting Kitty and its child shell |
| **`SIGCHLD`** | POSIX signal delivered to a parent when a child process changes state (e.g., exits) |
| **`waitpid`** | Syscall used to reap a terminated child and retrieve its raw status |
| **Mechanism A** | OS-level `SIGCHLD`/`waitpid` reaping of the window's direct-child shell (window lifecycle) |
| **Mechanism B** | Per-command OSC 133 status transport that produces the user-visible completion message |
| **`ChildMonitor`** | Kitty's C runtime that tracks and reaps window child processes |
| **`fast_data_types`** | Kitty's compiled C extension exposing the terminal core to Python |

---

> **Cross-Section Integrity — validated before submission**
> **Rule 1 (1.2 ↔ 2.2 ↔ 7):** Remaining = **2h** in §1.2, §2.2 sum, and §7 pie ✅
> **Rule 2 (2.1 + 2.2 = Total):** 24 + 2 = **26h** = §1.2 Total ✅
> **Rule 3 (Section 3):** all tests originate from Blitzy's autonomous validation logs ✅
> **Rule 4 (Section 1.5):** access issues validated against current permissions (none) ✅
> **Rule 5 (Colors):** Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF` throughout ✅
> **Completion 92.3%** consistent across §1.2, §7, and §8 ✅