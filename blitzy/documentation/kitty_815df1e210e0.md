# How kitty's keyboard-protocol progressive-enhancement flag stack behaves across main ↔ alternate screen buffers

> **Investigation target:** kitty at pinned commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.
> **Method:** run-first. Every behavioral claim below is backed by **captured, unedited runtime output** produced by building kitty in its default configuration and driving the real code paths headlessly, then corroborated at the real child/PTY boundary. Code-reading is used only to explain *why* the observed bytes are what they are, and every such explanation is tied to a `file:line` citation.
> **Scope:** this document concerns **only** the *progressive-enhancement flags stack* — the `CSI > flags u` push / `CSI < number u` pop stack held **per screen buffer** in the C screen model (`kitty/screen.h:L128`). It is **not** about the unrelated `kitty.conf` keyboard-*mapping* mode stack (`keyboard_mode_stack`, `kitty/keys.py:L67`); see §10.

---

## 1. TL;DR — the direct answer

**Each screen buffer owns a physically separate, 8-slot flags array. Switching buffers does not copy or clear anything — it merely repoints one active pointer. The stacks are therefore fully independent, and a stack pushed on the main screen survives an alternate-screen round-trip completely untouched.**

Concretely, the `Screen` C struct holds **two** distinct arrays plus one active pointer:

```c
// kitty/screen.h:L128
uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;
```

Switching to the alternate buffer runs `screen_toggle_screen_buffer` (`kitty/screen.c:L1068`), whose entire effect on keyboard flags is a **single pointer re-assignment** — to `alt_key_encoding_flags` when entering the alt screen (`kitty/screen.c:L1079`) and back to `main_key_encoding_flags` when leaving it (`kitty/screen.c:L1086`). Neither array is ever memcpy'd into the other and neither is cleared on switch. That one fact is the *cause* of every observed behavior:

- **Independence (Q1, Q6):** the two arrays are separate storage, so a push on one is invisible to the other.
- **Round-trip survival (Q2):** because the switch only repoints, the main array is byte-for-byte identical after a `main → alt → main` round-trip. Observed `current_key_encoding_flags()` readings across the round-trip are **1 → 0 → 8 → 1** (see §4).
- **Overflow isolation (Q4):** an overflow that evicts entries on one array cannot touch the other array (see §6).

The remaining nuances — the exact per-state key bytes, the capacity limit, the overflow/reset rules, and the one setting that changes encoding — are laid out with their raw evidence in §3–§9. One headline correction: the modified key `Ctrl+Shift+a` encodes to `\x1b[97;6u` in **every** state observed (main-legacy, main-disambiguate, alt-report-all, and back-on-main), *not* to the control byte `0x01`. `0x01` is `Ctrl+a` **without** Shift. The mode differences between buffers are proven instead by (i) the **plain, unmodified `a`** — which diverges by state — and (ii) the decisive `current_key_encoding_flags()` readings. Full detail and grounding in §5.

---

## 2. Environment and the exact build / invocation commands

### 2.1 Canonical build (state these exact commands)

All observation was performed inside the pinned Docker image specified by the project setup:

```
# Image (per project setup instructions):
#   ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0
#   (alias andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1)
# Canonical toolchain in-image: Python 3.12.3, gcc 13.3.0, Go 1.23.4, pkg-config 1.8.1.

cd /workspace                       # the repo, bind-mounted at commit 815df1e210e0…
python3 setup.py --skip-building-kitten
# -> compiles the C extension kitty/fast_data_types.so (which hosts Screen,
#    parse_bytes/test_parse_written_data, and encode_key_for_tty) and the
#    launcher ./kitty/launcher/kitty
```

`--skip-building-kitten` is acceptable for the **primary** (headless) evidence because only the C extension `kitty.fast_data_types` is required to instantiate a real `Screen`, feed it raw escape sequences, and call the real key encoder. The launcher `./kitty/launcher/kitty` is needed only for the full-GUI corroboration (§7). kitty's default build uses `-pedantic-errors -Werror`; no warning-suppression flag was passed (that would be non-canonical).

### 2.2 Headless investigation primitives (no display / no GPU)

Every primary observation uses this entry point, run from the repo root so that `sys.path.insert(0, '.')` (or `PYTHONPATH=/workspace`) resolves `kitty.fast_data_types`:

```
PYTHONPATH=/workspace python3 <script>.py
```

The primitives exercised, and the exact code they reach:

| Primitive | Reaches | Citation |
|-----------|---------|-----------|
| `create_screen(cols=80, lines=24)` (via `kitty_tests.BaseTest`) | builds a real headless `Screen` | `kitty_tests/__init__.py:L237`, `L208` |
| `parse_bytes(s, b'\x1b[>1u')` | the real VT parser → CSI-u dispatch | `kitty_tests/__init__.py:L30`; `kitty/vt-parser.c:L1217` |
| `s.toggle_alt_screen()` | `screen_toggle_screen_buffer(self, true, true)` | `kitty/screen.c:L4449`, `L4451`, `L1068` |
| `s.current_key_encoding_flags()` | `screen_current_key_encoding_flags` | `kitty/screen.c:L3951`, `L1204` |
| `fdt.encode_key_for_tty(key=…, mods=…, key_encoding_flags=…)` | `pyencode_key_for_tty` → `encode_glfw_key_event` | `kitty/keys.c:L311`, `L319` |

### 2.3 Determinism protocol

Every scenario below was run at least twice (the `main → alt → main` round-trip three times); the captured output of each script was hashed (`md5sum`) across repeats and is **byte-identical** run-to-run. The exact hashes are recorded in §11. Because the results are perfectly stable, the user's report of "order-dependent" behavior is **not** nondeterminism — it is the *real, deterministic* consequence of per-buffer stack independence, explained in §4 and §11.

### 2.4 One honest environment limitation (labeled non-canonical)

The interactive full-GUI capture (launch the compositor window, physically press keys, watch a child receive bytes) **could not be run in this headless environment**: kitty's GUI requires a display and none is available —

```
$ echo "DISPLAY=[$DISPLAY] WAYLAND_DISPLAY=[$WAYLAND_DISPLAY]"
DISPLAY=[] WAYLAND_DISPLAY=[]
$ which xvfb-run Xvfb || echo "(no xvfb available)"
(no xvfb available)
```

This affects exactly **one** hop of the pipeline — the GLFW-windowing-event → encoder call — and nothing else. That hop is covered by source grounding (§7): the GUI keypress path and the Python encoder binding call the **same** function, `encode_glfw_key_event` (`kitty/keys.c:L251` vs `L319`). The authoritative child-boundary bytes are instead captured with a **real forked child over a real PTY** (§7), which is canonical for the "bytes transmitted to the child process" question the user asked. The GUI-only commands and their `cat -v` renderings are documented and their rendering verified programmatically against the exact captured bytes.

---

## 3. Stack ownership model (Question 1)

**Answer: each screen buffer has its *own independent* flags stack. Nothing is copied on switch, nothing is shared; the buffer switch only repoints an active pointer.**

### 3.1 The storage — two arrays and one pointer

The per-buffer stacks are declared on the `Screen` struct as two fixed-size arrays plus a single pointer that selects which one is "active":

```c
// kitty/screen.h:L128
uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;
```

- `main_key_encoding_flags[8]` — the main screen's stack (8 slots).
- `alt_key_encoding_flags[8]` — the alternate screen's stack (8 slots).
- `key_encoding_flags` — a pointer that always points at whichever of the two arrays belongs to the currently active buffer.

All stack operations (push/pop/set/current/report) dereference the **pointer** `key_encoding_flags`, never a specific array by name. That indirection is the whole mechanism: to make a buffer's stack "the active one," kitty simply points `key_encoding_flags` at that buffer's array. The prototypes for the operations are declared together at `kitty/screen.h:L269-273`.

### 3.2 Initialization and reset both treat the arrays as separate

At screen construction the active pointer is seeded to the main array:

```c
// kitty/screen.c:L150
self->key_encoding_flags = self->main_key_encoding_flags;
```

A full reset zeroes **both** arrays independently (they are separate storage, so both must be cleared):

```c
// kitty/screen.c:L173-174   (reset path memsets BOTH arrays)
memset(self->main_key_encoding_flags, 0, sizeof(self->main_key_encoding_flags));
memset(self->alt_key_encoding_flags, 0, sizeof(self->alt_key_encoding_flags));
```

### 3.3 The buffer switch repoints — it does not copy or clear

The function that performs the main↔alt switch is `screen_toggle_screen_buffer` (`kitty/screen.c:L1068`). It decides direction from which line buffer is active:

```c
// kitty/screen.c:L1068-1069
static void
screen_toggle_screen_buffer(Screen *self, bool save_cursor, bool clear_alt_screen) {
    bool to_alt = self->linebuf == self->main_linebuf;
    ...
```

and its **entire** effect on the keyboard-flags state is one of two pointer assignments:

```c
// kitty/screen.c:L1079   (entering the ALT screen)
self->key_encoding_flags = self->alt_key_encoding_flags;
// kitty/screen.c:L1086   (returning to the MAIN screen)
self->key_encoding_flags = self->main_key_encoding_flags;
```

There is **no** `memcpy` between the two arrays and **no** memset of either array anywhere in the switch. The specific function that performs the work is `screen_toggle_screen_buffer`, and the specific statements are the two pointer re-assignments at `L1079`/`L1086`. **Cause → effect:** because switching only changes *which array is read*, and never changes the *contents* of either array, (a) the two stacks are independent and (b) whatever was on the main array before entering alt is still there, unchanged, when you come back. This is proven at runtime in §4 (round-trip) and §8 (rapid-switch leakage probe).

