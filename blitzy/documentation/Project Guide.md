# Blitzy Project Guide — Kitty SSH Kitten: Secure-Session & Connection-Sharing Answer Document

> **Project type:** Documentation / knowledge-extraction (onboarding Q&A) — rule set `SWE-AtlasQnA-Repo`
> **Repository:** `kovidgoyal/kitty` (Kitty terminal emulator) · source branch `kitty_815df1e210e0` · canonical source commit `815df1e21`
> **Sole deliverable:** `blitzy/documentation/kitty_815df1e210e0.md`
> **Brand colors:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This is a knowledge-extraction (onboarding Q&A) project against the **Kitty terminal emulator** codebase. The objective was to produce a single, evidence-backed Markdown document that traces — end-to-end and **from directly observed runtime behavior** — how Kitty's **SSH kitten** (`kitten ssh`) establishes a secure remote session and shares SSH connections, answering eight questions (Q1–Q8). The intended users are engineers onboarding to the SSH-kitten subsystem. The investigation is cross-cutting, spanning Go (kitten runtime), Python (terminal-side data server), C (VT parser), and POSIX-shell/Python remote bootstrap scripts. The single deliverable is `blitzy/documentation/kitty_815df1e210e0.md`, produced under a **strict read-only scope** — zero production source files were modified.

### 1.2 Completion Status

The completion percentage is computed with the **PA1 AAP-scoped hours methodology**: `Completed ÷ (Completed + Remaining)`, counting only work defined by the Agent Action Plan (the deliverable + the build/run investigation) plus standard path-to-production activities.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Project Completion — 86.6%
    "Completed Work (AI) — 42h" : 42
    "Remaining Work — 6.5h" : 6.5
