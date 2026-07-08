# How kitty Moves Data Between Its C Core and Python Kittens Under Load

*A runtime-verified investigation of the clipboard transport, timing, concurrency, object ownership, and races in the kitty terminal emulator.*

Branch `kitty_815df1e210e0` — commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

---

## 1. Title & Summary

**Direct answer.** kitty runs a **multi-threaded core but a single-threaded Python layer**. Raw bytes from a child process are read on a dedicated **I/O thread** that never touches the Python C-API; all VT parsing, all C→Python callbacks, and all expensive operations such as scrollback scans run on **one main thread** while it holds the **CPython Global Interpreter Lock (GIL)**. When a clipboard escape (OSC 52 or the extended OSC 5522) arrives, the parser wraps its **internal 1 MiB buffer** in a **zero-copy, read-only `memoryview`** and hands that view to a Python method on the window object via a C→Python `CALLBACK`. The Python clipboard manager then **copies the bytes it needs out of that transient view into owned Python objects** (a `WriteRequest` backed by a `Tempfile` that begins as an in-memory `io.BytesIO` and rolls over to an on-disk `TemporaryFile` at 16 MiB). Because everything Python runs on the one GIL-holding main thread, an expensive main-thread operation (e.g. scanning a large scrollback) **delays but never loses** event delivery to kittens: the I/O thread keeps buffering raw bytes (subject to backpressure at the 1 MiB buffer limit) the entire time, and the queued events are delivered in order the moment the main thread is free again. Timing and concurrency therefore matter at exactly two places — the **parser lock** guarding the single producer/consumer buffer, and the **GIL** serializing the main thread — while object ownership matters at the **RAII-scoped lifetime of the `memoryview`**: the only real hazard is a *C-level* one (retaining that view until the parser reuses its buffer), which kitty avoids by copying out; it is **not** a Python-level data race, because the GIL serializes all Python execution.

Every system-specific claim below is backed by a `file:line` citation and/or complete, unedited runtime output captured from a canonical build. Anything not directly observed is explicitly labeled **(inferred)**. Cross-checks that bypass the real escape-code entry point are explicitly labeled **(non-canonical)**.

---

## 2. Build & Environment (canonical)

All observation was performed against a from-source build of kitty inside the designated container.

### 2.1 Exact interpreter / toolchain / OS observed

```
$ cat /etc/os-release | head -2
PRETTY_NAME="Ubuntu 25.10 (Questing Quokka)"
NAME="Ubuntu"

$ python3 --version         # venv at /opt/kitty-venv (source /opt/kitty-venv/bin/activate)
Python 3.13.7

$ go version
go version go1.24.4 linux/amd64

$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ git rev-parse --abbrev-ref HEAD
blitzy-e887c911-d451-47bd-84e6-41695624502a
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

> **Note on stated versions.** The interpreter actually used is **Python 3.13.7** (not 3.12.3, which appears as a guess in the task's evidence base). `pyproject.toml:2` requires `>=3.8`; `go.mod:3` pins `go 1.22`; the installed Go 1.24.4 satisfies it. The destination working branch is `blitzy-e887c911-…`; its HEAD commit is `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, which is exactly the source identity `kitty_815df1e210e0` that names this document.

### 2.2 Build command and outcome

kitty's C core is compiled into the `fast_data_types` extension by `setup.py` (`kitty/fast_data_types` build ≈ `setup.py:1091`; launcher ≈ `setup.py:1230`). The canonical command is `python3 setup.py`:

```
$ python3 setup.py
... (kitty C core compiles cleanly through all 122 steps, including kitty/utmp.c,
     kitty/vt-parser.c, kitty/screen.c, kitty/history.c, kitty/child-monitor.c) ...
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT'
    not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
# exit=1
```

The **only** failure is in the GLFW **Wayland windowing backend** (`glfw/wl_window.c`), where the system's newer `wayland-protocols` (1.45) introduces `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values not handled by a `switch` compiled with `-Werror=switch`. This is a windowing-toolchain/environment mismatch that is **irrelevant to the clipboard/parser/history subject** of this investigation. The build completes with the documented accommodation flag, which only downgrades `-Werror` and makes **no source change** (the resulting core binary is functionally identical):

```
$ python3 setup.py --ignore-compiler-warnings
Linking library [1/5] ...  kitty/fast_data_types.so
Linking library [2/5] ...  glfw-x11
Linking library [3/5] ...  glfw-wayland
Linking library [4/5] ...  kittens/transfer/rsync
Linking executable [5/5] ... kitty/launcher/kitty
# exit=0  (~27 s)
```

> **(canonical/deviation label)** The strict-canonical `python3 setup.py` fails to *link the GUI launcher* on this image solely because of the Wayland enum/`-Werror=switch` mismatch above; `python3 setup.py --ignore-compiler-warnings` is used to obtain the runnable binary. This deviation touches only the GLFW backend's warning treatment; the C core under study (`vt-parser.c`, `screen.c`, `history.c`, `child-monitor.c`, `clipboard.py`) compiles identically either way.

### 2.3 Artifacts and import check

```
$ ls -la kitty/fast_data_types*.so kitty/launcher/kitty
-rwxr-xr-x 1 root root 1253792 ... kitty/fast_data_types.so
-rwxr-xr-x 1 root root   40384 ... kitty/launcher/kitty

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ ./kitty/launcher/kitty +runpy 'import kitty.fast_data_types as f; print("ok", bool(f.Screen), bool(f.HistoryBuf))'
ok True True
```

All runtime probes below are executed with the embedded interpreter via `./kitty/launcher/kitty +runpy '<code>'` or `./kitty/launcher/kitty +launch <script.py>`, both of which run the same Python 3.13.7 that is linked against `fast_data_types.so`.

---

## 3. SQ-1 — Core↔Python transport (the exact C→Python boundary crossing)

**Direct answer.** A clipboard payload crosses the boundary as a **single zero-copy, read-only `memoryview`** that points directly into the VT parser's internal byte buffer. The parser's OSC dispatcher constructs the view with `PyMemoryView_FromMemory(..., PyBUF_READ)`, then calls a method named `clipboard_control` on the `Screen`'s Python `callbacks` object through a C-API `CALLBACK` macro. No bytes are copied at the boundary itself.

### 3.1 The end-to-end path (each step cited)

1. **Bytes enter on the I/O thread.** `read_bytes` reads from the child fd into the parser's write region and commits it — with **no Python C-API involved** (`kitty/child-monitor.c:1341`,`:1344`,`:1354`).
2. **The main thread parses.** `parse_input` runs on the main loop (`kitty/child-monitor.c:1236`), driving the VT state machine.
3. **OSC dispatch creates the view.** In `dispatch_osc`, the `START_DISPATCH` macro wraps the buffer region in a read-only `memoryview` (`kitty/vt-parser.c:460-461`):

```c
#define START_DISPATCH {\
    RAII_PyObject(mv, PyMemoryView_FromMemory((char*)buf + i, limit - i, PyBUF_READ)); \
    if (mv) {
```

