# Blitzy Project Guide — kitty Keyboard-Protocol Flag-Stack Investigation

> Deliverable branch: `blitzy-e4a46922-de58-46cc-9578-befb377c660a` • Base commit: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> Task type: **Read-only documentation / runtime investigation** • Sole committed artifact: `blitzy/documentation/kitty_815df1e210e0.md`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-backed technical answer document explaining — from **observed runtime behavior**, not code reading alone — how kitty's keyboard-protocol progressive-enhancement flag stack (`CSI > u` push / `CSI < u` pop) behaves when the terminal switches between its **main** and **alternate** screen buffers. The target user is a developer writing a text-mode application that pushes its own keyboard-enhancement flags on the alternate screen and observed order-dependent behavior. The business impact is an authoritative, byte-exact resolution of that confusion, preventing subtle input-handling bugs. The technical scope is a read-only investigation of kitty's C screen model at commit `815df1e210e0`, producing one committed Markdown deliverable backed by reproducible captures and `file:line` citations.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'17px'}}}%%
pie showData title Completion — 90.6% Complete (hours)
    "Completed Work (AI)" : 29
    "Remaining Work" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **32** |
| Completed Hours (AI + Manual) | 29 (AI: 29 · Manual: 0) |
| Remaining Hours | 3 |
| **Percent Complete** | **90.6%**  — `29 / (29 + 3) = 90.6%` |

> Color key (applied throughout): **Completed / AI work = Dark Blue `#5B39F3`** · **Remaining = White `#FFFFFF`**.

### 1.3 Key Accomplishments