```

| Metric | Hours | Notes |
|---|---|---|
| **Total Hours** | **48.5** | AAP-scoped investigation + deliverable + path-to-production |
| **Completed Hours (AI + Manual)** | **42.0** (42.0 AI + 0.0 Manual) | 100% autonomous (Blitzy agents); no manual hours logged yet |
| **Remaining Hours** | **6.5** | Human path-to-production (review, publish, optional CI) |
| **Percent Complete** | **86.6%** | 42.0 ÷ 48.5 × 100 |

**Completion basis:** All **18 AAP-specified requirements are complete and validated**. The remaining **6.5 h is exclusively human path-to-production** (subject-matter accuracy review, publication, and an optional CI enhancement) — there is **no autonomous rework outstanding** and **no quality defect** (clean build, 100% test pass, all citations resolve, clean working tree). Per honest-assessment policy, completion is capped below 100% because a human acceptance/review gate remains.

### 1.3 Key Accomplishments

- [x] **Authored the single AAP deliverable** — `blitzy/documentation/kitty_815df1e210e0.md`, a 1,675-line evidence-backed answer document (git-confirmed as the *only* added file, +1,675 / −0).
- [x] **Answered all eight questions (Q1–Q8) exhaustively** — end-to-end trace, SHM credential channel, bootstrap generation/execution, archive build/transport, `connection_data` state, `ControlMaster` reuse, shell-specific encoding, and the terminal↔remote DCS protocol.
- [x] **Enumerated every sibling variant** — all six `-o` connection-sharing options, all four `sh`-mode character substitutions + `tr` inverse, the `py` base64 branch, all six remote base64 fallbacks, all 16 `connection_data` fields, every placeholder token, and every named environment variable.
- [x] **Exercised the real `kitten ssh` entry point** across **both** `script_type` variants (`sh` + `py`) and **both** credential paths (push + pull), with measured values confirmed stable across ≥2 runs.
- [x] **Grounded every claim in verbatim evidence** — ~200 `file:line` citations with `observed` / `inferred` / `non-canonical` labels applied one-claim-to-one-evidence.
- [x] **Canonical build is clean** — `./dev.sh build` → "Build successful"; canonical binaries report `kitten 0.35.2` / `kitty 0.35.2`.
- [x] **All reference tests pass 100%** — 7 Go SSH tests, `tools/utils/shm` + `tools/tui`, and 8 Python PTY tests (17 tests total, 0 failures) — independently re-run and confirmed during this assessment.
- [x] **Read-only scope honored** — zero production source files modified; working tree clean; temporary observation scripts removed.

### 1.4 Critical Unresolved Issues

No blocking (release-preventing) issues were identified. The build is clean, all reference tests pass, all citations resolve, and the working tree is clean. The items below are **acceptance gates**, not defects.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| SME technical-accuracy review not yet performed | Doc should be human-validated before being treated as canonical/authoritative | Subject-matter expert (SSH-kitten maintainer) | 4.0 h |
| Security-model claims (Q2 five-part SHM defense) awaiting human security sign-off | Authoritative security explanation should have a human check | Security reviewer / maintainer | (included in review) |
| Deliverable not yet published/merged to a discoverable location | Doc lives under `blitzy/documentation/`, outside the project's Sphinx docs tree | Repo maintainer | 1.0 h |

### 1.5 Access Issues

**No access issues identified.** The canonical build and all runtime observation were performed inside the specified container image with the Go/C/Python toolchains present; the SSH target was an in-container `sshd` on `localhost`. No external repository permissions, service credentials, or third-party API access are required to build, run, test, or publish the deliverable.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| — | — | No access issues identified | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform the **SME technical-accuracy review** of all eight answers (Q1–Q8), confirming each mechanism and cause→effect chain is correct.
2. **[High]** Complete the **security-model verification** of the Q2 five-part SHM defense (owner UID/GID, `0o600`, password match, request-id match, immediate unlink).
3. **[High]** Run the **citation-resolution & label audit** — spot-check a sample of the ~200 `file:line` citations against commit `815df1e21` and confirm the `observed`/`inferred`/`non-canonical` labels.
4. **[Medium]** **Publish/merge** the deliverable to its destination and link it from the appropriate index or knowledge base.
5. **[Low]** *(Optional)* Wire **markdownlint** into doc CI for `blitzy/documentation/` and assign a doc owner / mark the file as a point-in-time snapshot.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously by Blitzy agents (0 manual hours). Each component traces to AAP requirements (the deliverable + the mandated build-and-run investigation).

| Component | Hours | Description |
|---|---:|---|
| Environment setup & canonical build | 4.0 | `./dev.sh build` with `CFLAGS=-Wno-error=switch` (Ubuntu 25.10 wayland-protocols drift); stood up an in-container passwordless `sshd` on `localhost` as the SSH target; verified canonical binaries (`0.35.2`). |
| Cross-language SSH data-path investigation | 10.0 | Traced the full path across Go (`main.go` 897 L, `config.go`, `utils.go`, `askpass.go`), Python (`utils.py`, `main.py`), C (`vt-parser.c`), shell (`bootstrap.sh/.py`, `bootstrap-utils.sh`), and terminal-side host code (`window.py`, `shm.py`, `constants.py`) plus shared Go libs (`tools/utils/shm`, `tools/tui`) and `gen/go_code.py`. |
| Runtime observation harness + real-path execution | 8.0 | Built a PTY harness + transparent `ssh` argv-logging shim; drove the real `kitten ssh localhost`; exercised both `script_type`s (`sh` + `py`) and both credential paths (push + pull); observed cold- vs warm-cache invocation sequences and master health; re-ran for stability (×2). |
| Verbatim evidence capture & analysis | 5.0 | Captured the six `-o` options, the four `sh` char substitutions (decoded from argv), the `py` base64 payload (len 13,484), the DCS frames, the SHM five-part defense, the 254-byte transfer framing, and drain timing. |
| Document authoring | 10.0 | Wrote the 1,675-line Markdown answer (Section 0 methodology + Q1–Q8 + coverage pass), ~200 `file:line` citations, `observed`/`inferred`/`non-canonical` labels, tables, and cause→effect explanations. |
| Validation & QA hardening | 5.0 | Five-commit, review-driven iteration; re-ran the full reference suite (7 Go + 8 Python); audited ~200 citations for resolution; ran the coverage pass; verified read-only scope (clean tree). |
| **Total Completed** | **42.0** | **Matches Section 1.2 Completed Hours** |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production. There are **no code-fix or rework tasks** because the deliverable is defect-free.

| Category | Hours | Priority |
|---|---:|---|
| Human SME technical-accuracy review (8 answers + cause→effect + all enumerated sibling variants) | 4.0 | High |
| Publish/merge deliverable to destination and link from index/KB | 1.0 | Medium |
| *(Optional)* Wire markdownlint into doc CI + assign doc owner / mark point-in-time | 1.5 | Low |
| **Total Remaining** | **6.5** | **Matches Section 1.2 Remaining Hours & Section 7 pie** |

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 total (Completed) | 42.0 h |
| Section 2.2 total (Remaining) | 6.5 h |
| Section 2.1 + Section 2.2 | **48.5 h = Total (Section 1.2)** ✅ |
| Completion % = 42.0 ÷ 48.5 | **86.6%** ✅ |

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** and were **independently re-run and confirmed during this assessment**. They are precisely the reference harnesses the AAP cites as real-path evidence sources.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Go Unit — SSH kitten (`./kittens/ssh/`) | Go `testing` | 7 | 7 | 0 | — (not instrumented) | `TestSSHConfigParsing`, `TestCloneEnv`, `TestSSHBootstrapScriptLimit`, `TestSSHTarfile`, `TestGetSSHOptions`, `TestParseSSHArgs`, `TestRelevantKittyOpts` → `ok kitty/kittens/ssh` |
| Go Unit — SHM helper (`./tools/utils/shm/`) | Go `testing` | 1 | 1 | 0 | — | `TestSHM` — `0o600`, size-prefixed payload, unlink semantics |
| Go Unit — DCS/TUI (`./tools/tui/`) | Go `testing` | 1 | 1 | 0 | — | `TestRenderProgressBar` |
| Python Integration/PTY — SSH kitten (`./test.py --module ssh`) | Python `unittest` (PTY harness) | 8 | 8 | 0 | — | `test_basic_pty_operations`, `test_ssh_bootstrap_with_different_launchers`, `test_ssh_connection_data`, `test_ssh_copy`, `test_ssh_env_vars`, `test_ssh_leading_data`, `test_ssh_login_shell_detection`, `test_ssh_shell_integration` → `Ran 8 tests … OK` |
| **Total** | — | **17** | **17** | **0** | **100% pass rate** | Zero failures, zero blocked |

**Notes on coverage:** Line-coverage was not instrumented — this is a documentation task, and these suites serve as the AAP-cited **real-path evidence harnesses** (e.g., the `check_bootstrap` PTY harness runs the bootstrap scripts inside a real pseudo-terminal). The meaningful metric here is the **100% pass rate (17/17)**, independently reproduced this session:

- `go test -count=1 ./kittens/ssh/` → `ok kitty/kittens/ssh 0.050s` (7/7)
- `go test -count=1 ./tools/utils/shm/ ./tools/tui/` → both `ok`
- `./test.py --module ssh` → `Ran 8 tests in 9.963s` / `OK` (8/8)

---

## 4. Runtime Validation & UI Verification

Kitty is a terminal emulator; the "UI" surface exercised is the **`kitten ssh` command-line/TUI path**, driven through its **real entry point** (not a bypassing hook). There is **no web UI** in scope. Status legend: ✅ Operational · ⚠ Partial · ❌ Failing.

**Canonical binaries & build**
- ✅ `./dev.sh build` → "Build successful. Run kitty as: kitty/launcher/kitty" (`BUILD_EXIT=0`)
- ✅ `kitty/launcher/kitten --version` → `kitten 0.35.2 created by Kovid Goyal`
- ✅ `kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal`
- ✅ `go build ./kittens/ssh/ ./tools/utils/shm/ ./tools/tui/` → exit 0; `go mod verify` → "all modules verified"

**Real `kitten ssh` runtime behavior (reproduced from autonomous logs)**
- ✅ **Entry guards** — missing `KITTY_WINDOW_ID`/`KITTY_PID` → "The SSH kitten is meant to run inside a kitty window" (exit 1); non-tty stdin → "STDIN must be a terminal" (exit 1)
- ✅ **Six connection-sharing `-o` options** emitted exactly and stable ×2: `ControlMaster=auto`, `ControlPath=…/kssh-<PID>-%C`, `ControlPersist=yes`, `ServerAliveInterval=60`, `ServerAliveCountMax=5`, `TCPKeepAlive=no`
- ✅ **Both credential paths** — PUSH (`request_data="0"`, placeholders unsubstituted) and PULL (`request_data="1"`, substituted `id`/`pwfile`/`pw`)
- ✅ **Both `script_type` branches** — `sh` (four char substitutions live in argv) and `py` (base64; argv payload length 13,484 decoding to `#!/usr/bin/env python`)
- ✅ **Master health check** — `ssh -O check` → "Master running (pid=…)" (exit 0)
- ✅ **SHM credential channel** — `/dev/shm/kssh-…` object created with `0o600`, consumed locally, and unlinked; never transmitted