4. **The clipboard codes dispatch.** OSC `52` and `5522` share a case; an extended OSC 52 is remapped to `-52` (`kitty/vt-parser.c:531`,`:533`) and dispatched (`:534`):

```c
case 52: case 5522:
    ...
    if (is_extended_osc && code == 52) code = -52;
    ...
    DISPATCH_OSC_WITH_CODE(clipboard_control);
```

5. **C calls Python.** `clipboard_control` in the C `Screen` invokes the Python callback via the `CALLBACK` macro (`kitty/screen.c:2305-2307`), which is `PyObject_CallMethod(self->callbacks, ...)` (`kitty/screen.c:87-91`):

```c
// kitty/screen.c
static void
clipboard_control(Screen *self, int code, PyObject *data) {
    if (code == 52 || code == -52) { CALLBACK("clipboard_control", "OO", data, code == -52 ? Py_True : Py_False); }
    else CALLBACK("clipboard_control", "OO", data, Py_None);
}
```

6. **Python receives the view.** `Window.clipboard_control(self, data: memoryview, is_partial=...)` routes by the second argument (`kitty/window.py:1391-1395`): `None` → `parse_osc_5522(data)`, otherwise → `parse_osc_52(data, is_partial)`.

### 3.2 The complete dispatch mapping (confirmed at runtime)

| Escape at the PTY | C code | 2nd `CALLBACK` arg | `is_partial` in Python | Python routing |
|---|---|---|---|---|
| regular OSC 52 | `52` | `Py_False` | `False` | `parse_osc_52(mv, False)` → finalizes immediately |
| chunked/partial OSC 52 (payload > 256 KiB) | `-52` | `Py_True` | `True` | `parse_osc_52(mv, True)` → keeps `in_flight_write_request` |
| OSC 5522 (extended) | `5522` | `Py_None` | `None` | `parse_osc_5522(mv)` |

### 3.3 Runtime evidence — the real PTY → C parser → Python crossing

