# How kitty's Two-Tier History/Scrollback Behaves Under a Flood

## Summary

When a command floods a kitty terminal with an enormous amount of text in a very short interval, the scrollback is absorbed by a **two-tier** subsystem implemented in `kitty/history.c`. **Tier 1** is the interactively-scrollable, segmented in-memory buffer `HistoryBuf` `[kitty/data-types.h:283-290]`, which grows one 2048-line segment at a time (`SEGMENT_SIZE = 2048` `[kitty/history.c:15]`) until it reaches its line cap `ynum`, then evicts its oldest line on every further push. **Tier 2** is the pager-style ring buffer `PagerHistoryBuf` `[kitty/data-types.h:268-272]` — **disabled by default** (`scrollback_pager_history_size = 0` `[kitty/options/definition.py:406]`) — which, when enabled, receives each evicted line serialized to an ANSI byte stream, growing in ≥1 MB steps up to its `maximum_size` and then overwriting its oldest bytes in FIFO fashion. The "quiet interaction" between the tiers happens entirely inside `historybuf_push` `[kitty/history.c:276-286]`: once Tier 1 is full (`count == ynum`), the oldest line is spilled into Tier 2 via `pagerhist_push` before `start_of_data` advances. This document answers five sub-questions (R1–R5) from **actual runtime observation** of the real `HistoryBuf`/`Screen` objects, with every claim backed by captured output and a `file:line` reference.

---

## (a) Environment, build & methodology

### Build

The kitty C extension `kitty/fast_data_types` must be compiled before any observation is possible; it exposes `HistoryBuf`, `Screen`, and `LineBuf` to Python. The canonical build command is:

```
python3 setup.py build
```

(equivalently `make`), which compiles `kitty/fast_data_types*.so` via `build()` `[setup.py:1084]` → `compile_c_extension(..., 'kitty/fast_data_types', ...)` `[setup.py:1091]`.

**Build caveat (Observed).** On this environment's toolchain (Ubuntu 25.10, gcc 15.2.0, `wayland-protocols` 1.45), the plain `python3 setup.py build` aborts (exit 1) because the vendored glfw/Wayland backend `glfw/wl_window.c:668` hits `-Werror=switch` on the newer `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that its `switch` does not handle. The `fast_data_types` extension itself compiles cleanly; only the unrelated glfw/Wayland windowing step is affected. Complete, unedited tail of the failed plain build (command: `python3 setup.py build`):

```
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
```

The working build command used was:

```
python3 setup.py build --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag `[setup.py:2003]` empties the `werror` string `[setup.py:491]` (`werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'`). Note (Observed): it removes **both** `-pedantic-errors` **and** `-Werror` — visible in the failing compile line above, which is invoked with `... -std=c11 -pedantic-errors -Werror -O3 ...`. It only affects which warnings are promoted to errors; it does not change the source that is compiled. This build succeeds (exit 0) and the extension links and imports cleanly. Complete, unedited output of the import + extension-size check:

```
$ PYTHONPATH=$PWD python3 -c "import kitty.fast_data_types as f; print('import OK:', bool(f.HistoryBuf)); import os; print('size_bytes:', os.path.getsize('kitty/fast_data_types.so'))"
import OK: True
size_bytes: 1253792
```

The windowing build issue is a host-toolchain / newer-headers artifact, unrelated to the scrollback subsystem under test; headless `HistoryBuf`/`Screen` observation needs no GPU/display.

### Canonical entry point, zero bypass

All observations drive the real `HistoryBuf`/`Screen` objects imported from `kitty.fast_data_types`, mirroring the existing tests (`test_historybuf` `[kitty_tests/datatypes.py:487-540]` constructs `HistoryBuf(3000, 5)` and pushes 3000 lines across the 2048 boundary; `filled_line_buf`/`filled_cursor`/`filled_history_buf`/`create_screen` `[kitty_tests/__init__.py:166,175,184,237-240]`). **No remote-control, debug hook, mock, or synthetic stand-in was used.**

### Constructor signature

`HistoryBuf(ynum, xnum[, pagerhist_sz])` — **`ynum` FIRST** (the C `new_history_object` `[kitty/history.c:136-138]` parses `"II|I"` → `ynum, xnum, pagerhist_sz`). The optional third arg `pagerhist_sz` is the ring buffer's `maximum_size` **in raw bytes** at the C level (passed straight to `alloc_pagerhist` `[kitty/history.c:69-80]`); the MB→bytes conversion (`int(max(0, float(x)) * 1024 * 1024)`) applies only on the options path `[kitty/options/utils.py:564-565]`. Python-visible signals: `hb.push(line)` (takes a `Line` object `[kitty/history.c:337]`), `hb.count`, `hb.line(i)` (reverse-indexed, line 0 = newest), `hb.pagerhist_as_text()`, `hb.pagerhist_as_bytes()`, `hb.pagerhist_write(bytes)`, `hb.pagerhist_rewrap(xnum)`.

### Tier 2 is disabled by default

Tier 2 (`PagerHistoryBuf`) is **disabled by default**: `scrollback_pager_history_size = 0` `[kitty/options/definition.py:406]`, while `scrollback_lines = 2000` `[kitty/options/definition.py:372]`. To observe the inter-tier interaction the harness explicitly sets a non-zero pager history size, while this document also documents the default-disabled state.

**Units — user-facing MB vs internal bytes (important).** The `scrollback_pager_history_size` a user writes in `kitty.conf` is **in MB**; the option *parser* `scrollback_pager_history_size(x)` converts it with `int(max(0, float(x)) * 1024 * 1024)` `[kitty/options/utils.py:564-565]`. That conversion runs only on the parse path. The C layer stores and uses a **raw byte count**: `Screen` construction passes `OPT(scrollback_pager_history_size)` straight through to `alloc_historybuf(...)` `[kitty/screen.c:130]`, and the `HistoryBuf(ynum, xnum, pagerhist_sz)` constructor's third argument is that same raw byte count `[kitty/history.c:136-138]`. Consequences for this harness (Observed):