- ✅ **Sole deliverable authored & committed** — `blitzy/documentation/kitty_815df1e210e0.md` (1,084 lines, 12 sections) across 4 `agent@blitzy.com` commits atop the pinned base.
- ✅ **All 7 named questions answered** — stack ownership, round-trip survival, per-state bytes, overflow/exhaustion, real-PTY child capture, independence/leakage, and mode dependencies.
- ✅ **Run-first methodology honored** — every behavioral claim is backed by captured runtime output from a canonical build (not static reading).
- ✅ **Decisive proof reproduced** — round-trip readings `1 → 0 → 8 → 1` confirm physically independent per-buffer stacks (re-verified independently this session).
- ✅ **Headline correction established** — `Ctrl+Shift+a` encodes to `\x1b[97;6u` in every state (NOT `0x01`; `0x01` is `Ctrl+a` without Shift), with plain `a` proving distinct per-buffer modes (`61` under flag 1 vs `\x1b[97u` under flag 8).
- ✅ **Capacity recovered from source & confirmed at runtime** — exactly 8 slots per buffer with silent evict-oldest (`memmove`); pop-to-empty resets to 0.
- ✅ **Authoritative child bytes captured** — 6-case real forked-child-over-real-PTY capture, all `MATCH=True`.
- ✅ **Two-stack disambiguation** — progressive-enhancement C stack cleanly separated from the unrelated `kitty.conf` `keyboard_mode_stack`.
- ✅ **Determinism proven** — outputs md5-identical across ≥2 repeated runs; the user's "order-dependent" report explained as deterministic per-buffer independence.
- ✅ **Read-only constraint held** — `git diff` shows a single file added (+1,084 / −0); working tree clean; no source/test/config/doc modified.
- ✅ **91 `file:line` citations** verified at HEAD; canonical `kitty_tests` pass (keys 3/3, screen 36/36); clean `-Werror` build (122 objects, 0 warnings).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All AAP-scoped requirements are complete and independently verified. No blocking or release-critical issues remain. | — | — | — |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| Repository (`blitzy-…` branch) | Git read/write | None — branch present, deliverable committed, working tree clean | ✅ Resolved / N/A | — |
| Canonical build image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`) | Container runtime | None — image present and running (`kitty-setup`); build + tests reproduce | ✅ Resolved / N/A | — |
| GUI display (`DISPLAY` / `WAYLAND_DISPLAY`, `xvfb`) | Local display server | Not available in the headless environment; affects only the interactive full-window capture (one hop) | ⚠ Documented & Mitigated (source-equivalence + real-PTY capture) — not blocking | Reviewer (optional) |

> No blocking access issues identified. The single environment constraint (no GUI display) is honestly documented in the deliverable (§2.4 / §7.3) and closed by source-equivalence plus the canonical real-PTY capture.

### 1.6 Recommended Next Steps

1. **[High]** Perform the human technical review of `blitzy/documentation/kitty_815df1e210e0.md` — confirm it fully answers all 7 questions and the user's order-dependent scenario, and internalize the headline correction (`\x1b[97;6u`, not `0x01`).
2. **[High]** Verify read-only scope compliance (`git diff 815df1e21 --name-status` = single add) and accept/merge the single-file deliverable.
3. **[Low]** _(Optional)_ Independently reproduce the observation scripts in the canonical container and compare stdout md5s to the documented §11.1 hashes.
4. **[Low]** _(Optional)_ Share the "push on the buffer you intend to use" guidance with the requesting application team as the actionable takeaway.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All rows are AI/autonomous work and trace to specific AAP requirements (`R#` from the requirements inventory).

| Component | Hours | Description |
|-----------|------:|-------------|
| Canonical build + environment setup | 2 | `python3 setup.py` in the pinned Docker image; toolchain (Python 3.12.3, gcc 13.3.0); produced `fast_data_types.so` + launcher; import verified. [R1] |
| Source-code mechanism investigation | 4 | Read `screen.h`/`screen.c`, `vt-parser.c`, `keys.c`, `key_encoding.c`, `modes.h` + `docs/keyboard-protocol.rst` to establish the stack storage, buffer-switch pointer-repoint, dispatch, and encoder. [R2, R8] |
| Observation harness authoring | 6 | Wrote `kbd_probe` (TESTS 1–9), `kbd_legacy`, `kbd_capacity`, and `kbd_pty` (real forked-child-over-PTY) exercising the real code paths headlessly. [R3–R8, R10] |
| Runtime observation runs | 3 | Executed scenarios with ≥2–3 repetitions, md5-hashed stdout, real-PTY capture, and canonical `kitty_tests` (keys/screen). [R6, R12] |
| Web-search spec + RFC corroboration | 1 | Cross-checked the in-repo spec against the published kitty keyboard-protocol spec and RFC #3248. [R14] |
| Answer document authoring | 8 | Authored the 1,084-line, 12-section deliverable: TL;DR direct answer, per-question analysis, four-state byte table, raw outputs, cause→effect, two-stack disambiguation, determinism. [R2–R13, R17] |
| Citation map + grounding | 2 | Full citation map of 91 `file:line` references verified at HEAD. [R15] |
| Code-review response iterations | 3 | 3 follow-up commits addressing review findings, citation line-number fixes, and `show-key`/appendix-path/code-quote-elision fixes. [R17] |
| **Total Completed** | **29** | **Matches Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

Each category traces to a path-to-production need for a documentation deliverable (there is no deploy/CI/CD/integration path for a Markdown Q&A artifact).

| Category | Hours | Priority |
|----------|------:|----------|
| Human technical review & acceptance of the answer document | 2 | High |
| Optional independent reproduction of observation scripts in reviewer's environment | 1 | Low |
| **Total Remaining** | **3** | **Matches Remaining Hours in §1.2 and §7 pie.** |

### 2.3 Hours Reconciliation & Methodology

- **Methodology (PA1, AAP-scoped):** Completion % counts only work in the AAP scope plus path-to-production. Formula: `Completed / (Completed + Remaining) × 100`.
- **Calculation:** `29 / (29 + 3) = 29 / 32 = 90.625% → 90.6%`.
- **Cross-section checks:** §2.1 total (29) + §2.2 total (3) = **32** = §1.2 Total Hours (Rule 2 ✅). Remaining = **3** in §1.2, §2.2, and §7 pie (Rule 1 ✅).
- **Confidence:** High. The deliverable is single-file, complete, committed, and every core claim was independently reproduced this session in the canonical container.

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** and were re-confirmed this session in the canonical container (`kitty-setup`, Python 3.12.3). Coverage % is marked N/A because this is a **read-only** task that adds no new production code to cover.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Unit — key encoding | `kitty_tests` (Python `unittest` via `test.py`) | 3 | 3 | 0 | N/A | `--module keys`; re-verified 3/3 OK this session |
| Unit — screen / buffer | `kitty_tests` (Python `unittest` via `test.py`) | 36 | 36 | 0 | N/A | `--module screen`; re-verified 36/36 OK this session |
| Behavioral observation (headless) | Custom scripts on real `Screen` + VT-parser + encoder | 9 scenarios (4 scripts) | 9 | 0 | N/A | `kbd_probe` (T1–9), `kbd_legacy`, `kbd_capacity`, `kbd_pty`; stdout md5-identical across ≥2 repeats |
| Integration — real PTY child boundary | Forked child + real PTY (`kitty.child.openpty`) | 6 | 6 | 0 | N/A | All `MATCH=True` (encoder bytes == child-received bytes) |
| Build validation | `setup.py` under `-pedantic-errors -Werror` | 122 objects | 122 | 0 | N/A | From-scratch build exited 0 with 0 warnings |

**Aggregate:** 45 discrete pass/fail cases (3 + 36 + 6) — **all passing** — plus 9 behavioral-observation scenarios reproduced byte-identical, plus a clean `-Werror` build across 122 objects. **Zero failures.**

**Reproduction hashes (from deliverable §11.1, autonomous logs):**

| Script | md5sum of stdout | Result |
|--------|------------------|--------|
| `kbd_probe.py` | `c97899b427edf53a48b465609a6251e1` | identical every run |
| `kbd_legacy.py` | `287f2b8a82ccb4c34c9e0ab5a293eeca` | identical every run |
| `kbd_capacity.py` | `96d2ccf2d8e0babbde504024e9812454` | identical every run |
| `kbd_pty.py` | `d957420e7f562331ac7d908c50252632` | identical every run |

---

## 4. Runtime Validation & UI Verification

**Runtime health (canonical container, Python 3.12.3):**

- ✅ **Build & extension** — `kitty/fast_data_types.so` and `./kitty/launcher/kitty` present; `import kitty.fast_data_types` succeeds; `Screen` and `encode_key_for_tty` exposed.
- ✅ **Headless primitives** — `BaseTest().create_screen()`, `parse_bytes()`, `toggle_alt_screen()`, and `current_key_encoding_flags()` all functional.
- ✅ **Decisive round-trip** — reproduced `current_key_encoding_flags()` readings `[0, 1, 0, 8, 1]` (decisive `1 → 0 → 8 → 1` = True).
- ✅ **Per-state bytes** — `Ctrl+Shift+a` = `\x1b[97;6u` (`1b 5b 39 37 3b 36 75`); plain `a` = `61` (flag 1) vs `\x1b[97u` (flag 8); `Ctrl+a` = `0x01`.
- ✅ **Capacity & reset** — per-buffer capacity measured at exactly 8; pop-to-empty returns 0.
- ✅ **Canonical tests** — `kitty_tests` keys 3/3 OK, screen 36/36 OK.

**Child-boundary / API integration (the user's literal question):**

- ✅ **Real PTY capture** — 6/6 cases `MATCH=True`; encoder output equals bytes the forked child actually `read()`s at the real PTY boundary.

**UI verification:**

- ⚠ **Interactive GUI window** — Not exercised end-to-end: the headless environment has no `DISPLAY`/`WAYLAND_DISPLAY` and no `xvfb`. This affects exactly one hop (GLFW windowing event → `encode_glfw_key_event`). It is **mitigated** by source-equivalence — the GUI path (`kitty/keys.c:L251`) and the exercised binding (`kitty/keys.c:L319`) call the same encoder — and by the canonical real-PTY capture, which is authoritative for "bytes transmitted to the child." Honestly labeled in the deliverable (§2.4, §7.3). Not a defect.

> Note: This deliverable is a documentation artifact; there is no application UI to verify. "UI verification" is therefore limited to the terminal's key-encoding output behavior, which is fully validated via the PTY capture above.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the governing "SWE-AtlasQnA-Repo" rules to observed compliance.

| Benchmark / Rule | Requirement | Status | Evidence |
|------------------|-------------|:------:|----------|
| Deliverable rule | Single Markdown answer at `blitzy/documentation/<branch>.md` | ✅ Pass | `blitzy/documentation/kitty_815df1e210e0.md` committed (1,084 lines) |
| Run-first methodology | Build & run before writing; claims from observation | ✅ Pass | 4 observation scripts + canonical build; §2, §11 |
| Evidence & output rules | Complete, unedited output + producing command per claim | ✅ Pass | Raw outputs embedded per test; §4–§9 |
| Byte-exactness | Byte-sensitive results verified against emitted bytes | ✅ Pass | Hex + repr for every capture; §5, §7 |
| Real entry point (no bypass) | Exercise real code path, label non-canonical | ✅ Pass | Real VT-parser/encoder/PTY; GUI hop labeled §7.3 |
| Coverage & grounding | Answer every named item; `file:line` for each claim | ✅ Pass | All 7 Qs + user example + plain-`a`; 91 citations §12.1 |
| Two-stack disambiguation | Do not conflate with `keyboard_mode_stack` | ✅ Pass | §10 explicit separation |
| Determinism | Repeat identical trials ≥2×; report stability | ✅ Pass | md5-identical ×2–3; §11.1 |
| Scope rule (read-only) | No existing file modified; temp scripts removed | ✅ Pass | `git diff` = 1 file added; tree clean |
| Canonical build/config | Default build; state exact commands | ✅ Pass | `-Werror` default; commands in §2.1 |
| Documentation quality | Well-formed, no placeholders | ✅ Pass | 82 balanced code fences; 0 TODO/FIXME/placeholder |
| Citation accuracy | Citations valid at pinned commit | ✅ Pass | 91 refs confirmed at HEAD |

**Fixes applied during autonomous validation:** 3 review-driven commits (code-review findings; citation line-number corrections; `show-key` command + appendix citation paths + code-quote elision). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|:--------:|:-----------:|------------|:------:|
| Prebuilt `fast_data_types.so` links `libpython3.12`; reviewer on a different Python cannot import without rebuilding | Technical | Low | Medium | Deliverable §2.1 gives exact canonical build commands + Docker image tag; rebuild in-image | ⚠ Mitigated |
| Citation line numbers drift if read against a non-pinned commit | Technical | Low | Low | §12.1 pins commit `815df1e210e0`; reviewer checks out the pin | ⚠ Mitigated |
| Per-buffer capacity (8) is implementation-defined and could change in a future kitty release | Technical | Low | Low | Documented as implementation-defined + source-grounded (`screen.h:L128`) and runtime-confirmed | ✅ Accepted |
| No executable code / dependencies / credentials / network surface introduced | Security | None | N/A | Read-only Markdown deliverable; no attack surface | ✅ N/A |
| GUI display path (GLFW event → encoder) not exercised end-to-end (no `DISPLAY`/`xvfb`) | Operational | Low | Low | Source-equivalence (`keys.c:L251 ≡ L319`) + canonical real-PTY capture; labeled §2.4/§7.3 | ⚠ Mitigated |
| Reproducibility depends on the canonical Docker image remaining available | Operational | Low | Low | Image tag documented; observation scripts embedded verbatim in §12.4 for re-creation | ⚠ Mitigated |
| No external service / API / credential dependencies | Integration | None | N/A | The single un-exercised GUI→encoder hop is closed by source-equivalence | ✅ N/A |

**Overall risk posture: LOW** across all four categories — appropriate for a read-only, fully-validated documentation deliverable.

---

## 7. Visual Project Status

**Project hours — Completed vs Remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'17px'}}}%%
pie showData title Project Hours Breakdown (Total 32h)
    "Completed Work" : 29
    "Remaining Work" : 3
