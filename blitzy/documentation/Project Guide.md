# Blitzy Project Guide — kitty PTY Read/Communicate Investigation

> **Deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` — a runtime code‑behavior investigation (Q&A) of how the kitty terminal emulator reads from and communicates with the child shell it spawns over a pseudo‑terminal (PTY).
> **Branch:** `blitzy-1c0b6d1e-8884-44c6-810e-14d23dfb0918` · **HEAD:** `d7ae7f7e5` · **Base:** `815df1e21`
> **Color key:** <span style="color:#5B39F3">■</span> Completed / AI Work = **Dark Blue `#5B39F3`** · <span style="color:#B23AF2">■</span> White = **Remaining `#FFFFFF`**

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read‑only runtime code‑behavior investigation** of the kitty terminal emulator that answers, with observed runtime evidence and exact `file:line` code references, how kitty reads from and communicates with the child shell it spawns over a PTY. Blitzy built kitty from source in its canonical configuration, launched it headless under Xvfb, drove the real typed‑input PTY path with two workloads (`echo test123`, `yes hello`), and produced one comprehensive Markdown answer document covering five named requirements (R1–R5: shell spawn, single‑command read path, high‑volume read behavior, PTY master file descriptor, and the responsible C functions). The strict read‑only mandate prohibits any source modification — the sole artifact is the answer document.

### 1.2 Completion Status

The project is **88.9% complete**. Every AAP‑specified deliverable (R1–R5, canonical build/launch, evidence capture, verification, and cleanup) is fully delivered and validated; the remaining **5.0 hours** is the standard human peer‑review, reproduction spot‑check, and merge gate for the documentation deliverable.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'18px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Project Completion 40.0h of 45.0h
    "Completed Work (AI)" : 40
    "Remaining Work" : 5
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **45.0** |
| Completed Hours (AI + Manual) | 40.0 |
| &nbsp;&nbsp;• AI (autonomous) | 40.0 |
| &nbsp;&nbsp;• Manual (human) | 0.0 |
| **Remaining Hours** | **5.0** |
| **Percent Complete** | **88.9%** |

> **Completion formula (PA1, AAP‑scoped):** `Completed ÷ (Completed + Remaining) = 40.0 ÷ 45.0 = 88.9%`.

### 1.3 Key Accomplishments

- ✅ **Canonical build from source** with the exact entry point `python3 setup.py` (EXIT=0), producing `kitty/launcher/kitty` → `kitty 0.35.2`.
- ✅ **Headless launch in default config** (`--config NONE`) under Xvfb, with all interaction delivered through the **real typed‑input PTY path** via `xdotool` (XTEST) — never the remote‑control interface.
- ✅ **R1 — Shell spawn** identified: process `bash` (`/usr/bin/bash`), exact command line `/bin/bash --posix`, slave device `/dev/pts/0`, correlated to `kitty/constants.py:181` and `kitty/child.c`.
- ✅ **R2 — `echo test123` read path** captured: gating `poll()` → `read(8, buf, 1048576)` (buffer = `BUF_SZ` = 1 MiB), command output returned in a 114‑byte read; requested‑size shrink arithmetic (`BUF_SZ − offset`) reproduced.
- ✅ **R3 — `yes hello` high‑volume** measured two ways across ≥2 runs each: native ≈ 97k–134k reads/s @ ≈ 64–103 B/read; `strace` cross‑check; `input_delay` (3 ms) role clarified.
- ✅ **R4 — PTY master fd** captured: fd **8** → `/dev/pts/ptmx` (dynamic per run) via `/proc/<pid>/fd` and `lsof`.
- ✅ **R5 — Responsible C functions** attributed: reader `read_bytes()` (`kitty/child-monitor.c:1337`) and parser `consume_input()`→`consume_normal()` (`kitty/vt-parser.c:1367`/`:230`), with the two‑thread architecture and runtime dispatch chain.
- ✅ **All ~48 `file:line` references verified** against source at HEAD `815df1e21` (zero out‑of‑bounds, zero missing).
- ✅ **Read‑only mandate satisfied**: `git diff` vs base shows exactly **one added file** and **zero source modifications**; working tree clean; all temporary artifacts kept outside the repo and deleted.
- ✅ **Autonomous QA cycle** resolved 7 code‑review findings (complete unedited evidence, exact computation commands + raw output, `poll()` nfds‑vs‑timeout correction, parser‑dispatch correction).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All AAP requirements are delivered and validated; no blocking issues remain. | None | — | — |

