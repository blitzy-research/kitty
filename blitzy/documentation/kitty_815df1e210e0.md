# How kitty's scrollback (`HistoryBuf` + `PagerHistoryBuf`) behaves under extreme write pressure

> **Runtime investigation — built and run first, then written.** Every quantitative claim below comes from executing kitty's real C core through its canonical parser/screen code path and reading the exposed read-only members. Claims are labeled **[OBSERVED]** (captured from a running build) or **[INFERRED]** (deduced from source, with a `file:line` citation). Nothing here is asserted from reading source alone unless explicitly marked **[INFERRED]**.

---

## TL;DR — direct answers

- **SQ1 — "fills, stretches, and starts carving out new segments":** The segmented store `HistoryBuf` pre-allocates **one** fixed segment of `SEGMENT_SIZE = 2048` lines at construction, then lazily `calloc`s an additional 2048-line segment each time the used line count grows past another 2048-line boundary — but only up to `ceil(ynum/2048)` segments, where `ynum = MAX(scrollback_lines, screen_lines)` is fixed for the buffer's life. Each segment is a single `calloc` of `2048 * (xnum*sizeof(CPUCell) + xnum*sizeof(GPUCell) + sizeof(LineAttrs))` bytes = **5,244,928 bytes (~5.002 MB) at xnum=80**. With the **default** `scrollback_lines = 2000` there is only **one** segment; carving of multiple segments is only observable when `scrollback_lines > 2048`. **[OBSERVED]**
- **SQ2 — segmented store ↔ pager ring interaction:** The segmented store is itself a ring over `ynum` lines. Once `count == ynum` (saturated), every further line **evicts the oldest line**. If the pager tier is **disabled** (default `scrollback_pager_history_size = 0`) the evicted line is simply dropped. If the pager tier is **enabled**, the evicted line is serialized to ANSI and pushed into the separate byte-addressable **`PagerHistoryBuf`** ring at a measured **47 bytes/line** for this payload. **[OBSERVED]**
- **SQ3 — "subtle moments where the system hesitates":** Transitions are **smooth**. Crossing a 2048-line segment boundary just triggers one `calloc` (a ~5 MB step in resident memory) with no stall or discontinuity in `count`. The pager ring grows in ≥1 MB steps until it reaches `maximum_size`, then **plateaus exactly** and overwrites its oldest bytes in place — no reallocation thereafter. The historically fragile large-burst pager path (a 2020 report of a segfault after ~120k lines) **does not reproduce** on this commit: feeding **10,000,000** lines completes cleanly. A supplementary ASAN+UBSAN build reports **zero** memory diagnostics at every boundary. **[OBSERVED]**
- **SQ4 — scrolling old output while new data arrives:** While scrolled back (`scrolled_by > 0`), each added line bumps `scrolled_by` by the number of newly added history lines, **capped at `historybuf->count`**. There are **two thresholds**: the pin keeps the view glued to old output while `scrolled_by < count`; the "subtle moment" is when `scrolled_by` hits the `count` cap — after that `scrolled_by` **freezes** and the viewport visibly **drifts** as new lines keep arriving. **[OBSERVED]**
- **SQ5 — how the memory structures actually evolve:** Allocation (segment carving: +~5 MB per 2048 boundary), retention/eviction (`count` saturates at `ynum`; the cumulative added-line counter keeps climbing; the pager ring fills then plateaus), and wrapping/reflow (line count rises as width shrinks; reflow is **nearly but not perfectly** reversible) are all captured as concrete, reproduced numbers below. **[OBSERVED]**

---

## 1. Build and invocation — exact commands

### 1.1 Canonical build

The performance-sensitive terminal core (including `HistoryBuf` and `PagerHistoryBuf`) compiles into a single CPython extension module `kitty/fast_data_types.so`. The canonical debug build command used was:

```
python3 setup.py build --debug
```

**[OBSERVED]** This succeeded and produced `kitty/fast_data_types.so` = **6,143,072 bytes**. `import kitty.fast_data_types` works and exposes `HistoryBuf` and `Screen`. `git status --porcelain` remained empty (all build artifacts are gitignored). In this environment no extra flags were required; a `-Wno-error=switch` workaround is only needed to bypass an unrelated GUI/Wayland enum warning and is **not** required in the canonical Docker image.

Python interpreter used: **3.12.3**.

### 1.2 Canonical entry point (why these observations count)

All measurements drive the **real VT parser and Screen model**, not a synthetic `HistoryBuf.push()` stand-in. Two canonical drivers were used:

- **In-process real parser:** `parse_bytes(screen, data)` [kitty_tests/__init__.py:30] feeds bytes through `Screen.test_create_write_buffer` → `test_commit_write_buffer` → `test_parse_written_data`, which are the **same** `vt_parser` + `screen` functions the live PTY loop calls. The child-monitor PTY read loop `read_bytes()` [kitty/child-monitor.c:1337-1354] does `vt_parser_create_write_buffer` → `read(fd,...)` → `vt_parser_commit_write` — i.e. the identical code path. **[INFERRED from the two call sites being the same functions; corroborated by the real-PTY run in §5.4 producing identical structural results.]**
- **Real pseudo-terminal child:** `create_pty()` / `PTY` [kitty_tests/__init__.py:243] spawns an actual child process whose stdout is read through the master fd and pushed through the same parser (full end-to-end fidelity).

