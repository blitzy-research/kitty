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

kitty's C core is compiled into the `fast_data_types` extension by `setup.py` (`kitty/fast_data_types` is the extension name passed to `compile_c_extension(...)` at `setup.py:1091`; the GUI launcher is built by `build_launcher(...)` at `setup.py:1230`). Two builds were run and each was captured **in full** to a file outside the repo; the excerpts below are reproduced **verbatim** from those saved logs, with the elided middle clearly marked and the full log path/line-count stated so the excerpt makes no "complete" claim it does not keep.

**(a) Canonical `python3 setup.py` — fails only on the Wayland GUI backend (exit 1).** Full log: 134 lines, saved to `/tmp/ext_probes/build_canonical.log`. Verbatim head (first 6 lines) and decisive tail (last 14 lines):

```
$ { python3 setup.py; echo "CANONICAL_EXIT=$?"; } > /tmp/ext_probes/build_canonical.log 2>&1
$ head -6 /tmp/ext_probes/build_canonical.log
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
        <<< lines 7-120: compile steps [7/122]..[120/122], all succeed; see saved log >>>
$ tail -14 /tmp/ext_probes/build_canonical.log
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
 done
Compiling [wayland] glfw/wl_window.c ...
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
CANONICAL_EXIT=1
```

The **only** failure is in the GLFW **Wayland windowing backend** (`glfw/wl_window.c:668`), where the system's newer `wayland-protocols` (1.45) introduces `XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}` enum values not handled by a `switch` compiled with `-Werror=switch`. All 122 C-core compile steps — including `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/history.c`, `kitty/child-monitor.c` — complete before the error; the failure is a windowing-toolchain/environment mismatch that is **irrelevant to the clipboard/parser/history subject** of this investigation.

**(b) Accommodation `python3 setup.py --ignore-compiler-warnings` — links successfully (exit 0).** Full log: 130 lines, saved to `/tmp/ext_probes/build_accom.log`. Verbatim tail (last 9 lines):

```
$ { python3 setup.py --ignore-compiler-warnings; echo "ACCOM_EXIT=$?"; } > /tmp/ext_probes/build_accom.log 2>&1
$ tail -9 /tmp/ext_probes/build_accom.log
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
ACCOM_EXIT=0
```

> **(canonical/deviation label)** The strict-canonical `python3 setup.py` fails to *link the GUI launcher* on this image solely because of the Wayland enum/`-Werror=switch` mismatch above; `python3 setup.py --ignore-compiler-warnings` is used to obtain the runnable binary. The flag makes **no source change**: `build_launcher(...)` at `setup.py:1231` reads `werror = '' if args.ignore_compiler_warnings else '-pedantic-errors -Werror'`, so the accommodation only *drops `-Werror`* — the compiled C core under study (`vt-parser.c`, `screen.c`, `history.c`, `child-monitor.c`, `clipboard.py`) is byte-for-byte identical either way.

### 2.3 Artifacts and import check

```
$ ls -la kitty/fast_data_types*.so kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root  1253792 Jul  8 05:49 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 16429348 Jul  8 05:49 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  8 05:48 kitty/launcher/kitty

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ ./kitty/launcher/kitty +runpy 'import kitty.fast_data_types as f; print("ok", bool(f.Screen), bool(f.HistoryBuf), bool(f.ChildMonitor))'
ok True True True
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

4. **The clipboard codes dispatch.** OSC `52` and `5522` share a case; a **continued** (chunked) OSC 52 is flagged `is_extended_osc` and remapped to `-52`. Complete, unedited source (`kitty/vt-parser.c:531-535`):

```c
case 52: case 5522:
    START_DISPATCH
    if (is_extended_osc && code == 52) code = -52;
    DISPATCH_OSC_WITH_CODE(clipboard_control);
    END_DISPATCH
```

5. **C calls Python.** `clipboard_control` in the C `Screen` invokes the Python callback via the `CALLBACK` macro (`kitty/screen.c:2305-2307`), which is `PyObject_CallMethod(self->callbacks, ...)` (`kitty/screen.c:87-91`):

```c
void
clipboard_control(Screen *self, int code, PyObject *data) {
    if (code == 52 || code == -52) { CALLBACK("clipboard_control", "OO", data, code == -52 ? Py_True: Py_False); }
    else { CALLBACK("clipboard_control", "OO", data, Py_None);}
}
```

6. **Python receives the view.** `Window.clipboard_control(self, data: memoryview, is_partial=...)` routes by the second argument (`kitty/window.py:1391-1395`): `None` → `parse_osc_5522(data)`, otherwise → `parse_osc_52(data, is_partial)`.

### 3.2 The complete dispatch mapping (confirmed at runtime)

| Escape at the PTY | C code | 2nd `CALLBACK` arg | `is_partial` in Python | Python routing |
|---|---|---|---|---|
| regular OSC 52 (fits in one escape) | `52` | `Py_False` | `False` | `parse_osc_52(mv, False)` → finalizes immediately |
| continued/partial OSC 52 (accumulated > 256 KiB) | `-52` | `Py_True` | `True` | `parse_osc_52(mv, True)` → keeps `in_flight_write_request` |
| OSC 5522 (extended) | `5522` | `Py_None` | `None` | `parse_osc_5522(mv)` |

The partial (`is_partial=True`, code `-52`) path is entered by `accumulate_st_terminated_esc_code` (`kitty/vt-parser.c:406-417`): when an OSC 52 accumulates **more than `MAX_ESCAPE_CODE_LENGTH` bytes** (`= BUF_SZ/4 = 256 KiB`, `kitty/vt-parser.c:21`) before an ST terminator arrives, it dispatches the bytes accumulated so far as a **partial** (`dispatch(..., true)`, `:413`), then `continue_osc_52` (`:386-391`) rewinds 4 bytes and rewrites a synthetic `52;;` header so the next chunk re-enters the same code path with an empty `where`-field. The *per-callback* size is therefore "≥ 256 KiB, up to how much has been buffered when the main loop parses" — in Probe B's test-hook batching that is a full `BUF_SZ` worth (observed `nbytes=1048570`, just under the 1 MiB buffer); in production it depends on how much the I/O thread committed before the parse ran.

### 3.3 Canonical runtime evidence — genuine OSC through the real launcher (Probe A, PRIMARY)

This is the **canonical entry point**: a real `kitty` process (the built launcher, headless via `xvfb` + llvmpipe) runs a real child shell that emits a **genuine** OSC 52 / OSC 5522 escape through the real PTY. The bytes are read by the production I/O thread (`read_bytes`, §8.1), parsed on the production main loop, delivered to the real `Window.clipboard_control`, and the clipboard is actually set — then **read back** through the `kitten clipboard` client and integrity-checked. No test hook, remote-control, or debug shortcut is used. The **only** non-default option is `clipboard_control` (to permit non-interactive read-back; the default `*-ask` policy would open a permission popup with no user to answer it).

Child script that emits the genuine escapes (`/tmp/ext_probes/A_canonical_roundtrip.sh`):

```sh
#!/bin/sh
#   case = small | large | osc5522
set -eu
CASE="$1"; RUN="$2"
OUT="/tmp/ext_probes/A_${CASE}_run${RUN}.result"
case "$CASE" in
  small)
    payload=$(printf 'hello' | base64)
    printf '\033]52;c;%s\007' "$payload"
    sleep 0.4
    got=$(kitten clipboard --get-clipboard)
    printf 'case=small run=%s readback=[%s]\n' "$RUN" "$got" > "$OUT"
    ;;
  large)
    { printf '\033]52;c;'; base64 -w0 /tmp/ext_probes/big.txt; printf '\007'; }
    sleep 0.8
    kitten clipboard --get-clipboard > /tmp/ext_probes/big_out.txt
    si=$(sha256sum /tmp/ext_probes/big.txt      | awk '{print $1}')
    so=$(sha256sum /tmp/ext_probes/big_out.txt  | awk '{print $1}')
    nin=$(wc -c < /tmp/ext_probes/big.txt); nout=$(wc -c < /tmp/ext_probes/big_out.txt)
    if [ "$si" = "$so" ]; then m=MATCH; else m=MISMATCH; fi
    printf 'case=large run=%s bytes_in=%s bytes_out=%s sha_in=%s sha_out=%s integrity=%s\n' \
           "$RUN" "$nin" "$nout" "$si" "$so" "$m" > "$OUT"
    ;;
  osc5522)
    kitten clipboard --mime text/plain /tmp/ext_probes/o5522_in.txt
    sleep 0.4
    kitten clipboard --get-clipboard --mime text/plain /dev/stdout > /tmp/ext_probes/o5522_out.txt
    si=$(sha256sum /tmp/ext_probes/o5522_in.txt  | awk '{print $1}')
    so=$(sha256sum /tmp/ext_probes/o5522_out.txt | awk '{print $1}')
    got=$(cat /tmp/ext_probes/o5522_out.txt)
    if [ "$si" = "$so" ]; then m=MATCH; else m=MISMATCH; fi
    printf 'case=osc5522 run=%s readback=[%s] integrity=%s\n' "$RUN" "$got" "$m" > "$OUT"
    ;;