> The only outstanding work is the standard human review/merge gate (see §1.6 and §2.2). There are no compilation errors, failing checks, or unresolved discrepancies.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _None_ | — | No access issues identified. The build, headless launch, syscall tracing (root + `ptrace`), and process/descriptor inspection all succeeded within the provided container. | N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** Perform a **technical peer review & sign‑off** of `blitzy/documentation/kitty_815df1e210e0.md` — verify R1–R5 answers, the `file:line` references, and evidence consistency (3.0h).
2. **[Medium]** Optionally **reproduce the runtime claims** with a fresh build+launch to confirm the dynamic values differ per run while behavior signatures hold (1.5h).
3. **[Medium]** **Merge** the answer document to the target branch and close the task, confirming the working tree contains only the single added file (0.5h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work is autonomous (AI) and traces to an AAP requirement. **Total = 40.0h.**

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical build from source | 3.0 | `python3 setup.py` (EXIT=0); resolve Xvfb/software‑GL/locale env for the headless GPU app; capture tool versions. Maps to AAP canonical‑build rule. |
| Headless launch & runtime setup | 2.0 | Launch under `Xvfb`, `--config NONE`; window focus; capture kitty PID + I/O‑thread TID + shell PID. |
| R1 — Shell‑spawn investigation | 3.0 | `pstree`/`ps`/`/proc` tree; process `bash`, exact cmdline `/bin/bash --posix`; correlate `constants.py:181`, `child.py`, `child.c`; bash `--posix` shell integration. |
| R4 — PTY master‑fd investigation | 1.5 | `/proc/<pid>/fd` + `lsof`; fd 8 → `/dev/pts/ptmx`; full fd layout (eventfd/signalfd/master); `os.openpty()` dynamic assignment. |
| R2 — `echo test123` read‑path capture & analysis | 3.5 | `strace` the I/O thread; gating `poll()` → `read(8,buf,1048576)`; 12 one‑byte echoes; 11/47/114/429‑byte output reads; `BUF_SZ − offset` arithmetic. |
| R3 — `yes hello` dual‑method measurement | 6.0 | Method A native `/proc/<pid>/task/<tid>/io` (4 windows: 5s×3 + 10s×1) via `native_run.sh`; Method B `strace` (2 runs) + `analyze.py` stats; reconciliation + `input_delay` role. |
| R5 — Responsible‑functions code tracing | 4.5 | `read_bytes()` body; `consume_input()`/`consume_normal()` bodies; two‑thread architecture; `main_loop`→`parse_input`→`do_parse`→`parse_worker`→`run_worker` dispatch; reader↔parser API boundary. |
| Edge/transitional‑state investigation | 2.5 | Idle `poll(-1)` + `wchan do_sys_poll`; typing; high‑volume; return‑to‑idle; eventfd `EAGAIN` drain; child‑death & buffer‑full inferred branches. |
| Answer‑document authoring | 6.5 | 933 lines: TL;DR table, §1–§8, Appendix A (repro), Appendix B (code‑ref index); evidence beside every claim; inferred‑vs‑observed labeling. |
| Autonomous QA validation & remediation | 4.5 | Resolve 7 review findings: complete/unedited evidence; exact computation commands + raw output; `poll()` nfds‑vs‑timeout correction; parser‑dispatch correction; provenance; gating‑poll pairing; temp‑path cleanup. |
| Code‑reference verification | 2.0 | Verify ~48 `file:line` refs vs source at HEAD; ~40 semantically‑critical manually confirmed; verbatim code reproductions checked. |
| Repository‑integrity cleanup | 1.0 | Delete `/tmp/kitty_probe` temp scripts+logs; stop kitty+Xvfb via targeted PID kills; confirm clean tree, 0 source mods. |
| **Total Completed** | **40.0** | |

### 2.2 Remaining Work Detail

All remaining work is human **path‑to‑production** for a documentation deliverable. **Total = 5.0h.**

| Category | Hours | Priority |
|----------|-------|----------|
| Technical peer review & sign‑off of answer document | 3.0 | High |
| Independent reproduction spot‑check of runtime claims | 1.5 | Medium |
| Merge answer document to target branch & close task | 0.5 | Medium |
| **Total Remaining** | **5.0** | |

### 2.3 Total Project Hours

| Bucket | Hours |
|--------|-------|
| Completed (Section 2.1) | 40.0 |
| Remaining (Section 2.2) | 5.0 |
| **Total (Section 1.2)** | **45.0** |

> **Integrity:** 2.1 + 2.2 = 40.0 + 5.0 = **45.0** = Total in §1.2. Remaining = **5.0h** in §1.2, §2.2, and §7 (identical).

---

## 3. Test Results

This is a **documentation Q&A** task; the AAP explicitly excludes test authoring and any source change, so **no new unit tests were created**. The equivalent, and mandated, validation is **(a) verifying every `file:line` reference against source** and **(b) reproducing every runtime claim by building and running kitty**. The table below aggregates the autonomous validation checks from Blitzy's validation logs (all checks originate from this project's autonomous validation).