**API/service integration**
- ✅ SSH target: in-container `sshd` reachable on `localhost` (used to drive the real connection path)
- ✅ No external APIs or network services required (self-contained)

**Overall runtime status: ✅ Operational** — every observed value matches the source literals and is reproduced by the passing test harnesses.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and governing-rule directives to their validation status. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Directive | Requirement | Status | Progress | Evidence |
|---|---|---|---|---|
| Deliverable form | Exactly one new MD file `blitzy/documentation/<branch>.md` | ✅ Pass | 100% | `git diff --name-status` → single `A` for `kitty_815df1e210e0.md` |
| Run-before-write | Build & run real paths, then author from observation | ✅ Pass | 100% | Section 0 methodology + captured argv/console output; harnesses re-run |
| Exact entry point | Drive real `kitten ssh`; label non-canonical values | ✅ Pass | 100% | Real path via PTY + `ssh` shim; 22 `NON-CANONICAL` labels applied |
| Default/canonical config | Build/run default config; record exact commands | ✅ Pass | 100% | `./dev.sh build`; binaries `0.35.2`; commands recorded verbatim |
| Both branches & siblings | `py`+`sh`, push+pull, every `-o`/fallback/token/env/substitution | ✅ Pass | 100% | Coverage pass enumerates all; both branches observed |
| Evidence discipline | Verbatim, one-claim/one-evidence; label inferred | ✅ Pass | 100% | Labels: observed / inferred(15) / non-canonical(22) |
| Exhaustive coverage | Every Q part, named item, sibling addressed by name | ✅ Pass | 100% | Coverage-pass checklist; all 8 Q-sections present |
| Exactness & grounding | Exact literals + `file:line`; no paraphrase | ✅ Pass | 100% | ~200 citations (215 unique `file:line`); spot-checks resolve exactly |
| Read-only scope | No source modified; no extra code; temp scripts removed | ✅ Pass | 100% | Working tree clean; only the doc added; `/tmp` scratch removed |
| Stability of measured values | Confirm ≥2 runs for magnitudes/timings | ✅ Pass | 100% | 254-byte line ×2; drain 0.003 s ×2; `-o` options ×2 |
| Markdown quality | Valid, well-formed Markdown | ✅ Pass | 100% | 146 balanced fences; valid UTF-8; no CRLF/BOM; MD009 lint nit fixed |
| Human SME accuracy sign-off | Expert validation before canonical status | ⏳ Pending | 0% | Deferred to human (Section 2.2 High) |

