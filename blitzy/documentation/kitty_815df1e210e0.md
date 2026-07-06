# kitty scrollback `HistoryBuf` under heavy load — an empirical investigation

**Repository:** `kovidgoyal/kitty` · **Branch:** `kitty_815df1e210e0` · **HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

This document answers three questions about how kitty's scrollback history buffer (`HistoryBuf`) behaves when a program prints a very large amount of output. **Every number below was produced by building kitty's C extension and driving its real output-ingestion path (`parse_bytes → Screen → historybuf_add_line`) with temporary scripts, then reading process memory from `/proc/self/status` and the buffer's own `HistoryBuf.count`.** The scripts lived in `/tmp` (outside the repository) and were deleted after use; no tracked file in the repository is modified.

---

## TL;DR (direct answers)

- **Q1 — Memory under load.** At the **default** configuration (`scrollback_lines 2000`) memory is **bounded**: after the buffer fills, resident memory attributable to scrollback **plateaus at ≈5.5 MB and never grows again**, no matter whether you print 5,000 or 500,000 lines. The line count `HistoryBuf.count` climbs to `ynum = 2000` and then stays there because the buffer is a **ring** that overwrites its oldest line (`historybuf_push`, [`kitty/history.c:275-283`](#cit-history)). Only when scrollback is configured very large (or "infinite") does memory grow without bound — e.g. **≈123 MiB after 50,000 lines** with infinite scrollback.
- **Q2 — Responsiveness during concurrent scroll + output.** The model/output path is **not** the bottleneck: ingesting 10,000 lines takes **≈10 ms** (~1 ms per 1,000 lines), and a scroll command is an **O(1) integer update** to `scrolled_by` (`screen_history_scroll`, [`kitty/screen.c:4091`](#cit-screen)). Responsiveness under concurrent output is preserved by kitty's **threaded architecture** (input parsing and frame rendering run on separate threads, paced by `repaint_delay`/`input_delay`/`sync_to_monitor`) and by a **view re-anchor** that keeps the scrolled-back viewport pinned to the same content as new lines arrive ([`kitty/screen.c:2761`](#cit-screen)). One part of this — the live re-anchor — is a **render-thread** action and therefore cannot be timed in a headless model-only harness; that limitation is reported honestly below and explained from the source.
- **Q3 — Buffer boundaries / when new storage is allocated.** New backing storage is allocated **one ≈5 MiB segment at a time**, at each **2,048-line boundary**, by `add_segment` via `segment_for` ([`kitty/history.c:17-42`](#cit-history)). **Yes, this is observable through memory monitoring:** each allocation appears as a discrete **≈5 MiB step in *virtual* memory (`VmSize`/`VmData`)** exactly when `count` crosses a multiple of `SEGMENT_SIZE = 2048`, while **resident memory (`VmRSS`) rises approximately linearly** because Linux commits the freshly `calloc`'d pages lazily (demand paging).

---

## Environment & methodology

### Environment (as actually used for these measurements)

| Item | Value |
|------|-------|
| Repository path (shell) | `/tmp/blitzy/kitty/blitzy-11eb97ad-bc12-4d85-992e-5f365a22cf0b_dbec52` |
| Branch / HEAD commit | `blitzy-11eb97ad-bc12-4d85-992e-5f365a22cf0b` / `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Python | CPython **3.13.7** |
| C compiler | **gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0** |
| Screen geometry for all runs | **cols = 80, lines = 24** |
| Built C extension | `kitty/fast_data_types.so` (1,253,792 bytes) |

> **Note on portability of the numbers.** The scrollback mechanism under investigation lives entirely in the compiled C extension and is independent of the CPython version (the repository's `requires-python` is `>= 3.8`). The `HistoryBuf.count` series is deterministic and identical across runs; absolute `VmRSS`/`VmSize` baselines have small run-to-run variance from the OS allocator, which is called out where it occurs.

### Canonical build

kitty was built with the canonical command (the `build()` routine in `setup.py` compiles the `kitty/fast_data_types` C extension first, [`setup.py:L1084`](#cit-setup)):

```
python3 setup.py build --verbose
```

The build produces the `kitty/fast_data_types.so` extension used below. Build artifacts (`*.so`, `/build/`) are git-ignored ([`.gitignore:1`](#cit-gitignore) `*.so`, `.gitignore:14` `/build/`), so they never appear as repository changes.

**Build/environment verification (command + complete output):**

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ git status --porcelain
(empty output above = clean tree)

$ ls -l kitty/fast_data_types.so
-rwxr-xr-x 1 root root 1253792 Jul  6 22:07 kitty/fast_data_types.so

$ python3 --version
Python 3.13.7

$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ PYTHONPATH=. python3 -c "from kitty.fast_data_types import Screen, HistoryBuf; print('ok')"
ok
```

### The real entry point (and what is *not* canonical)

All values below come from the **canonical output-ingestion path**, driven headlessly through kitty's own test harness:

```
parse_bytes(screen, data)                     # kitty_tests/__init__.py:30
  → screen.test_create_write_buffer()         # kitty/screen.c:4755
  → screen.test_commit_write_buffer(...)      # kitty/screen.c:4762
  → screen.test_parse_written_data(...)       # kitty/screen.c:4772  (runs the VT parser)
     → on LF/VT/FF: screen_linefeed()         # kitty/vt-parser.c:101 → kitty/screen.c:1643
        → screen_index()                      # kitty/screen.c:1569
           → INDEX_UP(add_to_history)         # kitty/screen.c:1552-1567
              → historybuf_add_line()         # kitty/history.c:286-291
                 → historybuf_push()          # kitty/history.c:275-283
                    → segment_for()/add_segment()  # kitty/history.c:36-42, 17-29
```

`parse_bytes` feeds raw bytes through the exact VT state machine that real child output traverses ([`kitty_tests/__init__.py:30`](#cit-harness)). This is the same path the user's "printing hundreds of thousands of lines" exercises.

The unit-test helper `test_historybuf`, which constructs `HistoryBuf(5, 5)` directly and calls `.push()` ([`kitty_tests/datatypes.py:487-490`](#cit-datatypes)), is **NON-CANONICAL** for these questions — it bypasses the parser/`Screen` and pokes the buffer object directly. It is not used to produce any answer here.

### What the buffer exposes to Python (and why Q3 needs memory monitoring)

`HistoryBuf` exposes only three members to Python, all READONLY ([`kitty/history.c:556-558`](#cit-history)):

```c
    {"xnum", T_UINT, offsetof(HistoryBuf, xnum), READONLY, "xnum"},
    {"ynum", T_UINT, offsetof(HistoryBuf, ynum), READONLY, "ynum"},
    {"count", T_UINT, offsetof(HistoryBuf, count), READONLY, "count"},
```

The internal `num_segments` field is a C struct member ([`kitty/data-types.h:285`](#cit-datatypes-h)) that is **not** exposed to Python. Segment allocation therefore cannot be read directly; it must be **inferred via memory monitoring** — which is exactly what the user asked ("can I observe this happening through memory monitoring?"). We read `/proc/self/status` (`VmRSS`, `VmSize`, `VmData`) alongside the deterministic `count`.

### Harness detail that affects the "default" measurement

kitty's `BaseTest.set_options` defaults `scrollback_pager_history_size` to **1024** ([`kitty_tests/__init__.py:224`](#cit-harness)), which is **not** kitty's shipped default. To measure the true canonical default (**pager history OFF**), every script below installs options via a local `set_opts` helper that forces `scrollback_pager_history_size: 0` unless a test explicitly overrides it. Input is fed in modest chunks (1,000–2,000 lines per `parse_bytes` call) and each configuration runs in a **fresh process** to avoid transient-input and baseline-carryover artifacts.

---

## The data structure — why the numbers look the way they do

`HistoryBuf` is a **segmented ring buffer**. Understanding two constants and one function explains every measurement.

**1. Segment size.** Lines are stored in fixed blocks of `SEGMENT_SIZE = 2048` lines ([`kitty/history.c:15`](#cit-history)):

```c
#define SEGMENT_SIZE 2048
```

**2. Per-segment allocation.** Each segment is a **single `calloc`** covering that block's CPU cells, GPU cells, and per-line attributes ([`kitty/history.c:17-29`](#cit-history)):

```c
static void
add_segment(HistoryBuf *self) {
    self->num_segments += 1;
    self->segments = realloc(self->segments, sizeof(HistoryBufSegment) * self->num_segments);
    if (self->segments == NULL) fatal("Out of memory allocating new history buffer segment");
    HistoryBufSegment *s = self->segments + self->num_segments - 1;
    const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
    const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
    s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));
    if (!s->cpu_cells) fatal("Out of memory allocating new history buffer segment");
    s->gpu_cells = (GPUCell*)(((uint8_t*)s->cpu_cells) + cpu_cells_size);
    s->line_attrs = (LineAttrs*)(((uint8_t*)s->gpu_cells) + gpu_cells_size);
}
```

The cell sizes are fixed by `static_assert`s: `sizeof(GPUCell) == 20` ([`kitty/data-types.h:221`](#cit-datatypes-h)), `sizeof(CPUCell) == 12` ([`kitty/data-types.h:228`](#cit-datatypes-h)), and `LineAttrs` is a `union` whose storage is a single `uint8_t val` → `sizeof(LineAttrs) == 1` ([`kitty/data-types.h:230-239`](#cit-datatypes-h)). Hence one segment costs:

```
per_segment = SEGMENT_SIZE * (xnum*sizeof(CPUCell) + xnum*sizeof(GPUCell) + sizeof(LineAttrs))
            = 2048 * (xnum*12 + xnum*20 + 1)
            = 2048 * (xnum*32 + 1)

at xnum = 80:  2048 * (80*32 + 1) = 2048 * 2561 = 5,244,928 bytes = 5.0020 MiB  (= 5122 kB)
```

This exact figure (5,244,928 B) is what appears at runtime as the ≈5 MiB allocation step in Q3 (observed as **5132 kB**, i.e. the 5122 kB theoretical value plus ~10 kB of allocator/mmap rounding).

**3. Capacity and the ring.** The number of history rows is `ynum = MAX(scrollback, lines)`, set when the `Screen` allocates its buffer ([`kitty/screen.c:130`](#cit-screen)):

```c
self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));
```

The `scrollback` value comes from configuration. The default is `scrollback_lines '2000'` ([`kitty/options/definition.py:372`](#cit-definition)); a **negative** value is mapped to `2**32 - 1` (effectively infinite) by the parser ([`kitty/options/utils.py:557-560`](#cit-utils)):

```python
def scrollback_lines(x: str) -> int:
    ans = int(x)
    if ans < 0:
        ans = 2 ** 32 - 1
    return ans
```

Because the default `scrollback_lines = 2000` is **less than** `SEGMENT_SIZE = 2048`, `ynum = 2000` fits inside the **single** segment allocated eagerly at construction ([`kitty/history.c:127`](#cit-history)), so no further segments are ever needed at the default — the origin of the Q1 plateau. The pager-history overflow sink is a separate structure, default `scrollback_pager_history_size '0'` ([`kitty/options/definition.py:406`](#cit-definition)), parsed by [`kitty/options/utils.py:564`](#cit-utils).

---

## Q1 — What happens to memory as history accumulates?

> *"If I generate a massive amount of terminal output, say, printing hundreds of thousands of lines rapidly, what happens to memory consumption as the history accumulates? I'd like to see actual memory measurements, not just understand the theory."*

### Direct answer

At the **default** `scrollback_lines 2000`, memory consumption is **bounded and plateaus**. As lines stream in, `HistoryBuf.count` rises to `ynum = 2000` and **stops**; resident memory attributable to scrollback rises to **≈5.5 MB and then stays flat** — feeding 5,000, 50,000, 200,000, or 500,000 lines yields the **same** plateau. The buffer is a fixed-capacity **ring**: once full, each new line **overwrites the oldest** rather than allocating more memory.

### Command and complete, unedited output (two runs)

The observation script feeds up to 500,000 newline-terminated lines through the real `parse_bytes` path at the default scrollback, sampling `VmRSS` and `count` at checkpoints:

```
$ cd /tmp/blitzy/kitty/blitzy-11eb97ad-bc12-4d85-992e-5f365a22cf0b_dbec52
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py q1        # run 1
# Q1 default scrollback: ynum=2000 cols=80 lines=24
 lines_fed   count  VmRSS_kB  dRSS_kB
         0       0     26544        0
       977     954     29260     2716
      1977    1954     31776     5232
      2000    1977     31836     5292
      5000    2000     32024     5480
     50000    2000     32028     5484
    200000    2000     32032     5488
    500000    2000     32044     5500
```

```
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py q1        # run 2
# Q1 default scrollback: ynum=2000 cols=80 lines=24
 lines_fed   count  VmRSS_kB  dRSS_kB
         0       0     26548        0
       977     954     29264     2716
      1977    1954     31780     5232
      2000    1977     31840     5292
      5000    2000     32028     5480
     50000    2000     32032     5484
    200000    2000     32036     5488
    500000    2000     32044     5496
```

The **`count` series is identical across both runs** (`0 → 954 → 1954 → 1977 → 2000 → 2000 → 2000 → 2000`) — it is the deterministic primary signal. The `dRSS` plateau matches to within a few kB (`5500` vs `5496` kB at 500,000 lines); the small difference is ordinary run-to-run OS-allocator baseline variance.

### Reading the numbers

- **Plateau at the capacity.** `count` reaches `ynum = 2000` by ~2,023 lines fed and never exceeds it. `dRSS` climbs to ≈5.4–5.5 MB while the single segment fills and then **flattens** — going from 5,000 to 500,000 lines changes `dRSS` by only ~20 kB (noise), not by megabytes. This is the "actual memory measurement" the user asked for: **the curve is a rising ramp that saturates into a flat line.**
- **The `count = lines_fed − 23` offset.** At `lines = 24`, a line only migrates into history when the cursor sits on the bottom row (`lines − 1 = 23`) and a further line-feed scrolls the screen. So the first 23 lines populate the on-screen grid, and `count = max(0, lines_fed − 23)` (capped at `ynum`). This is visible directly: 977 → 954, 1977 → 1954, 2000 → 1977, all `= fed − 23`.
- **One segment only.** Because `ynum = 2000 < SEGMENT_SIZE = 2048`, exactly **one** ≈5 MiB segment is ever allocated (the eager one from construction, [`kitty/history.c:127`](#cit-history)). That is why the default plateau (~5.5 MB) is on the order of a single segment.

### Rationale — the ring overwrite

The plateau is produced by `historybuf_push` ([`kitty/history.c:275-283`](#cit-history)), the function every migrated line passes through:

```c
static index_type
historybuf_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
    index_type idx = (self->start_of_data + self->count) % self->ynum;
    init_line(self, idx, self->line);
    if (self->count == self->ynum) {
        pagerhist_push(self, as_ansi_buf);
        self->start_of_data = (self->start_of_data + 1) % self->ynum;
    } else self->count++;
    return idx;
}
```

While `count < ynum`, `count` increments (the buffer is filling). Once `count == ynum`, `count` **stops growing**; instead `start_of_data` advances modulo `ynum`, so the write index wraps and the **oldest** line is overwritten in place. No allocation occurs on overwrite — hence bounded memory regardless of how many lines are printed.

### The contrast: very large / "infinite" scrollback is *unbounded*

With `scrollback_lines` negative (→ `ynum = 2**32 - 1`), there is **no plateau** — memory grows with the line count:

```
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py infinite   # run 1
# Infinite scrollback: ynum=4294967295
 lines_fed    count  VmRSS_kB  dRSS_kB
      1000      977     29252     2772
      5000     4977     39400    12920
     10000     9977     51932    25452
     25000    24977     89520    63040
     50000    49977    152168   125688
```

```
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py infinite   # run 2
# Infinite scrollback: ynum=4294967295
 lines_fed    count  VmRSS_kB  dRSS_kB
      1000      977     29192     2772
      5000     4977     39340    12920
     10000     9977     51872    25452
     25000    24977     89460    63040
     50000    49977    152108   125688
```

Here `count` grows unbounded (`= lines_fed − 23`) and `dRSS` reaches **≈123 MiB (125,688 kB) at 50,000 lines** — about 25 segments' worth. The `dRSS` series is **identical across both runs**. This is the runtime demonstration of the warning in kitty's own docs, whose `long_text` reads: *"Note that using very large scrollback is not recommended as it can slow down performance of the terminal and also use large amounts of RAM. Instead, consider using scrollback_pager_history_size."* ([`kitty/options/definition.py:372-380`](#cit-definition)). The negative→infinite mapping is [`kitty/options/utils.py:557-560`](#cit-utils).

> **External corroboration.** kitty issue #970 reports that starting an empty kitty with 64k-line scrollback consumes ~50 MB, and filling that buffer adds roughly ~300 MB. That external data point is consistent with the per-segment, stair-step growth measured here (≈5 MiB per 2,048 lines at 80 columns scales to hundreds of MB across 64k lines).


---

## Q2 — Is the terminal responsive when scrolling a huge history during live output?

> *"When I scroll back through a very large history while new output is still being generated, does the terminal remain responsive? What latency or lag can I observe between my scroll input and the display updating? Are there any visible signs of the system prioritizing one operation over another?"*

### Direct answer

**Yes, the terminal stays responsive**, and the reason is structural:

1. **The output-ingest path is cheap.** Feeding 10,000 lines through the real parser takes **≈10 ms** (~1 ms per 1,000 lines) — the model layer is nowhere near a bottleneck even under a heavy stream.
2. **Scrolling is O(1).** A scroll command changes a single integer, `scrolled_by`; the cost is independent of history size (`screen_history_scroll`, [`kitty/screen.c:4091`](#cit-screen)). Scrolling back through 2,000 or 2,000,000 lines is the same cheap operation.
3. **Prioritization is by design: separate threads.** kitty runs input parsing and frame rendering on **separate threads**, with rendering paced by tunables (`repaint_delay`, `input_delay`, `sync_to_monitor`), so a burst of output cannot starve the UI of repaints, and vice-versa.
4. **The scrolled-back view stays pinned** to the content you're reading while new lines stream in, via a re-anchor of `scrolled_by` ([`kitty/screen.c:2761`](#cit-screen)). This particular step runs on the **render** thread, so — see the honest limitation below — it cannot be timed in a model-only headless harness and is explained from the source instead.

### Command and complete, unedited output (two runs)

```
$ PYTHONPATH=. python3 /tmp/obs_q2.py     # run 1
# SCROLL_LINE=-999999 SCROLL_PAGE=-999998 SCROLL_FULL=-999997
ingest 10000 lines: 10.464 ms total, 1.0464 ms per 1000 lines; count=9977
before any scroll: scrolled_by=0
scroll(SCROLL_LINE, up):  scrolled_by=1
scroll(SCROLL_LINE, up):  scrolled_by=2
scroll(SCROLL_PAGE, up):  scrolled_by=25
scroll(SCROLL_PAGE, down):scrolled_by=2
scroll(SCROLL_LINE, down):scrolled_by=1
scroll(SCROLL_FULL, up):  scrolled_by=9977 (count=9977)
scroll(SCROLL_FULL, down):scrolled_by=0

# Re-anchor test: scrolled back to scrolled_by=3, now feed 5000 more lines (NO render pass)
after feeding 5000 lines (headless, no render thread): scrolled_by=3  count=14977
```

```
$ PYTHONPATH=. python3 /tmp/obs_q2.py     # run 2
# SCROLL_LINE=-999999 SCROLL_PAGE=-999998 SCROLL_FULL=-999997
ingest 10000 lines: 10.120 ms total, 1.0120 ms per 1000 lines; count=9977
before any scroll: scrolled_by=0
scroll(SCROLL_LINE, up):  scrolled_by=1
scroll(SCROLL_LINE, up):  scrolled_by=2
scroll(SCROLL_PAGE, up):  scrolled_by=25
scroll(SCROLL_PAGE, down):scrolled_by=2
scroll(SCROLL_LINE, down):scrolled_by=1
scroll(SCROLL_FULL, up):  scrolled_by=9977 (count=9977)
scroll(SCROLL_FULL, down):scrolled_by=0

# Re-anchor test: scrolled back to scrolled_by=3, now feed 5000 more lines (NO render pass)
after feeding 5000 lines (headless, no render thread): scrolled_by=3  count=14977
```

Both runs are identical apart from the wall-clock ingest time (10.464 vs 10.120 ms), which is expected jitter.

### Reading the numbers

**Ingest latency.** 10,000 lines in ~10 ms means the per-line model cost of accepting output and migrating it into history is ~1 µs. The output path is not what makes a terminal feel laggy under a flood.

**Scroll is O(1) and its amounts are fixed by the code.** The real scroll entry point is `screen.scroll(amt, upwards)` — the Python method registered as `MND(scroll, METH_VARARGS)` ([`kitty/screen.c:4851`](#cit-screen)), implemented by `screen_history_scroll` ([`kitty/screen.c:4091`](#cit-screen)), and invoked by the UI in `kitty/window.py` (e.g. `self.screen.scroll(SCROLL_LINE, True)` at [`kitty/window.py:1834`](#cit-window)). The three scroll magnitudes are the `ScrollType` enum values `SCROLL_LINE = -999999`, `SCROLL_PAGE = -999998`, `SCROLL_FULL = -999997` ([`kitty/screen.h:13`](#cit-screen-h)) — which the run prints verbatim — and `screen_history_scroll` maps them:

```c
bool
screen_history_scroll(Screen *self, int amt, bool upwards) {
    switch(amt) {
        case SCROLL_LINE:
            amt = 1;
            break;
        case SCROLL_PAGE:
            amt = self->lines - 1;
            break;
        case SCROLL_FULL:
            amt = self->historybuf->count;
            break;
        default:
            amt = MAX(0, amt);
            break;
    }
    if (!upwards) {
        amt = MIN((unsigned int)amt, self->scrolled_by);
        amt *= -1;
    }
    if (amt == 0) return false;
    unsigned int new_scroll = MIN(self->scrolled_by + amt, self->historybuf->count);
    if (new_scroll != self->scrolled_by) {
        self->scrolled_by = new_scroll;
        dirty_scroll(self);
        return true;
    }
    return false;
}
```

The `default` branch clamps an arbitrary integer amount to `≥ 0`; the downward case clamps the step to the current `scrolled_by` (you cannot scroll below the live screen); and the final assignment `new_scroll = MIN(self->scrolled_by + amt, count)` is the O(1) update that every scroll performs. It touches no per-line data, which is why scroll cost is independent of history size.

The observed `scrolled_by` transitions match exactly:

| Command | Effect on `scrolled_by` | Observed |
|---------|-------------------------|----------|
| `scroll(SCROLL_LINE, up)` | `+1` | `0 → 1 → 2` |
| `scroll(SCROLL_PAGE, up)` | `+ (lines − 1) = +23` | `2 → 25` |
| `scroll(SCROLL_PAGE, down)` | `− 23` | `25 → 2` |
| `scroll(SCROLL_LINE, down)` | `− 1` | `2 → 1` |
| `scroll(SCROLL_FULL, up)` | `→ count` | `1 → 9977` |
| `scroll(SCROLL_FULL, down)` | `→ 0` | `9977 → 0` |

Each is a constant-time integer update — no per-history-line work — which is why scrolling remains snappy regardless of how large the history is.

### Prioritization between output and rendering

Responsiveness under simultaneous scroll + output is preserved because kitty **decouples** the two activities onto separate threads and paces the rendering, governed by three shipped options:

- `repaint_delay '10'` (ms) — the delay between repaints; caps redraw cadence at ~100 FPS ([`kitty/options/definition.py:866`](#cit-definition)).
- `input_delay '3'` (ms) — the delay before processing pending input after activity ([`kitty/options/definition.py:878`](#cit-definition)).
- `sync_to_monitor 'yes'` — synchronize redraws to the monitor's refresh to avoid tearing ([`kitty/options/definition.py:889`](#cit-definition)).

Because output is parsed on one thread and frames are drawn on another under this pacing, a heavy output burst is coalesced into periodic repaints rather than blocking input handling — the "sign of prioritization" the user asks about is precisely this render-cadence throttling, not a stall of one operation waiting on the other.

### Honest headless limitation — the live re-anchor

The one behavior that **cannot** be directly timed in a model-only (headless) harness is the "view stays pinned while output streams" re-anchor. In the run above, after scrolling back to `scrolled_by = 3` and then feeding 5,000 more lines, **`scrolled_by` stays at 3** (it does *not* auto-advance) while `count` climbs to 14,977. That is expected and correct for a headless harness, because the re-anchor lives in the **render** path, `screen_update_cell_data` ([`kitty/screen.c:2738`](#cit-screen)), at [`kitty/screen.c:2761`](#cit-screen):

```c
    if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);
```

This line runs only when a frame is prepared for the GPU — i.e. on the render thread during a repaint. The headless harness has no render thread and never calls `screen_update_cell_data`, so `scrolled_by` is not advanced. In a live terminal, each repaint (paced by `repaint_delay`) advances `scrolled_by` by the number of lines added since the last frame (`history_line_added_count`, incremented in `INDEX_UP` at [`kitty/screen.c:1559`](#cit-screen)), which keeps the content you're viewing pinned in place as new output pushes older lines further back. **This is reported as reasoned-from-source, not as a directly-timed headless measurement.**


---

## Q3 — Where are the buffer boundaries, and when is new storage allocated?

> *"I'm also curious about the boundaries in the buffer system. At what point does the buffer's behavior change as it grows, for example, when does allocation of new storage occur, and can I observe this happening through memory monitoring?"*

### Direct answer

New storage is allocated **one segment at a time**, and a segment is exactly **`SEGMENT_SIZE = 2048` lines** wide. **The behavior changes at each 2,048-line boundary:** whenever the write position crosses into a not-yet-backed 2,048-line block (and total capacity `ynum` has not been reached), `segment_for` calls `add_segment`, which does a single ≈5 MiB `calloc`. **Yes — this is observable through memory monitoring**, and the cleanest signal is *virtual* memory: `VmSize`/`VmData` jump by ≈5 MiB in a discrete **staircase**, one step per 2,048 lines, precisely when `count` crosses `2048, 4096, 6144, 8192, …`. Resident memory (`VmRSS`) rises **approximately linearly** in the same run because the OS commits the freshly-`calloc`'d pages lazily as each line is written.

### Command and complete, unedited output (two runs)

This probe uses a large capacity (`scrollback = 100000`, so many segments are possible) and samples every 256 lines while tracking `VmRSS`, `VmSize`, and `VmData`:

```
$ PYTHONPATH=. python3 /tmp/obs_q3_detail.py    # run 1
# ynum=100000 SEGMENT_SIZE=2048 per-seg~5.0020MiB (5244928 B)
   fed  count  dRSS_kB  dVmSize_kB  dVmData_kB
   256    233      872           0           0
   512    489     1512           0           0
   768    745     2152           0           0
  1024   1001     2792           0           0
  1280   1257     3436           0           0
  1536   1513     4076           0           0
  1792   1769     4716           0           0
  2048   2025     5356           0           0
  2304   2281     6008        5132        5132
  2560   2537     6648        5132        5132
  2816   2793     7288        5132        5132
  3072   3049     7928        5132        5132
  3328   3305     8572        5132        5132
  3584   3561     9212        5132        5132
  3840   3817     9852        5132        5132
  4096   4073    10492        5132        5132
  4352   4329    11140       10264       10264
  4608   4585    11780       10264       10264
  4864   4841    12420       10264       10264
  5120   5097    13060       10264       10264
  5376   5353    13704       10264       10264
  5632   5609    14344       10264       10264
  5888   5865    14984       10264       10264
  6144   6121    15624       10264       10264
  6400   6377    16272       15396       15396
  6656   6633    16912       15396       15396
  6912   6889    17552       15396       15396
  7168   7145    18192       15396       15396
  7424   7401    18836       15396       15396
  7680   7657    19476       15396       15396
  7936   7913    20116       15396       15396
  8192   8169    20760       15396       15396
  8448   8425    21408       20528       20528
  8704   8681    22048       20528       20528
  8960   8937    22688       20528       20528
```

```
$ PYTHONPATH=. python3 /tmp/obs_q3_detail.py    # run 2
# ynum=100000 SEGMENT_SIZE=2048 per-seg~5.0020MiB (5244928 B)
   fed  count  dRSS_kB  dVmSize_kB  dVmData_kB
   256    233      872           0           0
   512    489     1512           0           0
   768    745     2152           0           0
  1024   1001     2792           0           0
  1280   1257     3436           0           0
  1536   1513     4076           0           0
  1792   1769     4716           0           0
  2048   2025     5356           0           0
  2304   2281     6004        5132        5132
  2560   2537     6644        5132        5132
  2816   2793     7284        5132        5132
  3072   3049     7924        5132        5132
  3328   3305     8568        5132        5132
  3584   3561     9208        5132        5132
  3840   3817     9848        5132        5132
  4096   4073    10488        5132        5132
  4352   4329    11136       10264       10264
  4608   4585    11776       10264       10264
  4864   4841    12416       10264       10264
  5120   5097    13056       10264       10264
  5376   5353    13700       10264       10264
  5632   5609    14340       10264       10264
  5888   5865    14980       10264       10264
  6144   6121    15620       10264       10264
  6400   6377    16268       15396       15396
  6656   6633    16908       15396       15396
  6912   6889    17548       15396       15396
  7168   7145    18188       15396       15396
  7424   7401    18832       15396       15396
  7680   7657    19472       15396       15396
  7936   7913    20112       15396       15396
  8192   8169    20756       16420       16420
  8448   8425    21404       21552       21552
  8704   8681    22044       21552       21552
  8960   8937    22684       21552       21552
```

### Reading the numbers — the staircase, honestly

**The virtual-memory staircase (the allocation events).** `dVmSize`/`dVmData` are **0** while `count ≤ 2025` (still inside the first, eagerly-allocated segment). The **first step to `5132 kB`** appears at `fed = 2304` (`count = 2281`) — i.e. right after `count` crossed **2048**. The next steps land at `fed = 4352` (`count` crossed **4096**, → `10264 kB`), `fed = 6400` (`count` crossed **6144**, → `15396 kB`), and `fed = 8448` (`count` crossed **8192**, → `20528 kB`). Each increment is **one segment**:

```
5132 kB ≈ 5,244,928 B (theoretical per-segment) + ~10 kB allocator/mmap rounding
10264 kB = 2 × 5132     15396 kB = 3 × 5132     20528 kB = 4 × 5132
```

That is exactly `add_segment`'s single `calloc` ([`kitty/history.c:25`](#cit-history)) becoming visible. **The boundary is `count` crossing a multiple of `SEGMENT_SIZE = 2048`.**

**The resident-memory ramp (demand paging).** `dRSS` does *not* jump in ≈5 MiB steps; it rises **≈linearly**, about **640 kB per 256 lines ≈ 2.5 kB/line** — which is the per-line touch cost `xnum*32 + 1 = 2561 B` at 80 columns. This is because `calloc` reserves the whole segment's zero-backed pages up front (visible immediately in `VmSize`), but Linux only commits a physical page to `VmRSS` when a line is actually written into it. **Reporting both metrics is the honest picture: the *allocation* is a virtual-memory staircase; the *residency* is a demand-paged ramp.** A naive "RSS staircase" would be inaccurate.

**Run-to-run note.** The `count` series and the ≈5 MiB (`5132 kB`) step size are **identical** across both runs. The absolute `VmSize` *total* shows minor variance in the tail (run 2 shows a small intermediate `16420 kB` reading at `fed = 8192` before settling at `21552 kB`), which is ordinary glibc-arena/`realloc` behavior for the `self->segments` array and does not change the per-segment step size. This is why `HistoryBuf.count` is used as the deterministic primary signal and memory as corroboration.

### Rationale — `segment_for` gates `add_segment`

Allocation is driven by `segment_for`, called for every line write ([`kitty/history.c:36-42`](#cit-history)):

```c
static index_type
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
    if (UNLIKELY(seg_num >= self->num_segments)) fatal("Out of bounds access to history buffer line number: %u", y);
    return seg_num;
}
```

`seg_num = y / SEGMENT_SIZE` is the block index for line `y`. A new segment is `add_segment`'d only when the target block `seg_num` is beyond what is currently allocated **and** the allocated capacity is still below `ynum`. So allocation happens **on demand**, exactly at the 2,048-line boundaries, and stops once `ynum` rows are backed. One segment is allocated eagerly at construction ([`kitty/history.c:127`](#cit-history)), which is why the first 2,048 lines produce no step.

### Explicitly answering the two sub-questions

- **"At what point does the buffer's behavior change / when does allocation occur?"** — At each **2,048-line boundary**: precisely when `count` crosses a multiple of `SEGMENT_SIZE = 2048` while `count < ynum`. (At the default `ynum = 2000 < 2048`, that boundary is never reached, so the default allocates exactly one segment and then only overwrites — see Q1.)
- **"Can I observe this happening through memory monitoring?"** — **Yes.** Since `num_segments` is not exposed to Python, memory monitoring *is* the observation method: each allocation is a discrete **≈5 MiB step in `VmSize`/`VmData`** at the boundary, with `VmRSS` additionally showing the incremental page commit.


---

## Secondary conditions (every distinct case the questions imply)

This section enumerates the configurations and states the questions imply, each with its command and complete output.

### Scrollback size: default vs. large vs. effectively-infinite

| Condition | `ynum` | Behavior as lines accumulate | Evidence |
|-----------|--------|------------------------------|----------|
| **Default** (`scrollback_lines 2000`) | `2000` | Bounded: `count` caps at 2000, `dRSS` plateaus ≈5.5 MB; one segment only | Q1 output above |
| **Large** (`scrollback = 100000`) | `100000` | Segmented growth: ≈5 MiB `VmSize` step per 2,048 lines | Q3 output above |
| **Effectively-infinite** (`scrollback_lines` negative → `2**32-1`) | `4294967295` | Unbounded: `count = lines_fed − 23`, `dRSS` → ≈123 MiB @ 50k lines | Q1 "infinite" output above |

The `ynum` values are printed directly by the scripts (`ynum=2000`, `ynum=100000`, `ynum=4294967295`), confirming `ynum = MAX(scrollback, lines)` ([`kitty/screen.c:130`](#cit-screen)) and the negative→`2**32-1` parse ([`kitty/options/utils.py:557-560`](#cit-utils)).

### Main screen vs. alternate screen — the history gate

The alternate screen (used by full-screen programs like `vim`/`less`) does **not** feed scrollback. Feeding 20,000 lines on the alt screen leaves `count = 0`; switching back to the main screen resumes history migration (capped at `ynum`):

```
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py altscreen
# Alt-screen gate. ynum=2000
main start: count=0
after 20000 lines on ALT screen: count=0
after 20000 lines back on MAIN screen: count=2000
```

**Rationale.** History migration is gated by `add_to_history = self->linebuf == self->main_linebuf && self->margin_top == 0`, computed in `screen_index` ([`kitty/screen.c:1574`](#cit-screen)) and `screen_scroll` ([`kitty/screen.c:1593`](#cit-screen)); only when it is true does `INDEX_UP` call `historybuf_add_line` ([`kitty/screen.c:1558`](#cit-screen)). On the alternate screen `linebuf` points at `alt_linebuf`, so the gate is false and nothing is added — matching the observed `count = 0`.

### Pager history OFF (canonical default) vs. ON

With the shipped default `scrollback_pager_history_size 0`, lines overflowing the ring are **discarded**; with pager history enabled, overflow is **diverted** to a separate byte buffer and remains retrievable. The ring `count` is unaffected either way:

```
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py pager off
# Pager history OFF (0). ynum=2000
after 10000 lines: count=2000 pagerhist_as_text_len=0
```

```
$ PYTHONPATH=. python3 /tmp/obs_scrollback.py pager on
# Pager history ON (10MB). ynum=2000
after 10000 lines: count=2000 pagerhist_as_text_len=119655
```

**Rationale.** In `historybuf_push`, when the ring is full (`count == ynum`) the evicted line is handed to `pagerhist_push` ([`kitty/history.c:280`](#cit-history)) before `start_of_data` advances. With the pager buffer sized 0 there is nowhere to store it (`pagerhist_as_text()` returns 0 chars); with a 10 MB buffer the evicted text accumulates and `pagerhist_as_text()` returns a large positive length (**119,655 characters** in this run — this magnitude depends on line content/length; the salient fact is that it is a large positive number when enabled and exactly 0 when disabled). `count` stays `2000` in both cases because the pager buffer is **separate** from the scrollback ring. The default `'0'` is [`kitty/options/definition.py:406`](#cit-definition); the parser is [`kitty/options/utils.py:564`](#cit-utils).

### Buffer states over time: empty → filling → saturated

The three temporal states the questions imply are all visible in the Q1 default run:

| State | Observation (from Q1 default output) | Meaning |
|-------|--------------------------------------|---------|
| **Empty** | `lines_fed=0 → count=0, dRSS=0` | Buffer allocated (one eager segment) but no history yet |
| **Filling** | `977 → count=954`; `1977 → count=1954` | `count` rising toward `ynum`; `dRSS` ramping to ≈5.2 MB |
| **Saturated** | `5000..500000 → count=2000, dRSS≈5.5 MB flat` | Ring full; oldest line overwritten on each push; memory flat |

The transition from *filling* to *saturated* is precisely the moment `count` reaches `ynum` and `historybuf_push` switches from `count++` to advancing `start_of_data` ([`kitty/history.c:279-282`](#cit-history)).

---

## Read-only scope & reproducibility

- **The repository is unchanged.** The only new file is this document, `blitzy/documentation/kitty_815df1e210e0.md`. No source, test, configuration, or build file was modified, added, or deleted.
- **Temporary scripts lived in `/tmp` and were deleted.** The three observation scripts (`/tmp/obs_scrollback.py`, `/tmp/obs_q3_detail.py`, `/tmp/obs_q2.py`) were created outside the repository and removed after the measurements were captured.
- **Build artifacts are git-ignored.** `kitty/fast_data_types.so` and `/build/` are covered by `.gitignore` (`*.so` at [`.gitignore:1`](#cit-gitignore), `/build/` at `.gitignore:14`), verified with `git check-ignore kitty/fast_data_types.so`. They never appear as repository changes.
- **Clean tree.** `git status --porcelain` produced **empty** output both before and after all runs.
- **Magnitudes were confirmed across ≥2 runs.** The deterministic `HistoryBuf.count` series is the **primary signal** (identical across runs); `VmRSS`/`VmSize` deltas are **corroboration** and their absolute baselines vary slightly run-to-run, which is stated wherever it occurs. Input was fed in modest chunks and each configuration ran in a fresh process to avoid transient-input and baseline-carryover artifacts.

---

## Appendix — citations used

Line numbers are as they appear at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

<a id="cit-history"></a>**`kitty/history.c`**
- `#define SEGMENT_SIZE 2048` — L15
- `add_segment` (per-segment `realloc` + single `calloc`) — L17-29 (`calloc` at L25)
- `segment_for` (on-demand allocation gate) — L36-42
- eager first `add_segment(self)` at construction — L127
- `historybuf_clear` (resets `count`/`start_of_data`, keeps 1 segment) — L209-215
- `pagerhist_push` (overflow diversion) — L259, called from L280
- `historybuf_push` (ring overwrite) — L275-283 (`count == ynum` branch L279-282)
- `historybuf_add_line` — L286-291
- `pagerhist_as_text` — L486 (registered L547)
- Python-exposed READONLY members `xnum`/`ynum`/`count` — L556/L557/L558

<a id="cit-datatypes-h"></a>**`kitty/data-types.h`**
- `static_assert(sizeof(GPUCell) == 20)` — L221
- `static_assert(sizeof(CPUCell) == 12)` — L228
- `LineAttrs` union (`uint8_t val`, 1 byte) — L230-239
- `HistoryBuf` struct with `num_segments` (not exposed to Python) — L285-290

<a id="cit-screen"></a>**`kitty/screen.c`**
- `alloc_historybuf(MAX(scrollback, lines), ...)` — L130
- `INDEX_UP` macro → `historybuf_add_line` (L1558), `history_line_added_count++` (L1559) — L1552-1567
- `screen_index` + `add_to_history` gate — L1569 (gate L1574)
- `screen_scroll` + `add_to_history` gate — L1590 (gate L1593)
- `screen_linefeed` → `screen_index` — L1643
- `scrolled_by` re-anchor in `screen_update_cell_data` (fn L2738) — L2761
- `screen_history_scroll` (SCROLL_LINE/PAGE/FULL mapping) — L4091
- `scroll` Python method → `screen_history_scroll` — L4121 (registered `MND(scroll, METH_VARARGS)` L4851)
- `test_create_write_buffer`/`test_commit_write_buffer`/`test_parse_written_data` — L4755/L4762/L4772

<a id="cit-screen-h"></a>**`kitty/screen.h`**
- `SCROLL_LINE = -999999, SCROLL_PAGE, SCROLL_FULL` (`ScrollType` enum) — L13

<a id="cit-vtparser"></a>**`kitty/vt-parser.c`**
- `case LF: case VT: case FF: REPORT_COMMAND(screen_linefeed)` — L101

<a id="cit-linebuf"></a>**`kitty/line-buf.c`**
- `linebuf_index` (LineBuf row migration) — L317

<a id="cit-definition"></a>**`kitty/options/definition.py`**
- `scrollback_lines '2000'` + RAM-warning `long_text` — L372 (warning text L376-380)
- `scrollback_pager_history_size '0'` — L406
- `repaint_delay '10'` — L866 · `input_delay '3'` — L878 · `sync_to_monitor 'yes'` — L889

<a id="cit-utils"></a>**`kitty/options/utils.py`**
- `scrollback_lines` parser (negative → `2**32-1`) — L557-560
- `scrollback_pager_history_size` parser — L564

<a id="cit-harness"></a>**`kitty_tests/__init__.py`**
- `parse_bytes` (real VT-parser feed) — L30 · `Callbacks` — L39
- `set_options` (harness default pager = 1024, overridden to 0 here) — L223-224 · `create_screen` — L237

<a id="cit-datatypes"></a>**`kitty_tests/datatypes.py`**
- `test_historybuf` — `HistoryBuf(5,5)` + `.push()` — L487-490 **(NON-CANONICAL; not used for any answer here)**

<a id="cit-window"></a>**`kitty/window.py`**
- `self.screen.scroll(SCROLL_LINE, True)` (real UI scroll entry) — L1834

<a id="cit-setup"></a>**`setup.py`**
- `build()` compiles the `kitty/fast_data_types` C extension first — L1084

<a id="cit-gitignore"></a>**`.gitignore`**
- `*.so` — L1 · `/build/` — L14 · `/kitty/launcher/kitt*` — L18