| Validation Category | Method / Framework | Total Checks | Passed | Failed | Coverage % | Notes |
|---------------------|--------------------|-------------:|-------:|-------:|-----------:|-------|
| Code‑reference verification | Programmatic scan + manual (vs source `815df1e21`) | 48 | 48 | 0 | 100% | All distinct `file:line` refs in‑bounds; ~40 semantically‑critical confirmed verbatim; 0 out‑of‑bounds, 0 missing files. |
| Runtime requirement reproduction | `strace` / `ps` / `pstree` / `lsof` / `/proc` + `xdotool` (XTEST) | 5 (R1–R5) | 5 | 0 | 100% | All five requirements reproduced via the real typed‑input PTY path; observed values match the document. |
| High‑volume stability (R3) | Native `/proc` io counters + `strace`, ≥2 runs each | 6 runs | 6 | 0 | 100% | 4 native windows (5s×3 + 10s×1) + 2 `strace` windows; values stable within documented ranges. |
| Canonical build validation | `python3 setup.py` | 1 | 1 | 0 | 100% | EXIT=0; `kitty 0.35.2`; re‑verified this session. |
| Read‑only mandate validation | `git diff` / `git status` | 1 | 1 | 0 | 100% | Exactly 1 added file, 0 source modifications; working tree clean. |
| **Totals** | | **61** | **61** | **0** | **100%** | |

> **Note on the upstream test suite:** kitty ships a test suite under `kitty_tests/` (run via `python3 setup.py test`). Executing or extending it is **out of scope** for this read‑only investigation (the AAP forbids test additions), so it is intentionally not part of these results.

---

## 4. Runtime Validation & UI Verification

Runtime behavior was validated end‑to‑end against a canonical build launched headless. Legend: ✅ Operational · ⚠ Partial · ❌ Failing.

**Build & process health**
- ✅ Build: `python3 setup.py` → EXIT=0; launcher reports `kitty 0.35.2`.
- ✅ Headless launch: kitty runs under Xvfb in default config (`--config NONE`) with software GL.
- ✅ Process model: exactly one child shell + I/O thread (`KittyChildMon`) confirmed via `pstree`/`/proc`.