**Fixes applied during autonomous validation:** five-commit, review-driven hardening — final-gate review findings; QA finding F1 (replacements citation) and F2 (VCS anchor); Q1 `argv[18]/argv[19]` evidence correction to match Q7; and a markdownlint MD009 trailing-space fix. Verbatim code-block evidence (captured argv, indented Go source) was deliberately preserved per the verbatim-evidence rule.

**Outstanding compliance item:** human SME accuracy sign-off (the only ⏳ row) — an acceptance gate, not a defect.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| **Citation drift** — ~200 `file:line` citations anchored to `815df1e21`; upstream churn shifts line numbers | Technical | Low | Medium | Doc is explicitly commit-anchored; treat as point-in-time snapshot; re-validate on rebase | Mitigated |
| **Non-canonical / inferred evidence** — 22 non-canonical + 15 inferred claims not from the pure real path | Technical | Low | Low | Honestly labeled per rules; SME prioritizes non-canonical items in review | Mitigated |
| **Build reproducibility** — canonical build needs `CFLAGS=-Wno-error=switch` on Ubuntu 25.10 | Technical | Low | Low–Med | Exact command + rationale documented in Section 0.1 / Section 9 | Mitigated |
| **Security-model mischaracterization** — Q2 five-part SHM defense must be accurate | Security | Medium | Low | Spot-verified vs `utils.py:104-111` (exact match); pending human security sign-off | Open (low prob.) |
| **No product security risk introduced** — 0 source mods, 0 dependency changes | Security | N/A | — | Read-only scope; git-confirmed | N/A |
| **Discoverability/placement** — doc lives under `blitzy/documentation/`, outside the project's Sphinx docs | Operational | Low | Medium | Publish/link from an index or KB (Section 2.2 Medium) | Open |
| **Maintenance ownership** — onboarding snapshot has no defined owner | Operational | Low | Medium | Assign doc owner / mark point-in-time (Section 2.2 Low) | Open |
| **Merge integration** — additive-only (1 new file in new directory) | Integration | Low | Very Low | Standard merge; no conflict surface | Mitigated |
| **No code/build integration surface** — standalone Markdown | Integration | Low | Low | Optional markdownlint in CI | Low |