esac
```

Launcher invocation (`/tmp/ext_probes/A_driver.sh`), run N=2 per case — the loop body is:

```bash
timeout 150 xvfb-run -a -s "-screen 0 1280x800x24" \
  "$KITTY" -o close_on_child_death=yes -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
  sh "/tmp/ext_probes/A_canonical_roundtrip.sh" "$CASE" "$RUN"
```

Complete, unedited output (`/tmp/ext_probes/A_output.txt`) — `kitty_exit` is prefixed by the driver, the rest is written by the child:

```
kitty_exit=0 | case=small run=1 readback=[hello]
kitty_exit=0 | case=small run=2 readback=[hello]
kitty_exit=0 | case=large run=1 bytes_in=20971520 bytes_out=20971520 sha_in=9afe8b939d014359a5c6e27c7cdd9d26d7be031328b02d2ca0d21889c88308f6 sha_out=9afe8b939d014359a5c6e27c7cdd9d26d7be031328b02d2ca0d21889c88308f6 integrity=MATCH
kitty_exit=0 | case=large run=2 bytes_in=20971520 bytes_out=20971520 sha_in=9afe8b939d014359a5c6e27c7cdd9d26d7be031328b02d2ca0d21889c88308f6 sha_out=9afe8b939d014359a5c6e27c7cdd9d26d7be031328b02d2ca0d21889c88308f6 integrity=MATCH
kitty_exit=0 | case=osc5522 run=1 readback=[osc5522-mime-payload-12345] integrity=MATCH
kitty_exit=0 | case=osc5522 run=2 readback=[osc5522-mime-payload-12345] integrity=MATCH
```

The only stderr across all runs is the harmless headless-dbus warning `[t] Failed to open systemd user bus with error: Connection refused` (unrelated to the clipboard path).

**What this proves (canonically):**
- The C→Python clipboard crossing works end-to-end through the real entry point for **small** (`hello`), **very large** (20 MiB text: `sha_in==sha_out=9afe8b93…`, integrity=MATCH), and **OSC 5522** (`osc5522-mime-payload-12345`, MATCH) payloads — all `kitty_exit=0`, byte-identical across N=2.
- The 20 MiB case necessarily crosses **both** the parser's 256 KiB per-escape chunk limit (`MAX_ESCAPE_CODE_LENGTH = BUF_SZ/4`, `kitty/vt-parser.c:21`) **and** the clipboard manager's 16 MiB in-memory→on-disk rollover (§4.1), yet the bytes survive byte-exactly — proving the large path (partial chunks reassembled + on-disk `TemporaryFile`) is correct end-to-end.
- The read-back itself exercises the OSC 52/5522 **read** path canonically (the `kitten clipboard --get-clipboard` client): reads succeed because `clipboard_control` includes `read-clipboard`. Under the default `read-clipboard-ask` policy a real kitty would instead prompt via `ask_to_read_clipboard` (`kitty/clipboard.py:518`) → `get_boss().confirm(...)` (`kitty/clipboard.py:528`); writes are permitted by default, reads ask.

### 3.4 Non-canonical internal instrumentation — the boundary object itself (Probe B)

The canonical round trip proves the *bytes* cross correctly, but it does not expose the *object* at the boundary to stdout. To observe that object's identity and type, a **non-canonical** probe drives the **identical** production C functions (`vt_parser_create_write_buffer` / `vt_parser_commit_write` / `run_worker`, reached inline via the `Screen` test hooks `test_create_write_buffer` / `test_commit_write_buffer` / `test_parse_written_data` at `kitty/screen.c:4755-4783`) and wires a real `ClipboardRequestManager` exactly as `kitty/window.py:1391` does. **(non-canonical label)** the C routines are the same as production, but they are invoked directly rather than via the launcher's I/O thread + main loop; the values below are object-identity observations, cross-checked against the canonical run in §3.3.

The receiver snapshots each boundary object as the tuple `(type, readonly, data.obj is None, nbytes, is_partial)` before Python copies anything out (`/tmp/ext_probes/B_internal_state.py`). Complete, unedited output — **identical across 2 runs**:

```
--- case=osc52_small ---
callbacks=1 is_partial_true=0 is_partial_false=1 is_partial_None(osc5522)=0
boundary_first=('memoryview', True, True, 10, False)
boundary_last=('memoryview', True, True, 10, False)
backing_types_seen=[] rolled_over=False rollover_at_bytes=None
--- case=osc52_large_18MiB ---
callbacks=25 is_partial_true=24 is_partial_false=1 is_partial_None(osc5522)=0
boundary_first=('memoryview', True, True, 1048570, True)
boundary_last=('memoryview', True, True, 124, False)
backing_types_seen=['BufferedRandom', 'BytesIO'] rolled_over=True rollover_at_bytes=17301417
--- case=osc5522_write ---
callbacks=3 is_partial_true=0 is_partial_false=0 is_partial_None(osc5522)=3
boundary_first=('memoryview', True, True, 10, None)
boundary_last=('memoryview', True, True, 10, None)
backing_types_seen=['BytesIO'] rolled_over=False rollover_at_bytes=None
```

**What this proves (object identity at the boundary):**
- The object is always a **`memoryview`** with **`readonly=True`** and **`data.obj is None`** — i.e. it does not own or keep alive any Python buffer; it is a raw read-only window onto the C address, matching `PyMemoryView_FromMemory(..., PyBUF_READ)` at `kitty/vt-parser.c:461`.
- The `is_partial` routing is exactly as tabulated in §3.2: small OSC 52 → one callback with `is_partial=False`; the 18 MiB OSC 52 → **25 callbacks (24 `is_partial=True` + 1 final `False`)**, each a fresh view over the reused C buffer (max `nbytes=1048570`, just under `BUF_SZ`); OSC 5522 → all 3 callbacks with `is_partial=None`.
- The large case demonstrably crosses the 16 MiB rollover: the `Tempfile` backing transitions `BytesIO → BufferedRandom` at `rollover_at_bytes=17301417` (see §4.1).

---

## 4. SQ-2 — Clipboard crossing, small and large

**Direct answer.** Small and large payloads use the **same** boundary crossing (§3), but diverge in how the Python side *accumulates* them, governed by **two independent size thresholds**:

1. **16 MiB — the rollover threshold** (`WriteRequest.rollover_size = 16 * 1024 * 1024`, `kitty/clipboard.py:237`). The accumulating `Tempfile` begins as an in-memory `io.BytesIO` (`kitty/clipboard.py:29`) and **rolls over to an on-disk `TemporaryFile`** once a write would push its size past this (`rollover_if_needed`, `kitty/clipboard.py:32-36`). This threshold is **reachable and observed**.
2. **512 (MiB) — the `clipboard_max_size` truncation limit** (`kitty/options/definition.py:3111` `opt('clipboard_max_size','512',…)`; type default `clipboard_max_size: float = 512.0`, `kitty/options/types.py:498`). Beyond it, further data is dropped and `max_size_exceeded` is set (`kitty/clipboard.py:321-323`). Through the **real OSC path**, however, this limit is subject to a **double scaling** (below) that makes it effectively unreachable; the *mechanism* is demonstrated with an explicit small limit **(non-canonical)**.

Additionally, large OSC 52 payloads are **chunked by the C parser** at `MAX_ESCAPE_CODE_LENGTH = BUF_SZ/4 = 262144` bytes (256 KiB) (`kitty/vt-parser.c:21`), arriving as a sequence of partial `memoryview`s (`is_partial=True`) followed by a final one; OSC 5522 is **not** C-chunked (only `is_osc_52` matches `"52;"`) and instead carries its own application-level `wdata` records.

### 4.1 OSC 52 — small vs large, with before/during/after `in_flight_write_request`

The canonical proof that both sizes cross correctly is **Probe A** in §3.3 (small `hello` and 20 MiB, `sha_in==sha_out`, integrity=MATCH). To expose the *internal* size-dependent behavior that Probe A cannot print — the `in_flight_write_request` triad, the partial/final callback split, and the `BytesIO`→on-disk rollover byte count — a **non-canonical** instrumentation probe (`/tmp/ext_probes/B2_sizes.py`) drives the **same** real C dispatch (via the `Screen` test hooks, §3.4) and the real `ClipboardRequestManager`, and patches `WriteRequest.commit` to capture the committed length + SHA (checked against the fed bytes). Command and complete, unedited output (OSC 52 cases; **identical across 2 runs**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/B2_sizes.py
=== OSC52 small write "hello" ===
  BEFORE: in_flight_write_request=None
  DURING (first): in_flight_write_request=None
  callbacks=1  is_partial=True:0  is_partial=False:1  is_partial=None:0
  backing transitions (type, bytes_at_transition)=[]
  rollover BytesIO->on-disk first seen at bytes=None
  AFTER: in_flight_write_request='None'
  committed length=5  integrity_ok=True

=== OSC52 LARGE write 20 MiB (crosses 16 MiB rollover) ===
  BEFORE: in_flight_write_request=None
  DURING (first): in_flight_write_request=WriteRequest tempfile.file=BytesIO
  callbacks=27  is_partial=True:26  is_partial=False:1  is_partial=None:0
  backing transitions (type, bytes_at_transition)=[('BytesIO', 786426), ('BufferedRandom', 17301417)]
  rollover BytesIO->on-disk first seen at bytes=17301417
  AFTER: in_flight_write_request='None'
  committed length=20971520  integrity_ok=True
```

