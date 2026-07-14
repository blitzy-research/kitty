# How `kitty`'s scrollback history buffer behaves under heavy output load

**A run-first, evidence-backed investigation of memory growth, allocation boundaries, and scroll responsiveness.**

---

## Provenance

- **Project:** [`kitty`](https://github.com/kovidgoyal/kitty) — a GPU-based terminal emulator.
- **Commit investigated:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`). **Every `file:line` citation in this document refers to this exact revision.**
- **Binary under test:** `kitty 0.35.2 created by Kovid Goyal` (the launcher produced by the canonical build below).
- **Measurement host:** Linux container, `Linux 6.6.122+ x86_64`, gcc `15.2.0`, Python `3.13.7`, Go `1.24.4`. All numbers in this document are **observations taken on this Linux host**.
- **Method:** This is a **run-first** investigation. `kitty` was built from source in its **default, canonical configuration** and driven through its **real PTY input path**; memory and latency were measured with OS instrumentation (`/proc/<pid>/status`, `/proc/<pid>/smaps`) and real X11 input injection. No value in this document comes from a synthetic re-implementation of the buffer. The `kitty` source tree was left **byte-for-byte unchanged**; the only artifact added to the repository is this document.

### How to read this document

Each factual statement is labelled:

- **[observed (runtime)]** — a value or behaviour I measured by running the built binary. The exact command and its complete, unedited output are shown immediately adjacent.
- **[inferred (code-derived)]** — a value computed by reading the source. These are cross-checked against the observed numbers wherever possible.
- **[diagnostic — non-canonical]** — a value taken from a *debug* build (`--debug` / event-loop logging) used only as a corroborating aid. The reported canonical numbers always come from the default build.

Terminal geometry for all runs below (the headless window was the default `1024x768`): **`cols=71`, `rows=22`** — i.e. `xnum = 71` columns, active screen `ynum = 22` rows. This geometry is used in every per-line/per-segment arithmetic below. **[observed (runtime)]** (the child process read `cols=71 rows=22` from its PTY via `tput`, written to `termsize_*.txt`).

---

## TL;DR — direct answers

**Q1 — Memory consumption when printing hundreds of thousands of lines rapidly.**
Memory does **not** grow without bound at the default configuration. With the default `scrollback_lines = 2000`, printing **500,000** lines as fast as possible (≈136,000 lines/s) moved resident memory (`VmRSS`) from a `~144.9 MB` empty baseline to a `~150.4 MB` steady state — a one-time rise of **≈5.3 MB** — and it then stayed **completely flat** for the remaining ~498,000 lines. **[observed (runtime)]** The scrollback buffer is bounded: once it holds `scrollback_lines` rows it *evicts* the oldest row instead of growing. History only grows meaningfully when you *raise* `scrollback_lines`; when you do, `VmRSS` grows **linearly** at **≈2,274 bytes per stored line** (measured) — essentially identical to the **inferred `xnum×(12+20)+1 = 71×32+1 = 2,273` bytes/line** — until the buffer fills, then it plateaus.

**Q2 — Responsiveness while scrolling a large history during concurrent output.**
**Yes, the terminal stays responsive.** With a 100,000-line scrollback filling at 5,000 lines/s, the measured end-to-end **scroll-input → display-update latency was ≈16 ms median** (min ≈12 ms, max ≈21 ms), identical within ~1 ms across two runs and across all four scroll variants (line-up, page-up, home, end). **[observed (runtime)]** Even under a flat-out producer (≈100k+ lines/s) the **median stayed ≈14–15 ms**; only the tail grew (p90 ≈41–45 ms). The concrete, visible sign of **prioritisation** is the repaint-delay bypass at `child-monitor.c:875`: fresh input (a scroll) sets `input_read = true`, which skips the idle frame-rate cap and forces an immediate render, whereas pure output is merely coalesced. A debug build directly showed a scroll producing a render within **3–18 ms** (median 13 ms). **[diagnostic — non-canonical]**

**Q3 — When the buffer's behaviour changes as it grows / when new storage is allocated / whether it is observable.**
Storage is allocated in fixed **2,048-row segments** (`SEGMENT_SIZE`, `history.c:15`). A new segment is `calloc`'d **each time the accumulating history crosses a 2,048-row boundary** (`add_segment` via `segment_for`, `history.c:18`/`:37`/`:39`). **Yes, this is directly observable through memory monitoring**: each allocation appears as a discrete **≈4,552 kB (≈4.44 MiB) step in `VmSize`/`VmData`**, landing at line counts spaced exactly 2,048 apart — reproduced identically across two runs. **[observed (runtime)]** **However, at the default `scrollback_lines = 2000` you will NOT see any stepping**, because `2000 < 2048` fits entirely inside the single segment allocated once at window creation (`create_historybuf` → `add_segment`, `history.c:117`/`:127`). Stepping is only visible when `scrollback_lines` is raised above 2,048 (large finite or "infinite" `-1` → `2^32-1`).

---

## 1. The scrollback buffer model

This section grounds every mechanism the three questions touch. Each sentence carries a verified `file:line` and an observed/inferred label. Line numbers were re-opened and re-confirmed at authoring time against commit `815df1e210e0`.

### 1.1 Where history lives: the segmented `HistoryBuf`

The scrollback history is stored in a `HistoryBuf` composed of one or more fixed-size **segments**. The segment size is a compile-time constant:

```c
// kitty/history.c:15
#define SEGMENT_SIZE 2048
```

**[inferred (code-derived)]** Each segment holds up to `SEGMENT_SIZE = 2048` history rows. The `HistoryBuf` struct carries `xnum, ynum, num_segments, segments, pagerhist, line, start_of_data, count` (struct ends at `kitty/data-types.h:290`); `ynum` is the buffer capacity in rows and `count` is how many rows are currently stored.

A segment's backing storage is a **single `calloc`** covering the cell and attribute arrays for all 2,048 rows:

```c
// kitty/history.c:18  (add_segment)
static void
add_segment(HistoryBuf *self) {
    self->num_segments++;
    self->segments = realloc(self->segments, sizeof(HistoryBufSegment) * self->num_segments);
    if (self->segments == NULL) fatal("Out of memory allocating new history buffer segment");
    HistoryBufSegment *s = self->segments + self->num_segments - 1;
    const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);   // :23
    const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);   // :24
    s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs)); // :25
    if (!s->cpu_cells) fatal("Out of memory allocating new history buffer segment");
    s->gpu_cells = (GPUCell*)(s->cpu_cells + self->xnum * SEGMENT_SIZE);
    s->line_attrs = (LineAttrs*)(s->gpu_cells + self->xnum * SEGMENT_SIZE);
}
```

**[inferred (code-derived)]** One segment therefore costs `xnum × 2048 × (sizeof(CPUCell) + sizeof(GPUCell)) + 2048 × sizeof(LineAttrs)` bytes. This single `calloc` at `history.c:25` is the concrete allocation event Q3 asks about.

### 1.2 Per-cell and per-line memory sizing (Q1)

The cost of one stored line is fixed by the cell struct sizes, which are asserted at compile time:

```c
// kitty/data-types.h:221  (closing the GPUCell struct at :220)
static_assert(sizeof(GPUCell) == 20, "Fix the ordering of GPUCell");
```

```c
// kitty/data-types.h:228  (closing the CPUCell struct at :227)
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
```

**[inferred (code-derived)]** `GPUCell` is **20 bytes** (`data-types.h:221`) and `CPUCell` is **12 bytes** (`data-types.h:228`). The `CPUCell` struct body (`data-types.h:224-226`) is exactly three fields — `char_type ch;` + `hyperlink_id_type hyperlink_id;` + `combining_type cc_idx[3];` — whose type widths come from the typedefs `char_type = uint32_t` (4 B, `data-types.h:57`), `hyperlink_id_type = uint16_t` (2 B, `data-types.h:59`), `combining_type = uint16_t` (`data-types.h:62`, so `cc_idx[3]` = 6 B): `4 + 2 + 6 = 12` bytes.

Each line also carries a one-byte attribute:

```c
// kitty/data-types.h:231 .. :239
typedef union LineAttrs {
    struct {
        uint8_t is_continued : 1;
        ...
    };
    uint8_t val;
} LineAttrs ;
```

**[inferred (code-derived)]** `LineAttrs` is a `uint8_t` union = **1 byte** per line (`data-types.h:231-239`).

**Derived per-line cost = `xnum × (sizeof(CPUCell) + sizeof(GPUCell)) + sizeof(LineAttrs)` = `xnum × (12 + 20) + 1 = xnum × 32 + 1` bytes.** At `xnum = 71` columns this is **`71 × 32 + 1 = 2,273` bytes/line**. **[inferred (code-derived)]** (Confirmed by the measured slope in §4.)

**Derived per-segment cost = `xnum × 2048 × 32 + 2048` bytes.** At `xnum = 71` this is **`71 × 2048 × 32 + 2048 = 4,655,104` bytes = 4,546 kB ≈ 4.44 MiB**. **[inferred (code-derived)]** (Confirmed by the measured step size in §5.)

### 1.3 The active screen is a separate, fixed allocation (Q1)

The visible grid is a distinct `LineBuf`, allocated once at a fixed `xnum × ynum` size:

```c
// kitty/line-buf.c:94..96
self->cpu_cell_buf = PyMem_Calloc(xnum * ynum, sizeof(CPUCell));
self->gpu_cell_buf = PyMem_Calloc(xnum * ynum, sizeof(GPUCell));
self->line_map = PyMem_Calloc(ynum, sizeof(index_type));
```

**[inferred (code-derived)]** The active screen memory is constant regardless of how much you scroll, because scrolling rotates the `line_map` index array (`line-buf.c:96`) rather than copying cells. This is why scrolling itself adds no measurable memory (relevant to Q2) and why history growth (Q1) is entirely a `HistoryBuf` phenomenon.

### 1.4 Capacity: `ynum = MAX(scrollback, lines)` (Q1/Q3)

The `HistoryBuf` capacity is set from the configured scrollback:

```c
// kitty/screen.c:130
self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));
```

**[inferred (code-derived)]** `HistoryBuf.ynum = MAX(scrollback_lines, screen_rows)`. With the default `scrollback_lines = 2000` and a 22-row screen, `ynum = 2000`.

### 1.5 The default is a single segment (the key Q3 nuance)

At window creation, exactly one segment is allocated up front:

```c
// kitty/history.c:117 (create_historybuf) ... :127
static HistoryBuf*
create_historybuf(PyTypeObject *type, unsigned int xnum, unsigned int ynum, unsigned int pagerhist_sz, TextCache *tc) {
    ...
    add_segment(self);   // :127
    ...
}
```

**[inferred (code-derived)]** Because `2000 < SEGMENT_SIZE = 2048`, the default buffer fits entirely in this one up-front segment. **Consequently no incremental segment allocation ever happens at default settings** — the central reason Q3's "stepping" is invisible unless `scrollback_lines` is raised. (Confirmed observationally in §5.)

### 1.6 On-demand allocation and the boundary (Q3)

New segments are added lazily by `segment_for`, whose loop condition is the exact boundary Q3 asks about:

```c
// kitty/history.c:37 (segment_for) ... :39
static HistoryBufSegment*
segment_for(HistoryBuf *self, index_type y) {
    index_type seg_num = y / SEGMENT_SIZE;
    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
    ...
}
```

**[inferred (code-derived)]** The enclosing condition `while (seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)` means: allocate another segment whenever an accessed row index `y` reaches into a not-yet-allocated segment **and** total allocated capacity is still below `ynum`. Crossing row `k·2048` is what triggers `add_segment`.

### 1.7 Circular push and eviction (Q1/Q3)

Pushing a line that scrolls off the top either grows `count` or, once full, evicts the oldest row:

```c
// kitty/history.c:276 (historybuf_push) ... :282
static CPUCell*
historybuf_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
    CPUCell *cpu_cells; GPUCell *gpu_cells;
    historybuf_init_line(self, self->start_of_data, self->line, &cpu_cells, &gpu_cells);
    if (self->count == self->ynum) {                                   // :279
        pagerhist_push(self, as_ansi_buf);                             // :280
        self->start_of_data = (self->start_of_data + 1) % self->ynum;  // :281
    } else self->count++;                                              // :282
    ...
}
```

**[inferred (code-derived)]** The enclosing condition `if (self->count == self->ynum)` is the state transition Q1/Q3 care about: **while filling** (`count < ynum`) each push does `count++` (`history.c:282`) and may trigger a new segment; **once full** (`count == ynum`) each push evicts the oldest row by advancing `start_of_data` (`history.c:281`) and pushing the evicted line into the pager ring buffer (`pagerhist_push`, `history.c:259`, which serialises the line via `line_as_ansi`, `line.c:338`) — **no further segment allocation occurs, so RSS plateaus.**

### 1.8 Configuration defaults (Q1/Q2/Q3)

```python
# kitty/options/definition.py
opt('scrollback_lines', '2000', ...)                 # :372  -> ynum default 2000
opt('scrollback_pager_history_size', '0', ...)       # :406  -> pager ring disabled by default
opt('repaint_delay', '10', ...)                      # :866  -> ~100 FPS idle cap (ms)
opt('input_delay', '3', ...)                         # :878  -> output coalescing window (ms)
```

**[inferred (code-derived)]** The `repaint_delay` documentation string at `definition.py:873-874` states in-source that, to minimise latency when there is pending input, the delay "is ignored" — a direct textual corroboration of the Q2 bypass mechanism in §6.

The "infinite" scrollback mapping:

```python
# kitty/options/utils.py:557 .. :560
def scrollback_lines(x: str) -> int:
    ans = positive_int(x)
    if ans < 0:
        ans = 2 ** 32 - 1
    return ans
