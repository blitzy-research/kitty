# What actually happens inside `HistoryBuf` during an enormous, fast burst of output

> **The question.** *"If I trigger a command that pours out an enormous amount of text in a very short time, what actually unfolds inside the `HistoryBuf` as it fills, stretches, and starts carving out new segments? There seems to be a quiet interaction between the segmented scrollback storage and the pager-style ring buffer... Does everything transition smoothly as segments reach their limits, or are there subtle moments where the system hesitates? And what changes if someone is actively scrolling through old output while new data is still arriving at full speed? I want to observe how allocation, wrapping, and retention really behave at runtime."*

This document answers that question **empirically**. Every claim below was produced by **building kitty's native extension and running it**, not by reading code alone. The relevant subsystem — the `HistoryBuf` object in `kitty/history.c` together with its embedded pager ring buffer (`pagerhist`) — was driven under a simulated high-volume burst using throwaway instrumentation scripts, and the captured console output is quoted verbatim in fenced blocks alongside the exact command that produced it. Every factual/numeric claim is tied either to a quoted output block or to a precise `file:line` citation.

- **Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`).
- **Read-only investigation.** No existing source file was modified. The only artifact produced is this document. Observation scripts lived under `/tmp/instr` (outside the repository) and were deleted afterward; the compiled extension `kitty/fast_data_types.so` and `build/` are gitignored (`.gitignore:L1` `*.so`, `.gitignore:L14` `/build/`).

The five parts of the question are answered explicitly as **OBJ-1** through **OBJ-5**, followed by a coverage pass and a list of the points that could *not* be verified by running (and are therefore flagged rather than asserted, per the exactness rule).

---

## TL;DR — the two-tier design

Kitty's scrollback is **two separate stores**, and the burst stresses them in sequence:

1. **The in-memory *segmented ring* (interactive scrollback).** This is `HistoryBuf` itself (`kitty/data-types.h:L282-L290`). Storage is an array of fixed-size **segments of 2048 lines each** (`#define SEGMENT_SIZE 2048`, `kitty/history.c:L15`), allocated **on demand** as the buffer fills (`add_segment`/`segment_for`, `kitty/history.c:L18-L42`). The populated line count `count` grows toward the capacity `ynum`, and the write position wraps modulo `ynum` around a ring whose logical start is `start_of_data` (`historybuf_push`, `kitty/history.c:L275-L284`). This is what a user scrolls through interactively.

2. **The separate *pager ring buffer* (browse-only).** This is `pagerhist` (`PagerHistoryBuf`, `kitty/data-types.h:L268-L272`), a **byte-addressable FIFO ring buffer** physically backed by `3rdparty/ringbuf/ringbuf.h` (described in its own header at `3rdparty/ringbuf/ringbuf.h:L3` as a *"C ring buffer (FIFO)"* and `3rdparty/ringbuf/ringbuf.h:L18` a *"byte-addressable ring buffer FIFO implementation"*). It is **not** used for interactive scrolling; it is fed **only by eviction** — every time the in-memory ring is already full and a new line arrives, the oldest line is serialized into the pager (`pagerhist_push`, `kitty/history.c:L258-L273`).

The quiet interaction the question asks about is precisely this handoff: **the interactive ring fills first and pins at `ynum`; only then does eviction begin, and only eviction feeds the pager.** The state that Python can observe on the `HistoryBuf` object is deliberately small — the only read-only members exposed are `xnum`, `ynum`, and `count` (`kitty/history.c:L556-L558`). In particular `num_segments` is **not** exposed, so segment carving must be *inferred* (from resident-memory growth plus the analytic boundary `ceil(min(count, ynum) / 2048)`).

---

## Methodology (build + run first, then write)

**Build.** The native extension was built (in the canonical environment it is already present as `kitty/fast_data_types.so`) with:

```
CI=true python3 setup.py build --ignore-compiler-warnings
```

`--ignore-compiler-warnings` is required because `setup.py:L491` otherwise gates the C flags with `-pedantic-errors -Werror` (the argparse flag itself is defined at `setup.py:L2003`); the flag bypasses a glfw `-Werror=switch` failure. The build also requires the usual dev libraries (`libharfbuzz-dev`, `libpng-dev`, `liblcms2-dev`, `libfontconfig-dev`, `libfreetype-dev`, `libgl-dev`, `libxxhash-dev`, `libssl-dev`, X11/Wayland/xkb dev libs), plus `libsimde-dev`. None of this modifies a repository file.

