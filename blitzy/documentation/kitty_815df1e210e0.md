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

**Build caveat (Observed).** On the canonical container's toolchain (Ubuntu 24.04, gcc 13.3, newer Wayland headers), the plain build aborts because `glfw/wl_window.c:668` hits `-Werror=switch` on `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enums. The `fast_data_types` extension itself compiles cleanly; only the unrelated glfw/Wayland windowing step is affected. The working build command used was:

```
python3 setup.py build --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag `[setup.py:2003]` gates the `werror` string `[setup.py:491]` (`werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'`). This affects only `-Werror` promotion, not the code that is compiled. `kitty/fast_data_types.so` (1,213,072 bytes) links and imports cleanly. The windowing build issue is a host-toolchain / newer-headers artifact, unrelated to the scrollback subsystem under test; headless `HistoryBuf`/`Screen` observation needs no GPU/display.

### Canonical entry point, zero bypass

All observations drive the real `HistoryBuf`/`Screen` objects imported from `kitty.fast_data_types`, mirroring the existing tests (`test_historybuf` `[kitty_tests/datatypes.py:487-540]` constructs `HistoryBuf(3000, 5)` and pushes 3000 lines across the 2048 boundary; `filled_line_buf`/`filled_cursor`/`filled_history_buf`/`create_screen` `[kitty_tests/__init__.py:166,175,184,237-240]`). **No remote-control, debug hook, mock, or synthetic stand-in was used.**

### Constructor signature

`HistoryBuf(ynum, xnum[, pagerhist_sz])` — **`ynum` FIRST** (the C `new_history_object` `[kitty/history.c:136-138]` parses `"II|I"` → `ynum, xnum, pagerhist_sz`). The optional third arg `pagerhist_sz` is the ring buffer's `maximum_size` **in raw bytes** at the C level (passed straight to `alloc_pagerhist` `[kitty/history.c:69-80]`); the MB→bytes conversion (`int(max(0, float(x)) * 1024 * 1024)`) applies only on the options path `[kitty/options/utils.py:564-565]`. Python-visible signals: `hb.push(line)` (takes a `Line` object `[kitty/history.c:337]`), `hb.count`, `hb.line(i)` (reverse-indexed, line 0 = newest), `hb.pagerhist_as_text()`, `hb.pagerhist_as_bytes()`, `hb.pagerhist_write(bytes)`, `hb.pagerhist_rewrap(xnum)`.

### Tier 2 is disabled by default

Tier 2 (`PagerHistoryBuf`) is **disabled by default**: `scrollback_pager_history_size = 0` `[kitty/options/definition.py:406]` (measured in MB), while `scrollback_lines = 2000` `[kitty/options/definition.py:372]`. To observe the inter-tier interaction the harness explicitly sets a non-zero pager history size (via the `pagerhist_sz` constructor arg, or `Options(scrollback_pager_history_size=1024)`), while this document also documents the default-disabled state.

### Reading internal counters (canonical real-struct reads)

`num_segments` and the ring-buffer capacity are **not** exposed to Python. They were read from the **live process struct** two ways, both canonical reads of real production memory (NOT bypasses/mocks):

1. **`gdb`** attached to the live PID reading raw memory at struct offsets (the method specified in the plan), which corroborated `num_segments`.
2. A **`ctypes` read of the live `PyObject` struct** at validated offsets (an equivalent real-struct read). The offsets were self-validated at runtime: reads at `xnum@id+16`, `ynum@id+20`, `count@id+60` matched the Python-visible members exactly, confirming the layout. Corroborated further by the derived `ceil(ynum / 2048)`.

`scrolled_by`, `history_line_added_count`, `historybuf`, and `count` **are** exposed READONLY on `Screen`/`HistoryBuf` and were read directly (canonical) `[kitty/screen.c:4902-4903]`.

### Scale & stability

Runs were well beyond 2048 lines (up to 205,000 pushes) and beyond 1 MB of pager bytes (up to a 5 MB ring). Every reported magnitude was confirmed **stable across two identical runs** (RUN1/RUN2 below); the only non-deterministic quantity is process RSS, which is reported as an approximate range with its cause (lazy `calloc` paging).

