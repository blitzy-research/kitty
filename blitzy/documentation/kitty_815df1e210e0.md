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

Three builds of the C extension `kitty.fast_data_types` were produced. Because `kitty/fast_data_types.so` is git-ignored, swapping between them never dirties the working tree. Each build prints 122 compile steps then five link steps; the two blocks below are **excerpts** — the first compile step and the link tail are shown verbatim and the middle is elided with an explicit `[... 122 compile steps ...]` marker (both builds exited 0):

```
$ python3 setup.py build --debug --extra-logging=event-loop   # == make debug-event-loop (Makefile:25-26)
[1/122] Compiling kitty/screen.c ...
[... 122 compile steps ...]
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done

$ python3 setup.py build --debug --sanitize                   # == make asan (Makefile:29-30)
[1/122] Compiling kitty/screen.c ...
[... 122 compile steps ...]
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
```

`--extra-logging=event-loop` compiles in the event-loop tracing (`-DDEBUG_EVENT_LOOP`, wired at `` `setup.py:488-489` `` → `cppflags.append('-DDEBUG_{}'.format(el.upper().replace('-', '_')))`), enabling the `EVDBG(...)` sites at `` `kitty/child-monitor.c:872` ``, `` `kitty/child-monitor.c:1217` ``, and `` `kitty/child-monitor.c:1225` ``. `--sanitize` adds `-fsanitize=address,undefined` (`` `setup.py:380` ``). A plain release build (`-DNDEBUG -O3`) is used where realistic timing/memory numbers matter; it is the build active for every measurement below **except** the event-loop trace of §2.3 (debug-event-loop) and the sanitizer run of §4.7 (asan).

All observation scripts referenced below live under `/tmp/obs/` (never inside the repository) and were run from the repository root with the environment's Python 3.11 (shown as `python3`); each script begins with `sys.path.insert(0, '.')` so it imports the in-tree `kitty` package and the `kitty_tests` helpers (`Callbacks`, `parse_bytes`). They were deleted after capture (see the coverage pass in §5).

The citations in this document were re-verified in-checkout; a few AAP line numbers were slightly off and the *observed* value is used throughout. Sample proof (note that `grep` substring-matches `class Clipboard` against `ClipboardType` and `ClipboardRequestManager`, and `as_text_for_history_buf` also appears in the method table):

```
$ grep -n "class Clipboard\|def create_chunker\|def encode_osc52\|def ask_to_read_clipboard" kitty/clipboard.py
52:    def create_chunker(self, offset: int, size: int) -> Callable[[], Callable[[], bytes]]:
72:class ClipboardType(IntEnum):
82:class Clipboard:
222:def encode_osc52(loc: str, response: str) -> str:
332:class ClipboardRequestManager:
518:    def ask_to_read_clipboard(self, rr: ReadRequest) -> None:

$ grep -n "as_text_for_history_buf" kitty/screen.c
3495:as_text_for_history_buf(Screen *self, PyObject *args) {
4827:    MND(as_text_for_history_buf, METH_VARARGS)
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
$ python3 -c "import kitty.fast_data_types as f; print('VT_PARSER_BUFFER_SIZE =', f.VT_PARSER_BUFFER_SIZE); print('VT_PARSER_BUFFER_SIZE == 1024*1024 :', f.VT_PARSER_BUFFER_SIZE == 1024*1024)"
VT_PARSER_BUFFER_SIZE = 1048576
VT_PARSER_BUFFER_SIZE == 1024*1024 : True
```

So the whole cross-thread pipe is exactly **`1048576`** bytes (1 MiB), and `MAX_ESCAPE_CODE_LENGTH = BUF_SZ / 4u` = **262144** bytes (`` `kitty/vt-parser.c:21` ``; exposed as `VT_PARSER_MAX_ESCAPE_CODE_SIZE`, confirmed `= 262144` in Section 1) is the threshold past which an *unterminated* escape is either chunked (OSC 52 → partial callbacks) or rejected as `escape code too long ... ignoring it` (`` `kitty/vt-parser.c:419` ``) — a *complete* (terminator-delimited) escape is accepted **whole regardless of size**, so this is a floor for handling unterminated overflow, not a hard cap on escape length (see §1.3).

**The GIL boundary.** Only the thread that holds CPython's Global Interpreter Lock may run Python bytecode or call the CPython C-API. The main thread holds the GIL whenever it materializes C data into Python objects or dispatches a callback. The I/O thread does **not** need the GIL to append bytes to the 1 MiB buffer. This asymmetry is the key to Q2a: a long C-side loop on the main thread occupies that thread and therefore **defers** kitty's own Python-level dispatch — which runs on the *same* main thread — while byte **ingestion** on the I/O thread keeps going. There are **two** reasons the dispatch is deferred, and both hold on kitty's real extraction path: (1) *same-thread serialization* — dispatch and the scan are the same thread's work, so the event loop cannot start its next iteration until the scan returns; and (2) *GIL monopoly* — the scan's per-line callback on the production path is the **built-in** `list.append` (`` `kitty/window.py:377` ``/`` `394` ``/`` `459` ``), a C method that never re-enters the bytecode eval loop, so the C loop holds the GIL for essentially the whole scan and even a *separate* Python thread is starved (measured in §2.1: the competitor's max-gap ≈ the whole scan duration on the production path). Byte ingestion continues regardless, because it happens on the I/O thread and needs no GIL. This is demonstrated directly in Section 2.

The data-movement architecture the rest of this document explains:

```
child PTY bytes
      │  (I/O thread: io_loop @ kitty/child-monitor.c:291 — no GIL needed to read)
      ▼
shared VT-parser buffer  self->buf  BUF_SZ = 1 MiB  (kitty/vt-parser.c:18)
      │  (main thread: parse_input @ kitty/child-monitor.c:451/1236)
      ▼
consume_input  ── parser lock RELEASED around this step (kitty/vt-parser.c:1431-1433)
      │
      ├── in-process selection/scrollback text ──► PyUnicode_FromKindAndData per line (kitty/line.c:278, non-ANSI default)  ──► Python str
      │
      └── OSC 52 ──► clipboard_control callback (kitty/screen.c:2306) ──► Python Boss / Clipboard
                                                                        │
                                    small: stays in io.BytesIO ─────────┤
                                    large: rolls over to TemporaryFile ─┘ (kitty/clipboard.py:32-35)
                                                                        │
                                    ──► out-of-process clipboard kitten over OSC 52 (kittens/clipboard/main.py)
                                    ──► OS clipboard via GLFW _glfwSendClipboardText (glfw/wl_window.c:2034)
```

---

# Section 1 — Q1: Clipboard transfer C→Python, small vs large, under concurrency

Clipboard data reaches Python two different ways, and the two "legs" behave very differently for small vs. large payloads.

## 1.1 The in-process leg — screen/scrollback text materialized as Python `str`