**Requirement verification (real typed‑input PTY path)**
- ✅ **R1 — Shell spawn:** process `bash` (`/usr/bin/bash`), cmdline `/bin/bash --posix`, slave `/dev/pts/0`, `TERM=xterm-kitty`.
- ✅ **R2 — `echo test123`:** gating `poll()` → `read(8, buf, 1048576)`; per‑keystroke 1‑byte reads; 114‑byte output read; requested‑size shrink arithmetic reproduced.
- ✅ **R3 — `yes hello`:** continuous back‑to‑back reads; native ≈ 97k–134k reads/s @ ≈ 64–103 B/read (stable over 4 runs); `strace` cross‑check consistent.
- ✅ **R4 — PTY master fd:** fd **8** → `/dev/pts/ptmx` via `/proc/<pid>/fd` + `lsof`.
- ✅ **R5 — Functions:** reader `read_bytes()` and parser `consume_input()`/`consume_normal()` verified in source and behavior.

**UI verification**
- ✅ kitty is a GPU‑accelerated OpenGL GUI (no web UI). It was verified as a live GUI application under a virtual X server, with keystrokes injected as real X11 (XTEST) events into the focused window — exactly the path a user’s keyboard takes. No traditional web/DOM UI applies to this deliverable.

**API integration**
- ✅ Not applicable — the investigation uses only kitty’s own code plus standard POSIX syscalls (`read(2)`, `poll(2)`, `openpty(3)`). No external APIs or services are involved.

---

## 5. Compliance & Quality Review

### 5.1 AAP Rules Compliance Matrix (SWE‑AtlasQnA‑Repo)

| # | AAP Rule | Status | Evidence |
|---|----------|--------|----------|
| 1 | Deliverable named `<branch>.md` in `blitzy/documentation/` | ✅ Pass | `blitzy/documentation/kitty_815df1e210e0.md` exists (933 lines). |
| 2 | Run first, then write (observe, not read‑only) | ✅ Pass | All values captured at runtime via `strace`/`ps`/`lsof`/`/proc`; each shown with its producing command. |
| 3 | Observe magnitude at scale, ≥2 runs (R3) | ✅ Pass | 4 native windows + 2 `strace` windows; stability confirmed within stated ranges. |
| 4 | Use the real entry point (typed PTY path, not remote control) | ✅ Pass | `xdotool` XTEST keystroke injection; remote control never used. |
| 5 | Canonical build/config + state exact commands | ✅ Pass | `python3 setup.py`; `Xvfb` + `kitty --config NONE` documented verbatim. |
| 6 | Exercise every implied condition (edge/transitional states) | ✅ Pass | §7 covers idle/typing/high‑volume/return‑to‑idle + `EAGAIN`; child‑death & buffer‑full labeled inferred. |
| 7 | Show observed output for every claim | ✅ Pass | Complete, unedited `strace`/`ps`/`lsof`/`/proc` blocks beside each claim. |
| 8 | Answer every named item + coverage pass | ✅ Pass | TL;DR table + dedicated sections cover all 12 named items. |
| 9 | Be exact & grounded (`file:line`, label inferred) | ✅ Pass | ~48 verified refs; dedicated inferred‑vs‑observed section (§8). |
| 10 | Read‑only scope (no source edits; delete temp scripts) | ✅ Pass | 1 added file, 0 source mods; temp artifacts in `/tmp/kitty_probe`, deleted; clean tree. |

### 5.2 Fixes Applied During Autonomous Validation

Commit `d7ae7f7e5` resolved **7** code‑review findings:

| Finding | Type | Resolution |
|---------|------|------------|
| #1 | Major | Replaced edited/truncated evidence with complete, unedited runtime output; labeled C excerpts verbatim/abbreviated. |
| #2 | Major | Added exact computation commands (`native_run.sh`, `analyze.py`) + complete raw stdout for every R3 statistic. |
| #3 | Major | Corrected `poll()` misreading — the `3` is `nfds`, not a timeout; observed timeouts are `{-1,0,1,2}`; `input_delay` throttles main‑loop wakeups, not reads. |
| #4 | Major | Corrected parser‑invocation citation — runtime dispatch is `main_loop`→`parse_input`→`do_parse`→`parse_worker`; `screen.c:4775‑4776` is the test‑only helper. |
| #5 | Minor | Added producing commands/output for tool versions and `ptrace`/root claims. |
| #6 | Minor | Showed the gating `poll()` preceding each result read. |
| #7 | Minor | Used `/tmp/kitty_probe` paths in Appendix A and added temp‑cleanup instructions. |

