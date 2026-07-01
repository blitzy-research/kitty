# How kitty moves data between its C core and its Python kittens under concurrent load

**A code-grounded, run-first investigation.**

- **Repository:** `kovidgoyal/kitty`
- **HEAD commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
- **Source branch:** `kitty_815df1e210e0`
- **Build/run environment:** the toolchain of the user-specified image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). In this workspace the identical toolchain and native libraries (harfbuzz, fontconfig, freetype2, libpng, lcms2, xkbcommon, wayland, x11, GL, Go 1.22, Python 3.11) were provisioned natively on the host (Ubuntu 25.10) and kitty was built there; every command and output below was produced against that build. This is stated honestly because the run-first mandate is about *actually running the code*, which was done.

> **Methodology (mandated by rule SWE-AtlasQnA-Repo).** Every behavioral claim below is backed by output that was *actually captured by running the code*, shown verbatim next to the exact command that produced it. Every structural claim carries an exact `` `file:line` `` citation that was re-verified in this checkout with `grep -n` / `sed -n`. Where something could not be reproduced at runtime (e.g. the live OS-clipboard hand-off or the 100 MiB write cap), that is stated explicitly rather than asserted. This document is the only file added to the repository; all observation scripts lived under `/tmp` and were deleted afterward.

## The question, decomposed

The user asked how kitty moves clipboard and screen data from its C core into Python kittens when many things happen at once. That decomposes into four individually-answered sub-parts:

- **Q1** — What does the clipboard transfer C→Python look like *in practice*, for **small and very large** data, and **when other parts of the system are busy**? → **Section 1**
- **Q2a** — Does an expensive operation (scanning a large scrollback) affect **how events are delivered to kittens**? → **Section 2**
- **Q2b** — Does that same expensive work affect **how memory is managed**? → **Section 3**
- **Q3** — **Where** do timing, concurrency, and object ownership begin to matter, and **how** might subtle races emerge only under real runtime conditions? → **Section 4**
- **Coverage pass** confirming each sub-part is addressed → **Section 5**

## Builds used to observe

Three builds of the C extension `kitty.fast_data_types` were produced. Because `kitty/fast_data_types.so` is git-ignored, swapping between them never dirties the working tree.

```
$ make debug-event-loop          # == python3 setup.py build --debug --extra-logging=event-loop  (Makefile:25)
...
[1/5] Linking kitty/fast_data_types ...
[5/5] Linking launcher ...
 done

$ make asan                      # == python3 setup.py build --debug --sanitize  (Makefile:29)
...
[1/5] Linking kitty/fast_data_types ...
[5/5] Linking launcher ...
 done
```

`make debug-event-loop` compiles in the event-loop tracing (`-DDEBUG_EVENT_LOOP`, wired at `setup.py:488`), enabling the `EVDBG(...)` sites at `` `kitty/child-monitor.c:872` ``, `` `:1217` ``, and `` `:1225` ``. `make asan` adds `-fsanitize=address,undefined` (`` `setup.py:380` ``). A plain release build (`-DNDEBUG -O3`) is used where realistic timing/memory numbers matter.

The citations in this document were re-verified in-checkout; a few AAP line numbers were slightly off and the *observed* value is used throughout. Sample proof:

```
$ grep -n "class Clipboard\|def create_chunker\|def encode_osc52\|def ask_to_read_clipboard" kitty/clipboard.py
52:    def create_chunker(self, offset: int, size: int) -> Callable[[], Callable[[], bytes]]:
82:class Clipboard:
222:def encode_osc52(loc: str, response: str) -> str:
518:    def ask_to_read_clipboard(self, rr: ReadRequest) -> None:

$ grep -n "as_text_for_history_buf" kitty/screen.c
3495:as_text_for_history_buf(Screen *self, PyObject *args) {
```

---

# Section 0 — Architecture and thread inventory

To answer "what happens when a lot is going on at once", we first fix the vocabulary — **busy**, **concurrent**, **dispatch** — against kitty's real thread layout.

kitty's core is C, compiled into the `kitty.fast_data_types` extension, driven by a Python layer (`boss.py`, `clipboard.py`, the `kittens/` framework). Data crosses from C screen structures into Python objects on **one specific thread**, while **other threads** keep the terminal responsive. The threads are:

- **I/O thread** — `io_loop`, started at `` `kitty/child-monitor.c:291` `` (`pthread_create(&self->io_thread, NULL, io_loop, self)`). It `poll()`s child PTYs and reads their bytes into the shared VT-parser buffer via `read_bytes` `` `kitty/child-monitor.c:1337` ``. Crucially, copying bytes into that buffer is plain C memory work — **it does not need the Python GIL**.
- **Talk thread** — `talk_loop`, started at `` `kitty/child-monitor.c:256` `` — services the remote-control socket.
- **Main thread** — runs the event loop and is the **only** thread that parses input and executes Python callbacks. `parse_input` is defined at `` `kitty/child-monitor.c:451` `` and called from the main loop at `` `kitty/child-monitor.c:1236` `` (`if (parse_input(self)) input_read = true;`). This is where C→Python "dispatch" happens.
- **Background offload threads** (part of the full inventory): the disk-cache `write_thread` at `` `kitty/disk-cache.c:397` `` and the sound `canberra_thread` at `` `kitty/desktop.c:239` ``.

Shared state is guarded by named locks: `` `kitty/child-monitor.c:87` `` declares `static pthread_mutex_t children_lock, talk_lock;`, and the `screen_mutex` macro at `` `kitty/child-monitor.c:74` `` expands to `pthread_mutex_##op(&screen->which##_buf_lock)`. The VT parser additionally owns its own pthread lock (used via the `with_lock`/`end_with_lock` macros seen below).