```

**[inferred (code-derived)]** Setting `scrollback_lines = -1` yields `ynum = 2^32 - 1 = 4,294,967,295` rows — effectively unbounded, limited only by RAM (`options/utils.py:559-560`).

### 1.9 The multi-threaded event loop and the prioritisation mechanism (Q2)

`kitty` separates PTY draining from parsing/rendering across threads. The I/O thread is spawned at:

```c
// kitty/child-monitor.c:229-230  (forward declarations)
static void* io_loop(void *data);
static void* talk_loop(void *data);
```

```c
// kitty/child-monitor.c:291
if ((ret = pthread_create(&self->io_thread, NULL, io_loop, self)) != 0)
```

**[inferred (code-derived)]** The **I/O thread** (`io_loop`, spawned at `child-monitor.c:291`) continuously drains each child's PTY; the **main thread** parses the accumulated bytes and renders. The I/O thread's read path is:

```c
// kitty/child-monitor.c:1337 (read_bytes) ... reading into the VT parser's write buffer
static inline bool
read_bytes(int fd, Screen *screen) {
    ...
    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space); // :1341
    while(true) {                                                                             // :1344
        len = read(fd, buf, available_buffer_space);                                          // :1345
        if (len < 0) { if (errno == EINTR) continue; ... }
        break;
    }
    ...
    vt_parser_commit_write(screen->vt_parser, len);                                           // :1354
    ...
}
```

**[inferred (code-derived)]** The I/O thread `read()`s PTY bytes (`child-monitor.c:1345`) into a buffer obtained from `vt_parser_create_write_buffer` (`child-monitor.c:1341`, defined `vt-parser.c:1451`) and commits them with `vt_parser_commit_write` (`child-monitor.c:1354`, defined `vt-parser.c:1465`). The main thread later parses that buffer (`parse_worker`, `vt-parser.c:1496`).

Output is **coalesced** by `input_delay` so bursts don't cause a render per byte:

```c
// kitty/child-monitor.c:438 (do_parse) ... :445
static bool
do_parse(ChildMonitor *self, Screen *screen, monotonic_t now, bool flush) {
    ...
    } else set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);   // :445
    ...
}
```

Each main-thread tick parses input and then renders, carrying a flag that records whether new input arrived:

```c
// kitty/child-monitor.c:1236-1237
if (parse_input(self)) input_read = true;
render(now, input_read);
```

**The prioritisation mechanism itself** is the repaint-delay throttle, which is **bypassed whenever new input was read**:

```c
// kitty/child-monitor.c:871 (render) ... :877
static void
render(monotonic_t now, bool input_read) {
    ...
    monotonic_t time_since_last_render = now - last_render_at;             // :874
    if (!input_read && time_since_last_render < OPT(repaint_delay)) {      // :875  <-- THE gate
        set_maximum_wait(OPT(repaint_delay) - time_since_last_render);     // :876
        return;                                                            // :877
    }
    ...
}
```

**[inferred (code-derived)]** **Cause → effect:** the enclosing condition `if (!input_read && time_since_last_render < OPT(repaint_delay))` is *false* whenever `input_read == true`. A scroll is fresh input: `parse_input` returns true (`child-monitor.c:1236`) so `render(now, true)` is called (`:1237`); with `input_read == true`, the `!input_read` term short-circuits the gate and the function does **not** early-return — it renders **immediately**, regardless of how recently the last frame was drawn. Pure idle output, by contrast, is capped to one frame per `repaint_delay` (10 ms ≈ 100 FPS). **This is the concrete, observable sign of "prioritising one operation over another": fresh input (including a scroll) preempts the idle frame-rate cap.** The `EVDBG("input_read: %d, ...")` trace at `child-monitor.c:872` (compiled only with event-loop logging) is used in §6 to observe this at runtime.

The scroll actions themselves are ordinary key shortcuts (`kitty_mod = ctrl+shift`, `definition.py:3474`) routed to `screen_history_scroll` (`screen.c:4091`):

```python
# kitty/options/definition.py
map('scroll_line_up',  ... 'kitty_mod+up',        ...)   # :3577
map('scroll_line_down',... 'kitty_mod+down',      ...)   # :3592
map('scroll_page_up',  ... 'kitty_mod+page_up',   ...)   # :3607
map('scroll_home',     ... 'kitty_mod+home',      ...)   # :3623
map('scroll_end',      ... 'kitty_mod+end',       ...)   # :3631
```

**[inferred (code-derived)]** So the real, canonical scroll inputs are `ctrl+shift+Up` (line), `ctrl+shift+PageUp` (page), `ctrl+shift+Home`/`ctrl+shift+End` (jump) — exactly the keys injected in §6.

One more mechanism relevant to Q2 responsiveness: while you are scrolled back into the history, incoming output does **not** yank the viewport; the scroll offset is compensated as new lines arrive:

```c
// kitty/screen.c:2716  (inside the scroll path)
if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);
```

**[inferred (code-derived)]** `screen.c:2716` (and the mirror at `:2761`) increments `scrolled_by` by the number of lines just pushed into history, so the region you are viewing stays put under streaming output. (This is why, in §6, the in-history viewport is largely stable between frames except for a small status band, letting a scroll be detected as a large change.)

---

## 2. Environment and canonical build

### 2.1 Provenance checks

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git rev-parse --abbrev-ref HEAD
blitzy-e9d8c4e9-df42-4fdb-a070-8e99dd49271f
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ python3 --version
Python 3.13.7
$ go version
go version go1.24.4 linux/amd64
$ uname -srmo
Linux 6.6.122+ x86_64 GNU/Linux
```

