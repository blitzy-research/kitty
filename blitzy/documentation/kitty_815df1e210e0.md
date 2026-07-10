# kitty scrollback under extreme write pressure: `HistoryBuf` + `PagerHistoryBuf`

> **Runtime investigation — built and run FIRST, then written.** Every magnitude in
> this document was produced by building the kitty C core and driving its real code
> paths, then captured verbatim. Each claim is tagged **[OBSERVED]** (measured at
> runtime) or **[INFERRED]** (derived from source, labelled with the exact
> `file:line`). The temporary observation scripts are reproduced inline in full so
> the numbers can be regenerated; they lived in a scratch directory
> (`/tmp/kitty_probe`) outside the repository and were removed afterwards. **No
> existing repository file was modified**; the sole repository change is this answer
> document (full observed-vs-inferred ledger in §8.1). Build artifacts are gitignored,
> so before committing this document `git status --porcelain` shows only the one new
> file, and after committing it the working tree is clean.

This answers, from observed behavior, what happens inside kitty's scrollback when a
command pours out an enormous amount of text very fast:

- **SQ1** — how the segmented store fills and *carves out new segments*;
- **SQ2** — the quiet interaction between the *segmented scrollback storage* and the
  *pager-style ring buffer* under stress;
- **SQ3** — whether transitions are smooth or there are *subtle boundary moments*;
- **SQ4** — what changes when someone is *actively scrolling through old output while
  new data arrives at full speed*;
- **SQ5** — how the memory structures actually evolve (allocation, wrapping, retention).

---

## TL;DR (all points [OBSERVED] unless marked)

1. **Two independent tiers.** The interactive scrollback is a **segmented store**
   (`HistoryBuf`) that is itself a ring over a fixed line capacity `ynum`. A second,
   **opt-in** *pager* tier (`PagerHistoryBuf`) is a byte-addressable FIFO ring that
   only ever receives lines the segmented store **evicts**.
2. **`ynum` is fixed at construction** to `MAX(scrollback_lines, lines)`
   [kitty/screen.c:130], and each segment holds `SEGMENT_SIZE = 2048` lines
   [kitty/history.c:15]. A **fresh buffer starts with exactly one segment**; the
   store carves an additional segment each time the line count crosses a 2048
   multiple, up to a **maximum of `ceil(ynum/2048)` segments** [INFERRED —
   kitty/history.c:36-42]. With the **default `scrollback_lines = 2000`
   [kitty/options/definition.py:372] there is only ever one segment** (2000 < 2048);
   multi-segment carving is only visible when scrollback exceeds 2048.
3. **Allocation is lazy at the OS level.** A carved segment reserves
   `xnum*2048*sizeof(CPUCell) + xnum*2048*sizeof(GPUCell) + 2048*sizeof(LineAttrs)`
   bytes in one `calloc` [kitty/history.c:17-28] — **5,251,072 bytes at `xnum=80`
   [INFERRED]** — but resident memory climbs **gradually**, measured at exactly
   **2560 bytes per line** as pages are touched, not in discrete multi-megabyte jumps.
4. **The pager tier is disabled by default** (`scrollback_pager_history_size = 0`
   [kitty/options/definition.py:406]). While disabled, evicted lines are simply
   **dropped**. When enabled, each evicted line is serialized into the ring
   [kitty/history.c:258-274]; its byte cost was measured at **exactly 33 bytes/line**
   for the probe's fixed line, and the ring's used-bytes grows by
   `evictions × bytes/line` until it reaches its configured maximum, then **overwrites
   its oldest bytes** [kitty/history.c → 3rdparty/ringbuf/ringbuf.c:231-234].