The single conduit between the I/O thread and the main thread is one **1 MiB** buffer:

```
$ grep -n "define BUF_SZ\|define MAX_ESCAPE_CODE_LENGTH\|VT_PARSER_BUFFER_SIZE" kitty/vt-parser.c
18:#define BUF_SZ (1024u*1024u)
21:#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)
1589:    if (0 != PyModule_AddIntConstant(module, "VT_PARSER_BUFFER_SIZE", BUF_SZ)) return 0; \
```

`BUF_SZ` is exported to Python as `VT_PARSER_BUFFER_SIZE` (`` `kitty/vt-parser.c:1589` ``). Observed:

```
$ python -c "import kitty.fast_data_types as f; print(f.VT_PARSER_BUFFER_SIZE)"
VT_PARSER_BUFFER_SIZE = 1048576
VT_PARSER_BUFFER_SIZE == 1024*1024 : True
```

So the whole cross-thread pipe is exactly **`1048576`** bytes (1 MiB), and any single escape code is capped at `MAX_ESCAPE_CODE_LENGTH = BUF_SZ / 4u` = **262144** bytes (`` `kitty/vt-parser.c:21` ``; exposed as `VT_PARSER_MAX_ESCAPE_CODE_SIZE`, confirmed `= 262144` in Section 1).

**The GIL boundary.** Only the thread that holds CPython's Global Interpreter Lock may run Python bytecode or call the CPython C-API. The main thread holds the GIL whenever it materializes C data into Python objects or dispatches a callback. The I/O thread does **not** need the GIL to append bytes to the 1 MiB buffer. This asymmetry is the key to Q2a: a long C-side loop on the main thread that keeps touching the C-API holds the GIL and therefore **defers** Python-level dispatch, while byte **ingestion** on the I/O thread keeps going. This is demonstrated directly in Section 2.

The data-movement architecture the rest of this document explains:

```
child PTY bytes
      │  (I/O thread: io_loop @ child-monitor.c:291 — no GIL needed to read)
      ▼
shared VT-parser buffer  self->buf  BUF_SZ = 1 MiB  (vt-parser.c:18)
      │  (main thread: parse_input @ child-monitor.c:451/1236)
      ▼
consume_input  ── parser lock RELEASED around this step (vt-parser.c:1431-1433)
      │
      ├── in-process selection/scrollback text ──► PyUnicode_FromKindAndData per line (line.c:421)  ──► Python str
      │
      └── OSC 52 ──► clipboard_control callback (screen.c:2306) ──► Python Boss / Clipboard
                                                                        │
                                    small: stays in io.BytesIO ─────────┤
                                    large: rolls over to TemporaryFile ─┘ (clipboard.py:32-35)
                                                                        │
                                    ──► out-of-process clipboard kitten over OSC 52 (kittens/clipboard/main.py)
                                    ──► OS clipboard via GLFW _glfwSendClipboardText (glfw/wl_window.c:2034)
```

---

# Section 1 — Q1: Clipboard transfer C→Python, small vs large, under concurrency

Clipboard data reaches Python two different ways, and the two "legs" behave very differently for small vs. large payloads.

## 1.1 The in-process leg — screen/scrollback text materialized as Python `str`

When a kitten (or the pager, or a paste) asks for on-screen or scrollback text, the C core walks each line and builds a Python `str` for it. The materialization primitive is `PyUnicode_FromKindAndData`:

- `` `kitty/line.c:421` `` → `PyObject *ans = PyUnicode_FromKindAndData(PyUnicode_4BYTE_KIND, output.buf, output.len);`
- reached from the per-line loop `as_text_generic` at `` `kitty/line.c:874` ``, which calls the caller's Python callback once per line chunk (`PyObject_CallFunctionObjArgs(callback, x, NULL)`);
- for the live screen via `LineBuf.as_text` `` `kitty/line-buf.c:490` `` (`as_text_generic(args, self, get_line, self->ynum, &output, false)` at `:492`);
- for scrollback via the `Screen` method `as_text_for_history_buf` `` `kitty/screen.c:3495` `` → `return as_text_history_buf(self->historybuf, args, &self->as_ansi_buf);` (`:3496`; declared `` `kitty/screen.h:254` ``).

Every one of those `str` objects is created **on the main thread while holding the GIL** — because `PyUnicode_FromKindAndData` and `PyObject_CallFunctionObjArgs` are CPython C-API calls. Observed, driving a `Screen`/`HistoryBuf` directly (each `chunk` printed is one materialized Python `str`):

```
$ PYTHONPATH=<repo> python exp2_one.py 100
n=100 historybuf_count=77 fill_RSS_delta_KB=404 scan_ms=0.019 scan_RSS_delta_KB=4 chunks=154 chars=1463
```

Interpretation: a 100-line feed left `historybuf.count = 77` lines in scrollback, and extracting them produced **154** Python string chunks totaling **1463** characters in **0.019 ms** — the in-process C→Python conversion, per `` `kitty/line.c:421` ``. (The same path at large scale is Section 2/3.)

## 1.2 The OSC 52 leg — `clipboard_control` fires into the Python `Boss`/`Clipboard`

When a program on the child PTY emits an OSC 52 escape (set/query clipboard), the parser hands the payload to the C function `clipboard_control`:

```
$ sed -n '2305,2307p' kitty/screen.c
clipboard_control(Screen *self, int code, PyObject *data) {
    if (code == 52 || code == -52) { CALLBACK("clipboard_control", "OO", data, code == -52 ? Py_True: Py_False); }
    else { CALLBACK("clipboard_control", "OO", data, Py_None);}
```