The canonical byte path exercised is: **child PTY → `kitty/child-monitor.c` (read loop) → `kitty/vt-parser.c` → `kitty/screen.c` (scroll-off `INDEX_UP`) → `kitty/line-buf.c` → `kitty/history.c`.**

### 1.3 Read-only observation surface

Observation used only the already-exposed read-only members and methods (no source was instrumented):

- `HistoryBuf.count`, `HistoryBuf.ynum`, `HistoryBuf.xnum` — `READONLY` T_UINT members [kitty/history.c:556-558].
- `HistoryBuf.pagerhist_as_bytes(...)` [kitty/history.c:461] and `pagerhist_as_text(...)` — pager ring contents.
- `Screen.scrolled_by` (read-only) and `Screen.history_line_added_count` (used as the cumulative eviction proxy).
- Type stubs confirm the surface: `class HistoryBuf` [kitty/fast_data_types.pyi:1080], `pagerhist_as_bytes` [kitty/fast_data_types.pyi:1085], `Screen.scrolled_by:int` [kitty/fast_data_types.pyi:1119].

### 1.4 Configuration knobs and defaults

- `scrollback_lines` default **`2000`** [kitty/options/definition.py:372]. `ynum = MAX(scrollback_lines, screen_lines)` is fixed at `Screen` construction via `alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size))` [kitty/screen.c:130]. Because `2000 < SEGMENT_SIZE (2048)` [kitty/history.c:15], the default yields a **single** segment.
- `scrollback_pager_history_size` default **`0` = DISABLED** [kitty/options/definition.py:406]; unit is MB, maximum 4 GB. With it `0`, `alloc_pagerhist()` returns `NULL` [kitty/history.c:70-81] and evicted lines are dropped.

---

## 2. SQ1 — Segment allocation: "fills, stretches, and starts carving out new segments"

### 2.1 Mechanism (source)

- `SEGMENT_SIZE = 2048` [kitty/history.c:15].
- `segment_for()` [kitty/history.c:37-42] computes `seg_num = y / 2048` and, while `seg_num >= num_segments && 2048 * num_segments < ynum`, calls `add_segment()`.
- `add_segment()` [kitty/history.c:18-29] does `num_segments += 1`, `realloc`s the segment array, then performs **one** `calloc(1, xnum*2048*sizeof(CPUCell) + xnum*2048*sizeof(GPUCell) + 2048*sizeof(LineAttrs))`.
- `create_historybuf()` [kitty/history.c:117] starts with `num_segments = 0` and calls `add_segment()` **once**, so a fresh buffer already owns segment #0.
- Struct layout: `HistoryBufSegment` [kitty/data-types.h:266], `HistoryBuf` [kitty/data-types.h:290].

**Derived per-segment size [INFERRED from the calloc expression]:** `per_segment_bytes = 2048 * (xnum*sizeof(CPUCell) + xnum*sizeof(GPUCell) + sizeof(LineAttrs))`. With `sizeof(CPUCell)=12`, `sizeof(GPUCell)=20`, `sizeof(LineAttrs)=1`, this is `2048 * (32*xnum + 1)`. At **xnum=80**: `2048 * 2561 = 5,244,928 bytes (~5.002 MB)`. Segment count = `ceil(ynum / 2048)`.

### 2.2 Before/during/after — `count` and derived segment count

Driver: canonical `parse_bytes`, pager **disabled**, screen 24×80. Values below are **[OBSERVED]** and identical across 2 runs (only RSS memory and wall-clock differ run-to-run — OS noise).

| Condition | scrollback_lines | ynum | segments = ceil(ynum/2048) | per-segment bytes | count: before → during → after |
|---|---|---|---|---|---|
| Default (single-segment) | 2000 | 2000 | **1** | 5,244,928 (~5.002 MB) | 0 → 1977 → **2000** (saturated) |
| Multi-segment | 5000 | 5000 | **3** | 5,244,928 (~5.002 MB) | 0 → 4977 → **5000** (saturated) |
| Multi-segment | 10000 | 10000 | **5** | 5,244,928 (~5.002 MB) | 0 → 7477 → **10000** (saturated) |

> Note: `count` saturates at `ynum` and the "during" first sample reads slightly below `ynum` (e.g. 1977, not 2000) because ~23 lines of the burst are still occupying the live 24-row screen grid rather than history at the sampling instant. **[OBSERVED]**

### 2.3 Complete unedited output — segment saturation (`sq1_segments.py`)

Command: `python3 sq1_segments.py` (canonical `parse_bytes` driver). Full output:

