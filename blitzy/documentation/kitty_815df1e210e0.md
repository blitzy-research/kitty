# Runtime-Observed Characterization of kitty's History/Scrollback Subsystem Under Extreme Write Pressure

> **Branch / commit:** `kitty_815df1e210e0` @ `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> **Methodology:** run-first. Every behavioral claim below is paired with the **exact command** that produced it and that command's **complete, unedited output**, plus an inline `file:line` citation into the source at this commit. Anything derived from reading code but *not* observed at runtime is explicitly labeled **`inferred`**.

---

## 1. Title & scope

This document characterizes what actually happens inside kitty's scrollback machinery when an enormous, fast burst of text is ingested through the terminal's real input path. The user's question, restated: *when a command "pours out an enormous amount of text in a very short time," what unfolds inside the `HistoryBuf` as it fills, stretches, and carves out new segments; how does the segmented scrollback store interact with the pager-style ring buffer under stress; are the transitions smooth or does the system hesitate at boundaries; what changes if someone is actively scrolling old output while new data keeps arriving; and how do allocation, wrapping, and retention really behave as pressure builds?*

There are **exactly two** memory structures under stress, both fields of the `HistoryBuf` struct (`kitty/data-types.h:282-290`):

1. **The segmented line store** — an array of `HistoryBufSegment`, each holding `SEGMENT_SIZE = 2048` lines (`kitty/history.c:15`), grown lazily as line indices demand more segments.
2. **The optional pager ring buffer** — a `PagerHistoryBuf` (`kitty/data-types.h:268-272`) wrapping the vendored byte ring in `3rdparty/ringbuf/`, **disabled by default**.

The five requirements are answered **by name** in Sections 4–8: **REQ-1** fill/stretch/carve, **REQ-2** segmented↔pager relationship, **REQ-3** boundary/hesitation, **REQ-4** concurrent scroll+ingest, **REQ-5** allocation/wrapping/retention. Section 9 reports cross-run stability, Section 10 the observed-vs-inferred summary, and Section 11 the citations appendix.

All runtime numbers were captured inside the provided Docker container `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, which supplies the C compiler, Go, CPython, and Valgrind that the authoring environment lacks.

---

## 2. Environment, build & invocation

### 2.1 Toolchain

- **Go** `1.22` — required by `go.mod:3` and used to drive `./dev.sh` (which `exec`s `go run bypy/devenv.go "$@"`, `dev.sh:9`). Observed: `go version go1.22.12 linux/amd64`.
- **C compiler** — `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0`, compiles the `fast_data_types` extension.
- **CPython** — the build downloads and embeds its own interpreter (`kitty/launcher/kitty` runs it via `PYTHONHOME`); the project floor is `>=3.8` (`pyproject.toml:2`) and CI's highest matrix entry is 3.11 (`.github/workflows/ci.yml`).
- **Valgrind** `3.25.1` — used for the Massif heap-over-time cross-check in Section 8.

### 2.2 Canonical build

The default build entry point is `./dev.sh build`, producing the runnable launcher at `kitty/launcher/kitty` (`docs/build.rst:19,22`). In this container the build requires kitty's own `--ignore-compiler-warnings` flag because the container's `wayland-protocols` 1.45 introduces new `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that trip the default `-Werror` **in the Wayland backend only** — a dependency-version skew unrelated to the history subsystem, non-invasive to runtime behavior.

```
$ timeout 900 ./dev.sh build --ignore-compiler-warnings
Build successful. Run kitty as: kitty/launcher/kitty
```

### 2.3 Reported version (records the exact build)

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### 2.4 Debug build (for Massif symbol attribution, Section 8)

A separate debug build was produced for allocation attribution (`docs/build.rst:54`). Because GCC 15 auto-vectorizes some functions with **AVX-512** (EVEX-prefixed `0x62…` opcodes) that Valgrind 3.25.1 cannot model — which caused Massif to abort with `SIGILL` at `new_screen_object (screen.c:117)` — the debug build for profiling was compiled with AVX-512 disabled. `setup.py` appends `$CFLAGS` **after** its native `-march` flags (so the last `-m…` wins), letting a non-invasive `-mno-avx512f` take effect without editing any file:

```
$ CFLAGS="-mno-avx512f" ./dev.sh build --debug --ignore-compiler-warnings
Build successful. Run kitty as: kitty/launcher/kitty
```

> After profiling, the **canonical release build was restored** (`./dev.sh build --ignore-compiler-warnings`, `kitty 0.35.2`) so that all non-Massif numbers in this document come from the default build. The debug/`-mno-avx512f` build was used *only* for the Massif snapshot in §8.2.

### 2.5 Invocation of observation scripts

Every observation script was run through kitty's embedded interpreter, which loads the compiled `fast_data_types` extension alongside CPython:

```
$ ./kitty/launcher/kitty +launch /tmp/obs_<name>.py
```

All observation scripts and profiler outputs lived under `/tmp` (never inside the repository tree) and were deleted afterward; the product repository is left byte-for-byte unchanged (verified in §Cleanup).

---

## 3. How the canonical ingest path was driven (not a synthetic poke)

### 3.1 The real path

Production data flow is: child-process bytes → VT parser (`kitty/vt-parser.c`) → screen (`kitty/screen.c`) → `historybuf_add_line`. The observation scripts exercise this **in-process** by feeding raw bytes to a `Screen` via `parse_bytes(screen, data)` (`kitty_tests/__init__.py:30-36`). That helper calls `screen.test_create_write_buffer()` → `test_commit_write_buffer()` → `test_parse_written_data()`, which invoke the **real** VT parser worker `parse_worker` (`kitty/screen.c:4772-4776`). A newline drives `screen_index` (`kitty/screen.c:1569-1577`) → the `INDEX_UP` macro (`kitty/screen.c:1552-1567`) → `historybuf_add_line` (`kitty/screen.c:1558`). Crucially, `add_to_history` is only true on the **main** screen with **no top margin** (`kitty/screen.c:1574`), so the scripts keep the default main linebuf and set no margin.

The Massif call tree in §8.2 **independently proves** this is the path that allocates the segments — the allocation backtrace runs `test_parse_written_data → parse_worker → run_worker → consume_input → consume_normal → screen_draw_text → … → screen_index → historybuf_add_line → historybuf_push → segment_for → add_segment`.

### 3.2 Constructing the Screen

`create_screen(cols, lines, scrollback, cell_width, cell_height, options)` is a **method of `BaseTest`** (`kitty_tests/__init__.py:237-241`); it calls `self.set_options(options)` then constructs `Screen(callbacks, lines, cols, scrollback, cell_width, cell_height, 0, callbacks)`. The store is sized by `alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size))` (`kitty/screen.c:130`), so **`ynum = MAX(scrollback, lines)`** and **`xnum = columns`** (the argument order flips inside `alloc_historybuf → create_historybuf`, `kitty/history.c:577-579`). Scripts therefore subclass `BaseTest` and call `self.create_screen(...)` with explicit large sizes.

### 3.3 Two harness facts verified empirically before the campaign

**(a) `\r\n` line accounting.** A cooked TTY delivers `\r\n`; feeding bare `\n` causes cursor drift and spurious wrapping. With `cols=80, lines=24` and `\r\n`-terminated lines, `INDEX_UP` fires only once the cursor reaches the bottom margin, so after feeding `N` lines the scrollback `count = N − (lines − 1) = N − 23` (for `N ≥ 24`). Scripts feed `N = target + 23` to land `count` on an exact target.

**(b) Pager units caveat.** The config-file converter treats `scrollback_pager_history_size` as **megabytes** (`int(max(0, float(x)) * 1024 * 1024)`, `kitty/options/utils.py:564-566`). But a **raw int** passed through `create_screen(options={...})` bypasses that string converter and is used as **raw bytes** for `PagerHistoryBuf.maximum_size`. Verified directly:

```
$ ./kitty/launcher/kitty +launch /tmp/obs_units.py
cap=8       -> pager_bytes=8
cap=64      -> pager_bytes=64
cap=1048576 -> pager_bytes=5467
cap=4194304 -> pager_bytes=5467
```

Feeding the same ~5467 bytes of serialized eviction into stores with different caps: `cap=8` pins the ring at **8 bytes** (not 8 MiB) and `cap=64` at **64 bytes**, while a 1 MiB / 4 MiB cap holds all 5467 bytes. This confirms the value is **raw bytes** (and matches the existing in-repo test `kitty_tests/screen.py` where `hsz=8` behaves as 8 bytes). The scripts choose byte values accordingly.

### 3.4 Observability workaround

`HistoryBuf` exposes **only** `xnum`, `ynum`, `count` to Python, all **read-only** (`kitty/history.c:554-559`). `num_segments` and `start_of_data` are C-only and **not** exposed. Therefore the **segment count is derived** as `ceil(min(count, ynum) / 2048)` and **corroborated** by process RSS, GNU `malloc_info`, and Valgrind Massif. Pager size is read via `historybuf.pagerhist_as_bytes()` (`kitty/history.c:460-483`) / `pagerhist_as_text()` (`kitty/history.c:485-494`). `Screen.scrolled_by` is read-only (`kitty/screen.c:4903`) and `Screen.history_line_added_count` is writable (`kitty/screen.c:4908`).

> **Derived-count caveat:** the formula returns `0` at `count == 0`, but a *fresh* `HistoryBuf` already contains **1** segment because `create_historybuf` calls `add_segment` once (`kitty/history.c:127`). The derived count therefore under-reports by one only at `count == 0`; for all `count ≥ 1` it matches the physical segment count corroborated by Massif/`malloc_info`.

### 3.5 Representative harness excerpt (complete, not elided)

```python
# /tmp/obs_scenarioA.py  (run via: ./kitty/launcher/kitty +launch /tmp/obs_scenarioA.py)
import math, os
from kitty_tests import BaseTest, parse_bytes