The probe forks a genuine child over a real PTY (kitty's own `PTY`/`parse_bytes` path, whose test hooks call the **identical** `vt_parser_create_write_buffer`/`vt_parser_commit_write`/`run_worker` functions used in production, `kitty/screen.c:4755-4783`), emits a genuine escape sequence, and observes the object handed to `clipboard_control`. Complete, unedited output:

```
>>> OSC52 small write, decoded payload=b"hello", b64='aGVsbG8=', escape printf-octal='\\033]52;c;aGVsbG8=\\007'
  [diag] pty.received_bytes=b'\x1b]52;c;aGVsbG8=\x07'
=== OSC52 small write "hello" ===
  callback#0: type=memoryview is_memoryview=True readonly=True nbytes=10 len=10 is_partial=False
            first60=b'c;aGVsbG8='
  in_flight_write_request AFTER finalize: None
  capture-boss clipboard committed: {'text/plain': b'hello'}

>>> OSC52 read "?" escape printf-octal='\\033]52;c;?\\007'
  [diag] pty.received_bytes=b'\x1b]52;c;?\x07'
=== OSC52 read request ===
  callback#0: type=memoryview is_memoryview=True readonly=True nbytes=3 len=3 is_partial=False
            first60=b'c;?'

>>> OSC5522 read escape printf-octal='\\033]5522;type=read;dGV4dC9wbGFpbg==\\007', mimes_b64='dGV4dC9wbGFpbg=='
  [diag] pty.received_bytes=b'\x1b]5522;type=read;dGV4dC9wbGFpbg==\x07'
=== OSC5522 read request ===
  callback#0: type=memoryview is_memoryview=True readonly=True nbytes=26 len=26 is_partial=None
            first60=b'type=read;dGV4dC9wbGFpbg=='
```

**What this proves.**
- The object at the boundary is a **`memoryview`** that is **`readonly=True`** — matching `PyMemoryView_FromMemory(..., PyBUF_READ)` at `kitty/vt-parser.c:461`. The crossing is genuinely zero-copy.
- The view spans the OSC payload **after** the numeric code — e.g. `b'c;aGVsbG8='` is the `where`-field `c`, a `;`, then the base64 of `hello`.
- The dispatch mapping is exactly as tabulated: regular OSC 52 → `is_partial=False`; OSC 5522 → `is_partial=None`.
- The full write path runs end-to-end: the small write is decoded and committed as `{'text/plain': b'hello'}` — i.e. the bytes made it from a C buffer into an owned Python object and back out through the clipboard manager.

> The OSC 52 **read** request (`?`) additionally reaches a permission prompt: the default `clipboard_control` policy asks before reading (`kitty/clipboard.py:528`, `ask_to_read_clipboard` → `get_boss().confirm(...)`). In the probe this raises `AttributeError: 'CaptureBoss' object has no attribute 'confirm'` on the capture stub **after** the boundary callback has already fired — confirming the read path, not a defect. Writes are permitted by default; reads ask.

---

## 4. SQ-2 — Clipboard crossing, small and large

**Direct answer.** Small and large payloads use the **same** boundary crossing (§3), but diverge in how the Python side *accumulates* them, governed by **two independent size thresholds**:

1. **16 MiB — the rollover threshold** (`WriteRequest.rollover_size = 16 * 1024 * 1024`, `kitty/clipboard.py:237`). The accumulating `Tempfile` begins as an in-memory `io.BytesIO` (`kitty/clipboard.py:29`) and **rolls over to an on-disk `TemporaryFile`** once its size would exceed this (`kitty/clipboard.py:32-35`). This threshold is **reachable and observed**.
2. **512 (MiB) — the `clipboard_max_size` truncation limit** (`kitty/options/definition.py:3111` `opt('clipboard_max_size','512',…)`; type default `clipboard_max_size: float = 512.0`, `kitty/options/types.py:498`). Beyond it, further data is dropped and `max_size_exceeded` is set (`kitty/clipboard.py:321-323`). Through the **real OSC path**, however, this limit is subject to a **double scaling** (below) that makes it effectively unreachable; the *mechanism* is demonstrated with an explicit small limit **(non-canonical)**.

Additionally, large OSC 52 payloads are **chunked by the C parser** at `MAX_ESCAPE_CODE_LENGTH = BUF_SZ/4 = 262144` bytes (256 KiB) (`kitty/vt-parser.c:21`), arriving as a sequence of partial `memoryview`s (`is_partial=True`) followed by a final one; OSC 5522 is **not** C-chunked (only `is_osc_52` matches `"52;"`) and instead carries its own application-level `wdata` records.

### 4.1 OSC 52 — small vs large, with before/during/after `in_flight_write_request`

Complete, unedited output (real PTY child → real C parser):

```
=== OSC52 chunked write ~1 MiB (triad + chunking, in-memory) (decoded target=1048576 bytes, base64=1398104 bytes) ===
  BEFORE: crm.in_flight_write_request=None
  DURING (first partial): crm.in_flight_write_request=<kitty.clipboard.WriteRequest object at 0x7bd578d946e0> tempfile.file=BytesIO
  partial callbacks (is_partial=True): 5
  final callbacks   (is_partial=False): 1
  first partial memoryview len: 270324  (MAX_ESCAPE_CODE_LENGTH=262144)
  tempfile.file type transitions (type, decoded_bytes_at_transition): [('BytesIO', 202740)]
  rollover to on-disk TemporaryFile first seen at decoded bytes: None
  AFTER finalize: crm.in_flight_write_request=None
  committed length=1048576  integrity_ok=True

=== OSC52 LARGE write ~20 MiB (rollover to disk) (decoded target=20971520 bytes, base64=27962028 bytes) ===
  BEFORE: crm.in_flight_write_request=None
  DURING (first partial): crm.in_flight_write_request=<kitty.clipboard.WriteRequest object at 0x7bd578d42490> tempfile.file=BytesIO
  partial callbacks (is_partial=True): 105
  final callbacks   (is_partial=False): 1
  first partial memoryview len: 264874  (MAX_ESCAPE_CODE_LENGTH=262144)
  tempfile.file type transitions (type, decoded_bytes_at_transition): [('BytesIO', 198654), ('BufferedRandom', 16965567)]
  rollover to on-disk TemporaryFile first seen at decoded bytes: 16965567
  AFTER finalize: crm.in_flight_write_request=None
  committed length=20971520  integrity_ok=True
```

**Reading the output.**
- **Before/during/after triad** (state that changes over time): `in_flight_write_request` is `None` **before** the transfer, a live `WriteRequest` (with `tempfile.file=BytesIO`) **during** the partial chunks, and `None` again **after** finalization. This is the `parse_osc_52(..., is_partial=True)` behavior that *keeps* the in-flight request until a non-partial terminator arrives (`kitty/clipboard.py:406`).
- **Small/medium (1 MiB):** stays entirely in memory — the only `tempfile.file` type seen is `BytesIO`; `rollover … first seen: None`. Chunked into **5 partial + 1 final** callbacks (1 MiB > 256 KiB so it is C-chunked, but never crosses 16 MiB).
- **Large (20 MiB):** **105 partial + 1 final** callbacks; the first partial `memoryview` is `264874` bytes ≈ `MAX_ESCAPE_CODE_LENGTH` (262144). The `tempfile.file` transitions **`BytesIO` → `BufferedRandom`** (an on-disk `TemporaryFile`) first seen at `16965567` decoded bytes ≈ **16.18 MiB** — just past the fixed `16777216` (16 MiB) threshold; the small overage is base64 chunk-boundary alignment (the rollover check runs per decoded chunk). Integrity holds (`committed length=20971520 integrity_ok=True`).

### 4.2 OSC 5522 — small vs large (not C-chunked; application-level `wdata`)

The extended protocol is `<OSC>5522;metadata;payload<ST>` with colon-separated `key=value` metadata and a base64 payload (`docs/clipboard.rst`). A write is a multi-step transaction: `type=write` (start) → `type=wdata:mime=<base64 mime>` + base64 chunk (data) → `type=wdata` (commit). Complete, unedited output:

```
=== OSC5522 small write (11 bytes) (decoded=11, b64=16, wdata_records=1) ===
  BEFORE: in_flight_write_request=None
  DURING: in_flight_write_request=<kitty.clipboard.WriteRequest object at 0x7cb75b82cad0> tempfile.file=BytesIO
  total OSC5522 callbacks=3  is_partial values seen={None} (None => parse_osc_5522)
  tempfile.file transitions=[('BytesIO', 0)]
  rollover to on-disk first seen at decoded bytes=None
  AFTER commit: in_flight_write_request=None
  committed length=11 integrity_ok=True
  DONE response(s) sent to child=[b'5522;type=write:status=DONE']

=== OSC5522 LARGE write ~20 MiB (rollover) (decoded=20971520, b64=27962028, wdata_records=143) ===
  BEFORE: in_flight_write_request=None
  DURING: in_flight_write_request=<kitty.clipboard.WriteRequest object at 0x7cb75b7da5d0> tempfile.file=BytesIO
  total OSC5522 callbacks=145  is_partial values seen={None} (None => parse_osc_5522)
  tempfile.file transitions=[('BytesIO', 0), ('BufferedRandom', 16809984)]
  rollover to on-disk first seen at decoded bytes=16809984
  AFTER commit: in_flight_write_request=None
  committed length=20971520 integrity_ok=True
  DONE response(s) sent to child=[b'5522;type=write:status=DONE', b'5522;type=write:status=DONE']
```

**Reading the output.** Every OSC 5522 callback carries `is_partial=None` (routing to `parse_osc_5522`), confirming it is **not** C-chunked. The small write is **3 callbacks** (write / one `wdata` / commit), stays in `BytesIO`, and emits a `type=write:status=DONE` reply to the child. The large write arrives as **143 `wdata` records + write + commit = 145 callbacks**, each record ≤ 256 KiB at the application level, and rolls over **`BytesIO` → `BufferedRandom`** at `16809984` decoded bytes ≈ **16.03 MiB**. Both preserve integrity.

### 4.3 The truncation threshold — observed double scaling

**Direct answer:** through the real OSC path the documented 512 MiB truncation limit is **effectively unreachable** because the byte count is scaled by `1024*1024` twice. `WriteRequest.max_size` is set to `clipboard_max_size * 1024 * 1024` (already **bytes**) at `kitty/clipboard.py:247`; the guard at `kitty/clipboard.py:321` then compares `tempfile.tell()` against `self.max_size * 1024 * 1024` — scaling **again**. Complete, unedited output:

```
=== (A) DEFAULT-PATH max_size via the real OSC construction (WriteRequest with default max_size=-1) ===
  get_options().clipboard_max_size = 512.0  (MiB, from options)
  WriteRequest().max_size (bytes) = 536870912.0   == 512 * 1024 * 1024 = 536870912
  Tempfile rollover_size (bytes)  = 16777216  == 16 * 1024 * 1024 = 16777216
  EFFECTIVE truncation threshold used at clipboard.py:321 = self.max_size*1024*1024 = 562949953421312.0 bytes
      = 512.0 TiB   (line 247 already scaled to bytes; line 321 scales AGAIN -> double scaling)
  => Through the real OSC path, truncation is effectively UNREACHABLE at the documented 512 MiB.

=== (B) TRUNCATION MECHANISM demo — MODULE-LEVEL / NON-CANONICAL (WriteRequest built with explicit max_size) ===
  WriteRequest(max_size=1).max_size = 1  -> truncation threshold = max_size*1024*1024 = 1048576 bytes (1 MiB)
  chunk#4: max_size_exceeded flipped False->True at fed~=1310720 decoded bytes, tempfile.tell()=1310720
  total decoded fed to add_base64_data = 5242880 bytes (5.00 MiB)
  FINAL max_size_exceeded = True
  FINAL tempfile.tell() (bytes actually kept) = 1310720 (1.25 MiB); limit was 1 MiB = 1048576
  captured log_error messages: ['Clipboard write request has more data than allowed by clipboard_max_size (1), truncating']
```

**Reading the output.**
- **(A)** In the canonical path, `clipboard_max_size=512.0` → `max_size=536870912.0` bytes → the effective guard is `536870912.0 * 1024 * 1024 ≈ 5.63e14` bytes = **512.0 TiB**. So a real clipboard write is bounded in practice by the **16 MiB rollover** (which merely moves data to disk) rather than by truncation. This is reported exactly as observed; it is a latent double-multiply, **documented, not fixed** (read-only task).
- **(B) (non-canonical)** Constructing `WriteRequest(max_size=1)` (a 1 MiB limit) demonstrates the *mechanism*: fed 5 MiB in 256 KiB chunks, `max_size_exceeded` flips `False → True` at chunk #4 (~1,310,720 bytes), the emitted log is `Clipboard write request has more data than allowed by clipboard_max_size (1), truncating`, and the final kept size is `1310720` (1.25 MiB = the 1 MiB limit plus the one chunk that crossed it). Semantics: the chunk that crosses the limit is kept, subsequent writes are dropped.

### 4.4 Module-level confirmation of the chunk + ownership copy (non-canonical)

```
$ ./kitty/launcher/kitty +launch test.py --module clipboard
test_clipboard_write_request (kitty_tests.clipboard.TestClipboard) ... ok
```

This existing test (`kitty_tests/clipboard.py:11-30`) feeds base64 chunk-by-chunk into a `WriteRequest`, asserting the leftover-byte ownership copy `bytes(wr.current_leftover_bytes) == b'aw'` and `wr.data_for() == b'light work'`. It exercises `WriteRequest` **directly** (not through the escape path), so it is labeled **(non-canonical / module-level)**; the canonical SQ-1/SQ-2 evidence is the real PTY output in §3.3, §4.1, §4.2.

---

## 5. SQ-3 — Transfer under concurrent load

**Direct answer.** Under concurrent load the picture is: **one main thread serializes all parsing, all C→Python callbacks, and rendering, in strict FIFO order**, while a **separate I/O thread keeps reading raw bytes into the parser buffer** the whole time (holding no GIL). Clipboard callbacks are therefore never dropped or reordered by concurrency; they are simply **queued in the shared buffer and delivered in order** as the single consumer works through them. The shared buffer is bounded at **1 MiB (`BUF_SZ`)**; when it fills, kitty applies **backpressure** by removing the child fd from the poll set (`kitty/child-monitor.c:1501`) so the child blocks on `write()` until the main thread drains.

The single main loop is `run_main_loop(process_global_state, …)` (`kitty/child-monitor.c:1259-1262`); each iteration calls `parse_input` and then `render` (`kitty/child-monitor.c:1236`). The I/O thread's `read_bytes` performs `vt_parser_create_write_buffer` → `read(fd, …)` → `vt_parser_commit_write` with **no** Python C-API (`kitty/child-monitor.c:1341-1354`). The structural guarantee that the I/O thread never contends for the GIL is verified in §8.3.

### 5.1 Serialization + in-order + complete delivery (parser level)

Interleaving ~8.24 MB of heavy output (2000 × 4096-byte blocks) with 2000 OSC 52 clipboard writes and feeding them through the real single-consumer parser. Complete, unedited output, **3 runs**:

```
=== SQ-3 Part 1: serialization + in-order + complete delivery under heavy load (>=2 runs) ===
  run#0: stream_bytes=8240000 heavy_block=4096 clip_ops=2000 -> callbacks_delivered=2000 in_order=True wall=49.8ms
  run#1: stream_bytes=8240000 heavy_block=4096 clip_ops=2000 -> callbacks_delivered=2000 in_order=True wall=49.8ms
  run#2: stream_bytes=8240000 heavy_block=4096 clip_ops=2000 -> callbacks_delivered=2000 in_order=True wall=49.7ms
  STABILITY: callbacks_delivered set={2000} (stable if size 1); all_in_order=True
  wall-clock ms: min=49.7 median=49.8 max=49.8 N=3
```

**Scale & stability:** 8,240,000 bytes streamed + 2000 clipboard ops per run; **N=3 runs**; **all 2000 callbacks delivered, in order, every run** (`callbacks_delivered set={2000}`), wall-clock tightly stable at 49.7–49.8 ms. No loss, no reordering under heavy interleave.

### 5.2 Buffer bound + backpressure + delayed-but-complete drain

Committing input into the parser buffer **without** consuming (mimicking the I/O thread outrunning a busy main thread), then draining once. Complete, unedited output, **3 runs**:

```
=== SQ-3 Part 2: I/O-thread-style buffering bound + backpressure, then drain (>=2 runs) ===
  run#0: buffered_without_consume=1048576 bytes (BUF_SZ=1048576); first_avail=1048576 last_avail=0 (0 => backpressure/full)
            callbacks BEFORE consume=0 (stalled), AFTER single consume=5 (delivered in order=[b'stall-000', b'stall-001', b'stall-002', b'stall-003', b'stall-004'])
  run#1: buffered_without_consume=1048576 bytes (BUF_SZ=1048576); first_avail=1048576 last_avail=0 (0 => backpressure/full)
            callbacks BEFORE consume=0 (stalled), AFTER single consume=5 (delivered in order=[b'stall-000', b'stall-001', b'stall-002', b'stall-003', b'stall-004'])
  run#2: buffered_without_consume=1048576 bytes (BUF_SZ=1048576); first_avail=1048576 last_avail=0 (0 => backpressure/full)
            callbacks BEFORE consume=0 (stalled), AFTER single consume=5 (delivered in order=[b'stall-000', b'stall-001', b'stall-002', b'stall-003', b'stall-004'])
  STABILITY: buffered_bytes set={1048576} last_avail set={0} after_consume_callbacks set={5}
```

**Reading the output.** The buffer fills to **exactly `1048576` bytes = `BUF_SZ`** (`kitty/vt-parser.c:18`), after which the available write space reported by `vt_parser_create_write_buffer` is **0** — this is the backpressure point (`vt_parser_has_space_for_input` would return false, `kitty/vt-parser.c:1477`). While the consumer is stalled, **0** callbacks fire; a single `consume` then delivers the queued clipboard callbacks **in order** (`stall-000 … stall-004`). This is the core of "in practice": events are **delayed, not lost**, and ordering is preserved. Perfectly stable across 3 runs.

### 5.3 Full-kitty end-to-end under a concurrent output flood

Running the **real** launcher under `xvfb`: a child emits a **3,600,000-byte (60,000-line) output flood** while, concurrently, an OSC 52 write of `CONCURRENT-MARKER-98765` is issued and then read back with `kitten clipboard --get-clipboard`. Complete, unedited output:

```
run#1: kitty_exit=0  clipboard_readback='CONCURRENT-MARKER-98765'  match=YES
run#2: kitty_exit=0  clipboard_readback='CONCURRENT-MARKER-98765'  match=YES
run#3: kitty_exit=0  clipboard_readback='CONCURRENT-MARKER-98765'  match=YES
run#1: wall=3513ms readback='CONCURRENT-MARKER-98765' match=YES
run#2: wall=5031ms readback='CONCURRENT-MARKER-98765' match=YES
```

**Scale & stability:** ~3.6 MB flood concurrent with the clipboard op; **5 runs total, all `match=YES`**. Wall-clock varies (3513–5031 ms) because of `xvfb`/GLFW/dbus startup overhead — reported as-is rather than stabilized — but the **delivery correctness is invariant**: under a real concurrent flood the clipboard transfer is **delayed but never lost or corrupted**. This confirms end-to-end the serialization behavior demonstrated at the parser level in §5.1–§5.2.

> **Nuance — parse gating (`input_delay`).** The main loop does not necessarily parse on every wakeup: `run_worker` gates parsing on `OPT(input_delay)` (default **3 ms**, `docs/performance.rst:48`) unless a flush is forced or the buffer is nearly full (`kitty/vt-parser.c:1425`). Under a steady byte stream this batches parsing into ~3 ms quanta, which is what "the transfer looks like in practice" for interactive throughput.

---

## 6. SQ-4 — Expensive scrollback scan → event delivery

**Direct answer.** **Yes** — an expensive scrollback scan **delays** the delivery of events/callbacks to kittens, by approximately **the scan's own duration**, but it never loses them. The cause is direct and specific: the history-scan functions in `kitty/history.c` hold the **GIL** for their entire run (they contain **no** `Py_BEGIN_ALLOW_THREADS` — see §8.3), and they execute on the **single main thread** that also delivers every clipboard/kitten callback. While the scan runs, no other Python — including any kitten callback — can execute; the moment it returns, delivery resumes.

### 6.1 Method

A background "heartbeat" Python thread appends `time.monotonic()` roughly every 1 ms; it represents the cadence at which the main loop would deliver events to kittens if it were free. The main thread then runs a C history scan. Because the scan holds the GIL, the heartbeat thread is **starved** for the scan's whole duration, so the **maximum inter-heartbeat gap equals the event-delivery delay**. All five candidate scan functions are exercised **by name**:
`__str__` (`kitty/history.c:321`), `as_ansi` (`:348`), `as_text_for_history_buf`→`as_text_history_buf` (`:509`, reached via `Screen.as_text_for_history_buf`, `kitty/screen.c:3494`), `pagerhist_as_bytes` (`:461`), `pagerhist_as_text` (`:486`).

### 6.2 All five candidates, N=2 (80,000 lines × 200 cols)

Columns: `scan` (ms) · `base_gap_max` (heartbeat gap when main thread is free) · `heartbeat_gap_max` (gap during the scan = delivery delay). Complete, unedited output:

```
### SCALE: nlines=80000 cols=200 runs=2 (Python 3.13.7)
-- run#1: hb.count=80000 implied_segments(SEGMENT_SIZE=2048)=40 RSS0=22528KiB RSS_after_build=523264KiB Cbuild_delta=500736KiB ph_cap=2000 ph_pushed=80000
   __str__(history.c:321)                     scan=  128.67ms  base_gap_max=  1.14ms  heartbeat_gap_max=  129.42ms  outlen=  16079999  tracemalloc_cur=  16.08MB peak=  36.00MB getsizeof=  16.08MB
   as_ansi(history.c:348)                     scan=  163.14ms  base_gap_max=  2.06ms  heartbeat_gap_max=  163.29ms  outlen=  16080000  tracemalloc_cur=  20.07MB peak=  20.07MB getsizeof=  19.36MB
   as_text_for_history_buf(history.c:509)     scan=   99.42ms  base_gap_max=  1.11ms  heartbeat_gap_max=  100.13ms  outlen=  16079196  tracemalloc_cur=  20.56MB peak=  20.56MB getsizeof=  22.64MB
   pagerhist_as_bytes(history.c:461)          scan=    8.54ms  base_gap_max=  4.81ms  heartbeat_gap_max=    9.48ms  outlen=  15990000  tracemalloc_cur=  15.99MB peak=  15.99MB getsizeof=  15.99MB
   pagerhist_as_text(history.c:486)           scan=   12.23ms  base_gap_max=  1.13ms  heartbeat_gap_max=   12.58ms  outlen=  15990000  tracemalloc_cur=  15.99MB peak=  31.98MB getsizeof=  15.99MB
-- run#2: hb.count=80000 implied_segments(SEGMENT_SIZE=2048)=40 RSS0=1111908KiB RSS_after_build=1111908KiB Cbuild_delta=0KiB ph_cap=2000 ph_pushed=80000
   __str__(history.c:321)                     scan=   89.17ms  base_gap_max=  1.40ms  heartbeat_gap_max=   89.38ms  outlen=  16079999  tracemalloc_cur=  16.08MB peak=  36.00MB getsizeof=  16.08MB
   as_ansi(history.c:348)                     scan=  107.64ms  base_gap_max=  4.13ms  heartbeat_gap_max=  107.72ms  outlen=  16080000  tracemalloc_cur=  20.07MB peak=  20.07MB getsizeof=  19.36MB
   as_text_for_history_buf(history.c:509)     scan=   69.02ms  base_gap_max=  1.11ms  heartbeat_gap_max=   69.66ms  outlen=  16079196  tracemalloc_cur=  20.56MB peak=  20.56MB getsizeof=  22.64MB
   pagerhist_as_bytes(history.c:461)          scan=    2.77ms  base_gap_max=  1.13ms  heartbeat_gap_max=    3.43ms  outlen=  15990000  tracemalloc_cur=  15.99MB peak=  15.99MB getsizeof=  15.99MB
   pagerhist_as_text(history.c:486)           scan=   10.14ms  base_gap_max=  1.13ms  heartbeat_gap_max=   10.81ms  outlen=  15990000  tracemalloc_cur=  15.99MB peak=  31.98MB getsizeof=  15.99MB
```

**Reading the output.** For **every** scan and **both** runs, `heartbeat_gap_max ≈ scan` while `base_gap_max` stays ~1–5 ms. That is the delay, measured directly: e.g. `__str__` blocks event delivery for ~129 ms (run 1) / ~89 ms (run 2); `as_ansi` for ~163 / ~108 ms. The per-line scans (`__str__`, `as_ansi`, `as_text_for_history_buf`) are the expensive ones (~70–165 ms at this scale); the `pagerhist_*` scans are far cheaper (~3–12 ms) because they copy a pre-serialized ring buffer rather than iterating per line.

### 6.3 Timing distribution, N=5 (per the magnitude/timing rule)

Absolute scan times vary run-to-run (cold vs. warm allocator/cache), so the distribution is reported for the two heaviest scans. Complete, unedited output:

```
### TIMING DISTRIBUTION nlines=80000 cols=200 (Python 3.13.7)
  __str__(history.c:321): N=5
    scan_ms         : min=95.20 median=100.98 max=107.61
    heartbeat_gap_ms: min=95.49 median=101.53 max=107.09  (==> event-delivery delay tracks scan)
    base_gap_ms     : min=1.12 median=1.14 max=3.71  (main thread free -> ~1-4ms cadence)
  as_ansi(history.c:348): N=5
    scan_ms         : min=112.16 median=125.27 max=142.41
    heartbeat_gap_ms: min=108.07 median=120.64 max=139.64  (==> event-delivery delay tracks scan)
    base_gap_ms     : min=1.39 median=1.44 max=1.75  (main thread free -> ~1-4ms cadence)
```

**Invariant across the distribution:** `heartbeat_gap` tracks `scan` in every one of the N=5 runs (e.g. `__str__` median scan 100.98 ms vs. median gap 101.53 ms), while `base_gap` stays ~1–4 ms. The absolute value scales with scan size; the *relationship* (delivery delay ≈ scan duration) is stable and reproducible.

---

## 7. SQ-5 — Expensive scrollback scan → memory management

**Direct answer.** Memory has **two distinct parts**. (1) The **persistent, dominant** cost is the **C-side scrollback storage**: `HistoryBuf` holds cells in fixed **2048-line segments** (`SEGMENT_SIZE`, `kitty/history.c:15`) grown by `realloc` in `add_segment` (`kitty/history.c:18-22`) — measured at **~610 MiB for 100,000 lines × 200 cols** (~6.4 KB/line). (2) A scan additionally allocates a **transient Python object** on the Python heap (~16–22 MB at 80k×200 here), with a **construction peak of up to ~2×** for scans that hold two representations at once; this object is freed after the kitten consumes it. The scan does not change how the C scrollback itself is managed — it *reads* it and *materializes* a Python copy.

### 7.1 Python object footprint of each scan (stable across runs)

From the N=2 table in §6.2 (`getsizeof` = retained size of the produced object; `tracemalloc peak` = high-water during construction):

| Scan | `getsizeof` (retained) | `tracemalloc` peak | Note |
|---|---|---|---|
| `__str__` (`:321`) | **16.08 MB** | **36.00 MB** | builds a tuple of per-line `str`s then joins → both coexist (~2×) |
| `as_ansi` (`:348`) | **19.36 MB** | 20.07 MB | list of 80,000 line strings |
| `as_text_for_history_buf` (`:509`) | **22.64 MB** | 20.56 MB | per-line callback accumulation |
| `pagerhist_as_bytes` (`:461`) | **15.99 MB** | 15.99 MB | single `bytes` copied from the ring buffer |
| `pagerhist_as_text` (`:486`) | **15.99 MB** | **31.98 MB** | `bytes` + decoded `str` coexist during decode (~2×) |

The `getsizeof` values are **identical across both runs** (deterministic); only timing varied. The `~2×` peaks for `__str__` and `pagerhist_as_text` are the memory-management nuance: a large scan transiently needs roughly double its output size while it converts between representations.

### 7.2 C-side segment growth (current RSS, SEGMENT_SIZE = 2048)

Building progressively larger buffers and reading **current** RSS (`/proc/self/statm`, since `ru_maxrss` is a monotonic high-water mark — visible as `Cbuild_delta=0KiB` on run 2 in §6.2). Complete, unedited output:

```
### C-side segment growth (SEGMENT_SIZE=2048 lines/segment), cols=200, current RSS
  lines=   2048 count=   2048 implied_segments=  1 RSS_delta=       0KiB (~0.0MiB) per_line~0B
  lines=   4096 count=   4096 implied_segments=  2 RSS_delta=    9288KiB (~9.1MiB) per_line~2322B
  lines=  20480 count=  20480 implied_segments= 10 RSS_delta=  102464KiB (~100.1MiB) per_line~5123B
  lines= 100000 count= 100000 implied_segments= 49 RSS_delta=  625272KiB (~610.6MiB) per_line~6402B
```

**Reading the output.** The number of segments is `ceil(count / 2048)` — 1, 2, 10, 49 for 2048/4096/20480/100000 lines — matching `SEGMENT_SIZE=2048`. RSS grows with the scrollback (up to ~610 MiB at 100k lines × 200 cols). This is the **persistent** memory the scan competes with; the scan's own Python output (§7.1) is small by comparison and transient. Preserving the user's framing: the scan is an **expensive main-thread operation competing for the GIL** with clipboard/event delivery — it delays delivery (SQ-4) and briefly adds a Python-heap object (SQ-5), but it does not alter the segment-based C management of the scrollback itself.

---

## 8. SQ-6 — Where timing, concurrency, and object ownership begin to matter

**Direct answer.** There are exactly three loci:

1. **Concurrency / timing — the parser lock over a single producer/consumer buffer.** The parser owns one `BUF_SZ` (1 MiB) buffer partitioned into a *read* region (consumed by the main thread) and a *write* region (filled by the I/O thread), synchronized by one `pthread_mutex_t lock` (`kitty/vt-parser.c:206`).
2. **Timing — the GIL + `input_delay` gate on the single main thread** (already shown in §5–§7).
3. **Object ownership — the RAII lifetime of the boundary `memoryview`.**

### 8.1 The parser lock and the producer/consumer partition

Complete, unedited source (`kitty/vt-parser.c:1413-1490`):

```c
#define with_lock pthread_mutex_lock(&self->lock);
#define end_with_lock pthread_mutex_unlock(&self->lock);

static void
run_worker(void *p, ParseData *pd, bool flush) {
    Screen *screen = (Screen*)p;
    PS *self = (PS*)screen->vt_parser->state;
    with_lock {
        self->read.sz += self->write.pending; self->write.pending = 0;
        pd->has_pending_input = self->read.pos < self->read.sz;
        if (pd->has_pending_input) {
            pd->time_since_new_input = pd->now - self->new_input_at;
            if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
                pd->input_read = true;
                self->dump_callback = pd->dump_callback; self->now = pd->now;
                self->screen = screen;
                self->read.consumed = 0;
                do {
                    end_with_lock; {
                        consume_input(self, pd->dump_callback, screen->window_id);
                    } with_lock;
                    self->read.sz += self->write.pending; self->write.pending = 0;
                } while (self->read.pos < self->read.sz);
                self->new_input_at = 0;
                if (self->read.consumed) {
                    pd->write_space_created = self->read.sz >= BUF_SZ;
                    self->read.pos -= MIN(self->read.pos, self->read.consumed);
                    self->read.sz -= MIN(self->read.sz, self->read.consumed);
                    if (self->read.sz) memmove(self->buf, self->buf + self->read.consumed, self->read.sz);
                }
            }
        }
    } end_with_lock;
}
```

The key concurrency design: `run_worker` takes the lock to fold the producer's `write.pending` bytes into the consumer's `read.sz`, but then **releases the lock while it actually parses** — `end_with_lock { consume_input(...) } with_lock` — so the I/O thread can keep committing new writes during the (potentially long) parse, and re-acquires it afterward to compact the buffer with `memmove`. The gate `flush || time_since_new_input >= OPT(input_delay) || read.sz + 16*1024 > BUF_SZ` is where **timing** (`input_delay`, default 3 ms) enters.

The producer side, also under the lock (`kitty/vt-parser.c:1451-1487`):

```c
uint8_t*
vt_parser_create_write_buffer(Parser *p, size_t *sz) {
    PS *self = (PS*)p->state;
    uint8_t *ans;
    with_lock {
        if (self->write.sz) fatal("vt_parser_create_write_buffer() called with an already existing write buffer");
        self->write.offset = self->read.sz + self->write.pending;
        *sz = BUF_SZ - self->write.offset;
        self->write.sz = *sz;
        ans = self->buf + self->write.offset;
    } end_with_lock;
    return ans;
}

void
vt_parser_commit_write(Parser *p, size_t sz) {
    PS *self = (PS*)p->state;
    with_lock {
        size_t off = self->read.sz + self->write.pending;
        if (self->new_input_at == 0) self->new_input_at = monotonic();
        if (self->write.offset > off) memmove(self->buf + off, self->buf + self->write.offset, sz);
        self->write.pending += sz;
        self->write.sz = 0;
    } end_with_lock;
}

bool
vt_parser_has_space_for_input(const Parser *p) {
    PS *self = (PS*)p->state;
    bool ans;
    with_lock {
        ans = self->read.sz + self->write.pending < BUF_SZ;
    } end_with_lock;
    return ans;
}
```

`vt_parser_has_space_for_input` (`read.sz + write.pending < BUF_SZ`) is the **backpressure predicate**; the I/O thread consults it and clears `POLLIN` on the child fd when the buffer is full (`kitty/child-monitor.c:1501`):

```c
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

The I/O producer call site itself (`kitty/child-monitor.c:1341-1354`) uses only these functions plus `read()` — no Python:

```c
uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
if (!available_buffer_space) return true;
while(true) {
    len = read(fd, buf, available_buffer_space);
    ...
}
vt_parser_commit_write(screen->vt_parser, len);
```

### 8.2 Object ownership — the RAII lifetime of the memoryview

The boundary `memoryview` is scoped by `RAII_PyObject`, defined as a GCC/Clang cleanup attribute (`kitty/data-types.h:53`):

```c
#define RAII_PyObject(name, initializer) __attribute__((cleanup(cleanup_decref))) PyObject *name = initializer
```

So the view created at `kitty/vt-parser.c:461` is automatically `Py_DECREF`'d when the enclosing dispatch block exits. It is **valid only within that dispatch scope**, and — critically — it does not own the bytes it points at (`PyMemoryView_FromMemory` produces a view with **no base object**; confirmed at runtime in §9). Ownership therefore "begins to matter" precisely at the moment Python decides whether to *use the bytes now* (safe) or *keep the view* (unsafe — see SQ-7).

### 8.3 The central structural fact — the I/O thread runs no Python

The reason concurrency never corrupts Python state is that the I/O thread never executes the Python C-API, so it never contends for the GIL. Verified directly:

```
$ grep -rn "Py_BEGIN_ALLOW_THREADS\|PyEval_SaveThread" kitty/*.c
kitty/utmp.c:17:    Py_BEGIN_ALLOW_THREADS
```

The **only** GIL-releasing site across the entire C core is in `kitty/utmp.c` (utmp handling), not in the parser, child-monitor, screen, or history code. Consequently, when the main thread holds the GIL for an expensive scan, the I/O thread keeps buffering bytes but no Python runs concurrently — this is the unifying fact behind SQ-3, SQ-4, and SQ-7.

The per-screen *write-back* path (writing data **to** the child) has its own separate buffer and lock (`kitty/screen.h:114-116`):

```c
uint8_t *write_buf;
size_t write_buf_sz, write_buf_used;
pthread_mutex_t write_buf_lock;
```

This is distinct from the parser's read path and is where transient write threads synchronize with the main thread.

---

## 9. SQ-7 — How subtle races might emerge only under real runtime conditions

**Direct answer.** The only genuine hazard is a **C-level object-lifetime race**, *not* a Python-level data race. The boundary `memoryview` aliases the parser's single reusable buffer and does not own it; if Python were to **retain that view past the dispatch scope**, a subsequent parse would overwrite the very bytes the view points at, silently changing its contents. kitty **avoids** this by immediately **copying the bytes it needs out of the transient view into owned Python objects**. There is **no** Python-level data race because the GIL serializes every Python operation on the single main thread (§8.3) — two clipboard callbacks never run concurrently.

### 9.1 The hazard, demonstrated at runtime

The probe retains the boundary `memoryview` (without copying), then feeds a **second** clipboard escape through the **same** parser, and re-reads the first view. Complete, unedited output:

```
expected head for payload#1 (A): b'c;QUFBQUFBQUFBQUFB'
expected head for payload#2 (B): b'c;QkJCQkJCQkJCQkJC'
view#0 readonly: True  obj is None: True  nbytes: 402
view#0 head at dispatch      : b'c;QUFBQUFBQUFBQUFB'  ==payload#1? True
view#0 head immediately after: b'c;QUFBQUFBQUFBQUFB'  ==payload#1? True
view#1 head at dispatch      : b'c;QkJCQkJCQkJCQkJC'  ==payload#2? True
view#0 head AFTER 2nd parse  : b'c;QkJCQkJCQkJCQkJC'
  -> retained view#0 now aliases REUSED buffer (==payload#2)? True
  -> retained view#0 still shows original payload#1?          False
```

**Reading the output.**
- `view#0 readonly: True  obj is None: True` — the boundary view is read-only and has **no base object**: it does not own or keep alive any Python buffer; it is a raw window onto the C address (matching `PyMemoryView_FromMemory(..., PyBUF_READ)`).
- At dispatch and immediately after, the retained view shows **payload #1** (`c;QUFB…`, base64 of `A`s).
- After a **second** OSC 52 (payload #2, `B`s) is parsed through the same 1 MiB buffer, the retained **view #0 now reads `c;QkJC…`** — payload #2's bytes. `aliases REUSED buffer? True`; `still original? False`. This is the C-buffer-reuse hazard, reproduced live: a stale retained view silently mutates.

### 9.2 How kitty avoids it — the ownership copy

The clipboard manager never keeps the transient view. Undecodable base64 remainder is copied **out** of the view into owned bytes at `kitty/clipboard.py:286` (`self.current_leftover_bytes = memoryview(bytes(mv[-extra:]))` — the `bytes(...)` makes an owned copy). Demonstrated by mutating the source buffer after the copy. Complete, unedited output:

```
=== PART B: kitty's ownership COPY avoids the hazard (SQ-7) ===
full base64: b'bGlnaHQgd29yaw=='
current_leftover_bytes type: memoryview value: b'aH'
after mutating source, current_leftover_bytes: b'aH' -> UNCHANGED => it is an OWNED copy, not a view
final data_for(): b'light work'
```

`current_leftover_bytes` remains `b'aH'` even after the source bytearray is overwritten — proving it is an **owned copy**, not a view. Decoded data likewise goes into an owned `Tempfile` (§4). So in production the retained-view condition of §9.1 **never arises**.

### 9.3 C-level vs. GIL-serialized Python state (explicit distinction)

- **Not a Python data race.** All parsing, all callbacks, and all scans run on one main thread under the GIL (§8.3). Two Python clipboard callbacks cannot execute concurrently; there is no shared *Python* object mutated from two threads. Persistent Python state such as `in_flight_write_request` is safe by construction.
- **The real hazard is C-level buffer lifetime.** The danger is a Python object (the `memoryview`) outliving the validity of the C memory it aliases, combined with the parser's **single reused buffer**. kitty neutralizes it by copying out within the dispatch scope.
- **(inferred)** The claim that "a race *would* occur if the view were retained across dispatches" is supported by the forced demonstration in §9.1 (a probe that deliberately retains the view). Production code does not retain it, so the race does not manifest; that production-safety conclusion is grounded in the `bytes(...)` copy at `kitty/clipboard.py:286` and the RAII scope at `kitty/vt-parser.c:461`, both observed. The counterfactual ("if kitty retained it") is labeled **inferred** because kitty's real code path structurally prevents it.

---

## 10. Coverage checklist

| Item | Where addressed | Evidence |
|---|---|---|
| **SQ-1** Core↔Python transport | §3 | real PTY output; `vt-parser.c:460-461`, `screen.c:87-91,2305-2307`, `window.py:1391-1395` |
| **SQ-2** Clipboard small vs large | §4 | 16 MiB rollover + 512-MiB double-scaling; OSC 52 & 5522 output |
| **SQ-3** Transfer under concurrent load | §5 | 2000/2000 in-order (N=3); BUF_SZ backpressure; full-kitty flood (5 runs) |
| **SQ-4** Scan → event delivery | §6 | heartbeat-gap = scan (N=2 all 5 scans; N=5 distribution) |
| **SQ-5** Scan → memory | §7 | getsizeof/peak per scan; segment growth (49 segs / 610 MiB) |
| **SQ-6** Where timing/concurrency/ownership matter | §8 | parser lock + partition; RAII memoryview; GIL fact |
| **SQ-7** Emergent races | §9 | live aliasing of retained view; ownership copy; C-vs-GIL |
| clipboard / screen structures / Python objects | §3, §4 | `memoryview` → `WriteRequest`/`Tempfile` |
| scrollback / events / memory | §6, §7 | `HistoryBuf` scans; delay + footprint |
| timing / concurrency / object ownership / races | §5, §8, §9 | GIL + lock; RAII lifetime; buffer reuse |
| OSC 52 / OSC 5522 | §3.2, §4.1, §4.2 | dispatch mapping + both write paths |
| `memoryview` / `CALLBACK` / `clipboard_control` | §3 | `readonly=True`; `PyObject_CallMethod` |
| `Tempfile` / `io.BytesIO` / `TemporaryFile` (`BufferedRandom`) | §4.1, §4.2 | observed `BytesIO`→`BufferedRandom` transition |
| `WriteRequest` / `rollover_size` (16 MiB) / `clipboard_max_size` (512) | §4 | `clipboard.py:237,247,321` |
| `in_flight_write_request` / `is_partial` / `current_leftover_bytes` | §3.2, §4.1, §9.2 | before/during/after; ownership copy |
| `io_thread` / `parse_input` / `main_loop` / `read_bytes` | §5, §8 | `child-monitor.c:1236,1259-1262,1341-1354` |
| parser `lock` / `run_worker` / `vt_parser_commit_write` / `input_delay` | §8.1 | verbatim source |
| `HistoryBuf` / `as_ansi` / `pagerhist_as_bytes` / `pagerhist_as_text` / `as_text_history_buf` / `__str__` / `SEGMENT_SIZE` | §6, §7 | all five scans by name + segment growth |

---

## 11. Appendix — temporary scripts and cleanup

### 11.1 Temporary observation scripts (all external to the repository)

To satisfy the read-only mandate, **every** temporary script was written under `/tmp/ext_probes/` — **outside** the repository working tree — so none ever appears in `git status`. They were removed after evidence capture. The scripts used:

| Script | Purpose (sub-question) |
|---|---|
| `p1_boundary.py` | Real PTY → C parser → `clipboard_control`; boundary `memoryview` props (SQ-1) |
| `p2_large_rollover.py` | OSC 52 small/large; `BytesIO`→`TemporaryFile` rollover; before/during/after (SQ-2) |
| `p3_truncation.py` | Truncation double-scaling (default) + mechanism at explicit `max_size` (SQ-2) |
| `p4_osc5522_write.py` | OSC 5522 small/large write transactions; rollover; DONE replies (SQ-2) |
| `p5_concurrent.py` | Serialization + in-order delivery; BUF_SZ buffering + backpressure (SQ-3) |
| `concurrent_flood.sh`, `rt.sh` | Full-kitty round-trip / concurrent flood under `xvfb` (SQ-3) |
| `p6_scrollback.py`, `p6b_dist.py` | Five history scans; heartbeat-gap delay; memory + segment growth (SQ-4/5) |
| `p7_ownership.py`, `p7a_fixed.py` | Retained-view aliasing hazard; ownership copy (SQ-6/7) |

The `WriteRequest(max_size=…)` mechanism check in §4.3(B), the `test.py --module clipboard` run in §4.4, and any remote-control/module-level call are labeled **(non-canonical)** in the text; the canonical SQ-1/SQ-2 evidence is the genuine OSC-escape-through-PTY output.

### 11.2 Read-only / cleanup proof

The repository tree is unchanged apart from this single new document. Immediately after authoring:

```
$ git status --porcelain
?? blitzy/

$ find . -path '*blitzy*' -not -path './.git/*' | sort
./blitzy
./blitzy/documentation
./blitzy/documentation/kitty_815df1e210e0.md
./blitzy/screen_recordings
./blitzy/screenshots
```

`git status` reports only the untracked `blitzy/` path; the sole file added under it is `blitzy/documentation/kitty_815df1e210e0.md` (the pre-existing empty `blitzy/screenshots` and `blitzy/screen_recordings` directories are unrelated scratch dirs). No existing source, test, build, or documentation file was modified, added, or deleted. Build artifacts (`kitty/fast_data_types.so`, `kitty/launcher/kitty`) are git-ignored and do not appear. All temporary scripts under `/tmp/ext_probes/` were deleted after evidence capture.

*End of document.*