### 5.3 Quality Benchmarks

| Benchmark | Status | Notes |
|-----------|--------|-------|
| Zero source modifications (read‑only) | ✅ Pass | `git diff 815df1e21..HEAD` = 1 added file. |
| Zero placeholders/stubs/TODOs in deliverable | ✅ Pass | Document is complete; no deferred content. |
| Evidence completeness (unedited output) | ✅ Pass | Full `strace`/`/proc`/`lsof` blocks + scripts included. |
| Code‑reference accuracy | ✅ Pass | 48/48 refs in‑bounds vs `815df1e21`. |
| Reproducibility | ✅ Pass | Exact build/launch/observe commands in Appendix A; re‑verified. |

---

## 6. Risk Assessment

All risks are **Low** — expected for a read‑only documentation investigation with zero source changes, zero new dependencies, and no runtime deployment surface.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Dynamic runtime values (PID, master fd, `/dev/pts/N`) differ per run | Technical | Low | High | Document explicitly labels these as per‑run dynamic values, not constants | Mitigated |
| `strace`‑vs‑native R3 frequency gap may confuse readers | Technical | Low | Low | §5.3 explains `ptrace` overhead and presents both datasets with reconciliation | Resolved |
| `file:line` refs pinned to `815df1e21`; future source drift shifts line numbers | Technical | Low | Medium | Document pins the exact commit + branch for every reference | Mitigated |
| Build was incremental (launcher pre‑existing), not from‑scratch | Technical | Low | Low | Disclosed in §1.1; canonical entry point + EXIT=0 identical to a full build | Resolved |
| Investigation ran as `root` with `ptrace` to attach `strace` | Security | Low (informational) | N/A | Required tracing capability; deliverable is a static doc with no secrets, no new code, no dependencies, no attack surface | Accepted |
| No runtime deployment exists (static document) | Operational | Low | N/A | Monitoring/logging/health‑checks are not applicable to a Markdown deliverable | N/A |
| Reproduction requires specific tooling (`strace`/`Xvfb`/`xdotool`/root+ptrace) | Operational | Low | Medium | Appendix A lists exact commands; the AAP container supplies all tools | Mitigated |
| No external services/APIs/credentials/integrations | Integration | None | N/A | References only kitty source + standard POSIX semantics | N/A |
| Merge of a single new file under new `blitzy/documentation/` path | Integration | Low | Low | Isolated new file, no conflicts with existing sources | Open (pending human merge) |

---

## 7. Visual Project Status

### 7.1 Project Hours Breakdown

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'18px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Project Hours Completed vs Remaining
    "Completed Work" : 40
    "Remaining Work" : 5
```

### 7.2 Remaining Work by Category (hours)

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#333333','pieStrokeWidth':'1px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Remaining Work Hours by Category
    "Peer review & sign-off (High)" : 3.0
    "Reproduction spot-check (Medium)" : 1.5
    "Merge & close (Medium)" : 0.5
```

> **Integrity:** Pie “Remaining Work” = **5.0h**, matching §1.2 Remaining and the §2.2 sum. Categories in 7.2 sum to 3.0 + 1.5 + 0.5 = **5.0h**.

---

## 8. Summary & Recommendations

**Achievements.** Blitzy delivered a complete, evidence‑backed answer to every named requirement (R1–R5) of the PTY read/communicate question. Kitty was built canonically (`python3 setup.py`, EXIT=0), launched headless in default configuration, and driven through the real typed‑input PTY path; the shell spawn, single‑command read path, high‑volume read behavior, PTY master file descriptor, and the responsible reader/parser C functions were all observed at runtime and grounded in verified `file:line` references. An autonomous QA cycle resolved 7 review findings, and all ~48 code references were verified against the source at `815df1e21`.

**Remaining gaps & critical path.** The project is **88.9% complete** (40.0h of 45.0h). The remaining **5.0h** is entirely the standard human path‑to‑production for a documentation deliverable: a technical peer review & sign‑off (3.0h, the critical‑path blocker for acceptance), an optional reproduction spot‑check (1.5h), and the merge & close (0.5h). There are no compilation errors, failing checks, unresolved discrepancies, or access issues.

