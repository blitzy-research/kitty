# Blitzy Project Guide

> **Brand color legend** — **Completed / AI Work: Dark Blue `#5B39F3`** · **Remaining / Not Completed: White `#FFFFFF`** · Headings/Accents: Violet-Black `#B23AF2` · Highlight: Mint `#A8FDD9`. These colors are applied to every chart in this guide.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, code-grounded technical reference — `blitzy/documentation/kitty_815df1e210e0.md` — that explains, end to end, how the **kitty** terminal emulator's remote-control system processes a `kitten @ ls` command. Aimed at developers who will extend kitty with new remote-control commands, it traces the full path from client invocation to JSON response across kitty's **Python core, Go client, and C child-monitor**. Every claim is grounded in source citations at commit `815df1e210e0` and corroborated by **live capture** from a headless build. The scope is **documentation only** — no kitty source file is modified. Business impact: it removes a significant onboarding barrier for contributors and de-risks the user's planned custom remote-control command.

### 1.2 Completion Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeColor":"#B23AF2","pieSectionTextColor":"#111111","pieTitleTextColor":"#B23AF2","pieLegendTextColor":"#111111"}}}%%
pie showData title Completion Status — 93.75% Complete (45h of 48h)
    "Completed Work (AI)" : 45
    "Remaining Work" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **48** |
| **Completed Hours (AI + Manual)** | **45** (45 AI + 0 Manual) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **93.75%** |

> Completion is computed on **AAP-scoped + path-to-production** work only (PA1): `45 ÷ (45 + 3) = 93.75%`. Because this is a documentation artifact (no deployable service), the remaining 3h is genuine **human acceptance** work — no CI/infra/deployment work is fabricated.

### 1.3 Key Accomplishments

- ✅ Authored a **1,049-line, 12-section** code-grounded trace that explicitly answers all four user questions (transport · destination discovery / "why `/tmp` was empty" · shell-integration "without explicit configuration" · end-to-end parse/route/JSON/logging).
- ✅ Verified **67 distinct source citations** against commit `815df1e210e0` (6/6 independent spot-checks exact; **4 off-by-one citations corrected** during validation).
- ✅ Built **kitty 0.35.2** clean (`python3 setup.py` → exit 0) and ran it **headless under Xvfb**.
- ✅ Captured **real on-the-wire bytes**: 45-byte request frame and 11,025-byte response frame (hexdumped), plus the **live `ls` JSON window tree** (9 OS-window / 11 tab / 17 window keys).
- ✅ Reproduced the `docs/rc_protocol.rst` `socat` one-liner against a live socket; validated **abstract sockets** (`unix:@name`) and **malformed-frame logging**.
- ✅ Corroborated transport/protocol facts against kitty's **official documentation**.
- ✅ **Zero kitty source files modified** — full SWE-AtlasQnA-Repo compliance; working tree clean (only the deliverable added, +1,049/-0).

### 1.4 Critical Unresolved Issues

**No critical (release-blocking) issues.** The items below are **low-severity, non-blocking** follow-ups carried forward as standard human acceptance steps.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| SME has not yet independently reviewed the document | Authoritative reference should be human-validated before the team relies on it | Reviewing Engineer (SME) | 1.5h |
| Live-capture appendix not yet reproduced in target environment | Confirms portability of build/run steps beyond the Blitzy container | Reviewing Engineer | 0.5h |
| Citations are pinned to commit `815df1e210e0` | Line numbers will drift on a future kitty version bump | Maintainer | On version bump |

### 1.5 Access Issues