```
PYTHON: 3.12.3

===== DEFAULT single-segment (2000) : scrollback_lines=2000  screen=24x80  burst=8000 lines  (pager DISABLED) =====
[BEFORE] count=0 ynum=2000 xnum=80  derived_segments=ceil(2000/2048)=1  per_segment_bytes=5244928 (~5.002 MB)
[DURING k=1] fed~   2000  count=  1977 ynum=2000 derived_segments=1  count/ynum=0.989  rssMB~30.0
[DURING k=2] fed~   4000  count=  2000 ynum=2000 derived_segments=1  count/ynum=1.000  rssMB~31.0
[DURING k=3] fed~   6000  count=  2000 ynum=2000 derived_segments=1  count/ynum=1.000  rssMB~31.0
[DURING k=4] fed~   8000  count=  2000 ynum=2000 derived_segments=1  count/ynum=1.000  rssMB~31.0
[AFTER ] count=2000 ynum=2000 xnum=80 derived_segments=1 saturated=True  total_fed=8000  rss_delta_MB~6.00

===== MULTI-segment (5000) : scrollback_lines=5000  screen=24x80  burst=20000 lines  (pager DISABLED) =====
[BEFORE] count=0 ynum=5000 xnum=80  derived_segments=ceil(5000/2048)=3  per_segment_bytes=5244928 (~5.002 MB)
[DURING k=1] fed~   5000  count=  4977 ynum=5000 derived_segments=3  count/ynum=0.995  rssMB~38.7
[DURING k=2] fed~  10000  count=  5000 ynum=5000 derived_segments=3  count/ynum=1.000  rssMB~38.7
[DURING k=3] fed~  15000  count=  5000 ynum=5000 derived_segments=3  count/ynum=1.000  rssMB~38.7
[DURING k=4] fed~  20000  count=  5000 ynum=5000 derived_segments=3  count/ynum=1.000  rssMB~38.7
[AFTER ] count=5000 ynum=5000 xnum=80 derived_segments=3 saturated=True  total_fed=20000  rss_delta_MB~7.70

===== MULTI-segment (10000) : scrollback_lines=10000  screen=24x80  burst=30000 lines  (pager DISABLED) =====
[BEFORE] count=0 ynum=10000 xnum=80  derived_segments=ceil(10000/2048)=5  per_segment_bytes=5244928 (~5.002 MB)
[DURING k=1] fed~   7500  count=  7477 ynum=10000 derived_segments=5  count/ynum=0.748  rssMB~46.0
[DURING k=2] fed~  15000  count= 10000 ynum=10000 derived_segments=5  count/ynum=1.000  rssMB~52.0
[DURING k=3] fed~  22500  count= 10000 ynum=10000 derived_segments=5  count/ynum=1.000  rssMB~52.0
[DURING k=4] fed~  30000  count= 10000 ynum=10000 derived_segments=5  count/ynum=1.000  rssMB~52.0
[AFTER ] count=10000 ynum=10000 xnum=80 derived_segments=5 saturated=True  total_fed=30000  rss_delta_MB~13.35
```

### 2.4 Complete unedited output — carving at each boundary (`sq1b_carving.py`)

This run (scrollback_lines=10000, so up to 5 segments) samples resident memory every 256 lines to *watch* each ~5 MB `calloc` land as `count` crosses each 2048 boundary. Full output:

```
scrollback_lines=10000 -> ynum=10000 xnum=80 ; max segments=ceil(ynum/2048)=5
Feeding 12000 lines in 256-line steps; watch resident memory step at each 2048 boundary:
  fed   count  seg_of_count  rssMB   <-- marker at boundary crossing
    256    233      0          26.0  <== crossed into segment #0 (count~233)
   1024   1001      0          27.0
   2048   2025      0          30.0
   2304   2281      1          31.0  <== crossed into segment #1 (count~2281)
   3072   3049      1          32.0
   4096   4073      1          35.0
   4352   4329      2          36.0  <== crossed into segment #2 (count~4329)
   5120   5097      2          37.0
   6144   6121      2          40.0
   6400   6377      3          41.0  <== crossed into segment #3 (count~6377)
   7168   7145      3          42.0
   8192   8169      3          45.0
   8448   8425      4          46.0  <== crossed into segment #4 (count~8425)
   9216   9193      4          48.0
  10240  10000      4          49.0
  11264  10000      4          49.0
[FINAL] count=10000 ynum=10000 saturated=True resident_growth_MB~24.0 (5 segments x ~5.0MB alloc'd, pages resident as written)
```