```

**Remaining work by priority** (hours from §2.2):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'15px'}}}%%
pie showData title Remaining Hours by Priority (Total 3h)
    "High (review & accept)" : 2
    "Low (optional re-run)" : 1
```

**Remaining hours per category (bar view):**

| Category | Hours | Bar |
|----------|------:|-----|
| Human technical review & acceptance | 2 | ██████████████████████ |
| Optional independent reproduction | 1 | ███████████ |

> **Integrity:** "Remaining Work" = **3** here equals Remaining Hours in §1.2 and the sum of §2.2 (2 + 1). "Completed Work" = **29** equals Completed Hours in §1.2 and the sum of §2.1.

---

## 8. Summary & Recommendations

**Achievements.** The project fully delivers its sole AAP-scoped artifact: a rigorous, evidence-backed answer document proving that each kitty screen buffer owns a physically separate 8-slot keyboard-flags array, that switching buffers merely repoints an active pointer (no copy, no clear), and that a stack pushed on main therefore survives an alternate-screen round-trip untouched (`1 → 0 → 8 → 1`). All 7 named questions are answered with captured runtime output, the user's verbatim `Ctrl+Shift+a` example is exercised across all four states (with the important correction that it encodes to `\x1b[97;6u`, not `0x01`), and the flag-1-vs-flag-8 distinction is proven via a plain `a`. Overflow (silent evict-oldest at capacity 8), pop-to-empty reset, cross-buffer isolation, and mode dependencies (DECSET 1049/1047/47; legacy lock-strip) are all demonstrated. The authoritative child bytes are captured at a real PTY boundary (6/6 `MATCH`).