### 3.4 The spec agrees

The in-repo specification requires exactly this: it states that terminals must maintain separate stacks for the main and alternate screens (`docs/keyboard-protocol.rst:L299-303`, `L306-309`). The canonical published specification concurs — it says the main and alternate screens must maintain their own independent keyboard mode stacks, precisely so a program on the alternate screen (such as an editor) can change the mode without affecting, or even knowing, the main screen's mode (`https://sw.kovidgoyal.net/kitty/keyboard-protocol/`). The kitty C implementation is the concrete realization of that requirement.


---

## 4. Round-trip survival (Question 2)

**Answer: after `start on main, push A → toggle to alternate, push B → toggle back to main`, the encoding mode active back on main is the one main pushed (`A`) — the main stack survives the alternate-screen round-trip completely untouched. The freshly-entered alternate screen starts from its own empty stack, independent of main.**

### 4.1 Raw output — the exact `push → toggle → push → toggle` sequence, repeated ×3

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 1 block
```

Raw output (complete, unedited):

```
### TEST 1: main->alt->main round-trip (repeat x3 for stability) ###
 run1: start-main-empty=0 | push>1u-main=1 | toggled-ALT-prepush=0 | push>8u-alt=8 | toggled-back-MAIN=1
 run2: start-main-empty=0 | push>1u-main=1 | toggled-ALT-prepush=0 | push>8u-alt=8 | toggled-back-MAIN=1
 run3: start-main-empty=0 | push>1u-main=1 | toggled-ALT-prepush=0 | push>8u-alt=8 | toggled-back-MAIN=1