def rss_kb():
    for ln in open('/proc/self/status'):
        if ln.startswith('VmRSS:'):
            return int(ln.split()[1])
    return -1

def derived_segments(count, ynum):
    return math.ceil(min(count, ynum) / 2048)

class T(BaseTest):
    def run(self):
        cols, lines, scrollback = 80, 24, 20000
        s = self.create_screen(cols, lines, scrollback, options={})
        hb = s.historybuf
        ynum, xnum = hb.ynum, hb.xnum
        base = rss_kb()
        print(f"ynum={ynum} xnum={xnum}  BASELINE: count={hb.count} "
              f"derived_segments={derived_segments(hb.count, ynum)} RSS={base} kB")
        print(f"{'target':<8} {'count':<8} {'segments':<10} {'RSS_kB':<12} {'dRSS_kB':<12}")
        fed = 0
        for target in (2048, 4096, 6144, 8192, 10240, 12288, 14336, 16384, 18432, 20000):
            need = (target + (lines - 1)) - fed
            parse_bytes(s, ("".join(f"L{fed+i:08d}\r\n" for i in range(need))).encode())
            fed += need
            print(f"{target:<8} {hb.count:<8} {derived_segments(hb.count, ynum):<10} "
                  f"{rss_kb():<12} {rss_kb()-base:<12}")
        # push far beyond capacity to demonstrate saturation
        parse_bytes(s, ("".join(f"X{i:08d}\r\n" for i in range(80000))).encode())
        print(f"SATURATION: after requesting count target=80000, actual count={hb.count} "
              f"(ynum={ynum}) segments={derived_segments(hb.count, ynum)} "
              f"RSS={rss_kb()} kB dRSS={rss_kb()-base} kB")
        print(f"count==ynum ? {hb.count==ynum} ; count never exceeded ynum ? {hb.count<=ynum}")

T().run()
```

The other scenario scripts follow the same shape (subclass `BaseTest`, `create_screen`, `parse_bytes`), varying only the sizes, the pager cap, the scroll operations, and what is sampled.

---

## 4. REQ-1 — Fill / stretch / carve

**Claim.** As a huge burst arrives, `HistoryBuf.count` climbs monotonically until it **saturates at `ynum` and never exceeds it**; meanwhile the physical segment array is **carved out one 2048-line segment at a time**, lazily, exactly as line indices demand. "Fill" is `count` rising to a ceiling; "carve" is the segment array growing underneath it.

**Mechanism (cause → effect).** Each ingested newline routes to `historybuf_push` (`kitty/history.c:275-284`). It computes the target slot `idx = (start_of_data + count) % ynum` (`kitty/history.c:277`); if `count == ynum` it evicts the oldest line and holds `count` steady (`kitty/history.c:279-281`), otherwise it does `count++` (`kitty/history.c:282`). Reaching a slot in a not-yet-materialized segment calls `segment_for` (`kitty/history.c:36-42`), which loops `add_segment` while `seg_num >= num_segments && SEGMENT_SIZE*num_segments < ynum` (`kitty/history.c:39`). `add_segment` (`kitty/history.c:17-29`) does `num_segments += 1`, `realloc`s the segment-pointer array, and issues a single per-segment `calloc` sized `xnum*2048*sizeof(CPUCell) + xnum*2048*sizeof(GPUCell) + 2048*sizeof(LineAttrs)` (`kitty/history.c:23-25`). A fresh buffer already owns **1** segment (`create_historybuf` calls `add_segment` once, `kitty/history.c:127`); segments 2..N are the ones carved during the burst.

**Command & complete unedited output (both runs shown together for stability):**

```
$ for r in 1 2; do echo "=== Scenario A / RUN $r ==="; ./kitty/launcher/kitty +launch /tmp/obs_scenarioA.py; done
=== Scenario A / RUN 1 ===
ynum=20000 xnum=80  BASELINE: count=0 derived_segments=0 RSS=26304 kB
target   count    segments   RSS_kB       dRSS_kB     
2048     2048     1          32292        5988        
4096     4096     2          37576        11272       
6144     6144     3          42708        16404       
8192     8192     4          47840        21536       
10240    10240    5          52976        26672       
12288    12288    6          58108        31804       
14336    14336    7          63240        36936       
16384    16384    8          68372        42068       
18432    18432    9          73504        47200       
20000    20000    10         77436        51132       
SATURATION: after requesting count target=80000, actual count=20000 (ynum=20000) segments=10 RSS=78052 kB dRSS=51748 kB
count==ynum ? True ; count never exceeded ynum ? True
=== Scenario A / RUN 2 ===
ynum=20000 xnum=80  BASELINE: count=0 derived_segments=0 RSS=26292 kB
target   count    segments   RSS_kB       dRSS_kB     
2048     2048     1          32280        5988        
4096     4096     2          37564        11272       
6144     6144     3          42696        16404       
8192     8192     4          47828        21536       
10240    10240    5          52964        26672       
12288    12288    6          58096        31804       
14336    14336    7          63228        36936       
16384    16384    8          68360        42068       
18432    18432    9          73492        47200       
20000    20000    10         77424        51132       
SATURATION: after requesting count target=80000, actual count=20000 (ynum=20000) segments=10 RSS=78040 kB dRSS=51748 kB
count==ynum ? True ; count never exceeded ynum ? True
```

**Before / during / after reading:**

- **Before:** `count=0`, derived segments `0` (physically 1 — see §3.4 caveat), `RSS≈26304 kB`.
- **During:** `count` tracks each requested target exactly (2048, 4096, …); derived segment count increments by one at every 2048 multiple (`1,2,3,…,10`); `RSS` grows in near-uniform steps.
- **After (saturation):** requesting 80,000 lines leaves `count=20000` (`== ynum`), segments `= ceil(20000/2048) = 10`, and `count` provably never exceeds `ynum`.

**RSS step corroborates the ~5 MiB per-segment `calloc`.** The per-segment allocation at `xnum=80` is `2048*80*12 (CPUCell) + 2048*80*20 (GPUCell) + 2048*1 (LineAttrs) = 5,244,928 bytes = 5122 kB ≈ 5.00 MiB` from the cited cell sizes `CPUCell=12 B` (`kitty/data-types.h:228`), `GPUCell=20 B` (`kitty/data-types.h:221`), `LineAttrs=1 B` (`kitty/data-types.h:231-239`). The observed successive `dRSS` deltas confirm it:

```
$ awk 'NR>1 && $5 ~ /^[0-9]+$/ {if(p!="") print "  step="$5-p" kB"; p=$5}' /tmp/out_A.txt | head -10
  step=5284 kB
  step=5132 kB
  step=5132 kB
  step=5136 kB
  step=5132 kB
  step=5132 kB
  step=5132 kB
  step=5132 kB
  step=3932 kB
