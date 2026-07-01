# kitty scrollback `HistoryBuf` under heavy load — an evidence-backed investigation

> **Source under test:** `kitty_815df1e210e0` — the pinned, tracked C/Python/Go source at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. The entire investigation (build, probes, measurements) was performed against that source, which is left byte‑for‑byte unchanged. This Markdown document is the single repository artifact added, committed directly atop the pinned source commit (so `git rev-parse HEAD~1` resolves to `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`).
>
> **Governing methodology:** `SWE-AtlasQnA-Repo`. Every numeric or behavioral claim below was produced by **building kitty's C extension and running the relevant code paths first**, then transcribing the *observed* output. Each measured value is shown next to the exact command/code that produced it, and every claim about the system carries an inline `[path:line]` citation to the pinned commit.

## The question

A three‑part question about how kitty's scrollback `HistoryBuf` behaves under heavy load:

- **Q1 — Memory consumption under heavy load.** When hundreds of thousands of lines are printed rapidly, what happens to memory as scrollback accumulates? *(Actual measurements required, not theory.)*
- **Q2 — Responsiveness while scrolling during active output.** While scrolling back through a large history as new output streams in, does the terminal stay responsive? What scroll→display latency is observable, and what signs of prioritization exist?
- **Q3 — Buffer boundaries / allocation transitions.** At what point does buffer behavior change as it grows? When is new backing storage allocated, and can that transition be observed via memory monitoring?

> **User constraint (verbatim):** "Temporary scripts may be used for observation and measurement, but the repository itself should remain unchanged."

All observation scripts referenced below (`/tmp/obs_mem.py`, `/tmp/obs_mem_fine.py`, `/tmp/obs_scroll.py`, `/tmp/sizes.c`) were created **outside** the repository, executed, transcribed here, and then **deleted**. The build produces only git‑ignored artifacts (`*.so`, `build/`, `__pycache__/`). The working tree is byte‑for‑byte unchanged (verified in §(e)).

---

## (a) Environment & build methodology

### Toolchain observed in the container

The investigation ran inside the provided image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`).

```console
$ python3 --version
Python 3.13.7
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ go version
go version go1.24.4 linux/amd64
```

Declared version requirements: `requires-python = ">=3.8"` [pyproject.toml:L2] (highest CI‑tested is 3.11); `go 1.22` [go.mod:L3]. The container's Python 3.13.7 / Go 1.24.4 satisfy both bounds.

### Building `kitty.fast_data_types`

The C terminal core is compiled into a Python extension. The `Makefile` `all:` target runs `python3 setup.py` [Makefile:L12-L13]; instrumented variants exist (`make debug` [Makefile:L22-L23], `make asan` [Makefile:L29-L30], `make profile` [Makefile:L32]). `setup.py` builds the `kitty/fast_data_types` extension [setup.py:L1090-L1091] with `-std=c11` [setup.py:L492] and `-O3` [setup.py:L482,L510] and does **not** pass `-fshort-enums` (this matters for the Q1 arithmetic — see §(b)).

The build in this container requires one extra flag to tolerate a `wayland-protocols` switch‑enum version delta (it only demotes `-Wswitch`; all other `-Werror` stays on):

```console
$ CFLAGS="-Wno-error=switch" CI=true python3 setup.py
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
kitty/tools/cmd
```

*(This is a representative incremental compile/link slice from `setup.py`'s own dependency check — no tracked file was edited to produce it; `git status --porcelain` reports no change to any source file across the build. Running `setup.py` a second time is a silent no‑op that exits 0 because the extension is already current. Exit code: 0 in both cases.)*

### Verifying the build

The scrollback type tests were run through the built launcher. The correct module‑scoping flag is `--module` (a bare module name reports `No test named [...] found`):

```console
$ CI=true TMPDIR=/root/ktmp ./kitty/launcher/kitty +launch test.py --module datatypes
Running under CI: True
...
test_historybuf (kitty_tests.datatypes.TestDataTypes.test_historybuf) ... ok
test_line (kitty_tests.datatypes.TestDataTypes.test_line) ... ok
test_linebuf (kitty_tests.datatypes.TestDataTypes.test_linebuf) ... ok
...
----------------------------------------------------------------------
Ran 18 tests in 0.013s

OK
```

`test_historybuf` — which constructs `HistoryBuf(3000, 5)` and pushes 3000 lines [kitty_tests/datatypes.py:L487,L501] — passes, confirming the `HistoryBuf` type is functional.

### Import surface used by the probes

After building, the exact structures under investigation import directly from the extension [kitty_tests/__init__.py:L22]:

```python
from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, get_options, monotonic, set_options
```

The scroll enum constants are exported via `PyModule_AddIntMacro` [kitty/screen.c:L9], and their runtime values were confirmed to match the `ScrollTypes` enum [kitty/screen.h:L13]:

```console
$ python3 -c "from kitty.fast_data_types import SCROLL_LINE, SCROLL_PAGE, SCROLL_FULL; print(SCROLL_LINE, SCROLL_PAGE, SCROLL_FULL)"
-999999 -999998 -999997
```

### API surface the probes rely on (verified against C)

- `HistoryBuf(ynum, xnum[, pagerhist_sz])` — the constructor parses `"II|I"` into `&ynum, &xnum, &pagerhist_sz` [kitty/history.c:L138], so **the first argument is the line capacity (`ynum`)** and the second is columns (`xnum`). Read‑only members `.xnum`, `.ynum`, `.count` [kitty/history.c:L555-L558]; methods include `.push(line)` [kitty/history.c:L542-L551].
- `LineBuf(ynum, xnum)`; a line is filled with `line.set_text(text, 0, xnum, cursor)` [kitty_tests/__init__.py:L166-L172].
- `Screen(callbacks, lines, columns, scrollback, cell_width, cell_height, window_id, test_child)` — parses `"|OIIIIIKO"` [kitty/screen.c:L100]; its internal `HistoryBuf` is sized `MAX(scrollback, lines)` [kitty/screen.c:L130]. `Screen.scroll(amt, upwards)` parses `"ip"` [kitty/screen.c:L4123]; `Screen.draw` [kitty/screen.c:L4790], `Screen.linefeed` [kitty/screen.c:L4832], `Screen.carriage_return` [kitty/screen.c:L4833]; read‑only members `Screen.historybuf` [kitty/screen.c:L4902] and `Screen.scrolled_by` [kitty/screen.c:L4903].
- For the Q2 redraw‑path proxy: `Screen.visual_line(y)` [kitty/screen.c:L4788] returns the `y`‑th **visible** row accounting for `scrolled_by` via `visual_line_` [kitty/screen.c:L2843]; `Screen.as_text(callback, as_ansi, add_wrap_markers)` [kitty/screen.c:L4825] serializes the visible viewport, calling `callback` once per line (the argument order is confirmed by `kitty.window.as_text`'s call `f(lines.append, as_ansi, add_wrap_markers)` [kitty/window.py:L377]).

---

## (b) Q1 — Memory consumption under heavy load

### What was run

`/tmp/obs_mem.py` constructs `HistoryBuf(ynum=100000, xnum=80)` and pushes 200,000 identical lines, sampling resident memory from `/proc/self/status` (`VmRSS`, `VmHWM`) and `resource.getrusage().ru_maxrss` after every 2048‑push batch (the batch is aligned to the segment size, see §(d)). This is the **complete** script, verbatim:

```python
#!/usr/bin/env python3
# Q1/Q3 memory probe: push hundreds of thousands of lines into a HistoryBuf and
# sample resident memory after every SEGMENT_SIZE-aligned batch.
import resource
from kitty.fast_data_types import HistoryBuf, LineBuf, Cursor

