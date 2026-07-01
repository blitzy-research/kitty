# kitty: Moving data between the C core and the Python layer under concurrent load

A runtime-evidenced investigation of how the **kitty** terminal emulator moves data
between its performance-critical **C core** and its **Python layer** (the in-process
controller and the out-of-process "kittens"), using the **clipboard** as the worked
example, and analysing how an expensive operation such as **scanning a large
scrollback** affects event delivery and memory management — pinpointing where
**timing, concurrency, and object ownership** begin to matter and how subtle
**races** surface only at runtime.

---

## 1. Title & scope

### 1.1 What this document answers

This document answers six sub-questions, each in its own named section:

- **Q1** — How does kitty move data between its C core and the Python "kittens" when a lot is happening at once?
- **Q2** — "Clipboard data can be small or very large — how does it cross from internal screen structures into Python objects?"
- **Q3** — What does that transfer look like in practice, especially when other parts of the system are busy simultaneously?
- **Q4** — "If the terminal is doing something expensive like scanning a large scrollback, does that affect how events are delivered to kittens or how memory is managed?"
- **Q5** — Where do timing, concurrency, and object ownership start to matter?
- **Q6** — How might subtle races emerge only under real runtime conditions?

### 1.2 Investigation premise (read-only + run-the-code-first)

The source tree is treated as **read-only**: no existing file in the repository was
modified, added, or deleted. The only artifact produced is this Markdown document.

Every behavioural, magnitude, or timing claim below is backed by **verbatim output**
captured by running the *actual* compiled code paths. The observation scripts were
written under `/tmp/kitty_obs/` (outside the repository tree) and removed after use;
the repository was verified byte-for-byte unchanged (`git status --porcelain` empty).

### 1.3 Why the file is named `kitty_815df1e210e0.md`

The deliverable is named for the source branch/commit. The working copy is checked
out on an operational branch, but the resolved HEAD commit is the source revision
`815df1e210e0…`, which is the origin of the filename:

```
$ git rev-parse --abbrev-ref HEAD
blitzy-67dd7696-b709-4129-afd9-2c6c534a5de4
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

All `file:line` citations in this document refer to the source at commit
`815df1e210e0…`. Line numbers were re-verified against the on-disk source before
being quoted; where the pre-scoped line differed from the observed line, the
**observed** line is cited.

---

## 2. Environment & build evidence

### 2.1 Toolchain

kitty is a three-language system: a **C11** core (compiled into the
`kitty.fast_data_types` extension module), a **Python** controller/config/kittens
layer, and **Go** CLI tooling. Building requires a C compiler and the Go compiler —
<code>docs/build.rst:15</code> states the requirement is "a C compiler and the `go compiler`"; `setup.py:492`
selects the C standard (`std = '' if is_openbsd else '-std=c11'`); and `go.mod:3`
pins `go 1.22`. The C extension was already compiled in the container
(`kitty/fast_data_types.so` present).

### 2.2 The extension imports, and the parser constants are what the C core defines

Running an in-process script against the built extension confirms the interpreter
version and the two parser constants that govern the whole clipboard story
(`/tmp/kitty_obs/obs1_constants.py`):

```
PYVER: 3.11.15
VT_PARSER_BUFFER_SIZE: 1048576
VT_PARSER_MAX_ESCAPE_CODE_SIZE: 262144
BUF_SZ_is_1MiB: True
MAX_ESC_is_256KiB: True
MAX_ESC_eq_BUF/4: True
clipboard_max_size: 512.0
clipboard_max_size_type: float
clipboard_max_size_bytes: 536870912
get_options_matches: True
```

- The interpreter is **Python 3.11.15**. CI exercises Python 3.10 and 3.11
  (`.github/workflows/ci.yml:30`, `.github/workflows/ci.yml:85`).
- `VT_PARSER_BUFFER_SIZE` is **1048576** = 1 MiB, the parser's double buffer
  `#define BUF_SZ (1024u*1024u)` at `kitty/vt-parser.c:18` (exported at
  `kitty/vt-parser.c:1589`).
- `VT_PARSER_MAX_ESCAPE_CODE_SIZE` is **262144** = 256 KiB, the per-escape cap
  `#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)` at `kitty/vt-parser.c:21` (exported at
  `kitty/vt-parser.c:1590`); the runtime check `MAX_ESC_eq_BUF/4: True` confirms the
  `BUF_SZ/4` relationship.
- `clipboard_max_size` is the float **512.0** (MiB) — i.e. **536870912** bytes. This
  is the value defined at `kitty/options/types.py:498`
  (`clipboard_max_size: float = 512.0`). This directly **refutes** the stale external
  "8 MB" clipboard-limit figure (see §11).

Using the *observed* integers (1048576, 262144) rather than asserting "1 MiB"/"256 KiB"
from the `#define`s alone is deliberate: it is the "run the code first" rule in action.

---

## 3. Threading model overview