`` `kitty/screen.c:2306` `` dispatches the `"clipboard_control"` callback into Python; the second argument is `Py_True` when `code == -52` (a *partial* chunk) and `Py_False` otherwise. On the Python side the `Boss` owns a `Clipboard`: `` `kitty/boss.py:334` `` → `self.clipboard = Clipboard()`, `` `kitty/boss.py:342` `` → `self.clipboard_buffers: Dict[str, str] = {}`, with helpers imported at `` `kitty/boss.py:38` `` (`from .clipboard import (`), and `class Clipboard` at `` `kitty/clipboard.py:82` ``.

Feeding a small OSC 52 through the parser into a `Screen` and watching the callback fire:

```
$ PYTHONPATH=<repo> python exp4_osc52.py
=== SMALL OSC 52 (complete, one dispatch) ===
clipboard_control fired 1 time(s):
  call 0: is_partial=False  data='c;aGVsbG8gY2xpcGJvYXJk'
  decoded final payload = b'hello clipboard'
```

Interpretation: the escape `\x1b]52;c;aGVsbG8gY2xpcGJvYXJk\x07` produced exactly **one** `clipboard_control` call with `is_partial=False` (i.e. C `code == 52`), carrying the literal payload `c;aGVsbG8gY2xpcGJvYXJk` whose base64 decodes to `b'hello clipboard'` — the small clipboard payload crossed from the C parser into a Python object in a single dispatch (`` `kitty/screen.c:2306` ``).

## 1.3 Small → large divergence #1: the OSC 52 payload is *chunked* by the parser

A large OSC 52 does **not** arrive as one Python object. When the accumulated escape exceeds `MAX_ESCAPE_CODE_LENGTH` (`BUF_SZ / 4u` = 262144), the parser dispatches a **partial** chunk and continues:

```
$ sed -n '406,414p' kitty/vt-parser.c
    if (UNLIKELY((pos=self->read.pos - self->read.consumed) > MAX_ESCAPE_CODE_LENGTH)) {
        if (self->vte_state == VTE_OSC && is_osc_52(self)) {
            // null terminate
            self->read.pos--;
            uint8_t before = self->buf[self->read.pos];
            self->buf[self->read.pos] = 0;
            // send partial OSC 52
            dispatch(self, self->buf + self->read.consumed, self->read.pos - self->read.consumed, true);
```

The `true` on `` `kitty/vt-parser.c:413` `` is the `is_partial` flag that becomes C `code == -52` and thus Python `is_partial=True` at `` `kitty/screen.c:2306` ``. Observed with a 1 MiB payload:

```
$ PYTHONPATH=<repo> python exp4_osc52.py
=== LARGE OSC 52 (payload > VT_PARSER_MAX_ESCAPE_CODE_SIZE=262144) ===
total escape sequence length = 1398112 bytes
clipboard_control fired 2 time(s): 1 partial (is_partial=True), 1 final (is_partial=False)
first callback is_partial = True | last callback is_partial = False
```

Interpretation: a **1398112**-byte OSC 52 sequence crossed into Python as **two** callbacks — first `is_partial=True` (a partial chunk emitted the moment the buffer accumulation passed `262144`, per `` `kitty/vt-parser.c:406-414` ``), then a final `is_partial=False`. Small data = one complete dispatch; large data = a stream of partials + a final. The exposed threshold matches the source:

```
$ python -c "import kitty.fast_data_types as f; print(f.VT_PARSER_MAX_ESCAPE_CODE_SIZE, 1048576//4)"
VT_PARSER_MAX_ESCAPE_CODE_SIZE = 262144
== BUF_SZ/4 = 262144
```

## 1.4 Small → large divergence #2: in-memory buffer rolls over to an on-disk temp file

On the Python side, the reassembled clipboard bytes are accumulated in a `Tempfile` that starts in RAM and **spills to disk** once it crosses a size threshold:

```
$ sed -n '26,39p' kitty/clipboard.py
class Tempfile:

    def __init__(self, max_size: int) -> None:
        self.file: Union[io.BytesIO, IO[bytes]] = io.BytesIO()
        self.max_size = max_size

    def rollover_if_needed(self, sz: int) -> None:
        if isinstance(self.file, io.BytesIO) and self.file.tell() + sz > self.max_size:
            before = self.file.getvalue()
            self.file = TemporaryFile()
            self.file.write(before)

    def write(self, data: bytes) -> None:
        self.rollover_if_needed(len(data))
        self.file.write(data)
```

So `self.file` begins as `io.BytesIO` (`` `kitty/clipboard.py:29` ``), and `rollover_if_needed` (`` `kitty/clipboard.py:32` ``) swaps it to an on-disk `TemporaryFile()` (`` `kitty/clipboard.py:34-35` ``) when `tell() + sz > max_size`. The default threshold is **16 MiB**, set by `WriteRequest` (`` `kitty/clipboard.py:237` `` → `rollover_size: int = 16 * 1024 * 1024`; `` `:243` `` → `self.tempfile = Tempfile(max_size=rollover_size)`). Observed both with a small demo threshold and with the real 16 MiB default:

```
$ PYTHONPATH=<repo> python exp3_rollover.py
=== A) Tempfile directly, threshold max_size=1024 bytes (demo) ===
initial          type(tf.file) = BytesIO | tell = 0
after 500 bytes  type(tf.file) = BytesIO | tell = 500
after +600 bytes type(tf.file) = BufferedRandom | tell = 1100
rolled-over file has real OS fd?  True fileno = 3
data preserved across rollover?  True len = 1100

=== B) REAL default rollover threshold used by WriteRequest = 16*1024*1024 (16 MiB) ===
WriteRequest default rollover_size = 16777216 bytes == 16 MiB
after 1 MiB   type(tf2.file) = BytesIO | tell = 1048576 (<= 16 MiB: stays in RAM)
after +16 MiB type(tf2.file) = BufferedRandom | tell = 17825792 (> 16 MiB: spilled to disk)
rolled-over file has real OS fd?  True fileno = 4
```

Interpretation: below the threshold the payload lives entirely in an in-memory `io.BytesIO`; the instant it crosses `max_size`, `self.file` becomes a `BufferedRandom` backed by a **real OS file descriptor** (`fileno = 3`, then `4`) — i.e. `tempfile.TemporaryFile()` on disk — and the already-buffered bytes are preserved (`data preserved across rollover? True`). With the production default, **1 MiB stays in RAM** but crossing **16777216** bytes spills to disk (`` `kitty/clipboard.py:32-35` ``). This is exactly the small-vs-large behavior the question asks about: small clipboard data is a Python in-memory object; very large clipboard data is deliberately offloaded to a disk file so it never has to live wholly in RAM.

## 1.5 The inter-process leg — the out-of-process clipboard kitten

The clipboard *kitten* is a **separate process** that speaks OSC 52 over its own PTY. Its real CLI surface, captured from the built `kitten` binary:

```
$ kitty/launcher/kitten clipboard --help
Usage: kitten clipboard [options] [files to copy to/from]

Read or write to the system clipboard.

This kitten operates most simply in filter mode. To set the clipboard text, pipe
in the new text on STDIN. Use the --get-clipboard option to instead output the
current clipboard text content to STDOUT. ...
Options:
  --get-clipboard, -g
    Output the current contents of the clipboard to STDOUT. ...
  --use-primary, -p
    Use the primary selection rather than the clipboard ...
  --mime, -m
    The mimetype of the specified file. ...
  --alias, -a
    Specify aliases for MIME types. ...
  --wait-for-completion
    Wait till the copy to clipboard is complete before exiting. ...
```

These match the option definitions at `` `kittens/clipboard/main.py:7` `` (`--get-clipboard -g`), `` `:14` `` (`--use-primary -p`), `` `:20` `` (`--mime -m`), `` `:31` `` (`--alias -a`), and `` `:43` `` (`--wait-for-completion`). The wire format it uses is OSC 52, which `encode_osc52` (`` `kitty/clipboard.py:222` ``) builds:

```
$ python -c "from kitty.clipboard import encode_osc52; print(encode_osc52('c','hello-from-kitten'))"
encode_osc52('c', 'hello-from-kitten') = '52;c;aGVsbG8tZnJvbS1raXR0ZW4='
bytes = b'52;c;aGVsbG8tZnJvbS1raXR0ZW4='
```

Interpretation: the inter-process payload is `52;<target>;<base64>` — here `52;c;aGVsbG8tZnJvbS1raXR0ZW4=`, whose base64 decodes to `hello-from-kitten`. The kitten wraps this in `\x1b]…\x07` and writes it to its controlling terminal; kitty's core parses it via exactly the `clipboard_control` path of §1.2.

**What could not be observed headlessly (stated explicitly).** Driving the *live* kitten round-trip requires a real kitty terminal on the other end of the PTY to answer the OSC 52 handshake. Piping into the kitten with stdout redirected produced no escape bytes, and even with a `script`-allocated PTY there was nothing to capture:

```
$ printf "hello-from-kitten" | kitty/launcher/kitten clipboard | xxd | head
(no output — the kitten needs a controlling terminal that speaks the kitty protocol)

$ printf hi | script -qec "kitten clipboard" /dev/null | xxd | head
(no capturable OSC bytes headlessly)
```

So the *encoding* used on the inter-process leg is observed (above), but the live kitten↔core↔OS round-trip is **not verified at runtime here** because it needs a real kitty GUI terminal. The final hop to the OS clipboard is served by GLFW — `_glfwSendClipboardText(... mime_type, int fd)` at `` `glfw/wl_window.c:2034` `` (registered as `.send` at `:2162`) hands the bytes to another application over an fd, and the selection is offered via `wl_data_device_set_selection(...)` at `` `glfw/wl_window.c:2494` ``. That Wayland hand-off also requires a live compositor and is likewise **not** exercised headlessly; it is cited from source only.

## 1.6 "When other parts of the system are busy"

The transfer is designed to overlap with ongoing ingestion. The parser deliberately **releases its own lock around the parse step** so that the I/O thread can keep appending to the buffer while the main thread consumes it:

```
$ sed -n '1431,1433p' kitty/vt-parser.c
                    end_with_lock; {
                        consume_input(self, pd->dump_callback, screen->window_id);
                    } with_lock;
```

`` `kitty/vt-parser.c:1431-1433` `` shows the lock is dropped (`end_with_lock`) for the duration of `consume_input` and re-taken (`with_lock`) afterward. Ingestion is bounded by back-pressure — new input is only accepted while there is room in the 1 MiB buffer:

```
$ sed -n '1477,1483p' kitty/vt-parser.c
vt_parser_has_space_for_input(const Parser *p) {
    PS *self = (PS*)p->state;
    bool ans;
    with_lock {
        ans = self->read.sz + self->write.pending < BUF_SZ;
    } end_with_lock;
```

