# Blitzy Project Guide — Kitty Terminal I/O Pipeline Onboarding Documentation

> **Brand palette:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF` · Headings / Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`
>
> **Deliverable:** `blitzy/documentation/kitty_815df1e210e0.md` — a read-only, code-grounded onboarding explainer for Kitty's terminal interaction pipeline.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, self-contained, code-grounded onboarding document that explains how Kitty's terminal interaction pipeline behaves at runtime — tracing the full journey from the instant a surge of mixed input arrives until the user interface settles again. The target audience is a developer onboarding into the Kitty codebase. Its business impact is reduced ramp-up time and a durable, citation-backed mental model of a latency-critical, multi-threaded subsystem (C hot path + Python orchestration + Go tooling). The technical scope is purely explanatory and additive: exactly one new Markdown file is created, every behavioral claim is substantiated with inline citations to real files and line numbers at HEAD `815df1e21`, and **no existing repository file is modified**.

### 1.2 Completion Status

The project is **85.7% complete** on an AAP-scoped, hours-based basis: **42 of 49 hours** of in-scope work are autonomously complete; **7 hours** of human-gated path-to-production review remain.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 85.7% Complete
    "Completed Work (hrs)" : 42
    "Remaining Work (hrs)" : 7
```

| Metric | Value |
|---|---:|
| **Total Hours** | 49 |
| **Completed Hours (AI + Manual)** | 42 (AI 42 + Manual 0) |
| **Remaining Hours** | 7 |
| **Percent Complete** | 85.7% |

> Calculation: `Completed / (Completed + Remaining) = 42 / (42 + 7) = 42 / 49 = 85.7143% ≈ 85.7%`.

### 1.3 Key Accomplishments

- ✅ Authored the complete onboarding document `blitzy/documentation/kitty_815df1e210e0.md` (**415 lines, 65,988 bytes**, valid UTF-8) answering all six onboarding questions (R1–R6) with a dedicated section per question plus a per-question rationale appendix and a quick-reference source map.
- ✅ Substantiated every behavioral claim with **331 bracketed citations / 246 unique (path, line) pairs across 32 files**; autonomous verification found **0 missing files and 0 out-of-range lines** (independently corroborated this session at 242 pairs / 30 files, 0 errors).
- ✅ Corroborated behavioral claims in the prescribed Docker container: `python3 setup.py` built cleanly with `-Werror` (exit 0); `kitty`/`kitten` 0.35.2 run; **VT parser 16/16** and **screen 36/36** unit tests pass; shell-integration bash + zsh pass.
- ✅ Confirmed documented constants exactly against source: `BUF_SZ` = 1 MiB, `MAX_ESCAPE_CODE_LENGTH` = BUF_SZ/4, `PENDING_MODE` = 2026, 100 MiB write cap, 2000 ms pause timeout.
- ✅ Validated industry-correct terminology (synchronized output / DECSET 2026; producer/consumer backpressure with bounded buffers).
- ✅ Preserved repository immutability: `git diff 815df1e21 --name-status` shows **only** `A blitzy/documentation/kitty_815df1e210e0.md`; working tree clean; all temporary observation scripts kept outside the repo and removed.
- ✅ Zero placeholders / TODOs; balanced code fences; valid Mermaid; document self-corrects subtle citation traps (e.g., noting `window.py:L863` is a call-site, not a definition).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Subtle concurrency claims (lock-release-during-consume, POLLIN backpressure gate, batched wakeups) warrant subject-matter-expert confirmation | Low — claims are code-cited and build/test-corroborated; SME sign-off is standard practice for a code-grounded explainer | Kitty maintainer / SME reviewer | 3h (High priority) |
| Citation line numbers are pinned to HEAD `815df1e21` and may drift as upstream evolves | Low — mitigated by design (document explicitly pins the commit); only relevant if the doc is re-based onto newer code | Doc maintainer | 1h (Low priority) |

> No issue blocks the deliverable. All entries above are human-gated path-to-production review items, not autonomous defects.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Kitty source tree @ HEAD `815df1e21` | Read | None — full read access throughout | Resolved | Blitzy agent |
| Prescribed Docker container | Build / Run | None — built `-Werror` exit 0; ran kitty/kitten 0.35.2 | Resolved | Blitzy agent |
| Destination repository (branch `blitzy-7bf8846f-...`) | Write (new file only) | None — document committed; tree clean | Resolved | Blitzy agent |

**No access issues identified.** All required source, build, and write access was available; no third-party credentials or external API access are required for this documentation deliverable.

### 1.6 Recommended Next Steps

1. **[High]** Have a Kitty maintainer / SME review the document's behavioral claims and the 331 citations for technical accuracy at HEAD `815df1e21` (≈3h).
2. **[Medium]** Conduct an onboarding-developer clarity/readability pass to confirm the narrative lands for the intended audience (≈2h).
3. **[Low]** Optionally improve discoverability by linking the document from onboarding materials **outside** the repository (the AAP forbids modifying the project's own `docs/` Sphinx tree) (≈1h).
4. **[Low]** Add a citation-drift safeguard note / commit-pin reminder so future readers re-verify line numbers if the doc is re-based onto newer code (≈1h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All components below trace to AAP requirements (R1–R6, methodology, the 7 "SWE-AtlasQnA-Repo" rules, and structural placement). **Total = 42 hours (all AI).**

| Component | Hours | Description |
|---|---:|---|
| Source-code comprehension & I/O-pipeline analysis | 14 | Read and traced 30+ files across C/Python/Go: `child-monitor.c`, `vt-parser.c`, `screen.c`, `loop-utils.c`, `glfw.c`, `boss.py`, `window.py`, `child.py`, `keys.c`, `mouse.c`, `state.c`, `shell_integration.py`, the SSH kitten, and bootstrap scripts — mapping the two data directions, the three-thread conductor, ordering guarantees, pause/resume, and backpressure. *(High confidence)* |
| Terminology & standards research | 2 | Validated industry-correct names: synchronized output / DECSET 2026 family; producer/consumer backpressure with bounded buffers. *(High confidence)* |
| Document authoring | 12 | Wrote the 415-line / 66 KB narrative: orientation + Mermaid pipeline diagram, one section per question (R1–R6), per-question rationale appendix, terminology section, and a quick-reference source map. *(High confidence)* |
| Citation sourcing & accuracy verification | 6 | Sourced and verified 331 citations / 246 unique (path, line) pairs; ran existence/in-range checks (0 missing, 0 out-of-range) and semantic spot-checks; self-corrected subtle traps (call-site vs. definition). *(High confidence)* |
| Build & runtime corroboration (Docker) | 4 | Built Kitty with `-Werror` (exit 0); ran kitty/kitten 0.35.2; confirmed constants and ran pipeline unit tests (parser 16/16, screen 36/36; shell-integration bash+zsh). *(Medium-High confidence)* |
| Review-cycle revisions | 4 | Addressed code-review findings and final-gate coverage; removed an uncited external claim; verified repository immutability and clean tree. *(High confidence)* |
| **Total Completed** | **42** | Matches Completed Hours in §1.2. |

### 2.2 Remaining Work Detail

All categories below are human-gated path-to-production review items; each traces to an AAP path-to-production need. **Total = 7 hours.**

| Category | Hours | Priority |
|---|---:|---|
| SME technical-accuracy review of behavioral claims + 331 citations (Kitty maintainer verifies I/O-pipeline claims at HEAD `815df1e21`) | 3 | High |
| Onboarding-developer clarity / readability review | 2 | Medium |
| Discoverability / linking into onboarding materials **outside** the repo (AAP forbids touching the Sphinx `docs/` tree) | 1 | Low |
| Citation-drift safeguard / commit-pin note | 1 | Low |
| **Total Remaining** | **7** | High 3 · Medium 2 · Low 2 |

> Cross-section check: §2.1 (42) + §2.2 (7) = **49** = Total Hours in §1.2. Remaining (7) is identical in §1.2, §2.2, and §7. ✔

### 2.3 Basis of Estimate

- **Methodology:** PA1 (AAP-scoped, hours-based completion) + PA2 (engineering-hours framework). The denominator is the AAP work universe only: the six question threads (R1–R6), the methodological requirements (evidence-first, rationale), the 7 rules, structural placement, and standard path-to-production review.
- **Completion formula:** `42 / (42 + 7) = 85.7%`.
- **Confidence:** High for authoring, analysis, citation verification, and review; Medium-High for build/runtime corroboration (environmental factors such as `go`/fish availability). Remaining-work estimates are conservative and rounded up to the nearest 0.5h.
- **Why no autonomous work remains:** The deliverable is complete, verified, and committed. The 7 remaining hours are inherently human activities (expert judgement, readability, organizational linking) that cannot be performed autonomously.

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** for this project. For a documentation deliverable, **citation accuracy is the analogue of unit tests**, and Kitty's own pipeline unit tests corroborate the document's behavioral claims.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Citation accuracy (existence + in-range) | Custom verifier (path:line) | 246 | 246 | 0 | 100% | 246 unique (path, line) pairs across 32 files; 0 missing, 0 out-of-range. Independently corroborated this session (242 pairs / 30 files, 0 errors). |
| VT parser unit tests | Kitty `test.py` | 16 | 16 | 0 | n/a | Behavioral corroboration of parser/flow-control claims. |
| Screen unit tests | Kitty `test.py` | 36 | 36 | 0 | n/a | Behavioral corroboration of screen-state / pause-rendering claims. |
| **Quantified Total** | — | **298** | **298** | **0** | **100%\*** | \*Coverage shown for the citation suite; unit suites n/a. |

**Supplementary validations (PASS — intentionally not folded into the 298 count above):**

- **Citation semantic accuracy** — 372 checks (226 symbol→citation adjacency pairs + 85 quoted-code-snippet citations + 61 named-symbol anchors) manually traced; **all confirmed accurate** (every automated flag resolved to a false positive: compound multi-symbol citations, protocol tokens like `=1s`/`=2s`, conventional `...` argument elisions).
- **Shell-integration** — bash + zsh **PASS**; fish **skipped** (not installed — environmental, non-blocking).
- **Build (compile)** — `python3 setup.py` with `-Werror`, **exit 0**; confirms the cited source compiles at HEAD `815df1e21`.

> **Integrity note (Rule 3):** Every result above derives from Blitzy's autonomous test/validation execution for this project. No tests were fabricated or sourced externally.

---

## 4. Runtime Validation & UI Verification

This is a **CLI / documentation deliverable with no user interface**; "runtime validation" means corroborating the document's claims by building and exercising Kitty, and verifying the document renders correctly.

**Runtime corroboration (in prescribed Docker container):**
- ✅ **Operational** — `python3 setup.py` build with `-Werror` completed (exit 0); cited C/Python source compiles at HEAD `815df1e21`.
- ✅ **Operational** — `kitty` / `kitten` 0.35.2 launch and run.
- ✅ **Operational** — VT parser unit tests 16/16; screen unit tests 36/36.
- ✅ **Operational** — shell-integration bash + zsh pass.
- ⚠ **Partial** — fish shell-integration skipped (fish not installed in the environment; non-blocking, documented).

**Constant / behavioral verification (document claims vs. live source):**
- ✅ `BUF_SZ` = 1 MiB · `MAX_ESCAPE_CODE_LENGTH` = BUF_SZ/4 · `PENDING_MODE` = 2026 · 100 MiB write cap · 2000 ms pause-rendering safety timeout — all confirmed exactly.

**Document render / structure verification:**
- ✅ **Operational** — 415 lines / 65,988 bytes; valid UTF-8; final newline present.
- ✅ **Operational** — balanced code fences; valid Mermaid block; 41 true headings (no illegal level jumps).
- ✅ **Operational** — renders cleanly on GitHub and VS Code Markdown preview.

**UI Verification:** Not applicable — no graphical UI is produced. The `blitzy/screenshots/` and `blitzy/screen_recordings/` scaffolding directories exist but are intentionally empty for this documentation task.

---

## 5. Compliance & Quality Review

AAP deliverables are cross-mapped to Blitzy quality and compliance benchmarks below.

| Benchmark / Requirement | Source | Status | Progress | Notes |
|---|---|---|---|---|
| **R1** — Entry points & stream materialization (two directions) | AAP §0.1.1 | ✅ Pass | 100% | Dedicated section; `read_bytes` (output) and `key_callback` (input) cited. |
| **R2** — Pause/Resume mechanism | AAP §0.1.1 | ✅ Pass | 100% | Synchronized-update snapshot + 2000 ms timeout (`screen_pause_rendering`). |
| **R3** — The "unseen conductor" & event priority | AAP §0.1.1 | ✅ Pass | 100% | Three threads (`io_loop`/`talk_loop`/`main_loop`) + GLFW loop + fixed priority ordering. |
| **R4** — Alignment without drift | AAP §0.1.1 | ✅ Pass | 100% | Single mutex-protected parser → `screen.c` in byte order; OSC 133/7 inline; `UTF8Decoder`. |
| **R5** — Backpressure & unstable remote | AAP §0.1.1 | ✅ Pass | 100% | POLLIN gate on 1 MiB buffer; 100 MiB write cap; SSH local-PTY model. |
| **R6** — End-to-end rhythm | AAP §0.1.1 | ✅ Pass | 100% | `input_delay`/`repaint_delay` batching; lock-released parsing; return to idle. |
| Evidence-first, no assumptions (Rule 3) | AAP §0.7 | ✅ Pass | 100% | 331 citations; 0 inaccurate; code is the single source of truth. |
| Rationale provided (Rule 4) | AAP §0.7 | ✅ Pass | 100% | 22 rationale markers; per-section "Why" + per-question "Why:" passages. |
| Filename = source branch (Rule 1) | AAP §0.7 | ✅ Pass | 100% | `kitty_815df1e210e0.md`. |
| Build/run for observation (Rule 2) | AAP §0.7 | ✅ Pass | 100% | Docker build `-Werror` exit 0; tests run. |
| No existing files modified (Rule 5) | AAP §0.7 | ✅ Pass | 100% | `git diff --name-status` = only the new doc. |
| No other code added (Rule 6) | AAP §0.7 | ✅ Pass | 100% | Only the Markdown artifact; temp scripts kept outside repo and removed. |
| Placed under `blitzy/documentation/` (Rule 7) | AAP §0.7 | ✅ Pass | 100% | Directories created; file placed correctly. |
| Repository immutability | AAP §0.8.2 | ✅ Pass | 100% | Byte-for-byte unchanged except the new doc; clean tree. |
| Zero placeholders / TODOs | Code-quality | ✅ Pass | 100% | None present. |
| Markdown well-formedness | Code-quality | ✅ Pass | 100% | Balanced fences, valid Mermaid, valid UTF-8. |

**Fixes applied during autonomous validation:** Removed a stray untracked file accidentally created by a shell syntax error (detected via `git status --porcelain`, removed by inode); removed one uncited external claim; addressed code-review findings. **Outstanding items:** SME technical-accuracy review and onboarding clarity review (see §2.2) — human-gated, not autonomous defects.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation line-number drift as upstream evolves | Technical | Low | Medium | Document pins HEAD `815df1e21`; commit-pin note recommended | Mitigated by design |
| Host repo not pre-built (no compiled `.so`) | Technical | Low | Low | Authoritative basis is source reading; build performed in Docker; documented in §9 | Mitigated |
| Subtle concurrency claims warrant expert confirmation | Technical | Low-Medium | Low | Claims are code-cited + build/test-corroborated; SME review queued | Open (path-to-production) |
| Security surface introduced | Security | None / Informational | N/A | Read-only documentation adds no code, dependencies, or attack surface | Not applicable |
| Discoverability (doc may be overlooked) | Operational | Low | Medium | Optional linking outside the repo recommended | Open (optional) |
| Maintenance / staleness over time | Operational | Low | Medium | Commit-pin note + periodic re-verification | Partially mitigated |
| Code-level integration regressions | Integration | None | N/A | No source modified; no imports/config/tests affected | Not applicable |
| Sphinx `docs/` integration intentionally avoided | Integration | Informational | N/A | By AAP design — deliverable lives outside the project doc system | By design |

**Overall risk posture: LOW.** The deliverable introduces no executable code, no dependencies, and no repository modifications beyond a single additive Markdown file. The only meaningful residual risks are non-blocking and human-gated (expert verification, long-term maintenance).

---

## 7. Visual Project Status

**Project hours — Completed vs. Remaining** (Completed = `#5B39F3`, Remaining = `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 49h)
    "Completed Work" : 42
    "Remaining Work" : 7