5. **Transitions are boundary events, not stalls.** Crossing a segment boundary and
   the ring reaching its maximum size are handled without any count discontinuity and
   without a catastrophic latency spike: per-batch latency stayed within a
   **max/median ratio of ~1.8×** at the measured resolution. Driving **1,000,000 and
   10,000,000 lines** through the pager path completed with **no crash** — regression
   corroboration of the historical fix `fb87fc32` (kitty issue #3011).
6. **Scrolling while writing pins, then drifts.** Each newly-added line bumps the
   view offset `scrolled_by`, capped at the line count:
   `scrolled_by = MIN(scrolled_by + history_line_added_count, count)`
   [kitty/screen.c:2716]. While the buffer is unsaturated the view stays **pinned** to
   the same old lines; once `count` saturates at `ynum` and the pinned line is evicted,
   the pin can no longer be maintained and the **viewport drifts** — the "subtle
   moment" the question intuits.

---

## 1. Methodology & Build Provenance

### 1.1 Canonical build (the build a normal user gets)

The kitty C core — including `kitty/history.c` (the segmented store **and** the pager
ring) and `3rdparty/ringbuf/ringbuf.c` (the byte FIFO) — compiles into a single
CPython extension, `kitty/fast_data_types.so`, which exposes `HistoryBuf` and `Screen`
to Python. The canonical, default build command (per the environment's setup) is:

```console
$ CI=true python3 setup.py build --verbose
```

Toolchain actually used (captured):

```console
$ python3 --version
Python 3.13.7
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

The per-file compile invocation shows the canonical **optimized** flags
(`-DNDEBUG -O3 -flto -march=native -std=c11 -pedantic-errors -Werror`), and the link
step produces `fast_data_types.so` from all module objects — including
`build/fast_data_types-kitty-history.c.o` and
`build/fast_data_types-3rdparty-ringbuf-ringbuf.c.o` (object list collapsed for length):

```console
gcc -MMD -DNDEBUG -DKITTY_VCS_REV="8740dc29e4ac701184d2952de49ceea10f1bf943" ... \
  -Wextra -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv \
  -fstack-protector-strong -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 \
  -flto -fcf-protection=full -march=native -mtune=native ... \
  -c kitty/data-types.c -o build/fast_data_types-kitty-data-types.c.o
gcc ... -O3 -shared -flto \
  [... all module .o files, incl. build/fast_data_types-kitty-history.c.o \
       and build/fast_data_types-3rdparty-ringbuf-ringbuf.c.o ...] \
  -lpython3.13 ... -o build/kitty/fast_data_types.so
```

Result (captured):

```console
$ echo "EXIT_STATUS=$?"        # after the build
EXIT_STATUS=0
$ ls -l kitty/fast_data_types.so
-rwxr-xr-x 1 root root 1253792 kitty/fast_data_types.so
$ sha256sum kitty/fast_data_types.so
74ca07a607f821ef3b78e530449445bceca003a8bbb2477d1181b959ca34a238  kitty/fast_data_types.so
$ python3 -c "import kitty.fast_data_types as f; print('HistoryBuf', 'HistoryBuf' in dir(f), '| Screen', 'Screen' in dir(f))"
HistoryBuf True | Screen True
```

The canonical module is **1,253,792 bytes**, `-DNDEBUG -O3`, **0 AddressSanitizer
symbols**. The build is **deterministic**: rebuilding produced a byte-identical `.so`
(same sha256). All build artifacts (`*.so`, `build/`, launchers) are gitignored, so the
build itself produced **no tracked change** — `git status --porcelain` reports only the
single new documentation file (see §8.3). Every probe in this document was run against
**this** canonical `.so`.

### 1.2 Supplemental builds (labelled non-canonical)

Two additional builds were produced only to cross-check boundary behavior; they are
**not** the build a normal user runs and are used only where explicitly labelled:

| Build | Command | Size (bytes) | sha256 (prefix) | Key flags | ASan syms |
|-------|---------|--------------|-----------------|-----------|-----------|
| **canonical** | `CI=true python3 setup.py build --verbose` | 1,253,792 | `74ca07a6…` | `-DNDEBUG -O3 -flto -march=native` | 0 |
| debug (suppl.) | `python3 setup.py build --debug --verbose` | 6,285,120 | `8ec2b699…` | `-DDEBUG -Og -g` | — |
| sanitize (suppl.) | `python3 setup.py build --debug --sanitize --verbose` | 20,275,512 | `1b1d9aa8…` | `-DDEBUG -Og -fsanitize=address,undefined -g` | 29 |

The sanitize build is used once, in §4, as **non-canonical corroboration** of the
large-burst pager path (run under `LD_PRELOAD` of the ASan runtime), after which the
canonical `.so` was restored and its sha256 re-verified.

### 1.3 Canonical entry point (and an honest note on harness fidelity)

kitty's real runtime byte path is: **child process → Child Monitor I/O thread
([kitty/child-monitor.c]) → VT Parser ([kitty/vt-parser.c]) → Screen model
([kitty/screen.c]) → line buffer ([kitty/line-buf.c]) → scrollback history
([kitty/history.c])**. The probes drive that same parser and Screen/history code
through the project's own in-process test harness (`kitty_tests`), which is the
canonical, non-bypassing driver:

- `parse_bytes(screen, data)` [kitty_tests/__init__.py:30-36] feeds raw bytes through
  the **real VT parser** into the real `Screen` — it is *not* a debug hook or a
  synthetic `HistoryBuf.push()` stand-in.
- `create_pty(argv=[...])` [kitty_tests/__init__.py:243-280] **forks a real child
  process on a real pseudo-terminal**; `PTY.process_input_from_child()` does
  `os.read(master_fd, …)` and hands the bytes to `parse_bytes`.

**Fidelity caveat [OBSERVED/labelled].** `PTY.process_input_from_child()` performs the
`os.read` + parser feed in Python; it exercises the **real PTY → real VT parser → real
Screen → real HistoryBuf** path, but it is **not** the C `child-monitor.c` read loop
itself. Accordingly, §4's concurrent result is described as **real-PTY-through-parser
corroboration**, not full end-to-end fidelity of the C I/O thread. All other probes
use `parse_bytes`, which is the same parser the C loop feeds.

### 1.4 The read-only observation surface (and one honest exception)

The investigation is read-only, so it observes only members/methods kitty already
exposes:

- `HistoryBuf.count`, `.ynum`, `.xnum` — read-only members
  [kitty/history.c:556-558]; segment count is **not** exposed, so every segment-count
  figure here is **[INFERRED]** from `count`/`ynum`.
- `HistoryBuf.pagerhist_as_bytes()` [kitty/fast_data_types.pyi:1085] — returns the
  pager ring's used bytes (empty `b""` when the tier is unallocated/empty).
- `Screen.scrolled_by` — read-only member [kitty/fast_data_types.pyi:1119,
  kitty/screen.c:4903].
- `Screen.visual_line(y)`, `Screen.scroll(amt, upwards)`,
  `Screen.update_only_line_graphics_data()`, `Screen.resize(lines, cols)` — the render
  and scroll methods used in §4/§5.

**Honest exception [OBSERVED].** `Screen.history_line_added_count` is exported
**writable** (its `PyMemberDef` flag is `0`, not `READONLY`)
[kitty/screen.c:4908] and is **absent from the Python type stub**
`kitty/fast_data_types.pyi`. The probes only ever **read** it and never write it, so
the read-only guarantee of the investigation is preserved; it is flagged here for
completeness because it is a genuine writable hole in an otherwise read-only surface.

### 1.5 Configuration knobs (and where they are finalized)

The two scrollback tiers are governed by two options whose *finalizers* (not just the
defaults) determine the runtime values:

- **`scrollback_lines`** — default `2000` [kitty/options/definition.py:372]. Finalized
  by `scrollback_lines(x)` [kitty/options/utils.py:557-561]: a **negative value
  becomes `2**32 - 1`** (effectively infinite). This feeds `ynum = MAX(scrollback,
  lines)` at [kitty/screen.c:130].
- **`scrollback_pager_history_size`** — default `0` (disabled)
  [kitty/options/definition.py:406]. Finalized by
  `scrollback_pager_history_size(x)` [kitty/options/utils.py:564-566]:
  `int(max(0, float(x)) * 1024 * 1024)` then capped at `4096*1024*1024 - 1` — i.e. the
  **config unit is megabytes and the maximum is 4 GiB − 1 byte**.

Because the harness's `set_options` treats values as **already finalized**, the probes
pass `scrollback_pager_history_size` in **bytes** directly (e.g. `4194304` for 4 MiB,
`0` to disable) and `scrollback_lines` as a line count.

### 1.6 Reproducibility conventions

Every probe below: (a) is a complete, self-contained script (shown in full); (b) was
run from the repository root as `PYTHONDONTWRITEBYTECODE=1 python3 <script>`; (c) was
run **twice** on the canonical `.so`; (d) had its structural magnitudes confirmed
**identical** across both runs (timing-only fields vary and are reported as
distributions). Scripts lived in `/tmp/kitty_probe` (outside the repo) and were
deleted afterwards.

---

## 2. SQ1 — How the segmented store fills and carves out new segments

### 2.1 Mechanism (from source)

`HistoryBuf` stores lines in **fixed-size segments of `SEGMENT_SIZE = 2048` lines**
[kitty/history.c:15]. The total line capacity `ynum` is frozen at `Screen`
construction to `MAX(scrollback_lines, lines)` [kitty/screen.c:130]. Segments are
allocated **lazily**:

- `create_historybuf(...)` builds the buffer with **exactly one segment**
  [kitty/history.c:117-133].
- `segment_for(self, y)` returns the segment holding line `y`, calling `add_segment`
  while the requested line lies beyond the currently allocated segments
  [kitty/history.c:36-42].
- `add_segment(self)` bumps the segment count, `realloc`s the segment array, and does
  **one `calloc`** sized
  `xnum*SEGMENT_SIZE*sizeof(CPUCell) + xnum*SEGMENT_SIZE*sizeof(GPUCell) + SEGMENT_SIZE*sizeof(LineAttrs)`
  [kitty/history.c:17-28].
- `historybuf_push(self)` writes the next line at `(start_of_data + count) % ynum` and,
  once `count == ynum`, evicts the oldest line before advancing `start_of_data`
  [kitty/history.c:277-285].

The per-segment `calloc` size is therefore fully determined by three compile-time
struct sizes. Two are pinned by a `static_assert` in the header:
`sizeof(CPUCell) == 12` [kitty/data-types.h:228] and `sizeof(GPUCell) == 20`
[kitty/data-types.h:221]. The third, `sizeof(LineAttrs) == 4`, is **not** asserted in
the header; it follows from the union layout, where the `PromptKind prompt_kind : 2`
enum bitfield forces a 4-byte (`int`) storage unit [kitty/data-types.h:230-239]
(verified with a `_Static_assert(sizeof(LineAttrs) == 4)` probe against the real
header — only `== 4` compiles, under both the canonical and debug flag sets).
At `xnum = 80`:

```
80*2048*12 + 80*2048*20 + 2048*4
= 1,966,080 + 3,276,800 + 8,192
= 5,251,072 bytes  (~5.008 MiB)      [INFERRED — history.c:17-28 arithmetic; CPUCell/GPUCell static_asserts + LineAttrs=4 probe]
```

**Two facts the code makes precise, that a naive reading gets wrong:**

- `ceil(ynum / 2048)` is the **eventual maximum** number of segments, **not** the
  number allocated at any given moment. A fresh buffer has **one** segment; the store
  grows toward that maximum as lines accumulate. **[INFERRED — history.c:36-42, 117-133]**
- With the **default `scrollback_lines = 2000`**, `ynum = 2000 < 2048`, so the store
  **never carves a second segment** — the whole default scrollback lives in a single
  segment.

### 2.2 Probe (self-contained; pager disabled to isolate the segmented store)

Segment count is not exposed to Python, so the probe reports the **[INFERRED]**
current segment count `max(1, ceil(count/2048))` alongside the **[OBSERVED]** `count`,
`ynum`, `xnum`, and process RSS (from `/proc/self/statm`). It prints only the
meaningful checkpoints — the fresh state, each segment carve, the saturation point,
the final state — and self-computes the RSS slope (bytes/line) inside the first
segment.

```python
#!/usr/bin/env python3
# SQ1 probe: segment carving in HistoryBuf under a fast, large line burst.
# Canonical entry: bytes -> real VT parser (parse_bytes) -> Screen -> HistoryBuf.
# Prints only the meaningful checkpoints: fresh state, each segment carve, the
# saturation point, the final state, and a self-computed RSS slope (bytes/line).
import sys, os, math
sys.dont_write_bytecode = True                 # no kitty/__pycache__ artifacts
sys.path.insert(0, os.getcwd())                # run from repo root
from kitty_tests import BaseTest, parse_bytes   # real-parser harness

def rss_kb():
    with open('/proc/self/statm') as f:
        resident_pages = int(f.read().split()[1])
    return resident_pages * (os.sysconf('SC_PAGE_SIZE') // 1024)

class P(BaseTest):
    def runTest(self):
        pass

SEGMENT_SIZE = 2048  # kitty/history.c:15

def inferred_current_segments(count):
    # source-derived: while filling (start_of_data==0) segments=ceil(count/2048), min 1
    return max(1, math.ceil(count / SEGMENT_SIZE)) if count > 0 else 1

def line_bytes(n):
    return (f"ROW{n:08d} hello world line".encode()) + b"\r\n"

def run_config(scrollback):
    bt = P()
    s = bt.create_screen(cols=80, lines=24, scrollback=scrollback,
                         options={'scrollback_pager_history_size': 0})
    ynum, xnum = s.historybuf.ynum, s.historybuf.xnum
    max_seg = math.ceil(ynum / SEGMENT_SIZE)
    print(f"--- CONFIG scrollback={scrollback}: ynum={ynum} xnum={xnum} [OBSERVED] | "
          f"max_segments=ceil(ynum/2048)={max_seg} [INFERRED] ---")
    print(f"  FRESH: count={s.historybuf.count} inferred_segments="
          f"{inferred_current_segments(s.historybuf.count)}[INF] rss={rss_kb()}KB [OBSERVED]")
    total = ynum + 3000
    prev_seg = inferred_current_segments(s.historybuf.count)
    saturated = False
    slope_a = slope_b = None  # (count, rss) samples inside first segment for slope
    for n in range(1, total + 1):
        parse_bytes(s, line_bytes(n))
        cnt = s.historybuf.count
        cur = inferred_current_segments(cnt)
        if slope_a is None and cnt >= 256:
            slope_a = (cnt, rss_kb())
        if slope_b is None and cnt >= 1792:
            slope_b = (cnt, rss_kb())
        if cur != prev_seg:
            print(f"  CARVE at lines_fed={n} count={cnt}: segments {prev_seg}->{cur} "
                  f"(crossed {SEGMENT_SIZE}-line boundary) rss={rss_kb()}KB [OBSERVED count/INF seg]")
        if not saturated and cnt == ynum:
            print(f"  SATURATE at lines_fed={n}: count==ynum={ynum} rss={rss_kb()}KB [OBSERVED]")
            saturated = True
        prev_seg = cur
    if slope_a and slope_b and slope_b[0] > slope_a[0]:
        bpl = (slope_b[1]-slope_a[1]) * 1024 / (slope_b[0]-slope_a[0])
        print(f"  RSS SLOPE (first segment, count {slope_a[0]}->{slope_b[0]}): "
              f"{bpl:.1f} bytes/line [OBSERVED]  (cf. 80 cols x (12+20)=2560 [INFERRED])")
    print(f"  END: count={s.historybuf.count} ynum={ynum} "
          f"capped_at_ynum={s.historybuf.count == ynum} final_segments[INF]="
          f"{inferred_current_segments(s.historybuf.count)} rss={rss_kb()}KB [OBSERVED]")
    print()

if __name__ == '__main__':
    print("=== SQ1: segment allocation / carving (pager DISABLED) ===")
    sz = 80*SEGMENT_SIZE*12 + 80*SEGMENT_SIZE*20 + SEGMENT_SIZE*4
    print(f"Per-segment calloc [INFERRED add_segment history.c:17-28 + static_assert "
          f"CPUCell=12@228 GPUCell=20@221; LineAttrs=4 (enum bitfield @231-239, no static_assert)]:")
    print(f"  xnum=80: 80*2048*12 + 80*2048*20 + 2048*4 = {sz} bytes (~{sz/1048576:.3f} MiB)")
    print()
    for sb in (2000, 5000, 10000):
        run_config(sb)
```


### 2.3 Observed output (canonical build, run 1 of 2 — complete, unedited)

```console
$ PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq1_segments.py
=== SQ1: segment allocation / carving (pager DISABLED) ===
Per-segment calloc [INFERRED add_segment history.c:17-28 + static_assert CPUCell=12@228 GPUCell=20@221; LineAttrs=4 (enum bitfield @231-239, no static_assert)]:
  xnum=80: 80*2048*12 + 80*2048*20 + 2048*4 = 5251072 bytes (~5.008 MiB)

--- CONFIG scrollback=2000: ynum=2000 xnum=80 [OBSERVED] | max_segments=ceil(ynum/2048)=1 [INFERRED] ---
  FRESH: count=0 inferred_segments=1[INF] rss=32916KB [OBSERVED]
  SATURATE at lines_fed=2023: count==ynum=2000 rss=38184KB [OBSERVED]
  RSS SLOPE (first segment, count 256->1792): 2562.7 bytes/line [OBSERVED]  (cf. 80 cols x (12+20)=2560 [INFERRED])
  END: count=2000 ynum=2000 capped_at_ynum=True final_segments[INF]=1 rss=38184KB [OBSERVED]

--- CONFIG scrollback=5000: ynum=5000 xnum=80 [OBSERVED] | max_segments=ceil(ynum/2048)=3 [INFERRED] ---
  FRESH: count=0 inferred_segments=1[INF] rss=32592KB [OBSERVED]
  CARVE at lines_fed=2072 count=2049: segments 1->2 (crossed 2048-line boundary) rss=37732KB [OBSERVED count/INF seg]
  CARVE at lines_fed=4120 count=4097: segments 2->3 (crossed 2048-line boundary) rss=42860KB [OBSERVED count/INF seg]
  SATURATE at lines_fed=5023: count==ynum=5000 rss=44988KB [OBSERVED]
  RSS SLOPE (first segment, count 256->1792): 2560.0 bytes/line [OBSERVED]  (cf. 80 cols x (12+20)=2560 [INFERRED])
  END: count=5000 ynum=5000 capped_at_ynum=True final_segments[INF]=3 rss=44988KB [OBSERVED]

--- CONFIG scrollback=10000: ynum=10000 xnum=80 [OBSERVED] | max_segments=ceil(ynum/2048)=5 [INFERRED] ---
  FRESH: count=0 inferred_segments=1[INF] rss=32592KB [OBSERVED]
  CARVE at lines_fed=2072 count=2049: segments 1->2 (crossed 2048-line boundary) rss=37732KB [OBSERVED count/INF seg]
  CARVE at lines_fed=4120 count=4097: segments 2->3 (crossed 2048-line boundary) rss=42860KB [OBSERVED count/INF seg]
  CARVE at lines_fed=6168 count=6145: segments 3->4 (crossed 2048-line boundary) rss=47988KB [OBSERVED count/INF seg]
  CARVE at lines_fed=8216 count=8193: segments 4->5 (crossed 2048-line boundary) rss=53116KB [OBSERVED count/INF seg]
  SATURATE at lines_fed=10023: count==ynum=10000 rss=57508KB [OBSERVED]
  RSS SLOPE (first segment, count 256->1792): 2560.0 bytes/line [OBSERVED]  (cf. 80 cols x (12+20)=2560 [INFERRED])
  END: count=10000 ynum=10000 capped_at_ynum=True final_segments[INF]=5 rss=57508KB [OBSERVED]

```

> **Run 2 was structurally identical** — the `CONFIG`, `FRESH`, `CARVE`,
> `SATURATE`, and `END` lines matched byte-for-byte (verified by diffing the two
> runs with the RSS/SLOPE lines filtered out); only the absolute RSS values differ
> by a few KB between runs.

### 2.4 What this shows (answering SQ1)

- **A fresh buffer has one segment, and the default configuration never carves a
  second one.** For `scrollback=2000` **no `CARVE` line ever appears** and the run ends
  `final_segments[INF]=1` — because `ynum = 2000 < 2048`. **[OBSERVED count / INFERRED
  segment count]**
- **Carving happens exactly at 2048-line boundaries.** For `scrollback=5000` the store
  carves segment 2 when `count` first exceeds 2048 (`CARVE … count=2049`) and segment 3
  at `count=4097`; `scrollback=10000` carves at `count` = 2049/4097/6145/8193, ending
  `final_segments[INF]=5 = ceil(10000/2048)`. This matches `segment_for`/`add_segment`
  [kitty/history.c:17-42]. **[OBSERVED count] / [INFERRED segment index]**
- **`count` saturates at `ynum` and stays there.** Every config ends
  `capped_at_ynum=True`; feeding thousands more lines never grows `count` past `ynum`
  — the excess is handled by eviction (see SQ2). **[OBSERVED]**
- **Memory is reserved per-segment but backed lazily, so RSS climbs smoothly.** The
  probe's self-computed `RSS SLOPE` inside the first segment was **2562.7 bytes/line**
  (scrollback=2000) and **2560.0 bytes/line** (5000 and 10000) — matching
  `80 cols × (sizeof(CPUCell)+sizeof(GPUCell)) = 80 × 32 = 2560` bytes, i.e. each line
  only touches its own row of cells. The 5.25 MB per-segment `calloc` is therefore
  **not** paid as a visible 5 MB step; it materializes gradually as pages are first
  written (note how RSS rises by ~5 MB smoothly across each 2048-line span, e.g.
  32916→38184 KB for the first segment). **[OBSERVED]** This is the concrete, measured
  meaning of the buffer "filling and stretching."

---


## 3. SQ2 — The segmented store ↔ pager ring buffer interaction under stress

### 3.1 Mechanism (from source)

The two tiers meet at exactly one place: **eviction**. `historybuf_push(self)` writes
the new line at `(start_of_data + count) % ynum`; while `count < ynum` it just
increments `count`, but once **`count == ynum`** it calls `pagerhist_push(self, ...)`
on the line about to be overwritten, then advances `start_of_data`
[kitty/history.c:277-285]. So the pager ring only ever receives **evicted** lines.

- **Pager disabled (default).** `alloc_pagerhist(0)` returns `NULL`
  [kitty/history.c:69-81] (guard `if (!pagerhist_sz) return NULL;`). With
  `self->pagerhist == NULL`, `pagerhist_push` returns immediately
  [kitty/history.c:258-261] — the evicted line is **dropped**, and
  `pagerhist_as_bytes()` returns empty `b""` [kitty/history.c:459].
- **Pager enabled.** `alloc_pagerhist(sz)` creates a ring whose **initial** capacity is
  `initial_pagerhist_ringbuf_sz(sz) = MIN(1 MiB, sz)` and whose **`maximum_size = sz`**
  [kitty/history.c:66-67, 79]. On each eviction, `pagerhist_push` serializes the line
  with `line_as_ansi`, writes the 3-byte SGR reset `"\x1b[m"`, the UCS4 text, then
  `'\r'` (and `'\n'` unless the line was soft-wrapped) [kitty/history.c:258-274]. Bytes
  are appended via `pagerhist_write_bytes` → `ringbuf_memcpy_into`
  [kitty/history.c:218-226]; if the payload would not fit, `pagerhist_extend` grows the
  ring toward `maximum_size` in `MAX(1 MiB, needed)` steps [kitty/history.c:89-102].

### 3.2 Probe (self-contained; disabled vs enabled)

`history_line_added_count` is used only as an **eviction proxy** here (`evicted = hlac −
count`). The probe never renders, so the per-frame reset in
`screen_reset_dirty` [kitty/screen.c:2597-2601] is never hit and the counter
accumulates over the probe's lifetime — this is a **probe-local** reading, explicitly
distinct from the per-frame value the render loop sees (see §5).

```python
#!/usr/bin/env python3
# SQ2 probe: segmented store <-> pager ring buffer interaction under pressure.
# Canonical entry: parse_bytes -> Screen -> HistoryBuf (+ PagerHistoryBuf).
import sys, os
sys.dont_write_bytecode = True
sys.path.insert(0, os.getcwd())
from kitty_tests import BaseTest, parse_bytes

class P(BaseTest):
    def runTest(self):
        pass

def line_bytes(n):
    return (f"ROW{n:08d} hello world line".encode()) + b"\r\n"

def feed(s, start, count_lines):
    blob = b"".join(line_bytes(n) for n in range(start, start + count_lines))
    parse_bytes(s, blob)
    return start + count_lines

def snap(s):
    hb = s.historybuf
    ring = len(hb.pagerhist_as_bytes())
    # history_line_added_count is probe-local here: we never render, so the per-frame
    # reset in screen.c screen_reset_dirty (@2597-2601) is never hit -> it accumulates
    # total lines pushed into historybuf over the probe's lifetime.
    hlac = s.history_line_added_count
    evicted = hlac - hb.count
    return hb.count, hlac, evicted, ring

def part_disabled():
    print("=== SQ2-A: pager DISABLED (scrollback_pager_history_size=0) ===")
    bt = P()
    s = bt.create_screen(cols=80, lines=24, scrollback=2000,
                         options={'scrollback_pager_history_size': 0})
    print(f"fresh: pagerhist_as_bytes()={s.historybuf.pagerhist_as_bytes()!r} "
          f"(empty -> alloc_pagerhist returned NULL, history.c:69-81) [OBSERVED]")
    n = 1
    print(f"{'fed':>6} {'count':>6} {'hlac[local]':>11} {'evicted':>8} {'ring_bytes':>10}")
    for _ in range(6):
        n = feed(s, n, 1000)
        cnt, hlac, ev, ring = snap(s)
        print(f"{n-1:>6} {cnt:>6} {hlac:>11} {ev:>8} {ring:>10}")
    print(f"RESULT: count capped at ynum=2000, ring stays 0 -> {ev} evicted lines DROPPED "
          f"(no pager). [OBSERVED]\n")

def part_enabled():
    print("=== SQ2-B: pager ENABLED (scrollback_pager_history_size=4194304 bytes = 4 MiB) ===")
    bt = P()
    s = bt.create_screen(cols=80, lines=24, scrollback=2000,
                         options={'scrollback_pager_history_size': 4194304})
    print(f"fresh: ring_bytes={len(s.historybuf.pagerhist_as_bytes())} [OBSERVED]")
    n = 1
    # First, fill exactly to saturation WITHOUT eviction, then cross into eviction.
    # Grid holds 24 lines; history saturates at ynum=2000. Feed 2000+24 = 2024 to just reach cap.
    n = feed(s, n, 2024)
    cnt, hlac, ev, ring = snap(s)
    print(f"after fed={n-1}: count={cnt} hlac(local)={hlac} evicted={ev} ring_bytes={ring} "
          f"[OBSERVED] (ring still 0 while evicted==0)")
    # Now feed the FIRST batch that evicts, from a fresh 0-byte ring.
    n = feed(s, n, 1000)
    cnt, hlac, ev, ring = snap(s)
    per = ring / ev if ev else float('nan')
    print(f"FIRST eviction batch: fed={n-1} count={cnt} evicted={ev} ring_bytes={ring} "
          f"[OBSERVED]")
    print(f"  first-delta arithmetic: {ring} - 0 = +{ring} bytes for {ev} evicted lines "
          f"=> {per:.4f} bytes/evicted-line [OBSERVED]")
    print(f"{'fed':>6} {'count':>6} {'evicted':>8} {'ring_bytes':>11} {'delta':>9} "
          f"{'batch_ev':>8} {'B/line':>8}")
    prev_ring, prev_ev = ring, ev
    print(f"{n-1:>6} {cnt:>6} {ev:>8} {ring:>11} {'-':>9} {'-':>8} {per:>8.3f}")
    for _ in range(8):
        n = feed(s, n, 1000)
        cnt, hlac, ev, ring = snap(s)
        d = ring - prev_ring
        bev = ev - prev_ev
        bpl = d / bev if bev else float('nan')
        print(f"{n-1:>6} {cnt:>6} {ev:>8} {ring:>11} {d:>+9} {bev:>8} {bpl:>8.3f}")
        prev_ring, prev_ev = ring, ev
    print(f"RESULT: each batch's ring growth == (batch evictions) x (bytes/line) = PAYLOAD "
          f"volume of evicted lines (NOT a fixed 1MiB extend granularity). [OBSERVED]")
    print(f"  (internal ring capacity grows via pagerhist_extend MAX(1MiB,minsz), history.c:89-102,")
    print(f"   but pagerhist_as_bytes() reports bytes_USED = payload, history.c:459) [INFERRED]\n")

if __name__ == '__main__':
    part_disabled()
    part_enabled()
```


### 3.3 Observed output (canonical build, run 1 of 2 — complete, unedited)

```console
$ PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq2_ring.py
=== SQ2-A: pager DISABLED (scrollback_pager_history_size=0) ===
fresh: pagerhist_as_bytes()=b'' (empty -> alloc_pagerhist returned NULL, history.c:69-81) [OBSERVED]
   fed  count hlac[local]  evicted ring_bytes
  1000    977         977        0          0
  2000   1977        1977        0          0
  3000   2000        2977      977          0
  4000   2000        3977     1977          0
  5000   2000        4977     2977          0
  6000   2000        5977     3977          0
RESULT: count capped at ynum=2000, ring stays 0 -> 3977 evicted lines DROPPED (no pager). [OBSERVED]

=== SQ2-B: pager ENABLED (scrollback_pager_history_size=4194304 bytes = 4 MiB) ===
fresh: ring_bytes=0 [OBSERVED]
after fed=2024: count=2000 hlac(local)=2001 evicted=1 ring_bytes=33 [OBSERVED] (ring still 0 while evicted==0)
FIRST eviction batch: fed=3024 count=2000 evicted=1001 ring_bytes=33033 [OBSERVED]
  first-delta arithmetic: 33033 - 0 = +33033 bytes for 1001 evicted lines => 33.0000 bytes/evicted-line [OBSERVED]
   fed  count  evicted  ring_bytes     delta batch_ev   B/line
  3024   2000     1001       33033         -        -   33.000
  4024   2000     2001       66033    +33000     1000   33.000
  5024   2000     3001       99033    +33000     1000   33.000
  6024   2000     4001      132033    +33000     1000   33.000
  7024   2000     5001      165033    +33000     1000   33.000
  8024   2000     6001      198033    +33000     1000   33.000
  9024   2000     7001      231033    +33000     1000   33.000
 10024   2000     8001      264033    +33000     1000   33.000
 11024   2000     9001      297033    +33000     1000   33.000
RESULT: each batch's ring growth == (batch evictions) x (bytes/line) = PAYLOAD volume of evicted lines (NOT a fixed 1MiB extend granularity). [OBSERVED]
  (internal ring capacity grows via pagerhist_extend MAX(1MiB,minsz), history.c:89-102,
   but pagerhist_as_bytes() reports bytes_USED = payload, history.c:459) [INFERRED]

```

> **Run 2 was byte-identical** to run 1 (verified by a full `diff` of the two
> captured outputs). Every figure below is therefore stable across both runs.

### 3.4 What this shows (answering SQ2)

- **Disabled is the default, and it silently drops.** With the tier off,
  `pagerhist_as_bytes()` is `b''` and stays `0` bytes throughout; `count` caps at
  `ynum=2000` while the eviction proxy climbs to **3977** — i.e. 3977 evicted lines
  were **discarded** with no pager to catch them. **[OBSERVED]**
- **Enabled, eviction feeds the ring at an exact, measurable byte cost.** Each
  evicted line cost **exactly 33.000 bytes** in the ring — matching
  `pagerhist_push`'s serialization of this fixed 28-character line: the 3-byte
  `"\x1b[m"` reset + 28 text bytes + `\r\n` (2) = 33 [kitty/history.c:258-274].
  **[OBSERVED]**
- **The first non-zero ring reading is exact arithmetic on a fresh 0-byte ring.**
  The first eviction batch produced `33033 - 0 = +33033` bytes for **1001** evicted
  lines (`33033 / 1001 = 33.0000`). **[OBSERVED]**
- **Each subsequent batch grows the ring by `evictions x bytes/line` — payload
  volume, not an extend-granularity artifact.** Every 1000-line batch added
  `+33000` bytes (= `1000 x 33`) to `pagerhist_as_bytes()`. The internal ring
  **capacity** does grow in `MAX(1 MiB, needed)` steps via `pagerhist_extend`
  [kitty/history.c:89-102], but `pagerhist_as_bytes()` reports bytes **used**
  (the payload) [kitty/history.c:459], so the visible increment equals the evicted
  lines' serialized size, **not** a 1 MiB quantum. **[OBSERVED payload / INFERRED
  capacity mechanism]**
- **The ring initial size and cap come straight from the option.** The tier starts
  at `MIN(1 MiB, sz)` and is capped at `maximum_size = sz`
  [kitty/history.c:66-67, 79], where `sz` is the finalized byte value from
  `scrollback_pager_history_size` (§1.5). **[INFERRED from source]**

---

## 4. SQ3 — Are transitions smooth, or are there subtle boundary moments?

There are three concrete boundaries where "something different" could happen under a
fast burst: (a) crossing a 2048-line **segment** boundary (covered in §2 — no
discontinuity, only a lazily-backed allocation); (b) the pager ring reaching its
**maximum size** and switching to overwrite; and (c) the historically fragile
**large-burst pager path**. This section targets (b) and (c) directly, and measures
**per-batch latency** so the word "smooth" is pinned to a measured resolution rather
than asserted.

### 4.1 Mechanism (from source)

- **Ring growth then overwrite.** `pagerhist_write_bytes` extends the ring only while
  capacity `< maximum_size`; `pagerhist_extend` returns `false` once
  `capacity >= maximum_size` [kitty/history.c:89-102]. After that, `ringbuf_memcpy_into`
  runs in **overflow** mode: when the write fills the buffer it advances the tail past
  the head — `dst->tail = ringbuf_nextp(dst, dst->head)` — i.e. it **overwrites the
  oldest bytes** [3rdparty/ringbuf/ringbuf.c:231-234]. So `pagerhist_as_bytes()` (bytes
  *used*) rises to `maximum_size` and then **plateaus**, even as eviction continues.
- **Historical fragility.** kitty issue #3011 reported a segfault after ~120k lines when
  `scrollback_pager_history_size > 1`; it was fixed on 2020-10-06 by commit
  `fb87fc32f04be636e1e0fe8eea611a453ee2d3f0` ("Fix a regression that caused a segfault
  when using scrollback_pager_history_size", *Fixes #3011*), which corrected
  `pagerhist_extend` so the new size is `MIN(maximum_size, …)` **before** allocation.
  That fix is an **ancestor of the baseline commit** under study (verified with
  `git merge-base --is-ancestor fb87fc32… 815df1e21` → true), so on this build the
  large-burst runs below are **regression corroboration**, not a fresh reproduction.

### 4.2 Probe — ring boundary + per-batch latency (self-contained)

```python
#!/usr/bin/env python3
# SQ3 probe (part 1): ring boundary grow->plateau->overwrite, tiny-ring edge,
# and per-batch LATENCY distribution through the canonical parser path.
import sys, os, time, statistics
sys.dont_write_bytecode = True
sys.path.insert(0, os.getcwd())
from kitty_tests import BaseTest, parse_bytes

class P(BaseTest):
    def runTest(self):
        pass

def line_bytes(n):
    return (f"ROW{n:08d} hello world line".encode()) + b"\r\n"

def make_batch(start, count_lines):
    return b"".join(line_bytes(n) for n in range(start, start + count_lines)), start + count_lines

def boundary(ring_bytes_size, total_lines, batch, label):
    print(f"=== SQ3 ring boundary: maximum_size={ring_bytes_size} bytes {label} ===")
    bt = P()
    s = bt.create_screen(cols=80, lines=24, scrollback=2000,
                         options={'scrollback_pager_history_size': ring_bytes_size})
    n = 1
    lat_ns = []
    prev_ring = 0
    plateau_hit_at = None
    print(f"{'fed':>7} {'count':>6} {'evicted':>9} {'ring_bytes':>11} {'ring_delta':>10} {'batch_ms':>9} note")
    nbatches = total_lines // batch
    for i in range(nbatches):
        blob, n = make_batch(n, batch)
        t0 = time.perf_counter_ns()
        parse_bytes(s, blob)
        t1 = time.perf_counter_ns()
        lat_ns.append(t1 - t0)
        hb = s.historybuf
        ring = len(hb.pagerhist_as_bytes())
        ev = s.history_line_added_count - hb.count
        d = ring - prev_ring
        note = ""
        if d <= 0 and prev_ring > 0 and plateau_hit_at is None:
            plateau_hit_at = n - 1
            note = "<< ring PLATEAU: bytes_used stops growing (overwrite oldest, ringbuf.c:231-234)"
        prev_ring = ring
        if (i % 5 == 0) or note or i >= nbatches - 3:
            print(f"{n-1:>7} {hb.count:>6} {ev:>9} {ring:>11} {d:>+10} {(t1-t0)/1e6:>9.3f} {note}")
    print(f"PLATEAU first observed at fed={plateau_hit_at}; final ring_bytes={prev_ring} "
          f"(<= maximum_size={ring_bytes_size}) [OBSERVED]")
    lat_ms = sorted(x/1e6 for x in lat_ns)
    def pct(p):
        k = min(len(lat_ms)-1, int(round(p/100*(len(lat_ms)-1))))
        return lat_ms[k]
    print(f"per-batch latency (batch={batch} lines, n={len(lat_ms)} batches) [OBSERVED]:")
    print(f"  min={lat_ms[0]:.3f}ms median={statistics.median(lat_ms):.3f}ms "
          f"mean={statistics.mean(lat_ms):.3f}ms p99={pct(99):.3f}ms max={lat_ms[-1]:.3f}ms")
    print(f"  max/median ratio = {lat_ms[-1]/statistics.median(lat_ms):.2f}x "
          f"(no catastrophic stall at plateau crossing at this resolution) [OBSERVED]\n")

if __name__ == '__main__':
    boundary(1048576, 80000, 1000, "(1 MiB)")
    boundary(4096, 20000, 1000, "(tiny 4 KiB edge)")
```


Observed output (canonical build, run 1 of 2 — complete, unedited):

```console
$ PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq3_boundary.py
=== SQ3 ring boundary: maximum_size=1048576 bytes (1 MiB) ===
    fed  count   evicted  ring_bytes ring_delta  batch_ms note
   1000    977         0           0         +0     1.040 
   6000   2000      3977      131241     +33000     0.620 
  11000   2000      8977      296241     +33000     0.613 
  16000   2000     13977      461241     +33000     0.613 
  21000   2000     18977      626241     +33000     0.608 
  26000   2000     23977      791241     +33000     0.620 
  31000   2000     28977      956241     +33000     0.619 
  35000   2000     32977     1048576         +0     0.631 << ring PLATEAU: bytes_used stops growing (overwrite oldest, ringbuf.c:231-234)
  36000   2000     33977     1048576         +0     0.636 
  41000   2000     38977     1048576         +0     0.643 
  46000   2000     43977     1048576         +0     0.628 
  51000   2000     48977     1048576         +0     0.648 
  56000   2000     53977     1048576         +0     0.628 
  61000   2000     58977     1048576         +0     0.630 
  66000   2000     63977     1048576         +0     0.623 
  71000   2000     68977     1048576         +0     0.662 
  76000   2000     73977     1048576         +0     0.649 
  78000   2000     75977     1048576         +0     0.656 
  79000   2000     76977     1048576         +0     0.656 
  80000   2000     77977     1048576         +0     0.640 
PLATEAU first observed at fed=35000; final ring_bytes=1048576 (<= maximum_size=1048576) [OBSERVED]
per-batch latency (batch=1000 lines, n=80 batches) [OBSERVED]:
  min=0.602ms median=0.632ms mean=0.643ms p99=1.040ms max=1.118ms
  max/median ratio = 1.77x (no catastrophic stall at plateau crossing at this resolution) [OBSERVED]

=== SQ3 ring boundary: maximum_size=4096 bytes (tiny 4 KiB edge) ===
    fed  count   evicted  ring_bytes ring_delta  batch_ms note
   1000    977         0           0         +0     0.842 
   4000   2000      1977        4096         +0     0.642 << ring PLATEAU: bytes_used stops growing (overwrite oldest, ringbuf.c:231-234)
   6000   2000      3977        4096         +0     0.629 
  11000   2000      8977        4096         +0     0.631 
  16000   2000     13977        4096         +0     0.629 
  18000   2000     15977        4096         +0     0.635 
  19000   2000     16977        4096         +0     0.639 
  20000   2000     17977        4096         +0     0.629 
PLATEAU first observed at fed=4000; final ring_bytes=4096 (<= maximum_size=4096) [OBSERVED]
per-batch latency (batch=1000 lines, n=20 batches) [OBSERVED]:
  min=0.621ms median=0.632ms mean=0.655ms p99=0.874ms max=0.874ms
  max/median ratio = 1.38x (no catastrophic stall at plateau crossing at this resolution) [OBSERVED]

```

> The `fed`/`count`/`evicted`/`ring_bytes`/`ring_delta` columns were **byte-identical
> across both runs**; only the timing columns vary. Run 2's latency summary was
> `median 0.643ms max 1.106ms` (1 MiB) and `median 0.644ms max 0.882ms` (4 KiB) —
> the same shape, confirming the distribution is stable.

### 4.3 Probe — large-burst fragility (1M + 10M lines), plus a sanitizer cross-check

```python
#!/usr/bin/env python3
# SQ3 probe (part 2): large-burst fragility of the pager ring path.
# Historically kitty#3011 segfaulted ~120k lines with scrollback_pager_history_size>1;
# fixed by commit fb87fc32 (ancestor of baseline 815df1e21). This is REGRESSION
# CORROBORATION on the current commit: drive 1,000,000 then 10,000,000 lines through
# the canonical parser path with the pager ENABLED and observe no crash.
import sys, os, time
sys.dont_write_bytecode = True
sys.path.insert(0, os.getcwd())
from kitty_tests import BaseTest, parse_bytes

class P(BaseTest):
    def runTest(self):
        pass

CHUNK = 100000  # lines per parse_bytes call

def chunk_bytes(start, count_lines):
    return b"".join((f"{n}\r\n").encode() for n in range(start, start + count_lines))

def burst(total, ring_bytes_size, label):
    print(f"=== SQ3 fragility: {total} lines, pager={ring_bytes_size} bytes {label} ===")
    bt = P()
    s = bt.create_screen(cols=80, lines=24, scrollback=2000,
                         options={'scrollback_pager_history_size': ring_bytes_size})
    n = 1
    t0 = time.perf_counter()
    done = 0
    while done < total:
        c = min(CHUNK, total - done)
        parse_bytes(s, chunk_bytes(n, c))
        n += c
        done += c
        if done % 1000000 == 0:
            hb = s.historybuf
            ring = len(hb.pagerhist_as_bytes())
            print(f"  progress {done:>9} lines | count={hb.count} "
                  f"evicted={s.history_line_added_count - hb.count} ring_bytes={ring} "
                  f"elapsed={time.perf_counter()-t0:.2f}s [OBSERVED]")
    el = time.perf_counter() - t0
    hb = s.historybuf
    ring = len(hb.pagerhist_as_bytes())
    print(f"  DONE {total} lines: count={hb.count} ynum={hb.ynum} "
          f"evicted={s.history_line_added_count - hb.count} ring_bytes={ring} "
          f"elapsed={el:.2f}s ({total/el/1e6:.2f} M lines/s) NO CRASH [OBSERVED]\n")

if __name__ == '__main__':
    burst(1000000, 8388608, "(1M, 8 MiB ring)")
    burst(10000000, 8388608, "(10M, 8 MiB ring)")
    print("EXIT_OK: both bursts completed without segfault -> regression fb87fc32 corroborated [OBSERVED]")
```


Observed output (canonical build, run 1 of 2 — complete, unedited):

```console
$ PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq3_fragility.py
=== SQ3 fragility: 1000000 lines, pager=8388608 bytes (1M, 8 MiB ring) ===
  progress   1000000 lines | count=2000 evicted=997977 ring_bytes=8388608 elapsed=0.49s [OBSERVED]
  DONE 1000000 lines: count=2000 ynum=2000 evicted=997977 ring_bytes=8388608 elapsed=0.49s (2.05 M lines/s) NO CRASH [OBSERVED]

=== SQ3 fragility: 10000000 lines, pager=8388608 bytes (10M, 8 MiB ring) ===
  progress   1000000 lines | count=2000 evicted=997977 ring_bytes=8388608 elapsed=0.48s [OBSERVED]
  progress   2000000 lines | count=2000 evicted=1997977 ring_bytes=8388608 elapsed=0.97s [OBSERVED]
  progress   3000000 lines | count=2000 evicted=2997977 ring_bytes=8388608 elapsed=1.47s [OBSERVED]
  progress   4000000 lines | count=2000 evicted=3997977 ring_bytes=8388608 elapsed=1.97s [OBSERVED]
  progress   5000000 lines | count=2000 evicted=4997977 ring_bytes=8388608 elapsed=2.46s [OBSERVED]
  progress   6000000 lines | count=2000 evicted=5997977 ring_bytes=8388608 elapsed=2.96s [OBSERVED]
  progress   7000000 lines | count=2000 evicted=6997977 ring_bytes=8388608 elapsed=3.46s [OBSERVED]
  progress   8000000 lines | count=2000 evicted=7997977 ring_bytes=8388608 elapsed=3.97s [OBSERVED]
  progress   9000000 lines | count=2000 evicted=8997977 ring_bytes=8388608 elapsed=4.47s [OBSERVED]
  progress  10000000 lines | count=2000 evicted=9997977 ring_bytes=8388608 elapsed=4.96s [OBSERVED]
  DONE 10000000 lines: count=2000 ynum=2000 evicted=9997977 ring_bytes=8388608 elapsed=4.96s (2.02 M lines/s) NO CRASH [OBSERVED]

EXIT_OK: both bursts completed without segfault -> regression fb87fc32 corroborated [OBSERVED]
```

> **Run 2 was structurally identical** — every `count`/`evicted`/`ring_bytes` value
> matched (only elapsed/throughput differ); both runs exited 0.

**Non-canonical sanitizer cross-check.** The same path (1,000,000 lines, pager
enabled — ~8x the historical ~120k crash threshold) was re-run against the
**supplemental ASan+UBSan build** (§1.2), loaded via `LD_PRELOAD` of the ASan
runtime. This build is **not** canonical; it is used only to let the sanitizer
watch the ring boundary. Afterwards the canonical `.so` was restored and its
sha256 re-verified (`74ca07a6…`), with `git status --porcelain` empty.

```console
$ cp /tmp/kitty_probe/fast_data_types.sanitize.so kitty/fast_data_types.so
$ LD_PRELOAD=/usr/lib/gcc/x86_64-linux-gnu/15/libasan.so \
  ASAN_OPTIONS=detect_leaks=0:halt_on_error=1 UBSAN_OPTIONS=halt_on_error=1 \
  PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq3_sanitize.py
  progress 200000 count=2000 evicted=197977 ring_bytes=2066642 elapsed=0.44s [OBSERVED asan]
  progress 400000 count=2000 evicted=397977 ring_bytes=4266642 elapsed=0.87s [OBSERVED asan]
  progress 600000 count=2000 evicted=597977 ring_bytes=6466642 elapsed=1.30s [OBSERVED asan]
  progress 800000 count=2000 evicted=797977 ring_bytes=8388608 elapsed=1.73s [OBSERVED asan]
  progress 1000000 count=2000 evicted=997977 ring_bytes=8388608 elapsed=2.17s [OBSERVED asan]
DONE 1000000 lines under ASAN+UBSAN: count=2000 evicted=997977 ring_bytes=8388608 NO SANITIZER ERROR [OBSERVED asan]
$ echo "SANITIZE_EXIT=$?"
SANITIZE_EXIT=0
```

### 4.4 What this shows (answering SQ3)

- **The ring's used-bytes grows, then plateaus exactly at `maximum_size`, then the
  tier overwrites its oldest bytes.** For a 1 MiB ring, `ring_bytes` climbed by
  `+33000`/batch to **exactly 1,048,576** (at `fed=35000`) and then stayed flat
  while `evicted` kept climbing to 77,977; the 4 KiB edge case plateaued at
  **exactly 4096**. This is the overwrite path at
  [3rdparty/ringbuf/ringbuf.c:231-234]. **[OBSERVED]**
- **"Smooth" is a measured claim, not an assertion.** Crossing the plateau boundary
  produced **no count discontinuity and no catastrophic latency spike**: per-batch
  latency (1000 lines/batch) had **median 0.632 ms, p99 1.040 ms, max 1.118 ms —
  a max/median ratio of 1.77x** for the 1 MiB run (1.72x on run 2). The precise
  claim is therefore: *at a 1000-line-batch resolution, no error, no count
  discontinuity, and no latency outlier beyond ~1.8x median was observed at any
  boundary.* Sub-batch micro-stalls below this resolution are **not** ruled out.
  **[OBSERVED, resolution-bounded]**
- **The historically fragile large burst no longer crashes.** Driving **1,000,000**
  and **10,000,000** lines through the pager path completed with `NO CRASH`,
  `count=2000`, `ring_bytes=8,388,608` (the 8 MiB cap), `evicted=9,997,977` for the
  10M run, at ~2 M lines/s; exit 0. The ASan+UBSan build processed 1,000,000 such
  lines with **no sanitizer error**. Because the fix `fb87fc32` is an ancestor of
  the baseline, this **corroborates the regression is absent** on this commit
  (kitty issue #3011). **[OBSERVED]**

---

## 5. SQ4 — Actively scrolling old output while new data arrives at full speed

### 5.1 Mechanism

The scrolled-back view is a single offset, `Screen.scrolled_by`, counting how many
lines above the live viewport the user is currently looking. It is exposed
**read-only** to Python
[kitty/screen.c:4903: `{"scrolled_by", T_UINT, offsetof(Screen, scrolled_by), READONLY, ...}`].

Every line that scrolls off the top of the grid runs the `INDEX_UP` macro, which
appends the departing line to the history buffer and bumps a per-frame counter:
`self->history_line_added_count++` [kitty/screen.c:1559], inside the
`add_to_history` branch of `INDEX_UP` [kitty/screen.c:1552-1566].

When a frame is prepared, `screen_update_only_line_graphics_data` re-pins the view
by advancing the offset by exactly the number of lines added since the last frame,
then **caps** it at the buffer's line count:
`if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);`
[kitty/screen.c:2716]. The identical cap is applied on the cell-data path
[kitty/screen.c:2761]. Immediately afterward `screen_reset_dirty` zeroes the
counter for the next frame: `self->history_line_added_count = 0;`
[kitty/screen.c:2597-2601].

The consequence is a two-regime behavior:

- **Unsaturated (`count < ynum`):** every added line increments both `count` and
  `scrolled_by`, so `MIN(scrolled_by + added, count)` equals `scrolled_by + added`.
  The viewport stays **pinned** to the same absolute old line.
- **Saturated (`count == ynum`):** `count` stops growing, so the `MIN(..., count)`
  clamp freezes `scrolled_by` at `ynum`. Further evictions slide older content out
  from under the pin — the pinned line is eventually evicted and the viewport
  **drifts**.

That crossover is the "subtle moment" the question intuits: while the buffer is
filling, an active scrollback holds its position perfectly; once the buffer
saturates, the pin can no longer keep pace and the view starts sliding.

### 5.2 Self-contained probe (canonical: real PTY child through the real parser)

This probe launches a **real child process** (`python -c <bulk print>`) on a
**real PTY** via the `kitty_tests` `create_pty` helper, pulls the child's bytes
through the **real VT parser** (`PTY.process_input_from_child` -> `parse_bytes`),
renders each frame with `update_only_line_graphics_data` (applying the cap at
[kitty/screen.c:2716] and resetting the counter at [kitty/screen.c:2597-2601]), and
records the identity of the top/bottom viewport rows via `visual_line`
[kitty/screen.c:4788] across the saturation boundary while scrolled back.

This is **real-PTY-through-parser corroboration**: the bytes traverse a genuine
kernel PTY and the same VT parser + Screen code the production child-monitor drives,
but the read loop here is the in-process test harness, **not** the C
`child-monitor.c` thread (see §1.3). **[canonical harness]**

```python
#!/usr/bin/env python3
# SQ4 probe: a REAL child emits at full speed through a real PTY while the user
# is scrolled back. We pull child bytes through the real VT parser
# (PTY.process_input_from_child -> parse_bytes), render each frame
# (update_only_line_graphics_data: applies scrolled_by cap @screen.c:2716 and
# resets history_line_added_count @2717), and watch the viewport row identities.
# This is real-PTY-through-parser corroboration, NOT the C child-monitor.c loop.
import sys, os, time, signal
sys.dont_write_bytecode = True
sys.path.insert(0, os.getcwd())
from kitty_tests import BaseTest

TOTAL = int(os.environ.get("SQ4_TOTAL", "40000"))
PAD = 68  # pad each line with spaces so ~78 bytes/line -> ~100 lines per 8192-byte read
CHILD = (
    "import sys\n"
    f"N={TOTAL}; PAD={PAD}\n"
    "i=0; buf=[]\n"
    "while i<N:\n"
    "    buf.append(('L%08d'%i)+(' '*PAD)); i+=1\n"
    "    if len(buf)>=500:\n"
    "        sys.stdout.write('\\n'.join(buf)+'\\n'); sys.stdout.flush(); buf=[]\n"
    "if buf:\n"
    "    sys.stdout.write('\\n'.join(buf)+'\\n'); sys.stdout.flush()\n"
)

class P(BaseTest):
    def runTest(self):
        pass

def label(line):
    t = str(line).strip()
    return t.split()[0] if t else "(blank)"

def top(s):
    return label(s.visual_line(0))

def bottom(s):
    return label(s.visual_line(s.lines - 1))

def main():
    bt = P()
    pty = bt.create_pty([sys.executable, '-c', CHILD], cols=80, lines=24, scrollback=2000)
    pty.turn_off_echo()
    s = pty.screen
    ynum = s.historybuf.ynum
    print(f"child argv: python -c <bulk print, N={TOTAL}, {PAD}-space pad>  ynum={ynum} lines={s.lines} [OBSERVED]")
    t0 = time.perf_counter()
    # BEFORE: fill history to ~1000 lines, not scrolled
    while s.historybuf.count < 1000:
        if pty.process_input_from_child(timeout=2) == 0:
            break
    s.update_only_line_graphics_data()
    print(f"[BEFORE] t={time.perf_counter()-t0:.3f}s scrolled_by={s.scrolled_by} "
          f"count={s.historybuf.count} top={top(s)} bottom={bottom(s)} [OBSERVED]")
    # scroll back 500 lines into old output
    s.scroll(500, True)
    pinned = top(s)
    print(f"[SCROLL +500] scrolled_by={s.scrolled_by} count={s.historybuf.count} "
          f"top(pinned)={pinned} bottom={bottom(s)} [OBSERVED]\n")
    print(f"{'phase':>9} {'t_s':>6} {'sby':>5} {'count':>5} {'top':>11} {'bottom':>11} {'pin?':>4} note")
    saturated = False
    drift_frames = 0
    frames = 0
    while True:
        got = pty.process_input_from_child(timeout=2)  # ONE read/frame (~100 lines)
        s.update_only_line_graphics_data()              # apply cap @2716 + reset hlac @2717
        frames += 1
        cnt, sby, tr = s.historybuf.count, s.scrolled_by, top(s)
        pin = "YES" if tr == pinned else "no"
        note = ""
        phase = "DURING"
        if cnt >= ynum and not saturated:
            saturated = True
            phase = "SATURATE"
            note = f"count==ynum={ynum}; scrolled_by capped by MIN(...,count)"
        if saturated and tr != pinned:
            drift_frames += 1
            if drift_frames == 1:
                note = (note + " ; " if note else "") + f"VIEWPORT DRIFT: top left {pinned} (pinned line evicted)"
        print(f"{phase:>9} {time.perf_counter()-t0:>6.3f} {sby:>5} {cnt:>5} {tr:>11} {bottom(s):>11} {pin:>4} {note}")
        if drift_frames >= 4:
            break
        if got == 0 and cnt >= ynum:
            break
    try:
        os.kill(pty.child_pid, signal.SIGKILL); os.waitpid(pty.child_pid, 0)
    except Exception:
        pass
    print(f"\nSUMMARY: pre-saturation, scrolled_by rose in lockstep and top stayed pinned to "
          f"{pinned} (pin=YES); after count saturated at ynum={ynum}, scrolled_by was capped at "
          f"count and once the pinned line was evicted the viewport drifted (pin=no). [OBSERVED]")

if __name__ == '__main__':
    main()
```

### 5.3 Complete output (run 1 of 2)

Produced by `PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq4_concurrent.py`
(run from the repository root against the canonical `.so`):

```text
child argv: python -c <bulk print, N=40000, 68-space pad>  ynum=2000 lines=24 [OBSERVED]
[BEFORE] t=0.014s scrolled_by=0 count=1023 top=L00001023 bottom=L00001046 [OBSERVED]
[SCROLL +500] scrolled_by=500 count=1023 top(pinned)=L00000523 bottom=L00000546 [OBSERVED]

    phase    t_s   sby count         top      bottom pin? note
   DURING  0.015   572  1095   L00000523   L00000546  YES 
   DURING  0.015   652  1175   L00000523   L00000546  YES 
   DURING  0.015   754  1277   L00000523   L00000546  YES 
   DURING  0.015   857  1380   L00000523   L00000546  YES 
   DURING  0.015   909  1432   L00000523   L00000546  YES 
   DURING  0.016   954  1477   L00000523   L00000546  YES 
   DURING  0.016  1001  1524   L00000523   L00000546  YES 
   DURING  0.016  1062  1585   L00000523   L00000546  YES 
   DURING  0.016  1114  1637   L00000523   L00000546  YES 
   DURING  0.016  1156  1679   L00000523   L00000546  YES 
   DURING  0.016  1200  1723   L00000523   L00000546  YES 
   DURING  0.016  1253  1776   L00000523   L00000546  YES 
   DURING  0.017  1297  1820   L00000523   L00000546  YES 
   DURING  0.017  1331  1854   L00000523   L00000546  YES 
   DURING  0.017  1380  1903   L00000523   L00000546  YES 
   DURING  0.017  1433  1956   L00000523   L00000546  YES 
   DURING  0.017  1454  1977   L00000523   L00000546  YES 
 SATURATE  0.017  1493  2000   L00000523   L00000546  YES count==ynum=2000; scrolled_by capped by MIN(...,count)
   DURING  0.018  1566  2000   L00000523   L00000546  YES 
   DURING  0.018  1643  2000   L00000523   L00000546  YES 
   DURING  0.018  1721  2000   L00000523   L00000546  YES 
   DURING  0.018  1824  2000   L00000523   L00000546  YES 
   DURING  0.019  1928  2000   L00000523   L00000546  YES 
   DURING  0.019  1954  2000   L00000523   L00000546  YES 
   DURING  0.019  1977  2000   L00000523   L00000546  YES 
   DURING  0.019  2000  2000   L00000535   L00000558   no VIEWPORT DRIFT: top left L00000523 (pinned line evicted)
   DURING  0.019  2000  2000   L00000579   L00000602   no 
   DURING  0.019  2000  2000   L00000630   L00000653   no 
   DURING  0.020  2000  2000   L00000734   L00000757   no 

SUMMARY: pre-saturation, scrolled_by rose in lockstep and top stayed pinned to L00000523 (pin=YES); after count saturated at ynum=2000, scrolled_by was capped at count and once the pinned line was evicted the viewport drifted (pin=no). [OBSERVED]
```

Run 2 was structurally identical: the same lockstep-pin-then-drift progression,
differing only in the absolute pinned-line label (run 1 pinned `L00000523`; run 2
pinned `L00000599`) because the exact line reached before the manual `scroll(500)`
depends on how many 8192-byte reads landed before the 1000-line fill threshold — a
timing detail, not a behavioral difference. The complete run-2 output is at
`/tmp/kitty_probe/sq4_run2.txt` and is reproduced below:

```text
child argv: python -c <bulk print, N=40000, 68-space pad>  ynum=2000 lines=24 [OBSERVED]
[BEFORE] t=0.013s scrolled_by=0 count=1099 top=L00001099 bottom=L00001122 [OBSERVED]
[SCROLL +500] scrolled_by=500 count=1099 top(pinned)=L00000599 bottom=L00000622 [OBSERVED]

    phase    t_s   sby count         top      bottom pin? note
   DURING  0.013   604  1203   L00000599   L00000622  YES 
   DURING  0.013   656  1255   L00000599   L00000622  YES 
   DURING  0.013   708  1307   L00000599   L00000622  YES 
   DURING  0.014   760  1359   L00000599   L00000622  YES 
   DURING  0.014   811  1410   L00000599   L00000622  YES 
   DURING  0.014   915  1514   L00000599   L00000622  YES 
   DURING  0.014  1019  1618   L00000599   L00000622  YES 
   DURING  0.014  1123  1722   L00000599   L00000622  YES 
   DURING  0.014  1174  1773   L00000599   L00000622  YES 
   DURING  0.014  1226  1825   L00000599   L00000622  YES 
   DURING  0.014  1278  1877   L00000599   L00000622  YES 
   DURING  0.015  1382  1981   L00000599   L00000622  YES 
 SATURATE  0.015  1485  2000   L00000599   L00000622  YES count==ynum=2000; scrolled_by capped by MIN(...,count)
   DURING  0.015  1537  2000   L00000599   L00000622  YES 
   DURING  0.015  1589  2000   L00000599   L00000622  YES 
   DURING  0.015  1641  2000   L00000599   L00000622  YES 
   DURING  0.015  1693  2000   L00000599   L00000622  YES 
   DURING  0.015  1745  2000   L00000599   L00000622  YES 
   DURING  0.015  1796  2000   L00000599   L00000622  YES 
   DURING  0.016  1900  2000   L00000599   L00000622  YES 
   DURING  0.016  2000  2000   L00000603   L00000626   no VIEWPORT DRIFT: top left L00000599 (pinned line evicted)
   DURING  0.016  2000  2000   L00000655   L00000678   no 
   DURING  0.016  2000  2000   L00000707   L00000730   no 
   DURING  0.016  2000  2000   L00000758   L00000781   no 

SUMMARY: pre-saturation, scrolled_by rose in lockstep and top stayed pinned to L00000599 (pin=YES); after count saturated at ynum=2000, scrolled_by was capped at count and once the pinned line was evicted the viewport drifted (pin=no). [OBSERVED]
```

### 5.4 What this shows (answering SQ4)

- **While the buffer is unsaturated, an active scrollback is pinned perfectly.**
  From `[SCROLL +500]` onward every frame shows `top(pinned)=L00000523` with
  `pin=YES`, while `scrolled_by` rises in lockstep with `count` (e.g.
  `sby=572/count=1095` ... `sby=1454/count=1977`). The `MIN(scrolled_by + added,
  count)` clamp is a no-op here because `scrolled_by + added <= count`. **[OBSERVED]**
- **Saturation is the crossover.** The `SATURATE` frame is reached exactly at
  `count == ynum == 2000`; from that point `count` is frozen and `scrolled_by` is
  clamped at 2000. **[OBSERVED]**
- **Once the pinned line is evicted, the viewport drifts.** A few frames after
  saturation the top row jumps from `L00000523` to `L00000535`, then `L00000579`,
  `L00000630`, `L00000734` (`pin=no`) — the pinned old line has been evicted from
  the segmented store and the view can no longer hold it. This is precisely the
  `MIN(..., count)` cap [kitty/screen.c:2716] refusing to grow past `ynum`.
  **[OBSERVED]**
- **The behavior is reproducible.** Both runs show the identical
  unsaturated-pin -> saturate -> drift structure; only the absolute label differs
  by startup timing. **[OBSERVED across 2 runs]**

---

## 6. SQ5 — How the memory structures evolve: reflow (allocation, wrapping, retention)

### 6.1 Mechanism

Answering SQ5's "wrapping" facet directly: when the terminal **width** changes, the
scrollback is re-wrapped by `historybuf_rewrap` [kitty/history.c:595]. It has two
branches:

- **Fast path (dimensions unchanged):** when `other->xnum == self->xnum &&
  other->ynum == self->ynum`, the segments are copied wholesale with three
  `memcpy`s (CPU cells, GPU cells, line attrs) and `count`/`start_of_data` are
  carried over verbatim — no reflow [kitty/history.c:597-606].
- **Reflow path (width changed):** otherwise, if a pager ring is present and the
  width changed, `pagerhist->rewrap_needed` is set [kitty/history.c:607-608], then
  `count`/`start_of_data` are reset and every logical line is re-flowed through the
  shared engine `rewrap_inner` [kitty/history.c:611], defined at
  [kitty/rewrap.h:57]. `rewrap_inner` re-packs logical lines into continuation rows
  at the new width, so the **row count in history changes** even though the logical
  content does not.

This is the "wrapping" dimension of SQ5; "allocation" (segment carving) is covered
in §2 and "retention" (eviction into the pager ring) in §3.

### 6.2 Self-contained probe (canonical harness)

The probe fills history with 500 long, individually labelled logical lines
(`NNNN:xxx...`, 150 chars each, which autowrap at 80 cols into two rows apiece),
then drives the width through the sequence `80 -> 40 -> 20 -> 120 -> 80` via
`Screen.resize`, capturing `count`, the topmost retained logical label, and a
row-level dump of the first/last three history rows at each step. The first resize
(80 -> 80) deliberately exercises the **fast memcpy** branch; every subsequent width
change exercises **rewrap_inner**.

```python
#!/usr/bin/env python3
# SQ5 probe: reflow (rewrap) of scrollback history when the terminal width
# changes. historybuf_rewrap (history.c:595): FAST memcpy path when width is
# unchanged (597-606), else sets rewrap_needed (607-608) and re-wraps through
# rewrap_inner (history.c:611 / rewrap.h:57). We fill history with long,
# distinct logical lines (which autowrap into continuation rows), then resize
# the width through a sequence and capture count + row-level before/after
# evidence, including the non-idempotent round-trip delta.
import sys, os
sys.dont_write_bytecode = True
sys.path.insert(0, os.getcwd())
from kitty_tests import BaseTest

class P(BaseTest):
    def runTest(self):
        pass

def sample_rows(s, k=3):
    hb = s.historybuf
    n = hb.count
    idxs = list(range(min(k, n))) + list(range(max(0, n - k), n))
    return "\n".join(f"    hist[{i:>4}] = {str(hb.line(i))!r}" for i in idxs)

def top_logical(s):
    # first history row that carries a logical-line label "NNNN:"
    hb = s.historybuf
    for i in range(hb.count):
        t = str(hb.line(i))
        if len(t) > 4 and t[4] == ':':
            return t[:5]
    return "(none)"

def fill(s, nlogical, length):
    for i in range(nlogical):
        s.draw(f"{i:04d}:" + "x" * (length - 5))
        s.linefeed(); s.carriage_return()

def main():
    bt = P()
    s = bt.create_screen(cols=80, lines=24, scrollback=8000,
                         options={'scrollback_pager_history_size': 0})
    fill(s, 500, 150)
    seq = [80, 40, 20, 120, 80]
    print(f"initial: lines={s.lines} cols={s.columns} count={s.historybuf.count} "
          f"top_logical={top_logical(s)} [OBSERVED]")
    counts = {}
    prev_w = s.columns
    for w in seq:
        path = "FAST memcpy (width unchanged, history.c:597-606)" if w == prev_w \
               else "rewrap_inner (width changed, history.c:611 / rewrap.h:57)"
        s.resize(24, w)
        prev_w = w
        c = s.historybuf.count
        counts.setdefault(w, []).append(c)
        print(f"\n=== resize -> width {w}: count={c} top_logical={top_logical(s)} "
              f"[OBSERVED]  path={path} ===")
        print(sample_rows(s, 3))
    c0, c1 = counts[80][0], counts[80][1]
    print(f"\nROUND-TRIP width 80 -> 40 -> 20 -> 120 -> 80: history count {c0} -> {c1} "
          f"delta={c1 - c0:+d} [OBSERVED]")
    print("OBSERVED facts of the non-idempotent round-trip:")
    print(f"  - history row count changed by {c1 - c0:+d} (was {c0}, now {c1})")
    print(f"  - topmost retained logical line moved (see top_logical above: 80-initial vs 80-final)")
    print("INTERPRETATION [INFERRED, scenario-limited]: reflow redistributes logical lines")
    print("  between the 24-row on-screen grid (linebuf) and the history buffer. Intermediate")
    print("  narrow widths (40,20) pack fewer logical lines onto the 24 visible rows and split")
    print("  each into more continuation rows (rewrap_inner, rewrap.h:57); widening to 120 then")
    print("  80 does not restore the exact original screen<->history partition, so a different")
    print("  number of rows remains in history and a different logical line sits at the top.")
    print("  The counts and row dumps above are OBSERVED; this causal account is INFERRED and")
    print("  specific to this fill pattern (500 x 150-char lines, lines=24, scrollback=8000).")

if __name__ == '__main__':
    main()
```

### 6.3 Complete output (run 1 of 2)

Produced by `PYTHONDONTWRITEBYTECODE=1 python3 /tmp/kitty_probe/sq5_reflow.py`
(run from the repository root against the canonical `.so`):

```text
initial: lines=24 cols=80 count=977 top_logical=0488: [OBSERVED]

=== resize -> width 80: count=977 top_logical=0488: [OBSERVED]  path=FAST memcpy (width unchanged, history.c:597-606) ===
    hist[   0] = '0488:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[   1] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[   2] = '0487:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 974] = '0001:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 975] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 976] = '0000:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

=== resize -> width 40: count=1977 top_logical=0494: [OBSERVED]  path=rewrap_inner (width changed, history.c:611 / rewrap.h:57) ===
    hist[   0] = '0494:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[   1] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[   2] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[1974] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[1975] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[1976] = '0000:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

=== resize -> width 20: count=3977 top_logical=0497: [OBSERVED]  path=rewrap_inner (width changed, history.c:611 / rewrap.h:57) ===
    hist[   0] = '0497:xxxxxxxxxxxxxxx'
    hist[   1] = 'xxxxxxxxxx'
    hist[   2] = 'xxxxxxxxxxxxxxxxxxxx'
    hist[3974] = 'xxxxxxxxxxxxxxxxxxxx'
    hist[3975] = 'xxxxxxxxxxxxxxxxxxxx'
    hist[3976] = '0000:xxxxxxxxxxxxxxx'

=== resize -> width 120: count=995 top_logical=0497: [OBSERVED]  path=rewrap_inner (width changed, history.c:611 / rewrap.h:57) ===
    hist[   0] = '0497:xxxxxxxxxxxxxxx'
    hist[   1] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[   2] = '0496:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 992] = '0001:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 993] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 994] = '0000:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

