# kitty scrollback `HistoryBuf` under heavy load — empirical investigation

**Repository:** kovidgoyal/kitty  **HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Deliverable:** answer to three questions about the scrollback history buffer, produced by **building and running kitty first** and writing every claim from captured runtime output.

This document answers:

- **Q1 — Memory under massive output.** As kitty emits hundreds of thousands of lines and scrollback accumulates, what happens to process memory? (measured RSS before / during / after, at scale, across ≥2 runs)
- **Q2 — Responsiveness & latency during concurrent activity.** While scrolling back through a large history *as new output is still being produced*, does the terminal stay responsive, what is the input‑to‑display latency, and are there visible signs of one operation being prioritized over another?
- **Q3 — Buffer growth boundaries.** When does new backing storage get allocated as the buffer grows, and can that transition be observed through external memory monitoring?

All measurements were taken **inside the mandated Docker image** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (`Id sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288`). Q1/Q3 use the canonical headless `Screen`/`HistoryBuf` path (`kitty_tests.create_screen` + `parse_bytes`, which drive the real VT parser); Q2 uses the **full kitty GUI binary** from that image rendering to a display, with real key injection and real framebuffer read‑back. Temporary observation scripts lived outside the repository and were removed; the only repository change is this file.

---

## Summary of findings

| # | Question | Headline result (measured, canonical, ≥2 runs) | Where |
|---|----------|-----------------------------------------------|-------|
| Q1 | Memory under massive output | RSS grows **≈ 5.13 MB per 2,048 lines** of scrollback (one `HistoryBuf` segment). Default (`scrollback_lines=2000`) tops out at **≈ 47 MB** (one segment). Feeding 500,000 lines into a 300,000‑line buffer reaches **≈ 794 MB** (147 segments) then **stops growing**. An "infinite" buffer fed 200,000 lines reaches **≈ 543 MB** (98 segments) and keeps climbing. Python‑level `tracemalloc` stays ≈ 12 MB throughout — the growth is **C `calloc`, invisible to Python allocation tracing**. | [Q1](#q1--memory-consumption-under-massive-output) |
| Q2 | Responsiveness / latency | The terminal **stays responsive**. Scroll‑to‑render latency (real `ctrl+shift+home` injected, real framebuffer change detected): **idle ≈ 7 ms**; **during continuous output ≈ 14.8 ms median (default), ≈ 10.4 ms median (low‑latency‑tuned)**, worst case ≈ 16 ms — far below the ~100 ms human‑perceptible threshold. The during‑output distribution is **bimodal** (≈ 6.5 ms when the scroll lands in an output gap, ≈ 15 ms when it coincides with a repaint tick): the observable sign that a scroll **shares the main thread's throttled repaint cadence** with output rendering. | [Q2](#q2--responsiveness-and-latency-during-concurrent-activity) |
| Q3 | Buffer growth boundaries | New storage is allocated **one `SEGMENT_SIZE = 2048`‑row block at a time, on demand**, the instant a line is written into a not‑yet‑allocated block — observed externally as a **+5,132 kB `VmSize` step exactly as history `count` crosses 2048 → 2049**. The first segment is reserved **upfront** at `Screen` creation. Growth **plateaus** once `count == ynum` (capacity), after which the oldest line is overwritten circularly. A negative `scrollback_lines` maps to `2**32‑1`, so growth is effectively **unbounded** until `add_segment`'s `fatal("Out of memory")`. | [Q3](#q3--buffer-growth-boundaries) |

**Key correction over a naïve reading of the source:** `sizeof(LineAttrs)` is **4 bytes, not 1** — the `LineAttrs` union contains a `PromptKind prompt_kind : 2` bit‑field and `PromptKind` is an `int`‑backed enum, so the union is 4‑byte‑aligned/sized. The correct 80‑column segment backing size is therefore **5,251,072 bytes** (not 5,244,928), and the measured resident/virtual step is one 4,096‑byte page above that (`5,255,168 B`). This is confirmed below by the compiled size, the external memory step, and the kitty test suite.

---

## Environment & reproducibility

Every value below was produced **inside the mandated Docker image**. Image identity and toolchain:

```
RepoTags=[ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0] Id=sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288
IMG_HEAD=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
Python 3.12.3
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
```

Two build artefacts are relevant:

- The image ships a **pre‑built release** `kitty/fast_data_types.so` (1,221,264 bytes, dated Aug 2025). **All Q1/Q3 headless measurements and the kitty test suite below import this exact shipped module** (`so path imported: /app/kitty/fast_data_types.so`).
- To prove the segment math is **build‑independent**, the boundary experiment was *also* re‑run against a fresh local **debug** build produced with the canonical command:

```
$ touch kitty/history.c kitty/screen.c kitty/line-buf.c
$ PATH=$PATH:/usr/local/go/bin CI=true python3 setup.py build --debug --ignore-compiler-warnings
[1/122] Compiling kitty/screen.c ...
...
[10/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
--- resulting module ---
-rwxr-xr-x 1 root 1001 6142824 Jul 13 18:01 kitty/fast_data_types.so
```

(`--ignore-compiler-warnings` only bypasses a `-Werror=switch` in the vendored GLFW Wayland backend [setup.py:2003-2004]; it is unrelated to the history buffer. The debug `.so` is 6,142,824 bytes; the shipped release `.so` is 1,221,264 bytes. Both yield the identical 2048‑line / +5,132 kB boundary step — see Q3.)

**Canonical path (no bypass).** The headless driver is kitty's own test harness: `create_screen(cols, lines, scrollback, ...)` [kitty_tests/__init__.py:237] builds a real `Screen`, and `parse_bytes(screen, data)` [kitty_tests/__init__.py:30] feeds raw bytes through the **real VT parser** — `parse_bytes` → `Screen.test_create_write_buffer` → `vt_parser_create_write_buffer` [kitty/vt-parser.c:1451] → `test_commit_write_buffer` → `vt_parser_commit_write` [kitty/vt-parser.c:1465] → `parse_worker` [kitty/vt-parser.c:1496] → the screen ops that scroll lines off the top via the `INDEX_UP` macro [kitty/screen.c:1552] → `historybuf_add_line` [kitty/history.c:287-291] → `historybuf_push` [kitty/history.c:276-285]. At no point is `HistoryBuf.push()` called directly; this is exactly the path taken by a child program's PTY output. The harness forces `scrollback_pager_history_size` to a non‑zero test value at [kitty_tests/__init__.py:224], so every headless run **explicitly overrides it back to `0`** to reproduce the true default [kitty/options/definition.py:406].

**Q2 environment.** The GUI binary `./kitty/launcher/kitty` from the image renders to a host‑provided `Xvfb` display (`1280x800x24`), software GL (`LIBGL_ALWAYS_SOFTWARE=1`, Mesa 24.2.8 llvmpipe). kitty requires OpenGL ≥ 3.3 [kitty/data-types.h:19-21]. GUI rendering is proven by kitty's own startup log (`OS Window created`, `Child launched` under `--debug-rendering`), by the 32 Mesa `llvmpipe-*` GL worker threads, and by successful framebuffer read‑back (the scroll detector below). The mandated image itself has no `Xvfb`, so the display is supplied by the host and shared into the container via the X socket; the **kitty binary under test is the image's own** — see the exact commands in the [methodology appendix](#appendix-b--q2-gui-latency-harness).

> **Note on the "PIL is unavailable" claim in the prior draft.** That claim was false and is retracted: Pillow with `ImageGrab` is present both on the host (12.3.0) and in the mandated image (11.3.0). Q2 does not depend on it — the latency detector reads the framebuffer directly through `libX11`'s `XGetImage`.

---

## Q1 — Memory consumption under massive output

### Mechanism (grounded in source)

Scrollback lines live in a `HistoryBuf` [kitty/data-types.h:281-290], which stores rows in fixed blocks of `SEGMENT_SIZE = 2048` rows [kitty/history.c:15]. Each block is one `HistoryBufSegment` [kitty/data-types.h:261-266] allocated by a single `calloc` in `add_segment()` [kitty/history.c:18-29] of size:

```
xnum*2048*sizeof(CPUCell) + xnum*2048*sizeof(GPUCell) + 2048*sizeof(LineAttrs)
```

With `sizeof(CPUCell)==12` [kitty/data-types.h:228], `sizeof(GPUCell)==20` [kitty/data-types.h:221], and **`sizeof(LineAttrs)==4`** [kitty/data-types.h:231-239], an 80‑column segment is:

```
80*2048*(12+20) + 2048*4 = 5,242,880 + 8,192 = 5,251,072 bytes  (5.0078 MiB)
```

`create_historybuf()` reserves the **first** segment upfront at `Screen` creation [kitty/history.c:117-132]; further segments are requested **on demand** by `segment_for()` [kitty/history.c:37-42] as lines scroll off the top and are pushed by `historybuf_push()` [kitty/history.c:276-285]. The number of in‑memory rows is fixed at `Screen` creation as `ynum = MAX(scrollback, lines)` [kitty/screen.c:130].

### What was measured

A hardened observer (Appendix A) drives the **canonical** `parse_bytes` path into a `create_screen` `Screen` and samples, at every phase, `/proc/self/status` `VmRSS`/`VmSize`/`VmData`, `/proc/self/smaps_rollup` `Rss`/`Pss`, `getrusage().ru_maxrss`, and `tracemalloc` current/peak. Three conditions, **three identical runs each** (the run index only changes the header), complete unedited output shown for every run:

- **C1 — default** `scrollback_lines=2000`, feed 5,000 lines.
- **C2 — large finite** `scrollback=300000`, feed 500,000 lines.
- **C3 — negative / "infinite"** `scrollback_lines("-1") → 2**32‑1` [kitty/options/utils.py:557-561], feed 200,000 lines.

Command (run inside the image, from `/app`, once per condition/index):

```
BLITZY_REPO=/app python3 <workdir>/obs_mem.py <c1|c2|c3> <run-index>
```

### C1 — default `scrollback_lines=2000` (feed 5,000) — three runs

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c1 1`

```text
PREFLIGHT MemAvailable=3926228784 kB  required(safe)>=200000 kB
============================================================================================
C1 DEFAULT (scrollback_lines=2000, pager=0)  [run 1]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11912 kB
post-import VmSize=64756 kB VmRSS=41352 kB
post-create_screen(scrollback=2000) VmSize=69888 kB VmRSS=41604 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+252 kB)  ynum=2000 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69888       41604      41604      41600      39936    12243138             
fine seg~1         2048       2000     1       70508       46992      46992      46988      45056    12243138         +620
plateau            2548       2000     1       70508       46992      46992      46988      45056    12243138           +0
plateau            3048       2000     1       70508       46992      46992      46988      45056    12243138           +0
plateau            3548       2000     1       70508       46992      46992      46988      45056    12243138           +0
plateau            4048       2000     1       70508       46992      46992      46988      45056    12243138           +0
plateau            4548       2000     1       70508       46992      46992      46988      45056    12243138           +0
plateau            5000       2000     1       70508       46992      46992      46988      45056    12243138           +0
after              5000       2000     1       70508       46992      46992      46988      45056    12243138           +0
--------------------------------------------------------------------------------------------
FINAL count=2000 ynum=2000 inferred_segments=1 (num_segments NOT exposed -> inferred)
FINAL VmRSS=46992 kB VmSize=70508 kB smapsRss=46992 smapsPss=46988 ru_maxrss=45056 kB
FINAL tracemalloc current=7008210 B peak=12243138 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 2952 lines in 0.0173 s = 170378 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 1 * 5251072 B = 5128 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c1 2`

```text
PREFLIGHT MemAvailable=3926233972 kB  required(safe)>=200000 kB
============================================================================================
C1 DEFAULT (scrollback_lines=2000, pager=0)  [run 2]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11916 kB
post-import VmSize=64756 kB VmRSS=41356 kB
post-create_screen(scrollback=2000) VmSize=69888 kB VmRSS=41608 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+252 kB)  ynum=2000 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69888       41608      41608      41604      39936    12242968             
fine seg~1         2048       2000     1       70508       46996      46996      46992      45056    12242968         +620
plateau            2548       2000     1       70508       46996      46996      46992      45056    12242968           +0
plateau            3048       2000     1       70508       46996      46996      46992      45056    12242968           +0
plateau            3548       2000     1       70508       46996      46996      46992      45056    12242968           +0
plateau            4048       2000     1       70508       46996      46996      46992      45056    12242968           +0
plateau            4548       2000     1       70508       46996      46996      46992      45056    12242968           +0
plateau            5000       2000     1       70508       46996      46996      46992      45056    12242968           +0
after              5000       2000     1       70508       46996      46996      46992      45056    12242968           +0
--------------------------------------------------------------------------------------------
FINAL count=2000 ynum=2000 inferred_segments=1 (num_segments NOT exposed -> inferred)
FINAL VmRSS=46996 kB VmSize=70508 kB smapsRss=46996 smapsPss=46992 ru_maxrss=45056 kB
FINAL tracemalloc current=7011360 B peak=12242968 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 2952 lines in 0.0176 s = 167783 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 1 * 5251072 B = 5128 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c1 3`

```text
PREFLIGHT MemAvailable=3926239068 kB  required(safe)>=200000 kB
============================================================================================
C1 DEFAULT (scrollback_lines=2000, pager=0)  [run 3]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11976 kB
post-import VmSize=64752 kB VmRSS=41560 kB
post-create_screen(scrollback=2000) VmSize=69884 kB VmRSS=41824 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+264 kB)  ynum=2000 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69884       41824      41824      41820      40960    12243143             
fine seg~1         2048       2000     1       70508       47304      47304      47300      46080    12243143         +624
plateau            2548       2000     1       70508       47304      47304      47300      46080    12243143           +0
plateau            3048       2000     1       70508       47304      47304      47300      46080    12243143           +0
plateau            3548       2000     1       70508       47304      47304      47300      46080    12243143           +0
plateau            4048       2000     1       70508       47304      47304      47300      46080    12243143           +0
plateau            4548       2000     1       70508       47304      47304      47300      46080    12243143           +0
plateau            5000       2000     1       70508       47304      47304      47300      46080    12243143           +0
after              5000       2000     1       70508       47304      47304      47300      46080    12243143           +0
--------------------------------------------------------------------------------------------
FINAL count=2000 ynum=2000 inferred_segments=1 (num_segments NOT exposed -> inferred)
FINAL VmRSS=47304 kB VmSize=70508 kB smapsRss=47304 smapsPss=47300 ru_maxrss=46080 kB
FINAL tracemalloc current=7010837 B peak=12243143 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 2952 lines in 0.0279 s = 105943 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 1 * 5251072 B = 5128 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**C1 reading.** `post-create_screen` already shows `VmSize +5,132 kB` (the upfront first segment reserved) but only `+~252 kB` `VmRSS` — the `calloc`'d pages are not yet resident (lazy fault‑in). Feeding 2,048 lines fills that one segment: `VmRSS` rises to **≈ 47 MB** with **no new** `VmSize` step (`+620 kB` of touched pages, still one segment). Because `ynum=2000 < SEGMENT_SIZE=2048`, the default terminal **uses exactly one segment** and `count` plateaus at 2,000; every subsequent sample is byte‑identical (`+0`). Final `VmRSS` across the three runs: **46,992 / 46,996 / 47,304 kB** (spread 312 kB, 0.66 %).