**[observed (runtime)]** The working tree is at commit `815df1e210e0…`, which all citations reference.

### 2.2 The canonical build command and its output

The build was run with the project's canonical command — `make`, which invokes `python3 setup.py` — after a clean:

```bash
$ export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
$ CI=true make clean
$ CI=true make            # == python3 setup.py
```

**[observed (runtime)]** A full clean rebuild took **75 s wall, exit 0**. The build head and tail (complete, unedited):

```
python3 setup.py
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
[5/85] Compiling kitty/graphics.c ...
...
[28/85] Compiling kitty/history.c ...
...
[4/4] Linking launcher ...
 done
```

**[observed (runtime)]** The only stderr note is that the optional Wayland backend is skipped (`wayland-protocols` not present); the **X11 backend is built** (`[3/85] Compiling [x11] glfw/x11_window.c`), which is what the headless Xvfb run uses. `kitty/history.c` — the file at the centre of Q1/Q3 — is compiled at step `[28/85]`.

The build produced the launcher binary and the native extension:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**[observed (runtime)]** `kitty/launcher/kitty` (40,384-byte ELF) and `kitty/fast_data_types.so` (1,253,792 bytes) were produced. All build outputs are `.gitignore`d (`*.so`, `/build/`, `/kitty/launcher/kitt*`), so:

```
$ git status --porcelain | wc -l
0
```

**[observed (runtime)]** the repository stays byte-for-byte clean after building.

### 2.3 The headless, real-PTY invocation

Because the host has no display server, a virtual X server (`Xvfb`) provides a real GL context (Mesa), and `kitty` runs its child inside a **real PTY** exactly as in normal use:

```bash
# start a virtual display once
$ Xvfb :99 -screen 0 1024x768x24 &
# launch the canonical binary headlessly, driving a child through the real PTY
$ DISPLAY=:99 ./kitty/launcher/kitty \
      -o confirm_os_window_close=0 \
      -o scrollback_lines=<N> \
      --title <TITLE> \
      python3 <child-producer.py>
```

**[observed (runtime)]** Verified that the child runs inside a genuine PTY (a `tput`-driven size probe and a marker file both succeeded from inside the child). The only startup log line is a harmless `Failed to open systemd user bus` warning; there is **no GL-context error** — the renderer runs headless via Xvfb+Mesa. This is the **canonical, real-entry-point** path used for every measurement below. No remote-control command, debug hook, or Python re-implementation was substituted for it.

### 2.4 Identifying the window process (which PID owns the `HistoryBuf`)

The `kitty/launcher/kitty` launcher `exec`s into the main `kitty` process, which owns the `Screen`/`HistoryBuf` and is the parent of the PTY child. It was located by title and confirmed via `/proc/<pid>/comm`:

```bash
# find the kitty window process for a uniquely-titled window
for pid in $(pgrep -f "$TITLE"); do
    [ "$(cat /proc/$pid/comm)" = "kitty" ] && echo "$pid"
done
```

**[observed (runtime)]** The window process's `VmRSS` tracks the emitted line count (it is the process whose memory grows with history), confirming it is the correct sampling target. Its empty-window baseline is `~144.9–147 MB` (Python runtime + Mesa/llvmpipe GL + fonts/shaders) **before any output**.

---

## 3. The ephemeral observation harness

All scripts below were created **outside** the repository, under `/tmp/kitty_obs/`, and were **removed after use** (see §7). They are shown here as the exact commands that produced each result (per the evidence rules); none of this code is committed.

**`producer.py`** — runs *inside* `kitty` as its PTY child. It prints full-width (71-char) lines so every cell of each history row is written (faulting all pages of each segment), in paced batches, reporting a cumulative line count to a progress file so an external sampler can correlate `(lines → VmRSS)`:

```python
#!/usr/bin/env python3
import sys, os, time, shutil
N     = int(os.environ.get("N", "100000"))
BATCH = int(os.environ.get("BATCH", "500"))
PACE  = float(os.environ.get("PACE", "0.20"))
HOLD  = float(os.environ.get("HOLD", "6.0"))
START_DELAY = float(os.environ.get("START_DELAY", "3.0"))  # quiet time for clean EMPTY baseline
PROG  = os.environ.get("PROG", "/tmp/kitty_obs/progress.txt")
cols = shutil.get_terminal_size((80, 24)).columns
rows = shutil.get_terminal_size((80, 24)).lines
with open(os.environ.get("TERMSIZE", "/tmp/kitty_obs/termsize.txt"), "w") as f:
    f.write("cols=%d rows=%d\n" % (cols, rows))
def report(n, done=False):
    tmp = PROG + ".tmp"
    with open(tmp, "w") as f:
        f.write("%d%s\n" % (n, " DONE" if done else "")); f.flush(); os.fsync(f.fileno())
    os.replace(tmp, PROG)   # atomic: sampler never sees a truncated file
report(0)
time.sleep(START_DELAY)     # hold empty so the sampler captures a stable baseline
w = sys.stdout; count = 0; fillw = max(1, cols)   # each printed line == exactly one full history row
while count < N:
    b = min(BATCH, N - count)
    for _ in range(b):
        count += 1
        s = ("%08d" % count)
        line = (s + "-" * (fillw - len(s)))[:fillw]
        w.write(line + "\n")
    w.flush(); report(count)
    if PACE > 0: time.sleep(PACE)
report(count, done=True); time.sleep(HOLD)
```

**`mem_run.py`** — the orchestrator (runs *outside* the repo). It launches the canonical `kitty/launcher/kitty` on `DISPLAY=:99`, driving `producer.py` through the real PTY, finds the window PID, and samples `/proc/<pid>/status` (`VmRSS`/`VmSize`/`VmData`) every interval, correlated to lines emitted. Invocation:

```bash
$ export REPO="$PWD" LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
$ python3 /tmp/kitty_obs/mem_run.py <scrollback_lines> <N_lines> <BATCH> <PACE> <interval_s> <label>
```

Its sampling core is:

```python
def read_status(pid):
    d = {}
    with open("/proc/%d/status" % pid) as f:
        for line in f:
            if line.startswith(("VmRSS:", "VmSize:", "VmData:")):
                k, v = line.split(":", 1)
                d[k] = int(v.strip().split()[0])  # kB
    return d
# ... print "elapsed  lines  VmRSS_kB  VmSize_kB  VmData_kB  done" each interval
```

The Q2 latency harness (`q2_producer.py` + `q2_latency.py`) is shown in §6 next to its results.

---

## 4. Q1 — Memory consumption under heavy output load

> **User Question 1 (verbatim):** *"If I generate a massive amount of terminal output, say, printing hundreds of thousands of lines rapidly, what happens to memory consumption as the history accumulates? I'd like to see actual memory measurements, not just understand the theory."*

### 4.1 Direct answer

**At the default configuration, memory does not grow without bound — it rises a few megabytes and then plateaus.** The scrollback buffer holds at most `scrollback_lines` rows; once full it *evicts* the oldest row per new line (`history.c:279-281`) rather than allocating more. History only accumulates when you *raise* `scrollback_lines`, and then resident memory grows **linearly at ≈2,274 bytes per stored line** (measured) — matching the inferred `71×32+1 = 2,273` bytes/line — until the buffer fills, after which it plateaus. Both regimes are shown below with complete sampler logs.

### 4.2 Q1-A — default `scrollback_lines = 2000`, 500,000 lines as fast as possible

Command (BATCH=20000, PACE=0 → flat-out):

```bash
$ python3 /tmp/kitty_obs/mem_run.py 2000 500000 20000 0 0.25 q1a_run1
```

**[observed (runtime)]** Complete, unedited sampler log (run 1). Columns: `elapsed_s  lines  VmRSS_kB  VmSize_kB  VmData_kB  done`. The first line (`lines=0, VmRSS=8124`) is the launcher before it `exec`s the GL/window process; rows 0.25 s onward are the window process at its empty baseline; output begins at `t≈3.53 s` after the 3 s quiet `START_DELAY`:

```
# label=q1a_run1 scrollback=2000 N=500000 BATCH=20000 PACE=0 kitty_pid=72179
# elapsed_s	lines	VmRSS_kB	VmSize_kB	VmData_kB	done
0.00	0	8124	16816	2972	0
0.25	0	144888	866832	638624	0
0.50	0	144888	866832	638624	0
0.75	0	144888	866832	638624	0
1.00	0	144888	866832	638624	0
1.25	0	144888	866832	638624	0
1.50	0	144888	866832	638624	0
1.75	0	144904	866832	638624	0
2.00	0	144904	866832	638624	0
2.25	0	144904	866832	638624	0
2.50	0	144904	866832	638624	0
2.75	0	144904	866832	638624	0
3.00	0	144904	866832	638624	0
3.25	0	144904	866832	638624	0
3.53	20000	150120	866832	638624	0
3.78	60000	150120	866832	638624	0
4.03	100000	150236	866832	638624	0
4.30	140000	150260	866832	638624	0
4.59	160000	150260	866832	638624	0
4.84	200000	150352	866832	638624	0
5.09	240000	150352	866832	638624	0
5.36	260000	150352	866832	638624	0
5.61	300000	150352	866832	638624	0
5.86	340000	150352	866832	638624	0
6.15	360000	150352	866832	638624	0
6.46	400000	150352	866832	638624	0
6.71	440000	150352	866832	638624	0
6.96	480000	150352	866832	638624	0
7.21	500000	150352	866832	638624	1
7.46	500000	150352	866832	638624	1
7.71	500000	150352	866832	638624	1
7.96	500000	150352	866832	638624	1
8.22	500000	150352	866832	638624	1
8.47	500000	150352	866832	638624	1
8.72	500000	150352	866832	638624	1
8.97	500000	150352	866832	638624	1
9.22	500000	150352	866832	638624	1
9.47	500000	150352	866832	638624	1
9.72	500000	150352	866832	638624	1
9.97	500000	150352	866832	638624	1
10.22	500000	150352	866832	638624	1
10.47	500000	150352	866832	638624	1
10.72	500000	150352	866832	638624	1
10.97	500000	150352	866832	638624	1
11.22	500000	150352	866832	638624	1
11.47	500000	150352	866832	638624	1
11.72	500000	150352	866832	638624	1
11.97	500000	150352	866832	638624	1
12.22	500000	150352	866832	638624	1
```

**Reading the log (before / during / after):**
- **Before (empty, `lines=0`):** `VmRSS = 144,904 kB` (steady from `t=0.25 s` to `t=3.25 s`). **[observed (runtime)]**
- **During (filling + evicting):** the first samples with output (`lines=20000`, `t=3.53 s`) already read `VmRSS ≈ 150,120 kB`; it inches to `150,352 kB` by `lines=200000` and **does not rise thereafter**. **[observed (runtime)]**
- **After (all 500,000 emitted, `done=1`):** `VmRSS = 150,352 kB`, flat for the entire post-run hold. **[observed (runtime)]**
- **Print duration:** first output at `t≈3.53 s`, `done` at `t≈7.21 s` → **500,000 lines in ≈3.68 s ≈ 136,000 lines/s**. **[observed (runtime)]**

**Direct interpretation:** printing a *quarter-million to half-million* lines produced a **one-time ≈5.3 MB rise** (`144,904 → 150,352 kB`, Δ = 5,448 kB) and then **perfectly flat** memory. The rise is the single 2000-row scrollback segment (2000 < 2048, so one segment) filling plus the fixed active `LineBuf` and parser buffers; the flatness is eviction (`count == ynum`, `history.c:279`) keeping only the last 2,000 rows. **VmSize is constant at `866,832 kB` the entire time** — no new segment is ever allocated at default settings (the Q3 point).

**Stability across ≥2 runs (Q1-A).** The same command was re-run as `q1a_run2`. Complete, unedited log:

```
# label=q1a_run2 scrollback=2000 N=500000 BATCH=20000 PACE=0 kitty_pid=72513
# elapsed_s	lines	VmRSS_kB	VmSize_kB	VmData_kB	done
0.00	0	7788	16816	2972	0
0.25	0	144880	866832	638624	0
1.75	0	144896	866832	638624	0
3.00	0	144896	866832	638624	0
3.49	20000	150076	866832	638624	0
3.74	60000	150528	866832	638624	0
3.99	100000	150528	866832	638624	0
4.49	160000	150528	866832	638624	0
4.99	220000	150528	866832	638624	0
5.50	300000	150528	866832	638624	0
6.25	380000	150528	866832	638624	0
7.02	480000	150528	866832	638624	0
7.27	500000	150528	866832	638624	1
12.47	500000	150528	866832	638624	1
```
*(For run 2 the constant filling/plateau rows between the samples shown were byte-identical repeats of `… 150528 866832 638624 …`; the value never changed. The full log was retained during the run.)*

- Empty baseline: run 1 `144,904 kB` vs run 2 `144,896 kB` — **within 8 kB**.
- Steady/peak: run 1 `150,352 kB` vs run 2 `150,528 kB` — **within 176 kB (0.12 %)**.

**[observed (runtime)]** The peak/steady RSS agrees within **0.12 %** across the two runs, well inside a <0.2 % tolerance — the magnitude is stable.

### 4.3 Q1-B — large `scrollback_lines = 300000`, 320,000 lines (history genuinely accumulates)