**Remaining gaps.** None within the autonomous scope. The only outstanding work is the human review/acceptance gate inherent to any documentation deliverable, plus optional independent reproduction — **3 hours** total.

**Critical path to production.** (1) Human technical review & acceptance → (2) verify read-only scope & merge. That is the entire path; there is no build/deploy/CI/CD for a Markdown Q&A artifact.

**Success metrics.** 17/17 AAP-scoped requirements complete; 45/45 pass-fail test cases passing; 9/9 behavioral scenarios reproduced byte-identical; 6/6 PTY cases `MATCH`; 0 build warnings under `-Werror`; 91/91 citations valid at the pin; read-only constraint held (single file added).

**Production-readiness assessment.** **90.6% complete (29 of 32 hours).** The deliverable is complete, committed, byte-for-byte reproducible in the canonical environment, and read-only-compliant. It is **ready for human review and acceptance**; the remaining 9.4% reflects that review gate, not any autonomous work deficit.

---

## 9. Development Guide

This guide covers building the kitty C extension and reproducing every observation behind the deliverable. All commands were tested this session in the canonical container.

### 9.1 System Prerequisites

- **Canonical image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (alias `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`).
- **In-image toolchain (verified):** Python **3.12.3**, gcc **13.3.0**, Go **1.23.4**, pkg-config **1.8.1**.
- **Build-time native libs (pre-provisioned):** harfbuzz, libpng, lcms2, fontconfig, freetype, OpenGL, openssl/libcrypto, zlib, xxhash.
- **Repo:** bind-mounted at `/workspace`, checked out at commit `815df1e210e0`.

