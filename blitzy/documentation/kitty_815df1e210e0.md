# kitty Scrollback History Buffer Under Heavy Load — Code-Grounded, Measured Analysis

> **Commit pin:** every code locator and every claim in this document reflects kitty at commit
> **`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`**. Line numbers (`file:Lnnn`) are valid at this revision.
>
> **Methodology in one line:** every quantitative claim is *derived from the source* and then *confirmed by
> running the real compiled `HistoryBuf`* (from `kitty/fast_data_types.so`) while sampling process RSS from
> `/proc/self/statm` — code is the source of truth, measurement is the proof.

## The three questions

This document answers three questions about how kitty's scrollback history buffer behaves when an application
prints **hundreds of thousands of lines rapidly**:

1. **(OBJ-1) Memory consumption under massive output.** What happens to memory as scrollback history
   accumulates, and what do *actual* memory measurements show (not just the theory)?
2. **(OBJ-2) Responsiveness & latency under concurrent scroll + live output.** Does the terminal remain
   responsive when you scroll back through a very large history *while new output is still being generated*?
   Is there observable latency between scroll input and the display updating, and are there visible signs of
   the system prioritizing one operation over another?
3. **(OBJ-3) Buffer boundaries & allocation events.** At what point does the buffer's behavior change as it
   grows — when does allocation of new storage occur, and can that be observed through memory monitoring?

## ⚠ The single most important caveat — read this first

kitty's **default `scrollback_lines` is `2000`** (`kitty/options/definition.py:L372`). The scrollback grid is
allocated lazily in **fixed 2048-line segments** (`kitty/history.c:L15`), so a 2000-line cap fits inside a
**single segment**. Under the out-of-the-box configuration, the user's "hundreds of thousands of lines"
therefore produce an **essentially flat memory profile after the first ~2048 lines**: the grid is a fixed-size
ring that simply overwrites its oldest entries. We measured this directly — pushing **500,000** lines into a
default-sized buffer grew RSS by only **≈ 4.887 MiB** (one segment), with the live line `count` capped at 2000.

**Consequence for the investigation:** to *observe* the allocation steps the user asks about, you must
configure a **large or effectively-infinite scrollback**. Negative values map to `2³²−1` lines
(`kitty/options/utils.py:L557-L561`). All of the "growth" measurements below use a large `ynum`
(e.g. `100000`) precisely so the per-segment allocation steps become visible. Without this, naïve measurement
would dramatically understate the behavior. This is the key methodological insight of the whole analysis.

---

## 1. Memory consumption under massive output (OBJ-1)

### 1.1 How scrollback storage is laid out (code)

Scrollback lines are not stored one-allocation-per-line. They are stored in **fixed-size segments of 2048
lines** that are allocated **lazily, on demand**:

- `#define SEGMENT_SIZE 2048` — `kitty/history.c:L15`.
- A segment is allocated by `add_segment()` — `kitty/history.c:L17-L29`. It first grows the array of segment
  descriptors by one (`realloc(... sizeof(HistoryBufSegment) * self->num_segments)`), then performs a
  **single contiguous** `calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs))`
  (`kitty/history.c:L25`), where `cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell)` and
  `gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell)`. The three sub-arrays (GPU cells, CPU cells,
  line attributes) are then carved out of that one block via pointer arithmetic (`kitty/history.c:L27-L28`).
- Each segment descriptor is `HistoryBufSegment { GPUCell *gpu_cells; CPUCell *cpu_cells; LineAttrs *line_attrs; }`
  — `kitty/data-types.h:L262-L266`.

The grid grows one segment at a time. `segment_for()` is the function that maps a logical line index `y` to its
backing segment, and it is the **grow trigger** — `kitty/history.c:L37-L42`:

```c
static HistoryBufSegment*
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum))
        add_segment(self);                                   // kitty/history.c:L39
    return self->segments + seg_num;
}
```

A new segment is created only when a line index crosses into a not-yet-allocated 2048-line block **and** the
total allocated capacity is still below `ynum`. That second clause is what makes the memory *bounded*.

### 1.2 The per-line and per-segment byte cost (derivation)