```

The exact bytes driven into the real VT parser at each step were: `\x1b[>1u` (push flag `1` = disambiguate, on main), then `toggle_alt_screen()`, then `\x1b[>8u` (push flag `8` = report-all-keys, on alt), then `toggle_alt_screen()` back. The value read at each step is `screen.current_key_encoding_flags()`.

### 4.2 Reading — the decisive `1 → 0 → 8 → 1` sequence (before / intermediate / after)

Interpreting the readings as the state changes through the round-trip:

| Step | Action | `current_key_encoding_flags()` | Meaning |
|------|--------|-------------------------------|---------|
| start | fresh main buffer | **0** | main stack empty |
| push A | `\x1b[>1u` on main | **1** | main top = disambiguate |
| toggle → alt | `toggle_alt_screen()` | **0** | alt buffer's *own* stack, still empty — **not** main's `1` |
| push B | `\x1b[>8u` on alt | **8** | alt top = report-all-keys |
| toggle → main | `toggle_alt_screen()` | **1** | **main's `1` is exactly as left — it survived** |

Two independent facts fall directly out of this:

1. **The main stack survives untouched.** The final reading back on main is `1`, identical to the value main held before the excursion to the alternate screen. Nothing that happened on the alternate buffer (the `\x1b[>8u` push) changed it.
2. **The alternate buffer has its own, independent, initially-empty stack.** Immediately after `toggle_alt_screen()` the reading is `0`, *not* the `1` that was active on main. If flags were copied across the switch, this intermediate reading would have been `1`; it is `0`, so nothing is copied — consistent with §3.3's "repoint only" mechanism.

### 4.3 Cause → effect and grounding

The mechanism is the pointer repoint in `screen_toggle_screen_buffer` (`kitty/screen.c:L1079`/`L1086`, §3.3): entering alt repoints `key_encoding_flags` at `alt_key_encoding_flags`, whose contents are whatever the alt buffer independently holds (initially all-zero → `current` returns `0`); leaving alt repoints back at `main_key_encoding_flags`, whose contents were never modified while alt was active. The value returned at each read is computed by `screen_current_key_encoding_flags` (`kitty/screen.c:L1204`), which scans the *active* array top-down and returns the low 7 bits of the top occupied slot, or `0` if the array is empty (`kitty/screen.c:L1205-1208`).

This directly resolves the user's confusion about "pushing after switching to alternate behaves differently from pushing on main then switching": both orders are honored, but on **different** stacks. Pushing on alt changes only the alt stack; the main stack is only ever changed by pushes performed while main is active. The published spec confirms this is the intended, required behavior (`https://sw.kovidgoyal.net/kitty/keyboard-protocol/`; in-repo `docs/keyboard-protocol.rst:L306-309`).


---

## 5. Per-state key encoding (Question 3)

**Answer: the bytes a key press produces depend on the active flags value, which is read from the active buffer's stack at press time (`kitty/keys.c:L251`). Below are the exact captured bytes for the user's example key `Ctrl+Shift+a` and — crucially — for a plain unmodified `a`, in each of the four states the user named.**

The user framed the question as a concrete four-state sequence, preserved here verbatim:

> "For instance, press a modified key like Ctrl+Shift+a while on main with no flags pushed, then again after pushing disambiguate mode, then again after switching to alternate and pushing report-all-keys mode, then back to main."

Those four states map one-to-one onto the captured output that follows: **(A)** main, no flags pushed; **(B)** main, after pushing disambiguate (`\x1b[>1u`); **(C)** alternate, after pushing report-all-keys (`\x1b[>8u`); **(D)** back on main. §5.1 (raw bytes) and §5.2 (the four-state table) give the exact bytes each state produces.

### 5.1 Raw output — bytes in each of the four states

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 2 block
```

For each state the probe reads the live `current_key_encoding_flags()` off the real `Screen` and feeds exactly that value into `encode_key_for_tty(...)` — the same encoder kitty calls to write to the child. Raw output (complete, unedited):

```
### TEST 2: per-state bytes for Ctrl+Shift+a AND plain 'a' ###
  [A: main, no push (legacy)] flags=0
        Ctrl+Shift+a  -> b'\x1b[97;6u'  hex=1b 5b 39 37 3b 36 75
        plain 'a'     -> b'a'  hex=61
  [B: main, after push >1u (disambiguate)] flags=1
        Ctrl+Shift+a  -> b'\x1b[97;6u'  hex=1b 5b 39 37 3b 36 75
        plain 'a'     -> b'a'  hex=61
  [C: alt, after push >8u (report-all-keys)] flags=8
        Ctrl+Shift+a  -> b'\x1b[97;6u'  hex=1b 5b 39 37 3b 36 75
        plain 'a'     -> b'\x1b[97u'  hex=1b 5b 39 37 75
  [D: back on main (should equal B)] flags=1
        Ctrl+Shift+a  -> b'\x1b[97;6u'  hex=1b 5b 39 37 3b 36 75
        plain 'a'     -> b'a'  hex=61
```

### 5.2 The four-state table (captured bytes)

Produced by the command in §5.1 (`Ctrl+Shift+a` via `encode_key_for_tty(key=ord('a'), mods=CTRL|SHIFT, key_encoding_flags=<state>)`; plain `a` with `mods=0`):

| # | State | Active flags | `Ctrl+Shift+a` (captured) | plain `a` (captured) | Basis (`file:line`) |
|---|-------|--------------|----------------------------|----------------------|---------------------|
| A | Main, no push | `0` (legacy) | `\x1b[97;6u` (`1b 5b 39 37 3b 36 75`) | `a` (`61`) | legacy printable-ascii path has **no** `CTRL\|SHIFT`+letter branch → returns 0 → CSI-u fall-through; `kitty/key_encoding.c:L292,L317,L396` |
| B | Main, push `\x1b[>1u` | `1` | `\x1b[97;6u` | `a` (`61`) | disambiguate escape-codes the modified key; plain printable stays literal text; `kitty/key_encoding.c:L376` (not taken) |
| C | Alt, push `\x1b[>8u` | `8` | `\x1b[97;6u` | `\x1b[97u` (`1b 5b 39 37 75`) | flag 8 = report-all-keys → **every** key CSI-u; `kitty/key_encoding.c:L422,L376` |
| D | Back on main | `1` (survived) | `\x1b[97;6u` | `a` (`61`) | main buffer's stack independent & preserved (§4) |

### 5.3 CRITICAL nuance and an honest correction

**The modified key `Ctrl+Shift+a` is `\x1b[97;6u` in all four states — it is NOT `0x01`.** A prediction had been made that State A (`flags=0`, legacy) would yield the control byte `0x01`. **That prediction is wrong, and the runtime output above is reported as observed.** `0x01` is in fact `Ctrl+a` **without** Shift (see the legacy matrix in §5.5, and the PTY capture in §7). Three independent lines of evidence establish the correct value:

1. **Direct runtime capture (§5.1):** `Ctrl+Shift+a` at `flags=0` → `b'\x1b[97;6u'`.
2. **kitty's own test suite:** `kitty_tests/keys.py:L407` asserts `enc(key=ord('i'), mods=ctrl|shift) == csi(ctrl|shift, ord('i'))`, i.e. `Ctrl+Shift+i` at `flags=0` is canonically the CSI-u form `\x1b[105;6u`. `Ctrl+Shift`+letter at legacy flags is *defined* by kitty's tests to be CSI-u, not a control byte.
3. **The protocol author:** in the originating RFC (`kovidgoyal/kitty#3248`, 2021), Kovid Goyal states that between encoding `ctrl+shift+a` as `CSI 97;6` versus `CSI 65;5`, "the former is correct." That is exactly the observed `\x1b[97;6u`.

**Why (cause → effect, grounded):** the legacy encoder for printable ASCII, `encode_printable_ascii_key_legacy` (`kitty/key_encoding.c:L292`), has explicit branches for `mods==0` (returns the literal char, `L294`), `mods==CTRL` (returns the control byte via `ctrled_key`, `L309`), `mods==ALT`, and a `CTRL|SHIFT` case that fires **only** for the space key (`L314`, inside `if (key == ' ')`). There is **no** `CTRL|SHIFT`+letter branch, so for `Ctrl+Shift+a` the function reaches `return 0;` (`kitty/key_encoding.c:L317`). Back in `encode_key`, a legacy return of 0 means "not handled by legacy," so control falls through to `return serialize(&ed, output, 'u');` (`kitty/key_encoding.c:L396`) — the CSI-u form `\x1b[97;6u`. The modifier value `6` is `1 + (CTRL|SHIFT)` = `1 + (4|1)` = `1 + 5`, produced by `snprintf(..., "%u", ev->mods.value + 1)` (`kitty/key_encoding.c:L47`; masks at `L11`).

### 5.4 The consequence: why plain `a` is the real proof of distinct modes

Because `Ctrl+Shift+a` is `\x1b[97;6u` under flags `0`, `1`, **and** `8`, the modified-key bytes **alone cannot distinguish** which mode a buffer holds. The state-distinguishing evidence is the **plain, unmodified `a`**, which diverges by flag:

- flags `0` (legacy) and `1` (disambiguate): plain `a` → literal byte `a` (`0x61`).
- flag `8` (report-all-keys): plain `a` → `\x1b[97u` (`1b 5b 39 37 75`).

This is exactly the A/B (=`a`) vs C (=`\x1b[97u`) split in the table. Under flag 8, the encoder takes the early `if (ev->report_text) return serialize(&ed, output, 'u');` branch (`kitty/key_encoding.c:L376`; the field named `report_text` is the flag-8 field — see the naming note in §12.2), so *every* key including a bare printable becomes a CSI-u sequence. Under flags 0/1 that branch is not taken and a bare printable is sent as its literal UTF-8 byte. TEST 9 (§9.4) confirms flag 8 is the **only** single flag that escape-codes a plain key. Combined with the `1 → 0 → 8 → 1` readings of §4, this pair of facts is the decisive proof that main held mode `1` while alt independently held mode `8`.

### 5.5 Raw output — full legacy-vs-disambiguate modifier matrix (supporting evidence)

To show precisely where `0x01` really comes from and how the two regimes differ across modifiers, the full matrix for the `a` key:

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_legacy.py
```

Raw output (complete, unedited):

```
=== legacy(flags=0) modifier matrix for 'a' ===
  flags=0 a              -> b'a' hex=61
  flags=0 Ctrl+a         -> b'\x01' hex=01
  flags=0 Shift+a        -> b'a' hex=61
  flags=0 Ctrl+Shift+a   -> b'\x1b[97;6u' hex=1b 5b 39 37 3b 36 75
  flags=0 Alt+a          -> b'\x1ba' hex=1b 61
  flags=0 Ctrl+Alt+a     -> b'\x1b\x01' hex=1b 01
=== disambiguate(flags=1) same matrix ===
  flags=1 a              -> b'a' hex=61
  flags=1 Ctrl+a         -> b'\x1b[97;5u' hex=1b 5b 39 37 3b 35 75
  flags=1 Shift+a        -> b'\x1b[97;2u' hex=1b 5b 39 37 3b 32 75
  flags=1 Ctrl+Shift+a   -> b'\x1b[97;6u' hex=1b 5b 39 37 3b 36 75
  flags=1 Alt+a          -> b'\x1b[97;3u' hex=1b 5b 39 37 3b 33 75
  flags=1 Ctrl+Alt+a     -> b'\x1b[97;7u' hex=1b 5b 39 37 3b 37 75
```

Reading: at `flags=0`, `Ctrl+a` is `0x01` (the `mods==CTRL` branch, `kitty/key_encoding.c:L309`), `Shift+a` is a plain `a` (Shift is consumed into the character, `L294` after Shift folds away), and only `Ctrl+Shift+a` escapes to CSI-u `\x1b[97;6u` (no legacy branch → `L317` returns 0 → `L396` fall-through). At `flags=1` (disambiguate) *every* modified form becomes CSI-u — note `Ctrl+a` becomes `\x1b[97;5u` (modifier `5 = 1 + CTRL(4)`) whereas plain `a` (no modifiers) still stays the literal byte `a`. This is the concrete difference between the legacy and disambiguate regimes and confirms `0x01 ≡ Ctrl+a`, never `Ctrl+Shift+a`.


---

## 6. Stack exhaustion / overflow behavior (Question 4)

**Answer: each buffer's stack holds exactly 8 distinct flag-states. Pushing a 9th silently evicts the *oldest* entry (a `memmove` shift-down); it is never an error. Overflowing one buffer's stack has zero effect on the other buffer's stack. Popping past the bottom empties the stack, which resets all flags to 0.**

### 6.1 Raw output — overflow, cross-buffer isolation, and pop-to-empty

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 3, 4, 5 blocks
```

Raw output (complete, unedited):

```
### TEST 3: overflow / eviction (push >8 distinct values on MAIN) ###
    pushed >1u..>12u -> current tracks [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
    unwind sequence of current values (read then CSI<1u each): [12, 11, 10, 9, 8, 7, 6, 5, 0, 0]
### TEST 4: overflow on MAIN does NOT affect ALT ###
  after 12 pushes on main, main current = 12 | alt current (never pushed) = 0 | main current again = 12
### TEST 5: pop-to-empty resets all flags (current -> 0) ###
  after push >5u=5; after push >7u=7; after pop <9u(over-pop) current = 0
```

### 6.2 Raw output — capacity measurement (proves capacity = 8)

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_capacity.py
```

Raw output (complete, unedited):

```
### capacity measurement: push K distinct values on one buffer, then read-before-pop (K+2) times ###
  K= 1: top= 1  unwind=[1, 0, 0]                                distinct_retained=1
  K= 2: top= 2  unwind=[2, 1, 0, 0]                             distinct_retained=2
  K= 3: top= 3  unwind=[3, 2, 1, 0, 0]                          distinct_retained=3
  K= 4: top= 4  unwind=[4, 3, 2, 1, 0, 0]                       distinct_retained=4
  K= 5: top= 5  unwind=[5, 4, 3, 2, 1, 0, 0]                    distinct_retained=5
  K= 6: top= 6  unwind=[6, 5, 4, 3, 2, 1, 0, 0]                 distinct_retained=6
  K= 7: top= 7  unwind=[7, 6, 5, 4, 3, 2, 1, 0, 0]              distinct_retained=7
  K= 8: top= 8  unwind=[8, 7, 6, 5, 4, 3, 2, 1, 0, 0]           distinct_retained=8
  K= 9: top= 9  unwind=[9, 8, 7, 6, 5, 4, 3, 2, 0, 0, 0]        distinct_retained=8
  K=10: top=10  unwind=[10, 9, 8, 7, 6, 5, 4, 3, 0, 0, 0, 0]    distinct_retained=8
  K=11: top=11  unwind=[11, 10, 9, 8, 7, 6, 5, 4, 0, 0, 0, 0, 0] distinct_retained=8
  K=12: top=12  unwind=[12, 11, 10, 9, 8, 7, 6, 5, 0, 0, 0, 0, 0, 0] distinct_retained=8
```

### 6.3 Reading — capacity is exactly 8; the 9th push evicts the oldest

For `K ≤ 8`, unwinding the stack recovers **all K** distinct pushed values (`K=8` → `[8,7,6,5,4,3,2,1]`, `distinct_retained=8`). Starting at `K=9`, exactly one distinct value is lost — and it is the **oldest** (`K=9` unwind is `[9,8,7,6,5,4,3,2]`; value `1` is gone). By `K=12`, the four oldest values `1,2,3,4` have been evicted and only `[12,11,10,9,8,7,6,5]` remain. So the retained capacity saturates at **8** and each push beyond 8 drops the bottom-most (oldest) entry. TEST 3 shows the same from the "current tracks" view: `current` faithfully reports each just-pushed value `1..12` (the top is always the newest), and the unwind `[12,11,10,9,8,7,6,5,0,0]` shows only 8 real values survive before the stack reads empty.

### 6.4 Cause → effect — the `memmove` shift-down, and no error

The push operation is `screen_push_key_encoding_flags` (`kitty/screen.c:L1234`). Its logic, step by step:

```c
// kitty/screen.c:L1235   uint8_t q = val & 0x7f;                 // low 7 bits are the value
// kitty/screen.c:L1236   const unsigned sz = arraysz(self->main_key_encoding_flags);  // == 8
// kitty/screen.c:L1238-1239  find current_idx = index of the top occupied slot
// kitty/screen.c:L1241   if (current_idx == sz - 1) memmove(flags, flags + 1, (sz-1)*sizeof(flags[0]));
// kitty/screen.c:L1242   else key_encoding_flags[current_idx++] |= 0x80;
// kitty/screen.c:L1243   key_encoding_flags[current_idx] = 0x80 | q;   // write new top
```

When the stack is full (top occupied slot is the last index, `sz-1`), the `memmove` at `kitty/screen.c:L1241` shifts the whole array **down by one slot**, discarding index 0 — the **oldest** entry — and then the new value is written on top at `L1243`. There is no error path, no diagnostic, no rejection: overflow is a *silent* evict-oldest. The capacity `sz` is fixed at `8` by the array declaration `main_key_encoding_flags[8]` (`kitty/screen.h:L128`), read via `arraysz(...)` at `L1236`.

### 6.5 The base-zero seed — why the clean number is 8

There is a subtle detail that makes the capacity resolve to a clean 8 rather than 7. On the **first** push to an empty array, the top-occupied-slot scan finds nothing occupied, so `current_idx` stays `0`; the `else` branch at `kitty/screen.c:L1242` sets `key_encoding_flags[0] |= 0x80` — a *phantom* occupied slot holding value `0` — and advances `current_idx` to `1`; then `L1243` writes the real value at index `1`. So after the first push the array depth is 2: `[0(phantom), value]`. That phantom `0` sits at the bottom and is therefore the **first** thing the `memmove` evicts. This is exactly why the measurement shows `K=8` retaining all 8 real values (the phantom 0 is what gets shifted out when the 8th real value is pushed) and only `K=9` evicting a real (oldest) value. The capacity of 8 *distinct real flag-states* is thus an emergent consequence of the 8-slot array plus the base-zero seed.

### 6.6 Cross-buffer isolation under overflow

TEST 4 pushes 12 values on **main** (forcing eviction), then reads alt: main is `12`, alt is `0` (it was never pushed), main again is `12`. Overflowing the main array cannot touch the alt array because they are separate storage (`kitty/screen.h:L128`) and the push only ever operates on the array pointed to by the active `key_encoding_flags` (§3). Eviction is confined to the buffer being overflowed.

### 6.7 Pop-to-empty resets all flags

TEST 5 pushes `5` then `7` (top = `7`), then pops with `\x1b[<9u` (pop 9 — far more than are present). The result is `current = 0`: emptying the stack resets all flags. The pop operation is `screen_pop_key_encoding_flags` (`kitty/screen.c:L1248`), which clears `num` occupied slots from the top (`L1249-1250`); over-popping simply leaves the array all-zero, and `screen_current_key_encoding_flags` returns `0` for an empty array (`kitty/screen.c:L1208`) — i.e. all flags reset. This matches the spec requirement that a pop emptying the stack resets all flags (`docs/keyboard-protocol.rst:L299-303`).

### 6.8 Spec corroboration

The published specification states the same two rules the runtime exhibits: a push onto a full stack evicts the oldest entry, and a pop that empties the stack resets all flags (`https://sw.kovidgoyal.net/kitty/keyboard-protocol/`; in-repo `docs/keyboard-protocol.rst:L299-303`). The specification deliberately leaves the exact stack size implementation-defined — it says only that terminals "should limit the size of the stack as appropriate, to prevent Denial-of-Service attacks" (`docs/keyboard-protocol.rst:L299-300`) and prescribes **no** numeric depth. kitty's exact capacity is therefore not a spec figure: it is `8` because `kitty/screen.h:L128` declares 8-slot arrays (`main_key_encoding_flags[8]`, `alt_key_encoding_flags[8]`), confirmed by the runtime capacity probe in §6.2/§6.3.


---

## 7. Controlled child-process observation at the real PTY boundary (Question 5)

**Answer: a real child process, reading from a real PTY, receives byte-for-byte exactly the encoder's output for every state/key combination tested. The bytes written to the child are the encoder's output because the live keypress path and the encoder binding call the identical function, `encode_glfw_key_event`.**

### 7.1 Raw output — real forked child over a real PTY

This is the canonical evidence for the user's literal question ("the actual bytes transmitted to the child process"). The probe forks a genuine child whose **stdin is a real PTY slave** (obtained from `kitty.child.openpty` — the same `openpty` kitty uses for its child PTYs), puts the slave in raw mode, has the parent write the encoder bytes to the master, and captures what the child actually `read()`s.

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_pty.py
```

Raw output (complete, unedited):

```
  [Ctrl+Shift+a flags=0]  encoder -> b'\x1b[97;6u' (1b 5b 39 37 3b 36 75) ; child -> b'\x1b[97;6u' (1b 5b 39 37 3b 36 75) ; MATCH=True
  [Ctrl+Shift+a flags=1]  encoder -> b'\x1b[97;6u' (1b 5b 39 37 3b 36 75) ; child -> b'\x1b[97;6u' (1b 5b 39 37 3b 36 75) ; MATCH=True
  [Ctrl+Shift+a flags=8]  encoder -> b'\x1b[97;6u' (1b 5b 39 37 3b 36 75) ; child -> b'\x1b[97;6u' (1b 5b 39 37 3b 36 75) ; MATCH=True
  [plain 'a'    flags=1]  encoder -> b'a' (61) ; child -> b'a' (61) ; MATCH=True
  [plain 'a'    flags=8]  encoder -> b'\x1b[97u' (1b 5b 39 37 75) ; child -> b'\x1b[97u' (1b 5b 39 37 75) ; MATCH=True
  [Ctrl+a       flags=0]  encoder -> b'\x01' (01) ; child -> b'\x01' (01) ; MATCH=True
ALL MATCH: True
```

Every case matches byte-for-byte at the child boundary. Note the last three rows again make the key point concrete: plain `a` reaches the child as literal `a` under flag 1 but as `\x1b[97u` under flag 8, and `Ctrl+a` (not `Ctrl+Shift+a`) is the one that reaches the child as `0x01`.

### 7.2 Why the encoder output *is* the child's bytes (cause → effect, grounded)

The real keypress path in kitty encodes the event and writes exactly those bytes to the child:

```c
// kitty/keys.c:L251   int size = encode_glfw_key_event(ev, screen->modes.mDECCKM,
//                                    screen_current_key_encoding_flags(screen), encoded_key);
// kitty/keys.c:L259   schedule_write_to_child(w->id, 1, encoded_key, size);
```

That is: the flags argument to the encoder is `screen_current_key_encoding_flags(screen)` (the top of the active buffer's stack, §4/§6), and the resulting bytes are handed straight to `schedule_write_to_child`. The Python binding used for the headless evidence, `pyencode_key_for_tty` (`kitty/keys.c:L311`), calls the **same** function:

```c
// kitty/keys.c:L319   int num = encode_glfw_key_event(&ev, cursor_key_mode, key_encoding_flags, output);
```

Because `kitty/keys.c:L251` and `kitty/keys.c:L319` invoke the identical `encode_glfw_key_event` (`kitty/key_encoding.c:L414`) with the same flags semantics, the headless `encode_key_for_tty` output is **definitionally** the same byte string kitty writes to the child for that key + flags. The real-PTY capture in §7.1 confirms this end-to-end: the encoder bytes, once written to a PTY master, arrive unaltered at a child's stdin.

### 7.3 Labeling: primary vs canonical, and the one un-exercised hop

- **Deterministic primary evidence:** the headless `Screen` + `encode_key_for_tty` reproduction (§4–§6, §9). It drives the *real* VT parser, stack, and buffer-switch code and the *real* encoder; it is fully deterministic (§11).
- **Canonical child-boundary corroboration:** the real forked child over a real PTY (§7.1). This is the actual "bytes transmitted to the child" the user asked about, and it matches the encoder for all six cases.
- **The single un-exercised hop (labeled):** the GLFW windowing event → `encode_glfw_key_event` call at `kitty/keys.c:L251`. Exercising it requires a live GUI window, which cannot launch in this headless environment (no `DISPLAY`/`WAYLAND_DISPLAY`, no `xvfb`; see §2.4). It is covered by the source-equivalence argument in §7.2: the GUI path (`L251`) and the exercised binding (`L319`) call the same encoder. No value here is taken from a bypassing interface (remote control / debug hook); the primary and corroborating paths are both real code paths.

### 7.4 Full-GUI corroboration commands (documented; renderings verified)

In an environment with a display, the authoritative human-visible capture uses the in-repo `show-key` kitten and a `cat -v` child. The `show-key` kitten enters the enhanced protocol when invoked with `-m kitty` — its entry point selects the enhanced loop:

```go
// kittens/show_key/main.go:L14
if opts.KeyMode == "kitty" {
    err = run_kitty_loop(opts)      // kittens/show_key/main.go:L15
} else {
    err = run_legacy_loop(opts)
}
```

The commands to run (documented for a display-enabled host):

```
./kitty/launcher/kitty +kitten show-key -m kitty     # human-readable enhanced protocol; press Ctrl+Shift+a, then a
./kitty/launcher/kitty sh -c 'cat -v'                 # raw rendering; press Ctrl+Shift+a, then a, then Ctrl+a
# query current flags from inside the terminal:  printf '\x1b[?u'  ->  terminal replies  CSI ? <flags> u
```

A child running `cat -v` renders non-printing bytes visibly. Because §7.1 already proves the child receives exactly `\x1b[97;6u`, `\x1b[97u`, and `\x01`, we can render those *exact* captured bytes through real `cat -v` to show what the user would see on screen:

```
$ printf '\x1b[97;6u' | cat -v ; echo   ->   ^[[97;6u      # Ctrl+Shift+a
$ printf '\x1b[97u'   | cat -v ; echo   ->   ^[[97u        # plain 'a' under flag 8
$ printf '\x01'       | cat -v ; echo   ->   ^A            # Ctrl+a (0x01)
```

So on a display-enabled host, pressing `Ctrl+Shift+a` in `cat -v` would print `^[[97;6u`, a plain `a` under flag 8 would print `^[[97u`, and `Ctrl+a` would print `^A` — matching the byte-exact PTY capture in §7.1. (`cat -v` rendering verified programmatically against the exact captured bytes; the interactive keypress-through-GUI step is the one hop that requires a display, per §2.4/§7.3.)


---

## 8. Proof of independence and leakage under rapid switching (Question 6)

**Answer: yes — the captured bytes and flag readings prove the two stacks are independent, and there is no state leakage even under rapid, repeated buffer switching while keyboard mode is manipulated.**

### 8.1 Raw output — rapid-switch leakage probe (repeated ×2)

The probe sets main to `1` and alt to `8`, then rapidly toggles back and forth 6 times, reading the active flags on each side at every round-trip; the whole probe is itself repeated twice.

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 6 block
```

Raw output (complete, unedited):

```
### TEST 6: rapid buffer switching leakage probe (repeat x2) ###
  run1: 6 round-trips (main,alt) readings = [(1, 8), (1, 8), (1, 8), (1, 8), (1, 8), (1, 8)] | no-leakage=True
  run2: 6 round-trips (main,alt) readings = [(1, 8), (1, 8), (1, 8), (1, 8), (1, 8), (1, 8)] | no-leakage=True
```

### 8.2 Reading and grounding

Across all 6 rapid round-trips, and across both repetitions, main reads `1` every time and alt reads `8` every time — the tuple `(1, 8)` never varies. There is **no** leakage: the alt push (`8`) never bleeds into main, and the main push (`1`) never bleeds into alt, no matter how many times or how quickly the buffers are switched. This is the direct behavioral proof of the storage independence described in §3: since `screen_toggle_screen_buffer` only repoints `key_encoding_flags` between the two separate arrays (`kitty/screen.c:L1079`/`L1086`) and never mutates their contents, repeated switching is idempotent with respect to stack state.

### 8.3 The complete proof of independence

Independence is established by two complementary observations, neither of which alone is sufficient but which together are decisive:

1. **The `1 → 0 → 8 → 1` readings (§4).** `current_key_encoding_flags()` reads `1` on main, `0` on freshly-entered alt, `8` after the alt push, and `1` again on return to main. The intermediate `0` proves nothing was copied across the switch; the final `1` proves the main stack survived the alt excursion.
2. **The plain-`a` divergence (§5.4, §9.4).** A plain `a` is `\x1b[97u` under alt's flag 8 but the literal byte `a` under main's flag 1 — i.e. the two buffers genuinely produce *different key bytes* for the same key press, so they truly hold different modes, not merely different numbers in a shared cell.

The modified-key bytes (`Ctrl+Shift+a` = `\x1b[97;6u` everywhere) are deliberately **not** used as the independence proof, because they are identical across flags 0/1/8 (§5.3) and would therefore be silent on the question. The leakage probe (§8.1) then confirms this independence is stable under stress.


---

## 9. Mode / settings dependencies (Question 7)

**Answer: the per-buffer stack isolation does NOT break down under any alternate-screen mode — DECSET 1049, 1047, and 47 all give identical isolation. The one setting that changes the *encoding of a key* (as opposed to the isolation) is the active flags value itself: at `flags==0` (legacy) lock modifiers (Caps/Num Lock) are stripped, whereas at `flags!=0` they are reported. Also note flag 8 implies disambiguate for encoding purposes.**

### 9.1 Raw output — DECSET 1049 vs 1047 vs 47 all isolate identically

The three DECSET modes that switch to the alternate screen differ in cursor/clear semantics but should isolate keyboard flags identically. The probe pushes `1` on main, enters alt via each mode, pushes `8` on alt, then leaves.

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 7 block
```

Raw output (complete, unedited):

```
### TEST 7: DECSET 1049 vs 1047 vs 47 all give independent stacks ###
  DECSET 1049: main=1 enterAlt=0 altPush8=8 backMain=1
  DECSET 1047: main=1 enterAlt=0 altPush8=8 backMain=1
  DECSET 47  : main=1 enterAlt=0 altPush8=8 backMain=1
```

All three modes yield the identical `1 → 0 → 8 → 1` signature: isolation is independent of *which* alternate-screen mode is used.

### 9.2 Grounding — the mode constants and the shared switch

The three alternate-screen DECSET modes are defined in `kitty/modes.h`:

```c
// kitty/modes.h:L75   #define TOGGLE_ALT_SCREEN_1 (47   << 5)
// kitty/modes.h:L76   #define TOGGLE_ALT_SCREEN_2 (1047 << 5)
// kitty/modes.h:L77   #define ALTERNATE_SCREEN    (1049 << 5)
```

All three dispatch to the **same** `screen_toggle_screen_buffer` (`kitty/screen.c:L1068`) at the DECSET dispatch site (`kitty/screen.c:L1165-1169`). The only difference is that mode 1049 (`ALTERNATE_SCREEN`) also saves the cursor and clears the alt screen (the `save_cursor`/`clear_alt_screen` arguments are set to `mode == ALTERNATE_SCREEN`), whereas 1047/47 pass `false, false`. Because the keyboard-flags handling in that function is *only* the pointer repoint (`L1079`/`L1086`), and that repoint is unconditional on cursor/clear behavior, all three modes isolate the keyboard stacks identically. **The isolation therefore does not break down under any alternate-screen mode.**

### 9.3 Raw output — the one setting that changes encoding: legacy lock-strip

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 8 block
```

Raw output (complete, unedited):

```
### TEST 8: legacy lock-strip (key_encoding.c:L36) flags==0 vs !=0 ###
  Caps+a  flags=0 -> b'a'  hex=61
  Caps+a  flags=1 -> b'\x1b[97;65u'  hex=1b 5b 39 37 3b 36 35 75
```

Reading and grounding: with `flags==0` (legacy), pressing `a` while Caps Lock is on sends the literal byte `a` — the lock modifier is stripped. With `flags==1`, the same key press reports the lock modifier: `\x1b[97;65u`, where `65 = 1 + 64` and `64` is the Caps Lock bit. The cause is a single line in `convert_glfw_mods`:

```c
// kitty/key_encoding.c:L36   if (!key_encoding_flags) mods &= ~GLFW_LOCK_MASK;
```

When `key_encoding_flags == 0`, the lock bits (`CAPS_LOCK = 64`, `NUM_LOCK = 128`; masks at `kitty/key_encoding.c:L11`) are masked off; when any enhancement flag is active they are retained and encoded (as `value + 1`, `kitty/key_encoding.c:L47`). This is the one setting under which the *encoding* of a given physical key press differs — and it is a function of the flags value, i.e. of the per-buffer stack state, not of anything global. It does not compromise isolation; it is simply the flags value doing its job on each buffer independently. The published spec notes the same: lock modifiers are not reported for text-producing keys in the default mode, and one uses report-all-keys to get lock modifiers for all keys (`https://sw.kovidgoyal.net/kitty/keyboard-protocol/`).

### 9.4 Raw output — single-flag divergence for a plain key (flag 8 is special)

Command:

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py     # TEST 9 block
```

Raw output (complete, unedited):

```
### TEST 9: single-flag divergence for plain 'a' (bits 1,2,4,8,16) ###
  flags= 0 plain 'a' -> b'a'  hex=61
  flags= 1 plain 'a' -> b'a'  hex=61
  flags= 2 plain 'a' -> b'a'  hex=61
  flags= 4 plain 'a' -> b'a'  hex=61
  flags= 8 plain 'a' -> b'\x1b[97u'  hex=1b 5b 39 37 75
  flags=16 plain 'a' -> b'a'  hex=61
```

Reading and grounding: of all five single enhancement bits, **only** flag `8` (report-all-keys) turns a plain, unmodified `a` into a CSI-u escape (`\x1b[97u`); flags `1`, `2`, `4`, and `16` all leave a bare printable as its literal byte `a`. This is what makes flag 8 the observable "tell" that distinguishes alt's mode from main's in §4/§5/§8. In the encoder, flag 8 sets the field named `report_text` (`kitty/key_encoding.c:L422`) which triggers the early `if (ev->report_text) return serialize(&ed, output, 'u');` branch (`kitty/key_encoding.c:L376`) for *every* key. (Regarding "flag 8 implies disambiguate": a plain key becomes CSI-u under flag 8, and a modified key is already CSI-u under both flags 1 and 8 — so anything disambiguate would escape-code, report-all-keys also escape-codes, and more; the published spec states flag 8 makes every key an escape code, superseding the plain-text shortcut that flag 1 alone still permits.)

### 9.5 Summary of dependencies

- **Alternate-screen mode (1049 / 1047 / 47):** no effect on isolation — identical `1 → 0 → 8 → 1` for all three (§9.1). Cursor-save/clear differs, keyboard-flags isolation does not.
- **Active flags value (per buffer):** determines the key encoding. The notable threshold is `flags==0` vs `flags!=0`, which toggles legacy lock-modifier stripping (§9.3). This is a per-buffer property, so it too respects isolation.
- **No mode was found under which isolation breaks down.** Every alternate-screen entry mechanism keeps the two stacks separate.


---

## 10. Two-stack disambiguation — do not conflate the two subsystems

kitty has **two** unrelated "keyboard mode stacks," and this investigation concerns only the first:

1. **In scope — the progressive-enhancement flags stack (the subject of this document).** This is the `CSI > flags u` push / `CSI < number u` pop stack, held **per screen buffer**, implemented entirely in C. Its storage is the `main_key_encoding_flags[8]` / `alt_key_encoding_flags[8]` arrays plus the active pointer at `kitty/screen.h:L128`; its operations are `screen_push_key_encoding_flags` (`kitty/screen.c:L1234`), `screen_pop_key_encoding_flags` (`kitty/screen.c:L1248`), `screen_set_key_encoding_flags` (`kitty/screen.c:L1220`), and `screen_current_key_encoding_flags` (`kitty/screen.c:L1204`); it is dispatched by the VT parser at `kitty/vt-parser.c:L1217` (push `>` → `L1233`, pop `<` → `L1237`, set `=` → `L1229`, query `?` → `L1224-1225`). This is what the user's application manipulates when it "pushes its own keyboard-enhancement flags."

2. **Out of scope — the `kitty.conf` keyboard-*mapping* mode stack.** This is a Python-side stack, `keyboard_mode_stack` (`kitty/keys.py:L67`), driven by `pop_keyboard_mode` (`kitty/keys.py:L93`) and `_push_keyboard_mode` (`kitty/keys.py:L108`), which implements user-configured multi-key *mappings* (e.g. leader-key sequences defined in `kitty.conf`). It is **not** per-screen-buffer and has nothing to do with the CSI-u wire protocol.

These are entirely separate subsystems: different language (C vs Python), different storage, different lifecycle, different purpose. The question — how flags behave when switching between main and alternate buffers — is exclusively about subsystem (1). Everything in §1–§9 is about the C per-buffer progressive-enhancement stack. Subsystem (2) is named here only to prevent conflation.

---

## 11. Determinism and the resolution of the "order-dependent" observation

**Every scenario was run repeatedly and produced byte-identical output. The behavior is fully deterministic; the user's report that pushing after switching behaves differently from pushing before switching is REAL and EXPECTED — it is the deterministic consequence of per-buffer stack independence, not nondeterminism.**

### 11.1 Repetition and hashes

Each script's complete stdout was captured and hashed across repeated identical runs:

| Script | Scenarios | Repetitions | `md5sum` of stdout | Result |
|--------|-----------|-------------|--------------------|--------|
| `/tmp/kbd_probe.py` | TEST 1–9 (round-trip repeated ×3, leakage ×2 internally) | 3 full runs | `c97899b427edf53a48b465609a6251e1` | identical every run |
| `/tmp/kbd_legacy.py` | modifier matrix, flags 0 & 1 | 2 full runs | `287f2b8a82ccb4c34c9e0ab5a293eeca` | identical every run |
| `/tmp/kbd_capacity.py` | capacity K=1..12 | 2 full runs | `96d2ccf2d8e0babbde504024e9812454` | identical every run |
| `/tmp/kbd_pty.py` | real-PTY capture, 6 cases | 3 full runs | `d957420e7f562331ac7d908c50252632` | identical every run |

There is zero run-to-run variance. No scenario exhibited a "sometimes X, sometimes Y" distribution, so no distribution needs to be reported — the single observed value for each case is stable.

### 11.2 Resolving the user's "order-dependent" report

The user observed that "pushing flags *after* switching to the alternate buffer behaves differently from pushing on main first and *then* switching." That observation is **correct**, and it is fully explained — deterministically — by per-buffer independence:

- "Push on main, then switch to alt": the push lands on `main_key_encoding_flags`; after switching, the active pointer is `alt_key_encoding_flags`, whose stack is independent and (initially) empty — so the pushed flags are *not* active on alt. (This is the `1 → 0` step of §4.)
- "Switch to alt, then push": the push lands on `alt_key_encoding_flags` and *is* active on alt, but leaves `main_key_encoding_flags` untouched — so it is invisible after switching back to main. (This is the `8 → 1` step of §4.)

So the two orders genuinely differ, but the difference is *which stack the push modifies*, and that is completely deterministic. It is not a race or an intermittent glitch; it is the intended design (§3, spec §3.4). The right mental model for the user's own application: push your enhancement flags **on the buffer you intend to use them on**. If the application lives on the alternate screen, push after switching (or push on both), because a push performed on main will not carry over to alt.


---

## 12. Appendix

### 12.1 Full citation map (verified at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)

**Storage — `kitty/screen.h`:**
- `L128` — `uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;` — two separate 8-slot arrays + active pointer; **per-buffer capacity = 8**.
- `L269-273` — prototypes for set/push/pop/current/report.

**Operations and buffer switch — `kitty/screen.c`:**
- `L150` — init: `key_encoding_flags = main_key_encoding_flags`.
- `L173-174` — reset memsets **both** arrays.
- `L1068` — `screen_toggle_screen_buffer`; `L1069` `to_alt = linebuf == main_linebuf`.
- `L1079` — entering alt: repoint `→ alt_key_encoding_flags`.
- `L1086` — leaving alt: repoint `→ main_key_encoding_flags`. (**never copies/clears — the causal mechanism for independence + round-trip survival**.)
- `L1165-1169` — DECSET dispatch; only `ALTERNATE_SCREEN` (1049) saves cursor + clears; 1047/47 pass `false,false`; keyboard-flags isolation identical for all.
- `L1204` — `screen_current_key_encoding_flags`; scans active array top-down; occupied bit `0x80`, value low 7 bits `0x7f`; `L1208` returns `0` when empty.
- `L1220` — `screen_set_key_encoding_flags` (how==1 set / 2 OR / 3 AND-NOT).
- `L1234` — `screen_push_key_encoding_flags`; `L1235` `q = val & 0x7f`; `L1236` `sz = arraysz(...) == 8`; `L1238-1239` find top idx; **`L1241` `memmove` shift-down = silent evict-oldest**; `L1242` else set occupied bit + advance; `L1243` write new top.
- `L1248` — `screen_pop_key_encoding_flags`; clears `num` occupied slots top-down; over-pop → all-zero → reset.
- `L3951` — Python binding `Screen.current_key_encoding_flags()`.
- `L4449` — Python binding `Screen.toggle_alt_screen()`; `L4451` calls `screen_toggle_screen_buffer(self, true, true)` (DECSET-1049 semantics).

**VT-parser CSI-u dispatch — `kitty/vt-parser.c`:** `L1217` `case 'u'`; `L1224-1225` `?` → report; `L1229` `=` → set(0,1); `L1233` `>` → push(0) [default value 0]; `L1237` `<` → pop(1) [default number 1]; `~L1240` error.

**Encoder — `kitty/keys.c`:** `L251` live keypress path calls `encode_glfw_key_event(..., screen_current_key_encoding_flags(screen), ...)`; `L259` `schedule_write_to_child(...)`; `L311` `pyencode_key_for_tty` (the `defines.encode_key_for_tty` binding); `L319` calls the **same** `encode_glfw_key_event`; `L334` registers `encode_key_for_tty`. **`L251` ≡ `L319` is why headless encoder output = real PTY bytes.**

**Encoding — `kitty/key_encoding.c`:** `L11` masks `SHIFT=1, ALT=2, CTRL=4, SUPER=8, HYPER=16, META=32, CAPS_LOCK=64, NUM_LOCK=128`; `L35` `convert_glfw_mods`; `L36` `if (!key_encoding_flags) mods &= ~GLFW_LOCK_MASK` (legacy lock-strip); `L47` modifier encoded as `value + 1` (Ctrl+Shift = 5 → `6`); `L152` `legacy_mode = !report_all_event_types && !disambiguate`; `L292` `encode_printable_ascii_key_legacy`; `L294` `mods==0` → literal char; `L309` `mods==CTRL` → control byte (`Ctrl+a` → `0x01`); `L314` `CTRL|SHIFT` branch fires only for space; `L317` `return 0` (no `CTRL|SHIFT`+letter branch); `L367` `encode_key`; `L376` `if (report_text) return serialize(..., 'u')` (flag 8 → CSI-u for every key); `L396` fall-through `return serialize(..., 'u')` (→ `\x1b[97;6u` for `Ctrl+Shift+a`); `L414` `encode_glfw_key_event`; `L419-423` flag→field mapping.

**Alternate-screen mode constants — `kitty/modes.h`:** `L75` `TOGGLE_ALT_SCREEN_1 (47<<5)`; `L76` `TOGGLE_ALT_SCREEN_2 (1047<<5)`; `L77` `ALTERNATE_SCREEN (1049<<5)`.

**Spec — `docs/keyboard-protocol.rst`:** `L186` "1 + 0b101" for ctrl+shift (=6); `L275-282` flag bit table (1 disambiguate / 2 report-event-types / 4 report-alternate-keys / 8 report-all-keys / 16 report-associated-text); `L293-297` push `CSI > flags u` (default 0) / pop `CSI < number u` (default 1); `L299-303` separate stacks, pop-empties → reset, push-full → evict oldest; `L306-309` main/alt own independent stacks.

**Harness — `kitty_tests/`:** `__init__.py:L30` `parse_bytes`; `L208` `BaseTest`; `L237` `create_screen`; `L243` `create_pty`; `L277` `class PTY` (forks a real child). `keys.py:L16` `enc = defines.encode_key_for_tty`; `L22` `csi(...)`; `L407` `Ctrl+Shift+i` @flags0 → `\x1b[105;6u` (confirms the CSI-u form); `L412` plain `a` @flags0 → `a`; `L454-455` plain `a` @flag8 → `\x1b[97u`; `L457` `Ctrl+a` @flag8 → `\x1b[97;5u`. `screen.py:L501,L503` `toggle_alt_screen()` / `parse_bytes` usage pattern.

**Out-of-scope mapping stack (do not conflate) — `kitty/keys.py`:** `L67` `keyboard_mode_stack`; `L93` `pop_keyboard_mode`; `L108` `_push_keyboard_mode`.

**`show-key` kitten — `kittens/show_key/main.go`:** `L14` `if opts.KeyMode == "kitty"` → `L15` `run_kitty_loop(opts)` (enters the enhanced protocol under `-m kitty`).

### 12.2 The C-field naming inversion (call-out)

A reader diffing the C source against the spec should note an internal naming inversion in the flag→field mapping at `kitty/key_encoding.c:L419-423`:

- The field set by flag `0b1000` (**8**) is named `report_text` — but per the spec, flag 8 is "**report all keys as escape codes**."
- The field set by flag `0b10000` (**16**) is named `embed_text` — this is the spec's "report associated text."

So kitty's internal `report_text` field is the spec's *report-all-keys-as-escape-codes*, and the observable behavior is spec-correct (verified by the runtime captures in §5/§9 and by `kitty_tests/keys.py:L454-457`). Only the internal identifier is a misnomer; it does not affect behavior.

### 12.3 Web-search corroboration (copyright-safe paraphrase)

The in-repo `docs/keyboard-protocol.rst` was cross-checked against the canonical published specification at `https://sw.kovidgoyal.net/kitty/keyboard-protocol/` and the originating RFC `kovidgoyal/kitty#3248` (2021). The published spec corroborates every runtime finding with no contradictions:

- **Independent per-screen stacks:** the spec requires the main and alternate screens to maintain their own independent keyboard mode stacks, so a program on the alternate screen can change mode without affecting or knowing the main screen's mode — confirming §3/§4/§8.
- **Push-full evicts oldest; pop-empty resets:** the spec states a push onto a full stack evicts the oldest entry and a pop that empties the stack resets all flags — confirming §6.
- **Stack size is implementation-defined:** the spec prescribes no numeric depth — it says only that terminals should limit the stack size appropriately to prevent denial-of-service (`docs/keyboard-protocol.rst:L299-300`). kitty's exact capacity of 8 is fixed by the source (`kitty/screen.h:L128`) and confirmed at runtime (§6.2/§6.3), not by the spec.
- **Modifier value = 1 + bitmask; Ctrl+Shift = 6:** confirming §5's modifier encoding.
- **Flag 8 = all keys as escape codes; flag 1 alone still sends a plain letter literally:** the spec says report-all-keys makes every key — including plain printables — a CSI-u sequence, while under disambiguate alone a plain letter like `a` still sends `0x61` — confirming §5.4/§9.4.
- **Wire forms:** push `CSI > flags u`, pop `CSI < number u`, query `CSI ? u` → reply `CSI ? flags u` — confirming §10's dispatch citations.
- **`Ctrl+Shift+a` = `CSI 97;6`:** in RFC #3248 the protocol author states, of encoding ctrl+shift+a as `CSI 97;6` vs `CSI 65;5`, "the former is correct" — directly confirming the observed `\x1b[97;6u` and the correction in §5.3.
- **`show-key -m kitty`** is the documented debugging tool for the protocol — confirming §7.4.

### 12.4 Temporary observation scripts (transient; deleted; repository left unchanged)

Four temporary scripts were written under the container's `/tmp` (outside the repository tree), run from the repo root, and deleted after the outputs above were captured. They are listed here for reproducibility; none is committed.

- `/tmp/kbd_probe.py` — instantiates a real headless `Screen` via `kitty_tests.BaseTest.create_screen`, drives it with raw escape sequences through `parse_bytes`, and reads `current_key_encoding_flags()`; runs TEST 1–9 (round-trip; per-state bytes; overflow; cross-buffer; pop-to-empty; rapid-switch leakage; DECSET 1049/1047/47; legacy lock-strip; single-flag divergence).
- `/tmp/kbd_legacy.py` — calls `encode_key_for_tty(key=ord('a'), mods=..., key_encoding_flags=0|1)` across the modifier matrix.
- `/tmp/kbd_capacity.py` — pushes K=1..12 distinct sentinel values and unwinds to measure retained capacity.
- `/tmp/kbd_pty.py` — forks a real child over a real PTY (`kitty.child.openpty`), writes the encoder bytes to the master, and confirms the child reads back byte-identical output for all six cases.

The four scripts are reproduced below **in full — no logic is elided**. Each was written under the container's `/tmp` (outside the repository tree), run from the repo root (`/workspace`, where the built `kitty.fast_data_types` extension lives), and deleted afterward. Running them at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` reproduces every raw-output block quoted above byte-for-byte (verified across repeated runs).

**`/tmp/kbd_probe.py`**

```python
#!/usr/bin/env python3
# kbd_probe.py — headless runtime probe of kitty's per-buffer keyboard-protocol
# progressive-enhancement flags stack (CSI > u push / CSI < u pop) across the
# main<->alternate screen-buffer switch. Exercises TESTS 1-9.
#
# Run from the kitty repo root inside the canonical build container:
#   PYTHONPATH=/workspace python3 /tmp/kbd_probe.py
#
# Primitives (all real code paths, no bypass):
#   kitty_tests.BaseTest.create_screen() -> a real headless Screen object
#   kitty_tests.parse_bytes(screen, data) -> drives the real VT parser
#   screen.toggle_alt_screen()            -> real screen_toggle_screen_buffer
#   screen.current_key_encoding_flags()   -> real screen_current_key_encoding_flags
#   fast_data_types.encode_key_for_tty()  -> the SAME encoder kitty writes to the PTY
import sys
sys.path.insert(0, '.')
from kitty_tests import BaseTest, parse_bytes
import kitty.fast_data_types as fdt

bt = BaseTest()
CTRL = fdt.GLFW_MOD_CONTROL      # 4
SHIFT = fdt.GLFW_MOD_SHIFT       # 1
CAPS = fdt.GLFW_MOD_CAPS_LOCK    # 64


def enc_bytes(key, mods, flags):
    """Encode one key event exactly as kitty would for the child, return raw bytes."""
    out = fdt.encode_key_for_tty(key=key, mods=mods, key_encoding_flags=flags)
    return out.encode('latin-1') if isinstance(out, str) else bytes(out)


def hx(data):
    return ' '.join('%02x' % b for b in data)


def push(screen, flags):
    """Push flags onto the active buffer's stack via a real CSI > flags u sequence."""
    parse_bytes(screen, ('\x1b[>%du' % flags).encode('latin-1'))


def pop(screen, n=1):
    """Pop n entries from the active buffer's stack via a real CSI < n u sequence."""
    parse_bytes(screen, ('\x1b[<%du' % n).encode('latin-1'))


# ---- TEST 1: main->alt->main round-trip, repeated x3 on a fresh Screen each run ----
print("### TEST 1: main->alt->main round-trip (repeat x3 for stability) ###")
for run in (1, 2, 3):
    s = bt.create_screen()
    a = s.current_key_encoding_flags()          # fresh main: empty -> 0
    push(s, 1); b = s.current_key_encoding_flags()   # push disambiguate on main -> 1
    s.toggle_alt_screen(); c = s.current_key_encoding_flags()  # enter alt: own empty stack -> 0
    push(s, 8); d = s.current_key_encoding_flags()   # push report-all-keys on alt -> 8
    s.toggle_alt_screen(); e = s.current_key_encoding_flags()  # back to main: survived -> 1
    print(" run%d: start-main-empty=%d | push>1u-main=%d | toggled-ALT-prepush=%d | push>8u-alt=%d | toggled-back-MAIN=%d" % (run, a, b, c, d, e))

# ---- TEST 2: per-state bytes for Ctrl+Shift+a AND plain 'a' in the four named states ----
print("### TEST 2: per-state bytes for Ctrl+Shift+a AND plain 'a' ###")
s = bt.create_screen()
states = []
states.append(("A: main, no push (legacy)", s.current_key_encoding_flags()))
push(s, 1)
states.append(("B: main, after push >1u (disambiguate)", s.current_key_encoding_flags()))
s.toggle_alt_screen(); push(s, 8)
states.append(("C: alt, after push >8u (report-all-keys)", s.current_key_encoding_flags()))
s.toggle_alt_screen()
states.append(("D: back on main (should equal B)", s.current_key_encoding_flags()))
for label, fl in states:
    cs = enc_bytes(ord('a'), CTRL | SHIFT, fl)
    pa = enc_bytes(ord('a'), 0, fl)
    print("  [%s] flags=%d" % (label, fl))
    print("        Ctrl+Shift+a  -> %r  hex=%s" % (cs, hx(cs)))
    print("        plain 'a'     -> %r  hex=%s" % (pa, hx(pa)))

# ---- TEST 3: overflow / eviction — push 12 distinct values on MAIN, then unwind ----
print("### TEST 3: overflow / eviction (push >8 distinct values on MAIN) ###")
s = bt.create_screen()
tracks = []
for v in range(1, 13):
    push(s, v)
    tracks.append(s.current_key_encoding_flags())
print("    pushed >1u..>12u -> current tracks %s" % tracks)
unwind = []
for _ in range(10):
    unwind.append(s.current_key_encoding_flags())
    pop(s, 1)
print("    unwind sequence of current values (read then CSI<1u each): %s" % unwind)

# ---- TEST 4: overflowing MAIN's stack does NOT affect ALT's stack ----
print("### TEST 4: overflow on MAIN does NOT affect ALT ###")
s = bt.create_screen()
for v in range(1, 13):
    push(s, v)
m1 = s.current_key_encoding_flags()
s.toggle_alt_screen(); alt = s.current_key_encoding_flags()
s.toggle_alt_screen(); m2 = s.current_key_encoding_flags()
print("  after 12 pushes on main, main current = %d | alt current (never pushed) = %d | main current again = %d" % (m1, alt, m2))

# ---- TEST 5: popping past the bottom empties the stack -> all flags reset to 0 ----
print("### TEST 5: pop-to-empty resets all flags (current -> 0) ###")
s = bt.create_screen()
push(s, 5); f5 = s.current_key_encoding_flags()
push(s, 7); f7 = s.current_key_encoding_flags()
pop(s, 9); f0 = s.current_key_encoding_flags()
print("  after push >5u=%d; after push >7u=%d; after pop <9u(over-pop) current = %d" % (f5, f7, f0))

# ---- TEST 6: rapid buffer switching leakage probe (main=1, alt=8), repeated x2 ----
print("### TEST 6: rapid buffer switching leakage probe (repeat x2) ###")
for run in (1, 2):
    s = bt.create_screen()
    push(s, 1)                                   # main = 1
    s.toggle_alt_screen(); push(s, 8)            # alt  = 8
    s.toggle_alt_screen()                        # back on main
    readings = []
    for _ in range(6):
        m = s.current_key_encoding_flags()       # read on main
        s.toggle_alt_screen(); al = s.current_key_encoding_flags()  # read on alt
        s.toggle_alt_screen()                    # back on main
        readings.append((m, al))
    noleak = all(r == (1, 8) for r in readings)
    print("  run%d: 6 round-trips (main,alt) readings = %s | no-leakage=%s" % (run, readings, noleak))

# ---- TEST 7: the three alternate-screen DECSET modes all isolate identically ----
print("### TEST 7: DECSET 1049 vs 1047 vs 47 all give independent stacks ###")
for enter, leave, label in [
    (b"\x1b[?1049h", b"\x1b[?1049l", "DECSET 1049"),
    (b"\x1b[?1047h", b"\x1b[?1047l", "DECSET 1047"),
    (b"\x1b[?47h",   b"\x1b[?47l",   "DECSET 47  "),
]:
    s = bt.create_screen()
    push(s, 1); m = s.current_key_encoding_flags()
    parse_bytes(s, enter); ea = s.current_key_encoding_flags()
    push(s, 8); ap = s.current_key_encoding_flags()
    parse_bytes(s, leave); bm = s.current_key_encoding_flags()
    print("  %s: main=%d enterAlt=%d altPush8=%d backMain=%d" % (label, m, ea, ap, bm))

# ---- TEST 8: legacy lock-strip — Caps+a at flags==0 vs flags!=0 ----
print("### TEST 8: legacy lock-strip (key_encoding.c:L36) flags==0 vs !=0 ###")
c0 = enc_bytes(ord('a'), CAPS, 0)
c1 = enc_bytes(ord('a'), CAPS, 1)
print("  Caps+a  flags=0 -> %r  hex=%s" % (c0, hx(c0)))
print("  Caps+a  flags=1 -> %r  hex=%s" % (c1, hx(c1)))

# ---- TEST 9: single-flag divergence for a plain 'a' across bits 1,2,4,8,16 ----
print("### TEST 9: single-flag divergence for plain 'a' (bits 1,2,4,8,16) ###")
for fl in (0, 1, 2, 4, 8, 16):
    d = enc_bytes(ord('a'), 0, fl)
    print("  flags=%2d plain 'a' -> %r  hex=%s" % (fl, d, hx(d)))
```

**`/tmp/kbd_legacy.py`**

```python
#!/usr/bin/env python3
# kbd_legacy.py — full legacy(flags=0) vs disambiguate(flags=1) modifier matrix
# for the 'a' key, encoded with the SAME encoder kitty writes to the child.
# Shows precisely where 0x01 (Ctrl+a) comes from and how the two regimes differ.
#
#   PYTHONPATH=/workspace python3 /tmp/kbd_legacy.py
import sys
sys.path.insert(0, '.')
import kitty.fast_data_types as fdt

CTRL = fdt.GLFW_MOD_CONTROL      # 4
SHIFT = fdt.GLFW_MOD_SHIFT       # 1
ALT = fdt.GLFW_MOD_ALT           # 2


def enc_bytes(key, mods, flags):
    out = fdt.encode_key_for_tty(key=key, mods=mods, key_encoding_flags=flags)
    return out.encode('latin-1') if isinstance(out, str) else bytes(out)


def hx(data):
    return ' '.join('%02x' % b for b in data)


matrix = [
    ("a", 0),
    ("Ctrl+a", CTRL),
    ("Shift+a", SHIFT),
    ("Ctrl+Shift+a", CTRL | SHIFT),
    ("Alt+a", ALT),
    ("Ctrl+Alt+a", CTRL | ALT),
]

for flags, title in [
    (0, "legacy(flags=0) modifier matrix for 'a'"),
    (1, "disambiguate(flags=1) same matrix"),
]:
    print("=== %s ===" % title)
    for name, mods in matrix:
        d = enc_bytes(ord('a'), mods, flags)
        print("  flags=%d %s-> %r hex=%s" % (flags, name.ljust(15), d, hx(d)))
```

**`/tmp/kbd_capacity.py`**

```python
#!/usr/bin/env python3
# kbd_capacity.py — measure the exact per-buffer stack capacity by pushing K
# distinct sentinel values (K = 1..12) onto one buffer's stack, then unwinding
# (read current, pop one) K+2 times and counting how many distinct non-zero
# values were retained. Proves capacity = 8 (push-full evicts the oldest).
#
#   PYTHONPATH=/workspace python3 /tmp/kbd_capacity.py
import sys
sys.path.insert(0, '.')
from kitty_tests import BaseTest, parse_bytes

bt = BaseTest()


def push(screen, flags):
    parse_bytes(screen, ('\x1b[>%du' % flags).encode('latin-1'))


def pop(screen, n=1):
    parse_bytes(screen, ('\x1b[<%du' % n).encode('latin-1'))


print("### capacity measurement: push K distinct values on one buffer, then read-before-pop (K+2) times ###")
for K in range(1, 13):
    s = bt.create_screen()
    for v in range(1, K + 1):
        push(s, v)                                   # push sentinel value v
    top = s.current_key_encoding_flags()             # top of stack == last pushed
    unwind = []
    for _ in range(K + 2):
        unwind.append(s.current_key_encoding_flags())  # read current
        pop(s, 1)                                       # pop one entry
    distinct_retained = len(set(v for v in unwind if v != 0))
    print("  K=%2d: top=%2d  unwind=%-40s distinct_retained=%d" % (K, top, str(unwind), distinct_retained))
```

**`/tmp/kbd_pty.py`**

```python
#!/usr/bin/env python3
# kbd_pty.py — canonical child-boundary capture. For each key event, forks a
# real child whose stdin is a real PTY slave (from kitty.child.openpty, the same
# openpty kitty uses for its child PTYs), puts the slave in raw mode, has the
# parent write the encoder bytes to the master, and captures what the child
# actually read()s. Confirms the encoder output == the bytes the child receives.
#
#   PYTHONPATH=/workspace python3 /tmp/kbd_pty.py
import os
import sys
import termios
sys.path.insert(0, '.')
import kitty.fast_data_types as fdt
from kitty.child import openpty

CTRL = fdt.GLFW_MOD_CONTROL      # 4
SHIFT = fdt.GLFW_MOD_SHIFT       # 1


def enc_bytes(key, mods, flags):
    out = fdt.encode_key_for_tty(key=key, mods=mods, key_encoding_flags=flags)
    return out.encode('latin-1') if isinstance(out, str) else bytes(out)


def hx(data):
    return ' '.join('%02x' % b for b in data)


def capture_at_child(payload):
    """Fork a child reading from a real PTY slave in raw mode; return the exact
    bytes it read after the parent writes `payload` to the master."""
    master, slave = openpty()
    r, w = os.pipe()                       # child -> parent channel for the read bytes
    pid = os.fork()
    if pid == 0:                           # ---- child ----
        os.close(master)
        os.close(r)
        os.dup2(slave, 0)                  # child's stdin IS the PTY slave
        attrs = termios.tcgetattr(0)       # put the tty in raw mode so bytes pass through untouched
        try:
            import tty
            tty.cfmakeraw(attrs)
        except AttributeError:             # cfmakeraw added to `tty` in 3.12; fall back to termios flags
            attrs[0] = 0                   # iflag
            attrs[1] = 0                   # oflag
            attrs[3] = 0                   # lflag (no ICANON/ECHO/ISIG/IEXTEN)
        attrs[6][termios.VMIN] = 0
        attrs[6][termios.VTIME] = 5        # 0.5s idle window terminates the read
        termios.tcsetattr(0, termios.TCSANOW, attrs)
        buf = b''
        while True:
            chunk = os.read(0, 64)
            if not chunk:                  # idle timeout with no more data -> done
                break
            buf += chunk
        os.write(w, buf)
        os.close(w)
        os._exit(0)
    # ---- parent ----
    os.close(slave)
    os.close(w)
    os.write(master, payload)              # parent writes the encoder bytes to the master
    got = b''
    while True:
        chunk = os.read(r, 64)
        if not chunk:
            break
        got += chunk
    os.close(r)
    os.close(master)
    os.waitpid(pid, 0)
    return got


cases = [
    ("Ctrl+Shift+a", ord('a'), CTRL | SHIFT, 0),
    ("Ctrl+Shift+a", ord('a'), CTRL | SHIFT, 1),
    ("Ctrl+Shift+a", ord('a'), CTRL | SHIFT, 8),
    ("plain 'a'",    ord('a'), 0, 1),
    ("plain 'a'",    ord('a'), 0, 8),
    ("Ctrl+a",       ord('a'), CTRL, 0),
]

all_match = True
for name, key, mods, flags in cases:
    enc = enc_bytes(key, mods, flags)
    child = capture_at_child(enc)
    match = (child == enc)
    all_match = all_match and match
    label = name.ljust(12) + " flags=%d" % flags
    print("  [%s]  encoder -> %r (%s) ; child -> %r (%s) ; MATCH=%s" % (
        label, enc, hx(enc), child, hx(child), match))
print("ALL MATCH: %s" % all_match)
```

Invocation and cleanup (run from the repo root inside the build container, then delete the scripts so the repository is left unchanged):

```
PYTHONPATH=/workspace python3 /tmp/kbd_probe.py
PYTHONPATH=/workspace python3 /tmp/kbd_legacy.py
PYTHONPATH=/workspace python3 /tmp/kbd_capacity.py
PYTHONPATH=/workspace python3 /tmp/kbd_pty.py
rm -f /tmp/kbd_probe.py /tmp/kbd_legacy.py /tmp/kbd_capacity.py /tmp/kbd_pty.py
```

After deletion, `git status --porcelain` shows no modified tracked files (build artifacts such as `*.so` and `/kitty/launcher/kitt*` are gitignored); the only new file in the repository is this document, `blitzy/documentation/kitty_815df1e210e0.md`.