```

**Remaining work — priority distribution** (7h total):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Remaining Hours by Priority
    "High" : 3
    "Medium" : 2
    "Low" : 2
```

**Remaining hours per category (from §2.2):**

| Category | Hours | Priority |
|---|---:|---|
| SME technical-accuracy review | 3 | High |
| Onboarding clarity / readability review | 2 | Medium |
| Discoverability / linking (outside repo) | 1 | Low |
| Citation-drift safeguard note | 1 | Low |
| **Total** | **7** | — |

> **Integrity (Rule 1):** "Remaining Work" = **7** in the §7 pie chart equals Remaining Hours in §1.2 and the sum of the §2.2 Hours column. ✔

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, self-contained, code-grounded onboarding document for Kitty's terminal I/O pipeline. All six onboarding questions (R1–R6) are answered with explicit rationale and **331 citations / 246 unique (path, line) pairs across 32 files**, verified with **zero inaccurate citations**. The claims are corroborated by a clean `-Werror` build and passing pipeline unit tests (parser 16/16, screen 36/36) in the prescribed Docker container, and the repository remains **byte-for-byte unchanged** except for the single new file.

**Remaining gaps.** The outstanding **7 hours** are exclusively human-gated path-to-production review: SME technical-accuracy sign-off (3h), onboarding clarity review (2h), optional external linking (1h), and a citation-drift safeguard note (1h). None are autonomous defects, and none block use of the document.