**No access issues identified.**

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Repository (kitty @ `815df1e210e0`) | Read/Write | None — repository accessible, working tree clean | ✅ Resolved | — |
| Build toolchain (Python/Go/gcc/Xvfb) | Local execution | None — all present and functional | ✅ Resolved | — |
| Credentials / third-party APIs | N/A | None required for a documentation deliverable | ✅ N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** SME end-to-end technical review of `kitty_815df1e210e0.md` — read all 12 sections, verify explanations, and spot-check a sample of the 67 citations against source at commit `815df1e210e0`; confirm all four questions are answered. *(1.5h)*
2. **[High]** Reproduce the §11 live-capture appendix in your environment — build kitty, run headless under `xvfb-run`, then run the `socat` one-liner and `kitten @ ls` to confirm the wire frame and JSON match the document. *(0.5h)*
3. **[Medium]** Stakeholder/maintainer **acceptance & merge sign-off** of the documentation deliverable. *(0.5h)*
4. **[Low]** Governance follow-ups — confirm secret-redaction practice for any re-shared `ls` output, and record the commit-pinned citation caveat for future version bumps. *(0.5h)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| RC control-flow investigation & tracing | 11 | End-to-end trace of `kitten @ ls` across **three languages** (Python core, Go client, C child-monitor); ~20 reference files; identification of 67 citation sites *(AAP R1–R9)* |
| Document authoring | 13 | 12 numbered sections + appendices, 1,049 lines / 7,219 words; narrative rationale, annotated code excerpts, Mermaid flowchart *(AAP R1–R14)* |
| Build environment setup & kitty build | 4 | Python 3.13 venv, C (gcc) + Go 1.24 toolchain, 20 pkg-config modules, Xvfb; `setup.py` build to exit 0 *(AAP R12)* |
| Live capture & evidence collection | 7 | Real socket paths (incl. abstract); 45-byte request + 11,025-byte response hexdumps; live `ls` JSON tree; `socat` one-liner reproduction; malformed-frame log; **secret redaction**; cleanup *(AAP R4a/R4b/R4d/R4e/R7/R8/R18)* |
| Official-documentation corroboration | 2 | Web-search validation of transport/protocol facts vs kitty's remote-control / rc_protocol / invocation pages *(AAP R17)* |
| Citation verification & production-readiness gates | 5 | ~75 citations checked against source; 5 gates (build / tests / runtime / scope / AAP) *(AAP R13)* |
| QA / review fix cycles | 3 | 4 fix commits: review findings; SIGTERM socket lifecycle + §7.5 window keys; shell-integration env accuracy; 4 off-by-one citations *(AAP R13/R14)* |
| **Total Completed** | **45** | **Matches Completed Hours in §1.2** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Documentation SME Review & Verification (read-through + citation spot-check + reproduce live-capture appendix) | 2 | High |
| Acceptance, Sign-off & Governance Notes (stakeholder acceptance + redaction/citation-drift governance) | 1 | Medium |
| **Total Remaining** | **3** | **Matches Remaining Hours in §1.2 and §7 pie chart** |

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project (build, kitty test harness, and headless runtime). The deliverable is documentation; the relevant suites are the **remote-control–relevant** tests plus the end-to-end live exchange.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Unit — RC Protocol & Crypto | Python unittest (`kitty_tests`) | 2 | 2 | 0 | Not measured | `test_dcs_codes` (DCS wire-frame parsing), `test_elliptic_curve_data_exchange` (encrypted RC key exchange) |
| Integration — Shell | Python unittest (`kitty_tests`) | 5 | 5 | 0 | Not measured | `test_bash_integration` ×2, `test_zsh_integration` ×2, `test_completion`; **fish skipped** (not installed) |
| End-to-End — Live RC | Headless (Xvfb + `socat`/`kitten`) | 4 | 4 | 0 | N/A | `kitten @ ls` JSON tree; `rc_protocol.rst` `socat` one-liner; abstract socket `unix:@name`; malformed-frame logging |
| Build Verification | `setup.py` | 1 | 1 | 0 | N/A | Clean build, exit 0; `kitty --version` → "kitty 0.35.2" |
| **Totals (in-scope)** | — | **12** | **12** | **0** | — | **100% pass for all in-scope / RC-relevant tests** |

> **Integrity note (Rule 3):** Every test row above is sourced from the autonomous validation logs. **Coverage % is reported as "Not measured"** rather than fabricated — line-coverage instrumentation was not part of this documentation task; the relevant guarantee is that the RC code paths described in the document were exercised and pass.
>
> **Out-of-scope, non-defect:** two environment-specific font tests (`test_font_selection`, "ubuntu mono") fail in `kitty_tests/` because Ubuntu 25.10 ships a variable-font Ubuntu Mono that shadows the test's expected static font. These are **pre-existing baseline**, reside in forbidden-to-modify `kitty_tests/`, and are **unrelated** to the deliverable. They are excluded from the totals above.

---

## 4. Runtime Validation & UI Verification

This is a documentation deliverable for a headless-run terminal emulator; there is no application GUI to verify. "Runtime" here is the **live remote-control exchange**, validated end-to-end.

**Build & Process Health**
- ✅ **Operational** — `python3 setup.py` builds clean (exit 0); artifacts present: `kitty/launcher/kitty`, `kitty/launcher/kitten` (16.4 MB), `kitty/fast_data_types.so`.
- ✅ **Operational** — `kitty --version` and `kitten --version` both report **"0.35.2"** (matches the document and `constants.py:L25`).
- ✅ **Operational** — kitty launches **headless under `xvfb-run`** (Wayland backend auto-disabled; X11 path intended).