**(non-canonical label)** the C routines are identical to production but invoked inline (no I/O thread); the values are cross-checked against the canonical Probe A run in §3.3.

**Reading the output.**
- **Before/during/after triad** (state that changes over time): `in_flight_write_request` is `None` **before**, a live `WriteRequest` (`tempfile.file=BytesIO`) **during** the partial chunks, and `None` again **after** finalization — the `parse_osc_52(..., is_partial=True)` behavior that *keeps* the in-flight request until a non-partial terminator arrives (created/cleared at `kitty/clipboard.py:418-425`). For the **small** `hello` write there is a single `is_partial=False` callback, so the request is created and finalized inside one `parse_osc_52` call and the "during" snapshot never catches a non-`None` value.
- **Small (`hello`, 5 bytes):** exactly **1 callback** (`is_partial=False`), never C-chunked, stays in `BytesIO` (`backing transitions=[]`, no rollover), `committed length=5`.
- **Large (20 MiB):** **27 callbacks = 26 `is_partial=True` + 1 final** — C-chunked because it far exceeds the 256 KiB `MAX_ESCAPE_CODE_LENGTH` (§3.2). The `Tempfile` backing transitions **`BytesIO` → `BufferedRandom`** (an on-disk `TemporaryFile`, `kitty/clipboard.py:35`); the switch is triggered inside `rollover_if_needed` when `tell() + sz > 16 MiB` (`kitty/clipboard.py:32-36`), so the first `tell()` observed *after* the crossing is `17301417` (≈16.5 MiB) — the overshoot is exactly the chunk that crossed the fixed `16777216` (16 MiB) threshold. Integrity holds (`committed length=20971520 integrity_ok=True`), matching the canonical Probe A SHA.

### 4.2 OSC 5522 — small vs large (not C-chunked; application-level `wdata`)