kitty's child monitor runs **three** OS threads in addition to the main thread. The
main thread holds the **GIL** and is the *only* thread that calls into Python; the
other three are pure C. The thread names are set at runtime via `set_thread_name`
(`kitty/threading.h:26`).

### 3.1 The three named threads observed live

Instantiating a `ChildMonitor`, calling `start()`, and triggering a threaded stdin
write, then reading `/proc/self/task/*/comm`, shows all three named threads plus the
unnamed main thread (`/tmp/kitty_obs/obs6_threads.py`):

```
MAIN_TID: 58452
MAIN_THREAD_COMM: python
NUM_THREADS: 4
  tid=58452 comm=python (main/GIL-holder)
  tid=58454 comm=KittyPeerMon
  tid=58455 comm=KittyChildMon
  tid=58456 comm=KittyWriteStdin
KITTY_THREAD_NAMES: ['KittyChildMon', 'KittyPeerMon', 'KittyWriteStdin']
```

- **`KittyChildMon`** — the I/O thread (`io_loop`), named at `kitty/child-monitor.c:1489`.
  It reads PTY bytes and is **pure C**: it never touches Python objects. It is spawned by
  `start()` via `pthread_create(&self->io_thread, NULL, io_loop, self)` at
  `kitty/child-monitor.c:291`.
- **`KittyPeerMon`** — the talk thread (`talk_loop`) for the remote-control socket,
  named at `kitty/child-monitor.c:1808`. Spawned only when a talk/listen fd is present
  (`kitty/child-monitor.c:285`).
- **`KittyWriteStdin`** — the on-demand thread that writes bulk data to a child's stdin
  (`thread_write`), named at `kitty/child-monitor.c:967`.
- **The main thread** (tid == pid, `comm=python`) is *unnamed* by kitty and is the
  **GIL holder** — the only thread permitted to execute Python or invoke the C-API.

### 3.2 Two structurally different C↔Python boundaries

1. **In-process (C-API)** — the C core calls *into* embedded CPython on the main
   thread while holding the GIL, via the `CALLBACK` macro
   (`kitty/screen.c:87`). This is how clipboard bytes reach `kitty/window.py` /
   `kitty/clipboard.py`. Data crosses as a **zero-copy read-only `memoryview`**
   (see §5).
2. **Out-of-process (serialized over the escape protocol)** — kittens are *separate
   processes*. Their results are serialized as **JSON then base85** and written back
   over the protocol: `import json` at `kittens/runner.py:101` and
   `data = base64.b85encode(json.dumps(result).encode('utf-8'))` at
   `kittens/runner.py:102`. There is no shared address space here; nothing crosses by
   pointer.

These two boundaries have very different ownership and timing properties, which is
why Q1/Q5/Q6 must treat them separately.

---

## 4. Q1 — Cross-language data movement under load

> **How does kitty move data between its C core and the Python "kittens" when a lot is happening at once?**

Data moves across **two** boundaries with fundamentally different mechanics.

### 4.1 In-process: the C core calls Python on the main thread, under the GIL

When PTY output arrives, the pure-C I/O thread `KittyChildMon` reads it into the
parser's write buffer (`read_bytes` at `kitty/child-monitor.c:1337`) but **never
touches Python**. The bytes are turned into Python calls only later, on the **main
thread**, which holds the GIL. That the I/O thread is a distinct, pure-C thread while
the GIL-holder is a separate unnamed thread is shown directly by the live thread list:

```
  tid=58452 comm=python (main/GIL-holder)
  tid=58455 comm=KittyChildMon
```
(`/tmp/kitty_obs/obs6_threads.py`)

The actual C→Python call is performed by the `CALLBACK` macro at `kitty/screen.c:87`,
which invokes `PyObject_CallMethod(self->callbacks, ...)` at `kitty/screen.c:89`. The
crux of ownership on the C side is the immediately following line — the C core owns
and releases the returned `PyObject*`:

```c
if (callback_ret == NULL) PyErr_Print(); else Py_DECREF(callback_ret);
```
(`kitty/screen.c:90`)

"When a lot is happening at once," the design keeps the two threads decoupled: the
I/O thread fills the buffer while the main thread drains it. A single large clipboard
write, for instance, is delivered to Python as a **sequence** of C→Python calls
rather than one — 27 dispatches for a 20 MiB payload (§5.2), each a separate
`CALLBACK` on the main thread.

### 4.2 Out-of-process: kittens receive serialized bytes, not pointers

A "kitten" is a *separate process*, so nothing crosses by shared memory. Its result
is serialized to JSON and then base85-encoded before being written back over the
protocol:

```python
import json
data = base64.b85encode(json.dumps(result).encode('utf-8'))
```
(`kittens/runner.py:101` and `kittens/runner.py:102`)

So "moving data to the kittens" is: (a) in-process, the C core hands a `memoryview`
to the Python controller under the GIL; and (b) out-of-process, the controller
serializes and streams bytes to/from the kitten subprocess. The clipboard (Q2) is the
in-process case; the JSON+base85 path is the out-of-process case.