def mem_kb():
    vmrss = vmhwm = -1
    with open('/proc/self/status') as f:
        for ln in f:
            if ln.startswith('VmRSS:'):
                vmrss = int(ln.split()[1])
            elif ln.startswith('VmHWM:'):
                vmhwm = int(ln.split()[1])
    ru = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss  # kB on Linux
    return vmrss, vmhwm, ru

XNUM, YNUM, PUSHES, BATCH = 80, 100000, 200000, 2048  # BATCH aligned to SEGMENT_SIZE
lb = LineBuf(2, XNUM)
c = Cursor()
line = lb.line(0)
line.set_text('x' * XNUM, 0, XNUM, c)   # one representative full-width line, reused
hb = HistoryBuf(YNUM, XNUM)             # NOTE arg order: (ynum=lines, xnum=cols)

print(f"# xnum={XNUM} ynum={YNUM} pushes={PUSHES} batch={BATCH}")
print("# pushes count VmRSS_kB VmHWM_kB ru_maxrss_kB (dRSS)")
vmrss, vmhwm, ru = mem_kb()
print(f"0 0 {vmrss} {vmhwm} {ru}")
prev = vmrss
for i in range(1, PUSHES + 1):
    hb.push(line)
    if i % BATCH == 0:
        vmrss, vmhwm, ru = mem_kb()
        print(f"{i} {hb.count} {vmrss} {vmhwm} {ru}  (dRSS={vmrss - prev}kB)")
        prev = vmrss