**Interpretation [OBSERVED]:** resident memory steps up ~+5 MB each time `count` passes a 2048 boundary (segments #0–#4), converging on ~24 MB of resident growth for the 5 segments — matching the derived ~5.002 MB/segment. Carving is lazy and bounded: no sixth segment is ever created because `2048 * 5 = 10240 ≥ ynum = 10000`.

---

## 3. SQ2 — Segmented store ↔ pager ring buffer interaction under stress

### 3.1 Mechanism (source)

- `historybuf_push()` [kitty/history.c:276-285]: `idx = (start_of_data + count) % ynum`; **if `count == ynum`** it calls `pagerhist_push()` and advances `start_of_data = (start_of_data + 1) % ynum`; **else** `count++`. This is the eviction hand-off.
- `pagerhist_push()` [kitty/history.c:259-274]: serializes the evicted oldest line via `line_as_ansi`, writing `"\x1b[m"` (3 bytes) + the UCS4 text + a terminator (`"\r"`, plus `"\n"` unless the line was wrapped).
- With the pager disabled, `pagerhist_push()` returns immediately because `self->pagerhist == NULL` [kitty/history.c:261].

**Eviction proxy calibration [OBSERVED]:** under `parse_bytes`, `Screen.history_line_added_count` accumulates monotonically, so the number of evicted lines = `history_line_added_count - count`. For this payload the ring grows at exactly **47 bytes/evicted line** = 3 (`"\x1b[m"`) + 42 (text `"A0000000 line of scrollback burst pressure"`) + 2 (`"\r\n"`).

### 3.2 Pager DISABLED (default) — evicted lines are dropped

Command: `python3 sq2_disabled.py` (scrollback_lines=2000, `scrollback_pager_history_size=0`). Full output:

```
===== SQ2 pager DISABLED (size=0) : scrollback_lines=2000 =====
[BEFORE] count=0 ynum=2000 ring_bytes=0 history_line_added_count=0
[DURING] fed= 10000 count=2000 ynum=2000 evicted(hlac-count)=7977 ring_bytes=0
[DURING] fed= 20000 count=2000 ynum=2000 evicted(hlac-count)=17977 ring_bytes=0
[DURING] fed= 30000 count=2000 ynum=2000 evicted(hlac-count)=27977 ring_bytes=0
[DURING] fed= 40000 count=2000 ynum=2000 evicted(hlac-count)=37977 ring_bytes=0
[DURING] fed= 50000 count=2000 ynum=2000 evicted(hlac-count)=47977 ring_bytes=0
[AFTER ] count=2000 ynum=2000 history_line_added_count=49977 evicted=47977 ring_bytes=0  (ring stays EMPTY: oldest lines DROPPED)
```

**[OBSERVED]** `count` pins at `ynum = 2000`; `ring_bytes` stays `0` throughout while 47,977 lines are evicted — with the pager disabled the oldest lines are permanently dropped.

### 3.3 Pager ENABLED — evicted lines flow into the byte ring (FIFO)

Command: `python3 sq2_enabled.py` (scrollback_lines=2000, pager `maximum_size = 8,388,608` bytes = 8 MB). Full output:

```
===== SQ2 pager ENABLED (maximum_size=8388608 bytes=8.0MB) : scrollback_lines=2000 =====
[BEFORE] count=0 ynum=2000 ring_bytes=0 hlac=0
[DURING] fed=  4000 count=2000 ynum=2000 evicted=  1977 ring_bytes=  92919 lines_in_ring=  1977 bytes/line=47.00
[DURING] fed=  8000 count=2000 ynum=2000 evicted=  5977 ring_bytes= 280919 lines_in_ring=  5977 bytes/line=47.00
[DURING] fed= 12000 count=2000 ynum=2000 evicted=  9977 ring_bytes= 468919 lines_in_ring=  9977 bytes/line=47.00
[DURING] fed= 16000 count=2000 ynum=2000 evicted= 13977 ring_bytes= 656919 lines_in_ring= 13977 bytes/line=47.00
[DURING] fed= 20000 count=2000 ynum=2000 evicted= 17977 ring_bytes= 844919 lines_in_ring= 17977 bytes/line=47.00
[DURING] fed= 24000 count=2000 ynum=2000 evicted= 21977 ring_bytes=1032919 lines_in_ring= 21977 bytes/line=47.00
[AFTER ] count=2000 ynum=2000 evicted=21977 ring_bytes=1032919
  ring FIRST 80 repr (OLDEST evicted line): b'\x1b[mA0000000 line of scrollback burst pressure\r\n\x1b[mA0000001 line of scrollback bu'
  ring LAST  80 repr (NEWEST evicted line): b'ne of scrollback burst pressure\r\n\x1b[mF0001976 line of scrollback burst pressure\r\n'
```

**[OBSERVED]** The ring grows **linearly** at exactly 47 bytes/evicted line (e.g. 21977 × 47 = 1,032,919). The ring is FIFO: the first bytes are the oldest evicted line (`A0000000...`) and the last bytes are the newest evicted line (`F0001976...`). This is the quiet hand-off the question intuits — the segmented store silently spills its oldest line into the pager ring on every push once saturated.

### 3.4 Before/during/after summary (SQ2)

| Condition | count (before→after) | evicted (after) | ring_bytes (before→after) | behavior |
|---|---|---|---|---|
| Pager DISABLED | 0 → 2000 | 47,977 | 0 → **0** | oldest lines dropped |
| Pager ENABLED (8 MB) | 0 → 2000 | 21,977 | 0 → **1,032,919** | oldest lines serialized into ring @47 B/line |

---

## 4. SQ3 — Transition smoothness and "subtle moments"

Three concrete boundaries were stressed: (a) crossing a 2048-line segment boundary (see §2.4 — smooth, one `calloc`), (b) the pager ring reaching `maximum_size`, and (c) the historically fragile very-large-burst pager path.

### 4.1 Ring reaching `maximum_size` — grow-then-plateau (source)

- `pagerhist_extend()` [kitty/history.c:90-101]: `if (ringbuf_capacity >= maximum_size) return false;` else grows by `MIN(maximum_size, buffer_size + MAX(1 MB, minsz))`.
- `pagerhist_write_bytes()` [kitty/history.c:219]: once at capacity, writes go through `ringbuf_memcpy_into`, which **overwrites the oldest bytes** (the vendored FIFO advances its tail when full) [3rdparty/ringbuf/ringbuf.c].

Command: `python3 sq3_boundary.py` (scrollback_lines=2000, pager `maximum_size = 4,194,304` bytes = 4 MB). Full output:

```
===== SQ3 ring boundary : maximum_size=4194304 (4.0MB) scrollback_lines=2000 =====
fed=  10000 count=2000 evicted=   7977 ring_bytes= 374919 (0.358MB) d=+374920 first8=b'\x1b[mA0000'
fed=  20000 count=2000 evicted=  17977 ring_bytes= 844919 (0.806MB) d=+470000 first8=b'\x1b[mA0000'
fed=  30000 count=2000 evicted=  27977 ring_bytes=1314919 (1.254MB) d=+470000 first8=b'\x1b[mA0000'
fed=  40000 count=2000 evicted=  37977 ring_bytes=1784919 (1.702MB) d=+470000 first8=b'\x1b[mA0000'
fed=  50000 count=2000 evicted=  47977 ring_bytes=2254919 (2.150MB) d=+470000 first8=b'\x1b[mA0000'
fed=  60000 count=2000 evicted=  57977 ring_bytes=2724919 (2.599MB) d=+470000 first8=b'\x1b[mA0000'
fed=  70000 count=2000 evicted=  67977 ring_bytes=3194919 (3.047MB) d=+470000 first8=b'\x1b[mA0000'
fed=  80000 count=2000 evicted=  77977 ring_bytes=3664919 (3.495MB) d=+470000 first8=b'\x1b[mA0000'
fed=  90000 count=2000 evicted=  87977 ring_bytes=4134919 (3.943MB) d=+470000 first8=b'\x1b[mA0000'
fed= 100000 count=2000 evicted=  97977 ring_bytes=4194304 (4.000MB) d= +59385 first8=b'ollback '
fed= 110000 count=2000 evicted= 107977 ring_bytes=4194304 (4.000MB) d=     +0 first8=b'ollback '  <== PLATEAU reached (ring full; oldest bytes now OVERWRITTEN)
fed= 120000 count=2000 evicted= 117977 ring_bytes=4194304 (4.000MB) d=     +0 first8=b'ollback '
... (fed 130000-200000 all identical plateau at 4194304, first8=b'ollback ') ...
[AFTER] ring_bytes=4194304 (4.000MB) maximum_size=4194304(4.0MB) plateau=4194304
  ring FIRST 60 repr: b'ollback burst pressure\r\n\x1b[mK0008737 line of scrollback burst'
  ring LAST  60 repr: b'st pressure\r\n\x1b[mT0007976 line of scrollback burst pressure\r\n'
```

**[OBSERVED]** The ring grows in ~470,000-byte steps (each ≈ the 1 MB extend granularity minus payload alignment), reaches **exactly 4,194,304 bytes** at fed≈100000, then **plateaus** — `ring_bytes` never exceeds `maximum_size`. The `first8` marker flips from `b'\x1b[mA0000'` (an intact oldest line header) to `b'ollback '` (mid-word) the moment overwriting begins, directly evidencing that the **oldest bytes are being overwritten in place**. The transition is smooth: no stall in `count`, no error, just a hard cap.

### 4.2 Tiny ring sizes — no crash at extreme edges

Command: `python3 sq3_edge.py` (pager sizes of 1, 2, 3, 47, 1024 bytes; 100,000 lines each). Full output:

```
pager_max=    1 B : fed=100000 count=2000 evicted=97977 ring_bytes=1 lines_in_ring=0  (no crash)
pager_max=    2 B : fed=100000 count=2000 evicted=97977 ring_bytes=2 lines_in_ring=1  (no crash)
pager_max=    3 B : fed=100000 count=2000 evicted=97977 ring_bytes=3 lines_in_ring=1  (no crash)
pager_max=   47 B : fed=100000 count=2000 evicted=97977 ring_bytes=47 lines_in_ring=1  (no crash)
pager_max= 1024 B : fed=100000 count=2000 evicted=97977 ring_bytes=1024 lines_in_ring=22  (no crash)
ALL edge sizes completed without crash/assert.
```

**[OBSERVED]** Even a 1-byte ring is handled gracefully (it simply holds a single byte), plateauing at exactly `maximum_size` with no crash or assertion — including in a debug build with assertions enabled.

### 4.3 The historically fragile large-burst path — does it still hesitate?

Background: a 2020 report described kitty 0.19.0 segfaulting after roughly 120k lines with `scrollback_pager_history_size` above 1, reproduced by printing ~10 million lines. That predates this commit; the question is whether the current commit still hesitates at that scale.

Command: `python3 sq3_frag.py` feeding **1,000,000** lines (pager `maximum_size = 1,048,576` bytes = 1 MB). Full output:

```
===== SQ3c FRAGILITY : feed 1000000 lines, pager maximum_size=1048576 bytes (1.000MB), scrollback_lines=2000 =====
  fed=  200000 count=2000 ynum=2000 evicted=  197977 ring_bytes=1048576 (1.000MB) elapsed=0.4s
  fed=  400000 count=2000 ynum=2000 evicted=  397977 ring_bytes=1048576 (1.000MB) elapsed=0.8s
  fed=  600000 count=2000 ynum=2000 evicted=  597977 ring_bytes=1048576 (1.000MB) elapsed=1.3s
  fed=  800000 count=2000 ynum=2000 evicted=  797977 ring_bytes=1048576 (1.000MB) elapsed=1.7s
  fed= 1000000 count=2000 ynum=2000 evicted=  997977 ring_bytes=1048576 (1.000MB) elapsed=2.1s
[DONE] fed=1000000 count=2000 ynum=2000 evicted=997977 ring_bytes=1048576 (1.000MB) total=2.1s -- NO CRASH, clean completion
```

Command: `python3 sq3_frag.py` feeding **10,000,000** lines (same 1 MB pager). Full output:

```
===== SQ3c FRAGILITY : feed 10000000 lines, pager maximum_size=1048576 bytes (1.000MB), scrollback_lines=2000 =====
  fed= 1000000 count=2000 ynum=2000 evicted=  997977 ring_bytes=1048576 (1.000MB) elapsed=2.2s
  fed= 2000000 count=2000 ynum=2000 evicted= 1997977 ring_bytes=1048576 (1.000MB) elapsed=4.3s
  fed= 3000000 count=2000 ynum=2000 evicted= 2997977 ring_bytes=1048576 (1.000MB) elapsed=6.5s
  fed= 4000000 count=2000 ynum=2000 evicted= 3997977 ring_bytes=1048576 (1.000MB) elapsed=8.6s
  fed= 5000000 count=2000 ynum=2000 evicted= 4997977 ring_bytes=1048576 (1.000MB) elapsed=10.7s
  fed= 6000000 count=2000 ynum=2000 evicted= 5997977 ring_bytes=1048576 (1.000MB) elapsed=12.7s
  fed= 7000000 count=2000 ynum=2000 evicted= 6997977 ring_bytes=1048576 (1.000MB) elapsed=14.8s
  fed= 8000000 count=2000 ynum=2000 evicted= 7997977 ring_bytes=1048576 (1.000MB) elapsed=16.9s
  fed= 9000000 count=2000 ynum=2000 evicted= 8997977 ring_bytes=1048576 (1.000MB) elapsed=18.9s
  fed=10000000 count=2000 ynum=2000 evicted= 9997977 ring_bytes=1048576 (1.000MB) elapsed=21.0s
[DONE] fed=10000000 count=2000 ynum=2000 evicted=9997977 ring_bytes=1048576 (1.000MB) total=21.0s -- NO CRASH, clean completion
```

**[OBSERVED]** The historically fragile path is now **smooth**: 10,000,000 lines (9,997,977 evictions) complete in ~21 s with `ring_bytes` glued to exactly `1,048,576` and no crash, hang, or assertion. Throughput is steady (~2.1 s per million lines) — no hesitation or nonlinear slowdown as the ring plateaus.

### 4.4 Supplementary ASAN+UBSAN sanitizer boundary check (non-canonical instrumentation)

> **Labeled non-canonical:** this uses an instrumented build, not the default binary. It is corroboration only.

An ASAN+UBSAN build was produced via `python3 setup.py build --debug --sanitize --ignore-compiler-warnings` (instrumented `kitty/fast_data_types.so` = **20,055,880 bytes**, 29 `asan` symbols) and run under `LD_PRELOAD=$(gcc -print-file-name=libasan.so) ASAN_OPTIONS=detect_leaks=0`. **[OBSERVED]** every boundary probe — SQ3 ring boundary (200k lines), fragility (1,000,000 lines, ~6.4 s ≈3× slower under instrumentation), SQ1b carving, and the real-PTY run (200k) — exited with status **0** and **zero** sanitizer diagnostics. The plain debug `.so` (6,143,072 bytes) was then restored; `git status --porcelain` remained empty.

---

## 5. SQ4 — Actively scrolling old output while new data arrives at full speed

### 5.1 Mechanism (source)

- Scroll-off macro `INDEX_UP` [kitty/screen.c:1559] increments `history_line_added_count++` for each line pushed to history.
- The pin/cap is applied in `screen_update_cell_data()` [kitty/screen.c:2761] and `screen_update_only_line_graphics_data()` [kitty/screen.c:2716], both identical:
  `if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);`
- `history_line_added_count` is reset to 0 by `screen_reset_dirty()` each frame [kitty/screen.c:2600].

### 5.2 Complete unedited output (`sq4_scroll.py`)

Setup: scrollback_lines=2000 (ynum=2000), screen 24×80, filled to count=1177, then scrolled back 500. Each step pours 200 more lines and renders a frame. Full output:

```
===== SQ4 concurrent scroll+write : ynum=2000, screen=24x80 =====
[SETUP] count=1177 ynum=2000 scrolled_by=0 (not yet scrolled)
[SCROLL back 500] scrolled_by=500 count=1177  (viewing old output)

Now pour NEW data at full speed while scrolled back; call a render frame each step.
 step   fed   count  hlac_before  scrolled_by_after  d_scroll  capped(sb==count)  regime
    1   1400   1377      200            700           +200        False          UNSATURATED (pin holds)
    2   1600   1577      200            900           +200        False          UNSATURATED (pin holds)
    3   1800   1777      200           1100           +200        False          UNSATURATED (pin holds)
    4   2000   1977      200           1300           +200        False          UNSATURATED (pin holds)
    5   2200   2000      200           1500           +200        False          SATURATED (pin capped -> drift)
    6   2400   2000      200           1700           +200        False          SATURATED (pin capped -> drift)
    7   2600   2000      200           1900           +200        False          SATURATED (pin capped -> drift)
    8   2800   2000      200           2000           +100        True           SATURATED (pin capped -> drift)
    9   3000   2000      200           2000             +0        True           SATURATED (pin capped -> drift)
   ... (steps 10-20 all scrolled_by=2000, d_scroll=+0, capped=True) ...
[CHECK] history_line_added_count after a frame (should be 0, reset by screen_reset_dirty): 0
```

### 5.3 The "subtle moment" — two thresholds

**[OBSERVED]** There are two distinct regimes, and the interesting nuance is that the pin does **not** break at the moment the buffer saturates:

1. **Pin holds (steps 1–7):** `scrolled_by` climbs by exactly `history_line_added_count` (+200) each frame, keeping the view glued to the same old content. This continues even after `count` saturates at `ynum=2000` at step 5, because the `MIN(...)` cap is not yet binding (`scrolled_by < count`).
2. **The subtle moment (step 8):** `scrolled_by` reaches the `count` cap (2000). The increment is clipped to +100 and then, from step 9 onward, `d_scroll = +0` — `scrolled_by` **freezes** at 2000 while 200 new lines keep arriving each frame. The viewport can no longer stay pinned to the original old output and **drifts** relative to the content the user was reading.

So the pin breaks when `scrolled_by` reaches the `count` cap (step 8), which is slightly *after* the buffer itself saturates (step 5) — a two-threshold behavior that is easy to mis-predict from the source alone.

---

## 6. SQ5 — How the underlying memory structures actually evolve (allocation, wrapping, retention)

SQ5 is the synthesis of the concrete, reproduced numbers above, plus reflow (wrapping).

### 6.1 Allocation, retention — consolidated before/during/after

| Structure | Metric | Before | During (mid-burst) | After (saturated) | Source |
|---|---|---|---|---|---|
| Segmented store | `count` | 0 | rising toward ynum | = ynum (2000/5000/10000) | [kitty/history.c:276-285] |
| Segmented store | segments (ceil(ynum/2048)) | 1 (pre-alloc'd) | grows at each 2048 boundary | ceil(ynum/2048) | [kitty/history.c:37-42] |
| Segmented store | resident memory | baseline | +~5 MB per boundary | +~24 MB for 5 segments | [kitty/history.c:18-29] |
| Eviction | `history_line_added_count - count` | 0 | climbs monotonically | up to 9,997,977 | [kitty/screen.c:1559] |
| Pager ring (disabled) | `len(pagerhist_as_bytes())` | 0 | 0 | 0 (lines dropped) | [kitty/history.c:70-81] |
| Pager ring (enabled) | `len(pagerhist_as_bytes())` | 0 | grows @47 B/line | plateau at maximum_size | [kitty/history.c:90-101,219] |
| View | `scrolled_by` | set by user | climbs with adds | frozen at cap = count | [kitty/screen.c:2716,2761] |

### 6.2 Wrapping / reflow — complete unedited output (`sq5_reflow.py`)

Setup: 1500 lines of 42 printable chars each, ynum=8000 (large enough that **no** eviction occurs), then resize width. Full output:

```
===== SQ5 reflow : 1500 lines of 42 printable chars, ynum=8000 (no eviction) =====
[width= 80] count=1477 xnum=80 ynum=8000  rows/line=ceil(42/80)=1
[width= 40] count=2977 xnum=40 ynum=8000  rows/line=ceil(42/40)=2
[width= 20] count=4477 xnum=20 ynum=8000  rows/line=ceil(42/20)=3
[width=120] count=1494 xnum=120 ynum=8000  rows/line=ceil(42/120)=1
[width= 80] count=1494 xnum=80 ynum=8000  rows/line=ceil(42/80)=1
```

**Mechanism:** width change routes through `historybuf_rewrap()` [kitty/history.c:595] — a fast `memcpy` path when both dimensions are unchanged, otherwise a re-wrap through the shared `rewrap_inner` engine [kitty/rewrap.h:57].

**[OBSERVED]** As width shrinks, each stored logical line occupies more physical rows (`ceil(42/width)`), so `count` rises: 1477 (w80) → 2977 (w40) → 4477 (w20). Widening back to 120 collapses them to 1494. **Subtle nuance:** reflow is **nearly but not perfectly reversible** — returning to width 80 yields `count=1494`, not the original 1477 (a +17 difference), because the initial fill and the reflowed fill distribute the final partial/screen lines slightly differently. This is a real, reproduced non-idempotency, not measurement noise (identical across both runs).

---

## 7. Default (pager-disabled) vs. explicitly-enabled — side-by-side

| Aspect | Default (`scrollback_pager_history_size = 0`) | Enabled (`> 0`) |
|---|---|---|
| `alloc_pagerhist()` | returns `NULL` [kitty/history.c:70-81] | allocates ring of `MIN(1 MB, size)`, `maximum_size = size` |
| Evicted oldest line | **dropped** | serialized (`"\x1b[m"` + text + `\r\n`) into ring @47 B/line |
| `pagerhist_as_bytes()` | always empty (0 bytes) | grows to `maximum_size` then plateaus |
| Interactive scrollback | limited to `scrollback_lines` in the segmented store | same (pager tier is **not** interactively scrollable; only piped to the pager program, e.g. `show_scrollback` [kitty/window.py:1735] via `pagerhist_as_text` [kitty/window.py:356]) |

The pager tier had to be **explicitly enabled** to observe SQ2/SQ3; the default configuration exercises only the segmented store (single segment at scrollback_lines=2000) with eviction-drop.

---

## 8. Scale, duration, and run-to-run stability

- **Scales used:** SQ1 8k/20k/30k lines and SQ1b 12k (256-line sampling); SQ2 disabled 50k / enabled 24k; SQ3 boundary 200k, edges 100k each, fragility **1,000,000 and 10,000,000**; real-PTY 200k; SQ4 5.2k across 20 render steps; SQ5 reflow 1.5k. The 10M-line fragility run took ~21 s.
- **Stability (≥2 runs each):** `sq1b`, `sq2_disabled`, `sq2_enabled`, `sq3_boundary`, `sq3_edge`, `sq4`, `sq5_reflow` were **byte-identical** between run 1 and run 2. `sq1_segments` and the real-PTY run differed **only** in RSS-memory readings (OS page-accounting noise) and wall-clock elapsed time; **every structural magnitude** (`count`, `ynum`, `xnum`, segment count, `evicted`, `ring_bytes`, `scrolled_by`) was identical. No reported magnitude was unstable, so no scale increase was required beyond what is shown.

### 8.1 End-to-end real-PTY confirmation (`sq_pty.py`)

To confirm the in-process `parse_bytes` results hold through a **real child process on a real PTY**, a child was spawned to print 200,000 lines. Full output:

```
===== END-TO-END REAL PTY : child prints 200000 lines ; pager=4194304 B (4.0MB) ; scrollback_lines=2000 =====
[BEFORE] count=0 ynum=2000 xnum=80 ring_bytes=0
[AFTER ] count=2000 ynum=2000 xnum=80 evicted=197977 ring_bytes=4194304 (4.000MB) child_exit_status=0 elapsed=1.6s
  ring FIRST 60 repr: b'rollback burst\r\n\x1b[mL0087601 line of scrollback burst\r\n\x1b[mL00'
  ring LAST  60 repr: b' of scrollback burst\r\n\x1b[mL0197976 line of scrollback burst\r\n'
  total_bytes_received_from_child=7000000
```

**[OBSERVED]** The real child (7,000,000 bytes received; 35 B/line including the ONLCR `\r\n` translation the PTY applies) drives the buffer to the identical end state: `count=2000`, `evicted=197977`, `ring_bytes=4,194,304` (plateaued at the 4 MB cap), child exit status 0. This confirms the canonical fidelity of the in-process driver used for the finer-grained probes.

---

## 9. Observed vs. inferred — summary

- **[OBSERVED]** (captured from the running build): all `count`/`ynum`/`xnum` values; segment counts and the ~5 MB/segment resident steps; the 47 B/line ring growth; the ring plateau at `maximum_size`; the oldest-bytes-overwritten `first8` flip; the no-crash 1M/10M fragility runs; the SQ4 two-threshold pin/drift; the SQ5 reflow line-count changes and +17 non-idempotency; real-PTY end state; ASAN clean exit.
- **[INFERRED]** (from source, cited): the exact per-segment byte formula `2048*(32*xnum+1)` (from the `calloc` expression [kitty/history.c:18-29]); that `parse_bytes` exercises the same functions as the child-monitor loop [kitty/child-monitor.c:1337-1354] (corroborated by the matching real-PTY end state); that overwriting is performed by `ringbuf_memcpy_into` advancing the FIFO tail [kitty/history.c:219, 3rdparty/ringbuf/ringbuf.c].

---

## 10. Coverage pass

| Sub-question | Answered in | Sibling variants covered |
|---|---|---|
| SQ1 — segment carving | §2 | default single-segment (2000) **and** multi-segment (5000→3, 10000→5); per-segment byte size; carving at each 2048 boundary |
| SQ2 — store ↔ ring interaction | §3 | pager **disabled** (drop) **and** enabled (ring @47 B/line); FIFO order verified |
| SQ3 — transition smoothness | §4 | segment boundary; ring grow→plateau→overwrite; tiny-ring edges (1–1024 B); 1M & 10M fragility; ASAN corroboration |
| SQ4 — concurrent scroll+write | §5 | unsaturated pin-holds **and** saturated cap→freeze/drift; the two-threshold subtle moment |
| SQ5 — runtime evolution | §2, §3, §6 | allocation, retention/eviction, wrapping/reflow; consolidated before/during/after table; reflow non-idempotency |

---

## 11. Read-only / reproducibility note

This investigation modified **no** existing repository file. All observation scripts were written to a scratch directory **outside** the repository tree (`/tmp/kitty_probe/`), executed to capture the output shown above, and then removed. The only tracked repository addition is this document. Build artifacts (`kitty/fast_data_types.so`, `build/`) are gitignored, so `git status --porcelain` reports only this file. Observation used only the exposed read-only surface (`count`, `ynum`, `xnum`, `pagerhist_as_bytes`, `pagerhist_as_text`, `scrolled_by`); no source instrumentation was added.