---

## 5. Q2 — Clipboard crossing (small vs. large)

> **"Clipboard data can be small or very large — how does it cross from internal screen structures into Python objects?"**

The clipboard uses the OSC 52 (and the MIME-aware OSC 5522) escape codes
(`docs/clipboard.rst:5`, `docs/clipboard.rst:12`). The path from parser bytes to a
Python object is:

1. The VT parser routes OSC 52/5522: `case 52: case 5522:` at `kitty/vt-parser.c:531`,
   dispatched via `DISPATCH_OSC_WITH_CODE(clipboard_control)` at `kitty/vt-parser.c:534`.
2. The payload is wrapped **without copying** in a **read-only `memoryview`** over the
   parser's own C buffer: `PyMemoryView_FromMemory((char*)buf + i, limit - i, PyBUF_READ)`
   at `kitty/vt-parser.c:461`.
3. The C entry `clipboard_control` calls back into Python
   (`kitty/screen.c:2305`), passing that `memoryview` and an `is_partial` flag:
   `if (code == 52 || code == -52) { CALLBACK("clipboard_control", "OO", data, code == -52 ? Py_True: Py_False); }`
   at `kitty/screen.c:2306` (OSC 5522 passes `Py_None` instead).
4. `Window.clipboard_control(self, data: memoryview, is_partial)` at
   `kitty/window.py:1391` routes `is_partial is None` → `parse_osc_5522`, else →
   `parse_osc_52`.
5. `ClipboardRequestManager.parse_osc_52` (`kitty/clipboard.py:406`) accumulates the
   base64 into a `WriteRequest` whose backing store (`Tempfile`,
   `kitty/clipboard.py:26`) starts in RAM and rolls over to disk.

The small and large cases differ in **how many dispatches** occur and **where the data
is stored**.

### 5.1 Small clipboard write — a single, non-partial dispatch that stays in RAM

Feeding the 13-byte sequence `\x1b]52;c;aGk=\x1b\\` (payload `hi`) yields exactly one
dispatch (`/tmp/kitty_obs/obs2_small.py`):

```
RAW_PAYLOAD: b'hi' len= 2
BASE64: aGk=
OSC_BYTES_LEN: 13
NUM_DISPATCHES: 1
DISPATCH n=1 type=memoryview len=6 is_partial=False readonly=True
FINAL_WRITE_STATE: ('final', 'BytesIO', 2, False)
```

- The data crosses as a **`memoryview`** (`type=memoryview`), not a copy, of length 6
  (the bytes `c;aGk=` after `52;`), and it is `readonly=True` — ties to
  `PyBUF_READ` at `kitty/vt-parser.c:461`.
- There is exactly **one** dispatch and it is **non-partial** (`is_partial=False`).
- The manager consumes it immediately; the `WriteRequest.tempfile.file` is still an
  in-memory `io.BytesIO` holding the 2 decoded bytes (`'BytesIO', 2`). Small payloads
  never leave RAM. The in-memory start is `self.file … = io.BytesIO()` at
  `kitty/clipboard.py:29`.

### 5.2 Large clipboard write — repeated partial dispatches + rollover to disk

Feeding a **20 MiB** payload (base64 ≈ 27.96 MiB) produces a *sequence* of dispatches
and forces the backing store from RAM onto disk (`/tmp/kitty_obs/obs3_large.py`):

```
DECODED_TARGET_BYTES: 20971520
BASE64_LEN: 27962028
OSC_TOTAL_BYTES: 27962037
MAX_ESCAPE_CODE_SIZE: 262144
PARSE_WALL_SECONDS: 0.055
NUM_DISPATCHES: 27
NUM_PARTIAL(is_partial=True): 26
NUM_FINAL(is_partial=False): 1
FIRST_CHUNK_LEN: 1048570
SECOND_CHUNK_LEN: 1048572
LAST_CHUNK_LEN: 699186
MAX_CHUNK_LEN: 1048572
TEMPFILE_FIRST_TYPE: BytesIO
ROLLOVER_AT_DISPATCH: 22
ROLLOVER_SIZE_BYTES_AT_FLIP: 17301417
FINAL_WRITE_STATE: ('final', 'BufferedRandom', 20971520, False)
clipboard_max_size_MiB: 512.0
```

**(a) It arrives as many partial dispatches, then one final.** The large write
crosses as **27** separate `clipboard_control` calls — **26 partial** (`is_partial=True`)
followed by **1 final** (`is_partial=False`). The `is_partial` flag transitions
`True … True → False` on the last chunk, which is exactly the condition
`parse_osc_52` uses to keep accumulating vs. finish: `if is_partial:` → `return` at
`kitty/clipboard.py:422` (accumulate) versus `self.handle_write_request(wr)` at
`kitty/clipboard.py:425` on the final chunk.