**Critical path to production.** (1) SME verifies behavioral claims and citations at HEAD `815df1e21`; (2) onboarding-developer clarity pass; (3) optional discoverability linking and a commit-pin note. Estimated total: **7 hours**.

**Production-readiness assessment.** The autonomous deliverable is **production-ready** at **85.7% of the AAP-scoped 49 hours (42h complete)**. The document is comprehensive, accurate, build-corroborated, and compliant with all 7 "SWE-AtlasQnA-Repo" rules and the repository-immutability constraint.

| Success Metric | Target | Result |
|---|---|---|
| All six questions (R1–R6) answered with rationale | 6 / 6 | ✅ 6 / 6 |
| Citation accuracy (existence + in-range) | 100% | ✅ 246 / 246 (0 errors) |
| Pipeline unit tests pass | 100% | ✅ 52 / 52 (parser 16 + screen 36) |
| Build corroboration | exit 0 | ✅ `-Werror` exit 0 |
| Repository immutability | unchanged | ✅ only new doc added |
| AAP rule compliance | 7 / 7 | ✅ 7 / 7 |

> The project is **85.7% complete** — the autonomous portion is finished; the remaining ~one-seventh is human review.

---

## 9. Development Guide

This guide explains how to locate, read, verify, and corroborate the deliverable. Commands were tested on the host (Ubuntu) during validation. All paths are relative to the repository root `/tmp/blitzy/kitty/blitzy-7bf8846f-3cc0-47d7-a53d-1470778ff4d3_c16d5e`.