When a kitten (or the pager, or a paste) asks for on-screen or scrollback text, the C core walks each line and builds a Python `str` for it. Extraction is driven by the per-line loop `as_text_generic` (`` `kitty/line.c:874` ``), which calls the caller's Python callback once per line chunk (`PyObject_CallFunctionObjArgs(callback, x, NULL)`). Which materialization primitive it reaches depends on whether ANSI (SGR) formatting was requested — there are **two branches**, and both end in `PyUnicode_FromKindAndData`:

- **Default (non-ANSI) branch.** `as_text_generic` (`` `kitty/line.c:874` ``) calls `line_as_unicode` (`` `kitty/line.c:282` ``), which delegates to `unicode_in_range` (`` `kitty/line.c:253` ``), whose `return PyUnicode_FromKindAndData(PyUnicode_4BYTE_KIND, buf, n);` at `` `kitty/line.c:278` `` produces the `str`. This is the path taken by a plain-text extraction (the pager, a plain-text copy, most kitten reads).
- **ANSI branch.** When `as_ansi` is requested, `as_text_generic` instead calls `line_as_ansi` (`` `kitty/line.c:338` ``) to render the line (with SGR escapes) into a reusable `ANSIBuf`, then builds the `str` from that buffer at `` `kitty/line.c:900` `` → `t = PyUnicode_FromKindAndData(PyUnicode_4BYTE_KIND, ansibuf->buf, ansibuf->len);`.
- The standalone `Line.as_ansi` convenience method is a third, single-line entry to the same ANSI renderer: `as_ansi` (`` `kitty/line.c:416` ``) → `line_as_ansi` (`` `kitty/line.c:338` ``) → `PyUnicode_FromKindAndData` at `` `kitty/line.c:421` ``.

The same loop is reached for the live screen via `LineBuf.as_text` (`` `kitty/line-buf.c:490` `` → `as_text_generic(args, self, get_line, self->ynum, &output, false)` at `` `kitty/line-buf.c:492` ``), and for scrollback via the `Screen` method `as_text_for_history_buf` (`` `kitty/screen.c:3495` `` → `return as_text_history_buf(self->historybuf, args, &self->as_ansi_buf);` at `` `kitty/screen.c:3496` ``; declared `` `kitty/screen.h:254` ``).

Every one of those `str` objects is created **on the main thread while holding the GIL** — because `PyUnicode_FromKindAndData` and `PyObject_CallFunctionObjArgs` are CPython C-API calls. Both branches are observable directly, driving a `Screen`/`HistoryBuf` with 100 lines of `line-%05d`; each callback argument is a Python `str`:

```
$ python3 /tmp/obs/exp_line.py
non-ANSI  as_text_for_history_buf: chunks=154 types=['str'] first='line-00000'
ANSI      as_text_for_history_buf: chunks=231 types=['str'] first='\x1b[m'
Line.as_ansi() -> type=str value='line-00077'
```

Interpretation: the default (non-ANSI) extraction produced **154** `str` chunks (77 scrollback lines, each yielding a text chunk plus a newline chunk), first chunk `'line-00000'` — materialized at `` `kitty/line.c:278` ``. The ANSI extraction produced **231** chunks whose first chunk is the SGR reset `'\x1b[m'` — materialized at `` `kitty/line.c:900` ``. The standalone `Line.as_ansi()` returned a single `str` `'line-00077'` — materialized at `` `kitty/line.c:421` ``. All three report `type=str`, confirming the C→Python conversion in each branch. A second measurement drives the *same default path* at 100 lines and reports timing and memory (each `chunk` is one materialized `str`):

```
$ python3 /tmp/obs/exp2_one.py 100
n=100 historybuf_count=77 fill_RSS_delta_KB=280 scan_ms=0.037 scan_RSS_delta_KB=12 chunks=154 chars=3542
```

A 100-line feed left `historybuf.count = 77` lines in scrollback, and extracting them produced **154** Python string chunks totaling **3542** characters in **0.037 ms** — the in-process C→Python conversion via the default `line_as_unicode` path (`` `kitty/line.c:278` ``). (`fill_RSS_delta_KB`/`scan_RSS_delta_KB` are per-run resident-memory deltas and vary slightly between runs; `historybuf_count`, `chunks`, and `chars` are deterministic. The same path at large scale is Sections 2 and 3.)

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
$ python3 /tmp/obs/exp4_osc52.py | sed -n '1,4p'   # the SMALL section of the one run (it prints both)
=== SMALL OSC 52 (complete, one dispatch) ===
clipboard_control fired 1 time(s):
  call 0: is_partial=False  data='c;aGVsbG8gY2xpcGJvYXJk'
  decoded final payload = b'hello clipboard'