### C2 — large finite `scrollback=300000` (feed 500,000) — three runs

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c2 1`

```text
PREFLIGHT MemAvailable=3926233524 kB  required(safe)>=1200000 kB
============================================================================================
C2 LARGE FINITE (scrollback=300000, feed 500000, pager=0)  [run 1]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11904 kB
post-import VmSize=64760 kB VmRSS=41248 kB
post-create_screen(scrollback=300000) VmSize=69892 kB VmRSS=41504 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+256 kB)  ynum=300000 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69892       41504      41504      41500      39936    12243406             
fine seg~1         2048       2025     1       70512       46956      46956      46952      46080    12243406         +620
fine seg~2         4096       4073     2       75644       52088      52088      52084      51200    12243406        +5132
fine seg~3         6144       6121     3       80776       57220      57220      57216      56320    12243406        +5132
fine seg~4         8192       8169     4       85908       62352      62352      62348      61440    12243406        +5132
fine seg~5        10240      10217     5       91040       67484      67484      67480      66560    12243406        +5132
fine seg~6        12288      12265     6       96172       72616      72616      72612      71680    12243406        +5132
fine seg~7        14336      14313     7      101304       77748      77748      77744      76800    12243406        +5132
fine seg~8        16384      16361     8      106436       82880      82880      82876      81920    12243406        +5132
fine seg~9        18432      18409     9      111568       88012      88012      88008      87040    12243406        +5132
fine seg~10       20480      20457    10      116700       93144      93144      93140      92160    12243406        +5132
fine seg~11       22528      22505    11      121832       98276      98276      98272      97280    12243406        +5132
fine seg~12       24576      24553    12      126964      103408     103408     103404     102400    12243406        +5132
fine seg~13       26624      26601    13      132096      108540     108540     108536     107520    12243406        +5132
fine seg~14       28672      28649    14      137228      113672     113672     113668     112640    12243406        +5132
fine seg~15       30720      30697    15      142360      118804     118804     118800     117760    12243406        +5132
fine seg~16       32768      32745    16      147492      123936     123936     123932     122880    12243406        +5132
fine seg~17       34816      34793    17      152624      129068     129068     129064     128000    12243406        +5132
fine seg~18       36864      36841    18      157756      134200     134200     134196     133120    12243406        +5132
fine seg~19       38912      38889    19      162888      139332     139332     139328     138240    12243406        +5132
fine seg~20       40960      40937    20      168020      144464     144464     144460     143360    12243406        +5132
during           115960     115937    57      357904      332404     332404     332400     330752    12243406      +189884
during           190960     190937    94      547788      520344     520344     520340     519168    12243406      +189884
during           265960     265937   130      732540      708280     708280     708276     706560    12243406      +184752
plateau          340960     300000   147      819784      793640     793640     793636     792576    12243406       +87244
plateau          415960     300000   147      819784      793640     793640     793636     792576    12243406           +0
plateau          490960     300000   147      819784      793640     793640     793636     792576    12243406           +0
plateau          500000     300000   147      819784      793640     793640     793636     792576    12243406           +0
after            500000     300000   147      819784      793640     793640     793636     792576    12243406           +0
--------------------------------------------------------------------------------------------
FINAL count=300000 ynum=300000 inferred_segments=147 (num_segments NOT exposed -> inferred)
FINAL VmRSS=793640 kB VmSize=819784 kB smapsRss=793640 smapsPss=793636 ru_maxrss=792576 kB
FINAL tracemalloc current=7039305 B peak=12243406 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 459040 lines in 0.5431 s = 845145 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 147 * 5251072 B = 753816 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c2 2`

```text
PREFLIGHT MemAvailable=3926311264 kB  required(safe)>=1200000 kB
============================================================================================
C2 LARGE FINITE (scrollback=300000, feed 500000, pager=0)  [run 2]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=12084 kB
post-import VmSize=64760 kB VmRSS=41536 kB
post-create_screen(scrollback=300000) VmSize=69892 kB VmRSS=41796 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+260 kB)  ynum=300000 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69892       41796      41796      41792      40960    12243000             
fine seg~1         2048       2025     1       70512       47244      47244      47240      47104    12243000         +620
fine seg~2         4096       4073     2       75644       52376      52376      52372      52224    12243000        +5132
fine seg~3         6144       6121     3       80776       57508      57508      57504      57344    12243000        +5132
fine seg~4         8192       8169     4       85908       62640      62640      62636      62464    12243000        +5132
fine seg~5        10240      10217     5       91040       67772      67772      67768      67584    12243000        +5132
fine seg~6        12288      12265     6       96172       72904      72904      72900      72704    12243000        +5132
fine seg~7        14336      14313     7      101304       78036      78036      78032      77824    12243000        +5132
fine seg~8        16384      16361     8      106436       83168      83168      83164      82944    12243000        +5132
fine seg~9        18432      18409     9      111568       88300      88300      88296      88064    12243000        +5132
fine seg~10       20480      20457    10      116700       93432      93432      93428      93184    12243000        +5132
fine seg~11       22528      22505    11      121832       98564      98564      98560      98304    12243000        +5132
fine seg~12       24576      24553    12      126964      103696     103696     103692     103424    12243000        +5132
fine seg~13       26624      26601    13      132096      108828     108828     108824     108544    12243000        +5132
fine seg~14       28672      28649    14      137228      113960     113960     113956     113664    12243000        +5132
fine seg~15       30720      30697    15      142360      119092     119092     119088     118784    12243000        +5132
fine seg~16       32768      32745    16      147492      124224     124224     124220     123904    12243000        +5132
fine seg~17       34816      34793    17      152624      129356     129356     129352     129024    12243000        +5132
fine seg~18       36864      36841    18      157756      134488     134488     134484     134144    12243000        +5132
fine seg~19       38912      38889    19      162888      139620     139620     139616     139264    12243000        +5132
fine seg~20       40960      40937    20      168020      144752     144752     144748     144384    12243000        +5132
during           115960     115937    57      357904      332692     332692     332688     331776    12243000      +189884
during           190960     190937    94      547788      520632     520632     520628     520192    12243000      +189884
during           265960     265937   130      732540      708568     708568     708564     707584    12243000      +184752
plateau          340960     300000   147      819784      793928     793928     793924     793600    12243000       +87244
plateau          415960     300000   147      819784      793928     793928     793924     793600    12243000           +0
plateau          490960     300000   147      819784      793928     793928     793924     793600    12243000           +0
plateau          500000     300000   147      819784      793928     793928     793924     793600    12243000           +0
after            500000     300000   147      819784      793928     793928     793924     793600    12243000           +0
--------------------------------------------------------------------------------------------
FINAL count=300000 ynum=300000 inferred_segments=147 (num_segments NOT exposed -> inferred)
FINAL VmRSS=793928 kB VmSize=819784 kB smapsRss=793928 smapsPss=793924 ru_maxrss=793600 kB
FINAL tracemalloc current=7040553 B peak=12243000 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 459040 lines in 0.5824 s = 788245 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 147 * 5251072 B = 753816 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c2 3`

```text
PREFLIGHT MemAvailable=3926345572 kB  required(safe)>=1200000 kB
============================================================================================
C2 LARGE FINITE (scrollback=300000, feed 500000, pager=0)  [run 3]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11992 kB
post-import VmSize=64752 kB VmRSS=41588 kB
post-create_screen(scrollback=300000) VmSize=69884 kB VmRSS=41916 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+328 kB)  ynum=300000 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69884       41916      41916      41912      40960    12242934             
fine seg~1         2048       2025     1       70504       47368      47368      47364      47104    12242934         +620
fine seg~2         4096       4073     2       75636       52500      52500      52496      52224    12242934        +5132
fine seg~3         6144       6121     3       80768       57632      57632      57628      57344    12242934        +5132
fine seg~4         8192       8169     4       85900       62764      62764      62760      62464    12242934        +5132
fine seg~5        10240      10217     5       91032       67896      67896      67892      67584    12242934        +5132
fine seg~6        12288      12265     6       96164       73028      73028      73024      72704    12242934        +5132
fine seg~7        14336      14313     7      101296       78160      78160      78156      77824    12242934        +5132
fine seg~8        16384      16361     8      106428       83292      83292      83288      82944    12242934        +5132
fine seg~9        18432      18409     9      111560       88424      88424      88420      88064    12242934        +5132
fine seg~10       20480      20457    10      116692       93556      93556      93552      93184    12242934        +5132
fine seg~11       22528      22505    11      121824       98688      98688      98684      98304    12242934        +5132
fine seg~12       24576      24553    12      126956      103820     103820     103816     103424    12242934        +5132
fine seg~13       26624      26601    13      132088      108952     108952     108948     108544    12242934        +5132
fine seg~14       28672      28649    14      137220      114084     114084     114080     113664    12242934        +5132
fine seg~15       30720      30697    15      142352      119216     119216     119212     118784    12242934        +5132
fine seg~16       32768      32745    16      147484      124348     124348     124344     123904    12242934        +5132
fine seg~17       34816      34793    17      152616      129480     129480     129476     129024    12242934        +5132
fine seg~18       36864      36841    18      157748      134612     134612     134608     134144    12242934        +5132
fine seg~19       38912      38889    19      162880      139744     139744     139740     139264    12242934        +5132
fine seg~20       40960      40937    20      168012      144876     144876     144872     144384    12242934        +5132
during           115960     115937    57      357896      332816     332816     332812     331776    12242934      +189884
during           190960     190937    94      547780      520756     520756     520752     520192    12242934      +189884
during           265960     265937   130      732532      708692     708692     708688     707584    12242934      +184752
plateau          340960     300000   147      819776      794052     794052     794048     793600    12242934       +87244
plateau          415960     300000   147      819776      794052     794052     794048     793600    12242934           +0
plateau          490960     300000   147      819776      794052     794052     794048     793600    12242934           +0
plateau          500000     300000   147      819776      794052     794052     794048     793600    12242934           +0
after            500000     300000   147      819776      794052     794052     794048     793600    12242934           +0
--------------------------------------------------------------------------------------------
FINAL count=300000 ynum=300000 inferred_segments=147 (num_segments NOT exposed -> inferred)
FINAL VmRSS=794052 kB VmSize=819776 kB smapsRss=794052 smapsPss=794048 ru_maxrss=793600 kB
FINAL tracemalloc current=7044612 B peak=12242934 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 459040 lines in 0.5872 s = 781716 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 147 * 5251072 B = 753816 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**C2 reading.** The fine phase shows the signature clearly: the first block reuses the upfront segment (`+620 kB` touched, still 1 segment), then **each additional 2,048 lines adds one segment as a clean `+5,132 kB` `VmSize` step** (`fine seg~2 … seg~20`). The coarse phase keeps climbing (`during` rows) until `count == ynum == 300000` at **147 segments**, after which memory **plateaus** — the last four samples are byte‑identical at `VmRSS = 793,640 kB` (**≈ 794 MB**) even though 160,000 more lines were fed. Final `VmRSS` across three runs: **793,640 / 793,928 / 794,052 kB** (spread 412 kB, **0.05 %**). Throughput of the canonical parse path (timed with `perf_counter` over the coarse window only, divided by the exact lines fed in it): **≈ 845,000 lines/s**. `tracemalloc` peak stays flat at ≈ 12.2 MB while RSS climbs to 794 MB — proving the growth is **C `calloc`**, not Python objects.