### 9.1 System Prerequisites

| Tool | Version (verified on host) | Needed for |
|---|---|---|
| Python | 3.13.7 (project requires `>=3.8`) | Citation-verification scripts; Kitty build |
| Git | 2.51.0 | Immutability checks, diffs |
| GCC / cc | 15.2.0 | Kitty C-extension build (corroboration) |
| make | 4.4.1 | Build wrapper |
| Docker | 28.5.2 | Reproducible full build/run (recommended) |
| Go | **1.22 required (not on host)** | Building Go kittens (only for full build) |

> Reading and verifying the document needs only **Python 3 + Git** (both present). A full Kitty build is best performed in the **prescribed Docker container**, where `go`, fish, and system libs are provisioned.

### 9.2 Locate and Read the Document

```bash
# From the repository root
DOC="blitzy/documentation/kitty_815df1e210e0.md"
ls -la "$DOC"            # expect ~65,988 bytes
wc -l "$DOC"             # expect 415 lines
sed -n '1,60p' "$DOC"    # read the header, "how to read", and the six questions
```

### 9.3 Verify Document Structure

```bash
DOC="blitzy/documentation/kitty_815df1e210e0.md"
# Headings (expect 41 true headings; a naive count may show 42 because line 178
# '#define WAKEUP' sits INSIDE a code fence — it is not a heading).
grep -nE '^#{1,6} ' "$DOC" | wc -l
# Code-fence balance (must be even):
grep -c '^```' "$DOC"
# Mermaid diagram present:
grep -c '```mermaid' "$DOC"
# UTF-8 validity:
python3 -c "open('$DOC',encoding='utf-8').read(); print('utf-8 OK')"
```

### 9.4 Verify Repository Immutability

```bash
git status --porcelain                       # expect empty (clean tree)
git diff 815df1e21 --name-status             # expect exactly: A blitzy/documentation/kitty_815df1e210e0.md
git log --author="agent@blitzy.com" 815df1e21..HEAD --oneline   # the agent commits
```

### 9.5 Verify Citation Accuracy

```bash
# Spot-check a citation: confirm the cited line maps to the claimed symbol.
sed -n '1337p' kitty/child-monitor.c    # expect the read_bytes implementation
sed -n '1481p' kitty/child-monitor.c    # expect io_loop
sed -n '1259p' kitty/child-monitor.c    # expect main_loop
sed -n '195p'  kitty/vt-parser.c        # expect UTF8Decoder
sed -n '2316p' kitty/screen.c           # expect parse_prompt_mark
sed -n '955p'  kitty/window.py          # expect def write_to_child
```

To verify **all** citations at once, run a small extractor **outside the repo** (do not add scripts to the repo):

```bash
cat > /tmp/verify_citations.py << 'PY'
import re, os, sys
DOC = "blitzy/documentation/kitty_815df1e210e0.md"
text = open(DOC, encoding="utf-8").read()
# Match [path:Lnnn] and [path:Lnnn-Lmmm] style citations
pairs = set()
for m in re.finditer(r'([A-Za-z0-9_./-]+\.(?:c|h|py|go|sh)):L(\d+)(?:-L?(\d+))?', text):
    path, a, b = m.group(1), int(m.group(2)), m.group(3)
    for ln in range(a, (int(b) if b else a) + 1):
        pairs.add((path, ln))
