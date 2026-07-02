# Blitzy Project Guide — kitty PTY Communication Investigation

> **Deliverable under assessment:** `blitzy/documentation/kitty_815df1e210e0.md` — a single, evidence-backed technical document answering six questions (Q1–Q6) about how the kitty terminal emulator's C code communicates with its spawned shell over a pseudoterminal (PTY).
>
> **Task nature:** Read-only investigation / documentation (SWE-AtlasQnA-Repo ruleset). The entire kitty source tree is byte-for-byte unchanged from base commit `815df1e21`.
>
> **Blitzy brand colors:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF` · Headings / Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

This project produces one evidence-backed technical document explaining how kitty's C code communicates with the shell it spawns over a PTY. Working from a live, traced kitty process (built from source and launched headlessly), it answers six concrete questions: the spawned shell process, PID, exact command line, and PTY device path; the read system calls, buffer size, and byte counts for a small input (`echo test123`); the changed read behavior, frequency, and per-read byte count under a high-volume stream (`yes hello`); the PTY master file-descriptor number; and the specific C functions that read from the PTY and that separate printable text from escape sequences. The kitty source tree is strictly read-only; the sole artifact is one Markdown answer file.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#FFFFFF', 'pieStrokeColor':'#B23AF2', 'pieStrokeWidth':'2px', 'pieOuterStrokeColor':'#B23AF2', 'pieOuterStrokeWidth':'2px', 'pieTitleTextColor':'#B23AF2', 'pieSectionTextColor':'#B23AF2', 'pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion — 90.7% Complete
    "Completed Hours" : 19.5
    "Remaining Hours" : 2.0
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **21.5 h** |
| **Completed Hours (AI + Manual)** | **19.5 h** (AI autonomous: 19.5 h · Manual: 0 h) |
| **Remaining Hours** | **2.0 h** |
| **Percent Complete** | **90.7 %** (19.5 / 21.5 × 100) |

> Completion is computed strictly on AAP-scoped work (PA1): all six investigation questions plus authoring/QA/compliance are complete; the only remaining hours are human path-to-production (SME review + acceptance/merge).

### 1.3 Key Accomplishments

- ✅ Built kitty from source via `python3 setup.py build` → `kitty 0.35.2 created by Kovid Goyal` (exit 0, zero warnings) and launched it headlessly under Xvfb with software GL (llvmpipe).
- ✅ Characterized the spawned shell: process **`bash`**, exact command line **`/bin/bash --posix`**, PTY slave **`/dev/pts/0`** — grounded in `resolved_shell()`, native `spawn()`, and shell-integration argv injection.
- ✅ Traced the small-input read path for `echo test123`: `poll()`→`read()` on fd **10**, first request **1048576** bytes (`= BUF_SZ`), returning **23 / 47 / 114 / 169 = 353** bytes.
- ✅ Traced the high-volume stream `yes hello`: continuous `read→poll→read` loop; magnitude captured both traced and via an independent un-traced PTY probe; backpressure and `input_delay` batching explained.
- ✅ Identified the PTY master file descriptor: **fd 10 → `/dev/pts/ptmx`**.
- ✅ Named the C functions: reader **`read_bytes()`** [`kitty/child-monitor.c:1337`]; parser/classifier **`consume_input()`** [`kitty/vt-parser.c:1367`], separating printable text (`consume_normal()`) from escape sequences (`VTEState` machine).
- ✅ Authored the single deliverable with **39 unique `file:line` citations** (all verified byte-exact) and a coverage table confirming every named sub-item is answered.
- ✅ Honored the read-only constraint: **0 non-`blitzy/` files changed** vs base `815df1e21`; all temporary trace logs/scripts deleted; working tree clean.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None_ — the deliverable is complete, accurate, and reproducible; all 39 citations and arithmetic verified; Q1–Q6 reproduced on live traced sessions. | No release blocker | — | — |

> There are no unresolved defects. The only outstanding items are the standard human path-to-production steps in §1.6 / §2.2.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| — | — | No access issues identified. Build, headless launch, syscall tracing (`strace`), and `/proc` inspection all succeeded within the sanctioned environment. | N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** SME technical review — verify Q1–Q6 answers, all 39 `file:line` citations, and mechanism explanations against a fresh reading of the kitty source (optionally spot-reproduce one runtime observation). _(1.5 h)_
2. **[Medium]** Accept & merge the deliverable PR (`blitzy/documentation/kitty_815df1e210e0.md`) to the target branch. _(0.5 h)_
3. **[Low]** _(Optional)_ Archive one representative `strace` capture alongside the doc for future auditability (not required; the doc already quotes evidence verbatim).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Q1 — Build & headless launch | 3.0 | `setup.py build` of C extensions + native launcher; Xvfb + `LIBGL_ALWAYS_SOFTWARE=1` launch; version confirmation |
| Q2 — Shell spawn characterization | 2.5 | `ps`/`/proc` observation of process, PID, `/bin/bash --posix` cmdline, `/dev/pts/0`; grounding in `resolved_shell()`, `spawn()`, shell-integration |
| Q3 — `echo test123` read trace | 2.5 | `strace` of `poll()`/`read()` on fd 10; buffer size = `BUF_SZ`; byte-count analysis (23/47/114/169 = 353) |
| Q4 — `yes hello` high-volume trace | 3.5 | Continuous-stream trace + independent un-traced PTY probe; read-frequency & per-read byte-count analysis; backpressure/`input_delay` batching |
| Q5 — PTY master fd identification | 1.0 | `/proc/<pid>/fd` + `strace -yy` → fd 10 → `/dev/pts/ptmx`; grounding in `self.child_fd` |
| Q6 — C reader/parser analysis | 2.0 | Source analysis of `read_bytes()`, `consume_input()`, `consume_normal()`, `VTEState`, threading model |
| Deliverable authoring | 3.5 | 315-line Markdown, 39 verified citations, coverage table, fidelity notes, one-claim↔one-evidence discipline |
| QA documentation-fidelity pass | 1.5 | Second commit `6cfffc65c` addressing documentation-fidelity findings (+17/−5) |
| **Total Completed** | **19.5** | **Matches Completed Hours in §1.2** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human SME technical review of Q1–Q6 answers & citations | 1.5 | High |
| Final acceptance & merge of deliverable to target branch | 0.5 | Medium |
| **Total Remaining** | **2.0** | **Matches Remaining Hours in §1.2 and §7** |

### 2.3 Hours Reconciliation

| Check | Formula | Result |
|-------|---------|--------|
| Total = Completed + Remaining | 19.5 + 2.0 | **21.5 h** ✅ |
| Percent complete | 19.5 / 21.5 × 100 | **90.7 %** ✅ |
| Rule 1 (§1.2 ↔ §2.2 ↔ §7 remaining) | 2.0 = 2.0 = 2.0 | ✅ |
| Rule 2 (§2.1 + §2.2 = Total) | 19.5 + 2.0 = 21.5 | ✅ |

---

## 3. Test Results

For a Markdown Q&A deliverable, the meaningful "test suite" is Blitzy's autonomous validation of the document: source-citation accuracy, internal arithmetic consistency, and live reproduction of every answer. All entries below originate from Blitzy's autonomous validation logs for this project.

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---------------|--------------------|-------|--------|--------|-----------|-------|
| Source-citation accuracy | `sed`/`grep` byte-exact match vs source | 39 | 39 | 0 | 100% | All 39 unique `file:line` citations verified byte-exact against kitty source |
| Internal arithmetic | `python3` recomputation | 7 | 7 | 0 | 100% | `BUF_SZ=1048576`; Q3 total 353; Q3 decrements 1048553/1048506/1048392; Q4 decrements 1043314/1042175 |
| Q1–Q6 live reproduction | `setup.py` build + Xvfb + `strace`/`ps`/`/proc` | 6 | 6 | 0 | 100% | Each answer reproduced on a live traced kitty session |
| Build validation | `python3 setup.py build` | 1 | 1 | 0 | 100% | Exit 0, zero errors/warnings → `kitty 0.35.2` |
| **Total** | — | **53** | **53** | **0** | **100%** | 100% pass of the deliverable-validation suite |

> **Kitty upstream test suite (transparency):** kitty's own `test.py` suite was intentionally not run as part of validating this deliverable. It is a Markdown document with no unit tests of its own, and the kitty source is byte-for-byte unchanged from base `815df1e21`, so the pre-existing suite state (per setup log: all Go tests pass; 2 known ENV/FIXTURE failures in out-of-scope source that the read-only constraint forbids modifying) is unaffected — zero regression is possible. Correctness was instead established by the stronger, task-appropriate method of reproducing every answer on a live traced kitty process.

---

## 4. Runtime Validation & UI Verification

**Build & launch (Q1)**
- ✅ **Operational** — `python3 setup.py build` → `kitty/launcher/kitty` (ELF64 PIE); `./kitty/launcher/kitty --version` → `kitty 0.35.2 created by Kovid Goyal` (exit 0).
- ✅ **Operational** — Headless launch under `Xvfb :99 -screen 0 1280x1024x24` with `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1` (llvmpipe software GL); GUI window surfaced without a physical display.

**Shell spawn (Q2)**
- ✅ **Operational** — Spawned process `bash`; exact command line `/bin/bash --posix` (`/proc/<pid>/cmdline` NUL form `/bin/bash^@--posix^@`); PTY slave `/dev/pts/0` (child fds 0/1/2).

**Small-input read path (Q3)**
- ✅ **Operational** — `poll()`→`read()` on fd 10; first request `1048576` (`= BUF_SZ`); returned `23`, `47`, `114`, `169` bytes (total `353`); the 114-byte read ends with `test123\r\n`.

**High-volume stream (Q4)**
- ✅ **Operational** — `yes hello` drove a tight continuous `read→poll→read` loop (thousands of reads/second) versus the echo's 4-reads-then-idle; per-read byte counts and frequency observed both traced and via an independent un-traced PTY probe.
- ⚠ **Partial (honest negative)** — the POLLIN backpressure throttle (`events=0`) never fired for `yes hello`: the 1 MiB parser buffer never filled because the parser kept up. This path is therefore documented by source citation, not by live firing — explicitly flagged as cited-but-not-observed.

**PTY master descriptor (Q5)**
- ✅ **Operational** — `strace -yy` and `/proc/<pid>/fd` show master **fd 10 → `/dev/pts/ptmx`**; fd number deterministic across launches.

**C functions (Q6)**
- ✅ **Operational** — reader `read_bytes()` reachable only from `io_loop()` (I/O thread); `consume_input()` reached via the parse worker on the main thread — confirming the doc's reader = I/O-thread / parser = main-thread threading note.

**UI note:** This is a backend/systems investigation of PTY I/O. There is no user-facing UI component in the deliverable; "UI verification" reduces to confirming the GPU/GUI terminal launches and renders under the headless framebuffer, which it does.

---

## 5. Compliance & Quality Review

Cross-mapping AAP deliverables and the governing SWE-AtlasQnA-Repo rules to Blitzy's quality benchmarks.

| Requirement (AAP / Rule) | Benchmark | Status | Evidence / Notes |
|--------------------------|-----------|--------|------------------|
| Deliverable location & name | `blitzy/documentation/<branch>.md` | ✅ Pass | `blitzy/documentation/kitty_815df1e210e0.md` (matches branch `kitty_815df1e210e0`) |
| Answer every question (Q1–Q6) + every named item | 100% coverage | ✅ Pass | Coverage table + grep confirm process, PID, cmdline, PTY path, syscalls, buffer size, byte count, frequency, fd number, reader fn, parser fn all present |
| Run-first, observe-then-write | Runtime-grounded | ✅ Pass | Every runtime value paired with pasted `strace`/`ps`/`/proc` output (one claim ↔ one evidence line) |
| Quote observed output verbatim | No paraphrase | ✅ Pass | Byte counts, request sizes, cmdlines quoted verbatim |
| Exact `file:line` grounding | Cite literals | ✅ Pass | 39 unique citations; all verified byte-exact against source |
| Observe true magnitude (Q4) | Representative scale | ✅ Pass | `yes hello` run long enough; traced + independent un-traced probe corroborate the ~10× bracket |
| Read-only source tree | 0 source edits | ✅ Pass | `git diff 815df1e21..HEAD` on `kitty/`, `setup.py`, `go.mod`, `pyproject.toml`, `docs/`, `tools/`, `kittens/`, `3rdparty/` = none |
| Temporary-artifact cleanup | Repo left unchanged | ✅ Pass | All `/tmp` trace logs/scripts deleted; `git status --porcelain` empty; `git clean -nd` empty |
| No fabricated runtime values | Observed only | ✅ Pass | Doc frames PIDs as session-specific; fd 10 shown deterministic; honest negative documented |

**Fixes applied during autonomous validation:** QA documentation-fidelity pass (commit `6cfffc65c`, +17/−5) refined wording/fidelity of the answers. **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Run-to-run runtime variance (PIDs, exact byte counts, read frequency differ per session) | Technical | Low | High (certain) | Doc frames PIDs as session-specific and grounds each mechanism in source; fd number shown deterministic (10 across all sessions) | Mitigated / Acknowledged |
| `strace`/ptrace overhead skews Q4 magnitude (slows the read loop) | Technical | Low | High | Doc reports traced numbers alongside an independent un-traced PTY probe, with an explicit overhead caveat | Mitigated |
| Backpressure honest-negative: POLLIN throttle (`events=0`) never fired for `yes hello` | Technical | Low | Medium | Path documented by source citation and transparently flagged as cited-but-not-observed | Acknowledged |
| Secrets / auth / vulnerable deps | Security | Informational | None | Read-only investigation; no source/deps added; observation socket was a temporary `/tmp` unix socket, deleted afterward | N/A |
| Deployed runtime service failure | Operational | Low | None | No service ships; deliverable is static Markdown | Accepted |
| External integration / credentials | Integration | Low | None | No external integrations/APIs; reproduction needs only headless display (Xvfb) + software GL, which is documented | Documented |

**Overall risk posture: LOW.** The source tree is untouched → zero regression risk to kitty. The sole residual risk is inherent live-observation variance, which the deliverable explicitly and honestly addresses.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#FFFFFF', 'pieStrokeColor':'#B23AF2', 'pieStrokeWidth':'2px', 'pieOuterStrokeColor':'#B23AF2', 'pieOuterStrokeWidth':'2px', 'pieTitleTextColor':'#B23AF2', 'pieSectionTextColor':'#B23AF2', 'pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 19.5
    "Remaining Work" : 2.0
```