- Setting the field directly on an already-built `Options` object — e.g. `Options(... scrollback_pager_history_size=1024 ...)` via `merge_result_dicts` — bypasses the parser, so `1024` means **1,024 bytes**, not 1,024 MB.
- The `HistoryBuf` constructor arguments used below (`1048576`, `5*1024*1024`, `50`, `1024*1024`) are likewise **raw bytes** (so `1048576` = 1 MiB of pager ring, `50` = 50 bytes).

### Reading internal counters (canonical real-struct reads)

`num_segments` and the ring-buffer capacity are **not** exposed to Python. They were read from the **live process struct** two ways, both canonical reads of real production memory (NOT bypasses/mocks):

1. **`gdb`** attached to the live PID reading raw memory at the struct offset (the method specified in the plan). The target process constructs `HistoryBuf(6000, 80)`, pushes 6000 lines, and prints its PID and the live `PyObject` address; `gdb` then reads `num_segments` at offset `+24`. Complete, unedited session (Observed):

```
$ cat gdb_target.txt   # PID and live HistoryBuf PyObject address printed by the target
PID=101394
ADDR=132628561901808
ADDR_HEX=0x789ffe51b8f0
NUMSEG_ADDR=132628561901832
ctypes_num_segments=3
ctypes_count=6000

$ gdb -p 101394 -batch -ex 'print *(unsigned int*)(ADDR+24)'   # ADDR=id(hb); num_segments @ +24
$1 = 3
```

2. A **`ctypes` read of the live `PyObject` struct** at validated offsets (an equivalent real-struct read). The offsets were self-validated at runtime (Observed): reads at `xnum@id+16`, `ynum@id+20`, `count@id+60` matched the Python-visible members exactly, confirming the layout. The `gdb` read (`$1 = 3`) matches the `ctypes` read (`ctypes_num_segments=3`). Corroborated further by the derived value `ceil(ynum / 2048)` (Inferred — a calculation from `SEGMENT_SIZE`, not a direct measurement).

Member mutability (each read directly from the live object; reading succeeds regardless of write-protection): on `Screen`, `historybuf` `[kitty/screen.c:4902]` and `scrolled_by` `[kitty/screen.c:4903]` are declared `READONLY`, whereas `history_line_added_count` `[kitty/screen.c:4908]` is declared with member flags `0` (i.e. writable). On `HistoryBuf`, `count` is declared `READONLY` `[kitty/history.c:558]`. The harness only **reads** these fields.

### Scale & stability

Runs were well beyond 2048 lines (up to 205,000 pushes) and beyond 1 MB of pager bytes (up to a 5 MB ring). Every reported **structural** magnitude and every reported **exact byte string** was confirmed identical across two runs (RUN1/RUN2 below) — these are the deterministic, reported values. The non-deterministic quantities are process RSS (reported as an approximate range) and raw heap addresses (`segments_ptr`, `pagerhist_ptr` — inert pointers, never a reported magnitude). The attribution of the RSS variance to lazy `calloc` page commitment is **Inferred** (a causal explanation from code reading), not a measured quantity.

### Cleanup / read-only

All temporary observation and `gdb` scripts (kept in a scratch directory outside the tracked tree) and all gitignored build artifacts produced to enable the harness — `build/`, `kitty/*.so`, the generated Go/C/H sources, the `kitten`/launcher binaries, and source-tree `__pycache__/` — were removed after the evidence above was captured. The git-tracked repository is therefore unchanged apart from this document; the only remaining non-tracked item is the gitignored `.venv/` virtualenv used to build and run the harness (it is not part of the tracked repository and does not affect its state). Verified state (commands: `git status --short`, `git ls-files | wc -l`, `git status --ignored --short`):

```
$ git status --short
 M blitzy/documentation/kitty_815df1e210e0.md
$ git ls-files | wc -l
868
$ git status --ignored --short | grep '^!!'
!! .venv/
```

`git ls-files` counts **868** tracked files: the 867 pre-existing baseline files (all byte-for-byte unchanged) plus this one new document. The only working-tree modification `git status` reports is this document — every build artifact and temporary script has been removed.

### The observation harness (reproducible method)

The following read-only harness uses ONLY canonical `fast_data_types` entry points. It was run as `PYTHONPATH=$PWD python3 harness.py` from the repo root after building. The canonical feed for `Screen` is `s.draw(text); s.carriage_return(); s.linefeed()` (mirroring `kitty_tests/screen.py:766-770`); the canonical push for `HistoryBuf` is `line = lb.line(0); line.set_text(t, 0, n, cursor); hb.push(line)` (mirroring the `set_text`/`push` loop body at `kitty_tests/datatypes.py:504-507`).