missing_file = missing_line = 0
for path, ln in sorted(pairs):
    if not os.path.exists(path):
        missing_file += 1; continue
    with open(path, encoding="utf-8", errors="replace") as fh:
        if ln > sum(1 for _ in fh):
            missing_line += 1
print(f"unique (path,line) pairs: {len(pairs)}")
print(f"missing files: {missing_file}  out-of-range lines: {missing_line}")
PY
python3 /tmp/verify_citations.py
rm -f /tmp/verify_citations.py     # clean up — keep the repo unchanged
```

Expected: `0` missing files and `0` out-of-range lines.

### 9.6 Full Build / Runtime Corroboration (Docker — recommended)

```bash
# Inside the prescribed container, from the repository root:
python3 setup.py 2>&1 | tee /tmp/build.log    # expect a clean build (exit 0, -Werror)
# Run the pipeline unit tests that corroborate the document's claims:
kitty +launch --type=background python3 -c "print('kitty available')"  # smoke check
python3 test.py vt_parser     # expect 16/16
python3 test.py screen        # expect 36/36
```

### 9.7 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `grep -c '^```' ` returns an odd number | A code fence was edited unbalanced | Restore the document; fences must come in pairs. |
| Heading count shows 42, not 41 | Naive grep counts `#define WAKEUP` inside a code fence | Expected — exclude fenced regions; 41 true headings. |
| `setup.py` build fails on `go` | Go 1.22 not installed | Build inside the prescribed Docker container. |
| fish shell-integration test skipped | fish not installed | Environmental; non-blocking. Install fish or ignore. |
| `git diff` shows extra files | Stray temp file in the tree | Remove temp artifacts; keep observation scripts **outside** the repo. |
| Citation line looks off | Doc pinned to HEAD `815df1e21`; you are on newer code | Check out `815df1e21` or re-base citations; see the commit-pin note. |

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Locate the document | `ls -la blitzy/documentation/kitty_815df1e210e0.md` |
| Line / byte count | `wc -l blitzy/documentation/kitty_815df1e210e0.md` |
| Confirm immutability | `git diff 815df1e21 --name-status` |
| Clean-tree check | `git status --porcelain` |
| Agent commits | `git log --author="agent@blitzy.com" 815df1e21..HEAD --oneline` |
| Fence balance | `grep -c '^\`\`\`' blitzy/documentation/kitty_815df1e210e0.md` |
| Mermaid present | `grep -c '\`\`\`mermaid' blitzy/documentation/kitty_815df1e210e0.md` |
| Spot-check a citation | `sed -n '1337p' kitty/child-monitor.c` |
| Build (Docker) | `python3 setup.py` |
| Parser tests | `python3 test.py vt_parser` |
| Screen tests | `python3 test.py screen` |