### C3 — negative / "infinite" `scrollback_lines("-1") → 2**32‑1` (feed 200,000) — three runs

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c3 1`

```text
PREFLIGHT MemAvailable=3926324832 kB  required(safe)>=900000 kB
============================================================================================
C3 NEGATIVE/INFINITE (scrollback_lines(-1)=4294967295, feed 200000, pager=0)  [run 1]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11932 kB
post-import VmSize=64752 kB VmRSS=41376 kB
post-create_screen(scrollback=4294967295) VmSize=69884 kB VmRSS=41816 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+440 kB)  ynum=4294967295 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69884       41816      41816      41812      40960    12243864             
fine seg~1         2048       2025     1       70504       47268      47268      47264      47104    12243864         +620
fine seg~2         4096       4073     2       75636       52400      52400      52396      52224    12243864        +5132
fine seg~3         6144       6121     3       80768       57532      57532      57528      57344    12243864        +5132
fine seg~4         8192       8169     4       85900       62664      62664      62660      62464    12243864        +5132
fine seg~5        10240      10217     5       91032       67796      67796      67792      67584    12243864        +5132
fine seg~6        12288      12265     6       96164       72928      72928      72924      72704    12243864        +5132
fine seg~7        14336      14313     7      101296       78060      78060      78056      77824    12243864        +5132
fine seg~8        16384      16361     8      106428       83192      83192      83188      82944    12243864        +5132
fine seg~9        18432      18409     9      111560       88324      88324      88320      88064    12243864        +5132
fine seg~10       20480      20457    10      116692       93456      93456      93452      93184    12243864        +5132
during            70480      70457    35      244992      218752     218752     218748     218112    12243864      +128300
during           120480     120457    59      368160      344044     344044     344040     343040    12243864      +123168
during           170480     170457    84      496460      469336     469336     469332     468992    12243864      +128300
during           200000     199977    98      568308      543312     543312     543308     542720    12243864       +71848
after            200000     199977    98      568308      543312     543312     543308     542720    12243864           +0
--------------------------------------------------------------------------------------------
FINAL count=199977 ynum=4294967295 inferred_segments=98 (num_segments NOT exposed -> inferred)
FINAL VmRSS=543312 kB VmSize=568308 kB smapsRss=543312 smapsPss=543308 ru_maxrss=542720 kB
FINAL tracemalloc current=7024545 B peak=12243864 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 179520 lines in 0.3155 s = 569008 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 98 * 5251072 B = 502544 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c3 2`

```text
PREFLIGHT MemAvailable=3926339848 kB  required(safe)>=900000 kB
============================================================================================
C3 NEGATIVE/INFINITE (scrollback_lines(-1)=4294967295, feed 200000, pager=0)  [run 2]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=12096 kB
post-import VmSize=64756 kB VmRSS=41708 kB
post-create_screen(scrollback=4294967295) VmSize=69888 kB VmRSS=41972 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+264 kB)  ynum=4294967295 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69888       41972      41972      41968      40960    12243915             
fine seg~1         2048       2025     1       70348       47648      47648      47644      47104    12243915         +460
fine seg~2         4096       4073     2       75480       52780      52780      52776      52224    12243915        +5132
fine seg~3         6144       6121     3       80612       57912      57912      57908      57344    12243915        +5132
fine seg~4         8192       8169     4       85744       63044      63044      63040      62464    12243915        +5132
fine seg~5        10240      10217     5       90876       68176      68176      68172      67584    12243915        +5132
fine seg~6        12288      12265     6       96008       73308      73308      73304      72704    12243915        +5132
fine seg~7        14336      14313     7      101140       78440      78440      78436      77824    12243915        +5132
fine seg~8        16384      16361     8      106272       83572      83572      83568      82944    12243915        +5132
fine seg~9        18432      18409     9      111404       88704      88704      88700      88064    12243915        +5132
fine seg~10       20480      20457    10      116536       93836      93836      93832      93184    12243915        +5132
during            70480      70457    35      244836      219132     219132     219128     218112    12243915      +128300
during           120480     120457    59      368004      344424     344424     344420     344064    12243915      +123168
during           170480     170457    84      496304      469716     469716     469712     468992    12243915      +128300
during           200000     199977    98      568152      543692     543692     543688     542720    12243915       +71848
after            200000     199977    98      568152      543692     543692     543688     542720    12243915           +0
--------------------------------------------------------------------------------------------
FINAL count=199977 ynum=4294967295 inferred_segments=98 (num_segments NOT exposed -> inferred)
FINAL VmRSS=543692 kB VmSize=568152 kB smapsRss=543692 smapsPss=543688 ru_maxrss=542720 kB
FINAL tracemalloc current=7024871 B peak=12243915 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 179520 lines in 0.2930 s = 612639 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 98 * 5251072 B = 502544 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**Command:** `BLITZY_REPO=/app python3 obs_mem.py c3 3`

```text
PREFLIGHT MemAvailable=3926321340 kB  required(safe)>=900000 kB
============================================================================================
C3 NEGATIVE/INFINITE (scrollback_lines(-1)=4294967295, feed 200000, pager=0)  [run 3]  pid=1  cols=80 lines=24
============================================================================================
canonical constants: SIZEOF_CPUCELL=12 [data-types.h:228]  SIZEOF_GPUCELL=20 [data-types.h:221]  SIZEOF_LINEATTRS=4 [data-types.h:231-239]
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)  [add_segment kitty/history.c:18-29]
scrollback_lines('-1') canonical map: 2**32-1 = 4294967295  [utils.py:557-561]
pre-import  VmSize=17316 kB VmRSS=11936 kB
post-import VmSize=64760 kB VmRSS=41384 kB
post-create_screen(scrollback=4294967295) VmSize=69892 kB VmRSS=41824 kB  (upfront ONE segment: dVmSize=+5132 dVmRSS=+440 kB)  ynum=4294967295 count=0
phase         lines_fed      count  segs  VmSize(kB)   VmRSS(kB)   smapsRss   smapsPss  ru_maxrss   tmPeak(B)  dVmSize(kB)
--------------------------------------------------------------------------------------------------------------------------
before                0          0     1       69892       41824      41824      41820      39936    12244141             
fine seg~1         2048       2025     1       70512       47276      47276      47272      45056    12244141         +620
fine seg~2         4096       4073     2       75644       52408      52408      52404      50176    12244141        +5132
fine seg~3         6144       6121     3       80776       57540      57540      57536      56320    12244141        +5132
fine seg~4         8192       8169     4       85908       62672      62672      62668      61440    12244141        +5132
fine seg~5        10240      10217     5       91040       67804      67804      67800      66560    12244141        +5132
fine seg~6        12288      12265     6       96172       72936      72936      72932      71680    12244141        +5132
fine seg~7        14336      14313     7      101304       78068      78068      78064      76800    12244141        +5132
fine seg~8        16384      16361     8      106436       83200      83200      83196      81920    12244141        +5132
fine seg~9        18432      18409     9      111568       88332      88332      88328      87040    12244141        +5132
fine seg~10       20480      20457    10      116700       93464      93464      93460      92160    12244141        +5132
during            70480      70457    35      245000      218760     218760     218756     217088    12244141      +128300
during           120480     120457    59      368168      344052     344052     344048     342016    12244141      +123168
during           170480     170457    84      496468      469344     469344     469340     467968    12244141      +128300
during           200000     199977    98      568316      543320     543320     543316     541696    12244141       +71848
after            200000     199977    98      568316      543320     543320     543316     541696    12244141           +0
--------------------------------------------------------------------------------------------
FINAL count=199977 ynum=4294967295 inferred_segments=98 (num_segments NOT exposed -> inferred)
FINAL VmRSS=543320 kB VmSize=568316 kB smapsRss=543320 smapsPss=543316 ru_maxrss=541696 kB
FINAL tracemalloc current=7024560 B peak=12244141 B  (Python-only; C segment calloc is INVISIBLE here)
THROUGHPUT coarse window: fed 179520 lines in 0.2671 s = 672205 lines/s  [time.perf_counter, canonical parse_bytes]
CHECK inferred_segments*bytes/seg = 98 * 5251072 B = 502544 kB (expected virtual growth for segments)
ALL ASSERTIONS PASSED
```

**C3 reading.** A negative `scrollback_lines` maps to `ynum = 2**32‑1 = 4,294,967,295` [kitty/options/utils.py:557-561], so the buffer never reaches capacity for any realistic feed. Memory therefore **grows without plateau**: after 200,000 lines the buffer holds `count=199,977` across **98 segments** at `VmRSS ≈ 543 MB`, and would continue one `+5,132 kB` step per 2,048 lines until the process exhausts memory and `add_segment` calls `fatal("Out of memory")` [kitty/history.c:21,25] (the OOM ceiling is *inferred* from the source — the run was not driven to exhaustion). Final `VmRSS` across three runs: **543,312 / 543,692 / 543,320 kB** (spread 380 kB, 0.07 %).

### Q1 answer

- **Under massive output, process RSS rises in discrete ≈ 5.13 MB steps — one `HistoryBuf` segment (2,048 rows) at a time — and is bounded by `MAX(scrollback_lines, lines)` rows.** At the default `scrollback_lines=2000` the whole scrollback fits in a single segment, so a terminal flooded with millions of lines settles at **≈ 47 MB** and stays there. Raising the scrollback raises the ceiling linearly: 300,000 lines ≈ 794 MB (147 segments); an "infinite" buffer grows unbounded (543 MB after only 200,000 lines) until OOM.
- **Before / during / after** are all shown per run: `before` (empty, one upfront segment, ≈ 42 MB), `during` (linear per‑segment growth), `after`/`plateau` (flat once `count==ynum`).
- **Stability:** every reported figure is reproduced across three identical runs with ≤ 0.66 % spread (C2/C3 ≤ 0.07 %).
- **Measurement caveat (kernel‑documented).** `/proc/<pid>/status` `VmRSS` is an approximate counter; the Linux kernel recommends `smaps`/`smaps_rollup` for precision [kernel `Documentation/filesystems/proc.rst`]. The observer therefore also samples `smaps_rollup` `Rss`/`Pss` at every phase — they track `VmRSS` to within a few kB in every row above — and reports `ru_maxrss` (high‑water) alongside. `VmRSS`/`VmSize` is the signal *used*, corroborated by `smaps_rollup`; the sub‑percent stability figures are reported against that corroborated signal, not as an assertion of counter exactness.

### kitty's own test suite (same shipped module)

To confirm the imported module is healthy and the cell/attr sizes it was compiled with are the ones used above, kitty's scrollback‑relevant suites were run against the same shipped `.so`:

**Command:** `LANG=C.UTF-8 CI=true ./test.py --module datatypes screen parser  (+ image/.so identity)`

```text
IMG_HEAD=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
Python 3.12.3
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
--- fast_data_types.so identity (the module imported by the harness) ---
-rwxr-xr-x 1 root 1001 1221264 Aug 28  2025 kitty/fast_data_types.so
so path imported: /app/kitty/fast_data_types.so
=== TEST: datatypes ===
test_to_color (kitty_tests.datatypes.TestDataTypes.test_to_color) ... ok
test_url_at (kitty_tests.datatypes.TestDataTypes.test_url_at) ... ok
test_utils (kitty_tests.datatypes.TestDataTypes.test_utils) ... ok

----------------------------------------------------------------------
Ran 18 tests in 0.010s

OK
=== TEST: screen ===
test_wrapping_serialization (kitty_tests.screen.TestScreen.test_wrapping_serialization) ... ok
test_writing_with_cursor_on_trailer_of_wide_character (kitty_tests.screen.TestScreen.test_writing_with_cursor_on_trailer_of_wide_character) ... ok
test_zwj (kitty_tests.screen.TestScreen.test_zwj) ... ok

----------------------------------------------------------------------
Ran 36 tests in 0.090s

OK
=== TEST: parser ===
test_simple_parsing (kitty_tests.parser.TestParser.test_simple_parsing) ... ok
test_utf8_parsing (kitty_tests.parser.TestParser.test_utf8_parsing) ... ok
test_utf8_simd_decode (kitty_tests.parser.TestParser.test_utf8_simd_decode) ... ok

----------------------------------------------------------------------
Ran 16 tests in 0.055s

OK
```

---

## Q2 — Responsiveness and latency during concurrent activity

### Mechanism (grounded in source) — corrected thread model

kitty is **not** a "parse thread + separate render thread" design. The relevant threads are:

- **Main thread** — runs `main_loop()` [kitty/child-monitor.c:1259], which each iteration calls **`parse_input(self)` and then `render(...)`** back‑to‑back [kitty/child-monitor.c:1236-1237]. **Parsing of child output and GPU rendering both run here, on the one main thread.** `render()` [kitty/child-monitor.c:871] contains a throttle that early‑returns [kitty/child-monitor.c:875-878]; its `input_read` argument means *"the PTY parser did work this iteration"*, i.e. child‑program output activity — **not** user scroll input.
- **I/O thread** `KittyChildMon` — `io_loop()` [kitty/child-monitor.c:1481], name set at [kitty/child-monitor.c:1489], created at [kitty/child-monitor.c:291]. It **only** polls the PTY fds and drains bytes via `read_bytes` [kitty/child-monitor.c:1337] into a buffer, then wakes the main thread. It does no parsing and no rendering.
- **Talk thread** `KittyPeerMon` — `talk_loop()` [kitty/child-monitor.c:1805], name at [kitty/child-monitor.c:1808], **started only if a control socket is configured** [kitty/child-monitor.c:285]. It handles remote‑control peers and is **irrelevant to scroll responsiveness**. In the default secure configuration it does not exist (confirmed live below).