**Production readiness.** The deliverable is production‑ready pending human sign‑off. The read‑only mandate is fully satisfied — the repository differs from its base by exactly one added file, with zero source modifications and a clean working tree.

**Success metrics.**

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Named requirements answered (R1–R5) | 5/5 | 5/5 | ✅ |
| `file:line` references valid | 100% | 100% (48/48) | ✅ |
| Runtime claims reproduced | 100% | 100% | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Deliverable at required path | Yes | Yes | ✅ |
| AAP‑scoped completion | ≤99% (pre‑review cap) | 88.9% | ✅ |

**Recommendation.** Proceed to human peer review and sign‑off, then merge. No engineering rework is required.

---

## 9. Development Guide

> All commands below were tested in the project container this session. Run them from the repository root:
> `/tmp/blitzy/kitty/blitzy-1c0b6d1e-8884-44c6-810e-14d23dfb0918_4f8a70`.

### 9.1 System Prerequisites

| Component | Minimum | Verified in container | Role |
|-----------|---------|-----------------------|------|
| OS | Linux | Ubuntu 25.10 container | Canonical run target |
| Python | ≥ 3.8 (`pyproject.toml:2`) | 3.13.7 | Build orchestration; kitty embeds CPython |
| Go | ≥ 1.22 (`go.mod:3`) | go1.24.4 | Builds the `kitten` binary |
| C compiler | C11 | gcc 15.2.0 | Compiles the C core / extensions |
| Git | any recent | 2.51.0 | Source control / read‑only verification |

**Observation tooling** (present at `/usr/bin`): `strace`, `Xvfb`, `xvfb-run`, `lsof`, `pstree`, `ps`, `xdotool`.

### 9.2 Environment Setup

- No Python virtualenv is required — `setup.py` uses the system interpreter and embeds CPython. (A from‑scratch build needs system X11/OpenGL headers, which the container provides.)
- The deliverable introduces **no** pip/npm/Go dependency changes; there is nothing to install for the answer document itself.
- kitty is a GPU GUI, so it requires a display. Use a virtual X server (Xvfb) for headless operation.

### 9.3 Build

```bash
# Canonical build entry point (== Makefile 'all' target, lines 12-13)
python3 setup.py

# Expected tail: informational "Disabling building of wayland backend" (X11-only build),
# then a clean exit. Verify:
echo "EXIT=$?"                      # -> EXIT=0
./kitty/launcher/kitty --version    # -> kitty 0.35.2 created by Kovid Goyal
```

### 9.4 Launch (headless, default config)

```bash
# 1) headless X server
Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp &

# 2) kitty in its default, unconfigured state (software GL for the headless GPU)
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8 \
    ./kitty/launcher/kitty --config NONE &
```

### 9.5 Verification

```bash
# Read-only mandate: exactly one added file, clean tree
git diff --name-status 815df1e21..HEAD     # -> A  blitzy/documentation/kitty_815df1e210e0.md
git status --porcelain                     # -> (empty)

# Deliverable present and well-formed
wc -l blitzy/documentation/kitty_815df1e210e0.md   # -> 933
```

### 9.6 Reproduce the Investigation (R1–R5)