**Overall risk posture: LOW.** This is a clean, read-only documentation deliverable with no production code changes. The highest-attention item is the human SME accuracy review of the security-model explanation (Security · S1).

---

## 7. Visual Project Status

**Project hours — completed vs remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Project Hours Breakdown (Total 48.5h)
    "Completed Work" : 42
    "Remaining Work" : 6.5
```

**Remaining work by priority** (hours from Section 2.2 — sums to 6.5 h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Remaining Hours by Priority (6.5h)
    "High — SME accuracy review" : 4.0
    "Medium — Publish/merge" : 1.0
    "Low — Optional doc-CI" : 1.5
```

**Remaining hours per category (bar view):**

| Category | Hours | Bar |
|---|---:|---|
| High — SME accuracy review | 4.0 | ██████████████████████████████ |
| Medium — Publish/merge | 1.0 | ███████ |
| Low — Optional doc-CI + ownership | 1.5 | ███████████ |
| **Total** | **6.5** | — |

**Integrity note:** the pie "Remaining Work" (6.5) equals Section 1.2 Remaining Hours (6.5) and the sum of Section 2.2 Hours (4.0 + 1.0 + 1.5 = 6.5). ✅

---

## 8. Summary & Recommendations

**Achievements.** The project delivered its single AAP artifact — `blitzy/documentation/kitty_815df1e210e0.md`, a 1,675-line, evidence-backed answer document that traces, from directly observed runtime behavior, how Kitty's SSH kitten establishes a secure remote session and shares SSH connections. All eight questions (Q1–Q8) are answered exhaustively, every sibling variant is enumerated, and every behavioral claim is paired with verbatim evidence and a `file:line` citation. The real `kitten ssh` entry point was exercised across both `script_type` variants and both credential paths, and measured values were confirmed stable across ≥2 runs.

**Remaining gaps.** The project is **86.6% complete** (42.0 of 48.5 hours). The remaining **6.5 hours are entirely human path-to-production**: a subject-matter-expert technical-accuracy review (including a security-model check and a citation/label audit), publication/merge of the deliverable, and an optional doc-CI enhancement. **There is no autonomous rework outstanding and no quality defect.**

**Critical path to production.**
1. Human SME technical-accuracy review of all eight answers (**High**, 4.0 h) — this is the acceptance gate.
2. Publish/merge the deliverable and link it from a discoverable index/KB (**Medium**, 1.0 h).
3. *(Optional)* Wire markdownlint into doc CI and assign an owner (**Low**, 1.5 h).

**Success metrics (achieved).** Canonical build clean; 17/17 reference tests pass (0 failures); ~200 citations resolve (spot-checks exact); read-only scope honored (1 file added, 0 modified; clean working tree); all eight questions covered with a completed coverage pass.