```python
import sys, ctypes, math, tracemalloc
sys.path.insert(0, '.')
from kitty.fast_data_types import (HistoryBuf, LineBuf, Cursor, Screen,
    SCROLL_LINE, SCROLL_PAGE, SCROLL_FULL, set_options)
from kitty.options.types import Options, defaults
from kitty.options.parse import merge_result_dicts
from kitty.config import finalize_keys, finalize_mouse_mappings

class Callbacks:                      # minimal canonical Screen callbacks (see kitty_tests/__init__.py:39)
    def __init__(self): self.wtcbuf = b''
    def write(self, data): self.wtcbuf += bytes(data)
    def __getattr__(self, name): return lambda *a, **k: None

# --- live-struct readers (canonical reads of the real HistoryBuf/PagerHistoryBuf/ringbuf_t structs) ---
def u8(a):  return ctypes.c_uint8.from_address(a).value
def u32(a): return ctypes.c_uint32.from_address(a).value
def u64(a): return ctypes.c_uint64.from_address(a).value
def read_hb(hb):                      # offsets validated at runtime vs Python members
    a = id(hb)
    return dict(xnum=u32(a+16), ynum=u32(a+20), num_segments=u32(a+24),
                segments_ptr=u64(a+32), pagerhist_ptr=u64(a+40),
                start_of_data=u32(a+56), count=u32(a+60))
def read_ring(hb):                    # PagerHistoryBuf{ringbuf@0, maximum_size@8, rewrap_needed@16}; ringbuf_t{buf@0,head@8,tail@16,size@24}
    ph = u64(id(hb)+40)
    if ph == 0: return None
    maximum_size = u64(ph+8); rewrap_needed = u8(ph+16); rb = u64(ph+0)
    if rb == 0: return dict(maximum_size=maximum_size, capacity=0, rewrap_needed=rewrap_needed)
    size = u64(rb+24)
    return dict(maximum_size=maximum_size, capacity=size-1, rewrap_needed=rewrap_needed)
def mkline(xnum, text):
    lb = LineBuf(1, xnum); c = Cursor(); l = lb.line(0)
    l.set_text(text, 0, min(len(text), xnum), c); return l, lb
def rss_kb():
    for ln in open('/proc/self/status'):
        if ln.startswith('VmRSS:'): return int(ln.split()[1])
    return -1
def mk_options():
    o = Options(merge_result_dicts(defaults._asdict(), {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}))
    finalize_keys(o, {}); finalize_mouse_mappings(o, {}); return o

# R1: segment carving — HistoryBuf(6000, 80), Tier2 default-disabled
# R2: inter-tier spill — HistoryBuf(5, 5, 1048576); push A..E then F; read pagerhist before/after
# R3: (a) ring growth HistoryBuf(10,200,5MiB); (b) FIFO overwrite; (c) direct-write drop HistoryBuf(5,5,50); (c2) canonical long-line spill HistoryBuf(2,100,50); (d) resize reflow HistoryBuf(3,5,1MiB)
# R4: Screen(Callbacks(),5,10,100,10,20,0,Callbacks()); fill, scroll(SCROLL_PAGE,True), feed more, update_only_line_graphics_data()
# R5: HistoryBuf(200000,80); push 205000; measure num_segments, RSS, tracemalloc, retention cap
```

The full-body harness wraps the per-question functions R1..R5 in a `for tag in ("RUN1","RUN2")` loop; the key push/read/scroll calls for each observation are shown inline in each section below so a reader can reproduce each result. The command that produced all output in Sections (b)–(f) was:

```
python3 setup.py build --ignore-compiler-warnings   # build the extension (see caveat above)
PYTHONPATH=$PWD python3 harness.py                  # run the read-only harness (both runs)
```

---

## (b) R1 — Segment growth under flood

**Question.** What unfolds inside `HistoryBuf` as it fills, "stretches," and begins carving out new segments while a torrent of text arrives?

**Answer (prose).** Tier 1 is segmented: each segment holds `SEGMENT_SIZE = 2048` lines `[kitty/history.c:15]`. `num_segments` **starts at 1** because `create_historybuf` `[kitty/history.c:117]` calls `add_segment` once (Observed: `initial ... num_segments: 1`). A new segment is carved **lazily** by `segment_for` → `add_segment` `[kitty/history.c:18-40]` when a push needs a line beyond the current segments' coverage — Observed exactly at the FIRST push past each 2048 multiple (push #2049 → 2 segments, push #4097 → 3 segments). `add_segment` `[kitty/history.c:18-29]` does a `realloc` of the segments array and then a **single `calloc` block per segment** sized `2048 * (xnum*sizeof(GPUCell) + xnum*sizeof(CPUCell) + sizeof(LineAttrs))`. The cell sizes `sizeof(GPUCell)==20` and `sizeof(CPUCell)==12` (32 bytes/cell) are enforced by `static_assert` `[kitty/data-types.h:221,228]`; `sizeof(LineAttrs)==1` **follows from the `union LineAttrs` definition** `[kitty/data-types.h:231-239]`, whose storage is a single `uint8_t val` (there is no `static_assert` on `LineAttrs`). For `xnum=80` the per-segment size therefore works out to `2048*(80*32+1) = 5,244,928` bytes (5.002 MiB) — **Inferred**, a calculation from those struct sizes and `SEGMENT_SIZE`, not a direct allocation measurement; it is corroborated at runtime by the RSS step at each carve (R5 carve milestones, Observed). The segment count is likewise corroborated by the derived `ceil(6000/2048)=3` (Inferred). Structurally, each carve performs a `realloc` of the segments array plus one multi-MiB `calloc`; no elapsed time was measured in this harness, so this is described as an allocation boundary, not as a perceived pause.

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (RUN1; RUN2 identical for every structural value and exact byte string — only the heap address `segments_ptr` differs, see Stability):**

```
===== R1 SEGMENT CARVING [RUN1] =====
initial: {'xnum': 80, 'ynum': 6000, 'num_segments': 1, 'segments_ptr': 400636704, 'pagerhist_ptr': 0, 'start_of_data': 0, 'count': 0}
per-segment calloc = 2048*(80*32+1) = 5244928 bytes (5.002 MiB)
CARVE push#2049 count=2049 num_segments 1->2
CARVE push#4097 count=4097 num_segments 2->3
final: {'xnum': 80, 'ynum': 6000, 'num_segments': 3, 'segments_ptr': 400500352, 'pagerhist_ptr': 0, 'start_of_data': 0, 'count': 6000}
derived ceil(6000/2048)=3 == num_segments(3) -> True
```

