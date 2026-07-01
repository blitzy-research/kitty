# Blitzy Project Guide

> **Project:** Evidence-Grounded Q&A — *How `kitty` Handles Flow Control & Backpressure for Terminal Graphics Data*
> **Repository:** `kovidgoyal/kitty` · **Branch:** `blitzy-fc4c7761-f20d-4430-84cf-97e39e8e5da1` · **Base:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> **Deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` · **Task type:** Read-only technical documentation
> **Brand legend:** <span style="color:#5B39F3">■</span> Completed / AI Work = Dark Blue `#5B39F3` · <span style="color:#FFFFFF;background:#333">■</span> Remaining = White `#FFFFFF`

---

## 1. Executive Summary

### 1.1 Project Overview

This project authors a single, evidence-grounded technical answer document explaining how the `kitty` terminal emulator manages flow control and backpressure when terminal **graphics** data arrives faster than it can comfortably process — and how it writes responses back to the originating program when its own output path is congested. The audience is engineers and reviewers who need a source-of-truth explanation grounded in exact code literals and verbatim runtime output. Business impact is knowledge capture: a rigorously verified reference for kitty's inbound/outbound backpressure mechanics. Technical scope is deliberately narrow and **read-only** — the kitty core flow-control path (`vt-parser.c`, `child-monitor.c`, `graphics.c`, `screen.c`, and the relevant options) is investigated, built, and exercised, producing exactly one new Markdown file with zero changes to existing source.

### 1.2 Completion Status

The completion percentage is computed with the AAP-scoped hours methodology (PA1): only work defined by the Agent Action Plan plus minimal path-to-production (human review/merge) is counted. A documentation deliverable has no deployable runtime surface, so the work universe is the investigation + authoring + verification effort, and the remaining work is human acceptance plus optional enhancement hardening.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#FFFFFF', 'pieStroke':'#B23AF2', 'pieStrokeWidth':'2px', 'pieOuterStrokeWidth':'2px', 'pieTitleTextSize':'16px', 'pieSectionTextColor':'#B23AF2'}}}%%
pie showData
    title Completion Status — 89.5% Complete
    "Completed Work (hrs)" : 38.5
    "Remaining Work (hrs)" : 4.5