**Production-readiness assessment.** The deliverable is **content-complete and validated**; it is ready for human acceptance review. Because it is an authoritative explanation of a **security mechanism**, a human SME/security sign-off is recommended before it is treated as canonical — which is exactly why completion is reported at **86.6%** rather than 100%. Overall risk posture is **LOW**.

| Metric | Value |
|---|---|
| Completion | 86.6% (42.0 / 48.5 h) |
| AAP-specified requirements complete | 18 / 18 |
| Reference tests passing | 17 / 17 (100%) |
| Production source files modified | 0 |
| Blocking defects | 0 |
| Overall risk | Low |

---

## 9. Development Guide

This guide documents how to build, run, verify, and view the project. Every command below was tested in the assessment environment.

### 9.1 System Prerequisites

| Tool | Version (verified) | Purpose |
|---|---|---|
| Go | `go1.22.12` (repo pins `go 1.22`) | Builds the `kitten` binary (SSH-kitten runtime) |
| CPython | `3.13.7` local (`requires-python >=3.8`) | Runs Kitty, the terminal-side data server, and the PTY test harness. `./dev.sh build` also pulls a bundled CPython under `dependencies/linux-amd64/`. |
| C compiler | `cc (Ubuntu 15.2.0)` / clang | Compiles Kitty's native C core (incl. `kitty/vt-parser.c`) |
| OpenSSH client | `OpenSSH_10.0p2` | The kitten shells out to `ssh` for the connection, `ControlMaster` multiplexing, and `ssh -O check` |

> Recommended: build inside the specified container image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), which provides the Go/C toolchains.

### 9.2 Environment Setup & Canonical Build

```bash
# From the repository root.
# On Ubuntu 25.10, de-promote only the wayland-protocols library switch-warning
# (no source is modified; all genuine warnings still error):
export CFLAGS="${CFLAGS:+$CFLAGS }-Wno-error=switch"

# Canonical build (documented at docs/build.rst:19; dev.sh runs `go run bypy/devenv.go`):
./dev.sh build ; echo "BUILD_EXIT_CODE=$?"
```

Expected output (tail):

```text
Build successful. Run kitty as: kitty/launcher/kitty
BUILD_EXIT_CODE=0
```

This produces the two launcher binaries used throughout:

```bash
ls -la kitty/launcher/kitty kitty/launcher/kitten
kitty/launcher/kitten --version    # -> kitten 0.35.2 created by Kovid Goyal
kitty/launcher/kitty  --version    # -> kitty 0.35.2 created by Kovid Goyal
```

### 9.3 Fast Verification (no full rebuild needed)

```bash
# Compile-check the SSH kitten and shared libraries:
go build ./kittens/ssh/ ./tools/utils/shm/ ./tools/tui/   # -> exit 0
go mod verify                                             # -> all modules verified
```

### 9.4 Running the Reference Tests

```bash
# Go SSH-kitten unit tests (7 tests):
go test -count=1 ./kittens/ssh/            # -> ok  kitty/kittens/ssh

# Go shared-library tests (SHM + DCS/TUI):
go test -count=1 ./tools/utils/shm/ ./tools/tui/   # -> both ok

# Python PTY / integration tests for the SSH kitten (8 tests):
./test.py --module ssh                     # -> Ran 8 tests ... OK
```

### 9.5 Running the SSH Kitten (real entry point)

The SSH kitten is an **interactive TUI**; it must run inside a kitty window with a terminal stdin.

```bash
# Inside a kitty terminal window:
kitten ssh <host>                              # default: sh bootstrap, push credential path

# Force the Python bootstrap (py script_type):
kitten ssh --kitten interpreter=python3 <host>

# Force the pull credential path (remote requests data via SSH_ASKPASS):
kitten ssh --kitten askpass=ssh <host>
```

Entry guards (observed): running outside a kitty window prints `The SSH kitten is meant to run inside a kitty window` (exit 1); a non-terminal stdin prints `... STDIN must be a terminal` (exit 1).

### 9.6 Viewing the Deliverable

```bash
# The single deliverable (1,675 lines):
less blitzy/documentation/kitty_815df1e210e0.md
# or preview the methodology + first question:
sed -n '1,240p' blitzy/documentation/kitty_815df1e210e0.md
```