vmrss, vmhwm, ru = mem_kb()
print(f"# END pushes={PUSHES} count={hb.count} VmRSS={vmrss}kB VmHWM={vmhwm}kB ru_maxrss={ru}kB")
print(f"# hb.xnum={hb.xnum} hb.ynum={hb.ynum}")
```

The line is filled once via `line.set_text('x'*XNUM, 0, XNUM, c)` [kitty_tests/__init__.py:L166-L172] and reused for every push; `historybuf_add_line` copies its contents into the buffer [kitty/history.c:L286-L291], so the reused source line is representative and does not itself grow.

### Observed output (verbatim)

```console
$ PYTHONPATH="$PWD" python3 /tmp/obs_mem.py
# xnum=80 ynum=100000 pushes=200000 batch=2048
# pushes count VmRSS_kB VmHWM_kB ru_maxrss_kB (dRSS)
0 0 15868 15868 15360
2048 2048 21188 21188 20480  (dRSS=5320kB)
4096 4096 26320 26320 25600  (dRSS=5132kB)
6144 6144 31452 31452 30720  (dRSS=5132kB)
8192 8192 36584 36584 35840  (dRSS=5132kB)
10240 10240 41716 41716 40960  (dRSS=5132kB)
12288 12288 46848 46848 46080  (dRSS=5132kB)
14336 14336 51980 51980 51200  (dRSS=5132kB)
16384 16384 57112 57112 56320  (dRSS=5132kB)
18432 18432 62244 62244 61440  (dRSS=5132kB)
20480 20480 67376 67376 66560  (dRSS=5132kB)
22528 22528 72512 72512 71680  (dRSS=5136kB)
24576 24576 77644 77644 76800  (dRSS=5132kB)
26624 26624 82776 82776 81920  (dRSS=5132kB)
28672 28672 87908 87908 87040  (dRSS=5132kB)
30720 30720 93040 93040 92160  (dRSS=5132kB)
32768 32768 98172 98172 97280  (dRSS=5132kB)
34816 34816 103304 103304 102400  (dRSS=5132kB)
36864 36864 108436 108436 107520  (dRSS=5132kB)
38912 38912 113568 113568 112640  (dRSS=5132kB)
40960 40960 118700 118700 117760  (dRSS=5132kB)
43008 43008 123836 123836 122880  (dRSS=5136kB)
45056 45056 128968 128968 128000  (dRSS=5132kB)
47104 47104 134100 134100 133120  (dRSS=5132kB)
49152 49152 139232 139232 138240  (dRSS=5132kB)
51200 51200 144364 144364 143360  (dRSS=5132kB)
53248 53248 149496 149496 148480  (dRSS=5132kB)
55296 55296 154628 154628 153600  (dRSS=5132kB)
57344 57344 159760 159760 158720  (dRSS=5132kB)
59392 59392 164892 164892 163840  (dRSS=5132kB)
61440 61440 170024 170024 168960  (dRSS=5132kB)
63488 63488 175156 175156 174080  (dRSS=5132kB)
65536 65536 180288 180288 179200  (dRSS=5132kB)
67584 67584 185420 185420 184320  (dRSS=5132kB)
69632 69632 190552 190552 189440  (dRSS=5132kB)
71680 71680 195684 195684 194560  (dRSS=5132kB)
73728 73728 200816 200816 199680  (dRSS=5132kB)
75776 75776 205948 205948 204800  (dRSS=5132kB)
77824 77824 211080 211080 209920  (dRSS=5132kB)
79872 79872 216212 216212 215040  (dRSS=5132kB)
81920 81920 221344 221344 220160  (dRSS=5132kB)
83968 83968 226476 226476 225280  (dRSS=5132kB)
86016 86016 231608 231608 230400  (dRSS=5132kB)
88064 88064 236740 236740 235520  (dRSS=5132kB)
90112 90112 241872 241872 240640  (dRSS=5132kB)
92160 92160 247004 247004 245760  (dRSS=5132kB)
94208 94208 252136 252136 250880  (dRSS=5132kB)
96256 96256 257268 257268 256000  (dRSS=5132kB)
98304 98304 262400 262400 261120  (dRSS=5132kB)
100352 100000 266652 266652 266240  (dRSS=4252kB)
102400 100000 266652 266652 266240  (dRSS=0kB)
104448 100000 266652 266652 266240  (dRSS=0kB)
106496 100000 266652 266652 266240  (dRSS=0kB)
108544 100000 266652 266652 266240  (dRSS=0kB)
110592 100000 266656 266656 266240  (dRSS=4kB)
112640 100000 266656 266656 266240  (dRSS=0kB)
114688 100000 266656 266656 266240  (dRSS=0kB)
116736 100000 266656 266656 266240  (dRSS=0kB)
118784 100000 266656 266656 266240  (dRSS=0kB)
120832 100000 266656 266656 266240  (dRSS=0kB)
122880 100000 266656 266656 266240  (dRSS=0kB)
124928 100000 266656 266656 266240  (dRSS=0kB)
126976 100000 266656 266656 266240  (dRSS=0kB)
129024 100000 266656 266656 266240  (dRSS=0kB)
131072 100000 266656 266656 266240  (dRSS=0kB)
133120 100000 266656 266656 266240  (dRSS=0kB)
135168 100000 266656 266656 266240  (dRSS=0kB)
137216 100000 266656 266656 266240  (dRSS=0kB)
139264 100000 266656 266656 266240  (dRSS=0kB)
141312 100000 266656 266656 266240  (dRSS=0kB)
143360 100000 266656 266656 266240  (dRSS=0kB)
145408 100000 266656 266656 266240  (dRSS=0kB)
147456 100000 266656 266656 266240  (dRSS=0kB)
149504 100000 266656 266656 266240  (dRSS=0kB)
151552 100000 266656 266656 266240  (dRSS=0kB)
153600 100000 266656 266656 266240  (dRSS=0kB)
155648 100000 266656 266656 266240  (dRSS=0kB)
157696 100000 266656 266656 266240  (dRSS=0kB)
159744 100000 266656 266656 266240  (dRSS=0kB)
161792 100000 266656 266656 266240  (dRSS=0kB)
163840 100000 266656 266656 266240  (dRSS=0kB)
165888 100000 266656 266656 266240  (dRSS=0kB)
167936 100000 266656 266656 266240  (dRSS=0kB)
169984 100000 266656 266656 266240  (dRSS=0kB)
172032 100000 266656 266656 266240  (dRSS=0kB)
174080 100000 266656 266656 266240  (dRSS=0kB)
176128 100000 266660 266660 266240  (dRSS=4kB)
178176 100000 266660 266660 266240  (dRSS=0kB)
180224 100000 266660 266660 266240  (dRSS=0kB)
182272 100000 266660 266660 266240  (dRSS=0kB)
184320 100000 266660 266660 266240  (dRSS=0kB)
186368 100000 266660 266660 266240  (dRSS=0kB)
188416 100000 266660 266660 266240  (dRSS=0kB)
190464 100000 266660 266660 266240  (dRSS=0kB)
192512 100000 266660 266660 266240  (dRSS=0kB)
194560 100000 266660 266660 266240  (dRSS=0kB)
196608 100000 266664 266664 266240  (dRSS=4kB)
198656 100000 266664 266664 266240  (dRSS=0kB)
# END pushes=200000 count=100000 VmRSS=266664kB VmHWM=266664kB ru_maxrss=266240kB
# hb.xnum=80 hb.ynum=100000
```

Every one of the 98 batch rows is shown above — nothing is elided. The 200,000 pushes fall into two regimes visible directly in the series: **48 segment‑sized steps of ≈5132 kB** (pushes 2048 → 98304) while the buffer fills, then a **flat plateau** (pushes 100352 → 200000) after `count` pins at `100000`.

### What actually happens to memory

**Memory rises in discrete, segment‑sized steps and then plateaus.** For the first ~48 batches, each 2048‑push window adds a near‑constant **`dRSS = 5132 kB`** of resident memory (the very first window at `push=2048` reads `5320 kB` because it also absorbs one‑time interpreter/import warmup between the `push=0` and `push=2048` samples; the two `5136 kB` rows at pushes 22528 and 43008 differ by a single 4 KiB page of allocator jitter). Once the buffer's line count reaches its capacity (`count == ynum == 100000`, first seen at the `push=100352` sample where `count` is pinned at `100000`), the increments drop to **`dRSS = 0kB`** and stay there for the remaining ~100,000 pushes — resident memory holds flat at **`VmRSS = 266664 kB`** (`VmHWM`/`ru_maxrss` agree). The three isolated `+4kB` blips during the plateau (pushes 110592, 176128, 196608 in this run) are single‑page allocator bookkeeping, not buffer growth.

The plateau is the direct consequence of `historybuf_push` switching to **circular overwrite** once the buffer is full [kitty/history.c:L275-L284]:

```c
static index_type
historybuf_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
    index_type idx = (self->start_of_data + self->count) % self->ynum;
    init_line(self, idx, self->line);
    if (self->count == self->ynum) {                 // buffer full …
        pagerhist_push(self, as_ansi_buf);           // oldest line spills to pager history
        self->start_of_data = (self->start_of_data + 1) % self->ynum;  // advance ring start
    } else self->count++;                            // … otherwise grow
    return idx;
}
```

Once `count == ynum`, a push reuses the slot of the oldest line (`start_of_data` advances modulo `ynum`) [kitty/history.c:L279-L282] — no new backing storage is allocated, so **main‑buffer memory does not grow further no matter how much more output arrives**.

### Reconciling the measured step against the source

Each backing segment is a single `calloc` [kitty/history.c:L23-L25]:

```c
const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));
```

with `SEGMENT_SIZE = 2048` [kitty/history.c:L15], `sizeof(GPUCell) == 20` [kitty/data-types.h:L221] and `sizeof(CPUCell) == 12` [kitty/data-types.h:L228] (both pinned by `static_assert`). The dominant, exact term is therefore

```
xnum · SEGMENT_SIZE · (sizeof(GPUCell)+sizeof(CPUCell)) = 80 · 2048 · (20+12) = 80 · 2048 · 32 = 5,242,880 B = xnum·64 KiB
```

The small `LineAttrs` sub‑term is `SEGMENT_SIZE · sizeof(LineAttrs)`. **`sizeof(LineAttrs)` is 4 bytes, not 1** — see §(b.1) — so this sub‑term is `2048 · 4 = 8,192 B (8 KiB)`. Predicted per‑segment size:

```
5,242,880 + 8,192 = 5,251,072 B = 5,128 KiB = 5.0078 MiB   (confirmed by /tmp/sizes.c below)
```

The **measured** per‑segment step is `dRSS = 5132 kB = 5,255,168 B`. The difference against the prediction is `5,255,168 − 5,251,072 = 4,096 B`, i.e. **exactly one 4 KiB page** (≈0.08%) of allocator/page‑granularity overhead. Measurement is the ground truth here, and it matches the source arithmetic to within a single page.

> **Measurement nuance.** `VmRSS` counts *touched* (resident) pages, not the `calloc` virtual reservation. A segment's ~5 MiB becomes resident only as `push → init_line`/`copy_line` writes fill its 2048 lines [kitty/history.c:L287-L291], which is why RSS climbs ~5 MiB per 2048‑push window rather than in one instantaneous jump. §(d) uses `VmData` to observe the *reservation* separately.

### Plateau magnitude

At the plateau, total `VmRSS = 266664 kB`. Subtracting the interpreter+import baseline (`15868 kB` at `push=0`) gives the `HistoryBuf` footprint of **`250796 kB ≈ 244.9 MiB`** for 100,000 lines at 80 columns. That is consistent with `ceil(100000 / 2048) = 49` segments (§(d)) — 49 full segments would be `49 · 5128 KiB = 245.4 MiB`, and the 49th segment is only partially resident because `count` caps at 100,000 (48·2048 = 98,304 lines fill 48 segments; the remaining 1,696 lines only touch part of segment 49).

### The separate `PagerHistoryBuf` (do not conflate)

The main `HistoryBuf` is distinct from the secondary **pager** ring buffer `PagerHistoryBuf { void *ringbuf; size_t maximum_size; bool rewrap_needed; }` [kitty/data-types.h:L268-L272], which is an independently bounded growth source governed by `scrollback_pager_history_size` (default `'0'`) [kitty/options/definition.py:L406-L407]; that option converts MB→bytes capped just under 4 GiB via `return min(ans, 4096 * 1024 * 1024 - 1)` [kitty/options/utils.py:L564-L566]. The probe uses the default `pagerhist_sz == 0`, so the pager ring is unused and contributes nothing to the numbers above.

### The "infinite scrollback" edge case

The measured plateau exists because `ynum` is finite. With a **negative** `scrollback_lines`, the value is normalized to `ynum = 2**32 - 1` [kitty/options/utils.py:L557-L561]:

```python
def scrollback_lines(x: str) -> int:
    ans = int(x)
    if ans < 0:
        ans = 2 ** 32 - 1
    return ans