```

| Metric | Hours |
|---|---|
| **Total Hours** | **43.0** |
| Completed Hours — AI (autonomous) | 38.5 |
| Completed Hours — Manual (human) | 0.0 |
| **Completed Hours (AI + Manual)** | **38.5** |
| **Remaining Hours** | **4.5** |
| **Percent Complete** | **89.5%** |

> **Formula:** `Completion % = Completed ÷ Total × 100 = 38.5 ÷ 43.0 × 100 = 89.5%`.

### 1.3 Key Accomplishments

- ✅ Authored the complete evidence-grounded Q&A `blitzy/documentation/kitty_815df1e210e0.md` (658 lines) answering all six sub-parts **R1–R6** explicitly, each with mechanism + exact literals + verbatim observed output + rationale.
- ✅ Built kitty and ran the **real compiled VT parser + PTY harness** first, capturing **8 verbatim observed-output blocks** before writing — satisfying the "run before writing" rule.
- ✅ Grounded every claim in **78 exact `file:line` citations** across 11 files; each key value (buffer sizes, delays, caps, error strings) quoted as an exact literal, never paraphrased.
- ✅ Demonstrated the inbound **1 MiB** buffer saturation, the `input_delay=3 ms` / `16 KiB` throttle constants, and the write-back bytes (`EFBIG`/`EINVAL` graphics replies) with reproducible output.
- ✅ Completed a coverage pass confirming R1–R6 are each answered, plus **3 honest "could not verify headlessly" disclosures** for items requiring a live GPU/PTY session.
- ✅ Left the repository **byte-for-byte unchanged** except the single deliverable; all temporary observation scripts removed; working tree clean.
- ✅ Independent Final Validator confirmed **zero defects**; build `EXIT 0` (0 warnings); test suites **parser 16/16** and **graphics 19/19** pass.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None._ No blocking or release-critical issues were identified. The deliverable is complete, accurate, and verified defect-free. | None | — | — |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Live GPU / display + real PTY session | Runtime environment | 3 disclosed items (sub-ms `input_delay` batching latency, forced 100 MiB `write_buf` overflow, live event-loop observability) cannot be exercised in the headless build sandbox | Not blocking — honestly disclosed in the document and cited from source; optional to verify in a live session | Human reviewer (optional) |

No repository-permission, credential, or third-party API access issues were identified. The task is read-only and self-contained.

### 1.6 Recommended Next Steps

1. **[High]** Have a kitty/terminal-emulator subject-matter expert review the R1–R6 narrative and spot-check the 78 `file:line` citations and 8 observed-output blocks against the source at HEAD. *(~1.5h)*
2. **[High]** Approve and accept the deliverable for merge (reviewer sign-off). *(~0.5h)*
3. **[Low]** Optionally verify the 3 disclosed items in a live kitty session (GPU/display + real PTY): sub-ms `input_delay` batching, the forced 100 MiB overflow log line, and live event-loop observability flags. *(~2.0h)*
4. **[Low]** Editorial polish (Markdown lint, render-check the Mermaid diagram) and merge the PR. *(~0.5h)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously (AI). Each component traces to an AAP requirement (investigation, observation, authoring, verification, or cleanliness).

| Component | Hours | Description |
|---|---:|---|
| Environment build & runtime enablement | 2.5 | Compile the C extension (`setup.py build`, `EXIT 0`), verify `kitty.fast_data_types` importable, confirm `VT_PARSER_BUFFER_SIZE=1048576` at runtime |
| Repository scope discovery & flow-control path mapping | 3.0 | Identify the 4 core flow-control files among 868 tracked files; sweep `write_buf`/`read_buf`/`O_NONBLOCK`/`EAGAIN`; exclude unrelated I/O subsystems (disk-cache, fast-file-copy, kittens, loop-utils, desktop) |
| Inbound-path code investigation — `vt-parser.c` | 3.0 | `BUF_SZ` (1 MiB), `MAX_ESCAPE_CODE_LENGTH`, the fixed `buf` array, `vt_parser_has_space_for_input`, the `input_delay` consume gate, the resume flag, and the `VT_PARSER_BUFFER_SIZE` export |
| I/O-loop & write-buffer code investigation — `child-monitor.c` | 3.5 | `POLLIN` read-gate, the 100 MiB write cap + drop-and-log, `POLLOUT` drain arm/dispatch, the blocking `KittyWriteStdin` path, `read_bytes`, `wakeup_io_loop` |
| Graphics caps + write-back wiring — `graphics.c`, `screen.c`, options | 4.5 | `MAX_DATA_SZ` (~400 MB), `EFBIG`/`EINVAL` aborts, response builders; `write_escape_code_to_child` wiring + test hooks; `input_delay`/`repaint_delay`/`sync_to_monitor` defaults |
| Observation scripts authored, run & 8 verbatim outputs captured | 6.0 | 3 temporary scripts (inbound 1 MiB saturation, throttle constants, write-back + graphics `EFBIG`/`EINVAL`) driving the real parser/PTY hooks; verbatim output captured then scripts deleted |
| Answer authoring R1–R6 | 8.0 | Each sub-part written with mechanism + exact literals + verbatim observed output + rationale (6 rationale paragraphs) |
| Supporting sections | 3.0 | Question restatement, one-paragraph summary, verification-method, standard-model (XON/XOFF vs OS-level) web-research framing, Mermaid flow diagram |
| Coverage pass + honest "could not verify" disclosures | 1.5 | R1–R6 coverage checklist (all `[x]`) + 3 disclosures for headlessly-unverifiable items |
| Citation verification (78 refs) + code-review fix iterations | 3.0 | Confirm every `file:line` anchor exact at HEAD across 3 commits (incl. the `graphics.c` off-by-one fix) |
| Repository cleanliness + deliverable placement/naming | 0.5 | Delete all temp scripts (byte-for-byte unchanged repo); place/name deliverable per branch rule |
| **Total Completed** | **38.5** | **Sum of Completed Hours — matches Section 1.2** |

### 2.2 Remaining Work Detail

Each remaining category traces to a path-to-production need (human acceptance) or optional enhancement hardening.

| Category | Hours | Priority |
|---|---:|---|
| Human SME Technical Review & Acceptance (verify narrative + citations + observed output; merge sign-off) | 2.0 | High |
| Live-Environment Verification of the 3 Disclosed Items (sub-ms `input_delay` batching; forced 100 MiB overflow log; live event-loop observability) — needs GPU/PTY | 2.0 | Low |
| Editorial Polish (Markdown lint, render-check Mermaid) & PR Merge | 0.5 | Low |
| **Total Remaining** | **4.5** | **Matches Section 1.2 & Section 7 pie** |

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 total (Completed) | 38.5h |
| Section 2.2 total (Remaining) | 4.5h |
| Section 2.1 + Section 2.2 | 38.5 + 4.5 = **43.0h** = Total (Section 1.2) ✅ |
| Remaining consistent across §1.2, §2.2, §7 | 4.5h everywhere ✅ |
| Completion % | 38.5 ÷ 43.0 = **89.5%** ✅ |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project (kitty's own Python test harness driving the real compiled parser), re-confirmed during this assessment.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Unit — Parser | Python `unittest` (`./test.py --module parser`) | 16 | 16 | 0 | n/a (behavioral) | Includes `test_parser_threading` which exercises the 1 MiB buffer saturation (`kitty_tests/parser.py:140-148`), the core R1 evidence path |
| Unit — Graphics | Python `unittest` (`./test.py --module graphics`) | 19 | 19 | 0 | n/a (behavioral) | Exercises the graphics protocol handlers underpinning the R3 `EFBIG`/`EINVAL` write-back evidence |
| Compilation | `python3 setup.py build --verbose` | 1 | 1 | 0 | — | `EXIT 0`, **0 warnings** under default `-pedantic-errors -Werror` |
| **Total** | — | **36** | **36** | **0** | — | 100% pass rate |

**Evidence markers (verbatim from the runner):**

```text
$ python3 ./test.py --module parser --verbosity 1
Running under CI: False
................
----------------------------------------------------------------------
Ran 16 tests in 0.054s