**(b) The chunk size is ~1 MiB, bounded by `BUF_SZ` — not 256 KiB.** Observed chunks
are `FIRST_CHUNK_LEN: 1048570` and `MAX_CHUNK_LEN: 1048572`, i.e. essentially
`BUF_SZ` = 1048576 (§2.2), **not** `MAX_ESCAPE_CODE_SIZE` = 262144. The reason is in
`accumulate_st_terminated_esc_code` at `kitty/vt-parser.c:395`: if the ST terminator
is present it dispatches the *whole* code even when it exceeds the cap (the source
comment is explicit — "lets be generous … we have a full escape code"); only when
**no** terminator is found *and* the accumulated length exceeds `MAX_ESCAPE_CODE_LENGTH`
does it emit a **partial** — the check `(pos = self->read.pos - self->read.consumed) > MAX_ESCAPE_CODE_LENGTH`
is at `kitty/vt-parser.c:406` and the partial call
`dispatch(..., /*is_partial=*/true)` is at `kitty/vt-parser.c:413`, and it flushes
*all* the accumulated unterminated bytes — which, since the parser buffer is `BUF_SZ`,
is up to ~1 MiB. So **262144 is the partial-dispatch *trigger threshold*, while ~1 MiB
(`BUF_SZ`) is the effective chunk *cap*.** After each partial it calls `continue_osc_52`
(`kitty/vt-parser.c:416`) and recurses (`kitty/vt-parser.c:417`). (This refines the naive "≤256 KiB partials"
reading; see §11.)

**(c) The Python accumulator spills from RAM to disk at 16 MiB.** The backing store
begins as `BytesIO` (`TEMPFILE_FIRST_TYPE: BytesIO`) and rolls over to an on-disk
temp file at dispatch 22, once the accumulated size crosses the rollover threshold
(`ROLLOVER_AT_DISPATCH: 22`, `ROLLOVER_SIZE_BYTES_AT_FLIP: 17301417` ≈ 16.5 MiB). The
threshold is `rollover_size: int = 16 * 1024 * 1024` at `kitty/clipboard.py:237`,
passed to `Tempfile(max_size=rollover_size)` at `kitty/clipboard.py:243`; the rollover
itself is `if isinstance(self.file, io.BytesIO) and self.file.tell() + sz > self.max_size:`
at `kitty/clipboard.py:33`, replacing the buffer with `self.file = TemporaryFile()` at
`kitty/clipboard.py:35`. The final store type is `BufferedRandom` (the Python type of
a POSIX `tempfile.TemporaryFile()`), holding exactly **20971520** bytes = the full
20 MiB, with `max_size_exceeded=False`.

**(d) The cap is 512.0 MiB, not 8 MB.** The run reports `clipboard_max_size_MiB: 512.0`
(`kitty/options/types.py:498`); the 20 MiB payload is far below it, so nothing is
truncated. Truncation would set the `max_size_exceeded` flag in `write_base64_data`
(`kitty/clipboard.py:316`), which sets `self.max_size_exceeded = True` at
`kitty/clipboard.py:323`.

**Summary of the crossing.** Internal screen/parser bytes become a Python object as a
*zero-copy read-only `memoryview`* over the C parser buffer. Small payloads cross once
and live in a `BytesIO`; large payloads cross as many ~1 MiB partial `memoryview`s that
the Python `WriteRequest` accumulates, transparently spilling to an on-disk temp file
past 16 MiB and capped at 512 MiB.

---

## 6. Q3 — The transfer in practice under contention

> **What does that transfer look like in practice, especially when other parts of the system are busy simultaneously?**

Two independent mechanisms govern contention: a **parser `pthread` mutex** that
serializes the I/O thread and the main thread over the shared buffer, and the **GIL**
that serializes all Python execution onto the main thread.

### 6.1 The double buffer and the parser mutex

The parser owns a 1 MiB buffer (`BUF_SZ` at `kitty/vt-parser.c:18`, observed
`VT_PARSER_BUFFER_SIZE: 1048576` in §2.2) guarded by a `pthread_mutex_t` declared at
`kitty/vt-parser.c:206`. The I/O thread writes into it; the main thread reads/parses
from it. The mutex is taken and released via the `with_lock` / `end_with_lock` macros
(`kitty/vt-parser.c:1413`-`1414`).

### 6.2 The main thread releases the parser lock *while* calling Python

The pivotal contention behaviour is in `run_worker` (`kitty/vt-parser.c:1417`): the
main thread deliberately **drops the parser lock while dispatching to Python**, so the
I/O thread can keep filling the buffer during the (potentially slow) Python callbacks:

```c
end_with_lock; {
    consume_input(self, pd->dump_callback, screen->window_id);
} with_lock;
```
(`kitty/vt-parser.c:1431`-`1433`)