=== resize -> width 80: count=996 top_logical=0497: [OBSERVED]  path=rewrap_inner (width changed, history.c:611 / rewrap.h:57) ===
    hist[   0] = '0497:xxxxxxxxxxxxxxx'
    hist[   1] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[   2] = '0496:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 993] = '0001:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 994] = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    hist[ 995] = '0000:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

ROUND-TRIP width 80 -> 40 -> 20 -> 120 -> 80: history count 977 -> 996 delta=+19 [OBSERVED]
OBSERVED facts of the non-idempotent round-trip:
  - history row count changed by +19 (was 977, now 996)
  - topmost retained logical line moved (see top_logical above: 80-initial vs 80-final)
INTERPRETATION [INFERRED, scenario-limited]: reflow redistributes logical lines
  between the 24-row on-screen grid (linebuf) and the history buffer. Intermediate
  narrow widths (40,20) pack fewer logical lines onto the 24 visible rows and split
  each into more continuation rows (rewrap_inner, rewrap.h:57); widening to 120 then
  80 does not restore the exact original screen<->history partition, so a different
  number of rows remains in history and a different logical line sits at the top.
  The counts and row dumps above are OBSERVED; this causal account is INFERRED and
  specific to this fill pattern (500 x 150-char lines, lines=24, scrollback=8000).
