# kitty keyboard-protocol flag stack across main / alternate screen buffers — an evidence-backed investigation

**Subject:** the *per-screen* keyboard-protocol **progressive-enhancement flag stack** that kitty stores on each `Screen` object (`kitty/screen.h`, `kitty/screen.c`), and how it behaves when the terminal switches between the **main** screen buffer and the **alternate** screen buffer.

**Source / base commit.** This investigation targets the kitty source at base commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`). That value is the branch's **source/base** commit; it is an **ancestor** of the live `HEAD`, whose tip advances as this document itself is committed — so it is deliberately **not** quoted as the current tip. Resolve the live tip with `git -C <repo> rev-parse HEAD`, and confirm provenance with `git -C <repo> merge-base --is-ancestor 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD` (exit `0` ⇒ the base commit is an ancestor of `HEAD`). The probe below enforces exactly this ancestry check at runtime and refuses to run otherwise.

> **Scope disambiguation (read this first).** This document is about the **C protocol-enhancement flag stack** on the `Screen` object — the two fixed arrays `main_key_encoding_flags[8]` / `alt_key_encoding_flags[8]` at `kitty/screen.h:128`. It is **not** about `keyboard_mode_stack` (`kitty/keys.py:67`), which is kitty's internal key-*mapping* mode stack — a different mechanism that is out of scope and is mentioned here only to make that distinction explicit.

Every behavioral claim below is placed next to the **actual, unedited runtime output** that demonstrates it. Output was produced through kitty's **real** input path (raw CSI bytes → VT parser → live `Screen` → key encoder), never through a remote-control hook, debug hook, mock, or synthetic bypass. The probe that produced this output is **self-checking**: it `assert`s every flag value, byte sequence, drain list, query reply and mode side-effect it prints, so a mismatch aborts the run with a **non-zero exit** rather than emitting a wrong value. Where a fact comes from reading the source rather than from captured output it is grounded in a `file:line` citation and labeled as such; in particular the stack capacity **8** is a code fact (the array length at `kitty/screen.h:128`), not a number the specification fixes.

---

## Direct answers at a glance

| # | Question | Direct answer |
|---|----------|---------------|
| **OBJ-1** | After pushing flags on main, toggling to alternate, pushing different flags, and toggling back — which mode is active, does main's stack survive, and what bytes does a key press produce at each stage? | The mode pushed on **main survives** the round-trip and is active again at the end (`flags=1`, disambiguate). The alternate buffer has its **own independent, initially empty** stack. `Ctrl+Shift+a` emits `b'\x1b[97;6u'` in **every** intermediate state (it is not legacy-representable). |
| **OBJ-2** | What is the real push limit, what happens on overflow, and does exhausting one buffer affect the other? | The limit is exactly **8** entries per buffer (a code fact from the array size, not a documented number). Overflow does **not** error — the **oldest** entry is silently evicted (FIFO). Exhausting one buffer's stack does **not** affect the other's (verified in **both** directions, and a multi-level main stack is shown surviving a round-trip intact). |
| **OBJ-3** | Observe the exact bytes for the *same* key press under different stack states / buffers. | For `Ctrl+Shift+a` the **primary-key** encoding is `b'\x1b[97;6u'` (`ESC [ 9 7 ; 6 u`, seven bytes) in **all** stack states exercised and on **both** buffers. The bytes change only when report-alternate-keys (`0b100`) is enabled **and** the encoder is handed alternate key codepoints, which are appended as `unicode-key:shifted-key:base-layout-key` — giving `b'\x1b[97:65;6u'` (shifted only), `b'\x1b[97::98;6u'` (base-layout only, note the empty middle field), or `b'\x1b[97:65:98;6u'` (both); detailed in §5. Modifier `6` = `1 + ctrl(4) + shift(1)`; `97` = `ord('a')`. |
| **OBJ-4** | Prove the two buffers keep independent stacks; check for leakage during rapid switching. | Proven: across four rapid main↔alt cycles the values never cross (`main flags=1`, `alt flags=8` every cycle). **Zero** leakage. Independence is structural (two separate arrays; the toggle only repoints a pointer). |
| **OBJ-5** | Identify mode/setting-dependent differences and any conditions where isolation breaks down. | The per-screen isolation itself does **not** break down under any observed condition. The mode/setting-dependent *behavioral* differences observed are: (a) which DEC mode performs the switch (**1049** also saves cursor + clears alt screen; **47**/**1047** do not); (b) empty/over-pop **resets to legacy 0**; (c) flags genuinely change the encoding of legacy-representable keys (report-all-keys turns plain `a` into `b'\x1b[97u'`); (d) the query `CSI ? u` reports the current flags as `CSI ? flags u`; (e) a full-reset **RIS** (`ESC c`) returns to the main buffer and clears **both** stacks to `0` (each independently — not a cross-stack leak). Omitted set/push/pop parameters fall back to documented defaults (shown in §2 and §7). |

---

## Section 1 — Methodology (how these observations were produced)

**Environment.** Docker container `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`.

**Build (default configuration).** The native terminal-core extension `kitty/fast_data_types.so` is built from source with the canonical command:

```
python3 setup.py build --debug --ignore-compiler-warnings
```

`--ignore-compiler-warnings` is required **only** to bypass an unrelated GLFW **Wayland backend** `-Werror=switch` failure at `glfw/wl_window.c:668` (caused by newer `wayland-protocols` enum values). The terminal-core extension compiles cleanly, and **no source file is modified** by that flag. In this run the extension was already present in a warm, reusable state and imported cleanly (`kitty.fast_data_types.Screen`, `encode_key_for_tty`, `GLFW_MOD_CONTROL == 4`, `GLFW_MOD_SHIFT == 1`), so it was used as-is.

**Interpreter.** Every observation in this document was captured on **Python 3.13.7** (`/usr/bin/python3`) — the interpreter the extension is linked against (`kitty/fast_data_types.so` → `libpython3.13.so.1.0`) and the only Python interpreter present in the environment. The project declares `requires-python = ">=3.8"` (`pyproject.toml:2`), which 3.13.7 satisfies, so this is a canonical, in-support configuration. Every byte-sensitive result below is the direct, unedited output of the probe that follows; its run-to-run stability is demonstrated by the two-run SHA-256 check shown after the probe. No cross-interpreter claim is made: 3.13.7 is the sole interpreter these bytes were observed on.

**Real entry point (no bypass).** A live `Screen` is constructed through the `kitty_tests` harness. Raw CSI byte sequences are fed through the **real VT parser** with `kitty_tests.parse_bytes(screen, data)` (`kitty_tests/__init__.py:30`). The active mode is read with `screen.current_key_encoding_flags()` (backed by `screen_current_key_encoding_flags`, `kitty/screen.c:1204`). Keys are encoded with `kitty.fast_data_types.encode_key_for_tty(...)` (the Python entry `pyencode_key_for_tty`, `kitty/keys.c:311`, registered at `kitty/keys.c:334`). Bytes destined for the child process are captured from the harness accumulator `Callbacks.write` → `self.wtcbuf` (`kitty_tests/__init__.py:50-51`).

**Self-checking + provenance (fail-closed).** The probe is not print-only: it `assert`s every value it prints (45 assertions), so any divergence from the expected flag/byte/drain/query/mode result raises `AssertionError` and exits non-zero. It also **fails closed** on provenance — it refuses to run unless `KITTY_REPO` actually contains the built `kitty/fast_data_types.so`, refuses to proceed if the imported `fast_data_types` is not that exact file, and requires the base commit above to be an **ancestor of `HEAD`** (checked with `git merge-base --is-ancestor` in list form, so `KITTY_REPO` can never be interpreted as a shell command). There is **no** silent fallback to an unintended checkout.

**Stability.** Every byte-sensitive observation was run **twice** in the same process (`RUN 1` and a `RUN 2` stability re-run); the two transcripts were SHA-256-hashed and confirmed **byte-for-byte identical** (and the same digest reproduces across separate process launches). The exact invocation command, the complete `RUN 1` transcript, and both SHA-256 hashes (with a one-line command to reproduce them under `pipefail`) are shown immediately after the probe below.

**Cleanup discipline.** The observation script lived **outside** the repository, at `/tmp/kbd_stack_probe.py`, and was deleted after use. `git status --porcelain` was used to confirm the source tree is unchanged apart from this single new document (only git-ignored `build/` artifacts and the compiled `*.so` remain, and those are ignored).

**The exact probe (verbatim).** The complete, self-contained observation script is reproduced below exactly as it was run — byte-for-byte the file at `/tmp/kbd_stack_probe.py`. It touches only kitty's real input path (raw CSI bytes → `kitty_tests.parse_bytes` → live `Screen` → `encode_key_for_tty`), **asserts** every value it prints, and **fails closed** on provenance as described above. It executes `run_once()` **twice** in one process and SHA-256-hashes the two transcripts to prove run-to-run stability:

```python
#!/usr/bin/env python3
"""
kitty keyboard-protocol flag-stack investigation probe (self-validating).

Drives kitty's REAL input path (raw CSI bytes -> VT parser -> live Screen -> key
encoder) headlessly and prints byte-exact observed output for OBJ-1..OBJ-5.

Provenance (fail-closed): the repository under observation is taken from
KITTY_REPO (default: the current working directory). The script refuses to run
unless that root actually contains the built native extension
kitty/fast_data_types.so, refuses to import a fast_data_types from anywhere
else, and asserts that the intended source/base commit is an ancestor of HEAD.
There is NO silent fallback to an unintended checkout.

Self-validation (fail-loud): every expected flag value, byte sequence, drain
list, query reply and DEC-mode side effect is checked with `assert`; any
mismatch raises AssertionError and the process exits non-zero. The full
transcript is still printed. run_once() is executed TWICE in one process and the
two transcripts are SHA-256 hashed and asserted equal, proving run-to-run
stability.

Run from the repo root with:

    CI=true ASAN_OPTIONS=detect_leaks=0 KITTY_REPO="$PWD" python3 /tmp/kbd_stack_probe.py
"""
import hashlib
import os
import subprocess
import sys

# ---- Provenance guard (fail-closed; addresses fail-open KITTY_REPO) --------
EXPECTED_BASE_COMMIT = "815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1"
REPO = os.path.abspath(os.environ.get("KITTY_REPO", os.getcwd()))
_EXT = os.path.join(REPO, "kitty", "fast_data_types.so")
if not os.path.isfile(_EXT):
    sys.exit("FATAL: KITTY_REPO=%r does not contain kitty/fast_data_types.so; "
             "refusing to fall back to an unintended checkout." % REPO)
sys.path.insert(0, REPO)

import kitty.fast_data_types as fdt
from kitty.fast_data_types import Screen, encode_key_for_tty
from kitty_tests import BaseTest, Callbacks, parse_bytes

# The imported native module MUST be the one under REPO (no silent fallback).
if os.path.realpath(fdt.__file__) != os.path.realpath(_EXT):
    sys.exit("FATAL: imported kitty.fast_data_types from %r, not the intended %r"
             % (os.path.realpath(fdt.__file__), os.path.realpath(_EXT)))
# The intended source/base commit must be an ancestor of HEAD in REPO. This uses
# git in list form (no shell), so KITTY_REPO can never be interpreted as a
# command; a bad/foreign checkout fails this check and the script exits.
try:
    subprocess.run(["git", "-C", REPO, "merge-base", "--is-ancestor",
                    EXPECTED_BASE_COMMIT, "HEAD"],
                   check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
except Exception as exc:
    sys.exit("FATAL: expected source commit %s is not an ancestor of HEAD in %r (%s)"
             % (EXPECTED_BASE_COMMIT, REPO, exc))

CTRL = fdt.GLFW_MOD_CONTROL      # 4
SHIFT = fdt.GLFW_MOD_SHIFT       # 1
PRESS = fdt.GLFW_PRESS           # 1
RELEASE = fdt.GLFW_RELEASE       # 0
REPEAT = fdt.GLFW_REPEAT         # 2
A = ord('a')                     # 97
SHIFTED_A = ord('A')             # 65
BASE_B = ord('b')                # 98  (stand-in base-layout key codepoint)
FKEY_ESCAPE = fdt.GLFW_FKEY_ESCAPE   # 57344 -- the Escape KEY (not codepoint 27)


class _H(BaseTest):
    def runTest(self):
        pass


def new_screen():
    """Construct a live Screen through the real test harness (no bypass)."""
    h = _H()
    h.set_options(None)
    cb = Callbacks()
    # Screen(callbacks, lines, cols, scrollback, cell_width, cell_height, window_id, test_child)
    s = Screen(cb, 5, 5, 5, 10, 20, 0, cb)
    return s, cb


def as_bytes(text):
    # encode_key_for_tty returns a str; show the exact wire bytes.
    return text.encode('utf-8')


def enc(screen, key=A, mods=0, shifted_key=0, action=PRESS, text=None):
    """Encode a key under the CURRENT flags of the active buffer."""
    flags = screen.current_key_encoding_flags()
    out = encode_key_for_tty(
        key=key, shifted_key=shifted_key, mods=mods,
        action=action, key_encoding_flags=flags, text=text)
    return flags, as_bytes(out)


def enc_flags(key=A, mods=0, shifted_key=0, alternate_key=0, action=PRESS, flags=0, text=None):
    """Encode a key under an explicit flags value (no Screen needed)."""
    out = encode_key_for_tty(
        key=key, shifted_key=shifted_key, alternate_key=alternate_key, mods=mods,
        action=action, key_encoding_flags=flags, text=text)
    return as_bytes(out)


def screen_text(scr):
    return '|'.join(scr.line(i).as_ansi().rstrip() for i in range(scr.lines))


def run_once(o):
    """Append every deterministic observation line to list `o` AND assert it."""
    # ---- OBJ-1: round-trip main -> alternate -> main -----------------------
    o.append("=== OBJ-1: round-trip (main -> alternate -> main) ===")
    s, cb = new_screen()
    f, b = enc(s, key=A, mods=CTRL | SHIFT)
    o.append("A)  main, no flags pushed                flags=%d   bytes=%r" % (f, b))
    assert (f, b) == (0, b'\x1b[97;6u'), "OBJ-1 A"
    parse_bytes(s, b'\x1b[>1u')                 # push disambiguate (0b1) on main
    f, b = enc(s, key=A, mods=CTRL | SHIFT)
    o.append("B)  main, after CSI>1u (disambiguate)     flags=%d   bytes=%r" % (f, b))
    assert (f, b) == (1, b'\x1b[97;6u'), "OBJ-1 B"
    parse_bytes(s, b'\x1b[?1049h')              # switch to alternate screen
    f, b = enc(s, key=A, mods=CTRL | SHIFT)
    o.append("C0) alt, just switched (before push)      flags=%d   bytes=%r" % (f, b))
    assert (f, b) == (0, b'\x1b[97;6u'), "OBJ-1 C0"
    parse_bytes(s, b'\x1b[>8u')                 # push report-all-keys (0b1000) on alt
    f, b = enc(s, key=A, mods=CTRL | SHIFT)
    o.append("C)  alt, after CSI>8u (report-all)        flags=%d   bytes=%r" % (f, b))
    assert (f, b) == (8, b'\x1b[97;6u'), "OBJ-1 C"
    parse_bytes(s, b'\x1b[?1049l')              # switch back to main
    f, b = enc(s, key=A, mods=CTRL | SHIFT)
    o.append("D)  main, after switching back            flags=%d   bytes=%r   <-- main flag 1 SURVIVED" % (f, b))
    assert (f, b) == (1, b'\x1b[97;6u'), "OBJ-1 D"
    f, b = enc(s, key=A, mods=0)
    o.append("D') main, plain 'a' (no mods)             flags=%d   bytes=%r" % (f, b))
    assert (f, b) == (1, b'a'), "OBJ-1 D'"
    o.append("")

    # ---- OBJ-2: capacity / FIFO / cross-buffer isolation -------------------
    o.append("=== OBJ-2: capacity (8), FIFO eviction, cross-buffer isolation ===")

    def push_then_drain(n):
        sc, _ = new_screen()
        for v in range(1, n + 1):
            parse_bytes(sc, ('\x1b[>%du' % v).encode())
        seq = []
        for _ in range(n + 3):                  # over-drain to show the reset floor
            seq.append(sc.current_key_encoding_flags())
            parse_bytes(sc, b'\x1b[<1u')         # pop one
        return seq

    seq8 = push_then_drain(8)
    o.append("push 1..8 -> pop sequence (top first): %s" % seq8)
    assert seq8 == [8, 7, 6, 5, 4, 3, 2, 1, 0, 0, 0], "OBJ-2 push8"
    seq9 = push_then_drain(9)
    o.append("push 1..9 -> pop sequence (top first): %s" % seq9)
    assert seq9 == [9, 8, 7, 6, 5, 4, 3, 2, 0, 0, 0, 0], "OBJ-2 push9"
    # cross-buffer isolation under exhaustion of main
    sc, _ = new_screen()
    parse_bytes(sc, b'\x1b[?1049h')             # go to alt
    parse_bytes(sc, b'\x1b[>3u')                # preset alt flags = 3
    parse_bytes(sc, b'\x1b[?1049l')             # back to main
    # overflow + fully drain main
    for v in range(1, 10):
        parse_bytes(sc, ('\x1b[>%du' % v).encode())
    for _ in range(12):
        parse_bytes(sc, b'\x1b[<1u')
    main_after = sc.current_key_encoding_flags()
    parse_bytes(sc, b'\x1b[?1049h')
    alt_after = sc.current_key_encoding_flags()
    o.append("after overflow+drain main: main flags=%d  alt flags=%d  (alt preset 3 intact)" % (main_after, alt_after))
    assert (main_after, alt_after) == (0, 3), "OBJ-2 cross-buffer main-overflow"
    o.append("")

    # ---- OBJ-2 (reciprocal): overflow the ALTERNATE buffer; main intact ----
    o.append("=== OBJ-2 reciprocal: overflow ALTERNATE, main preset intact ===")
    sc, _ = new_screen()
    parse_bytes(sc, b'\x1b[>2u')                # preset main flags = 2
    parse_bytes(sc, b'\x1b[?1049h')             # go to alt
    for v in range(1, 10):                      # overflow alt (9 pushes > capacity 8)
        parse_bytes(sc, ('\x1b[>%du' % v).encode())
    for _ in range(12):                         # fully drain alt
        parse_bytes(sc, b'\x1b[<1u')
    alt_drained = sc.current_key_encoding_flags()
    parse_bytes(sc, b'\x1b[?1049l')             # back to main
    main_intact = sc.current_key_encoding_flags()
    o.append("after overflow+drain alt : alt flags=%d  main flags=%d  (main preset 2 intact)" % (alt_drained, main_intact))
    assert (alt_drained, main_intact) == (0, 2), "OBJ-2 reciprocal alt-overflow"
    o.append("")

    # ---- OBJ-2 (restoration): a MULTI-LEVEL main stack survives round-trip -
    o.append("=== OBJ-2 restoration: multi-level main stack survives round-trip ===")
    sc, _ = new_screen()
    parse_bytes(sc, b'\x1b[>1u')                # push level 1 (value 1)
    parse_bytes(sc, b'\x1b[>2u')                # push level 2 (value 2)
    parse_bytes(sc, b'\x1b[>4u')                # push level 3 (value 4)
    top_before = sc.current_key_encoding_flags()
    parse_bytes(sc, b'\x1b[?1049h')             # to alt
    parse_bytes(sc, b'\x1b[>8u')                # push 8 on alt
    parse_bytes(sc, b'\x1b[?1049l')             # back to main
    top_after = sc.current_key_encoding_flags()
    drain = []
    for _ in range(4):                          # read-top-then-pop, 3 levels + floor
        drain.append(sc.current_key_encoding_flags())
        parse_bytes(sc, b'\x1b[<1u')
    o.append("main pushed [1,2,4]; top before=%d after round-trip=%d; drain (top first): %s"
             % (top_before, top_after, drain))
    assert top_before == 4 and top_after == 4 and drain == [4, 2, 1, 0], "OBJ-2 restoration"
    o.append("")

    # ---- OBJ-3: Ctrl+Shift+a modifier arithmetic cross-check ---------------
    o.append("=== OBJ-3: Ctrl+Shift+a exact bytes + modifier arithmetic ===")
    m = 0
    if SHIFT:
        m |= 1
    if CTRL:
        m |= 4
    o.append("ord('a') = %d" % A)
    o.append("GLFW_MOD_SHIFT = %d (csi maps ->1)" % SHIFT)
    o.append("GLFW_MOD_CONTROL = %d (csi maps ->4)" % CTRL)
    o.append("computed m = %d -> ;{m+1} = ;%d" % (m, m + 1))
    got = enc_flags(key=A, mods=CTRL | SHIFT, flags=0)
    expected = ("\x1b[%d;%du" % (A, m + 1)).encode()
    o.append("encode_key_for_tty(Ctrl+Shift+a, flags=0) = %r" % got)
    o.append("expected CSI form                          = %r" % expected)
    o.append("MATCH: %s" % (got == expected))
    assert m == 5, "OBJ-3 modifier math"
    assert got == expected, "OBJ-3 computed-vs-observed"
    assert got == b'\x1b[97;6u', "OBJ-3 literal bytes"
    o.append("")

    # ---- OBJ-4: rapid switching (independence, no leakage) -----------------
    o.append("=== OBJ-4: rapid switching (independence / no leakage) ===")
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[>1u')                 # main flags = 1
    parse_bytes(s, b'\x1b[?1049h')
    parse_bytes(s, b'\x1b[>8u')                 # alt flags = 8
    parse_bytes(s, b'\x1b[?1049l')
    for cyc in range(4):
        parse_bytes(s, b'\x1b[?1049h')
        alt = s.current_key_encoding_flags()
        parse_bytes(s, b'\x1b[?1049l')
        main = s.current_key_encoding_flags()
        o.append("rapid switch cycle %d: main flags=%d  alt flags=%d" % (cyc, main, alt))
        assert (main, alt) == (1, 8), "OBJ-4 cycle %d" % cyc
    o.append("")

    # ---- OBJ-5(a): DEC mode 47 vs 1047 vs 1049 -----------------------------
    o.append("=== OBJ-5(a): DEC 47 vs 1047 vs 1049 (stack + cursor + screen-clear) ===")
    for mode in (47, 1047, 1049):
        s, cb = new_screen()
        parse_bytes(s, b'\x1b[4;5H')            # CUP row4 col5 -> cursor (x=4,y=3)
        parse_bytes(s, b'\x1b[>1u')             # push disambiguate (1) on main
        main_before = s.current_key_encoding_flags()
        cur_before = (s.cursor.x, s.cursor.y)
        parse_bytes(s, ('\x1b[?%dh' % mode).encode())   # ENTER alt
        alt_during = s.current_key_encoding_flags()
        cur_alt_entry = (s.cursor.x, s.cursor.y)
        parse_bytes(s, b'\x1b[>8u')             # push report-all (8) on alt
        parse_bytes(s, b'ZZ')                   # write marker on alt screen
        alt_txt_during = screen_text(s)
        parse_bytes(s, ('\x1b[?%dl' % mode).encode())   # BACK to main
        main_after = s.current_key_encoding_flags()
        cur_main_after = (s.cursor.x, s.cursor.y)
        parse_bytes(s, ('\x1b[?%dh' % mode).encode())   # RE-ENTER alt
        alt_reentry = s.current_key_encoding_flags()
        alt_txt_reentry = screen_text(s)
        restored = cur_main_after == cur_before
        cleared = alt_txt_reentry.strip('|') == ''
        o.append("mode %-4d | stack: main_before=%d alt_during=%d main_after=%d alt_reentry=%d"
                 % (mode, main_before, alt_during, main_after, alt_reentry))
        o.append("           cursor: main_before=%s alt_entry=%s main_after=%s%s"
                 % (cur_before, cur_alt_entry, cur_main_after,
                    "  (RESTORED)" if restored else "  (NOT restored)"))
        o.append("           alt screen: during=%r reentry=%r%s"
                 % (alt_txt_during, alt_txt_reentry,
                    "  (CLEARED on re-entry)" if cleared else "  (NOT cleared)"))
        # stack isolation identical for all three modes:
        assert (main_before, alt_during, main_after, alt_reentry) == (1, 0, 1, 8), "OBJ-5a stack mode %d" % mode
        # only 1049 saves the cursor and clears the alt screen:
        assert restored == (mode == 1049), "OBJ-5a cursor mode %d" % mode
        assert cleared == (mode == 1049), "OBJ-5a clear mode %d" % mode
    o.append("")

    # ---- OBJ-5(b): empty / over-pop reset + query --------------------------
    o.append("=== OBJ-5(b): empty/over-pop reset + query response ===")
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[>5u')                 # push 5
    o.append("after push 5: flags=%d" % s.current_key_encoding_flags())
    assert s.current_key_encoding_flags() == 5, "OBJ-5b push5"
    cb.wtcbuf = b''
    parse_bytes(s, b'\x1b[?u')                  # query
    o.append("query CSI ?u response to child: %r" % bytes(cb.wtcbuf))
    assert bytes(cb.wtcbuf) == b'\x1b[?5u', "OBJ-5b query5"
    parse_bytes(s, b'\x1b[<3u')                 # pop 3 (> pushed) -> reset
    o.append("after pop 3 (>pushed): flags=%d (reset)" % s.current_key_encoding_flags())
    assert s.current_key_encoding_flags() == 0, "OBJ-5b overpop reset"
    cb.wtcbuf = b''
    parse_bytes(s, b'\x1b[?u')
    o.append("query after reset:               %r" % bytes(cb.wtcbuf))
    assert bytes(cb.wtcbuf) == b'\x1b[?0u', "OBJ-5b query0"
    o.append("")

    # ---- omitted set/push/pop parameters (defaults) via the REAL parser ----
    o.append("=== omitted parameters: set / push / pop defaults via parse_bytes ===")
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[=5;1u')               # set flags=5 (replace)
    before_set_default = s.current_key_encoding_flags()
    parse_bytes(s, b'\x1b[=u')                  # SET with omitted flags AND mode
    after_set_default = s.current_key_encoding_flags()
    o.append("CSI=5;1u then CSI=u (omit flags+mode) : %d -> %d (replace w/ default flags 0)"
             % (before_set_default, after_set_default))
    assert (before_set_default, after_set_default) == (5, 0), "omit set"
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[>u')                  # PUSH with omitted flags
    after_push_default = s.current_key_encoding_flags()
    parse_bytes(s, b'\x1b[>7u')                 # push 7 so pop has something to reveal
    parse_bytes(s, b'\x1b[<u')                  # POP with omitted count (default 1)
    after_pop_default = s.current_key_encoding_flags()
    o.append("CSI>u (omit push flags) -> %d ; push 7 then CSI<u (omit pop count=1) -> %d"
             % (after_push_default, after_pop_default))
    assert (after_push_default, after_pop_default) == (0, 0), "omit push/pop"
    o.append("")

    # ---- OBJ-5(c): plain 'a' under flags 0/1/8/16 --------------------------
    o.append("=== OBJ-5(c): plain 'a' under flags 0/1/8/16 (flags alter encoding) ===")
    r0 = enc_flags(key=A, mods=0, flags=0)
    r1 = enc_flags(key=A, mods=0, flags=1)
    r8 = enc_flags(key=A, mods=0, flags=8)
    r16 = enc_flags(key=A, mods=0, flags=16)
    o.append("plain 'a' under flags=0  (legacy)          : %r" % r0)
    o.append("plain 'a' under flags=1  (disambiguate)    : %r" % r1)
    o.append("plain 'a' under flags=8  (report-all-keys) : %r" % r8)
    o.append("plain 'a' under flags=16 (report-text)     : %r" % r16)
    assert (r0, r1, r8, r16) == (b'a', b'a', b'\x1b[97u', b'a'), "OBJ-5c plain-a flags"
    o.append("")

    # ---- bit-1 disambiguate: FUNCTIONAL evidence on the Escape KEY ----------
    o.append("=== bit-1 (disambiguate) functional evidence — the Escape KEY ===")
    esc0 = enc_flags(key=FKEY_ESCAPE, mods=0, flags=0)
    esc1 = enc_flags(key=FKEY_ESCAPE, mods=0, flags=1)
    o.append("Escape key under flags=0  (legacy)         : %r" % esc0)
    o.append("Escape key under flags=1  (disambiguate)   : %r" % esc1)
    assert (esc0, esc1) == (b'\x1b', b'\x1b[27u'), "bit-1 Escape"
    o.append("")

    # ---- bit-2 event types --------------------------------------------------
    o.append("=== bit-2 (report event types) on plain 'a': press/repeat/release ===")
    exp_bit2 = {
        (0, PRESS): b'a', (0, REPEAT): b'a', (0, RELEASE): b'',
        (2, PRESS): b'a', (2, REPEAT): b'\x1b[97;1:2u', (2, RELEASE): b'\x1b[97;1:3u',
    }
    for fl in (0, 2):
        rp = enc_flags(key=A, mods=0, action=PRESS, flags=fl)
        rr = enc_flags(key=A, mods=0, action=REPEAT, flags=fl)
        rl = enc_flags(key=A, mods=0, action=RELEASE, flags=fl)
        o.append("flags=%-2d press  : %r" % (fl, rp))
        o.append("flags=%-2d repeat : %r" % (fl, rr))
        o.append("flags=%-2d release: %r" % (fl, rl))
        assert rp == exp_bit2[(fl, PRESS)], "bit-2 press fl=%d" % fl
        assert rr == exp_bit2[(fl, REPEAT)], "bit-2 repeat fl=%d" % fl
        assert rl == exp_bit2[(fl, RELEASE)], "bit-2 release fl=%d" % fl
    o.append("")

    # ---- bit-4 report alternate keys: shifted AND base-layout subfields -----
    o.append("=== bit-4 (report alternate keys) — Ctrl+Shift+a with shifted_key=65 ===")
    b_none = enc_flags(key=A, shifted_key=0, mods=CTRL | SHIFT, flags=0)
    b_bit4_none = enc_flags(key=A, shifted_key=0, mods=CTRL | SHIFT, flags=4)
    b_shift = enc_flags(key=A, shifted_key=SHIFTED_A, mods=CTRL | SHIFT, flags=4)
    b_base = enc_flags(key=A, alternate_key=BASE_B, mods=CTRL | SHIFT, flags=4)
    b_both = enc_flags(key=A, shifted_key=SHIFTED_A, alternate_key=BASE_B, mods=CTRL | SHIFT, flags=4)
    o.append("flags=0 (no bit4), no shifted_key      : %r" % b_none)
    o.append("flags=4 (bit4), no shifted_key         : %r" % b_bit4_none)
    o.append("flags=4 (bit4), shifted_key=65 ('A')   : %r" % b_shift)
    o.append("flags=4 (bit4), alternate_key=98 ('b') : %r" % b_base)
    o.append("flags=4 (bit4), shifted=65 + alt=98    : %r" % b_both)
    assert b_none == b'\x1b[97;6u', "bit-4 none"
    assert b_bit4_none == b'\x1b[97;6u', "bit-4 no-subfields"
    assert b_shift == b'\x1b[97:65;6u', "bit-4 shifted"
    assert b_base == b'\x1b[97::98;6u', "bit-4 base-layout"
    assert b_both == b'\x1b[97:65:98;6u', "bit-4 both"
    o.append("")

    # ---- bit-16 report associated text (with real text; needs report-all) --
    o.append("=== bit-16 (report associated text) — needs report-all (bit 8), real text ===")
    t16_alone = enc_flags(key=A, mods=0, flags=16, text='a')
    t8_none = enc_flags(key=A, mods=0, flags=8, text=None)
    t24_a = enc_flags(key=A, mods=0, flags=24, text='a')
    t24_A = enc_flags(key=A, mods=0, flags=24, text='A')
    t24_e = enc_flags(key=A, mods=0, flags=24, text='\u00e9')
    t24_ab = enc_flags(key=A, mods=0, flags=24, text='ab')
    o.append("flags=16 alone,     text='a'           : %r" % t16_alone)
    o.append("flags=8  report-all,text=None          : %r" % t8_none)
    o.append("flags=24 (8|16),    text='a'  (ASCII)  : %r" % t24_a)
    o.append("flags=24 (8|16),    text='A'           : %r" % t24_A)
    o.append("flags=24 (8|16),    text='\\u00e9' (non-ASCII 233): %r" % t24_e)
    o.append("flags=24 (8|16),    text='ab' (multi)  : %r" % t24_ab)
    assert t16_alone == b'a', "bit-16 alone stays literal"
    assert t8_none == b'\x1b[97u', "bit-16 report-all no text"
    assert t24_a == b'\x1b[97;;97u', "bit-16 ascii a"
    assert t24_A == b'\x1b[97;;65u', "bit-16 A"
    assert t24_e == b'\x1b[97;;233u', "bit-16 non-ascii"
    assert t24_ab == b'\x1b[97;;97:98u', "bit-16 multi"
    o.append("")

    # ---- all five flags combined (one concrete documented input) -----------
    o.append("=== all five flags (31) combined: Ctrl+Shift+a, shifted=65, alt=98, text=é, action=repeat ===")
    full = enc_flags(key=A, shifted_key=SHIFTED_A, alternate_key=BASE_B,
                     mods=CTRL | SHIFT, action=REPEAT, flags=31, text='\u00e9')
    o.append("flags=31 full combination              : %r" % full)
    assert full == b'\x1b[97:65:98;6:2;233u', "all-five combined"
    o.append("")

    # ---- CSI = set modes 1/2/3/default through the REAL parser --------------
    o.append("=== CSI = set modes 1(replace) / 2(OR) / 3(AND-NOT) / default via parse_bytes ===")
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[=5;1u')
    v1 = s.current_key_encoding_flags()
    o.append("CSI=5;1u (replace)     -> flags=%d" % v1)
    parse_bytes(s, b'\x1b[=2;1u')
    v2 = s.current_key_encoding_flags()
    o.append("CSI=2;1u (replace)     -> flags=%d" % v2)
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[=5u')
    v3 = s.current_key_encoding_flags()
    o.append("CSI=5u   (default=1)   -> flags=%d" % v3)
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[=1;1u')
    v4 = s.current_key_encoding_flags()
    o.append("CSI=1;1u (base)        -> flags=%d" % v4)
    parse_bytes(s, b'\x1b[=8;2u')
    v5 = s.current_key_encoding_flags()
    o.append("CSI=8;2u (OR onto 1)   -> flags=%d" % v5)
    parse_bytes(s, b'\x1b[=1;3u')
    v6 = s.current_key_encoding_flags()
    o.append("CSI=1;3u (AND-NOT 1)   -> flags=%d" % v6)
    parse_bytes(s, b'\x1b[=8;3u')
    v7 = s.current_key_encoding_flags()
    o.append("CSI=8;3u (AND-NOT 8)   -> flags=%d" % v7)
    assert [v1, v2, v3, v4, v5, v6, v7] == [5, 2, 5, 1, 9, 8, 0], "set-mode arithmetic"
    o.append("")

    # ---- RIS (ESC c) full reset: returns to main, clears BOTH stacks -------
    o.append("=== RIS (ESC c) full reset: returns to main, clears BOTH stacks to 0 ===")
    s, cb = new_screen()
    parse_bytes(s, b'\x1b[>1u')                 # main flags = 1
    parse_bytes(s, b'\x1b[?1049h')              # to alt
    parse_bytes(s, b'\x1b[>8u')                 # alt flags = 8
    on_alt = s.current_key_encoding_flags()
    parse_bytes(s, b'\x1bc')                    # RIS
    after_ris_current = s.current_key_encoding_flags()
    parse_bytes(s, b'\x1b[?1049h')              # re-check alt
    alt_after_ris = s.current_key_encoding_flags()
    parse_bytes(s, b'\x1b[?1049l')              # back to main
    main_after_ris = s.current_key_encoding_flags()
    o.append("before RIS (on alt): alt flags=%d" % on_alt)
    o.append("after RIS: current=%d  main=%d  alt=%d  (both cleared, active buffer=main)"
             % (after_ris_current, main_after_ris, alt_after_ris))
    assert on_alt == 8, "RIS pre"
    assert (after_ris_current, main_after_ris, alt_after_ris) == (0, 0, 0), "RIS reset both stacks"


def main():
    buf1, buf2 = [], []
    run_once(buf1)
    run_once(buf2)
    text1 = "\n".join(buf1) + "\n"
    text2 = "\n".join(buf2) + "\n"
    h1 = hashlib.sha256(text1.encode()).hexdigest()
    h2 = hashlib.sha256(text2.encode()).hexdigest()

    print("################  RUN 1 (scenario transcript)  ################")
    print(text1, end="")
    print("################  RUN 2 (stability re-run)  ################")
    print(text2, end="")
    print("################  STABILITY  ################")
    print("RUN 1 SHA-256: %s" % h1)
    print("RUN 2 SHA-256: %s" % h2)
    print("IDENTICAL: %s" % (h1 == h2))
    # Fail loudly if the two unchanged runs ever diverge.
    assert h1 == h2, "STABILITY: RUN 1 and RUN 2 transcripts differ"


if __name__ == "__main__":
    main()
```

**Invocation (exact command).** Run from the repository root (`KITTY_REPO` is set explicitly so the provenance guard binds to *this* checkout rather than silently defaulting):

```
CI=true ASAN_OPTIONS=detect_leaks=0 KITTY_REPO="$PWD" python3 /tmp/kbd_stack_probe.py
```

**Complete captured output — the `RUN 1` transcript (byte-for-byte identical to `RUN 2`).** This is the full, unedited transcript printed between the `RUN 1` and `RUN 2` banners; every later section quotes its relevant excerpt from this transcript verbatim:

```text
=== OBJ-1: round-trip (main -> alternate -> main) ===
A)  main, no flags pushed                flags=0   bytes=b'\x1b[97;6u'
B)  main, after CSI>1u (disambiguate)     flags=1   bytes=b'\x1b[97;6u'
C0) alt, just switched (before push)      flags=0   bytes=b'\x1b[97;6u'
C)  alt, after CSI>8u (report-all)        flags=8   bytes=b'\x1b[97;6u'
D)  main, after switching back            flags=1   bytes=b'\x1b[97;6u'   <-- main flag 1 SURVIVED
D') main, plain 'a' (no mods)             flags=1   bytes=b'a'

=== OBJ-2: capacity (8), FIFO eviction, cross-buffer isolation ===
push 1..8 -> pop sequence (top first): [8, 7, 6, 5, 4, 3, 2, 1, 0, 0, 0]
push 1..9 -> pop sequence (top first): [9, 8, 7, 6, 5, 4, 3, 2, 0, 0, 0, 0]
after overflow+drain main: main flags=0  alt flags=3  (alt preset 3 intact)

=== OBJ-2 reciprocal: overflow ALTERNATE, main preset intact ===
after overflow+drain alt : alt flags=0  main flags=2  (main preset 2 intact)

=== OBJ-2 restoration: multi-level main stack survives round-trip ===
main pushed [1,2,4]; top before=4 after round-trip=4; drain (top first): [4, 2, 1, 0]

=== OBJ-3: Ctrl+Shift+a exact bytes + modifier arithmetic ===
ord('a') = 97
GLFW_MOD_SHIFT = 1 (csi maps ->1)
GLFW_MOD_CONTROL = 4 (csi maps ->4)
computed m = 5 -> ;{m+1} = ;6
encode_key_for_tty(Ctrl+Shift+a, flags=0) = b'\x1b[97;6u'
expected CSI form                          = b'\x1b[97;6u'
MATCH: True

=== OBJ-4: rapid switching (independence / no leakage) ===
rapid switch cycle 0: main flags=1  alt flags=8
rapid switch cycle 1: main flags=1  alt flags=8
rapid switch cycle 2: main flags=1  alt flags=8
rapid switch cycle 3: main flags=1  alt flags=8

=== OBJ-5(a): DEC 47 vs 1047 vs 1049 (stack + cursor + screen-clear) ===
mode 47   | stack: main_before=1 alt_during=0 main_after=1 alt_reentry=8
           cursor: main_before=(4, 3) alt_entry=(0, 0) main_after=(2, 0)  (NOT restored)
           alt screen: during='ZZ||||' reentry='ZZ||||'  (NOT cleared)
mode 1047 | stack: main_before=1 alt_during=0 main_after=1 alt_reentry=8
           cursor: main_before=(4, 3) alt_entry=(0, 0) main_after=(2, 0)  (NOT restored)
           alt screen: during='ZZ||||' reentry='ZZ||||'  (NOT cleared)
mode 1049 | stack: main_before=1 alt_during=0 main_after=1 alt_reentry=8
           cursor: main_before=(4, 3) alt_entry=(0, 0) main_after=(4, 3)  (RESTORED)
           alt screen: during='ZZ||||' reentry='||||'  (CLEARED on re-entry)

=== OBJ-5(b): empty/over-pop reset + query response ===
after push 5: flags=5
query CSI ?u response to child: b'\x1b[?5u'
after pop 3 (>pushed): flags=0 (reset)
query after reset:               b'\x1b[?0u'

=== omitted parameters: set / push / pop defaults via parse_bytes ===
CSI=5;1u then CSI=u (omit flags+mode) : 5 -> 0 (replace w/ default flags 0)
CSI>u (omit push flags) -> 0 ; push 7 then CSI<u (omit pop count=1) -> 0

=== OBJ-5(c): plain 'a' under flags 0/1/8/16 (flags alter encoding) ===
plain 'a' under flags=0  (legacy)          : b'a'
plain 'a' under flags=1  (disambiguate)    : b'a'
plain 'a' under flags=8  (report-all-keys) : b'\x1b[97u'
plain 'a' under flags=16 (report-text)     : b'a'

=== bit-1 (disambiguate) functional evidence — the Escape KEY ===
Escape key under flags=0  (legacy)         : b'\x1b'
Escape key under flags=1  (disambiguate)   : b'\x1b[27u'

=== bit-2 (report event types) on plain 'a': press/repeat/release ===
flags=0  press  : b'a'
flags=0  repeat : b'a'
flags=0  release: b''
flags=2  press  : b'a'
flags=2  repeat : b'\x1b[97;1:2u'
flags=2  release: b'\x1b[97;1:3u'

=== bit-4 (report alternate keys) — Ctrl+Shift+a with shifted_key=65 ===
flags=0 (no bit4), no shifted_key      : b'\x1b[97;6u'
flags=4 (bit4), no shifted_key         : b'\x1b[97;6u'
flags=4 (bit4), shifted_key=65 ('A')   : b'\x1b[97:65;6u'
flags=4 (bit4), alternate_key=98 ('b') : b'\x1b[97::98;6u'
flags=4 (bit4), shifted=65 + alt=98    : b'\x1b[97:65:98;6u'

=== bit-16 (report associated text) — needs report-all (bit 8), real text ===
flags=16 alone,     text='a'           : b'a'
flags=8  report-all,text=None          : b'\x1b[97u'
flags=24 (8|16),    text='a'  (ASCII)  : b'\x1b[97;;97u'
flags=24 (8|16),    text='A'           : b'\x1b[97;;65u'
flags=24 (8|16),    text='\u00e9' (non-ASCII 233): b'\x1b[97;;233u'
flags=24 (8|16),    text='ab' (multi)  : b'\x1b[97;;97:98u'

=== all five flags (31) combined: Ctrl+Shift+a, shifted=65, alt=98, text=é, action=repeat ===
flags=31 full combination              : b'\x1b[97:65:98;6:2;233u'

=== CSI = set modes 1(replace) / 2(OR) / 3(AND-NOT) / default via parse_bytes ===
CSI=5;1u (replace)     -> flags=5
CSI=2;1u (replace)     -> flags=2
CSI=5u   (default=1)   -> flags=5
CSI=1;1u (base)        -> flags=1
CSI=8;2u (OR onto 1)   -> flags=9
CSI=1;3u (AND-NOT 1)   -> flags=8
CSI=8;3u (AND-NOT 8)   -> flags=0

=== RIS (ESC c) full reset: returns to main, clears BOTH stacks to 0 ===
before RIS (on alt): alt flags=8
after RIS: current=0  main=0  alt=0  (both cleared, active buffer=main)
```

**Two-run stability (SHA-256).** After the two transcripts, the probe prints their SHA-256 digests and their equality:

```text
RUN 1 SHA-256: d8e2bf220ba812e151c81a9eb8cb0f9865e68884a429dc1f6c604e1d31c4a610
RUN 2 SHA-256: d8e2bf220ba812e151c81a9eb8cb0f9865e68884a429dc1f6c604e1d31c4a610
IDENTICAL: True
```

Reproduce the two hashes independently with (note `bash -o pipefail`, so a probe failure is **not** masked by the trailing `sed`):

```
bash -o pipefail -c 'CI=true ASAN_OPTIONS=detect_leaks=0 KITTY_REPO="$PWD" python3 /tmp/kbd_stack_probe.py | sed -n "/RUN 1 SHA-256/,\$p"'
```

Without `pipefail` the pipeline would report the exit status of `sed` (`0`) even if the probe aborted on a failed assertion or provenance check; with `pipefail` any probe failure propagates as a non-zero pipeline exit.

---

## Section 2 — Flag-bit semantics and escape-code vocabulary (reference)

The keyboard protocol's *progressive enhancement* is a set of bit-flags. kitty documents five of them in `docs/keyboard-protocol.rst:278-282`:

| Bit | Value | Meaning | Spec |
|-----|-------|---------|------|
| `0b1` | 1 | disambiguate escape codes | `docs/keyboard-protocol.rst:278`, detail `:319-327` |
| `0b10` | 2 | report event types | `docs/keyboard-protocol.rst:279`, detail `:353-356` |
| `0b100` | 4 | report alternate keys | `docs/keyboard-protocol.rst:280`, detail `:370-372` |
| `0b1000` | 8 | report all keys as escape codes | `docs/keyboard-protocol.rst:281`, detail `:384-387` |
| `0b10000` | 16 | report associated text | `docs/keyboard-protocol.rst:282`, detail `:398-400` |

The escape-code vocabulary that manipulates the flags and the per-screen stack (all end in the final byte `u`):

| Operation | Escape code | Spec | Handler |
|-----------|-------------|------|---------|
| **set** current flags | `CSI = flags ; mode u` — `mode` 1 = replace, 2 = OR (set bits), 3 = AND-NOT (reset bits) | `docs/keyboard-protocol.rst:266,271-273` | `screen_set_key_encoding_flags` (`kitty/screen.c:1220`) |
| **query** current flags | `CSI ? u` → terminal replies `CSI ? flags u` | `docs/keyboard-protocol.rst:287,291` | `screen_report_key_encoding_flags` (`kitty/screen.c:1212`) |
| **push** onto the stack | `CSI > flags u` (flags default to 0 if omitted) | `docs/keyboard-protocol.rst:296` | `screen_push_key_encoding_flags` (`kitty/screen.c:1234`) |
| **pop** off the stack | `CSI < number u` (number defaults to 1) | `docs/keyboard-protocol.rst:297` | `screen_pop_key_encoding_flags` (`kitty/screen.c:1248`) |

The specification additionally mandates the stack-size limit, separate per-screen stacks, the empty-pop reset, and the oldest-first eviction policy at `docs/keyboard-protocol.rst:299-303`, with the design rationale for independent stacks at `docs/keyboard-protocol.rst:305-312`. The full CSI-`u` key report format — `CSI unicode-key-code:alternate-key-codes ; modifiers:event-type ; text-as-codepoints u` — is defined at `docs/keyboard-protocol.rst:118`, and the alternate-key sub-field structure (the shifted key and the base-layout key, joined to the primary key by colons) at `docs/keyboard-protocol.rst:139-159`.

**Observed effect of each flag bit (real parser + encoder).** These bits are not merely documented — **each of the five was exercised at runtime with functional evidence** below: bit `0b1` on the Escape *key*, bit `0b10` via repeat/release events, bit `0b100` via alternate-key sub-fields, bit `0b1000` by forcing a plain `a` into an escape code, and bit `0b10000` by attaching real associated text. `Ctrl+Shift+a` is not legacy-representable, so it isolates the *modifier / alternate-key* machinery; a plain `a` *is* legacy-representable, so it isolates the *disambiguate / report-all / report-text* machinery.

- **`0b1` disambiguate** and **`0b10000` report-associated-text** leave a legacy-representable key literal, while **`0b1000` report-all-keys** forces it into an escape code (full discussion in OBJ-5(c)):

```text
plain 'a' under flags=0  (legacy)          : b'a'
plain 'a' under flags=1  (disambiguate)    : b'a'
plain 'a' under flags=8  (report-all-keys) : b'\x1b[97u'
plain 'a' under flags=16 (report-text)     : b'a'
```

- **`0b1` disambiguate — functional evidence on a key it actually changes.** A plain `a` is unchanged by disambiguate, so to *show* the bit working it must be exercised on a key whose legacy encoding is ambiguous. The **Escape key** (`GLFW_FKEY_ESCAPE = 57344`, the key — not the codepoint `27`) is the canonical case: under legacy flags it emits the bare byte `b'\x1b'`, but under disambiguate it becomes the unambiguous `CSI 27 u` (`docs/keyboard-protocol.rst:319-327`):

```text
Escape key under flags=0  (legacy)         : b'\x1b'
Escape key under flags=1  (disambiguate)   : b'\x1b[27u'
```

- **`0b10` report event types** adds a per-event sub-parameter (`:1` press [default], `:2` repeat, `:3` release). With only this bit set, a *press* of a legacy key stays literal, but *repeat* and *release* — which legacy encoding cannot express — become CSI-`u` events:

```text
flags=0  press  : b'a'
flags=0  repeat : b'a'
flags=0  release: b''
flags=2  press  : b'a'
flags=2  repeat : b'\x1b[97;1:2u'
flags=2  release: b'\x1b[97;1:3u'
```

- **`0b100` report alternate keys** appends the alternate key codepoints to the primary key, joined by colons as `unicode-key:shifted-key:base-layout-key` (`docs/keyboard-protocol.rst:139-159`). Two independent sub-fields exist — the **shifted** key and the **base-layout** key — and either, both, or neither may be present. With `Ctrl+Shift+a`, supplying `shifted_key = ord('A') = 65` adds `:65`; supplying only a base-layout key (`ord('b') = 98`) yields the **double-colon** form `97::98` (empty shifted field, `docs/keyboard-protocol.rst:155-159`); supplying both yields `97:65:98`:

```text
flags=0 (no bit4), no shifted_key      : b'\x1b[97;6u'
flags=4 (bit4), no shifted_key         : b'\x1b[97;6u'
flags=4 (bit4), shifted_key=65 ('A')   : b'\x1b[97:65;6u'
flags=4 (bit4), alternate_key=98 ('b') : b'\x1b[97::98;6u'
flags=4 (bit4), shifted=65 + alt=98    : b'\x1b[97:65:98;6u'
```

- **`0b10000` report associated text** attaches the text a key press generates as a third `;`-separated group of codepoints (`docs/keyboard-protocol.rst:398-400`). Associated text is only emitted when the event is *also* reported as an escape code, so the bit is exercised together with report-all-keys (`0b1000`); `flags=24` is `8 | 16`. The text field carries real codepoints — ASCII, upper-case, non-ASCII (`é` = 233), and multi-character text (joined by `:`):

```text
flags=16 alone,     text='a'           : b'a'
flags=8  report-all,text=None          : b'\x1b[97u'
flags=24 (8|16),    text='a'  (ASCII)  : b'\x1b[97;;97u'
flags=24 (8|16),    text='A'           : b'\x1b[97;;65u'
flags=24 (8|16),    text='\u00e9' (non-ASCII 233): b'\x1b[97;;233u'
flags=24 (8|16),    text='ab' (multi)  : b'\x1b[97;;97:98u'
```

Note the **empty middle group** in `b'\x1b[97;;97u'`: the modifier group is omitted (no modifiers), leaving `unicode-key ; ; text` — exactly the `CSI unicode-key-code ; ; text-as-codepoints u` layout of `docs/keyboard-protocol.rst:118`.

- **All five flags at once.** Combining every sub-field in a single encode — `Ctrl+Shift+a`, `shifted_key=65`, `alternate_key=98`, `flags=31` (`1|2|4|8|16`), `text='é'`, `action=repeat` — produces the fully-populated report `unicode:shifted:base ; modifiers:event-type ; text`:

```text
flags=31 full combination              : b'\x1b[97:65:98;6:2;233u'
```

That is `97:65:98` (key + shifted + base-layout), `;6:2` (modifier `6` + event-type `2` = repeat), `;233` (the associated text `é`), matching the full format at `docs/keyboard-protocol.rst:118`.

**Observed set-mode arithmetic (`CSI = flags ; mode u`, through the real parser).** Feeding the set sequences through `parse_bytes` confirms `screen_set_key_encoding_flags` (`kitty/screen.c:1220`): mode `1` = replace, mode `2` = OR (set bits), mode `3` = AND-NOT (reset bits), and an omitted mode defaults to replace:

```text
CSI=5;1u (replace)     -> flags=5
CSI=2;1u (replace)     -> flags=2
CSI=5u   (default=1)   -> flags=5
CSI=1;1u (base)        -> flags=1
CSI=8;2u (OR onto 1)   -> flags=9
CSI=1;3u (AND-NOT 1)   -> flags=8
CSI=8;3u (AND-NOT 8)   -> flags=0
```

**Observed default handling of omitted parameters (through the real parser).** The push/pop/set operations all have documented defaults, exercised here by feeding parameter-less sequences through `parse_bytes`: `CSI = u` sets with default flags `0` (replace, so it clears), `CSI > u` pushes flags `0`, and `CSI < u` pops the default count `1`:

```text
CSI=5;1u then CSI=u (omit flags+mode) : 5 -> 0 (replace w/ default flags 0)
CSI>u (omit push flags) -> 0 ; push 7 then CSI<u (omit pop count=1) -> 0
```

These defaults are applied in the CSI-`u` dispatch (`kitty/vt-parser.c:1229,1233,1237`) — set `how=1`, push flags `0`, pop number `1` — and match `docs/keyboard-protocol.rst:296-297`.

---

## Section 3 — OBJ-1: Round-trip state survival (main → alternate → main)

**Direct answer.** The disambiguate flag pushed on the **main** buffer (`flags=1`) **survives** the full main → alternate → main round-trip and is the active mode again at the end (stage **D**). The **alternate** buffer begins with its **own independent, empty** stack (stage **C0**, `flags=0`); the report-all-keys flag pushed there (`flags=8`) never touches main. For the key press itself, `Ctrl+Shift+a` produces the **identical** bytes `b'\x1b[97;6u'` in every intermediate state (the reason is explained in OBJ-3), while a plain `a` on return reflects main's surviving `flags=1`.

**The mandatory controlled-test sequence.** The user's example maps verbatim onto these escape sequences:

> *"press Ctrl+Shift+a while on main with NO flags pushed → again after pushing disambiguate mode → again after switching to alternate and pushing report-all-keys mode → back to main."*

- `CSI > 1 u` (`b'\x1b[>1u'`) — push **disambiguate** (`0b1`) on main
- `CSI ? 1049 h` (`b'\x1b[?1049h'`) — switch to the **alternate** screen
- `CSI > 8 u` (`b'\x1b[>8u'`) — push **report-all-keys** (`0b1000`) on alternate
- `CSI ? 1049 l` (`b'\x1b[?1049l'`) — switch **back** to main

**Observed output (unedited):**

```text
A)  main, no flags pushed                flags=0   bytes=b'\x1b[97;6u'
B)  main, after CSI>1u (disambiguate)     flags=1   bytes=b'\x1b[97;6u'
C0) alt, just switched (before push)      flags=0   bytes=b'\x1b[97;6u'
C)  alt, after CSI>8u (report-all)        flags=8   bytes=b'\x1b[97;6u'
D)  main, after switching back            flags=1   bytes=b'\x1b[97;6u'   <-- main flag 1 SURVIVED
D') main, plain 'a' (no mods)             flags=1   bytes=b'a'
```

Reading the `flags=` column stage by stage answers *"which keyboard encoding mode is active"* at each point: `0` (legacy) → `1` (disambiguate, main) → `0` (legacy, alt just switched) → `8` (report-all-keys, alt) → **`1` again** (disambiguate, main restored). The `bytes=` column is *"what escape sequences a key press produces in each intermediate state"*.

**Cause → effect.** The buffer toggle is `screen_toggle_screen_buffer` (`kitty/screen.c:1068`). Switching to the alternate screen merely **repoints** the active pointer:

- to the alternate array — `self->key_encoding_flags = self->alt_key_encoding_flags;` (`kitty/screen.c:1079`)
- back to the main array — `self->key_encoding_flags = self->main_key_encoding_flags;` (`kitty/screen.c:1086`)

It never copies or clears either array. Because main's array is left completely untouched while the terminal is on the alternate screen, main's `flags=1` is exactly where it was when the pointer swings back — hence stage **D** reads `1`, and stage **D'** (plain `a` with no modifiers) confirms main's surviving mode is in effect. The initial value of the pointer is main (`self->key_encoding_flags = self->main_key_encoding_flags;`, `kitty/screen.c:150`), which is why stage **A** starts at `flags=0` on main with an empty stack.

---

## Section 4 — OBJ-2: Stack exhaustion, the real limit, and cross-buffer isolation under exhaustion

**Direct answer.** The push limit is exactly **8** entries per buffer. Pushing more than 8 does **not** raise an error and is **not** rejected — the **oldest** entry is silently **evicted** (FIFO). Exhausting one buffer's stack does **not** affect the other buffer's stack — verified by overflowing **main** (alt intact) *and*, reciprocally, by overflowing **alt** (main intact) — and a **multi-level** main stack is shown surviving an intervening round-trip fully intact.

**Observed output (unedited):**

```text
push 1..8 -> pop sequence (top first): [8, 7, 6, 5, 4, 3, 2, 1, 0, 0, 0]
push 1..9 -> pop sequence (top first): [9, 8, 7, 6, 5, 4, 3, 2, 0, 0, 0, 0]
```

```text
after overflow+drain main: main flags=0  alt flags=3  (alt preset 3 intact)
```

Reading the first block: after pushing the values `1..8` and then repeatedly reading-the-top-then-popping, all eight values `8,7,6,5,4,3,2,1` come back out before the stack is empty (then `0`), so **all 8 are retained**. After pushing `1..9`, the sequence that comes back is `9,8,7,6,5,4,3,2` — the oldest value **`1` has been evicted**, confirming FIFO eviction on overflow.

The second block is the cross-buffer isolation check: a value (`3`) is preset on the **alternate** stack, then the **main** stack is overflowed (nine pushes) and fully drained. Main ends at `flags=0` (drained to legacy), yet the alternate stack still reports its preset `flags=3` — **untouched** by anything that happened to main.

**Reciprocal check — overflow the *alternate* buffer, main preset intact (observed).** To prove isolation is not one-directional, the symmetric experiment presets **main** to `2`, switches to alternate, overflows and fully drains the **alternate** stack, then switches back to main:

```text
after overflow+drain alt : alt flags=0  main flags=2  (main preset 2 intact)
```

The alternate stack drains to `0`, while main's preset `flags=2` is exactly where it was left — so exhausting **either** buffer leaves the **other** untouched.

**Multi-level restoration across a round-trip (observed).** OBJ-1 shows a single pushed level surviving; here a **three-level** main stack (`push 1`, `push 2`, `push 4`) is left in place, the terminal switches to the alternate screen and pushes there, then switches back — and the entire main stack is intact, popping back down level-by-level:

```text
main pushed [1,2,4]; top before=4 after round-trip=4; drain (top first): [4, 2, 1, 0]
```

The top is `4` both before and after the round-trip, and draining yields `4 → 2 → 1 → 0` — every level preserved in order, confirming the round-trip restores the *whole* stack, not merely its top entry.

**Cause → effect.**

- *Capacity 8 is a code fact, not a documented number.* Storage is the fixed pair `uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;` (`kitty/screen.h:128`). The array length **8** determines the capacity. The specification (`docs/keyboard-protocol.rst:299-303`) only mandates that a limit *exists* ("Terminals should limit the size of the stack …") and that eviction is oldest-first ("If a push request is received and the stack is full, the oldest entry from the stack must be evicted"); the concrete number **8** comes solely from the array size at `kitty/screen.h:128`.
- *FIFO eviction.* `screen_push_key_encoding_flags` (`kitty/screen.c:1234`) finds the highest occupied slot (marked by the high bit `0x80`). When that top slot is already the last index (`current_idx == sz - 1`, i.e. index 7) it performs `memmove(self->key_encoding_flags, self->key_encoding_flags + 1, (sz - 1) * sizeof(...))` (`kitty/screen.c:1241`), sliding the whole array down by one and thereby **dropping the oldest slot (index 0)**; the new value is then written into the top slot. No exception path exists, so overflow is handled silently by design.
- *Cross-buffer isolation under exhaustion* follows directly from the two **separate** arrays (`kitty/screen.h:128`) and the pointer-only toggle (`kitty/screen.c:1079,1086`): operations on main touch only `main_key_encoding_flags`, so `alt_key_encoding_flags` cannot change no matter how hard main is exercised — and vice versa, as the reciprocal capture shows.

---

## Section 5 — OBJ-3: Controlled test with real byte capture

**Direct answer.** For the same key press `Ctrl+Shift+a`, the exact bytes transmitted to the child are `ESC [ 9 7 ; 6 u` (`b'\x1b[97;6u'`, **seven** bytes) in **every** stack state exercised in the round-trip (`flags` = 0, 1, 8) and on **both** buffers — see the stage table in OBJ-1 (rows A–D). This invariance is specifically for the **primary-key** encoding, which holds because `Ctrl+Shift+a` is not legacy-representable and always uses the CSI-`u` form. The bytes change when **report-alternate-keys** (`0b100`) is enabled **and** the encoder is handed alternate key codepoints, which are appended to the primary key as `unicode-key:shifted-key:base-layout-key`. There are therefore several distinct outputs depending on *which* alternate sub-fields are supplied (shifted only, base-layout only, or both), enumerated below — so the accurate claim is *"flag-invariant for the primary-key encoding of `Ctrl+Shift+a`"*, not *"invariant under every possible flag and parameter combination."*

**Observed output (unedited) — the same seven-byte result at every stage:**

```text
A)  main, no flags pushed                flags=0   bytes=b'\x1b[97;6u'
B)  main, after CSI>1u (disambiguate)     flags=1   bytes=b'\x1b[97;6u'
C0) alt, just switched (before push)      flags=0   bytes=b'\x1b[97;6u'
C)  alt, after CSI>8u (report-all)        flags=8   bytes=b'\x1b[97;6u'
D)  main, after switching back            flags=1   bytes=b'\x1b[97;6u'   <-- main flag 1 SURVIVED
```

**Cause → effect.** A `Ctrl+Shift`+letter combination **cannot be represented in legacy encoding**, so kitty's encoder always falls back to the disambiguating CSI-`u` form regardless of which progressive-enhancement flags are active *for the primary-key encoding* — which is why the bytes are invariant across the tested states here (the report-alternate-keys sub-fields are shown afterward). Decoding the seven bytes:

- `\x1b[` is the CSI introducer (`ESC [`).
- `97` is the key's Unicode codepoint, `ord('a')` = 97 (decimal).
- `;6` is the modifier parameter. The modifier value is `1 + (shift=1, alt=2, ctrl=4, super=8, hyper=16, meta=32, …)`. This is exactly the arithmetic in the test harness helper `csi()` (`kitty_tests/keys.py:22`): it computes `m` with `shift → m|=1`, `alt → m|=2`, `ctrl → m|=4`, … (`kitty_tests/keys.py:38-49`) and then emits `;{m+1}` (`kitty_tests/keys.py:51`). For `Ctrl+Shift`, `m = 4 | 1 = 5`, so `;{m+1}` = `;6`.
- `u` is the CSI-`u` final byte.

**Cross-check (observed):** encoding `Ctrl+Shift+a` directly and comparing against the hand-computed CSI string confirmed the arithmetic (the probe **asserts** this equality, so a mismatch would abort the run):

```text
ord('a') = 97
GLFW_MOD_SHIFT = 1 (csi maps ->1)
GLFW_MOD_CONTROL = 4 (csi maps ->4)
computed m = 5 -> ;{m+1} = ;6
encode_key_for_tty(Ctrl+Shift+a, flags=0) = b'\x1b[97;6u'
expected CSI form                          = b'\x1b[97;6u'
MATCH: True
```

**Alternate-key sub-fields — how the `Ctrl+Shift+a` bytes *do* change (observed).** The invariance above is for the *primary-key* encoding. When report-alternate-keys (`0b100`) is set, the encoder appends whichever alternate codepoints it is given, joined to the primary key by colons as `unicode-key:shifted-key:base-layout-key` (`docs/keyboard-protocol.rst:139-159`). Both sub-fields are independent, so `Ctrl+Shift+a` has **four** observed forms depending on what is supplied:

```text
flags=0 (no bit4), no shifted_key      : b'\x1b[97;6u'
flags=4 (bit4), no shifted_key         : b'\x1b[97;6u'
flags=4 (bit4), shifted_key=65 ('A')   : b'\x1b[97:65;6u'
flags=4 (bit4), alternate_key=98 ('b') : b'\x1b[97::98;6u'
flags=4 (bit4), shifted=65 + alt=98    : b'\x1b[97:65:98;6u'
```

So with bit 4 set: a shifted key alone gives `97:65` (`a`/`A`); a base-layout key alone gives the **double-colon** `97::98` (empty shifted field, `docs/keyboard-protocol.rst:155-159`); and both give `97:65:98`. Note that merely setting bit 4 without supplying any alternate codepoint leaves the bytes at `b'\x1b[97;6u'` — the sub-fields appear only when the encoder is actually handed the alternate key(s). These are the conditions under which the `Ctrl+Shift+a` byte stream departs from `b'\x1b[97;6u'`, which is why the OBJ-3 invariance is stated for the *primary-key encoding* rather than unconditionally.

**The path these bytes travel.** In the live runtime, a key event is composed by `encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)` (`kitty/keys.c:250-251`) — note the third argument is the **current** flags of the **currently active** buffer, read via `screen_current_key_encoding_flags` (`kitty/screen.c:1204`). The Python entry point used for this investigation, `pyencode_key_for_tty` (`kitty/keys.c:311-319`), composes the **same** `encode_glfw_key_event(...)` call and is registered to Python as `encode_key_for_tty` at `kitty/keys.c:334`; its keyword arguments (`key`, `shifted_key`, `alternate_key`, `mods`, `action`, `key_encoding_flags`, `text`, `cursor_key_mode`) are exactly the sub-fields exercised above. That is why encoding through `encode_key_for_tty` is a faithful stand-in for what the terminal sends to the child.

---

## Section 6 — OBJ-4: Proof of independence (and no leakage under rapid switching)

**Direct answer.** The two buffers keep **independent** stacks, and there is **zero** leakage across rapid buffer switching while the keyboard mode is being manipulated. With main holding `flags=1` and alternate holding `flags=8`, four consecutive main↔alt cycles show each buffer reporting exactly its own value every time.

**Observed output (unedited):**

```text
rapid switch cycle 0: main flags=1  alt flags=8
rapid switch cycle 1: main flags=1  alt flags=8
rapid switch cycle 2: main flags=1  alt flags=8
rapid switch cycle 3: main flags=1  alt flags=8
```

The alternate value is read immediately after `CSI ? 1049 h` and the main value immediately after `CSI ? 1049 l`, on every cycle. `main` is *always* `1` and `alt` is *always* `8`; the values never bleed into each other. This is reinforced by the OBJ-2 cross-buffer results — after overflowing and draining **main**, the alternate stack still reported its preset value (`alt flags=3`), and reciprocally after overflowing **alt**, main's preset (`flags=2`) was intact.

**Cause → effect.** Independence is **structural, not copy-based**. The `Screen` owns two separate fixed arrays plus one active pointer — `uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;` (`kitty/screen.h:128`). A buffer switch only **repoints** `key_encoding_flags` (`kitty/screen.c:1079` to alt, `kitty/screen.c:1086` to main) and never copies data from one array to the other nor clears either. Because the arrays are physically distinct storage and nothing is ever transferred between them, no sequence of switches — however rapid — can move a value from one buffer's stack to the other's. This exactly implements the specification's requirement that *"the main and alternate screens in the terminal emulator must maintain their own, independent, keyboard mode stacks"* and its stated rationale that a program on the alternate screen (e.g. an editor) can change the keyboard mode there *"without affecting the mode in the main screen or even knowing what that mode is"* (`docs/keyboard-protocol.rst:300-301,305-312`).

---

## Section 7 — OBJ-5: Mode/setting-dependent differences and conditions where isolation could break down

**Direct answer.** The per-screen stack **isolation itself does not break down** under any condition observed here. What *does* depend on terminal modes/settings are these behavioral differences, each covered below with its own evidence: (a) **which DEC mode** performs the buffer switch, (b) the **empty/over-pop reset** to legacy `0`, (c) **flag-dependent encoding** of legacy-representable keys, (d) the **query response**, and (e) a full-reset **RIS** (`ESC c`) that returns to the main buffer and clears **both** stacks. None of these move a value from one buffer's stack into the other's: RIS resets each stack independently to `0`, and the DEC-mode choice changes cursor/screen side effects but not the stacks. Omitted set/push/pop parameters resolve to their documented defaults (see §2).

### (a) DEC mode used for switching — 1049 vs 47 / 1047

All three DEC private modes toggle the alternate screen, but they differ in side effects. Only `1049` additionally **saves the cursor** and **clears the alternate screen**; `47` and `1047` do neither. The constants are:

- `SAVE_CURSOR` = `1048 << 5` — `kitty/modes.h:72`
- `TOGGLE_ALT_SCREEN_1` = `47 << 5` — `kitty/modes.h:75`
- `TOGGLE_ALT_SCREEN_2` = `1047 << 5` — `kitty/modes.h:76`
- `ALTERNATE_SCREEN` = `1049 << 5` — `kitty/modes.h:77`

In the mode dispatch, all three fall through to the same call, and the `save_cursor` / `clear_alt_screen` arguments are both the boolean `mode == ALTERNATE_SCREEN` (`kitty/screen.c:1165-1169`):

```
case TOGGLE_ALT_SCREEN_1:
case TOGGLE_ALT_SCREEN_2:
case ALTERNATE_SCREEN:
    if (val && self->linebuf == self->main_linebuf) screen_toggle_screen_buffer(self, mode == ALTERNATE_SCREEN, mode == ALTERNATE_SCREEN);
    else if (!val && self->linebuf != self->main_linebuf) screen_toggle_screen_buffer(self, mode == ALTERNATE_SCREEN, mode == ALTERNATE_SCREEN);
```

The **mechanism** is that the key-encoding-flag pointer is repointed **identically for all three modes**: the repoint at `kitty/screen.c:1079`/`1086` runs unconditionally inside `screen_toggle_screen_buffer`, independent of the `save_cursor`/`clear_alt_screen` arguments. Rather than rely on that reading alone, each mode was driven through the **real parser** and compared directly. For every mode the probe positions the cursor at row 4 / col 5 (`CSI 4 ; 5 H` → cursor `(x=4, y=3)`), pushes disambiguate on main (`CSI > 1 u`), enters the alternate screen (`CSI ? <mode> h`), pushes report-all-keys on alternate (`CSI > 8 u`) and writes `ZZ`, switches back (`CSI ? <mode> l`), then re-enters (`CSI ? <mode> h`) — reading the flag stacks, cursor, and alt-screen text at each stage:

**Observed output (unedited) — DEC 47 vs 1047 vs 1049, stack + cursor + alt-screen at each stage (before / during / after / re-entry):**

```text
mode 47   | stack: main_before=1 alt_during=0 main_after=1 alt_reentry=8
           cursor: main_before=(4, 3) alt_entry=(0, 0) main_after=(2, 0)  (NOT restored)
           alt screen: during='ZZ||||' reentry='ZZ||||'  (NOT cleared)
mode 1047 | stack: main_before=1 alt_during=0 main_after=1 alt_reentry=8
           cursor: main_before=(4, 3) alt_entry=(0, 0) main_after=(2, 0)  (NOT restored)
           alt screen: during='ZZ||||' reentry='ZZ||||'  (NOT cleared)
mode 1049 | stack: main_before=1 alt_during=0 main_after=1 alt_reentry=8
           cursor: main_before=(4, 3) alt_entry=(0, 0) main_after=(4, 3)  (RESTORED)
           alt screen: during='ZZ||||' reentry='||||'  (CLEARED on re-entry)
```

**What is observed.** For **all three** modes the flag-stack columns are **identical** — `main_before=1`, `alt_during=0` (the alternate buffer's own initially-empty stack), `main_after=1` (main's pushed flag survived the round-trip), and `alt_reentry=8` (the alternate buffer's own pushed flag survived). So the **choice of DEC mode does not affect the flag-stack isolation** — this is an **observed** result, not merely a code inference. What **does** differ is precisely the cursor and alt-screen side effects: under **`1049`** the cursor is **restored** to its pre-switch `(4, 3)` and the alternate screen is **cleared** on re-entry (`reentry='||||'`), whereas under **`47`** and **`1047`** the cursor is **not** restored (it stays at `(2, 0)`, where writing `ZZ` left it) and the alternate-screen content **persists** (`reentry='ZZ||||'`). This matches the source exactly: `save_cursor` / `clear_alt_screen` are `mode == ALTERNATE_SCREEN` (true only for `1049`) while the flag-pointer repoint is unconditional. (The rest of the investigation uses `1049`; the mode difference matters for cursor/screen contents, not for the stacks.)

### (b) Empty / over-pop reset, and the query response

**Observed output (unedited):**

```text
after push 5: flags=5
query CSI ?u response to child: b'\x1b[?5u'
after pop 3 (>pushed): flags=0 (reset)
query after reset:               b'\x1b[?0u'
```

**Cause → effect.** After a single `CSI > 5 u`, exactly one user entry (value 5) sits above the base, so `flags=5`. Popping **3** with `CSI < 3 u` when fewer than three user entries exist drains the stack: `screen_pop_key_encoding_flags` (`kitty/screen.c:1248`) clears the top `num` occupied (`0x80`-marked) slots, and when none remain set, `screen_current_key_encoding_flags` (`kitty/screen.c:1204`) returns `0` (legacy). This matches the spec: *"If a pop request is received that empties the stack, all flags are reset"* (`docs/keyboard-protocol.rst:301-302`). The query handler `screen_report_key_encoding_flags` (`kitty/screen.c:1212`) builds the reply with `snprintf(buf, sizeof(buf), "?%uu", screen_current_key_encoding_flags(self))` and then `write_escape_code_to_child(self, ESC_CSI, buf)`, so `CSI ? u` is answered with `CSI ? flags u` — observed as `b'\x1b[?5u'` before the reset and `b'\x1b[?0u'` after.

### (c) Flags genuinely alter encoding (why the active buffer's mode matters)

For a key that legacy mode **can** represent (a plain `a`), the active flags change the emitted bytes:

**Observed output (unedited):**

```text
plain 'a' under flags=0  (legacy)          : b'a'
plain 'a' under flags=1  (disambiguate)    : b'a'
plain 'a' under flags=8  (report-all-keys) : b'\x1b[97u'
plain 'a' under flags=16 (report-text)     : b'a'
```

**Cause → effect.** Report-all-keys (`0b1000`) forces even a plain `a` into an escape code, `CSI 97 u` (`b'\x1b[97u'`), because that flag *"turns on key reporting even for key events that generate text"* (`docs/keyboard-protocol.rst:384-385`). Disambiguate (`0b1`, `docs/keyboard-protocol.rst:319-327`) and report-associated-text (`0b10000`, `docs/keyboard-protocol.rst:398-400`) leave the plain key as the literal byte `b'a'`. This is the concrete reason the buffer a key is encoded under matters: because each buffer carries its **own** current flags, the **same** physical key can yield **different** bytes depending on which buffer is active — e.g. a plain `a` typed on an alternate screen running under report-all-keys would be `b'\x1b[97u'`, while the same key on a main screen in legacy mode would be `b'a'`. (Contrast this with `Ctrl+Shift+a` in OBJ-3, whose *primary-key* encoding is flag-invariant because it is not legacy-representable; its bytes change only when report-alternate-keys is set and an alternate key codepoint is supplied, appending `:65` and/or `::98`.)

### (d) How the parser routes these operations

So readers can see how each escape code reaches the handlers above, the CSI-`u` family is dispatched in `kitty/vt-parser.c:1217-1237`:

- a **bare** `CSI u` (no modifiers, no params) calls `screen_restore_cursor` — this is a cursor operation, **not** a keyboard-protocol operation; it is called out here to avoid confusion (`kitty/vt-parser.c:1220`).
- `CSI ? u` → `screen_report_key_encoding_flags` (query) (`kitty/vt-parser.c:1225`).
- `CSI = u` → `screen_set_key_encoding_flags` with `how = 1` (set; omitted flags default to 0) (`kitty/vt-parser.c:1229`).
- `CSI > u` → `screen_push_key_encoding_flags`, flags defaulting to 0 (push) (`kitty/vt-parser.c:1233`).
- `CSI < u` → `screen_pop_key_encoding_flags`, number defaulting to 1 (pop) (`kitty/vt-parser.c:1237`).

These are the defaults exercised by the omitted-parameter block in §2.

### (e) RIS (`ESC c`) full reset — returns to main and clears *both* stacks

A hard reset (RIS, `ESC c`) is the one observed operation that touches **both** stacks at once — but it does so by resetting each independently, not by leaking one into the other. With `flags=1` pushed on main and `flags=8` pushed on alternate (currently active), issuing `ESC c` returns the terminal to the **main** buffer and clears **both** stacks to `0`:

**Observed output (unedited):**

```text
before RIS (on alt): alt flags=8
after RIS: current=0  main=0  alt=0  (both cleared, active buffer=main)
```

**Cause → effect.** `ESC c` is dispatched as RIS in the parser — `case ESC_RIS: CALL_ED(screen_reset)` (`kitty/vt-parser.c:277-278`) — invoking `screen_reset` (`kitty/screen.c:162`). That function first switches back to the main buffer if the alternate is active (`if (self->linebuf == self->alt_linebuf) screen_toggle_screen_buffer(self, true, true)`, `kitty/screen.c:165`), which is why the active buffer afterward is **main**; it then zeroes **both** flag arrays explicitly — `memset(self->main_key_encoding_flags, 0, sizeof(...))` and `memset(self->alt_key_encoding_flags, 0, sizeof(...))` (`kitty/screen.c:173-174`). Because each array is cleared on its own, the reset preserves — rather than violates — the independence of the two stacks; it simply returns the whole terminal to the legacy keyboard state.

**Summary for OBJ-5.** No observed condition caused the two stacks to leak into each other or the per-screen isolation to break down. The mode/setting-dependent *differences* are those itemized above: only (c) changes the bytes a key press produces; (a) changes cursor/screen side effects but leaves stack isolation intact; (b) and (e) reset flags to legacy `0` (over-pop resets the active stack; RIS resets both, each independently); (d) is the read-back query; and omitted parameters fall back to their documented defaults.

---

## Section 8 — Appendix: storage, operations, and harness (grounding)

**Per-screen storage and the stack operation prototypes.**

- `uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;` — `kitty/screen.h:128`
- Prototypes `screen_set_key_encoding_flags` / `screen_push_key_encoding_flags` / `screen_pop_key_encoding_flags` / `screen_current_key_encoding_flags` / `screen_report_key_encoding_flags` — `kitty/screen.h:269-273`

**Operation bodies (all in `kitty/screen.c`).**

| Operation | Function | Line | Behavior |
|-----------|----------|------|----------|
| current | `screen_current_key_encoding_flags` | `1204` | scans the active array from the top; returns the first `0x80`-marked slot masked with `0x7f`, else `0` |
| report/query | `screen_report_key_encoding_flags` | `1212` | `snprintf("?%uu", current)` → `write_escape_code_to_child` → `CSI ? flags u` |
| set | `screen_set_key_encoding_flags` | `1220` | `how` 1 = replace, 2 = OR, 3 = AND-NOT on the top slot |
| push | `screen_push_key_encoding_flags` | `1234` | advances the `0x80` marker; on overflow (`current_idx == sz-1`) `memmove`s down one, evicting the oldest (`kitty/screen.c:1241`) |
| pop | `screen_pop_key_encoding_flags` | `1248` | clears the top `num` occupied slots; over-pop resets to legacy `0` |
| toggle buffer | `screen_toggle_screen_buffer` | `1068` | repoints `key_encoding_flags` to alt (`1079`) / main (`1086`); never copies/clears |
| reset (RIS) | `screen_reset` | `162` | switches to main if on alt (`165`), then `memset`s **both** flag arrays to `0` (`173-174`) |

**Parser dispatch.** The CSI-`u` family (set / query / push / pop / bare-restore) is routed at `kitty/vt-parser.c:1217-1237`; RIS (`ESC c`) is routed at `kitty/vt-parser.c:277-278` to `screen_reset`.

**Python-side helpers (`kitty/key_encoding.py`).** `class EventType(IntEnum)` (`:175`), `class KeyEvent(NamedTuple)` (`:201`), `def encode_key_event` (`:365`).

**Headless harness (`kitty_tests/`).** `parse_bytes` drives the real VT parser (`kitty_tests/__init__.py:30`); `Callbacks.write` accumulates child-bound bytes into `wtcbuf` (`kitty_tests/__init__.py:50-51`); `create_screen` is the standard `Screen` factory (`kitty_tests/__init__.py:237`). Alternate-buffer test patterns live in `kitty_tests/screen.py`, and the modifier-encoding `csi()` helper cross-checked in OBJ-3 is in `kitty_tests/keys.py:22`.

**Out-of-scope note.** `keyboard_mode_stack` (`kitty/keys.py:67`) is kitty's internal key-*mapping* mode stack and is a **different** mechanism from the per-screen C protocol-enhancement flag stack that is the subject of this document. It is named here solely to keep the two from being conflated.