`consume_input` is where the OSC dispatch and the C→Python `CALLBACK`s happen. Because
the lock is *not* held across it, "other parts of the system being busy" (the I/O
thread reading a flood of PTY data) does not block the main thread's Python dispatch,
and vice versa. That the transfer really is chunked across many such dispatches under
a large load is exactly what §5.2 shows: `NUM_DISPATCHES: 27` for one large paste.

### 6.3 The `input_delay` batching throttle

To avoid waking the main thread for every tiny read, dispatch is throttled by
`input_delay`. The parse is performed only when a flush is forced, enough time has
elapsed, or the buffer is nearly full:

```c
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
```
(`kitty/vt-parser.c:1425`)

So under light load the main thread batches input over an `input_delay` window; under
heavy load the third clause (`self->read.sz + 16 * 1024 > BUF_SZ`) forces a parse as
the 1 MiB buffer approaches full, independent of the timer.

### 6.4 PTY backpressure when the buffer fills

If the main thread cannot keep up and the 1 MiB buffer fills, the I/O thread stops
asking the kernel for more PTY data by clearing `POLLIN`:

```c
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```
(`kitty/child-monitor.c:1501`)

In practice, then, the transfer is a producer/consumer pipeline: the I/O thread
produces into a bounded 1 MiB buffer under the parser mutex; the main thread consumes,
releasing the mutex while it calls Python; `input_delay` batches the hand-offs; and
when the consumer falls behind, `POLLIN` backpressure throttles the producer at the
kernel boundary.


---

## 7. Q4 — Impact of expensive work (scanning a large scrollback)

> **"If the terminal is doing something expensive like scanning a large scrollback, does that affect how events are delivered to kittens or how memory is managed?"**

Yes — on **both** counts. A scrollback scan runs **synchronously on the GIL-holding
main thread** and the scan code **never releases the GIL**, so it both blocks event
delivery and, because the scrollback itself is large, dominates memory.

### 7.1 Memory: the scrollback scales linearly with line count

The history buffer is allocated in fixed segments of 2048 lines (`SEGMENT_SIZE 2048`
at `kitty/history.c:15`). Each segment allocates per-column `CPUCell` + `GPUCell`
arrays plus per-line `LineAttrs`:

```c
const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));
```
(`kitty/history.c:23`-`25`)

Per cell that is `sizeof(GPUCell) + sizeof(CPUCell)` = 20 + 12 = **32 bytes**, fixed by
static assertions `static_assert(sizeof(GPUCell) == 20, …)` at `kitty/data-types.h:221`
and `static_assert(sizeof(CPUCell) == 12, …)` at `kitty/data-types.h:228`. So an
80-column scrollback should cost ≈ 80 × 32 = 2560 bytes/line. Measuring resident
memory (`VmRSS`) while populating a scrollback at two sizes confirms linear scaling
(`/tmp/kitty_obs/obs5_scrollback.py`):

```
COLS: 80 SCROLLBACK: 4000000
RSS0_kb: 25596 RSS_500k_kb: 1282896 RSS_1M_kb: 2535828
RSS_delta_500k_bytes: 1287475200
RSS_delta_1M_bytes: 2570477568
bytes_per_line_500k: 2575.0
bytes_per_line_1M: 2570.5
incremental_bytes_per_line_2nd_500k: 2566.0
theoretical_cell_bytes_per_line(80*(20+12)): 2560
```

At 500k lines the delta is **1,287,475,200** bytes (**2575.0** bytes/line); at 1M lines
it is **2,570,477,568** bytes (**2570.5** bytes/line); the incremental cost of the
second 500k lines is **2566.0** bytes/line. The per-line cost is essentially constant
(≈ 2570, i.e. the theoretical 2560 plus ~10 bytes for amortized `LineAttrs` and
rounding), which is exactly the **linear** scaling implied by `kitty/history.c:23`-`25`.
So "managing memory" for a large scrollback means holding gigabytes: **1M lines ≈ 2.57 GB**
resident.

### 7.2 The scan is synchronous on the main thread and holds the GIL

`kitty/screen.c` contains **no** `Py_BEGIN_ALLOW_THREADS` at all — confirmed by
`grep -rln Py_BEGIN_ALLOW_THREADS kitty/*.c`, which returns only `kitty/utmp.c`. So
every screen/scrollback scan runs with the GIL held. Timing the canonical text
extraction over 1,000,000 history lines
(`kitty.window.as_text(screen, add_history=True)`, which calls
`screen.as_text_non_visual` at `kitty/screen.c:3491` and
`screen.as_text_for_history_buf` at `kitty/screen.c:3495`):

```
SCAN_wall_seconds: 0.462
SCAN_total_logical_lines: 1000001
SCAN_total_chars: 51000000
```

The scan of 1M lines takes **0.462 s** of wall-clock on the calling (main) thread,
producing 1,000,001 logical lines / 51,000,000 characters. For that entire 0.462 s the
main thread is inside the C scan and is doing nothing else.

### 7.3 Event delivery is blocked for the full duration of the expensive work