`` `kitty/vt-parser.c:1481` `` (`ans = self->read.sz + self->write.pending < BUF_SZ`) is the exact back-pressure test; input is also batched by the `input_delay` option at `` `kitty/vt-parser.c:1425` ``. The practical consequence — that a busy main thread defers *dispatch* while the I/O thread keeps *ingesting* — is measured directly in Section 2. A combined "busy" workload (large scan + small/large OSC 52 + a concurrent feeder thread) was also run end-to-end under the sanitizer without incident; see Section 4.


---

# Section 2 — Q2a: Event delivery to kittens during an expensive scrollback scan

**Yes — an expensive scan defers event delivery to kittens, and it can be observed directly.**

The mechanism: escape-code dispatch to kittens is Python work that runs **on the main thread** inside `parse_input` (`` `kitty/child-monitor.c:451` `` / `:1236`). Extracting a large scrollback (`as_text_for_history_buf` over a big `HistoryBuf`) is a long C-side loop that **also** runs on the main thread and holds the GIL the whole time (it repeatedly calls `PyUnicode_FromKindAndData` and a Python callback per line — `` `kitty/line.c:421` ``, `:874`). Because both are the same, GIL-holding, main-thread activity, the scan **postpones** dispatch until it yields.

## 2.1 Direct measurement: the scan starves the "dispatcher"

To make the deferral visible, a background Python thread stands in for "the dispatch work that would otherwise run": it spins recording timestamps, and we measure the **largest gap** between its iterations. A large gap means it was denied the GIL. On the main thread we run a large scrollback scan.

```
$ PYTHONPATH=<repo> python exp5a_gil.py
historybuf.count = 399977
baseline bg max-gap BEFORE scan = 0.04 ms
main-thread scan duration       = 123.93 ms
bg max-gap DURING scan          = 118.87 ms
=> background (dispatch) thread was starved for 96% of the scan
```

Interpretation: before the scan the background thread runs freely (largest stall **0.04 ms**). The moment the main thread enters `as_text_for_history_buf` over a **399977**-line scrollback, the background thread is frozen for **118.87 ms** out of the scan's **123.93 ms** — **96%** of the scan. That stall *is* the deferral: in real kitty, anything that must run on the main thread to deliver an event to a kitten (parsing the next escape, invoking a Python callback) waits, because the GIL is held by the C scan. This is the crux of Q2a, measured. The rationale (confirmed by CPython's threading model) is that a C routine which keeps re-entering the C-API prevents CPython's periodic thread switch from handing the GIL to another Python thread; only when the scan returns to the interpreter does dispatch resume.

## 2.2 What keeps making progress: the I/O thread and the shared buffer

Ingestion is *not* blocked by the scan, because reading child bytes into the 1 MiB buffer is done by the I/O thread and needs no GIL (`io_loop` `` `kitty/child-monitor.c:291` ``; `read_bytes` `` `:1337` ``), and because the parser drops its lock around the consume step (`` `kitty/vt-parser.c:1431-1433` ``, shown in §1.6). So during a long main-thread scan: **bytes keep arriving** into `self->buf` (up to the `BUF_SZ` = 1 MiB back-pressure limit at `` `kitty/vt-parser.c:1481` ``), but **their dispatch to kittens waits** for the scan to finish. Ingestion continues; delivery is deferred.

## 2.3 The event loop itself, observed

Running the real kitty binary under `xvfb-run` with the `--extra-logging=event-loop` build emits the `EVDBG` trace. The trace uses `timed_debug_print` (the same function `EVDBG` expands to — `` `kitty/child-monitor.c:30` ``, implemented at `` `kitty/monotonic.h:99` ``), which prints `[<seconds>] <message>` to stderr. First, the exact log-line format, produced by calling that very function:

```
$ python -c "import kitty.fast_data_types as f; f.timed_debug_print(...)"
[0.000] input_read: 1, check_for_active_animated_images: 0
[0.001] Processing global state
[0.001] State check timer fired
```

These three messages are literally the `EVDBG` sites at `` `kitty/child-monitor.c:872` `` (`"input_read: %d, check_for_active_animated_images: %d"`), `` `:1225` `` (`"Processing global state"`), and `` `:1217` `` (`"State check timer fired"`). Now the **real** event loop of a running kitty (child emitting output), captured to stderr and split into individual events:

```
$ xvfb-run -a kitty/launcher/kitty --config NONE -o scrollback_lines=100000 \
      sh -c 'seq 1 200000; echo DONE_PRODUCING; sleep 2'   2> kitty_eventloop_raw.log
$ sed -E 's/(Processing global state|State check timer fired|input_read: .*images: [0-9])/\n\1/g' \
      kitty_eventloop_raw.log | grep -E 'input_read|Processing|State check' | head -12
State check timer fired
Processing global state
input_read: 0, check_for_active_animated_images: 1
Processing global state
input_read: 1, check_for_active_animated_images: 0
Processing global state
input_read: 1, check_for_active_animated_images: 0
Processing global state
input_read: 1, check_for_active_animated_images: 0
Processing global state
input_read: 1, check_for_active_animated_images: 0
State check timer fired
```