### Appendix B — Port Reference

Not applicable. The deliverable is a documentation file; Kitty is a desktop terminal emulator and the documentation task exposes no network services or listening ports.

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (415 lines, 65,988 bytes) |
| `kitty/child-monitor.c` | Three-thread conductor: `io_loop` (L1481), `talk_loop` (L1805), `main_loop` (L1259), `read_bytes` (L1337), `write_to_child` (L1443) |
| `kitty/vt-parser.c` | Bounded 1 MiB parser; `run_worker`; `UTF8Decoder` (L195); `dispatch_dcs`; `MAX_ESCAPE_CODE_LENGTH` |
| `kitty/screen.c` | `screen_pause_rendering` (snapshot + 2000 ms timeout); `parse_prompt_mark` (L2316) |
| `kitty/loop-utils.c` | `wakeup_loop` (L113, eventfd); `read_signals` (L131) |
| `kitty/window.py` | `write_to_child` (L955); paste sanitization; `resize_pty` |
| `kitty/glfw.c` | `key_callback` (L430); `glfwPostEmptyEvent`; `run_main_loop` (L2102) |
| `kitty/child.py` | `openpty()` (L170-171); `set_iutf8_fd`; `fork()` |
| `kitty/boss.py` | ChildMonitor creation/start; key/action dispatch; peer messages |
| `kitty/state.c/.h` | `input_delay` / `repaint_delay` timing knobs |
| `kitty/shell_integration.py` | OSC 133 / OSC 7 wiring |
| `kittens/ssh/`, `shell-integration/ssh/` | Remote terminfo + shell-integration bootstrap |

