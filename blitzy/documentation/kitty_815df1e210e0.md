# What actually unfolds inside Kitty's `HistoryBuf` during an enormous, fast output burst

*A runtime-grounded investigation of the segmented scrollback line-store, its companion pager byte‑ring, and how the two behave under stress — with live evidence, `file:line` citations, and cause→effect reasoning.*

---

## TL;DR (the direct, plain‑reading answer)

When a process floods the terminal, every line that scrolls off the top of the grid is handed to `HistoryBuf` through the real ingest path `INDEX_UP → historybuf_add_line → historybuf_push` (`kitty/screen.c:1552-1568`, `kitty/history.c:286-291`, `kitty/history.c:275-284`). Inside `HistoryBuf` the lines live in a **fixed ring of `ynum` slots** addressed by `start_of_data` + `count`. During the burst `count` climbs one‑per‑line until `count == ynum`; from that instant on it **stops growing** and each new line **evicts the oldest** (retention hard‑limit = `ynum`). The backing memory is carved in fixed **2048‑line segments** (`SEGMENT_SIZE`, `kitty/history.c:15`): **segment 0 is allocated eagerly at construction** and further segments are allocated **lazily** the first time a line index crosses a 2048 boundary — I observed process RSS rise in **5 discrete ~5 MiB steps** for a 10000‑line buffer, corroborated by exactly **5** segment‑sized `mmap`s under `strace` and **4** lazy `add_segment` calls under `gdb`.

The "quiet interaction" the question senses is real and has **exactly one coupling point**: the pager byte‑ring is fed **only when the line ring evicts** — `historybuf_push()` calls `pagerhist_push()` precisely at `count == ynum` (`kitty/history.c:280`). The ring is **disabled by default** (`scrollback_pager_history_size = 0`, `kitty/options/definition.py:406-407`) in which case `pagerhist_push` is a no‑op; when enabled it accretes evicted (oldest‑first) lines and grows in discrete 1 MiB steps with a full `ringbuf_copy` per extend, then **plateaus at `maximum_size`** — I watched it grow 1→2→3→4 MiB and stop at exactly 4,194,304 bytes.

Is the transition smooth? **Mostly incremental, but punctuated by discrete allocation/copy spikes** at the 2048‑line segment boundaries (a `realloc` + a ~5 MiB `calloc`) and at ring‑growth boundaries (a `ringbuf_copy` whose cost grows with occupancy); at the true extreme, memory exhaustion is a **hard `fatal()` abort, not graceful degradation** (this one path is *inferred*, deliberately not triggered). And if someone is scrolled back while data pours in, there is **no lock/thread contention** — ingest merely accumulates `history_line_added_count`, and the viewport is re‑anchored at render time via `scrolled_by = MIN(scrolled_by + history_line_added_count, count)` (`kitty/screen.c:2716` and `:2761`), so the view stays pinned to the same old line until `count` saturates, after which the oldest content is evicted and the view "slides off."

Everything below is reproduced from live runs of the real `Screen` ingest path; every behavioral claim carries its own unedited output and a `file:line` citation.

---

## 1. How this was investigated (methodology + build)

### 1.1 Canonical build

- **Canonical command:** `python3 setup.py build` → builds the CPython extension `kitty/fast_data_types.so`, which exposes `HistoryBuf` and `Screen`.
- **Import‑check command:** `PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q0_import_check.py`

The import check confirms the observable substrate (unedited, stable across 2 runs):

```
HistoryBuf present: True
Screen present: True
PagerHistoryBuf top-level present: False
SCROLL_LINE: -999999 SCROLL_PAGE: -999998 SCROLL_FULL: -999997
HistoryBuf public attrs: ['as_ansi', 'count', 'dirty_lines', 'line', 'pagerhist_as_bytes', 'pagerhist_as_text', 'pagerhist_rewrap', 'pagerhist_write', 'push', 'rewrap', 'xnum', 'ynum']
```

### 1.2 Sandbox environment artifacts — disclosed as **NON‑CANONICAL** (not project defaults)

- **Toolchain / interpreter actually observed here:** Python **3.13.7**; the built `kitty/fast_data_types.so` is **1,253,792 bytes**. (These are environment‑specific values, reported honestly rather than assumed.)
- **Build‑config override actually used:** the canonical `python3 setup.py build` hit the wayland‑protocols 1.45 enum skew:
  ```
  glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT' not handled in switch [-Werror=switch]
  cc1: all warnings being treated as errors
  ```
  so the extension was built with the override **`CFLAGS='-Wno-error=switch' python3 setup.py build`**. This override is an **environment artifact, NOT a canonical default** — it only suppresses an unrelated GLFW `-Werror=switch` in the Wayland window code and does not touch the scrollback subsystem. In this sandbox that build **succeeds fully (exit 0)** including the launcher link, because the Go toolchain (Go 1.24.4) is present; the trailing Go step that produces the `kitten` binary is not exercised by this investigation.
- **Observation tools installed via apt (non‑canonical, for observation only):** `strace` and `gdb`. These do not alter the build or the project.

### 1.3 Runtime observability method

`HistoryBuf` exposes **only** `xnum`, `ynum`, and `count` to Python — a `PyMemberDef` array with exactly those three `READONLY` members (`kitty/history.c:555-559`):

```c
static PyMemberDef members[] = {
    {"xnum", T_UINT, offsetof(HistoryBuf, xnum), READONLY, "xnum"},
    {"ynum", T_UINT, offsetof(HistoryBuf, ynum), READONLY, "ynum"},
    {"count", T_UINT, offsetof(HistoryBuf, count), READONLY, "count"},
    {NULL}  /* Sentinel */
};
```