```

Interpretation: the escape `\x1b]52;c;aGVsbG8gY2xpcGJvYXJk\x07` produced exactly **one** `clipboard_control` call with `is_partial=False` (i.e. C `code == 52`), carrying the literal payload `c;aGVsbG8gY2xpcGJvYXJk` whose base64 decodes to `b'hello clipboard'` — the small clipboard payload crossed from the C parser into a Python object in a single dispatch (`` `kitty/screen.c:2306` ``).

## 1.3 Small → large divergence #1: the OSC 52 payload is *chunked* by the parser

A large OSC 52 does **not** always arrive as one Python object — but the split point is the **1 MiB parser buffer**, not `MAX_ESCAPE_CODE_LENGTH`. The parser first calls `find_st_terminator` (`` `kitty/vt-parser.c:397-404` ``): if the *complete* escape (its `ST`/`BEL` terminator included) is already in the buffer, it is dispatched **whole regardless of size** — the source comment there says to "be generous ... we have a full escape code". Only when the escape is still **unterminated** *and* the accumulation has exceeded `MAX_ESCAPE_CODE_LENGTH` (`BUF_SZ / 4u` = 262144) does the parser fall through to the **partial**-chunk branch and continue. Because the write buffer is `BUF_SZ` (`1048576`) bytes, an unterminated OSC 52 only reaches that branch once it has filled ~1 MiB:

```
$ sed -n '397,414p' kitty/vt-parser.c
    if (find_st_terminator(self, &pos)) {
        // technically we should check MAX_ESCAPE_CODE_LENGTH here but lets be generous in what we accept since  we
        // have a full escape code
        uint8_t *buf = self->buf + self->read.consumed;
        size_t sz = pos - self->read.consumed;
        buf[sz] = 0;  // ensure null termination, this is anyway an ST termination char
        dispatch(self, buf, sz, false);
        return true;
    }
    if (UNLIKELY((pos=self->read.pos - self->read.consumed) > MAX_ESCAPE_CODE_LENGTH)) {
        if (self->vte_state == VTE_OSC && is_osc_52(self)) {
            // null terminate
            self->read.pos--;
            uint8_t before = self->buf[self->read.pos];
            self->buf[self->read.pos] = 0;
            // send partial OSC 52
            dispatch(self, self->buf + self->read.consumed, self->read.pos - self->read.consumed, true);
            // continue OSC 52
```

The `true` on `` `kitty/vt-parser.c:413` `` is the `is_partial` flag that becomes C `code == -52` and thus Python `is_partial=True` at `` `kitty/screen.c:2306` ``. Observed with a 1 MiB payload:

```
$ python3 /tmp/obs/exp4_osc52.py | sed -n '5,8p'   # the LARGE section of the same run
=== LARGE OSC 52 (payload > VT_PARSER_MAX_ESCAPE_CODE_SIZE=262144) ===
total escape sequence length = 1398112 bytes
clipboard_control fired 2 time(s): 1 partial (is_partial=True), 1 final (is_partial=False)
first callback is_partial = True | last callback is_partial = False
```

Interpretation: a **1398112**-byte OSC 52 sequence crossed into Python as **two** callbacks — first `is_partial=True`, then a final `is_partial=False`. The partial is emitted only because the escape is still *unterminated when the 1 MiB write buffer fills*: once the accumulation overflows `BUF_SZ` (`1048576`) the parser dispatches a **1048570**-byte partial chunk (the partial branch at `` `kitty/vt-parser.c:406-414` ``) and continues. Shorter *complete* escapes never take that branch — `find_st_terminator` (`` `kitty/vt-parser.c:397-404` ``) dispatches them whole regardless of size — so `262144` (`MAX_ESCAPE_CODE_LENGTH`) is the branch **floor**, *not* the observed trigger point. A boundary sweep across both thresholds makes the real trigger explicit: escapes up to **1048544** bytes (well past `262144`) still fire **one** complete callback, and the first partial appears only at **1048580** bytes, just past `BUF_SZ`:

```
$ /opt/kitty-venv/bin/python -c '
import sys, base64
sys.path.insert(0, ".")
from kitty.fast_data_types import Screen, VT_PARSER_BUFFER_SIZE as BUF, VT_PARSER_MAX_ESCAPE_CODE_SIZE as MAXE
from kitty_tests import Callbacks, parse_bytes
def probe(raw):
    esc = b"\x1b]52;c;" + base64.standard_b64encode(b"A"*raw) + b"\x07"
    cb = Callbacks(); s = Screen(cb, 5, 5, 5, 10, 20, 0, cb); parse_bytes(s, esc)
    npart = sum(1 for c in cb.cc_buf if c[1])
    psz = next((len(c[0]) for c in cb.cc_buf if c[1]), 0)
    return len(esc), len(cb.cc_buf), npart, psz
print("BUF_SZ =", BUF, "| MAX_ESCAPE_CODE_LENGTH =", MAXE)
print("escape_total  callbacks  partials  partial_bytes")
for raw in (196602, 786000, 786400, 786427, 1048576):
    et, n, np_, ps = probe(raw)
    print("%-12d  %-9d  %-8d  %d" % (et, n, np_, ps))
'
BUF_SZ = 1048576 | MAX_ESCAPE_CODE_LENGTH = 262144
escape_total  callbacks  partials  partial_bytes
262144        1          0         0
1048008       1          0         0
1048544       1          0         0
1048580       2          1         1048570
1398112       2          1         1048570
```

Small data = one complete dispatch; large data past the 1 MiB buffer = a stream of partials + a final. The `MAX_ESCAPE_CODE_LENGTH` constant that gates that partial branch is itself exposed to Python and matches its source definition (`BUF_SZ / 4u`):

```
$ python3 -c "import kitty.fast_data_types as f; print('VT_PARSER_MAX_ESCAPE_CODE_SIZE =', f.VT_PARSER_MAX_ESCAPE_CODE_SIZE); print('== BUF_SZ/4 =', 1048576 // 4)"
VT_PARSER_MAX_ESCAPE_CODE_SIZE = 262144
== BUF_SZ/4 = 262144
```

## 1.4 Small → large divergence #2: in-memory buffer rolls over to an on-disk temp file

On the Python side, the reassembled clipboard bytes are accumulated in a `Tempfile` that starts in RAM and **spills to disk** once it crosses a size threshold:

```
$ sed -n '26,40p' kitty/clipboard.py
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
$ python3 /tmp/obs/exp3_rollover.py
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

The clipboard *kitten* is a **separate process** that speaks OSC 52 over its own PTY. Its real CLI surface, captured from the built `kitten` binary. This is an **excerpt** of the full `--help`: every line shown is verbatim, and the two elisions (a block of filename/MIME usage examples, and the tail of three long option descriptions) are marked explicitly with `[... elided ...]`:

```
$ kitty/launcher/kitten clipboard --help
Usage: kitten clipboard [options] [files to copy to/from]

Read or write to the system clipboard.

This kitten operates most simply in filter mode. To set the clipboard text, pipe
in the new text on STDIN. Use the --get-clipboard option to instead output the
current clipboard text content to STDOUT. Note that copying from the clipboard
will cause a permission popup, see clipboard_control for details.

[... block of filename/MIME copy-paste usage examples elided ...]

Options:
  --get-clipboard, -g
    Output the current contents of the clipboard to STDOUT. Note that by default
    kitty will prompt for permission to access the clipboard. Can be controlled
    by clipboard_control.

  --use-primary, -p
    Use the primary selection rather than the clipboard on systems that support
    it, such as Linux.

  --mime, -m
    The mimetype of the specified file. [... description elided ...]

  --alias, -a
    Specify aliases for MIME types. [... description elided ...]

  --wait-for-completion
    Wait till the copy to clipboard is complete before exiting. Useful if
    running the kitten in a dedicated, ephemeral window. Only needed in filter
    mode.

  --help, -h
    Show help for this command

kitten clipboard 0.35.2 created by Kovid Goyal
```

These match the option definitions at `` `kittens/clipboard/main.py:7` `` (`--get-clipboard -g`), `` `kittens/clipboard/main.py:14` `` (`--use-primary -p`), `` `kittens/clipboard/main.py:20` `` (`--mime -m`), `` `kittens/clipboard/main.py:31` `` (`--alias -a`), and `` `kittens/clipboard/main.py:43` `` (`--wait-for-completion`). The wire format it uses is OSC 52, which `encode_osc52` (`` `kitty/clipboard.py:222` ``) builds:

```
$ python3 -c "from kitty.clipboard import encode_osc52; r = encode_osc52('c', 'hello-from-kitten'); print(\"encode_osc52('c', 'hello-from-kitten') =\", repr(r)); print('bytes =', r.encode('ascii'))"
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
$ sed -n '1477,1482p' kitty/vt-parser.c
vt_parser_has_space_for_input(const Parser *p) {
    PS *self = (PS*)p->state;
    bool ans;
    with_lock {
        ans = self->read.sz + self->write.pending < BUF_SZ;
    } end_with_lock;
```

`` `kitty/vt-parser.c:1481` `` (`ans = self->read.sz + self->write.pending < BUF_SZ`) is the exact back-pressure test; input is also batched by the `input_delay` option at `` `kitty/vt-parser.c:1425` ``. The practical consequence — that a busy main thread defers *dispatch* while the I/O thread keeps *ingesting* — is measured directly in Section 2. A combined "busy" workload (large scan + small/large OSC 52 + a concurrent feeder thread) was also run end-to-end under the sanitizer without incident; see Section 4.

## 1.7 Reading the clipboard is gated by an ask/allow permission policy

The OSC 52 leg is asymmetric: a program can *write* the clipboard fairly freely, but *reading* it is gated. When a program requests a clipboard read, `ClipboardRequestManager.handle_read_request` (`` `kitty/clipboard.py:450` ``) consults the `clipboard_control` option (`` `kitty/clipboard.py:451` ``) and picks an *ask* flag and an *allow* flag depending on whether the primary selection or the clipboard is targeted:

```
$ sed -n '450,461p' kitty/clipboard.py
    def handle_read_request(self, rr: ReadRequest) -> None:
        cc = get_options().clipboard_control
        if rr.is_primary_selection:
            ask_for_permission = 'read-primary-ask' in cc
            allowed = 'read-primary' in cc
        else:
            ask_for_permission = 'read-clipboard-ask' in cc
            allowed = 'read-clipboard' in cc
        if ask_for_permission:
            self.ask_to_read_clipboard(rr)
        else:
            self.fulfill_read_request(rr, allowed=allowed)
```

So the clipboard branch tests `'read-clipboard-ask'` (`` `kitty/clipboard.py:456` ``) and `'read-clipboard'` (`` `kitty/clipboard.py:457` ``); the primary-selection branch tests `'read-primary-ask'` (`` `kitty/clipboard.py:453` ``) and `'read-primary'` (`` `kitty/clipboard.py:454` ``). If the *ask* flag is present control goes to `ask_to_read_clipboard`; otherwise the request is fulfilled or refused according to the *allow* flag. The shipped default enables **ask** for both, which I read directly from the compiled defaults and route through the same predicates:

```
$ python3 /tmp/obs/exp_clip_perm.py
default clipboard_control = ('write-clipboard', 'write-primary', 'read-clipboard-ask', 'read-primary-ask')
policy=('read-clipboard-ask',) [default] -> ask_to_read_clipboard(rr)  -> confirmation prompt (clipboard.py:518)
policy=('read-clipboard',)     [allow]   -> fulfill_read_request(allowed=True)   -> encode_osc52(text) sent
policy=()                      [deny]    -> fulfill_read_request(allowed=False)  -> encode_response(status='EPERM') (clipboard.py:474)
```

(The first line is the real `defaults.clipboard_control` tuple read from `kitty.options.types`; the `route()` helper mirrors the `handle_read_request` predicates `'read-clipboard-ask' in cc` / `'read-clipboard' in cc` at `` `kitty/clipboard.py:456` ``–`457` against sample policies.) The three outcomes are:

- **Ask (the default).** `ask_to_read_clipboard` (`` `kitty/clipboard.py:518` ``) raises a confirmation prompt via `get_boss().confirm(...)` (`` `kitty/clipboard.py:528` ``) with the exact text `A program running in this window wants to read from the system clipboard. Allow it to do so, once?` (`` `kitty/clipboard.py:529` ``); the answer is handled by `handle_clipboard_confirmation` (`` `kitty/clipboard.py:534` ``). If a prompt is already outstanding (`currently_asking_permission_for is not None`) the new request is rejected via `reject_read_request` (`` `kitty/clipboard.py:501` ``), which replies `EPERM` (`` `kitty/clipboard.py:506` ``).
- **Allow.** `fulfill_read_request` (`` `kitty/clipboard.py:463` ``) sends the clipboard text back with status `OK`.
- **Deny.** `fulfill_read_request` replies `EPERM` (`` `kitty/clipboard.py:474` ``) when the allow flag is absent; if the targeted selection is disabled entirely it replies `ENOSYS` (`` `kitty/clipboard.py:471` ``).

This is the security-relevant gate on the C→Python→child clipboard-read path: the same `clipboard_control` callback of §1.2 that carries an OSC 52 *read* request lands here, and by default it will **not** silently return clipboard contents — it prompts, and refuses with `EPERM` when disallowed. The write direction is governed symmetrically by `'write-clipboard'` / `'write-primary'` in `clipboard_control`, checked at `` `kitty/clipboard.py:430` `` with `EPERM` / `ENOSYS` at `` `kitty/clipboard.py:442` ``. The interactive confirmation dialog itself needs a live window and was not exercised headlessly — that limitation is stated rather than asserted.


---

# Section 2 — Q2a: Event delivery to kittens during an expensive scrollback scan

**Yes — an expensive scrollback scan defers event delivery to kittens. What I measured *directly* is (a) that the scan occupies the main thread for its full duration, and (b) that on kitty's real extraction path — whose per-line callback is the built-in `list.append` — a competing Python thread is starved for almost the entire scan, i.e. the GIL *is* effectively monopolized (competitor max-gap ≈ scan duration). The GIL result is callback-shape dependent: only an *artificial* Python `def` callback lets the competitor run at the ~5 ms switch interval, and that shape is not what kitty uses. The combined single-capture "this exact scan blocks this exact dispatch" line could not be staged headlessly; that limitation is stated in the honesty note at the end of §2.3. The mechanism below is therefore drawn from the direct measurements plus the source paths that connect them.**

The mechanism has **two** parts, and on kitty's real extraction path **both** hold. First, **same-thread serialization**: escape-code dispatch to kittens is Python work that runs **on the main thread** inside `parse_input` (`` `kitty/child-monitor.c:451` ``), which is driven by the main-thread event loop `main_loop` (`` `kitty/child-monitor.c:1259` ``; documented as "The main thread loop" at `` `kitty/child-monitor.c:1260` ``). Extracting a large scrollback (`as_text_for_history_buf` over a big `HistoryBuf`) is a long operation that **also** runs on the main thread. Because dispatch and the scan are the *same thread's* work, the event loop cannot reach its next `parse_input` iteration until the scan returns — so dispatch is **postponed**. Second, **GIL monopoly on the production path**: the scan invokes one callback per line (`PyObject_CallFunctionObjArgs` at `` `kitty/line.c:875` ``, via the `APPEND`/`APPEND_AND_DECREF` macros in the per-line loop of `as_text_generic`, `` `kitty/line.c:874` ``), and in production that callback is the **built-in** `list.append` (`h.append`/`lines.append` at `` `kitty/window.py:394` ``/`` `377` ``/`` `459` ``). A built-in C method executes entirely in C and never re-enters the CPython bytecode eval loop, so the interpreter's thread-switch check (the "eval breaker") is never reached during the scan — the C loop therefore holds the GIL for essentially the whole scan, and a *separate* Python thread is starved too. (Only an *artificial* Python-`def` callback re-enters the eval loop and lets other threads run at the switch interval — see the callback-shape comparison in §2.1.)

## 2.1 Direct measurement: the scan defers the main thread and, on the production path, monopolizes the GIL

To make the deferral visible I run the expensive scan on the main thread while a **competing Python thread** spins and records the **largest gap** between its own iterations. If the scan monopolizes the GIL, that gap is as long as the whole scan; if it does not, the gap stays near CPython's thread-switch interval and the competitor keeps iterating. The result depends on the **shape of the per-line callback**, so I measure all three: the built-in `list.append` that kitty actually uses (`` `kitty/window.py:394` ``), an artificial Python `def`, and the real `kitty.window.as_text(add_history=True)` API:

```
$ python3 /tmp/obs/exp5_defer_v2.py
historybuf.count = 399977
sys.getswitchinterval() = 5.000 ms
direct_list_append_callback [PRODUCTION shape: kitty/window.py:394 h.append]
    baseline competitor max-gap (main idle)  = 0.04 ms
    main-thread scan duration                = 172.65 ms
    competitor max-gap DURING scan           = 159.73 ms
    competitor iterations DURING scan        = 41660
    scan result (lines/chars)                = 799954
direct_python_function_callback [artificial Python def]
    baseline competitor max-gap (main idle)  = 0.03 ms
    main-thread scan duration                = 384.07 ms
    competitor max-gap DURING scan           = 8.11 ms
    competitor iterations DURING scan        = 768673
    scan result (lines/chars)                = 799954
kitty.window.as_text(add_history=True) [real production API]
    baseline competitor max-gap (main idle)  = 0.02 ms
    main-thread scan duration                = 202.20 ms
    competitor max-gap DURING scan           = 151.86 ms
    competitor iterations DURING scan        = 61424
    scan result (lines/chars)                = 18399999
```

Interpretation: the main-thread scan of a **399977**-line scrollback ran for **172.65 ms** on the production `list.append` path — that is how long kitty's main-thread event loop (and therefore its same-thread dispatch to kittens) is deferred. On that path the competing Python thread was **starved for almost the entire scan**: its largest gap during the scan was **159.73 ms** — essentially the whole **172.65 ms** — and it managed only **41660** iterations. The real `kitty.window.as_text(add_history=True)` API behaves identically (scan **202.20 ms**, competitor max-gap **151.86 ms**). So on kitty's actual extraction path the GIL **is** effectively monopolized: the per-line callback is the built-in `list.append`, a C method that never re-enters the bytecode eval loop, so the C loop `as_text_generic` (`` `kitty/line.c:874` ``) calling `PyObject_CallFunctionObjArgs` (`` `kitty/line.c:875` ``) holds the GIL for the whole scan. Only the **artificial** Python-`def` callback tells a different story — there the competitor's max-gap was **8.11 ms** (near the **5.000 ms** `sys.getswitchinterval()`) with **768673** iterations, because a Python function re-enters the eval loop each call and gives the interpreter a thread-switch point. That artificial shape is *not* what kitty uses, so it does not describe production behavior. Either way, event delivery to kittens is deferred because kitty performs that delivery on the **same main thread** as the scan, in `parse_input` (`` `kitty/child-monitor.c:451` ``), not on a separate thread. (Scan durations, max-gaps, and iteration counts are timing-dependent and vary run to run — a second run gave `list.append` scan **145.82 ms** / max-gap **132.92 ms** and Python-`def` max-gap **11.32 ms** — but the *relationship* is stable: the built-in-callback max-gap tracks the scan duration, while the Python-`def` max-gap stays near the switch interval. `historybuf.count` = 399977, the 5.000 ms switch interval, and the **799954** / **18399999** result counts are stable.)

## 2.2 What keeps making progress: the I/O thread and the shared buffer

Ingestion is *not* blocked by the scan, because reading child bytes into the 1 MiB buffer is done by the I/O thread and needs no GIL (`io_loop` `` `kitty/child-monitor.c:291` ``; `read_bytes` `` `kitty/child-monitor.c:1337` ``), and because the parser drops its lock around the consume step (`` `kitty/vt-parser.c:1431-1433` ``, shown in §1.6). So during a long main-thread scan: **bytes keep arriving** into `self->buf` (up to the `BUF_SZ` = 1 MiB back-pressure limit at `` `kitty/vt-parser.c:1481` ``), but **their dispatch to kittens waits** for the scan to finish. Ingestion continues; delivery is deferred.

## 2.3 The event loop itself, observed

Running the real kitty binary under `xvfb-run` with the `--extra-logging=event-loop` build emits the `EVDBG` trace. `EVDBG` expands to `timed_debug_print` (`` `kitty/child-monitor.c:30` ``), implemented at `` `kitty/monotonic.h:99` ``, which prints `[<seconds>] <message>` to stderr, where `<seconds>` is a monotonic offset from the first call. `timed_debug_print` is also exposed to Python (it is compiled into the release build, guarded by `MONOTONIC_IMPLEMENTATION` rather than `DEBUG_EVENT_LOOP`), so the exact log-line format can be reproduced directly by calling it with the three real `EVDBG` message strings. The script (`/tmp/obs/exp_timedprint.py`):

```
import sys; sys.path.insert(0, '.')
import kitty.fast_data_types as f
f.timed_debug_print("input_read: 1, check_for_active_animated_images: 0\n")
f.timed_debug_print("Processing global state\n")
f.timed_debug_print("State check timer fired\n")
```

Each printed line is prefixed with a `[<seconds>]` monotonic offset whose exact digits vary run to run (a representative raw capture is `[0.000] input_read: 1, check_for_active_animated_images: 0`, then `[0.001] Processing global state`, then `[0.001] State check timer fired`; each message ends in `\n`, which is why each gets its own timestamp — see `` `kitty/monotonic.h:99` ``). To show the three message strings *deterministically*, the volatile prefix is stripped with `sed`; this pipeline reproduces byte-for-byte on every run:

```
$ python3 /tmp/obs/exp_timedprint.py 2>&1 | sed -E 's/^\[[0-9.]+\] //'
input_read: 1, check_for_active_animated_images: 0
Processing global state
State check timer fired
```

These three strings are literally the `EVDBG` sites at `` `kitty/child-monitor.c:872` `` (`"input_read: %d, check_for_active_animated_images: %d"`), `` `kitty/child-monitor.c:1225` `` (`"Processing global state"`), and `` `kitty/child-monitor.c:1217` `` (`"State check timer fired"`). Now the **real** event loop of a running kitty (child emitting output). The debug-event-loop build was run **once** under `xvfb-run`, capturing stderr to a log file (that capture is timing-dependent and one-time); the individual event lines are then extracted from the saved log with a deterministic `grep`:

```
$ xvfb-run -a kitty/launcher/kitty --config NONE -o scrollback_lines=100000 \
      sh -c 'seq 1 200000; echo DONE_PRODUCING; sleep 2'  2> /tmp/kitty_eventloop_raw.log
$ grep -oE 'input_read: [0-9]+, check_for_active_animated_images: [0-9]|Processing global state|State check timer fired' \
      /tmp/kitty_eventloop_raw.log | head -12
State check timer fired
Processing global state
input_read: 0, check_for_active_animated_images: 1
Processing global state
input_read: 1, check_for_active_animated_images: 0
Processing global state
input_read: 1, check_for_active_animated_images: 0
Processing global state
input_read: 0, check_for_active_animated_images: 0
State check timer fired
Processing global state
input_read: 0, check_for_active_animated_images: 0
```

Interpretation: each main-loop iteration logs `Processing global state` (`` `kitty/child-monitor.c:1225` ``) and, when it serviced a child fd, an `input_read: N, check_for_active_animated_images: M` line (`` `kitty/child-monitor.c:872` ``) — `input_read: 1` when the child's `seq 1 200000` output was read and parsed **on the main thread**, `input_read: 0` when there was nothing to read that iteration. Across the whole saved log the extraction matched `Processing global state` **9** times, `input_read` events **9** times (**2** of them `input_read: 1`), and `State check timer fired` **5** times; the block above is the first **12** matches (`head -12`). This confirms that reading-and-dispatching input is a *main-loop, main-thread* activity — the same thread the scan monopolizes in §2.1 — which is exactly why a long scan defers it.

> **Honesty note.** I could not stage the two events *simultaneously in one capture* headlessly (a scrollback scan is triggered by GUI/kitten actions that need a live window, and the child-monitor loop only runs inside the full app). I therefore measured the two halves directly and quantitatively — the main-thread scan duration and the concurrent competitor thread in §2.1 (showing same-thread deferral *and*, on the production `list.append` path, GIL monopoly), and the real event-loop dispatch trace in §2.3 — and tied them together through the shared source paths (`parse_input` on the main thread, the released parser lock, the no-GIL I/O read). The combined single-capture "this scan blocks this exact `input_read`" line is *not* directly shown, and that limitation is stated rather than papered over.


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
$ for n in 100 50000 100000 200000 400000; do python3 /tmp/obs/exp2_one.py $n; done
n=100 historybuf_count=77 fill_RSS_delta_KB=280 scan_ms=0.037 scan_RSS_delta_KB=12 chunks=154 chars=3542
n=50000 historybuf_count=49977 fill_RSS_delta_KB=125316 scan_ms=22.248 scan_RSS_delta_KB=5728 chunks=99954 chars=2298942
n=100000 historybuf_count=99977 fill_RSS_delta_KB=250620 scan_ms=45.670 scan_RSS_delta_KB=11208 chunks=199954 chars=4598942
n=200000 historybuf_count=199977 fill_RSS_delta_KB=501208 scan_ms=85.468 scan_RSS_delta_KB=22160 chunks=399954 chars=9198942
n=400000 historybuf_count=399977 fill_RSS_delta_KB=1002380 scan_ms=171.418 scan_RSS_delta_KB=44380 chunks=799954 chars=18398942
```

Interpretation — memory grows **linearly** with scrollback size:

| lines fed | `historybuf.count` | `fill_RSS_delta_KB` | KB / line | `scan_RSS_delta_KB` |
|---:|---:|---:|---:|---:|
| 50000 | 49977 | 125316 | 2.51 | 5728 |
| 100000 | 99977 | 250620 | 2.51 | 11208 |
| 200000 | 199977 | 501208 | 2.51 | 22160 |
| 400000 | 399977 | 1002380 | 2.51 | 44380 |

Doubling the lines roughly doubles the resident memory (125316 → 250620 → 501208 → 1002380 KB), converging to ~**2.51 KB per stored line** — i.e. memory is `O(lines)`, exactly what a segmented `HistoryBuf` of `SEGMENT_SIZE 2048`-line blocks predicts (`` `kitty/history.c:15` ``, `` `kitty/history.c:18` ``). The **scan time** grows linearly too (22.248 → 45.670 → 85.468 → 171.418 ms; ~0.43 µs/line at 400000), so the small-vs-large contrast is stark: the 100-line scan took **0.037 ms** while the 200000-line scan took **85.468 ms** — a **~2310×** difference — and the resident footprint after fill went from **280 KB** to **501208 KB**. (RSS deltas are per-run measurements and vary slightly between runs; the `historybuf_count`, `chunks`, and `chars` columns are deterministic.)

There is a second, transient cost during the scan: materializing the lines into Python `str` objects. The `scan_RSS_delta_KB` column shows this — for 200000 lines the extraction itself added **22160 KB** on top of the scrollback storage, to hold **399954** string chunks (**9198942** chars). That is the C→Python `str` cost of the default `line_as_unicode` path (`` `kitty/line.c:278` ``) at scale, and it grows linearly with line count (5728 → 11208 → 22160 → 44380 KB).

The pager-history ringbuffer is separately capped rather than unbounded — its initial size is `MIN(1024u * 1024u, pagerhist_sz)` (`` `kitty/history.c:67` ``), i.e. at most 1 MiB up front.

## 3.2 Offload mechanisms that bound peak memory

kitty avoids holding very large payloads wholly in RAM in two concrete ways:

1. **Clipboard disk rollover.** As observed in §1.4, a clipboard `Tempfile` swaps from `io.BytesIO` to an on-disk `TemporaryFile()` once it exceeds `max_size` (default 16 MiB) — `` `kitty/clipboard.py:32-35` ``. Re-quoting the observed transition verbatim from §1.4 (produced by `$ python3 /tmp/obs/exp3_rollover.py`):

   ```
   after 1 MiB   type(tf2.file) = BytesIO | tell = 1048576 (<= 16 MiB: stays in RAM)
   after +16 MiB type(tf2.file) = BufferedRandom | tell = 17825792 (> 16 MiB: spilled to disk)
   rolled-over file has real OS fd?  True fileno = 4
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

## 4.4 Ownership seam #3 — child-object refcounting across `children_lock`

Each child's `screen` is a **Python object shared across threads**: the main thread parses into it under the GIL, while the child-monitor's add/remove bookkeeping mutates the `children` / `add_queue` / `remove_queue` arrays under `children_lock`. kitty keeps the `screen` alive across that boundary with explicit reference-count bracketing:

```
$ sed -n '105p;108,110p' kitty/child-monitor.c
#define FREE_CHILD(x) \
#define XREF_CHILD(x, OP) OP(x.screen);
#define INCREF_CHILD(x) XREF_CHILD(x, Py_INCREF)
#define DECREF_CHILD(x) XREF_CHILD(x, Py_DECREF)
```

`INCREF_CHILD` / `DECREF_CHILD` (`` `kitty/child-monitor.c:109` `` / `` `kitty/child-monitor.c:110` ``) are thin wrappers over `Py_INCREF` / `Py_DECREF` on `x.screen`; `FREE_CHILD` (`` `kitty/child-monitor.c:105` ``) does `Py_CLEAR((x).screen)`. The critical pattern is in `parse_input` (`` `kitty/child-monitor.c:451` ``): it takes the lock, drains the removal queue (incref'ing each into `remove_notify` at `` `kitty/child-monitor.c:460` `` and freeing the live slot at `` `kitty/child-monitor.c:462` ``), then **snapshots the live children into a private `scratch[]` array and increfs each**, and only then **releases the lock**:

```
$ sed -n '479,480p;483p;530p;532p' kitty/child-monitor.c
            scratch[i] = children[i];
            INCREF_CHILD(scratch[i]);
    children_mutex(unlock);
            if (do_parse(self, scratch[i].screen, now, false)) input_read = true;
        DECREF_CHILD(scratch[i]);
```

The snapshot copy is `scratch[i] = children[i]` at `` `kitty/child-monitor.c:479` `` with `INCREF_CHILD(scratch[i])` at `` `kitty/child-monitor.c:480` ``, and the lock is dropped at `` `kitty/child-monitor.c:483` ``. The actual parse — `do_parse(self, scratch[i].screen, now, false)` at `` `kitty/child-monitor.c:530` `` — then runs **outside** `children_lock`, against the incref'd snapshot, and each entry is released with `DECREF_CHILD(scratch[i])` at `` `kitty/child-monitor.c:532` `` once its parse completes. **Why this matters for object ownership across threads:** without that incref, a concurrent removal (`remove_children`, `` `kitty/child-monitor.c:1313` ``, which stages a child into `remove_queue`) followed by the next parse pass draining that queue could drop the last reference and `Py_CLEAR` the `screen` *while the main thread is still parsing into it* — a classic cross-thread use-after-free on a Python object. The incref'd private `scratch[]` snapshot guarantees the `screen` stays live for the entire unlocked parse window, and the matching decref hands ownership back so a pending removal can finalize on a later pass. Adds are symmetric: `add_child` (`` `kitty/child-monitor.c:305` ``) increfs into `add_queue` at `` `kitty/child-monitor.c:316` `` under the lock, and `add_children` (`` `kitty/child-monitor.c:1281` ``) moves those into the live `children` array. So the ownership seam is made safe by refcount *bracketing* — not by holding `children_lock` across the (potentially long) parse, which would serialize all children behind one slow scan.

## 4.5 Timing seam — the parser's lock-release window

The most subtle timing window is the one kitty opens *on purpose*: the parser releases its lock around `consume_input` so the I/O thread can append while the main thread consumes (`` `kitty/vt-parser.c:1431-1433` ``, quoted in §1.6). During that window the 1 MiB buffer is concurrently **appended** (I/O thread) and **consumed** (main thread). It is safe only because the read region and the write/pending region are disjoint and re-synchronized after the window (`self->read.sz += self->write.pending; self->write.pending = 0;` immediately after `with_lock`). This is the canonical "correct but delicate" seam: any change to the index bookkeeping here could turn the intended overlap into a data race. Back-pressure (`` `kitty/vt-parser.c:1481` ``) keeps the producer from lapping the consumer by refusing input once `read.sz + write.pending >= BUF_SZ`.

## 4.6 Bounds/edge guards that gate these paths

- **Write-to-child 100 MiB cap.** If pending writes would exceed 100 MiB, the data is dropped with a specific message:

  ```
  $ grep -n "100 \* 1024 \* 1024\|Too much data being sent" kitty/child-monitor.c
  341:                if (screen->write_buf_used + sz > 100 * 1024 * 1024) { \
  342:                    log_error("Too much data being sent to child with id: %lu, ignoring it", id); \
  $ python3 -c "v = 100*1024*1024; print(f'100 * 1024 * 1024 = {v} bytes = {v//(1024*1024)} MiB')"
  100 * 1024 * 1024 = 104857600 bytes = 100 MiB
  ```

  The cap is `screen->write_buf_used + sz > 100 * 1024 * 1024` (= **104857600** bytes) at `` `kitty/child-monitor.c:341` ``, and the exact log line is `Too much data being sent to child with id: %lu, ignoring it` at `` `kitty/child-monitor.c:342` ``. **Not triggered at runtime here (stated explicitly):** this guard lives inside the `schedule_write_to_child` macro, which iterates the `ChildMonitor`'s registered children and touches `screen->write_buf`; the headless `Screen`-only harness has no live `ChildMonitor` with a registered child and a >100 MiB backlog, so the warning was not provoked. It is cited from source, not asserted as observed.
- **History bounds-fatal.** An out-of-range scrollback index is fatal: `fatal("Out of bounds access to history buffer line number: %u", y);` at `` `kitty/history.c:40` `` — a hard guard on the very indexing the large scan of §2/§3 exercises.

## 4.7 Sanitizer pass — what it found (and its limits)

The `make asan` build (`-fsanitize=address,undefined`, `` `setup.py:380` ``) was run over a "busy" workload combining a large scrollback scan, small **and** large OSC 52 dispatch, and a **concurrent feeder thread** parsing input on another thread while the main thread scans (i.e. it exercises the lock-release window of §4.5). ASan/UBSan write any diagnostic to **stderr** and ASan aborts the process on a memory-safety fault, so a clean run is one that prints only the workload's own line and exits 0. Both streams were captured to `/tmp/asan_clean.log`, and the absence of diagnostics is then verified with `grep` so it is auditable rather than asserted:

```
$ LD_PRELOAD=/lib/x86_64-linux-gnu/libasan.so.8 ASAN_OPTIONS=detect_leaks=0 \
      python3 /tmp/obs/exp_asan_workload.py > /tmp/asan_clean.log 2>&1; echo "exit=$?"
exit=0
$ cat /tmp/asan_clean.log
WORKLOAD_COMPLETED_OK chunks=209954 cc=3 hb=104977
$ grep -cE 'AddressSanitizer|runtime error|SUMMARY:|heap-|stack-|use-after-' /tmp/asan_clean.log
0
```

Interpretation: the workload completed cleanly — exit **0**; **209954** materialized chunks (= 104977 scrollback lines × 2, a text chunk plus a newline chunk each); **3** `clipboard_control` callbacks (1 small + the 2 from the large partial+final of §1.3); `historybuf.count` **104977** — and the capture contains **0** AddressSanitizer/UBSan diagnostic lines. That means no memory-safety error (use-after-free, overflow) or undefined behavior was triggered on these paths.

> **Critical caveat, stated explicitly.** `--sanitize` enables **AddressSanitizer + UndefinedBehaviorSanitizer**, *not* ThreadSanitizer (`-fsanitize=address,undefined` at `` `setup.py:380` `` — there is no `thread`). ASan/UBSan do **not** detect data races. Therefore a clean run here does **not** prove the absence of the timing/ownership races discussed in §4.2–§4.5; it only shows no memory-safety/UB fault occurred. The race seams above are real *by construction of the code* (a deliberately released lock, a raw-allocator buffer touched off-GIL, a detached thread). They are documented, not patched, and confirming or refuting a race would require a ThreadSanitizer build, which this environment's `--sanitize` does not produce — flagged here rather than asserted either way.


---

# Section 5 — Coverage pass

Every distinct sub-part of the question, mapped to where it is answered and the key observed evidence.

| Sub-part of the question | Answered in | Key observed evidence (verbatim above) |
|---|---|---|
| **Q1** — Clipboard C→Python transfer, **small vs very large**, **when other parts are busy** | **§1** (+ §0 for vocabulary, §1.6 for concurrency) | In-process `str` via `PyUnicode_FromKindAndData` — default (non-ANSI) path `` `kitty/line.c:278` ``, ANSI path `` `kitty/line.c:900` `` — **154** non-ANSI / **231** ANSI chunks observed; OSC 52 → `clipboard_control` (`` `kitty/screen.c:2306` ``) small = 1 complete callback `b'hello clipboard'`; large **1398112**-byte OSC 52 = partial(`True`)+final(`False`); `Tempfile` rollover `BytesIO`→`BufferedRandom` at 16 MiB default (`` `kitty/clipboard.py:32-35` ``); kitten CLI + `encode_osc52('c',…)='52;c;aGVsbG8tZnJvbS1raXR0ZW4='`; clipboard *reads* gated by `clipboard_control` ask/allow (§1.7) — default asks via `ask_to_read_clipboard` (`` `kitty/clipboard.py:518` ``), else `EPERM` (`` `kitty/clipboard.py:474` ``) |
| **Q2a** — Does an expensive scrollback scan affect **event delivery to kittens**? | **§2** | Deferred by same-thread serialization **and**, on kitty's production extraction path (built-in `list.append`, `` `kitty/window.py:394` ``), GIL monopoly: a **172.65 ms** main-thread scan postpones main-loop dispatch that long while the competing Python thread is starved (max-gap **159.73 ms** ≈ the full scan, only **41660** iterations); `kitty.window.as_text` matches (**202.20 ms** / **151.86 ms**); only an artificial Python-`def` callback yields at the **5.000 ms** switch interval (max-gap **8.11 ms**) — not the production shape; dispatch runs on the main thread in `parse_input` (`` `kitty/child-monitor.c:451` ``) driven by `main_loop` (`` `kitty/child-monitor.c:1259` ``); per-line callback `PyObject_CallFunctionObjArgs` (`` `kitty/line.c:875` ``); real event-loop trace (`input_read: 1`, `Processing global state`); parser lock released around `consume_input` (`` `kitty/vt-parser.c:1431-1433` ``) |
| **Q2b** — Does it affect **memory management**? | **§3** | Linear RSS growth ~**2.51 KB/line** (125316→250620→501208→1002380 KB for 50k→400k lines) over segmented `HistoryBuf` (`` `kitty/history.c:15` ``); transient scan `str` cost `scan_RSS_delta_KB` 5728→11208→22160→44380 KB; offload via clipboard disk rollover (`` `kitty/clipboard.py:32-35` ``) and disk-cache `write_thread` (`` `kitty/disk-cache.c:397` ``) |
| **Q3** — **Where** do timing/concurrency/ownership matter; **how** do subtle races emerge? | **§4** | Detached write-helper's private `memcpy` copy (`` `kitty/child-monitor.c:1001` ``→`1002`→`1004`); child-object refcounting across `children_lock` (`INCREF_CHILD`/`DECREF_CHILD` `` `kitty/child-monitor.c:109-110` ``, snapshot `` `kitty/child-monitor.c:480` ``); raw-allocator `write_buf` off-GIL (`` `kitty/child-monitor.c:347` ``/`360`); the lock-release window (`` `kitty/vt-parser.c:1431-1433` ``); 100 MiB cap (`` `kitty/child-monitor.c:341-342` ``, not runtime-triggered); ASan/UBSan clean but **ASan≠TSan** (`` `setup.py:380` ``) |

**Explicitly flagged as not verified at runtime** (cited from source, not asserted as observed): the live out-of-process kitten OSC 52 round-trip and the OS-clipboard hand-off through GLFW/Wayland (`glfw/wl_window.c:2034`, `:2494`) — both need a live kitty GUI terminal / compositor (§1.5); the 100 MiB write-to-child cap warning (§4.6); and, because `--sanitize` is ASan+UBSan and not ThreadSanitizer, the *presence or absence of a data race* in the §4.5 lock-release window is not decided by the clean sanitizer run (§4.7).

## One-paragraph synthesis

In practice, clipboard and screen data cross from kitty's C core into Python on **one thread** — the main thread, under the GIL. Small data crosses in a single step (one `clipboard_control` callback, one in-memory `io.BytesIO`, a handful of `PyUnicode` strings); very large data is deliberately *fragmented and offloaded* — an unterminated OSC 52 is chopped into partial callbacks once it overflows the 1 MiB parser buffer (`BUF_SZ = 1048576`, a ~1048570-byte partial), and the reassembled bytes spill from RAM to an on-disk `TemporaryFile` past 16 MiB. When other parts of the system are busy, a separate I/O thread keeps reading child bytes into the shared 1 MiB buffer without needing the GIL, and the parser drops its lock so ingestion overlaps parsing — so **ingestion continues even while a long C-side scrollback scan occupies the main thread**. That scan (linear in line count: ~0.43 µs and ~2.51 KB per line) runs on the *same* main thread that kitty uses to dispatch events to kittens, which is precisely why *event delivery to kittens is deferred* while *memory grows linearly and is bounded only by the disk-offload paths* — and on kitty's real extraction path that deferral comes from **both** same-thread serialization **and** GIL monopoly: the per-line callback is the built-in `list.append`, a C method that never re-enters the bytecode eval loop, so the scan holds the GIL for its whole duration and a separate Python thread is starved too (competitor max-gap **159.73 ms** ≈ the **172.65 ms** scan; only an *artificial* Python-`def` callback lets another thread run at the ~5 ms switch interval, and that is not the shape kitty uses). The seams where this can go wrong are exactly the boundaries kitty engineers around: a deliberately released parser lock, a raw-allocator C buffer touched off-GIL, and a detached writer that copies its payload to sever Python ownership — real, delicate, and (on the paths exercised here) free of memory-safety faults, though a definitive race verdict would require ThreadSanitizer, which the available `--sanitize` build does not provide.