**Import surface.** With the extension built, `HistoryBuf` is instantiable. A sanity check confirms the exposed surface and — importantly — the constructor argument **order**:

```
$ PYTHONPATH=. python3 -c "
import kitty.fast_data_types as f
hb = f.HistoryBuf(5, 10)
print('HistoryBuf(5,10).ynum =', hb.ynum, '(expect 5)')
print('HistoryBuf(5,10).xnum =', hb.xnum, '(expect 10)')
print('HistoryBuf members:', sorted(x for x in dir(hb) if not x.startswith('__')))
print('has num_segments attr:', hasattr(hb, 'num_segments'), '(expect False)')
"
HistoryBuf(5,10).ynum = 5 (expect 5)
HistoryBuf(5,10).xnum = 10 (expect 10)
HistoryBuf members: ['as_ansi', 'count', 'dirty_lines', 'line', 'pagerhist_as_bytes', 'pagerhist_as_text', 'pagerhist_rewrap', 'pagerhist_write', 'push', 'rewrap', 'xnum', 'ynum']
has num_segments attr: False (expect False)
```

So the constructor is `HistoryBuf(ynum, xnum, [pagerhist_sz_bytes])` — **`ynum` first** (matching `PyArg_ParseTuple(args, "II|I", &ynum, &xnum, &pagerhist_sz)` at `kitty/history.c:L138`), and `num_segments` is confirmed absent from the Python object.

**Observe.** Four throwaway Python scripts and one C probe were authored under `/tmp/instr` (never in the repository) and run from the repository root with `PYTHONPATH=.` so the `kitty` package resolves. They drive `HistoryBuf.push` (and, for OBJ-4, a full `Screen`) in tight loops that simulate the burst, and capture `count`, pager length/content, resident memory (`resource.getrusage(RUSAGE_SELF).ru_maxrss`), and per-push timings (`time.perf_counter_ns`). The scripts were deleted after capture, leaving the repository unchanged.

**A note on determinism.** The OBJ-2, OBJ-4, OBJ-5 outputs and the `sizes.c` probe are **deterministic** and are quoted exactly. For OBJ-1/OBJ-3 the *segment-boundary structure* and the *shape* of the timing/memory behavior are stable and reproducible, but the exact **RSS figures and per-push nanosecond timings are environment- and run-specific**; the block quoted below is a **real captured run in this container**, presented as representative (see the "Unverifiable / environment-specific" section).

---

## OBJ-1 — Fill, "stretch", and carving new segments

**What the question asks:** what unfolds inside `HistoryBuf` as it fills, "stretches", and "starts carving out new segments" during a fast, enormous burst.

**How the sizes work (deterministic probe).** Each segment is a single `calloc`'d block sized `xnum * SEGMENT_SIZE * (sizeof(CPUCell) + sizeof(GPUCell)) + SEGMENT_SIZE * sizeof(LineAttrs)` (`add_segment`, `kitty/history.c:L18-L28`, specifically the allocation at `kitty/history.c:L25`). A small C probe compiled against `kitty/data-types.h` measures the exact sizes:

```
$ gcc -I kitty $(python3-config --includes) /tmp/instr/sizes.c -o /tmp/instr/sizes && /tmp/instr/sizes
sizeof(CPUCell)=12
sizeof(GPUCell)=20
sizeof(LineAttrs)=4
per-segment bytes @xnum=200 = 13115392 (12.51 MiB)
```

So at `xnum=200`, one segment is `200*2048*(12+20) + 2048*4 = 13,115,392` bytes = **12.51 MiB**. This is the "quantum" of memory the buffer grabs each time it carves a new segment.

**The burst (representative captured run).** `obj1_obj3.py` builds `HistoryBuf(ynum=10240, xnum=200)` and pushes `ynum+10` lines in a tight loop, printing `count`, the inferred `segments = ceil(min(count, ynum) / 2048)`, resident memory, and the per-push time:

```
$ PYTHONPATH=. python3 /tmp/instr/obj1_obj3.py
push#     0 count=     1 segments=1 RSS=   15360 KB dt=    6060 ns
push#     1 count=     2 segments=1 RSS=   15360 KB dt=    3906 ns
push#  2047 count=  2048 segments=1 RSS=   28672 KB dt=     310 ns
push#  2048 count=  2049 segments=2 RSS=   28672 KB dt=   13997 ns
push#  2049 count=  2050 segments=2 RSS=   28672 KB dt=    3130 ns
push#  4096 count=  4097 segments=3 RSS=   40960 KB dt=   19937 ns
push#  6144 count=  6145 segments=4 RSS=   54272 KB dt=   13380 ns
push#  8192 count=  8193 segments=5 RSS=   67584 KB dt=   11793 ns
push# 10239 count= 10240 segments=5 RSS=   80896 KB dt=     329 ns
push# 10240 count= 10240 segments=5 RSS=   80896 KB dt=    1273 ns
push# 10241 count= 10240 segments=5 RSS=   80896 KB dt=     790 ns
push# 10248 count= 10240 segments=5 RSS=   80896 KB dt=    1350 ns
FINAL count 10240 ==ynum True RSS growth KB 66560
push-time distribution: min=217 ns  median=3011 ns  p99=6890 ns  max=1042442 ns
slowest push at # 3521 = 1042442 ns
```

**Interpretation.**

- **Fill:** `count` rises by one on every push (`else self->count++`, `kitty/history.c:L282`) until it reaches the capacity `ynum=10240`, then **pins** there — the last three sampled pushes show `count=10240` unchanged, and `FINAL count 10240 ==ynum True`. This pinning is the ring behavior: once full, `count` stops growing and the write instead evicts (see OBJ-2).
- **"Stretch" / carve:** the buffer does not pre-allocate all its memory. Segments are created lazily by `segment_for` (`kitty/history.c:L37-L42`), which computes `seg_num = y / SEGMENT_SIZE` and loops `while (seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum) add_segment(self)`. Because `num_segments` is not exposed to Python (`kitty/history.c:L556-L558` expose only `xnum`, `ynum`, `count`), carving is inferred two ways, which agree:
  - The analytic boundary `segments = ceil(min(count, ynum) / 2048)` steps up exactly at pushes **#2048, #4096, #6144, #8192** (segments `1→2→3→4→5`).
  - Resident memory climbs in discrete steps of ~12–13 MB — the RSS samples `15360 → 28672 → 40960 → 54272 → 67584 → 80896 KB` are increments of `13312 / 12288 / 13312 / 13312 / 13312 KB` — each step ≈ the 12.51 MiB per-segment block measured above. Total `RSS growth KB 66560` ≈ five segments' worth.
- **Capacity note:** the burst crosses the 2048-line segment boundary five times only because `ynum=10240` was chosen well above the default. A **default-configured** window uses `scrollback_lines = 2000` (`kitty/options/definition.py:L372`), and `2000 < 2048 = SEGMENT_SIZE`, so a default window's entire scrollback fits inside a **single segment** — segment carving is *only observable when scrollback exceeds the default*. This is why the harness uses `ynum=10240`.
- `add_segment` is **fatal-on-OOM**: if the `realloc` or `calloc` fails it calls `fatal(...)` and the process dies (`kitty/history.c:L21` and `kitty/history.c:L26`) — there is no soft-fail path for a segment allocation.


---

## OBJ-2 — The quiet interaction: segmented storage ↔ pager ring buffer

**What the question asks:** the "quiet interaction" between the in-memory segmented scrollback ring and the pager-style ring buffer under stress.

`obj2.py` uses a tiny buffer, `HistoryBuf(5, 10, 1<<20)` — capacity `ynum=5`, width `xnum=10`, and a **1 MiB pager** (note: the third argument is in **bytes**, so `1<<20` = 1,048,576; see the corrections section) — and pushes past capacity while watching the pager's byte length and content:

```
$ PYTHONPATH=. python3 /tmp/instr/obj2.py
=== Phase A: ring filling (count 1..ynum) ===
push line0: count=1  pagerhist len=0
push line1: count=2  pagerhist len=0
push line2: count=3  pagerhist len=0
push line3: count=4  pagerhist len=0
push line4: count=5  pagerhist len=0
=== Phase B: ring FULL, evictions begin ===
push line5: count=5  pagerhist len=15  bytes=b'\x1b[mline0     \r\n'
push line6: count=5  pagerhist len=30  bytes=b'\x1b[mline0     \r\n\x1b[mline1     \r\n'
push line7: count=5  pagerhist len=45  bytes=b'\x1b[mline0     \r\n\x1b[mline1     \r\n\x1b[mline2     \r\n'
=== Phase C: default pager sz=0 discards ===
after 8 pushes count=5  pagerhist len=0  bytes=b''
```

**Interpretation.**

- **Phase A — the pager stays empty while the ring fills.** For the first `ynum=5` pushes, `count` grows `1 → 5` and `pagerhist len=0` the whole time. Nothing is handed to the pager while there is still room in the interactive ring.
- **Phase B — eviction begins exactly at capacity, and *only then* feeds the pager.** The trigger is in `historybuf_push`: `if (self->count == self->ynum)` (`kitty/history.c:L279`) it calls `pagerhist_push(self, as_ansi_buf)` (`kitty/history.c:L280`) for the line about to be overwritten and advances the ring start `start_of_data = (start_of_data + 1) % ynum` (`kitty/history.c:L281`). From `push line5` onward `count` holds at `5` while the pager grows by exactly **15 bytes** per eviction.
- **The 15-byte layout is exact.** `pagerhist_push` (`kitty/history.c:L258-L273`) serializes the evicted line as: the 3-byte SGR reset `"\x1b[m"` (`kitty/history.c:L266`), then the line's UTF-8 (here `line0` padded to `xnum=10` → the 10 bytes `line0` + five spaces), then `\r` (`kitty/history.c:L269`), then a **conditional** `\n` (`kitty/history.c:L270`). `3 + 10 + 1 + 1 = 15`. The captured `bytes=b'\x1b[mline0     \r\n'` shows exactly that — note the five significant padding spaces between `line0` and `\r\n`.
- **Phase C — with the default pager the eviction is discarded.** Constructing `HistoryBuf(5, 10)` with **no** third argument leaves `pagerhist_sz = 0`, and `alloc_pagerhist` returns `NULL` for a zero size (`kitty/history.c:L72`). Then `pagerhist_push` bails immediately at its guard `if (!ph) return;` (`kitty/history.c:L261`). The captured result — `after 8 pushes count=5 pagerhist len=0 bytes=b''` — confirms evictions are simply dropped. This matches the config default `scrollback_pager_history_size` `'0'` (`kitty/options/definition.py:L406-L407`), whose docstring states *"A value of zero or less disables this feature"* (`kitty/options/definition.py:L414`).

**Pager capacity and overflow (how the byte ring itself behaves under stress).** The pager ring is initialized to `MIN(1024u * 1024u, pagerhist_sz)` bytes — i.e. at most 1 MiB up front (`initial_pagerhist_ringbuf_sz`, `kitty/history.c:L67`) — and grows on demand: `pagerhist_write_bytes` extends it when a write would not fit (`kitty/history.c:L222-L223`), via `pagerhist_extend`, which computes `newsz = MIN(ph->maximum_size, buffer_size + MAX(1024u*1024u, minsz))` (`kitty/history.c:L93`) — so it grows in **≥1 MiB steps** up to `maximum_size` (which was set to `pagerhist_sz` at `kitty/history.c:L78`). Two hard limits are worth stating exactly: `pagerhist_extend` refuses to grow once `buffer_size >= ph->maximum_size` (`kitty/history.c:L92`), and `pagerhist_write_bytes` refuses any single write larger than the maximum, `if (sz > ph->maximum_size) return false` (`kitty/history.c:L220`). Because it is a FIFO ring buffer (`3rdparty/ringbuf/ringbuf.h`), once it is at maximum size the oldest pager bytes are overwritten by new evictions — the browse-only history is itself bounded.

---

## OBJ-3 — Does it transition smoothly, or hesitate?

**What the question asks:** whether everything "transitions smoothly as segments reach their limits", or whether there are "subtle moments where the system hesitates or behaves differently".

The answer comes from the **per-push timings** in the same `obj1_obj3.py` run quoted under OBJ-1. The relevant lines are the boundary pushes and the distribution summary:

```
push#  2047 count=  2048 segments=1 RSS=   28672 KB dt=     310 ns   <- last push before a new segment
push#  2048 count=  2049 segments=2 RSS=   28672 KB dt=   13997 ns   <- carves segment 2
push#  4096 count=  4097 segments=3 RSS=   40960 KB dt=   19937 ns   <- carves segment 3
push#  6144 count=  6145 segments=4 RSS=   54272 KB dt=   13380 ns   <- carves segment 4
push#  8192 count=  8193 segments=5 RSS=   67584 KB dt=   11793 ns   <- carves segment 5
...
push-time distribution: min=217 ns  median=3011 ns  p99=6890 ns  max=1042442 ns
slowest push at # 3521 = 1042442 ns
```

**Interpretation — mostly smooth, with real but small and predictable hesitations, plus rare large jitter.**

- **Predictable hesitation at each 2048 boundary.** The four segment-carving pushes (**#2048 = 13997 ns, #4096 = 19937 ns, #6144 = 13380 ns, #8192 = 11793 ns**) are markedly slower than the `median=3011 ns` — roughly **4–7× the median**. This is deterministic in *location* (always at a multiple of `SEGMENT_SIZE`) and is caused directly by the work in `add_segment`: a `realloc` of the segments array (`kitty/history.c:L20`) followed by a fresh `calloc` of a ~12.51 MiB block (`kitty/history.c:L25`). The line immediately *before* a boundary (`#2047 = 310 ns`) is an ordinary fast push — the cost is localized to the push that actually crosses the boundary.
- **Steady-state is smooth.** Away from boundaries, pushes are cheap and tightly clustered: `min=217 ns`, `median=3011 ns`, `p99=6890 ns`. Once `count` pins at `ynum` (pushes ≥ #10240), each push does the constant-time eviction path, and timings stay small (`1273 / 790 / 1350 ns` in the tail samples).
- **Occasional large, *non-deterministic* spikes off the boundaries.** The absolute-slowest push in this run was **#3521 at 1,042,442 ns** (~1 ms) — and #3521 is **not** a segment boundary. Spikes like this are sporadic allocator / page-fault / OS-scheduler jitter, not a structural feature of `HistoryBuf`; their magnitude and location vary from run to run.

So: transitions across segment limits are smooth in the sense that there is no stall, reflow, or data movement of existing lines — a new segment is simply appended — but there **is** a small, repeatable per-boundary cost from the allocation itself, and the memory footprint jumps in ~12.51 MiB steps rather than growing byte-by-byte. The only catastrophic (non-graceful) transition is allocation failure, which is fatal by design (`kitty/history.c:L21`, `kitty/history.c:L26`).

> The exact nanosecond values and RSS figures above are **environment- and run-specific** (see the final section). The *structure* — count pinning at `ynum`, boundaries at 2048/4096/6144/8192, ~12–13 MB RSS steps, and boundary pushes several× the median with rarer off-boundary spikes — is the stable, reproducible finding.


---

## OBJ-4 — Scrolling through old output while new data floods in

**What the question asks:** what changes when someone is "actively scrolling through old output while new data is still arriving at full speed".

The key structural fact is that **the scroll position is a `Screen`-layer concept, not a `HistoryBuf` one.** `obj4.py` drives a real `Screen` (via the test harness `Callbacks`/`parse_bytes`), feeds it a burst, scrolls up mid-flood, feeds more, then scrolls to the bottom:

```
$ PYTHONPATH=. python3 /tmp/instr/obj4.py
initial:                                  scrolled_by=0  historybuf.count=0
after burst-1 (40 lines):                 scrolled_by=0  count=36  newest_in_hist='row35'
user scrolls up 10 (scroll(10,True)):     scrolled_by=10  count=36  (count UNCHANGED by scrolling)
after burst-2 (30 lines) WHILE scrolled:  scrolled_by=10  count=66  newest_in_hist='row65'
after scroll to bottom:                   scrolled_by=0  count=66
```

**Interpretation.**

- **`scrolled_by` lives on `Screen`, `count` lives on `HistoryBuf`.** `scrolled_by` is a read-only `Screen` member (`kitty/screen.c:L4903`) representing how many lines up from the live view the user is looking; the `Screen` reaches its history via its own read-only `historybuf` member (`kitty/screen.c:L4902`). These are independent: scrolling up 10 changed `scrolled_by` `0 → 10` but left `count=36` **unchanged**.
- **Ingestion never pauses for the viewer.** While the user is "parked" at `scrolled_by=10`, the second 30-line burst kept pushing lines into history: `count` advanced `36 → 66` and the newest stored line moved from `'row35'` to `'row65'`. The storage ring keeps evicting/retaining regardless of where the user is scrolled — the burst does not stop, slow, or wait for the reader. (Lines scroll off the live grid into history through `historybuf_add_line`, `kitty/screen.c:L1558`, which also bumps `self->history_line_added_count`, `kitty/screen.c:L1559`.)
- **The interactive `scroll()` clamps to `count`.** Scrolling "down to the bottom" via `s.scroll(10000, False)` resets `scrolled_by` to `0`. The Python `scroll()` method (`kitty/screen.c:L4121`) computes `new_scroll = MIN(self->scrolled_by + amt, self->historybuf->count)` (`kitty/screen.c:L4111`), so the viewport offset can never exceed the number of retained history lines. Clearing scrollback resets it too: `screen_clear_scrollback` sets `self->scrolled_by = 0` (`kitty/screen.c:L1916-L1917`).

**Flagged as verified-by-reading, NOT by running (per the exactness rule).** In a *real* rendering loop, kitty tries to keep a scrolled-back user "pinned" to the same old content as new lines arrive, by advancing `scrolled_by` by the number of lines just added: `if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count)` — this appears **twice**, at `kitty/screen.c:L2716` and `kitty/screen.c:L2761`, inside the screen-update/render path. In this **headless, parse-only** harness that render re-anchoring was **not** exercised: `scrolled_by` stayed at `10` across the second burst rather than being pushed up by 30. So the "your view stays pinned to the old text as the flood continues" behavior is confirmed *by reading the code*, but was *not* reproduced by this observation harness, and is reported as such rather than asserted from runtime.

---

## OBJ-5 — How allocation, wrapping, and retention really behave

**What the question asks:** direct observation of how allocation, line wrapping (continuation lines), and retention "really behave at runtime".

`obj5.py` builds three lines at width `xnum=8`, where the first line `"AAAAAAAA"` exactly fills the width (so it is marked *continued* into the next line), pushes them into `HistoryBuf(3, 8)`, and dumps the results; it then evicts a wrapped-then-plain pair into a pager:

```
$ PYTHONPATH=. python3 /tmp/instr/obj5.py
LineBuf.is_continued: [False, True, False]
HistoryBuf.as_ansi pieces: ['AAAAAAAA', 'bbb\n', 'CCCCCCCC\n']
concatenated: 'AAAAAAAAbbb\nCCCCCCCC\n'
stored round-trip (reverse index, 0=newest): ['CCCCCCCC', 'bbb', 'AAAAAAAA']
pager eviction of wrapped+plain: b'\x1b[mAAAAAAAA\r\x1b[mbbb\r\n'
```

**Interpretation.**

- **Wrapping is a per-cell flag, not a stored newline.** Marking a line continued sets the *last cell's* `next_char_was_wrapped` attribute: `set_continued(y, val)` (`kitty/line-buf.c:L217-L223`) calls `linebuf_set_last_char_as_continuation(self, y-1, val)` (`kitty/line-buf.c:L223`), which sets `...[self->xnum - 1].attrs.next_char_was_wrapped = continued` (`kitty/line-buf.c:L194`, `kitty/line-buf.c:L196`). The captured `LineBuf.is_continued: [False, True, False]` reflects that line index 1 continues line 0.
- **`as_ansi` joins wrapped lines with no newline, and terminates unwrapped lines with `\n`.** In `as_ansi` (`kitty/history.c:L347-L358`) the newline is appended only when the last cell was **not** wrapped: `if (!l.gpu_cells[l.xnum - 1].attrs.next_char_was_wrapped)` (`kitty/history.c:L356`) then `output.buf[output.len++] = '\n'` (`kitty/history.c:L358`). That is exactly why the pieces are `['AAAAAAAA', 'bbb\n', 'CCCCCCCC\n']` — the wrapped `'AAAAAAAA'` has **no** trailing `\n`, and concatenation yields `'AAAAAAAAbbb\nCCCCCCCC\n'` (the wrapped first line fuses seamlessly onto its continuation). Retention is faithful: the text comes back byte-for-byte.
- **Retrieval is reverse-indexed (0 = newest).** `hb.line(i)` uses `index_of` (`kitty/history.c:L152-L159`), whose own comment (`kitty/history.c:L155`) reads verbatim `// This is reverse indexing, i.e. lnum = 0 corresponds to the *last* line in the buffer.`. The round-trip `['CCCCCCCC', 'bbb', 'AAAAAAAA']` confirms it: index 0 returns the newest line `CCCCCCCC`, and index 2 the oldest `AAAAAAAA`.
- **On eviction, wrapping changes the pager line terminator.** The final line shows a wrapped line followed by a plain line evicted into the pager: `b'\x1b[mAAAAAAAA\r\x1b[mbbb\r\n'`. The wrapped `AAAAAAAA` ends with `\r` **only** (no `\n`), while the plain `bbb` ends with `\r\n` — the same conditional-`\n` rule as `as_ansi`, applied in `pagerhist_push` at `kitty/history.c:L269-L270`.
- **Allocation backing this retention.** The cells that store these lines live in the segment blocks allocated by `add_segment` (`kitty/history.c:L18-L28`): each line occupies `xnum` `CPUCell`s + `xnum` `GPUCell`s plus one `LineAttrs`, laid out per `HistoryBufSegment { GPUCell *gpu_cells; CPUCell *cpu_cells; LineAttrs *line_attrs; }` (`kitty/data-types.h:L262-L266`). The `next_char_was_wrapped` bit that governs all of the above lives in a cell's attributes within that GPU-cell array.

---

## Two runtime corrections (verified against the tree)

While investigating, two labels from higher-level notes needed correcting; both are verified true here and matter for anyone reproducing this:

1. **The ringbuf backing store lives at the repository root: `3rdparty/ringbuf/ringbuf.h` — not `kitty/3rdparty/...`.** Proof: `kitty/history.c:L12` includes it as `#include "../3rdparty/ringbuf/ringbuf.h"` — the `../` climbs out of `kitty/` to the repository root. Confirmed on disk: `kitty/3rdparty/` does not exist, whereas `3rdparty/ringbuf/ringbuf.h` does (252 lines). All pager-ring citations here therefore use `3rdparty/ringbuf/ringbuf.h`.

2. **The `HistoryBuf(ynum, xnum, pagerhist_sz)` third argument is in BYTES, not megabytes.** Proof: the constructor parses three unsigned ints, `PyArg_ParseTuple(args, "II|I", &ynum, &xnum, &pagerhist_sz)` (`kitty/history.c:L138`), and `alloc_pagerhist` stores that raw value as the maximum size: `ph->maximum_size = pagerhist_sz;` (`kitty/history.c:L78`). So a 1 MiB pager is `1<<20` (as used in OBJ-2/OBJ-5); `HistoryBuf(5, 10, 1)` would make a useless **1-byte** pager. This is distinct from the *config option* `scrollback_pager_history_size`, which is expressed in MB per its docstring (`kitty/options/definition.py:L409`) — the C constructor argument is raw bytes.

Two related binding facts, also verified at runtime (see Methodology): the constructor order is **`ynum` first** (`HistoryBuf(5,10)` yields `ynum=5, xnum=10`), and the only read-only members exposed to Python are `xnum` (`kitty/history.c:L556`), `ynum` (`kitty/history.c:L557`), and `count` (`kitty/history.c:L558`) — so `num_segments` is **not** exposed, and there is no standalone `PagerHistoryBuf` Python type (the pager is reached only through `HistoryBuf.pagerhist_*` methods).

---

## Coverage pass — every sub-question answered

| Sub-question (from the prompt) | Answered in | Core empirical finding |
|---|---|---|
| **OBJ-1** — what unfolds as it fills, "stretches", and carves new segments | [OBJ-1](#obj-1--fill-stretch-and-carving-new-segments) | `count` rises to `ynum` then pins; segments carved lazily at each 2048-line boundary (`ceil(min(count,ynum)/2048)` → 1→2→3→4→5 at #2048/#4096/#6144/#8192), each ≈12.51 MiB (`sizes.c`; `kitty/history.c:L15,L18-L42`) |
| **OBJ-2** — the quiet storage ↔ pager interaction | [OBJ-2](#obj-2--the-quiet-interaction-segmented-storage--pager-ring-buffer) | Pager stays empty while the ring fills; at `count==ynum` each push evicts the oldest line into the pager (15 bytes: `\x1b[m` + text + `\r`[+`\n`]); default `sz=0` discards (`kitty/history.c:L258-L284`) |
| **OBJ-3** — smooth, or subtle hesitations? | [OBJ-3](#obj-3--does-it-transition-smoothly-or-hesitate) | Mostly smooth; predictable ~4–7× median hesitation at each 2048 boundary (realloc+calloc in `add_segment`); rarer large off-boundary jitter; fatal-on-OOM (`kitty/history.c:L21,L26`) |
| **OBJ-4** — scrolling old output while new data floods in | [OBJ-4](#obj-4--scrolling-through-old-output-while-new-data-floods-in) | `scrolled_by` is a `Screen` viewport index (`kitty/screen.c:L4903`) independent of `HistoryBuf.count`; ingestion advanced `count` 36→66 while the user stayed parked; interactive `scroll()` clamps to `count` (`kitty/screen.c:L4111`) |
| **OBJ-5** — allocation, wrapping, retention at runtime | [OBJ-5](#obj-5--how-allocation-wrapping-and-retention-really-behave) | Continuation is the per-cell `next_char_was_wrapped` flag; `as_ansi` joins wrapped lines with no `\n` and terminates plain lines with `\n` (`kitty/history.c:L356-L358`); retrieval reverse-indexed 0=newest (`kitty/history.c:L152-L159`); retention faithful |

All five sub-questions are explicitly addressed with a producing command, verbatim captured output, and `file:line` citations.

---

## Unverifiable-by-running / environment-specific (flagged, not asserted)

Per the exactness rule, the following are called out explicitly rather than presented as freshly measured runtime facts:

- **OBJ-4 render re-anchoring was NOT exercised.** The clamp `self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count)` at `kitty/screen.c:L2716` and `kitty/screen.c:L2761` — which keeps a scrolled-back user pinned to old content as new lines flood in — is confirmed **by reading the code only**. The headless, parse-only harness left `scrolled_by` at `10` across the second burst, so this specific behavior was not reproduced at runtime here.
- **OBJ-1/OBJ-3 RSS and timing numbers are environment- and run-specific.** The quoted block (e.g. `median=3011 ns`, `max=1042442 ns`, `slowest push at #3521`, and the exact RSS values `15360…80896 KB`) is one **representative captured run in this container**. Re-running will produce different absolute nanoseconds and RSS figures. What is stable and reproducible is the *structure*: `count` pins at `ynum`; segment boundaries fall at multiples of 2048 (2048/4096/6144/8192); RSS rises in ~12–13 MB steps matching the 12.51 MiB per-segment allocation; boundary pushes are several× the median; and off-boundary spikes are sporadic jitter.
- **`num_segments` is not directly observable from Python** (`kitty/history.c:L556-L558` expose only `xnum`, `ynum`, `count`), so segment carving is *inferred* from RSS steps plus the analytic boundary `ceil(min(count, ynum) / 2048)` — the two agree in every sample, but neither is a direct read of the C `num_segments` field.

---

### Reproduction summary (commands used)

```
# build (native extension; produces gitignored kitty/fast_data_types.so)
CI=true python3 setup.py build --ignore-compiler-warnings

# exact allocation math
gcc -I kitty $(python3-config --includes) /tmp/instr/sizes.c -o /tmp/instr/sizes && /tmp/instr/sizes

# the four observation scripts (run from repo root)
PYTHONPATH=. python3 /tmp/instr/obj1_obj3.py   # OBJ-1 fill/carve + OBJ-3 timings
PYTHONPATH=. python3 /tmp/instr/obj2.py         # OBJ-2 storage <-> pager
PYTHONPATH=. python3 /tmp/instr/obj4.py         # OBJ-4 concurrent scroll
PYTHONPATH=. python3 /tmp/instr/obj5.py         # OBJ-5 wrapping/retention
```

All observation scripts lived under `/tmp/instr` (outside the repository) and were removed after capture; `kitty/fast_data_types.so` and `build/` are gitignored, leaving the source tree unchanged.