`num_segments` and `start_of_data` are **not** exposed. Segment carving is therefore observed **indirectly** by three independent methods, all used below:
- **(a) process RSS** growth in deterministic ~5 MiB steps (primary);
- **(b) `strace`** of the `calloc`‑backed segment `mmap`s, bracketed with `write(2, "=MARK …")` markers;
- **(c) `gdb`** breakpoint on `add_segment` (bound by its LTO‑mangled symbol — see Q1 and the Divergences section).

### 1.4 Canonical entry point (mandatory)

All observations drive the **real `Screen` ingest path** using the in‑repo `kitty_tests` harness:

```python
from kitty_tests import BaseTest
class _T(BaseTest):
    def runTest(self): pass
s = _T().create_screen(cols=..., lines=..., scrollback=..., options={...})
s.draw(text); s.linefeed(); s.carriage_return()   # repeated for the burst
```

`create_screen` builds a genuine `Screen(...)` (`kitty_tests/__init__.py:237-240`); `draw` + `linefeed` push lines off the grid top into history through `INDEX_UP → historybuf_add_line` (`kitty/screen.c:1552-1568`). The direct `HistoryBuf.push()` method exists but is **NON‑CANONICAL** and is **not** used for any behavioral claim here.

---

## 2. Divergences the reader must know (honest caveats)

1. **`PagerHistoryBuf` is NOT a top‑level Python type.** It is an internal C struct — `{ void *ringbuf; size_t maximum_size; bool rewrap_needed; }` (`kitty/data-types.h:268-272`) — reached only as the `pagerhist` pointer member of `HistoryBuf` (`kitty/data-types.h:287`). It is accessed exclusively through `HistoryBuf` methods: `pagerhist_as_bytes()`, `pagerhist_as_text()`, `pagerhist_rewrap()`, `pagerhist_write()`. The constructor is `HistoryBuf(ynum, xnum[, pagerhist_sz])`; `push` is the NON‑CANONICAL direct push. Verified at runtime by `PagerHistoryBuf top-level present: False` (§1.1). *(Citation correction vs. the working notes: the `pagerhist` member is at `data-types.h:287`, not `:288` — line 288 is `Line *line;`.)*

2. **Segment 0 is carved EAGERLY at construction, not lazily.** `create_historybuf` sets `num_segments = 0` and then immediately calls `add_segment(self)` (`kitty/history.c:126-127`). Further segments are lazy via `segment_for()` (`kitty/history.c:36-42`). So a 5‑segment buffer shows **1 eager + 4 lazy** carves — confirmed by the `strace`/`gdb` evidence in Q1. (This refines the looser "allocated lazily" phrasing: the *first* segment is not lazy.)

3. **Scroll re‑anchoring appears at BOTH `kitty/screen.c:2716` and `:2761`** — identical `scrolled_by = MIN(scrolled_by + history_line_added_count, historybuf->count)` in `screen_update_only_line_graphics_data` and `screen_update_cell_data` respectively. Q4 exercises the `:2716` path (the one reachable from the Python harness) but both are cited.

4. **Harness pager default gotcha.** The `kitty_tests` harness `set_options()` defaults `scrollback_pager_history_size = 1024` (already a *byte* count — it bypasses the MB→bytes parser at `kitty/options/utils.py:564`) — see `kitty_tests/__init__.py:224`. Therefore, to reproduce the **canonical default OFF** you must pass `options={'scrollback_pager_history_size': 0}`; for **ON** you pass a raw **byte** count (e.g. `4*1024*1024`). Every script below does exactly this.

5. **±1 fed‑index jitter (reported honestly).** The exact line index at which `count == ynum` is first reached (and the first pager byte appears) can shift by ±1 between scripts because of initial grid state and where the sampling happens in the loop. **In my runs I observed line‑ring‑full at fed index `11` and the first pager bytes one line later at fed index `12`** (see Q2), stable across repeats. This is a sampling detail, not a behavioral difference.

---

## 3. Q1 — Segment fill / stretch / carve

### 3.1 Direct answer (plain reading first)

The line store is a **fixed ring of `ynum` slots** addressed by `start_of_data` + `count`. During the burst `count` climbs **one‑per‑line** until `count == ynum`, then it **stops growing** and the oldest line is evicted per new line (retention limit = `ynum`). Backing memory is carved in fixed **2048‑line segments** (`SEGMENT_SIZE`, `kitty/history.c:15`): **segment 0 is allocated eagerly at construction** (`create_historybuf → add_segment`, `kitty/history.c:127`); further segments are allocated **lazily** by `segment_for()` (`kitty/history.c:36-42`) as line indices first cross 2048 boundaries. Total segments = `ceil(ynum / 2048)` (for `ynum = 10000`, that is 5).

### 3.2 Command

Scale: **20000 lines fed** into a `scrollback=10000` buffer (`ynum = 10000 ≫ 2048`, so multiple segments are forced), pager OFF, 80 columns. Run twice for stability.

```
PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q1_segments.py
```

The script (`/tmp/blitzy_adhoc/q1_segments.py`):

```python
from kitty_tests import BaseTest
class _T(BaseTest):
    def runTest(self): pass
def rss_kb():
    for ln in open('/proc/self/status'):
        if ln.startswith('VmRSS'): return int(ln.split()[1])
def run(tag):
    bt=_T()
    s=bt.create_screen(cols=80, lines=24, scrollback=10000, options={'scrollback_pager_history_size':0})
    hb=s.historybuf
    print(f"[{tag}] BEFORE: count={hb.count} ynum={hb.ynum} xnum={hb.xnum} RSS={rss_kb()}KB")
    milestones=[2000,2048,2100,4096,6144,8192,9999,10000]; snap={}
    for i in range(20000):
        s.draw("X"*40); s.linefeed(); s.carriage_return()
        if hb.count in milestones and hb.count not in snap: snap[hb.count]=rss_kb()
    for m in milestones:
        if m in snap:
            exp=min(-(-m//2048), -(-hb.ynum//2048))
            print(f"    count={m:6d} RSS={snap[m]:8d}KB expected_segments={exp}")
    print(f"[{tag}] AFTER 20000 lines: count={hb.count} ynum={hb.ynum} RSS={rss_kb()}KB")
    return hb.count, hb.ynum
c1,y1=run("RUN1"); c2,y2=run("RUN2")
print("STABLE=", c1==c2 and y1==y2)
```