**Remaining hours by category (from §2.2):**

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review | 1.5 | High |
| Acceptance & merge | 0.5 | Medium |
| **Total** | **2.0** | — |

> **Integrity:** "Remaining Work" = **2.0 h**, identical to §1.2 metrics and the §2.2 Hours sum. "Completed Work" = **19.5 h**, identical to §1.2 and §2.1. Colors: Completed = `#5B39F3`, Remaining = `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, evidence-backed answer to all six questions about kitty's PTY communication path, from shell spawn (`/bin/bash --posix` over `/dev/pts/0`) through the non-blocking master descriptor (**fd 10**), the I/O-thread reader (`read_bytes()`, `read()` into a 1 MiB `BUF_SZ` buffer), and the main-thread parser that separates printable text from escape sequences (`consume_input()` → `consume_normal()` / `VTEState`). Every runtime claim is paired with verbatim observed output, and every structural claim carries a byte-exact `file:line` citation (39 total).

**Remaining gaps.** No engineering gaps remain. The **2.0 h** of remaining work is entirely human path-to-production: SME technical review (1.5 h) and acceptance/merge (0.5 h).

**Critical path to production.** SME review of the answers and citations → accept → merge. No build, configuration, integration, or security work is required.

**Success metrics.** Build exit 0; 39/39 citations byte-exact; all arithmetic consistent; Q1–Q6 reproduced on live traced sessions three independent times (shell `/bin/bash --posix` and master fd `10` agree across all runs); source tree byte-for-byte unchanged; working tree clean.

**Production-readiness assessment.** The single in-scope deliverable is **production-ready** and committed. The project is **90.7% complete** (19.5 / 21.5 h); the residual 9.3% is human review/acceptance, consistent with the policy of never claiming 100% before human sign-off.

| Metric | Value |
|--------|-------|
| AAP-scoped completion | 90.7% |
| Completed / Total hours | 19.5 / 21.5 h |
| Blocking issues | 0 |
| Overall risk | Low |

---

## 9. Development Guide

All commands below were executed and verified in the sanctioned environment during validation.

### 9.1 System Prerequisites

- **OS:** Linux (container: Ubuntu-based). A virtual framebuffer is required because kitty is a GPU/GUI terminal.
- **Toolchain (verified versions):**

| Tool | Verified Version |
|------|------------------|
| Python | 3.13.7 (AAP requires ≥ 3.8) |
| gcc | 15.2.0 |
| Go | 1.24.4 (satisfies `go.mod` ≥ 1.22) |
| strace | 6.16 |
| Xvfb | present at `/usr/bin/Xvfb` |

- **Build libraries (verified via `pkg-config`):** harfbuzz 10.2.0, libpng 1.6.50, freetype2 26.2.20, fontconfig 2.15.0, lcms2 2.16, x11 1.8.12, xkbcommon 1.7.0, dbus-1 1.16.2; `simde` header at `/usr/include/simde/simde-common.h`; Pillow 12.3.0; Pygments 2.20.0.

### 9.2 Environment Setup

```bash
# From the repository root
cd /path/to/kitty-repo