OK
```

```text
$ python3 ./test.py --module graphics --verbosity 1
Running under CI: False
...................
----------------------------------------------------------------------
Ran 19 tests in 0.188s

OK
```

> **Integrity note:** These suites are the code paths the deliverable cites as evidence; the document quotes their `OK` markers, and this assessment re-ran both to confirm the counts (16 and 19) and pass state.

---

## 4. Runtime Validation & UI Verification

This is a documentation deliverable with **no user-facing UI**; "runtime validation" here means the real compiled kitty code paths were exercised and produced the verbatim evidence quoted in the document.

- ✅ **Build & import** — `kitty/fast_data_types.so` compiled; `from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE` → `1048576` (Operational).
- ✅ **Inbound 1 MiB saturation** — the real parser accepts exactly `1048576` bytes, refuses the 20 excess bytes, and reports `test_create_write_buffer() len = 0` (FULL) — the exact state the read-gate keys on (Operational).
- ✅ **Throttle constants** — runtime defaults reproduced: `input_delay=3`, `repaint_delay=10`, `sync_to_monitor=True`, near-full threshold `16*1024=16384` (Operational).
- ✅ **Write-back path** — device-attribute queries elicit real return bytes `b'\x1b[?62;c'` and `b'\x1b[>1;4000;35c'` through the `write_buf` sink (Operational).
- ✅ **Graphics overflow responses** — oversized transmissions elicit verbatim protocol errors `b'\x1b_Gi=1;EFBIG:Too much data\x1b\\'` and `b'\x1b_Gi=2;EINVAL:PNG data size too large\x1b\\'` (Operational).
- ⚠ **Live event-loop observability** — `--dump-commands`, `--debug-input`, `--debug-rendering`, and `make debug-event-loop` require a live terminal session (spawned child + PTY + GPU/display) unavailable headlessly; cited from source and honestly disclosed (Partial — by design).
- ⚠ **Forced 100 MiB `write_buf` overflow** — requires a live `ChildMonitor` with a registered child; the lightweight `Screen` harness bypasses the cap. The exact code + verbatim log string are cited as source-of-truth (Partial — by design).

_No ❌ failing items._

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the governing "SWE-AtlasQnA-Repo" rules to their verified status.

| Requirement / Benchmark | Status | Progress | Evidence |
|---|---|---|---|
| Deliverable location & name = `blitzy/documentation/kitty_815df1e210e0.md` (branch-named) | ✅ Pass | 100% | File present; matches source branch `kitty_815df1e210e0` |
| Investigate by **running** first, then write (build + observation scripts + captured output) | ✅ Pass | 100% | 8 verbatim observed-output blocks paired with commands; build `EXIT 0` |
| Quote observed output **verbatim** (with the command that produced it) | ✅ Pass | 100% | Every evidence block shows `$ ...` command + literal output |
| Answer **every** sub-part R1–R6 + coverage pass | ✅ Pass | 100% | R1–R6 sections + coverage checklist all `[x]` |
| Be exact & grounded — exact literals + `file:line`, never paraphrase, disclose unverifiable | ✅ Pass | 100% | 78 exact citations; 3 honest "could not verify" disclosures |
| Provide rationale for each answer | ✅ Pass | 100% | 6 `**Rationale.**` paragraphs |
| Read-only scope — no existing file modified; no code added beyond the doc; temp scripts removed | ✅ Pass | 100% | `git diff <base> --name-status` → single `A` file; tree clean; no leaked scripts |
| Citation accuracy at HEAD `815df1e210e0` | ✅ Pass | 100% | Validator verified all 49+ (78 total) refs; re-spot-checked this session |
| Zero placeholders / TODO / stub content | ✅ Pass | 100% | grep for TODO/FIXME/placeholder → none |
| Build quality — 0 compiler warnings under `-Werror` | ✅ Pass | 100% | `setup.py build --verbose` → `EXIT 0`, 0 warnings |

**Fixes applied during autonomous validation:** the Final Validator found **zero defects requiring edits**; prior fix commits (`dbf6291db` code-review findings, `f0a2640ec` `graphics.c` citation off-by-one + O2 after-parse reproducibility clarification) had already resolved review findings. **Outstanding items:** none in autonomous scope.

---

## 6. Risk Assessment

All identified risks are Low, None, or Informational — consistent with a read-only documentation deliverable that introduces no runtime surface, dependencies, or attack surface.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation anchor drift if source is later edited | Technical | Low | Low | Document pins HEAD `815df1e210e0`; re-verify anchors on any rebase | Mitigated |
| 3 claims verified from source, not run headlessly (sub-ms `input_delay` timing; forced 100 MiB overflow log; live event-loop observability) | Technical | Low | Low | Honestly disclosed in-doc; optional live-env (GPU/PTY) verification available | Open (accepted) |
| No security-relevant changes (read-only doc; no code/deps/secrets/attack surface) | Security | None | N/A | Nothing to mitigate | Closed |
| Doc placed outside the Sphinx `docs/` tree (`blitzy/documentation/`) → not in rendered kitty docs | Operational | Low | Medium | Intentional per AAP rule; standalone Markdown by design | Accepted by design |
| Reproducing observed output requires the canonical build env (gcc/Go/Python + libs) | Integration | Low | Low | Environment documented in the doc's "How this was verified" section and Guide §9; build validated `EXIT 0` | Mitigated |
| Pre-existing build note: `wayland-protocols` absent → wayland backend auto-disabled | Operational | Informational | Low | Out of AAP scope; no effect on any documented flow-control path or observation script | Accepted |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3', 'pie2':'#FFFFFF', 'pieStroke':'#B23AF2', 'pieStrokeWidth':'2px', 'pieOuterStrokeWidth':'2px', 'pieSectionTextColor':'#B23AF2'}}}%%
pie showData
    title Project Hours Breakdown (Total 43.0h)
    "Completed Work" : 38.5
    "Remaining Work" : 4.5
```