### 3.3 Observed output (complete, unedited; stable across 2 runs)

```
[RUN1] BEFORE: count=0 ynum=10000 xnum=80 RSS=26332KB
    count=  2000 RSS=   31476KB expected_segments=1
    count=  2048 RSS=   31596KB expected_segments=1
    count=  2100 RSS=   31736KB expected_segments=2
    count=  4096 RSS=   36728KB expected_segments=2
    count=  6144 RSS=   41864KB expected_segments=3
    count=  8192 RSS=   46996KB expected_segments=4
    count=  9999 RSS=   51524KB expected_segments=5
    count= 10000 RSS=   51528KB expected_segments=5
[RUN1] AFTER 20000 lines: count=10000 ynum=10000 RSS=51528KB
[RUN2] BEFORE: count=0 ynum=10000 xnum=80 RSS=26624KB
    count=  2000 RSS=   31540KB expected_segments=1
    count=  2048 RSS=   31652KB expected_segments=1
    count=  2100 RSS=   31876KB expected_segments=2
    count=  4096 RSS=   36780KB expected_segments=2
    count=  6144 RSS=   41908KB expected_segments=3
    count=  8192 RSS=   47036KB expected_segments=4
    count=  9999 RSS=   51572KB expected_segments=5
    count= 10000 RSS=   51572KB expected_segments=5
[RUN2] AFTER 20000 lines: count=10000 ynum=10000 RSS=51572KB
STABLE= True
```

**Interpretation:** RSS grows from ~26.3 MB to ~51.5 MB — about **25 MB in 5 discrete ~5.0 MiB steps** = 5 segments — and `count` **saturates at `ynum = 10000`** and never exceeds it, even though 20000 lines were fed (the extra 10000 lines were evicted).

### 3.4 Deeper evidence — `strace` (exactly 5 segment `mmap`s: 1 eager + 4 lazy)

Bracketing the run with `=MARK` writes and filtering the segment‑sized `mmap`s (segment bytes at 80 cols = `2048*(80*32+1)` = 5,244,928, which glibc rounds up to a 5,255,168‑byte `mmap`):

```
121:=MARK before_create_screen
123:mmap(NULL, 5255168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7dbb7106e000
124:=MARK after_create_screen count=0
125:mmap(NULL, 5255168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7dbb70b6b000
126:mmap(NULL, 5255168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7dbb70668000
127:mmap(NULL, 5255168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7dbb70165000
128:mmap(NULL, 5255168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7dbb6fc62000
129:=MARK after_burst count=8977 ynum=10000
```

Exactly **one** segment `mmap` fires between `before_create_screen` and `after_create_screen count=0` — that is **segment 0, carved eagerly at `kitty/history.c:127` while `count` is still 0** — and **four more** fire during the burst. That is the "1 eager + 4 lazy" pattern of Divergence 2, made concrete.

### 3.5 Deeper evidence — `gdb` (`add_segment` fires 4× during the burst)

`add_segment` is a `static` function that LTO builds emit only under the mangled local symbol `add_segment.lto_priv.0` (confirmed with `nm`/`objdump`), so a plain `break add_segment` never resolves. Bootstrapping at the clean symbol `historybuf_add_line` (so the `.so` is mapped) and then `rbreak ^add_segment` binds it. It fires **4 times** during the burst, with the backtrace:

```
>>> add_segment fired
#0  0x00007ffff6e6aff0 in add_segment.lto_priv () from .../kitty/fast_data_types.so
#1  0x00007ffff6e6b774 in init_line.lto_priv () from .../kitty/fast_data_types.so
#2  0x00007ffff6e6bfef in historybuf_add_line () from .../kitty/fast_data_types.so
#3  0x00007ffff6e95796 in linefeed.lto_priv () from .../kitty/fast_data_types.so
#4  0x00000000005746f4 in _PyEval_EvalFrameDefault ()
```

(The eager seg‑0 carve happened inside `create_screen`, *before* the bootstrap breakpoint on `historybuf_add_line` was armed, which is exactly why `gdb` sees 4 rather than 5. Combined with the 5 `mmap`s in §3.4 this closes the loop: 1 eager + 4 lazy = 5.)

### 3.6 Cause→effect (the specific code doing the work)

Each new history line is written through `historybuf_add_line` (`kitty/history.c:286-291`) → `historybuf_push` (`kitty/history.c:275-284`) → `init_line`, which resolves the line's slot via `segment_for()` (`kitty/history.c:36-42`). The carve is triggered by the loop condition inside `segment_for`:

```c
while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
```

i.e. when the requested line's segment index `y / 2048` reaches the current `num_segments` (and the buffer is not already at `ynum`), it calls `add_segment` (`kitty/history.c:17-29`), which does `num_segments += 1`, a **`realloc`** of the segment‑pointer array (`:20`), and one **`calloc`** of a whole segment block (`:25`). Segment 0 pre‑exists from construction (`create_historybuf → add_segment`, `:127`). `count` is capped in `historybuf_push`: `if (self->count == self->ynum) { … } else self->count++;` (`:279-282`) — that `else` is why `count` freezes at `ynum`.

---

## 4. Q2 — Segmented storage ↔ pager ring ("the quiet interaction")

### 4.1 Direct answer (plain reading first)

The two stores are coupled at **exactly one point**: the pager byte‑ring is fed **only when the line ring evicts a line**, i.e. only once `count == ynum`. In `historybuf_push()` (`kitty/history.c:275-284`):

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

