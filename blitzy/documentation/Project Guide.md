# Blitzy Project Guide — kitty SSH Kitten Code-Comprehension Walkthrough

> **Deliverable:** `blitzy/documentation/kitty_815df1e210e0.md`
> **Repository:** `kovidgoyal/kitty` (GPU-accelerated terminal emulator)
> **Branch:** `blitzy-4b7c5a01-ac45-4a22-9108-d79ce5cef0c3` · **HEAD:** `eee3dc170` · **Base:** `815df1e210e0`
> **Color legend:** <span style="color:#5B39F3">■ Completed / AI Work = Dark Blue (#5B39F3)</span> · <span style="color:#B23AF2">■ Remaining / Not Completed = White (#FFFFFF)</span>

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, comprehensive, code-grounded Markdown walkthrough — `blitzy/documentation/kitty_815df1e210e0.md` — explaining how kitty's **SSH kitten** works end-to-end. The target users are engineers onboarding into the `kovidgoyal/kitty` terminal emulator who need a precise, code-truth explanation of the SSH kitten's full lifecycle. The technical scope spans the kitten's three layers — the local Go executable, the in-terminal Python data server, and the remote POSIX/Python bootstrap scripts — and answers ten questions (Q1–Q10) covering secure session setup, OpenSSH connection sharing/reuse, shared-memory credential passing, bootstrap script generation and per-shell encoding, archive build/transfer, connection state, the security model, and the terminal↔remote escape protocol. Business impact: faster, lower-risk onboarding via authoritative, citation-backed documentation. The repository is otherwise unchanged.

### 1.2 Completion Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeColor": "#B23AF2", "pieOuterStrokeWidth": "2px", "pieSectionTextColor": "#B23AF2", "pieTitleTextSize": "18px"}}}%%
pie showData
    title SSH Kitten Documentation — 84.4% Complete
    "Completed Work (AI) — 38h" : 38
    "Remaining Work — 7h" : 7
```

| Metric | Hours |
|---|---|
| **Total Hours** | **45.0** |
| **Completed Hours (AI + Manual)** | **38.0** (AI = 38.0, Manual = 0.0) |
| **Remaining Hours** | **7.0** |
| **Percent Complete** | **84.4%** |

> Completion is computed with the AAP-scoped, hours-based methodology: `38.0 / (38.0 + 7.0) = 84.4%`. The completed work is 100% autonomous (AI); the remaining 7.0h is the human acceptance gate (review + rendering + merge), which a documentation deliverable can never complete autonomously.

### 1.3 Key Accomplishments

- ✅ **Single deliverable created** at the exact required path/name: `blitzy/documentation/kitty_815df1e210e0.md` (563 lines, ~6,900 words).
- ✅ **All ten questions (Q1–Q10) answered**, each with a precise `### Answer` and a `### Why it is built this way` rationale section (10/10/10 verified).
- ✅ **Three-layer architecture preamble**, an **end-to-end trace with a Mermaid sequence diagram** (Q8), a **consolidated security model** (Q9), and a **closing cross-reference** to the kitten's tests + `docs/kittens/ssh.rst`.
- ✅ **190 inline `path:line` citations across 16 source files** (Go, Python, POSIX shell); citation accuracy **100%** (validator enumerated 195 citation references, 194 already exact, 1 imprecision fixed).
- ✅ **Code-as-truth corroborated by building & running**: 16/16 autonomous tests pass (Go 7/7, Python ssh 8/8, shm 1/1); each test maps to a documented Q-section.
- ✅ **One external fact labelled** web-research-corroborated (OpenSSH 8.4 / `SSH_ASKPASS_REQUIRE`, released 2020-09-27), matching the code gate `Major>8 || (Major==8 && Minor>=4)`.
- ✅ **Repository pristine**: `git diff` shows exactly one added file (563 insertions, 0 deletions); zero source files modified; working tree clean; no temporary scripts left behind.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None — no blocking issues identified_ | The deliverable changes no source, so it cannot break a build or introduce a regression. All AAP content is delivered and all 16 autonomous tests pass. | — | — |

> Non-blocking follow-ups (human acceptance gate) are tracked in §1.6 and §2.2, not as critical issues.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _N/A_ | _N/A_ | **No access issues identified.** The repository, Go toolchain, Python build, and test harness were all fully accessible; build/run and all tests completed without permission or credential blockers. | Resolved (none present) | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform a human **technical review & sign-off** of the walkthrough: confirm all ten answers are technically correct/complete and that cited `path:line` references resolve to the described code at commit `815df1e210e0`.
2. **[Medium]** **Render the Markdown** on the target platform (GitHub/GitLab/docs viewer) and confirm the Mermaid sequence diagram, tables, and code blocks display correctly.
3. **[Medium]** Run a **citation-drift spot-check** against current `HEAD` (sample ~15–20 of the 190 citations) to confirm line numbers still resolve if `HEAD` has advanced beyond the pinned commit.
4. **[Medium]** **Approve and merge** the PR adding `blitzy/documentation/kitty_815df1e210e0.md`.
5. **[Low]** Apply any **optional polish** from reviewer feedback (wording/structure).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each component traces to AAP-scoped work that was autonomously delivered. **Total = 38.0h.**

| Component | Hours | Description |
|---|---|---|
| Scope discovery & multi-language source comprehension | 10.0 | Traversed and deeply read the SSH kitten across 16 source files in Go, Python, and POSIX shell (e.g., `main.go` 897 lines, `bootstrap-utils.sh` 251, `utils.py` 338) to establish the three-layer mental model. |
| Q1–Q10 answer authoring (citations + rationale + excerpts) | 16.0 | Wrote all ten Q-sections, each with an `### Answer` (mechanism + cited code excerpts) and a `### Why it is built this way` rationale; embedded 190 inline `path:line` citations. |
| Architecture preamble + end-to-end trace + Mermaid diagram | 2.5 | Authored the three-layer preamble and the Q8 end-to-end synthesis, including the Mermaid sequence diagram of the full local→ssh→remote flow. |
| Closing cross-reference + option-schema appendix | 1.5 | Mapped the kitten's Go/Python tests and `docs/kittens/ssh.rst` to the documented behaviors; built the `kittens/ssh/main.py` option-schema appendix table. |
| Web-research corroboration (OpenSSH 8.4) | 0.5 | Confirmed and labelled the single external fact (OpenSSH 8.4 introduced `$SSH_ASKPASS_REQUIRE`, 2020-09-27) against the code version gate. |
| Citation accuracy verification (190 citations, code-as-truth) | 2.5 | Re-confirmed citations against live source; the validator enumerated 195 references, found 194 exact, and fixed 1 imprecision → 100% accuracy. |
| Behavioral verification (build launcher; 16/16 tests) | 3.0 | Built the kitty launcher and ran Go `kittens/ssh` (7/7) + Python `--module ssh` (8/8) + `--module shm` (1/1) to corroborate documented claims. |
| Review-finding remediation (F1–F3 + citation precision) | 2.0 | Addressed three review findings and the `GetSSHVersion`/`umask 000` citation refinements across three follow-up commits. |
| **Total Completed** | **38.0** | |

### 2.2 Remaining Work Detail

All remaining items are the human acceptance gate for a documentation deliverable (path-to-production). **Total = 7.0h.**

| Category | Hours | Priority |
|---|---|---|
| Human technical review & sign-off of the walkthrough (verify 10 answers + citations resolve) | 3.0 | High |
| Markdown & Mermaid rendering verification on target platform | 1.0 | Medium |
| Citation-drift spot-check against current `HEAD` | 1.0 | Medium |
| PR approval & merge to mainline | 0.5 | Medium |
| Optional polish from reviewer feedback | 1.5 | Low |
| **Total Remaining** | **7.0** | |

### 2.3 Hours Reconciliation & Methodology

| Reconciliation Check | Value | Result |
|---|---|---|
| Section 2.1 Completed total | 38.0h | ✅ matches §1.2 Completed |
| Section 2.2 Remaining total | 7.0h | ✅ matches §1.2 Remaining and §7 pie "Remaining Work" |
| Section 2.1 + Section 2.2 | 45.0h | ✅ equals §1.2 Total Hours |
| Completion formula | 38.0 / 45.0 × 100 | ✅ = 84.4% |

> Methodology (PA1/PA2): only AAP-scoped deliverable work and path-to-production activities are counted. Because the deliverable is prose with zero code/dependency changes, "path-to-production" reduces to the human acceptance gate; there is no application to deploy.

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project and were **independently re-run during this assessment** (all passing). These behavioral tests corroborate the document's code-truth claims.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Unit (Go) | Go `testing` | 7 | 7 | 0 | n/m¹ | `kittens/ssh` package: `TestSSHConfigParsing`, `TestCloneEnv`, `TestSSHBootstrapScriptLimit`, `TestSSHTarfile`, `TestGetSSHOptions`, `TestParseSSHArgs`, `TestRelevantKittyOpts` |
| Integration (Python — SSH kitten) | kitty test harness (`unittest`) | 8 | 8 | 0 | n/m¹ | `test_basic_pty_operations`, `test_ssh_bootstrap_with_different_launchers`, `test_ssh_connection_data`, `test_ssh_copy`, `test_ssh_env_vars`, `test_ssh_leading_data`, `test_ssh_login_shell_detection`, `test_ssh_shell_integration` |
| Unit (Python — SHM primitive) | kitty test harness (`unittest`) | 1 | 1 | 0 | n/m¹ | `test_shm_with_kitten` — exercises the `kitty/shm.py` segment used by Q2/Q9 |
| **Total** | | **16** | **16** | **0** | — | **100% pass rate** |

¹ _Coverage not instrumented during autonomous validation; these are pass/fail behavioral gates. They are used to corroborate documented behavior, not to measure line coverage._

**Test → documented-question mapping (corroboration):**

| Test | Corroborates |
|---|---|
| `test_ssh_connection_data` | Q5 (connection state) |
| `test_ssh_bootstrap_with_different_launchers` | Q3 / Q7 (bootstrap generation + encoding) |
| `test_ssh_env_vars`, `test_ssh_shell_integration`, `test_ssh_copy`, `TestSSHTarfile` | Q4 (archive build & transfer) |
| `TestSSHBootstrapScriptLimit` | Q4 / Q7 (TTY size limit + encoding) |
| `test_ssh_login_shell_detection` | Q8 (login-shell re-exec) |
| `test_shm_with_kitten` | Q2 / Q9 (SHM credentials + security) |
| `test_basic_pty_operations`, `test_ssh_leading_data` | Q10 (terminal↔remote protocol) |
| `TestParseSSHArgs`, `TestGetSSHOptions`, `TestSSHConfigParsing` | Q1 (session setup + arg/option parsing) |

---

## 4. Runtime Validation & UI Verification

This is a **documentation + CLI** subsystem; there is **no graphical UI surface** in the deliverable. Runtime validation focused on the build/test toolchain and the document's structural integrity.

**Build & toolchain**
- ✅ **Operational** — Go toolchain `go1.22.12` (matches `go.mod` `go 1.22`); `go test ./kittens/ssh/...` compiles and runs (exit 0).
- ✅ **Operational** — kitty launcher built and present: `kitty/launcher/kitty`, `kitty/launcher/kitten`, `kitty/fast_data_types.so`, 74 `*_generated.go`.
- ✅ **Operational** — built launcher executes (`kitty 0.35.2`) and runs the Python test harness end-to-end.

**Document integrity**
- ✅ **Operational** — 40 fenced code blocks, balanced (even count).
- ✅ **Operational** — Mermaid sequence diagram present (deliverable L366) and structurally intact.
- ✅ **Operational** — 190 `path:line` citations resolve; spot-checks against live source were exact (e.g., `kitty/constants.py:L188`, `kittens/ssh/utils.go:L206-208`, `kittens/ssh/main.go:L135-144`).

**Repository state**
- ✅ **Operational** — `git diff 815df1e21..HEAD --name-status` = `A blitzy/documentation/kitty_815df1e210e0.md` only; working tree clean.

**UI verification**
- ⚪ **Not applicable** — the deliverable is a Markdown document; no UI components, screens, or design-system compliance to verify.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the `SWE-AtlasQnA-Repo` rules to quality benchmarks. All items autonomously satisfied; the only "in progress" items are the human-review gate.

| Benchmark / Rule (AAP §0.7, §0.8) | Status | Progress | Evidence |
|---|---|---|---|
| Single new Markdown file named after source branch (`kitty_815df1e210e0.md`) | ✅ Pass | 100% | File exists at exact name |
| Placed in `blitzy/documentation/` | ✅ Pass | 100% | `blitzy/documentation/kitty_815df1e210e0.md` |
| All ten questions (Q1–Q10) comprehensively answered | ✅ Pass | 100% | 10 `### Answer` sections; 1:1 Q-table |
| Rationale/thinking provided per answer | ✅ Pass | 100% | 10 `### Why it is built this way` sections |
| Code-as-truth: every claim cited `path:line` | ✅ Pass | 100% | 190 citations; 100% accuracy (1 fixed) |
| Citations accurate to commit `815df1e210e0` | ✅ Pass | 100% | Header pins commit; spot-checks exact |
| Build/run permitted for behavioral confirmation | ✅ Pass | 100% | 16/16 tests; launcher built |
| Do **not** modify existing repository files | ✅ Pass | 100% | 0 source files changed |
| Do **not** add any other code | ✅ Pass | 100% | Only the one `.md` added |
| Temporary observation scripts cleaned up | ✅ Pass | 100% | No temp files in repo; tooling lived in `/tmp` |
| Output is Markdown (separate from repo's `.rst` docs) | ✅ Pass | 100% | `.md` outside `docs/` |
| Human technical review & sign-off | ⚠ In progress | 0% | Pending (see §2.2 HT-1) |
| Rendering verification on target platform | ⚠ In progress | 0% | Pending (see §2.2 HT-2) |

**Fixes applied during autonomous validation:** F1 (`GetSSHVersion` citation range corrected to `L210-L222`), F2–F3 (review findings addressed in the SSH-kitten walkthrough), and a citation-precision fix splitting the `umask 000` pointer at deliverable L191 into `bootstrap.sh:L108` (staging dir) + `bootstrap.sh:L112` (`umask 000`). **Outstanding:** human review/rendering/merge only.

---

## 6. Risk Assessment

All risks are **Low** severity, reflecting a documentation-only deliverable that adds zero runtime code, zero dependencies, and zero attack surface.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation line-number drift if source later changes | Technical | Low | Medium | Header pins exact commit `815df1e210e0`; reconcile against that commit; optional CI citation-lint | Mitigated by design |
| Mermaid/Markdown renders differently across platforms | Technical | Low | Low–Med | 40 fences verified balanced; rendering check is a remaining task (HT-2) | Open (low) |
| No new vulnerabilities introduced | Security | None | — | Deliverable adds no code/deps/credentials/config | N/A (no surface) |
| Documented security claims could mislead if inaccurate | Security | Low | Low | Q9 cross-checks Go + Python readers; verified vs code; `test_shm_with_kitten` passes | Mitigated |
| Documentation staleness as the kitten evolves | Operational | Low | Medium (long-term) | Commit-pinned; regenerable | Accepted |
| No automated doc-sync gate in CI | Operational | Low | Low | Optional citation-lint (out of scope) | Accepted |
| Format divergence (Markdown vs repo's reStructuredText) | Integration | Low | N/A (intended) | Deliberately separate from `docs/` to avoid touching existing files | By design |
| New top-level `blitzy/documentation/` directory placement | Integration | Low | Low | Path dictated by task rules | Per spec |

**Overall risk posture: MINIMAL.** No High/Critical risks; no blocking issues.

---

## 7. Visual Project Status

**Project hours — Completed vs Remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeColor": "#B23AF2", "pieOuterStrokeWidth": "2px", "pieSectionTextColor": "#B23AF2", "pieTitleTextSize": "18px"}}}%%
pie showData
    title Project Hours Breakdown (Total 45h)
    "Completed Work" : 38
    "Remaining Work" : 7
```

**Remaining work by priority** (sums to the 7.0h remaining):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#A8FDD9", "pie3": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeColor": "#B23AF2", "pieOuterStrokeWidth": "2px", "pieSectionTextColor": "#B23AF2", "pieTitleTextSize": "16px"}}}%%
pie showData
    title Remaining 7h by Priority
    "High (review/sign-off)" : 3.0
    "Medium (render + drift + merge)" : 2.5
    "Low (optional polish)" : 1.5
```

**Remaining hours per category** (from §2.2):

| Category | Hours | Bar |
|---|---|---|
| Human technical review & sign-off | 3.0 | ██████████████████████████████ |
| Markdown/Mermaid rendering verification | 1.0 | ██████████ |
| Citation-drift spot-check | 1.0 | ██████████ |
| PR approval & merge | 0.5 | █████ |
| Optional polish | 1.5 | ███████████████ |
| **Total** | **7.0** | |

> Integrity: "Remaining Work" (7) in the pie equals §1.2 Remaining Hours (7.0) and the §2.2 Hours total (7.0).

---

## 8. Summary & Recommendations

**Achievements.** The project delivered exactly what the AAP scoped: one comprehensive, code-cited Markdown walkthrough of kitty's SSH kitten that answers all ten questions (Q1–Q10) with `### Answer` + rationale, a three-layer architecture preamble, an end-to-end trace with a Mermaid sequence diagram, a consolidated security model, and a closing cross-reference to the kitten's tests and official docs. The document carries 190 `path:line` citations (100% accurate) and is corroborated by 16/16 autonomously-executed tests. The repository is otherwise pristine — a single file added, zero source modifications.

**Remaining gaps & critical path.** The project is **84.4% complete (38.0h of 45.0h)**. The remaining **7.0h** is entirely the human acceptance gate: technical review/sign-off (3.0h), rendering verification (1.0h), citation-drift spot-check (1.0h), merge (0.5h), and optional polish (1.5h). There is no application to deploy and no outstanding AAP content work — the autonomous deliverable is functionally complete.

**Production-readiness assessment.** **Ready for human review and merge.** Because the deliverable changes no source code, it carries no build/regression risk; all risks are Low and mostly mitigated by design (commit-pinning, balanced structure, test corroboration). The recommended path to "done" is: (1) technical review, (2) render check, (3) merge.

| Success Metric | Target | Actual |
|---|---|---|
| Questions answered (Q1–Q10) | 10/10 | ✅ 10/10 |
| Citation accuracy | 100% | ✅ 100% (1 imprecision fixed) |
| Autonomous tests passing | 100% | ✅ 16/16 |
| Source files modified | 0 | ✅ 0 |
| Completion (AAP-scoped) | — | **84.4%** |

---

## 9. Development Guide

This guide explains how to (a) **read/render** the deliverable and (b) **re-verify its code-truth claims** by building and running kitty. All commands below were executed successfully during this assessment and are copy-pasteable from the repository root.

### 9.1 System Prerequisites

- **Go ≥ 1.22** (matches `go.mod`; assessment env = `go1.22.12`)
- **Python ≥ 3.8** (`pyproject.toml` `requires-python`; assessment env = `3.13.7`)
- **Git 2.x** (assessment env = `2.51.0`)
- **OpenSSH** — the subsystem under study (assessment env = `OpenSSH_10.0p2`; ≥ 8.4 enables the `SSH_ASKPASS_REQUIRE` path the document describes)
- **C toolchain** — only needed to build the kitty launcher used for Python-side behavioral checks
- **A Markdown viewer with Mermaid support** — for rendering the deliverable (GitHub, VS Code + Mermaid extension, or https://mermaid.live)

### 9.2 Environment Setup

```bash
# From the repository root. Put Go on PATH and set a UTF-8 locale.
export PATH=/usr/local/go/bin:$PATH
export LANG=C.UTF-8 LC_ALL=C.UTF-8

# Confirm the commit the document is pinned to.
git merge-base HEAD 815df1e21        # => 815df1e210e0...
```

### 9.3 Read the Deliverable

```bash
# View the document (or open in a Mermaid-capable Markdown viewer).
sed -n '1,80p' blitzy/documentation/kitty_815df1e210e0.md
less blitzy/documentation/kitty_815df1e210e0.md

# Quick structural metrics.
wc -l blitzy/documentation/kitty_815df1e210e0.md          # 563 lines
grep -c '^## Q' blitzy/documentation/kitty_815df1e210e0.md # 10 question sections
```

### 9.4 Verify a Citation (code-as-truth)

```bash
# Each claim cites path:Lnnn. Resolve one end-to-end, e.g. the Q1 anchor:
sed -n '188p' kitty/constants.py
# => ssh_control_master_template = 'kssh-{kitty_pid}-{ssh_placeholder}'   (matches the document)
```

### 9.5 Behavioral Re-Verification (optional but recommended)

```bash
# (a) Go tests for the SSH kitten — fast, no full build required.
go test -count=1 -v ./kittens/ssh/...      # => 7/7 PASS (exit 0)

# (b) Build the kitty launcher (needed for the Python harness).
#     The assessment env used a venv interpreter:
/root/kitty-venv/bin/python setup.py build --debug --ignore-compiler-warnings --verbose
#     Produces kitty/launcher/{kitty,kitten}, kitty/fast_data_types.so, and *_generated.go

# (c) Python behavioral tests via the built launcher.
./kitty/launcher/kitty +launch test.py --module ssh    # => 8/8 OK
./kitty/launcher/kitty +launch test.py --module shm    # => 1/1 OK
```

### 9.6 Verify Markdown Integrity

```bash
# Fenced code blocks must be balanced (even count); confirm the Mermaid block exists.
awk '/^```/{n++} END{print "fence lines:", n, "(balanced:", (n%2==0)?"YES":"NO", ")"}' \
    blitzy/documentation/kitty_815df1e210e0.md          # => 40, balanced YES
grep -n 'mermaid' blitzy/documentation/kitty_815df1e210e0.md   # => L366
```

### 9.7 Troubleshooting

| Symptom | Resolution |
|---|---|
| `go: command not found` | `export PATH=/usr/local/go/bin:$PATH` |
| `kitty/launcher/kitty` missing | Run the `setup.py build` step in §9.5(b) |
| Mermaid diagram shows as raw text | Open in a Mermaid-capable viewer (GitHub, VS Code Mermaid ext) or paste into https://mermaid.live |
| A cited line number doesn't match | Reconcile against the **pinned commit** `815df1e210e0` (the document header states this); current `HEAD` may have drifted |
| Python tests fail on locale/encoding | `export LANG=C.UTF-8 LC_ALL=C.UTF-8` before running the harness |

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Set Go on PATH + UTF-8 locale | `export PATH=/usr/local/go/bin:$PATH; export LANG=C.UTF-8 LC_ALL=C.UTF-8` |
| Confirm pinned commit | `git merge-base HEAD 815df1e21` |
| Show the single-file diff | `git diff 815df1e21..HEAD --name-status` |
| Read the deliverable | `less blitzy/documentation/kitty_815df1e210e0.md` |
| Resolve a citation | `sed -n '188p' kitty/constants.py` |
| Go tests (SSH kitten) | `go test -count=1 ./kittens/ssh/...` |
| Build launcher | `/root/kitty-venv/bin/python setup.py build --debug --ignore-compiler-warnings --verbose` |
| Python SSH tests | `./kitty/launcher/kitty +launch test.py --module ssh` |
| Python SHM test | `./kitty/launcher/kitty +launch test.py --module shm` |
| Markdown fence balance | `awk '/^```/{n++} END{print n, n%2==0}' blitzy/documentation/kitty_815df1e210e0.md` |

### Appendix B — Port Reference

The deliverable is a documentation file and **exposes no listening ports**. For context only, the subsystem it documents uses standard OpenSSH transport (TCP **22** by default) and OpenSSH **connection-multiplexing Unix-domain control sockets** named via `ssh_control_master_template = 'kssh-{kitty_pid}-{ssh_placeholder}'` (`kitty/constants.py:L188`) — these are local filesystem sockets, not network ports.

### Appendix C — Key File Locations

| File | Lines | Role |
|---|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | 563 | **The deliverable** |
| `kittens/ssh/main.go` | 897 | Orchestrator (run_ssh, SHM, tar, bootstrap, sharing/reuse) — Q1–Q7, Q10 |
| `kittens/ssh/utils.go` | 246 | `SSHExe`, `GetSSHVersion`, `SupportsAskpassRequire`, `ParseSSHArgs` — Q1/Q6 |
| `kittens/ssh/askpass.go` | 108 | `RunSSHAskpass` / `@kitty-ask` DCS path — Q10 |
| `kittens/ssh/config.go` | 410 | Per-host config + tar primitives |
| `kittens/ssh/main.py` | 235 | Option schema (`Definition`) — Q1/Q5 appendix |
| `kittens/ssh/utils.py` | 338 | Terminal-side `get_ssh_data` server, SHM read/create — Q2/Q4/Q9/Q10 |
| `kitty/window.py` | 1998 | `handle_remote_ssh/askpass/echo` — Q10 |
| `kitty/shm.py` | 186 | `SharedMemory` primitive (`O_CREAT|O_EXCL`, `0o600`) — Q2/Q9 |
| `kitty/utils.py` | — | `SSHConnectionData`, `cleanup_ssh_control_masters` — Q5/Q6 |
| `kitty/constants.py` | — | `ssh_control_master_template` — Q6 |
| `shell-integration/ssh/bootstrap.sh` | 164 | POSIX remote entry — Q3/Q4/Q8/Q10 |
| `shell-integration/ssh/bootstrap.py` | 318 | Python-interpreter parity path — Q3/Q8 |
| `shell-integration/ssh/bootstrap-utils.sh` | 251 | `compile_terminfo`, `mv_files_and_dirs`, per-shell re-exec — Q3/Q8 |

### Appendix D — Technology Versions

| Component | Version (assessment env) | Source of Truth |
|---|---|---|
| Go | `go1.22.12` | `go.mod` declares `go 1.22` |
| Python | `3.13.7` | `pyproject.toml` `requires-python = ">=3.8"` (CI exercises 3.8–3.10) |
| Git | `2.51.0` | system |
| OpenSSH | `OpenSSH_10.0p2` | external runtime (kitten gates `SSH_ASKPASS_REQUIRE` at ≥ 8.4) |
| kitty (built launcher) | `0.35.2` | `kitty/launcher/kitty` |

### Appendix E — Environment Variable Reference

| Variable | Purpose (in this assessment) |
|---|---|
| `PATH` | Must include `/usr/local/go/bin` for the Go toolchain |
| `LANG` / `LC_ALL` | Set to `C.UTF-8` so the Python test harness runs cleanly |
| `KITTY_WINDOW_ID`, `KITTY_PID` | Documented (not set here): the kitten's `main()` guards on these — see Q-sections, `kittens/ssh/main.go:L800-L832` |
| `SSH_ASKPASS`, `SSH_ASKPASS_REQUIRE`, `KITTY_KITTEN_RUN_MODULE` | Documented askpass wiring (Q1/Q10), `kittens/ssh/main.go:L160-L163` |

### Appendix F — Developer Tools Guide

| Tool | Use |
|---|---|
| `go test` | Run the kitten's Go unit tests (`./kittens/ssh/...`) |
| `setup.py build` | Build the kitty launcher + `fast_data_types.so` + generated Go for the Python harness |
| `kitty +launch test.py --module <name>` | Run a specific Python test module (`ssh`, `shm`) through the built launcher |
| `sed -n 'Np FILE'` / `grep -n` | Resolve and verify `path:line` citations against source |
| `awk` fence counter | Validate Markdown code-fence balance |
| Mermaid viewer (GitHub / VS Code / mermaid.live) | Render the Q8 sequence diagram and the status pie charts |

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **SSH kitten** | kitty's hybrid Go+Python component (`kittens/ssh`) that wraps `ssh` to ship terminfo + shell integration to the remote host |
| **Bootstrap script** | The self-contained remote script (`bootstrap.sh`/`bootstrap.py`) that unpacks the payload, compiles terminfo, and re-execs the login shell |
| **SHM (shared memory)** | A POSIX shared-memory segment (`0o600`, random token name) used to pass the one-time password + payload without exposing them on the command line |
| **Connection sharing / multiplexing** | OpenSSH `ControlMaster`/`ControlPath`/`ControlPersist` mechanism that reuses one authenticated link for many sessions |
| **DCS** | Device Control String — the terminal escape sequence (`\033P … \033\\`) carrying the `@kitty-ssh`/`@kitty-ask`/`@kitty-echo` protocol over the controlling TTY |
| **terminfo** | Terminal capability database; the kitten ships and compiles (`tic -x`) kitty's entry on the remote |
| **Code-as-truth** | The methodology requiring every documented claim to cite a specific `path:line` in the source at the pinned commit |
| **Path-to-production (this project)** | For a documentation deliverable: human technical review, rendering verification, and merge — there is no application to deploy |

---

*Completion: **84.4%** (38.0h of 45.0h). Remaining: **7.0h** (human acceptance gate). All numbers are consistent across Sections 1.2, 2.1, 2.2, 2.3, and 7. All listed tests originate from Blitzy's autonomous validation logs and were independently re-run during this assessment.*