To make the "affects how events are delivered to kittens" claim concrete, a background
Python thread (standing in for event delivery) timestamps itself as fast as it can
while the main thread performs a **pure-C history rewrap** — a 1-column resize, which
triggers `realloc_hb` → `historybuf_rewrap` (`kitty/screen.c:217`-`221`) with no
per-line Python callback:

```
REWRAP_wall_seconds: 1.286
WORKER_samples: 1033629
WORKER_median_gap_seconds: 0.000000
WORKER_max_gap_seconds: 1.281
max_gap_over_rewrap_ratio: 0.996
```

The rewrap takes **1.286 s**. During normal operation the worker records timestamps
essentially continuously (`WORKER_median_gap_seconds: 0.000000`), but its **largest
gap was 1.281 s** — **99.6 %** of the rewrap duration. In other words, the background
thread could not run *at all* for essentially the whole expensive operation, because
the GIL-holding C scan never yielded. This is the direct, measured link between
"doing something expensive" and "how events are delivered to kittens": while the main
thread is in a scrollback scan/rewrap, it is neither running its poll/dispatch loop nor
releasing the GIL, so no event can be delivered until the scan returns. If PTY output
keeps arriving meanwhile, the 1 MiB parser buffer fills and `POLLIN` backpressure
(`kitty/child-monitor.c:1501`, §6.4) throttles the child.

---

## 8. Q5 — Where timing, concurrency, and object ownership start to matter

> **Where do timing, concurrency, and object ownership start to matter?**

### 8.1 Object ownership — the zero-copy `memoryview` is read-only and has a bounded lifetime

The clipboard `memoryview` is a *window* over the parser's C buffer
(`PyMemoryView_FromMemory(..., PyBUF_READ)` at `kitty/vt-parser.c:461`), not a copy.
Ownership therefore matters the instant Python receives it. It is **read-only**, and
attempting to mutate it raises immediately (`/tmp/kitty_obs/obs4_memoryview.py`):

```
readonly: True
write_error: 'cannot modify read-only memory'
```

And its **lifetime is bounded by the dispatch**: if Python retains the `memoryview`
past the call and the parser reuses the buffer, the retained view silently shows the
*new* bytes. The same retained view returned the first payload immediately after the
dispatch, then a *different* payload after more parsing:

```
retained_content_right_after: b'c;QUFBQUFBQUE='
retained_content_later: b'c;WlpaWlpaWlo='
retained_changed_after_more_parsing: True
```

The correct, ownership-safe pattern is to **copy during the call** (`data.tobytes()` /
decode), which yields an independent object:

```
copied_tobytes: b'c;QUFBQUFBQUE='
copy_still_correct: True
```

This is why `parse_osc_52` decodes/accumulates *synchronously* inside the dispatch
(`kitty/clipboard.py:406`) rather than stashing the view for later.

### 8.2 Concurrency — two serializers: the parser mutex and the GIL

Concurrency matters at two boundaries. (a) The parser `pthread_mutex_t`
(`kitty/vt-parser.c:206`) serializes the pure-C I/O thread and the main thread over the
shared 1 MiB buffer; the main thread releases it while calling Python
(`kitty/vt-parser.c:1431`-`1433`, §6.2). (b) The **GIL** serializes *all* Python
execution onto the single main thread — which is why only the unnamed `python`
main thread appears alongside the pure-C `KittyChildMon`/`KittyPeerMon`/`KittyWriteStdin`
in the live thread list (§3.1). The pure-C threads may run concurrently with the main
thread precisely *because* they never touch Python objects.

### 8.3 Timing — GIL-serialized dispatch, `input_delay` batching, and the `Py_DECREF`

Timing matters because Python work is serialized: a long main-thread operation delays
everything else (§7.3, max gap 1.281 s). Hand-off timing is shaped by the `input_delay`
throttle (`kitty/vt-parser.c:1425`, §6.3). Ownership timing also shows up per call: the
C core `Py_DECREF`s the callback's return value right after the call
(`kitty/screen.c:90`), so the returned `PyObject*`'s lifetime is exactly one dispatch.

### 8.4 The contrast: the single place kitty *does* release the GIL

The reason confining Python to the GIL-holding main thread is safe is the CPython rule
that only a GIL-holding thread may touch Python objects or refcounts. kitty honours
this by *not* calling Python from its pure-C threads. The lone place the C core
explicitly releases the GIL is `kitty/utmp.c` — `Py_BEGIN_ALLOW_THREADS` at
`kitty/utmp.c:17` and `Py_END_ALLOW_THREADS` at `kitty/utmp.c:23` — and
`grep -rln Py_BEGIN_ALLOW_THREADS kitty/*.c` returns *only* that file, confirming the
scan/dispatch paths deliberately keep the GIL.

---

## 9. Q6 — Runtime-only races

> **How might subtle races emerge only under real runtime conditions?**

These are races that a static reading tends to miss but that the runtime evidence makes
concrete.