This was verified at runtime by enumerating the kitty process's threads inside the container:

```
== kitty terminal thread names (/proc/1/task/*/comm inside container) ==
      1 KittyChildMon
     33 kitty
      1 kitty:disk$0
== Mesa llvmpipe GL worker threads (rendering backend, NOT terminal core) == 32
== KittyPeerMon (talk_loop) present? (0 => remote control OFF, secure default) == 0
```

So the terminal‑core threads are the **main `kitty` thread**, the **`KittyChildMon` I/O thread**, and a `kitty:disk$0` cache thread; **`KittyPeerMon` is absent** (remote control off — the default). The 32 `llvmpipe-*` threads are Mesa's software‑GL worker pool (a property of the rendering backend, not of kitty's terminal core).

### The real user‑scroll path (grounded in source)

A scrollback key does **not** flow through the PTY/parse path at all. `scroll_home` is bound to `kitty_mod+home` where `kitty_mod` defaults to `ctrl+shift` [kitty/options/definition.py:3474,3623]. The chain is:

```
GLFW key event (main thread)
  -> Window.scroll_home            [kitty/window.py:1860]
  -> Screen.scroll(SCROLL_FULL, True)
  -> screen_history_scroll         [kitty/screen.c:4091-4118]   (sets scrolled_by, marks dirty_scroll)
  -> next render() repaints at the new scroll position
```