### Cleanup / read-only

All temporary observation and `gdb` scripts and all build artifacts were removed after capture; `git status` was verified clean (`git ls-files` = 867 unchanged, HEAD unchanged), so the repository is byte-for-byte unchanged apart from this document.

### The observation harness (reproducible method)

The following read-only harness uses ONLY canonical `fast_data_types` entry points. It was run as `PYTHONPATH=$PWD python3 harness.py` from the repo root after building. The canonical feed for `Screen` is `s.draw(text); s.carriage_return(); s.linefeed()` (mirroring `kitty_tests/screen.py:766-770`); the canonical push for `HistoryBuf` is `line = lb.line(0); line.set_text(t, 0, n, cursor); hb.push(line)` (mirroring `kitty_tests/datatypes.py:508-513`).

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
# R3: (a) ring growth HistoryBuf(10,200,5MiB); (b) FIFO overwrite; (c) hard-drop HistoryBuf(5,5,50); (d) resize reflow HistoryBuf(3,5,1MiB)
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

**Answer (prose).** Tier 1 is segmented: each segment holds `SEGMENT_SIZE = 2048` lines `[kitty/history.c:15]`. `num_segments` **starts at 1** because `create_historybuf` `[kitty/history.c:117]` calls `add_segment` once. A new segment is carved **lazily** by `segment_for` → `add_segment` `[kitty/history.c:18-40]` when a push needs a line beyond the current segments' coverage — observed exactly at the FIRST push past each 2048 multiple (push #2049 → 2 segments, push #4097 → 3 segments). `add_segment` `[kitty/history.c:18-29]` does a `realloc` of the segments array and then a **single `calloc` block per segment** sized `2048 * (xnum*sizeof(GPUCell) + xnum*sizeof(CPUCell) + sizeof(LineAttrs))`. With `sizeof(GPUCell)==20`, `sizeof(CPUCell)==12` (so 32 bytes/cell) and `sizeof(LineAttrs)==1` (all verified via `static_assert` `[kitty/data-types.h:215-239]`), for `xnum=80` this is `2048*(80*32+1) = 5,244,928` bytes (5.002 MiB). Corroborated by the derived `ceil(6000/2048)=3`. This is a "hesitation" point: the carve is a `realloc` plus a multi-MiB `calloc`.

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (RUN1; RUN2 identical for all values):**

```
===== R1 SEGMENT CARVING [RUN1] =====
initial: {'xnum': 80, 'ynum': 6000, 'num_segments': 1, 'segments_ptr': 328920240, 'pagerhist_ptr': 0, 'start_of_data': 0, 'count': 0}
per-segment calloc = 2048*(80*32+1) = 5244928 bytes (5.002 MiB)
CARVE push#2049 count=2049 num_segments 1->2
CARVE push#4097 count=4097 num_segments 2->3
final: {'xnum': 80, 'ynum': 6000, 'num_segments': 3, 'segments_ptr': 328698624, 'pagerhist_ptr': 0, 'start_of_data': 0, 'count': 6000}
derived ceil(6000/2048)=3 == num_segments(3) -> True
```