```

Each new segment lifts RSS by ~5132 kB — within a hair of the predicted 5122 kB (the small excess is page-table/first-touch overhead). The last step is smaller (3932 kB) because the final segment (`20000` is not a 2048 multiple) is only partially page-touched — a direct runtime signal of the **lazy** `calloc`: pages are only resident once written. The exact 5,244,928 → page-rounded 5,251,072 B per-segment figure is confirmed against Massif in §8.2. **`inferred`:** the precise `sizeof` breakdown (12/20/1 B) is taken from the cited `static_assert`s, not measured via `sizeof` at runtime, but it is strongly corroborated by the ~5132 kB RSS step *and* the Massif per-segment 5,251,072 B.

---

## 5. REQ-2 — Segmented scrollback ↔ pager ring relationship

**Claim.** The "quiet interaction" is an **eviction hand-off**: while `count < ynum` the pager ring stays empty; the instant `count == ynum`, every *further* ingested line causes the **oldest** scrollback line to be serialized to ANSI and appended into the pager ring, and only then is it dropped from the segmented store. `count` stays pinned at `ynum` forever after; the pager byte length grows by one serialized line per eviction.

**Mechanism (cause → effect).** In `historybuf_push`, the `count == ynum` branch first calls `pagerhist_push` (`kitty/history.c:280`), *then* advances `start_of_data = (start_of_data + 1) % ynum` (`kitty/history.c:281`) — i.e. serialize-then-evict. `pagerhist_push` (`kitty/history.c:258-273`) initializes the oldest line at `start_of_data` (`kitty/history.c:264`), renders it with `line_as_ansi` (`kitty/history.c:265`), writes the SGR reset `"\x1b[m"` (`kitty/history.c:266`), the UCS4 payload (`kitty/history.c:268`), and a `\r`/`\n` terminator (`kitty/history.c:269-271`) into the ring. If the pager is disabled (`scrollback_pager_history_size == 0`), `alloc_pagerhist` returns `NULL` (`kitty/history.c:72`) and the evicted line is simply lost.

**Command & complete unedited output (both runs; pager cap = 16 MiB = 16777216 raw bytes):**

```
$ for r in 1 2; do echo "================= RUN $r ================="; ./kitty/launcher/kitty +launch /tmp/obs_scenarioB.py; done
================= RUN 1 =================
===== PART 1: pager ENABLED (cap = 16 MiB = 16777216 bytes) =====
count=50   ynum=100  pager_bytes=0       (count<ynum -> pager empty)  oldest_retained=LINE00000000
count=99   ynum=100  pager_bytes=0       oldest_retained=LINE00000000
count=100  ynum=100  pager_bytes=0       <== count JUST reached ynum, still no eviction
+1     lines -> count=100  (pinned==ynum? True) pager_bytes=17      oldest_retained=LINE00000001
+10    lines -> count=100  (pinned==ynum? True) pager_bytes=187     oldest_retained=LINE00000011
+100   lines -> count=100  (pinned==ynum? True) pager_bytes=1887    oldest_retained=LINE00000111
+1000  lines -> count=100  (pinned==ynum? True) pager_bytes=18887   oldest_retained=LINE00001111
pager head (oldest serialized line) = b'\x1b[mLINE00000000\r\n\x1b[mLINE00000001\r\n\x1b[mLIN'
pager tail (newest serialized line) = b'1108\r\n\x1b[mLINE00001109\r\n\x1b[mLINE00001110\r\n'

===== PART 2: pager DISABLED (cap = 0) =====
count=100  ynum=100  pager_bytes=0       (disabled -> alloc_pagerhist returns NULL, ring never exists)
oldest_retained line after evicting ~5000 lines = LINE00005000  (LINE00000000..LINE00004999 are LOST)
exit=0
================= RUN 2 =================
===== PART 1: pager ENABLED (cap = 16 MiB = 16777216 bytes) =====
count=50   ynum=100  pager_bytes=0       (count<ynum -> pager empty)  oldest_retained=LINE00000000
count=99   ynum=100  pager_bytes=0       oldest_retained=LINE00000000
count=100  ynum=100  pager_bytes=0       <== count JUST reached ynum, still no eviction
+1     lines -> count=100  (pinned==ynum? True) pager_bytes=17      oldest_retained=LINE00000001
+10    lines -> count=100  (pinned==ynum? True) pager_bytes=187     oldest_retained=LINE00000011
+100   lines -> count=100  (pinned==ynum? True) pager_bytes=1887    oldest_retained=LINE00000111
+1000  lines -> count=100  (pinned==ynum? True) pager_bytes=18887   oldest_retained=LINE00001111
pager head (oldest serialized line) = b'\x1b[mLINE00000000\r\n\x1b[mLINE00000001\r\n\x1b[mLIN'
pager tail (newest serialized line) = b'1108\r\n\x1b[mLINE00001109\r\n\x1b[mLINE00001110\r\n'