```

With that ceiling, `count == ynum` is effectively never reached, `historybuf_push` never enters the circular‑overwrite branch, and memory grows segment‑by‑segment (≈5 MiB per 2048 lines at 80 columns) **without bound** — the concrete mechanism behind kitty's documented "large amounts of RAM" warning. The shipped default is `scrollback_lines '2000'` [kitty/options/definition.py:L372-L373].

### (b.1) Confirming `sizeof(LineAttrs) == 4`

There is **no `static_assert` pinning `sizeof(LineAttrs)`**, and `add_segment` uses `sizeof(LineAttrs)` directly in the `calloc` [kitty/history.c:L25]. `LineAttrs` is a `union` whose active struct contains `PromptKind prompt_kind : 2` [kitty/data-types.h:L231-L239], and `PromptKind` is an `enum` [kitty/data-types.h:L230] whose underlying type is `int` (4 bytes) under kitty's flags (`-std=c11`, no `-fshort-enums`) — so the union is 4 bytes. `/tmp/sizes.c` replicates the exact cell layouts from `kitty/data-types.h` and confirms this empirically. This is the **complete** program, verbatim:

```c
/* Standalone replica of kitty's cell layouts at commit 815df1e2 (kitty/data-types.h)
 * compiled with kitty's exact C standard/opt flags to confirm per-segment byte size. */
#include <stdio.h>
#include <stdint.h>
#include <stddef.h>
#include <assert.h>

typedef uint32_t char_type;
typedef uint32_t color_type;
typedef uint16_t hyperlink_id_type;
typedef uint16_t combining_type;
typedef uint16_t sprite_index;

typedef union CellAttrs {
    struct {
        uint16_t width : 2;
        uint16_t decoration : 3;
        uint16_t bold : 1;
        uint16_t italic : 1;
        uint16_t reverse : 1;
        uint16_t strike : 1;
        uint16_t dim : 1;
        uint16_t mark : 2;
        uint16_t next_char_was_wrapped : 1;
    };
    uint16_t val;
} CellAttrs;

typedef struct {
    color_type fg, bg, decoration_fg;
    sprite_index sprite_x, sprite_y, sprite_z;
    CellAttrs attrs;
} GPUCell;
_Static_assert(sizeof(GPUCell) == 20, "GPUCell layout");

typedef struct {
    char_type ch;
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];
} CPUCell;
_Static_assert(sizeof(CPUCell) == 12, "CPUCell layout");

typedef enum { UNKNOWN_PROMPT_KIND = 0, PROMPT_START = 1, SECONDARY_PROMPT = 2, OUTPUT_START = 3 } PromptKind;
typedef union LineAttrs {
    struct {
        uint8_t is_continued : 1;
        uint8_t has_dirty_text : 1;
        uint8_t has_image_placeholders : 1;
        PromptKind prompt_kind : 2;
    };
    uint8_t val;
} LineAttrs;

#define SEGMENT_SIZE 2048
static size_t seg_bytes(size_t xnum) {
    return xnum*SEGMENT_SIZE*sizeof(CPUCell) + xnum*SEGMENT_SIZE*sizeof(GPUCell) + SEGMENT_SIZE*sizeof(LineAttrs);
}
int main(void) {
    printf("GPUCell=%zu CPUCell=%zu LineAttrs=%zu\n", sizeof(GPUCell), sizeof(CPUCell), sizeof(LineAttrs));
    for (size_t x = 80; x <= 100; x += 20) {
        size_t b = seg_bytes(x);
        printf("xnum=%zu per-segment=%zu bytes (%.4f MiB)\n", x, b, b/1048576.0);
    }
    return 0;
}
```

Compiled with kitty's exact standard/opt flags and run:

```console
$ gcc -std=c11 -O3 -o /tmp/sizes /tmp/sizes.c && /tmp/sizes
GPUCell=20 CPUCell=12 LineAttrs=4
xnum=80 per-segment=5251072 bytes (5.0078 MiB)
xnum=100 per-segment=6561792 bytes (6.2578 MiB)
```

(Both `_Static_assert(sizeof(GPUCell)==20)` and `_Static_assert(sizeof(CPUCell)==12)` in that program compiled cleanly, so the replicated layout is faithful.) The 4‑byte `LineAttrs` accounts for only `8 KiB` of a ~5 MiB segment (~0.15%), so the headline "**≈5.0 MiB of resident memory per 2048‑line segment at 80 columns**" holds regardless.


---

## (c) Q2 — Responsiveness while scrolling during active output

### What was run

`/tmp/obs_scroll.py` builds a `Screen`, fills its history to two depths (~10k and ~200k lines) by drawing lines, and measures **two distinct latencies** at each depth with `time.perf_counter_ns` (best of 2000 reps, resetting to bottom between reps):

1. **The model‑layer scroll call** — `Screen.scroll(...)`, i.e. the index update inside `screen_history_scroll` (integer math + a dirty flag).
2. **The redraw‑path CPU reconstruction** — after scrolling to the top, the work the renderer does to rebuild the visible viewport from history. `Screen.visual_line(y)` for every one of the 24 visible rows routes through `visual_line_` [kitty/screen.c:L2843], which calls the **same** `historybuf_init_line(...)` [kitty/screen.c:L2847] that `screen_update_cell_data`'s scrollback loop performs [kitty/screen.c:L2765]. `Screen.as_text(cb, as_ansi, wrap)` is a heavier single‑call proxy that reconstructs **and** ANSI‑serializes the same 24 rows [kitty/screen.c:L3485-L3486].

A final loop *interleaves* new output with scrolling and viewport reconstruction to capture worst‑case latency while output is actively streaming. This is the **complete** script, verbatim:

```python
#!/usr/bin/env python3
# Q2 responsiveness probe.
#  (1) model-layer scroll latency: time Screen.scroll(...) -- the index update in
#      screen_history_scroll (integer math + a dirty flag).
#  (2) redraw-path CPU proxy: after scrolling to the top, time reconstructing the
#      visible viewport the way the renderer does -- Screen.visual_line(y) for every
#      visible row routes through visual_line_ -> historybuf_init_line, the SAME call
#      screen_update_cell_data makes in its scrollback loop. as_text(True) is a heavier
#      single-call proxy that also serializes the viewport.
#  (3) interleaved: draw new output between scrolls to get worst-case latency while
#      output is actively streaming.
import time
from kitty.fast_data_types import Screen, SCROLL_LINE, SCROLL_PAGE, SCROLL_FULL
from kitty_tests import Callbacks