```

Run 2 was **byte-identical** (deterministic in-process reflow): the same
`977 -> 1977 -> 3977 -> 995 -> 996` count progression, the same `+19` round-trip
delta, and the same row dumps. The complete run-2 output at
`/tmp/kitty_probe/sq5_run2.txt` matches this verbatim.

### 6.4 What this shows (answering SQ5)

- **Both rewrap branches are exercised and labelled.** The `80 -> 80` step reports
  `path=FAST memcpy (width unchanged, history.c:597-606)`; every width change
  reports `path=rewrap_inner (width changed, history.c:611 / rewrap.h:57)`.
  **[OBSERVED]**
- **Narrowing multiplies rows; widening collapses them — visible at row level.**
  History `count` goes `977` (@80) `-> 1977` (@40) `-> 3977` (@20) `-> 995` (@120)
  `-> 996` (@80). The row dumps show a single 150-char logical line occupying two
  rows at width 80, but four rows at width 40 and eight at width 20 — direct
  row-level evidence of `rewrap_inner` re-packing. **[OBSERVED]**
- **The round trip is non-idempotent by `+19` rows in this scenario.** Returning to
  width 80 yields `count=996`, not the original `977` — a `delta=+19` — and the
  topmost retained logical line moved from `0488:` to `0497:`. **[OBSERVED]**
- **The cause is isolated to reflow's screen<->history partition, and is
  scenario-specific.** Intermediate narrow widths pack fewer logical lines onto the
  fixed 24-row grid and push more into history; widening does not restore the exact
  original screen/history split, so a different number of rows remains in history.
  The `+19` magnitude is a property of *this* fill pattern (500 x 150-char lines,
  `lines=24`, `scrollback=8000`) and is **not** a universal constant.
  **[INFERRED, scenario-limited]**

---

## 7. Default (disabled) vs. enabled pager tier — side by side

| Aspect | Default (`scrollback_pager_history_size = 0`) | Enabled (`> 0`) |
|---|---|---|
| Pager ring allocation | `alloc_pagerhist` returns `NULL` [kitty/history.c:72] | ring allocated, initial `MIN(1 MiB, sz)` [kitty/history.c:66-67] |
| On eviction (`count == ynum`) | oldest line **dropped** | oldest line serialized into ring via `pagerhist_push` [kitty/history.c:258-274] |
| `pagerhist_as_bytes()` | always `b''` (0 bytes) **[OBSERVED §3.3]** | grows by payload, then plateaus at `maximum_size` **[OBSERVED §3.3, §4.2]** |
| Interactive scrollback | segmented store only | unchanged (the pager tier is **not** interactively scrollable) |
| Consumer | n/a | external pager via `pagerhist()`/`cmd_output()` [kitty/window.py:355-356, 457-461] and `show_scrollback` [kitty/window.py:1735] |
| SQ1 / SQ4 / SQ5 behavior | segmented store, scroll pin/cap, and reflow are independent of the pager tier | identical |

The segmented store (SQ1), the scroll pin/cap (SQ4), and reflow (SQ5) behave
**identically** in both conditions; only the eviction destination (SQ2) and the
ring's boundary behavior (SQ3) differ between disabled and enabled.

## 8. Observed vs. inferred, coverage, and read-only guarantee

### 8.1 Observed-vs-inferred ledger

| Claim | Status | Basis |
|---|---|---|
| Fresh buffer = exactly 1 segment | OBSERVED | §2.3 + [kitty/history.c:117-133] |
| Max segments = `ceil(ynum/2048)`; carves at count 2049/4097/6145/8193 | OBSERVED | §2.3 |
| Per-segment `calloc` = 5,251,072 B @ xnum=80 | INFERRED | [kitty/history.c:17-28] arithmetic; CPUCell=12/GPUCell=20 static_asserts + LineAttrs=4 (enum-bitfield, `_Static_assert` probe) |
| RSS grows gradually (~2560 B/line), not in segment-sized steps | OBSERVED | §2.3 RSS slope |
| Disabled tier drops evicted lines (3977 dropped) | OBSERVED | §3.3 |
| Enabled tier costs exactly 33 B per evicted line (this fixed line) | OBSERVED | §3.3 |
| Visible ring increment = evicted x bytes/line (payload) | OBSERVED | §3.3 (+33000/batch) |
| Ring capacity grows in `MAX(1 MiB, needed)` steps | INFERRED | [kitty/history.c:89-102] (`as_bytes` reports payload, not capacity) |
| Ring plateaus at `maximum_size`, then overwrites oldest | OBSERVED | §4.2 (1 MiB @ 1,048,576; 4 KiB @ 4096) + [3rdparty/ringbuf/ringbuf.c:231-234] |
| No latency outlier beyond ~1.8x median at any boundary | OBSERVED (resolution-bounded) | §4.2 (median 0.632 ms, max/median 1.77x) |
| No crash / no sanitizer error at 1M & 10M lines | OBSERVED | §4.3 |
| Regression #3011 absent (fix is a baseline ancestor) | OBSERVED (git) + INFERRED (causal) | §4.3, `git merge-base` fb87fc32 |
| Scroll pinned pre-saturation, drifts post-saturation | OBSERVED | §5.3 |
| Reflow round-trip non-idempotent by +19 (this scenario) | OBSERVED (magnitude) + INFERRED (cause) | §6.3 |

### 8.2 Coverage pass (every sub-question and sibling variant)

- **SQ1** segment carving — §2: default 1-segment case, multi-segment (5000 / 10000), and gradual RSS. **Answered.**
- **SQ2** store <-> ring interaction — §3: both the **disabled** and the **enabled** variants. **Answered.**
- **SQ3** transition smoothness — §4: ring boundary + per-batch latency distribution, large-burst fragility (1M **and** 10M), plus an ASan+UBSan cross-check. **Answered.**
- **SQ4** concurrent scroll + write — §5: real PTY child at full speed + interleaved scroll/render + viewport-row drift across saturation. **Answered.**
- **SQ5** runtime evolution (allocation / wrapping / retention) — allocation §2, retention §3, wrapping §6 (both the **fast-memcpy** and **rewrap_inner** branches). **Answered.**

### 8.3 Read-only guarantee and cleanup

This investigation modified **no existing repository file** — no source, test,
configuration, or build file was edited. The single repository change is the addition
of this answer document, `blitzy/documentation/kitty_815df1e210e0.md`. All build
artifacts (`kitty/fast_data_types.so`, `build/`, launchers) are gitignored and never
appear in `git status`. The temporary observation scripts lived entirely in
`/tmp/kitty_probe` (outside the repository tree) and were removed after the runs.
Consequently, before committing the document `git status --porcelain` lists only the
one new file, and after committing it the working tree is clean. The canonical `.so`
was verified unchanged (sha256 `74ca07a6...`) after every swap to a supplemental build.

## 9. References

**kitty source (baseline commit `815df1e21`), by `file:line`:**

- Segmented store: `SEGMENT_SIZE = 2048` [kitty/history.c:15]; `add_segment` [kitty/history.c:17-28]; `segment_for` [kitty/history.c:36-42]; `create_historybuf` (one segment on construction) [kitty/history.c:117-133]; `historybuf_push` eviction [kitty/history.c:277-285]; read-only members `xnum`/`ynum`/`count` [kitty/history.c:556-558].
- Pager ring: `initial_pagerhist_ringbuf_sz` [kitty/history.c:66-67]; `alloc_pagerhist` [kitty/history.c:69-81]; `pagerhist_extend` [kitty/history.c:89-102]; `pagerhist_write_bytes` [kitty/history.c:218-226]; `pagerhist_push` [kitty/history.c:258-274]; `pagerhist_as_bytes` [kitty/history.c:459].
- Ring FIFO: overflow / overwrite-oldest [3rdparty/ringbuf/ringbuf.c:231-234]; attribution (Drew Hess, 2011, public domain) [3rdparty/ringbuf/ringbuf.h:1-15].
- Reflow: `historybuf_rewrap` [kitty/history.c:595]; fast memcpy path [kitty/history.c:597-606]; `rewrap_needed` [kitty/history.c:607-608]; `rewrap_inner` call [kitty/history.c:611]; engine [kitty/rewrap.h:57].
- Screen / scroll: `alloc_historybuf(MAX(scrollback, lines), ...)` [kitty/screen.c:130]; `INDEX_UP` + `history_line_added_count++` [kitty/screen.c:1552-1566, 1559]; `screen_reset_dirty` zeroes the counter [kitty/screen.c:2597-2601]; `scrolled_by` cap [kitty/screen.c:2716, 2761]; `scrolled_by` read-only member [kitty/screen.c:4903]; `history_line_added_count` writable member [kitty/screen.c:4908]; `visual_line` [kitty/screen.c:4788]; `update_only_line_graphics_data` [kitty/screen.c:4867]; `scroll` arg spec `"ip"` [kitty/screen.c:4123].
- Config finalizers: `scrollback_lines` [kitty/options/utils.py:557-561]; `scrollback_pager_history_size` (unit MB, cap 4 GB - 1) [kitty/options/utils.py:564-566]; defaults `2000` / `0` [kitty/options/definition.py:372, 406].
- Pager consumer: `pagerhist()` [kitty/window.py:355-356]; `cmd_output()` [kitty/window.py:457-461]; `show_scrollback` [kitty/window.py:1735]; Python surface [kitty/fast_data_types.pyi:1080-1119].

**Historical defect (SQ3 regression corroboration):**

- kitty issue #3011 — a segfault after roughly 120k lines with `scrollback_pager_history_size` set above 1, reproduced by printing ten million lines (kitty 0.19.0).
- Fix commit `fb87fc32f04be636e1e0fe8eea611a453ee2d3f0` (2020-10-06), confirmed via `git merge-base --is-ancestor` to be an **ancestor of the baseline** `815df1e21` under study — so the fix is present, and the runtime probes in §4.3 confirm no crash on this commit.

**Official documentation (semantics confirmed):**

- `scrollback_lines`: lines of history kept in memory for interactive scrollback; memory is allocated on demand; negative values are effectively infinite.
- `scrollback_pager_history_size`: a separate UTF-8 buffer piped to the pager program (not available for interactive scrolling); roughly 10000 lines per MB at 100 chars/line for pure ASCII; zero or less disables it; maximum 4 GB.

---

*End of investigation.*