**Remote-Control Runtime (the documented path)**
- ✅ **Operational** — `kitten @ --to unix:/tmp/… ls` returns the **real JSON OS-window tree**.
- ✅ **Operational** — the `docs/rc_protocol.rst` `socat` one-liner works against the live socket and decodes via `jq '.data | fromjson'`.
- ✅ **Operational** — **on-the-wire bytes** match the document exactly: 45-byte request frame (`ESC P @kitty-cmd … ESC \`); 11,025-byte response envelope `{"ok": true, "data": "…"}`.
- ✅ **Operational** — **abstract socket** (`unix:@name`) works with **no filesystem entry** (visible via `/proc/net/unix`).
- ✅ **Operational** — **socket lifecycle**: file created `srwxr-xr-x`; `SIGTERM` is caught → graceful shutdown → `atexit` removes the socket (`kitty/child-monitor.c:L1365-1368`, `kitty/boss.py:L181`).
- ✅ **Operational** — **malformed frame** → empty response + non-fatal log line *"Failed to parse JSON payload of remote command, ignoring it"* (`kitty/remote_control.py:L62`).

**UI Verification**
- ⚠ **Partial (by design / N/A)** — no graphical UI is in scope; the "interface" under test is the JSON/CLI response of `kitten @ ls`, which is fully validated above.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP deliverables and the **SWE-AtlasQnA-Repo** rule to quality benchmarks. All checks pass.

| Benchmark / Rule | Status | Evidence | Progress |
|------------------|--------|----------|----------|
| Document named `<source_branch_name>.md` | ✅ PASS | `kitty_815df1e210e0.md` (branch `kitty_815df1e210e0`) | 100% |
| Placed in `blitzy/documentation/` | ✅ PASS | `blitzy/documentation/kitty_815df1e210e0.md` | 100% |
| Comprehensively answers the question(s) | ✅ PASS | Dedicated "four questions" section; §1–§12 cover transport, discovery, shell-integration, parse/route/JSON/logging | 100% |
| Build & run to verify (no assumptions) | ✅ PASS | Clean `setup.py` build; headless run; live capture of bytes + JSON | 100% |
| Code-grounded citations `[path:locator]` | ✅ PASS | 67 distinct citations; 6/6 spot-checks exact at `815df1e210e0` | 100% |
| Provide thinking / rationale | ✅ PASS | "Why `/tmp` was empty", "why two transports", "why dispatch is dynamic"; 27 rationale phrases | 100% |
| Do **not** modify existing source files | ✅ PASS | `git`: 0 source files changed; only the `.md` added (+1,049/-0) | 100% |
| Do **not** add any other code | ✅ PASS | Single artifact; temporary capture scripts deleted | 100% |
| Web-search corroboration vs official docs | ✅ PASS | References to `sw.kovidgoyal.net/kitty`; X25519/AES-256-GCM corroboration | 100% |
| Temporary-script cleanup | ✅ PASS | Working tree clean; zero repo trace of capture scripts | 100% |

**Fixes applied during autonomous validation:** (1) review findings addressed; (2) SIGTERM socket-lifecycle correction; (3) §7.5 window-key set corrected to exactly 17 keys; (4) shell-integration env-var producer/consumer accuracy; (5) **4 off-by-one line citations** corrected in `remote_control.py` references.

**Outstanding quality items:** none blocking — only the human SME spot-check (§1.6 step 1) remains as standard acceptance.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation line-number drift (citations valid only at commit `815df1e210e0`) | Technical | Low | Medium | Document explicitly pins the commit and version 0.35.2; re-verify on version bump | 🟦 Documented / Mitigated |
| Residual off-by-one citation in low-traffic references | Technical | Low | Low | SME spot-check during review (HT-1) | ⬜ Open (low) |
| Secret exposure via captured env vars in `ls` output | Security | Medium (if mishandled) | Low | Real keys/passwords **redacted** in the doc; note that env is de-duplicated unless `--all-env-vars`; appendix warns | 🟦 Mitigated |
| Reproducibility drift in build/capture appendix (toolchain versions) | Operational | Low | Low–Medium | Appendix documents exact versions and caveats (`--start-as=hidden` invalid at 0.35.2 → use `minimized`; Wayland auto-disable) | 🟦 Documented / Mitigated |
| External-integration risk | Integration | Very Low | — | Deliverable integrates no services and adds no interfaces; only file placement (confirmed) | ✅ Closed |
| Misreading the 2 pre-existing font-test failures as a regression | Integration | Low (info) | — | Documented as out-of-scope, pre-existing, non-defect | 🟦 Known / Accepted |

---

## 7. Visual Project Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeColor":"#B23AF2","pieSectionTextColor":"#111111","pieTitleTextColor":"#B23AF2","pieLegendTextColor":"#111111"}}}%%
pie showData title Project Hours Breakdown (Total 48h)
    "Completed Work" : 45
    "Remaining Work" : 3
```

**Remaining Hours by Category (from §2.2)** — bars rendered in Completed-blue `#5B39F3`:

```mermaid
%%{init: {"theme":"base","themeVariables":{"xyChart":{"plotColorPalette":"#5B39F3"}}}%%
xychart-beta
    title "Remaining Hours by Category"
    x-axis ["SME Review & Verification", "Acceptance & Governance"]
    y-axis "Hours" 0 --> 3
    bar [2, 1]
```

> **Integrity check (Rule 1):** "Remaining Work" = **3h** here, equal to §1.2 Remaining Hours (3h) and the §2.2 Hours total (3h). "Completed Work" = **45h** equals §1.2 Completed Hours. The bar-chart categories (2 + 1) sum to 3h.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, validated, code-grounded reference for kitty's remote-control system. All **22 AAP requirements** (13 content + 9 process/rule) are **COMPLETED**: the document answers every user question, every claim is cited to source at commit `815df1e210e0`, and the "on-the-wire" and "JSON response" sections are backed by **live capture** from a real headless build. Compliance with the SWE-AtlasQnA-Repo rule is total — **zero source files modified**.

**Remaining gaps & critical path.** At **93.75% complete (45h of 48h)**, the only remaining work is **3h of human acceptance**: an SME read-through with citation spot-checks and a one-time reproduction of the live-capture appendix (2h), followed by stakeholder sign-off and governance notes (1h). There are **no release-blocking issues** and **no fabricated deployment work** — appropriate for a documentation artifact.

**Success metrics.** Build clean (exit 0); 12/12 in-scope tests pass; 6/6 citation spot-checks exact; request/response bytes match the document byte-for-byte; working tree clean.

**Production-readiness assessment.** **Ready for human review and merge.** The deliverable is self-contained, accurate, and reproducible. Recommended path: complete §1.6 steps 1–2 (High), then 3 (Medium), then 4 (Low). Confidence: **High** — the scope is well-defined and the work has been independently re-verified against source.

| Metric | Value |
|--------|-------|
| AAP requirements completed | 22 / 22 |
| AAP-scoped completion | 93.75% |
| In-scope test pass rate | 100% (12/12) |
| Source files modified | 0 |
| Release-blocking issues | 0 |

---

## 9. Development Guide

All commands below were tested in the Blitzy container at commit `815df1e210e0` and are copy-pasteable. Run from the **repository root**.

### 9.1 System Prerequisites

| Requirement | Version (verified) | Notes |
|-------------|--------------------|-------|
| OS | Linux x86_64 (Ubuntu 25.10) | X11 / Xvfb path (Wayland auto-disabled) |
| Python | 3.13.7 (≥ 3.8 required per `pyproject.toml`) | Runs the core + `setup.py` build |
| Go | 1.24.4 (`go.mod` requires `go 1.22`) | Builds the `kitten` Go client |
| C compiler | gcc 15.2.0 | Compiles the C extension / child-monitor |
| Capture utilities | `socat`, `jq`, `xvfb-run`, `hexdump`, `awk` | For headless run + raw-frame inspection |
| Disk | ~150 MB free | Build artifacts |

### 9.2 Environment Setup

```bash
# Activate the project virtualenv (created during build provisioning)
source /root/kitty-venv/bin/activate

# Compiler / toolchain hints
export CC=gcc
export GOTOOLCHAIN=local
```

### 9.3 Build

```bash
# Standard build (Makefile target: all)
python3 setup.py

# OR a verbose/debug build with extra event-loop logging
#   (Makefile target: debug-event-loop)
python3 setup.py build --debug --extra-logging=event-loop
```

Expected artifacts (git-ignored): `kitty/launcher/kitty`, `kitty/launcher/kitten`, `kitty/fast_data_types.so`.

### 9.4 Verification

```bash
./kitty/launcher/kitty   --version     # -> kitty 0.35.2 created by Kovid Goyal
./kitty/launcher/kitten  --version     # -> kitten 0.35.2 created by Kovid Goyal
```

### 9.5 Run Headless (for live RC capture)

```bash
# Start kitty under a virtual display with remote control enabled and a socket.
# NOTE: at 0.35.2 --start-as choices are normal/minimized/maximized/fullscreen
#       ('hidden' is NOT valid at this commit), so we use 'minimized'.
xvfb-run -a ./kitty/launcher/kitty \
  --config NONE \
  -o allow_remote_control=yes \
  --listen-on unix:/tmp/kitty-rc-demo.sock \
  --start-as=minimized \
  sh -c 'sleep 300' &
pid=$!         # capture the exact PID for safe cleanup later
```

### 9.6 Example Usage

```bash
# High-level client query — returns the JSON OS-window tree:
./kitty/launcher/kitten @ --to unix:/tmp/kitty-rc-demo.sock ls

# Raw protocol (docs/rc_protocol.rst one-liner), pretty-printed:
echo -en '\eP@kitty-cmd{"cmd":"ls","version":[0,14,2]}\e\\' \
  | socat - unix:/tmp/kitty-rc-demo.sock \
  | awk '{ print substr($0, 13, length($0) - 14) }' \
  | jq -c '.data | fromjson' | jq .

# See the literal escape-code frames on the wire:
echo -en '\eP@kitty-cmd{"cmd":"ls","version":[0,14,2]}\e\\' \
  | socat - unix:/tmp/kitty-rc-demo.sock | hexdump -C
```

### 9.7 Cleanup (safe pattern)

```bash
# Kill ONLY the exact PID captured at launch — never a broad pkill/killall.
kill "$pid"                     # SIGTERM -> graceful shutdown -> atexit removes the socket
rm -f /tmp/kitty-rc-demo.sock   # belt-and-braces (only needed after SIGKILL/crash)
```

### 9.8 Troubleshooting

- **`/tmp` has no socket** — none is created unless `--listen-on` is set **and** `allow_remote_control` ∈ {`yes`,`socket`,`socket-only`,`password`} (`kitty/boss.py:L364`). Config-file paths get `-{kitty_pid}` auto-appended, so look for `/tmp/<name>-<pid>` (or an **abstract** socket with no file at all).
- **`kitten @ ls` errors/hangs with no `--to`** — outside a kitty window there is no controlling-TTY transport and no `KITTY_LISTEN_ON`; pass `--to unix:/tmp/kitty-rc-demo.sock` or run inside a kitty window.
- **"Disabling Wayland backend"** — expected under Xvfb; the X11 path is used.
- **Stale socket after a crash** — only `SIGKILL`/crash leaves a stale socket; `SIGTERM` is caught and triggers `atexit` cleanup. Remove manually with `rm -f` if needed.
- **`xxd` not found** — use `hexdump -C` (as the document does).

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Activate venv | `source /root/kitty-venv/bin/activate` |
| Standard build | `python3 setup.py` |
| Debug build | `python3 setup.py build --debug --extra-logging=event-loop` |
| Version check | `./kitty/launcher/kitty --version` |
| Run headless + socket | `xvfb-run -a ./kitty/launcher/kitty --config NONE -o allow_remote_control=yes --listen-on unix:/tmp/kitty-rc-demo.sock --start-as=minimized sh -c 'sleep 300' &` |
| Query `ls` | `./kitty/launcher/kitten @ --to unix:/tmp/kitty-rc-demo.sock ls` |
| Raw protocol | `echo -en '\eP@kitty-cmd{"cmd":"ls","version":[0,14,2]}\e\\' \| socat - unix:/tmp/kitty-rc-demo.sock \| awk '{ print substr($0, 13, length($0) - 14) }' \| jq -c '.data \| fromjson' \| jq .` |
| Inspect wire bytes | `… \| socat - unix:/tmp/kitty-rc-demo.sock \| hexdump -C` |

### Appendix B — Port / Socket Reference

| Endpoint | Default | Notes |
|----------|---------|-------|
| Remote-control transport | **UNIX-domain socket** | No fixed TCP port by default. Forms: `unix:/path`, abstract `unix:@name` (no filesystem entry), or `tcp:host:port` via `--listen-on` |
| Demo socket (this guide) | `unix:/tmp/kitty-rc-demo.sock` | Created only because `--listen-on` was passed with a permitting `allow_remote_control` mode |
| Virtual display (Xvfb) | auto-assigned by `xvfb-run -a` | Headless rendering surface |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** |
| `kitty/remote_control.py` | Client transport selection (`L383`), DCS framing (`L308-310`, `L52-53`), `parse_cmd` (`L56`), `handle_cmd` (`L213`) |
| `kitty/boss.py` | Socket gate (`L364`), `listen_on` + `atexit` (`L177`/`L181`), dispatch (`L590`), `list_os_windows` (`L432`), auth gate (`L623-633`) |
| `kitty/rc/base.py` | `RemoteCommand` base (`L319`), dynamic router `command_for_name` (`L449-456`) |
| `kitty/rc/ls.py` | `ls` handler — payload (`L45-46`), `response_from_kitty` (`L48`), `json.dumps` (`L76`), instance `ls = LS()` (`L79`) |
| `kitty/child.py` / `kitty/tabs.py` | Env-var producers (`KITTY_PID` `L244`, `KITTY_LISTEN_ON` `L246-249`; `KITTY_WINDOW_ID` `L491`) |
| `kitty/child-monitor.c` | C socket-accept + DCS detect (`L504`, `L1614`, `L1650`, `L1686`) |
| `tools/cmd/at/main.go` / `socket_io.go` | Go client transport selection (`L280`) + DCS constants (`L82-83`) |
| `docs/rc_protocol.rst` / `docs/remote-control.rst` | Authoritative protocol & usage references |

### Appendix D — Technology Versions

| Component | Version |
|-----------|---------|
| kitty | 0.35.2 (`constants.py:L25`) |
| Python | 3.13.7 (requires ≥ 3.8) |
| Go | 1.24.4 (`go.mod`: `go 1.22`) |
| gcc | 15.2.0 |
| Commit (source of truth) | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |

### Appendix E — Environment Variable Reference

| Variable | Set by (producer) | Consumed by | Purpose |
|----------|-------------------|-------------|---------|
| `KITTY_PID` | `kitty/child.py:L244` | shell-integration (`kitty.bash:L215-216`) | kitty process id |
| `KITTY_WINDOW_ID` | `kitty/tabs.py:L491` | shell-integration | per-window identifier |
| `KITTY_LISTEN_ON` | `kitty/child.py:L246-249` (only when a socket is configured) | `kitten @` client (`remote_control.py:L271`, `tools/cmd/at/main.go:L372`) | default RC destination |
| `KITTY_PUBLIC_KEY` | `kitty/child.py:L245` | encrypted-RC client | public key for encrypted remote control |
| `CC` / `GOTOOLCHAIN` | developer (build) | `setup.py` / Go | compiler + toolchain selection |

### Appendix F — Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `setup.py` | Primary build entry (Makefile `all` / `debug-event-loop`) |
| `dev.sh` | Dev runner → `go run bypy/devenv.go` |
| `xvfb-run` | Headless virtual display for the GPU terminal |
| `socat` | Drives the raw RC socket frame (rc_protocol example) |
| `jq` | Parses the `{"ok":true,"data":"…"}` envelope and re-parses the nested `data` JSON |
| `hexdump -C` | Reveals the literal `ESC P @kitty-cmd … ESC \` wire frames |
| `git` | `git diff --stat 815df1e210e0..HEAD` confirms the single-file change |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **RC** | Remote Control — kitty's mechanism for controlling a running instance from another process |
| **DCS frame** | Device Control String wrapping the JSON command: `ESC P @kitty-cmd <JSON> ESC \` |
| **`SocketIO` / `RCIO`** | The two client transports — UNIX/TCP socket vs controlling-TTY escape codes — selected at `remote_control.py:L383` |
| **`kitten @`** | The remote-control client subcommand (e.g., `kitten @ ls`) |
| **`command_for_name`** | Dynamic dispatcher that `import_module('kitty.rc.<name>')` to reach a handler; the extension point for new commands (39 command modules exist today) |
| **Abstract socket** | A Linux UNIX socket in the abstract namespace (`unix:@name`) with no filesystem entry |
| **`allow_remote_control`** | Gate controlling whether RC is permitted; modes `yes`/`socket`/`socket-only`/`password`/`no` |
| **AAP** | Agent Action Plan — the governing scope document for this task |

---

*Generated by the Blitzy Platform. Completion (93.75%) is computed on AAP-scoped + path-to-production work only. Colors: Completed `#5B39F3`, Remaining `#FFFFFF`.*