**Remaining hours by priority** (sums to 4.5h — matches §2.2):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2', 'pie2':'#A8FDD9', 'pieStroke':'#5B39F3', 'pieStrokeWidth':'2px', 'pieSectionTextColor':'#333333'}}}%%
pie showData
    title Remaining Work by Priority (4.5h)
    "High (review + accept)" : 2.0
    "Low (optional verify + polish/merge)" : 2.5
```

**Remaining hours by category** (from §2.2):

| Category | Hours | Bar |
|---|---:|---|
| Human SME Review & Acceptance | 2.0 | ████████ |
| Live-Env Verification (optional) | 2.0 | ████████ |
| Editorial Polish & Merge | 0.5 | ██ |
| **Total** | **4.5** | |

> **Integrity:** the "Remaining Work" pie value (4.5) equals Remaining Hours in §1.2 and the sum of the §2.2 Hours column.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a rigorously evidence-grounded technical Q&A that answers all six sub-parts (R1–R6) of how kitty handles graphics flow control and backpressure. It was produced the correct way — **kitty was built and run first**, and the document quotes **verbatim observed output** from the real compiled parser and PTY harness alongside **78 exact `file:line` citations**. The inbound **1 MiB** VT-parser buffer and its `POLLIN` read-gate, the `input_delay` (3 ms) coalescing throttle with its 16 KiB near-full bypass, the growable `write_buf` with its **100 MiB** ceiling and drop-and-log, and the graphics per-image `MAX_DATA_SZ` (~400 MB) cap with `EFBIG`/`EINVAL` replies are each explained, cited, and demonstrated. The quiet-vs-visible split (R6) is drawn precisely from the presence/absence of log lines on each path.

**Remaining gaps & critical path.** No autonomous work remains. The critical path to production is short: a subject-matter expert reviews and accepts the document (2.0h, High), after which it can be merged. Optionally, a reviewer with a live GPU/PTY session can verify the 3 honestly-disclosed items (2.0h, Low) and apply editorial polish before merge (0.5h, Low).

**Success metrics.** Build `EXIT 0` (0 warnings); parser 16/16 and graphics 19/19 tests pass; all spot-checked citations exact; every runtime evidence value reproduced (`1048576`, `input_delay=3`, `repaint_delay=10`, `16384`, `104857600`, `400000000`); repository byte-for-byte unchanged except the single deliverable.

**Production readiness.** The deliverable is **production-ready** at **89.5% completion** (38.5h of 43.0h). The remaining 4.5h is human review/acceptance plus optional enhancement — there are no blocking issues, no failing tests, and no unresolved defects. Recommendation: proceed to SME review and merge.

| Metric | Value |
|---|---|
| AAP-scoped completion | 89.5% |
| Completed / Total hours | 38.5 / 43.0 |
| Remaining hours | 4.5 |
| Blocking issues | 0 |
| Test pass rate | 36/36 (100%) |
| Files changed | 1 added (658 lines), 0 modified |

---

## 9. Development Guide

This guide documents how to build kitty, verify the runtime, reproduce the document's evidence, and review the deliverable. All commands are copy-pasteable from the repository root and were tested during this assessment.

### 9.1 System Prerequisites

| Component | Version (verified) | Notes |
|---|---|---|
| C compiler (gcc/clang) | gcc 15.2.0 | Compiles the `fast_data_types` C extension (`docs/build.rst:99`) |
| Go toolchain | go 1.24.4 | `>= 1.22` (`docs/build.rst:101`) |
| Python | 3.13.7 | `>= 3.8`; build driver + observation runtime |
| GNU Make | 4.4.1 | Optional convenience targets |
| HarfBuzz | 10.2.0 | `>= 2.2.0` (`docs/build.rst:84`) |
| FreeType | 26.2.20 | Font rasterization on non-macOS |
| OpenGL + GLFW | vendored (`glfw/`, `glad/`) | GPU rendering backend |
| libpython3-dev | distro | Python headers for the C extension (`docs/build.rst:118`) |

> **Canonical environment:** Docker image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`.