# Confirm toolchain
python3 --version          # -> Python 3.13.7
gcc --version | head -1     # -> gcc ... 15.2.0
go version                  # -> go1.24.4
strace --version | head -1  # -> strace 6.16
command -v Xvfb             # -> /usr/bin/Xvfb

# Confirm build libraries are discoverable
for lib in harfbuzz libpng freetype2 fontconfig lcms2 x11 xkbcommon dbus-1; do
  printf '  %-12s ' "$lib"; pkg-config --modversion "$lib" 2>/dev/null || echo '(missing)'
done
```

### 9.3 Build (Q1)

```bash
# Compiles the C extension modules and the native launcher kitty/launcher/main.c
python3 setup.py build
```

**Expected:** completes with exit 0 and no compiler warnings; produces `kitty/launcher/kitty`.

```bash
./kitty/launcher/kitty --version
# Expected: kitty 0.35.2 created by Kovid Goyal
```

### 9.4 Headless Launch (Q1)

```bash
# 1) Start a virtual framebuffer
Xvfb :99 -screen 0 1280x1024x24 >/tmp/xvfb.log 2>&1 &
XVFB_PID=$!

# 2) Launch kitty on it with software GL (no GPU in container)
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty \
    --listen-on unix:/tmp/kitty.sock -o allow_remote_control=yes \
    >/tmp/kitty.log 2>&1 &