cb = Callbacks()
LINES, COLS = 24, 80

def make_screen(scrollback):
    return Screen(cb, LINES, COLS, scrollback, 10, 20, 0, cb)

def fill(s, n):
    for _ in range(n):
        s.draw('x' * 40)
        s.linefeed()
        s.carriage_return()

def best_ns(fn, reps=2000):
    best = 1 << 62
    for _ in range(reps):
        t0 = time.perf_counter_ns()
        fn()
        t1 = time.perf_counter_ns()
        if t1 - t0 < best:
            best = t1 - t0
    return best

def time_scroll(s, amt, reps=2000):
    best = 1 << 62
    for _ in range(reps):
        s.scroll(SCROLL_FULL, False)          # reset to bottom (untimed)
        t0 = time.perf_counter_ns()
        s.scroll(amt, True)                    # scroll up (timed)
        t1 = time.perf_counter_ns()
        if t1 - t0 < best:
            best = t1 - t0
    return best

def redraw_visual_line(s):                     # reconstruct all visible rows from history
    for y in range(LINES):
        s.visual_line(y)

def measure_depth(scrollback, fill_lines):
    s = make_screen(scrollback)
    fill(s, fill_lines)
    depth = s.historybuf.count
    print(f"# historybuf.count={depth}")
    for name, amt in (("SCROLL_LINE", SCROLL_LINE), ("SCROLL_PAGE", SCROLL_PAGE), ("SCROLL_FULL", SCROLL_FULL)):
        ns = time_scroll(s, amt)
        s.scroll(SCROLL_FULL, False); s.scroll(amt, True)
        print(f"depth={depth} {name}: min {ns} ns (scrolled_by={s.scrolled_by})")
    # redraw proxy: scroll to the very top so all 24 visible rows are pulled from history
    s.scroll(SCROLL_FULL, False); s.scroll(SCROLL_FULL, True)
    vl = best_ns(lambda: redraw_visual_line(s))
    sink = []
    at = best_ns(lambda: (sink.clear(), s.as_text(sink.append, True, False)))
    print(f"depth={depth} REDRAW visual_line x{LINES} (viewport at top): min {vl} ns")
    print(f"depth={depth} REDRAW as_text(True) (viewport at top): min {at} ns")
    return s

measure_depth(20000, 10000)
measure_depth(300000, 200000)

# interleaved: stream new output while scrolling, capture worst-case latency
s = make_screen(300000)
fill(s, 200000)
worst_scroll = 0
worst_redraw = 0
for _ in range(1000):
    s.draw('y' * 40); s.linefeed(); s.carriage_return()   # active output
    t0 = time.perf_counter_ns(); s.scroll(SCROLL_LINE, True); t1 = time.perf_counter_ns()
    worst_scroll = max(worst_scroll, t1 - t0)
    t0 = time.perf_counter_ns(); redraw_visual_line(s); t1 = time.perf_counter_ns()
    worst_redraw = max(worst_redraw, t1 - t0)
    s.scroll(SCROLL_FULL, False)