### 9.1 The retained-`memoryview` / buffer-reuse race (demonstrated)

The strongest example is object ownership over the parser buffer. Nothing in the type
system stops Python from keeping the `memoryview` handed to `clipboard_control`. At
runtime, once the parser reuses/compacts the buffer, that retained view points at
*different* data. §8.1 demonstrated exactly this: the same view changed from
`b'c;QUFBQUFBQUE='` to `b'c;WlpaWlpaWlo='` after further parsing
(`retained_changed_after_more_parsing: True`). Under real conditions the compaction is
the post-parse `memmove` — `if (self->read.sz) memmove(self->buf, self->buf + self->read.consumed, self->read.sz);`
at `kitty/vt-parser.c:1441` — which shifts buffer contents and invalidates any retained
view. The bug only manifests if code (a) retains the view and (b) reads it after more
input arrives — i.e. it is timing-dependent and can hide until the buffer happens to be
reused.

### 9.2 The lock-release-during-dispatch window

Because the main thread **releases the parser mutex while calling Python**
(`kitty/vt-parser.c:1431`-`1433`, §6.2), the I/O thread can mutate the buffer *during*
a dispatch. This is safe *as written* because the dispatched `memoryview` is consumed
synchronously and not retained — but it is precisely the window in which a
retained-view bug (§9.1) would become a genuine data race between the pure-C I/O thread
writing the buffer and Python reading a stale view. The safety depends on a runtime
invariant (consume-during-dispatch), not on a compile-time guarantee.

### 9.3 `input_delay` timing interacting with buffer-full backpressure

The parse trigger mixes a timer with a capacity check:
`flush || time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ`
(`kitty/vt-parser.c:1425`). Which clause fires is timing-dependent: under a large fast
paste the capacity clause dominates (and §5.2 showed the payload arriving as 27
~1 MiB chunks with backpressure available via `kitty/child-monitor.c:1501`), whereas
under trickled input the timer dominates. Behaviour that only appears when the buffer
is near-full — e.g. the ~1 MiB partial-chunk size rather than a full escape — is thus a
runtime-only characteristic, invisible to a small-input test. This is the concrete
reason the observed chunk size was ~1 MiB, not the 256 KiB one might infer from
`MAX_ESCAPE_CODE_LENGTH` alone (§5.2, §11).

### 9.4 The refcount race the GIL prevents (validated against the C-API model)

Per the CPython C-API model, reference counts are non-atomic and GIL-protected; two
threads concurrently incrementing/decrementing a refcount without the GIL can lose an
update and corrupt an object's lifetime. kitty avoids this class of race *by
construction*: its pure-C threads never touch Python objects, and the one deliberate
GIL release is isolated to `kitty/utmp.c:17`-`23` (§8.4). The race is therefore
*latent* — it would emerge only if a future change called the C-API (e.g. the
`CALLBACK` at `kitty/screen.c:87`, or a `Py_DECREF` like `kitty/screen.c:90`) from the
`KittyChildMon` I/O thread without holding the GIL. The runtime thread list (§3.1)
showing Python confined to the single `python` main thread is the evidence that this
invariant currently holds.


---

## 10. Coverage-pass checklist

Every sub-question and every named example is answered with adjacent verbatim evidence
and `file:line` citations.

| Item to cover | Where answered | Key evidence |
|---|---|---|
| **Q1** — cross-language data movement under load | §4 | `KittyChildMon` vs `python` GIL-holder (obs6); `CALLBACK`/`Py_DECREF` `kitty/screen.c:87`,`:90`; JSON+base85 `kittens/runner.py:101`-`102` |
| **Q2** — clipboard crossing small vs. large | §5 | small: `NUM_DISPATCHES: 1`, `type=memoryview len=6 is_partial=False readonly=True` (obs2); large: `NUM_DISPATCHES: 27`, rollover `BytesIO`→`BufferedRandom` (obs3) |
| **Q3** — transfer under contention | §6 | lock release `kitty/vt-parser.c:1431`-`1433`; `input_delay` `kitty/vt-parser.c:1425`; backpressure `kitty/child-monitor.c:1501` |
| **Q4** — expensive scrollback scan effect | §7 | linear memory `bytes_per_line_1M: 2570.5` (obs5); scan `SCAN_wall_seconds: 0.462`; starvation `WORKER_max_gap_seconds: 1.281` |
| **Q5** — where timing/concurrency/ownership matter | §8 | `readonly: True`, `'cannot modify read-only memory'` (obs4); parser mutex `kitty/vt-parser.c:206`; GIL contrast `kitty/utmp.c:17`-`23` |
| **Q6** — runtime-only races | §9 | retained-view change `retained_changed_after_more_parsing: True` (obs4); compaction `kitty/vt-parser.c:1441` |
| **User example (a): a large clipboard payload** | §5.2 | 20 MiB → 27 dispatches (26 partial + 1 final), ~1 MiB chunks, disk rollover at 16 MiB, cap 512.0 MiB (obs3) |
| **User example (b): scanning a large scrollback** | §7 | 1M-line scrollback (~2.57 GB RSS), `as_text` scan 0.462 s, rewrap 1.286 s starving a concurrent thread for 1.281 s (obs5) |
| In-process (C-API) vs out-of-process (JSON+base85) boundaries | §3.2, §4 | `CALLBACK` `kitty/screen.c:87`; `base64.b85encode(json.dumps(...))` `kittens/runner.py:102` |
| Zero-copy read-only `memoryview` | §5, §8.1 | `PyMemoryView_FromMemory(..., PyBUF_READ)` `kitty/vt-parser.c:461`; `readonly: True` (obs4) |
| Three named threads + GIL-holding main thread | §3.1 | `KittyChildMon`/`KittyPeerMon`/`KittyWriteStdin` + `python` (obs6) |
| `Tempfile` `BytesIO`→on-disk rollover at 16 MiB | §5.2 | `rollover_size … 16 * 1024 * 1024` `kitty/clipboard.py:237`; `ROLLOVER_AT_DISPATCH: 22` (obs3) |
| `clipboard_max_size` runtime value vs stale "8 MB" | §2.2, §5.2, §11 | `clipboard_max_size: 512.0` (obs1/obs3); `kitty/options/types.py:498` |