KITTY_PID=$!
sleep 5
kill -0 "$KITTY_PID" && echo "kitty running (PID $KITTY_PID)"
```

### 9.5 Observe the Spawned Shell (Q2, Q5)

```bash
# Q2: process, PID, exact command line
ps --ppid "$KITTY_PID" -o pid,ppid,comm,args
cat /proc/<shell_pid>/cmdline | tr '\0' ' '; echo   # -> /bin/bash --posix

# Q2: PTY device path (slave side, from the shell's fds)
ls -l /proc/<shell_pid>/fd/0                          # -> /dev/pts/0

# Q5: kitty PTY master fd number
ls -l /proc/"$KITTY_PID"/fd | grep -iE 'ptmx|pts'     # -> 10 -> /dev/pts/ptmx
```

### 9.6 Trace the Read Path (Q3, Q4)

```bash
# Attach to kitty; follow threads; filter to read/poll; resolve fd paths (-yy)
strace -f -e trace=read,poll -yy -ttt -p "$KITTY_PID" 2>/tmp/kitty.strace &

# Q3: type a small input in the kitty window (via remote control), then inspect
#     -> poll() then read(10, ..., 1048576) returning 23 / 47 / 114 / 169 bytes (total 353)

# Q4: run a high-volume generator, then inspect the trace
#     -> continuous read->poll->read loop; thousands of reads/second
#     Cross-check magnitude with an independent un-traced PTY probe to avoid strace overhead skew
```

### 9.7 Example Usage (the user's two commands)

- **`echo test123`** — the small single-line stimulus for Q3 (one `read()` on the master fd yields the buffer size and byte count).
- **`yes hello`** — the continuous generator for Q4 (reveals read frequency and typical per-read byte count, and demonstrates the backpressure/`input_delay` batching behavior).

### 9.8 Verification & Troubleshooting

| Symptom | Cause | Resolution |
|---------|-------|-----------|
| kitty exits immediately / GL errors | No GPU in container | Ensure `LIBGL_ALWAYS_SOFTWARE=1` is set (forces llvmpipe) |
| `cannot open display` | `DISPLAY` not pointing at a running Xvfb | Start `Xvfb :99 ...` first, then export `DISPLAY=:99` |
| Warning about systemd user bus | Benign in a container | Harmless — safe to ignore |
| `read()` returns `EAGAIN` in trace | Master fd is non-blocking (`os.set_blocking(child_fd, False)`) | Expected; the C reader retries on `EAGAIN`/`EINTR` |
| Q4 numbers look too small/slow | `strace` overhead throttles the loop | Corroborate with an un-traced PTY probe; report both |

**Cleanup (leave repo pristine):**

```bash
kill "$KITTY_PID" 2>/dev/null; kill "$XVFB_PID" 2>/dev/null   # exact PIDs only — never broad pkill
rm -f /tmp/kitty.sock /tmp/kitty.log /tmp/xvfb.log /tmp/kitty.strace
git status --porcelain   # expect empty
```

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Build | `python3 setup.py build` |
| Version check | `./kitty/launcher/kitty --version` |
| Start framebuffer | `Xvfb :99 -screen 0 1280x1024x24 &` |
| Headless launch | `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty --listen-on unix:/tmp/kitty.sock -o allow_remote_control=yes &` |
| Shell process | `ps --ppid <kitty_pid> -o pid,ppid,comm,args` |
| Exact cmdline | `cat /proc/<shell_pid>/cmdline \| tr '\0' ' '` |
| Master fd | `ls -l /proc/<kitty_pid>/fd \| grep -iE 'ptmx\|pts'` |
| Syscall trace | `strace -f -e trace=read,poll -yy -ttt -p <kitty_pid>` |
| Verify clean tree | `git status --porcelain` |

### B. Port Reference

Not applicable — kitty is a local GUI terminal. No network ports are opened. The only "endpoint" used for observation is a temporary UNIX-domain control socket (`unix:/tmp/kitty.sock`), deleted after use.

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only created file) |
| `kitty/child.py` | Creates PTY (`os.openpty()`), stores master (`self.child_fd = master`), sets non-blocking |
| `kitty/child.c` | Native `spawn()`: slave path via `ttyname_r`, `fork()`, `execvp()` |
| `kitty/child-monitor.c` | I/O thread: `io_loop()`, `read_bytes()`, `read()` on the master fd |
| `kitty/vt-parser.c` | `BUF_SZ` (1 MiB), `consume_input()`, `consume_normal()`, `VTEState` |
| `kitty/utils.py` | `resolved_shell()` — resolves the shell command |
| `kitty/constants.py` | `shell_path` default |
| `kitty/shell_integration.py` | Injects shell-integration argv (`--posix`) / environment |
| `kitty/boss.py` | `add_child()` wires the master fd into the monitor |
| `kitty/options/definition.py` | `input_delay` = 3 ms, `repaint_delay` = 10 ms |
| `setup.py` · `kitty/launcher/main.c` | Build entry point · native launcher |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| kitty (built) | 0.35.2 |
| Python | 3.13.7 |
| gcc | 15.2.0 |
| Go | 1.24.4 |
| strace | 6.16 |
| harfbuzz / libpng / freetype2 | 10.2.0 / 1.6.50 / 26.2.20 |
| fontconfig / lcms2 / x11 | 2.15.0 / 2.16 / 1.8.12 |
| xkbcommon / dbus-1 | 1.7.0 / 1.16.2 |
| Pillow / Pygments / simde | 12.3.0 / 2.20.0 / present |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `DISPLAY` | `:99` | Points kitty at the Xvfb virtual framebuffer |
| `LIBGL_ALWAYS_SOFTWARE` | `1` | Forces llvmpipe software GL (no GPU in container) |

### F. Developer Tools Guide

- **`strace -f -e trace=read,poll -yy -ttt -p <pid>`** — `-f` follows the I/O thread; `-e trace=read,poll` filters to the relevant syscalls; `-yy` prints the device path behind each fd (revealing `/dev/pts/ptmx`); `-ttt` adds timestamps for frequency analysis.
- **`ps` / `/proc/<pid>/{cmdline,fd}`** — process, PID, exact command line, PTY device path, and fd number.
- **Independent un-traced PTY probe** — a small helper that opens a PTY and reads in a loop to measure read frequency and per-read byte counts without `strace` overhead, used to corroborate the traced Q4 magnitude.

### G. Glossary

| Term | Meaning |
|------|---------|
| PTY | Pseudoterminal: a master/slave device pair; the terminal reads the master, the shell owns the slave |
| Master fd | The file descriptor kitty reads from (observed as **fd 10** → `/dev/pts/ptmx`) |
| `BUF_SZ` | Parser buffer size, `1024 × 1024 = 1048576` bytes (1 MiB), the read-request size |
| Backpressure | The I/O thread requests `POLLIN` only when the parser reports free space, protecting the 1 MiB buffer |
| `input_delay` | Parse-batching interval (default 3 ms) that governs how reads are grouped for parsing |
| `VTEState` | The parser state machine that routes escape sequences (ESC/CSI/OSC/DCS) apart from printable text |
| Honest negative | A documented condition that did not fire during observation (here, the POLLIN throttle for `yes hello`), disclosed rather than asserted |