`pagerhist_push()` is called at **`kitty/history.c:280`**, *before* the `start_of_data` advance, so it serializes the **oldest** line (the one about to be overwritten). If the pager is disabled (`scrollback_pager_history_size == 0` → `alloc_pagerhist` returns `NULL` at `kitty/history.c:72`), `pagerhist_push()` is a **no‑op** via its NULL‑guard (`kitty/history.c:261`, `if (!ph) return;`).

### 4.2 Command

`ynum = 8` (via `cols=20, lines=5, scrollback=8`), feeding 40 short lines `line0000…line0039`, in two configurations — OFF (`scrollback_pager_history_size = 0`) and ON (`= 4 MiB`). Run twice.

```
PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q2_pager.py
```

### 4.3 Observed output (complete, unedited; stable across 2 runs)

```
=== RUN A ===
[OFF] pager_sz=0 ynum=8 final count=8 final pager_bytes=0
[OFF] pagerhist_as_bytes()=b'' len=0
[ON ] pager_sz=4194304 ynum=8 final count=8 final pager_bytes=364
[ON ] first count==ynum (line-ring FULL): (11, 8, 0)   # (fed_index, count, pager_bytes)
[ON ] first pager_bytes>0 (eviction feeds ring): (12, 8, 13)
[ON ] pager head: '\x1b[mline0000\r\n\x1b[mline0001\r\n\x1b[mline0002\r\n\x1b[mline0003\r\n\x1b[mline0004\r\n\x1b[mline0005\r\n\x1b['
=== RUN B (stability) ===
[OFF] pager_sz=0 ynum=8 final count=8 final pager_bytes=0
[OFF] pagerhist_as_bytes()=b'' len=0
[ON ] pager_sz=4194304 ynum=8 final count=8 final pager_bytes=364
[ON ] first count==ynum (line-ring FULL): (11, 8, 0)   # (fed_index, count, pager_bytes)
[ON ] first pager_bytes>0 (eviction feeds ring): (12, 8, 13)
[ON ] pager head: '\x1b[mline0000\r\n\x1b[mline0001\r\n\x1b[mline0002\r\n\x1b[mline0003\r\n\x1b[mline0004\r\n\x1b[mline0005\r\n\x1b['
STABLE final_pager_bytes: True | A= 364 B= 364
first_full A vs B: (11, 8, 0) (11, 8, 0)
first_pager A vs B: (12, 8, 13) (12, 8, 13)
```

### 4.4 Cause→effect

- **OFF:** the line ring still fills and evicts (`count` reaches `ynum = 8`), but the pager stays **empty (`b''`)** the whole time. That is the NULL‑guard at `pagerhist_push` (`kitty/history.c:261`) short‑circuiting every call — the coupling exists structurally but transmits nothing because `self->pagerhist == NULL` (from `alloc_pagerhist` returning `NULL` at `:72` when the size is 0).
- **ON:** `pager_bytes` stays `0` while `count` climbs to `ynum` (observed `first count==ynum` at **fed index 11**, pager still 0). The instant the ring is full and the *next* line arrives (**fed index 12**), the oldest line `line0000` is serialized into the ring — `0 → 13 bytes`. Each evicted line becomes exactly **13 bytes**: `ESC[m` (3) + `line0000` (8) + `\r\n` (2). The ring then keeps accreting evicted lines; with 40 lines fed and the first eviction at index 12, that is **28 evicted lines × 13 = 364 bytes** — matching the observed final `364`.
- **Eviction order is OLDEST‑first**, visible directly in the pager head: `line0000, line0001, line0002, …`. That ordering is a direct consequence of `pagerhist_push` running *before* `start_of_data` advances (`kitty/history.c:280` then `:281`), so it always serializes the slot at the current `start_of_data` — the oldest surviving line.

The per‑line serialization (`ESC[m` prefix, then content, then `\r`/`\r\n`) is produced by `pagerhist_push` at `kitty/history.c:259-273`; the `\r` vs `\r\n` distinction is examined in Q5.

*(Scale note: a larger reference run feeding ~60 lines reports 624 bytes; feeding 40 lines here yields 364. Both reconcile to the same **13 bytes per evicted line** — 48×13=624, 28×13=364 — so this is a scale difference, not a behavioral one.)*

---

## 5. Q3 — Smooth transition vs. "hesitation"

### 5.1 Direct answer (plain reading first)

It is **not perfectly smooth** — the growth is mostly cheap per‑line increments, **punctuated by discrete allocation/copy spikes** at two kinds of boundary, and a hard abort at the true extreme:

1. **Segment carve (every 2048 lines):** one `realloc` of the segment‑pointer array (`kitty/history.c:20`) + one `calloc` of a full ~5 MiB block (`kitty/history.c:25`). (Observed as the 5 discrete RSS steps / 5 `mmap`s in Q1.)
2. **Pager‑ring extend:** the ring starts at `MIN(1 MiB, configured)` (`kitty/history.c:66-67`). When incoming bytes exceed free space, `pagerhist_write_bytes` (`kitty/history.c:218-225`) calls `pagerhist_extend` (`:223`); `pagerhist_extend` (`kitty/history.c:89-101`) allocates a new ring `MIN(maximum_size, capacity + MAX(1 MiB, minsz))` (`:93`) and performs a **full `ringbuf_copy`** of the used bytes (`:97`) before freeing the old ring — a copy‑cost spike that grows with occupancy. Once `capacity >= maximum_size` it returns `false` (`:92`) and the ring simply overwrites its oldest bytes in place.
3. **OOM = hard abort (NOT graceful).** `add_segment` calls `fatal()` if `realloc`/`calloc` return `NULL` (`kitty/history.c:21, 26`); `segment_for` calls `fatal()` on an out‑of‑bounds line index (`kitty/history.c:40`). **This path is labeled *inferred*** — it is deliberately **not** triggered because `fatal()` aborts the process.