---

## 11. Notes on discrepancies & limitations

### 11.1 The "8 MB" clipboard limit is stale; the runtime value is 512.0 MiB

Older external references cite an "8 MB" clipboard accumulation limit. The source and
the runtime disagree: `clipboard_max_size: float = 512.0` at
`kitty/options/types.py:498`, and the runtime reports `clipboard_max_size: 512.0`
(= 536870912 bytes) in both obs1 and obs3. Per the "resolve conflicts in favour of
source + observed runtime" rule, the authoritative cap is **512.0 MiB**, and the "8 MB"
figure is refuted.

### 11.2 Partial-chunk size is ~1 MiB (`BUF_SZ`), not 256 KiB (`MAX_ESCAPE_CODE_LENGTH`)

A plausible static reading is that a large OSC 52 is split into ≤256 KiB partials
(the `MAX_ESCAPE_CODE_LENGTH` cap). The runtime refutes this: observed partials are
~1 MiB (`FIRST_CHUNK_LEN: 1048570`, `MAX_CHUNK_LEN: 1048572` in obs3), bounded by
`BUF_SZ` = 1048576. As explained in §5.2, `MAX_ESCAPE_CODE_LENGTH` (262144) is only the
*trigger threshold* that forces a partial when no ST terminator has been seen; the
partial then flushes *all* accumulated unterminated bytes, which is up to a full
buffer. The exact chunk size is therefore load- and buffering-dependent (it is largest
under a fast bulk paste). This is a direct instance of the "report what you observe,
even if unexpected" rule.

### 11.3 Line-number drift from the pre-scoped map

Two pre-scoped anchors had drifted by the observed commit and are cited here at their
**observed** lines: the post-parse compaction `memmove` is at `kitty/vt-parser.c:1441`
(not `:1443`), and `HistoryBuf.as_ansi` is at `kitty/history.c:348` (not `:347`). All
other anchors matched.

### 11.4 The on-disk temp file's Python type

The rollover target is `tempfile.TemporaryFile()` (`kitty/clipboard.py:35`). Its
observed Python type is `io.BufferedRandom` (`FINAL_WRITE_STATE: ('final', 'BufferedRandom', …)`
in obs3), which is what `TemporaryFile()` returns on POSIX; this is the same object the
source refers to as the on-disk temp file — noted to avoid confusion between the
`tempfile` module name and the runtime type.

### 11.5 Scope and method limitations

- The clipboard accumulation was exercised up to `handle_write_request`'s first action
  (`wr.flush_base64_data()`); the final `fulfill_write_request` step calls `get_boss()`
  and requires the GUI controller, so it was not driven headlessly. This does not affect
  the crossing/accumulation/rollover claims (Q2), which are fully observed.
- The scrollback was populated by scrolling drawn lines into history via the parser;
  memory and scan timings are therefore representative of real content. The
  concurrent-event delay is demonstrated with a Python stand-in thread rather than a
  live kitten process, because a live kitten requires the GUI/event loop; the GIL
  starvation it measures (max gap = 99.6 % of the rewrap) is the same mechanism that
  delays real kitten event delivery.
- All measurements were taken on Python 3.11.15 in the provided container; absolute
  timings will vary with hardware, but the *relationships* (linear memory, scan
  duration ≈ main-thread block, chunk size ≈ `BUF_SZ`) are structural.

### 11.6 Repository left unchanged

All observation scripts lived under `/tmp/kitty_obs/` (outside the tree) and were
removed after use. `git status --porcelain` shows only this new document under
`blitzy/documentation/`; no existing source file was modified.