### 9.2 Environment Setup

```bash
# Confirm the canonical container is running
docker ps --filter "name=kitty-setup" --format "{{.Names}} | {{.Image}} | {{.Status}}"

# Enter it (or prefix subsequent commands with: docker exec kitty-setup bash -lc '...')
docker exec -it kitty-setup bash
cd /workspace
python3 --version   # expect: Python 3.12.3
gcc --version | head -1
```

### 9.3 Dependency Installation & Build

```bash
# From the repo root inside the container. Headless-only build is sufficient
# for the primary (deterministic) evidence; it also builds the launcher.
cd /workspace
python3 setup.py --skip-building-kitten
# -> compiles kitty/fast_data_types.so (hosts Screen + encode_key_for_tty)
#    and ./kitty/launcher/kitty
# Default flags include -pedantic-errors -Werror (do NOT suppress).
```

Expected artifacts:

```bash
ls -la kitty/fast_data_types.so kitty/launcher/kitty
# -rwxr-xr-x ... 1213072 ... kitty/fast_data_types.so
# -rwxr-xr-x ...   36224 ... kitty/launcher/kitty
```

### 9.4 Verification

```bash
# 1) Extension imports and primitives are present
PYTHONPATH=/workspace python3 -c \
  "import kitty.fast_data_types as f; print('import OK; Screen=%s encode=%s' % (hasattr(f,'Screen'), hasattr(f,'encode_key_for_tty')))"
# -> import OK; Screen=True encode=True

# 2) Canonical unit tests
LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch test.py --module keys     # Ran 3 tests ... OK
LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch test.py --module screen   # Ran 36 tests ... OK
```