### 5.2 Command

`ynum = 8`, pager `maximum_size = 4 MiB`, 100 columns, feeding **60000** ~90‑char lines. Run twice.

```
PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q3_pager_growth.py
```

### 5.3 Observed output (complete, unedited; stable across 2 runs)

```
[RUN1] pager maximum_size=4194304 bytes (~4.00 MiB); starts at MIN(1MiB,max)=1048576 bytes; ynum=8
    fed= 10000 count=8 pager_bytes=   948860 (~0.905 MiB) RSS=27492KB
    fed= 20000 count=8 pager_bytes=  1898860 (~1.811 MiB) RSS=28420KB
    fed= 30000 count=8 pager_bytes=  2848860 (~2.717 MiB) RSS=29348KB
    fed= 40000 count=8 pager_bytes=  3798860 (~3.623 MiB) RSS=30276KB
    fed= 50000 count=8 pager_bytes=  4194304 (~4.000 MiB) RSS=30664KB
    fed= 60000 count=8 pager_bytes=  4194304 (~4.000 MiB) RSS=34756KB
[RUN1] FINAL pager_bytes=4194304 -> PLATEAU at maximum_size => retention bound
[RUN2] pager maximum_size=4194304 bytes (~4.00 MiB); starts at MIN(1MiB,max)=1048576 bytes; ynum=8
    fed= 10000 count=8 pager_bytes=   948860 (~0.905 MiB) RSS=31640KB
    fed= 20000 count=8 pager_bytes=  1898860 (~1.811 MiB) RSS=34524KB
    fed= 30000 count=8 pager_bytes=  2848860 (~2.717 MiB) RSS=38428KB
    fed= 40000 count=8 pager_bytes=  3798860 (~3.623 MiB) RSS=39644KB
    fed= 50000 count=8 pager_bytes=  4194304 (~4.000 MiB) RSS=40028KB
    fed= 60000 count=8 pager_bytes=  4194304 (~4.000 MiB) RSS=40028KB
[RUN2] FINAL pager_bytes=4194304 -> PLATEAU at maximum_size => retention bound
STABLE final: True | RUN1= 4194304 RUN2= 4194304
```

The pager grows ~0.95 MiB per 10000 lines and **plateaus at exactly `maximum_size` = 4,194,304 bytes** — after which it retains at most that many bytes and overwrites the oldest.

### 5.4 Deeper evidence — `strace` of the ring extends (the copy‑on‑grow spikes)

`ringbuf_new` does `malloc(capacity+1)` (`3rdparty/ringbuf/ringbuf.c`), which for ≥1 MiB is served by `mmap`. Bracketed with `=MARK` writes:

```
=MARK before_create
mmap(NULL, 1052672, ...)   <- RING init 1MiB (MIN(1MiB,max)); alloc_pagerhist -> ringbuf_new
mmap(NULL, 6565888, ...)   <- history SEGMENT 0 @100 cols (eager): 2048*(100*32+1)=6,555,648 + overhead
mmap(NULL, 1052672, ...)   <- RING init 1MiB again (reset -> pagerhist_clear)
=MARK after_create
mmap(NULL, 2101248, ...)   <- RING extend ->2MiB + full ringbuf_copy
mmap(NULL, 3149824, ...)   <- RING extend ->3MiB + full ringbuf_copy
mmap(NULL, 4198400, ...)   <- RING extend ->4MiB (=maximum_size) + full ringbuf_copy
mmap(NULL, 4198400, ...)   <- pagerhist_as_bytes() read-out buffer (PyBytes of used size)
=MARK after_burst count=8 pager_bytes=4194304
```

There is **no allocation beyond 4 MiB**, confirming the `if (buffer_size >= ph->maximum_size) return false;` cap at `kitty/history.c:92` and the plateau in §5.3. Note `pagerhist_extend` has **no standalone symbol** — LTO inlines it into `pagerhist_write_bytes` — which is precisely why the ring extends are observed via `strace` on the `mmap` sizes rather than a `gdb` breakpoint on the function.

### 5.5 Cause→effect

The two observed spike families map directly to code: the ~5 MiB segment `calloc` at `kitty/history.c:25` (fired 4× lazily + 1× eagerly in Q1) and the ring `ringbuf_copy` at `kitty/history.c:97` (fired 3× here: 1→2, 2→3, 3→4 MiB). The copy cost is `O(bytes_used)` and therefore *increases* with each extend — the "hesitation" is largest just before the plateau. Beyond the plateau there are no more allocations at all: the ring degrades to in‑place overwrite (`pagerhist_extend` returns `false` at `:92`). The only non‑graceful behavior is the `fatal()` abort on allocation failure (`:21, :26, :40`), which is *inferred* and not exercised.

---

## 6. Q4 — Concurrent scroll while data arrives at full speed

### 6.1 Direct answer (plain reading first)

Ingest and scroll re‑anchoring are on the **same logical path — there is NO lock or thread contention on `HistoryBuf`.** Each new line that scrolls into history increments `history_line_added_count` inside the `INDEX_UP` macro (`kitty/screen.c:1552-1568`, specifically the `self->history_line_added_count++;` at `:1559`). While the user is scrolled back (`scrolled_by > 0`), the viewport is re‑anchored **at render time** by

```c
if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);
```

present identically at **`kitty/screen.c:2716`** (`screen_update_only_line_graphics_data`) and **`:2761`** (`screen_update_cell_data`); `history_line_added_count` is then reset to 0 by `screen_reset_dirty` (`kitty/screen.c:2598-2601`). The scroll *amount* is chosen by `screen_history_scroll` (`kitty/screen.c:4091`): `SCROLL_LINE → 1`, `SCROLL_PAGE → lines-1`, `SCROLL_FULL → count`.