**Observed vs Inferred.** **Observed:** `num_segments` growth (1→2→3) and the carve push indices (#2049, #4097), read from the live struct; corroborated by `gdb` (attaching to the live PID and reading `num_segments` at the struct offset returned `$1 = 3`, matching the `ctypes` read — see section (a)). **Inferred (derived, not directly measured):** the per-segment byte size `5,244,928` (a calculation from the `static_assert` cell sizes plus `SEGMENT_SIZE`) and the `ceil(6000/2048)=3` segment-count check. `segments_ptr` is an inert heap address — non-deterministic, not a reported magnitude.

**Stability.** RUN1 == RUN2 for every structural value (`num_segments`, carve indices `#2049`/`#4097`, final `count=6000`) and for the derived per-segment byte size. Only `segments_ptr` (a heap address) varies between runs.

---

## (c) R2 — The "quiet interaction" between the two tiers (inter-tier spill)

**Question.** How do the segmented storage (Tier 1) and the ring buffer (Tier 2) cooperate when both are pressured?

**Answer (prose).** The cooperation is entirely inside `historybuf_push` `[kitty/history.c:276-286]`. With Tier 2 enabled (`HistoryBuf(5, 5, 1048576)` → `maximum_size = 1048576` bytes), the buffer first fills to `count == ynum == 5` with **no** pager bytes yet. The 6th push triggers the spill: because `count == ynum`, `historybuf_push` calls `pagerhist_push` `[kitty/history.c:259-272]`, which serializes the OLDEST line (`'AAAAA'`) to ANSI via `line_as_ansi` `[kitty/line.c:338]` and writes it into the ring buffer, after which `start_of_data` advances `(start_of_data + 1) % ynum` (eviction from Tier 1) and `count` stays pinned at 5. The serialized form is byte-exact: a literal `\x1b[m` (3-byte SGR reset, written first in `pagerhist_push`), then the line text, then `\r\n` — i.e. `b'\x1b[mAAAAA\r\n'`. Further pushes accumulate in FIFO order (A, then B, C, D…). `hb.line(i)` is reverse-indexed so index 0 is the newest (`index_of` `[kitty/history.c:152-158]`).

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (RUN1; RUN2 identical for count/state/content and the exact serialized bytes — only the heap address `pagerhist_ptr` differs):**

```
===== R2 INTER-TIER SPILL [RUN1] =====
pagerhist_ptr: 400478128 (nonzero => Tier2 enabled) maximum_size: 1048576
BEFORE: count= 5 start_of_data= 0
BEFORE Tier1 newest->oldest: ['EEEEE', 'DDDDD', 'CCCCC', 'BBBBB', 'AAAAA']
BEFORE pagerhist_as_text(): ''
BEFORE pagerhist_as_bytes(): b''
AFTER 6th push (count==ynum triggers spill): count= 5 start_of_data= 1
AFTER Tier1 newest->oldest: ['FFFFF', 'EEEEE', 'DDDDD', 'CCCCC', 'BBBBB']
AFTER pagerhist_as_text(): '\x1b[mAAAAA\r\n'
AFTER pagerhist_as_bytes(): b'\x1b[mAAAAA\r\n'
AFTER +3 (FIFO accumulate) pagerhist_as_text(): '\x1b[mAAAAA\r\n\x1b[mBBBBB\r\n\x1b[mCCCCC\r\n\x1b[mDDDDD\r\n'
```

**Observed vs Inferred.** **Observed:** the before/after `count` (pinned at 5), `start_of_data` (0→1), Tier-1 contents (oldest `'AAAAA'` evicted, newest `'FFFFF'` added), and the exact bytes in Tier 2. The byte-sensitive result is verified against the exact bytes emitted: `b'\x1b[mAAAAA\r\n'`. **Inferred:** nothing in this section. `pagerhist_ptr` printed above is an **inert, non-deterministic heap address** — used only as a nonzero/zero flag for "Tier 2 enabled", never as a reported magnitude.

**Stability.** RUN1 == RUN2 for `count`, `start_of_data`, the Tier-1 contents, and the exact serialized bytes (`b'\x1b[mAAAAA\r\n'` and the FIFO accumulation). Only the inert `pagerhist_ptr` heap address differs between runs.

---

## (d) R3 — Smoothness versus hesitation at segment/buffer limits

**Question.** Are the transitions at boundaries seamless, or are there subtle moments where the system behaves differently than expected?

Each boundary transition is captured individually below.

**(a) Ring growth in ≥1 MB chunks (`pagerhist_extend` `[kitty/history.c:89-101]`).** With `HistoryBuf(10, 200, 5*1024*1024)`, the ring starts at capacity `1048576` (= `MIN(1 MB, maximum_size)` via `initial_pagerhist_ringbuf_sz` `[kitty/history.c:67]`) and grows by 1 MB each time it fills — at pushes #5126, #10241, #15356, #20471 — until capacity == `maximum_size` = 5242880, then stops growing. `pagerhist_extend` `[kitty/history.c:89-101]` computes the new size as `MIN(maximum_size, buffer_size + MAX(1 MB, minsz))`.

**(b) FIFO overwrite-oldest at the ceiling (`ringbuf_memcpy_into` `[3rdparty/ringbuf/ringbuf.h:140-154]`).** Once at `maximum_size`, stored bytes stay pinned at `5242880` and the oldest bytes are overwritten in FIFO fashion. The overwrite happens inside `ringbuf_memcpy_into` — the copy primitive `pagerhist_write_bytes` uses `[kitty/history.c:224]` — whose documented contract states "old data will simply be overwritten in FIFO fashion" `[3rdparty/ringbuf/ringbuf.h:140-154]` (the `:127-136` block documents `ringbuf_memset`, a different primitive not used on this path). Here the content is uniform `'x'*200` lines, so the oldest surviving bytes are `x`'s and SGR/line-ending framing.

**(c) Oversized single-call drop — reachable only through the non-canonical direct-write interface.** `pagerhist_write_bytes` rejects any *single* write whose size exceeds the cap: `if (sz > ph->maximum_size) return false` `[kitty/history.c:220]`. Whole-line, this branch is reachable only through the **non-canonical** debug/test method `HistoryBuf.pagerhist_write(bytes)`, which for a `bytes` argument performs one `pagerhist_write_bytes` call carrying the full payload `[kitty/history.c:437]`. With `HistoryBuf(5, 5, 50)`, a direct `pagerhist_write(b'Z'*100)` (100 > 50 in a single call) is DROPPED (the pager stays empty); a direct `pagerhist_write(b'Q'*40)` (40 ≤ 50) is accepted. **This direct-write drop is *not* the terminal-line eviction path.** The canonical spill (`historybuf_push` → `pagerhist_push` `[kitty/history.c:276-286,259-272]`) never writes an evicted line as one call: it emits the SGR reset `"\x1b[m"` (3 bytes) `[kitty/history.c:266]`, then each character one codepoint at a time via `pagerhist_write_ucs4` `[kitty/history.c:248-256]` (each a 1–4-byte call), then the `\r`/`\r\n` line ending `[kitty/history.c:268-271]`. Every sub-write is far smaller than any realistic `maximum_size`, so the `sz > maximum_size` guard is never tripped by a spilled line; instead, when the ring fills, the copy FIFO-overwrites the oldest bytes.

**(c2) Canonical long-line spill FIFO-overwrites (it is *not* dropped whole).** To confirm behavior on the real eviction path, `HistoryBuf(2, 100, 50)` is filled with two 100-column `'x'` lines and a third is pushed, forcing the oldest 100-column line to spill through `historybuf_push` → `pagerhist_push`. The serialized line (`\x1b[m` + 100 codepoints + `\r\n` = 105 bytes) is written in small pieces into the 50-byte ring, which FIFO-overwrites down to its capacity: the pager retains the **last 50 bytes** (`b'x'*48 + b'\r\n'`, `pager_after_len = 50`), NOT an empty/dropped result. A canonical long line is therefore truncated-from-the-front by FIFO overwrite, never discarded whole. Latency at this boundary was not timed; it is described structurally (a bounded sequence of small ring writes with in-place overwrite once at the cap), not as a perceived pause.

**(d) Resize-while-scrolled reflow (`pagerhist_rewrap`/`pagerhist_rewrap_to` `[kitty/history.c:392]`).** With `HistoryBuf(3, 5, 1 MB)` holding `11111`/`22222`/`33333` in the pager, `pagerhist_rewrap(3)` reflows each line to width 3: `\x1b[m11111\r\n` → `\x1b[m111\r11\n` (the `\r` marks the intra-line wrap point).

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (RUN1; RUN2 identical):**

```
===== R3 EDGES [RUN1] =====
(a) growth: initial capacity= 1048576  maximum_size= 5242880
    GROW push#5126: capacity 1048576->2097152
    GROW push#10241: capacity 2097152->3145728
    GROW push#15356: capacity 3145728->4194304
    GROW push#20471: capacity 4194304->5242880
    final capacity= 5242880  == maximum_size?  True
(b) FIFO: bytes_stored= 5242880  == maximum_size?  True
    oldest 24 bytes: b'xxx\r\n\x1b[mxxxxxxxxxxxxxxxx'
(c) direct-write drop: write 100 bytes into maximum_size=50 -> before= b''  after= b''  DROPPED?  True
    write 40 bytes (<=50) -> stored= b'QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ'  ACCEPTED?  True
(c2) canonical 100-col line spill into maximum_size=50 ring: pager_before= b''  pager_after_len= 50  retained= b'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx\r\n'
     NOT_DROPPED (canonical long line FIFO-overwrites, not whole-drop)?  True
(d) reflow BEFORE pagerhist_rewrap: as_text= '\x1b[m11111\r\n\x1b[m22222\r\n\x1b[m33333\r\n'  rewrap_needed= 0
    AFTER pagerhist_rewrap(3): as_text= '\x1b[m111\r11\n\x1b[m222\r22\n\x1b[m333\r33\n'  rewrap_needed= 0
```

**Observed vs Inferred.** **Observed:** all four ring-growth steps and the ceiling equality (a); the `bytes_stored == maximum_size` pinning and the oldest surviving bytes (b); the direct-write drop/accept booleans (c) and the canonical long-line FIFO-overwrite retention (c2: `pager_after_len = 50`, `retained = b'x'*48 + b'\r\n'`); and the exact reflow byte transformation (d). **Inferred:** (i) that the canonical spill path cannot trip the single-call `sz > maximum_size` drop — reasoned from the small per-sub-write sizes in `pagerhist_push`/`pagerhist_write_ucs4` `[kitty/history.c:259-272,248-256]` and corroborated by the (c2) capture; and (ii) the note that the *deferred* `rewrap_needed = 1` path `[kitty/history.c:607-608]` is set by `historybuf_rewrap` on a differing `xnum` when pager bytes already exist — labeled inferred-from-code because the direct `pagerhist_rewrap(3)` call applies the rewrap immediately, so `rewrap_needed` reads `0` in the output above.

**Stability.** RUN1 == RUN2 for every value (growth push indices, capacities, direct-write drop/accept booleans, the canonical (c2) `pager_after_len = 50` and retained bytes, and the reflow bytes). No heap addresses appear in this section's output, so every reported value is deterministic.

---

## (e) R4 — Concurrent scroll while writing

**Question.** What changes when the user is actively scrolling through old output while new data arrives at full speed?

**Answer (prose).** Build a real `Screen` (lines=5, cols=10, scrollback=100 ⇒ history `ynum = MAX(100, 5) = 100` via `alloc_historybuf` `[kitty/screen.c:130]`), fill 50 lines, then scroll back one page. `screen.scroll(SCROLL_PAGE, True)` scrolls up by `lines - 1 = 4` `[kitty/screen.c:4091-4118]`, so `scrolled_by = 4`. While scrolled back, feeding 10 more lines increments `history_line_added_count` to 10 (each `INDEX_UP` with `add_to_history` bumps it at `[kitty/screen.c:1559]`, within the ingest macro `[kitty/screen.c:1552-1567]`) but `scrolled_by` stays 4 until a render applies the re-clamp. The re-clamp is `scrolled_by = MIN(scrolled_by + history_line_added_count, count) = MIN(4 + 10, 56) = 14`, which keeps the SAME old content pinned in view as new lines push it back, then `history_line_added_count` is reset to 0 by `screen_reset_dirty` `[kitty/screen.c:2598-2601]`. Continuing to feed eventually SATURATES `scrolled_by` at `count`, which itself caps at `ynum = 100`; beyond that the oldest content scrolls away and evicts (the view can no longer anchor further back).

**Note on canonicality (R4).** The harness triggers the re-clamp through the no-arg Python method `update_only_line_graphics_data` `[kitty/screen.c:4867]` → `screen_update_only_line_graphics_data` `[kitty/screen.c:2713]`, which applies the re-clamp at line `2716`. That C function is **test-only, not the production render path** — its own comment states it is "used exclusively for testing unicode placeholders" `[kitty/screen.c:2709-2711]`. It is used here as **non-canonical corroboration** because the production render entry point `screen_update_cell_data` `[kitty/screen.c:2738]` requires a `FONTS_DATA_HANDLE` (a live GPU/font backend) and cannot be driven in a headless harness. **Inferred:** production applies the identical re-clamp — `screen_update_cell_data` contains the byte-for-byte same statement at `[kitty/screen.c:2761]` as the test-only function does at `[kitty/screen.c:2716]` (`if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);`), so the observed re-clamp value is representative of production behavior — labeled inferred because the production entry point itself was not exercised headlessly.

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (RUN1; RUN2 identical):**

```
===== R4 SCROLL-WHILE-WRITE [RUN1] =====
after fill: count= 46  scrolled_by= 0  hlac= 0
BEFORE more writes: scrolled_by= 4  count= 46
DURING (pre-render): scrolled_by= 4  history_line_added_count= 10  count= 56
AFTER re-clamp: scrolled_by=14 == MIN(4+10,56)=14  hlac_reset=0
SATURATION: scrolled_by= 100  count= 100  ynum= 100
```

**Observed vs Inferred.** **Observed:** all four states of `scrolled_by` (0 after fill, 4 after scroll-back, still 4 during pre-render while `history_line_added_count` accumulates to 10, then 14 after the re-clamp applied by the test-only method), the `history_line_added_count` accumulation and reset, and the saturation at `scrolled_by == count == ynum == 100`. Member mutability (canonical Python-exposed members): `Screen.historybuf` is READONLY `[kitty/screen.c:4902]`, `Screen.scrolled_by` is READONLY `[kitty/screen.c:4903]`, `Screen.history_line_added_count` is WRITABLE (flags = 0) `[kitty/screen.c:4908]`, and `HistoryBuf.count` is READONLY `[kitty/history.c:558]`; the harness only reads these values, never writes them. **Inferred:** that production (`screen_update_cell_data` `[kitty/screen.c:2761]`) performs the identical re-clamp — the observed value came from the byte-identical statement in the test-only `screen_update_only_line_graphics_data` `[kitty/screen.c:2716]` (non-canonical corroboration), since the production render path requires a GPU/font backend not available headlessly.

**Stability.** RUN1 == RUN2 for every value.

---

## (f) R5 — Runtime allocation, wrapping, and retention at scale

**Question.** How do the underlying memory structures evolve as pressure builds — allocation (segment `calloc` plus ring-buffer growth), wrapping (circular indexing plus line reflow), and retention (the `ynum` cap on Tier 1, the `maximum_size` cap on Tier 2, and eviction/overwrite)?

**Answer (prose).** With `HistoryBuf(200000, 80)` (Tier 2 disabled), pushing 205,000 lines pins `count` at `ynum = 200000` — the Tier-1 **retention cap** (`historybuf_push` `[kitty/history.c:276-286]` overwrites in place once full, evicting the oldest each push). `num_segments` reaches `98 == ceil(200000 / 2048)`. Total Tier-1 allocation is a derived `98 * 5,244,928 = 514,002,944` bytes (490.2 MiB) — a calculation from the observed `num_segments` and the per-segment `calloc` size, not a direct measurement (see Observed vs Inferred) — which the measured process RSS delta tracks closely (measured ~479–488 MiB across the two runs; this RSS figure is the only non-deterministic value, reported as approximate, and its run-to-run variance is attributed to lazy `calloc` page commitment). `tracemalloc` shows only ~5–9 KB of Python-object delta because the large `calloc` blocks are C allocations invisible to `tracemalloc`. **Wrapping/retention:** circular indexing `index_of` `[kitty/history.c:152-158]` maps logical→physical with `(start_of_data + idx) % ynum` (reverse-indexed, line 0 = newest); the Tier-1 retention cap is `ynum`, and the Tier-2 retention cap is `maximum_size` with FIFO overwrite (from R2/R3).

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (BOTH runs, to show structural determinism + RSS variance):**

```
===== R5 ALLOCATION/RETENTION [RUN1] =====
HistoryBuf(200000,80) pagerhist_ptr=0 (0=Tier2 disabled)
per-segment calloc = 5244928 B (5.002 MiB)
pushed 205000 -> count=200000 RETENTION count==ynum? True num_segments=98
ceil(200000/2048)=98 == num_segments? True
total seg alloc = 98*5244928 = 514002944 B (490.2 MiB)
RSS delta = 490336 KB (~478.8 MiB) ; tracemalloc py-delta = 5254 B (C calloc untracked)
carve milestones {count:(num_segments,rss_KB)} = {1: (1, 64), 2048: (1, 64), 2049: (2, 64), 4096: (2, 64), 4097: (3, 2624), 200000: (98, 490336)}

===== R5 ALLOCATION/RETENTION [RUN2] =====
HistoryBuf(200000,80) pagerhist_ptr=0 (0=Tier2 disabled)
per-segment calloc = 5244928 B (5.002 MiB)
pushed 205000 -> count=200000 RETENTION count==ynum? True num_segments=98
ceil(200000/2048)=98 == num_segments? True
total seg alloc = 98*5244928 = 514002944 B (490.2 MiB)
RSS delta = 499608 KB (~487.9 MiB) ; tracemalloc py-delta = 9478 B (C calloc untracked)
carve milestones {count:(num_segments,rss_KB)} = {1: (1, 628), 2048: (1, 3940), 2049: (2, 4088), 4096: (2, 9072), 4097: (3, 9216), 200000: (98, 499608)}
```

**Observed vs Inferred.** **Observed:** the `count == ynum` retention cap (pinned at 200000) and `num_segments == 98` — both read from the live struct; the measured RSS delta (490336 KB / ~478.8 MiB in RUN1, 499608 KB / ~487.9 MiB in RUN2); the measured `tracemalloc` py-delta (5254 B / 9478 B, confirming the large C `calloc` blocks are untracked by `tracemalloc`); and the carve milestones. **Inferred (derived, not directly measured):** (i) the total Tier-1 allocation `98 * 5,244,928 = 514,002,944` bytes (490.2 MiB) — a calculation from the observed `num_segments` times the per-segment `calloc` size (itself derived from the cell/attrs struct sizes, see R1), not a directly measured quantity; and (ii) the *cause* of the RSS run-to-run variance (lazy `calloc` page commitment), which explains why RSS is non-deterministic while the structural values are not. The structural values (`count`, `num_segments`, total seg alloc) are IDENTICAL across both runs; only the measured RSS (490336 vs 499608 KB → ~479–488 MiB), the `tracemalloc` py-delta (5254 vs 9478 B), and the per-milestone RSS columns differ run-to-run — reproduced, NOT engineered away.

**Stability.** Structural values (`count = 200000`, `num_segments = 98`, total seg alloc = 514,002,944 B) are identical RUN1 == RUN2 at a scale of 205,000 pushes into `ynum = 200000`. The measured RSS delta is approximate and non-deterministic (490336 KB / ~478.8 MiB vs 499608 KB / ~487.9 MiB, i.e. ~479–488 MiB), with the cause stated (lazy `calloc` page commitment); the `tracemalloc` py-delta (5254 vs 9478 B) and the per-milestone RSS columns likewise vary. The variance is reproduced across the two identical runs, not smoothed away.

---

## Synthesis: "the quiet interaction" and "smoothness versus hesitation"

**The quiet interaction (R2/R5).** The two tiers cooperate in exactly one place — `historybuf_push` `[kitty/history.c:276-286]`. While Tier 1 is filling (`count < ynum`) the two tiers do not interact at all; Tier 2 stays empty (R2 BEFORE: `pagerhist_as_text()` is `''`). The interaction begins precisely when Tier 1 saturates (`count == ynum`): every subsequent push **evicts** the oldest Tier-1 line, **serializes** it to an ANSI byte stream via `line_as_ansi` `[kitty/line.c:338]`, and **ingests** it into the Tier-2 ring — Tier 1 → serialize → Tier 2, then `start_of_data` advances. This is "quiet" because it is invisible to the Python API surface except through `pagerhist_as_text()`/`pagerhist_as_bytes()`; `count` never changes once saturated (R2), and the interactively-scrollable view keeps working off Tier 1 while the overflow archive silently accretes in Tier 2.

**Smoothness versus hesitation (R1/R3).** No elapsed time was measured in this investigation, so the boundaries below are described structurally — by the work each performs — not as timed pauses. In steady state, pushing a line within an existing segment and evicting/overwriting once Tier 1 is full are in-place operations that allocate nothing. The structural discontinuities — points where the work-per-push changes — are:

- **The segment carve (R1)** — a `realloc` of the segments array plus one multi-MiB `calloc` (a derived `5,244,928` bytes for `xnum=80`; see R1) at each 2048-line crossing (`add_segment` `[kitty/history.c:18-29]`). Observed at pushes #2049 and #4097; the relative size or duration of this allocation versus others was not measured or ranked.
- **Tier-1 saturation onset (R2/R4)** — the exact push where `count == ynum` flips behavior from "grow" to "evict"; from that push onward every write does eviction work (and, if Tier 2 is enabled, serialization + ring write).
- **The 1 MB ring-growth steps (R3a)** — Tier 2 grows in ≥1 MB chunks (`pagerhist_extend` `[kitty/history.c:89-101]`) at pushes #5126/#10241/#15356/#20471; each step allocates a new ring and copies the used bytes across.
- **The FIFO overwrite ceiling (R3b)** — once the ring hits `maximum_size`, growth stops and the oldest bytes are overwritten in place by `ringbuf_memcpy_into` `[3rdparty/ringbuf/ringbuf.h:140-154]`; this performs no allocation but silently discards the oldest archived bytes.
- **The oversized single-call drop (R3c)** — a *single* write larger than `maximum_size` returns `false` and is dropped whole (`pagerhist_write_bytes` `[kitty/history.c:220]`). Whole-line, this is reachable only through the **non-canonical** direct-write method `pagerhist_write(bytes)` `[kitty/history.c:437]`. The canonical eviction path serializes each evicted line as many small sub-writes (R3c2), so a long *terminal* line is FIFO-overwritten down to the cap — retaining the last `maximum_size` bytes — never dropped whole.

In short: the steady state does in-place eviction/overwrite (no allocation), while allocation events (segment carve, ring growth) and the two bounded limits (FIFO ceiling, oversized single-call drop) are the points where the work-per-push changes or the retention model differs from a naïve unbounded model. These are described structurally from the observations above; no elapsed time or perceived pause was measured.

---

## References

Every `file:line` below was confirmed accurate at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

| Reference | `file:line` | Role |
|-----------|-------------|------|
| `SEGMENT_SIZE = 2048` | `kitty/history.c:15` | Tier-1 segment size |
| `add_segment` | `kitty/history.c:18-29` | `realloc` + per-segment `calloc` |
| `segment_for` | `kitty/history.c:37-40` | Lazy segment carve trigger |
| initial ring size (`MIN(1 MB, sz)`) | `kitty/history.c:67` | Tier-2 initial ring capacity |
| `alloc_pagerhist` | `kitty/history.c:69-80` | Tier-2 ring allocation |
| `pagerhist_extend` | `kitty/history.c:89-101` | ≥1 MB ring growth up to `maximum_size` |
| `create_historybuf` | `kitty/history.c:117` | Calls `add_segment` once → `num_segments` starts at 1 |
| `new_history_object` (parses `"II\|I"` → `ynum, xnum, pagerhist_sz`) | `kitty/history.c:136-138` | Constructor `ynum` FIRST |
| `index_of` (reverse `(start_of_data + idx) % ynum`) | `kitty/history.c:152-158` | Circular indexing, line 0 = newest |
| `pagerhist_write_bytes` (`if (sz > maximum_size) return false`) | `kitty/history.c:218-226` | Oversized single-call drop (non-canonical direct-write path); canonical spill FIFO-overwrites |
| `pagerhist_push` (writes `\x1b[m`, line, `\r`/`\n`) | `kitty/history.c:259-272` | Serialize evicted line into Tier 2 |
| `historybuf_push` (spill at `count == ynum`, advance `start_of_data`) | `kitty/history.c:276-286` | Inter-tier cooperation point |
| `push` (Python; takes a `Line`) | `kitty/history.c:337` | Canonical push entry |
| `pagerhist_rewrap_to` | `kitty/history.c:392` | Tier-2 reflow on resize |
| `pagerhist_as_bytes` | `kitty/history.c:461` | Read Tier-2 bytes |
| `pagerhist_as_text` | `kitty/history.c:486` | Read Tier-2 text |
| method table | `kitty/history.c:542-552` | Python-visible `HistoryBuf` methods |
| `rewrap_needed = true` set | `kitty/history.c:607-608` | Deferred reflow flag (inferred path) |
| `HistoryBufSegment {gpu_cells, cpu_cells, line_attrs}` | `kitty/data-types.h:262-266` | Segment struct |
| `PagerHistoryBuf {ringbuf, maximum_size, rewrap_needed}` | `kitty/data-types.h:268-272` | Tier-2 struct |
| `HistoryBuf {…xnum, ynum, num_segments; segments; pagerhist; line; start_of_data, count}` | `kitty/data-types.h:283-290` | Tier-1 struct |
| `static_assert` GPUCell==20 `[:221]` / CPUCell==12 `[:228]`; `union LineAttrs` (`uint8_t val`, no assert) `[:231-239]` | `kitty/data-types.h:215-239` | Per-cell byte sizes (GPU+CPU = 32 B/cell); LineAttrs is 1 B by union definition |
| `alloc_historybuf(MAX(scrollback, lines), …)` | `kitty/screen.c:130` | `ynum = MAX(scrollback, lines)` |
| `INDEX_UP` macro (`history_line_added_count++` at `:1559`) | `kitty/screen.c:1552-1567` | Canonical ingest path |
| `screen_reset_dirty` (resets `history_line_added_count`) | `kitty/screen.c:2598-2601` | Post-render reset |
| `screen_update_only_line_graphics_data` (**test-only**; re-clamp at `:2716`; comment `:2709-2711`) | `kitty/screen.c:2713-2735` | Test-only re-clamp `scrolled_by = MIN(scrolled_by + hlac, count)` (non-canonical corroboration) |
| `screen_update_cell_data` (production render; byte-identical re-clamp at `:2761`) | `kitty/screen.c:2738,2761` | Production re-clamp; needs `FONTS_DATA_HANDLE` (inferred identical to test-only) |
| `update_only_line_graphics_data` method | `kitty/screen.c:4867` | No-arg Python render step (test-only path) |
| `historybuf` READONLY `[:4902]` / `scrolled_by` READONLY `[:4903]` / `history_line_added_count` writable `[:4908]` | `kitty/screen.c:4902-4908` | Member mutability (harness reads only) |
| `screen_history_scroll` (`SCROLL_PAGE` → `lines - 1`) | `kitty/screen.c:4091-4118` | Scroll-back mechanics |
| `line_as_ansi` | `kitty/line.c:338` | Serialize a line to ANSI bytes |
| `ringbuf_memcpy_into` FIFO overwrite ("old data will simply be overwritten") | `3rdparty/ringbuf/ringbuf.h:140-154` | Tier-2 overflow semantics |
| `scrollback_lines = 2000` | `kitty/options/definition.py:372` | Tier-1 default line cap |
| `scrollback_pager_history_size = 0` | `kitty/options/definition.py:406` | Tier-2 default-disabled |
| MB→bytes (`int(max(0, float(x)) * 1024 * 1024)`) | `kitty/options/utils.py:564-565` | Options-path size conversion |
| `test_historybuf` (`HistoryBuf(3000, 5)`, crosses 2048) | `kitty_tests/datatypes.py:487-540` | Canonical usage pattern |
| `filled_line_buf` / `filled_cursor` / `filled_history_buf` / `create_screen` | `kitty_tests/__init__.py:166,175,184,237-240` | Harness patterns |
| `draw` / `carriage_return` / `linefeed` feed | `kitty_tests/screen.py:766-770` | Canonical `Screen` feed |
| `build()` | `setup.py:1084` | Build entry |
| `compile_c_extension(..., 'kitty/fast_data_types', ...)` | `setup.py:1091` | Extension compile |
| `werror` gate | `setup.py:491` | `-Werror` promotion |
| `--ignore-compiler-warnings` | `setup.py:2003` | Disables `-Werror` promotion |

---

*All observations use the canonical `HistoryBuf`/`Screen` path imported from `kitty.fast_data_types`, with one explicitly-labeled exception: the R4 scroll re-clamp is corroborated through the **test-only** `update_only_line_graphics_data` method (its comment states it is "used exclusively for testing unicode placeholders" `[kitty/screen.c:2709-2711]`), because the production render entry point `screen_update_cell_data` `[kitty/screen.c:2738]` requires a live GPU/font backend and cannot be driven headlessly; production applies the byte-identical re-clamp at `[kitty/screen.c:2761]` (inferred). The two read-only helpers used to read internal counters — `gdb` attached to the live PID, and a `ctypes` read of the live `PyObject` struct at validated offsets — are reads of the real production structs, not bypasses. The investigation was strictly read-only: temporary observation/`gdb` scripts and all gitignored build artifacts were removed after capture, so the git-tracked repository is unchanged apart from this document (the only remaining non-tracked item is the gitignored `.venv/` used to run the harness).*