The cell struct sizes are fixed and enforced by `static_assert` at compile time:

- `static_assert(sizeof(GPUCell) == 20, ...)` — `kitty/data-types.h:L221`.
- `static_assert(sizeof(CPUCell) == 12, ...)` — `kitty/data-types.h:L228`.
- `LineAttrs` is a one-byte union (its largest member is `uint8_t val`) — `kitty/data-types.h:L231-L239`.

A scrollback line stores **one CPUCell and one GPUCell per column**, plus **one byte of line attributes per
line**. Therefore:

```
per-line bytes    = xnum * sizeof(CPUCell) + xnum * sizeof(GPUCell) + sizeof(LineAttrs)
                  = xnum*12 + xnum*20 + 1
                  = xnum*32 + 1

per-segment bytes = SEGMENT_SIZE * (xnum*32 + 1)
                  = 2048 * (xnum*32 + 1)
```

Evaluating the formula:

| Columns (`xnum`) | per-line bytes | per-segment bytes | per-segment (MiB) |
|---|---|---|---|
| 80  | `80*32+1`  = **2561** | `2048*2561`  = **5,244,928**  | **5.002 MiB** |
| 200 | `200*32+1` = **6401** | `2048*6401` = **13,109,248** | **12.502 MiB** |

So each freshly allocated segment adds **~5.0 MiB at 80 columns** (the "~5 MiB step") and **~12.5 MiB at 200
columns**. The total grid capacity is `ynum` lines, where:

- `self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));`
  — `kitty/screen.c:L130`, i.e. **`ynum = MAX(scrollback_lines, screen_lines)`**.

### 1.3 Measured RSS — large scrollback makes the steps visible

Driving the **real** `HistoryBuf(100000, 80, 0)` and pushing 100,000 lines while sampling RSS from
`/proc/self/statm` reproduced the ~5 MiB step at **every** 2048-line boundary. (Measured this session;
Python 3.13.7; `kitty/fast_data_types.so` = 1,253,792 bytes; page size 4096 B.)

| Lines pushed | Measured RSS delta | Live `count` | Notes |
|---|---|---|---|
| 2,048   | **+5.01 MiB**   | 2,048   | first segment fully populated → 1 × 5.002 MiB |
| 4,096   | **+10.02 MiB**  | 4,096   | 2 segments |
| 6,144   | **+15.03 MiB**  | 6,144   | 3 segments |
| 8,192   | **+20.04 MiB**  | 8,192   | 4 segments |
| 10,240  | **+25.05 MiB**  | 10,240  | 5 segments |
| 50,000  | **+122.36 MiB** | 50,000  | ~24.4 segments |
| 99,999  | **+244.71 MiB** | 99,999  | ~48.8 segments |
| 100,000 | (cap reached)   | 100,000 | `count == ynum` — ring saturated |

The measured per-step increment (~5.0 MiB) matches the derived per-segment size (5.002 MiB @ 80 cols) within
RSS sampling noise. **The model is empirically confirmed:** memory grows linearly in the number of *allocated*
segments, in discrete ~5 MiB jumps, up to the `ynum` ceiling.

### 1.4 The default configuration: massive output does *not* balloon grid memory

With the default `scrollback_lines = 2000` (`kitty/options/definition.py:L372`), `ynum = MAX(2000, screen_lines)`
fits in a single segment. Pushing **500,000** lines into `HistoryBuf(2000, 80, 0)` grew RSS by only
**≈ 4.887 MiB total**, and the live `count` stayed pinned at 2000:

| Configuration | Lines pushed | Total RSS growth | Final `count` |
|---|---|---|---|
| `HistoryBuf(2000, 80, 0)` (default) | 500,000 | **≈ 4.887 MiB** (one segment) | 2,000 (capped) |

This is the direct answer to "what happens to memory under hundreds of thousands of lines": **at defaults,
almost nothing after the first segment** — the grid is a fixed ring that overwrites its oldest lines. The
dramatic growth only appears when scrollback is configured large (§1.3), which is why §4 foregrounds this
caveat.

---

## 2. Responsiveness & latency under concurrent scroll + live output (OBJ-2)