### 6.2 Command

`cols=80, lines=5, scrollback=10000`; ~200 lines pre‑loaded, then for each mode: scroll up, feed a **300‑line burst with no render**, then render once. Plus a saturation case at `scrollback=30`. The render step uses the exposed `update_only_line_graphics_data()` method, which runs the `:2716` re‑anchor. Run twice.

```
PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q4_scroll.py
```

### 6.3 Observed output (complete, unedited; stable across 2 runs)

```
ScrollType enum values: SCROLL_LINE=-999999 SCROLL_PAGE=-999998 SCROLL_FULL=-999997  (kitty/screen.h:13)
[SCROLL_LINE] after scroll(SCROLL_LINE,upwards=True): scrolled_by=1  (count=196)
[SCROLL_LINE] burst of 300 more WHILE scrolled (no render yet): scrolled_by=1 (FROZEN==1) count=496 hlac_inferred=300
[SCROLL_LINE] after render (re-anchor): scrolled_by=301 = MIN(1+300, count=496) ; hlac reset=0

[SCROLL_PAGE] after scroll(SCROLL_PAGE,upwards=True): scrolled_by=4  (count=196)
[SCROLL_PAGE] burst of 300 more WHILE scrolled (no render yet): scrolled_by=4 (FROZEN==4) count=496 hlac_inferred=300
[SCROLL_PAGE] after render (re-anchor): scrolled_by=304 = MIN(4+300, count=496) ; hlac reset=0

[SCROLL_FULL] after scroll(SCROLL_FULL,upwards=True): scrolled_by=196  (count=196)
[SCROLL_FULL] burst of 300 more WHILE scrolled (no render yet): scrolled_by=196 (FROZEN==196) count=496 hlac_inferred=300
[SCROLL_FULL] after render (re-anchor): scrolled_by=496 = MIN(196+300, count=496) ; hlac reset=0

=== SATURATION (scrollback=30) ===
count=30(==ynum=30 SATURATED) scrolled_by=30(==count, pinned at oldest)
after 50 more + render: count=30(still 30) scrolled_by=30 -- cannot exceed count; oldest content already evicted, view slides off
```

### 6.4 Cause→effect

- **SCROLL_LINE** sets `scrolled_by = 1` (`screen_history_scroll` maps `SCROLL_LINE → amt=1`, `kitty/screen.c:4094`). **SCROLL_PAGE** sets `4` (`lines-1 = 5-1`, `:4097`). **SCROLL_FULL** sets `196` (`= count`, `:4100`).
- During the 300‑line burst **without a render**, `scrolled_by` stays **frozen** at its pre‑burst value while `count` grows (196 → 496) and `history_line_added_count` silently accumulates to 300. Nothing re‑anchors because the re‑anchor only runs inside the render functions.
- The next render folds `hlac` in via `MIN(scrolled_by + hlac, count)`: `1+300→301`, `4+300→304`, `196+300→496`. This keeps the viewport pinned to the *same old line* as new data arrives — the user does not get yanked around.
- **Saturation:** once `count` has hit its ceiling (`ynum = 30`), `scrolled_by` is clamped by the `MIN(…, count)` term and **cannot grow past `count`**. After 50 more lines + render, `count` is still 30 and `scrolled_by` is still 30 — the oldest content has already been evicted, so the anchored view "slides off" the bottom of history.

This is cause→effect through `history_line_added_count` accumulated between renders, **not** true multi‑threaded contention: there is no lock around `HistoryBuf`, and the same code path performs both the push and the re‑anchor. (`history_line_added_count` is not exposed to Python, so its value is *inferred* from the number of lines fed during the frozen window — 300 — which the post‑render `scrolled_by` arithmetic confirms exactly.)

---

## 7. Q5 — Runtime observation of wrapping & retention

### 7.1 Direct answer (plain reading first)

- **Wrapping:** continuation is tracked by the per‑cell `next_char_was_wrapped` attribute. In `pagerhist_push` (`kitty/history.c:268-271`), a physical row whose last cell has `next_char_was_wrapped == true` is terminated with **`\r` only**; a logical (unwrapped) line‑end gets **`\r\n`**. A column change sets `rewrap_needed` (`kitty/history.c:608`); the pager bytes are reflowed lazily by `pagerhist_rewrap_to()` (`kitty/history.c:392-432`, triggered in `pagerhist_as_bytes` at `:467`), and the line ring itself is reflowed by `historybuf_rewrap` (`kitty/history.c:595`, invoked from screen resize at `kitty/screen.c:221`).
- **Retention:** the line ring holds at most **`ynum`** lines; the pager ring holds at most **`maximum_size`** bytes.

### 7.2 The serialization condition (source)

```c
    pagerhist_write_bytes(ph, (const uint8_t*)"\x1b[m", 3);
    if (pagerhist_write_ucs4(ph, as_ansi_buf->buf, as_ansi_buf->len)) {
        char line_end[2]; size_t num = 0;
        line_end[num++] = '\r';
        if (!l.gpu_cells[l.xnum - 1].attrs.next_char_was_wrapped) line_end[num++] = '\n';
        pagerhist_write_bytes(ph, (const uint8_t*)line_end, num);
    }
```
(`kitty/history.c:266-272` — `\r` is unconditional at `:269`; `\n` is appended only when the last cell is *not* a wrap continuation, `:270`.)

### 7.3 Command + observed output — Q5a wrapping (complete, unedited; stable 2×)

```
PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q5_wrap.py
```

Drawing a single 25‑char logical line at `cols=10` (so it wraps into 3 physical rows `ABCDEFGHIJ | KLMNOPQRST | UVWXY`), pager ON, then evicting it with short unwrapped lines:

```
=== Q5a wrapping (cols=10, draw 25 chars -> 3 physical rows) ===
pagerhist_as_text()='\x1b[mABCDEFGHIJ\r\x1b[mKLMNOPQRST\r\x1b[mUVWXY\r\n\x1b[mz0\r\n\x1b[mz1\r\n\x1b[mz2\r\n\x1b[mz3\r\n\x1b[mz4\r\n\x1b[mz5\r\n\x1b[mz6\r\n\x1b[m'
  -> wrapped rows end '\r' only; unwrapped row ends '\r\n'  (history.c:269-270)
```

The two wrapped physical rows (`ABCDEFGHIJ`, `KLMNOPQRST`) end with **`\r` only**; the logical line‑end `UVWXY` and each short line `z0…z6` end with **`\r\n`** — exactly the `:270` condition.

### 7.4 Command + observed output — Q5b rewrap on resize (complete, unedited; stable 2×)

```
PYTHONPATH="$(pwd)" python3 /tmp/blitzy_adhoc/q5b_pager_rewrap.py
```

```
=== Q5b line-ring rewrap on resize (20 wrapped logical lines @10 cols) ===
BEFORE resize: count=56 xnum=10
AFTER resize->40 cols: count=19 xnum=40   (widening reflows: fewer physical rows)
AFTER resize->5  cols: count=96 xnum=5    (narrowing: more physical rows)
=== Q5b-2 pager-ring rewrap (ynum=5 so eviction fills pager) ===
BEFORE resize xnum=10: pager='\x1b[mABCDEFGHIJ\r\x1b[mKLMNOPQRST\r\x1b[mUVWXY\r\n\x1b[mABCDEFGHIJ\r\x1b[mKLMNO' len=646
AFTER  resize->40 xnum=40: pager='\x1b[mABCDEFGHIJ\x1b[mKLMNOPQRST\x1b[mUVWXY\n\x1b[mABCDEFGHIJ\x1b[mKLMNOPQRS'    len=595  reflowed=True
=== Q5c retention: line ring never exceeds ynum ===
fed 500, count=40 == ynum=40 (line ring never exceeds ynum)
```

> Implementation note: `screen.resize()` **replaces** the `HistoryBuf` object (a new buffer is built by `historybuf_rewrap`, `kitty/screen.c:221`). The script therefore re‑reads `s.historybuf` after each resize; a stale reference to the pre‑resize object shows an unchanged `xnum`/`count` and is a measurement error, not a behavior.

### 7.5 Cause→effect

- **Line‑ring rewrap:** widening from 10 → 40 columns merges wrapped physical rows back into fewer rows, so `count` drops **56 → 19**; narrowing 10 → 5 splits them into more rows, so `count` rises to **96**. This is `historybuf_rewrap` → `rewrap_inner` (`kitty/rewrap.h`) recomputing physical rows for the new width.
- **Pager‑ring rewrap:** on resize, `historybuf_rewrap` sets `other->pagerhist->rewrap_needed = true` (`kitty/history.c:608`); the next `pagerhist_as_bytes()` sees the flag and calls `pagerhist_rewrap_to(self, self->xnum)` (`:467`). At the wider width the previously‑wrapped rows `ABCDEFGHIJ`+`KLMNOPQRST` now fit on one logical line, so the intermediate **`\r` wrap‑markers are consumed** (visible in the `AFTER` bytes: the `\r` between the two segments is gone, only the logical `\n` remains) and the byte count changes **646 → 595**.
- **Retention:** feeding 500 lines into a `ynum = 40` buffer leaves `count == 40` — the line ring **never exceeds `ynum`** (the `else self->count++;` cap at `kitty/history.c:282`). The pager‑ring's complementary bound is `maximum_size`, demonstrated by the Q3 plateau at 4,194,304 bytes.

*(My observed rewrap counts — 56→19→96, pager 646→595 — differ in magnitude from other reference runs that used slightly different content/line counts, e.g. 38→19→78 and 333→310; the **direction and mechanism are identical** and my values are internally consistent and stable across two runs.)*

---

## 8. Segment‑size math (derived from code; confirmed by RSS / `mmap`)

Each segment block is a single `calloc` (`kitty/history.c:25`) sized:

```
segment_bytes = SEGMENT_SIZE * (xnum*sizeof(CPUCell) + xnum*sizeof(GPUCell) + sizeof(LineAttrs))
              = 2048 * (xnum*12 + xnum*20 + 1)
              = 2048 * (xnum*32 + 1)
```

using `sizeof(CPUCell) == 12` (`kitty/data-types.h:228`), `sizeof(GPUCell) == 20` (`kitty/data-types.h:221`), and `LineAttrs` = 1 byte (`kitty/data-types.h:231-239`, a union of a bitfield struct with a `uint8_t`).

| Columns (`xnum`) | `2048*(xnum*32+1)` | ≈ MiB | Observed `mmap` |
|---|---|---|---|
| 80  | `2048*2561` = **5,244,928 B** | 5.00 | 5,255,168 B (calloc + glibc overhead) — Q1 §3.4 |
| 100 | `2048*3201` = **6,555,648 B** | 6.25 | 6,565,888 B — Q3 §5.4 |

Segment count = `ceil(ynum / 2048)`. For `ynum = 10000` that is **5**, matching the 5 RSS steps and 5 `mmap`s in Q1. The ~5.0 MiB per‑step RSS growth observed in §3.3 is the runtime confirmation of the 80‑column figure.

---

## 9. Verified `file:line` anchor appendix

All anchors confirmed against HEAD `815df1e21`.