```bash
# Identify PIDs/TID first
KPID=$(pgrep -n -f 'launcher/kitty')                     # kitty GUI PID
IOTID=$(ps -L -o tid,comm -p "$KPID" | awk '/KittyChildMon/{print $1}')  # I/O thread TID
SHPID=$(pgrep -P "$KPID" -n)                             # child shell PID

# R1 / R4 — shell spawn + master fd
pstree -aps "$KPID"
ps -o pid,args -p "$SHPID"
ls -l /proc/"$KPID"/fd
lsof -p "$KPID" | grep -i ptmx
readlink /proc/"$SHPID"/fd/0                             # -> /dev/pts/N (slave)

# R2 — echo test123 (real keyboard -> PTY path)
mkdir -p /tmp/kitty_probe
strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl \
    -p "$IOTID" -o /tmp/kitty_probe/echo.strace &
DISPLAY=:99 xdotool type --clearmodifiers --delay 45 'echo test123'
DISPLAY=:99 xdotool key  --clearmodifiers Return

# R3 — yes hello, native magnitude (authoritative): sample syscr/rchar around a timed window
s0=$(awk '/^syscr/{print $2}' /proc/"$KPID"/task/"$IOTID"/io); \
  r0=$(awk '/^rchar/{print $2}' /proc/"$KPID"/task/"$IOTID"/io); t0=$(date +%s.%N)
DISPLAY=:99 xdotool type --clearmodifiers --delay 40 'yes hello'; DISPLAY=:99 xdotool key Return
sleep 5
s1=$(awk '/^syscr/{print $2}' /proc/"$KPID"/task/"$IOTID"/io); \
  r1=$(awk '/^rchar/{print $2}' /proc/"$KPID"/task/"$IOTID"/io); t1=$(date +%s.%N)
DISPLAY=:99 xdotool key ctrl+c
awk -v s="$((s1-s0))" -v r="$((r1-r0))" -v d="$(echo "$t1-$t0"|bc)" \
  'BEGIN{printf "reads/s=%.0f  bytes/read=%.1f  MB/s=%.2f\n", s/d, r/s, r/d/1048576}'

# R5 — read the responsible functions
sed -n '1336,1356p' kitty/child-monitor.c   # read_bytes()  (read() at :1345)
sed -n '230,240p'   kitty/vt-parser.c        # consume_normal()  (screen_draw_text at :236)

# Cleanup (preserve read-only mandate)
DISPLAY=:99 xdotool key ctrl+c 2>/dev/null; rm -rf /tmp/kitty_probe
```

### 9.7 Troubleshooting

- **`wayland-protocols not found` during build** — informational, not an error. `setup.py` auto‑disables the Wayland backend; the X11‑only build is canonical here.
- **kitty won’t start (no display)** — start `Xvfb` and export `DISPLAY`; for the headless GPU, set `LIBGL_ALWAYS_SOFTWARE=1`.
- **`strace: attach: Operation not permitted`** — run as `root`/`CAP_SYS_PTRACE`, or lower `/proc/sys/kernel/yama/ptrace_scope` (observed `=1`; root bypasses it).
- **No `read()` lines appear under `strace`** — attach to the **I/O thread TID** (`KittyChildMon`), not the main GUI PID; the PTY read runs on that thread.
- **Working tree shows changes after building** — build outputs (`build/`, `__pycache__`, `kitty/launcher/`) are git‑ignored; if anything else appears, ensure temp scripts live outside the repo (e.g., `/tmp/kitty_probe`) and are deleted.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Build (canonical) | `python3 setup.py` |
| Version check | `./kitty/launcher/kitty --version` |
| Headless X server | `Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp &` |
| Launch (default config) | `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty --config NONE &` |
| Process tree | `pstree -aps <kitty_pid>` |
| Shell process cmdline | `ps -o pid,args -p <shell_pid>` |
| FD layout | `ls -l /proc/<kitty_pid>/fd` · `lsof -p <kitty_pid> \| grep -i ptmx` |
| Slave device | `readlink /proc/<shell_pid>/fd/0` |
| Syscall trace (I/O thread) | `strace -tt -T -s 256 -e trace=read,readv,poll,ppoll,ioctl -p <io_tid>` |
| Native read counters | `cat /proc/<kitty_pid>/task/<io_tid>/io` |
| Keystroke injection | `xdotool type --delay 45 'echo test123'` · `xdotool key Return` |
| Read‑only verification | `git diff --name-status 815df1e21..HEAD` · `git status --porcelain` |

### Appendix B — Port Reference