### 9.7 Verifying Read-Only Scope

```bash
git diff --name-status 815df1e21..HEAD
# Expected: exactly one line ->  A  blitzy/documentation/kitty_815df1e210e0.md
git status --porcelain          # Expected: empty (clean working tree)
```

### 9.8 Troubleshooting

- **Build fails on a `switch` warning (Ubuntu 25.10):** prepend `export CFLAGS="${CFLAGS:+$CFLAGS }-Wno-error=switch"` before `./dev.sh build`. This de-promotes only the wayland-protocols library-drift warning; no source is changed.
- **`The SSH kitten is meant to run inside a kitty window`:** you launched `kitten ssh` from a plain shell. Run it inside a kitty terminal (it requires `KITTY_WINDOW_ID` and `KITTY_PID`).
- **`STDIN must be a terminal`:** stdin is not a tty (e.g., piped). Run interactively.
- **`ControlPath too long` (macOS):** the kitten atomically symlinks the runtime dir to `/tmp/kssh-rdir-<uid>` when the path exceeds ~35 chars (`kittens/ssh/main.go:128-133`).
- **Go build cannot resolve modules:** run `go mod verify` and ensure `GOFLAGS=-mod=mod` if needed.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---|---|
| `export CFLAGS="${CFLAGS:+$CFLAGS }-Wno-error=switch"` | De-promote Ubuntu 25.10 wayland-protocols switch-warning |
| `./dev.sh build` | Canonical build (→ `kitty/launcher/{kitty,kitten}`) |
| `go build ./kittens/ssh/ ./tools/utils/shm/ ./tools/tui/` | Fast compile check |
| `go mod verify` | Verify module integrity |
| `go test -count=1 ./kittens/ssh/` | Run Go SSH-kitten tests (7) |
| `go test -count=1 ./tools/utils/shm/ ./tools/tui/` | Run shared-library tests |
| `./test.py --module ssh` | Run Python PTY/integration tests (8) |
| `kitten ssh <host>` | Real SSH-kitten entry point |
| `kitten ssh --kitten interpreter=python3 <host>` | Force `py` bootstrap |
| `kitten ssh --kitten askpass=ssh <host>` | Force pull credential path |
| `git diff --name-status 815df1e21..HEAD` | Verify read-only scope |
| `less blitzy/documentation/kitty_815df1e210e0.md` | View the deliverable |

### Appendix B — Port Reference

| Port | Service | Notes |
|---|---|---|
| 22 | in-container `sshd` (`localhost`) | SSH target used to drive the real `kitten ssh` path |

No application listens on a fixed TCP port for this deliverable; the only network endpoint is the SSH target used for observation. The kitten's remote-control forwarding uses `-R 0:<listen_on>` (dynamic remote port) when `forward_remote_control=yes`.

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (sole added file) |
| `kittens/ssh/main.go` | Kitten runtime: `run_ssh`, `connection_data`, `connection_sharing_args`, `make_tarfile`, `bootstrap_script`, `wrap_bootstrap_script` |
| `kittens/ssh/utils.py` | Terminal-side `get_ssh_data` server + `read_data_from_shared_memory` validator |
| `kittens/ssh/config.go` / `utils.go` / `askpass.go` | Config parsing, OpenSSH version detection, askpass runner |
| `shell-integration/ssh/bootstrap.sh` / `bootstrap.py` / `bootstrap-utils.sh` | Remote-side bootstrap (POSIX + Python) and shared utilities |
| `kitty/window.py` / `kitty/vt-parser.c` | DCS reception (`handle_remote_ssh`) and `kitty-` prefix dispatch |
| `kitty/shm.py` / `tools/utils/shm/shm.go` | Dual-language shared-memory abstraction (`0o600`, size-prefixed, unlink) |
| `kitty/constants.py` / `gen/go_code.py` | `ssh_control_master_template` constant + Go export |
| `kitty_tests/ssh.py` | PTY `check_bootstrap` harness + SSH tests |
| `kittens/ssh/{main,config,utils}_test.go` | Go tests (`TestSSHTarfile`, `TestCloneEnv`, `TestSSHBootstrapScriptLimit`, …) |
| `dev.sh` / `docs/build.rst` | Canonical build entry + build docs |