**`kitty/history.c`:** `SEGMENT_SIZE` `:15`; `add_segment` `:17-29` (`num_segments++` `:19`, `realloc` `:20`, `fatal` `:21`, `calloc` `:25`, `fatal` `:26`); `segment_for` `:36-42` (lazy carve loop `:39`, `fatal` `:40`); `attrptr` `:61-64`; `initial_pagerhist_ringbuf_sz` `:66-67`; `alloc_pagerhist` `:69-80` (NULL‑if‑0 `:72`, `ringbuf_new` `:76`, `maximum_size` `:78`); `pagerhist_extend` `:89-101` (cap `:92`, newsz `:93`, `ringbuf_copy` `:97`); `create_historybuf` eager `add_segment` `:126-127`, `alloc_pagerhist` `:130`; `pagerhist_write_bytes` `:218-225` (extend‑on‑overflow `:223`); `pagerhist_push` `:258-273` (NULL‑guard `:261`, `ESC[m` `:266`, `\r` `:269`, `\n`‑if‑unwrapped `:270`); `historybuf_push` `:275-284` (**`pagerhist_push` at `:280`**, `start_of_data` advance `:281`, `count++` else `:282`); `historybuf_add_line` `:286-291`; `pagerhist_rewrap_to` `:392-432`; `historybuf_rewrap` `:595` (`rewrap_needed = true` `:608`); `pagerhist_as_bytes` `:461` (rewrap trigger `:467`); `pagerhist_as_text` `:486`; exposed `PyMemberDef` `xnum`/`ynum`/`count` only `:555-559`.

**`kitty/data-types.h`:** `sizeof(GPUCell)==20` `:221`; `sizeof(CPUCell)==12` `:228`; `LineAttrs` union `:231-239`; `HistoryBufSegment` `:262-266`; `PagerHistoryBuf` `:268-272`; `HistoryBuf` `:282-290` (`pagerhist` member at **`:287`**).

**`kitty/screen.c`:** `alloc_historybuf` `:130`; `INDEX_UP` macro `:1552`, `historybuf_add_line` `:1558`, `history_line_added_count++` `:1559`; `screen_reset_dirty` (resets hlac) `:2598-2601`; `scrolled_by` re‑anchor `:2716` **and** `:2761`; `historybuf_rewrap` on resize `:221`; `screen_history_scroll` `:4091` (`SCROLL_LINE`→1 `:4094`, `SCROLL_PAGE`→`lines-1` `:4097`, `SCROLL_FULL`→`count` `:4100`); `scroll` wrapper `:4121`.

**`kitty/screen.h`:** `ScrollType` enum `:13` (`SCROLL_LINE=-999999, SCROLL_PAGE, SCROLL_FULL`).

**`kitty/options/definition.py`:** `scrollback_lines` = `2000` `:372`; `scrollback_pager_history_size` = `0` `:406-407`. **`kitty/options/utils.py`:** MB→bytes parse `:564`.

**`3rdparty/ringbuf/ringbuf.h` / `.c`:** `ringbuf_new` `:41`; `ringbuf_reset` `:65`; `ringbuf_capacity` `:73`; `ringbuf_bytes_free` `:80`; `ringbuf_bytes_used` `:87`; `ringbuf_memcpy_into` `:154`; `ringbuf_copy` `:252`; buffer `malloc` in `ringbuf.c`.

**`kitty_tests/__init__.py`:** `parse_bytes` `:30`; `Callbacks` `:39`; harness pager default (`scrollback_pager_history_size = 1024`) `:224`; `create_screen` `:237`; `Screen(...)` `:240`. **`kitty_tests/datatypes.py`:** `test_historybuf` `:487`; `HistoryBuf(3000,5)` `:501`. **`kitty/window.py`:** `pagerhist()` `:355-356`; `as_text()` `:363`; `cmd_output()` `:457`.

---

## 10. Coverage pass

Confirming every named item in the prompt is addressed with observed evidence:

| Item | Where addressed | Observed value / evidence |
|---|---|---|
| **Q1** segment fill / stretch / carve | §3 | `count` 0→10000 (saturates); RSS 5×~5 MiB; 5 `mmap`s (1 eager+4 lazy); `gdb` 4× |
| **Q2** segmented ↔ pager coupling at `count==ynum` | §4 | OFF `b''`; ON 0→13→364 B, oldest‑first |
| **Q3** smooth vs. hesitation | §5 | segment `calloc`; ring extend 1→2→3→4 MiB + `ringbuf_copy`; plateau 4,194,304; `fatal()` OOM (*inferred*) |
| **Q4** concurrent scroll (LINE / PAGE / FULL + saturation) | §6 | 1→301, 4→304, 196→496; saturation pinned at 30 |
| **Q5** wrapping via `next_char_was_wrapped` + retention by `ynum` / `maximum_size` | §7 | `\r` vs `\r\n`; rewrap 56→19→96 & 646→595; `count==ynum=40` |

**Named mechanisms/functions/flags — each named explicitly above:** `SEGMENT_SIZE` (§3,§8), `add_segment` (§3), `segment_for` (§3), `historybuf_push` (§3,§4), `pagerhist_push` (§4), `pagerhist_extend` (§5), `alloc_pagerhist` (§4,§5), `pagerhist_rewrap_to` (§7), `historybuf_add_line` (§1,§3), `INDEX_UP` (§1,§6), `history_line_added_count` (§6), `scrolled_by` (§6), `screen_history_scroll` (§6), `scrollback_lines` (§9), `scrollback_pager_history_size` (§2,§4), `ringbuf_copy` (§5), `ringbuf_new` (§5), `next_char_was_wrapped` (§7).

**Conditions exercised:** pager OFF vs ON (§4); single‑segment (≤2048) trivially vs multi‑segment (≫2048, `ynum=10000`) (§3); 2048‑boundary crossings (§3); ring‑full growth (§5); concurrent scroll during burst (§6); column‑resize rewrap (§7); before/during/after snapshots of `count`/`ynum`/`xnum`/RSS/pager length (§3–§7).

**Evidence discipline:** every behavioral claim above is backed by its own unedited command output; the **only** *inferred* statement is the `fatal()` OOM abort (§5), which is deliberately not triggered because it would abort the process.