> **Framing — this section is code-reasoned.** Wall-clock interaction latency is a property of the *live GUI
> event loop*, which the headless `HistoryBuf` harness does not exercise. Accordingly, the answer here is
> established from the **source mechanisms** that govern responsiveness and is corroborated by the
> **documented option semantics**. It is not a wall-clock micro-benchmark; it is a faithful reading of how the
> code decouples, throttles, prioritizes, and anchors.

The short answer: **yes, kitty stays responsive while scrolling a huge history during live output**, and that
is by design. Four mechanisms cooperate.

### 2.1 Three-thread decoupling — PTY reads never block on rendering

kitty's child monitor runs a **dedicated I/O thread** and a **Talk thread** in addition to the **main thread**:

- The I/O thread is spawned via `pthread_create(&self->io_thread, NULL, io_loop, self)` —
  `kitty/child-monitor.c:L291` (the `io_loop` body is at `kitty/child-monitor.c:L1481`).
- The Talk thread is spawned via `pthread_create(... talk_loop ...)` — `kitty/child-monitor.c:L256` and
  `:L286` (the `talk_loop` body is at `kitty/child-monitor.c:L1805`).
- The **main thread** parses the PTY byte stream and renders.

Because a separate I/O thread reads PTY output, the producer side (the application generating "hundreds of
thousands of lines") is **decoupled** from the consumer side (parse + render). Heavy output keeps draining into
buffers on the I/O thread even while the main thread is busy compositing a frame, so a flood of output does not
stall the read path, and the read path does not stall the UI.

### 2.2 Render throttling — frames are rate-limited to `repaint_delay`

Rendering is not run on every event. `render()` (`kitty/child-monitor.c:L871`) defers when nothing new was read
and not enough time has elapsed since the last paint — `kitty/child-monitor.c:L875-L878`:

```c
if (!input_read && time_since_last_render < OPT(repaint_delay)) {
    set_maximum_wait(OPT(repaint_delay) - time_since_last_render);
    return;
}
```

`repaint_delay` defaults to **10 ms** (`kitty/options/definition.py:L866`). This caps redraw work so that a
torrent of output cannot monopolize the CPU with back-to-back repaints, leaving headroom to service input.

### 2.3 Input/output prioritization — latency is minimized when input is pending

The throttle above is **deliberately bypassed when there is input to handle**. The `repaint_delay`
documentation states that, to minimize latency when there is pending input to be processed, the option is
ignored — `kitty/options/definition.py:L872-L875`. Two related knobs complete the picture:

- `input_delay` default **3 ms** (`kitty/options/definition.py:L878`); its docs note it is ignored when the
  input buffer is almost full (`kitty/options/definition.py:L885`) — i.e. when output is heavy, kitty stops
  waiting and processes immediately.
- `sync_to_monitor` default **`yes`** (`kitty/options/definition.py:L889`) gates rendering on the monitor's
  refresh (vsync) via `#define USE_RENDER_FRAMES (global_state.has_render_frames && OPT(sync_to_monitor))`
  (`kitty/child-monitor.c:L40`). Its own docs note you "may notice some slight input latency"
  (`kitty/options/definition.py:L895-L896`) because scrolling is synced to the refresh rate — an explicit,
  documented trade of a few ms of latency for tear-free scrolling.

Together these are the "visible signs of the system prioritizing one operation over another" the question asks
about: output processing is prioritized (delays ignored) when input/output is pending, while steady-state
repaints are throttled and vsync-aligned.

### 2.4 Scroll-view anchoring — the viewport stays put as new lines arrive

When you have scrolled back and new lines arrive, kitty does **not** snap you to the bottom. The scrolled-back
offset is re-anchored to the same content:

```c
if (self->scrolled_by)
    self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);
```

This appears at **both** `kitty/screen.c:L2716` and `kitty/screen.c:L2761`. As `history_line_added_count` new
lines are pushed, `scrolled_by` is advanced by the same amount (clamped to `historybuf->count`), so the lines
you are looking at remain in view rather than scrolling away under you. This is the mechanism by which
"scrolling remains responsive and stable" during live output — the view is content-anchored, and the clamp to
`historybuf->count` keeps it valid even as the ring saturates and evicts.

### 2.5 Why this answers the latency question

- **Decoupling (§2.1)** ensures the scroll/render path is never blocked behind PTY reads.
- **Throttling (§2.2)** prevents output floods from starving the UI of CPU.
- **Prioritization (§2.3)** explicitly drops artificial delays when there is work pending, minimizing
  input-to-display latency; `sync_to_monitor` trades a documented "slight" latency for tear-free frames.
- **Anchoring (§2.4)** keeps the scrolled-back viewport stable so scrolling feels consistent under live
  output.

These are design-level guarantees readable directly in the source; the documented semantics of `repaint_delay`,
`input_delay`, and `sync_to_monitor` corroborate them.


---

## 3. Buffer boundaries & allocation events (OBJ-3)

The buffer's behavior changes at **three** exact, code-level transition points. Each one is either an
**observable RSS step** or an **observable flattening** of the RSS curve — directly answering "when does
allocation of new storage occur, and can I observe this happening through memory monitoring?"

### 3.1 Boundary #1 — segment allocation at every 2048-line crossing (an RSS step)

Every time a line index crosses into a new, not-yet-allocated 2048-line block (and capacity is still below
`ynum`), `segment_for()` calls `add_segment()` — guard at `kitty/history.c:L39`, allocation at
`kitty/history.c:L17-L29`. That is a **single ~5 MiB `calloc`** (`kitty/history.c:L25`) at 80 columns.

**Observable as:** the discrete **+5.0 MiB RSS steps** in the §1.3 table — at 2,048 (+5.01 MiB), 4,096
(+10.02), 6,144 (+15.03), … 99,999 (+244.71). Each step is exactly one segment. This is the literal "allocation
of new storage" the user asked to watch, and memory monitoring shows it as a clean staircase.

### 3.2 Boundary #2 — ring saturation at `count == ynum` (the curve goes flat)

Pushing a line is `historybuf_push()` — `kitty/history.c:L276-L284`. It computes the target slot
`idx = (self->start_of_data + self->count) % self->ynum` (`kitty/history.c:L277`), initializes that line
(`kitty/history.c:L278`), and then branches (`kitty/history.c:L279-L282`):

```c
if (self->count == self->ynum) {
    pagerhist_push(self, as_ansi_buf);                       // evict oldest into pager history (L280)
    self->start_of_data = (self->start_of_data + 1) % self->ynum;   // advance circularly (L281)
} else self->count++;                                        // still filling (L282)
```

While `count < ynum` the buffer is still filling and crossing 2048 boundaries triggers `add_segment` (the
Boundary 1 case above). But once **`count == ynum`** the buffer is **saturated**: it evicts the oldest line and
advances `start_of_data` **circularly instead of allocating**. Grid growth flattens to **zero**.

**Observable as:** in the §1.3 run, after the cap was reached we pushed **+30,000 more lines** and measured
**+0.000 MiB** further growth, with `count` frozen at 100,000:

| Action | Measured RSS delta | `count` |
|---|---|---|
| push +30,000 lines beyond the 100,000 cap | **+0.000 MiB** | 100,000 (unchanged) |

So the contrast is itself the answer: a **rising staircase while filling**, then a **dead-flat line at
saturation**. Memory monitoring distinguishes the two regimes unambiguously.

### 3.3 Boundary #3 — pager history, a second, independent growth axis (bounded)

The grid is not the only place evicted lines can go. When `scrollback_pager_history_size > 0`, the line evicted
at saturation (the `pagerhist_push` call at `kitty/history.c:L280`) is **serialized as UTF-8 text into a
separate ring buffer**. This axis is governed by:

- `initial_pagerhist_ringbuf_sz()` = `MIN(1024u*1024u, pagerhist_sz)` — `kitty/history.c:L67`: the ring starts
  at `MIN(1 MiB, configured size)`.
- `alloc_pagerhist()` — `kitty/history.c:L70-L80`: returns `NULL` when the configured size is 0 (feature
  disabled), otherwise backs the ring via `ringbuf_new` (`3rdparty/ringbuf/ringbuf.h:L41`).
- `pagerhist_extend()` — `kitty/history.c:L90-L101`: grows the ring by
  `newsz = MIN(ph->maximum_size, buffer_size + MAX(1024u*1024u, minsz))` (`kitty/history.c:L93`) — i.e. in
  chunks of **at least 1 MiB**, up to `maximum_size`.
- The configured maximum is `scrollback_pager_history_size` (default **0 MB**,
  `kitty/options/definition.py:L406`), parsed by `scrollback_pager_history_size()` as
  `int(max(0, float(x)) * 1024 * 1024)` capped at `min(ans, 4096*1024*1024 - 1)` —
  `kitty/options/utils.py:L564-L566` — i.e. **≤ 4 GiB − 1 bytes**.

The backing ring buffer is the vendored `ringbuf`, included at `kitty/history.c:L12` via
`#include "../3rdparty/ringbuf/ringbuf.h"` — note the `../`: the header lives at **`3rdparty/ringbuf/ringbuf.h`
at the repository root** (a sibling of `kitty/`), *not* inside `kitty/`.

**Observable as:** with `HistoryBuf(2000, 80, 8 MiB)` and 200,000 lines pushed, the **grid `count` stayed flat
at 2000** while `pagerhist_as_text()` returned **exactly 8,388,608 characters** — precisely the configured
8 MiB cap:

| Configuration | Lines pushed | Grid `count` | `pagerhist_as_text()` length |
|---|---|---|---|
| `HistoryBuf(2000, 80, 8 MiB)` | 200,000 | 2,000 (flat) | **8,388,608 chars** (= 8 MiB cap) |

This proves the pager history is a **distinct, independently-bounded growth axis** that the grid's `count`
metric does not reflect — you must monitor it separately (e.g. via `pagerhist_as_text()` length or process RSS).

### 3.4 Putting the boundaries together (the direct answer)

| Event | Code trigger | What memory monitoring shows |
|---|---|---|
| New segment allocated | `add_segment` via `segment_for` (`history.c:L37-L42`, guard L39) | **+5.0 MiB RSS step** at each 2048 crossing |
| Ring saturates | `count == ynum` in `historybuf_push` (`history.c:L279-L282`) | **flat** RSS — eviction, no allocation |
| Pager history grows | `pagerhist_extend` (`history.c:L90-L101`) in ≥1 MiB chunks, ≤ 4 GiB−1 | **separate, bounded** RSS/text-length growth |

"When does allocation occur?" — at each 2048-line crossing while filling (Boundary #1), and (if pager history
is enabled) in ≥1 MiB chunks as text is evicted (Boundary #3). "Can I observe it?" — yes: each grid allocation
is a ~5 MiB RSS step, saturation is a flat curve, and the pager axis is a separately bounded growth you can read
off `pagerhist_as_text()` or RSS.


---

## 4. Methodology & rationale

### 4.1 Build — running the real code

The analysis is grounded in the **actual compiled code**, not a re-implementation. `setup.py` is kitty's
authoritative builder; it compiles the `fast_data_types` C extension, which embeds the entire scrollback C core
(including `HistoryBuf`). In this environment the artifact `kitty/fast_data_types.so` is already built
(**1,253,792 bytes**) and imports headlessly under **Python 3.13.7**. The Go `kitten` binary and the GUI
windowing backends are **not** needed for scrollback memory analysis, so they are not exercised here.

### 4.2 Drive the real `HistoryBuf`

The constructor is `create_historybuf(...)` (`kitty/history.c:L117-L133`), exposed to Python through
`PyArg_ParseTuple(args, "II|I", &ynum, &xnum, &pagerhist_sz)` (`kitty/history.c:L138`). **The Python argument
order is therefore `HistoryBuf(ynum, xnum[, pagerhist_sz_bytes])`** — the **first** positional argument is
`ynum` (the line capacity / effective scrollback) and the **second** is `xnum` (columns). This is
counterintuitive (rows-before-columns) and is called out explicitly because getting it backwards silently
changes every memory number. (`create_historybuf` also calls `add_segment(self)` eagerly at
`kitty/history.c:L127`, so one segment exists immediately at construction, and wires the pager via
`alloc_pagerhist(pagerhist_sz)` at `kitty/history.c:L130`.)

Lines are produced and pushed exactly as the test suite does it
(`kitty_tests/__init__.py:L166`, `:L184-L188`; `kitty_tests/datatypes.py:L501` uses `HistoryBuf(3000, 5)` to
exercise multi-segment allocation):

```python
from kitty.fast_data_types import HistoryBuf, LineBuf, Cursor
hb = HistoryBuf(ynum, xnum, pagerhist_sz_bytes)   # e.g. HistoryBuf(100000, 80, 0)
lb = LineBuf(1, xnum)
line = lb.line(0)
line.set_text("x" * xnum, 0, xnum, Cursor())       # set_text: kitty/line.c:L485-L486
for _ in range(N):
    hb.push(line)                                  # historybuf_push: kitty/history.c:L276-L284
n = hb.count                                        # live line count
```

### 4.3 Sample RSS

Resident set size is read from `/proc/self/statm`: **field index 1 (resident pages) × `os.sysconf("SC_PAGE_SIZE")`**
(page size 4096 B here). RSS is sampled at and around each 2048-line boundary, and as a before/after delta for
the saturation and default-scrollback runs.

### 4.4 🔑 The default-scrollback caveat (restated, because it is load-bearing)

The default `scrollback_lines` is **2000** (`kitty/options/definition.py:L372`). Because the grid is allocated
in 2048-line segments (`kitty/history.c:L15`), a 2000-line cap is a **single segment**. Under the default
configuration, "hundreds of thousands of lines" produce an **essentially flat memory profile after the first
~2048 lines** — measured: **500,000 lines → ≈ 4.887 MiB total** grid growth, `count` capped at 2000. To
**observe** the allocation steps, you must configure a **large or effectively-infinite scrollback**; negative
values map to `2³²−1` lines via `scrollback_lines()` (`kitty/options/utils.py:L557-L561`). Skipping this step
is the single most common way a measurement would *understate* the behavior.

### 4.5 Repository immutability statement

This investigation produced **zero changes to the source repository**. All measurement harnesses live in
`/tmp` (outside the repo): `/tmp/hb_real.py` (drives the real `HistoryBuf` and samples RSS) and
`/tmp/hb_mem_probe.c` → `/tmp/hb_mem_probe` (the standalone C allocator model). The build artifacts
`kitty/fast_data_types.so` and `build/` are **git-ignored** (`.gitignore` entries `*.so` and `/build/`), so
they are not repository changes. `git status --porcelain` was **empty** before and after, and `HEAD` remained
pinned at `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. The only tracked file added by the task is this document
under `blitzy/documentation/`.

---

## Appendix — cross-validation & reproduction notes

### A.1 Allocation flowchart

```mermaid
flowchart TD
    Push["historybuf_push line<br/>history.c L276-L284"] --> Full{"count == ynum ?"}
    Full -->|"No (growing)"| Seg{"crossed a 2048<br/>line boundary?"}
    Seg -->|Yes| Alloc["add_segment: calloc ~5 MiB<br/>history.c L17-L29 -> RSS STEP"]
    Seg -->|No| NoOp["write into existing segment<br/>no allocation"]
    Full -->|"Yes (saturated)"| Ring["evict oldest -> pagerhist_push<br/>advance start_of_data -> RING<br/>history.c L280-L282 -> NO grid growth"]
    Ring --> PH{"pager history<br/>capacity exceeded?"}
    PH -->|Yes| Extend["pagerhist_extend >=1 MiB<br/>history.c L90 -> separate RSS growth"]
    PH -->|No| PHwrite["append text to ringbuf"]
```

### A.2 Standalone C cross-validation

To confirm the measured RSS steps reflect the **C allocator** and not Python interpreter overhead, a standalone
C model (`/tmp/hb_mem_probe.c`, outside the repo) replicates `add_segment()` (`kitty/history.c:L17-L29`) using
the exact asserted struct sizes (`sizeof(GPUCell)==20`, `sizeof(CPUCell)==12`, `sizeof(LineAttrs)==1`). It
`calloc`s `2048*(xnum*32+1)` bytes per segment, touches every page, and reads RSS from `/proc/self/statm`.

| Columns | per-segment (model = measured) | per-segment RSS step |
|---|---|---|
| 80  | **5,244,928 B = 5.002 MiB** (`2048*(80*32+1)`)  | ~5.00 MiB per segment |
| 200 | **13,109,248 B = 12.502 MiB** (`2048*(200*32+1)`) | ~12.5 MiB per segment |

The C model reproduces **5.002 MiB @ 80 cols** and **12.502 MiB @ 200 cols** to the byte, matching both the
derived formula and the Python-measured steps — confirming the staircase is the allocator.

### A.3 Reproduction notes

- Build (already done in this environment):
  `CI=true CFLAGS="-Wno-error" python3 setup.py build --ignore-compiler-warnings` produces the git-ignored
  `kitty/fast_data_types.so`.
- Drive headlessly with the repository root on `PYTHONPATH`:
  `PYTHONPATH=. python3 /tmp/hb_real.py`.
- Expected results (reproduced this session): large `HistoryBuf(100000,80,0)` → ~5 MiB steps at each 2048
  boundary; +30k beyond the cap → ~0 growth; default `HistoryBuf(2000,80,0)` + 500k lines → ≈ 4.887 MiB total;
  `HistoryBuf(2000,80,8 MiB)` + 200k lines → `pagerhist_as_text()` length = 8,388,608 chars.
- All scripts must remain in `/tmp`; the source repository stays byte-for-byte unchanged.

### A.4 Code locator index (all at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)

| Concern | Locator(s) |
|---|---|
| Segment size (2048) | `kitty/history.c:L15` |
| Segment allocation (`add_segment`, contiguous `calloc`) | `kitty/history.c:L17-L29` (calloc at `:L25`) |
| Grow trigger (`segment_for`) | `kitty/history.c:L37-L42` (guard `:L39`) |
| Pager init / alloc / extend | `kitty/history.c:L67`, `:L70-L80`, `:L90-L101` (newsz `:L93`) |
| `create_historybuf` / arg parse | `kitty/history.c:L117-L133` (`add_segment` `:L127`, `alloc_pagerhist` `:L130`); parse `:L138` |
| Push / eviction / saturation | `kitty/history.c:L276-L284` (branch `:L279-L282`) |
| Cell/attr sizes | `kitty/data-types.h:L221` (GPUCell=20), `:L228` (CPUCell=12), `:L231-L239` (LineAttrs=1) |
| Struct layouts | `kitty/data-types.h:L262-L266` (segment), `:L268-L272` (pager), `:L282-L290` (HistoryBuf) |
| Ringbuf header (ROOT) | `3rdparty/ringbuf/ringbuf.h` (e.g. `ringbuf_new` `:L41`); included at `kitty/history.c:L12` |
| `ynum = MAX(scrollback, lines)` | `kitty/screen.c:L130` |
| Scroll-view anchoring | `kitty/screen.c:L2716`, `:L2761` |
| Threads / render throttle / vsync | `kitty/child-monitor.c:L291` (io), `:L256`/`:L286` (talk), `:L871`/`:L875-L878` (render), `:L40` (vsync) |
| Config defaults & docs | `kitty/options/definition.py:L372`, `:L406`, `:L420`, `:L866`/`:L872-L875`, `:L878`, `:L889`/`:L895-L896` |
| Config parsers (limits) | `kitty/options/utils.py:L557-L561` (neg → 2³²−1), `:L564-L566` (≤ 4 GiB−1) |
| Test API patterns | `kitty_tests/__init__.py:L22`, `:L166`, `:L184-L188`; `kitty_tests/datatypes.py:L489`/`:L492`/`:L501` |
| `set_text` / line ops | `kitty/line.c:L485-L486`; `kitty/line-buf.c:L171`; `kitty/lineops.h:L122`/`:L125` |
| Build entry point | `setup.py` |
| Terminology | `docs/overview.rst:L248`/`:L250`; `docs/glossary.rst:L33`; `docs/unscroll.rst:L10` |

---

*Every quantitative claim above is either derived from the cited source at commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` or measured against the built `kitty/fast_data_types.so`; the two
agree within RSS sampling noise. No claim depends on web research — the code is the source of truth.*