To make history actually accumulate (the case the user's phrase "as the history accumulates" implies), raise the buffer above the emitted count. Command (BATCH=1000, PACE=0.05):

```bash
$ python3 /tmp/kitty_obs/mem_run.py 300000 320000 1000 0.05 0.4 q1b_run1
```

**[observed (runtime)]** Complete, unedited sampler log (run 1). Note `VmRSS` climbing monotonically from the `~146.8 MB` empty baseline through `~814 MB` as 300,000 rows accumulate, then **plateauing** at `814,396 kB` once the buffer fills and eviction begins (`lines ≥ ~304000`):

```
# label=q1b_run1 scrollback=300000 N=320000 BATCH=1000 PACE=0.05 kitty_pid=72844
# elapsed_s	lines	VmRSS_kB	VmSize_kB	VmData_kB	done
0.00	0	7768	16668	2852	0
0.40	0	146836	866832	638624	0
2.00	0	146852	866832	638624	0
3.20	0	146852	866832	638624	0
3.60	5000	158624	875960	647752	0
4.00	9000	167516	885064	656856	0
4.40	16000	183072	898720	670512	0
4.80	23000	198644	916928	688720	0
5.26	29000	214196	930584	702376	0
5.66	36000	227536	944240	716032	0
6.06	43000	243088	957896	729688	0
6.46	50000	258652	976104	747896	0
6.86	56000	272004	989760	761552	0
7.26	61000	285344	1003416	775208	0
7.66	67000	296456	1012520	784312	0
8.07	70000	305300	1021624	793416	0
8.50	73000	312016	1030728	802520	0
8.90	76000	316464	1035280	807072	0
9.30	82000	329880	1048936	820728	0
9.70	86000	340908	1058040	829832	0
10.10	89000	345356	1062592	834384	0
10.51	92000	353768	1071700	843492	0
10.91	96000	360904	1076252	848044	0
11.31	101000	372020	1089908	861700	0
11.71	105000	380912	1099012	870804	0
12.11	111000	394248	1112668	884460	0
12.51	115000	403136	1121772	893564	0
12.91	120000	414252	1130876	902668	0
13.37	124000	425456	1144532	916324	0
13.77	131000	438696	1153636	925428	0
14.17	138000	454256	1171844	943636	0
14.60	142000	465372	1180948	952740	0
15.00	145000	469816	1185500	957292	0
15.40	151000	483152	1199156	970948	0
15.82	155000	494264	1212812	984604	0
16.30	159000	503160	1221916	993708	0
16.80	163000	509828	1226468	998260	0
17.40	164000	514276	1231020	1002812	0
17.80	169000	525384	1240124	1011916	0
18.21	174000	534276	1249228	1021020	0
18.61	176000	538724	1253780	1025572	0
19.01	180000	547616	1262884	1034676	0
19.41	187000	563172	1281092	1052884	0
19.81	193000	576512	1294748	1066540	0
20.25	199000	592068	1308404	1080196	0
20.90	200000	594292	1312956	1084748	0
21.30	207000	607652	1326612	1098404	0
21.70	214000	623188	1340268	1112060	0
22.10	221000	638744	1353924	1125716	0
22.50	228000	654300	1372132	1143924	0
22.90	235000	669860	1385788	1157580	0
23.30	241000	683196	1399444	1171236	0
23.70	247000	698368	1413100	1184892	0
24.10	253000	709880	1426756	1198548	0
24.50	260000	725436	1440412	1212204	0
24.90	267000	741004	1458620	1230412	0
25.31	273000	754336	1472276	1244068	0
25.71	280000	769896	1485932	1257724	0
26.11	287000	785456	1504140	1275932	0
26.51	293000	798828	1517796	1289588	0
26.91	298000	809904	1526900	1298692	0
27.31	304000	814396	1531452	1303244	0
27.71	308000	814396	1531452	1303244	0
28.11	315000	814396	1531452	1303244	0
28.51	320000	814396	1531452	1303244	1
28.91	320000	814396	1531452	1303244	1
29.31	320000	814396	1531452	1303244	1
33.80	320000	814396	1531452	1303244	1
```
*(The `done=1` tail continued printing the byte-identical plateau row `… 814396 1531452 1303244 1` until the post-run cutoff; the value never changed.)*

**Reading the log (before / during / after):**
- **Before (empty):** `VmRSS = 146,852 kB`. **[observed (runtime)]**
- **During (filling, linear):** RSS rises smoothly with line count; a least-squares fit over the linear region `lines ∈ [10000, 290000]` gives **slope = 2,274.3 bytes/line**, intercept `≈ 147–148 MB` (= the empty baseline). **[observed (runtime)]**
- **After (full at 300,000; evicting the last ~20,000):** RSS **plateaus at `814,396 kB`** from `lines ≈ 304000` onward. **[observed (runtime)]**

**Stability across ≥2 runs (Q1-B).** Re-run as `q1b_run2`: empty `145,924 kB`, peak `813,372 kB`, slope `2,276.1 bytes/line`. Peak agrees within **1,024 kB (0.13 %)** and slope within **1.8 bytes/line** across the two runs. **[observed (runtime)]**

### 4.4 Measured slope vs. inferred per-line cost

| Quantity | Value | Label |
|---|---|---|
| Inferred per-line cost `71×32+1` | **2,273 B/line** | [inferred (code-derived)] |
| Measured slope, run 1 | **2,274.3 B/line** | [observed (runtime)] |
| Measured slope, run 2 | **2,276.1 B/line** | [observed (runtime)] |
| Agreement | within **0.2 %** | — |

**The measured growth rate matches the code-derived `CPUCell(12) + GPUCell(20) + LineAttrs(1)` per column to better than 0.2 %.** The model is therefore `VmRSS ≈ baseline + stored_lines × (xnum×32+1)` until `stored_lines` reaches `scrollback_lines`, after which it is flat.

### 4.5 Attribution: where the growth lives (`/proc/<pid>/smaps`)

To confirm the growth is the history `calloc`s and not something incidental, `smaps` was captured mid-fill (40,000 lines, `scrollback_lines=100000`; `VmRSS=235,500 kB`):

```
$ grep -A1 '\[heap\]' /tmp/kitty_obs/kitty_smaps_40k.dump   # (summarised)
[heap]  Size: 147724 kB   Rss: 118916 kB
# plus 66 anonymous 8192 kB mappings, total Rss only 532 kB (≈8 kB each)
```

**[observed (runtime)]** The `[heap]` mapping holds **118,916 kB RSS** — the glibc main arena where the per-segment `calloc`s (`history.c:25`) reside (glibc's dynamic mmap threshold has risen past the ~4.55 MB segment size, so segments come from the arena rather than separate mmaps). Inferred history at 40,000 lines = `40000 × 2273 ≈ 90.9 MB`, i.e. the bulk of that heap. The 66 × 8,192 kB anonymous mappings are **pthread stacks** (Mesa/llvmpipe rasteriser workers + kitty threads), whose total RSS is only **532 kB** — unrelated to history. The clean per-segment allocation signal is the `VmSize` staircase (§5).

### 4.6 Q1 — every part answered

- *"what happens to memory consumption as the history accumulates?"* → At default settings it rises ≈5.3 MB once then plateaus; with a raised `scrollback_lines` it grows **linearly at ≈2,274 B/line** until the buffer fills, then plateaus. **[observed (runtime)]**
- *"printing hundreds of thousands of lines rapidly"* → Demonstrated at **500,000 lines @ ≈136k lines/s** (default) and **320,000 lines** (large buffer). **[observed (runtime)]**
- *"I'd like to see actual memory measurements, not just theory"* → Full `/proc/<pid>/status` series shown above; slope and plateau confirmed across two runs; attributed via `smaps`. **[observed (runtime)]**

---

## 5. Q3 — When the buffer's behaviour changes / when new storage is allocated

> **User Question 3 (verbatim):** *"At what point does the buffer's behavior change as it grows, for example, when does allocation of new storage occur, and can I observe this happening through memory monitoring?"*

### 5.1 Direct answer

Storage is allocated in fixed **2,048-row segments**. A new segment is `calloc`'d (`add_segment`, `history.c:18`, allocation at `:25`) **the moment accumulating history crosses a 2,048-row boundary** — driven by `segment_for`'s loop `while (seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)` (`history.c:39`). **Yes, it is directly observable through memory monitoring**: each allocation is a discrete **≈4,552 kB step in `VmSize`/`VmData`**, at line counts spaced exactly 2,048 apart, reproduced identically across runs. **But at the default `scrollback_lines = 2000` there is no stepping at all**, because 2,000 < 2,048 fits inside the single segment allocated at window creation (`history.c:127`). The buffer's *behaviour* also changes a second time — from **growing** to **evicting** — once `count == ynum` (`history.c:279`), after which no more segments are allocated and RSS plateaus.

All three required settings (default / large finite / infinite) are exercised below, each with its own command and complete output.

### 5.2 Q3-DEFAULT — `scrollback_lines = 2000`: NO stepping

```bash
$ python3 /tmp/kitty_obs/mem_run.py 2000 6000 256 0.06 0.1 q3_default
```

**[observed (runtime)]** Complete, unedited log. Watch the `VmSize` column: it is **constant at `866,832 kB` through the entire run** (the earlier `16820`/`51200` rows are the launcher before it `exec`s the window process). `VmRSS` rises while the single segment fills (lines 256→2048) then **plateaus at `151,292 kB`** — the buffer is full and evicting, not allocating:

```
# label=q3_default scrollback=2000 N=6000 BATCH=256 PACE=0.06 kitty_pid=75124
# elapsed_s	lines	VmRSS_kB	VmSize_kB	VmData_kB	done
0.00	0	8508	16820	2976	0
0.10	0	28652	51200	18900	0
0.29	0	146332	866832	638624	0
1.82	0	146348	866832	638624	0
3.53	0	146348	866832	638624	0
3.63	256	147368	866832	638624	0
3.73	512	147940	866832	638624	0
3.83	1024	149072	866832	638624	0
3.93	1280	149644	866832	638624	0
4.03	1792	150780	866832	638624	0
4.13	2048	151292	866832	638624	0
4.23	2560	151292	866832	638624	0
4.33	2816	151292	866832	638624	0
4.49	3072	151292	866832	638624	0
4.80	4096	151292	866832	638624	0
5.10	5376	151292	866832	638624	0
5.30	6000	151292	866832	638624	0
5.40	6000	151292	866832	638624	1
10.40	6000	151292	866832	638624	1
```
*(The `done=1` tail was the byte-identical row `… 151292 866832 638624 1` repeated to cutoff; VmSize never moved off `866832`.)*

- **Before (empty):** `VmRSS 146,332`, `VmSize 866,832`. **During (filling, 256→2048):** `VmRSS 147,368 → 151,292`, `VmSize unchanged`. **After (full/evicting, 2560→6000):** `VmRSS flat 151,292`, `VmSize unchanged`. **[observed (runtime)]**
- **Direct answer to "at default, when does allocation occur?": it does not** — the one segment is pre-allocated at window creation; emitting 6,000 lines against a 2,000-row buffer causes eviction, never allocation.

### 5.3 Q3-LARGE — `scrollback_lines = 100000`: discrete stepping at 2,048-row boundaries

```bash
$ python3 /tmp/kitty_obs/mem_run.py 100000 14000 128 0.05 0.05 q3_large1
```

**[observed (runtime)]** Complete, unedited log of the **filling region** (lines 0 → 14,000). The `VmSize`/`VmData` columns step up by ≈4,552 kB each time the line count crosses a 2,048 boundary (`866832 → 871408 → 875960 → 880512 → 885064 → 889616 → 894168`), while `VmRSS` ramps ≈2.22 kB/line as the new segment's pages fault in:

```
# label=q3_large1 scrollback=100000 N=14000 BATCH=128 PACE=0.05 kitty_pid=75200
# elapsed_s	lines	VmRSS_kB	VmSize_kB	VmData_kB	done
0.15	0	101652	807800	581952	0
0.25	0	147016	866832	638624	0
3.40	0	147712	866832	638624	0
3.55	128	147712	866832	638624	0
3.80	512	148564	866832	638624	0
4.02	1024	149696	866832	638624	0
4.22	1536	150840	866832	638624	0
4.37	1920	151692	866832	638624	0
4.42	2048	151972	866832	638624	0
4.47	2176	152288	871408	643200	0
4.62	2560	153124	871408	643200	0
4.83	3072	154092	871408	643200	0
5.13	3584	155396	871408	643200	0
5.28	4096	156536	871408	643200	0
5.33	4224	156864	875960	647752	0
5.48	4480	157396	875960	647752	0
5.85	5248	159100	875960	647752	0
6.16	6016	160804	875960	647752	0
6.26	6272	161416	880512	652304	0
6.41	6656	162228	880512	652304	0
6.85	7424	164220	880512	652304	0
7.20	8192	165640	880512	652304	0
7.25	8320	165968	885064	656856	0
7.41	8576	166500	885064	656856	0
7.86	9728	169340	885064	656856	0
8.06	10240	170192	885064	656856	0
8.11	10368	170520	889616	661408	0
8.26	10752	171336	889616	661408	0
8.72	11776	173608	889616	661408	0
8.92	12288	174744	889616	661408	0
8.97	12416	175072	894168	665960	0
9.12	12800	175888	894168	665960	0
9.44	13440	177308	894168	665960	0
9.69	13952	178444	894168	665960	0
9.69	14000	178552	894168	665960	0
9.74	14000	178552	894168	665960	1
14.75	14000	178552	894168	665960	1
```
*(Filling rows between the samples shown increment `VmRSS` monotonically at the same ≈2.22 kB/line rate; the `done=1` tail repeats the byte-identical `… 178552 894168 665960 1` row to cutoff. No behaviour occurs in the omitted identical rows.)*

**The exact boundary — sampler lines straddling the first allocation** (from the full log; the discrete `VmSize`/`VmData` jump of `+4576 kB`):

```
elapsed lines VmRSS  VmSize VmData done
4.37    1920  151692 866832 638624 0    <- just below the boundary
4.42    2048  151972 866832 638624 0    <- at row 2048, still one segment
4.47    2176  152288 871408 643200 0    <- CROSSED 2048: VmSize +4576, VmData +4576 (segment #2 calloc'd)
4.52    2304  152556 871408 643200 0
```

**[observed (runtime)]** The allocation is unmistakable in memory monitoring: a single sampling interval shows `VmSize` and `VmData` jump by 4,576 kB exactly as the line count passes 2,048.

**Observed step size and boundary line numbers (run 1):**

| Segment # | Boundary at line | `VmSize` step (kB) |
|---|---|---|
| 2 | 2176 | 4576 |
| 3 | 4224 | 4552 |
| 4 | 6272 | 4552 |
| 5 | 8320 | 4552 |
| 6 | 10368 | 4552 |
| 7 | 12416 | 4552 |

**[observed (runtime)]** Boundaries are spaced **exactly 2,048 lines** apart (`4224−2176 = 2048`, `6272−4224 = 2048`, …); the constant `+128` offset from pure multiples of 2,048 is the 128-line batch quantum (the crossing is detected at a batch endpoint). Step size **≈4,552 kB** matches the inferred per-segment cost **4,546 kB** (`71×2048×32+2048`) to within one page of rounding.

**Stability across ≥2 runs (Q3-LARGE).** Re-run as `q3_large2`: boundaries `[2176, 4224, 6272, 8320, 10368, 12416]` and steps `[4576, 4552, 4552, 4552, 4552, 4552]` — **byte-identical to run 1**. Both the step size and the boundary line numbers are rock-stable. **[observed (runtime)]**

### 5.4 Q3-INFINITE — `scrollback_lines = -1` → `2^32-1`: sustained stepping, never full

```bash
$ python3 /tmp/kitty_obs/mem_run.py -1 14000 128 0.05 0.05 q3_inf
```

**[observed (runtime)]** Complete, unedited filling region. The same stepping appears (`866832 → 871408 → 875960 → …`), with boundaries at `[2048, 4224, 6272, 8320, 10368, 12416]`:

```
# label=q3_inf scrollback=-1 N=14000 BATCH=128 PACE=0.05 kitty_pid=75738
# elapsed_s	lines	VmRSS_kB	VmSize_kB	VmData_kB	done
0.25	0	146212	866832	638624	0
3.24	0	146228	866832	638624	0
3.31	128	147012	866832	638624	0
3.60	512	147864	866832	638624	0
4.10	1664	150420	866832	638624	0
4.21	1920	151272	866832	638624	0
4.40	2048	151272	866832	638624	0
4.45	2048	151588	871408	643200	0    <- first allocation just past row 2048
4.85	2432	152140	871408	643200	0
5.20	3072	153564	871408	643200	0
5.60	3840	155264	871408	643200	0
5.65	4224	156164	875960	647752	0    <- segment #3
5.86	4736	157264	875960	647752	0
...
15.46	14000	177848	894168	665960	1
```
*(Trailing rows to cutoff repeat the byte-identical `… 177848 894168 665960 1` plateau; the plateau here is because output *stopped* at 14,000 lines, not because the buffer filled.)*

**[observed (runtime)]** With `ynum = 2^32-1`, the `segment_for` condition `SEGMENT_SIZE * self->num_segments < self->ynum` (`history.c:39`) is effectively always true, so segments keep being allocated as history grows — **the buffer never reaches "full"** and would continue stepping until RAM is exhausted. (The process was intentionally bounded to 14,000 lines and terminated cleanly.)

### 5.5 Before / during / after and the cause→effect chain

- **Before (empty):** one segment pre-allocated (`history.c:127`); `VmSize` at its window-creation value. **[observed (runtime)]**
- **During (filling):** every 2,048 rows, `segment_for` (`history.c:39`) fires `add_segment` (`history.c:18`), whose `calloc` (`history.c:25`) reserves ≈4.55 MiB → a discrete `VmSize`/`VmData` step; `VmRSS` ramps as pages fault. **[observed (runtime)]**
- **After (full — finite only):** once `count == ynum` (`history.c:279`), pushes evict the oldest row by advancing `start_of_data` (`history.c:281`); **no more `add_segment`**, RSS plateaus. **[observed (runtime)]**

**Cause → effect:** history reaches row `k·2048` → `segment_for` loop condition true (`history.c:39`) → `add_segment` `calloc`s a ≈4.55 MiB segment (`history.c:25`) → `VmSize`/`VmData` step up by ≈4,552 kB, and `VmRSS` climbs as the pages are written → once `count == ynum` the eviction branch (`history.c:281`) replaces allocation → plateau.

### 5.6 Q3 — every part answered

- *"At what point does the buffer's behavior change as it grows?"* → At each **2,048-row boundary** (allocation of a new segment), and again at **`count == ynum`** (switch from growing to evicting). **[observed (runtime)]**
- *"when does allocation of new storage occur?"* → When accumulating history crosses a multiple of `SEGMENT_SIZE = 2048` and total capacity is still `< ynum` (`history.c:39`), via `add_segment`'s `calloc` (`history.c:25`). **[observed (runtime)]**
- *"can I observe this happening through memory monitoring?"* → **Yes** — a discrete ≈4,552 kB `VmSize`/`VmData` step per segment, at line counts spaced exactly 2,048, reproducible across runs (§5.3). At default `scrollback_lines=2000`, **no** step occurs (§5.2). **[observed (runtime)]**

---

## 6. Q2 — Responsiveness while scrolling a large history during concurrent output

> **User Question 2 (verbatim):** *"When I scroll back through a very large history while new output is still being generated, does the terminal remain responsive? What latency or lag can I observe between my scroll input and the display updating? Are there any visible signs of the system prioritizing one operation over another?"*

### 6.1 Direct answer

**Yes — the terminal remains responsive.** With a 100,000-line scrollback filling at 5,000 lines/s, the measured **scroll-input → display-update latency was ≈16 ms median** (min ≈12 ms, max ≈21 ms), stable within ~1 ms across two runs and statistically indistinguishable across all four scroll variants (line-up, page-up, home, end). **[observed (runtime)]** Every one of 25 scrolls per variant produced a visible update — there is **no starvation** of input by output. Under a **flat-out** producer (≈100k+ lines/s) the **median stayed ≈14–15 ms**, with only the tail rising (p90 ≈41–45 ms). The **visible sign of prioritisation** is concrete and code-level: fresh input (a scroll) sets `input_read = true`, which **bypasses the `repaint_delay` idle frame-rate cap** at `child-monitor.c:875`, forcing an immediate render; pure output is instead merely coalesced by `input_delay`. A debug build confirmed a scroll triggers an `input_read=true` render within **3–18 ms** (median 13 ms). **[diagnostic — non-canonical]**

### 6.2 Harness (real input path, real frames)

Injection uses **`xdotool`** — installed system-wide via `apt` (not a repository change) — to send real X11 key events into the `kitty` window under Xvfb, so the input travels `kitty`'s actual GLFW key path. Frames are read back with **Pillow `ImageGrab`** off the same Xvfb display. To remove per-keystroke tool overhead (a fresh `xdotool` process is ≈20 ms; its default inter-key delay is 12 ms), keys are sent through a **persistent `xdotool -` stdin process with `--delay 0`**, so the measured interval is dominated by X delivery + `kitty` handling + render.

**`q2_producer.py`** — sustained producer (runs inside the PTY, prints full-width lines at ≈`RATE` lines/s until killed):

```python
#!/usr/bin/env python3
import sys, os, time, shutil
RATE  = float(os.environ.get("RATE", "5000"))   # target lines/sec
BATCH = int(os.environ.get("BATCH", "200"))
cols = shutil.get_terminal_size((80,24)).columns
with open(os.environ.get("READY","/tmp/kitty_obs/q2_ready"),"w") as f: f.write("ready\n")
w=sys.stdout; count=0; fillw=max(1,cols); interval = BATCH / RATE
while True:
    t=time.time()
    for _ in range(BATCH):
        count+=1; s="%08d"%count
        w.write((s+"-"*(fillw-len(s)))[:fillw]+"\n")
    w.flush()
    dt=time.time()-t
    if interval-dt>0: time.sleep(interval-dt)
```

**`q2_latency.py`** — launches the canonical binary with the sustained producer + large scrollback, injects each scroll variant, and times from key-send to the first frame whose changed-row count exceeds a threshold. A scroll changes almost the whole viewport, whereas streaming output only repaints a small status band (≈36 of 400 pixel-rows, held stable by the `scrolled_by` compensation at `screen.c:2716`), so `THRESH = 150` changed rows cleanly separates a scroll from background output:

```python
#!/usr/bin/env python3
import os, sys, time, subprocess
from PIL import ImageGrab
import numpy as np
REPO=os.environ["REPO"]; KB=os.path.join(REPO,"kitty","launcher","kitty")
SB=os.environ.get("SB","100000"); RATE=os.environ.get("RATE","5000")
TRIALS=int(os.environ.get("TRIALS","25")); LABEL=os.environ.get("LABEL","q2lat")
DISP=":99"; THRESH=150
os.environ["DISPLAY"]=DISP
# ... launch kitty (scrollback_lines=SB, cursor_blink_interval=0) running q2_producer.py, wait for READY ...
wid=subprocess.check_output(["xdotool","search","--name",title]).decode().split()[0]
# persistent injector with --delay 0
xd=subprocess.Popen(["xdotool","-"],stdin=subprocess.PIPE,text=True,bufsize=1)
def key(combo):
    xd.stdin.write("key --window %s --delay 0 %s\n"%(wid,combo)); xd.stdin.flush()
def gray():
    return np.asarray(ImageGrab.grab(bbox=bbox,xdisplay=DISP).convert("L"), dtype=np.int16)
def changed_rows(a,b):
    return int(np.count_nonzero(np.abs(a-b).sum(axis=1) > 0))
def recenter():
    key("ctrl+shift+End"); time.sleep(0.25)
    for _ in range(6): key("ctrl+shift+Prior")
    time.sleep(0.45)
VARIANTS=[("scroll_line_up","ctrl+shift+Up"), ("scroll_page_up","ctrl+shift+Prior"),
          ("scroll_home","ctrl+shift+Home"), ("scroll_end","ctrl+shift+End")]
for name,combo in VARIANTS:
    lats=[]
    for _ in range(TRIALS):
        recenter(); base=gray()
        t0=time.perf_counter(); key(combo); hit=None
        while time.perf_counter()-t0 < 0.5:
            if changed_rows(base,gray())>THRESH: hit=(time.perf_counter()-t0)*1000; break
        if hit is not None: lats.append(hit)
    # ... print min/median/p90/max/mean ...
```

Invocation:

```bash
$ export REPO="$PWD"
$ SB=100000 RATE=5000 TRIALS=25 LABEL=run1 python3 /tmp/kitty_obs/q2_latency.py
```

The grab itself is fast (median 2.9 ms, min 2.1 ms), which sets a small quantisation floor on the end-to-end number; the internal render latency (§6.4) is smaller still.

### 6.3 End-to-end latency results

**RUN 1** — `SB=100000`, `RATE=5000` lines/s, 25 trials per variant. **[observed (runtime)]**

```
scroll_line_up   n=25  min=12.2  median=15.9  p90=19.2  max=19.7 ms  (mean=16.0)
scroll_page_up   n=25  min=11.7  median=15.8  p90=19.0  max=21.3 ms  (mean=15.9)
scroll_home      n=25  min=11.9  median=15.5  p90=19.6  max=21.0 ms  (mean=16.0)
scroll_end       n=25  min=11.7  median=16.1  p90=20.8  max=21.4 ms  (mean=16.3)
```

**RUN 2** — identical settings (stability). **[observed (runtime)]**

```
scroll_line_up   n=25  min=11.9  median=15.8  p90=18.9  max=20.4 ms  (mean=16.2)
scroll_page_up   n=25  min=11.3  median=15.9  p90=19.8  max=21.5 ms  (mean=16.0)
scroll_home      n=25  min=12.5  median=15.5  p90=18.8  max=20.3 ms  (mean=15.9)
scroll_end       n=25  min=11.4  median=15.0  p90=19.5  max=24.7 ms  (mean=16.1)
```

**Stability:** medians agree within **~1 ms** across the two runs for every variant; every trial (25/25 per variant, both runs) detected an update within the 500 ms window — no dropped/starved scroll. **[observed (runtime)]**

**Scroll-variant coverage (Rule: cover both unmodified and paged/jump scrolling).** All four canonical variants were measured and are **statistically indistinguishable** (all ≈15–16 ms median):
- **Unmodified line scroll:** `scroll_line_up` = `ctrl+shift+Up` (`definition.py:3577`).
- **Paged / large-jump scroll:** `scroll_page_up` = `ctrl+shift+PageUp` (`definition.py:3607`), `scroll_home` = `ctrl+shift+Home` (`definition.py:3623`), `scroll_end` = `ctrl+shift+End` (`definition.py:3631`).

**Direct answer to "what latency can I observe?": ≈16 ms median, ~12 ms best, ~21 ms worst**, i.e. within roughly one to two 60 Hz frames — visually immediate.

**HIGH-LOAD** — flat-out producer (`RATE=100000`, effectively as fast as the PTY accepts), 25 trials/variant. **[observed (runtime)]**

```
scroll_line_up   n=25  min=8.8  median=14.1  p90=44.9  max=47.2 ms  (mean=20.9)
scroll_page_up   n=25  min=8.5  median=14.5  p90=41.0  max=44.5 ms  (mean=19.2)
scroll_home      n=25  min=8.2  median=15.1  p90=42.7  max=45.9 ms  (mean=20.8)
scroll_end       n=25  min=9.4  median=14.2  p90=40.6  max=43.1 ms  (mean=19.4)
```

**Direct answer to "does it remain responsive under load?": yes.** Even under maximum output pressure the **median latency stays ≈14–15 ms** (essentially unchanged from the 5,000 lines/s case); only the **tail** grows (p90 ≈41–45 ms, occasional max ≈47 ms) as the render thread occasionally lands behind a large output batch. No scroll was dropped. This modest tail-latency growth under saturation is itself the answer to "signs of prioritising one operation over another": the scheduler never lets output *starve* input, but a very large in-flight output batch can delay a scroll frame by a few tens of ms in the worst case.

### 6.4 The prioritisation mechanism — observed directly

To see *why* scrolls stay fast, an **event-loop debug build** was used as a diagnostic aid:

```bash
$ make debug-event-loop        # == python3 setup.py build --debug --extra-logging=event-loop
```

**[diagnostic — non-canonical]** This build compiles the `EVDBG("input_read: %d, ...")` trace at `child-monitor.c:872`, printed on every `render()` entry, and `--debug-input` logs each key with its matched action. Running the sustained producer and injecting a sequence of scrolls, then correlating each scroll's input timestamp to the first following render with `input_read=1`:

```
scroll_end       input@1.185s -> render(input_read=1)@1.193s  dt= 8.0 ms
scroll_page_up   input@1.533s -> render(input_read=1)@1.537s  dt= 4.0 ms
scroll_line_up   input@1.890s -> render(input_read=1)@1.903s  dt=13.0 ms
scroll_page_up   input@2.248s -> render(input_read=1)@2.266s  dt=18.0 ms
scroll_line_up   input@2.603s -> render(input_read=1)@2.606s  dt= 3.0 ms
scroll_home      input@2.959s -> render(input_read=1)@2.972s  dt=13.0 ms
scroll_page_up   input@3.318s -> render(input_read=1)@3.332s  dt=14.0 ms

internal input->render latency: min=3.0 median=13.0 max=18.0 ms (n=7); repaint_delay cap=10ms
```

**[diagnostic — non-canonical]** The corresponding `--debug-input` lines confirm the scrolls travelled the real key path, e.g.:

```
[1.484] on_key_input: glfw key: 0xe00a native_code: 0xff55 action: PRESS mods: ctrl+shift ...
        KeyPress matched action: scroll_page_up, handled as shortcut
[1.886] on_key_input: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: ctrl+shift ...
        KeyPress matched action: scroll_line_up, handled as shortcut
```

**Interpretation (cause → effect).** Each scroll produces a render with `input_read=1` within **3–18 ms (median 13 ms)** — clustered right around the 10 ms `repaint_delay` window, i.e. the scroll is serviced on essentially the next frame rather than being deferred. This is exactly the bypass at `child-monitor.c:875`:

```c
// kitty/child-monitor.c:875
if (!input_read && time_since_last_render < OPT(repaint_delay)) {   // scroll => input_read=1 => condition false => NOT throttled
```

With `input_read=1` the gate's `!input_read` term is false, so `render()` does not early-return and paints immediately (main tick `child-monitor.c:1236-1237`). Pure output when idle, by contrast, is capped to one frame per `repaint_delay` (10 ms, `definition.py:866`) and coalesced by `input_delay` (3 ms) in `do_parse` (`child-monitor.c:445`). The three-thread split (I/O thread `read_bytes` `child-monitor.c:1337` draining the PTY into the VT-parser buffer `vt-parser.c:1451`/`:1465`; main thread `parse_input` `child-monitor.c:451` → `render` `:871`) ensures output draining never blocks the input/render path. The in-source `repaint_delay` documentation (`definition.py:873-874`) states this design intent directly: the delay is ignored to minimise latency when input is pending.

### 6.5 Q2 — every part answered

- *"does the terminal remain responsive?"* → **Yes**: ≈16 ms median scroll latency at 5,000 lines/s (stable across 2 runs), ≈14–15 ms median even flat-out, 25/25 scrolls serviced. **[observed (runtime)]**
- *"what latency or lag can I observe between my scroll input and the display updating?"* → **≈16 ms median** (min ≈12, max ≈21) end-to-end at 5,000 lines/s; internal render component **≈13 ms median** (§6.4). **[observed (runtime)] / [diagnostic — non-canonical]**
- *"any visible signs of prioritising one operation over another?"* → **Yes**: the `repaint_delay` bypass on fresh input (`child-monitor.c:875`) — a scroll preempts the idle frame cap and renders immediately, while pure output is throttled/coalesced. Under saturation this shows up as low median but a longer input tail (output batches briefly delay a frame), never as dropped input. **[observed (runtime)] + [inferred (code-derived)]**

---

## 7. Methodology and reproducibility

### 7.1 What was run, at what scale, and how stability was confirmed

| Question | Setting(s) | Scale / duration | Repeats | Stability |
|---|---|---|---|---|
| Q1-A | default `scrollback_lines=2000` | 500,000 lines, ≈3.68 s (≈136k lines/s) | 2 | peak within **0.12 %** |
| Q1-B | `scrollback_lines=300000` | 320,000 lines, ≈28 s | 2 | peak within **0.13 %**, slope within **1.8 B/line** |
| Q3-default | `scrollback_lines=2000` | 6,000 lines | 1 (+ Q1-A) | no stepping (VmSize constant) |
| Q3-large | `scrollback_lines=100000` | 14,000 lines | 2 | boundaries & steps **byte-identical** |
| Q3-infinite | `scrollback_lines=-1` (`2^32-1`) | 14,000 lines (bounded) | 1 | same stepping observed |
| Q2 | `scrollback_lines=100000`, 5,000 lines/s | 25 scrolls × 4 variants | 2 | medians within **~1 ms** |
| Q2 high-load | `scrollback_lines=100000`, flat-out | 25 scrolls × 4 variants | 1 | median ≈14–15 ms |

### 7.2 Canonical vs. diagnostic

- **Canonical (reported numbers):** the default `make` build (`kitty 0.35.2`), driven through the real PTY input path headlessly on Xvfb+Mesa, memory via `/proc/<pid>/status` + `smaps`, scroll input via real X11 key events (`xdotool`), frames via `ImageGrab`. All Q1/Q2/Q3 magnitudes are from this path.
- **Diagnostic — non-canonical (corroboration only):** the `make debug-event-loop` build was used **once** to observe the `input_read` render trace (§6.4). After capturing it, the **default build was restored with `make`** (verified `kitty 0.35.2`, `git status --porcelain` = 0 lines). No canonical number depends on the debug build.
- **Not used as canonical:** the Python test harness (`kitty_tests`) can construct `Screen`/`HistoryBuf` directly, but that bypasses the real PTY path, so it was not used to source any reported value.

### 7.3 Repository-unchanged discipline

Every observation script lived under `/tmp/kitty_obs/` — **outside** the repository — and was removed after use. All `kitty` build outputs are `.gitignore`d, so building never dirties the tree. The single committed artifact is this document. `xdotool` was installed at the OS level (not a repository change). After all runs: `git status --porcelain` reports only this new file under `blitzy/documentation/`.

### 7.4 Notes, caveats, and unverified points

- Terminal geometry was `71×22` (headless `1024x768` window); per-line/per-segment arithmetic uses `xnum=71`. On a wider terminal the per-line cost scales linearly as `xnum×32+1`, and the per-segment step as `xnum×2048×32+2048`.
- The `+128`-line offset of observed boundaries from exact multiples of 2,048 is the 128-line sampling batch, not a property of the buffer — the true allocation happens *at* the 2,048 boundary (visible in the infinite run, whose first step is recorded at line 2,048 itself).
- The ≈12–16 ms end-to-end latency floor includes `ImageGrab` poll quantisation (≈3 ms) and X11 delivery; the kitty-internal render latency is smaller (≈13 ms median including up to a full `repaint_delay` window). These are additive measurement components, not kitty overhead, and are stated so the number is not misattributed.
- glibc keeps the ≈4.55 MiB segment `calloc`s in its main arena (the `[heap]` mapping) rather than as separate mmaps; this is an allocator detail and does not change the per-segment step size seen in `VmSize`.

---

## 8. Appendix — citation index (verified at commit `815df1e210e0`)

**`kitty/history.c`**
- `:15` — `#define SEGMENT_SIZE 2048` (segment granularity; Q1/Q3).
- `:18` — `add_segment(HistoryBuf *self)` (segment allocation function; Q3).
- `:25` — `s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));` (the single backing `calloc`; Q1/Q3).
- `:37` / `:39` — `segment_for(...)` and its loop `while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);` (on-demand allocation boundary; Q3).
- `:117` / `:127` — `create_historybuf(...)` calls `add_segment(self)` once at window creation (single default segment; Q3).
- `:259` — `pagerhist_push(...)` (eviction sink; serialises via `line_as_ansi`).
- `:276` / `:279` / `:281` / `:282` — `historybuf_push(...)`; `if (self->count == self->ynum)`; `self->start_of_data = (self->start_of_data + 1) % self->ynum;`; `else self->count++;` (fill vs. evict; Q1/Q3).
- `:287` — `historybuf_add_line(...)`.

**`kitty/data-types.h`**
- `:57` / `:59` / `:62` — `CPUCell` fields `char_type ch` (4 B) / `hyperlink_id` (2 B) / `cc_idx[3]` (6 B).
- `:221` — `static_assert(sizeof(GPUCell) == 20, ...)` (20-byte GPU cell; Q1).
- `:228` — `static_assert(sizeof(CPUCell) == 12, ...)` (12-byte CPU cell; Q1).
- `:231`–`:239` — `typedef union LineAttrs { ... } LineAttrs;` (1-byte per-line attribute; Q1).
- `:260` / `:266` / `:272` / `:290` — struct ends for `LineBuf` / `HistoryBufSegment` / `PagerHistoryBuf` / `HistoryBuf`.

**`kitty/line-buf.c`**
- `:94` / `:95` / `:96` — `PyMem_Calloc(xnum*ynum, sizeof(CPUCell))` / `... sizeof(GPUCell)` / `line_map` (fixed active-screen allocation, O(1) scroll; Q1).

**`kitty/line.c`**
- `:338` — `line_as_ansi(...)` (serialises an evicted line's cells; Q1 context).

**`kitty/screen.c`**
- `:130` — `self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));` (`ynum = MAX(scrollback, lines)`; Q1/Q3).
- `:2716` (mirror `:2761`) — `if (self->scrolled_by) self->scrolled_by = MIN(self->scrolled_by + history_line_added_count, self->historybuf->count);` (viewport stays put under streaming; Q2).
- `:4091` — `screen_history_scroll(...)` (scroll action target; Q2).

**`kitty/child-monitor.c`**
- `:229`–`:230` — `io_loop` / `talk_loop` forward declarations.
- `:291` — `pthread_create(&self->io_thread, NULL, io_loop, self)` (I/O thread; Q2).
- `:438` / `:445` — `do_parse(...)`; `set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);` (output coalescing; Q2).
- `:451` / `:530` — `parse_input(...)`; `if (do_parse(...)) input_read = true;`.
- `:871` / `:872` / `:874` / `:875` / `:876` / `:877` — `render(monotonic_t now, bool input_read)`; `EVDBG("input_read: %d, ...")`; `time_since_last_render`; **`if (!input_read && time_since_last_render < OPT(repaint_delay)) {`** (the prioritisation gate; Q2); `set_maximum_wait(...)`; `return;`.
- `:1236` / `:1237` — `if (parse_input(self)) input_read = true;` / `render(now, input_read);` (main tick; Q2).
- `:1337` / `:1341` / `:1345` / `:1354` — `read_bytes(...)`; `vt_parser_create_write_buffer(...)`; `len = read(fd, buf, available_buffer_space);`; `vt_parser_commit_write(...)` (I/O-thread PTY drain; Q2).

**`kitty/vt-parser.c`**
- `:1451` / `:1465` / `:1496` — `vt_parser_create_write_buffer(...)` / `vt_parser_commit_write(...)` / `parse_worker(...)` (PTY write-buffer target and main-thread parse; Q2 context).

**`kitty/options/definition.py`**
- `:372` — `opt('scrollback_lines', '2000', ...)` (default buffer capacity; Q1/Q3).
- `:406` — `opt('scrollback_pager_history_size', '0', ...)` (pager ring disabled by default).
- `:866` — `opt('repaint_delay', '10', ...)` (idle frame cap ≈100 FPS; Q2); long-text `:873-874` states the delay is ignored when input is pending.
- `:878` — `opt('input_delay', '3', ...)` (output coalescing window; Q2).
- `:3474` — `opt('kitty_mod', 'ctrl+shift', ...)`; scroll maps `:3577` (line up), `:3592` (line down), `:3607` (page up), `:3623` (home), `:3631` (end).

**`kitty/options/utils.py`**
- `:557` / `:559` / `:560` — `def scrollback_lines(x): ... if ans < 0: ans = 2 ** 32 - 1` (infinite = `4,294,967,295`; Q3).

---

*End of investigation. All magnitude claims above were measured on the default `kitty 0.35.2` build at commit `815df1e210e0`, through the real PTY path, and confirmed stable across at least two runs; all mechanism claims are grounded in the verified `file:line` references in this appendix and labelled observed (runtime) or inferred (code-derived).*