Because the scroll only sets `scrolled_by`/`dirty_scroll` and the actual redraw happens in the **same main‑thread `render()`** that also draws incoming output, a scroll issued *while output streams* must wait for the next repaint tick. That render cadence is governed by three options (defaults = condition **C4**): `input_delay=3` ms [kitty/options/definition.py:878], `repaint_delay=10` ms [kitty/options/definition.py:866], `sync_to_monitor=yes` [kitty/options/definition.py:889]. The low‑latency‑tuned set (condition **C5**) is `input_delay=0`, `repaint_delay=2`, `sync_to_monitor=no` (kitty's documented latency‑minimizing tuning).

### What was measured

Using the **image's own kitty binary** on a host `Xvfb`, a large scrollback was filled whose oldest 40 lines are a distinctive `#` banner (Appendix B). A real `ctrl+shift+home` was injected via the X11 **XTEST** extension (`libXtst`), and the framebuffer top strip was polled via `XGetImage` (~0.3 ms/grab) until it matched the banner. **Latency = t(banner rendered) − t(key injected)**, `time.perf_counter()`. Two conditions × two configs, **two identical runs × 15 trials = 30 trials per cell**; all distribution statistics computed programmatically from the raw per‑trial CSV (below):

- **idle** — history filled, then the child is quiescent (this is both the *before‑burst* and *after‑burst* steady state: no output is being produced while latency is sampled).
- **during** — the child streams output continuously while scrolls are injected (true concurrent activity).

### Latency results (raw per‑trial data)

Raw CSV rows are `label,trial,latency_ms,detector_gap,frames_polled`; the two blocks per file are the two independent runs.

**Command:** `q2_run.sh c4 idle measure 15   (2 runs)`

```text
c4_idle,1,8.5649,619219,23
c4_idle,2,9.3907,619219,29
c4_idle,3,8.1852,619219,23
c4_idle,4,8.5124,619219,30
c4_idle,5,8.9032,619219,21
c4_idle,6,8.7707,619219,23
c4_idle,7,8.3890,619219,21
c4_idle,8,6.6858,619219,22
c4_idle,9,8.5789,619219,22
c4_idle,10,6.9886,619219,24
c4_idle,11,7.4038,619219,20
c4_idle,12,6.7522,619219,23
c4_idle,13,8.0734,619219,19
c4_idle,14,6.6643,619219,22
c4_idle,15,8.6481,619219,22
c4_idle,1,9.5665,619219,28
c4_idle,2,6.7922,619219,19
c4_idle,3,6.5531,619219,19
c4_idle,4,6.6888,619219,22
c4_idle,5,7.1204,619219,20
c4_idle,6,7.0192,619219,22
c4_idle,7,7.0204,619219,19
c4_idle,8,6.7113,619219,20
c4_idle,9,6.9045,619219,20
c4_idle,10,6.9498,619219,20
c4_idle,11,7.0787,619219,20
c4_idle,12,6.5371,619219,18
c4_idle,13,6.5297,619219,20
c4_idle,14,7.0619,619219,21
c4_idle,15,7.3295,619219,15
```

**Command:** `q2_run.sh c4 during measure 15 (2 runs)`

```text
c4_during,1,15.9467,766410,49
c4_during,2,8.3376,767645,26
c4_during,3,15.1592,758535,48
c4_during,4,7.7929,762117,22
c4_during,5,15.9451,759852,51
c4_during,6,6.7237,757376,21
c4_during,7,14.9130,749178,47
c4_during,8,15.7939,770568,49
c4_during,9,14.8926,754640,41
c4_during,10,6.5267,758535,22
c4_during,11,6.6087,774077,20
c4_during,12,7.3978,775680,23
c4_during,13,14.9994,768608,47
c4_during,14,15.5601,765984,36
c4_during,15,13.2942,784150,44
c4_during,1,12.0486,757988,36
c4_during,2,14.7134,766051,43
c4_during,3,14.9831,765613,47
c4_during,4,14.5945,754133,45
c4_during,5,14.9396,763348,50
c4_during,6,14.8889,757162,33
c4_during,7,6.6020,754538,20
c4_during,8,6.2339,760612,12
c4_during,9,6.6710,765314,21
c4_during,10,14.7736,756504,45
c4_during,11,16.1275,768889,51
c4_during,12,15.0315,756060,46
c4_during,13,15.2258,772542,51
c4_during,14,15.2004,774077,51
c4_during,15,6.4942,788084,23
```

**Command:** `q2_run.sh c5 idle measure 15   (2 runs)`

```text
c5_idle,1,6.8943,619219,16
c5_idle,2,7.6473,619219,20
c5_idle,3,7.3104,619219,20
c5_idle,4,9.1760,619219,26
c5_idle,5,6.9688,619219,21
c5_idle,6,6.6820,619219,21
c5_idle,7,7.1880,619219,16
c5_idle,8,6.4372,619219,21
c5_idle,9,7.3941,619219,21
c5_idle,10,7.4663,619219,20
c5_idle,11,7.3099,619219,19
c5_idle,12,7.0483,619219,23
c5_idle,13,9.2398,619219,31
c5_idle,14,6.7807,619219,20
c5_idle,15,6.6852,619219,20
c5_idle,1,6.5498,619219,19
c5_idle,2,6.6829,619219,19
c5_idle,3,6.9774,619219,24
c5_idle,4,8.7398,619219,21
c5_idle,5,7.2588,619219,20
c5_idle,6,10.0929,619219,21
c5_idle,7,9.3932,619219,25
c5_idle,8,6.5711,619219,20
c5_idle,9,6.6073,619219,20
c5_idle,10,6.8368,619219,20
c5_idle,11,6.5261,619219,22
c5_idle,12,7.2561,619219,23
c5_idle,13,6.6071,619219,18
c5_idle,14,6.9953,619219,19
c5_idle,15,9.7782,619219,19
```

**Command:** `q2_run.sh c5 during measure 15 (2 runs)`

```text
c5_during,1,11.6489,757330,36
c5_during,2,12.1661,780474,35
c5_during,3,12.9261,768237,39
c5_during,4,9.6130,746508,30
c5_during,5,6.9683,764819,22
c5_during,6,9.8563,755298,30
c5_during,7,6.6702,759852,22
c5_during,8,6.8488,782708,21
c5_during,9,6.9229,756204,23
c5_during,10,12.1174,759081,38
c5_during,11,11.5978,759194,38
c5_during,12,11.6716,757078,36
c5_during,13,11.1310,748520,40
c5_during,14,12.8171,782619,42
c5_during,15,12.3613,780868,39
c5_during,1,10.2929,765032,24
c5_during,2,12.6279,773617,39
c5_during,3,8.4271,764955,24
c5_during,4,9.0622,770002,26
c5_during,5,12.5460,765972,39
c5_during,6,8.3714,750936,26
c5_during,7,12.1153,754538,37
c5_during,8,6.3141,762476,19
c5_during,9,10.4470,758313,34
c5_during,10,11.4785,768908,38
c5_during,11,10.5150,768889,27
c5_during,12,8.6339,760365,26
c5_during,13,9.1488,762188,28
c5_during,14,9.0663,774735,24
c5_during,15,8.5837,769931,27
```

Distribution statistics computed programmatically from the raw rows above (`n=30` per cell):

| Condition | config (input_delay / repaint_delay / sync) | n | min | **median** | mean | p90 | max |
|-----------|---------------------------------------------|---|-----|-----------|------|-----|-----|
| idle   | **C4** default 3 / 10 / yes | 30 | 6.53 | **7.07**  | 7.55  | 8.77  | 9.57  |
| during | **C4** default 3 / 10 / yes | 30 | 6.23 | **14.83** | 12.28 | 15.79 | 16.13 |
| idle   | **C5** tuned 0 / 2 / no      | 30 | 6.44 | **7.02**  | 7.44  | 9.24  | 10.09 |
| during | **C5** tuned 0 / 2 / no      | 30 | 6.31 | **10.37** | 10.10 | 12.55 | 12.93 |

Per‑run medians (2 runs each) confirm run‑to‑run stability: c4_idle 8.39 / 6.95; c4_during 14.89 / 14.77; c5_idle 7.19 / 6.98; c5_during 11.60 / 9.15 ms.

### Q2 answer

- **Does it stay responsive? Yes.** Across all 120 trials every injected scroll produced a detected banner render; the worst single latency was **16.13 ms**, far below the ~100 ms threshold at which lag becomes perceptible. The terminal never stalled, dropped a scroll, or failed to catch up while output streamed.
- **What is the latency/lag?** Idle scroll‑to‑render is **≈ 7 ms** regardless of config (the ≈ 6.3 ms floor is XTEST delivery + GL draw + buffer swap + detector poll). Under continuous output it rises to **≈ 14.8 ms median at the default settings (C4)** and **≈ 10.4 ms median with the low‑latency tuning (C5)** — the *before/after* quiescent state is the idle row; the *during* state is the elevated row; once output stops, latency returns to the idle distribution.
- **Signs of prioritizing one operation over another? Yes — and it is visible in the distribution.** The during‑output latencies are **bimodal**: a cluster near the idle floor (≈ 6.5 ms, when the injected scroll happens to arrive in a gap between output bursts) and a cluster near ≈ 15 ms (when it arrives just after a repaint has been scheduled and must wait for the next tick). This is the direct signature of the scroll repaint **sharing the single main‑thread render loop** with output rendering, throttled by `repaint_delay`. The C4→C5 comparison confirms the cause: dropping `repaint_delay` from 10 ms to 2 ms cuts the during‑output **median by ~30 %** (14.83 → 10.37 ms) and the **max from 16.1 → 12.9 ms**, while idle is unchanged (nothing to contend with). The I/O thread (`KittyChildMon`) keeps draining the PTY the whole time, so output is never blocked by the user's scroll either — the two simply take turns on the main thread's repaint cadence.

### Throughput context — the `__benchmark__` kitten (NOT a latency measurement)

kitty ships a hidden throughput benchmark, `kitten __benchmark__`. It is a **parser/PTY‑throughput** tool, not a latency or rendering tool: its own output states *"These results measure the time it takes the terminal to fully parse all the data sent to it"* and *"rendering is suppressed … to better benchmark parser performance"* [tools/cmd/benchmark/main.go:302-305,323]. It must run inside a real terminal (it sends data and waits for query responses); it hangs on a raw pty and also under `script`, but runs correctly as a **direct child of the image's kitty** (verified). Results were read from the rendered screen (screenshots archived).

**Argument‑order correctness.** kitty's CLI stops parsing options after the first positional argument: `Command.AllowOptionsAfterArgs` defaults to 0 — *"0 means no options after the first non-option arg"* [tools/cli/command.go] — enforced by `if self.AllowOptionsAfterArgs <= len(self.Args) { options_allowed = false }` [tools/cli/parse-args.go]. The `__benchmark__` command does not raise that limit [tools/cmd/benchmark/main.go:317-352], so any option placed **after** `ascii` is silently ignored. `--repetitions` defaults to `100` and `--with-scrollback` switches from the alt screen to the main screen (`Alternate_screen: !opts.WithScrollback`) [tools/cmd/benchmark/main.go:59,337-347]. The prior draft's command put `ascii` first, so its options were ignored — which the following two runs demonstrate directly:

| Invocation | reps used | screen | run 1 | run 2 | run 3 |
|-----------|-----------|--------|-------|-------|-------|
| **correct** `kitten __benchmark__ --with-scrollback --repetitions 20 ascii` | 20 | main (scrollback) | 593.22 ms @ 67.4 MB/s | 629.24 ms @ 63.6 MB/s | 593.58 ms @ 67.4 MB/s |
| **wrong** (options after `ascii`) `kitten __benchmark__ ascii --with-scrollback --repetitions 20` | **100** (ignored) | **alt** (ignored) | 2.22 s @ 90.0 MB/s | 2.23 s @ 89.8 MB/s | 2.19 s @ 91.4 MB/s |

The wrong order takes ~2.2 s (≈ 100 repetitions) versus ~0.6 s (20 repetitions) for the correct order — an unambiguous ~3.7× proof that `--repetitions 20` was ignored. The prior draft's reported "82.2 MB/s" falls squarely in the wrong‑order (alt‑screen, 100‑rep) range, confirming it measured the **default alt‑screen condition, not the scrollback path**. Interpreted correctly and **scoped narrowly**: under the `ascii` benchmark with rendering suppressed, the image's kitty parses the main‑screen (scrollback) stream at **≈ 63.6–67.4 MB/s**. This is *parser/PTY throughput only* — it says nothing about rendering or about whether "the terminal is the bottleneck", and it is **not** a latency figure (the latency numbers above are the responsiveness evidence). The MB/s spread between runs reflects the benchmark's randomized payload per run.

> **Methodology note (Typometer).** Typometer, the standard software tool for terminal keyboard‑to‑screen latency, *includes* the GPU pipeline, buffering, window manager and VSync in its measurement and *excludes* only the physical keyboard and display‑device delay. The measurement performed here (synthetic X input event → framebuffer change) is a Typometer‑style software input‑to‑visible‑update measurement with the same inclusion/exclusion boundary.

---

## Q3 — Buffer growth boundaries

### When does new backing storage get allocated?

New storage is allocated **one `SEGMENT_SIZE = 2048`‑row segment at a time, lazily, the moment a line is written into a row index that falls in a not‑yet‑allocated block.** The trigger is `segment_for(index)` [kitty/history.c:37-42], which calls `add_segment()` [kitty/history.c:18-29] (one `calloc` of 5,251,072 bytes at 80 columns) only when the target block is missing. The first segment is the exception: it is reserved **upfront** when the `Screen`/`HistoryBuf` is created [kitty/history.c:117-132], which is why the C1/C2/C3 runs show a `+5,132 kB` `VmSize` jump at `post-create_screen` before any line is fed.

Because the allocation is one discrete `calloc`, it is directly observable through external memory monitoring as a **step in `VmSize`**. To capture the *exact* boundary, the observer fed **one line at a time** across the second‑segment transition and recorded history `count` and `VmSize` at every single‑line step. (Feeding must be keyed on history `count`, not lines fed: the 24‑row visible screen holds the newest rows, so a line only enters *history* after the screen fills — history `count` lags lines‑fed by the screen height. This is exactly the offset the prior draft's fixed 2048‑stride sampling missed.)

### Exact boundary — one‑line‑at‑a‑time (bracketing count 2047 / 2048 / 2049)

**Command:** `BLITZY_REPO=/app python3 obs_mem.py boundary 1  (shipped release .so)`

```text
PREFLIGHT MemAvailable=3926281228 kB  required(safe)>=400000 kB
============================================================================================
BOUNDARY exact 2nd-segment allocation  [run 1]  pid=1 cols=80 lines=24 scrollback=300000
============================================================================================
computed bytes/segment @80c = 5251072  (= 5.0078 MiB)
step      count   segs   VmSize(kB)    VmRSS(kB)  dVmSize(kB)
-------------------------------------------------------------
2064       2041      1        69892        47076             
2065       2042      1        69892        47076           +0
2066       2043      1        69892        47080           +0
2067       2044      1        69892        47084           +0
2068       2045      1        69892        47088           +0
2069       2046      1        69892        47092           +0
2070       2047      1        69892        47092           +0
2071       2048      1        69892        47092           +0
2072       2049      2        75024        47104        +5132
2073       2050      2        75024        47104           +0
2074       2051      2        75024        47108           +0
2075       2052      2        75024        47108           +0
2076       2053      2        75024        47112           +0
2077       2054      2        75024        47116           +0
2078       2055      2        75024        47116           +0
2079       2056      2        75024        47120           +0
2080       2057      2        75024        47124           +0
2081       2058      2        75024        47124           +0
2082       2059      2        75024        47128           +0
2083       2060      2        75024        47128           +0
-------------------------------------------------------------
OBSERVED VmSize step first occurs at history count = 2049
ASSERTION PASSED: 2nd-segment allocation bracketed at count in {2048,2049}
```

Reruns 2 and 3 reproduced the step at `count=2049` identically. The same experiment against the **fresh debug build** (`fast_data_types.so` = 6,142,824 bytes) also stepped at `count=2049` (absolute `VmSize` differs because the debug module is larger; the boundary and step do not):

**Command:** `BLITZY_REPO=/app python3 obs_mem.py boundary 1  (fresh --debug .so)`

```text
build exit=0 ; debug .so:
-rwxr-xr-x 1 root 1001 6142824 Jul 13 18:02 kitty/fast_data_types.so
2075       2052      2        75336        46948           +0
2076       2053      2        75336        46952           +0
2077       2054      2        75336        46956           +0
2078       2055      2        75336        46956           +0
2079       2056      2        75336        46960           +0
2080       2057      2        75336        46964           +0
2081       2058      2        75336        46964           +0
2082       2059      2        75336        46968           +0
2083       2060      2        75336        46968           +0
-------------------------------------------------------------
OBSERVED VmSize step first occurs at history count = 2049
ASSERTION PASSED: 2nd-segment allocation bracketed at count in {2048,2049}
```

**Boundary reading.** `VmSize` is **flat at 69,892 kB** while history `count` climbs through **2047 and 2048** (still one segment). The **instant `count` crosses 2048 → 2049** — i.e. the first write into history row index 2048, which lives in the second block — `add_segment` fires and `VmSize` **steps +5,132 kB to 75,024 kB** (segment count 1 → 2). It then stays flat again through `count=2060`. So the *k*‑th segment is allocated as `count` crosses `k*2048 → k*2048+1`. This reproduced at `count=2049` in **all three runs** and **also** against the fresh debug build (`fast_data_types.so` = 6,142,824 bytes), so the boundary is **build‑independent**. The `+5,132 kB` step (= 5,255,168 B) is the 5,251,072‑byte segment rounded up to the next 4,096‑byte page — reconciling the compiled size, the assembly, and the external monitor.

### What happens as the buffer keeps growing — plateau and the negative case

- **Plateau at capacity.** Once `count == ynum`, `historybuf_push()` stops growing and **overwrites the oldest slot circularly**, advancing `start_of_data` and spilling the evicted line into the pager ring buffer via `pagerhist_push()` [kitty/history.c:276-285,259]. Externally this is the flat tail in C2 (147 segments, `VmRSS` pinned at 793,640 kB while 160,000 further lines are fed). Total segments at capacity = `ceil(ynum / 2048)`.
- **Capacity = `MAX(scrollback, lines)`** [kitty/screen.c:130]. Verified directly through the canonical `create_screen` path:

```
lines=24  scrollback=10    -> historybuf.ynum=24     (MAX(sb,lines)=24)  xnum=80 count=0
lines=24  scrollback=2000  -> historybuf.ynum=2000   (MAX(sb,lines)=2000)  xnum=80 count=0
lines=24  scrollback=0     -> historybuf.ynum=24     (MAX(sb,lines)=24)  xnum=80 count=0
lines=5   scrollback=100   -> historybuf.ynum=100    (MAX(sb,lines)=100)  xnum=80 count=0
```

  So with the documented default `scrollback_lines=2000` and 24 visible rows, `ynum=2000 < 2048` and the terminal lives in **exactly one segment** (matches C1). A configured `scrollback=10` still yields `ynum=24` because the visible rows dominate.
- **Negative → effectively unbounded.** `scrollback_lines("-1")` maps to `ynum = 2**32‑1` [kitty/options/utils.py:557-561], so the plateau is never reached in practice: C3 shows uninterrupted `+5,132 kB`‑per‑2048‑lines growth to 98 segments / 543 MB after 200,000 lines, and it would continue until `add_segment`'s `calloc` fails and it calls `fatal("Out of memory")` [kitty/history.c:21,25] (OOM ceiling *inferred* from source, not driven to exhaustion). The pager‑history buffer is a separate megabyte‑sized ring [kitty/data-types.h:268-272] backed by the vendored ringbuf [3rdparty/ringbuf/ringbuf.h:30,41,73,87]; at the default `scrollback_pager_history_size=0` [kitty/options/definition.py:406] `alloc_pagerhist` returns `NULL` [kitty/history.c:70-79] and no pager storage is allocated — which is why the runs above override the harness's forced test value back to 0.

### Q3 answer

- **The buffer's behavior changes at two boundaries, both observable externally.** (1) **Segment allocation** — a new `calloc` of one 2,048‑row segment (≈ 5.13 MB at 80 cols) each time history `count` crosses a multiple of 2,048 (first observed transition: `count 2048 → 2049`, `VmSize +5,132 kB`), plus one segment reserved upfront at creation. (2) **Capacity plateau** — at `count == ynum = MAX(scrollback, lines)` growth stops entirely and the buffer recycles oldest‑first; `VmRSS` goes perfectly flat (C2). A negative `scrollback_lines` removes the practical plateau (C3) up to the OOM `fatal`.

---

## Grounding — every responsible symbol with `file:line`

All citations are anchored to HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

| Claim / value | Symbol (function / struct / macro) | `file:line` |
|---------------|-----------------------------------|-------------|
| Segment size = 2048 rows | `#define SEGMENT_SIZE` | kitty/history.c:15 |
| One `calloc` per segment (the growth step) | `add_segment()` | kitty/history.c:18-29 |
| `fatal("Out of memory")` on alloc failure (OOM ceiling) | `add_segment()` guards | kitty/history.c:21,25 |
| On‑demand allocation trigger | `segment_for()` | kitty/history.c:37-42 |
| First segment reserved upfront at creation | `create_historybuf()` | kitty/history.c:117-132 |
| Push line; plateau/overwrite at `count==ynum` | `historybuf_push()` | kitty/history.c:276-285 |
| Spill evicted line to pager ring | `pagerhist_push()` | kitty/history.c:259 |
| Add line entry point | `historybuf_add_line()` | kitty/history.c:287-291 |
| Pager buffer: NULL when size 0; extend | `alloc_pagerhist()` / `pagerhist_extend()` | kitty/history.c:70-79 / 90-101 |
| Python read‑only members `xnum/ynum/count` (no `num_segments`) | members table | kitty/history.c:556-558 |
| `HistoryBuf` / `HistoryBufSegment` / `PagerHistoryBuf` structs | struct defs | kitty/data-types.h:281-290 / 261-266 / 268-272 |
| `sizeof(GPUCell)==20` | `static_assert` | kitty/data-types.h:221 |
| `sizeof(CPUCell)==12` | `static_assert` | kitty/data-types.h:228 |
| `sizeof(LineAttrs)==4` (`PromptKind` int‑enum bit‑field) | `union LineAttrs` | kitty/data-types.h:231-239 |
| Required OpenGL ≥ 3.3 | `OPENGL_REQUIRED_VERSION_*` | kitty/data-types.h:19-21 |
| Capacity `ynum = MAX(scrollback, lines)` | `screen_history_scroll` alloc site | kitty/screen.c:130 |
| Scroll‑off into history (write trigger) | `INDEX_UP` macro (`historybuf_add_line`, `history_line_added_count++`) | kitty/screen.c:1552,1558-1559 |
| User scrollback navigation | `screen_history_scroll()` (sets `scrolled_by`, `dirty_scroll`) | kitty/screen.c:4091-4118 |
| Parse+render both on main thread | `main_loop()` calling `parse_input` then `render` | kitty/child-monitor.c:1259,1236-1237 |
| Render throttle (on PTY parse activity, not scroll) | `render()` early‑return | kitty/child-monitor.c:871,875-878 |
| I/O thread drains PTY only | `io_loop()` / `read_bytes()` / name `KittyChildMon` | kitty/child-monitor.c:1481,1337,1489 |
| Talk thread optional (remote control) | `talk_loop()` / name `KittyPeerMon` / gate | kitty/child-monitor.c:1805,1808,285 |
| Real VT parser driven by the harness | `vt_parser_create_write_buffer` / `vt_parser_commit_write` / `parse_worker` | kitty/vt-parser.c:1451,1465,1496 |
| Line ops invoked by `INDEX_UP` | `linebuf_init_line` / `linebuf_index` | kitty/line-buf.c:141,317 |
| `historybuf_add_line` declaration | header decl | kitty/lineops.h:122 |
| Default `scrollback_lines=2000` | option def | kitty/options/definition.py:372 |
| Default `scrollback_pager_history_size=0` | option def | kitty/options/definition.py:406 |
| `repaint_delay=10` / `input_delay=3` / `sync_to_monitor=yes` | option defs | kitty/options/definition.py:866,878,889 |
| `kitty_mod=ctrl+shift`; `scroll_home` binding | option defs | kitty/options/definition.py:3474,3623 |
| Negative scrollback → `2**32‑1` | `scrollback_lines()` | kitty/options/utils.py:557-561 |
| Pager size MB→bytes | `scrollback_pager_history_size()` | kitty/options/utils.py:564-566 |
| Window scroll methods | `scroll_home` / `scroll_page_up` / `scroll_line_up` | kitty/window.py:1860,1846,1832 |
| Canonical headless driver | `parse_bytes` / `create_screen` (+ forced pager override) | kitty_tests/__init__.py:30,237,224 |
| Pager ring buffer API | `ringbuf_t` / `ringbuf_new` / `ringbuf_capacity` / `ringbuf_bytes_used` | 3rdparty/ringbuf/ringbuf.h:30,41,73,87 |
| Global options / render‑scheduling state | global state backing `render()` | kitty/state.c |
| CLI stops options after first positional | `Command.AllowOptionsAfterArgs` / parse loop | tools/cli/command.go; tools/cli/parse-args.go |
| Benchmark measures parse time; rendering suppressed; option defaults | `present_result` / `EntryPoint` / `main` | tools/cmd/benchmark/main.go:59,302-305,317-352 |

**Inferred (not directly observed) items, labelled as such:** (a) the **segment count** is inferred as `(count-1)//2048 + 1` because `num_segments` is not exposed to Python (only `xnum/ynum/count` are read‑only members [kitty/history.c:556-558]); (b) the **OOM ceiling** of the negative/infinite buffer is inferred from `add_segment`'s `fatal` path [kitty/history.c:21,25] — the C3 run was intentionally not driven to memory exhaustion.

**External sources.** Linux `/proc` semantics and the recommendation to prefer `smaps`/`smaps_rollup` over approximate `status` RSS: kernel documentation *Documentation/filesystems/proc.rst* (`https://www.kernel.org/doc/html/latest/filesystems/proc.html`). kitty scrollback / performance option semantics: kitty configuration docs (`https://sw.kovidgoyal.net/kitty/conf/`) and performance docs (`https://sw.kovidgoyal.net/kitty/performance/`); Typometer methodology: the Typometer project (`https://github.com/pavelfatin/typometer`). Each was verified against this checkout's source per the `file:line` grounding above.

---

## Appendix A — Q1/Q3 headless memory observer (verbatim)

Created under a private `mktemp -d` (mode 0700) workdir outside the repository and removed afterward. It validates the checkout root from `BLITZY_REPO` before touching `sys.path` (does not trust bare CWD), does a `MemAvailable` preflight, times only the coarse window with `perf_counter` divided by the exact lines fed in it, samples every metric at every phase, and asserts behaviour (monotonic `count`, `count==ynum` plateau flatness, per‑segment step ≈ one segment).

```python
#!/usr/bin/env python3
"""
Temporary observation script (Q1 memory / Q3 allocation boundaries) for the
kitty scrollback HistoryBuf investigation.

CANONICAL PATH ONLY: lines are driven into the HistoryBuf exclusively by
feeding real PTY bytes through kitty's VT parser via
    kitty_tests.parse_bytes(screen, data)     [kitty_tests/__init__.py:30]
into a Screen built by
    BaseTest.create_screen(...)                [kitty_tests/__init__.py:237]
The harness parse_bytes drives the REAL parser test shims
    Screen.test_create_write_buffer -> vt_parser_create_write_buffer  [kitty/vt-parser.c:1451]
    Screen.test_commit_write_buffer -> vt_parser_commit_write         [kitty/vt-parser.c:1465]
    Screen.test_parse_written_data  -> parse_worker/run_worker        [kitty/vt-parser.c:1496]
which reach screen.c INDEX_UP [kitty/screen.c:1552-1559] ->
historybuf_add_line [kitty/history.c:287-291] -> historybuf_push
[kitty/history.c:276-285].  NO HistoryBuf.push() is called anywhere
(that would be a non-canonical synthetic bypass).

Usage:  python3 obs_mem.py <c1|c2|c3|boundary> [run_index]

Security/safety hardening:
  * Repo root is taken from BLITZY_REPO and *validated* (must contain the
    canonical source files) before it is prepended to sys.path, instead of
    blindly trusting os.getcwd().
  * A MemAvailable preflight aborts before large runs if free memory is low.
This file lives under a private mktemp-created directory (outside the repo)
and is removed after use.
"""
import os
import sys
import time
import resource
import tracemalloc

# ---- validated import root (fixes: do not trust bare CWD) -------------------
REPO = os.environ.get("BLITZY_REPO", os.getcwd())
_needed = ("kitty/history.c", "kitty_tests/__init__.py", "kitty/data-types.h")
if not all(os.path.exists(os.path.join(REPO, p)) for p in _needed):
    sys.stderr.write("FATAL: BLITZY_REPO=%r is not a valid kitty checkout\n" % REPO)
    sys.exit(3)
REPO = os.path.realpath(REPO)
sys.path.insert(0, REPO)

# ---- canonical, source-derived constants ------------------------------------
SEGMENT_SIZE = 2048          # kitty/history.c:15
COLS = 80                    # 80-column terminal for the memory math
LINES = 24                   # default visible rows
SIZEOF_CPUCELL = 12          # static_assert kitty/data-types.h:228
SIZEOF_GPUCELL = 20          # static_assert kitty/data-types.h:221
SIZEOF_LINEATTRS = 4         # measured (union LineAttrs, data-types.h:231-239);
                             # PromptKind is an int-backed enum bit-field -> 4 B
BYTES_PER_SEG_80c = (COLS * SEGMENT_SIZE * (SIZEOF_CPUCELL + SIZEOF_GPUCELL)
                     + SEGMENT_SIZE * SIZEOF_LINEATTRS)   # = 5,251,072


def meminfo_available_kb():
    with open("/proc/meminfo") as f:
        for line in f:
            if line.startswith("MemAvailable:"):
                return int(line.split()[1])
    return None


def preflight(cond):
    need = {"c1": 200_000, "c2": 1_200_000, "c3": 900_000, "boundary": 400_000}
    avail = meminfo_available_kb()
    req = need.get(cond, 200_000)
    print("PREFLIGHT MemAvailable=%s kB  required(safe)>=%d kB" % (avail, req))
    if avail is not None and avail < req:
        sys.stderr.write("FATAL: insufficient free memory for %s (avail %d < %d kB)\n"
                         % (cond, avail, req))
        sys.exit(4)


def vm():
    """Selected /proc/self/status counters (kB) — external memory monitoring."""
    d = {}
    with open("/proc/%d/status" % os.getpid()) as f:
        for line in f:
            p = line.split()
            if p and p[0] in ("VmRSS:", "VmSize:", "VmData:", "VmHWM:"):
                d[p[0][:-1]] = int(p[1])
    return d


def smaps_rollup():
    """/proc/self/smaps_rollup Rss/Pss (kB). The kernel documents /status RSS as
    approximate; smaps_rollup is the more precise per-process aggregate."""
    d = {}
    try:
        with open("/proc/%d/smaps_rollup" % os.getpid()) as f:
            for line in f:
                p = line.split()
                if p and p[0] in ("Rss:", "Pss:"):
                    d[p[0][:-1]] = int(p[1])
    except OSError:
        pass
    return d


def inferred_segments(count):
    """num_segments is NOT exposed to Python (only xnum/ynum/count are read-only
    members: kitty/history.c:556-558), so it is INFERRED. create_historybuf
    reserves the first segment upfront (kitty/history.c:117-132) so a fresh
    buffer already has 1 segment at count==0."""
    if count <= 0:
        return 1
    return (count - 1) // SEGMENT_SIZE + 1


def make_line(i):
    """One ~78-column line of real text + CRLF (canonical terminal output)."""
    s = "L%08d " % (i % 100000000)
    s = s + "x" * (78 - len(s))
    return s.encode("ascii") + b"\r\n"


class Feeder:
    def __init__(self):
        from kitty_tests import parse_bytes
        self._pb = parse_bytes
        self.total = 0
        # Pre-build one SEGMENT_SIZE-line block of bytes for fast reuse.
        self._block = b"".join(make_line(i) for i in range(SEGMENT_SIZE))

    def feed(self, screen, nlines):
        remaining = nlines
        while remaining >= SEGMENT_SIZE:
            self._pb(screen, self._block)
            self.total += SEGMENT_SIZE
            remaining -= SEGMENT_SIZE
        if remaining:
            buf = b"".join(make_line(self.total + j) for j in range(remaining))
            self._pb(screen, buf)
            self.total += remaining


def full_sample(label, screen, feeder):
    m = vm()
    sr = smaps_rollup()
    cur, peak = tracemalloc.get_traced_memory()
    ru = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss  # kB on Linux
    c = screen.historybuf.count
    return {
        "label": label, "fed": feeder.total, "count": c,
        "seg_inferred": inferred_segments(c),
        "VmSize": m["VmSize"], "VmRSS": m["VmRSS"], "VmData": m["VmData"],
        "smaps_Rss": sr.get("Rss", -1), "smaps_Pss": sr.get("Pss", -1),
        "ru_maxrss": ru, "tm_cur": cur, "tm_peak": peak,
    }


def print_full_table(rows):
    cols = ("phase", "lines_fed", "count", "segs", "VmSize(kB)", "VmRSS(kB)",
            "smapsRss", "smapsPss", "ru_maxrss", "tmPeak(B)", "dVmSize(kB)")
    fmt = "%-12s %10s %10s %5s %11s %11s %10s %10s %10s %11s %12s"
    hdr = fmt % cols
    print(hdr)
    print("-" * len(hdr))
    prev = None
    for r in rows:
        dstep = "" if prev is None else ("%+d" % (r["VmSize"] - prev))
        print(fmt % (r["label"], r["fed"], r["count"], r["seg_inferred"],
                     r["VmSize"], r["VmRSS"], r["smaps_Rss"], r["smaps_Pss"],
                     r["ru_maxrss"], r["tm_peak"], dstep))
        prev = r["VmSize"]


def run(cond, run_index):
    preflight(cond)
    # Capture pre-import baseline, then start tracemalloc BEFORE importing the
    # kitty extension so Python allocations are traced (C callocs are not — that
    # blind spot is exactly what we demonstrate).
    m_proc0 = vm()
    tracemalloc.start()
    from kitty_tests import BaseTest  # canonical harness
    m_import = vm()
    bt = BaseTest()

    if cond == "boundary":
        return run_boundary(bt, run_index, m_proc0, m_import)

    if cond == "c1":
        scrollback, total_lines, fine_segs, coarse_step = 2000, 5000, 1, 500
        title = "C1 DEFAULT (scrollback_lines=2000, pager=0)"
    elif cond == "c2":
        scrollback, total_lines, fine_segs, coarse_step = 300000, 500000, 20, 75000
        title = "C2 LARGE FINITE (scrollback=300000, feed 500000, pager=0)"
    elif cond == "c3":
        from kitty.options.utils import scrollback_lines  # canonical handler
        scrollback = scrollback_lines("-1")   # negative -> 2**32-1 (utils.py:557-561)
        total_lines, fine_segs, coarse_step = 200000, 10, 50000
        title = "C3 NEGATIVE/INFINITE (scrollback_lines(-1)=%d, feed 200000, pager=0)" % scrollback
    else:
        print("unknown condition", cond)
        return 2

    print("=" * 92)
    print("%s  [run %d]  pid=%d  cols=%d lines=%d" % (title, run_index, os.getpid(), COLS, LINES))
    print("=" * 92)
    print("canonical constants: SIZEOF_CPUCELL=%d [data-types.h:228]  SIZEOF_GPUCELL=%d [data-types.h:221]  "
          "SIZEOF_LINEATTRS=%d [data-types.h:231-239]" % (SIZEOF_CPUCELL, SIZEOF_GPUCELL, SIZEOF_LINEATTRS))
    print("computed bytes/segment @%dc = %d  (= %.4f MiB)  [add_segment kitty/history.c:18-29]"
          % (COLS, BYTES_PER_SEG_80c, BYTES_PER_SEG_80c / (1024 * 1024)))
    print("scrollback_lines('-1') canonical map: 2**32-1 = %d  [utils.py:557-561]" % (2 ** 32 - 1))
    print("pre-import  VmSize=%d kB VmRSS=%d kB" % (m_proc0["VmSize"], m_proc0["VmRSS"]))
    print("post-import VmSize=%d kB VmRSS=%d kB" % (m_import["VmSize"], m_import["VmRSS"]))

    # create the Screen (canonical): reserves the FIRST segment upfront
    s = bt.create_screen(cols=COLS, lines=LINES, scrollback=scrollback,
                         options={"scrollback_pager_history_size": 0})
    m_screen = vm()
    print("post-create_screen(scrollback=%d) VmSize=%d kB VmRSS=%d kB  "
          "(upfront ONE segment: dVmSize=%+d dVmRSS=%+d kB)  ynum=%d count=%d"
          % (scrollback, m_screen["VmSize"], m_screen["VmRSS"],
             m_screen["VmSize"] - m_import["VmSize"],
             m_screen["VmRSS"] - m_import["VmRSS"],
             s.historybuf.ynum, s.historybuf.count))
    # assertion: ynum == MAX(scrollback, lines)  [screen.c:130]
    assert s.historybuf.ynum == max(scrollback, LINES), "ynum != MAX(scrollback,lines)"
    assert s.historybuf.count == 0, "fresh buffer count must be 0"

    feeder = Feeder()
    rows = [full_sample("before", s, feeder)]

    # fine phase: one SEGMENT_SIZE block at a time to expose per-segment steps
    for k in range(fine_segs):
        feeder.feed(s, SEGMENT_SIZE)
        prev_count = rows[-1]["count"]
        rows.append(full_sample("fine seg~%d" % (k + 1), s, feeder))
        assert rows[-1]["count"] >= prev_count, "count must be monotonic non-decreasing"

    # coarse phase: large steps up to total_lines; time ONLY this window and
    # divide by the EXACT number of lines fed inside it (fixes prior timing bug)
    coarse_start_fed = feeder.total
    t0 = time.perf_counter()
    while feeder.total < total_lines:
        step = min(coarse_step, total_lines - feeder.total)
        feeder.feed(s, step)
        lbl = "plateau" if s.historybuf.count >= s.historybuf.ynum else "during"
        rows.append(full_sample(lbl, s, feeder))
    t1 = time.perf_counter()
    coarse_lines = feeder.total - coarse_start_fed
    coarse_secs = t1 - t0

    after = full_sample("after", s, feeder)
    rows.append(after)
    print_full_table(rows)

    # ---- behaviour-sensitive assertions -------------------------------------
    final_count = s.historybuf.count
    if scrollback <= total_lines:
        expected_count = min(scrollback, total_lines) if scrollback >= LINES else LINES
        # buffer must plateau exactly at ynum when capacity is reachable
        if s.historybuf.ynum <= total_lines:
            assert final_count == s.historybuf.ynum, \
                "expected plateau count==ynum=%d got %d" % (s.historybuf.ynum, final_count)
            # last two plateau rows must show ZERO VmSize growth
            plateau_rows = [r for r in rows if r["label"] == "plateau"]
            assert len(plateau_rows) >= 2, "need >=2 plateau samples to prove flatness"
            assert plateau_rows[-1]["VmSize"] == plateau_rows[-2]["VmSize"], \
                "VmSize must be flat during plateau"
    # segment step magnitude sanity: a fresh (post-upfront) allocation step must
    # be within one page of the computed per-segment size
    steps = [rows[i]["VmSize"] - rows[i - 1]["VmSize"] for i in range(2, len(rows))
             if rows[i]["label"].startswith("fine")]
    seg_kb = BYTES_PER_SEG_80c / 1024.0
    for st in steps:
        assert abs(st - seg_kb) <= 8, "fine-phase step %d kB not ~ one segment (%.1f kB)" % (st, seg_kb)

    print("-" * 92)
    print("FINAL count=%d ynum=%d inferred_segments=%d (num_segments NOT exposed -> inferred)"
          % (final_count, s.historybuf.ynum, inferred_segments(final_count)))
    print("FINAL VmRSS=%d kB VmSize=%d kB smapsRss=%d smapsPss=%d ru_maxrss=%d kB"
          % (after["VmRSS"], after["VmSize"], after["smaps_Rss"], after["smaps_Pss"], after["ru_maxrss"]))
    print("FINAL tracemalloc current=%d B peak=%d B  (Python-only; C segment calloc is INVISIBLE here)"
          % (after["tm_cur"], after["tm_peak"]))
    print("THROUGHPUT coarse window: fed %d lines in %.4f s = %.0f lines/s  [time.perf_counter, canonical parse_bytes]"
          % (coarse_lines, coarse_secs, coarse_lines / coarse_secs if coarse_secs else 0))
    segs = inferred_segments(final_count)
    print("CHECK inferred_segments*bytes/seg = %d * %d B = %.0f kB (expected virtual growth for segments)"
          % (segs, BYTES_PER_SEG_80c, segs * BYTES_PER_SEG_80c / 1024.0))
    print("ALL ASSERTIONS PASSED")
    return 0


def run_boundary(bt, run_index, m_proc0, m_import):
    """Q3 exact boundary: feed ONE line at a time across the 2nd-segment
    allocation and record count + VmSize at each single-line step so the
    external transition is bracketed at count 2047/2048/2049 (the 24-row
    visible screen offsets fed-lines, so we key on history *count*)."""
    scrollback = 300000
    s = bt.create_screen(cols=COLS, lines=LINES, scrollback=scrollback,
                         options={"scrollback_pager_history_size": 0})
    print("=" * 92)
    print("BOUNDARY exact 2nd-segment allocation  [run %d]  pid=%d cols=%d lines=%d scrollback=%d"
          % (run_index, os.getpid(), COLS, LINES, scrollback))
    print("=" * 92)
    print("computed bytes/segment @%dc = %d  (= %.4f MiB)" % (COLS, BYTES_PER_SEG_80c,
                                                              BYTES_PER_SEG_80c / (1024 * 1024)))
    from kitty_tests import parse_bytes
    # bulk-feed until history count reaches ~2040, then switch to 1-line steps
    i = 0
    while s.historybuf.count < SEGMENT_SIZE - 8:
        parse_bytes(s, make_line(i)); i += 1
    hdr = "%-6s %8s %6s %12s %12s %12s" % ("step", "count", "segs", "VmSize(kB)", "VmRSS(kB)", "dVmSize(kB)")
    print(hdr); print("-" * len(hdr))
    prev = None
    step_at = None
    # single-line steps from count~2040 to count~2056
    for _ in range(20):
        parse_bytes(s, make_line(i)); i += 1
        m = vm(); c = s.historybuf.count; seg = inferred_segments(c)
        d = "" if prev is None else ("%+d" % (m["VmSize"] - prev))
        print("%-6d %8d %6d %12d %12d %12s" % (i, c, seg, m["VmSize"], m["VmRSS"], d))
        if prev is not None and (m["VmSize"] - prev) > 1000 and step_at is None:
            step_at = c
        prev = m["VmSize"]
    print("-" * len(hdr))
    print("OBSERVED VmSize step first occurs at history count = %s" % step_at)
    # the 2nd segment (index 1) is allocated when history index 2048 is first
    # written, i.e. as count crosses 2048 -> 2049.
    assert step_at is not None, "no VmSize step observed across the boundary"
    assert 2048 <= step_at <= 2049, "step expected at count 2048/2049, got %s" % step_at
    print("ASSERTION PASSED: 2nd-segment allocation bracketed at count in {2048,2049}")
    return 0


if __name__ == "__main__":
    cond = sys.argv[1] if len(sys.argv) > 1 else "c1"
    ri = int(sys.argv[2]) if len(sys.argv) > 2 else 1
    sys.exit(run(cond, ri))
```

---

## Appendix B — Q2 GUI latency harness (verbatim)

Three files. `q2_run.sh` launches the **image's own kitty** against a host `Xvfb` with an owned PID + `EXIT` trap, a readiness marker, timeouts, `allow_remote_control=no` (no CWE‑284 remote‑control socket), and no `--hold`. `fill.py` runs *inside* kitty and prints the 40‑line `#` banner + 120,000 lines, then either idles or streams. `q2_controller.py` injects the real `ctrl+shift+home` via XTEST and detects the banner via `XGetImage`.

**`q2_run.sh`:**

```bash
#!/bin/bash
# Q2 GUI harness: run the MANDATED-IMAGE kitty binary against a host Xvfb,
# fill a large scrollback (top banner), then measure real scroll_home
# input->display latency from the host (XTEST inject + Pillow capture).
# Hardened: owned Xvfb PID, EXIT trap cleanup, readiness marker, timeouts,
# private display, NO remote control (allow_remote_control=no).
set -u
IMG="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0"
WORK="$(cd "$(dirname "$0")" && pwd)"
CFG="${1:?c4|c5}"; COND="${2:?idle|during}"; MODE="${3:?selftest|measure}"
NTRIALS="${4:-10}"; OUTCSV="${5:-$WORK/lat_${CFG}_${COND}.csv}"
TAG="${CFG}_${COND}_$$"
DISP=":$(( (RANDOM % 300) + 130 ))"
CNAME="q2_${TAG}"
XVFB_PID=""
cleanup(){
  docker rm -f "$CNAME" >/dev/null 2>&1 || true
  [ -n "$XVFB_PID" ] && kill "$XVFB_PID" 2>/dev/null || true
  rm -f "$WORK/READY_$TAG" "/tmp/.X${DISP#:}-lock" 2>/dev/null || true
}
trap cleanup EXIT
echo "### CFG=$CFG COND=$COND MODE=$MODE DISPLAY=$DISP container=$CNAME"
# 1) host Xvfb (owned)
Xvfb "$DISP" -screen 0 1280x800x24 -ac >"$WORK/xvfb_$TAG.log" 2>&1 &
XVFB_PID=$!
for i in $(seq 1 50); do DISPLAY="$DISP" python3 -c "import ctypes,os;x=ctypes.CDLL('libX11.so.6');x.XOpenDisplay.restype=ctypes.c_void_p;x.XOpenDisplay.argtypes=[ctypes.c_char_p];exit(0 if x.XOpenDisplay(os.environ['DISPLAY'].encode()) else 1)" 2>/dev/null && break; sleep 0.1; done
echo "Xvfb pid=$XVFB_PID ready on $DISP"
# 2) kitty options
COMMON="-o scrollback_lines=2000000 -o cursor_blink_interval=0 -o window_padding_width=0 -o remember_window_size=no -o initial_window_width=1200 -o initial_window_height=760 -o enable_audio_bell=no -o allow_remote_control=no -o font_size=12"
if [ "$CFG" = c4 ]; then CFGOPTS="-o input_delay=3 -o repaint_delay=10 -o sync_to_monitor=yes";
else CFGOPTS="-o input_delay=0 -o repaint_delay=2 -o sync_to_monitor=no"; fi
# 3) launch mandated-image kitty (exec -> kitty is PID 1 in container)
rm -f "$WORK/READY_$TAG"
docker run -d --rm --name "$CNAME" --network host \
  -e DISPLAY="$DISP" -e LIBGL_ALWAYS_SOFTWARE=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix -v "$WORK":/work \
  --entrypoint /bin/bash "$IMG" \
  -lc "cd /app && exec ./kitty/launcher/kitty --debug-rendering $COMMON $CFGOPTS python3 /work/fill.py $COND $TAG" \
  >/dev/null
# 4) readiness: wait for the fill marker (history populated)
for i in $(seq 1 300); do [ -f "$WORK/READY_$TAG" ] && break; sleep 0.1; done
if [ ! -f "$WORK/READY_$TAG" ]; then echo "FATAL: fill did not complete"; docker logs "$CNAME" 2>&1 | tail -20; exit 5; fi
sleep 1.5   # let kitty render the live view
# capture GL / window / child-launch evidence + thread names (once)
docker logs "$CNAME" 2>&1 | grep -E "GL version|OS Window created|Child launched" | head -5 | tee "$WORK/gl_$TAG.txt"
echo "-- kitty thread names (/proc/1/task/*/comm inside container) --" | tee -a "$WORK/threads_$TAG.txt"
docker exec "$CNAME" bash -lc 'for f in /proc/1/task/*/comm; do cat "$f"; done' 2>/dev/null | sort | uniq -c | tee -a "$WORK/threads_$TAG.txt"
# 5) run the controller (host)
BBOX="10,10,500,120"
if [ "$MODE" = selftest ]; then
  DISPLAY="$DISP" timeout 90 python3 "$WORK/q2_controller.py" selftest "$DISP" "$BBOX"; RC=$?
else
  DISPLAY="$DISP" timeout 180 python3 "$WORK/q2_controller.py" measure "$DISP" "$BBOX" "${CFG}_${COND}" "$NTRIALS" "$OUTCSV"; RC=$?
fi
echo "controller rc=$RC"
exit $RC
```

**`fill.py`:**

```python
import sys, time
mode = sys.argv[1] if len(sys.argv) > 1 else 'idle'
tag  = sys.argv[2] if len(sys.argv) > 2 else 't'
W = 76
# 40-line high-ink '#' banner = the oldest history (top). Visually unique; not
# produced by the numeric stream, so scroll_home landing on it is unambiguous.
for _ in range(40):
    sys.stdout.write('#' * W + '\n')
for i in range(120000):
    sys.stdout.write('out %08d %s\n' % (i, 'y' * 40))
sys.stdout.flush()
open('/work/READY_' + tag, 'w').close()
if mode == 'idle':
    time.sleep(600)
else:  # 'during' : continuous output while we scroll (concurrent activity)
    i = 0
    while True:
        sys.stdout.write('stream %08d %s\n' % (i, 'z' * 40)); i += 1
        if i % 100 == 0:
            sys.stdout.flush(); time.sleep(0.01)   # steady heavy stream
```

**`q2_controller.py`:**

```python
#!/usr/bin/env python3
"""
Q2 host-side controller. Inject a REAL scroll key via the X11 XTEST extension
(libXtst via ctypes -- no xdotool, no dev headers) into the focused kitty
window on the shared Xvfb display, and detect the exact moment the scrolled
view is rendered by reading the framebuffer with XGetImage over a persistent
X connection (~0.05-0.1 ms/grab -> sub-ms timing resolution).

Latency = t(scrolled view rendered) - t(key injected), via time.perf_counter().

Scroll target: scroll_home = kitty_mod(ctrl+shift)+home  [definition.py:3623]
  -> Window.scroll_home  [window.py:1860]
  -> Screen.scroll(SCROLL_FULL, True)
  -> screen_history_scroll  [kitty/screen.c:4091-4118]  (sets scrolled_by + dirty_scroll)
scroll_home jumps to the very TOP of history, which is a fixed 40-line '#'
banner (oldest scrolled-off lines). The banner is visually unique and never
produced by the numeric stream, so the SAME detector works idle AND while
output streams (top-of-history does not move: count < ynum, no eviction).

Usage:
  q2_controller.py selftest <DISPLAY> <bx,by,bw,bh>
  q2_controller.py measure  <DISPLAY> <bx,by,bw,bh> <label> <ntrials> <out.csv>
"""
import ctypes, os, sys, time

x = ctypes.CDLL("libX11.so.6")
xt = ctypes.CDLL("libXtst.so.6")
x.XOpenDisplay.restype = ctypes.c_void_p; x.XOpenDisplay.argtypes = [ctypes.c_char_p]
x.XDefaultRootWindow.restype = ctypes.c_ulong; x.XDefaultRootWindow.argtypes = [ctypes.c_void_p]
x.XStringToKeysym.restype = ctypes.c_ulong; x.XStringToKeysym.argtypes = [ctypes.c_char_p]
x.XKeysymToKeycode.restype = ctypes.c_ubyte; x.XKeysymToKeycode.argtypes = [ctypes.c_void_p, ctypes.c_ulong]
x.XGetImage.restype = ctypes.c_void_p
x.XGetImage.argtypes = [ctypes.c_void_p, ctypes.c_ulong, ctypes.c_int, ctypes.c_int,
                        ctypes.c_uint, ctypes.c_uint, ctypes.c_ulong, ctypes.c_int]
x.XDestroyImage.argtypes = [ctypes.c_void_p]
x.XFlush.argtypes = [ctypes.c_void_p]
xt.XTestFakeKeyEvent.argtypes = [ctypes.c_void_p, ctypes.c_uint, ctypes.c_int, ctypes.c_ulong]

DISPLAY = sys.argv[2]; os.environ["DISPLAY"] = DISPLAY
dpy = x.XOpenDisplay(DISPLAY.encode())
if not dpy:
    sys.stderr.write("cannot open display %s\n" % DISPLAY); sys.exit(2)
ROOT = x.XDefaultRootWindow(dpy)
ZPIX = 2; ALLPLANES = (1 << 32) - 1
BX, BY, BW, BH = (int(v) for v in sys.argv[3].split(","))


def kc(name):
    return x.XKeysymToKeycode(dpy, x.XStringToKeysym(name.encode()))
KC = {n: kc(n) for n in ("Control_L", "Shift_L", "Home", "End", "Prior", "Next")}


def combo(key):
    xt.XTestFakeKeyEvent(dpy, KC["Control_L"], 1, 0)
    xt.XTestFakeKeyEvent(dpy, KC["Shift_L"], 1, 0)
    xt.XTestFakeKeyEvent(dpy, KC[key], 1, 0)
    xt.XTestFakeKeyEvent(dpy, KC[key], 0, 0)
    xt.XTestFakeKeyEvent(dpy, KC["Shift_L"], 0, 0)
    xt.XTestFakeKeyEvent(dpy, KC["Control_L"], 0, 0)
    x.XFlush(dpy)


def sig():
    """Fast framebuffer signature (sum of sampled bytes) of the top strip."""
    img = x.XGetImage(dpy, ROOT, BX, BY, BW, BH, ALLPLANES, ZPIX)
    if not img:
        return -1
    dataptr = ctypes.cast(img + 16, ctypes.POINTER(ctypes.c_void_p))[0]
    bpl = ctypes.cast(img + 44, ctypes.POINTER(ctypes.c_int))[0]
    buf = ctypes.string_at(dataptr, BH * bpl)
    x.XDestroyImage(img)
    return sum(buf[::8])


def grab_ms():
    t = time.perf_counter()
    for _ in range(200):
        sig()
    return (time.perf_counter() - t) / 200 * 1000.0


def banner_sig():
    combo("Home"); time.sleep(0.4)
    s = sig(); time.sleep(0.05); s2 = sig()
    combo("End"); time.sleep(0.3)
    return (s + s2) // 2


def one_trial(bsig, tol, timeout_s=2.0):
    combo("End"); time.sleep(0.15)
    base = sig()
    gap = abs(base - bsig)
    t0 = time.perf_counter()
    combo("Home")
    frames = 0; hit = 0; deadline = t0 + timeout_s
    while time.perf_counter() < deadline:
        s = sig(); frames += 1
        if abs(s - bsig) <= tol:
            hit += 1
            if hit >= 2:                    # stable landing on the banner
                return (time.perf_counter() - t0), gap, frames
        else:
            hit = 0
    return None, gap, frames


if __name__ == "__main__":
    mode = sys.argv[1]
    print("keycodes:", {k: int(v) for k, v in KC.items()})
    gms = grab_ms()
    print("grab region (%d,%d,%d,%d)  XGetImage ~ %.3f ms/grab (timing resolution)" % (BX, BY, BW, BH, gms))
    bsig = banner_sig()
    # calibrate tolerance from the live-vs-banner gap
    combo("End"); time.sleep(0.3)
    base = sig(); gap = abs(base - bsig)
    tol = max(50, int(0.05 * gap))
    print("banner_sig=%d  live_sig=%d  gap=%d  match_tol=%d" % (bsig, base, gap, tol))
    if mode == "selftest":
        lat, g, frames = one_trial(bsig, tol)
        print("selftest scroll_home detected=%s latency=%s frames=%d gap=%d"
              % (lat is not None, ("%.2f ms" % (lat * 1000)) if lat else "NONE", frames, g))
        combo("End")
        sys.exit(0 if (lat is not None and gap > 500) else 1)
    elif mode == "measure":
        label = sys.argv[4]; n = int(sys.argv[5]); out = sys.argv[6]
        lats = []
        with open(out, "a") as fh:
            for i in range(n):
                lat, g, frames = one_trial(bsig, tol)
                if lat is not None:
                    lats.append(lat * 1000)
                    fh.write("%s,%d,%.4f,%d,%d\n" % (label, i + 1, lat * 1000, g, frames))
                    print("  %s trial %2d: %.2f ms (gap=%d frames=%d)" % (label, i + 1, lat * 1000, g, frames))
                else:
                    print("  %s trial %2d: NO-DETECT (gap=%d)" % (label, i + 1, g))
                time.sleep(0.2)
        if lats:
            ls = sorted(lats); m = len(ls)
            med = ls[m // 2] if m % 2 else (ls[m // 2 - 1] + ls[m // 2]) / 2
            mean = sum(ls) / m
            print("SUMMARY %s: n=%d min=%.2f median=%.2f mean=%.2f max=%.2f ms (grab_res~%.3f ms)"
                  % (label, m, ls[0], med, mean, ls[-1], gms))
        else:
            print("SUMMARY %s: NO SUCCESSFUL DETECTIONS" % label)
        sys.exit(0)
```

---

## Coverage checklist

Every sub‑question and every named mechanism/flag/condition is answered with its value, `file:line`, and evidence pointer.

| # | Item | Answer (measured / grounded) | `file:line` | Evidence |
|---|------|------------------------------|-------------|----------|
| Q1a | Memory as scrollback accumulates | Rises ≈ 5.13 MB per 2048 lines (one segment); linear in segment count | history.c:18-29 | C1/C2/C3 tables |
| Q1b | Before / during / after | before ≈ 42 MB (1 upfront seg); during linear; after flat at plateau | — | C1/C2/C3 `before`/`during`/`after` rows |
| Q1c | Default ceiling | ≈ 47 MB (1 segment, `ynum=2000<2048`) | screen.c:130; definition.py:372 | C1 (46,992/46,996/47,304 kB) |
| Q1d | Large finite ceiling | ≈ 794 MB at 300k lines / 147 segments, then flat | history.c:276-285 | C2 (793,640/793,928/794,052 kB) |
| Q1e | Python allocation tracing | `tracemalloc` peak ≈ 12 MB flat while RSS→794 MB (C `calloc` invisible) | — | C2 `tmPeak` column |
| Q1f | Scale / ≥2 runs / stability | 5k / 500k / 200k lines; 3 runs each; ≤ 0.66 % spread (C2/C3 ≤ 0.07 %) | — | three run files per condition |
| Q1g | procfs precision caveat | `smaps_rollup` Rss/Pss corroborate VmRSS to a few kB at every phase | proc.rst (kernel) | `smapsRss`/`smapsPss` columns |
| Q2a | Remains responsive? | Yes; 120/120 scrolls rendered; worst 16.13 ms ≪ 100 ms | child-monitor.c:1236-1237 | latency CSVs |
| Q2b | Latency value (idle) | ≈ 7 ms median (both configs) | window.py:1860; screen.c:4091 | c4_idle/c5_idle |
| Q2c | Latency value (during output) | C4 14.83 ms median; C5 10.37 ms median | definition.py:866,878,889 | c4_during/c5_during |
| Q2d | Prioritization signal | Bimodal during‑output dist.; scroll shares main‑thread repaint cadence | child-monitor.c:871,875-878 | c*_during CSV bimodality |
| Q2e | `input_delay` / `repaint_delay` / `sync_to_monitor` | C4 3/10/yes vs C5 0/2/no; C5 cuts during median ~30 %, max 16.1→12.9 | definition.py:878,866,889 | matrix table |
| Q2f | Thread model | parse+render on main thread; `KittyChildMon`=I/O; `KittyPeerMon` absent | child-monitor.c:1259,1489,1808,285 | live `/proc/1/task/*/comm` |
| Q2g | Real scroll path (`scroll_home`) | key→window.py→`Screen.scroll`→`screen_history_scroll` | window.py:1860; screen.c:4091 | injected ctrl+shift+home detected |
| Q2h | Benchmark arg order / value | options after `ascii` ignored; correct = 63.6–67.4 MB/s (throughput only) | command.go; parse-args.go; main.go | correct vs wrong run table |
| Q2i | Typometer scope | includes GPU/WM/VSync, excludes physical kbd/display device | — | methodology note |
| Q3a | When new storage allocated | On demand, per 2048‑row block, when a row in an unallocated block is written | history.c:37-42 | boundary run |
| Q3b | Exact transition | `VmSize +5,132 kB` as history `count` crosses 2048→2049 (seg 1→2) | history.c:18-29 | boundary tables (all runs + debug build) |
| Q3c | Upfront first segment | Reserved at `create_screen` (+5,132 kB VmSize, lazy RSS) | history.c:117-132 | C1/C2/C3 `post-create_screen` line |
| Q3d | Segment backing size @80c | 5,251,072 B; external step 5,255,168 B (= size + 1 page) | data-types.h:221,228,231-239 | computed line + boundary step |
| Q3e | Plateau | At `count==ynum`; overwrite oldest circularly; RSS flat | history.c:276-285 | C2 flat tail |
| Q3f | Capacity rule | `ynum = MAX(scrollback, lines)`; sb=10→ynum=24; sb=0→24 | screen.c:130 | edge‑case output |
| Q3g | Negative / infinite | `ynum=2**32‑1`; unbounded growth to OOM `fatal` (inferred) | utils.py:557-561; history.c:21,25 | C3 (no plateau, 98 segs) |
| Q3h | Pager history default | `scrollback_pager_history_size=0` → `alloc_pagerhist` returns NULL | definition.py:406; history.c:70-79 | override to 0 in every run |
| — | Canonical path (no bypass) | `parse_bytes`→vt-parser→INDEX_UP→history; no `HistoryBuf.push` | vt-parser.c:1451-1496; kitty_tests:30,237 | Environment section |
| — | Mandated build/run env | image `sha256:c0824992ad0b`, py3.12.3, gcc13.3.0 | — | Environment section |
| — | Read‑only scope | only this file changed; temp scripts removed | — | Acceptance section |

---

## Acceptance — scope & cleanup evidence

This investigation modified **no existing repository file**. The sole repository change is this document, `blitzy/documentation/kitty_815df1e210e0.md`. All observation scripts were created under a private `mktemp -d` (mode 0700) directory outside the repository, all GUI runs used disposable `--rm` containers and owned `Xvfb` PIDs with `EXIT`‑trap cleanup, and everything was removed after use. The final working‑tree state captured immediately before commit was:

```console
$ git status --porcelain          # only the deliverable is modified
 M blitzy/documentation/kitty_815df1e210e0.md

$ git status --porcelain --untracked-files=all | grep -v kitty_815df1e210e0.md   # no other tracked/untracked changes
(no output — nothing else changed or untracked)

$ for f in <all source files cited in this doc>; do git diff --quiet HEAD -- "$f" && echo "UNCHANGED  $f"; done
UNCHANGED  kitty/history.c
UNCHANGED  kitty/data-types.h
UNCHANGED  kitty/screen.c
UNCHANGED  kitty/child-monitor.c
UNCHANGED  kitty/state.c
UNCHANGED  kitty/options/definition.py
UNCHANGED  kitty/options/utils.py
UNCHANGED  kitty/vt-parser.c
UNCHANGED  kitty/line-buf.c
UNCHANGED  kitty/lineops.h
UNCHANGED  kitty/window.py
UNCHANGED  3rdparty/ringbuf/ringbuf.h
UNCHANGED  kitty_tests/__init__.py
UNCHANGED  setup.py
UNCHANGED  test.py
UNCHANGED  tools/cli/command.go
UNCHANGED  tools/cli/parse-args.go
UNCHANGED  tools/cmd/benchmark/main.go

$ ls -d /tmp/blitzy_adhoc_* 2>/dev/null   # all temporary observation scripts/workdir removed
(none — removed)

$ ps -eo comm | grep -xE "Xvfb|kitty|kitten"   # no leftover GUI/observation processes
(none)

$ docker ps -a --format "{{.Image}}" | grep swe-atlas   # no leftover observation containers
(none — all runs used docker run --rm)
```

Every in‑repo source file cited anywhere in this document — including all files in the grounding table above — was confirmed byte‑identical to the baseline (`git diff --quiet HEAD -- <file>` for each; regression‑only, unchanged).