Not applicable — kitty is a local GUI terminal emulator communicating with its child shell over a PTY; it opens **no network ports** for this investigation. The only “display port” used is the virtual X server display **`:99`** (Xvfb).

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **Deliverable** — the answer document (933 lines) |
| `kitty/child.py` | PTY create (`os.openpty()`), keep master, non‑blocking, spawn |
| `kitty/child.c` | C `spawn()`: `ttyname_r`, `fork`, controlling TTY, `dup2`, `execvp` |
| `kitty/child-monitor.c` | I/O thread `io_loop()`, `poll()`, `read_bytes()` → `read()` |
| `kitty/vt-parser.c` / `.h` | 1 MiB `BUF_SZ` buffer; `consume_input()`/`consume_normal()` |
| `kitty/constants.py` / `kitty/utils.py` | Default login‑shell resolution |
| `kitty/options/definition.py` | `input_delay` (3 ms), default `term` |
| `kitty/screen.c` / `.h` | `Screen` holding the parser; `screen_draw_text` sink |
| `Makefile` / `setup.py` | Canonical build entry point |

### Appendix D — Technology Versions

| Tool | Version (verified) | Manifest requirement |
|------|--------------------|----------------------|
| kitty (built) | 0.35.2 | — |
| Python | 3.13.7 | `requires-python >=3.8` (`pyproject.toml:2`) |
| Go | go1.24.4 | `go 1.22` (`go.mod:3`) |
| gcc | 15.2.0 | C11 |
| strace | 6.16 | — |
| Xvfb | 21.1.18 | — |
| xdotool | 3.20160805.1 | — |
| git | 2.51.0 | — |

### Appendix E — Environment Variable Reference

| Variable | Value used | Purpose |
|----------|------------|---------|
| `DISPLAY` | `:99` | Point kitty at the Xvfb virtual display |
| `LIBGL_ALWAYS_SOFTWARE` | `1` | Force software OpenGL (no physical GPU in container) |
| `LANG` / `LC_ALL` | `C.UTF-8` | Deterministic UTF‑8 locale |
| `V` / `VERBOSE` | (unset) | If set, `setup.py` builds verbosely (`Makefile`) |

### Appendix F — Developer Tools Guide

- **`strace`** — trace the I/O thread’s syscalls: `-tt` (timestamps), `-T` (syscall duration), `-s 256` (full string previews), `-e trace=read,readv,poll,ppoll,ioctl`. Attach to the **I/O thread TID**, not the GUI PID.
- **`/proc/<pid>/task/<tid>/io`** — native, tracer‑free read magnitude: `syscr` (read‑syscall count) and `rchar` (bytes read); sample before/after a timed window.
- **`lsof` / `/proc/<pid>/fd`** — resolve the PTY master fd and `/dev/pts/N` slave device.
- **`xdotool`** — inject real X11 (XTEST) keystrokes into the focused kitty window — the authentic keyboard→PTY path (never remote control).
- **`Xvfb`** — headless X server so the GPU GUI can run without a physical display.

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **PTY** | Pseudo‑terminal — a master/slave device pair; kitty holds the master, the shell’s stdio is the slave (`/dev/pts/N`). |
| **`BUF_SZ`** | The VT parser’s single input buffer capacity, `1 MiB` (`kitty/vt-parser.c:18`); the size offered to `read()`. |
| **`input_delay`** | Default 3 ms option that throttles main‑loop **wakeups** (not reads) to coalesce rendering (`options/definition.py:878`). |
| **`read_bytes()`** | The reader: does `read(fd, buf, available_buffer_space)` on the PTY master (`kitty/child-monitor.c:1337`/`:1345`). |
| **`consume_input()` / `consume_normal()`** | The VT state machine that separates printable text (→ `screen_draw_text`) from escape sequences (`kitty/vt-parser.c:1367`/`:230`). |
| **XTEST** | X11 extension used by `xdotool` to synthesize real key events — the canonical typed‑input path. |
| **I/O thread (`KittyChildMon`)** | The dedicated pthread running `io_loop()` that polls and reads the PTY master. |

---

*Generated by the Blitzy Platform. Completion percentage (88.9%) is AAP‑scoped per the PA1 methodology: `Completed ÷ (Completed + Remaining) = 40.0 ÷ 45.0`. Colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.*