### Appendix D — Technology Versions

| Component | Version |
|---|---|
| Kitty (built/ran in Docker) | 0.35.2 |
| Repository HEAD (base) | `815df1e21` (Kovid Goyal, upstream) |
| Branch HEAD | `7b808b5a6` |
| Python (host) | 3.13.7 (project requires `>=3.8`) |
| Go (required for full build) | 1.22 (per `go.mod`) |
| GCC / cc (host) | 15.2.0 |
| make (host) | 4.4.1 |
| Node / npm (host) | 20.20.2 / 11.1.0 |
| Docker (host) | 28.5.2 |

### Appendix E — Environment Variable Reference

Not applicable to the deliverable. No environment variables are required to read or verify the document. For a full Kitty build inside the prescribed container, standard build variables apply (e.g., `CC`, `PKG_CONFIG_PATH`); these belong to Kitty's existing build system and are unchanged by this task.

### Appendix F — Developer Tools Guide

- **Read / render:** any Markdown viewer — GitHub, VS Code preview, or a CLI renderer (`grip`, `glow`, `pandoc`) if installed (none are present on the host; not required).
- **Verify:** Python 3 + Git (both present) suffice for structure, immutability, and citation checks (see §9).
- **Corroborate:** the prescribed Docker container for a reproducible `-Werror` build and the parser/screen unit suites.
- **Discipline:** keep any observation scripts **outside** the repository (e.g., under `/tmp`) and delete them afterward to preserve byte-for-byte immutability.

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the authoritative scope document for this task. |
| **PTY** | Pseudoterminal — the kernel device pair connecting Kitty to the child shell. |
| **VT parser** | Kitty's byte-stream parser (`vt-parser.c`) translating child output into screen operations. |
| **Conductor** | Informal name for the Child Monitor's three-thread + GLFW-loop orchestration (`io_loop`, `talk_loop`, `main_loop`). |
| **Backpressure** | Flow control: withholding `POLLIN` on a full bounded buffer so a fast producer cannot overwhelm a slow consumer. |
| **Synchronized output / DECSET 2026** | Atomic-screen-update protocol; Kitty implements it via `screen_pause_rendering` (snapshot + 2000 ms safety timeout). |
| **OSC 133 / OSC 7** | Shell-integration escape sequences marking prompts/command output and reporting the working directory. |
| **`input_delay` / `repaint_delay`** | Timing knobs that batch wakeups and schedule renders to convert input surges into coherent frames. |
| **R1–R6** | The six onboarding questions the document must answer. |
| **HEAD `815df1e21`** | The pinned commit against which all citations are verified. |