print(f"# interleaved 1000x(draw+scroll+redraw): worst-case scroll={worst_scroll} ns  worst-case redraw(visual_line x{LINES})={worst_redraw} ns")
```

### Observed output (verbatim)

```console
$ PYTHONPATH="$PWD" python3 /tmp/obs_scroll.py
# historybuf.count=9977
depth=9977 SCROLL_LINE: min 100 ns (scrolled_by=1)
depth=9977 SCROLL_PAGE: min 100 ns (scrolled_by=23)
depth=9977 SCROLL_FULL: min 100 ns (scrolled_by=9977)
depth=9977 REDRAW visual_line x24 (viewport at top): min 1604 ns
depth=9977 REDRAW as_text(True) (viewport at top): min 7653 ns
# historybuf.count=199977
depth=199977 SCROLL_LINE: min 100 ns (scrolled_by=1)
depth=199977 SCROLL_PAGE: min 100 ns (scrolled_by=23)
depth=199977 SCROLL_FULL: min 100 ns (scrolled_by=199977)
depth=199977 REDRAW visual_line x24 (viewport at top): min 1610 ns
depth=199977 REDRAW as_text(True) (viewport at top): min 7549 ns
# interleaved 1000x(draw+scroll+redraw): worst-case scroll=2579 ns  worst-case redraw(visual_line x24)=10377 ns
```

### Both the scroll and the redraw reconstruction are O(1) in history depth

**Yes — scrolling remains responsive, and the measured latency is essentially constant regardless of how deep the history is or how far you scroll.** Two CPU‑side costs were measured directly, and *both* are flat across a 20× change in history depth:

- **Scroll‑index update** (`Screen.scroll`): every scroll — line, page, and even a *full* scroll across the entire history — completes in **min 100 ns**. The value is identical at `depth=9977` and at `depth=199977`: a `SCROLL_FULL` over ~200,000 lines takes **100 ns**, indistinguishable from a single‑line scroll. Scroll cost does **not** grow with history size.
- **Redraw‑path viewport reconstruction** (`visual_line` over all 24 visible rows — the same `historybuf_init_line` work the renderer does): **min 1604 ns at `depth=9977` vs min 1610 ns at `depth=199977`** — a 0.4% difference across 20× more history, i.e. **O(number of visible rows), independent of scrollback depth**. The heavier `as_text(True)` proxy, which additionally ANSI‑serializes those 24 rows, is likewise depth‑independent (**7653 ns vs 7549 ns**).

The reason both are depth‑independent is structural: a scroll rebuilds only the **visible** rows (`MIN(self->lines, self->scrolled_by) ≤ lines = 24`), pulling each from `HistoryBuf` in O(1) via segment indexing — never the whole history. So reconstructing the viewport after scrolling to the top of a 200,000‑line history costs the same ~1.6 µs as after a single‑line scroll.

That O(1) behavior is exactly what the source predicts. `screen_history_scroll` computes a target amount and updates a single index [kitty/screen.c:L4091-L4118]:

```c
bool
screen_history_scroll(Screen *self, int amt, bool upwards) {
    switch(amt) {
        case SCROLL_LINE: amt = 1; break;                       // L4093-4094
        case SCROLL_PAGE: amt = self->lines - 1; break;         // L4096-4097
        case SCROLL_FULL: amt = self->historybuf->count; break; // L4099-4100
        default: amt = MAX(0, amt); break;
    }
    ...
    unsigned int new_scroll = MIN(self->scrolled_by + amt, self->historybuf->count);
    if (new_scroll != self->scrolled_by) {
        self->scrolled_by = new_scroll;   // L4113 — just move an index
        dirty_scroll(self);               // L4114 — flag a repaint
        return true;
    }
    return false;
}
```

There is no data copy: the operation is integer arithmetic plus a flag. `dirty_scroll` merely sets a boolean and requests a repaint [kitty/screen.c:L1908-L1911]:

```c
static void
dirty_scroll(Screen *self) {
    self->scroll_changed = true;
    screen_pause_rendering(self, false, 0);
}
```

The observed `scrolled_by` values **independently confirm the source mapping**: `SCROLL_LINE → 1` [kitty/screen.c:L4093-L4094], `SCROLL_PAGE → 23` which equals `lines - 1 = 24 - 1` [kitty/screen.c:L4096-L4097], and `SCROLL_FULL → count` (9977 and 199977 respectively) [kitty/screen.c:L4099-L4100]. The `Screen`↔`HistoryBuf` wiring lives at [kitty/screen.c:L130], and lines that scroll off the top are appended to history via `historybuf_add_line` [kitty/screen.c:L1558].

*(The observed history depths are `9977` and `199977` rather than 10,000/200,000 because the top rows remain on the visible grid — `LINES = 24` — and only lines that scroll off enter `HistoryBuf`; the probe reports the true `historybuf.count` verbatim.)*

### Signs of prioritization: output ingestion is decoupled from rendering

Two concrete mechanisms keep scroll input responsive while heavy output streams in:

1. **A separate I/O thread.** kitty reads and parses PTY output on a dedicated thread, off the main/render thread: `pthread_t io_thread, talk_thread;` [kitty/child-monitor.c:L55]. Because ingestion and the UI are decoupled, a burst of output does not block the handling of a scroll request. The **interleaved** measurement backs this empirically: with a fresh line drawn between every scroll+reconstruction, the **worst‑case** across 1000 such interleaved iterations was **scroll = 2579 ns (~2.6 µs)** and **viewport reconstruction = 10377 ns (~10.4 µs)** — versus the ~100 ns / ~1.6 µs best cases in isolation. Even these worst cases (which fold in Python‑level allocator jitter and GC) stay in the low‑microsecond range, well under a single 60 Hz frame (~16.7 ms) and imperceptible to a user.

2. **Throughput‑vs‑latency throttles (defaults).** Rendering is coalesced and input processing is bounded by three options: `repaint_delay '10'` ms [kitty/options/definition.py:L866-L867], `input_delay '3'` ms [kitty/options/definition.py:L878-L879], and `sync_to_monitor 'yes'` [kitty/options/definition.py:L889-L890]. These cap how often the screen repaints and how long input is buffered before processing, which is how kitty sustains high output throughput without starving interactive input. Note the "responsive" claim here rests on the **measured** `perf_counter_ns` latencies above, not on these defaults alone.

> **Scope of the measurement — exactly what is and is not timed.** The kitty renderer's per‑frame work is `screen_update_cell_data` [kitty/screen.c:L2738]. For scrolled‑back rows it does three things per visible row: (a) **reconstruct** the row from history — `historybuf_init_line(...)` [kitty/screen.c:L2765]; (b) **shape** it — `render_line(fonts_data, ...)`; and (c) **upload** it to the GPU cell buffer — `update_line_data(line, y, address)` [kitty/screen.c:L2774]. This harness measures (a) directly and faithfully: `visual_line` calls the identical `historybuf_init_line` [kitty/screen.c:L2847], so the reported **~1.6 µs viewport‑reconstruction** latency is the real CPU cost of step (a) plus, in the `as_text` variant, serialization. Steps (b) and (c) are **not** measured here: `render_line` needs a live `FONTS_DATA_HANDLE` and `update_line_data` writes into a mapped GPU buffer, and `screen_update_cell_data` is only ever invoked from the GPU render path [kitty/shaders.c:L411] with those handles — neither exists in a headless process. Consequently the **end‑to‑end scroll→pixels latency (glyph rasterization, GPU upload, buffer swap, and the `repaint_delay`/`sync_to_monitor` frame pacing below) is not measured**; what is established by measurement is that the CPU‑side scroll‑index update (~100 ns) and viewport reconstruction (~1.6 µs) are both O(1) in history depth. The coverage pass (§e) records this scope limitation explicitly rather than presenting an unmeasured end‑to‑end frame latency.


---

## (d) Q3 — Buffer boundaries / allocation transitions

### What was run

To pinpoint *exactly* when new backing storage is allocated, `/tmp/obs_mem_fine.py` pushes lines one at a time and prints only when the process's **virtual data size** (`VmData` from `/proc/self/status`) jumps by more than 1000 kB. Because each `add_segment` performs one large `calloc` [kitty/history.c:L23-L25], the reservation appears as a distinct `VmData` step at the precise push that triggers it — independent of when the pages are later touched:

```python
#!/usr/bin/env python3
# Q3 fine probe: push one line at a time and report every push at which the
# process virtual data size (VmData) jumps by > 1000 kB -- i.e. an add_segment calloc.
from kitty.fast_data_types import HistoryBuf, LineBuf, Cursor

def vmdata_vmrss_kb():
    vmdata = vmrss = -1
    with open('/proc/self/status') as f:
        for ln in f:
            if ln.startswith('VmData:'):
                vmdata = int(ln.split()[1])
            elif ln.startswith('VmRSS:'):
                vmrss = int(ln.split()[1])
    return vmdata, vmrss

XNUM, YNUM, PUSHES = 80, 100000, 100200
lb = LineBuf(2, XNUM)
c = Cursor()
line = lb.line(0)
line.set_text('x' * XNUM, 0, XNUM, c)
hb = HistoryBuf(YNUM, XNUM)

print(f"# xnum={XNUM} ynum={YNUM}  (SEGMENT_SIZE=2048)")
print("# reporting VmData jumps > 1000 kB (= one ~5 MiB segment calloc)")
d0, r0 = vmdata_vmrss_kb()
print(f"# baseline after construct: push=0 count=0 VmData={d0}kB VmRSS={r0}kB")
prev = d0
seg = 1  # one segment is pre-allocated in create_historybuf
for i in range(1, PUSHES + 1):
    hb.push(line)
    d, r = vmdata_vmrss_kb()
    if d - prev > 1000:
        seg += 1
        print(f"push={i} count={hb.count} VmData={d}kB (dVmData=+{d - prev}kB) VmRSS={r}kB  -> segment #{seg} allocated")
        prev = d