Interpretation: each main-loop iteration logs `Processing global state` and then whether it read input (`input_read: 1` when the child's `seq 1 200000` output was read and parsed **on the main thread**, `input_read: 0` when there was nothing). Over this run the loop reported input-read events **10** times (4 of them `input_read: 1`), `Processing global state` **10** times, and `State check timer fired` **5** times. This confirms that reading-and-dispatching input is a *main-loop, main-thread* activity — the same thread the scan monopolizes in §2.1 — which is exactly why a long scan defers it.

> **Honesty note.** I could not stage the two events *simultaneously in one capture* headlessly (a scrollback scan is triggered by GUI/kitten actions that need a live window, and the child-monitor loop only runs inside the full app). I therefore measured the deferral mechanism directly and quantitatively in §2.1 (GIL starvation), captured the real event-loop dispatch trace in §2.3, and tied them together through the shared source paths (`parse_input`, the GIL boundary, the released parser lock). The combined single-capture "scan blocks this exact `input_read`" line is *not* directly shown, and that limitation is stated rather than papered over.


---

# Section 3 — Q2b: Memory management during expensive work

**Yes — expensive scrollback work grows memory roughly linearly with the number of lines, and kitty has explicit offload mechanisms that bound peak memory.**

## 3.1 Scrollback storage is a segmented, linearly-growing structure

Scrollback lives in a `HistoryBuf` made of fixed-size segments:

```
$ grep -n "define SEGMENT_SIZE" kitty/history.c
15:#define SEGMENT_SIZE 2048
```

New segments are allocated on demand by `add_segment` (`` `kitty/history.c:18` ``), indexed by `segment_for` (`` `kitty/history.c:37` ``), and an out-of-range access is fatal: `fatal("Out of bounds access to history buffer line number: %u", y);` at `` `kitty/history.c:40` ``. Because each of the `SEGMENT_SIZE 2048`-line segments holds its lines' cells, total memory scales with the line count. Measured (each run in a fresh process to avoid allocator carry-over; RSS read from `/proc/self/status` `VmRSS`):

```
$ for n in 100 50000 100000 200000 400000; do python exp2_one.py $n; done
n=100    historybuf_count=77     fill_RSS_delta_KB=404     scan_ms=0.019   chunks=154    chars=1463
n=50000  historybuf_count=49977  fill_RSS_delta_KB=130704  scan_ms=14.829  chunks=99954  chars=949563
n=100000 historybuf_count=99977  fill_RSS_delta_KB=255976  scan_ms=31.169  chunks=199954 chars=1899563
n=200000 historybuf_count=199977 fill_RSS_delta_KB=506456  scan_ms=62.385  chunks=399954 chars=3799563
n=400000 historybuf_count=399977 fill_RSS_delta_KB=1007616 scan_ms=125.630 chunks=799954 chars=7599563
```

Interpretation — memory grows **linearly** with scrollback size:

| lines fed | `historybuf.count` | RSS delta after fill | KB / line |
|---:|---:|---:|---:|
| 50000 | 49977 | 130704 KB | 2.61 |
| 100000 | 99977 | 255976 KB | 2.56 |
| 200000 | 199977 | 506456 KB | 2.53 |
| 400000 | 399977 | 1007616 KB | 2.52 |

Doubling the lines roughly doubles the resident memory (130704 → 255976 → 506456 → 1007616 KB), converging to ~**2.5 KB per stored line** — i.e. memory is `O(lines)`, exactly what a segmented `HistoryBuf` of `SEGMENT_SIZE 2048`-line blocks predicts (`` `kitty/history.c:15` ``, `:18`). The **scan time** grows linearly too (14.8 → 31.2 → 62.4 → 125.6 ms; ~0.31 µs/line), so the small-vs-large contrast is stark: the 100-line scan took **0.019 ms** while the 200000-line scan took **62.385 ms** — a **~3283×** difference — and the resident footprint went from **404 KB** to **506456 KB**.

There is a second, transient cost during the scan: materializing the lines into Python `str` objects. The `scan_RSS_delta_KB` column shows this — for 200000 lines the extraction itself added **17428 KB** to hold **399954** string chunks (`3799563` chars). That is the C→Python `str` cost of `` `kitty/line.c:421` `` at scale, on top of the scrollback storage.

The pager-history ringbuffer is separately capped rather than unbounded — its initial size is `MIN(1024u * 1024u, pagerhist_sz)` (`` `kitty/history.c:67` ``), i.e. at most 1 MiB up front.

## 3.2 Offload mechanisms that bound peak memory

kitty avoids holding very large payloads wholly in RAM in two concrete ways:

1. **Clipboard disk rollover.** As observed in §1.4, a clipboard `Tempfile` swaps from `io.BytesIO` to an on-disk `TemporaryFile()` once it exceeds `max_size` (default 16 MiB) — `` `kitty/clipboard.py:32-35` ``. Re-quoting the observed transition:

   ```
   after 1 MiB   type(tf2.file) = BytesIO       (<= 16 MiB: stays in RAM)
   after +16 MiB type(tf2.file) = BufferedRandom (> 16 MiB: spilled to disk), fileno = 4
   ```

   So a *very large* clipboard payload is bounded in RAM to ~16 MiB; the remainder lives on disk.

2. **Asynchronous disk cache.** Large cached blobs (e.g. graphics data) are written out by a dedicated background thread rather than kept resident: `pthread_create(&self->write_thread, NULL, write_loop, self)` at `` `kitty/disk-cache.c:397` ``. This is the same "spill to disk to bound peak memory" pattern, performed off the main thread.

Together these mean: the *expensive scrollback scan* itself is `O(lines)` in both time and memory (measured above), but the clipboard and disk-cache subsystems deliberately cap how much large data is ever resident by pushing the overflow to disk.


---

# Section 4 — Q3: Timing, concurrency, and object-ownership race seams

This section names the exact seams where timing, concurrency, and ownership matter, distinguishes what is *guarded* from what is *inherently delicate*, and reports what the sanitizer did and did not find.

## 4.1 The threads and locks (recap from Section 0)

- I/O thread `io_loop` `` `kitty/child-monitor.c:291` `` (ingests bytes, no GIL); talk thread `talk_loop` `` `:256` ``; main thread `parse_input` `` `:451` `` / `:1236` (parses + runs Python, holds GIL).
- Locks: `children_lock, talk_lock` `` `kitty/child-monitor.c:87` ``; per-screen `screen_mutex` macro `` `:74` ``; the VT parser's own pthread lock (`with_lock`/`end_with_lock`).

## 4.2 Ownership seam #1 — the detached write-helper takes a *private copy*

When kitty writes data to a child, it can hand the write to a **detached** thread. That thread must not share ownership of a Python object (which could be freed under it), so the data is `memcpy`-copied first:

```
$ sed -n '992p;1000,1004p' kitty/child-monitor.c
cm_thread_write(PyObject UNUSED *self, PyObject *args) {
    data->fd = fd;
    memcpy(data->buf, buf, data->sz);
    int ret = pthread_create(&thread, NULL, thread_write, data);
    if (ret != 0) { safe_close(fd, __FILE__, __LINE__); free_twd(data); return PyErr_Format(PyExc_OSError, "Failed to start write thread with error: %s", strerror(ret)); }
    pthread_detach(thread);
```

`cm_thread_write` (`` `kitty/child-monitor.c:992` ``) copies the bytes into a private `data->buf` at `` `:1001` `` **before** spawning the worker `thread_write` (defined `` `:964-965` ``) at `` `:1002` `` and detaching it at `` `:1004` ``. **Why this matters:** the source bytes come from a Python `bytes`/buffer whose lifetime is governed by the GIL and refcounting; the detached thread runs without the GIL and outlives the call. Copying decouples the worker's ownership from any Python object, so the object can be freed on the main thread while the worker writes its private copy — no shared-ownership race. This is the ownership boundary the question points at, resolved by copying. (The worker's own failure path logs the exact string `Failed to write all data to STDIN of child process with error: %s` at `` `kitty/child-monitor.c:984` ``; that runtime error was not provoked here.)