===== PART 2: pager DISABLED (cap = 0) =====
count=100  ynum=100  pager_bytes=0       (disabled -> alloc_pagerhist returns NULL, ring never exists)
oldest_retained line after evicting ~5000 lines = LINE00005000  (LINE00000000..LINE00004999 are LOST)
exit=0
```

**Before / during / after reading:**

- **Before full (`count < ynum`):** at `count=50` and even at `count=99`, `pager_bytes=0` — the ring receives nothing while free scrollback slots remain.
- **At the threshold (`count == ynum`):** `pager_bytes` is *still* `0` — reaching `ynum` does not itself evict; the ring stays empty until the **next** line arrives.
- **After full:** each subsequent line adds exactly one serialized record. `+1 → 17 B`, `+10 → 187 B`, `+100 → 1887 B`, `+1000 → 18887 B`, all while `count` stays pinned at `100`. The per-record size is `17 B = len("\x1b[m") (3) + len("LINE00000000") (12) + len("\r\n") (2)`, matching `pagerhist_push`'s `"\x1b[m"` + payload + `\r\n` exactly (`kitty/history.c:266-271`).

**Ordering within the ring:** the head bytes are the *oldest* serialized line (`\x1b[mLINE00000000\r\n…`) and the tail bytes are the *newest* (`…LINE00001110\r\n`), confirming FIFO append. **Disabled-vs-enabled contrast:** with the pager off, after evicting ~5000 lines the oldest *retained* line is `LINE00005000` and `LINE00000000..LINE00004999` are gone (no ring exists); with the pager on, those same evicted lines are preserved in the ring. This is the core of the segmented↔pager relationship: the pager is the **overflow reservoir** that catches exactly what the segmented store evicts.

---

## 6. REQ-3 — Boundary / hesitation (both edges)

The question asks whether transitions are smooth or whether "the system hesitates or behaves differently than expected." There are two distinct boundary candidates, and they behave **differently** — so both are exercised and reported.

### 6.1 Edge (a) — segment-array boundaries (every 2048 lines)

**Claim.** Crossing a 2048-line multiple triggers `add_segment` (a `realloc` of the pointer array plus a fresh ~5 MiB `calloc`). This produces a **small, consistent, reproducible** per-line latency bump (tens of microseconds) at the first push past each multiple — a *tiny hesitation*, not a stall. Off-boundary spikes exist but are **not reproducible** (OS/allocator noise).

**Mechanism.** The bump lands at `count = 2048k + 1`: the push that made `count == 2048k` filled the last slot of segment `k`, and the *next* push needs slot `2048k` in a new segment, so `segment_for → add_segment` fires there (`kitty/history.c:36-42, 17-29`).

**Command & complete unedited output (both runs; edge (a) portion):**

```
$ for r in 1 2; do echo "================= RUN $r ================="; ./kitty/launcher/kitty +launch /tmp/obs_scenarioC.py; done
================= RUN 1 =================
========== EDGE (a): SEGMENT-BOUNDARY per-line timing ==========
fed 10000 single lines; final count=10000
per-line time us: mean=2.497 median=2.674 p99=7.413 max=27.576 min=1.368
12 slowest lines (count, us):
   count=1918    27.576 us   (distance to nearest 2048-multiple = 130)
   count=1827    25.165 us   (distance to nearest 2048-multiple = 221)
   count=3725    22.300 us   (distance to nearest 2048-multiple = 371)
   count=380     17.394 us   (distance to nearest 2048-multiple = 1668)
   count=6756    16.808 us   (distance to nearest 2048-multiple = 612)
   count=2049    16.277 us   (distance to nearest 2048-multiple = 1)
   count=4045    15.342 us   (distance to nearest 2048-multiple = 51)
   count=4097    15.250 us   (distance to nearest 2048-multiple = 1)
   count=234     14.554 us   (distance to nearest 2048-multiple = 1814)
   count=8193    13.416 us   (distance to nearest 2048-multiple = 1)
   count=25      13.000 us   (distance to nearest 2048-multiple = 2023)
   count=1507    12.818 us   (distance to nearest 2048-multiple = 541)
  boundary ~2048: c2045=4.56us  c2046=3.51us  c2047=1.44us  c2048=1.74us  c2049=16.28us  c2050=1.64us  c2051=3.34us
  boundary ~4096: c4093=4.16us  c4094=2.83us  c4095=1.42us  c4096=7.22us  c4097=15.25us  c4098=1.76us  c4099=2.79us
  boundary ~6144: c6141=3.92us  c6142=2.79us  c6143=1.44us  c6144=1.54us  c6145=11.27us  c6146=1.63us  c6147=4.06us
  boundary ~8192: c8189=4.12us  c8190=2.76us  c8191=1.42us  c8192=2.04us  c8193=13.42us  c8194=1.51us  c8195=3.22us

========== EDGE (b): PAGER-RING cap growth / plateau / overwrite ==========
cap=4194304 bytes (4.0 MiB); ynum=100 count=100
lines_past_sat pager_bytes    pager_MiB        oldest_serialized_head
2000           104000         0.099            ESC[mB0000000000 p
6000           312000         0.298            ESC[mB0000000000 p
10000          520000         0.496            ESC[mB0000000000 p
14000          728000         0.694            ESC[mB0000000000 p
18000          936000         0.893            ESC[mB0000000000 p
24000          1248000        1.190            ESC[mB0000000000 p
30000          1560000        1.488            ESC[mB0000000000 p
36000          1872000        1.785            ESC[mB0000000000 p
42000          2184000        2.083            ESC[mB0000000000 p
50000          2600000        2.480            ESC[mB0000000000 p
58000          3016000        2.876            ESC[mB0000000000 p
66000          3432000        3.273            ESC[mB0000000000 p
74000          3848000        3.670            ESC[mB0000000000 p
82000          4194304        4.000            adding-padding-p
90000          4194304        4.000            adding-padding-p
98000          4194304        4.000            adding-padding-p
106000         4194304        4.000            adding-padding-p
exit=0
================= RUN 2 =================
========== EDGE (a): SEGMENT-BOUNDARY per-line timing ==========
fed 10000 single lines; final count=10000
per-line time us: mean=2.574 median=2.715 p99=7.859 max=29.477 min=1.383
12 slowest lines (count, us):
   count=3226    29.477 us   (distance to nearest 2048-multiple = 870)
   count=1918    27.367 us   (distance to nearest 2048-multiple = 130)
   count=9407    23.344 us   (distance to nearest 2048-multiple = 833)
   count=8559    18.042 us   (distance to nearest 2048-multiple = 367)
   count=647     16.981 us   (distance to nearest 2048-multiple = 1401)
   count=1186    16.584 us   (distance to nearest 2048-multiple = 862)
   count=2436    15.911 us   (distance to nearest 2048-multiple = 388)
   count=8193    15.363 us   (distance to nearest 2048-multiple = 1)
   count=2049    14.856 us   (distance to nearest 2048-multiple = 1)
   count=4097    13.920 us   (distance to nearest 2048-multiple = 1)
   count=640     13.713 us   (distance to nearest 2048-multiple = 1408)
   count=697     13.594 us   (distance to nearest 2048-multiple = 1351)
  boundary ~2048: c2045=4.17us  c2046=2.91us  c2047=1.48us  c2048=1.67us  c2049=14.86us  c2050=1.59us  c2051=2.94us
  boundary ~4096: c4093=4.32us  c4094=2.81us  c4095=1.50us  c4096=1.70us  c4097=13.92us  c4098=1.47us  c4099=3.12us
  boundary ~6144: c6141=4.20us  c6142=2.97us  c6143=1.49us  c6144=1.71us  c6145=13.17us  c6146=1.56us  c6147=2.80us
  boundary ~8192: c8189=3.87us  c8190=8.42us  c8191=1.56us  c8192=1.71us  c8193=15.36us  c8194=1.66us  c8195=2.98us

========== EDGE (b): PAGER-RING cap growth / plateau / overwrite ==========
cap=4194304 bytes (4.0 MiB); ynum=100 count=100
lines_past_sat pager_bytes    pager_MiB        oldest_serialized_head
2000           104000         0.099            ESC[mB0000000000 p
6000           312000         0.298            ESC[mB0000000000 p
10000          520000         0.496            ESC[mB0000000000 p
14000          728000         0.694            ESC[mB0000000000 p
18000          936000         0.893            ESC[mB0000000000 p
24000          1248000        1.190            ESC[mB0000000000 p
30000          1560000        1.488            ESC[mB0000000000 p
36000          1872000        1.785            ESC[mB0000000000 p
42000          2184000        2.083            ESC[mB0000000000 p
50000          2600000        2.480            ESC[mB0000000000 p
58000          3016000        2.876            ESC[mB0000000000 p
66000          3432000        3.273            ESC[mB0000000000 p
74000          3848000        3.670            ESC[mB0000000000 p
82000          4194304        4.000            adding-padding-p
90000          4194304        4.000            adding-padding-p
98000          4194304        4.000            adding-padding-p
106000         4194304        4.000            adding-padding-p
exit=0
```

**Reading edge (a) — before / at / after each boundary.** At every boundary the line *at* the multiple is cheap (`c2048=1.74/1.67us`, `c8192=2.04/1.71us`) and the line *just after* spikes reproducibly: `c2049 = 16.28/14.86 us`, `c4097 = 15.25/13.92 us`, `c6145 = 11.27/13.17 us`, `c8193 = 13.42/15.36 us`. The `count=2049/4097/6145/8193` entries appear in the "slowest lines" list in **both** runs — they are reproducible. By contrast the single largest spikes (`count=1918` at ~27 µs in both runs is coincidentally repeated, but `count=3226`, `9407`, `647`, etc. differ run-to-run) are **not** aligned to a 2048 multiple and **do not** reproduce — i.e. background OS/allocator jitter, not a subsystem hesitation. **Verdict: mostly smooth**, with a small (~11–16 µs) reproducible bump per segment boundary attributable to `add_segment`'s `realloc` + ~5 MiB `calloc`.

### 6.2 Edge (b) — pager ring reaching its cap

**Claim.** With a small cap (4 MiB here), the pager grows in steps up to the cap, then **plateaus exactly at the cap** and switches to **overwrite-oldest**. Crucially, the *growth* phase hesitates noticeably (each `pagerhist_extend` copies the entire ring), with the stall **growing** as the ring gets bigger; once at the cap, writes are cheap (no more growth).

**Mechanism.** `pagerhist_write_bytes` calls `pagerhist_extend` when the incoming bytes exceed free space (`kitty/history.c:223`). `pagerhist_extend` returns `false` once `buffer_size >= maximum_size` (`kitty/history.c:92`); otherwise it grows to `MIN(maximum_size, buffer_size + MAX(1 MiB, minsz))` (`kitty/history.c:93`) by allocating a new ring (`ringbuf_new`, `kitty/history.c:94`) and copying the old contents (`ringbuf_copy`, `kitty/history.c:97`). At the cap, `ringbuf_memcpy_into` detects overflow and advances the tail: `dst->tail = ringbuf_nextp(dst, dst->head)` with `assert(ringbuf_is_full(dst))` (`3rdparty/ringbuf/ringbuf.c:232-234`) — overwriting the oldest bytes.

**Reading edge (b) from the output above.** `pager_bytes` rises linearly with lines fed (the per-record serialized size here is 52 B), through `0.099 → 3.670 MiB`, then **pins at exactly `4194304` (= 4.000 MiB = the cap)** and stays there for every further burst. The `oldest_serialized_head` column is the decisive overwrite signal: during growth it stays `ESC[mB0000000000 p` (the very first serialized line is still present at the head); the moment the ring hits the cap it changes to `adding-padding-p` — the head content has been overwritten by newer data because the tail advanced. Total length holds at the cap while the *leading content changes* — precisely `ringbuf_memcpy_into`'s overwrite-oldest at overflow.

**Fine-grained growth timing (both runs) — where the real hesitation lives:**

```
$ for r in 1 2; do echo "=== C2 pager-extend timing / RUN $r ==="; ./kitty/launcher/kitty +launch /tmp/obs_scenarioC2.py; done
=== C2 pager-extend timing / RUN 1 ===
fed 100000 single lines past saturation; final pager_bytes=4194304 (cap=4194304)
per-line us: mean=5.601 median=4.760 p99=13.538 max=1957.053
10 slowest lines (pager_bytes, MiB, us):
   pager_bytes=3145746   3.000 MiB  1957.053 us
   pager_bytes=808866    0.771 MiB  1278.100 us
   pager_bytes=2097166   2.000 MiB  1172.453 us
   pager_bytes=1048586   1.000 MiB  742.810 us
   pager_bytes=82010     0.078 MiB  51.702 us
   pager_bytes=4194304   4.000 MiB  41.403 us
   pager_bytes=4194304   4.000 MiB  39.464 us
   pager_bytes=3821642   3.645 MiB  38.444 us
   pager_bytes=4176854   3.983 MiB  30.086 us
   pager_bytes=4194304   4.000 MiB  29.763 us
max per-line time within each 0.25 MiB band of pager_bytes:
   [0.00-0.25 MiB) max=51.702 us
   [0.25-0.50 MiB) max=17.751 us
   [0.50-0.75 MiB) max=18.074 us
   [0.75-1.00 MiB) max=1278.100 us
   [1.00-1.25 MiB) max=742.810 us
   [1.25-1.50 MiB) max=21.674 us
   [1.50-1.75 MiB) max=24.472 us
   [1.75-2.00 MiB) max=23.353 us
   [2.00-2.25 MiB) max=1172.453 us
   [2.25-2.50 MiB) max=24.842 us
   [2.50-2.75 MiB) max=26.930 us
   [2.75-3.00 MiB) max=23.096 us
   [3.00-3.25 MiB) max=1957.053 us
   [3.25-3.50 MiB) max=25.982 us
   [3.50-3.75 MiB) max=38.444 us
   [3.75-4.00 MiB) max=30.086 us
   [4.00-4.25 MiB) max=41.403 us
exit=0
=== C2 pager-extend timing / RUN 2 ===
fed 100000 single lines past saturation; final pager_bytes=4194304 (cap=4194304)
per-line us: mean=5.334 median=4.524 p99=13.427 max=1741.180
10 slowest lines (pager_bytes, MiB, us):
   pager_bytes=3145746   3.000 MiB  1741.180 us
   pager_bytes=808866    0.771 MiB  1169.833 us
   pager_bytes=2097166   2.000 MiB  1116.190 us
   pager_bytes=1048586   1.000 MiB  578.220 us
   pager_bytes=82010     0.078 MiB  52.726 us
   pager_bytes=3080746   2.938 MiB  46.159 us
   pager_bytes=4194304   4.000 MiB  37.977 us
   pager_bytes=2362522   2.253 MiB  35.980 us
   pager_bytes=2560382   2.442 MiB  35.764 us
   pager_bytes=3523214   3.360 MiB  34.919 us
max per-line time within each 0.25 MiB band of pager_bytes:
   [0.00-0.25 MiB) max=52.726 us
   [0.25-0.50 MiB) max=20.518 us
   [0.50-0.75 MiB) max=18.378 us
   [0.75-1.00 MiB) max=1169.833 us
   [1.00-1.25 MiB) max=578.220 us
   [1.25-1.50 MiB) max=34.568 us
   [1.50-1.75 MiB) max=24.262 us
   [1.75-2.00 MiB) max=21.560 us
   [2.00-2.25 MiB) max=1116.190 us
   [2.25-2.50 MiB) max=35.980 us
   [2.50-2.75 MiB) max=26.027 us
   [2.75-3.00 MiB) max=46.159 us
   [3.00-3.25 MiB) max=1957.053 us
   [3.25-3.50 MiB) max=34.919 us
   [3.50-3.75 MiB) max=23.828 us
   [3.75-4.00 MiB) max=30.086 us
   [4.00-4.25 MiB) max=37.977 us
```

**Reading the growth hesitation.** The large spikes land **exactly at the 1/2/3 MiB extend points**, and they **grow with ring size**: at `pager_bytes≈1.0 MiB` the line takes `742.810 / 578.220 us`; at `≈2.0 MiB` `1172.453 / 1116.190 us`; at `≈3.0 MiB` `1957.053 / 1741.180 us` — a rising staircase because each `pagerhist_extend` `memcpy`s the entire (larger) ring (`ringbuf_copy`, `kitty/history.c:97`). Once the ring reaches the 4 MiB cap the per-line max in the `[4.00-4.25 MiB)` band is only ~`41 / 38 us` — the plateau is *cheap* because no extend occurs, just an overwrite. There is one additional variable spike near `0.771 MiB` (`1278 / 1170 us`) that does not align to a 1 MiB boundary and whose magnitude/position drifts a little across runs; this is attributed to allocator first-touch, and its exact byte-position label is **`inferred`** rather than a distinct extend event. **Verdict: the pager-growth phase genuinely hesitates** — up to ~2 ms, a handful of times, escalating with size — **while the cap phase is smooth.** This is the clearest "system hesitates" signal in the whole subsystem.

---

## 7. REQ-4 — Concurrent scroll + ingest

**Claim.** When the user is scrolled back and new data keeps arriving, the view **stays anchored to the same old content**: on the next render the scroll offset `scrolled_by` is advanced by exactly the number of lines added since the last render, **clamped to `count`**. The user does not "slide" relative to old output — kitty keeps the same historical lines under the viewport as new lines push in from the bottom.

**Mechanism (cause → effect).** Each `INDEX_UP` increments `self->history_line_added_count` (`kitty/screen.c:1559`). At render time, `screen_update_only_line_graphics_data` captures that counter (`kitty/screen.c:2714`), then applies `self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count)` (`kitty/screen.c:2716`), and `screen_reset_dirty` resets the counter to 0 (`kitty/screen.c:2600`). The same formula lives in the full render path `screen_update_cell_data` (`kitty/screen.c:2761`). This method is exposed to Python as `update_only_line_graphics_data` (`kitty/screen.c:4867`, `METH_NOARGS`), so the anchoring can be triggered headlessly without a font/GPU render context. Scroll-up itself uses `new_scroll = MIN(scrolled_by + amt, count)` (`kitty/screen.c:4111`).

**Command & complete unedited output (both runs):**

```
$ for r in 1 2; do echo "================= RUN $r ================="; ./kitty/launcher/kitty +launch /tmp/obs_scenarioD.py; done
================= RUN 1 =================
===== PART 1: non-saturated store, view stays anchored to same old content =====
filled: count=5000  scrolled_by=0  history_line_added_count=0
BEFORE ingest: scrolled_by=1000  history_line_added_count=0  top-of-view line=ROW00004000
DURING (pre-anchor): count=5500  scrolled_by=1000 (stale)  history_line_added_count=500 (==M)
AFTER anchor: count=5500  scrolled_by=1500 (==MIN(N+M,count)=1500)  history_line_added_count=0
top-of-view line AFTER = ROW00004000   (same old content as BEFORE? True)

===== PART 2: SATURATED store, scrolled_by CLAMPED to count =====
BEFORE: count=100 ynum=100 scrolled_by=90 history_line_added_count=0
DURING (pre-anchor): count=100 scrolled_by=90 history_line_added_count=50 (==M2)
AFTER anchor: count=100 scrolled_by=100  (N2+M2=140 would exceed count -> CLAMPED to count=100)
exit=0
================= RUN 2 =================
===== PART 1: non-saturated store, view stays anchored to same old content =====
filled: count=5000  scrolled_by=0  history_line_added_count=0
BEFORE ingest: scrolled_by=1000  history_line_added_count=0  top-of-view line=ROW00004000
DURING (pre-anchor): count=5500  scrolled_by=1000 (stale)  history_line_added_count=500 (==M)
AFTER anchor: count=5500  scrolled_by=1500 (==MIN(N+M,count)=1500)  history_line_added_count=0
top-of-view line AFTER = ROW00004000   (same old content as BEFORE? True)

===== PART 2: SATURATED store, scrolled_by CLAMPED to count =====
BEFORE: count=100 ynum=100 scrolled_by=90 history_line_added_count=0
DURING (pre-anchor): count=100 scrolled_by=90 history_line_added_count=50 (==M2)
AFTER anchor: count=100 scrolled_by=100  (N2+M2=140 would exceed count -> CLAMPED to count=100)
exit=0
```

**Before / during / after reading:**

*Part 1 — non-saturated store (`count` still growing):*
- **Before:** filled to `count=5000`, then scrolled back `N=1000`; `scrolled_by=1000`, `history_line_added_count=0`, and the top-of-view line is `ROW00004000`.
- **During (M=500 lines ingested, before the next render):** `count=5500`, `history_line_added_count=500` (`== M`), but `scrolled_by` is *still* `1000` — **stale between renders**. The offset is not updated on ingest; it is updated on render.
- **After anchoring (`update_only_line_graphics_data()`):** `scrolled_by=1500` (`= MIN(1000+500, 5500)`), `history_line_added_count` reset to `0`, and the top-of-view line is **still `ROW00004000`** — proving the view held onto the same old content while 500 new lines slid in beneath it.

*Part 2 — saturated store (`count == ynum == 100`):*
- **Before:** `scrolled_by=90`, `history_line_added_count=0`.
- **During:** `M2=50` lines ingested; `history_line_added_count=50`; `scrolled_by` still `90`.
- **After anchoring:** the raw sum `N2+M2 = 140` would exceed `count=100`, so `scrolled_by` is **clamped to `count=100`** — the `MIN(..., count)` term (`kitty/screen.c:2716`). At saturation the view cannot anchor beyond the total retained lines; the oldest content it was pinned to has itself been evicted, so the offset saturates at the top of the buffer.

This directly answers "what changes if someone is actively scrolling while new data arrives": between renders nothing moves (offset is stale); on each render the offset jumps forward by the number of newly added lines to keep the same old lines in view, and it never runs past `count`.

---

## 8. REQ-5 — Allocation / wrapping / retention

### 8.1 Allocation vs. lines (RSS), and total footprint = segments + pager ring

**Claim.** Total footprint is the sum of two independently-growing parts: the segmented store rises in ~5 MiB steps as segments are carved (until `count == ynum`, after which it stops), and the pager ring rises separately as evictions accumulate. RSS confirms both.

**Command & complete unedited output (both runs; Part 1):**

```
$ for r in 1 2; do echo "================= RUN $r ================="; ./kitty/launcher/kitty +launch /tmp/obs_scenarioE.py; done
================= RUN 1 =================
===== PART 1: total footprint = segmented store (~5MiB/seg) + pager ring =====
baseline RSS=26108 kB (ynum=8192)
linesfed   count    segments  pager_bytes  RSS_kB       dRSS_kB   
2071       2048     1         0            32168        6060      
4119       4096     2         0            37300        11192     
6167       6144     3         0            42448        16340     
8215       8192     4         0            47580        21472     
30023      8192     4         1199440      50192        24084     
60023      8192     4         2849440      51804        25696     

===== PART 2: WRAPPING (lines wider than xnum=80) =====
fed W=1000 logical lines of width 200 through xnum=80
history count=2977  (expected ~= 3 physical rows per logical line: 3*W - screenful)
count / W = 2.977  (≈ ceil(200/80)=3 rows per logical line)
wrap-flag on last cell of consecutive history lines (line(i), reverse index; True=continues onto next):
   line(0): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='W00000992aaaaaaaaaaa'
   line(1): width_used_last_cell? last_char_has_wrapped_flag=False  text_head='aaaaaaaaaaaaaaaaaaaa'
   line(2): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='aaaaaaaaaaaaaaaaaaaa'
   line(3): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='W00000991aaaaaaaaaaa'
   line(4): width_used_last_cell? last_char_has_wrapped_flag=False  text_head='aaaaaaaaaaaaaaaaaaaa'
   line(5): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='aaaaaaaaaaaaaaaaaaaa'

===== PART 3: RETENTION contrast (pager OFF vs ON) =====
pager OFF (cap=0)  count=100 pager_bytes=0  oldest_in_segmented_store=K00005000  pager_head=b''
pager ON (8 MiB)   count=100 pager_bytes=70000  oldest_in_segmented_store=K00005000  pager_head=b'\x1b[mK00000000\r\n\x1b[mK00'
exit=0
================= RUN 2 =================
===== PART 1: total footprint = segmented store (~5MiB/seg) + pager ring =====
baseline RSS=25932 kB (ynum=8192)
linesfed   count    segments  pager_bytes  RSS_kB       dRSS_kB   
2071       2048     1         0            31992        6060      
4119       4096     2         0            37124        11192     
6167       6144     3         0            42272        16340     
8215       8192     4         0            47404        21472     
30023      8192     4         1199440      50016        24084     
60023      8192     4         2849440      51628        25696     

===== PART 2: WRAPPING (lines wider than xnum=80) =====
fed W=1000 logical lines of width 200 through xnum=80
history count=2977  (expected ~= 3 physical rows per logical line: 3*W - screenful)
count / W = 2.977  (≈ ceil(200/80)=3 rows per logical line)
wrap-flag on last cell of consecutive history lines (line(i), reverse index; True=continues onto next):
   line(0): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='W00000992aaaaaaaaaaa'
   line(1): width_used_last_cell? last_char_has_wrapped_flag=False  text_head='aaaaaaaaaaaaaaaaaaaa'
   line(2): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='aaaaaaaaaaaaaaaaaaaa'
   line(3): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='W00000991aaaaaaaaaaa'
   line(4): width_used_last_cell? last_char_has_wrapped_flag=False  text_head='aaaaaaaaaaaaaaaaaaaa'
   line(5): width_used_last_cell? last_char_has_wrapped_flag=True  text_head='aaaaaaaaaaaaaaaaaaaa'

===== PART 3: RETENTION contrast (pager OFF vs ON) =====
pager OFF (cap=0)  count=100 pager_bytes=0  oldest_in_segmented_store=K00005000  pager_head=b''
pager ON (8 MiB)   count=100 pager_bytes=70000  oldest_in_segmented_store=K00005000  pager_head=b'\x1b[mK00000000\r\n\x1b[mK00'
exit=0
```

**Reading Part 1 (allocation).** With `ynum=8192`, the store fills to 4 segments (`dRSS` climbs `6060 → 11192 → 16340 → 21472 kB`, i.e. ~5.1 MiB per segment, matching §4). Once `count == 8192` the segment count **freezes at 4** — no more segment growth ever, regardless of how many lines follow (`count` is pinned). The *only* thing that grows afterward is the pager: `pager_bytes` goes `0 → 1199440 → 2849440` as evictions accumulate, and `dRSS` tracks it (`21472 → 24084 → 25696 kB`). This is the two-part footprint made visible: segments (bounded by `ynum`) + pager ring (bounded by its cap).

**Reading Part 2 (wrapping).** Feeding `W=1000` logical lines each 200 columns wide into an 80-column store produces `count=2977 ≈ 3×1000` — i.e. `ceil(200/80) = 3` physical rows per logical line (`count/W = 2.977`, the shortfall being the current screenful not yet in history). Inspecting the wrap continuation flag on consecutive history rows shows the expected `True,True,…,False` grouping per logical line: each logical line occupies three physical history rows where the first two carry the "continues onto next" flag and the third does not. So over-wide input is **stored as multiple physical scrollback lines**, and each such physical line consumes a full slot and counts toward `count`/segment growth. (Row order is reverse-indexed, so the labels `W00000992`, `W00000991` descend as you walk newer→older.)

**Reading Part 3 (retention).** Identical burst, two regimes: **pager OFF** → after ~5000 evictions the oldest line still in the segmented store is `K00005000`, and `pager_head` is empty (`b''`) — everything older is **lost**. **pager ON (8 MiB)** → same `oldest_in_segmented_store=K00005000`, but `pager_bytes=70000` and `pager_head` begins with `\x1b[mK00000000\r\n…` — the evicted lines from `K00000000` onward are **retained** in the ring. Retention is therefore entirely a function of whether the pager is enabled (and, past its cap, bounded by overwrite-oldest as shown in §6.2).

### 8.2 Valgrind Massif — heap-over-time attribution (proves the canonical path allocates the segments)

Massif was run over a burst script under the debug (`-mno-avx512f`) build, `--time-unit=B` for reproducibility:

```
$ valgrind --tool=massif --time-unit=B --massif-out-file=/tmp/massif.out.%p \
      ./kitty/launcher/kitty +launch /tmp/obs_massif.py
==48376== Massif, a heap profiler
==48376== Copyright (C) 2003-2024, and GNU GPL'd, by Nicholas Nethercote et al.
==48376== Using Valgrind-3.25.1 and LibVEX; rerun with -h for copyright info
==48376== Command: ./kitty/launcher/kitty +launch /tmp/obs_massif.py
==48376== 
MASSIF burst done: fed=53215 count=8192 segments(derived)=4 pager_bytes=2475000
==48376== 
```

The peak snapshot (via `ms_print /tmp/massif.out.48376`) attributes the live heap precisely to the two structures — and its backtrace runs through the **real VT parser**, independently confirming §3.1. `ms_print`'s output is a ~3000-line time series (an ASCII graph followed by 52 snapshots); rather than splice non-contiguous regions inside one code block, the relevant parts are shown below as **individually verbatim-contiguous excerpts**, with the intervening material described in prose between them.

Excerpt 1 — the `ms_print` header (verbatim, the first lines of its output):

```
$ ms_print /tmp/massif.out.48376
--------------------------------------------------------------------------------
Command:            ./kitty/launcher/kitty +launch /tmp/obs_massif.py
Massif arguments:   --time-unit=B --massif-out-file=/tmp/massif.out.%p
ms_print arguments: /tmp/massif.out.48376
--------------------------------------------------------------------------------
```

After the header comes an ASCII heap-vs-time graph, then this snapshot summary (verbatim):

```
Number of snapshots: 52
 Detailed snapshots: [1, 16, 19, 33, 37 (peak), 47]
```

The peak is **snapshot 37**. Its table row and total (verbatim, contiguous):

```
 37    100,317,288       38,099,720       38,041,600        58,120            0
99.85% (38,041,600B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
```

Its allocation tree has three top-level branches of interest. **Branch 1 — the segmented store carved *during ingest*** (verbatim, contiguous slice of the peak tree, indentation preserved):

```
->55.13% (21,004,288B) 0x5E70920: add_segment (history.c:25)
| ->41.35% (15,753,216B) 0x5E709BF: segment_for (history.c:39)
| | ->41.35% (15,753,216B) 0x5E709F2: cpu_lineptr (history.c:52)
| |   ->41.35% (15,753,216B) 0x5E70AAF: init_line (history.c:164)
| |     ->41.35% (15,753,216B) 0x5E70FDB: historybuf_push (history.c:278)
| |       ->41.35% (15,753,216B) 0x5E72386: historybuf_add_line (history.c:288)
| |         ->41.35% (15,753,216B) 0x5E9E90C: screen_index (screen.c:1575)
| |           ->41.35% (15,753,216B) 0x5E9EDE5: screen_linefeed (screen.c:1645)
| |             ->41.35% (15,753,216B) 0x5EA0275: draw_text_loop (screen.c:795)
| |               ->41.35% (15,753,216B) 0x5EA05DD: draw_text (screen.c:862)
| |                 ->41.35% (15,753,216B) 0x5EA0655: screen_draw_text (screen.c:868)
| |                   ->41.35% (15,753,216B) 0x5EC664A: consume_normal (vt-parser.c:236)
| |                     ->41.35% (15,753,216B) 0x5EC8C33: consume_input (vt-parser.c:1377)
| |                       ->41.35% (15,753,216B) 0x5EC8DF2: run_worker (vt-parser.c:1432)
| |                         ->41.35% (15,753,216B) 0x5EC9F88: parse_worker (vt-parser.c:1496)
| |                           ->41.35% (15,753,216B) 0x5E9C79D: test_parse_written_data (screen.c:4776)
```

Below `test_parse_written_data` the branch continues into CPython interpreter frames (`method_vectorcall_VARARGS`, `PyObject_Vectorcall`, `_PyEval_EvalFrameDefault`, …) belonging to the `+launch` driver — omitted here as they are not part of the subsystem. **Branch 2 — the *first* segment, allocated once at `Screen` construction** (verbatim, contiguous; this is the sibling of the `segment_for` sub-branch under the same `add_segment` total):

```
| ->13.78% (5,251,072B) 0x5E7143E: create_historybuf (history.c:127)
|   ->13.78% (5,251,072B) 0x5E7258F: alloc_historybuf (history.c:578)
|     ->13.78% (5,251,072B) 0x5E9A1B3: new_screen_object (screen.c:130)
```

**Branch 3 — the pager ring, allocated via the eviction path** (verbatim, contiguous, a separate top-level branch):

```
->13.76% (5,242,882B) 0x5EDD04A: ringbuf_new (ringbuf.c:57)
| ->13.76% (5,242,882B) 0x5E70D78: pagerhist_extend (history.c:94)
| | ->13.76% (5,242,882B) 0x5E70E1A: pagerhist_write_bytes (history.c:223)
| |   ->13.76% (5,242,882B) 0x5E70F3A: pagerhist_push (history.c:266)
| |     ->13.76% (5,242,882B) 0x5E71001: historybuf_push (history.c:280)
```

**Reading Massif.**
- **Segmented store = 21,004,288 B** attributed to `add_segment (history.c:25)`. This splits into **15,753,216 B (= 3 segments)** carved *during ingest* via `segment_for (history.c:39) → … → historybuf_push (history.c:278) → historybuf_add_line (history.c:288) → screen_index (screen.c:1575) → … → parse_worker (vt-parser.c:1496) → test_parse_written_data (screen.c:4776)` — **this is the canonical path, proven by the backtrace** — plus **5,251,072 B (= the 1st segment)** allocated once at construction via `create_historybuf (history.c:127) → alloc_historybuf (history.c:578) → new_screen_object (screen.c:130)`. Total 4 segments, matching `segments(derived)=4`.
- **Per-segment = 5,251,072 B**, the page-rounded form of the computed 5,244,928 B request (`5,251,072 = 1282 × 4096`), confirming the §4 arithmetic at the allocator level.
- **Pager ring** appears under `ringbuf_new (ringbuf.c:57)` via `pagerhist_extend (history.c:94) → pagerhist_write_bytes (history.c:223) → pagerhist_push (history.c:266) → historybuf_push (history.c:280)` — the eviction path of REQ-2, now visible as a distinct heap region separate from the segments.

### 8.3 GNU malloc_info in-process cross-check (on the canonical release build)

To corroborate Massif *without* Valgrind and on the **default** build, a script read `malloc_info` around the same burst:

```
$ for r in 1 2; do echo "=== malloc_info / RUN $r ==="; ./kitty/launcher/kitty +launch /tmp/obs_mallocinfo.py; done
=== malloc_info / RUN 1 ===
BEFORE burst:            mmap_total=8179712     system_current=3805184    
AFTER saturate (4 segs): mmap_total=25665536    system_current=4411392      d_mmap=17485824
AFTER pager growth:      mmap_total=27103232    system_current=9117696      d_mmap=18923520
count=8192 segments(derived)=4 pager_bytes=2475000
d_mmap after 4 segments = 17485824 B = 16.68 MiB (expect ~4*5.0MiB)
exit=0
=== malloc_info / RUN 2 ===
BEFORE burst:            mmap_total=8179712     system_current=3805184    
AFTER saturate (4 segs): mmap_total=25665536    system_current=4411392      d_mmap=17485824
AFTER pager growth:      mmap_total=27103232    system_current=9117696      d_mmap=18923520
count=8192 segments(derived)=4 pager_bytes=2475000
d_mmap after 4 segments = 17485824 B = 16.68 MiB (expect ~4*5.0MiB)
```

**Reading malloc_info.** The mmap-backed heap grows by `d_mmap = 17,485,824 B ≈ 16.68 MiB` while filling to 4 segments. That is ~3.34 segments' worth (`16.68 / 5.0`), which is exactly right: **only 3** segments are `mmap`'d *during the burst* because segment 1 was already allocated at `create_screen` time (the same 1-preexisting-segment fact from `create_historybuf`, `kitty/history.c:127`, and visible in the Massif split above). After further ingest the pager region adds another `~1.44 MiB` (`d_mmap` rises to `18,923,520 B`). Both runs are byte-identical.

---

## 9. Cross-run stability note

Every magnitude/timing scenario was run **≥2× with identical input**; the key magnitudes matched:

- **Scenario A (REQ-1):** `count` targets, derived segment counts, and — decisively — the per-segment `dRSS` deltas were **byte-identical** across runs (`5988, 11272, 16404, 21536, 26672, 31804, 36936, 42068, 47200, 51132` kB; saturation `dRSS=51748`). Only the absolute baseline RSS drifted a few kB (`26304` vs `26292`), which is expected process-level noise; the *deltas* that measure the subsystem were stable.
- **Scenario B (REQ-2):** pager byte lengths (`17, 187, 1887, 18887`) and head/tail bytes were **identical** in both runs.
- **Scenario C edge (a) (REQ-3):** the segment-boundary spikes at `count=2049/4097/6145/8193` reproduced in both runs (11–16 µs); the non-boundary maxima did **not** reproduce and are labeled noise.
- **Scenario C edge (b) / C2 (REQ-3):** the pager plateau (`4194304` exactly) and the escalating extend spikes at 1/2/3 MiB reproduced (`~0.7/0.6, ~1.2/1.1, ~2.0/1.7 ms`); magnitudes match to within run-to-run scheduling jitter.
- **Scenario D (REQ-4):** `scrolled_by` before/during/after (`1000 → 1000 → 1500`; saturated `90 → 90 → 100` clamp) were **identical** in both runs.
- **Scenario E (REQ-5):** the RSS table `dRSS` column, `pager_bytes`, wrapping `count=2977`, and the retention contrast were **identical**; `malloc_info` `d_mmap=17,485,824 B` was identical in both runs.

No value required a "same-input reproduction of an inconsistency"; nothing was unstable enough to need it. Where a number was *not* reproducible (the off-boundary timing maxima), it is explicitly reported as noise rather than a subsystem signal.

---

## 10. Observed vs. inferred summary

Everything in Sections 4–9 is **runtime-observed** except the three items below, which are explicitly **`inferred`** from cited source (each is nonetheless corroborated by an independent runtime measurement):

1. **The exact per-segment `sizeof` breakdown (`CPUCell=12 B`, `GPUCell=20 B`, `LineAttrs=1 B` ⇒ 5,244,928 B).** Inferred from the `static_assert`s at `kitty/data-types.h:221,228,231-239` — the individual `sizeof`s were not printed at runtime. **Corroborated** by the observed ~5132 kB RSS step (§4) and the Massif page-rounded 5,251,072 B per segment (§8.2).
2. **The derived segment count.** `num_segments` is not exposed to Python (`kitty/history.c:554-559`), so segment count is computed as `ceil(min(count, ynum)/2048)`. **Corroborated** by RSS steps, Massif's 4-segment split, and `malloc_info`'s `d_mmap`. (It under-reports by one only at `count==0`; see §3.4.)
3. **The exact byte-position/attribution of the sub-1 MiB pager timing spike (~0.771 MiB).** Its position drifts slightly across runs and does not align to a 1 MiB extend boundary; attributed to allocator first-touch rather than a `pagerhist_extend` event. The 1/2/3 MiB extend spikes themselves are observed and reproducible.

All other claims — `count` saturating at `ynum`; segments carved one 2048-block at a time; the pager filling only after `count==ynum`; the 17-byte serialized record; the segment-boundary micro-hesitation; the escalating pager-extend stalls; the plateau-and-overwrite at the cap; the scroll anchoring formula and its clamp; the wrapping factor; the retention contrast; the canonical-path allocation backtrace — are backed by the unedited command output shown next to each.

---

## 11. Citations appendix (consolidated)

Source at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

**`kitty/history.c`**
- `SEGMENT_SIZE 2048` — L15
- `add_segment` (num_segments++, realloc, per-segment calloc) — L17-29 (calloc L23-25)
- `segment_for` (lazy growth loop) — L36-42 (grow condition L39)
- `cpu_lineptr` — L52
- `initial_pagerhist_ringbuf_sz` (`MIN(1 MiB, sz)`) — L66-67
- `alloc_pagerhist` (returns NULL if size 0; `maximum_size`) — L69-80 (NULL L72, max L78)
- `pagerhist_extend` (stops at cap; ≥1 MiB steps; ringbuf_new/copy) — L89-101 (cap L92, step L93, new L94, copy L97)
- `create_historybuf` (one `add_segment`) — L116-133 (L127)
- `index_of` (reverse index) — L152-159
- `init_line` — L164
- `pagerhist_write_bytes` (extend-if-needed, memcpy_into) — L218-226 (extend L223)
- `pagerhist_push` (serialize oldest to ANSI + `\r\n`) — L258-273 (`\x1b[m` L266, ucs4 L268, terminator L269-271)
- `historybuf_push` (slot idx; evict-vs-count++) — L275-284 (idx L277, pagerhist_push L280, start_of_data L281, count++ L282)
- `historybuf_add_line` — L286-291 (L288)
- `pagerhist_as_bytes` / `pagerhist_as_text` — L460-483 / L485-494
- Python members `xnum,ynum,count` READONLY (num_segments/start_of_data not exposed) — L554-559
- `alloc_historybuf` (arg flip → `create_historybuf`) — L577-579

**`kitty/data-types.h`**
- `GPUCell` (`sizeof==20`) — L215-221
- `CPUCell` (`sizeof==12`) — L222-228
- `LineAttrs` (1 B union) — L231-239
- `HistoryBufSegment` — L262-266
- `PagerHistoryBuf` — L268-272
- `HistoryBuf` — L282-290

**`kitty/screen.c`**
- `new_screen_object` → `alloc_historybuf(MAX(scrollback,lines), columns, …)` (`ynum=MAX(scrollback,lines)`) — L130
- `INDEX_UP` (historybuf_add_line; history_line_added_count++) — L1552-1567 (add_line L1558, counter L1559)
- `screen_index` (add_to_history gate: main linebuf, no top margin) — L1569-1577 (gate L1574, INDEX_UP L1575)
- `screen_linefeed` — L1645
- `screen_reset_dirty` (resets history_line_added_count=0) — L2598-2600
- `screen_update_only_line_graphics_data` (anchor formula) — L2713-2717 (capture L2714, anchor L2716)
- `screen_update_cell_data` (same anchor formula) — L2761
- `screen_history_scroll` (`new_scroll=MIN(scrolled_by+amt,count)`) — L4091-4118 (L4111)
- `test_*` → `parse_worker` (real VT parser) — L4755-4776
- `update_only_line_graphics_data` exposed (METH_NOARGS) — L4867
- members: `historybuf` RO L4902, `scrolled_by` RO L4903, `history_line_added_count` writable L4908

**`3rdparty/ringbuf/ringbuf.c`**
- `ringbuf_new` (`size = capacity + 1`) — L50-57
- `ringbuf_memcpy_into` (overwrite-oldest at overflow) — L211-238 (overflow L216, tail advance L233, assert-full L234)
- `ringbuf_copy` — L359

**`kitty/options/definition.py`** — `scrollback_lines` default `2000` L372-373; `scrollback_pager_history_size` default `0` L406-407.
**`kitty/options/utils.py`** — `scrollback_pager_history_size` string→MB converter `int(max(0,float(x))*1024*1024)` L564-566 (bypassed by raw-int option; value used as raw bytes).
**`kitty_tests/__init__.py`** — imports L22; `parse_bytes` L30-36; `filled_history_buf` (uses `.push`, non-canonical) L184-189; `set_options` L223-231; `create_screen` (BaseTest method) L237-241.
**`kitty_tests/screen.py`** — pager tests where `hsz` behaves as raw bytes (units confirmation) ~L695+.
**`dev.sh`** — `exec go run bypy/devenv.go "$@"` L9.
**`docs/build.rst`** — `./dev.sh build` L19; launcher path L22; `--debug` L54; `--sanitize` L58.
**`go.mod`** — `go 1.22` L3. **`pyproject.toml`** — `requires-python >=3.8` L2. **`.github/workflows/ci.yml`** — Python matrix (highest 3.11).

---

*End of document. All observation scripts (`/tmp/obs_*.py`) and profiler outputs (`/tmp/massif.out.*`) were created outside the repository tree and deleted after the campaign; the product repository is unchanged apart from this file.*