### Appendix D — Technology Versions

| Component | Version |
|---|---|
| Kitty / kitten (built) | 0.35.2 |
| Go toolchain | go1.22.12 (repo pins `go 1.22`) |
| CPython (local) | 3.13.7 (`requires-python >=3.8`) |
| C compiler | cc (Ubuntu 15.2.0) |
| OpenSSH client | OpenSSH_10.0p2 |
| Source commit (canonical) | `815df1e21` (branch `kitty_815df1e210e0`) |

### Appendix E — Environment Variable Reference

| Variable | Role in the SSH-kitten path |
|---|---|
| `KITTY_WINDOW_ID` | Required by the entry guard; part of the request-id `<KITTY_PID>-<KITTY_WINDOW_ID>` |
| `KITTY_PID` | Required by the entry guard; part of the request-id and SHM object name |
| `SSH_ASKPASS` | Exported by `set_askpass` on the kitty-askpass (push-enabling) path |
| `SSH_ASKPASS_REQUIRE` | Set to `force` when `need_to_request_data == false` (new-enough OpenSSH) |
| `KITTY_KITTEN_RUN_MODULE` | Set to `ssh_askpass` to route the askpass helper |
| `CFLAGS` | Build-time only; `-Wno-error=switch` de-promotes the Ubuntu 25.10 library warning |
| `GOFLAGS` | `-mod=mod` if module resolution is needed during `go build`/`go test` |

### Appendix F — Developer Tools Guide

- **Build:** `./dev.sh build` (wraps `go run bypy/devenv.go`) — see Section 9.2.
- **Go tests:** `go test -count=1 <pkg>` — use `-v` for per-test `--- PASS` markers; use `-count=1` to bypass the test cache.
- **Python tests:** `./test.py --module ssh` — runs the `kitty_tests/ssh.py` PTY harness (`check_bootstrap`).
- **Static/compile checks:** `go build ./…` (compile only), `go mod verify` (module integrity).
- **Scope audit:** `git diff --name-status 815df1e21..HEAD` and `git status --porcelain`.
- **Markdown lint (optional, path-to-production):** `markdownlint blitzy/documentation/*.md` — the deliverable already passes (MD009 nit fixed; 146 balanced fences; valid UTF-8; no CRLF/BOM).

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **SSH kitten** | Kitty's `kitten ssh` subcommand that wraps `ssh` to inject shell integration and shared-memory credential passing |
| **Bootstrap script** | The generated `sh`/`py` script run on the remote host to fetch shell-integration assets and exec the login shell |
| **SHM (shared memory)** | POSIX `/dev/shm` object (`kssh-…`) carrying `{tarfile, pw, hostname, username}`, created and consumed locally, never transmitted |
| **DCS** | Device Control String — the terminal escape channel (`\033P@kitty-<verb>|<base64>\033\\`) used for local↔remote coordination |
| **ControlMaster** | OpenSSH connection-multiplexing feature the kitten uses to reuse an existing connection |
| **Push vs Pull** | Credential delivery: kitten writes the request down the TTY (push) vs. the remote requesting via `SSH_ASKPASS`/`REQUEST_DATA=1` (pull) |
| **`script_type`** | `sh` (POSIX bootstrap, char-substitution encoded) or `py` (Python bootstrap, base64 encoded) |
| **Observed / Inferred / Non-canonical** | Evidence labels: seen at runtime on the real path / read from source / obtained from a harness or stand-in (not the pure real path) |
| **Canonical build** | Default-configuration build (`./dev.sh build`) run as a normal user, per the governing rules |

---

*This Blitzy Project Guide was generated from the Agent Action Plan, the autonomous agent action logs, and independent verification performed during assessment (git analysis, re-run tests, citation spot-checks, and tested build/run commands). All hours and percentages are consistent across Sections 1.2, 2.1, 2.2, 7, and 8: **Total 48.5 h · Completed 42.0 h · Remaining 6.5 h · 86.6% complete**.*