### 9.5 Example Usage — reproduce the decisive round-trip

```bash
cat > /tmp/dg_probe.py << 'PYEOF'
import sys; sys.path.insert(0, ".")
from kitty_tests import BaseTest, parse_bytes
bt = BaseTest(); s = bt.create_screen()
def push(f): parse_bytes(s, ("\x1b[>%du" % f).encode("latin-1"))
r = [s.current_key_encoding_flags()]            # fresh main -> 0
push(1); r.append(s.current_key_encoding_flags())    # push disambiguate on main -> 1
s.toggle_alt_screen(); r.append(s.current_key_encoding_flags())  # enter alt (own stack) -> 0
push(8); r.append(s.current_key_encoding_flags())    # push report-all-keys on alt -> 8
s.toggle_alt_screen(); r.append(s.current_key_encoding_flags())  # back to main (survived) -> 1
print("readings:", r, "| decisive 1,0,8,1:", r[1:] == [1, 0, 8, 1])
PYEOF
PYTHONPATH=/workspace python3 /tmp/dg_probe.py && rm -f /tmp/dg_probe.py
# -> readings: [0, 1, 0, 8, 1] | decisive 1,0,8,1: True
```

To reproduce the full evidence set, copy the four scripts embedded verbatim in the deliverable §12.4 (`kbd_probe.py`, `kbd_legacy.py`, `kbd_capacity.py`, `kbd_pty.py`) into `/tmp`, run each with `PYTHONPATH=/workspace python3 /tmp/<script>.py`, and compare `md5sum` of stdout to the hashes in §11.1.

### 9.6 Read-only Scope Verification

```bash
git diff 815df1e21 --stat        # 1 file changed, 1084 insertions(+)
git status --porcelain           # (empty output => working tree clean)
wc -l blitzy/documentation/kitty_815df1e210e0.md   # 1084
```

### 9.7 Troubleshooting

- **`ImportError: libpython3.12.so.1.0: cannot open shared object file`** — you are running outside the canonical container (e.g., host Python 3.13). Rebuild and run inside the Python 3.12 image.
- **GUI/full-window capture fails (no `DISPLAY`/`WAYLAND_DISPLAY`, no `xvfb`)** — expected in the headless environment. Use the real-PTY capture (`kbd_pty.py`), which is canonical for the child bytes; the GUI hop is closed by source-equivalence (`kitty/keys.c:L251 ≡ L319`).
- **Citation line numbers don't match** — ensure the working tree is at commit `815df1e210e0`.
- **`parse_bytes` not found on `fast_data_types`** — it is a `kitty_tests` helper (wrapping `test_create_write_buffer` / `test_parse_written_data`); import it from `kitty_tests`, not from `fast_data_types`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Build (headless) | `cd /workspace && python3 setup.py --skip-building-kitten` |
| Import check | `PYTHONPATH=/workspace python3 -c "import kitty.fast_data_types"` |
| Unit tests (keys) | `./kitty/launcher/kitty +launch test.py --module keys` |
| Unit tests (screen) | `./kitty/launcher/kitty +launch test.py --module screen` |
| Run an observation script | `PYTHONPATH=/workspace python3 /tmp/<script>.py` |
| Hash stdout | `PYTHONPATH=/workspace python3 /tmp/<script>.py \| md5sum` |
| Read-only diff | `git diff 815df1e21 --stat` |
| Query flags from a live kitty | `printf '\x1b[?u'` → terminal replies `CSI ? <flags> u` |
| Debug the protocol interactively | `kitten show-key -m kitty` |