### 9.2 Environment Setup

```bash
# From the repository root (branch blitzy-fc4c7761-…, base HEAD 815df1e210e0)
git status                 # expect: working tree clean
git rev-parse HEAD         # confirm the commit under review
```

On an externally-managed system Python (PEP 668), prefer a virtualenv for any extra tooling:

```bash
python3 -m venv .venv && source .venv/bin/activate   # optional, for auxiliary tools
```

### 9.3 Build

```bash
# Primary build (produces kitty/fast_data_types.so). Expect EXIT 0, 0 warnings.
python3 setup.py build --verbose

# Alternatives:
make                       # standard build
make debug-event-loop      # == setup.py build --debug --extra-logging=event-loop (Makefile:25-26)
```

### 9.4 Verification Steps

```bash
# 1) Confirm the C extension is importable and the 1 MiB buffer literal is live
python3 -c "from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE; print(VT_PARSER_BUFFER_SIZE)"
# Expected: 1048576

# 2) Confirm the throttle option defaults
python3 -c "import sys; sys.path.insert(0,'.'); from kitty.options.types import defaults; \
print('input_delay', defaults.input_delay, '| repaint_delay', defaults.repaint_delay, '| sync_to_monitor', defaults.sync_to_monitor)"
# Expected: input_delay 3 | repaint_delay 10 | sync_to_monitor True

# 3) Run the evidence test suites
python3 ./test.py --module parser --verbosity 1     # Expected: Ran 16 tests ... OK
python3 ./test.py --module graphics --verbosity 1   # Expected: Ran 19 tests ... OK
```