The extended protocol is `<OSC>5522;metadata;payload<ST>` with colon-separated `key=value` metadata and a base64 payload (`docs/clipboard.rst:12-18`). A write is a multi-step transaction (`docs/clipboard.rst:83-89`): `type=write` (start) → one or more `type=wdata:mime=<base64 mime>;<base64 chunk>` (data) → `type=wdata` with no payload (commit), to which the terminal replies `type=write:status=DONE` (`docs/clipboard.rst:97`). Same non-canonical probe as §4.1 (`/tmp/ext_probes/B2_sizes.py`); complete, unedited output (OSC 5522 cases; **identical across 2 runs**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/B2_sizes.py
=== OSC5522 small write ===
  BEFORE: in_flight_write_request=None
  DURING (first): in_flight_write_request=WriteRequest tempfile.file=BytesIO
  callbacks=3  is_partial=True:0  is_partial=False:0  is_partial=None:3
  backing transitions (type, bytes_at_transition)=[('BytesIO', 0)]
  rollover BytesIO->on-disk first seen at bytes=None
  AFTER: in_flight_write_request='None'
  committed length=26  integrity_ok=True
  DONE response(s) sent to child=[b'5522;type=write:status=DONE']

=== OSC5522 LARGE write 20 MiB (crosses 16 MiB rollover) ===
  BEFORE: in_flight_write_request=None
  DURING (first): in_flight_write_request=WriteRequest tempfile.file=BytesIO
  callbacks=109  is_partial=True:0  is_partial=False:0  is_partial=None:109
  backing transitions (type, bytes_at_transition)=[('BytesIO', 0), ('BufferedRandom', 16908288)]
  rollover BytesIO->on-disk first seen at bytes=16908288
  AFTER: in_flight_write_request='None'
  committed length=20971520  integrity_ok=True
  DONE response(s) sent to child=[b'5522;type=write:status=DONE']
```

**Reading the output.** Every OSC 5522 callback carries `is_partial=None` (routing to `parse_osc_5522`, `kitty/clipboard.py:339`), confirming it is **not** C-chunked — each `wdata` record is its own escape, delivered as its own callback. The **small** write is **3 callbacks** (`type=write` / one `type=wdata` / commit `type=wdata`), stays in `BytesIO`, and emits exactly one `type=write:status=DONE` reply to the child (`kitty/clipboard.py:404`). The **large** write arrives as **109 callbacks** (107 `type=wdata` data records at 256 KiB base64 each + `type=write` + commit), and rolls over **`BytesIO` → `BufferedRandom`** at `16908288` bytes ≈ **16.12 MiB** — the crossing record pushes `tell()` just past the fixed 16 MiB. Both preserve integrity (`integrity_ok=True`) and emit one `DONE`. The distinct rollover offset vs OSC 52 (`16908288` vs `17301417`) reflects the different chunk granularity (application-level 256 KiB base64 records here, vs C-parser 256 KiB-triggered partials there).

### 4.3 The truncation threshold — observed double scaling

**Direct answer:** through the real OSC path the documented 512 MiB truncation limit is **effectively unreachable** because the byte count is scaled by `1024*1024` twice. `WriteRequest.max_size` is set to `clipboard_max_size * 1024 * 1024` (already **bytes**) at `kitty/clipboard.py:247`; the guard at `kitty/clipboard.py:321` then compares `tempfile.tell()` against `self.max_size * 1024 * 1024` — scaling **again**.

This is demonstrated as a **failed condition**: a payload **larger than 512 MiB** (520 MiB decoded) is fed as a single genuine OSC 52 escape through a real `Screen` + `kitty/vt-parser.c` (auto-split into partial chunks) into the real `parse_osc_52` with a **default** `WriteRequest`, and *no* truncation occurs. Script `/tmp/ext_probes/C_maxsize_failcondition.py`; complete, unedited output (**identical across 2 runs**, ~4 s):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/C_maxsize_failcondition.py
--- default_512_over_threshold_520MiB (CANONICAL full vt-parser path) ---
clipboard_max_size(option)=512.0
wr.max_size(after L247, bytes)=536870912.0
L321 effective_compare_threshold_bytes=562949953421312.0 (~512 TiB)
bytes_fed_decoded=545259520 bytes_retained_in_tempfile=545259520
retained_equals_fed=True
max_size_exceeded=False
truncation_log_fired=False log_lines=[]
backing=BufferedRandom
--- mechanism_demo_max_size_1_feed_2MiB (LABELED non-canonical) ---
wr.max_size=1 L321_threshold_bytes=1048576
bytes_retained=2097152 max_size_exceeded=True
truncation_log_fired=True log_lines=['Clipboard write request has more data than allowed by clipboard_max_size (1), truncating']
```

**Reading the output.**
- **Canonical failed-condition (default `clipboard_max_size=512.0`):** `wr.max_size` after `kitty/clipboard.py:247` is already `536870912.0` **bytes**; the guard at `kitty/clipboard.py:321` then compares against `536870912.0 * 1024 * 1024 = 562949953421312` bytes ≈ **512 TiB**. Feeding **545,259,520 bytes (520 MiB > 512 MiB)** through the real path, `bytes_retained_in_tempfile=545259520` (`retained_equals_fed=True`), `max_size_exceeded=False`, and `truncation_log_fired=False` — the documented limit **did not fire**. In practice a real clipboard write is bounded only by the **16 MiB rollover** (which merely moves data to an on-disk `BufferedRandom`), not by truncation. This latent double-multiply is reported exactly as observed and is **documented, not fixed** (read-only task; root cause at `kitty/clipboard.py:247` + `:321`).
- **(B) mechanism demo (LABELED non-canonical):** constructing `WriteRequest(max_size=1)` makes the (still double-scaled) bound reachable at `1*1024*1024 = 1048576` bytes (1 MiB). Feeding 2 MiB trips it: `max_size_exceeded=True` and the guard emits `Clipboard write request has more data than allowed by clipboard_max_size (1), truncating` (`kitty/clipboard.py:322`). `bytes_retained=2097152` because the crossing write completes *before* the post-write `tell()` check at `:321`; the flag then blocks **subsequent** writes (guard `if not self.max_size_exceeded` at `:318`). This proves the truncation code path is functional — only the **default** value is unreachable.

### 4.4 Module-level confirmation of the chunk + ownership copy (non-canonical)

```
$ ./kitty/launcher/kitty +launch test.py --module clipboard
test_clipboard_write_request (kitty_tests.clipboard.TestClipboard.test_clipboard_write_request) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
```

This existing test (`kitty_tests/clipboard.py:12-29`, the method `test_clipboard_write_request`) feeds base64 chunk-by-chunk into a `WriteRequest`, asserting the leftover-byte ownership copy `bytes(wr.current_leftover_bytes) == b'aw'` (`kitty_tests/clipboard.py:15`) and `wr.data_for() == b'light work'` (`kitty_tests/clipboard.py:17`). It exercises `WriteRequest` **directly** (not through the escape path), so it is labeled **(non-canonical / module-level)**. The **canonical** SQ-1/SQ-2 evidence is the genuine full-kitty round trip in **§3.3** (Probe A); §3.4, §4.1, §4.2, and §4.3(B) are **non-canonical internal instrumentation** that expose object identity, chunk counts, rollover byte counts, and the truncation mechanism, each cross-checked against the canonical run.

---

## 5. SQ-3 — Transfer under concurrent load

**Direct answer.** Under concurrent load the picture is: **one main thread serializes all parsing, all C→Python callbacks, and rendering, in strict FIFO order**, while a **separate I/O thread keeps reading raw bytes into the parser buffer** the whole time (holding no GIL). Clipboard callbacks are therefore never dropped or reordered by concurrency; they are simply **queued in the shared buffer and delivered in order** as the single consumer works through them. The shared buffer is bounded at **1 MiB (`BUF_SZ`)**; when it fills, kitty applies **backpressure** by removing the child fd from the poll set (`kitty/child-monitor.c:1501`) so the child blocks on `write()` until the main thread drains.

The single main loop is `run_main_loop(process_global_state, …)` (`kitty/child-monitor.c:1259-1262`); each iteration calls `parse_input` and then `render` (`kitty/child-monitor.c:1236`). The I/O thread's `read_bytes` performs `vt_parser_create_write_buffer` → `read(fd, …)` → `vt_parser_commit_write` with **no** Python C-API (`kitty/child-monitor.c:1341-1354`). The structural guarantee that the I/O thread never contends for the GIL is verified in §8.3.

### 5.1 Serialization + in-order + complete delivery (parser level)

A non-canonical parser-level probe (`/tmp/ext_probes/E2_serialize.py`) interleaves **2000 OSC 52 clipboard writes** — each tagged with a monotonically increasing marker `clip-000000 … clip-001999` — with **4096-byte heavy non-clipboard output blocks**, feeds the whole byte stream through the **real** `kitty/vt-parser.c` dispatch (via the `Screen` test hooks) and the real `ClipboardRequestManager`, and records the arrival order of every clipboard callback. Command and complete, unedited output (**identical across 2 runs**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/E2_serialize.py
=== SQ-3 serialization + in-order + complete delivery (parser level), N=3 ===
  per run: stream has 2000 OSC52 clip ops interleaved with 4096-byte heavy blocks (cols=200)
  callbacks_delivered set=[2000] (stable if size 1)
  all_runs_in_order=True
  expected_ops=2000
```

**Scale & stability:** 2000 clipboard ops interleaved with heavy output per run; **N=3 inner runs, repeated across 2 process runs**; **all 2000 callbacks delivered, in strict order, every run** (`callbacks_delivered set=[2000]`, `all_runs_in_order=True`). No loss, no reordering under heavy interleave — the single consumer drains the shared buffer in FIFO order.

### 5.2 Buffer bound + backpressure + delayed-but-complete drain

A non-canonical probe (`/tmp/ext_probes/E_backpressure.py`) drives the **exact** C producer entry points — `vt_parser_create_write_buffer` / `vt_parser_commit_write` — through the `Screen` test hooks, committing 256 KiB at a time **without** consuming (mimicking the I/O thread outrunning a busy main thread) and watching the reported available write space shrink to 0, then reclaiming it with a single parse pass. Command and complete, unedited output (**identical across 2 runs; probe runs N=3 inner iterations**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/E_backpressure.py
=== run 1 ===
  fresh_capacity_bytes=1048576 (== BUF_SZ? True)
  space_trace [committed_KiB -> avail_bytes]:
        0 KiB committed ->  1048576 bytes available
      256 KiB committed ->   786432 bytes available
      512 KiB committed ->   524288 bytes available
      768 KiB committed ->   262144 bytes available
     1024 KiB committed ->        0 bytes available  <-- BACKPRESSURE (no space, reads would stop)
  committed_before_parse=1048576  capacity_after_parse=1048576
=== run 2 ===
  fresh_capacity_bytes=1048576 (== BUF_SZ? True)
  space_trace [committed_KiB -> avail_bytes]:
        0 KiB committed ->  1048576 bytes available
      256 KiB committed ->   786432 bytes available
      512 KiB committed ->   524288 bytes available
      768 KiB committed ->   262144 bytes available
     1024 KiB committed ->        0 bytes available  <-- BACKPRESSURE (no space, reads would stop)
  committed_before_parse=1048576  capacity_after_parse=1048576
=== run 3 ===
  fresh_capacity_bytes=1048576 (== BUF_SZ? True)
  space_trace [committed_KiB -> avail_bytes]:
        0 KiB committed ->  1048576 bytes available
      256 KiB committed ->   786432 bytes available
      512 KiB committed ->   524288 bytes available
      768 KiB committed ->   262144 bytes available
     1024 KiB committed ->        0 bytes available  <-- BACKPRESSURE (no space, reads would stop)
  committed_before_parse=1048576  capacity_after_parse=1048576
--- stability across 3 runs ---
fresh_capacity: [1048576, 1048576, 1048576]
committed_before_parse: [1048576, 1048576, 1048576]
capacity_after_parse: [1048576, 1048576, 1048576]
BUF_SZ = 1048576
```

**Reading the output.** The buffer starts with a full **`1048576` bytes = `BUF_SZ`** of write space (`kitty/vt-parser.c:18`), and each 256 KiB commit shrinks the space reported by `vt_parser_create_write_buffer` (`*sz = BUF_SZ - (read.sz + write.pending)`, `kitty/vt-parser.c:1457`) — `1048576 → 786432 → 524288 → 262144 → 0`. At **0** the buffer is full: this is the backpressure point where `vt_parser_has_space_for_input` (`read.sz + write.pending < BUF_SZ`, `kitty/vt-parser.c:1481`) returns false, `read_bytes` bails immediately (`if (!available_buffer_space) return true;`, `kitty/child-monitor.c:1342`), and the poll loop clears `POLLIN` on the child fd (`kitty/child-monitor.c:1501`) so the child blocks on `write()`. A single `test_parse_written_data` (the main-thread consume) then reclaims the **full `1048576`** bytes (`capacity_after_parse=1048576`). This is the mechanical core of "in practice" under load: the producer is **throttled, not dropped** — bytes are held in the bounded buffer and drained in order the moment the consumer runs. Perfectly stable across 3 inner runs and 2 process runs.

### 5.3 Full-kitty end-to-end under a concurrent output flood (canonical)

This is the **canonical** end-to-end confirmation: the **real** `kitty` launcher runs under `xvfb`, and a child shell (`/tmp/ext_probes/D_child_flood.sh`) emits an **8,000-line output flood** (two 4,000-line bursts of normal terminal text) with a **genuine OSC 52 clipboard write of a ~4 MiB (4,194,304-byte) deterministic payload sandwiched in the middle**, then reads it back with the real `kitten clipboard --get-clipboard`. Integrity is checked by SHA-256 of payload-in vs. readback-out. The child emits the escape directly:

```sh
# excerpt of D_child_flood.sh — genuine OSC 52 write through the PTY, mid-flood
# (two 4,000-line flood loops surround this write; full script in /tmp/ext_probes):
{ printf '\033]52;c;'; base64 -w0 "$PAY"; printf '\007'; }   # OSC 52 write of ~4 MiB payload
# (4,000-line flood loop here — elided; see /tmp/ext_probes/D_child_flood.sh)
kitten clipboard --get-clipboard > "/tmp/ext_probes/D_out_${RUN}.txt"   # genuine read-back
```

Driver (real launcher under xvfb, 5 runs) and complete, unedited output — **all 5 runs**:

```
$ bash /tmp/ext_probes/D_driver.sh
# each run: xvfb-run -a -s "-screen 0 1280x800x24" \
#   $REPO/kitty/launcher/kitty -o close_on_child_death=yes \
#   -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
#   sh /tmp/ext_probes/D_child_flood.sh $RUN
kitty_exit=0 | run=1 bytes_in=4194304 bytes_out=4194304 integrity=MATCH wall_s=0.719
kitty_exit=0 | run=2 bytes_in=4194304 bytes_out=4194304 integrity=MATCH wall_s=0.721
kitty_exit=0 | run=3 bytes_in=4194304 bytes_out=4194304 integrity=MATCH wall_s=0.725
kitty_exit=0 | run=4 bytes_in=4194304 bytes_out=4194304 integrity=MATCH wall_s=0.720
kitty_exit=0 | run=5 bytes_in=4194304 bytes_out=4194304 integrity=MATCH wall_s=0.721
```

**Scale & stability:** a ~4 MiB clipboard payload crossed concurrently with an 8,000-line flood, per run; **N=5 runs**. Every run: `kitty_exit=0`, `bytes_in == bytes_out == 4194304`, `integrity=MATCH`. In-window round-trip wall time (measured by the child, so it excludes `xvfb`/GLFW startup): `wall_s` = `[0.719, 0.721, 0.725, 0.720, 0.721]` → **min 0.719 s, median 0.721 s, max 0.725 s, mean 0.7212 s, N=5** — tightly stable. Under a real concurrent flood the clipboard transfer is **delayed but never lost or corrupted**: this confirms end-to-end the serialization behavior demonstrated at the parser level in §5.1–§5.2.

> **Nuance — parse gating (`input_delay`).** The main loop does not necessarily parse on every wakeup: `run_worker` gates parsing on `OPT(input_delay)` (default **3 ms**, `docs/performance.rst:48`) unless a flush is forced or the buffer is nearly full (`kitty/vt-parser.c:1425`). Under a steady byte stream this batches parsing into ~3 ms quanta, which is what "the transfer looks like in practice" for interactive throughput.

---

## 6. SQ-4 — Expensive scrollback scan → event delivery

**Direct answer.** **Yes** — an expensive scrollback scan **delays** the delivery of events/callbacks to kittens by approximately **the scan's own duration**, and never loses them. The evidence separates cleanly into what is *directly observed* and what is *inferred*:

- **Directly observed.** All five history-scan functions run on the **single main thread** and none releases the GIL (zero `Py_BEGIN_ALLOW_THREADS` in `kitty/history.c`, §8.3). For the **monolithic** scans — `as_text`/`__str__`, `pagerhist_as_bytes`, `pagerhist_as_text` — even a *separate* Python thread is starved for ~the **entire** scan (`gap/scan ≈ 0.87–1.01`). The **callback-driven** scans — `as_ansi`, `as_text_for_history_buf` — invoke a Python callback per line, creating periodic GIL switch points, so a *separate* thread interleaves at ~`sys.getswitchinterval()` cadence (`worst gap ≈ 6.2 ms`, `gap/scan ≈ 0.08–0.11`).
- **Inferred (labeled INFERRED).** Real **kitten event delivery** happens on the **same main thread** as the scan (kittens are separate processes reached only through main-thread escape-code I/O). It is therefore serialized **behind the entire scan regardless** of the monolithic-vs-callback distinction — the per-line GIL yields go to *other threads*, never to the main thread's own event loop, so a kitten's event is delayed by ~the **full** scan duration in **all five** cases. This is inferred from the single-main-thread + no-GIL-release model (§8.3), not from injecting a real kitten dispatch mid-scan.

### 6.1 Method

Probe `/tmp/ext_probes/F_scan_delay.py` builds a **120,000-line × 80-col** `HistoryBuf`, then measures, for each of the **five** named scan functions:

1. **(Primary, direct)** the raw scan duration timed around the call, **N=5** — the magnitude of the main-thread stall, measured with no proxy.
2. **(Secondary, proxy)** a background "heartbeat" thread ticking ~1 ms; its worst inter-tick gap during a scan is an explicit **PROXY** for "any other Python work on the interpreter." Real kitten delivery is **INFERRED** (see Direct answer).

The five functions, by name and location: `as_text`/`__str__` (`kitty/history.c:321`), `as_ansi` (`kitty/history.c:348`), `pagerhist_as_bytes` (`kitty/history.c:461`), `pagerhist_as_text` (`kitty/history.c:486`), and `as_text_for_history_buf` (`kitty/screen.c:3495-3496`, which calls `as_text_history_buf`, `kitty/history.c:509`).

### 6.2 DIRECT scan durations (primary, canonical), N=5 × 2 runs

Command and complete, unedited output — section (1) of **both** process runs:

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/F_scan_delay.py
[run 1]
setup: HistoryBuf 120000x80 built in 158 ms; hb.count=120000; screen scrollback lines fed=120000
=== (1) DIRECT scan durations (ms), N=5 each ===
  as_text(__str__)  history.c:321
      runs_ms=[61.1, 56.3, 55.6, 56.7, 59.2] min=55.6 median=56.7 max=61.1
  as_ansi           history.c:348
      runs_ms=[64.2, 64.4, 64.0, 64.2, 63.7] min=63.7 median=64.2 max=64.4
  pagerhist_as_bytes history.c:461
      runs_ms=[30.1, 28.3, 29.3, 28.8, 30.8] min=28.3 median=29.3 max=30.8
  pagerhist_as_text  history.c:486
      runs_ms=[57.1, 55.8, 56.2, 54.5, 54.8] min=54.5 median=55.8 max=57.1
  as_text_for_history_buf screen.c:3495
      runs_ms=[58.8, 62.4, 61.1, 66.0, 59.0] min=58.8 median=61.1 max=66.0
[run 2]
setup: HistoryBuf 120000x80 built in 167 ms; hb.count=120000; screen scrollback lines fed=120000
=== (1) DIRECT scan durations (ms), N=5 each ===
  as_text(__str__)  history.c:321
      runs_ms=[67.2, 59.0, 57.4, 57.9, 56.9] min=56.9 median=57.9 max=67.2
  as_ansi           history.c:348
      runs_ms=[66.6, 66.5, 65.2, 65.5, 68.8] min=65.2 median=66.5 max=68.8
  pagerhist_as_bytes history.c:461
      runs_ms=[29.5, 28.5, 28.0, 28.8, 27.7] min=27.7 median=28.5 max=29.5
  pagerhist_as_text  history.c:486
      runs_ms=[55.0, 53.9, 56.6, 53.6, 54.1] min=53.6 median=54.1 max=56.6
  as_text_for_history_buf screen.c:3495
      runs_ms=[58.3, 62.1, 61.2, 59.5, 63.4] min=58.3 median=61.2 max=63.4
```

**Reading the output.** At 120,000 lines, the median direct scan cost is **`as_text`/`__str__` ~57 ms, `as_ansi` ~64–67 ms, `as_text_for_history_buf` ~61 ms, `pagerhist_as_text` ~54 ms, `pagerhist_as_bytes` ~29 ms**, stable across both runs. That per-scan cost is the **magnitude of the main-thread stall**: for the whole of it, no other main-thread Python — including kitten event delivery — runs. `pagerhist_as_bytes` is cheapest because it copies a pre-serialized ring buffer rather than reconstructing text per line.

### 6.3 Heartbeat PROXY + monolithic-vs-callback nuance, N=5 × 2 runs

The heartbeat is a **PROXY** for other-thread Python work. Its worst inter-tick gap during a scan proves whether the scan holds the GIL monolithically or yields it per line. Command and complete, unedited output — section (2) of **both** process runs of the same probe invocation as §6.2 (the NOTE is emitted verbatim by the probe):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/F_scan_delay.py
[run 1]
=== (2) heartbeat thread: worst inter-tick gap during a scan (DIRECT GIL-starvation proof) ===
  NOTE: the heartbeat runs on a SEPARATE thread (a PROXY for "other Python work").
  Real kitten event delivery is on the SAME main thread as the scan, so it is
  serialized behind the ENTIRE scan regardless (INFERRED from the single-main-thread
  + no-GIL-release model). Pure-C scans (no per-line callback) hold the GIL for the
  whole scan -> even a separate thread is starved ~the full scan. Callback-driven
  scans (as_ansi, as_text_for_history_buf) invoke Python per line, creating periodic
  GIL-switch points, so a SEPARATE thread interleaves (~switchinterval); the SAME
  main thread still waits the full scan.
  as_text(__str__)  history.c:321  [MONOLITHIC (holds GIL whole scan)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(70.0, 70.6), (59.0, 59.0), (59.0, 59.1), (56.9, 56.8), (59.3, 60.4)]
      scan median=59 ms; worst_gap median=59.1 ms; gap/scan=1.00
  as_ansi           history.c:348  [CALLBACK-DRIVEN (yields GIL per line to other threads)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(69.5, 6.2), (68.1, 6.2), (69.2, 6.2), (65.3, 6.2), (115.6, 6.3)]
      scan median=69 ms; worst_gap median=6.2 ms; gap/scan=0.09
  pagerhist_as_bytes history.c:461  [MONOLITHIC (holds GIL whole scan)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(39.1, 35.0), (38.2, 33.1), (38.1, 33.3), (38.8, 34.3), (31.5, 28.5)]
      scan median=38 ms; worst_gap median=33.3 ms; gap/scan=0.87
  pagerhist_as_text  history.c:486  [MONOLITHIC (holds GIL whole scan)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(55.8, 52.5), (56.5, 53.2), (56.6, 53.3), (73.3, 68.6), (64.6, 61.7)]
      scan median=57 ms; worst_gap median=53.3 ms; gap/scan=0.94
  as_text_for_history_buf screen.c:3495  [CALLBACK-DRIVEN (yields GIL per line to other threads)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(60.5, 6.2), (60.4, 6.2), (62.7, 6.2), (60.7, 6.2), (62.3, 6.2)]
      scan median=61 ms; worst_gap median=6.2 ms; gap/scan=0.10
[run 2]
=== (2) heartbeat thread: worst inter-tick gap during a scan (DIRECT GIL-starvation proof) ===
  NOTE: the heartbeat runs on a SEPARATE thread (a PROXY for "other Python work").
  Real kitten event delivery is on the SAME main thread as the scan, so it is
  serialized behind the ENTIRE scan regardless (INFERRED from the single-main-thread
  + no-GIL-release model). Pure-C scans (no per-line callback) hold the GIL for the
  whole scan -> even a separate thread is starved ~the full scan. Callback-driven
  scans (as_ansi, as_text_for_history_buf) invoke Python per line, creating periodic
  GIL-switch points, so a SEPARATE thread interleaves (~switchinterval); the SAME
  main thread still waits the full scan.
  as_text(__str__)  history.c:321  [MONOLITHIC (holds GIL whole scan)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(58.9, 59.0), (57.4, 57.5), (59.3, 59.4), (73.8, 74.8), (77.7, 78.6)]
      scan median=59 ms; worst_gap median=59.4 ms; gap/scan=1.00
  as_ansi           history.c:348  [CALLBACK-DRIVEN (yields GIL per line to other threads)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(65.9, 6.2), (67.9, 6.2), (65.0, 6.2), (65.5, 6.1), (64.8, 6.2)]
      scan median=66 ms; worst_gap median=6.2 ms; gap/scan=0.09
  pagerhist_as_bytes history.c:461  [MONOLITHIC (holds GIL whole scan)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(30.1, 26.9), (37.1, 33.3), (31.4, 27.7), (30.8, 27.5), (30.2, 26.8)]
      scan median=31 ms; worst_gap median=27.5 ms; gap/scan=0.89
  pagerhist_as_text  history.c:486  [MONOLITHIC (holds GIL whole scan)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(57.4, 54.9), (57.6, 54.3), (56.5, 54.0), (58.2, 54.6), (55.4, 52.1)]
      scan median=57 ms; worst_gap median=54.3 ms; gap/scan=0.95
  as_text_for_history_buf screen.c:3495  [CALLBACK-DRIVEN (yields GIL per line to other threads)]
      (scan_ms, worst_heartbeat_gap_ms) x5 = [(58.4, 6.2), (58.7, 6.2), (59.6, 6.2), (57.7, 6.2), (58.1, 6.2)]
      scan median=58 ms; worst_gap median=6.2 ms; gap/scan=0.11
```

**Invariant across the distribution.** The **classification and the gap/scan ratio are the stable invariants** across both runs (and a third confirming run): monolithic scans starve a separate thread for ~the whole scan (`__str__` `gap/scan=1.00`; `pagerhist_as_text` `0.94–0.95`; `pagerhist_as_bytes` `0.87–0.89`), while callback-driven scans let a separate thread interleave (`as_ansi` and `as_text_for_history_buf` both `~6.2 ms` worst gap, `gap/scan ≈ 0.09–0.11`). Absolute scan times drift with allocator/cache warmth (as the magnitude/timing rule anticipates), but the relationship is reproducible. **Crucially, the callback-driven per-line yields benefit *other threads*, not the main loop** — so, as stated in the Direct answer, kitten event delivery (same main thread) is delayed by ~the full scan duration in every case (INFERRED).

---

## 7. SQ-5 — Expensive scrollback scan → memory management

**Direct answer.** Memory has **two distinct parts**. (1) The **persistent, dominant** cost is the **C-side scrollback storage**: `HistoryBuf` holds cells in fixed **2048-line segments** (`SEGMENT_SIZE`, `kitty/history.c:15`), each segment allocated by a **single `calloc`** in `add_segment` (`kitty/history.c:18-28`) sized `xnum·2048·sizeof(CPUCell) + xnum·2048·sizeof(GPUCell) + 2048·sizeof(LineAttrs)` (`kitty/history.c:23-25`). At `xnum=80` with `sizeof(CPUCell)=12`, `sizeof(GPUCell)=20`, `sizeof(LineAttrs)=4`, that is exactly **5,251,072 bytes (~5.01 MiB) per 2048-line segment** — and the measured RSS step per segment matches that theory to a ratio of **1.00**. (2) A scan additionally materializes a **transient, owned Python object** on the Python heap (measured **~4.86 MB** for `as_text`, **~16.78 MB** for `pagerhist_as_bytes`/`pagerhist_as_text` at 60,000 lines × 80 cols); this object is freed once the kitten consumes it. The scan does not change how the C scrollback itself is managed — it **reads** the segmented C storage and **materializes a Python copy**.

### 7.1 Python object footprint of each scan (owned copies), N=2

Probe `/tmp/ext_probes/G_memory.py` builds a 60,000-line × 80-col `HistoryBuf`, runs three scans, and reports the retained size (`sys.getsizeof`) of each produced Python object. Command and complete, unedited output (**identical across 2 process runs; N=2 inner runs each**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/G_memory.py
=== (1) Python OBJECT footprint produced by scans (owned copies), N=2 ===
  run1: hb.count=60000 xnum=80
    as_text(str)       len_chars=4859999   sys.getsizeof=4860040 bytes
    pagerhist_as_bytes len_bytes=16777206  sys.getsizeof=16777239 bytes
    pagerhist_as_text  len_chars=16777206  sys.getsizeof=16777247 bytes
  run2: hb.count=60000 xnum=80
    as_text(str)       len_chars=4859999   sys.getsizeof=4860040 bytes
    pagerhist_as_bytes len_bytes=16777206  sys.getsizeof=16777239 bytes
    pagerhist_as_text  len_chars=16777206  sys.getsizeof=16777247 bytes
```

**Reading the output.** The retained sizes are **bit-for-bit identical across both inner runs and both process runs** (deterministic): `as_text` produces a `str` of **4,860,040 bytes**, while both `pagerhist_*` scans produce **~16.78 MB** objects (the pager-history ring buffer is larger than the visible-line reconstruction here). These are the **transient** allocations a scan adds; they are dwarfed by the persistent C storage below and are released after consumption. *(Inferred from the code: `pagerhist_as_text` decodes the `bytes` result to `str`, so both representations coexist momentarily — a transient ~2× construction peak — but this is not separately measured here and is labeled inferred.)*

### 7.2 C-side segment growth (RSS step per 2048-line segment), N=3

Section (2) of the same probe grows the buffer one 2048-line segment at a time and records the **current** RSS delta per segment (`/proc/self/statm`, avoiding the monotonic high-water bias of `ru_maxrss`). Complete, unedited output (**identical deltas across 2 process runs; N=3 inner runs**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/G_memory.py
=== (2) C-side HistoryBuf segment growth (RSS step per 2048-line segment), N=3 ===
  run1: base_rss=34377728 bytes; per-segment RSS deltas (bytes) = [5111808, 5251072, 5251072, 5251072, 5251072, 5251072, 5251072, 5251072]
  run2: base_rss=39616512 bytes; per-segment RSS deltas (bytes) = [0, 5124096, 5251072, 5251072, 5251072, 5251072, 5251072, 5251072]
  run3: base_rss=39616512 bytes; per-segment RSS deltas (bytes) = [0, 5124096, 5251072, 5251072, 5251072, 5251072, 5251072, 5251072]
  theoretical_per_segment_bytes=5251072 (~5.01 MiB)
  observed per-segment RSS delta: min=0 median=5251072.0 max=5251072 n=24
  median_observed/theoretical = 1.00
```

**Reading the output.** Each new segment adds a stable **5,251,072-byte** RSS step, matching the `add_segment` `calloc` formula (`kitty/history.c:23-25`) exactly — **median observed / theoretical = 1.00** across `n=24` samples. The occasional leading `0` (run 2/3 first delta) is the constructor pre-allocating segment 0 before measurement, and the first non-zero step is slightly under theory (`5,111,808`/`5,124,096`) because the initial `realloc` reuses already-resident pages; steady-state steps are exactly `5,251,072`. This is the **persistent** memory the scan competes with, and it grows **linearly and predictably** — one fixed 5.01 MiB block per 2048 lines. Preserving the user's framing: the scan is an **expensive main-thread operation competing for the GIL** with clipboard/event delivery — it delays delivery (SQ-4) and briefly adds a Python-heap object (§7.1), but it does not alter the segment-based C management of the scrollback itself.

---

## 8. SQ-6 — Where timing, concurrency, and object ownership begin to matter

**Direct answer.** There are exactly three loci:

1. **Concurrency / timing — the parser lock over a single producer/consumer buffer.** The parser owns one `BUF_SZ` (1 MiB) buffer partitioned into a *read* region (consumed by the main thread) and a *write* region (filled by the I/O thread), synchronized by one `pthread_mutex_t lock` (`kitty/vt-parser.c:206`).
2. **Timing — the GIL + `input_delay` gate on the single main thread** (already shown in §5–§7).
3. **Object ownership — the RAII lifetime of the boundary `memoryview`.**

### 8.1 The parser lock and the producer/consumer partition

Complete, unedited source — the two lock macros plus `run_worker` in full (`kitty/vt-parser.c:1413-1446`):

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

The producer side, also under the lock — the three functions in full (`kitty/vt-parser.c:1450-1484`):

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

The I/O producer call site itself is the whole of `read_bytes`, reproduced **complete and unedited** (`kitty/child-monitor.c:1336-1356`). It calls only `vt_parser_create_write_buffer` / `read()` / `vt_parser_commit_write` — **no Python C-API**:

```c
static bool
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;

    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;

    while(true) {
        len = read(fd, buf, available_buffer_space);
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);
    return len != 0;
}
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
$ grep -rn "Py_BEGIN_ALLOW_THREADS\|PyEval_SaveThread\|PyGILState_Ensure" kitty/*.c
kitty/utmp.c:17:    Py_BEGIN_ALLOW_THREADS

$ for f in child-monitor.c vt-parser.c screen.c history.c; do \
      echo "kitty/$f: $(grep -c 'Py_BEGIN_ALLOW_THREADS\|PyEval_SaveThread\|PyGILState_Ensure' kitty/$f)"; done
kitty/child-monitor.c: 0
kitty/vt-parser.c: 0
kitty/screen.c: 0
kitty/history.c: 0
```

The **only** GIL-releasing site across the entire C core is in `kitty/utmp.c` (utmp handling), not in the parser, child-monitor, screen, or history code. In particular the two files central to this investigation's load scenarios — `kitty/history.c` (the expensive scrollback scan, SQ-4/SQ-5) and `kitty/child-monitor.c` (the I/O thread, SQ-3) — contain **zero** GIL-releasing calls, so a scan on the main thread holds the GIL for its entire duration. Consequently, when the main thread holds the GIL for an expensive scan, the I/O thread keeps buffering bytes but no Python runs concurrently — this is the unifying fact behind SQ-3, SQ-4, and SQ-7.

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

A non-canonical internal-instrumentation probe (`/tmp/ext_probes/H_race_ownership.py`) drives the **real** `kitty/vt-parser.c` dispatch (via the `Screen` test hooks) and **deliberately retains** the boundary `memoryview` from the first OSC 52 callback **without copying**, then feeds a **second** OSC 52 escape (a different payload) through the **same** 1 MiB parser buffer, and re-reads the retained view. Command and complete, unedited output (Part A; **identical across 2 runs**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/H_race_ownership.py
=== PART A: retained-view C-buffer-reuse hazard (SQ-7) ===
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

The clipboard manager never keeps the transient view. Undecodable base64 remainder is copied **out** of the view into owned bytes at `kitty/clipboard.py:286` (`self.current_leftover_bytes = memoryview(bytes(mv[-extra:]))` — the inner `bytes(...)` makes an owned copy of the last `extra` bytes). Part B of the same probe shows this two ways: **(B1)** a canonical reproduction of `kitty_tests/clipboard.py:12-17` (feeding `'bGlnaHQgd29yaw'` leaves a 2-byte leftover `b'aw'`, and after flush `data_for()` is `b'light work'`), and **(B2)** a source-mutation proof — the leftover is captured, the **mutable source is then overwritten**, and the leftover is re-read. Command and complete, unedited output (Part B, same invocation as §9.1; **identical across 2 runs**):

```
$ ./kitty/launcher/kitty +launch /tmp/ext_probes/H_race_ownership.py
=== PART B: kitty's ownership COPY avoids the hazard (SQ-7) ===
B1 (canonical, kitty_tests/clipboard.py:12-17):
  full base64 fed: b'bGlnaHQgd29yaw'
  current_leftover_bytes: b'aw'  (expected b'aw')
  data_for(): b'light work'  (expected b'light work')
B2 (source-mutation proof of owned copy at clipboard.py:286):
  current_leftover_bytes BEFORE source mutation: b'aw'
  source bytearray mutated tail -> b'ZZ'
  current_leftover_bytes AFTER  source mutation: b'aw' -> UNCHANGED => OWNED copy, not a view
```

In **B1**, the leftover `b'aw'` and final `b'light work'` match the in-repo unit test exactly (`kitty_tests/clipboard.py:15,17`). In **B2**, `current_leftover_bytes` stays `b'aw'` even after the source `bytearray`'s tail is overwritten with `b'ZZ'` — proving the `bytes(...)` at `kitty/clipboard.py:286` produced an **owned copy**, not a view onto the caller's buffer. Decoded data likewise goes into an owned `Tempfile` (§4). So in production the retained-view condition of §9.1 **never arises**.

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
| **SQ-4** Scan → event delivery | §6 | direct scan durations N=5×2 (all 5 scans by name); heartbeat PROXY (monolithic vs callback-driven, gap/scan 0.09–1.00); kitten delivery **INFERRED** |
| **SQ-5** Scan → memory | §7 | Python object footprint N=2 (`getsizeof`); C-side segment growth N=3 (5,251,072 B/segment, observed/theoretical = 1.00) |
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

To satisfy the read-only mandate, **every** temporary script was written under `/tmp/ext_probes/` — **outside** the repository working tree (`git status` therefore never lists any of them) — and removed after evidence capture. The actual scripts, by name and sub-question:

| Script (`/tmp/ext_probes/`) | Canonical? | Purpose (sub-question) |
|---|---|---|
| `A_canonical_roundtrip.sh`, `A_driver.sh` | **canonical** | Full-kitty `xvfb` OSC 52 small/large (SHA integrity) + OSC 5522 round-trip through a real child PTY, read back via the clipboard kitten (SQ-1, SQ-2) |
| `child_rt.sh`, `child_large.sh`, `child_5522.sh` | canonical | Child helper scripts emitting the genuine OSC escapes for the round-trip / large / OSC 5522 cases (SQ-1, SQ-2) |
| `B_internal_state.py` | non-canonical | Internal instrumentation: boundary `memoryview` props, `is_partial` routing, `in_flight_write_request` before/during/after, `Tempfile` `BytesIO`→`BufferedRandom` rollover (SQ-2) |
| `B2_sizes.py` | non-canonical | Per-callback lengths + SHA, exact rollover byte counts, OSC 5522 `DONE` replies (SQ-2) |
| `C_maxsize_failcondition.py` | canonical + labeled demo | 512-MiB default-path failed-condition (no truncation, double-scaling) + labeled `max_size=1` mechanism demo (SQ-2) |
| `D_child_flood.sh`, `D_driver.sh` | **canonical** | Full-kitty concurrent flood: 8,000-line burst around a genuine OSC 52 ~4 MiB write + read-back, 5 runs under `xvfb` (SQ-3) |
| `E_backpressure.py` | non-canonical | `BUF_SZ` single-buffer bound + backpressure trace to 0 then reclaim (SQ-3, SQ-6) |
| `E2_serialize.py` | non-canonical | 2000 OSC 52 ops interleaved with heavy blocks; in-order + complete delivery (SQ-3) |
| `F_scan_delay.py` | non-canonical | Five history scans: direct durations N=5 + heartbeat PROXY + monolithic-vs-callback classification (SQ-4) |
| `G_memory.py` | non-canonical | Python object footprint N=2 + C-side segment-growth RSS deltas N=3 (SQ-5) |
| `H_race_ownership.py` | non-canonical | Retained-view aliasing hazard + ownership `bytes()` copy with source-mutation proof (SQ-7) |
| `sizeof_probe.c` | build check | `sizeof(CPUCell/GPUCell/LineAttrs)` for the per-segment memory formula (SQ-5) |
| `diag_osc52.py`, `diag_realparser.py`, `mini.py` | diagnostic | Supporting diagnostics for the OSC 52 partial-chunk re-framing mechanism (SQ-2 supporting) |

Non-canonical probes drive the **same** real C functions (`vt_parser_create_write_buffer` / `vt_parser_commit_write` / the parse worker) and the **same** real `ClipboardRequestManager`/`WriteRequest` as production, but inject bytes through the `Screen` test hooks rather than the PTY I/O thread; they expose internal object states not observable from outside. The `WriteRequest(max_size=1)` demo in §4.3, the `test.py --module clipboard` run in §4.4, and every internal-instrumentation probe are labeled **non-canonical** in the text; the **canonical** SQ-1/SQ-2/SQ-3 evidence is the genuine OSC-escape-through-PTY output from the `A_*` and `D_*` probes.

### 11.2 Read-only / cleanup proof

The repository tree is unchanged apart from this single new document. The authoritative proof is a name-status diff against the **baseline commit** — `815df1e21`, the original branch tip (the branch is named after it, `kitty_815df1e210e0`), which is the parent of the deliverable commit:

```
$ git log --oneline -1 815df1e21
815df1e21 Wire up applying of font config

# The ONLY change to the entire repository versus that baseline:
$ git diff --name-status 815df1e21
A	blitzy/documentation/kitty_815df1e210e0.md

# Authored by the Blitzy agent, not a source maintainer:
$ git log -1 --format='%an <%ae>' -- blitzy/documentation/kitty_815df1e210e0.md
Blitzy Agent <agent@blitzy.com>
```

The name-status diff contains **exactly one line**, `A blitzy/documentation/kitty_815df1e210e0.md` — a single **A**(dded) file, with **no** `M`(odified) or `D`(eleted) entries anywhere. No existing source, test, build, or documentation file was modified, added, or deleted. The `blitzy/screenshots` and `blitzy/screen_recordings` directories are **pre-existing, empty platform scratch directories** (used by the tooling for optional screenshots/recordings); being empty, git does not track them and they never appear in `git diff`/`git status`. Build artifacts (`kitty/fast_data_types.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`) are git-ignored and do not appear. All temporary observation scripts lived under `/tmp/ext_probes/` — **outside** the repository — and were deleted after evidence capture, leaving the working tree identical to baseline apart from this one document.

*End of document.*