### B. Port Reference

Not applicable — this project involves no network services, servers, or listening ports. All observation is in-process (headless `Screen`) or over a local PTY.

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **The deliverable** (only committed change) |
| `kitty/screen.h` | Per-buffer flag arrays + active pointer (`:L128`) |
| `kitty/screen.c` | Stack ops, buffer switch, Python bindings (`:L1068, L1204–L1252, L3951, L4449`) |
| `kitty/vt-parser.c` | CSI-u dispatch (`:L1217–L1240`) |
| `kitty/keys.c` | Encoder call sites + `encode_key_for_tty` (`:L251, L311, L319`) |
| `kitty/key_encoding.c` | Modifier/flag byte encoding + legacy path (`:L11, L36, L34–L50`) |
| `kitty/modes.h` | Alternate-screen DECSET constants (`:L76–L77`) |
| `docs/keyboard-protocol.rst` | Authoritative in-repo spec |
| `kitty_tests/__init__.py` | Headless harness: `parse_bytes` (`:L30`), `create_screen` (`:L237`) |
| `kitty/launcher/kitty` | Built launcher (for `test.py` + GUI corroboration) |
| `kitty/fast_data_types.so` | Built C extension (hosts `Screen`, `encode_key_for_tty`) |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.12.3 | canonical image (verified) |
| gcc | 13.3.0 | canonical image (verified) |
| Go | 1.23.4 | canonical image |
| pkg-config | 1.8.1 | canonical image |
| kitty | commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` | pinned base |

### E. Environment Variable Reference

| Variable | Purpose | Value used |
|----------|---------|------------|
| `PYTHONPATH` | Resolve `kitty.fast_data_types` from the repo root | `/workspace` |
| `LANG` / `LC_ALL` | Deterministic locale for the test runner | `C.UTF-8` |
| `DISPLAY` / `WAYLAND_DISPLAY` | GUI display (empty in headless env — GUI capture skipped by design) | _(unset)_ |

### F. Developer Tools Guide

- **`kitten show-key -m kitty`** — the documented interactive tool to display the raw keyboard-protocol bytes for pressed keys (full-GUI corroboration).
- **`test.py`** — launches `kitty_tests` via `./kitty/launcher/kitty +launch`; supports `--module <name>`.
- **`kitty_tests.BaseTest.create_screen()`** — instantiates a real headless `Screen`; **`parse_bytes()`** feeds it raw escape sequences through the real VT parser.
- **`fast_data_types.encode_key_for_tty(...)`** — the exact encoder kitty uses to write key bytes to the child PTY.
- **`git diff <base> --name-status` / `--stat` / `--numstat`** — verify the read-only scope (single file added).

### G. Glossary

| Term | Meaning |
|------|---------|
| **Progressive-enhancement flags stack** | The per-screen-buffer keyboard-mode stack driven by `CSI > flags u` (push) / `CSI < number u` (pop); the subject of this investigation. |
| **`keyboard_mode_stack`** | An **unrelated** `kitty.conf` key-mapping mode stack (Python, `kitty/keys.py`) — explicitly out of scope. |
| **Main / alternate screen buffer** | The two terminal screen buffers; the alternate is entered via DECSET 1049/1047/47. |
| **Round-trip** | The sequence main→alt→main used to prove main-stack survival (`1 → 0 → 8 → 1`). |
| **Disambiguate (flag 1)** | Progressive-enhancement flag that escape-codes ambiguous/modified keys; a plain letter still sends its literal byte. |
| **Report-all-keys (flag 8)** | Flag that makes every key — including plain printables — a CSI-u sequence. |
| **Evict-oldest** | On overflow past 8 slots, the oldest stack entry is silently dropped via `memmove`; no error is raised. |
| **PTY** | Pseudo-terminal; the real boundary at which the child process receives key bytes. |