### 9.5 Reviewing the Deliverable (doc-review workflow)

```bash
# Count the file:line citations
grep -oE "kitty/[a-zA-Z_/-]+\.(c|h|py)[: ]*[0-9]+" blitzy/documentation/kitty_815df1e210e0.md | wc -l
# Expected: 78

# Spot-check specific anchors against source
for ref in kitty/vt-parser.c:18 kitty/vt-parser.c:1425 kitty/child-monitor.c:1501 \
           kitty/child-monitor.c:342 kitty/graphics.c:521 kitty/graphics.c:533 kitty/graphics.c:638; do
  f="${ref%:*}"; ln="${ref##*:}"; printf '%-28s -> %s\n' "$ref" "$(sed -n "${ln}p" "$f" | sed 's/^[[:space:]]*//')"
done

# Confirm the read-only constraint (single file added, tree clean)
git diff 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 --name-status   # -> A blitzy/documentation/kitty_815df1e210e0.md
git status --porcelain                                            # -> (empty)
```

### 9.6 Example Usage — Reproducing an Evidence Block

The document's throttle-constants block is reproduced directly:

```bash
python3 - <<'PY'
import sys; sys.path.insert(0, '.')
from kitty.options.types import defaults
from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE
print("defaults.input_delay   =", defaults.input_delay, "(ms)")
print("defaults.repaint_delay =", defaults.repaint_delay, "(ms)")
print("defaults.sync_to_monitor =", defaults.sync_to_monitor)
print("VT_PARSER_BUFFER_SIZE =", VT_PARSER_BUFFER_SIZE)
print("near-full immediate-consume threshold (16 * 1024) =", 16*1024, "bytes")
PY
# Expected output matches the document's s2_throttle_options.py block exactly.
```

### 9.7 Troubleshooting