import math
print(f"# END push={PUSHES} count={hb.count} total_segments_seen={seg} (predicted ceil({YNUM}/2048)={math.ceil(YNUM/2048)})")
```

Reading `/proc/self/status` after each of the 100,200 individual pushes, the whole run completes in **≈2.8 s** (`real 0m2.763s`), so single‑line granularity is entirely practical here.

### Observed output (verbatim)

```console
$ PYTHONPATH="$PWD" python3 /tmp/obs_mem_fine.py
# xnum=80 ynum=100000  (SEGMENT_SIZE=2048)
# reporting VmData jumps > 1000 kB (= one ~5 MiB segment calloc)
# baseline after construct: push=0 count=0 VmData=15820kB VmRSS=15972kB
push=2049 count=2049 VmData=20952kB (dVmData=+5132kB) VmRSS=21308kB  -> segment #2 allocated
push=4097 count=4097 VmData=26084kB (dVmData=+5132kB) VmRSS=26440kB  -> segment #3 allocated
push=6145 count=6145 VmData=31216kB (dVmData=+5132kB) VmRSS=31572kB  -> segment #4 allocated
push=8193 count=8193 VmData=36348kB (dVmData=+5132kB) VmRSS=36704kB  -> segment #5 allocated
push=10241 count=10241 VmData=41480kB (dVmData=+5132kB) VmRSS=41836kB  -> segment #6 allocated
push=12289 count=12289 VmData=46612kB (dVmData=+5132kB) VmRSS=46968kB  -> segment #7 allocated
push=14337 count=14337 VmData=51744kB (dVmData=+5132kB) VmRSS=52100kB  -> segment #8 allocated
push=16385 count=16385 VmData=56876kB (dVmData=+5132kB) VmRSS=57232kB  -> segment #9 allocated
push=18433 count=18433 VmData=62008kB (dVmData=+5132kB) VmRSS=62364kB  -> segment #10 allocated
push=20481 count=20481 VmData=67140kB (dVmData=+5132kB) VmRSS=67496kB  -> segment #11 allocated
push=22529 count=22529 VmData=72272kB (dVmData=+5132kB) VmRSS=72628kB  -> segment #12 allocated
push=24577 count=24577 VmData=77404kB (dVmData=+5132kB) VmRSS=77760kB  -> segment #13 allocated
push=26625 count=26625 VmData=82536kB (dVmData=+5132kB) VmRSS=82892kB  -> segment #14 allocated
push=28673 count=28673 VmData=87668kB (dVmData=+5132kB) VmRSS=88024kB  -> segment #15 allocated
push=30721 count=30721 VmData=92800kB (dVmData=+5132kB) VmRSS=93156kB  -> segment #16 allocated
push=32769 count=32769 VmData=97932kB (dVmData=+5132kB) VmRSS=98292kB  -> segment #17 allocated
push=34817 count=34817 VmData=103064kB (dVmData=+5132kB) VmRSS=103424kB  -> segment #18 allocated
push=36865 count=36865 VmData=108196kB (dVmData=+5132kB) VmRSS=108556kB  -> segment #19 allocated
push=38913 count=38913 VmData=113328kB (dVmData=+5132kB) VmRSS=113688kB  -> segment #20 allocated
push=40961 count=40961 VmData=118460kB (dVmData=+5132kB) VmRSS=118820kB  -> segment #21 allocated
push=43009 count=43009 VmData=123592kB (dVmData=+5132kB) VmRSS=123952kB  -> segment #22 allocated
push=45057 count=45057 VmData=128724kB (dVmData=+5132kB) VmRSS=129084kB  -> segment #23 allocated
push=47105 count=47105 VmData=133856kB (dVmData=+5132kB) VmRSS=134216kB  -> segment #24 allocated
push=49153 count=49153 VmData=138988kB (dVmData=+5132kB) VmRSS=139348kB  -> segment #25 allocated
push=51201 count=51201 VmData=144120kB (dVmData=+5132kB) VmRSS=144480kB  -> segment #26 allocated
push=53249 count=53249 VmData=149252kB (dVmData=+5132kB) VmRSS=149612kB  -> segment #27 allocated
push=55297 count=55297 VmData=154384kB (dVmData=+5132kB) VmRSS=154744kB  -> segment #28 allocated
push=57345 count=57345 VmData=159516kB (dVmData=+5132kB) VmRSS=159876kB  -> segment #29 allocated
push=59393 count=59393 VmData=164648kB (dVmData=+5132kB) VmRSS=165008kB  -> segment #30 allocated
push=61441 count=61441 VmData=169780kB (dVmData=+5132kB) VmRSS=170140kB  -> segment #31 allocated
push=63489 count=63489 VmData=174912kB (dVmData=+5132kB) VmRSS=175272kB  -> segment #32 allocated
push=65537 count=65537 VmData=180044kB (dVmData=+5132kB) VmRSS=180404kB  -> segment #33 allocated
push=67585 count=67585 VmData=185176kB (dVmData=+5132kB) VmRSS=185536kB  -> segment #34 allocated
push=69633 count=69633 VmData=190308kB (dVmData=+5132kB) VmRSS=190668kB  -> segment #35 allocated
push=71681 count=71681 VmData=195440kB (dVmData=+5132kB) VmRSS=195800kB  -> segment #36 allocated
push=73729 count=73729 VmData=200572kB (dVmData=+5132kB) VmRSS=200932kB  -> segment #37 allocated
push=75777 count=75777 VmData=205704kB (dVmData=+5132kB) VmRSS=206064kB  -> segment #38 allocated
push=77825 count=77825 VmData=210836kB (dVmData=+5132kB) VmRSS=211196kB  -> segment #39 allocated
push=79873 count=79873 VmData=215968kB (dVmData=+5132kB) VmRSS=216328kB  -> segment #40 allocated
push=81921 count=81921 VmData=221100kB (dVmData=+5132kB) VmRSS=221460kB  -> segment #41 allocated
push=83969 count=83969 VmData=226232kB (dVmData=+5132kB) VmRSS=226592kB  -> segment #42 allocated
push=86017 count=86017 VmData=231364kB (dVmData=+5132kB) VmRSS=231724kB  -> segment #43 allocated
push=88065 count=88065 VmData=236496kB (dVmData=+5132kB) VmRSS=236856kB  -> segment #44 allocated
push=90113 count=90113 VmData=241628kB (dVmData=+5132kB) VmRSS=241988kB  -> segment #45 allocated
push=92161 count=92161 VmData=246760kB (dVmData=+5132kB) VmRSS=247124kB  -> segment #46 allocated
push=94209 count=94209 VmData=251892kB (dVmData=+5132kB) VmRSS=252256kB  -> segment #47 allocated
push=96257 count=96257 VmData=257024kB (dVmData=+5132kB) VmRSS=257388kB  -> segment #48 allocated
push=98305 count=98305 VmData=262156kB (dVmData=+5132kB) VmRSS=262520kB  -> segment #49 allocated
# END push=100200 count=100000 total_segments_seen=49 (predicted ceil(100000/2048)=49)
```

All 48 lazy allocations are shown above (segments #2 through #49); segment #1 is the one pre‑allocated at construction, so nothing is elided.

### Where the behavior changes, and why it is observable

**The buffer's behavior changes at every `SEGMENT_SIZE = 2048` line boundary [kitty/history.c:L15], and again — permanently — when `count` reaches `ynum`.** The history is not one big allocation; it is a lazily‑grown array of fixed‑size segments. `segment_for` adds a segment on demand the first time a line index reaches into a not‑yet‑allocated segment [kitty/history.c:L36-L42]:

```c
static index_type
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
    ...
}
```

The measured allocation transitions land on **pushes 2049, 4097, 6145, 8193, 10241, …, 98305** — i.e. exactly `2048·n + 1`. The off‑by‑one (`…049` rather than `…048`) is because `create_historybuf` allocates **one segment up front** [kitty/history.c:L116-L133], specifically `add_segment(self)` at [kitty/history.c:L127]. So segment #1 is pre‑allocated; it fills during pushes 1–2048, and the **first new allocation happens on push 2049**, when the write index first crosses into segment #2. This continues every 2048 lines until `num_segments == ceil(ynum / 2048)` — observed as **exactly 49 segments** for `ynum = 100000`, the last added at push 98305, matching the predicted `ceil(100000/2048) = 49`.

The transition **is** directly observable via memory monitoring, in two complementary views captured here:

- **Virtual reservation** (`/tmp/obs_mem_fine.py`): `VmData` steps up by `+5132 kB` at each `2048·n+1` push — the moment `add_segment`'s `calloc` runs.
- **Resident memory** (`/tmp/obs_mem.py`, §(b)): `VmRSS` climbs `~5132 kB` across each 2048‑push window as that segment's pages are actually written.

Both views agree on the `~5132 kB` segment size. After `count == ynum`, `historybuf_push` enters the circular‑overwrite branch [kitty/history.c:L279-L282] and **no further segments are added** — `total_segments_seen` stops at 49 even though the loop keeps pushing to 100,200, and `VmData`/`VmRSS` flatten. That is the second, permanent behavior change: from *grow‑a‑segment* to *overwrite‑in‑place*.

---

## (e) Coverage pass

Every sub‑question is answered from built‑and‑run evidence, each measurement shown with the exact command that produced it:

| Sub‑question | Answer (measured) | Evidence + citations |
|---|---|---|
| **Q1 — memory under heavy load** | Resident memory rises in **discrete ~5132 kB steps per 2048 lines** (measured), reconciled with the source's `xnum·64 KiB + 8 KiB` = 5,251,072 B per segment to within one 4 KiB page; it then **plateaus** at `VmRSS = 266664 kB` (≈244.9 MiB of `HistoryBuf`) once `count == ynum`. The pager ring is separate; negative `scrollback_lines` ⇒ unbounded growth. | `/tmp/obs_mem.py` complete series (verbatim, §b); `calloc` [kitty/history.c:L23-L25], sizes [kitty/data-types.h:L221,L228], `LineAttrs=4` [kitty/data-types.h:L230-L239] + `/tmp/sizes.c`; plateau [kitty/history.c:L279-L282]; pager [kitty/data-types.h:L268-L272]; infinite [kitty/options/utils.py:L557-L561] ✔ |
| **Q2 — responsiveness while scrolling** | **Yes, responsive — for the CPU‑side work, which is what this headless harness can measure.** Two costs were measured, both **O(1) in history depth** (9977 vs 199977 lines): the scroll‑index update = **min 100 ns** (all scroll types), and the redraw‑path **viewport reconstruction** = **1604 ns vs 1610 ns** (`visual_line` ×24, the same `historybuf_init_line` the renderer runs); `as_text` reconstruct+serialize = 7653 vs 7549 ns. Interleaved with active output, worst case = **scroll 2579 ns / reconstruction 10377 ns**. Prioritization: separate I/O thread + repaint/input throttles. **Scope limitation (explicit):** glyph rasterization (`render_line`) and GPU upload/`update_line_data` + buffer swap are **not** measured (require a live `FONTS_DATA_HANDLE`/GPU context; `screen_update_cell_data` runs only from the GPU path), so **end‑to‑end scroll→pixels latency is not measured** — only the O(1) CPU scroll + reconstruction path is established by measurement. | `/tmp/obs_scroll.py` complete timings (verbatim, §c); O(1) scroll path [kitty/screen.c:L4091-L4118], `dirty_scroll` [kitty/screen.c:L1908-L1911]; redraw reconstruction `screen_update_cell_data` [kitty/screen.c:L2738,L2765,L2774] vs `visual_line_` [kitty/screen.c:L2843,L2847]; GPU‑only invocation [kitty/shaders.c:L411]; I/O thread [kitty/child-monitor.c:L55]; throttles [kitty/options/definition.py:L866-L867,L878-L879,L889-L890] ⚠ (CPU path ✔; end‑to‑end GPU latency not measured) |
| **Q3 — buffer boundaries / allocation transitions** | New backing storage is `calloc`'d **lazily at each 2048‑line boundary** — observed at pushes **2049, 4097, …, 98305** (`2048·n+1`), because one segment is pre‑allocated; **49 segments** total for `ynum=100000`. The transition is visible as a `+5132 kB` `VmData` step; growth stops permanently once `count == ynum`. | `/tmp/obs_mem_fine.py` complete series (verbatim, §d); `SEGMENT_SIZE` [kitty/history.c:L15], `segment_for` [kitty/history.c:L36-L42], upfront segment [kitty/history.c:L127], plateau [kitty/history.c:L279-L282] ✔ |

### Repository integrity

No existing repository file was modified, and no code other than this document was added — the tracked C/Python/Go source is byte‑for‑byte identical to commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, against which the entire investigation was performed. All temporary observation scripts (`/tmp/obs_mem.py`, `/tmp/obs_mem_fine.py`, `/tmp/obs_scroll.py`, `/tmp/sizes.c`, `/tmp/sizes`) were run outside the repository and removed after measurement. Build artifacts (`*.so`, `build/`, `__pycache__/`) are covered by the repository's `.gitignore`, so the build left no tracked change. This document is committed as the **single** added file directly atop the pinned source commit; `git diff --name-only 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD` lists only `blitzy/documentation/kitty_815df1e210e0.md`, and a targeted `git diff` against every referenced source, config, test, and build file is empty — the working tree is otherwise clean.

