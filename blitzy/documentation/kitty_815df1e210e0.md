# kitty scrollback `HistoryBuf` under heavy load — an empirical investigation

**Repository:** `kovidgoyal/kitty` · **Branch:** `kitty_815df1e210e0` · **HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

This document answers three questions about how kitty's scrollback history buffer (`HistoryBuf`) behaves when a program prints a very large amount of output. **Every number below was produced by building kitty's C extension and driving its real output-ingestion path (`parse_bytes → Screen → historybuf_add_line`) with temporary scripts, then reading process memory from `/proc/self/status` and the buffer's own `HistoryBuf.count`.** The full source of every temporary script is reproduced below (see *Temporary observation scripts*); the scripts lived in `/tmp` (outside the repository) and were deleted after use, so no tracked file in the repository is modified.

---

## TL;DR (direct answers)

- **Q1 — Memory under load.** At the **default** configuration (`scrollback_lines 2000`) memory is **bounded**: after the buffer fills, resident memory attributable to scrollback **plateaus at ≈5.3 MB and never grows again**, no matter whether you print 5,000 or 500,000 lines. The line count `HistoryBuf.count` climbs to `ynum = 2000` and then stays there because the buffer is a **ring** that overwrites its oldest line (`historybuf_push`, [`kitty/history.c:275-283`](#cit-history)). Only when scrollback is configured very large (or "infinite") does memory grow without bound — e.g. **≈123 MiB after 50,000 lines** with infinite scrollback.
- **Q2 — Responsiveness during concurrent scroll + output.** The model/output path is **not** the bottleneck: ingesting 10,000 lines takes **≈10 ms** (~1 ms per 1,000 lines), and a scroll command is an **O(1) integer update** to `scrolled_by` (`screen_history_scroll`, [`kitty/screen.c:4091`](#cit-screen)). Responsiveness under concurrent output comes from kitty's structure: a dedicated **I/O thread** only `read()`s child bytes and wakes the main loop — coalesced by `input_delay` ([`kitty/child-monitor.c:291`](#cit-childmon), [`:1562-1569`](#cit-childmon)) — while the **main loop parses *then* renders sequentially** (`parse_input` → `render`, [`kitty/child-monitor.c:1236-1237`](#cit-childmon)) under `input_delay`/`repaint_delay` pacing. There is **no separate render thread**. So a burst of output is coalesced into periodic repaints rather than stalling input, and vice-versa. A **view re-anchor** keeps the scrolled-back viewport pinned to the same content as new lines arrive ([`kitty/screen.c:2761`](#cit-screen)); that step runs in the **render step of the main loop**, so it cannot be timed in a headless model-only harness — reported honestly below and explained from the source.
- **Q3 — Buffer boundaries / when new storage is allocated.** New backing storage is allocated **one ≈5 MiB segment at a time**, at each **2,048-line boundary**, by `add_segment` via `segment_for` ([`kitty/history.c:17-42`](#cit-history)). **Yes, this is observable through memory monitoring:** each allocation appears as a discrete **≈5 MiB step in *virtual* memory (`VmSize`/`VmData`)** exactly when `count` crosses a multiple of `SEGMENT_SIZE = 2048`, while **resident memory (`VmRSS`) rises approximately linearly** because Linux commits the freshly `calloc`'d pages lazily (demand paging).

---

## Environment & methodology

### Environment (as actually used for these measurements)

| Item | Value |
|------|-------|
| Repository path (shell) | `/tmp/blitzy/kitty/blitzy-11eb97ad-bc12-4d85-992e-5f365a22cf0b_dbec52` |
| Branch / HEAD commit | `blitzy-11eb97ad-bc12-4d85-992e-5f365a22cf0b` / `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Python | CPython **3.13.7** (`/root/kitty-venv/bin/python`) |
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

**Build/environment verification (command + complete output).** The `git status --porcelain` command emits no lines when the tree is clean, so a trailing marker (`echo status_exit=$?`) is printed to prove the command ran and produced empty output:

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ git status --porcelain; echo "status_exit=$?"
status_exit=0

$ ls -l kitty/fast_data_types.so
-rwxr-xr-x 1 root root 1253792 Jul  6 22:07 kitty/fast_data_types.so

$ /root/kitty-venv/bin/python --version
Python 3.13.7

$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ PYTHONPATH=. /root/kitty-venv/bin/python -c "from kitty.fast_data_types import Screen, HistoryBuf; print('ok')"
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

kitty's `BaseTest.set_options` defaults `scrollback_pager_history_size` to **1024** ([`kitty_tests/__init__.py:223-224`](#cit-harness)), which is **not** kitty's shipped default. To measure the true canonical default (**pager history OFF**), every script below installs options via a local `set_opts` helper that forces `scrollback_pager_history_size: 0` unless a test explicitly overrides it. Input is fed in modest chunks (1,000 lines per `parse_bytes` call) and each configuration runs in a **fresh process** to avoid transient-input and baseline-carryover artifacts.

---

## Temporary observation scripts (full source)

The measurements below are produced by three temporary Python scripts and one C ABI probe, all under `/tmp` (outside the repository). Their **complete source is reproduced here so every displayed command is reproducible as written** from a fresh checkout: create each file with the `cat > … <<'EOF'` command shown, then run the commands in the Q1/Q2/Q3 sections with `PYTHONPATH=.` and `/root/kitty-venv/bin/python`. They are removed afterward (see *Read-only scope & reproducibility*), leaving the repository unchanged.

### `/tmp/obs_scrollback.py` — backs Q1, infinite, alt-screen, and pager runs

```
$ cat > /tmp/obs_scrollback.py <<'PYEOF'
#!/usr/bin/env python3
# Temporary observation script (lives in /tmp, outside the repo; deleted after use).
# Drives kitty's REAL output-ingestion path: parse_bytes -> Screen -> historybuf_add_line.
# Usage: PYTHONPATH=. python3 /tmp/obs_scrollback.py {q1|infinite|altscreen|pager off|pager on}
import sys

from kitty.fast_data_types import Screen, set_options
from kitty.options.types import Options, defaults
from kitty.options.parse import merge_result_dicts
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty_tests import Callbacks, parse_bytes

COLS, LINES = 80, 24

def vmrss_kb():
    with open('/proc/self/status') as f:
        for line in f:
            if line.startswith('VmRSS:'):
                return int(line.split()[1])
    return -1

def set_opts(pager_size=0):
    final = {'scrollback_pager_history_size': pager_size, 'click_interval': 0.5}
    opts = Options(merge_result_dicts(defaults._asdict(), final))
    finalize_keys(opts, {})
    finalize_mouse_mappings(opts, {})
    set_options(opts)
    return opts

def new_screen(scrollback, pager_size=0):
    set_opts(pager_size)
    cb = Callbacks()
    return Screen(cb, LINES, COLS, scrollback, 10, 20, 0, cb)

def feed(screen, nlines, chunk=1000):
    # Feed nlines newline-terminated lines through the real VT parser, in modest chunks
    # (avoids a huge transient input buffer inflating RSS).
    done = 0
    while done < nlines:
        k = min(chunk, nlines - done)
        parse_bytes(screen, (b'X\r\n' * k))
        done += k

def q1():
    s = new_screen(2000)
    base = vmrss_kb()
    print("# Q1 default scrollback: ynum=%d cols=%d lines=%d" % (s.historybuf.ynum, COLS, LINES))
    print("%10s %7s %9s %8s" % ("lines_fed", "count", "VmRSS_kB", "dRSS_kB"))
    print("%10d %7d %9d %8d" % (0, s.historybuf.count, base, 0))
    fed = 0
    for target in (1000, 2000, 5000, 50000, 200000, 500000):
        feed(s, target - fed); fed = target
        r = vmrss_kb()
        print("%10d %7d %9d %8d" % (fed, s.historybuf.count, r, r - base))

def infinite():
    s = new_screen(2**32 - 1)
    base = vmrss_kb()
    print("# Infinite scrollback: ynum=%d" % s.historybuf.ynum)
    print("%10s %8s %9s %8s" % ("lines_fed", "count", "VmRSS_kB", "dRSS_kB"))
    fed = 0
    for target in (1000, 5000, 10000, 25000, 50000):
        feed(s, target - fed); fed = target
        r = vmrss_kb()
        print("%10d %8d %9d %8d" % (fed, s.historybuf.count, r, r - base))

def altscreen():
    s = new_screen(2000)
    print("# Alt-screen gate. ynum=%d" % s.historybuf.ynum)
    print("main start: count=%d" % s.historybuf.count)
    parse_bytes(s, b'\x1b[?1049h')          # enter alternate screen
    feed(s, 20000)
    print("after 20000 lines on ALT screen: count=%d" % s.historybuf.count)
    parse_bytes(s, b'\x1b[?1049l')          # leave alternate screen (back to main)
    feed(s, 20000)
    print("after 20000 lines back on MAIN screen: count=%d" % s.historybuf.count)

def pager(state):
    if state == 'off':
        s = new_screen(2000, pager_size=0)
        print("# Pager history OFF (0). ynum=%d" % s.historybuf.ynum)
    else:
        s = new_screen(2000, pager_size=10 * 1024 * 1024)
        print("# Pager history ON (10MB). ynum=%d" % s.historybuf.ynum)
    feed(s, 10000)
    print("after 10000 lines: count=%d pagerhist_as_text_len=%d" % (
        s.historybuf.count, len(s.historybuf.pagerhist_as_text())))

if __name__ == '__main__':
    cmd = sys.argv[1]
    if cmd == 'q1': q1()
    elif cmd == 'infinite': infinite()
    elif cmd == 'altscreen': altscreen()
    elif cmd == 'pager': pager(sys.argv[2])
    else: sys.exit("unknown: " + cmd)
PYEOF
```

### `/tmp/obs_q2.py` — backs the Q2 ingest-latency and scroll runs

```
$ cat > /tmp/obs_q2.py <<'PYEOF'
#!/usr/bin/env python3
# Temporary observation script (lives in /tmp, outside the repo; deleted after use).
# Q2: ingest latency of the real parser path + O(1) scroll state updates (scrolled_by),
# plus the headless limitation of the render-path re-anchor.
# Usage: PYTHONPATH=. python3 /tmp/obs_q2.py
import time

from kitty.fast_data_types import (
    Screen, set_options, SCROLL_LINE, SCROLL_PAGE, SCROLL_FULL)
from kitty.options.types import Options, defaults
from kitty.options.parse import merge_result_dicts
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty_tests import Callbacks, parse_bytes

COLS, LINES = 80, 24

def new_screen(scrollback, pager_size=0):
    final = {'scrollback_pager_history_size': pager_size, 'click_interval': 0.5}
    o = Options(merge_result_dicts(defaults._asdict(), final))
    finalize_keys(o, {}); finalize_mouse_mappings(o, {}); set_options(o)
    cb = Callbacks()
    return Screen(cb, LINES, COLS, scrollback, 10, 20, 0, cb)

def feed(screen, nlines, chunk=1000):
    done = 0
    while done < nlines:
        k = min(chunk, nlines - done)
        parse_bytes(screen, (b'X\r\n' * k))
        done += k

def main():
    s = new_screen(100000)
    print("# SCROLL_LINE=%d SCROLL_PAGE=%d SCROLL_FULL=%d" % (SCROLL_LINE, SCROLL_PAGE, SCROLL_FULL))
    t0 = time.perf_counter()
    feed(s, 10000)
    ms = (time.perf_counter() - t0) * 1000.0
    print("ingest 10000 lines: %.3f ms total, %.4f ms per 1000 lines; count=%d" % (ms, ms/10.0, s.historybuf.count))
    print("before any scroll: scrolled_by=%d" % s.scrolled_by)
    s.scroll(SCROLL_LINE, True);  print("scroll(SCROLL_LINE, up):  scrolled_by=%d" % s.scrolled_by)
    s.scroll(SCROLL_LINE, True);  print("scroll(SCROLL_LINE, up):  scrolled_by=%d" % s.scrolled_by)
    s.scroll(SCROLL_PAGE, True);  print("scroll(SCROLL_PAGE, up):  scrolled_by=%d" % s.scrolled_by)
    s.scroll(SCROLL_PAGE, False); print("scroll(SCROLL_PAGE, down):scrolled_by=%d" % s.scrolled_by)
    s.scroll(SCROLL_LINE, False); print("scroll(SCROLL_LINE, down):scrolled_by=%d" % s.scrolled_by)
    s.scroll(SCROLL_FULL, True);  print("scroll(SCROLL_FULL, up):  scrolled_by=%d (count=%d)" % (s.scrolled_by, s.historybuf.count))
    s.scroll(SCROLL_FULL, False); print("scroll(SCROLL_FULL, down):scrolled_by=%d" % s.scrolled_by)
    print()
    s.scroll(SCROLL_LINE, True); s.scroll(SCROLL_LINE, True); s.scroll(SCROLL_LINE, True)  # -> scrolled_by=3
    print("# Re-anchor test: scrolled back to scrolled_by=%d, now feed 5000 more lines (NO render pass)" % s.scrolled_by)
    feed(s, 5000)
    print("after feeding 5000 lines (headless, no render thread): scrolled_by=%d  count=%d" % (s.scrolled_by, s.historybuf.count))

if __name__ == '__main__':
    main()
PYEOF
```

### `/tmp/obs_q3_detail.py` — backs the Q3 fine-grained staircase runs

```
$ cat > /tmp/obs_q3_detail.py <<'PYEOF'
#!/usr/bin/env python3
# Temporary observation script (lives in /tmp, outside the repo; deleted after use).
# Q3: fine-grained sampling around 2048-line SEGMENT_SIZE boundaries to observe each
# add_segment() calloc as a discrete step in virtual memory (VmSize/VmData), while VmRSS
# rises ~linearly (demand paging). Real path: parse_bytes -> Screen -> historybuf_add_line.
# Usage: PYTHONPATH=. python3 /tmp/obs_q3_detail.py
from kitty.fast_data_types import Screen, set_options
from kitty.options.types import Options, defaults
from kitty.options.parse import merge_result_dicts
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty_tests import Callbacks, parse_bytes

COLS, LINES = 80, 24
SEGMENT_SIZE = 2048
# Per-segment calloc size (kitty/history.c:23-25):
#   xnum*2048*sizeof(CPUCell) + xnum*2048*sizeof(GPUCell) + 2048*sizeof(LineAttrs)
#   = 2048*(xnum*12 + xnum*20 + 4)  = 2048*(xnum*32 + 4)   [sizeof(LineAttrs)==4]
PER_SEG = SEGMENT_SIZE * (COLS * 32 + 4)

def mem():
    vm = {}
    with open('/proc/self/status') as f:
        for line in f:
            for k in ('VmRSS:', 'VmSize:', 'VmData:'):
                if line.startswith(k):
                    vm[k[:-1]] = int(line.split()[1])
    return vm

def main():
    final = {'scrollback_pager_history_size': 0, 'click_interval': 0.5}
    o = Options(merge_result_dicts(defaults._asdict(), final))
    finalize_keys(o, {}); finalize_mouse_mappings(o, {}); set_options(o)
    cb = Callbacks()
    s = Screen(cb, LINES, COLS, 100000, 10, 20, 0, cb)
    base = mem()
    print("# ynum=%d SEGMENT_SIZE=%d per-seg=%.7fMiB (%d B)" % (
        s.historybuf.ynum, SEGMENT_SIZE, PER_SEG / 1024.0 / 1024.0, PER_SEG))
    print("%6s %6s %8s %11s %10s" % ("fed", "count", "dRSS_kB", "dVmSize_kB", "dVmData_kB"))
    fed = 0
    while fed < 9000:
        parse_bytes(s, b'X\r\n' * 256)
        fed += 256
        m = mem()
        print("%6d %6d %8d %11d %10d" % (
            fed, s.historybuf.count,
            m['VmRSS'] - base['VmRSS'],
            m['VmSize'] - base['VmSize'],
            m['VmData'] - base['VmData']))

if __name__ == '__main__':
    main()
PYEOF
```

### `/tmp/abi_probe.c` — measures the exact struct sizes that fix per-segment cost

```
$ cat > /tmp/abi_probe.c <<'CEOF'
#include "data-types.h"
#include <stdio.h>
int main(void) {
    printf("sizeof(PromptKind) = %zu\n", sizeof(PromptKind));
    printf("sizeof(LineAttrs)  = %zu\n", sizeof(LineAttrs));
    printf("sizeof(CPUCell)    = %zu\n", sizeof(CPUCell));
    printf("sizeof(GPUCell)    = %zu\n", sizeof(GPUCell));
    size_t per_seg = 80u*2048u*sizeof(CPUCell) + 80u*2048u*sizeof(GPUCell) + 2048u*sizeof(LineAttrs);
    printf("per_segment(xnum=80) = %zu bytes = %.4f KiB = %.7f MiB\n",
           per_seg, per_seg/1024.0, per_seg/1024.0/1024.0);
    return 0;
}
CEOF
```

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

The cell sizes are fixed by `static_assert`s: `sizeof(GPUCell) == 20` ([`kitty/data-types.h:221`](#cit-datatypes-h)) and `sizeof(CPUCell) == 12` ([`kitty/data-types.h:228`](#cit-datatypes-h)). `LineAttrs` is a `union` of a bit-field `struct` and a `uint8_t val` ([`kitty/data-types.h:230-239`](#cit-datatypes-h)):

```c
typedef enum { UNKNOWN_PROMPT_KIND = 0, PROMPT_START = 1, SECONDARY_PROMPT = 2, OUTPUT_START = 3 } PromptKind;
typedef union LineAttrs {
    struct {
        uint8_t is_continued : 1;
        uint8_t has_dirty_text : 1;
        uint8_t has_image_placeholders : 1;
        PromptKind prompt_kind : 2;
    };
    uint8_t val;
} LineAttrs ;
```

Although `val` is a single byte, the struct contains `PromptKind prompt_kind : 2` ([`kitty/data-types.h:236`](#cit-datatypes-h)), and `PromptKind` is an **`int`-sized enum** (`sizeof(PromptKind) == 4`). A C bit-field of enum type is allocated in a storage unit the size of that type, so the struct — and therefore the whole union — takes **4 bytes**: **`sizeof(LineAttrs) == 4`**, not 1. This is not a guess; it is confirmed by compiling a probe against the project's own header with the same compiler that built the extension:

```
$ cat /tmp/abi_probe.c
#include "data-types.h"
#include <stdio.h>
int main(void) {
    printf("sizeof(PromptKind) = %zu\n", sizeof(PromptKind));
    printf("sizeof(LineAttrs)  = %zu\n", sizeof(LineAttrs));
    printf("sizeof(CPUCell)    = %zu\n", sizeof(CPUCell));
    printf("sizeof(GPUCell)    = %zu\n", sizeof(GPUCell));
    size_t per_seg = 80u*2048u*sizeof(CPUCell) + 80u*2048u*sizeof(GPUCell) + 2048u*sizeof(LineAttrs);
    printf("per_segment(xnum=80) = %zu bytes = %.4f KiB = %.7f MiB\n",
           per_seg, per_seg/1024.0, per_seg/1024.0/1024.0);
    return 0;
}

$ gcc -I kitty -I /usr/include/python3.13 /tmp/abi_probe.c -o /tmp/abi_probe && /tmp/abi_probe
sizeof(PromptKind) = 4
sizeof(LineAttrs)  = 4
sizeof(CPUCell)    = 12
sizeof(GPUCell)    = 20
per_segment(xnum=80) = 5251072 bytes = 5128.0000 KiB = 5.0078125 MiB
```

Hence one segment costs:

```
per_segment = SEGMENT_SIZE * (xnum*sizeof(CPUCell) + xnum*sizeof(GPUCell) + sizeof(LineAttrs))
            = 2048 * (xnum*12 + xnum*20 + 4)
            = 2048 * (xnum*32 + 4)

at xnum = 80:  2048 * (80*32 + 4) = 2048 * 2564 = 5,251,072 bytes = 5.0078125 MiB  (= 5128 KiB)
```

This exact figure (**5,251,072 B = 5128 KiB**) is what appears at runtime as the ≈5 MiB allocation step in Q3 (observed as **5132 kB**, i.e. the 5128 KiB request rounded up by **~4 KiB — one page** — of allocator/`mmap` bookkeeping).

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

At the **default** `scrollback_lines 2000`, memory consumption is **bounded and plateaus**. As lines stream in, `HistoryBuf.count` rises to `ynum = 2000` and **stops**; resident memory attributable to scrollback rises to **≈5.3 MB and then stays flat** — feeding 5,000, 50,000, 200,000, or 500,000 lines yields the **same** plateau. The buffer is a fixed-capacity **ring**: once full, each new line **overwrites the oldest** rather than allocating more memory.

### Command and complete, unedited output (two runs)

The observation script feeds up to 500,000 newline-terminated lines through the real `parse_bytes` path at the default scrollback, sampling `VmRSS` and `count` at checkpoints:

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py q1        # run 1
# Q1 default scrollback: ynum=2000 cols=80 lines=24
 lines_fed   count  VmRSS_kB  dRSS_kB
         0       0     26588        0
      1000     977     29312     2724
      2000    1977     31820     5232
      5000    2000     31876     5288
     50000    2000     31880     5292
    200000    2000     31884     5296
    500000    2000     31892     5304
```

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py q1        # run 2
# Q1 default scrollback: ynum=2000 cols=80 lines=24
 lines_fed   count  VmRSS_kB  dRSS_kB
         0       0     26508        0
      1000     977     29232     2724
      2000    1977     31740     5232
      5000    2000     31796     5288
     50000    2000     31800     5292
    200000    2000     31804     5296
    500000    2000     31828     5320
```

The **`count` series is identical across both runs** (`0 → 977 → 1977 → 2000 → 2000 → 2000 → 2000`) — it is the deterministic primary signal. The `dRSS` plateau matches to within a few kB (`5304` vs `5320` kB at 500,000 lines); the small difference is ordinary run-to-run OS-allocator baseline variance.

### Reading the numbers

- **Plateau at the capacity.** `count` reaches `ynum = 2000` by ~2,023 lines fed and never exceeds it. `dRSS` climbs to ≈5.3 MB while the single segment fills and then **flattens** — going from 5,000 to 500,000 lines changes `dRSS` by only ~16–32 kB (noise), not by megabytes. This is the "actual memory measurement" the user asked for: **the curve is a rising ramp that saturates into a flat line.**
- **The `count = lines_fed − 23` offset.** At `lines = 24`, a line only migrates into history when the cursor sits on the bottom row (`lines − 1 = 23`) and a further line-feed scrolls the screen. So the first 23 lines populate the on-screen grid, and `count = max(0, lines_fed − 23)` (capped at `ynum`). This is visible directly: 1000 → 977 and 2000 → 1977 (both `= fed − 23`), then capped at 2000.
- **One segment only.** Because `ynum = 2000 < SEGMENT_SIZE = 2048`, exactly **one** ≈5 MiB segment is ever allocated (the eager one from construction, [`kitty/history.c:127`](#cit-history)). That is why the default plateau (~5.3 MB) is on the order of a single segment.

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
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py infinite   # run 1
# Infinite scrollback: ynum=4294967295
 lines_fed    count  VmRSS_kB  dRSS_kB
      1000      977     29252     2720
      5000     4977     39284    12752
     10000     9977     51812    25280
     25000    24977     89400    62868
     50000    49977    152052   125520
```

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py infinite   # run 2
# Infinite scrollback: ynum=4294967295
 lines_fed    count  VmRSS_kB  dRSS_kB
      1000      977     29224     2720
      5000     4977     39256    12752
     10000     9977     51784    25280
     25000    24977     89372    62868
     50000    49977    152024   125520
```

Here `count` grows unbounded (`= lines_fed − 23`) and `dRSS` reaches **≈123 MiB (125,520 kB) at 50,000 lines** — about 24 segments' worth (`125,520 / 5,132 ≈ 24.5`). The `dRSS` series is **identical across both runs**. This is the runtime demonstration of the warning in kitty's own docs, whose `long_text` reads: *"Note that using very large scrollback is not recommended as it can slow down performance of the terminal and also use large amounts of RAM. Instead, consider using scrollback_pager_history_size."* ([`kitty/options/definition.py:372-380`](#cit-definition)). The negative→infinite mapping is [`kitty/options/utils.py:557-560`](#cit-utils).

> **External corroboration (secondary).** kitty issue #970 — <https://github.com/kovidgoyal/kitty/issues/970> — reports that starting an empty kitty with a 64k-line scrollback consumes roughly ~50 MB, and filling that buffer adds on the order of a few hundred MB more. That external data point is consistent with the per-segment, stair-step growth measured here (≈5 MiB per 2,048 lines at 80 columns scales to hundreds of MB across 64k lines). It is cited only as secondary corroboration; the primary evidence is this document's own measurements above.


---

## Q2 — Is the terminal responsive when scrolling a huge history during live output?

> *"When I scroll back through a very large history while new output is still being generated, does the terminal remain responsive? What latency or lag can I observe between my scroll input and the display updating? Are there any visible signs of the system prioritizing one operation over another?"*

### Direct answer

**Yes, the terminal stays responsive**, and the reason is structural:

1. **The output-ingest path is cheap.** Feeding 10,000 lines through the real parser takes **≈10 ms** (~1 ms per 1,000 lines) — the model layer is nowhere near a bottleneck even under a heavy stream.
2. **Scrolling is O(1).** A scroll command changes a single integer, `scrolled_by`; the cost is independent of history size (`screen_history_scroll`, [`kitty/screen.c:4091`](#cit-screen)). Scrolling back through 2,000 or 2,000,000 lines is the same cheap operation.
3. **Prioritization is by design: buffered I/O + a paced main loop.** Reading child output happens on a dedicated **I/O thread** that only `read()`s bytes and wakes the main loop, coalesced by `input_delay` ([`kitty/child-monitor.c:291`](#cit-childmon), [`:1562-1569`](#cit-childmon)). Parsing and drawing then run **sequentially on the main loop** (`parse_input` then `render`, [`kitty/child-monitor.c:1236-1237`](#cit-childmon)) — there is **no separate render thread** — paced by `input_delay`/`repaint_delay` so a burst of output cannot starve input handling, and vice-versa.
4. **The scrolled-back view stays pinned** to the content you're reading while new lines stream in, via a re-anchor of `scrolled_by` ([`kitty/screen.c:2761`](#cit-screen)). This particular step runs in the **render step of the main loop** (`screen_update_cell_data`), so — see the honest limitation below — it cannot be timed in a model-only headless harness and is explained from the source instead.

### Command and complete, unedited output (two runs)

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_q2.py     # run 1
# SCROLL_LINE=-999999 SCROLL_PAGE=-999998 SCROLL_FULL=-999997
ingest 10000 lines: 10.592 ms total, 1.0592 ms per 1000 lines; count=9977
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
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_q2.py     # run 2
# SCROLL_LINE=-999999 SCROLL_PAGE=-999998 SCROLL_FULL=-999997
ingest 10000 lines: 10.341 ms total, 1.0341 ms per 1000 lines; count=9977
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

Both runs are identical apart from the wall-clock ingest time (10.592 vs 10.341 ms), which is expected jitter.

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

Responsiveness under simultaneous scroll + heavy output is preserved by three structural facts, all grounded in `kitty/child-monitor.c`:

1. **Reading child output is off the main loop.** A dedicated **I/O thread** (`io_loop`, started by `pthread_create(&self->io_thread, NULL, io_loop, self)` [`kitty/child-monitor.c:291`](#cit-childmon)) does nothing but `read()` raw bytes from the child ptys into per-`Screen` buffers (`read_bytes` [`kitty/child-monitor.c:1531`](#cit-childmon) → `read(fd, …)` [`kitty/child-monitor.c:1345`](#cit-childmon)) and then wake the main loop. Crucially, it does **not** wake the main loop on every read: wakeups are **coalesced** by `input_delay` — *"we only wakeup the main loop after input_delay as wakeup is an expensive operation"* [`kitty/child-monitor.c:1562-1569`](#cit-childmon). So a flood of output turns into a *bounded rate* of main-loop wakeups, never a per-byte storm.
2. **Parsing and rendering are sequential on the main loop — there is no separate render thread.** The only threads created here are the I/O thread and a remote-control "talk" thread ([`kitty/child-monitor.c:256, 286, 291`](#cit-childmon)); rendering is **not** on its own thread. On each tick, the main-loop callback `process_global_state` ([`kitty/child-monitor.c:1224`](#cit-childmon)) runs the VT parser and then draws, in this order:
   ```c
   if (parse_input(self)) input_read = true;   // child-monitor.c:1236  → VT parse → Screen → history
   render(now, input_read);                    // child-monitor.c:1237  → prepare + draw frame
   ```
   `parse_input` *"Parse[s] all available input that was read in the I/O thread"* ([`kitty/child-monitor.c:451-452`](#cit-childmon)). Because they run on one thread in sequence, neither preempts the other mid-operation; instead both are **paced** so neither can monopolize the tick.
3. **The pacing knobs bound both rates.** Parsing coalesces new input over `input_delay` before the frame (`set_maximum_wait(OPT(input_delay) - …)` in `do_parse` [`kitty/child-monitor.c:445-446`](#cit-childmon)); rendering is throttled by `repaint_delay` — `render` returns early if a frame was drawn too recently and no new input arrived (`if (!input_read && time_since_last_render < OPT(repaint_delay)) { … return; }` [`kitty/child-monitor.c:871-877`](#cit-childmon)). The shipped values are `repaint_delay '10'` ms (~100 FPS cap) [`kitty/options/definition.py:866`](#cit-definition), `input_delay '3'` ms [`kitty/options/definition.py:878`](#cit-definition), and `sync_to_monitor 'yes'` [`kitty/options/definition.py:889`](#cit-definition).

The net effect — the **"sign of prioritization"** the user asks about — is **render-cadence throttling driven by input arrival**, not one operation stalling on the other: heavy output is buffered by the I/O thread, coalesced by `input_delay`, parsed in batches on the main loop, and turned into periodic repaints capped by `repaint_delay`, while your O(1) scroll update to `scrolled_by` is applied on the same main loop and is essentially free.

### Honest headless limitation — the live re-anchor

The one behavior that **cannot** be directly timed in a model-only (headless) harness is the "view stays pinned while output streams" re-anchor. In the run above, after scrolling back to `scrolled_by = 3` and then feeding 5,000 more lines, **`scrolled_by` stays at 3** (it does *not* auto-advance) while `count` climbs to 14,977. That is expected and correct for a headless harness, because the re-anchor lives in the **render step**, `screen_update_cell_data` ([`kitty/screen.c:2738`](#cit-screen)), at [`kitty/screen.c:2761`](#cit-screen):

```c
    if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);
```

`screen_update_cell_data` runs only when a frame is prepared for the GPU — i.e. inside the `render` step of the main loop ([`kitty/child-monitor.c:1237`](#cit-childmon)), never in a model-only harness that calls neither `render` nor `screen_update_cell_data`. So `scrolled_by` is not advanced here. In a live terminal, each repaint (paced by `repaint_delay`) advances `scrolled_by` by the number of lines added since the last frame (`history_line_added_count`, incremented in `INDEX_UP` at [`kitty/screen.c:1559`](#cit-screen)), which keeps the content you're viewing pinned in place as new output pushes older lines further back. **This is reported as reasoned-from-source, not as a directly-timed headless measurement.**


---

## Q3 — Where are the buffer boundaries, and when is new storage allocated?

> *"I'm also curious about the boundaries in the buffer system. At what point does the buffer's behavior change as it grows, for example, when does allocation of new storage occur, and can I observe this happening through memory monitoring?"*

### Direct answer

New storage is allocated **one segment at a time**, and a segment is exactly **`SEGMENT_SIZE = 2048` lines** wide. **The behavior changes at each 2,048-line boundary:** whenever the write position crosses into a not-yet-backed 2,048-line block (and total capacity `ynum` has not been reached), `segment_for` calls `add_segment`, which does a single ≈5 MiB `calloc`. **Yes — this is observable through memory monitoring**, and the cleanest signal is *virtual* memory: `VmSize`/`VmData` jump by ≈5 MiB in a discrete **staircase**, one step per 2,048 lines, precisely when `count` crosses `2048, 4096, 6144, 8192, …`. Resident memory (`VmRSS`) rises **approximately linearly** in the same run because the OS commits the freshly-`calloc`'d pages lazily as each line is written.

### Command and complete, unedited output (two runs)

This probe uses a large capacity (`scrollback = 100000`, so many segments are possible) and samples every 256 lines while tracking `VmRSS`, `VmSize`, and `VmData`:

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_q3_detail.py    # run 1
# ynum=100000 SEGMENT_SIZE=2048 per-seg=5.0078125MiB (5251072 B)
   fed  count  dRSS_kB  dVmSize_kB dVmData_kB
   256    233      860           0          0
   512    489     1504           0          0
   768    745     2144           0          0
  1024   1001     2784           0          0
  1280   1257     3428           0          0
  1536   1513     4068           0          0
  1792   1769     4712           0          0
  2048   2025     5352           0          0
  2304   2281     6000        5132       5132
  2560   2537     6640        5132       5132
  2816   2793     7280        5132       5132
  3072   3049     7920        5132       5132
  3328   3305     8568        5132       5132
  3584   3561     9208        5132       5132
  3840   3817     9848        5132       5132
  4096   4073    10488        5132       5132
  4352   4329    11136       10264      10264
  4608   4585    11776       10264      10264
  4864   4841    12416       10264      10264
  5120   5097    13056       10264      10264
  5376   5353    13700       10264      10264
  5632   5609    14340       10264      10264
  5888   5865    14980       10264      10264
  6144   6121    15620       10264      10264
  6400   6377    16268       15396      15396
  6656   6633    16908       15396      15396
  6912   6889    17548       15396      15396
  7168   7145    18188       15396      15396
  7424   7401    18832       15396      15396
  7680   7657    19472       15396      15396
  7936   7913    20112       15396      15396
  8192   8169    20752       15396      15396
  8448   8425    21400       20528      20528
  8704   8681    22040       20528      20528
  8960   8937    22680       20528      20528
  9216   9193    23320       20528      20528
```

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_q3_detail.py    # run 2
# ynum=100000 SEGMENT_SIZE=2048 per-seg=5.0078125MiB (5251072 B)
   fed  count  dRSS_kB  dVmSize_kB dVmData_kB
   256    233      860           0          0
   512    489     1504           0          0
   768    745     2144           0          0
  1024   1001     2784           0          0
  1280   1257     3428           0          0
  1536   1513     4068           0          0
  1792   1769     4708           0          0
  2048   2025     5348           0          0
  2304   2281     5996        5132       5132
  2560   2537     6640        5132       5132
  2816   2793     7280        5132       5132
  3072   3049     7920        5132       5132
  3328   3305     8568        5132       5132
  3584   3561     9208        5132       5132
  3840   3817     9848        5132       5132
  4096   4073    10488        5132       5132
  4352   4329    11136       10264      10264
  4608   4585    11776       10264      10264
  4864   4841    12416       10264      10264
  5120   5097    13056       10264      10264
  5376   5353    13700       10264      10264
  5632   5609    14340       10264      10264
  5888   5865    14980       10264      10264
  6144   6121    15620       10264      10264
  6400   6377    16268       15396      15396
  6656   6633    16908       15396      15396
  6912   6889    17548       15396      15396
  7168   7145    18188       15396      15396
  7424   7401    18832       15396      15396
  7680   7657    19472       15396      15396
  7936   7913    20112       15396      15396
  8192   8169    20752       15396      15396
  8448   8425    21400       20528      20528
  8704   8681    22040       20528      20528
  8960   8937    22680       20528      20528
  9216   9193    23320       20528      20528
```

### Reading the numbers — the staircase, honestly

**The virtual-memory staircase (the allocation events).** `dVmSize`/`dVmData` are **0** while `count ≤ 2025` (still inside the first, eagerly-allocated segment). The **first step to `5132 kB`** appears at `fed = 2304` (`count = 2281`) — i.e. right after `count` crossed **2048**. The next steps land at `fed = 4352` (`count` crossed **4096**, → `10264 kB`), `fed = 6400` (`count` crossed **6144**, → `15396 kB`), and `fed = 8448` (`count` crossed **8192**, → `20528 kB`). Each increment is **one segment**:

```
5132 kB  = 5,251,072 B (5128 KiB, theoretical per-segment) + ~4 kB (one page) allocator/mmap rounding
10264 kB = 2 × 5132     15396 kB = 3 × 5132     20528 kB = 4 × 5132
```

That is exactly `add_segment`'s single `calloc` ([`kitty/history.c:25`](#cit-history)) becoming visible. **The boundary is `count` crossing a multiple of `SEGMENT_SIZE = 2048`.**

**The resident-memory ramp (demand paging).** `dRSS` does *not* jump in ≈5 MiB steps; it rises **≈linearly**, about **640 kB per 256 lines ≈ 2.5 kB/line** — matching the per-line footprint `xnum*32 + sizeof(LineAttrs) = 80*32 + 4 = 2564 B` at 80 columns. This is because `calloc` reserves the whole segment's zero-backed pages up front (visible immediately in `VmSize`), but Linux only commits a physical page to `VmRSS` when a line is actually written into it. **Reporting both metrics is the honest picture: the *allocation* is a virtual-memory staircase; the *residency* is a demand-paged ramp.** A naive "RSS staircase" would be inaccurate.

**Run-to-run note.** The `count` series and the ≈5 MiB (`5132 kB`) step size are **identical** across both runs. Absolute `VmSize` baselines can occasionally show a small extra glibc-arena increment (~1 MB) from the `realloc` of the `self->segments` array / Python allocations; it shifts the baseline but not the per-segment step size. This is why `HistoryBuf.count` is used as the deterministic primary signal and memory as corroboration.

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
| **Default** (`scrollback_lines 2000`) | `2000` | Bounded: `count` caps at 2000, `dRSS` plateaus ≈5.3 MB; one segment only | Q1 output above |
| **Large** (`scrollback = 100000`) | `100000` | Segmented growth: ≈5 MiB `VmSize` step per 2,048 lines | Q3 output above |
| **Effectively-infinite** (`scrollback_lines` negative → `2**32-1`) | `4294967295` | Unbounded: `count = lines_fed − 23`, `dRSS` → ≈123 MiB @ 50k lines | Q1 "infinite" output above |

The `ynum` values are printed directly by the scripts (`ynum=2000`, `ynum=100000`, `ynum=4294967295`), confirming `ynum = MAX(scrollback, lines)` ([`kitty/screen.c:130`](#cit-screen)) and the negative→`2**32-1` parse ([`kitty/options/utils.py:557-560`](#cit-utils)).

### Main screen vs. alternate screen — the history gate

The alternate screen (used by full-screen programs like `vim`/`less`) does **not** feed scrollback. Feeding 20,000 lines on the alt screen leaves `count = 0`; switching back to the main screen resumes history migration (capped at `ynum`):

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py altscreen
# Alt-screen gate. ynum=2000
main start: count=0
after 20000 lines on ALT screen: count=0
after 20000 lines back on MAIN screen: count=2000
```

**Rationale.** History migration is gated by `add_to_history = self->linebuf == self->main_linebuf && self->margin_top == 0`, computed in `screen_index` ([`kitty/screen.c:1574`](#cit-screen)) and `screen_scroll` ([`kitty/screen.c:1593`](#cit-screen)); only when it is true does `INDEX_UP` call `historybuf_add_line` ([`kitty/screen.c:1558`](#cit-screen)). On the alternate screen `linebuf` points at `alt_linebuf`, so the gate is false and nothing is added — matching the observed `count = 0`.

### Pager history OFF (canonical default) vs. ON

With the shipped default `scrollback_pager_history_size 0`, lines overflowing the ring are **discarded**; with pager history enabled, overflow is **diverted** to a separate byte buffer and remains retrievable. The ring `count` is unaffected either way:

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py pager off
# Pager history OFF (0). ynum=2000
after 10000 lines: count=2000 pagerhist_as_text_len=0
```

```
$ PYTHONPATH=. /root/kitty-venv/bin/python /tmp/obs_scrollback.py pager on
# Pager history ON (10MB). ynum=2000
after 10000 lines: count=2000 pagerhist_as_text_len=47862
```

**Rationale.** In `historybuf_push`, when the ring is full (`count == ynum`) the evicted line is handed to `pagerhist_push` ([`kitty/history.c:280`](#cit-history)) before `start_of_data` advances. With the pager buffer sized 0 there is nowhere to store it (`pagerhist_as_text()` returns 0 chars); with a 10 MB buffer the evicted text accumulates and `pagerhist_as_text()` returns a large positive length (**47,862 characters** in this run — this magnitude depends on line content/length; the salient fact is that it is a large positive number when enabled and exactly 0 when disabled). `count` stays `2000` in both cases because the pager buffer is **separate** from the scrollback ring. The default `'0'` is [`kitty/options/definition.py:406`](#cit-definition); the parser is [`kitty/options/utils.py:564`](#cit-utils).

### Buffer states over time: empty → filling → saturated

The three temporal states the questions imply are all visible in the Q1 default run:

| State | Observation (from Q1 default output) | Meaning |
|-------|--------------------------------------|---------|
| **Empty** | `lines_fed=0 → count=0, dRSS=0` | Buffer allocated (one eager segment) but no history yet |
| **Filling** | `1000 → count=977`; `2000 → count=1977` | `count` rising toward `ynum`; `dRSS` ramping to ≈5.2 MB |
| **Saturated** | `5000..500000 → count=2000, dRSS≈5.3 MB flat` | Ring full; oldest line overwritten on each push; memory flat |

The transition from *filling* to *saturated* is precisely the moment `count` reaches `ynum` and `historybuf_push` switches from `count++` to advancing `start_of_data` ([`kitty/history.c:279-282`](#cit-history)).

---

## Read-only scope & reproducibility

- **The repository is unchanged.** The only new file is this document, `blitzy/documentation/kitty_815df1e210e0.md`. No source, test, configuration, or build file was modified, added, or deleted.
- **Temporary scripts lived in `/tmp` and were deleted.** The three observation scripts (`/tmp/obs_scrollback.py`, `/tmp/obs_q3_detail.py`, `/tmp/obs_q2.py`) and the C ABI probe (`/tmp/abi_probe.c`) were created outside the repository (their full source appears above) and removed after the measurements were captured.
- **Magnitudes were confirmed across ≥2 runs.** The deterministic `HistoryBuf.count` series is the **primary signal** (identical across runs); `VmRSS`/`VmSize` deltas are **corroboration** and their absolute baselines vary slightly run-to-run, which is stated wherever it occurs. Input was fed in modest chunks and each configuration ran in a fresh process to avoid transient-input and baseline-carryover artifacts.

**Complete, unedited read-only proof (command + output).** `git status --porcelain` and `git check-ignore` print nothing decorative, so a trailing `echo …_exit=$?` marker proves each command ran; the `ls` block proves the temporary scripts are gone:

```
$ git status --porcelain; echo "status_exit=$?"
status_exit=0

$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md

$ git check-ignore kitty/fast_data_types.so build; echo "check_ignore_exit=$?"
kitty/fast_data_types.so
build
check_ignore_exit=0

$ ls -l /tmp/obs_scrollback.py /tmp/obs_q2.py /tmp/obs_q3_detail.py /tmp/abi_probe.c
ls: cannot access '/tmp/obs_scrollback.py': No such file or directory
ls: cannot access '/tmp/obs_q2.py': No such file or directory
ls: cannot access '/tmp/obs_q3_detail.py': No such file or directory
ls: cannot access '/tmp/abi_probe.c': No such file or directory
```

The `git diff --name-status` output confirms exactly **one** file added relative to the base commit; `git check-ignore` confirms the build artifacts are ignored (so they never appear as changes); and the `ls` failures confirm the temporary scripts were removed.

---

## Final coverage pass

Every part of every question, and every implied condition, is answered above. This checklist confirms coverage; the "Where" column points to the section holding the command + complete output and the `file:line` grounding.

| # | Question part / implied condition | Answer (one line) | Where |
|---|-----------------------------------|-------------------|-------|
| 1 | **Q1** memory consumption as history accumulates | Bounded at default; plateaus ≈5.3 MB once `count` hits `ynum=2000` | Q1 |
| 2 | **Q1** actual memory measurements, not theory | Two runs of real `VmRSS` + `count` at up to 500,000 lines | Q1 output |
| 3 | **Q2** does the terminal remain responsive | Yes — ingest ≈1 µs/line, scroll O(1) | Q2 |
| 4 | **Q2** latency/lag between scroll input and display | Scroll = O(1) `scrolled_by` update; ingest ≈10 ms/10k lines | Q2 |
| 5 | **Q2** visible signs of prioritizing one operation over another | Render-cadence throttling: buffered I/O thread + `input_delay`/`repaint_delay`-paced main loop | Q2 "Prioritization" |
| 6 | **Q3** at what point does behavior change as it grows | At each 2,048-line (`SEGMENT_SIZE`) boundary | Q3 |
| 7 | **Q3** when does allocation of new storage occur | When `count` crosses a multiple of 2048 (while `count < ynum`), via `segment_for`→`add_segment` | Q3 rationale |
| 8 | **Q3** can I observe this through memory monitoring | Yes — discrete ≈5 MiB `VmSize`/`VmData` step per boundary; RSS demand-paged ramp | Q3 output |
| 9 | Default scrollback (`2000`) | One segment; bounded plateau | Q1, Secondary |
| 10 | Large scrollback (`100000`) | Segmented staircase, ≈5 MiB per 2048 lines | Q3, Secondary |
| 11 | Effectively-infinite scrollback (negative → `2**32-1`) | Unbounded; ≈123 MiB @ 50k lines | Q1 infinite, Secondary |
| 12 | Main screen vs. alternate screen | Alt screen does **not** feed history (`count=0`); main resumes | Secondary (alt-screen gate) |
| 13 | Pager history OFF vs. ON | OFF → `pagerhist=0`; ON → large positive (47,862); ring `count` unchanged | Secondary (pager) |
| 14 | Before / during / after (empty → filling → saturated) | `count 0 → rising → capped at 2000`; `dRSS 0 → ramp → flat` | Secondary (states) |
| 15 | Real entry point (`parse_bytes → Screen → historybuf_add_line`) | Used for every number; direct `HistoryBuf` unit path labeled non-canonical | Methodology |
| 16 | `sizeof(LineAttrs)` / per-segment arithmetic | `sizeof(LineAttrs)=4` (compiled), per-segment `5,251,072 B = 5128 KiB` | Data structure (ABI probe) |
| 17 | Read-only scope; temporary scripts removed; one new file | Clean tree; one-file diff; artifacts ignored; scripts deleted | Read-only scope |

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
- `PromptKind` enum (`int`-sized; `sizeof(PromptKind)=4`) — L230
- `LineAttrs` union — L230-239; the `PromptKind prompt_kind : 2` bit-field (L236) forces the union to **4 bytes**, so `sizeof(LineAttrs)=4` (confirmed by the ABI probe)
- `HistoryBuf` struct with `num_segments` (not exposed to Python) — L283-290

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

<a id="cit-childmon"></a>**`kitty/child-monitor.c`**
- `talk_thread` creation (remote control) — L256, L286
- I/O thread creation `pthread_create(&self->io_thread, NULL, io_loop, self)` — L291
- `read(fd, …)` from child pty — L1345
- `do_parse` sets `input_delay`-based wait — L445-446
- `parse_input` ("Parse all available input that was read in the I/O thread") — L451-452
- `render` (early-return paced by `repaint_delay`) — L871 (pacing check L875-877)
- `process_global_state` main-loop tick — L1224 (`parse_input` L1236, then `render` L1237)
- `io_loop` (I/O thread body) — L1481; reads child bytes via `read_bytes` — L1531
- wakeup coalescing over `input_delay` (`WAKEUP` macro + comment) — L1562-1569

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