- **`error: externally-managed-environment` (PEP 668)** — the system Python blocks global `pip install`. Use a venv (`python3 -m venv .venv && source .venv/bin/activate`) or pass `--break-system-packages` for auxiliary tools. kitty itself builds via `setup.py`/`make` and needs no extra pip installs.
- **`wayland-protocols` not found → wayland backend disabled** — pre-existing, expected in a headless build; harmless and out of scope. It does not affect any documented flow-control path or observation script.
- **Live-session items can't be observed headlessly** — the 3 disclosed items (sub-ms `input_delay` batching, forced 100 MiB overflow log, live event-loop observability) require a spawned child + real PTY + GPU/display. Run kitty interactively with `--dump-commands` / `--debug-input` / `--debug-rendering`, or use `make debug-event-loop`, to observe them live.
- **Import fails after a clean checkout** — rebuild first (`python3 setup.py build`); the `.so` must exist for `kitty.fast_data_types` to import.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---|---|
| `python3 setup.py build --verbose` | Build kitty C extension (produces `kitty/fast_data_types.so`) |
| `make` / `make debug-event-loop` | Standard build / debug build with event-loop logging (`Makefile:25-26`) |
| `python3 ./test.py --module parser --verbosity 1` | Run the parser suite (16 tests) |
| `python3 ./test.py --module graphics --verbosity 1` | Run the graphics suite (19 tests) |
| `git diff 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 --name-status` | Verify read-only constraint (single added file) |
| `grep -oE "kitty/[...]\.(c\|h\|py)[: ]*[0-9]+" <doc> \| wc -l` | Count citations (78) |

### Appendix B — Port Reference

Not applicable — the deliverable is a documentation file with no network service, listener, or bound port. kitty run interactively opens no TCP port for the flow-control paths documented (it uses PTYs and the local GPU).

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only file added) |
| `kitty/vt-parser.c` | Inbound 1 MiB buffer, read-gate predicate, `input_delay` consume gate, resume flag |
| `kitty/child-monitor.c` | I/O poll loop (`POLLIN`/`POLLOUT`), `write_buf` + 100 MiB cap, blocking stdin write |
| `kitty/graphics.c` | Graphics `MAX_DATA_SZ` cap, `EFBIG`/`EINVAL` aborts, response builders |
| `kitty/screen.c` | Write-back wiring (`write_escape_code_to_child`), test hooks |
| `kitty/options/definition.py`, `kitty/options/types.py` | `input_delay`, `repaint_delay`, `sync_to_monitor` defaults |
| `kitty_tests/parser.py` | The "full write" test modeling the 1 MiB saturation observation |

### Appendix D — Technology Versions

| Component | Version |
|---|---|
| gcc | 15.2.0 |
| Go | 1.24.4 |
| Python | 3.13.7 |
| GNU Make | 4.4.1 |
| HarfBuzz | 10.2.0 |
| FreeType | 26.2.20 |

### Appendix E — Environment Variable Reference

No environment variables are required to build, run the observation scripts, or review the deliverable. (kitty honors standard terminal env vars such as `TERM`/`TERMINFO` at runtime, but none affect the documented flow-control code paths or the review workflow.)

### Appendix F — Developer Tools Guide

Runtime observability surfaces cited in the deliverable (for a live session):

| Tool / Flag | Purpose | Source |
|---|---|---|
| `kitty --dump-commands` | Output commands received from the child process to STDOUT | `kitty/cli.py:972` |
| `kitty --dump-bytes <path>` | Store raw bytes received from the child | `kitty/cli.py:985` |
| `kitty --debug-input --debug-keyboard` | Debug input/keyboard handling | `kitty/cli.py:996` |
| `kitty --debug-rendering --debug-gl` | Debug rendering / GL | `kitty/cli.py:989` |
| `make debug-event-loop` | Build with `--debug --extra-logging=event-loop` for I/O-loop logging | `Makefile:25-26` |

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **Backpressure** | Slowing or stopping data intake so a fast producer cannot overwhelm a slower consumer |
| **PTY** | Pseudoterminal — the bidirectional master/slave channel between kitty and its child process |
| **`POLLIN` / `POLLOUT`** | `poll()` event flags kitty toggles to gate reading from / writing to the child fd |
| **XON/XOFF** | Software flow-control bytes (`0x11`/`0x13`); kitty does **not** use these — it uses OS-level gating |
| **`_G` APC** | The graphics-protocol escape sequence `<ESC>_G<control data>;<payload><ESC>\` |
| **`EFBIG` / `EINVAL`** | Graphics protocol error codes returned when a transmission exceeds `MAX_DATA_SZ` |
| **R1–R6** | The six sub-parts of the question the deliverable answers |
| **AAP** | Agent Action Plan — the governing specification for this task |

---

*End of Blitzy Project Guide.*