**Observed vs Inferred.** **Observed:** `num_segments` growth (1→2→3), the carve push indices (#2049, #4097), and the per-segment byte size (via the live-struct read plus the `static_assert` sizes). Note `segments_ptr` differs run-to-run (it is a heap address — expected, non-deterministic, and not a reported magnitude). The `gdb` corroboration confirms this: attaching to the live PID and reading `num_segments` at the struct offset returned 3, matching the `ctypes` read. **Inferred:** nothing — all values are observed.

**Stability.** RUN1 == RUN2 for every value (`num_segments`, carve indices, per-segment bytes). Only `segments_ptr` varies (heap address).

---

## (c) R2 — The "quiet interaction" between the two tiers (inter-tier spill)

**Question.** How do the segmented storage (Tier 1) and the ring buffer (Tier 2) cooperate when both are pressured?

**Answer (prose).** The cooperation is entirely inside `historybuf_push` `[kitty/history.c:276-286]`. With Tier 2 enabled (`HistoryBuf(5, 5, 1048576)` → `maximum_size = 1048576` bytes), the buffer first fills to `count == ynum == 5` with **no** pager bytes yet. The 6th push triggers the spill: because `count == ynum`, `historybuf_push` calls `pagerhist_push` `[kitty/history.c:259-272]`, which serializes the OLDEST line (`'AAAAA'`) to ANSI via `line_as_ansi` `[kitty/line.c:338]` and writes it into the ring buffer, after which `start_of_data` advances `(start_of_data + 1) % ynum` (eviction from Tier 1) and `count` stays pinned at 5. The serialized form is byte-exact: a literal `\x1b[m` (3-byte SGR reset, written first in `pagerhist_push`), then the line text, then `\r\n` — i.e. `b'\x1b[mAAAAA\r\n'`. Further pushes accumulate in FIFO order (A, then B, C, D…). `hb.line(i)` is reverse-indexed so index 0 is the newest (`index_of` `[kitty/history.c:152-158]`).

**Command.**

```
python3 setup.py build --ignore-compiler-warnings
PYTHONPATH=$PWD python3 harness.py
```

**Complete, unedited output (RUN1; RUN2 identical):**

```
===== R2 INTER-TIER SPILL [RUN1] =====
pagerhist_ptr: 329083008 (nonzero => Tier2 enabled) maximum_size: 1048576
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

**Observed vs Inferred.** **Observed:** the before/after `count` (pinned at 5), `start_of_data` (0→1), Tier-1 contents (oldest `'AAAAA'` evicted, newest `'FFFFF'` added), and the exact bytes in Tier 2. The byte-sensitive result is verified against the exact bytes emitted: `b'\x1b[mAAAAA\r\n'`. **Inferred:** nothing.

**Stability.** RUN1 == RUN2 for every value, including the exact serialized bytes.

---

## (d) R3 — Smoothness versus hesitation at segment/buffer limits

**Question.** Are the transitions at boundaries seamless, or are there subtle moments where the system behaves differently than expected?

Each boundary transition is captured individually below.

**(a) Ring growth in ≥1 MB chunks (`pagerhist_extend` `[kitty/history.c:89-101]`).** With `HistoryBuf(10, 200, 5*1024*1024)`, the ring starts at capacity `1048576` (= `MIN(1 MB, maximum_size)` via `initial_pagerhist_ringbuf_sz` `[kitty/history.c:67]`) and grows by 1 MB each time it fills — at pushes #5126, #10241, #15356, #20471 — until capacity == `maximum_size` = 5242880, then stops growing. `pagerhist_extend` `[kitty/history.c:89-101]` computes the new size as `MIN(maximum_size, buffer_size + MAX(1 MB, minsz))`.

**(b) FIFO overwrite-oldest at the ceiling (`[3rdparty/ringbuf/ringbuf.h:127-136]`).** Once at `maximum_size`, stored bytes stay pinned at `5242880` and the oldest bytes are overwritten in FIFO fashion — "old data will simply be overwritten in FIFO fashion" `[3rdparty/ringbuf/ringbuf.h:127-136]`. Here the content is uniform `'x'*200` lines, so the oldest surviving bytes are `x`'s and SGR/line-ending framing.

**(c) Single-line hard-drop (`pagerhist_write_bytes`: `if (sz > ph->maximum_size) return false` `[kitty/history.c:218-226]`).** With `HistoryBuf(5, 5, 50)`, `pagerhist_write(b'Z'*100)` (100 > 50) is DROPPED (the pager stays empty); `pagerhist_write(b'Q'*40)` (40 ≤ 50) is accepted. This is the sharpest "hesitation": an oversized line is silently discarded rather than truncated.

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
(c) hard-drop: write 100 bytes into maximum_size=50 -> before= b''  after= b''  DROPPED?  True
    write 40 bytes (<=50) -> stored= b'QQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQQ'  ACCEPTED?  True
(d) reflow BEFORE pagerhist_rewrap: as_text= '\x1b[m11111\r\n\x1b[m22222\r\n\x1b[m33333\r\n'  rewrap_needed= 0
    AFTER pagerhist_rewrap(3): as_text= '\x1b[m111\r11\n\x1b[m222\r22\n\x1b[m333\r33\n'  rewrap_needed= 0
```

**Observed vs Inferred.** **Observed:** all four ring-growth steps and the ceiling equality (a); the `bytes_stored == maximum_size` pinning and the oldest surviving bytes (b); the drop/accept booleans (c); and the exact reflow byte transformation (d). **Inferred:** only the note that the *deferred* `rewrap_needed = 1` path `[kitty/history.c:604-608]` is set by `historybuf_rewrap` on a differing `xnum` when pager bytes already exist — labeled inferred-from-code because the direct `pagerhist_rewrap(3)` call applies the rewrap immediately, so `rewrap_needed` reads `0` in the output above.

**Stability.** RUN1 == RUN2 for every value (growth push indices, capacities, drop/accept booleans, reflow bytes).

---

## (e) R4 — Concurrent scroll while writing

**Question.** What changes when the user is actively scrolling through old output while new data arrives at full speed?

**Answer (prose).** Build a real `Screen` (lines=5, cols=10, scrollback=100 ⇒ history `ynum = MAX(100, 5) = 100` via `alloc_historybuf` `[kitty/screen.c:130]`), fill 50 lines, then scroll back one page. `screen.scroll(SCROLL_PAGE, True)` scrolls up by `lines - 1 = 4` `[kitty/screen.c:4091-4118]`, so `scrolled_by = 4`. While scrolled back, feeding 10 more lines increments `history_line_added_count` to 10 (each `INDEX_UP` with `add_to_history` bumps it `[kitty/screen.c:1560]`, within the ingest macro `[kitty/screen.c:1552-1567]`) but `scrolled_by` stays 4 until a render. The canonical render step `screen_update_only_line_graphics_data` `[kitty/screen.c:2709-2737]` (exposed as the no-arg Python method `update_only_line_graphics_data` `[kitty/screen.c:4867]`) applies the re-clamp at line 2716: `scrolled_by = MIN(scrolled_by + history_line_added_count, count) = MIN(4 + 10, 56) = 14`, keeping the SAME old content pinned in view as new lines push it back, then resets `history_line_added_count` to 0 (`screen_reset_dirty` `[kitty/screen.c:2598-2601]`). Continuing to feed eventually SATURATES `scrolled_by` at `count`, which itself caps at `ynum = 100`; beyond that the oldest content scrolls away and evicts (the view can no longer anchor further back).

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

**Observed vs Inferred.** **Observed:** all four states of `scrolled_by` (0 after fill, 4 after scroll-back, still 4 during pre-render while `history_line_added_count` accumulates to 10, then 14 after the re-clamp), the `history_line_added_count` accumulation and reset, and the saturation at `scrolled_by == count == ynum == 100`. `scrolled_by`/`history_line_added_count`/`count` are canonical READONLY members `[kitty/screen.c:4902-4903]`. **Inferred:** nothing.

**Stability.** RUN1 == RUN2 for every value.

---

## (f) R5 — Runtime allocation, wrapping, and retention at scale

**Question.** How do the underlying memory structures evolve as pressure builds — allocation (segment `calloc` plus ring-buffer growth), wrapping (circular indexing plus line reflow), and retention (the `ynum` cap on Tier 1, the `maximum_size` cap on Tier 2, and eviction/overwrite)?

**Answer (prose).** With `HistoryBuf(200000, 80)` (Tier 2 disabled), pushing 205,000 lines pins `count` at `ynum = 200000` — the Tier-1 **retention cap** (`historybuf_push` `[kitty/history.c:276-286]` overwrites in place once full, evicting the oldest each push). `num_segments` reaches `98 == ceil(200000 / 2048)`. Total Tier-1 allocation is `98 * 5,244,928 = 514,002,944` bytes (490.2 MiB), which the process RSS delta tracks closely (~465–490 MiB; this range is the only non-deterministic figure — it is reported as approximate and attributed to lazy `calloc` page commitment). `tracemalloc` shows only ~5 KB of Python-object delta because the large `calloc` blocks are C allocations invisible to `tracemalloc`. **Wrapping/retention:** circular indexing `index_of` `[kitty/history.c:152-158]` maps logical→physical with `(start_of_data + idx) % ynum` (reverse-indexed, line 0 = newest); the Tier-1 retention cap is `ynum`, and the Tier-2 retention cap is `maximum_size` with FIFO overwrite (from R2/R3).

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
RSS delta = 488504 KB (~477.1 MiB) ; tracemalloc py-delta = 5102 B (C calloc untracked)
carve milestones {count:(num_segments,rss_KB)} = {1: (1, 0), 2048: (1, 0), 2049: (2, 0), 4096: (2, 0), 4097: (3, 140), 200000: (98, 488504)}

===== R5 ALLOCATION/RETENTION [RUN2] =====
HistoryBuf(200000,80) pagerhist_ptr=0 (0=Tier2 disabled)
per-segment calloc = 5244928 B (5.002 MiB)
pushed 205000 -> count=200000 RETENTION count==ynum? True num_segments=98
ceil(200000/2048)=98 == num_segments? True
total seg alloc = 98*5244928 = 514002944 B (490.2 MiB)
RSS delta = 475696 KB (~464.5 MiB) ; tracemalloc py-delta = 5070 B (C calloc untracked)
carve milestones {count:(num_segments,rss_KB)} = {1: (1, 0), 2048: (1, 0), 2049: (2, 0), 4096: (2, 0), 4097: (3, 0), 200000: (98, 475696)}
```

**Observed vs Inferred.** **Observed:** the `count == ynum` retention cap (pinned at 200000), `num_segments == 98`, total `calloc` bytes (514,002,944), the RSS delta (approximate range), the `tracemalloc` delta (~5 KB, confirming the C `calloc` blocks are untracked), and the carve milestones. **Inferred:** nothing. The structural values (`count`, `num_segments`, `total seg alloc`) are IDENTICAL across both runs (deterministic); RSS differs (488504 vs 475696 KB) and is reported as an approximate ~465–490 MiB due to lazy page commitment — reproduced, NOT engineered away.

**Stability.** Structural values are identical RUN1 == RUN2. RSS is approximate (~465–490 MiB) with cause stated (lazy `calloc` paging).

---

## Synthesis: "the quiet interaction" and "smoothness versus hesitation"

**The quiet interaction (R2/R5).** The two tiers cooperate in exactly one place — `historybuf_push` `[kitty/history.c:276-286]`. While Tier 1 is filling (`count < ynum`) the two tiers do not interact at all; Tier 2 stays empty (R2 BEFORE: `pagerhist_as_text()` is `''`). The interaction begins precisely when Tier 1 saturates (`count == ynum`): every subsequent push **evicts** the oldest Tier-1 line, **serializes** it to an ANSI byte stream via `line_as_ansi` `[kitty/line.c:338]`, and **ingests** it into the Tier-2 ring — Tier 1 → serialize → Tier 2, then `start_of_data` advances. This is "quiet" because it is invisible to the Python API surface except through `pagerhist_as_text()`/`pagerhist_as_bytes()`; `count` never changes once saturated (R2), and the interactively-scrollable view keeps working off Tier 1 while the overflow archive silently accretes in Tier 2.

**Smoothness versus hesitation (R1/R3).** Most of the flood is seamless — pushing lines within a segment and evicting/overwriting once full are constant-time in-place operations. The discontinuities ("hesitations") are:

- **The segment carve (R1)** — a `realloc` of the segments array plus a multi-MiB `calloc` (5,244,928 bytes for `xnum=80`) at each 2048-line crossing (`add_segment` `[kitty/history.c:18-29]`). Observed at pushes #2049 and #4097. This is the largest single allocation event.
- **Tier-1 saturation onset (R2/R4)** — the exact push where `count == ynum` flips behavior from "grow" to "evict"; from that push onward every write does eviction work (and, if Tier 2 is enabled, serialization + ring write).
- **The 1 MB ring-growth steps (R3a)** — Tier 2 grows in ≥1 MB chunks (`pagerhist_extend` `[kitty/history.c:89-101]`) at pushes #5126/#10241/#15356/#20471; each step allocates a new ring and copies the used bytes across.
- **The FIFO overwrite ceiling (R3b)** — once the ring hits `maximum_size`, growth stops and the oldest bytes are overwritten in place `[3rdparty/ringbuf/ringbuf.h:127-136]`; this is seamless (no allocation) but silently discards the oldest archived bytes.
- **The single-line hard-drop (R3c)** — the sharpest discontinuity: a serialized line larger than `maximum_size` is silently dropped whole (`pagerhist_write_bytes` returns `false` `[kitty/history.c:218-226]`), not truncated.

In short: the steady state is smooth (in-place eviction/overwrite), while allocation events (segment carve, ring growth) and the two hard limits (FIFO ceiling, single-line drop) are the moments where the system "hesitates" or behaves differently than a naïve unbounded model would predict — all grounded in the observations above.

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
| `pagerhist_write_bytes` (`if (sz > maximum_size) return false`) | `kitty/history.c:218-226` | Single-line hard-drop |
| `pagerhist_push` (writes `\x1b[m`, line, `\r`/`\n`) | `kitty/history.c:259-272` | Serialize evicted line into Tier 2 |
| `historybuf_push` (spill at `count == ynum`, advance `start_of_data`) | `kitty/history.c:276-286` | Inter-tier cooperation point |
| `push` (Python; takes a `Line`) | `kitty/history.c:337` | Canonical push entry |
| `pagerhist_rewrap_to` | `kitty/history.c:392` | Tier-2 reflow on resize |
| `pagerhist_as_bytes` | `kitty/history.c:461` | Read Tier-2 bytes |
| `pagerhist_as_text` | `kitty/history.c:486` | Read Tier-2 text |
| method table | `kitty/history.c:542-552` | Python-visible `HistoryBuf` methods |
| `rewrap_needed = true` set | `kitty/history.c:604-608` | Deferred reflow flag (inferred path) |
| `HistoryBufSegment {gpu_cells, cpu_cells, line_attrs}` | `kitty/data-types.h:262-266` | Segment struct |
| `PagerHistoryBuf {ringbuf, maximum_size, rewrap_needed}` | `kitty/data-types.h:268-272` | Tier-2 struct |
| `HistoryBuf {…xnum, ynum, num_segments; segments; pagerhist; line; start_of_data, count}` | `kitty/data-types.h:283-290` | Tier-1 struct |
| `static_assert` GPUCell==20 / CPUCell==12 / LineAttrs | `kitty/data-types.h:215-239` | Per-cell byte sizes (32 B/cell) |
| `alloc_historybuf(MAX(scrollback, lines), …)` | `kitty/screen.c:130` | `ynum = MAX(scrollback, lines)` |
| `INDEX_UP` macro (`history_line_added_count++` at `:1560`) | `kitty/screen.c:1552-1567` | Canonical ingest path |
| `screen_reset_dirty` (resets `history_line_added_count`) | `kitty/screen.c:2598-2601` | Post-render reset |
| `screen_update_only_line_graphics_data` (re-clamp at `:2716`) | `kitty/screen.c:2709-2737` | `scrolled_by = MIN(scrolled_by + hlac, count)` |
| `update_only_line_graphics_data` method | `kitty/screen.c:4867` | No-arg Python render step |
| `historybuf` / `scrolled_by` READONLY members | `kitty/screen.c:4902-4903` | Canonical readonly signals |
| `screen_history_scroll` (`SCROLL_PAGE` → `lines - 1`) | `kitty/screen.c:4091-4118` | Scroll-back mechanics |
| `line_as_ansi` | `kitty/line.c:338` | Serialize a line to ANSI bytes |
| FIFO overwrite ("old data will simply be overwritten") | `3rdparty/ringbuf/ringbuf.h:127-136` | Tier-2 overflow semantics |
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

*All observations use the canonical `HistoryBuf`/`Screen` path imported from `kitty.fast_data_types`. The two read-only helpers used to read internal counters — `gdb` attached to the live PID, and a `ctypes` read of the live `PyObject` struct at validated offsets — are reads of the real production structs, not bypasses. The investigation was strictly read-only: temporary observation/`gdb` scripts and all build artifacts were removed after capture, and the repository is byte-for-byte unchanged apart from this document.*