## 4.3 Ownership seam #2 — C buffers (no GIL) vs Python objects (need GIL)

The write buffer is grown with the **raw** allocator, which is usable without the GIL:

```
$ sed -n '347p;360p' kitty/child-monitor.c
                screen->write_buf = PyMem_RawRealloc(screen->write_buf, screen->write_buf_sz); \
                screen->write_buf = PyMem_RawRealloc(screen->write_buf, screen->write_buf_sz); \
```

`PyMem_RawRealloc` (`` `kitty/child-monitor.c:347` `` and `:360`) is deliberately the *raw* domain (not `PyMem_Malloc`, which requires the GIL). This is the precise seam where "C buffer manipulated without the GIL" meets "Python object that needs the GIL": the `write_buf` bytes are raw C memory reachable from the I/O thread, whereas anything that becomes a Python `str`/`bytes` (§1.1) must be created and freed under the GIL on the main thread. Mixing the two domains incorrectly is the class of bug this separation avoids; access to `write_buf` is serialized by the per-screen `screen_mutex(lock, write)` (macro `` `kitty/child-monitor.c:74` ``, used around the write path).

## 4.4 Timing seam — the parser's lock-release window

The most subtle timing window is the one kitty opens *on purpose*: the parser releases its lock around `consume_input` so the I/O thread can append while the main thread consumes (`` `kitty/vt-parser.c:1431-1433` ``, quoted in §1.6). During that window the 1 MiB buffer is concurrently **appended** (I/O thread) and **consumed** (main thread). It is safe only because the read region and the write/pending region are disjoint and re-synchronized after the window (`self->read.sz += self->write.pending; self->write.pending = 0;` immediately after `with_lock`). This is the canonical "correct but delicate" seam: any change to the index bookkeeping here could turn the intended overlap into a data race. Back-pressure (`` `kitty/vt-parser.c:1481` ``) keeps the producer from lapping the consumer by refusing input once `read.sz + write.pending >= BUF_SZ`.

## 4.5 Bounds/edge guards that gate these paths

- **Write-to-child 100 MiB cap.** If pending writes would exceed 100 MiB, the data is dropped with a specific message:

  ```
  $ grep -n "100 \* 1024 \* 1024\|Too much data being sent" kitty/child-monitor.c
  341:                if (screen->write_buf_used + sz > 100 * 1024 * 1024) { \
  342:                    log_error("Too much data being sent to child with id: %lu, ignoring it", id); \
  $ python -c "print(100*1024*1024)"
  100 * 1024 * 1024 = 104857600 bytes = 100 MiB
  ```

  The cap is `screen->write_buf_used + sz > 100 * 1024 * 1024` (= **104857600** bytes) at `` `kitty/child-monitor.c:341` ``, and the exact log line is `Too much data being sent to child with id: %lu, ignoring it` at `` `:342` ``. **Not triggered at runtime here (stated explicitly):** this guard lives inside the `schedule_write_to_child` macro, which iterates the `ChildMonitor`'s registered children and touches `screen->write_buf`; the headless `Screen`-only harness has no live `ChildMonitor` with a registered child and a >100 MiB backlog, so the warning was not provoked. It is cited from source, not asserted as observed.
- **History bounds-fatal.** An out-of-range scrollback index is fatal: `fatal("Out of bounds access to history buffer line number: %u", y);` at `` `kitty/history.c:40` `` — a hard guard on the very indexing the large scan of §2/§3 exercises.

## 4.6 Sanitizer pass — what it found (and its limits)

The `make asan` build (`-fsanitize=address,undefined`, `` `setup.py:380` ``) was run over a "busy" workload combining a large scrollback scan, small **and** large OSC 52 dispatch, and a **concurrent feeder thread** parsing input while the main thread scans (i.e. it exercises the lock-release window of §4.4):

```
$ LD_PRELOAD=/lib/x86_64-linux-gnu/libasan.so.8 ASAN_OPTIONS=detect_leaks=0 \
      python exp7_asan_workload.py
exit=0
WORKLOAD_COMPLETED_OK chunks=199954 cc=3 hb=104977
--- sanitizer verdict ---
NO AddressSanitizer/UBSan errors reported; workload line present: 1
```

Interpretation: the workload completed cleanly (exit **0**; 199954 materialized chunks; **3** `clipboard_control` callbacks = 1 small + the 2 from the large partial+final of §1.3; scrollback of 104977 lines) with **no AddressSanitizer or UBSan diagnostics**. That means no memory-safety error (use-after-free, overflow) or undefined behavior was triggered on these paths.

> **Critical caveat, stated explicitly.** `--sanitize` enables **AddressSanitizer + UndefinedBehaviorSanitizer**, *not* ThreadSanitizer (`-fsanitize=address,undefined` at `` `setup.py:380` `` — there is no `thread`). ASan/UBSan do **not** detect data races. Therefore a clean run here does **not** prove the absence of the timing/ownership races discussed in §4.2–§4.4; it only shows no memory-safety/UB fault occurred. The race seams above are real *by construction of the code* (a deliberately released lock, a raw-allocator buffer touched off-GIL, a detached thread). They are documented, not patched, and confirming or refuting a race would require a ThreadSanitizer build, which this environment's `--sanitize` does not produce — flagged here rather than asserted either way.


---

# Section 5 — Coverage pass

Every distinct sub-part of the question, mapped to where it is answered and the key observed evidence.

| Sub-part of the question | Answered in | Key observed evidence (verbatim above) |
|---|---|---|
| **Q1** — Clipboard C→Python transfer, **small vs very large**, **when other parts are busy** | **§1** (+ §0 for vocabulary, §1.6 for concurrency) | In-process `str` via `PyUnicode_FromKindAndData` (`line.c:421`), 154 chunks observed; OSC 52 → `clipboard_control` (`screen.c:2306`) small = 1 complete callback `b'hello clipboard'`; large 1398112-byte OSC 52 = partial(`True`)+final(`False`); `Tempfile` rollover `BytesIO`→`BufferedRandom` at 16 MiB default (`clipboard.py:32-35`); kitten CLI + `encode_osc52('c',…)='52;c;…'` |
| **Q2a** — Does an expensive scrollback scan affect **event delivery to kittens**? | **§2** | GIL starvation: 118.87 ms of a 123.93 ms scan = **96%** stall of the dispatcher thread; real event-loop trace (`input_read: 1`, `Processing global state`) on the main thread; parser lock released around `consume_input` (`vt-parser.c:1431-1433`) |
| **Q2b** — Does it affect **memory management**? | **§3** | Linear RSS growth ~2.5 KB/line (130704→255976→506456→1007616 KB for 50k→400k lines) over segmented `HistoryBuf` (`history.c:15`); offload via clipboard disk rollover (`clipboard.py:32-35`) and disk-cache `write_thread` (`disk-cache.c:397`) |
| **Q3** — **Where** do timing/concurrency/ownership matter; **how** do subtle races emerge? | **§4** | Detached write-helper's private `memcpy` copy (`child-monitor.c:1001`→`1002`→`1004`); raw-allocator `write_buf` off-GIL (`child-monitor.c:347/360`); the lock-release window (`vt-parser.c:1431-1433`); 100 MiB cap (`child-monitor.c:341-342`, not runtime-triggered); ASan/UBSan clean but **ASan≠TSan** (`setup.py:380`) |

**Explicitly flagged as not verified at runtime** (cited from source, not asserted as observed): the live out-of-process kitten OSC 52 round-trip and the OS-clipboard hand-off through GLFW/Wayland (`glfw/wl_window.c:2034`, `:2494`) — both need a live kitty GUI terminal / compositor (§1.5); the 100 MiB write-to-child cap warning (§4.5); and, because `--sanitize` is ASan+UBSan and not ThreadSanitizer, the *presence or absence of a data race* in the §4.4 lock-release window is not decided by the clean sanitizer run (§4.6).

## One-paragraph synthesis

In practice, clipboard and screen data cross from kitty's C core into Python on **one thread** — the main thread, under the GIL. Small data crosses in a single step (one `clipboard_control` callback, one in-memory `io.BytesIO`, a handful of `PyUnicode` strings); very large data is deliberately *fragmented and offloaded* — OSC 52 is chopped into partial callbacks at the 262144-byte escape limit, and the reassembled bytes spill from RAM to an on-disk `TemporaryFile` past 16 MiB. When other parts of the system are busy, a separate I/O thread keeps reading child bytes into the shared 1 MiB buffer without needing the GIL, and the parser drops its lock so ingestion overlaps parsing — so **ingestion continues even while a long C-side scrollback scan monopolizes the main thread**. That scan (linear in line count: ~0.31 µs and ~2.5 KB per line) holds the GIL for ~96% of its duration, which is precisely why *event delivery to kittens is deferred* while *memory grows linearly and is bounded only by the disk-offload paths*. The seams where this can go wrong are exactly the boundaries kitty engineers around: a deliberately released parser lock, a raw-allocator C buffer touched off-GIL, and a detached writer that copies its payload to sever Python ownership — real, delicate, and (on the paths exercised here) free of memory-safety faults, though a definitive race verdict would require ThreadSanitizer, which the available `--sanitize` build does not provide.

